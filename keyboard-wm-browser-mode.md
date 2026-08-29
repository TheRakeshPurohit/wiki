# 平铺 WM + launcher 全键盘下的浏览器使用模式

> **状态**: 方案设计 / 实现设计已定（2026-08-28）；**架构转向 tag 森林**（2026-08，方向已定、落地中）
> **触发**: qutebrowser 的 modal 输入法痛点 → 一种"非模态、最小 UI、全键盘、外部可控"的浏览器使用方式
> **定位**: 从全键盘 UX 工具升级为**信息流精炼**——浏览器是信息的主要流入途径
> **上游参考**: qutebrowser/qutebrowser#3444（OPEN）
> **通用内容组织模型**: 见 [标签森林](tag-forest.md)

---

## 一、定位与动机

**信息流精炼**：对多数场景而言，浏览器是信息的主要流入途径。本模式不止是"全键盘操作浏览器"的 UX 改进，更是把浏览器入口变成一条**信息流管道**：**捕获 → 属性化 → 精炼 → 消费**。

- **捕获**：每个流入的页面被外部可控地接收、归档（mudra 维护每页记录）。
- **属性化**：给每个页面打 **tag**（归属）与**评分**（importance / urgency）。
- **精炼**：按重要/紧急筛选、排序、沉淀，决定看什么、先看什么、归档什么。
- **消费**：跳转、打开、导出（后续可扩展：LLM 总结、转 Markdown）。

浏览器从"一个看网页的工具"变成"一个替你先分拣信息的入口"。这正建立在**外部可控**之上——只有完全可编程地列出、控制、标记每一页，才可能在它之上做分类与精炼。

要的是**一种「浏览器使用模式」，不是某一款浏览器**：在 Niri 下，复刻 qutebrowser"最大化 webview、最小化 UI"的观感，同时满足非模态、全键盘、外部可控，并与 launcher（walker）工作流自洽。

**硬约束 · 几乎 niri 独享**：整套模式依赖两个**结构性**条件——① WM 提供**接口/CLI 级精细控制**（窗口移动 / 聚焦 / 设列宽 / workspace 路由等 IPC 动作）；② 该 WM 的 **workspace 模型**适合「一实例一 workspace / 隔离维度路由」。

- **IPC 层面**：niri、hyprland 满足；cosmic-de 目前似乎还不满足（窗口层控制能力不足）。
- **但仅 IPC 不够，workspace 模型才是分水岭**：
  - **平铺类（如 hyprland）不合适**：workspace 有**上限**，单个 workspace 内能排布的窗口数也有限——实际限制了「隔离维度任意增长 + 每维度多窗口平铺」的形态。
  - **堆叠类（如 cosmic-de）**：即便 IPC 强到足以支撑，理论上可行，但**体验不佳**（无平铺布局优势）。
  - 结论：这套机制**几乎是 niri 独享**——niri 新建 workspace 廉价、可增长；且**单列内可上下叠放窗口**，实现灵活布局，常见场景如**开发者工具叠加摆放**（编辑器 + 终端 + 调试器同列上下）。

触发痛点有二：

- **qutebrowser 的 modal 输入法无解**：qutebrowser 有插入/normal 模式切换，但没有模式切换 hook（上游 #3444，autocmd/插件系统未实现），输入法在 normal 模式、文本框聚焦时吃键。无法干净地"只在插入模式开启输入法"。
- **扩展层控制力度不足**：Vimium / SurfingKeys 这类扩展进不了错误页（4xx/5xx 显示的 `chrome-error` 页）和内置页，控制不完整。

## 二、选型结论

- qutebrowser modal：输入法无解（上游 #3444），排除。
- 纯扩展层：错误页/内置页注入不了，不能完全控制，排除。
- Nyxt / EVE / uzbl：全可编程但系于 WebKit / Emacs / Lisp，门槛与兼容不划算。
- **chromium `--app` + CDP + SurfingKeys + sqlite/mudra**：折中，各层各司其职。容器不复刻 qutebrowser"嵌入引擎画标签条"，而是**完整 Chromium + 外部控制器**（保留 SurfingKeys 扩展与错误页可控）。组织与精炼由 mudra 的 **tag 森林**承担。

## 三、架构（各层职责）

- **`--app` 每页一窗口**：无 omnibox、无标签条 → 最大化 webview。Niri 默认无标题栏 → 打开就是一张干净网页。"标签"（tab）不在浏览器里，由外部控制器（mudra）按 **tag 森林**管理。**CDP 操纵不了 Chromium 的 UI chrome**（标签条/omnibox 是浏览器自身 UI，非 CDP 能力范围）——最小 UI 只能靠 `--app`，不能用 CDP 去缩/藏 chrome。
- **CDP 脊梁**：控制浏览器里所有 target（含错误页、内置页）+ `Target.targetCreated/Destroyed` 事件实时同步到 sqlite（满足"关闭时更新列表"）+ 恢复。
- **SurfingKeys**：普通页面的键盘 / 插入 / 输入法切换（JS 驱动，能自理 IME）；创建实例时**提前种进 profile**。
- **sqlite + launcher**：**tag 森林**组织与精炼（多 tag 维度 × 页面）、isolated 实例隔离、窗口↔进程映射、url 记录与过滤、网站列宽记忆；walker 列 tag 过滤唤起 + 地址输入。

  **引擎控制面（核实 2026-08）**：chromium 是唯一具备成熟 **CDP** 的引擎（`/json` 端点 + WebSocket，`Target.*`/`Page.*`/`Runtime.*` 完备控制面——目标/窗口/导航/注入，mudra 全依赖它）。替代引擎 Ladybird、Servo **都有**远程控制，但都走 **Firefox 远程调试协议（RDP）**——actor + TCP JSON 包，且都标着不完整（Servo `protocol.rs` 自注 "currently only supports JSON packets"；Ladybird `LibDevTools` 含 `FirefoxClient.*`/`Actor.*`，即 RDP actor 架构）——**不是 CDP**，也非成熟的窗口/会话管理表面，能力面远不够当 daily-driver 受控引擎。→ 引擎抽象方向：mudra 控制层抽象成 `BrowserEngine` 接口（chromium=CDP backend），等哪个引擎协议长成熟再补，而非现在改包。

## 四、资源占用（多窗口 vs 多实例）

Chromium 是 **process-per-page**（每页面一个 renderer，无论一窗多标签还是一个标签一窗）。因此：

- **同一实例内**开多个 `--app` 窗口 ≈ 一个窗口开同样多的标签：1 个 browser 进程 + 每页 1 个 renderer + 共享 cache/GPU。
- **多个独立实例**（不同 profile）= 各自 browser 进程 + 各自 cache，明显更重。

结论：**按 isolated 实例隔离**——每个 `isolated` 标签的维度一个独立实例（profile），并发实例数 = **隔离维度数**，而非页面/主题数。代价是多跑几个 browser 进程 + cache，换取 **cookie / 登录态 / 隐私**的硬隔离。非隔离标签不新增实例，在共享实例内多页。

## 五、地址输入、历史与命令（热键）

- "地址栏"由 **launcher（walker）URL/历史提示 → CDP navigate** 承担：热键唤起 prompt，输入或选历史即跳转。
- 前进 / 后退 / 刷新 = CDP history 命令（`Page.getNavigationHistory` / `navigateToHistoryEntry` / reload）绑键。
- 比内置 omnibox 更贴合"全键盘 + launcher"工作流。

## 六、内容组织：tag 森林（取代 session）

页面不再按单归属的 session 组织，而是落进**标签森林**（多棵树、树间不互斥、树内单选、无单一根；退化情形覆盖普通多标签与传统分类两端，完整模型见 [标签森林](tag-forest.md)）。

- **situation 树（必选、单选、默认 inbox）**：`inbox / work / personal / privacy`。决定**当前上下文**与隔离。新信息默认落 `inbox`，处理后移到对应维度值——**inbox 分流**。
- **importance / urgency 树（两级、rank 排序）**：叶节点 `☆..☆☆☆☆☆`。评分即树，不是独立字段——`importance` 表示内容质量/价值，`urgency` 表示时效。域名/子域名规则表给叶 tag 打默认分（如 infoq 高重要、其用户贡献子域名低重要），手动覆盖。
- **topic 树（普通、多选）**：`项目:x / 新闻 / 技术` 等可多选归属。

**隔离实例**：`isolated=true` 的 tag 命中 → 该内容运行在独立实例（独立 profile/cookie/进程）。目前 situation 四个子节点都隔离，互为互斥不会冲突；多命中取第一个是未定义行为，不特殊处理。

**pin 常驻（PWA）**：IM / RSS 等固定在某实例长期常驻、独立窗口，对应渐进式 Web 应用（**PWA**）——像原生 app 一样常驻的"应用槽"。

**术语**：**tab → page**（标签条里的"标签"改名 page，避免与 tag 混淆）。

**页面树（`parent_id`）**：一个页面里打开的另一个页面是**子页**（CDP `openerId`）——页与打开它的页形成**树**。既有标签森林（内容组织轴）之外，这是页面的**打开关系**轴：用于**链接挖掘**（从"高价值父页引出的子页"反向发现）与**分拣**。

## 七、窗口管理与热键

窗口管理目标：浏览器窗口自成一组，既方便单独唤起，又不淹没通用窗口切换器。

- **独立 walker 菜单（类 cwdhist / windowsmru）**：专列浏览器窗口，查 `niri msg -j windows` 按过滤标准列出，独立热键触发。通用 windowsmru 菜单按同一标准排除浏览器窗口 → 不被淹没。**注意**：niri **无原生**「从切换器排除某类窗口」的开关——默认配置里根本没有 Alt+Tab 绑定（其导航是 workspace 式 Mod+HJKL + Mod+O overview）。浏览器窗口不被淹没 靠的是**一隔离维度一 workspace 的结构性隔离** + 专用 walker 菜单，而非过滤原生 Alt+Tab。
- **Alt+Tab 过滤（实现）**：把 Alt+Tab 重绑为一个 walker 窗口菜单，列表**排除浏览器前缀** → 即「不含浏览器的 Alt+Tab」；niri 原生切换无法过滤，只能由 launcher 层顶替。
- **过滤标准**：原生 Wayland 窗口元数据只有 `app_id` 与 `title`（无任意 tag）→ "元数据过滤"的现实选项 = title 前缀 / app_id / workspace / **url**。
  - **title 前缀**：wrapper 打开 `--app` 窗口时加统一前缀（如 `◆` 或实例名），wrapper 自身可控，最稳。
  - **workspace**：隔离实例的窗口路由到专用 workspace（如 `web:<name>`），菜单按 workspace 过滤，兼具分组意义。
  - **url**：页面 url 实时记录，可按地址过滤。
- **walker 模式与切换**：把"当前上下文"（situation）作为切换主键——在 `inbox / work / personal / privacy` 间切换（默认 inbox），列表按 situation 过滤对应实例的 open pages；`#` 进入**操作模式**（对当前聚焦 page 提供关闭、收藏、复制链接等）。**page** 搜索 = 当前上下文实例的打开窗口，`Enter` 切过去（`activateTarget` + niri focus window）。全局 page（IM 常驻）在任意上下文都可访问（pin 实例），不随 situation 改变。
- **关闭语义**：mudra 主动关闭 → 从归档**删除**该页（不留痕）；通过 niri 关窗 / 意外（崩溃）关闭 → **保留**（仅标 closed，不删）——主动关不留痕，外部/意外关不丢。

**分拣：页面树 → workspace**：把某个页面连同它的**整棵子树**移到独立 workspace，用于归类（WM 层操作）。

**launcher 交互（walker `p`/`t`/`a`/`s`，键位定稿）**——实际页面管理操作都在 launcher 层做成列选动作：
- `p` = **Page 模式**：页面列表 → 聚焦 / 关闭 / 移动到当前窗口 / 交换；
- `t` = **tag 模式**：默认 situation 树（当前上下文）；**跨树可多选 / 树内单选**（situation / importance / urgency 单选、topic 可多选）；多选实现 = **累积缓冲（A，`keyboardShortcut`）** / **文本分隔（B 兜底）**；
- `a` = **Action 模式**：当前聚焦页动作（关闭 / 复制链接 / 打标 / 评分 / 隔离）；
- `s` = 排序（MRU / 时间 / 星序 → 写 `state.sort`）。
动作回调 `mudra CLI`；列表按 tag / situation 过滤。

## 八、实现设计（`mudra` CLI，方向 tag 森林）

选型：**B（每 isolated 实例一实例/工作区）并发版 + Python + 命令名 `mudra`**。

### 交互分层（设计主线）
交互分三层，职责分离、各自可脚本化/接入：
1. **接口 / CLI（核心操作）**：`mudra` 命令——数据与页面操作的事实源（open / ls / focus / tag / star / col），无 UI 假设。
2. **Launcher（实际的页面管理操作）**：walker 菜单——`p`/`t`/`a`/`s` 列选动作，选中回调 `mudra CLI`。
3. **WM（展示相关）**：niri——workspace 布局、列宽、窗口映射；**页面树 → workspace** 移动（分拣）。

### 组件
- **`mudra`**（CLI，Python）：命令入口，读 sqlite + 把控制命令发给 daemon。
- **`mudrad`**（daemon，Python 常驻）：持各运行实例的 CDP WebSocket，订阅 Target 事件实时写 sqlite、转发控制命令、拉起/停掉实例。
- **每 isolated 实例一个 Chromium 实例**：默认不跑，`mudra open` 才拉起；**可并发多个**（隔离维度各一个）。
- **sqlite**：`~/.local/share/mudra/mudra.sqlite`。

### 数据模型
```sql
tag(id, parent_id, name, alias, isolated, required, rank, hidden, note)  -- 邻接表; 递归 CTE 查树
page_tag(page_id, tag_id)                                      -- 树间多行=多选; 树内单选 app 约束
instances(id, profile TEXT, port INT, pid INT, running INT,    -- 隔离实例(1 isolated tag ↔ 1 instance)
         proxy TEXT, extensions TEXT)
pages(id, instance_id FK, target_id, url, title, position INT,
      opened_at INT, closed_at INT NULL,
      parent_id INT REFERENCES pages(id))                     -- 页面树: 子页 = 由它打开的页(openerId)
site_widths(site TEXT PRIMARY KEY, proportion REAL)           -- 网址→列宽比例(0~1)
state(key TEXT PRIMARY KEY, value TEXT)                       -- current_context(situation) / sort / ...
**DB 迁移策略（原型阶段）**：schema 变更直接删除 `mudra.sqlite` 重建，不做 ALTER 迁移——原型数据无持久价值。
```
- `pages` 关联 `instances`（窗口属于哪个实例），`url` 由 CDP `infoChanged` **实时更新** → 可地址过滤。
- `current_context` 取代 `current_session`（situation，默认 inbox）。

### 隔离实例 = workspace（并发）
- 每个 isolated tag 绑定一个 niri **workspace**（如 `web:<tag>`），**可同时开多个实例，一个实例一个 workspace**。
- `mudra open` = 切到该 workspace + 拉起该实例、重建其 pages。
- 实例窗口进对应 workspace：title 前缀（app_id）经 niri 窗口规则路由 + `mudra open` 时 `niri msg action focus-workspace <name>` 双保险。
- 切/混上下文 = 关掉或移动某实例窗口，另起目标；**不同实例窗口可混合摆在同一 workspace，也可移动 workspace 分开**。

### 窗口↔进程映射 与 批量移动
- 记录每个窗口属于哪个实例：CDP target 归属（经该实例调试端口）+ title/app_id 对齐到 niri 窗口。
- **一条命令把某实例的全部窗口移到别的 workspace**：`mudra move <instance|tag> <workspace>` → 遍历该实例的窗口 → `niri msg action move-window-to-workspace <ws>`。
- **可混合**：不同实例窗口可摆同一 workspace，再移动来分开/分组——**workspace 移动即"分开"手段**。

### 链接地址记录与过滤
- CDP `infoChanged` 实时更新 `pages.url`。
- 可按地址过滤窗口：`mudra ls` / walker 按 url 子串过滤。

### 代理 与 扩展插件列表
- **代理**：每独立实例可设 `proxy`，拉起时 `chromium --proxy-server=<url>`（可加 `--proxy-bypass-list`）。
- **插件列表（可配置）**：实例允许设置预装的扩展清单（如 SurfingKeys、Bitwarden）。拉起时 `chromium --load-extension=<dir1>,<dir2>,...` 加载 unpacked 扩展，清单随实例配置插入。**已实测（2026-08-28）**：`--app` + `--load-extension` 能把 SurfingKeys（构建产物 `dist/production/chrome/`）载入 `--app` 窗口（扩展 id `fbnpkpganphpmhekgfkanhdpombfanpj`，其 service_worker 与注入 iframe 在 CDP 可见）。unpacked 扩展需先构建（SurfingKeys 源 `npm install` 走官方 registry 直连、webpack `build:prod` 出 dist）。
- 两个均为标准 Chromium 命令行开关（NixOS 包装脚本透传）。

### 网站列宽记忆（Win+R）
- **niri 列宽 = 输出宽度比例**：`preset-column-widths { default-column-width { proportion 0.5; } }`；`switch-preset-column-width`（Mod+R）循环档位 1/3 1/2 2/3 1（= 0.33/0.5/0.67/1.0）。
- **读取（已核实 2026-08）**：niri `msg -j windows` 顶层无宽度字段，但聚焦窗嵌套 `layout.tile_size[0]` 即列宽；除以 `focused-output.logical.width` 得比例。
- **设置（已核实）**：`set-column-width <N%>`(百分比) 或 `-N`(像素减)；分数形式如 `1/2` **报错**。恢复时把比例 snap 到档位后按整数百分比发射。
- **捕获**：调整列宽时按 Win+R → `mudra` 读聚焦窗列宽比例 + 当前聚焦 url 域名 → snap 到最近档位写 `site_widths`。
- **恢复**：打开该域名 → 聚焦其列 → 按记录比例整数百分比 `set-column-width`。
- 比率方案本就契合 niri 模型（宽度即比例）。

### CLI 命令（v1，tag 森林视角）
```
mudra use <context>           切当前上下文（situation：inbox/work/personal/privacy，默认 inbox）
mudra open <url> [--tag t]    打开页面（默认落 inbox；可指定/补 tag）
mudra tag <page> <tags>       给页面打标签（situation / importance / urgency / topic）
mudra star <page> <☆☆☆>      设 importance/urgency 等级（重要性/紧急性树）
mudra ls [tag]                按 tag/上下文过滤列出页面
mudra focus <page | query>   聚焦（CDP activate + niri focus window）
mudra move <instance|tag> <ws> 把某实例全部窗口移到 workspace
mudra isolate <tag>           建 isolated 实例（含 proxy/extensions）
mudra pin <url>               固定到 pin 实例（PWA 常驻）
mudra goto <url> | back | forward | reload
mudra close <page>            关该实例/页面（主动关删除，外部关保留）
mudra yank                    复制当前聚焦窗口地址到剪贴板
mudra col remember|show        列宽记忆
mudra quit / mudra daemon start|stop|status
```

### CDP 映射
- 拉起：`chromium --app=<url> --remote-debugging-port=<动态端口> --user-data-dir=<该实例 profile> --no-first-run`（profile 内预置 SurfingKeys）
- 开页 `Target.createTarget` / 关页 `Page.close` → 事件同步
- 导航 `Page.navigate` / `reload` / `getNavigationHistory` / `navigateToHistoryEntry` / 聚焦 `Target.activateTarget`
- **新窗口拦截（级联）**：`--app` 里 `_blank`/`window.open` 默认开成 chrome 窗口；mudrad 维护注入脚本（拦 `window.open` 与 `a[target=_blank]` → Image beacon → 本地 `/open` → 新 `--app`）。**必须在每个新 page target 出现时注入**（`Target.targetCreated` → `Page.addScriptToEvaluateOnNewDocument`），否则 mudrad 自己开出的新 `--app` 窗口没带脚本，里面再点新窗口又退回 chrome 默认（非 `--app`）。
- 按地址精确控页：`Target.getTargets` 列所有 target（url/title/type）→ 按地址/标题搜 → 命中 → `Target.activateTarget` 激活；错误页亦可。

### 事件同步（核心）
`mudrad` 收到 `targetCreated/infoChanged` → upsert page（url/title 实时）；`targetDestroyed` → 置 `closed_at`。实时，无需轮询。

### 实现备注
- Python 依赖仅一个非标准库：**`websocket-client`**（CDP 走 WebSocket，标准库没有；其余 sqlite3 stdlib）。
- CDP 即 JSON 消息，直接手写很轻，不需 jsonrpc 封装。
- 剪贴板（`mudra yank`）走 Wayland `wl-copy`：用 CDP 取聚焦窗口 URL → 管道到 `wl-copy`。

## 九、已验证 / 待验证

- ✅ `--app` 模式无 omnibox / 标签条，Niri 下纯页面（截图实测）。
- ✅ CDP 能列出并 attach `--app` 窗口（它就是普通 page target）。
- ✅ niri `move-window-to-workspace` / `move-column-to-workspace` / `set-column-width` 存在；列宽读取（`layout.tile_size[0]`/`logical.width`）与设置（百分比）已核实。
- ✅ walker **支持多字符前缀**（`src/data.rs` `starts_with`）+ `argument_delimiter`；mudra 走 elephant `menus` provider（widget 见 mudra `docs/EXTENSIONS.md`）。
- ⏳ SurfingKeys 在 `--app` 窗口的注入未实测（本环境未安装扩展）。
- ⏳ CDP 对错误页（`chrome-error`）reload / navigate 未实测（按机制应当可控）。
- ⏳ 同实例多窗口 ≈ 多标签的资源等价逻辑成立，未实测；走 B 并发版，多实例资源为预期代价。
- ⏳ tag 森林落地：situation 切换 / isolated 实例 / importance·urgency 树 / ML 打标，未实现（重构进行中）。

## 十、开放问题

- **双控制路由**：SurfingKeys（普通页）与 CDP（错误页/内置页）之间，按键谁管、如何按 target 类型判定切换。
- **引擎 UI 缺口**：下载 / 打印等原本由浏览器 chrome 提供的功能，需另行接（CDP / 省略）。
- **tag 森林落地**：树内单选的 app 约束、isolated 多命中（现为未定义行为）、重要性/紧急性的规则迭代与 ML 辅助；inbox 分流的 UX 细节。
- **ML 打标**：朴素贝叶斯 / 逻辑回归从手动评分学"该选哪个值"，生成规则供审（非 LLM 黑箱）。