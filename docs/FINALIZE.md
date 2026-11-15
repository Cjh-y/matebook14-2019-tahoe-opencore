# 收尾：去掉啰嗦模式 + 把引导从 U 盘搬进内置硬盘

两个目标：

1. 启动时不再滚一屏白字（去掉 `-v`）
2. 不插 U 盘也能开机

---

## 一、去掉啰嗦模式（已完成）

`config.plist` 里的 `boot-args` 已从

```
-v keepsyms=1 -wegnoegpu -igfxblr -igfxblt -igfxonln=1 revpatch=sbvmm -no_compat_check
```

改成

```
keepsyms=1 -wegnoegpu -igfxblr -igfxblt -igfxonln=1 revpatch=sbvmm -no_compat_check
```

只去掉了 `-v`。其余参数都保留：`-wegnoegpu`（关独显）、`-igfxblr/-igfxblt`（背光）、`-igfxonln=1`（强制点亮）、`revpatch=sbvmm`（机型伪装）、`-no_compat_check`（跳过机型检查）、`keepsyms=1`（panic 时保留符号，不影响正常启动）。

改完之后启动就是 **Apple logo + 进度条**，不再滚字。

> 排障时怎么办：`EFI/OC/` 里留了一份 `config-verbose.plist`，把它改名成 `config.plist` 就恢复啰嗦模式。**别删它，这是出事时的救命绳。**

---

## 二、把 EFI 搬进内置硬盘的 ESP

### 1. 找到 ESP 并分配盘符

以**管理员身份**打开 CMD：

```
diskpart
list disk
select disk 0
list partition
```

找到那个 **约 100~550 MB、类型为「系统」的 EFI 分区**（通常就是分区 1），记下编号，然后：

```
select partition 1
assign letter=S
exit
```

资源管理器里就会出现 `S:` 盘，里面是 `EFI` 文件夹。

### 2. 备份 + 复制

先备份：

```
ren S:\EFI\BOOT\BOOTX64.EFI BOOTX64.win.efi
```

> 为什么要备份：Windows 会在 `\EFI\BOOT\` 放一份自己的 `BOOTX64.EFI`（仅在找不到 Boot#### 项时才用）。我们用它作为 OpenCore 的落点，但先留个后路。

然后从 U 盘复制：

- 整个 `EFI\OC` 文件夹 → `S:\EFI\OC`
- U 盘的 `EFI\BOOT\BOOTX64.EFI` → 覆盖 `S:\EFI\BOOT\BOOTX64.EFI`

> Windows 自己的启动文件在 `\EFI\Microsoft\Boot\bootmgfw.efi`，不受影响。

复制完检查一下目录，应该是：

```
S:\EFI\
├── BOOT\BOOTX64.EFI          ← OpenCore 的
├── Microsoft\...             ← Windows 的，别动
└── OC\
    ├── OpenCore.efi
    ├── ACPI\  Drivers\  Kexts\  Resources\
    └── config.plist
```

空间占用约 **11 MB**，ESP 里通常够。

### 3. 添加固件启动项

推荐用 **BOOTICE**（免费，界面简单）：

1. 右键以管理员运行
2. 切到 **UEFI** 标签页 → **修改启动序列**
3. 点 **添加** → 分区选 `S:`（EFI 分区）→ 文件选 `\EFI\OC\OpenCore.efi` → 菜单标题填 `OpenCore`
4. 用 **上移** 把它排到第一位
5. 点 **保存当前启动序列**

或者用命令行（`bcdedit`，效果相同）：

```cmd
bcdedit /copy {bootmgr} /d "OpenCore"
:: 复制返回的 GUID，替换下面的 {GUID}
bcdedit /set {GUID} path \EFI\OC\OpenCore.efi
bcdedit /set {fwbootmgr} displayorder {GUID} /addfirst
```

### 4. 验证

1. 拔掉 U 盘，重启
2. 应该出现 **OpenCore 文本菜单**（Windows / macOS 两个条目）
3. 选 macOS → Apple logo → 进桌面
4. 再重启一次，选 Windows → 确认 Windows 也能正常启动

两条都能进，就算收官。

---

## 三、收尾检查清单

- [ ] 不插 U 盘能进 macOS
- [ ] OC 菜单里能进 Windows
- [ ] 启动无白字（Apple logo + 进度条）
- [ ] U 盘的 EFI **不要删**，留着当救援盘
- [ ] `config-verbose.plist` 保留在 ESP 里

## 四、出问题了怎么办

| 情况 | 处理 |
|---|---|
| 开机直接进 Windows，没有 OC 菜单 | 启动项没加成功，用 BOOTICE 重新添加并置顶 |
| OC 菜单出现但选 macOS 报错 | 插回 U 盘启动，进 macOS 后把 `config-verbose.plist` 改名 `config.plist`，重启看日志 |
| OC 菜单里没有 macOS 条目 | 检查 `S:\EFI\OC\config.plist` 是否完整，或 U 盘那份直接覆盖 |
| 想彻底回到 Windows 原生引导 | 把 `S:\EFI\BOOT\BOOTX64.win.efi` 改名回 `BOOTX64.EFI`，再用 BOOTICE 把 Windows Boot Manager 置顶 |

> 最坏情况也死不了：U 盘还是可引导的救援盘，随时能进系统改配置。

## 五、以后更新 EFI

改配置只需要覆盖 `S:\EFI\OC\config.plist`，不用重做整个分区。建议本地保留一份 `C:\bs\EFI` 作为母本。
