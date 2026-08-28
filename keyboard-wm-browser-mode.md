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
- **chromium `--app` + CDP + SurfingKeys + sqlite/qw**：折中，各层各司其职。

## 三、架构（各层职责）

- **`--app` 每页一窗口**：无 omnibox、无标签条 → 最大化 webview。Niri 默认无标题栏 → 打开就是一张干净网页。"标签"不在浏览器里，归 WM 层（niri 切窗）+ 外部 session 管理。
- **CDP 脊梁**：控制浏览器里所有 target（含错误页、内置页）+ `Target.targetCreated/Destroyed` 事件实时同步到 sqlite（满足"关闭时更新列表"）+ 会话恢复。
- **SurfingKeys**：普通页面的键盘 / 插入 / 输入法切换（JS 驱动，能自理 IME）。
- **sqlite + launcher**：session 管理（多 session × 多页面），walker 列 session 唤起；地址输入也走 launcher。

## 四、资源占用（多窗口 vs 多标签）

Chromium 是 **process-per-page**（每页面一个 renderer，无论一窗多标签还是一个标签一窗）。因此：

- **同一实例内**开多个 `--app` 窗口 ≈ 一个窗口开同样多的标签：1 个 browser 进程 + 每页 1 个 renderer + 共享 cache/GPU。
- **多个独立实例**（不同 profile）= 各自 browser 进程 + 各自 cache，明显更重。
- 结论：**本方案选 B**——每 session 一个独立实例/工作区，一次只激活一个，换取强隔离与清晰的工作区分组，接受相对更高的并发资源占用。

## 五、地址输入、历史与命令（热键）

- "地址栏"由 **launcher（walker）URL/历史提示 → CDP navigate** 承担：热键唤起 prompt，输入或选历史即跳转。
- 前进 / 后退 / 刷新 = CDP history 命令（`Page.getNavigationHistory` / `navigateToHistoryEntry` / reload）绑键。
- 比内置 omnibox 更贴合"全键盘 + launcher"工作流。

## 六、窗口管理与热键

窗口管理目标：浏览器窗口自成一组，既方便单独唤起，又不淹没通用窗口切换器。

- **独立 walker 菜单（类 cwdhist / windowsmru）**：专列浏览器窗口，查 `niri msg -j windows` 按过滤标准列出，独立热键触发。通用 windowsmru / niri Alt+Tab 按同一标准排除浏览器窗口 → 不被淹没。
- **过滤标准**：原生 Wayland 窗口元数据只有 `app_id` 与 `title`（无任意 tag）→ "元数据过滤"的现实选项 = title 前缀 / app_id / workspace。
  - **title 前缀**：wrapper 打开 `--app` 窗口时加统一前缀（如 `◆` 或 session 名），wrapper 自身可控，最稳。
  - **workspace**：niri 窗口规则把浏览器窗口路由到专用 workspace（如 `web`），菜单按 workspace 过滤，兼具分组意义。
- **热键分配**（暂缓，先纯命令行）：`Mod+Space` → 浏览器窗口菜单；cwdhist 暂不动。

## 七、实现设计（`qw` CLI，方案已定 2026-08-28）

选型：**B（每 session 一实例/工作区）+ Python + 命令名 `qw`**。

### 组件
- **`qw`**（CLI，Python）：命令入口，读 sqlite + 把控制命令发给 daemon。
- **`qwd`**（daemon，Python 常驻）：持各运行实例的 CDP WebSocket，订阅 Target 事件实时写 sqlite、转发控制命令、拉起/停掉实例。
- **每 session 一个 Chromium 实例**：默认不跑，`qw open` 才拉起。
- **sqlite**：`~/.local/share/qw/qw.sqlite`。

### 数据模型
```sql
instances(id, profile TEXT, port INT, pid INT, running INT)   -- 1 session ↔ 1 instance
sessions(id, name UNIQUE, workspace TEXT, instance_id FK,
         created_at, last_opened_at)
pages(id, session_id FK CASCADE, target_id, url, title,
      position INT, opened_at INT, closed_at INT NULL)
```

### session = 工作区
- 每个 session 绑定一个 niri **workspace**（如 `web:<name>`）。
- `qw open <name>` = 切到该 workspace + 拉起该 session 的实例、重建其 pages。
- 实例窗口进对应 workspace：title 前缀（app_id）经 niri 窗口规则路由 + `qw open` 时 `niri msg action focus-workspace <name>` 双保险。
- 切 session = 关当前、开目标；一般一次只活一个实例。

### CLI 命令（v1）
```
qw new <name>              创建 session（配 workspace）
qw open <name> [url...]    启动该 session（切 workspace + 拉实例 + 重建页面）
qw add <name> <url>        当前/指定 session 加一页
qw ls [name]               列 session / 列页面
qw close <name>            关该实例（窗口 + workspace 收起）
qw rm <name>               删 session（+其 profile/workspace）
qw goto <url> | back | forward | reload
qw focus <page>
qw quit                    关全部实例
qw daemon start|stop|status
```

### CDP 映射
- 拉起：`chromium --app=<url> --remote-debugging-port=<动态端口> --user-data-dir=<该 session profile> --no-first-run`
- 开页 `Target.createTarget` / 关页 `Page.close` → 事件同步
- 导航 `Page.navigate` / `reload` / `getNavigationHistory` / `navigateToHistoryEntry` / 聚焦 `Target.activateTarget`

### 事件同步（核心）
`qwd` 收到 `targetCreated/infoChanged` → upsert page；`targetDestroyed` → 置 `closed_at`。实时，无需轮询。

### 实现备注
- Python 依赖仅一个非标准库：**`websocket-client`**（CDP 走 WebSocket，标准库没有；其余 sqlite3 stdlib）。
- CDP 即 JSON 消息，直接手写很轻，不需 jsonrpc 封装。

## 八、已验证 / 待验证

- ✅ `--app` 模式无 omnibox / 标签条，Niri 下纯页面（截图实测）。
- ✅ CDP 能列出并 attach `--app` 窗口（它就是普通 page target）。
- ⏳ SurfingKeys 在 `--app` 窗口的注入未实测（本环境未安装扩展）。
- ⏳ CDP 对错误页（`chrome-error`）reload / navigate 未实测（按机制应当可控）。
- ⏳ 同实例多窗口 ≈ 多标签的资源等价逻辑成立，未实测（本方案走 B，多实例资源为预期代价）。

## 九、开放问题

- **双控制路由**：SurfingKeys（普通页）与 CDP（错误页/内置页）之间，按键谁管、如何按 target 类型判定切换。
- **引擎 UI 缺口**：下载 / 打印等原本由浏览器 chrome 提供的功能，需另行接（CDP / 省略）。