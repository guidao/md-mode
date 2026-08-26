# md-mode 架构问题审查

> 状态：问题清单与证据
>
> 审查模型：深模块模型（Deep Module Model）。判断标准是：模块接口应尽可能小，
> 实现细节应被封装；跨模块调用只依赖稳定事实，不依赖另一个模块的内部算法。

## 总判断

当前系统已经有几个真正的深模块，尤其是 `md-render-replace-markup`、
`md-render-reconstruct`、表格像素布局和 `md-mode--heading-entries`。它们把复杂度
集中在实现内部，并用 text property、结构 descriptor 或 keyword 参数暴露小接口。

问题不在于“代码太多”，而在于少数边界没有随着功能增长一起显式化：

- view policy 的状态协调和 renderer 的布局实现被一条私有函数调用连接；
- `truncate-lines` 同时被表格策略、render wrapping 和 `visual-line-mode` 写入；
- continuation 语义从 face 名字和可见字符串反推，而不是由结构事实传递；
- rendered 生命周期的快照字段分散，进入/退出和 hook 刷新的来源不完全对称；
- 编辑解析器和渲染解析器在 heading、list、table 上存在有意但未命名的重复；
- 测试证明了属性状态，却还没有充分证明真实窗口里的显示结果。

这些问题目前没有让 207 个 ERT 测试失败，但它们会让下一个显示功能变成“再加一个
hook、再写一个 setter”，从而逐步削弱模块深度。

## 1. P0：`visual-line-mode` 在 edit-mode 会被 mode 自己覆盖

### 代码证据

Emacs 的 `visual-line-mode` 会把 `truncate-lines` 设为 `nil`，并启用
`word-wrap`。但 md-mode 的 jit 刷新函数在 `md-mode.el:1636-1650` 重新计算
`truncate-lines`：

```elisp
(let ((wanted (if (and md-mode--rendered-p
                       (or md-render-wrap-lines
                           (bound-and-true-p visual-line-mode)))
                  nil
                (not md-mode-clip-wide-tables))))
  (setq-local truncate-lines wanted))
```

`visual-line-mode` 只有在 `md-mode--rendered-p` 为真时才参与这个条件。更直接地，
`visual-line-mode-hook` 在 `md-mode.el:2438-2439` 只调用
`md-mode--apply-rendered-wrapping`，而该函数在 `md-mode.el:1674-1677` 以
`(when md-mode--rendered-p ...)` 开头。因此在 edit-mode 中它是 no-op；随后
`md-mode--apply-table-clipping`（`md-mode.el:1665-1672`）和 jit-lock 注册的
`md-mode--truncate-tables-in-region` 又会把 `truncate-lines` 按宽表策略写回去。

这解释了“视觉换行没效果”的另一种表现：即使用户主动打开了
`visual-line-mode`，表格刷新、fontification 或初始化仍可能恢复横向截断/滚动策略。
这不是 Emacs visual-line 的能力问题，而是 md-mode 的 view policy 有两个写入者。

### 影响

- rendered view 的 visual-line 测试（`test/md-mode-tests.el:942-956`）能通过，不能证明
  edit-mode 正常；
- `md-mode-clip-wide-tables` 的表格行为被当成了整个 buffer 的默认
  `truncate-lines` 行为（变量文档 `md-mode.el:65-87` 也说明它影响整个 buffer）；
- 以后增加 edit-mode 默认换行选项时，若继续沿用当前 helper，会再次和表格策略互相
  覆盖。

### 判断

这是一个真实的正确性 bug，也是当前最高优先级的架构问题。edit-mode 可以支持换行，
但语义应分成两件事：

1. 用户打开的 `visual-line-mode` 应在 source 和 rendered 两个 view 都被尊重；
2. 若未来提供默认换行选项，应单独命名为 `md-mode-wrap-lines` 或
   `md-mode-source-wrap-lines`，不能复用只描述 rendered view 的
   `md-render-wrap-lines`。

## 2. P1：布局算法和 view policy 之间缺少稳定 seam

`md-render-wrap-lines` 在 `md-mode.el:89-103` 定义，setter 还会遍历所有
`md-mode` buffer 并调用 `md-mode--apply-rendered-wrapping`。真正生成 continuation
prefix 的代码却在 `md-render.el:1888-1953`，调用点是
`md-mode.el:1695` 的私有函数调用：

```elisp
(md-render--apply-wrap-prefixes wrap-lines-p)
```

这带来三个具体问题：

1. **调用者知道实现名字是 private 的。** `md-mode` 不只是请求“应用 rendered line
   layout”，而是知道 renderer 内部的 wrap-prefix 实现名称。
2. **renderer 的独立接口和 mode 的显示接口不对称。** `md-render-convert` 和直接
   `md-render-replace-markup` 只负责转换文本/属性，不会自动应用这层 view policy；
   但 option 名字和文档容易让人以为它是 renderer 全局能力。
3. **布局刷新与渲染生命周期没有统一入口。** 现在至少有 custom setter、render
   enter、`visual-line-mode-hook`、jit-lock、window configuration 和 text scale
   这些来源，分别触发不同的 helper。

这不是说 option 必须立刻搬到 `md-render.el`。`truncate-lines`、`word-wrap` 和
`truncate-partial-width-windows` 是 Emacs buffer/view policy，最合理的 owner 仍然
是 `md-mode`。应当把“请求布局”的 renderer 接口公开，而不是让 renderer 反过来
拥有 mode 状态。

## 3. P1：heading continuation 从 face 反推语义

`md-render--header-face-at`（`md-render.el:1905-1918`）逐字符扫描 `face`，
`md-render--wrap-prefix-for-line`（`md-render.el:1932-1937`）再把
`md-render-header-2` 的 symbol 名字截取成数字。当前测试
`test/md-mode-tests.el:958-980` 正好验证了这个实现。

这个做法短小，但依赖了一个不稳定方向：

```text
语义 heading level -> header face -> face symbol name -> 数字 -> wrap-prefix
```

潜在问题包括：

- 用户或其它 font-lock 代码给普通行加了 `md-render-header-1` face，它就可能被当成
  heading；
- face composition、theme remap 或第三方 face alias 可能改变 `face` 的形状；
- renderer 需要知道“哪个 face 名字的后缀是层级”，而不是从结构 pass 直接传递层级；
- `md-render--header-faces` 只覆盖 1..6，任何未来 heading 语义扩展都要继续修改这个
  反向解析。

列表 continuation 也由可见字符串正则决定（`md-render.el:190-198、1920-1931`）。
这对当前 Markdown marker 是可用的，但 tab width、嵌套列表、带 display 属性的 marker
和非标准 marker 会让 `string-width` 不一定等于屏幕上的内容起始位置。

### 判断

这是一个“表示层泄漏”：face 是输出，不应成为下一步结构决策的唯一输入。应在
renderer 完成结构 pass 时写入轻量的 line-context text property，布局只消费这个
context。heading 至少要传 `:level` 和 `:face`；list 至少要传已计算的 continuation
width 或 prefix 字符串。

## 4. P1：buffer state 的快照和派生状态分散

rendered 生命周期字段分散在 `md-mode.el`：

- rendered 标志在 `md-mode.el:52-53`；
- `word-wrap` 与 `truncate-partial-width-windows` 的快照在
  `md-mode.el:134-144`；
- 进入/退出转换在 `md-mode--set-rendered-p`，`md-mode.el:1782-1814`；
- display property 的 font-lock 管理在 `md-mode.el:1788-1792` 和
  `md-mode.el:2381-2385`；
- `truncate-lines` 并没有保存，而是在退出时按照 visual-line 和 table clip 重新推导
  （`md-mode.el:1810-1813`）。

当前实现能恢复常见的 source 状态，但它有几个边界：

- 如果进入 rendered 前用户已经在 source view 开了 `visual-line-mode`，它不是
  `md-mode--set-rendered-p` 的快照字段，而是另一个 minor mode 的独立状态；这本身可以
  是正确决定，但必须由一个明确的 policy 函数解释，而不是隐含在多个 setter 中；
- custom setter 直接扫描 `buffer-list`，而 lifecycle 函数又直接写相同的 buffer-local
  变量，状态变更来源多于一个；
- `md-mode--apply-rendered-wrapping` 只在 rendered 时工作，导致同一名称的
  visual-line 事件在 source/render 两个状态中行为不一致。

### 判断

不需要立刻引入大型 view object；一个很小的 buffer-local snapshot（保存进入
rendered 前由 mode 接管的变量）加一个纯的 desired-policy 函数，就能让 enter/leave/
hook 共用同一个实现。

## 5. P1：布局属性的清理边界没有完全进入 renderer 协议

`md-render--clear-wrap-prefixes` 只清除带自有标记
`md-render-wrap-prefix` 的 `wrap-prefix`（`md-render.el:1888-1903`），这是好的
可逆策略。但 `md-render--carry-properties` 的排除列表
（`md-render.el:2831-2853`）包含 face、display、frozen、source 等属性，却没有
generic layout 的 `wrap-prefix`、`md-render-wrap-prefix` 和未来的 line-context。
`line-prefix` 不能被无条件归入这一类：source block 自己用它建立视觉 gutter，
它属于 source-block renderer 的正式视觉契约。

在正常 `md-mode-render` 流程中，renderer 先运行，prefix 后应用，所以暂时不容易触发；
但如果发生以下组合，旧布局属性可能被带进 delete+insert：

- 已渲染 buffer 进行 streaming re-render；
- table 或 source block 的替换调用 `md-render--carry-properties`；
- 外部 renderer 在带有旧 layout property 的区域内再次声明 frozen；
- 直接调用 renderer API，而不是走 `md-mode--set-rendered-p`。

另外，`md-render--wrap-prefix-for-line` 只检查行首的 frozen/line-prefix/wrap-prefix
（`md-render.el:1920-1927`）。行内若有 display、frozen media 或 field 边界，是否允许
给整行加通用 prefix 没有明确契约。

### 判断

布局属性应当被明确标成“临时 presentation state”：generic renderer 的 carry/replace
操作必须丢弃 `wrap-prefix`、自有标记和 line-context，重新渲染后由唯一的 layout
adapter 再生成。source block 的 `line-prefix`/`wrap-prefix` 需要保留现有视觉契约，
不能被 generic prefix 覆盖；若未来 generic adapter 也需要 `line-prefix`，必须再加
一个 owner marker，不能按属性名全局清除。

## 6. P2：有意的 parser 重复没有被命名为边界

当前至少有三类重复：

### Heading

- source/edit 侧 regexp：`md-mode.el:233-235`；
- Outline：`md-mode--outline-search`，`md-mode.el:420-447`；
- completion/Imenu 的 source/view 统一入口：`md-mode--heading-entries`，
  `md-mode.el:858-887`；
- renderer 的 heading pass：`md-render--replace-headers`，
  `md-render.el:1186-1239`；
- rendered continuation 目前再从 face 反推层级。

`7c1e409` 已经把 completion 和 Imenu 合并到一个 source-of-truth，但 renderer layout
仍未使用同一个语义事实。这是可以进一步收敛的地方。

### Table

edit-mode 使用 `md-mode--split-table-row`、`md-mode--table-bounds` 和
`md-mode--format-table`（`md-mode.el:1387-1468、1856-1910`）；renderer 使用自己
的 cell descriptor、pixel measurement 和 rendered source 协议
（`md-render.el:2118-2157、2710-2829`）。两者不应简单合并：编辑需要源码插入/删除，
渲染需要 face、frozen、像素宽度和 streamed rows。但它们共享 Markdown table 的
语法事实，未来应在测试先行的情况下提取一个小的 syntax descriptor，而不是复制
完整 formatter。

### 判断

重复本身不是失败；失败是没有区分“共享语义”与“不同执行策略”。当前不应为了追求
文件数量少而抽出一个万能 Markdown AST。应先解决 P0/P1，再根据重复导致的实际 bug
决定是否提取 descriptor。

## 7. P2：测试证明了内部状态，尚未完全证明屏幕契约

当前测试基线很好：本次运行 `test/md-render-tests.el` 和
`test/md-mode-tests.el` 共 207 个测试全部通过。与 wrapping 直接相关的测试在
`test/md-mode-tests.el:912-980`，覆盖了：

- 默认关闭时 `truncate-lines` 为真；
- option 打开时 `word-wrap`、partial-width 和 jit refresh 状态；
- rendered view 中 `visual-line-mode` 的 prefix；
- list/heading prefix 的 text property 值。

但还缺少以下 contract test：

1. 在真实窗口中，长 rendered paragraph 的 `count-screen-lines` 是否大于一；
2. edit-mode 打开 `visual-line-mode` 后，jit-lock、表格刷新、text-scale 和 window
   configuration 是否仍保持 `truncate-lines=nil`；
3. rendered/source 交替时，用户进入前的 `word-wrap` 和 partial-width 设置是否精确
   恢复；
4. source block、inline code、图片/media 和 table frozen 行不会被通用 continuation
   prefix 再包一层；
5. streaming re-render、layout disable/re-enable 后不会留下旧 `wrap-prefix`；
6. `md-render-replace-markup` 这个 renderer 接口本身不应偷偷改变 mode 的 display
   variables。

## 8. 优先级结论

| 优先级 | 问题 | 建议 |
| --- | --- | --- |
| P0 | edit-mode 的 visual-line 被表格/jit policy 覆盖 | 先统一 desired view policy；这是最小且直接的正确性修复。 |
| P1 | `md-mode` 调用 `md-render` 私有布局函数 | 增加公开的 layout seam；保留 option 在 md-mode，避免 renderer 依赖 mode state。 |
| P1 | heading/list continuation 依赖 face/可见字符串 | renderer 生成轻量 line-context，layout 只消费结构 metadata。 |
| P1 | lifecycle 快照分散，临时 layout 属性未纳入 carry 契约 | 集中 snapshot 和 layout refresh；明确 presentation property 的清理规则。 |
| P2 | heading/table parser 有意重复但边界不明显 | 先不大拆；以 descriptor 和回归测试决定是否共享。 |
| P2 | 缺屏幕级和 edit-mode 组合测试 | 先补 contract tests，再做任何优化。 |
