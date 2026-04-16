# 分页算法归纳与冲突分析

本文根据 `vscode/chapter1.html` 中当前实现进行梳理。

## 1) 当前实际存在的分页算法

### A. 容量比（fill ratio）驱动算法
- 通过 `getBodyCapacity` 估算一页可承载内容高度（扣除 header/title/footer/padding/gap）。
- 通过 `getBodyContentHeight` 读取 `.slide-body` 的 `scrollHeight`。
- 用 `pageFillRatio = content / capacity` 作为核心判据。
- `PAGE_SPLIT_FILL_THRESHOLD = 0.9`，在 smart/fast/balanced 流程中，当 `pageFillRatio >= 0.9` 即可触发分裂尝试。
- `isOverflowing` 用 `pageFillRatio > 1.0005` 作为溢出判据。

### B. 打印溢出（print overflow）驱动算法
- `slideOverflowInPrint` 使用 `scrollHeight/clientHeight` 与 `scrollWidth/clientWidth` 的溢出像素进行判定。
- 先看横向溢出 `wOverflow > 2`，再看最大溢出 `maxOverflow`（<=6 视为可接受，>14 视为溢出，中间区间再回退到 `pageFillRatio > 1.01`）。
- 该算法用于 strict 模式与打印阶段（`runStrictPaginationPass` / `splitSlideStrict`）。

### C. 先压缩后拆分算法（fit-first, split-next）
- `tightenToFit` 在每页上按 fit level 逐级压缩（`compact` / `tight`）。
- 若压缩后仍溢出，再进入拆分流程 `splitSlideWithOverflowTester`。
- 拆分单位采用“从尾部抽块”策略：
  - 多子元素时，直接移除最后一个 child；
  - 单子元素但为 `UL/OL/merge-block/section-block/panel` 时，允许从该容器尾部继续抽子块。

### D. 自动续页与后续再平衡算法
- 源页分裂后，创建 `auto-generated` 续页，并重写标题/眉标/序号（“自动续N”）。
- 每个源页最多 `MAX_AUTO_CONTINUATIONS = 2`。
- 分裂后执行 `rebalanceAutoPages`，通过 `pullFromNext` 把后页开头内容拉回前页，目标是让前页尽量达到 0.58~0.8 的填充区间。

### E. 三档分页模式
- `strict`：打印几何优先。对每页先尝试 `print-fit-*` 压缩，再严格按 `slideOverflowInPrint` 分裂，循环最多 8 轮。
- `balanced`：调用 `autoLayoutDeck`，以 fill ratio + 分裂阈值 + 再平衡的折中策略。
- `fast`：与 balanced 接近，但不做全量严格循环，优先速度。

## 2) 主要冲突点（算法/阈值/流程层面）

### 冲突 1：0.9 分裂阈值 vs >1.0 溢出阈值（策略目标不一致）
- 在 smart/balanced/fast 中，`>=0.9` 就可能分裂；但 `isOverflowing` 是 `>1.0005`。
- 这意味着“未溢出页”也可能被提前拆分，偏向“留白安全”。
- strict 模式则偏向“仅在真实打印溢出时拆分”。
- 结果：同一份内容在 strict 与 balanced/fast 下，页数和断点可能显著不同。

### 冲突 2：容量估算模型 vs 真实打印盒模型
- `pageFillRatio` 依赖手工估算容量（`min-height - header/title/footer - gap - padding`）。
- strict 的 `slideOverflowInPrint` 依赖真实 `scroll/client` 几何。
- 当 CSS 改动（字体、行高、gap、媒体查询、print 样式）后，两套判据可能相互“打架”：
  - ratio 认为应拆，但 print 认为不溢出；
  - 或 ratio 认为不溢出，但 print 因横向/细小纵向溢出判为溢出。

### 冲突 3：分裂后 `pullFromNext` 回拉，可能抵消分裂意图
- 分裂逻辑把尾部内容移到下一页；再平衡又把下一页开头内容拉回前页。
- 在边界情况下会出现“先拆后并”的振荡倾向（虽有限次循环约束，仍有不稳定性风险）。
- 特别是当 `targetFill` 与 `isOverflowing` 判据接近临界值时，更容易反复触边。

### 冲突 4：续页上限（2 页）与大段不可切分内容之间的冲突
- `MAX_AUTO_CONTINUATIONS = 2` 会限制一个源页最多拆出 2 个续页。
- 若单页内容超大且尾块不可继续抽取，会直接标记 `auto-overflow-warning`。
- 这在“高密度公式 + 大面板”场景下，可能导致 warning 但仍残留溢出。

### 冲突 5：strict 的 print-fit 压缩与常规 fit level 压缩叠加
- 常规布局中已有 `compact/tight` 压缩；strict 中又叠加 `print-fit-*`。
- 两套压缩层级不是同一标度，可能造成不同模式下字号/行距收缩力度不可预测。
- 同时也会让“分页结果差异”既来自判据，也来自压缩手段，不利于调参。

### 冲突 6：largeDeckMode 直接短路分页
- `enforcePagination` 在 `largeDeckMode && !forceFull` 时直接返回（不执行分页流程）。
- 若用户未注意该模式，可能误以为分页算法失效或冲突（其实是被短路）。

## 3) 结论（当前系统的“真实状态”）

当前并非单一分页算法，而是“多判据 + 多模式 + 多阶段处理”混合系统：
1. 估算型（fill ratio）用于常规/快速布局；
2. 几何型（print overflow）用于严格打印安全；
3. 再叠加压缩、拆分、回拉再平衡与续页上限。

因此“冲突”本质不是代码互相覆盖，而是**优化目标不同（视觉均衡 vs 打印零溢出）导致的行为分叉**。只要模式切换存在，这种差异会持续存在；要减少冲突，需要统一主判据或做判据优先级收敛。
