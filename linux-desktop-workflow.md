# Linux 桌面工作流：从平铺 WM 到 COSMIC DE 的架构选择

**结论**：全系统平铺是伪需求。真正的高效工作流是操作系统层走最大化/堆叠，应用层内部搞平铺，以 Nushell 承接系统胶水与意图跳转。当前两套方案并存：

- **常规方案**：COSMIC DE + Zellij + Nushell + Pop-Launcher——堆叠式 DE 开箱即用，适合作为基线参照。
- **前沿方案（当前日常）**：niri + Noctalia + walker + Ghostty + herdr + mudra + Nushell——以 workspace 无限增长和 IPC 精细控制为结构性前提的自建体系，详见 §四。

## 一、平铺 WM 的结构性缺陷

### 1. 现代 GUI 软件为全屏而生

Chrome、JetBrains 系列 IDE、VS Code、Blender 等专业软件的交互设计默认用户全屏或大窗口使用。屏幕空间就是生命线。强制将这些充满复杂菜单、侧边栏和多列网格布局的应用以 5:5 甚至更小的比例平铺，两边会触发响应式折叠，核心内容被严重挤压。专业 GUI 软件本应最大化、独占工作区。

### 2. CLI 工具的生产力价值不在于"被看见"

CLI 程序天然适合平铺——Unix 哲学下每个工具只做一件事，内部是单一内容区域，没有 GUI 软件那种多面板、多列的复杂布局划分，窗口缩放不会触发响应式错位。

但正因如此，CLI 工具的真正生产力价值不在于拉出来平铺占地方，而在于后台自动化与无感运行：

- 内存或 CPU 异常，系统通过通知或脚本自动告警
- 日志在跑，后台 `grep` 关键报错并触发自动化动作
- 编译任务后台静默运行，完成后弹轻量通知

平铺 WM 拥趸喜欢常驻 4 个终端盯着进程，但这纯粹是把关注点放错了位置——你不需要为了"被看见"而给系统找一个监工。

### 3. 截图中的 GUI 元素是伪命题：它们只是 Widget

在平铺 WM 截图中偶尔出现的 GUI 元素（精美的时钟、天气、系统监控图表）并不构成反例。它们的本质是状态栏的延伸或 Dashboard 叠加层，并不是通用的生产力应用。纵观整个平铺 WM 生态，几乎找不到将浏览器、IDE、文档编辑器等通用 GUI 在平铺布局中发挥实际生产力的说服力案例。

那些在 r/unixporn 里密密麻麻、被切得极小的终端，本质上是用图形合成器层强行拼凑一个大号"系统控制面板"。用户产生了一种"一切尽在掌握"的掌控感幻觉，但在以创造力代码为核心的输出中，这种幻觉会被通用 GUI 巨头（浏览器、IDE）击碎。

### 4. 多工作区与 Tag 机制的记忆灾难

平铺 WM 强迫用户在大脑中维护 10 个工作区的"路由表"——Workspace 1 编代码、Workspace 2 查文档、Workspace 3 通讯。高频切窗时，两三个程序间切换还需在大脑中检索它在哪个房间，认知开销极其沉重。Tag 标签分配机制（如 dwm）在快节奏的排障和编码过程中极其考验系统洁癖，几小时后沦为找窗口的困境，比传统虚拟桌面还要混乱。

## 二、外层堆叠，内层平铺

全系统强行平铺是伪需求。真正高效的工作流是"外层大窗口走最大化/堆叠逻辑，内层核心工具内部搞平铺"：

- **操作系统层（DE）**：IDE、浏览器、终端以全尺寸呈现。切换不靠物理位置（左/右/上/下），而是利用 LRU（最近最少使用）历史栈，通过高频盲操瞬间对调前台。
- **应用层内部平铺**：IDE 内部最懂代码布局；终端内部使用 Zellij 或 Tmux 进行文本平铺。纯文本对屏幕空间利用率极高，切成小块不拥挤、不错位。

### 核心场景对比

#### 场景 1：浏览器与 IDE 高频对调

**平铺 WM 方案**：两个软件分放不同工作区（Workspace 1/2），切换用 `Super + 1/2`。大脑需持续维护"浏览器在 2、IDE 在 1"的路由表。随着工作推进，顺手在 Workspace 3 开了临时网页，路由表开始混乱，按错快捷键的概率上升，心流被打断。

**COSMIC 堆叠方案**：两个软件全屏最大化，切换用 `Alt+Tab`。LRU 算法自动追踪最近使用——刚看完文档，按下快捷键系统必然带回 IDE；再按一下，必然回到文档。脑力检索开销为零，肌肉记忆纯由直觉驱动。

#### 场景 2：临时跑命令或看监控

**平铺 WM 方案**：拉起终端时全系统平铺算法启动，带来"屏幕地震效应"——原本全屏的 IDE 瞬间被挤压成 50% 畸形比例，代码文字疯狂换行，眼睛和大脑被迫花 1-2 秒适应突然改变的视觉布局。关闭终端后屏幕再次"弹回"。一天下来视觉疲劳严重。

**COSMIC 方案**：按下快捷键闪现到全屏最大化的终端窗口，内部通过 Zellij 新建面板（如 `Alt+n`）。外层 GUI 静止如水，IDE 在后台等待，没有任何形变。平铺切分隔离在纯文本流内部，视觉焦点始终平滑。

#### 场景 3：进入半个月没动过的老项目

**平铺 WM 方案**：拉起终端 → `cd ~/Workspace/...`（配合 zoxide 模糊匹配）→ `nvim .` → 手动最大化编辑器。至少 4 步操作 + 思考路径名。

**Nushell + walker 方案**：唤醒启动器 → 输入前缀 + 项目名两三个字母 → 回车。2 步纯盲操，大脑无需检索路径。cwdhist（Nushell 维护的 cwd 历史）在后台自动记录使用权重，跨启动器与终端通用——walker、Nushell 内补全、nvim、herdr 的项目跳转都消费同一份 cwd 历史数据库；底层管道完成寻址、增量计数、传参、异步拉起全屏 Neovide。

| 维度 | 平铺 WM | COSMIC + Zellij + Nushell |
|:--|:--|:--|
| 按键步数 | 较多（组合键切换/调整/最大化） | 极少（一键闪现，一键直达） |
| 认知开销 | 极高（维护工作区路由表） | 极低（LRU 历史栈 + 模糊搜索） |
| 视觉连贯性 | 较差（频繁窗口形变和闪烁） | 极佳（外层静止，平铺隔离在终端内） |

## 三、Nushell + walker 动态数据流

COSMIC DE 的配置采用 RON 格式的目录化结构（`.config/cosmic/`），具备跨设备迁移能力。与之配合的是 Nushell 的结构化数据流和 walker/elephant 的动态 provider 生态。

以"项目极速跳转"为例，看看如何打通 Shell、GUI 启动器与编辑器的壁垒：

### Nushell：结构化数据替代命令拼接

Bash 的语法陈旧、缺乏现代数据结构，充满 awk/sed/jq 样板代码。Nushell 将一切输出视为可过滤、映射的表格。

利用 `hooks.env_change.pwd` 钩子，每次在终端里 `cd` 的路径和频次自动增量更新到本地 SQLite 数据库：

```nu
$env.config.hooks.env_change.PWD = [{|_, _|
    let cwd = ($env.PWD | path expand)
    sqlx execute "INSERT INTO cwd_history (cwd, count) VALUES ($cwd, 1) ON CONFLICT(cwd) DO UPDATE SET count = count + 1"
}]
```

### walker 插件集成（elephant menus provider）

插件继承 `PopLauncherPlugin` 框架，经 elephant 的 menus provider 接入 walker，触发前缀设为特殊符号（如 `,`）。用户唤醒启动器并输入关键字时，provider 读取 Nushell 维护的 SQLite 历史数据库：

```python
# 核心查询：Frecency（频次）权重自动排序
"SELECT cwd, count FROM cwd_history WHERE cwd LIKE ? ORDER BY count DESC LIMIT ?"
```

选中项目后，插件调用 `subprocess.Popen` 以全屏无边框形式拉起 Neovide（或 Ghostty + Neovim），工作路径绑定到对应项目目录。整个过程手不离开键盘——按下启动器快捷键，敲两三个字母，回车，全屏 Neovide 窗口覆盖最前台。

**关键实现细节**：

- Neovide 必须用 `--chdir` 参数（`subprocess.Popen` 的 `cwd=` 对 Neovide 无效）
- 必须设置 `stdin=subprocess.DEVNULL`——子进程继承插件的 stdin（provider 的 RPC 流），Neovide 读取残留数据后误判为 stdin 模式，导致 session 不加载
- IPC 协议：Activate 返回字符串 `"Close"`（非 `{"Close": null}` 对象）立即关闭菜单

## 四、前沿方案：niri 自建体系（当前日常）

常规方案验证了「外层堆叠、内层平铺」的方向，但它受限于 COSMIC 的窗口层控制能力（IPC 精细控制不足）。前沿方案换用 niri 作为地基，把每一层换成可控、可编程的组件——各层职责不变，控制权全部收回：

- **WM（niri）**：workspace 无限增长（新建廉价，不设上限），单列内可上下叠放窗口；`niri msg` 提供接口/CLI 级精细控制（窗口移动 / 聚焦 / 设列宽 / workspace 路由）。这是 mudra「一实例一 workspace / 隔离维度路由」的结构性前提。
- **桌面 shell（Noctalia）**：状态栏 + 通知。图标主题、bar 模块、sysmon 挂件随 niri 会话以 systemd user 服务拉起（de-session 关联表统一维护）。
- **launcher（walker + elephant）**：动态插件生态承接意图跳转（替代常规方案中的 Pop-Launcher 角色），elephant 后端承载 menus provider。
- **终端（Ghostty）**：默认 shell 即 nushell；窗口内默认启动 herdr。
- **终端复用（herdr）**：接替 Zellij 的位置——面板/标签平铺隔离在终端内部，侧边栏多占一点空间，换来 git 信息、agent 状态常显；workspace 模式对任务透明，与 walker 的跳转集成也顺（详见「终端复用器对比」小节）。
- **浏览器（mudra）**：chromium `--app` + CDP 外部可控模式，tag 森林组织信息流——浏览器使用模式的完整设计见 [浏览器使用模式](keyboard-wm-browser-mode.md)。
- **Shell（Nushell）**：结构化数据流不变，仍承担系统胶水与 cwd 历史统计。

组件装配关系（NixOS 侧）：de-session 集中维护「DE → 组件」关联（niri 会话拉起 noctalia/swayidle），greetd 登录选择会话，swayidle 承担自动锁屏与 DPMS 熄屏（niri 原生 `power-off-monitors`），Rime（拼音 + 五笔）由 input-method 单元接管输入法。

### 与常规方案的对照

| 层 | 常规方案 | 前沿方案 | 换层原因 |
|:--|:--|:--|:--|
| WM/DE | COSMIC | niri | IPC 精细控制 + workspace 无限增长 |
| shell | COSMIC 自带 | Noctalia | 随 niri 会话拉起，模块可配置 |
| launcher | Pop-Launcher | walker + elephant | 动态 provider 生态更贴合自建插件 |
| 终端复用 | Zellij | herdr | 自研，免 prefix chord + workspace picker + cwd 跟随（详见下节） |
| 浏览器 | （未覆盖） | mudra | 外部可控 + tag 森林信息流 |
| Shell | Nushell | Nushell | 不变 |

### 终端复用器对比：herdr 与 Zellij

herdr 的能力面：

- workspace picker（open-project 插件，skim 模糊搜项目）
- cwd 跟随：新面板继承当前目录
- 免 prefix 高频 chord
- 侧边栏常显 git 信息、agent 状态——对多线程工作友好，即使不考虑 AI agent，日常使用也划算

更关键的是 workspace 设计打破了传统 session 模式：

- session 不透明，依赖人的命名和纪律——命名不好就难以区分任务，内部开 tab 也常不按命名，从外部很难找到任务归属
- workspace 相对扁平、透明，任务归属一眼可见
- session 只是实现逻辑清晰，使用时并不清晰

这与 mudra 的去 session 化（tag 森林取代 session）是同一个道理。

Zellij 有 CLI 接口（`zellij list-sessions` / `zellij attach`），「walker 列出 → 选中 → 切换」技术上可行。实现得了，但体验差一档，差别在数据模型，不在接口：herdr 的 CLI 暴露的是 workspace（项目级，含 git/agent 状态），walker 里看到的就是「任务」本身；Zellij 的 CLI 只能枚举 session 名字字符串，「这个 session 在干什么」是黑盒。切换语义也不同——`zellij attach` 是新开终端实例挂上去，herdr 由复用器自身感知焦点、复用现有面板，上下文连续；herdr 的 workspace 打开自带项目目录语义，Zellij 的 session 与目录只是弱关联。

结论不是「做不到」，而是 Zellij 的 CLI 和它的 session 模型绑在一起——接口能接上 walker，接上来的内容贫乏。herdr 的优势在于模型先对了（workspace 扁平透明），CLI 才顺手。

架构、多机协作、agent 状态感知与键盘交互的完整对比（含 herdr 0.9 的客户端渲染架构与 prefix-free chord），见 [终端复用器的代际交替](terminal-multiplexer-herdr-zellij.md)。


## 五、结论

三层职责分离（常规方案的分工，前沿方案各层对应组件见 §四）：

1. **渲染、通知、全局视觉与环境依赖** → COSMIC DE（Rust 实现，开箱即用）。通知系统原生支持回看、全局免打扰与历史常驻面板，省去了手写 Mako/Dunst 配置却无法与全局主题融合的割裂感。
2. **多窗平铺与视窗最大化** → IDE 和 Zellij 内部处理，专业软件在全尺寸保护下输出最大价值。
3. **系统胶水与意图跳转** → Nushell 结构化数据 + 动态 provider 插件。

让工具各司其职，将关注点从"配置系统"重新聚焦于"自动化与代码本身"。堆叠优先于平铺的判断贯穿两套方案——前沿方案不是回到全系统平铺，而是把「外层最大化/堆叠」的控制精度提高到能支撑隔离维度路由的程度。

---

## 交叉引用

- [Nushell 介绍](nushell-introduction.md)：Nushell 入门
- [NixOS 配置](projects/nixos-config.md)：COSMIC + Hyprland 主机配置
- [编辑器选型 2026](editor-selection-2026.md)：编辑器对比与选型
