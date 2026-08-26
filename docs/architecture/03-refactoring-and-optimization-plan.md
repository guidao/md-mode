# md-mode 重构与优化方案

> 状态：可执行的增量方案
>
> 目标：保持现有兼容行为，把 view policy、renderer layout 和 source semantics 的
> 责任边界做深；不引入万能 AST，不把两个 view 复制成两套 buffer，不在没有测试的
> 情况下做大规模拆分。

## 1. 目标架构

```text
                 ┌──────────────────────────────┐
source buffer ──▶│ md-mode: session/controller   │
                 │  source commands + view state │
                 │  desired display policy       │
                 └──────────────┬───────────────┘
                                │ small public seams
                                ▼
                 ┌──────────────────────────────┐
                 │ md-render: reversible render  │
                 │  passes + source properties   │
                 │  frozen/adapter protocol      │
                 │  line layout metadata         │
                 └──────────────┬───────────────┘
                                ▼
                 ┌──────────────────────────────┐
                 │ Emacs display layer           │
                 │  word-wrap/truncate/prefixes  │
                 └──────────────────────────────┘
```

核心原则是：

- `md-mode` 决定“当前 buffer 是 source 还是 rendered、当前 view 想怎么显示”；
- `md-render` 决定“Markdown 结构如何变成带属性的可逆文本，并提供可调用的布局
  adapter”；
- Emacs display variables 是 policy 的执行结果，不成为 renderer 的隐式全局输入；
- `md-render-source`、`md-render-frozen` 和新的 line-context 是数据协议，不靠 face
  名字反推结构。

## 2. 精确的第一阶段实现

第一阶段只做最小的正确性收敛，建议按下面顺序实施。

### 2.1 统一 view policy 的计算

在 `md-mode.el` 增加一个只读的内部判定函数，名字可取
`md-mode--view-wrap-lines-p`：

```text
wrap-lines = visual-line-mode
             OR (rendered-p AND md-render-wrap-lines)
```

然后增加 `md-mode--desired-truncate-lines` 或等价的纯函数：

```text
if wrap-lines
  truncate-lines = nil
else
  truncate-lines = not md-mode-clip-wide-tables
```

这个函数不创建 overlay、不扫描 buffer、不调用 renderer。把它接入所有现在会写
`truncate-lines` 的路径：

- `md-mode--truncate-tables-in-region`（`md-mode.el:1636-1650`）；
- `md-mode--apply-table-clipping`（`md-mode.el:1665-1672`）；
- `md-mode--apply-rendered-wrapping`（`md-mode.el:1674-1695`）；
- `md-mode--set-rendered-p` 的退出分支（`md-mode.el:1801-1813`）；
- `visual-line-mode-hook`、jit-lock、window configuration、text scale 刷新。

推荐把它们收敛到一个 `md-mode--refresh-view-layout`：

1. 计算并设置 `truncate-lines`；
2. 只有 rendered view 才把 `word-wrap` 和
   `truncate-partial-width-windows` 设成 rendered policy 的值；
3. source view 不接管 `word-wrap`，让 Emacs 自己的 `visual-line-mode` 管理它；
4. rendered view 再调用 renderer 的公开 continuation-layout seam；
5. 最后刷新 table overflow overlays。

这样，edit-mode 的 `visual-line-mode` 不再被 `md-mode-clip-wide-tables` 的默认策略
覆盖；同时 `md-render-wrap-lines` 仍然只影响 rendered view，兼容现有默认值和名称。

### 2.2 明确 table overflow 的优先级

`md-mode--add-table-overflow` 当前只看 `md-mode-clip-wide-tables`
（`md-mode.el:1609-1634`）。当 `visual-line-mode` 或 rendered line wrapping 正在
接管行布局时，overflow overlay 不应把已经可以换行的内容隐藏掉。因此 overlay 的
准入条件应由同一个 policy 提供：

```text
table-overflow = md-mode-clip-wide-tables AND NOT wrap-lines
```

此处不改变 `md-render-table-wrap-columns` 的语义。表格 cell 内部换行是 renderer
自己的像素布局；普通段落/列表/heading 的逻辑行换行是 view policy 的另一个轴，二者
必须分别测试。

### 2.3 把 renderer 布局调用变成稳定接口

在 `md-render.el` 增加一个公开、窄的函数，例如：

```elisp
(cl-defun md-render-apply-continuation-layout (&key enabled)
  "Apply or remove rendered continuation layout in current buffer.")
```

它只负责 text-property 侧的工作：清理自己创建的 prefix，并在 `enabled` 为真时
按 renderer 已知的 line context 添加 `wrap-prefix`。它不读写
`truncate-lines`、`word-wrap`、`visual-line-mode` 或 mode name。

原来的 `md-render--apply-wrap-prefixes` 可在一个过渡期内保留为 private wrapper，
但 `md-mode.el` 应改为调用公开接口。这样完成后：

- `md-mode--refresh-view-layout` 是 view policy 的唯一协调者；
- `md-render-apply-continuation-layout` 是 rendered text layout 的唯一协调者；
- `md-render-convert`/`md-render-replace-markup` 仍然只做转换，不会因为一个全局 option
  偷改调用者 buffer 的 display policy。

这比把 `md-render-wrap-lines` 的 defcustom 搬进 `md-render.el` 更安全：option 的
影响对象是 `md-mode` 的 rendered buffer，而不是所有使用 renderer 的临时 buffer。
为了兼容现有用户配置，第一阶段保留变量名；是否调整 customization group 可以在
后续 API 清理中另行决定。

## 3. 第二阶段：用结构 metadata 替代展示反推

### 3.1 metadata 形状

在 renderer 内部增加一个 presentation-only text property，例如
`md-render-line-context`。只放在逻辑行的首字符，值保持小而明确：

```text
heading: (:kind heading :level 2 :face md-render-header-2)
list:    (:kind list :width 4)
normal:  nil
```

heading pass 已经在 `md-render.el:1215-1239` 计算 level 并应用 face；应在同一个
语义点保存 level，而不是之后再扫描 face。list marker 当前没有被移除，可以由一个
独立的 line-context pass 记录 prefix width；这仍然只扫描一次逻辑行，但不再依赖
face symbol。

### 3.2 layout adapter 的算法

`md-render-apply-continuation-layout` 只做以下事情：

1. 调用 `md-render--clear-wrap-prefixes` 清除自己上次生成的属性；
2. 逐逻辑行读取 `md-render-line-context`；
3. 如果行属于 frozen/source-block，或已有 renderer-owned `line-prefix`，跳过；
4. list 用 context 中的 width 生成空格；heading 用 level 生成带 heading face 的
   prefix；
5. 在行内容上写 `wrap-prefix` 和 `md-render-wrap-prefix`；
6. `enabled` 为 nil 时只保留源代码块等其它模块自己的 prefix。

不要再调用 `md-render--header-face-at`，也不要从
`(symbol-name header-face)` 截取数字。`md-render--list-prefix-regexp` 可以保留为
line-context 的输入工具，但其结果应在 context 中被固化，而不是每次 layout toggle
重新从可见文本猜一次。

### 3.3 property 生命周期

将 generic layout 的以下属性明确列为临时 layout property，并加入
`md-render--carry-properties` 的排除列表（当前函数在 `md-render.el:2831-2853`）：

```text
wrap-prefix
md-render-wrap-prefix
md-render-line-context
```

`line-prefix` 不能按属性名全局清除：已有 source block 的
`line-prefix`/`wrap-prefix` 是其自身视觉契约的一部分，不是 generic layout 创建的
属性。清理函数只能清理带 `md-render-wrap-prefix` 标记的 generic `wrap-prefix`；
源代码块的 prefix 应继续由 `md-render--style-source-blocks` 在
`md-render.el:1706-1886` 管理。若未来 generic adapter 需要 `line-prefix`，应为它
增加独立 owner marker。

## 4. 第三阶段：集中 rendered 生命周期状态

不建议引入跨整个项目的 `Document` 或 `AST` 对象。只需在 `md-mode` 内集中进入
rendered 时的 snapshot：

```text
md-mode--render-snapshot
  - word-wrap
  - truncate-partial-width-windows
  - saved-p flag
```

`truncate-lines` 不保存为用户状态，因为它是由 view policy 和 table clip 派生的；
进入/退出、option setter、visual-line hook、jit-lock 和窗口刷新都重新调用
`md-mode--refresh-view-layout`。`md-mode--set-rendered-p` 仍负责 read-only、mode-name、
font-lock managed props 和 snapshot 生命周期，但不再自己复制一份 layout policy。

这一步的关键是不主动关闭 `visual-line-mode`。用户在 source view 打开的
`visual-line-mode` 应在切换 rendered/source 后继续存在；它只是两个 view 都尊重的
layout input，而不是 rendered lifecycle 的 snapshot。

如果用户在 rendered view 中动态打开或关闭 `visual-line-mode`，离开 rendered 时不能
无条件恢复进入前的 `word-wrap` 快照：minor mode 已经改变了 source view 的用户意图。
snapshot 应同时记录进入时的 visual-line 状态；只有状态未改变时才恢复该 minor mode
所管理的值，否则保留当前 minor mode 的值。这样既能恢复 mode 在 rendered 期间临时
接管的变量，也不会把“用户刚关闭 visual-line”错误恢复成 `word-wrap=t`。

## 5. 不立即做的大重构

### 5.1 不引入万能 Markdown AST

当前系统的 source property 和 frozen range 协议已经是一个适合流式文本的轻量模型。
一次性引入完整 AST 会让增量渲染、异步媒体、原文重建和 text property 的边界全部
重新定义，接口面积会比当前问题大很多。除非后续需求需要跨多种输出格式共享完整
语义，否则不做。

### 5.2 不强行合并 edit/render table parser

先保留 `md-mode` 的源编辑 parser 和 `md-render` 的带属性/像素 parser。若重复导致
具体 bug，再提取只描述 Markdown table syntax 的小 descriptor：行边界、separator
位置、escaped pipe 和 cell source。descriptor 不负责格式化、测量、显示或编辑命令。

### 5.3 不在没有 profiling 的情况下缓存整份 line layout

当前 prefix scan 是 O(buffer size)，但它只发生在 option/visual-line 状态变化时，且
现有测试和普通文档规模不足以证明它是瓶颈。先通过唯一入口和 metadata 使算法正确；
若真实文档 profiling 证明慢，再按 changed region 或 text-property change 做增量刷新。

## 6. 精确回归测试计划

测试应放在现有两个 ERT 文件中，不另起测试框架。

### 6.1 View policy 矩阵（`test/md-mode-tests.el`）

新增一个覆盖 source/render、visual-line、render option、table clip 的组合测试，至少
检查以下结果：

| view | visual-line | render option | table clip | `truncate-lines` |
| --- | ---: | ---: | ---: | ---: |
| source | off | off | off | `t` |
| source | on | off | off/on | `nil` |
| rendered | off | off | off | `t` |
| rendered | off | on | off/on | `nil` |
| rendered | on | off/on | off/on | `nil` |

建议测试名：

- `md-mode-visual-line-mode-wraps-edit-view`；
- `md-mode-visual-line-mode-survives-source-table-refresh`；
- `md-mode-view-layout-policy-matrix`；
- `md-mode-rendered-layout-restores-source-display-state`。

每个 case 都要显式调用 `md-mode--truncate-tables-in-region`，并触发一次
`md-mode--truncate-tables-in-buffer`；这能覆盖当前真实的 jit/window/text-scale 写入
路径，而不是只检查 hook 初始结果。

### 6.2 真实屏幕契约

用 `save-window-excursion` 把测试 buffer 放入选中窗口，插入一段明显长于窗口的、带
空格的文本，然后比较 `count-screen-lines`：

- `md-render-wrap-lines=nil`：rendered paragraph 仍是一条逻辑/屏幕行的横向滚动语义
  （断言 `truncate-lines` 和 hscroll 相关状态，不依赖具体终端 glyph）；
- `md-render-wrap-lines=t`：同一文本的 `count-screen-lines` 大于一；
- list 的第二屏幕行起始位置与 marker 后的内容列一致；
- heading continuation 有正确的 heading face 和 level 对应的 prefix。

测试不要只用一个超长无空格 token；要同时覆盖普通空格、CJK、tab 和 variable-pitch
边界，避免把 `word-wrap` 的断词行为误判成 mode policy。

### 6.3 renderer contract tests（`test/md-render-tests.el`）

新增或扩展以下单元测试：

- `md-render-apply-continuation-layout-does-not-change-display-variables`：调用公开
  layout adapter 前后，`truncate-lines`、`word-wrap` 和 partial-width 不变；
- `md-render-line-context-does-not-follow-face-accidentally`：普通文本即便被额外加
  `md-render-header-1` face，也没有 heading continuation；
- `md-render-layout-skips-frozen-regions`：fenced code、inline code、media、table
  output 不被 generic prefix 包裹；
- `md-render-layout-properties-are-not-carried-across-replacement`：streaming 或
  table replacement 后没有 stale `md-render-wrap-prefix`；
- `md-render-reconstruct-ignores-layout-properties`：开启/关闭 layout 后 source
  reconstruction 字节级相等。

### 6.4 保持的现有回归

现有 `md-mode-render-round-trips-source`（`test/md-mode-tests.el:1262-1275`）、
`md-mode-toggle-markup-keeps-current-screen-row`（`1290-1312`）、所有
`md-render-reconstruct-*`、streaming、media 和 table tests 必须继续通过。新增布局
属性不能改变 modified flag、read-only 生命周期、TOC marker 或 source point mapping。

## 7. 分阶段交付顺序

1. 先加 P0 policy 和 edit-mode regression tests；不改变 option 名称和默认值。
2. 把 renderer 私有布局函数包成公开窄接口，更新 `md-mode` 调用点；保留兼容 wrapper。
3. 加 line-context，迁移 heading/list prefix 算法，补 frozen/carry tests。
4. 集中 rendered snapshot 和 layout refresh，删除重复的 `truncate-lines` 写入路径。
5. 跑全量 ERT、byte compilation/checkdoc/package-lint；只有 profiling 证明必要时才
   做增量布局缓存。

每一步都可以独立回滚：第一步修正确性，第二步只改 seam，第三步改算法输入，第四步
收敛状态，最后才考虑性能优化。这满足“小而深”的目标，也避免为了一个换行 bug
把当前已经稳定的 source reconstruction、媒体和表格系统一起重写。
