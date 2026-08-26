# md-mode 最终架构文档

> 状态：三轮深模块模型审查后的实现基线
>
> 审查模型：深模块模型（Deep Module Model）。本文综合
> `01-system-evolution-and-layering.md`、`02-architecture-audit.md` 和
> `03-refactoring-and-optimization-plan.md`，并记录三轮审查如何修改方案。
>
> 结论：架构已经可以开始分阶段写代码；剩余事项是显式的实现风险和测试边界，
> 不再存在未命名的核心责任冲突。本文不表示本次已经修改了运行时代码。

## 1. 最终结论

md-mode 的最小正确架构不是“一个 Markdown AST + 两个 renderer”，而是：

```text
source text
   │ authoritative source + text properties
   ▼
md-mode session/controller ────── md-render reversible renderer
   │                                  │
   │ view policy                       │ render passes
   │ lifecycle snapshot                │ frozen/source protocol
   │ source commands/index             │ line-context metadata
   ▼                                  ▼
Emacs display state              rendered buffer text
```

它由三个深模块和一个明确的 adapter seam 组成：

1. `md-mode` session：负责一个 buffer 的 source/render 生命周期和用户可见的 view
   policy；
2. `md-render` core：负责可逆的原地 Markdown 转换、streaming watermark、frozen/source
   协议和 pass 顺序；
3. source semantics helpers：负责 heading/list/table 等编辑结构；已有的
   `md-mode--heading-entries`、`md-mode--table-bounds` 是这层的局部深模块；
4. renderer layout adapter：负责 rendered text 的 continuation metadata/prefix，
   只读写 text properties，不触碰 mode 的 `truncate-lines` 等 display variables。

## 2. 模块契约

### 2.1 `md-mode`：session/controller

稳定接口：

- `md-mode-render`（`md-mode.el:2303-2328`）：把 source 转为 rendered，并建立进入
  rendered 的 snapshot；
- `md-mode-show-source`（`md-mode.el:2331-2356`）：通过 reconstruction 恢复 source；
- `md-mode-toggle-markup`（`md-mode.el:2359-2376`）：保存当前屏幕相对位置并切换；
- `md-mode--refresh-view-layout`（建议新增）：根据当前 view、visual-line 和 option
  计算并应用 Emacs display policy；
- `md-mode--heading-entries`（`md-mode.el:858-887`）：向 completion、Imenu 和
  rendered/source 两侧提供统一 heading 事实。

调用者不应知道：

- renderer 的 pass 顺序；
- `md-render-frozen` 如何收集 avoid ranges；
- continuation prefix 如何从 line context 生成；
- table pixel width 如何分配。

`md-mode` 可以知道 `md-render-wrap-lines`，因为这是集成 mode 的 rendered view
选项；但它只通过公开 layout seam 请求 renderer 应用布局。该 option 的默认值继续
为 `nil`，保留水平滚动的兼容行为。

### 2.2 `md-render`：reversible render core

稳定接口：

- `md-render-replace-markup`（`md-render.el:430-580`）：原地转换并写入 source/frozen
  properties；
- `md-render-reconstruct`（`md-render.el:2855-2900`）：按 span 恢复原始 Markdown；
- `md-render-context`（`md-render.el:597-626`）：给静态和 streaming renderer 提供
  相同的结构 context；
- `md-render-render-functions`（`md-render.el:354-375`）：外部 adapter 的窄协议；
- `md-render-apply-continuation-layout`（建议新增）：只应用/清理 rendered text 的
  continuation layout properties。

核心不变量：

1. 渲染可以改变可见字符，但完整 rendered span 必须能由
   `md-render-source` 重建；
2. frozen region 不会被通用 pass 或 generic layout 再处理；
3. renderer 不能隐式改变调用者的 `truncate-lines`、`word-wrap` 或 mode state；
4. layout properties 是可重建的 presentation state，不是 source data；
5. streaming watermark 只优化扫描范围，不改变 source/reconstruction 的语义。

### 2.3 Source semantics

当前不拆成新文件，先保持高内聚的 `md-mode.el` 内部模块：

- heading：`md-mode--outline-search`、`md-mode--heading-entries`；
- list：`md-mode--list-item-info` 和相关移动/缩进命令；
- table：`md-mode--table-bounds`、row/cell descriptor 和 source formatter。

这些 helper 的输出应该是结构事实，而不是显示副作用。以后如果 edit/render 的
table parser 因真实 bug 需要共享，只提取 syntax descriptor，不提取“万能 table
object”。

### 2.4 Layout adapter

布局 adapter 的输入是 renderer 产生的 line context：

```text
(:kind heading :level N :face md-render-header-N)
(:kind list :width W)
```

它的输出是：

- generic continuation 的 `wrap-prefix`；
- 自己的清理标志 `md-render-wrap-prefix`；
- 必要时带 heading face 的 prefix 字符串。

它不得：

- 读取或修改 `visual-line-mode`；
- 决定 `truncate-lines`；
- 通过 face symbol 名字解析 heading level；
- 覆盖 source block 已有的 `line-prefix`/`wrap-prefix`；
- 将 `md-render-wrap-lines` 变成 renderer 全局副作用。

## 3. 最终状态和事件模型

### 3.1 状态输入

对 buffer 来说，布局只接受四个显式输入：

```text
view          = source | rendered
visual-line   = on | off
render-option = md-render-wrap-lines (rendered only)
table-clip    = md-mode-clip-wide-tables
```

desired policy：

```text
wrap-lines = visual-line
             OR (view == rendered AND md-render-wrap-lines)

truncate-lines = if wrap-lines
                   nil
                 else
                   not table-clip

table-overflow-overlay = table-clip AND not wrap-lines
```

`word-wrap` 和 `truncate-partial-width-windows` 的保存/恢复只在 rendered lifecycle
由 `md-mode` 接管；source view 的 `visual-line-mode` 继续由 Emacs minor mode 管理。
snapshot 还要记录进入 rendered 时的 visual-line 状态：如果用户在 rendered view 中
改变了该 minor mode，离开时不能盲目恢复旧的 `word-wrap` 快照，而应保留当前 minor
mode 的值；只有 minor mode 状态没有改变时才恢复 mode 接管前的值。

### 3.2 事件归一化

所有这些事件都只调用 `md-mode--refresh-view-layout`，不再各自决定 policy：

- `md-mode-render` 完成文本转换后；
- `md-mode-show-source` 完成 source 重建后；
- `md-render-wrap-lines` custom setter；
- `visual-line-mode-hook`；
- jit-lock table refresh；
- window configuration change；
- text scale change；
- `md-mode-clip-wide-tables` custom setter。

layout adapter 只在 rendered view 被调用。source view 不需要 generic continuation
prefix，但必须让 `visual-line-mode` 的 `truncate-lines=nil` 保持不被 table refresh
覆盖。

## 4. 三轮深模块模型审查

### Round 1：接口大小、模块深度和 ownership

**审查问题：** 哪些调用者事实被暴露了？哪个模块在承担不属于自己的知识？

**发现：** `md-render-replace-markup` 是深模块：接口只有少量 keyword，内部包含
watermark、pass、frozen、source property 和 face mirror。它应保留。相反，
`md-mode--apply-rendered-wrapping` 既写 Emacs display variables，又直接调用
`md-render--apply-wrap-prefixes`；调用者知道了 renderer 的 private algorithm。

**修订：** 不把 `md-render-wrap-lines` 直接搬到 renderer，因为它描述的是
`md-mode` rendered buffer 的 view policy；在 renderer 增加公开的
`md-render-apply-continuation-layout` seam。`md-mode` 只传 `enabled`，renderer 只
处理 text properties。

**Round 1 结论：** ownership 明确为“mode 决策、renderer 执行文本布局”；接口变小，
没有引入新 AST 或跨模块全局状态。

### Round 2：生命周期、可逆性和状态传播

**审查问题：** enter/leave、hook、streaming re-render 和 property carry 是否有同一
套不变量？

**发现：** 当前 `md-mode--truncate-tables-in-region` 只在 rendered 时尊重
`visual-line-mode`；edit-mode 会被 `truncate-lines` policy 覆盖。当前
`md-render--carry-properties` 也没有把 generic layout properties 明确列为临时
presentation state。进入 rendered 的 snapshot 分散在多个 buffer-local 变量和
分支中。

**修订：** 引入纯的 `view-wrap-lines`/`desired-truncate-lines` policy 和唯一的
`md-mode--refresh-view-layout`；将 generic `wrap-prefix`、layout metadata 加入 carry
排除，但不按属性名全局清除 source block 所有的 `line-prefix`；用一个小 snapshot
保存 mode 接管的 display settings 和进入时的 visual-line 状态，不主动关闭用户的
`visual-line-mode`。frozen/source block 自己的 prefix 继续由 source-block renderer
所有。

**Round 2 结论：** source/render toggle、jit refresh 和 visual-line edit-mode 的
写入者收敛为一个 policy；渲染可逆性不依赖 layout properties。

### Round 3：兼容性、测试面和优化边界

**审查问题：** 新 seam 会不会破坏现有 renderer 使用者？测试是否能验证屏幕契约？
是否过早拆出模块或缓存？

**发现：** 当前测试 207 个全通过，但 wrapping 主要检查 text property 和变量，
没有 edit-mode visual-line 或 `count-screen-lines` contract。`md-render-convert` 是
可能被独立调用的接口，不能因为 mode option 引入 display side effect。表格 renderer
和 edit parser 的重复虽明显，却承担不同职责；当前没有 profiling 证明 O(buffer) 的
prefix scan 是瓶颈。

**修订：** 公开 layout seam 只改 text properties；option 默认和名字不变；增加
source/render/visual-line/clip matrix、真实窗口屏幕行数、frozen/carry/streaming 和
renderer purity tests；暂不合并 table parser，不引入全量 AST，不做未测量的增量缓存。

**Round 3 结论：** 兼容边界、测试入口和性能边界都被写成了可验收条件；没有需要
靠猜测解决的架构问题。

## 5. 边界条件与明确处理方式

| 边界 | 处理 |
| --- | --- |
| `md-render-wrap-lines=nil` | 保留现有 rendered 水平滚动默认行为。 |
| `visual-line-mode` 在 edit-mode | 视为独立的用户 layout input，jit/table/window refresh 后仍保持 `truncate-lines=nil`。 |
| rendered 中切换 `visual-line-mode` | 记录进入时状态；若状态改变，离开 rendered 时尊重 minor mode 当前值，不盲目恢复旧 `word-wrap` 快照。 |
| 未来希望 source view 默认换行 | 新增独立 `md-mode-wrap-lines`/`md-mode-source-wrap-lines`，默认关闭；不能复用 rendered option。 |
| list marker 是 tab、数字很长或嵌套 | line-context 固化实际 continuation width；测试 tab、ordered marker 和 nested indentation。 |
| heading face 被第三方重映射 | prefix 使用 metadata 中的 level/face，不扫描 face symbol 名字。 |
| fenced code/source block | frozen 和已有 line/wrap prefix 优先，generic adapter 跳过。 |
| inline code、图片、数学/图表 | 外部 renderer 的 frozen/source 契约优先；不因为有 display property 就自动覆盖。 |
| 表格 cell 内部换行 | 由 `md-render-table-wrap-columns` 和像素测量负责；不和普通行 policy 混成一个开关。 |
| CJK、emoji、variable-pitch | 继续使用现有 table pixel measurement；普通 list prefix 以 context width 测试实际 display。 |
| narrowing/streaming | layout 只在完整 rendered view refresh 时应用；carry 丢弃临时 layout 属性，watermark 仍由 render core 管。 |
| Emacs 29.1 | 不依赖更高版本的 `visual-wrap` 包；只使用当前 mode 已使用的 `wrap-prefix`/`line-prefix` 和基础 display variables。 |

## 6. 开始写代码前的验收门

以下条件全部满足后，架构层面即可进入实现：

1. 保留现有 207 个测试，并新增 view policy matrix；
2. `md-render-wrap-lines` 默认仍为 `nil`，旧用户的 rendered horizontal-scroll 行为
   不变；
3. option 打开时 rendered 普通长行真实产生多屏幕行，list continuation 对齐 marker
   后内容，heading continuation 保留 level context；
4. edit-mode 的 `visual-line-mode` 经 jit、table、window 和 text-scale 刷新仍有效；
5. source/render toggle 精确恢复 `word-wrap`、partial-width、modified/read-only、
   TOC marker 和 source point mapping；
6. renderer 独立 API 不改变 mode display variables；
7. frozen/source block/table/media 不被 generic layout 破坏；
8. disable/re-enable、streaming replacement 和 reconstruction 后没有 stale layout
   properties；
9. 完成 byte compilation、checkdoc、package-lint 和全量 ERT；
10. 若要做性能优化，先有真实文档 profiling 和单独的基准测试。

## 7. 实现顺序

建议提交切片保持垂直且可回滚：

1. **Policy slice**：新增 edit-mode visual-line 回归测试，统一
   `truncate-lines`/table-overflow 的 desired policy。
2. **Seam slice**：增加公开 continuation-layout 接口，替换跨文件 private call，
   保留过渡 wrapper。
3. **Semantic layout slice**：加入 line-context metadata，移除 face-name 反推，
   补 frozen/carry/streaming tests。
4. **Lifecycle slice**：集中 rendered snapshot 和 refresh 入口，删除重复状态写入。
5. **Verification slice**：真实窗口测试、全量质量检查和必要的 profiling。

这套顺序先修正确性，再加深接口，再优化内部算法。它保持当前系统“小接口、大
实现、可逆 source、组合式 adapter”的优点，也为 edit-mode 换行留下了不与 rendered
option 冲突的扩展点。
