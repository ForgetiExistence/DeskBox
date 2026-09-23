# DeskBox 卡顿 / 延迟根因分析

> 本文是一份**独立第三方代码审查**，非 DeskBox 官方文档。
> 配套文档：[`optimization-plan.md`](optimization-plan.md)

- **对象**：https://github.com/Tianyu199509/DeskBox （v1.5.5，GPL-3.0，作者 Tianyu Zhu）
- **方法**：静态代码分析。拉取 `main` 分支源码包（1511 个文件，`src/DeskBox` 下 625 个 C# 文件，约 17 万行），逐条追踪卡顿相关链路。
- **说明**：本文未运行程序，也未做性能采样。所有结论均附代码位置（`文件:行号`）供复核。文末列出可直接验证的指标。

---

## 一、结论摘要

DeskBox 的卡顿**不是某一处 bug，而是三类机制叠加**：一个被放大的刷新链路、一批被限流的异步管线、以及一个和「激进省内存」相冲突的全局输入钩子。

按对用户观感的贡献排序：

| # | 根因 | 机制类型 | 观感 | 严重度 |
|---|------|----------|------|--------|
| 1 | 索引监视通道**强制全量重载**，把细粒度增量路径整体旁路 | 卡顿（主线程被占） | 任意文件变动后界面顿一下 | **高** |
| 2 | 列表重建是 **O(n²)**，且每个元素变更都逐条触发 UI 集合事件 | 卡顿 | 大文件夹（数百项以上）明显掉帧 | **高** |
| 3 | 图标管线被限流到 **2 个 Shell 并发槽位**，超时后进 30 秒黑名单 | 延迟 | 图标「一个一个往外冒」，甚至长时间显示通用图标 | **高** |
| 4 | 全局低级输入钩子（鼠标 + 键盘）与「空闲换出工作集」策略冲突 | 卡顿（整机输入） | 偶发的、看起来随机的系统级点击/按键迟钝 | **中**（仅用户开启相关功能时） |
| 5 | `settings.json` 的**序列化在进程级全局锁内**，而设置读取走同一把锁 | 卡顿 | 批量操作期间界面间歇性停顿 | **中** |
| 6 | 一批**设计内延迟**（悬停展开 360ms、去抖 250ms/1s） | 延迟 | 「慢半拍」的手感，不是卡，但被归入「卡」 | **中** |
| 7 | 若干次要问题：埋点热路径字符串分配、双 `ItemsControl` 绑定、`RepositionThemeTransition` | 轻微 | 边际 | 低 |

关键判断：**第 1 条是全局放大器**。它把 3、4、5、6 的代价从「偶发」变成「每次文件变动都付一遍」。

---

## 二、项目自述的性能史（先给作者记一分）

`CHANGELOG.md` 里性能相关条目异常密集，说明作者一直在正面处理这个问题，且修复过多次真实卡顿：

- **1.5.5**：新增「空闲时修剪内存」；热键自愈看门狗。
- **1.5.4**：修复「导入/整理/剪切数千文件后私有内存逼近 1 GB 直到重启」；修复「从格子剪切大量文件时的可见卡顿」；把 `settings.json` 里约 20 MB 的臃肿撤销历史收缩到 100 KB 以下；用二分插入的作用域路径索引替代逐文件扫描。
- **1.5.1**：大文件夹改为窗口化渲染（不再一次布局数千个磁贴）；**把原生右键菜单的鼠标钩子从 UI 线程移到专职泵线程**——此前「点击原生菜单项会令整机卡顿 1-2 秒」；新装默认「节省资源」预设。
- **1.4.x**：把格子托盘动画从固定 16ms 定时器换成 `CompositionTarget.Rendering`（跟随 VSync）；把 CPU 驱动的透明度/缩放动画换成 GPU Composition 关键帧；用 `SetWindowPos` 替代 `AppWindow.Move` 降低延迟。

**但这恰恰是问题的旁证**：一个产品的 changelog 需要连续五个版本都在修卡顿，说明它的**架构在持续制造卡顿**。下面逐条拆。

---

## 三、根因链路

### 3.1 双通道监视器 + 索引通道强制全量重载

`FolderWatcherService` 对每个受管文件夹同时开三条监视：

| 监视器 | 类型 | 作用 | 位置 |
|--------|------|------|------|
| `_legacyWatcher` | `FileSystemWatcher`，缓冲 32 KB，非递归 | 提供细粒度变更 | `FolderWatcherService.cs:57,497` |
| `_queryWatcher` | WinRT `StorageItemQueryResult.ContentsChanged` | 「索引通道」兜底 | `FolderWatcherService.cs:291-339` |
| `_desktopIniWatcher` | `FileSystemWatcher`，递归，只盯 `desktop.ini` | 检测文件夹图标被改 | `FolderWatcherService.cs:387-412` |

代码注释明确说这是**故意重叠**的：

> *The two mechanisms intentionally overlap: Explorer operations can overflow a native watcher buffer, while indexed queries can lag or omit folder-only changes.*

问题出在重叠的代价上。WinRT 的 `ContentsChanged` **不携带任何变更细节**，只能当作「有东西变了」，于是：

```csharp
// FolderWatcherService.cs:367-379
private void OnQueryContentsChanged(IStorageQueryResultBase sender, object args)
{
    ...
    // StorageFileQueryResult.ContentsChanged does not provide details
    // about what changed — it only signals that something in the folder
    // changed.  We treat this as a full-reload signal.
    QueueFullReload(generation);
}
```

而 `QueueFullReload` 做的事是**置位一个标志**（`:890-906`），该标志最终进入 `RequiresFullReload`，然后在消费侧被这样使用：

```csharp
// WidgetViewModel.SortingAndWatchers.cs:419-439
private bool ShouldUseFullReload(FolderChangeBatch changeBatch, string mappedFolderPath)
{
    if (changeBatch.RequiresFullReload || changeBatch.Changes.Count == 0
        || changeBatch.Changes.Count > IncrementalRefreshBatchThreshold)   // 24
        return true;

    var (userDesktop, publicDesktop) = FileService.GetDesktopPaths();
    if (mappedFolderPath.Equals(userDesktop, StringComparison.OrdinalIgnoreCase))
        return true;          // ← 桌面格子：任何变更都全量重载
    ...
}
```

**推论：**

1. 索引通道对**任何**它察觉到的变化都置 `RequiresFullReload = true`。两条通道故意重叠，所以细粒度通道几乎总会被索引通道「拖下水」——**增量刷新路径在真实使用中大概率根本不生效**。
2. 桌面是这类工具的主战场，而 `mappedFolderPath == userDesktop` 时**直接返回 `true`**，无条件全量重载。
3. 增量阈值只有 **24** 项（`WidgetViewModel.cs:17`），而 `FileSystemWatcher` 累积超过 **64** 项也会强制全量（`FolderWatcherService.cs:47,880`）。复制几十个文件就必然触发全量。

### 3.2 全量重载的真实代价：两次整目录枚举 + O(n²) 列表重建

**代价一：增量路径也要整目录枚举。**

即使走到了「增量」分支，第一步是：

```csharp
// WidgetViewModel.SortingAndWatchers.cs:327-328
FolderPathSnapshot snapshot =
    await FileService.CaptureDirectChildSnapshotAsync(changeBatch.WatchedPath);
```

而 `CaptureDirectChildSnapshotAsync`（`FileService.cs:569`）做的事情就是**重新枚举整个目录**取路径集合。所以增量与全量的差别只在「重建列表的方式」，**目录枚举是两条路径都省不掉的**。

全量路径更贵：桌面格子要枚举 **用户桌面 + 公共桌面两次**，再做一次分组去重与全量排序（`WidgetViewModel.ItemHydration.cs:78-117`）。

**代价二：列表重建是二次方复杂度。**

```csharp
// WidgetViewModel.ItemHydration.cs:246-268
for (int targetIndex = 0; targetIndex < refreshedItems.Count; targetIndex++)
{
    var refreshedItem = refreshedItems[targetIndex];
    if (!existingByPath.TryGetValue(refreshedItem.Path, out var existingItem))
    {
        Items.Insert(targetIndex, refreshedItem);
        continue;
    }
    ApplyRuntimeItemData(existingItem, refreshedItem, preserveExistingIconWhenMissing: true);
    int currentIndex = Items.IndexOf(existingItem);      // ← ObservableCollection.IndexOf 是 O(n)
    if (currentIndex < 0)
        Items.Insert(targetIndex, existingItem);
    else if (currentIndex != targetIndex)
        Items.Move(currentIndex, targetIndex);
}
```

`Items` 是 `ObservableCollection<WidgetItem>`，`IndexOf` 走引用相等性线性扫描。外层循环 n 次，每次一次 O(n) 扫描 → **O(n²)**。同一模式在 `ApplyReconciledManualOrder`（`:284-296`）里重复了一遍。1000 项 = 约 100 万次引用比较；3000 项 = 约 900 万次，全部在 UI 线程上。

**代价三：每次 `Insert` / `Move` 都是一次 UI 集合事件。**

`ObservableCollection` 的每次变更都触发 `CollectionChanged`，而文件格子的 `GridView` / `ListView` 的 `ItemContainerTransitions` 里配了 `RepositionThemeTransition`（`FileSurfaceContent.xaml:618-622,649-653`）。一次全量重载里的成百上千次 `Move` → 成百上千次重定位动画请求。作者已经关掉了 `IsStaggeringEnabled`，但动画本身还在。

**代价四：`NormalizeSortOrder` 对每一项写一次可观察属性。**

```csharp
// WidgetViewModel.SortingAndWatchers.cs:731-737
private void NormalizeSortOrder()
{
    for (int index = 0; index < Items.Count; index++)
        Items[index].SortOrder = index;    // WidgetItem.SortOrder 是 [ObservableProperty]
}
```

`WidgetItem : ObservableObject`，`SortOrder` 是 `[ObservableProperty]`（`Models/WidgetItem.cs:12,82-83`）。顺序有变动时逐项发 `PropertyChanged`。

**代价五：桌面场景下监视器数量翻倍。** 桌面格子同时挂 `_folderWatcher` 和 `_publicFolderWatcher`（`WidgetViewModel.cs:406,410`），事件量翻倍。

### 3.3 图标管线：被限流成 2 个并发槽位

这是「延迟」感最主要的来源，而且作者是**有意为之**——为了防止刷爆 Shell。

```csharp
// Helpers/BoundedBackgroundWorkScheduler.cs:28
internal static BoundedBackgroundWorkScheduler SharedShell { get; } = new(2);

// Helpers/IconHelper.cs:91-94
private static readonly SemaphoreSlim s_thumbLoadSemaphore = new(2, 2);
private static readonly SemaphoreSlim s_shellIconLoadSemaphore = new(8, 8);
private static readonly BoundedBackgroundWorkScheduler s_iconSourceScheduler =
    BoundedBackgroundWorkScheduler.SharedShell;      // ← 全局只有 2 个槽位

// Helpers/IconHelper.cs:44-47
IconSourceResolutionTimeout = 1500ms;
IconBytesLoadTimeout        = 2500ms;
```

而且这个调度器**超时也不释放槽位**：

> *A timed-out operation keeps its slot until the underlying call really returns because Windows Shell COM calls cannot be cancelled safely once they have started.*
> —— `BoundedBackgroundWorkScheduler.cs:18-23`

把这条和图标水合的批处理拼起来（`WidgetViewModel.ItemHydration.cs:394-474`，批次大小 8）：

```
一批 8 个图标请求
  └─ 每个请求先过 s_iconSourceScheduler（2 槽位）
     └─ 8 ÷ 2 = 4 轮串行，每轮上限 1500ms
        └─ 单批最坏 ≈ 6 秒
           └─ 300 项 = 38 批 → 最坏 ≈ 3.8 分钟
```

再叠加两个放大因素：

- **超时后进 30 秒黑名单**：`IconSourceTimeoutRetryMs = 30_000`（`IconHelper.cs:43`）。`ResolveIconSourceWithCacheKeyAsync` 在调用调度器**之前**先查黑名单，命中就直接返回兜底图标（`:1746-1749`）。一次超时 = 该文件 30 秒内显示通用图标。
- **滚动会反复取消并重启水合**：`GrowRenderWindow`（`WidgetViewModel.Windowing.cs:229-244`）每增长 200 项就调一次 `StartItemHydration()`，而 `StartItemHydration` 会取消上一代（`:306-313`）。但被取消的那批图标请求**仍占着那 2 个槽位**（因为不可取消）。快速滚动 = 不断往一个只进不出的队列里塞活 → 图标永远追不上。

**这一条是「感觉卡」和「其实没卡」的分界**：界面主线程可能是流畅的，但图标以肉眼可见的速度逐个出现，用户把它描述成卡顿/延迟。

### 3.4 全局低级输入钩子 × 激进工作集换出

> **重要前提（2026-09-23 修正）**：这两个钩子**都不是默认常驻的**，本节的结论只对主动开启了相关功能的用户成立。
> - `WH_MOUSE_LL` 由「双击桌面空白处显示/隐藏全部格子」驱动，而 `DesktopDoubleClickEnabled` 是裸 `bool`（`Models/CoreSettingsSlice.cs:67`），默认 `false`，`ApplyDefaultPreferences` 也显式置 false（`SettingsService.cs:645`）。
> - `WH_KEYBOARD_LL` 只在 `RegisterHotKey` 无法保留该手势时降级启用（`GlobalHotkeyService.cs:169-178` 的 `_usesReservedHook = true` 分支），常规热键走的是标准 `RegisterHotKey`/`WM_HOTKEY`，不装钩子。
>
> 因此本节的严重度低于初版判断。但因为触发条件（用户开启双击桌面切换）恰恰是该工具的高频用法，冲突仍然真实存在，且修复成本极低（见优化方案 P1）。

DeskBox 在开启相应功能后会安装两个**全局低级钩子**：

| 钩子 | 位置 | 触发频率 | 默认状态 |
|------|------|----------|----------|
| `WH_MOUSE_LL` | `DesktopDoubleClickActivationService.cs:326-330` | 系统内每一次鼠标左键按下 | **关**（需开启「双击桌面空白」） |
| `WH_KEYBOARD_LL` | `ReservedHotkeyHookService.cs:339` | 系统内每一次按键的 down/up | **关**（仅热键手势降级时启用） |

低级钩子的语义是：**整个系统的输入事件都要先经过本进程的回调**。回调必须在 `LowLevelHooksTimeout` 内返回，否则 Windows 会**静默摘掉钩子**——项目自己的注释就写了这一点（`HookHealthWatchdog.cs:8-13`），还专门做了 15 秒周期的心跳看门狗 + 合成事件探针来检测和自愈（`HookHealthWatchdog.cs:55`）。

回调本身的实现是克制的（鼠标只处理 `WM_LBUTTONDOWN` 并投递到线程池）。**但键盘回调里有两次加锁**：

```csharp
// ReservedHotkeyHookService.cs:420-457
lock (_sync)                        // ← 第 1 把锁
{
    hookHandle = _hookHandle;
    notificationWindow = _notificationWindow;
    notificationMessage = _notificationMessage;
}
...
lock (_stateSync)                   // ← 第 2 把锁
{
    mode = _mode;
    disposition = mode switch { ... };   // 状态机运算在锁内
}
```

每一次系统级按键都要过这两把锁。UI 线程只要在这两个临界区上停留（模式切换、`RecoverHook` 在 UI dispatcher 上执行），**整机键盘输入就会被拖住**。

**更麻烦的是它和内存策略的冲突。** DeskBox 有一套很激进的内存回收：

```csharp
// Platform/Win32Helper.cs:1959-1968
public static bool TrimWorkingSet()
{
    return SetProcessWorkingSetSize(GetCurrentProcess(), (IntPtr)(-1), (IntPtr)(-1));
}
```

`(-1, -1)` 的语义是**把整个工作集换出**（等同最小化窗口）。作者在注释里也承认了这个风险：

> *Historically only safe while every widget was hidden and the user was idle: touched pages fault back in afterwards, which would jitter interaction or frame pacing if anything were visible at the time.*

而 1.5.5 的新选项「Trim memory when idle」把它的触发条件放宽到了**格子仍然可见**时（`App.QuiescenceWorkingSetTrim.cs:12` 起，2 秒轮询一次），changelog 里的描述是「350 MB 压到 10 MB 以内」。

**冲突点**：进程带着两个全局低级输入钩子，却把自己的工作集换出去。换出之后，**下一次系统级点击/按键触发的钩子回调要把代码和数据页重新换回来**——这个缺页代价直接落在系统输入路径上，表现为**几百毫秒的、看起来毫无规律的整机点击迟钝**。这与用户描述的「经常会有卡顿、延迟」高度吻合：偶发、随机、不局限于 DeskBox 窗口内。

### 3.5 设置序列化在全局锁内，而设置读取走同一把锁

```csharp
// Services/SettingsService.cs:445-448
public AppSettings Settings
{
    get { lock (_lock) return _settings; }
}
```

```csharp
// Services/SettingsService.cs:1071-1128（节选）
private Task WriteSettingsTempFileAsync(string tempPath, bool stripLayoutKeys) => Task.Run(() =>
{
    ...
    lock (_lock)                                  // ← 与 Settings getter 同一把锁
    {
        PerformanceSettingsPolicy.Normalize(_settings);
        NormalizePresentationSettings(_settings);
        NormalizeAppearanceSettings(_settings);
        NormalizeFeatureWidgetSettings(_settings);
        NormalizeWidgetContentSettings(_settings);
        NormalizeWidgetTopologyLayouts(_settings);
        NormalizeOrganizerSettings(_settings);
        NormalizeHotkeySettings(_settings);
        NormalizeSearchSettings(_settings);
        NormalizeQuickCaptureSettings(_settings);
        NormalizeTodoSettings(_settings);
        NormalizeWeatherSettings(_settings);      // 12 趟归一化
        JsonSerializer.Serialize(stream, _settings, SettingsJsonContext.Default.AppSettings);
    }                                             // ← 序列化也在锁内
    stream.Flush();
});
```

作者的注释声称「UI 线程只付锁时间，不付编码时间」——但**编码本身就在锁内**，所以任何 UI 线程的 `Settings` 读取都会被整个归一化 + 序列化的耗时挡住。`_lock` 在全文件里被用了 13 次，竞争面不小。

而序列化的**规模随文件总量增长**，因为设置图里存着每个格子的完整文件清单：

```csharp
// Models/WidgetConfig.cs:131-135
/// <summary>Ordered list of items displayed in this widget.</summary>
public List<WidgetItemConfig> Items { get; set; } = [];
/// <summary>When each file was first added to or observed by this DeskBox widget.</summary>
public Dictionary<string, DateTimeOffset> FileAddedAtByPath { get; set; } = [];
```

`FileAddedAtByPath` 会随目录变化裁剪（`WidgetViewModel.AddedAt.cs:33-38`），所以它是「当前文件数」量级而非「历史累计」量级——但对几千个文件的桌面，`settings.json` 仍在数百 KB 到数 MB 区间（changelog 记录过 20 MB 的极端值）。**每次保存都要在全局锁里把这份图重新归一化并序列化一遍。**

触发频率也不低：`SaveDebounced`（1 秒去抖，`SettingsService.cs:1159`）挂在拖拽、缩放、文件操作上；而且它在去抖**之前**就同步广播变更通知（`:1163-1166`），一波操作会产生成串的通知。

### 3.6 设计内延迟：这些是「慢」而不是「卡」

这几项是产品决策，不是缺陷，但会被用户归入同一个体感：

| 参数 | 值 | 位置 | 后果 |
|------|-----|------|------|
| 悬停展开延迟（默认） | **360 ms** | `SettingsService.cs:190` | 鼠标停上去要等 1/3 秒才开始展开 |
| 悬停展开延迟（敏感） | 100 ms | `SettingsService.cs:200` | — |
| 悬停展开延迟（防误触） | 620 ms | `SettingsService.cs:202` | 超过半秒 |
| 展开/收起动画时长 | 220 ms（慢档 360 ms） | `SettingsService.cs:185-186` | 叠加在延迟之上 |
| 文件监视去抖 | 250 ms | `FolderWatcherService.cs:46` | 变更后至少等 250ms 才开始刷新 |
| 设置保存去抖 | 1000 ms | `SettingsService.cs:1159` | — |

「悬停 360ms + 动画 220ms」意味着一次展开的总响应接近 **0.6 秒**。这不是卡顿，但在主观评价里是同一件事。

### 3.7 定时器密度

全项目 `DispatcherQueue.CreateTimer()` 共 **47 处**。仅 `WidgetShell` 一个控件就创建 5 个，其中**三个是 50ms（20 fps）的持续动画定时器**，靠 UI 线程每帧写 `Opacity`：

```csharp
// Controls/WidgetShell.xaml.cs:3021  （紧凑态「呼吸」进度条）
_compactLiveBreathingTimer.Interval = TimeSpan.FromMilliseconds(50);
...
// Controls/WidgetShell.xaml.cs:3532  （音乐底部辉光）
_bottomGlowTimer.Interval = TimeSpan.FromMilliseconds(50);
...
// Controls/WidgetShell.xaml.cs:3589  （搜索呼吸边框）
_breathBorderTimer.Interval = TimeSpan.FromMilliseconds(50);

private void BreathBorderTimer_Tick(DispatcherQueueTimer sender, object args)
{
    _breathBorderPhase += 0.03;
    CompactEdgeGlow.Opacity = 0.35 + 0.15 * Math.Sin(_breathBorderPhase);  // 每 50ms 一次属性写
}
```

这些只在紧凑/胶囊模式下运行，但每个处于该模式的格子都各持一份。**每个格子每秒 20 次 UI 线程属性写入**，N 个格子就是 20N 次/秒，且每次都触发渲染失效。同样的模式在托盘层级恢复（`WidgetManager.ZOrder.cs:954`，50/200ms）和鼠标采样（`:988`，50ms）上也存在。

---

## 四、次要问题

| 问题 | 位置 | 影响 |
|------|------|------|
| 性能埋点在热路径上无条件分配字符串 | `PerformanceLogger.cs:433` 接收已构造的 `string? details`；调用方如 `IconHelper.cs:121` 写 `$"path={path}"` | 每个图标/每次目录变更都分配一个小字符串，即使埋点关闭 |
| 两个 `ItemsControl` 同时绑定同一个 `RenderedItems` | `FileSurfaceContent.xaml:593,630` | 一次集合变更通知两个控件（其中一个 `Collapsed`） |
| `RepositionThemeTransition` 挂在 `ItemContainerTransitions` | `FileSurfaceContent.xaml:618-622,649-653` | 大量 `Move` 时产生成串重定位动画 |
| `RenderedItems.Contains` / `IndexInVisibleItems` 线性扫描 | `WidgetViewModel.Windowing.cs:254,280-294` | 受渲染窗口限制，量级可控，但滚动揭示路径上仍是 O(n) |
| 内联编辑器的 100ms 重试轮询 | `Helpers/InlineEditorFocus.cs:25` | 轻微 |
| 40ms 前台对话框轮询（窗口期 10 秒） | `Helpers/ShellUiForegroundMonitor.cs:11` | 仅原生对话框打开期间 |
| `StorageItemQueryResult` 的创建本身开销不小 | `FolderWatcherService.cs:291-339`（`GetFolderFromPathAsync` + `CreateItemQueryWithOptions` + `GetItemsAsync`） | 每次切换目录都要建一次索引查询 |

---

## 五、如何验证（建议的实测方案）

静态分析的结论需要用数据确认。按性价比排序：

1. **验证 3.1 / 3.2（全量重载）** — 打开性能日志，在受管文件夹里制造一次变更，看是否出现
   `[FolderRefresh]` + `fullReload=True`，以及 `[FolderLoad] Slow load items=... totalMs=...`。
   若 1 项变更也报 `fullReload=True`，则「索引通道旁路增量路径」成立。
   代码里已经内置了这条日志：`WidgetViewModel.ItemHydration.cs:178-194`（>300ms 才输出）。

2. **验证 3.3（图标漏斗）** — 清空图标缓存后打开一个 300+ 项的文件夹，记录「首屏图标全部就位」的耗时；同时抓 `[IconHelper] Icon source resolution timed out` 与 `queue timed out` 两种日志。如果大量出现 `queue timed out`，说明瓶颈是**排队**而不是 Shell 本身慢。

3. **验证 3.4（钩子 × 换出）** — 打开「空闲时修剪内存」+ 全部格子隐藏，静置到工作集被换出，然后立刻在任意其它程序里点击，测量输入到响应的延迟。对照实验：关掉该选项后重复。这是最能区分「DeskBox 卡」还是「整机卡」的一组对照。

4. **验证 3.5（设置锁）** — 把 `settings.json` 撑到数 MB（受管文件夹放几千个文件），然后在拖拽格子窗口的同时观察是否出现规律性停顿；对照 `[FolderLoad]` 日志的时间戳与保存时机。

5. **基线对照** — 用 `scripts/` 下的内存测量脚本（仓库自带）配合 Windows Performance Recorder 抓一次 CPU 采样，看 UI 线程栈里 `SyncFolderItems` / `NormalizeSortOrder` / `JsonSerializer` 的占比。

---

## 六、如果要修：改动优先级

不改架构、按投入产出排序：

| 优先级 | 改动 | 预期收益 | 风险 |
|--------|------|----------|------|
| P0 | 给索引通道加**变更过滤**：`ContentsChanged` 不再直接置 `RequiresFullReload`，而是先与 `FileSystemWatcher` 已上报的变更做集合比对，只在确实缺失时补一次全量 | 直接消灭大部分全量重载 | 中（要保证不漏变更，需覆盖测试） |
| P0 | `SyncFolderItems` 用**路径→索引字典**替代 `Items.IndexOf`，把 O(n²) 降到 O(n)；对连续 `Move` 用批量替换或 `Reset` | 大文件夹重建时间下降一个数量级 | 低（有现成测试） |
| P0 | 取消桌面格子的无条件全量重载（`SortingAndWatchers.cs:426-430`），或降级为「全量枚举 + 增量 diff」 | 桌面场景直接受益 | 低 |
| P1 | 图标调度器把**超时与槽位解耦**：超时即归还槽位，让迟到结果自生自灭；或把并发从 2 提到 4-6 并单独限制真实 Shell 调用 | 图标出现速度显著改善 | 中（可能增加 Shell 压力） |
| P1 | 缩短或移除超时黑名单（30s → 3-5s） | 兜底图标不再长期占位 | 低 |
| P1 | 「Trim memory when idle」在**检测到低级钩子已安装**时禁用，或把钩子移入独立进程 | 消除系统级输入迟钝 | 中 |
| P2 | 设置保存改为**写快照到不可变副本**再序列化，`Settings` getter 读 `volatile` 引用而不加锁 | 消除保存期间 UI 阻塞 | 中（要重新论证一致性契约） |
| P2 | 悬停展开默认延迟 360ms → 200ms 左右 | 手感改善 | 低（产品决策） |
| P2 | 50ms 呼吸类动画改走 Composition 关键帧而非 UI 定时器写 `Opacity` | 空闲 CPU 下降 | 低 |

---

## 附：核心代码位置索引

| 主题 | 文件 |
|------|------|
| 双通道监视 + 全量重载触发 | `src/DeskBox/Services/FolderWatcherService.cs` |
| 刷新决策 / 全量判定 / 排序归一化 | `src/DeskBox/ViewModels/WidgetViewModel.SortingAndWatchers.cs` |
| 全量加载 / 列表重建 / 图标水合 | `src/DeskBox/ViewModels/WidgetViewModel.ItemHydration.cs` |
| 渲染窗口（大文件夹分片） | `src/DeskBox/ViewModels/WidgetViewModel.Windowing.cs` |
| 图标提取 / 缓存 / 并发控制 | `src/DeskBox/Helpers/IconHelper.cs` |
| Shell 工作调度器（2 槽位） | `src/DeskBox/Helpers/BoundedBackgroundWorkScheduler.cs` |
| 设置持久化与全局锁 | `src/DeskBox/Services/SettingsService.cs` |
| 全局鼠标钩子 | `src/DeskBox/Services/DesktopDoubleClickActivationService.cs` |
| 全局键盘钩子 | `src/DeskBox/Services/ReservedHotkeyHookService.cs` |
| 钩子健康看门狗 | `src/DeskBox/Services/HookHealthWatchdog.cs` |
| 工作集换出 | `src/DeskBox/Platform/Win32Helper.cs`（`TrimWorkingSet`）、`App.QuiescenceWorkingSetTrim.cs` |
| 紧凑动画 / 帧预算 | `src/DeskBox/Services/WidgetCompactAnimationCoordinator.cs` |
| 格子外壳（50ms 呼吸定时器） | `src/DeskBox/Controls/WidgetShell.xaml.cs` |
| 文件格子视图与列表控件 | `src/DeskBox/Controls/WidgetContents/FileSurfaceContent.xaml` |
