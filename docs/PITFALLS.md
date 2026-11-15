# 踩坑全记录：从「黑屏重启」到桌面

按时间顺序，记录每个现象、排查过程和真正原因。每一条都附上真实日志证据，方便后来者对照。

---

## 一、起点：三份 EFI 混搭失败

最初手里有三份配置：

1. 本机 macOS 14.5 时期的 EFI（能跑）
2. 同为 i5-8265U 的 ThinkPad T490 在 macOS 26 上的 EFI（能跑）
3. 之前用这两份「混」出来的 Tahoe EFI（选 Recovery 后黑屏，约 1 分钟后自动重启进 Windows）

直觉会以为「同 CPU 的配置可以直接抄」，但 ③ 的问题正是抄的时候没筛选。

### 逐项对比发现的问题

| 项目 | ① 14.5 可用 | ③ 混搭 | 后果 |
|---|---|---|---|
| `Kernel → Quirks → DisableIoMapper` | **true** | false | **最致命**，见下 |
| 显卡 `force-online` | 无（14.5 不需要） | 无 | Tahoe 下黑屏 |
| `CustomSMBIOSGuid` vs `UpdateSMBIOSMode` | false + Create | true + Create | ocvalidate 直接报错 |
| `UEFI → ReservedMemory` | 空 | 0x8B000 / 4KB RuntimeCode | T490 治休眠的补丁，本机无用 |
| `Misc → BlessOverride` | 空 | `\EFI\fedora\grubx64.efi` | 无用 |
| `Booter → Patch` | 空 | 「Skip Board ID check」 | 来路不明的二进制补丁 |
| `Kernel → Patch` | 空 | 两条 `IOPCIFamily` PCI 枚举补丁 | 本机从未需要 |
| `Misc → Boot → HibernateMode` | None | NVRAM + `hbfx-ahbm` | 全新安装没有休眠镜像 |
| `UEFI → Output → GopPassThrough` | Disabled | Enabled | 与本机固件不匹配 |
| `UEFI → Audio → PlayChime` | Auto | Enabled 但没有 `AudioDxe.efi` | 自相矛盾 |
| `UEFI → Input → TimerResolution` | 50000 | **0**（非法） | 按键滤波异常 |
| `Misc → Boot → Timeout` | 5000 | **5**（毫秒） | 菜单一闪而过 |

### `DisableIoMapper` 为什么是致命的

华为 Insyde BIOS 不开放 VT-d 开关，VT-d 永远开启。`DisableIoMapper=false` 时，macOS 在 PCI 枚举阶段拿不到设备映射，整机卡住，几十秒后被固件复位——**外观上就是「黑屏一分钟然后重启」**，完全看不出是配置问题。

这条是整件事里最贵的一课：**同 CPU ≠ 同固件**，BIOS 提供不了的选项必须靠 quirk 补。

---

## 二、加日志：先学会让 OpenCore 说话

### 关键参数

```
Misc → Debug → Target = 67      # 0x43 = 日志(0x01) + 屏幕(0x02) + 文件(0x40)
Misc → Debug → DisplayLevel = 0x80000042
Misc → Debug → AppleDebug = true
Misc → Debug → ApplePanic = true
```

- **文件日志只有 DEBUG 版 OpenCore 才写**
- 日志文件名 `opencore-YYYY-MM-DD-HHMMSS.txt`，落在 ESP 根目录
- 取日志：Windows 里 `diskpart` → 选 U 盘 → `select partition 1` → `assign letter=X`

### 第一个陷阱：日志本身把机器拖死

DEBUG 版 + 刷屏时，每行日志约 **180 ms**（屏幕刷新 + 写 FAT 分区）。401 行 = **59.9 秒**，正好到固件超时：

```
00:178 OC: Setting devprop PciRoot(0x0)/Pci(0x2,0x0):device-id - Success
00:205 OCBP: Predefined <nil> \EFI\BOOT\BOOTX64.EFI was found
...
59:946 OCB: Registering entry OPENCORE [Auto] (T:1|F:0|G:1|E:1|B:0)
```

日志到 59.946 秒戛然而止，**一行错误都没有**——因为机器是被超时复位的，不是软件出错。

> 教训：**排查「卡住」时，先把日志改成 `Target=65`（只写文件，不刷屏）**，否则日志本身就是变量。

---

## 三、菜单根本出不来

改好配置后，连 OpenCore 菜单都不显示了。日志里：

```
OCUI: Failed to load image ...\GoldenGateExt\Background.icns - Not Found
OCUI: Failed to load image ...\OpticalDrive.icns - Not Found
OCUI: Failed to load image ...\ExtTool.icns - Not Found
```

检查主题目录：144 个图标里**缺 `Background.icns` 等 5 个文件**。OpenCanopy 没有背景图就不渲染，菜单一帧都不显示，OpenCore 直接退出 → 固件继续启动 Windows。

**解决**：`Misc → Boot → PickerMode = Builtin`，并把 `OpenCanopy.efi` 从 Drivers 移除。文本菜单零依赖，排查阶段强烈推荐。

---

## 四、恢复镜像：三个独立的问题叠在一起

### 4.1 镜像本身是好的

先把「镜像坏了」这个怀疑排除掉——直接向苹果接口查当前下发的镜像：

```
BaseSystem.dmg  Content-Length: 960,530,321   (916.03 MiB)
```

和本地文件字节数完全一致，说明下载完整。所以问题不在镜像。

### 4.2 但 OpenCore 加载它的方式会崩

选 `(dmg)` 条目后黑屏约 30 秒（916 MB ÷ 30 MB/s，正好是 USB 2.0 的速度），日志最后一行：

```
13:177 OCB: Should boot from 3. OPENCORE (T:4|F:1|G:0|E:1|DEF:0)
13:287 OCRAM: Extent allocation of 960530321 bytes (A) gave 100000000
```

OpenCore 在 **4 GB 地址上申请 916 MB 连续内存**，把整个 DMG 读进去，然后……没有然后了。

> **这个固件/内存组合没法完成这一步。**

### 4.3 正确解法：还原成真实分区

```bash
dmg2img -i BaseSystem.dmg -o BaseSystem.img
```

镜像内部结构（用 `plistlib` 解析 UDIF 资源叉得到）：

```
MBR / Primary GPT / Apple_HFS 2.36 GiB / Apple_Free 128 MiB / Backup GPT
整盘 2.49 GiB
```

也就是说它本来就是一个**完整的 GPT 磁盘镜像**，按 DD 模式整盘写入 U 盘即可，之后 OpenCore 会走普通的 `\System\Library\CoreServices\boot.efi` bless 路径，**完全不需要那块 916 MB 内存**。

---

## 五、换了三个 U 盘才通

### 现象

还原成真实分区后，卡在：

```
Waiting on <dict ...><string ID="2">IOBSD</string><string ID="4">boot-uuid-media</string>
AppleFileSystemDriver: using boot-uuid
...
IOSCSIPeripheralDeviceType00: ... (Flash Disk): MODE_SENSE_06 failed after 0 retries
```

`boot-uuid-media` 是内核在等根分区出现；`MODE_SENSE_06 failed` 是 USB 大容量存储的 SCSI 命令失败。

### 判断

**固件能读这根盘（能引导 OpenCore、能加载 boot.efi），但 macOS 的 USB 栈发布不了块设备。**

- 换 USB-A / USB-C / 拓展坞 → 无效
- 换一根普通 U 盘 → **直接通过**

> 教训：**「固件能读」不等于「macOS 能读」**。卡在 `boot-uuid-media` + `MODE_SENSE` 报错时，别在配置上耗，换介质最省时间。SD 卡 + 读卡器成功率通常最高。

---

## 六、能进恢复界面之后

### 6.1 触控板键盘全失效

换成了 `config-B-minimal`（只有 Lilu / VirtualSMC / WhateverGreen / NVMeFix / RestrictEvents）——**里面没有任何输入驱动**，内置键盘和 I2C 触控板当然全废。这是排查用的最小配置，属于预期行为。

### 6.2 电池图标

macOS 的「电量不足，需要充电才能启动」。恢复环境从 U 盘读系统非常费电，**全程必须插电**。

### 6.3 *An Internet connection is required to install macOS*

BaseSystem 恢复环境要联网下载约 15 GB 安装包。

- 无线网卡是 Intel CNVi（PCI `8086:9df0`），`AirportItlwm_Tahoe` 2.4.0 可用
- 判断网卡是否能被驱动，用 `ioreg` 比 `system_profiler` 靠谱（恢复环境**没有** `system_profiler`）：

```bash
ioreg -r -c IOPCIDevice -l -w0 | grep -o 'pci[0-9a-f]*,[0-9a-f]*' | sort -u
ifconfig -a
```

`en0` 存在但 `status: inactive` + 随机 MAC（`d2:` 开头）＝驱动已挂载但未关联。

### 6.4 磁盘工具看不到 Linux 分区

ext4 不被 macOS 识别，磁盘工具默认只显示能挂载的卷。

- **⌘2「显示所有设备」**
- 点 `Erase`（不是 `Partition`）。如果误开了「分区」对话框，提示 *This volume can't be resized*，直接把 **Format** 从 `Unknown` 改成 **APFS** → `Apply` 也一样可行。

---

## 七、时间线小结

| 阶段 | 耗时占比 | 真正的转折点 |
|---|---|---|
| 对比三份 EFI、重写配置 | 20% | 发现 `DisableIoMapper` 被改坏 |
| 让日志能取出来 | 25% | `Target=65`（只写文件） |
| 处理 `(dmg)` 加载崩溃 | 25% | `dmg2img` 还原成真实分区 |
| 换 U 盘 | 15% | 换介质 |
| 装系统本身 | 15% | 插电 + 选对重启条目 |

**如果重来一次**，正确顺序应该是：

1. 先做一份 `PickerMode=Builtin` + `Target=65` 的最小配置
2. 用 `dmg2img` 走真实分区，不碰 `(dmg)`
3. 准备两根 U 盘（一根备用）
4. 全程插电

这样大概一个下午能搞定，不用来回猜。
