# DeskBox 前五点优化方案

> 本文是一份**独立第三方代码审查**，非 DeskBox 官方文档。基于 `main` 分支 `44e7a0d` 的静态代码分析，未运行程序、未做性能采样。

配套文档：[`root-cause-analysis.md`](root-cause-analysis.md)
日期：2026-09-23　｜　目标版本：v1.5.5（`main`）

---

## 一、结论

**五点全部可优化。** 但难度和风险分三档，且**第 3 点的改法和我上一轮的直觉相反**——不是把并发调大。

| # | 问题 | 能否优化 | 核心改法 | 风险 | 改动量 |
|---|------|----------|----------|------|--------|
| 1 | 索引通道强制全量重载 | ✅ 高性价比 | 把索引通道从「覆盖」降级为「兜底」 | 低-中 | 小 |
| 2 | 列表重建 O(n²) | ✅ 纯机械 | 补一个调用级「引用→索引」字典 + 抑制重排动画 | 低 | 小 |
| 3 | 图标只有 2 个 Shell 槽位 | ✅ 但思路要换 | **超时即归还额度**（归还 ≠ 取消） | 中 | 小 |
| 4 | 钩子 × 工作集换出 | ✅（仅对开启该功能的用户） | `TrimWorkingSet` 加钩子闸门 | 低 | 很小 |
| 5 | 设置序列化在全局锁内 | ✅ 但要先测 | 锁内取快照、锁外序列化 | 中 | 中 |

**先修正我上一轮的判断**：第 4 点里我把两个全局钩子说成了「安装了两个」，读默认值后发现不成立——

- `DesktopDoubleClickEnabled` 是裸 `bool`（`Models/CoreSettingsSlice.cs:67`），默认 `false`；`ApplyDefaultPreferences` 也显式置 false（`SettingsService.cs:645`）。**鼠标钩子默认不装。**
- 键盘钩子只在 `RegisterHotKey` 无法保留手势时降级启用（`GlobalHotkeyService.cs:169-178`）。常规热键走标准 `RegisterHotKey`/`WM_HOTKEY`，**不装钩子**。

所以第 4 点在默认配置下不存在，只对主动开启「双击桌面空白切换格子」的用户成立。严重度已从「中高」下调为「中」。但修复成本极低，仍建议做。

---

## 二、P0-1　索引通道降级为兜底

### 问题定位

`QueueChange`（原生通道）和 `QueueFullReload`（索引通道 / watcher 错误 / 根不可用）**写同一个 `_requiresFullReload` 标志**，然后在 `DebounceTimer_Tick` 里合并成一批：

```
FolderWatcherService.cs:880   _pendingChanges.Count > 64        → _requiresFullReload = true
FolderWatcherService.cs:902   QueueFullReload()（索引通道）      → _requiresFullReload = true
FolderWatcherService.cs:927   批次携带 _requiresFullReload
```

索引通道的 `ContentsChanged` 不携带变更明细，所以它只能整体作废——于是**只要索引通道响过，原生通道提供的明细就全部作废**，增量路径被旁路。

### 改法：按来源拆分标志

关键洞察：**这个去抖窗口里原生通道有没有提供明细，是可以判断的。** 有明细 → 索引通道的信号已被覆盖；没明细 → 才是真盲区。

```csharp
// FolderWatcherService.cs — 字段
private bool _requiresFullReload;      // 硬性：缓冲溢出 / 根不可用 / 原生 watcher 错误
private bool _blindReloadRequested;    // 软性：索引通道「看到有变化，但说不出是什么」

// 索引通道不再直接置硬标志
private void QueueFullReload(int? generation = null)
{
    lock (_lock)
    {
        int effectiveGeneration = generation ?? _watchGeneration;
        if (string.IsNullOrWhiteSpace(WatchedPath) || effectiveGeneration != _watchGeneration) return;
        _pendingGeneration = effectiveGeneration;
        _blindReloadRequested = true;          // ← 唯一改动
    }
    _dispatcherQueue.TryEnqueue(RestartDebounceTimer);
}

// 新增：给真正必须全量的路径用（OnWatcherError / HandleUnavailableRootFromCallback）
private void QueueMandatoryFullReload(int generation)
{
    lock (_lock)
    {
        if (string.IsNullOrWhiteSpace(WatchedPath) || generation != _watchGeneration) return;
        _pendingGeneration = generation;
        _requiresFullReload = true;
    }
    _dispatcherQueue.TryEnqueue(RestartDebounceTimer);
}

// DebounceTimer_Tick 内组装批次时
bool requiresFullReload = _requiresFullReload
    // 索引通道报了变化，但原生通道在这个窗口里一条明细都没给 → 真盲区，才全量
    || (_blindReloadRequested && _pendingChanges.Count == 0);

batch = new FolderChangeBatch(WatchedPath, _pendingChanges.ToList(), requiresFullReload, ...);
_pendingChanges.Clear();
_pendingGeneration = 0;
_requiresFullReload = false;
_blindReloadRequested = false;             // ← 别忘了这里
```

`Stop()` 里也要一并清 `_blindReloadRequested`。

### 需要配套的兜底（建议一起做）

上面这版有一个残留风险：索引通道可能在原生事件**之后**才到（顺序不保证），此时该窗口内 `_pendingChanges.Count > 0`，我们走增量、漏掉那条索引独有的变更。

建议加一道**迟到的对账**：只要该窗口出现过 `_blindReloadRequested`，就在去抖结束后 1 秒补一次「条目数比对」——用 `CaptureDirectChildSnapshotAsync` 的路径集合与 `Items` 的路径集合做一次 `SetEquals`（`FileService.cs:422-427` 已有这个写法可复用），不一致才升级为全量。

代价：一次目录枚举（本来增量路径也要付，见根因分析 3.2）。收益：不漏变更。

### 关于桌面特例

`ShouldUseFullReload` 对 `userDesktop` 无条件 `return true`（`WidgetViewModel.SortingAndWatchers.cs:426-430`），代码里**没有注释解释原因**，看起来是防御性选择（桌面是用户+公共两个目录的合并视图，且有 shell 过滤项）。

**不建议直接删掉。** 建议：
1. 先做上面的监视器改动，观察桌面场景是否仍频繁全量；
2. 若仍频繁，再单独给桌面加**可回退的增量路径**：先用 `SetEquals` 对账，一致时走增量、不一致时回落全量。这样即使增量逻辑有疏漏，也不会漏更新，只是退回原行为。

`tests/` 里目前只有一处 `ItemMutationBatchContractTests.cs:261-263` 用字符串断言这个分支的形状，**没有桌面增量刷新的行为测试**——改之前需要先补测试。

---

## 三、P0-2　列表重建去 O(n²)

### 问题定位

```csharp
// WidgetViewModel.ItemHydration.cs:246-268
for (int targetIndex = 0; targetIndex < refreshedItems.Count; targetIndex++)
{
    ...
    int currentIndex = Items.IndexOf(existingItem);   // ← ObservableCollection 线性扫描，O(n)
    ...
}
```

外层 n 次 × 每次 O(n)。**注意 `SyncFolderItems` 开头已经建好了 `existingByPath`（path→item）**，缺的只是反向的 item→index。同一模式在 `ApplyReconciledManualOrder`（`:284-296`）重复一次。

### 改法：调用级引用→索引字典，只修复被跨越的区间

不要引入永久索引——作者在 `WidgetViewModel.ItemMutationBatch.cs:22-29` 明确论证过永久 path-index 是「bug nursery」（Move/Sort/rename 的簿记负担）。这里只需要一个**本次调用内有效**的局部字典，和已有的 `existingByPath` 同级。

```csharp
private void SyncFolderItems(IReadOnlyList<WidgetItem> refreshedItems)
{
    var existingByPath = ...;                       // 保持现状
    var refreshedPaths = ...;                       // 保持现状

    if (Config.SortMode == WidgetSortMode.Manual) { ...; return; }

    // 1) 先一次性摘除（反向遍历，不维护 map）
    for (int index = Items.Count - 1; index >= 0; index--)
        if (!refreshedPaths.Contains(Items[index].Path))
            Items.RemoveAt(index);

    // 2) 摘除完成后建一次 map，O(n)
    var indexOf = new Dictionary<WidgetItem, int>(ReferenceEqualityComparer.Instance);
    for (int i = 0; i < Items.Count; i++) indexOf[Items[i]] = i;

    // 3) 主循环：查询 O(1)，Move 只修复被跨越的区间
    for (int targetIndex = 0; targetIndex < refreshedItems.Count; targetIndex++)
    {
        var refreshedItem = refreshedItems[targetIndex];
        if (!existingByPath.TryGetValue(refreshedItem.Path, out var existingItem))
        {
            Items.Insert(targetIndex, refreshedItem);
            for (int j = targetIndex; j < Items.Count; j++) indexOf[Items[j]] = j;   // 新增项不参与后续查询，可延后
            continue;
        }

        ApplyRuntimeItemData(existingItem, refreshedItem, preserveExistingIconWhenMissing: true);

        if (!indexOf.TryGetValue(existingItem, out int currentIndex))
        {
            Items.Insert(targetIndex, existingItem);
            for (int j = targetIndex; j < Items.Count; j++) indexOf[Items[j]] = j;
            continue;
        }

        if (currentIndex != targetIndex)
        {
            Items.Move(currentIndex, targetIndex);
            // 只修复被跨越的区间：from > to 时 [to, from) 整体 +1
            if (currentIndex > targetIndex)
                for (int j = targetIndex; j < currentIndex; j++) indexOf[Items[j]] = j;
            else
                for (int j = currentIndex + 1; j <= targetIndex; j++) indexOf[Items[j]] = j;
        }
    }

    NormalizeSortOrder();
}
```

**诚实说明复杂度**：修复代价 = 所有 Move 的距离之和。列表「大致有序、只增删几项」时接近 O(n)；**完全逆序时仍是 O(n²)**（n 次 Move × n 距离）。但它把每次查询的 O(n) 扫描消掉了，常数改善很大。要硬保证 O(n) 需要 LIS 最小移动集算法（React/Vue 的 diff 做法）——建议先做上面这版，实测不够再上 LIS。

`ApplyReconciledManualOrder` 同理。

### 附带收益：批量重排期间摘掉重排动画

`FileSurfaceContent.xaml:618-622,649-653` 给两个列表都配了 `RepositionThemeTransition`。批量重排期间成百上千次 `Move` → 成百上千次重定位动画请求。

```csharp
// 批量重排前
var saved = ItemsGrid.ItemContainerTransitions;
ItemsGrid.ItemContainerTransitions = null;
ItemsList.ItemContainerTransitions = null;
try { /* SyncFolderItems 主循环 */ }
finally
{
    ItemsGrid.ItemContainerTransitions = saved;
    ItemsList.ItemContainerTransitions = saved;
}
```

**不要用 `Reset` 通知来减少事件**——`ObservableCollection.Clear()` 走 Reset，会让 `GridView` 重建全部容器，通常比现在更慢。`WidgetViewModel.Windowing.cs` 的注释已经写明他们刻意「never a Reset」，这个判断是对的。

---

## 四、P0-3　图标管线：归还额度 ≠ 取消调用

### 问题定位（这一条最反直觉）

直觉是「2 个并发太小，调大」。**错。** Shell COM 调用（`SHGetFileInfo` / `IThumbnailProvider`）是跨进程的，并发调高会直接拖慢 `explorer.exe` 本身，作者的限流是有道理的。

真正的问题是**超时不归还额度**：

```csharp
// BoundedBackgroundWorkScheduler.cs:47-98
if (!await _slots.WaitAsync(timeout))                 // 排队超时：不占槽 ✅
    return new(BoundedBackgroundWorkStatus.QueueTimedOut);

workTask = Task.Run(work);
_ = workTask.ContinueWith(... ((SemaphoreSlim)state!).Release() ...);   // ← 真实返回才释放
try { return new(Completed, await workTask.WaitAsync(remaining)); }
catch (TimeoutException) { return new(ExecutionTimedOut); }             // ← 槽位仍被占住
```

两个后果：

1. **饿死**：2 个槽位被两个卡住的调用占住 10 秒，这 10 秒内**所有**其它图标请求都拿不到槽位。
2. **超时预算被排队吃掉**：`remaining = timeout - elapsed`，所以一个排队的调用可以把 1500ms 全花在等槽位上，然后拿到 0ms 执行预算——日志里的 `queue timed out` 就是这么来的。

作者的注释把两件事混为一谈了：

> *A timed-out operation keeps its slot until the underlying call really returns because Windows Shell COM calls cannot be cancelled safely once they have started.*

**「不能取消」是对的，但「不能归还额度」是推论错误。** 归还信号量不会取消任何 COM 调用——那个调用会继续跑到结束。归还只是**不再拖住后面排队的调用**。

### 改法：归还与取消解耦

```csharp
// BoundedBackgroundWorkScheduler.cs
private int _slotHeld;   // 每次 RunAsync 一个，保证幂等归还

// 释放点改为一处幂等方法
private void ReleaseSlotOnce(ref int heldFlag)
{
    if (Interlocked.Exchange(ref heldFlag, 0) == 1) _slots.Release();
}

// 执行阶段超时
catch (TimeoutException)
{
    // 归还额度，但不取消底层调用：它会跑完，结果由调用方的 generation
    // 校验丢弃（IconHelper 侧已有该机制）。继续占位只会饿死整个队列。
    ReleaseSlotOnce(ref slotHeld);
    return new(BoundedBackgroundWorkStatus.ExecutionTimedOut);
}
```

同时给执行阶段**独立预算**，不要用 `timeout - elapsed`：

```csharp
// 排队用 timeout，执行用独立预算（例如同样的 timeout），
// 否则排队把预算吃光，调用方永远拿不到执行机会
TimeSpan executionBudget = TimeoutBudget;   // 新参数，默认 = timeout
```

### 必须配的安全阀

归还后，**实际在飞的 COM 调用数**不再有上限（超时的调用还在跑）。所以需要：

- 一个独立的在飞计数（`Interlocked`），软上限比如 32；
- 超过软上限时，新请求退回 `QueueTimedOut`（此时是真的该排队的）。

这样得到**两级准入**：逻辑队列不阻塞（超时即走），真实并发有硬顶（保护 explorer.exe）。这才是原设计想达到的效果。

### 顺带两项

1. **30 秒黑名单太长**：`IconSourceTimeoutRetryMs = 30_000`（`IconHelper.cs:43`），超时一次该文件 30 秒内直接返回通用图标（`IconHelper.cs:1746-1749`）。建议降到 3-5 秒——「慢一点但正确」比「30 秒错误图标」观感好得多。
2. **滚动时的水合抖动**：`GrowRenderWindow`（`WidgetViewModel.Windowing.cs:229-244`）每增长 200 项就 `StartItemHydration()`，会取消上一代；但被取消的请求仍占着槽位（不可取消）。上面解耦之后这个抖动会明显缓解。另外可以给增长加一个短去抖（例如 120ms），避免连续滚动时反复重启水合。

---

## 五、P1-4　`TrimWorkingSet` 加钩子闸门

### 改法：唯一收口点加判断

4 个调用点全部走 `Win32Helper.TrimWorkingSet()`（`App.ImmediateHiddenWorkingSetTrim.cs:87`、`App.QuiescenceWorkingSetTrim.cs:126`、`App.xaml.cs:3314`、`App.xaml.cs:4307`），所以只改这一个方法就能覆盖全部路径。

```csharp
// Platform/Win32Helper.cs
private static int s_lowLevelHookCount;

internal static void NoteLowLevelHookInstalled() =>
    Interlocked.Increment(ref s_lowLevelHookCount);

internal static void NoteLowLevelHookUninstalled() =>
    Interlocked.Decrement(ref s_lowLevelHookCount);

public static bool TrimWorkingSet()
{
    if (Volatile.Read(ref s_lowLevelHookCount) > 0)
    {
        // 进程带着全局低级钩子时不得换出工作集：换出后下一次系统级
        // 输入触发的钩子回调要缺页换入，代价直接落在整机输入延迟上。
        return false;
    }

    try
    {
        return SetProcessWorkingSetSize(GetCurrentProcess(), (IntPtr)(-1), (IntPtr)(-1));
    }
    catch { return false; }
}
```

**调用侧**：在 `SetWindowsHookEx` 成功后调 `NoteLowLevelHookInstalled()`，在 `UnhookWindowsHookEx` 之后调 `NoteLowLevelHookUninstalled()`（`DesktopDoubleClickActivationService.HookThreadMain` 的 `finally`、`ReservedHotkeyHookService` 同样位置）。注意钩子线程可能被 Windows 静默摘掉——所以 `HookHealthWatchdog` 检测到钩子已死、走 `RecoverHook()` 的路径上也要保证计数不被漏减。

更稳的做法是**让计数跟随「钩子句柄是否非零」**，而不是跟随安装/卸载动作——在 `TrimWorkingSet` 里改成回调一个 `Func<bool>` 询问「当前是否有活跃钩子」，由两个服务各自实现。这样不存在漏减。

### 不建议做

把钩子移入独立进程是理论上的正解（宿主进程换出不影响钩子进程），但那是架构级改动，收益/风险比不划算。

---

## 六、P2-5　设置序列化离开全局锁

### 先测，再改

**动手前必须先测量。** 依据是 1.5.4 的 changelog：`settings.json` 从约 20 MB 被收缩到 100 KB 以下。如果实际是 100 KB 量级，序列化耗时只有几毫秒，那么**锁持有时间不是主要矛盾，锁竞争才是**。这种情况下第 5 点的收益远小于前四点。

测量方法：在 `WriteSettingsTempFileAsync` 的 `lock (_lock)` 前后各打一个时间戳，记录锁持有耗时；同时统计 `settings.json` 的实际大小。如果锁持有 < 5ms，建议跳过本项，只做下面的「读侧无锁」。

### 如果确实需要改

```csharp
// SettingsService.cs
private AppSettings _settings;                   // 写侧：仍由 _lock 保护
private volatile AppSettings _settingsSnapshot;  // 读侧：无锁

public AppSettings Settings => _settingsSnapshot;
```

保存路径的关键改动是**把序列化的输入在锁内变成不可变快照，锁外再做编码**：

```csharp
private Task WriteSettingsTempFileAsync(string tempPath, bool stripLayoutKeys) => Task.Run(() =>
{
    AppSettings snapshot;
    lock (_lock)
    {
        // 12 趟归一化仍就地作用于 _settings（保持既有语义不变）
        PerformanceSettingsPolicy.Normalize(_settings);
        ... 其余 11 趟 ...
        snapshot = CloneForSerialization(_settings);   // 锁内克隆：便宜且必须
    }

    using var stream = new FileStream(tempPath, FileMode.Create, FileAccess.Write, FileShare.None, 64 * 1024);
    JsonSerializer.Serialize(stream, snapshot, SettingsJsonContext.Default.AppSettings);   // 锁外编码
    stream.Flush();
});
```

为什么这样是对的：「一个保存 = 一个连贯快照」的契约**保留**了——克隆发生在锁内，拿到的仍然是一个自洽的瞬间；只是把最贵的 UTF-8 编码搬出了临界区。前提是克隆比编码便宜（通常成立，少了编码和格式化）。

**需要重新论证的点**：现在 12 趟 `Normalize` 是**就地修改 `_settings`** 的。改成快照化后要确认没有任何代码依赖「归一化的副作用会反映到 `_settings` 上」——比如外部读 `Settings` 时期待已归一化的值。这是本项的主要风险，也是为什么它排 P2。

**彻底解法**是 copy-on-write（所有写路径都替换 `_settings` 引用而不是就地改），但那是一次真正的重构，不建议在没有实测数据支撑时启动。

---

## 七、不建议做的四件事

| 别做 | 原因 |
|------|------|
| 把 `SharedShell` 并发从 2 调到 8+ | Shell COM 是跨进程的，会拖慢 `explorer.exe` 本身。要解决的是**归还**，不是**并发**。 |
| 引入永久 `path → item` 索引 | 作者已在 `ItemMutationBatch.cs:22-29` 论证：Move/Sort/rename 的簿记会让它变成 bug 温床。用调用级局部字典即可。 |
| 用 `Reset` 通知减少集合事件 | `GridView` 会重建全部容器，通常更慢。作者「never a Reset」的判断是对的。 |
| 删掉索引监视通道 | 它是真兜底（原生 watcher 缓冲溢出、索引能看到的变更）。要改的是它的**权重**，不是存在性。 |

---

## 八、建议的落地顺序与验证

| 步骤 | 改动 | 验证方式 |
|------|------|----------|
| 1 | P0-3 图标额度解耦 + 黑名单缩短 | 清图标缓存开 300+ 项文件夹，记录「首屏图标就位」耗时；`queue timed out` 日志应大幅减少 |
| 2 | P0-2 索引字典 + 摘动画 | `[FolderLoad] Slow load` 日志的 `syncMs` 应显著下降 |
| 3 | P0-1 监视器来源拆分 | 制造单文件变更，`[FolderRefresh]` 应出现 `fullReload=False`；再补迟到的 `SetEquals` 对账确认不漏 |
| 4 | P1-4 钩子闸门 | 开启「双击桌面空白」，静置到工作集被换出，测任意程序点击延迟；关闭该功能后重复对照 |
| 5 | P2-5 设置快照 | **先测锁持有耗时**，< 5ms 则跳过 |

前三步都是小改动、可独立上线、可独立回滚，建议先做。

**注意**：该项目作者目前**暂不接受外部 Pull Request 的合并**，但明确说明**这不是永久政策**——

> *This is not a permanent policy. Once the core architecture and plugin system become more stable, I plan to revisit external contributions...*
> —— `CONTRIBUTING.md:13`

所以这份方案更适合：① 走 Issues/Discussions 提交（作者欢迎 bug 报告与架构讨论，且明确表示想先稳定核心架构）；② 自己 fork 后自用。考虑到第 1、3 点都涉及架构层面的取舍（监视器权重、并发准入模型），先发 Discussion 讨论设计意图、确认我对其取舍的理解是否正确，比直接给 patch 更有效。
