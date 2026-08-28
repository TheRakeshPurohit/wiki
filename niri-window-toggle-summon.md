# Niri 悬浮窗显隐与通知召唤

> **状态**: 配方（2026-08-28）。来源为 AI 生成方案，已对 niri API 实测核实并修正一处编造：`ghostty --app-id` 不存在，识别改为 window title。
> **场景**: AI 聊天悬浮窗常驻后台，一键显/隐 + 来通知自动召回 + 窗口内按钮隐藏。

---

## 一、核心机制

- **实例常驻后台，绝不销毁**。隐藏 = 用 `move-floating-window` 把窗口放逐到屏幕外坐标（如 `9999,9999`）；显示 = 拉回可视坐标（如 `1400,50`）。
- **状态文件**（`/tmp/niri_ai_state`，`in`/`out`）统一三个入口的同步：全局快捷键、窗口内"隐藏"按钮、D-Bus 通知召唤。三者都对同一坐标与状态文件操作，逻辑闭环。

## 二、已核实的 niri API（实测真实）

- `niri msg action move-floating-window -x <X> -y <Y>` —— "Move the floating window on screen"
- `niri msg action focus-window --id <ID>` —— 聚焦指定窗口
- `niri msg action focus-window-down` —— 聚焦下方窗口（隐藏释放焦点用）

## 三、关键修正：识别用 title，不用 app-id

- **`ghostty --app-id <id>` 不存在**：ghostty 的 CLI 与配置都无 `app_id` 可设。Wayland 下 app_id 由 GTK 应用 ID 内部决定，外部给不出去 → 靠 `match app-id="^..."$` 认窗口永远匹配不到。
- **改为 title 识别**：
  - ghostty 用 `window-title` 配置或用 OSC 2 把窗口标题钉成固定值：`printf '\033]2;ai-chat-float\007'`。
  - niri `window-rule match title="^ai-chat-float$"`；脚本里 `win.get("title") == "ai-chat-float"` 判等。
- 与 `keyboard-wm-browser-mode.md` 的窗口过滤同思路：原生 Wayland 窗口可判元数据 = `app_id` 与 `title`；当 app_id 不可控时，title 前缀/全等是最稳的受控身份。

## 四、脚本

### `slide_ai.py`（一键开关，title 识别）
```python
#!/usr/bin/env python3
# ~/.config/niri/scripts/slide_ai.py
import json, subprocess
from pathlib import Path

TITLE      = "ai-chat-float"
STATE_FILE = Path("/tmp/niri_ai_state")
VIEW_X, VIEW_Y = 1400, 50
HIDE_X, HIDE_Y = 9999, 9999

def run_niri(args):
    subprocess.run(["niri", "msg", "action"] + args, check=True)

def get_ai_window():
    out = subprocess.run(["niri", "msg", "-j", "windows"],
                         capture_output=True, text=True, check=True).stdout
    return next((w for w in json.loads(out) if w.get("title") == TITLE), None)

def main():
    win = get_ai_window()
    if not win:                                   # 冷启动
        subprocess.Popen(["ghostty", "-e", "aichat"])
        STATE_FILE.write_text("in"); return
    state = STATE_FILE.read_text().strip() if STATE_FILE.exists() else "in"
    wid = win["id"]
    if state == "in":                             # 放逐隐藏
        run_niri(["focus-window", "--id", str(wid)])
        run_niri(["move-floating-window", "-x", str(HIDE_X), "-y", str(HIDE_Y)])
        run_niri(["focus-window-down"])
        STATE_FILE.write_text("out")
    else:                                         # 召回显示
        run_niri(["focus-window", "--id", str(wid)])
        run_niri(["move-floating-window", "-x", str(VIEW_X), "-y", str(VIEW_Y)])
        STATE_FILE.write_text("in")

if __name__ == "__main__":
    main()
```

### `notification_summon.py`（D-Bus 通知监听召回，title 识别）
- 用 `dbus-monitor` 子进程管道流式读 `interface='org.freedesktop.Notifications',member='Notify'`，匹配到目标来源 App 后立即召回 AI 窗（聚焦 + 拉回 + 状态 `in`）；进程没起则冷启动。
- 识别同样用 `win.get("title") == TITLE`。

## 五、Niri 配置

```
window-rule {
    match title="^ai-chat-float$"
    open-floating true
    default-column-width { fixed 450px; }
    default-window-height { fixed 650px; }
}

spawn-at-startup "/usr/bin/env" "python3" "~/.config/niri/scripts/notification_summon.py"

binds {
    Ctrl+Return { spawn "python3" "~/.config/niri/scripts/slide_ai.py"; }
}
```

## 六、客户端内"隐藏"按钮

前端代码里（Python/TUI）点按钮即执行两行：`move-floating-window -x 9999 -y 9999` + 状态文件写 `out`，与全局快捷键无缝同步。

## 七、备注

- **dbus-monitor 已被上游标记弃用**。现状能跑；若要"原生 D-Bus 库"，换 `python-dbus-next`（类型安全）或 `dbus-python`，替代子进程管道。
- 本机 niri 配置与脚本通常走 NixOS 资产（`modules/desktop/assets/niri/` + `nh os switch` 部署），`~/.config/niri/scripts/` 这类裸路径若在 store symlink 下不会持久，需据实际部署方式调整路径。