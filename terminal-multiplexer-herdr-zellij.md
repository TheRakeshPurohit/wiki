# 终端复用器的代际交替：Herdr 与 Zellij

Zellij 是传统终端复用器的现代化巅峰，Herdr 则是为 AI 编码代理时代重新设计的终端 workspace 管理器。二者的分野不是功能多寡，而是架构前提不同：Zellij 延续 tmux 的「服务端包办一切」模型，Herdr 把渲染与状态感知拆回本地客户端。本文从架构、会话感知、多机协作与键盘交互四个维度对比，session 模型与 walker 集成的完整论证见 [Linux 桌面工作流](linux-desktop-workflow.md)。

## 架构：服务端渲染 vs 客户端渲染

| | Zellij | Herdr |
|:--|:--|:--|
| 渲染位置 | 服务端渲染，通过网络传输 ANSI/VT 控制流 | 客户端渲染 UI，远程场景下服务端经 SSH 传输终端内容与会话状态，本地客户端以本地主题绘制 |
| 多端挂载同一会话 | 经典 tmux 行为：视口向下兼容最小客户端，大屏出现大片空白（「画中画」） | 选中哪台机器/客户端，终端输入与尺寸就跟随哪边（「最后交互者接管」） |
| 部署模型 | 服务器上各跑各的实例，互相孤立 | 本地与多台 SSH 机器的 workspace 并排出现在同一侧边栏与主视窗（v0.9.0+ Connecting machines），单机断连不影响其他 |

多端自适应是客户端渲染的直接收益：服务端不必为一个「最小公倍数」尺寸渲染，尺寸意图由本地发出、远程状态机重置。这也是为什么 Herdr 的手机场景（SSH 客户端直连）能自适应窄屏，而不是缩小版桌面。

> 注：社区流传的「Herdr 用 `libghostty-vt` 维护状态机」说法在官方文档中无出处，其状态机实现细节未公开核实，只确认「服务端维护状态、客户端渲染」这一分工。

## 会话感知：黑盒命名 vs 结构化大地图

Zellij 的任务管理本质是黑盒：session 靠人命名，tab 与 pane 无顶层感知。多 session 时命名失效，被迫收敛成单 session，然后单 session 内几十个 tab/pane 靠记忆检索——找某个耗时任务或日志要逐个翻。

Herdr 的侧边栏是一张常驻的结构化地图：workspace → tab → pane 树状呈现，pane 内前台进程名直接可见。更关键的是 agent 状态感知——Herdr 为每个 agent pane 维护 `idle` / `working` / `blocked` 三态（lifecycle hooks 优先，screen manifest 兜底），状态向上汇总到 tab 和 workspace。人不需要记住 pane 叫什么，点红色（blocked）直接降落到需要回答的 agent。

| | Zellij | Herdr |
|:--|:--|:--|
| 可枚举单元 | session 名字字符串 | workspace（含 git 信息、agent 状态） |
| 任务定位 | 靠命名纪律 + 逐个翻找 | 视觉锚点直接降落 |
| agent 忙闲 | 无感知 | 三态自动标记 + 汇总 |

## 键盘交互：Zellij 领先，Herdr 可补齐

这是 Zellij 真正反击的一节。它的 Vim 式模态设计（`Ctrl+p` 进入 pane 模式后单键 `n`/`x`/`hjkl`）比「前缀 + 第二单键」的双击流更顺手，高频操作免前缀通达。

传统前缀模型在中文输入法下的痛点是真实的：按下前缀后第二个单键若落输入法激活态，会被吞进输入框变候选拼音，必须 Esc 切回英文重来，分屏高频操作时思路反复中断。但「Zellij 免疫输入法拦截」的说法过强——输入法拦截发生在终端/OS 层，对两种复用器一视同仁；Zellij 的优势主要是模态单键流本身少按一次前缀，且组合键形态（`Alt+箭头`）更少撞上悬空单键。

Herdr 对此的一等公民解法是 prefix-free direct chord：官方推荐 `ctrl+alt` 修饰键族（终端、DE、macOS option 组合都几乎不占用），在 `config.toml [keys]` 中给同一动作双绑前缀与直达 chord：

```toml
[keys]
focus_pane_left  = ["prefix+h", "ctrl+alt+h"]
new_tab          = ["prefix+c", "ctrl+alt+c"]
zoom             = ["prefix+z", "ctrl+alt+z"]
```

本机配置已按此落地（13 个高频动作双绑 ctrl+alt 直达 chord，见 linux-desktop-workflow.md），高频操作实际无需前缀，输入法卡壳问题随之消解。

## 选型

- 多机协作、常驻 AI agent、需要忙闲感知 → Herdr 是代际性差异，不是增量改进。
- 纯人类单机键盘流、不跑 agent → Zellij 的模态交互与开箱体验仍然成立，无需迁移。

## 参考

- Herdr 官方文档：keyboard / connecting-machines / agents（herdr.dev，`herdr --help` 内置索引）
- session 模型对比与 walker 集成：[Linux 桌面工作流](linux-desktop-workflow.md)
