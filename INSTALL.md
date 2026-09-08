# Installation Guide / 安装步骤（干净版）

> **EN** · This is the complete zero-to-working path, executed in order. All commands are field-tested. When something goes wrong, consult [INSTALL-LOG.md](INSTALL-LOG.md).
> **中文** · 本指南是从零到可用系统的完整路径，按顺序执行即可。所有命令均经实测验证。遇到问题时，去 [INSTALL-LOG.md](INSTALL-LOG.md) 查踩坑实录。

---

## Prerequisites / 前提条件

| Need / 需要 | Details / 说明 |
|---|---|
| This hardware (or very similar) / 本硬件（或高度相似） | Z490 chipset + Comet Lake CPU + Navi 21 GPU (RX 6800/6900 series) |
| A working macOS environment / 一个能工作的 macOS | For preparing the target disk (an existing install on another disk, or another Mac) |
| `Install macOS Tahoe.app` | Via App Store or `softwareupdate --fetch-full-installer`; place in `/Applications` |
| **An ethernet cable / 一根网线** | **MANDATORY** — Tahoe's install phase 2 requires internet for AEA personalization; it will deadlock without network / **必须**——第二阶段需要联网做 AEA 个性化，无网必卡死 |
| USB keyboard, direct-connected / USB 键盘（直插主板后置口） | Keyboards/mice behind hubs randomly die in the installer environment; mouse optional — keyboard alone can complete everything / 安装期键鼠经 Hub 会随机失联；鼠标可不接 |
| Target disk / 目标盘 | A dedicated NVMe (this guide: 800GB Intel P3600); coexists fine with your existing system disk / 独立一块 NVMe；与现有系统盘共存互不干扰 |

## Step 0: BIOS Settings / 第 0 步：BIOS 设置

Enter BIOS (DEL at boot) and verify / 进 BIOS（开机按 DEL）确认：

- Boot mode: UEFI only, CSM disabled / 启动模式：UEFI only，CSM 关闭
- Secure Boot: Disabled (or Other OS)
- Everything else: defaults work / 其余默认即可 (config carries `DisableIoMapper=true` + `AppleXcpmCfgLock=true` as safety nets)

## Step 1: Partition the Target Disk / 第 1 步：目标盘分区

From your working macOS, identify the target disk / 在现有 macOS 中确认目标盘编号：

```bash
diskutil list
# Find your target disk — assume disk2 (⚠️ verify capacity & model — wrong disk = data loss)
# 找到你的目标盘，假设是 disk2（⚠️ 认准容量和型号，别选错盘）
```

One command builds the 3-partition layout (auto EFI 200MB + 30GB installer + rest APFS) / 一条命令建好三分区：

```bash
diskutil partitionDisk disk2 GPT JHFS+ "Install macOS Tahoe" 30G APFS "macOS Tahoe" R
```

Result / 分区结果: `disk2s1` = EFI (200MB) · `disk2s2` = installer volume (JHFS+ 30GB) · `disk2s3` = APFS container (rest, containing volume "macOS Tahoe")

## Step 2: Create the Installer / 第 2 步：制作安装器

```bash
sudo /Applications/Install\ macOS\ Tahoe.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/Install\ macOS\ Tahoe --nointeraction
```

Wait ~15-20 minutes. **Always use createinstallmedia — hand-copied installer volumes do not boot. / 必须用 createinstallmedia，手工拷贝的安装器无法引导。**

## Step 3: Deploy EFI + Serials / 第 3 步：部署 EFI + 序列号

Mount the target ESP and copy this repo's EFI / 挂载目标盘的 ESP 并拷入本仓库的 EFI：

```bash
sudo diskutil mount disk2s1   # if mount fails: sudo diskutil repairVolume disk2s1, retry
cp -R /path/to/this-repo/EFI/ /Volumes/EFI/
```

Generate your serials (see README) and edit **both** configs / 生成序列号（见 README）并改**两个** config：

```bash
sudo nano /Volumes/EFI/EFI/OC/config.plist              # PlatformInfo → Generic
sudo nano /Volumes/EFI/EFI/OC/config-postinstall.plist  # same values / 保持一致
```

The ESP's `config.plist` is already the installer profile (minimal 5 kexts) — no changes needed beyond serials. / 此时 config.plist 就是安装器模式，除序列号外无需改动。

## Step 4: Install / 第 4 步：安装

1. **Plug in ethernet now and keep it plugged until the system is fully installed / 从现在起一直插着网线，直到系统装完**;
2. Keyboard directly into a rear motherboard USB port (**no hubs, no front panel / 不要经 Hub，不要用前面板**);
3. Reboot → press **F8** (ASUS boot menu) → select the target disk's UEFI entry (e.g. `UEFI: INTEL SSDPE2ME800G4`);
4. OpenCore picker → select **Install macOS Tahoe**;
5. In the installer GUI (sluggish UI is normal — installer config loads no GPU acceleration / 界面偏慢属正常):
   - Disk Utility → Show All Devices → select **macOS Tahoe** volume (inside the target APFS container) → Erase (APFS) → quit Disk Utility;
   - Select **Install macOS Tahoe** → target = **macOS Tahoe** → begin.

### The Three Install Phases / 安装的三个阶段

| Phase / 阶段 | What you see / 表现 | Duration / 耗时 |
|---|---|---|
| 1 — GUI file copy / 文件复制 | Apple progress bar | ~10-20 min, auto-reboots when done |
| 2 — AEA personalize + system install / 个性化 + 系统安装 | White Apple logo + progress bar + time estimate | ~20-30 min, **internet REQUIRED**, 1-2 auto-reboots |
| 3 — Setup Assistant / 设置助手 | Country/language/account selection | manual / 你操作 |

> ⚠️ **Phase 2 is the critical part / 第二阶段是全程关键**：seeing "About N minutes remaining" means RestrictEvents is working and AEA personalization passed. Frozen at 5% → check RestrictEvents enabled + `revpatch=sbvmm` in boot-args. Frozen at "N minutes remaining" → check the ethernet cable. (Details: INSTALL-LOG.md pits 10/11.)
> **看到"About N minutes remaining"说明 RestrictEvents 正常、AEA 个性化已通过。卡 5% → 查 RestrictEvents/boot-args；卡"剩余 N 分钟"→ 查网线。**

### After every reboot / 每次重启后

Press F8 → select the target disk's UEFI entry. In the OC picker:
- If **macOS Tahoe** (install continuation / 安装续装条目) appears → select it to continue;
- After install fully completes → **macOS Tahoe** is now the real system / 就是正式系统了。

## Step 5: Swap to the Daily Config / 第 5 步：装完换日常 config

At the desktop (no sound, no WiFi, no GPU acceleration — all expected, still on installer config / 进入桌面后无声、无 WiFi、显卡无加速——都是预期)：

```bash
sudo diskutil mount disk2s1
sudo cp /Volumes/EFI/EFI/OC/config.plist /Volumes/EFI/EFI/OC/config-installer-backup.plist
sudo cp /Volumes/EFI/EFI/OC/config-postinstall.plist /Volumes/EFI/EFI/OC/config.plist
```

Reboot → GPU acceleration, ethernet, Bluetooth (native), full USB live. Onboard audio stays off by default in this build (AirPods/DP-monitor audio work; re-enable AppleALC + `alcid=` if you need rear jacks). / 重启后显卡加速、有线网卡、蓝牙（原生）、USB 全速恢复。本构建默认不开板载音频（AirPods/DP 音频可用；需要背板口就启用 AppleALC + `alcid=`）。

> **Expect 1–3 automatic reboots on this first daily-config boot** — first-boot housekeeping (cryptex/Preboot maintenance), benign and self-resolving. Observed twice in the clean-install validation run: system rebooted itself mid-verbose twice, reached desktop on the third attempt. / **首次以日常 config 启动会自动重启 1–3 次**——首启维护（cryptex/Preboot），良性自愈。干净安装验证中实测两次。

## Step 6: WiFi + AirDrop via the Hybrid Formula / 第 6 步：WiFi + AirDrop（混合配方）

The daily config already EFI-injects the legacy kernel WiFi stack (IOSkywalkFamily 1.0 + IO80211FamilyLegacy + the **AirPortBrcmNIC plugin entry**, native Skywalk blocked) alongside AMFIPass under SIP `0xFFFF`. The only missing half is the userspace — the patched frameworks: / 日常 config 已 EFI 注入旧内核 WiFi 栈（含 AirPortBrcmNIC 插件条目、屏蔽原生 Skywalk）+ AMFIPass + SIP `0xFFFF`。只缺用户态那半——补丁框架：

1. Get [OCLP-Mod](https://github.com/laobamac/OCLP-Mod) — ethernet already works (or copy the app over from another install). / 拿到 OCLP-Mod——有线网此时可用（或从另一系统拷贝 app）。
2. Run it → **Post-Install Root Patch** → reboot when prompted. / 运行 → Post-Install Root Patch → 按提示重启。
3. WiFi toggle now works — join your network (hidden network: Other → type SSID). / WiFi 开关恢复——加入网络（隐藏网络：其他→输入 SSID）。

Verify / 验证:

```bash
sudo kmutil inspect | grep -icE "skywalk|80211|brcm|amfipass"   # expect >= 5
```

AirDrop (send + receive), DRM (Chrome/Netflix), and Bluetooth (AirPods — the Apple-firmware card is native, no BT kexts) should all be live. For the full formula rationale and the alternative no-AirDrop BCMC route, see README's "Winning Formula" section. / AirDrop 双向、DRM、蓝牙（苹果固件卡原生免驱）全部就绪。完整配方原理与 BCMC 备选路线见 README"制胜配方"节。

### If AirDrop receive ever goes silent / 若日后 AirDrop 接收静默失效

First prescription: **Apple ID sign-out → sign-in → REBOOT → wait 10 min** — wedged IDS session state, most often collateral damage from a crash-loop; NOT a config problem (INSTALL-LOG pit 17). / 第一处方：**重登 Apple ID → 重启 → 等 10 分钟**——IDS 会话淤塞，多为崩溃循环附带损伤，不是 config 问题（坑 17）。

## Step 7: Verification Checklist / 第 7 步：验证清单

| Item / 项目 | How / 验证方法 |
|---|---|
| GPU acceleration / 显卡加速 | About This Mac → Graphics shows RX 6900 XT 16GB |
| Ethernet / 有线网络 | System Settings → Network → en0 has IP |
| Audio / 音频 | AirPods (BT) or DP monitor output — onboard codec off by default / AirPods 或 DP 显示器输出——板载默认关闭 |
| WiFi | Toggle works, connects at full speed / 开关可用，满速连接 |
| AirDrop | Send (drag to iPhone) AND receive (iPhone share sheet lists this Mac) / 发送 + iPhone 分享列表能看到本机 |
| DRM | Chrome → Netflix plays without protected-content errors / Chrome 放 Netflix 无报错 |
| Bluetooth / 蓝牙 | AirPods connect and play / AirPods 可连可放 |
| USB | Plug a USB drive, normal speed |
| Sensors / 传感器 | Install [Stats](https://github.com/exelban/stats) — CPU/GPU temps |

## Maintenance / 日常维护

- **Before macOS point updates / 小版本更新前**: revert root patches in OCLP-Mod (Revert Root Patches) → update → re-patch;
- **OC / kext updates**: mount ESP (`sudo diskutil mount`), replace files; always run [ocvalidate](https://github.com/acidanthera/OpenCorePkg/releases) on modified configs;
- **Switching OSes / 双系统切换**: no NVRAM reset needed — the config's Delete+Add pins boot-args/SIP per boot. / 无需清 NVRAM——config 的 Delete+Add 每次启动固化生效；
- **AirDrop receive goes silent / AirDrop 接收失效**: Apple ID re-login + reboot (pit 17) before touching anything else. / 先重登 Apple ID + 重启（坑 17），别急着动别的；
- WhateverGreen is enabled in the daily config — **never enable it in the installer environment** (INSTALL-LOG.md pit 2). / WEG 在日常 config 中启用——**绝不要在安装器环境开启它**。
