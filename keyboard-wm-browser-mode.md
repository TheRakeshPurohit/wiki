# 平铺 WM + launcher 全键盘下的浏览器使用模式

> **状态**: 方案设计 / 实现设计已定（2026-08-28）
> **触发**: qutebrowser 的 modal 输入法痛点 → 探索一种"非模态、最小 UI、全键盘、外部可控"的浏览器使用方式
> **上游参考**: qutebrowser/qutebrowser#3444（OPEN）

---

## 一、定位与动机

要的是**一种「浏览器使用模式」，不是某一款浏览器**：在 Niri（或任意平铺 WM）下，复刻 qutebrowser "最大化 webview、最小化 UI" 的观感，同时满足非模态、全键盘、外部可控，并与 launcher（walker）工作流自洽。

触发痛点有二：

- **qutebrowser 的 modal 输入法无解**：qutebrowser 有插入/normal 模式切换，但没有模式切换 hook（上游 #3444，autocmd/插件系统未实现），输入法在 normal 模式、文本框聚焦时吃键。无法干净地"只在插入模式开启输入法"。
- **扩展层控制力度不足**：Vimium / SurfingKeys 这类扩展进不了错误页（4xx/5xx 显示的 `chrome-error` 页）和内置页，控制不完整。

## 二、选型结论

- qutebrowser modal：输入法无解（上游 #3444），排除。
- 纯扩展层：错误页/内置页注入不了，不能完全控制，排除。
- Nyxt / EVE / uzbl：全可编程但系于 WebKit / Emacs / Lisp，门槛与兼容不划算。
- **chromium `--app` + CDP + SurfingKeys + sqlite/qw**：折中，各层各司其职。容器不复刻 qutebrowser"嵌入引擎画标签条"，而是**完整 Chromium + 外部控制器**（保留 SurfingKeys 扩展与错误页可控）。

## 三、架构（各层职责）

- **`--app` 每页一窗口**：无 omnibox、无标签条 → 最大化 webview。Niri 默认无标题栏 → 打开就是一张干净网页。"标签"不在浏览器里，由外部控制器（qw）按 session 管理。**CDP 操纵不了 Chromium 的 UI chrome**（标签条/omnibox 是浏览器自身 UI，非 CDP 能力范围）——最小 UI 只能靠 `--app`，不能用 CDP 去缩/藏 chrome。
- **CDP 脊梁**：控制浏览器里所有 target（含错误页、内置页）+ `Target.targetCreated/Destroyed` 事件实时同步到 sqlite（满足"关闭时更新列表"）+ 会话恢复。
- **SurfingKeys**：普通页面的键盘 / 插入 / 输入法切换（JS 驱动，能自理 IME）；创建 session 时**提前种进 profile**。
- **sqlite + launcher**：session 管理（多 session × 多页面）、多进程并发、窗口↔进程映射、url 记录与过滤、网站列宽记忆；walker 列 session 唤起 + 地址输入。

## 四、资源占用（多窗口 vs 多标签）

Chromium 是 **process-per-page**（每页面一个 renderer，无论一窗多标签还是一个标签一窗）。因此：

- **同一实例内**开多个 `--app` 窗口 ≈ 一个窗口开同样多的标签：1 个 browser 进程 + 每页 1 个 renderer + 共享 cache/GPU。
- **多个独立实例**（不同 profile）= 各自 browser 进程 + 各自 cache，明显更重。
- 结论：本方案 **B 的并发版**——可**同时开多个进程，一个进程一个 workspace**（不强求"一次只激活一个"）。每个受管进程 = 独立 browser 进程 + 独立 cache；并发越多资源占用越高，以换取隔离与工作区分组。

## 五、地址输入、历史与命令（热键）

- "地址栏"由 **launcher（walker）URL/历史提示 → CDP navigate** 承担：热键唤起 prompt，输入或选历史即跳转。
- 前进 / 后退 / 刷新 = CDP history 命令（`Page.getNavigationHistory` / `navigateToHistoryEntry` / reload）绑键。
- 比内置 omnibox 更贴合"全键盘 + launcher"工作流。

## 六、窗口管理与热键

窗口管理目标：浏览器窗口自成一组，既方便单独唤起，又不淹没通用窗口切换器。

- **独立 walker 菜单（类 cwdhist / windowsmru）**：专列浏览器窗口，查 `niri msg -j windows` 按过滤标准列出，独立热键触发。通用 windowsmru 菜单按同一标准排除浏览器窗口 → 不被淹没。**注意**：niri **无原生**「从切换器排除某类窗口」的开关——默认配置里根本没有 Alt+Tab 绑定（其导航是 workspace 式 Mod+HJKL + Mod+O overview）。浏览器窗口不被淹没 靠的是**一进程一 workspace 的结构性隔离** + 专用 walker 菜单，而非过滤原生 Alt+Tab。
- **Alt+Tab 过滤（实现）**：把 Alt+Tab 重绑为一个 walker 窗口菜单，列表**排除浏览器前缀** → 即「不含浏览器的 Alt+Tab」；niri 原生切换无法过滤，只能由 launcher 层顶替。
- **过滤标准**：原生 Wayland 窗口元数据只有 `app_id` 与 `title`（无任意 tag）→ "元数据过滤"的现实选项 = title 前缀 / app_id / workspace / **url**。
  - **title 前缀**：wrapper 打开 `--app` 窗口时加统一前缀（如 `◆` 或 session 名），wrapper 自身可控，最稳。
  - **workspace**：进程的窗口路由到专用 workspace（如 `web:<name>`），菜单按 workspace 过滤，兼具分组意义。
  - **url**：页面 url 实时记录，可按地址过滤（见 §七）。
- **热键分配**（暂缓，先纯命令行）：`Mod+Space` → 浏览器窗口菜单；cwdhist 暂不动。

## 七、实现设计（`qw` CLI，方案已定 2026-08-28）

选型：**B（每 session 一实例/工作区）并发版 + Python + 命令名 `qw`**。

### 组件
- **`qw`**（CLI，Python）：命令入口，读 sqlite + 把控制命令发给 daemon。
- **`qwd`**（daemon，Python 常驻）：持各运行实例的 CDP WebSocket，订阅 Target 事件实时写 sqlite、转发控制命令、拉起/停掉实例。
- **每 session 一个 Chromium 实例**：默认不跑，`qw open` 才拉起；**可并发多个**。
- **sqlite**：`~/.local/share/qw/qw.sqlite`。

### 数据模型
```sql
instances(id, profile TEXT, port INT, pid INT, running INT,
         proxy TEXT, extensions TEXT)                       -- 1 session ↔ 1 instance
sessions(id, name UNIQUE, workspace TEXT, instance_id FK,
         created_at, last_opened_at)
pages(id, session_id FK CASCADE, target_id, url, title,
      position INT, opened_at INT, closed_at INT NULL)
site_widths(site TEXT PRIMARY KEY, proportion REAL)           -- 网址→列宽比例(0~1)
```
- `pages` 关联 `instances`（窗口属于哪个进程），`url` 由 CDP `infoChanged` **实时更新** → 可地址过滤。

### session = 工作区（并发）
- 每个 session/进程绑定一个 niri **workspace**（如 `web:<name>`），**可同时开多个进程，一个进程一个 workspace**。
- `qw open <name>` = 切到该 workspace + 拉起该 session 的实例、重建其 pages。
- 实例窗口进对应 workspace：title 前缀（app_id）经 niri 窗口规则路由 + `qw open` 时 `niri msg action focus-workspace <name>` 双保险。
- 切/混 session = 关掉或移动某进程窗口，另起目标；**不同进程窗口可混合摆在同一 workspace，也可移动 workspace 分开**。

### 窗口↔进程映射 与 批量移动
- 记录每个窗口属于哪个进程：CDP target 归属（经该实例调试端口）+ title/app_id 对齐到 niri 窗口。
- **一条命令把某进程的全部窗口移到别的 workspace**：`qw move <name> <workspace>` → 遍历该进程的窗口 → `niri msg action move-window-to-workspace <ws>`（已核实存在；或按列 `move-column-to-workspace <ws>`）。
- **可混合**：不同进程窗口可摆同一 workspace，再移动来分开/分组——**workspace 移动即"分开"手段**。

### 链接地址记录与过滤
- CDP `infoChanged` 实时更新 `pages.url`。
- 可按地址过滤窗口：`qw ls` / walker 按 url 子串过滤。

### 代理 与 扩展插件列表
- **代理**：每实例/进程可设 `proxy`，拉起时 `chromium --proxy-server=<url>`（可加 `--proxy-bypass-list`）。
- **插件列表（可配置）**：实例允许设置预装的扩展清单（如 SurfingKeys、Bitwarden）。拉起时 `chromium --load-extension=<dir1>,<dir2>,...` 加载 unpacked 扩展，清单随 session 配置插入。**已实测（2026-08-28）**：`--app` + `--load-extension` 能把 SurfingKeys（构建产物 `dist/production/chrome/`）载入 `--app` 窗口（扩展 id `fbnpkpganphpmhekgfkanhdpombfanpj`，其 service_worker 与注入 iframe 在 CDP 可见）。unpacked 扩展需先构建（SurfingKeys 源 `npm install` 走官方 registry 直连、webpack `build:prod` 出 dist）。
- 两个均为标准 Chromium 命令行开关（NixOS 包装脚本透传）。

### 网站列宽记忆（Win+R）
- **niri 列宽 = 输出宽度比例**：`preset-column-widths { default-column-width { proportion 0.5; } }`；`switch-preset-column-width`（Mod+R）循环档位 1/3 1/2 2/3 1（= 0.33/0.5/0.67/1.0）。
- **无直接 getter**：niri `msg` 取不到"当前列宽档位"，窗口 JSON 不干净给出平铺列比例 → 用比率方案。
- **捕获**：调整列宽时按 Win+R → `qw` 计算 `proportion = window_width / output_width`（窗口几何宽 ÷ 桌面宽）+ 当前聚焦 url 域名 → snap 到最近档位写 `site_widths`。
- **恢复**：打开该域名 → 聚焦其列 → `niri msg action set-column-width <比例>`（Length 支持 px/% ；分数形式如 `1/2` 实现时确认）。
- 比率方案本就契合 niri 模型（宽度即比例）。

### CLI 命令（v1）
```
qw new <name>              创建 session（配 workspace）
qw open <name> [url...]    启动该 session（切 workspace + 拉实例 + 重建页面）
qw add <name> <url>        当前/指定 session 加一页
qw move <name> <workspace> 把某进程全部窗口移到另一 workspace（分开/分组）
qw ls [name]               列 session / 列页面（可 url 过滤）
qw close <name>            关该实例（窗口 + workspace 收起）
qw rm <name>               删 session（+其 profile/workspace）
qw goto <url> | back | forward | reload
qw focus <page>
qw quit                    关全部实例
qw daemon start|stop|status
```

### CDP 映射
- 拉起：`chromium --app=<url> --remote-debugging-port=<动态端口> --user-data-dir=<该 session profile> --no-first-run`（profile 内预置 SurfingKeys）
- 开页 `Target.createTarget` / 关页 `Page.close` → 事件同步
- 导航 `Page.navigate` / `reload` / `getNavigationHistory` / `navigateToHistoryEntry` / 聚焦 `Target.activateTarget`
- 按地址精确控标签：`Target.getTargets` 列所有 target（url/title/type）→ 按地址/标题搜 → 命中 → `Target.activateTarget` 激活；错误页亦可。

### 事件同步（核心）
`qwd` 收到 `targetCreated/infoChanged` → upsert page（url/title 实时）；`targetDestroyed` → 置 `closed_at`。实时，无需轮询。

### 实现备注
- Python 依赖仅一个非标准库：**`websocket-client`**（CDP 走 WebSocket，标准库没有；其余 sqlite3 stdlib）。
- CDP 即 JSON 消息，直接手写很轻，不需 jsonrpc 封装。

## 八、已验证 / 待验证

- ✅ `--app` 模式无 omnibox / 标签条，Niri 下纯页面（截图实测）。
- ✅ CDP 能列出并 attach `--app` 窗口（它就是普通 page target）。
- ✅ niri `move-window-to-workspace` / `move-column-to-workspace` / `set-column-width` 存在。
- ⏳ SurfingKeys 在 `--app` 窗口的注入未实测（本环境未安装扩展）。
- ⏳ CDP 对错误页（`chrome-error`）reload / navigate 未实测（按机制应当可控）。
- ⏳ 同实例多窗口 ≈ 多标签的资源等价逻辑成立，未实测；本方案走 B 并发版，多实例资源为预期代价。

## 九、开放问题

- **双控制路由**：SurfingKeys（普通页）与 CDP（错误页/内置页）之间，按键谁管、如何按 target 类型判定切换。
- **引擎 UI 缺口**：下载 / 打印等原本由浏览器 chrome 提供的功能，需另行接（CDP / 省略）。
- **列宽读取**：niri 列宽无直接 getter，用 `window_width/output_width` 比率；恢复时确认 `set-column-width` 的 Length 分数语法。