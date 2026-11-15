# Huawei MateBook 14 (2019) — macOS Tahoe 26 黑苹果 EFI

[![macOS](https://img.shields.io/badge/macOS-26.7.3-black?logo=apple)](https://www.apple.com/macos/)
[![OpenCore](https://img.shields.io/badge/OpenCore-1.0.9-blue)](https://github.com/acidanthera/OpenCorePkg)
[![Status](https://img.shields.io/badge/status-working-brightgreen)]()

华为 MateBook 14 2019（KLV-W19 / KLV-W29）安装 **macOS Tahoe 26.7.3** 的 OpenCore 配置与完整踩坑记录。

> ⚠️ **本项目只提供 EFI 与经验，不含任何 Apple 版权文件。使用前请自行生成 SMBIOS 三码（仓库内的序列号已清空）。**

---

## 硬件

| 部件 | 型号 | 状态 |
|---|---|---|
| 机型 | Huawei MateBook 14 2019 (KLV-W19/W29) | — |
| CPU | Intel Core i5-8265U（Whiskey Lake-U，4C8T） | ✅ 8 线程全开 |
| 核显 | Intel UHD Graphics 620（真实 ID `0x3EA0`，伪装 `0x3EA5`） | ✅ 1536 MB，硬解正常 |
| 独显 | NVIDIA MX250 | ❌ macOS 无解，SSDT `_OFF` 关闭 |
| 内存 | 8 GB LPDDR3 2133 MHz（板载） | ✅ |
| 硬盘 | WDC PC SN720 SDAPNTW-512G NVMe | ✅ 原生支持，无需 NVMeFix 也能跑 |
| 屏幕 | 14" 2160×1440 3:2 | ✅ 内屏点亮、亮度可调 |
| 无线 | Intel CNVi（PCI `8086:9df0`，9560 同类） | ✅ `AirportItlwm_Tahoe` 原生界面 |
| 蓝牙 | 同上 CNVi 模块 | ✅ 见「状态」 |
| 声卡 | Realtek ALC256 | ⚠️ 见「未验证」 |
| 触控板 | Synaptics I2C HID | ✅ `VoodooI2C` + `VoodooI2CHID` |

## 状态

| 功能 | 状态 | 说明 |
|---|---|---|
| 内屏 / 亮度 | ✅ | `framebuffer-con0` + PNLF |
| Metal 硬件加速 | ✅ | UHD 620 1536 MB |
| Wi-Fi | ✅ | `AirportItlwm_Tahoe` 2.4.0，系统原生界面，无需 HeliPort |
| 蓝牙 | ✅ | `IntelBluetoothFirmware` 2.5.2 + `BTPatcher` + `BlueToolFixup` 2.7.3 |
| 键盘 / 触控板 | ✅ | `VoodooPS2` + `VoodooI2C` |
| 电池百分比 | ✅ | `SMCBatteryManager` |
| 睡眠 / 唤醒 | ⏳ | 未长时间验证 |
| 音频 ALC256 | ⏳ | `AppleALC` 已载入，layout 76，未系统验证 |
| HDMI 输出 | ⏳ | `framebuffer-con1/con2` 已按 14.5 时期可用配置写入 |
| 读卡器 / 指纹 | ❌ | 无解 |
| MX250 | ❌ | 无解（已被 SSDT 关闭，省电） |

---

## 安装方式（全程 Windows，无需 Mac）

这是本项目最有价值的部分：**没有 Mac 也能从零装机**。

### 1. 拿恢复镜像

```cmd
python macrecovery.py -b Mac-27AD2F918AE68F61 -m 0000000000000000 -os latest download
```

得到 `com.apple.recovery.boot\BaseSystem.dmg`（约 916 MB）+ `BaseSystem.chunklist`。

### 2. ⚠️ 不要直接用 `com.apple.recovery.boot`

> **这是本项目的核心教训。** OpenCore 加载 `BaseSystem.dmg` 的方式是**把整个 916 MB 镜像读进内存**（日志里是 `OCRAM: Extent allocation of 960530321 bytes`）。在 8 GB 机器上，这一步会直接导致复位重启——现象是「选中恢复镜像后黑屏约 30 秒，然后自动重启」。

正确做法是把它还原成**真实的 HFS+ 分区**：

```bash
# Linux / 有 WSL 或 Arch 的机器
dmg2img -i BaseSystem.dmg -o BaseSystem.img     # 输出约 2.49 GB（含 GPT 的完整磁盘镜像）
```

Windows 下用 [dmg2img Win32 版](http://vu1tur.eu.org/tools/)，然后用 **Rufus / balenaEtcher** 把 `.img` 按 **DD 模式**整盘写入 U 盘。

写完 U 盘上会是一个 **2.4 GB 的 Apple HFS 分区** + 剩余未分配空间。

### 3. 在剩余空间建 EFI 分区

用 DiskGenius：

1. 右键未分配空间 → 建立新分区 → **FAT32** → 保存更改
2. 右键新分区 → 修改分区参数 → 类型改为 **EFI System Partition**
3. 把本仓库的 `EFI` 文件夹拷到该分区根目录

> 那个 2.4 GB 的 HFS 分区**不要再做任何操作**（不格式化、不建分区）。

### 4. BIOS

- Secure Boot → **Disabled**
- Fast Boot → **Disabled**
- 启动模式 UEFI

### 5. 安装

1. F12 选 U 盘 → OpenCore 菜单 → 选**不带 `(dmg)` 后缀**的那个条目
2. 进入「macOS 恢复」→ 磁盘工具 → **⌘2 显示所有设备**
3. 选中要装 macOS 的分区（Linux 分区在这里会显示为 `Unknown` 格式）→ **Erase** → APFS
4. 退出磁盘工具 → 重新安装 macOS → 需要联网（下载约 15 GB）

> 首次显示「约 9 小时」是正常的高估，一般 10 分钟内会掉到真实值。

### 6. 安装中的重启

**每次重启都要在 OpenCore 菜单里手动选对条目**：

```
macOS Installer        ← 有就优先选它
Tahoe                  ← Installer 消失后选硬盘
```

---

## EFI 组成

### OpenCore

- **1.0.9**（release），Boot Picker 用 **Builtin 文本菜单**
  - ⚠️ 不要用 `PickerMode=External` + 残缺主题包：日志会报
    `OCUI: Failed to load image ...Background.icns - Not Found`，**OpenCanopy 画不出界面，菜单一帧都不显示**，机器直接回落到 Windows。

### ACPI

| 文件 | 作用 |
|---|---|
| `SSDT-MB14.aml` | PNLF 亮度、dGPU `_OFF`、USBX、fake EC、`_Q0A/_Q0B/_Q10` 热键 |

配套 `ACPI → Patch` 三条改名：`_Q0A→XQ0A`、`_Q0B→XQ0B`、`_Q10→XQ10`。

### Kext（完整配置）

| Kext | 版本 |
|---|---|
| Lilu | 1.7.3 |
| VirtualSMC | 1.3.9 |
| WhateverGreen | 1.7.2 |
| RestrictEvents | 1.1.7 |
| NVMeFix | 1.1.4 |
| SMCBatteryManager | 1.3.9 |
| VoodooPS2Controller | 2.3.8 |
| VoodooI2C / I2CHID | 2.9.1 / 1.0 |
| AirportItlwm_Tahoe | 2.4.0 |
| IntelBluetoothFirmware / BTPatcher | 2.5.2 |
| BlueToolFixup | 2.7.3 |
| AppleALC | 1.9.9 |
| HibernationFixup | 1.5.5 |

### 关键配置项

```
boot-args:
  -v keepsyms=1 -wegnoegpu -igfxblr -igfxblt -igfxonln=1
  revpatch=sbvmm -no_compat_check
```

| 项目 | 值 | 为什么 |
|---|---|---|
| `Kernel → Quirks → DisableIoMapper` | **true** | 华为 Insyde BIOS **不开放 VT-d 开关**，VT-d 始终开启。这一项设成 false 会在 PCI 枚举阶段卡死，表现为「黑屏约 1 分钟后复位」——**本项目最隐蔽的坑** |
| `Kernel → Quirks → CustomSMBIOSGuid` | false | 与 `UpdateSMBIOSMode=Create` 互斥 |
| `Booter → Quirks → DevirtualiseMmio` | false | 本机固件不需要，保持 14.5 时期实测可用值 |
| `PlatformInfo → UpdateSMBIOSMode` | Create | — |
| `Misc → Security → SecureBootModel` | Disabled | — |
| `NVRAM → revpatch` | `sbvmm` | MacBookPro15,2 不在 Tahoe 支持列表，靠 RestrictEvents 伪装 board-id |
| 显卡 `AAPL,ig-platform-id` | `0x3EA504` | 14.5 时期实测可用，沿用 |
| 显卡 `device-id` | `0x3EA5`（伪装自 `0x3EA0`） | — |
| 显卡 `force-online` | `01000000` | Tahoe 下 eDP 检测异常时会黑屏，此项强制点亮 |
| `UEFI → Drivers` | HfsPlus / OpenRuntime / ResetNvramEntry | 不用 OpenCanopy |

---

## 踩坑记录（按踩到的顺序）

详细版见 [docs/PITFALLS.md](docs/PITFALLS.md)，这里是速查表：

| # | 现象 | 真正原因 | 解决 |
|---|---|---|---|
| 1 | 选 Recovery → 黑屏 1 分钟 → 自动重启进 Windows | `DisableIoMapper=false`，华为 BIOS 关不掉 VT-d | 改成 `true` |
| 2 | 同上，改完仍然黑屏 | 显卡缺 `force-online` | 加 `01000000` |
| 3 | 照搬 T490 配置后开不了机 | 混入了 T490 专属残留：`ReservedMemory(0x8B000)`、`BlessOverride(fedora)`、`HibernateMode=NVRAM`、`IOPCIFamily` 补丁 | 全部删除 |
| 4 | 换了配置后**连菜单都不出现** | 主题包缺 `Background.icns`，OpenCanopy 渲染失败 | 改 `PickerMode=Builtin` |
| 5 | 开了 DEBUG 版日志后，60 秒后复位，日志停在半路 | 每行日志约 180 ms（刷屏 + 写 U 盘），401 行 ≈ 60 秒，被拖到超时 | `Target=65`（只写文件不刷屏）或 `0` |
| 6 | 找不到 `opencore-*.txt` | release 版没有文件日志；`Target` 必须含 `0x40` | 用 DEBUG 版 + `Target=67` |
| 7 | 选中恢复后黑屏 30 秒后复位，日志最后一行 `OCRAM: Extent allocation of 960530321 bytes` | OpenCore 把 916 MB 的 DMG 读进内存 | 改用 `dmg2img` 还原成真实分区 |
| 8 | 卡在 `Waiting on ... boot-uuid-media`，最后一行 `(Flash Disk): MODE_SENSE_06 failed` | 那根 U 盘在 macOS 的 USB 栈下无法发布块设备（固件能读、系统读不到） | 换 U 盘 / SD 卡 + 读卡器 |
| 9 | 换最小配置后触控板键盘全废 | `config-minimal.plist` 只有 5 个 kext，没有输入驱动 | 预期行为，换回完整配置 |
| 10 | 出现电池图标 | macOS 的「电量不足需充电」提示 | 插电 |
| 11 | 装到一半提示 *An Internet connection is required* | BaseSystem 需要联网下载 15 GB | 连 Wi-Fi（`AirportItlwm_Tahoe`） |
| 12 | 磁盘工具里看不到 Linux 分区 | ext4 不被识别，默认只显示可挂载卷 | **⌘2** 显示所有设备；或用「Partition」对话框把 Format 直接改成 APFS |
| 13 | 分区对话框提示 *This volume can't be resized* | 点成了「分区」而不是「抹掉」 | 直接改 Format → APFS → Apply，或用 Erase |

---

## 仓库结构

```
.
├── EFI/
│   ├── BOOT/BOOTx64.efi
│   └── OC/
│       ├── ACPI/SSDT-MB14.aml
│       ├── Drivers/{HfsPlus,OpenRuntime,ResetNvramEntry}.efi
│       ├── Kexts/            16 个 kext（config.plist 启用 14 个）
│       ├── Resources/        仅 Font（Builtin 菜单用不到图片）
│       ├── config.plist          ← 默认：输入 + Wi-Fi + 电池（装机验证过）
│       ├── config-full.plist     ← 追加蓝牙 / 音频 / 休眠
│       └── config-minimal.plist  ← 仅 5 个 kext，用于二分排查
├── docs/PITFALLS.md
├── LICENSE
└── README.md
```

## 使用前必须做的两件事

### 1. 生成自己的 SMBIOS

仓库内的三码已清空。用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 或 OCAT 为 **MacBookPro15,2** 生成：

- `SystemSerialNumber`
- `MLB`
- `SystemUUID`（可用 `uuidgen`）
- `ROM`（填本机有线网卡 MAC，或任意 `112233445566` 形式的 6 字节）

填到 `PlatformInfo → Generic`，并把 `UpdateNVRAM` 改回 `true`。

> MacBookPro15,2 不在 Tahoe 官方支持列表，靠 `revpatch=sbvmm` + `-no_compat_check` 启动，不需要也不建议改成 MacBookPro16,x（会牵扯 USB 端口映射与 T2 相关行为）。

### 2. 确认 `DisableIoMapper = true`

如果你不是华为/Insyde 固件，这一项按 Dortania 指南取值即可；**MateBook 上必须是 `true`**。

## 已知问题 / 待办

- [ ] 音频（ALC256 / layout 76）未系统验证
- [ ] 睡眠唤醒、合盖休眠未长期验证
- [ ] HDMI 外接未验证
- [ ] CPU 电源管理未做 `SSDT-PLUG`（当前靠默认，未测功耗温度）
- [ ] USB 端口未做 `USBMap`（当前靠系统默认，USB 3.0 速度未经测试）
- [ ] iServices（iMessage / FaceTime）需自行生成三码后测试

## 致谢

- [Acidanthera](https://github.com/acidanthera) — OpenCore / Lilu / VirtualSMC / WhateverGreen / RestrictEvents
- [5T33Z0 / Thinkpad-T490-Hackintosh-OpenCore](https://github.com/5T33Z0/Thinkpad-T490-Hackintosh-OpenCore) — 同为 i5-8265U，本项目的 kext 版本与结构参考来源
- [laobamac](https://github.com/laobamac) — Tahoe 版 `AirportItlwm`
- [VoodooI2C](https://github.com/VoodooI2C) — Synaptics I2C 触控板

## License

配置与文档以 MIT 发布。EFI 内的所有二进制均为各自项目所有，遵循其原始许可证。
