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

Reboot → GPU acceleration, onboard audio, ethernet, Bluetooth, full USB should all be live. / 重启后显卡加速、声卡、有线网卡、蓝牙、USB 全速恢复。

## Step 6: WiFi via BCMC (native driver, no root patch) / 第 6 步：WiFi 用 BCMC 原生驱动方案

The daily config already ships with **AppleBCMWLANCompanion** enabled — it makes Apple's *native* Tahoe Broadcom driver accept your legacy BCM94360-family card. No root patches, no AMFIPass, SIP stays fully on. / 日常 config 已内置 **AppleBCMWLANCompanion**——让 Apple 在 Tahoe 里的原生 Broadcom 驱动认你的老卡。不打 root patch、不需要 AMFIPass、SIP 全开。

What's left to do / 还需要做的：

1. Download the firmware for your chip / 下载对应固件:
   - BCM43602 (`14e4:43ba`): [`brcmfmac43602-pcie_7.35.177.61.bin`](https://github.com/0xFireWolf/AppleBCMWLANCompanion/raw/main/Firmwares/BCM43602/brcmfmac43602-pcie_7.35.177.61.bin) — sha256 `bf4cfc23ee952a3d82ef33a0f5f87853201c98f1bed034876a910f354f37862d`
   - BCM4350 (`14e4:43a3`): see the [BCMC repo Firmwares dir](https://github.com/0xFireWolf/AppleBCMWLANCompanion/tree/main/Firmwares)
2. Install it / 安装固件:
   ```bash
   sudo mkdir -p /usr/local/share/firmware/wifi/
   sudo cp brcmfmac43602-pcie_7.35.177.61.bin /usr/local/share/firmware/wifi/
   shasum -a 256 /usr/local/share/firmware/wifi/brcmfmac43602-pcie_7.35.177.61.bin  # must match the hash above / 必须匹配上面的哈希
   ```
3. Verify the config's DeviceProperties point at YOUR WiFi card's PCI path / 确认 config 里的 DeviceProperties 指向你的网卡 PCI 路径:
   - This repo targets `PciRoot(0x0)/Pci(0x1C,0x7)/Pci(0x0,0x0)` (RP08 slot on this board). On the same board it's correct; verify with Hackintool if unsure. / 本仓库指向 RP08 槽；同板即正确，不确定就用 Hackintool 核对。
4. Reboot → WiFi should come up natively. Verify / 重启后验证:
   ```bash
   kextstat | grep bcmc   # expect science.firewolf.bcmc
   sudo dmesg | grep bcmc # chip bring-up log
   ```

### Optional: full-speed WiFi (VT-d) / 可选：满速 WiFi（开 VT-d）

The shipped config uses the IOMapper path. For full speed (~500/250 Mbps vs ~130/130): enable **VT-d** in BIOS (Advanced → System Agent/CPU Configuration). The config already has `DisableIoMapper=false`, so it just works. If AppleVTD turns out broken on your board (symptom: WiFi driver won't load — system still boots fine), re-add device property `bcmc-disable-io-mapper` = `01000000` and disable VT-d again. / 出厂 config 走 IOMapper 路径。要满速就在 BIOS 开 VT-d（config 已配好）。若 AppleVTD 在你板子上不工作（症状：WiFi 驱动不加载，系统照常启动），加回 `bcmc-disable-io-mapper` 并关 VT-d 即可。

### Known BCMC limitations (Beta) / BCMC 已知限制（Beta 阶段）

- **No AWDL → no AirDrop/Continuity** (firmware doesn't support it). Use LocalSend as an alternative. / **无 AWDL → AirDrop/Continuity 不可用**（固件不支持）。可用 LocalSend 替代。
- Sleep/wake may panic (Beta). / 睡眠唤醒可能 panic（Beta 阶段）。
- Disable BCMC before OTA updates (`-bcmcoff` boot-arg), re-enable after. / OTA 更新前加 `-bcmcoff` 禁用，更新完再启用。
- Wrong Tx rate shown in WiFi menu (24 Mbps display bug). / WiFi 菜单显示速率不准（24 Mbps 显示 bug）。

## Step 7: Verification Checklist / 第 7 步：验证清单

| Item / 项目 | How / 验证方法 |
|---|---|
| GPU acceleration / 显卡加速 | About This Mac → Graphics shows RX 6900 XT 16GB |
| Ethernet / 有线网络 | System Settings → Network → en0 has IP |
| Audio / 音频 | System Settings → Sound → output device list |
| WiFi / BT | Menu bar icons appear and connect |
| USB | Plug a USB drive, normal speed |
| Sensors / 传感器 | Install [Stats](https://github.com/exelban/stats) — CPU/GPU temps |

## Maintenance / 日常维护

- **Before macOS point updates / 小版本更新前**: revert root patches in OCLP-Plus (Revert Root Patches) → update → re-patch;
- **OC / kext updates**: mount ESP (`sudo diskutil mount`), replace files; always run [ocvalidate](https://github.com/acidanthera/OpenCorePkg/releases) on modified configs;
- **NVRAM weirdness / NVRAM 异常** (boot anomalies, boot-args not applying): OC picker → ResetNvram;
- WhateverGreen is enabled in the daily config — **never enable it in the installer environment** (INSTALL-LOG.md pit 2). / WEG 在日常 config 中启用——**绝不要在安装器环境开启它**。
