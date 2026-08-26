# md-mode 当前系统的演进与分层

> 状态：架构基线分析
>
> 范围：当前工作树中的 `md-mode.el`、`md-render.el`、测试和提交历史。
> 行号以 2026-08-25 的工作树为准；本文只记录分析，不定义新的运行时行为。

## 结论

md-mode 并不是从一个“大而全”的 Markdown AST 开始的。它从几个很小的
不变量逐步长出来：

1. 一个 buffer 同时承担 Markdown 源文本和渲染结果；
2. 渲染是可逆的，源文本永远可以重建；
3. 结构命令只处理它真正懂的 Markdown 结构；
4. 展示层可以通过 text property、`display` 和 `wrap-prefix` 组合出更丰富的视图；
5. 复杂的媒体、表格和流式输出都通过“冻结区域 + 原文属性”接入，而不另建一份文档。

这些不变量让系统从一个普通 major mode 演进成了一个小型的双视图文档系统。当前的
主要架构边界是清楚的，但视图布局策略后来直接接进了 `md-mode` 的状态协调代码，
导致 `md-mode` 和 `md-render` 之间出现了一个尚未充分命名的 seam：渲染器拥有布局
算法，major mode 拥有布局开关和 Emacs buffer 状态。

## 1. 从最小想法到当前系统

### 1.1 第一层：一个可编辑、可 font-lock 的 Markdown buffer

`md-mode` 先解决的是源文本编辑，而不是渲染。语法的最小词汇集中在
`md-mode.el:225-247`：表格行、fence、ATX heading、列表项、任务项和链接都有
独立的 regexp。随后 `md-mode.el:1728-1775` 把这些结构映射到 font-lock faces；
因此编辑视图仍保留原始 Markdown 字符，但用户能看到 heading、链接、表格和代码块
的结构。

这一步的接口很小：major mode 只需要把正则、font-lock、syntax-propertize 和命令
挂进 Emacs 的 mode 生命周期。`define-derived-mode` 在
`md-mode.el:2379-2443` 集中完成这些安装，也注册了后续布局、TOC 和表格刷新所需的
buffer-local hooks。

### 1.2 第二层：把结构能力抽成可组合的小函数

在源视图上，结构命令逐渐围绕几个局部抽象组合起来：

- heading 导航使用 `md-mode--outline-search` 和 `md-mode--outline-level`
  （`md-mode.el:420-447`）；
- 列表移动和缩进使用 `md-mode--list-item-info`
  （`md-mode.el:572-596`），先求出一个列表项的 begin/end/indent，再由相邻命令复用；
- 表格编辑先用 `md-mode--split-table-row`、`md-mode--table-row-cells` 和
  `md-mode--table-bounds` 得到统一的局部上下文
  （`md-mode.el:1387-1468`），然后插入、删除、移动和对齐操作都基于这个上下文。

这是一个重要的“深模块”雏形：调用者只需要知道“当前列表项/表格在哪里”，不需要
知道扫描空行、fence 或 separator 的细节。`md-mode--table-bounds` 返回
`(BEGIN END COLUMNS)`，而不是让每个表格命令重复扫描表头和分隔行。

### 1.3 第三层：同一个 buffer 的可逆 rendered view

渲染视图没有复制到另一个 buffer，而是把当前 buffer 原地改写。调用链如下：

```text
md-mode-toggle-markup                 md-mode.el:2359-2376
  ├─ md-mode-render                    md-mode.el:2303-2328
  │    ├─ md-render-replace-markup     md-render.el:430-580
  │    └─ md-mode--set-rendered-p t    md-mode.el:1782-1814
  └─ md-mode-show-source               md-mode.el:2331-2356
       ├─ md-mode--source-position-at-point
       └─ md-render-reconstruct        md-render.el:2855-2900 附近
```

`md-mode-render` 把 view change 包在 `inhibit-read-only`、`save-restriction` 和
`with-silent-modifications` 中，调用 `md-render-replace-markup :force t`，然后才把
buffer 标成 read-only（`md-mode.el:2307-2327`）。`md-mode-show-source` 反过来调用
`md-render-reconstruct`，重建原文并恢复可编辑状态（`md-mode.el:2334-2356`）。

这一步的核心抽象不是“把 Markdown 解析成 AST”，而是一个可逆的 text-property
协议：渲染出来的字符保留 `md-render-source`，代码块、表格和外部渲染区用
`md-render-frozen` 保护。`md-render-reconstruct` 只在一个完整渲染 span 被选中时
替换为保存的源文本，部分选择则保持可见文本（`md-render.el:2855-2900`）。
因此，渲染、复制、恢复源文本和点位映射可以复用同一份事实。

### 1.4 第四层：把渲染过程拆成顺序 pass

`md-render-replace-markup` 是目前最有深度的模块接口之一。它对外暴露一组 keyword
参数（`force`、`render-images`、`highlight-blocks`、`image-cache-directory`），
内部却组合了完整的渲染流程：

1. 根据 watermark 建立 streaming 范围；
2. 由 `md-render-context` 构造 fenced block 和 inline-code 范围；
3. 运行外部 renderer，并收集 frozen ranges；
4. 反复处理粗体、斜体和删除线；
5. 处理 heading、inline code、链接、图片、divider、callout、blockquote 和源码块；
6. 最后处理表格；
7. 镜像 face 到 `font-lock-face`、设置 yank handler，并更新 watermark。

具体编排在 `md-render.el:477-580`。它把“顺序”和“保护区”集中起来，下面的各个
pass 可以专注于自己的语法转换。表格放在最后，是因为表格 cell 需要消费前面已经
处理过的 face、链接和 inline-code 属性（`md-render.el:541-554`）。

### 1.5 第五层：通过 text property 接入外部 renderer 和流式内容

`md-render-render-functions` 是一个真实的 adapter seam，而不是普通的实现细节。
它的约定是：外部函数接收 `md-render-context` 的 alist，可以原地渲染某个区域，
并给结果加 `md-render-frozen`；如果要支持恢复源文本，还必须加
`md-render-source`。接口文档位于 `md-render.el:354-375`，context 的构造和执行在
`md-render.el:597-645`。

这个设计允许数学、mermaid、plantuml 和图片缓存等能力逐步接入，而不要求核心
renderer 认识每一种媒体。`md-render--frozen-ranges` 在
`md-render.el:3656-3678` 把这些 text property 重新投影成 avoid ranges，保证
后续 pass 和 streaming re-run 不会再次处理已完成的内容。

### 1.6 第六层：从字符宽度到像素宽度的表格布局

表格是系统中最明显的“专用小模块”。编辑侧的表格操作在 `md-mode.el`，渲染侧的
表格测量、分配和换行在 `md-render.el`：

- cell 解析：`md-render--parse-table-row`，`md-render.el:2118-2157`；
- 像素测量和 `variable-pitch` 防护：`md-render.el:2159-2210`；
- 自然宽度、最小宽度和列宽分配：`md-render.el:2710-2772`；
- 结果替换与 `md-render-table-source` 保存：`md-render.el:2774-2829`；
- streamed rows 的收集：`md-render.el:2997-3014`。

这部分的演进可从提交历史直接看到：`71811c9` 将表格创建、删除、对齐和移动
提升成 first-class lifecycle；`4b3baaa` 引入 variable-pitch 下的测量修正；
`892dc4b` 将宽表的自然宽度和横向滚动行为明确化。这里没有复用 `org-table`，因为
Markdown 表格的源语法、渲染属性和恢复协议由 md-mode 自己拥有。

### 1.7 第七层：视图布局和 continuation prefix

最新的 wrapping 功能是叠加在上述协议之上的一层显示策略：

- `md-render-wrap-lines` 在 `md-mode.el:89-103` 定义，默认 `nil`；
- `md-mode--apply-rendered-wrapping` 在 `md-mode.el:1674-1695` 设置
  `truncate-lines`、`word-wrap`、`truncate-partial-width-windows`，再调用 renderer 的
  私有函数；
- `md-render--apply-wrap-prefixes` 在 `md-render.el:1939-1953` 遍历逻辑行，给列表和
  heading 添加 `wrap-prefix`；
- 列表宽度来自可见列表 marker（`md-render.el:190-198、1920-1931`），heading 层级
  则从 header face 名字反推（`md-render.el:1905-1937`）。

这是一个合理的最小增量：没有改动源文本，也没有改变 `md-render-reconstruct`。但它
也标志着一个新边界：布局算法在 `md-render`，开关、Emacs buffer 变量和 hooks 在
`md-mode`，且调用关系穿过了私有函数名。

## 2. 演进历史如何体现架构意图

提交标题和实现共同显示出系统是按“不变量”逐步加深，而不是按文件堆功能：

| 节点 | 提交 | 形成的抽象或不变量 |
| --- | --- | --- |
| 结构索引 | `7c1e409` | 用 `md-mode--heading-entries` 给 completion、Imenu、source/view 两侧提供同一份 heading 事实（实现 `md-mode.el:858-887`）。 |
| 表格生命周期 | `71811c9` | 表格创建、删除、对齐、行列移动共享 `table-bounds/context`，并保持源文本可编辑。 |
| 字体和像素 | `4b3baaa` | 把显示测量隔离在 renderer 表格模块，处理 CJK、emoji、variable-pitch。 |
| 宽表策略 | `892dc4b` | `truncate-lines`、overflow overlay、auto-hscroll 形成可测试的显示策略。 |
| 可逆媒体 | 前序媒体相关提交 | 外部 renderer 用 frozen/source 属性接入，异步结果可以重建源文本。 |
| 普通长行 wrapping | `09a8e55` | 增加 `md-render-wrap-lines`、word wrapping 和 list/heading continuation prefix。 |

`7c1e409` 的提交说明明确拒绝“分别维护 completion 和 Imenu 扫描器”，而当前
`md-mode--heading-entries` 同时处理编辑视图和 rendered view。这是系统最成功的一次
抽象收敛，也为后续布局设计提供了判断标准：同一语义不应由可见 face 和另一套正则
各自推导。

## 3. 当前分层地图

| 层 | 主要职责 | 当前接口/证据 | 深度评价 |
| --- | --- | --- | --- |
| Source semantics | 识别 heading/list/table/fence，提供编辑命令和结构索引 | `md-mode--heading-entries`、`md-mode--table-bounds`、`md-mode--list-item-info` | 多数局部模块较深；heading 已有共享入口，表格仍有两套 parser。 |
| View/session coordinator | 切换 source/render、只读状态、font-lock managed props、Emacs display 变量 | `md-mode-render`、`md-mode-show-source`、`md-mode--set-rendered-p` | 生命周期清楚，但 layout policy、table clipping、visual-line hook 分散。 |
| Render core | 原地转换、streaming watermark、pass 顺序、source reconstruction | `md-render-replace-markup`、`md-render-reconstruct` | 深模块；外部接口小，内部复杂度集中且有测试。 |
| Frozen/adapter protocol | 外部媒体、代码块、表格等保护和恢复 | `md-render-render-functions`、`md-render-context`、`md-render-frozen` | seam 真实且可复用；context 是 alist，约束主要靠文档和测试。 |
| Display/layout | 表格像素布局、line wrapping、continuation prefix | `md-render--render-table`、`md-render--apply-wrap-prefixes` | 表格较深；普通 line wrapping 目前是跨文件私有调用，接口尚未命名。 |
| Verification | 结构、重建、streaming、媒体和 view toggle 的 ERT | `test/md-mode-tests.el`、`test/md-render-tests.el`，当前 207 tests | 功能覆盖广；屏幕级 wrapping 和 edit-mode visual-line 仍有缺口。 |

## 4. 这套组合为什么能承载复杂性

系统最关键的复用点不是某一个大 parser，而是三条协议：

1. **`md-render-source` 协议**：所有需要删除可见 markup 的 pass 都把原文放回同一
   个 text property，恢复、点位映射和复制都复用它；
2. **`md-render-frozen` 协议**：不能被通用 Markdown pass 再解释的内容统一变成
   avoid range，媒体、代码块、表格因此可以组合；
3. **buffer-local view state 协议**：`md-mode--rendered-p` 决定只读、mode name、
   font-lock managed props 和 TOC 行为，渲染结果仍留在原 buffer 中。

这三条协议使每个新增能力只需回答“我的源文本如何保存”“我的区域如何保护”“我在
哪一个 view 生命周期加入/退出”，而不必重新设计整个系统。问题也正由此而来：
当布局策略加入时，它复用了现有 state/hook，但没有形成同样清晰的独立接口。下一篇
将具体审查这个边界和其它剩余问题。
