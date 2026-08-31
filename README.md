# ASUS ROG STRIX Z490-F GAMING + RX 6900 XT — macOS Tahoe 26 Hackintosh

**English** · A fully working OpenCore 1.0.7 EFI and a field-tested installation guide for macOS Tahoe 26.6.2 on the ASUS ROG STRIX Z490-F GAMING with i9-10900K and Radeon RX 6900 XT. Features a dual-config architecture (installer-safe minimal config + full daily config) and a complete troubleshooting chronicle covering every failure mode hit during a 5-day bring-up.

**中文** · 本仓库提供一套完整可用的 OpenCore 1.0.7 EFI 和经过实战验证的安装指南，用于在 ASUS ROG STRIX Z490-F GAMING（i9-10900K + RX 6900 XT）上安装 macOS Tahoe 26.6.2。核心设计是"双 config 架构"（安装期最小化配置 + 日常全量配置），并附完整的 5 天调试实录——11 个坑的逐一定位与解法。

---

## Hardware / 硬件配置

| Component / 部件 | Model / 型号 | Compatibility / 兼容性 |
|---|---|---|
| Motherboard / 主板 | ASUS ROG STRIX Z490-F GAMING (BIOS 2094.80.5.0.0) | ✅ |
| CPU | Intel Core i9-10900K (Comet Lake, 10-core) | ✅ Native / 原生支持 |
| GPU / 显卡 | AMD Radeon RX 6900 XT 16GB (Navi 21, `0x73BF`) | ✅ Native / 原生支持 (MacPro7,1 ships W6900X — same silicon) |
| RAM / 内存 | 64GB DDR4 3600 | ✅ |
| Ethernet / 有线网卡 | Intel I225-V (onboard) | ✅ Injected AppleIntelI210Ethernet 2.3.1 + device-id spoof `15F3→15F2` + boot-arg `e1000=0` |
| WiFi / BT / 无线蓝牙 | Broadcom BCM94360 family (upgraded) | ⚠️ Needs OCLP-Plus root patch |
| Display / 显示器 | 6K (6144×3456) | ✅ |
| Storage / 存储 | PM1735 6.4TB (Sequoia) + Intel DC P3600 800GB U.2/PCIe (Tahoe target) | ✅ |

## Status / 系统状态

| Feature / 功能 | Status | Notes / 说明 |
|---|---|---|
| macOS Tahoe 26.6.2 | ✅ Running | SMBIOS MacPro7,1 (natively supported) |
| GPU acceleration / 显卡加速 | ✅ | Native Navi 21 + WhateverGreen 1.7.0 (daily config) |
| Ethernet / 有线网络 | ✅ | Works during install too (AEA personalization requires it) |
| Audio / 音频 | ✅ | AppleALC 1.9.7, layout-id 1 |
| USB | ✅ | Custom USBMap, Tahoe dual-format keys |
| Sensors / 传感器 | ✅ | VirtualSMC suite + SMCRadeonSensors |
| WiFi / BT | 🔧 Root patch | OCLP-Plus 3.2.2 Post-Install Root Patch |
| Sleep / 睡眠 | ❓ Untested / 未测试 | |

## Repository Layout / 仓库内容

```
├── EFI/                              ← Complete bootable EFI (OpenCore 1.0.7)
│   ├── BOOT/BOOTx64.efi
│   └── OC/
│       ├── config.plist              ← INSTALL config (minimal — field-tested through full install)
│       ├── config-postinstall.plist  ← DAILY config (swap in after install)
│       ├── ACPI/                     ← 4 SSDTs (PLUG / AWAC / EC-USBX / RHUB)
│       ├── Kexts/                    ← 15 kexts
│       ├── Drivers/ Tools/ Resources/
│       └── OpenCore.efi
├── INSTALL.md                        ← Clean step-by-step guide / 干净安装步骤
└── INSTALL-LOG.md                    ← Full troubleshooting chronicle / 完整踩坑实录
```

## ⚠️ REQUIRED: Generate Your Own Serials / 使用前必改：生成你自己的序列号

Both configs ship with **placeholder SMBIOS serials** — Apple services will reject them as-is. / 仓库中两个 config 的 SMBIOS 序列号是**占位符**，不换无法通过 Apple 服务验证。

```bash
git clone https://github.com/corpnewt/GenSMBIOS && cd GenSMBIOS && chmod +x *.command
./GenSMBIOS.command   # menu: 1 (install deps) → model: MacPro7,1
# Fill SystemSerialNumber / SystemUUID / MLB into BOTH files
# (PlatformInfo → Generic in config.plist AND config-postinstall.plist).
# ROM = any 6-byte MAC (e.g. your ethernet MAC address).
```

## Quick Start / 快速开始

→ **[INSTALL.md — Clean install guide / 干净安装步骤](INSTALL.md)**

→ **[INSTALL-LOG.md — Full chronicle / 完整踩坑实录](INSTALL-LOG.md)** — check here first when troubleshooting / 遇到问题先查这里

## The Dual-Config Architecture / 双 config 架构（核心设计）

**EN:** Why not one config for everything? Tahoe's installer environment (`BaseSystemKernelExtensions.kc`) behaves differently from the installed system (PrelinkedKernel KC). Two components poison installs specifically:

1. **WhateverGreen freezes the Tahoe installer** when hooking AMDSupport ([WhateverGreen PR #124](https://github.com/acidanthera/WhateverGreen/pull/124)) — yet works perfectly on the installed system.
2. **`-amfipassbeta`** (AMFIPass 1.4.1, a 2024-era kext) corrupts boot on Sequoia 15.7.9 / Tahoe 26.6.2 — yet is required for the OCLP root-patch stage.

So: install with the minimal config (5 kexts), swap to the full config (15 kexts) after install.

**中文：** 为什么不能一个 config 走天下？Tahoe 的安装环境与正式系统使用不同的内核缓存，有两个组件专门毒害安装环境：

1. **WhateverGreen 在 Tahoe 安装器中 hook AMDSupport 时死机**——但在装好的系统上完全正常；
2. **`-amfipassbeta`**（AMFIPass 1.4.1，2024 年的 kext）会破坏 Sequoia 15.7.9 / Tahoe 26.6.2 的引导——但 OCLP root patch 阶段又需要它。

因此：安装期用最小 config（5 个 kext），装完换全量 config（15 个 kext + 完整 boot-args）。

| | Installer / 安装器 | Daily / 日常 |
|---|---|---|
| File / 文件 | `config.plist` | `config-postinstall.plist` → overwrite `config.plist` after install / 装完覆盖 |
| Kexts ON / 启用 | 5 (Lilu, VirtualSMC, USBMap, AppleIntelI210Ethernet, RestrictEvents) | 15 (all / 全部) |
| boot-args | `-v keepsyms=1 debug=0x100 revpatch=sbvmm e1000=0` | `+ agdpmod=pikera alcid=1 ipc_control_port_options=0 -amfipassbeta` |
| WhateverGreen | ❌ | ✅ |

## Software Versions / 软件版本清单

| Component / 组件 | Version / 版本 | Notes / 备注 |
|---|---|---|
| OpenCore | 1.0.7 (2026-03-20) | — |
| Lilu | 1.7.2 | macOS 26 AMDSupport fix included / 含修复 |
| VirtualSMC + SMCProcessor + SMCSuperIO | 1.3.7 | — |
| WhateverGreen | 1.7.0 | Daily config only / 仅日常 config |
| AppleALC | 1.9.7 | — |
| NVMeFix | 1.1.3 | — |
| BlueToolFixup | 2.7.2 | Tahoe bluetoothd patches / 含 Tahoe 补丁 |
| AppleIntelI210Ethernet | 2.3.1 | spoof + `e1000=0` — all three required / 三件缺一不可 |
| SMCRadeonSensors | 2.4.0 | Replaces RadeonSensor+SMCRadeonGPU / 取代旧双 kext |
| RestrictEvents | 1.1.6 | **Critical for Tahoe install** / **安装关键** — `revpatch=sbvmm` |
| AMFIPass | 1.4.1 | Final release (upstream repo deleted) / 绝版 |
| IO80211FamilyLegacy + AirPortBrcmNIC + IOSkywalkFamily | OCLP payload originals | WiFi injection stack / WiFi 注入栈 |
| USBMap | Custom / 定制 | Dual-format keys (Sequoia + Tahoe) / 双格式键名 |

## BIOS Settings / BIOS 设置

Mostly defaults; verify these / 基本默认即可，确认以下几项：

| Setting / 设置项 | Value / 值 |
|---|---|
| Boot mode / 启动模式 | UEFI only, CSM off / 关闭 |
| Secure Boot | Disabled / Other OS |
| VT-d | Any / 任意 (`DisableIoMapper=true` in config) |
| CFG Lock | No unlock needed / 无需解锁 (`AppleXcpmCfgLock=true` in config) |

## Credits / 致谢

- [Acidanthera](https://github.com/acidanthera) — OpenCore, Lilu, WhateverGreen, AppleALC, VirtualSMC, RestrictEvents, NVMeFix, BrcmPatchRAM
- [Dortania](https://dortania.github.io/) — OpenCore guides & Tahoe notes
- [OCLP-Plus (YBronst)](https://github.com/YBronst/OCLP-Plus) — Tahoe WiFi root patch (archived; 3.2.2 still works)
- [laobamac/OCLP-Mod](https://github.com/laobamac/OCLP-Mod) — alternative root patcher / 备选
- [ChefKissInc/SMCRadeonSensors](https://github.com/ChefKissInc/SMCRadeonSensors) — AMD GPU sensors
- [corpnewt](https://github.com/corpnewt) — USBMap, GenSMBIOS

## License

Bundled Acidanthera kexts follow their original licenses. Documentation: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
