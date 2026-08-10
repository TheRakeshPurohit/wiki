# 键盘键位重映射设计

> 与 [vim-movement-critique.md](vim-movement-critique.md)（移动键/范式批判）互补：本文聚焦**修饰键**的重映射实现，记录在 evdev 层用 kanata 做键位重映射的设计决策与完整配置。决策档案见 NixOS `docs/adr/019-keyboard-keymap-design.md`。

---

## 一、定位：evdev 层 vs XKB keysym 层

键盘重映射有两个层次，解决的问题不同：

| 层次 | 作用对象 | 生效范围 | 局限 |
|------|---------|---------|------|
| **XKB** | keysym（逻辑键） | 走 keysym 的常规应用 | **Blender 等以 keycode（物理键码）为输入源的软件不认**——这正是最初发现 caps↔ctrl 不生效的根因 |
| **kanata（evdev）** | keycode（内核输入层） | 先于 TTY、X11、Wayland、Blender，对所有应用一视同仁，TTS 控制台也生效 | 不提供 XKB 的 **per-window 布局记忆 / 死键 / Compose 等有状态的输入组合逻辑**（kanata 自带 deflayer 多层切换，但那是重映射动作层，与 XKB 的布局层是两回事） |

选择 kanata 的理由：问题出在"某些应用不认 keysym 映射"，只有下沉到 evdev 层才能对所有应用（含 Blender）一致生效。

---

## 二、核心决策

只保留两条映射，其余键全部还原为普通键：

| 物理键 | 行为 |
|--------|------|
| CapsLock | 点击输出 `esc`，按住输出 Ctrl（双模） |
| 物理左 Ctrl | 改回 caps（大小写锁定） |

shift / alt / 右 Ctrl **全部还原为普通键，不做任何双模映射**。

---

## 三、关键取舍

### 1. 为什么用 kanata（evdev 层）而非 XKB

XKB 的 `ctrl:swapcaps` 作用在 keysym 层，只对走 keysym 的常规应用生效。**Blender 等以 keycode（物理键码）为输入源的软件不认 XKB 映射**——这正是最初发现问题（Blender 中 caps↔ctrl 不生效）的根因。kanata 作用在 evdev（内核输入层，keycode），先于所有应用层，对所有软件一致生效。

### 2. 为什么 shift 完全还原，不做双模

最初尝试过把括号绑定到 shift 双模（点击 shift 输出 `(`, 按住保持 shift），但**占满 shift 会连锁破坏依赖 shift 原始语义的功能**：

- RIME 输入法单击 shift 切换中英文
- idea 双击 shift 搜索
- 大写输入

这些功能靠"shift 键本身被点击"的键事件触发。一旦 shift 的双模把 tap 重定向成输出字符，应用层就永远收不到 shift 被点击的事件。**牺牲点按副作用最小的键，会破坏所有依赖该键原生语义的应用。**

### 3. 为什么括号不设快捷方式

尝试过把括号挪到 Alt 双模（左 Alt 点击 `(`, 右 Alt 点击 `)`），但：

- 右 Alt 位置难按
- Alt 键位容易受空格键长度影响
- `()` 输入本身已习惯，用物理 `shift+9` / `shift+0` 即可

**程序员对符号区（shift+数字）的熟练度和主键区区别不大**，不需要额外快捷键。占满 Alt 还会破坏 Alt+Tab 等组合键，得不偿失。

### 4. tap-hold 的延迟权衡

caps 双模用 `tap-hold-press`：

- 按住 CapsLock 再按另一键 → 立即判定为 hold（组合键零延迟）
- 点按 CapsLock → 输出 esc

`tap-hold-require-prior-idle 150` 让打字流中的点按立即判 tap。组合键零延迟，CJK 等场景无感。

---

## 四、配置方法

### 1. kanata.nix（evdev 层重映射）

```nix
{ lib, ... }:
{
  # ── kanata：evdev 层键盘重映射（Rust 实现） ─────────────
  # 作用在内核输入层（keycode），对所有应用生效（含 Blender、游戏、TTY、Wayland 原生）。
  # 与 XKB（keysym 层）不同：XKB 的 ctrl:swapcaps 只对走 keysym 的应用生效，
  # kanata 直接替换物理键码，Blender 等以 keycode 为输入源的软件也认。
  #
  # 键位取舍与理由见 ADR-019: docs/adr/019-keyboard-keymap-design.md
  #
  # 设计（用户为 QMK 用户，kanata 的 tap-hold 对应 QMK 的 tap-hold）：
  # 1. caps 双模（物理 CapsLock）：点击输出 esc，按住作 Ctrl。
  # 2. 物理左 ctrl → caps（改回大小写锁定键）。
  #    其余键（shift/alt/右 ctrl）全部还原为普通键：
  #    - shift 保留 RIME 单击切换中英文、idea 双击 shift 搜索等原语义
  #    - 括号不设快捷方式，直接用物理键 shift+9 / shift+0（程序员对符号区熟练）
  #
  # 延迟控制（对应 QMK 的 HOLD_ON_OTHER_KEY_PRESS + require-prior-idle）：
  # - tap-hold-press：按住 tap-hold 键时再按下另一键 → 立即判 hold（组合键零延迟）
  # - tap-hold-require-prior-idle 150：打字中（距上个键 <150ms）按下 tap-hold 键
  #   → 立即判 tap（打字流中的符号输出零延迟）
  # - 停顿后孤立点按（如句首）仍走 wait 状态 → 保留一段判定窗口，这是 tap-hold
  #   的物理边界，任何方案（含 QMK/keyd）都无法消除。
  #
  # process-unmapped-keys yes：未被 defsrc 列出的键（字母/数字/功能键等）照常
  # pass-through，且让 tap-hold 的提前判定能看到它们（消除组合键延迟）。
  services.kanata = {
    enable = true;

    keyboards.default = {
      # devices = [] 自动检测所有键盘设备
      # defcfg 的其余选项走 extraDefCfg（模块自动生成 defcfg 头部，含 linux-dev）
      extraDefCfg = ''
        process-unmapped-keys yes
        tap-hold-require-prior-idle 150
      '';

      config = ''
        (defalias
          ;; caps（物理 CapsLock）：点击 esc，按住左 ctrl
          caps-mod (tap-hold-press 200 200 esc lctl)
          ;; 物理左 ctrl → caps（改回大小写锁定键）
          lctl-mod caps)

        (defsrc
          caps lctl)

        (deflayer base
          @caps-mod @lctl-mod)
      '';
    };
  };
}
```

### 2. sys.nix（移除 XKB swapcaps，避免双层冲突）

```nix
  # 注意：caps/ctrl 交换已由 kanata（evdev 层）接管，见 ./kanata.nix。
  # 这里不设 options（避免与 kanata 双重交换）。
  services.xserver.xkb = {
    # 不设置 options = "ctrl:swapcaps"
  };
```

**关键**：必须从 `sys.nix` 移除 XKB 的 `options = "ctrl:swapcaps"`，否则与 kanata 双层交换冲突。

---

## 五、后果

- 物理 CapsLock 不再是大写锁定（改由物理左 Ctrl 充当 caps）
- 物理左 Ctrl 失去 Ctrl 功能（改由 CapsLock 按住充当）
- 需要 `process-unmapped-keys yes` 让未映射键（字母/数字/shift/alt 等）照常 pass-through
- kanata 全局启用（所有 host），**SSH 输入不受影响**（evdev 层，不碰网络输入）

---

## 六、与 QMK 的对应

用户是 QMK 用户，kanata 的 tap-hold 概念与 QMK 一一对应：

| kanata | QMK |
|--------|-----|
| `tap-hold-press` | `HOLD_ON_OTHER_KEY_PRESS` |
| `tap-hold-require-prior-idle 150` | `require-prior-idle` |
| 双模键输出 | `LT` / tap-hold |

物理键盘层（QMK 固件）与系统层（kanata）可协同：QMK 处理硬件层，kanata 处理系统层，二者互不冲突。