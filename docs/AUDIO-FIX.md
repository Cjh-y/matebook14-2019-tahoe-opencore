# 声卡修复：为什么没声音 + 怎么弄

## 原因很明确：配置里根本没装声卡驱动

我核对了一下你现在 U 盘 / ESP 上那份 `config.plist`：

```
已启用的 14 个 kext：
Lilu / VirtualSMC / WhateverGreen / NVMeFix / RestrictEvents
VoodooPS2 / VoodooI2C 系列 / SMCBatteryManager / AirportItlwm_Tahoe
```

**没有 `AppleALC`**，而且 `DeviceProperties` 里**只有显卡那一项**，没有声卡的 `layout-id`。

等于说：macOS 认到了 HD Audio 控制器，但没有任何东西告诉它「这个 codec 是 ALC256、用 76 号布局」，所以输出设备列表里什么都没有。

> 这不是驱动坏了，是当初为了「先把系统装起来」，声卡被我特意剔掉了（`config-minimal` / 安装配置都去掉了音频）。现在补回来。

## 修复包

本目录里的文件：

```
EFI/OC/config.plist              ← 替换
EFI/OC/config-full.plist         ← 替换
EFI/OC/config-verbose.plist      ← 替换（排障用）
EFI/OC/Kexts/AppleALC.kext       ← 新增（1.9.9）
EFI/OC/Kexts/SimpleHDA.kext      ← 新增（1.0.0，laobamac）
```

**操作**：把这几项按原路径覆盖到 `S:\EFI\OC\` 里（`Kexts` 是往里放，`config*.plist` 是覆盖）。然后重启。

### 为什么还要 SimpleHDA

macOS Tahoe 的 AppleHDA 已经不再按老方式工作，同配置的 T490 在 Tahoe 上用的是 **AppleALC 1.9.9 + SimpleHDA** 这套组合（SimpleHDA 是 laobamac 重写的 AppleHDA 替代品，挂在 `IOResources` 上，不需要 root patch）。

所以这次两个都放进去，顺序是：

```
Lilu → VirtualSMC → WhateverGreen → AppleALC → SimpleHDA → ...
```

## 重启后怎么验证

1. 右上角菜单栏的**音量图标**应该出现（不再是灰的或者插孔图标）
2. 系统设置 → 声音 → **输出**里应该能看到 `内置扬声器` / `Internal Speakers`
3. 终端里跑：

```bash
system_profiler SPAudioDataType | head -40
```

正常会列出 codec（ALC256）、输出设备、当前音量。

再看驱动有没有加载：

```bash
kextstat | grep -iE 'applealc|simplehda|applehda'
```

应该能看到 `as.vit9696.AppleALC` 和 `cn.laobamac.SimpleHDA`。

## 如果还是不响

按顺序试：

### 1. 换 layout-id

ALC256 有好几个可用布局，76 是这台机器在 14.5 上验证过的，但 Tahoe 上不一定最合适。改 `config.plist` 里的：

```
DeviceProperties → Add → PciRoot(0x0)/Pci(0x1f,0x3) → layout-id
```

值是一段 4 字节 data。常用的几个（十六进制，小端）：

| layout | 填的值 |
|---|---|
| 76 | `4c000000` |
| 21 | `15000000` |
| 3 | `03000000` |
| 11 | `0b000000` |
| 56 | `38000000` |
| 66 | `42000000` |
| 69 | `45000000` |

改一个 → 重启 → 看声音设置。建议 76 → 21 → 3 这个顺序试，每次只改一个。

### 2. 只保留 AppleALC（去掉 SimpleHDA）

如果 AppleHDA 其实还在（部分版本），SimpleHDA 反而可能抢占。检查：

```bash
ls -d /System/Library/Extensions/AppleHDA.kext
```

- 存在 → 把 `config.plist` 里 `SimpleHDA.kext` 那一条的 `Enabled` 改成 `false`，只留 AppleALC
- 不存在 → 说明确实需要 SimpleHDA，保留两者

### 3. 看日志

把 `config-verbose.plist` 改名成 `config.plist` 启动，屏幕上会滚字，找这几类：

```
AppleALC: ...
AppleHDA: ...
SimpleHDA: ...
```

如果 `AppleALC` 一行都没有，说明 kext 没被加载（检查 `Kernel → Add` 里是否启用、路径是否正确）。

## 一个提醒

改完记得**不要动 `csr-active-config`**（当前是 `0x803`）。AppleALC + SimpleHDA 这套不需要 root patch，SIP 保持现状即可。
