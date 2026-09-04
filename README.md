# ASUS ROG STRIX Z490-F GAMING + RX 6900 XT — macOS Tahoe 26 Hackintosh

**English** · A fully working OpenCore 1.0.7 EFI and a field-tested installation guide for macOS Tahoe 26.6.2 on the ASUS ROG STRIX Z490-F GAMING with i9-10900K and Radeon RX 6900 XT. **WiFi + AirDrop both fully working** via a hybrid formula (EFI-injected legacy WiFi stack + OCLP root-patched frameworks + AMFIPass under SIP `0xFFFF`), after every off-the-shelf root patcher failed on 26.6.2. Features a dual-config architecture (installer-safe minimal config + full daily config) and a complete troubleshooting chronicle covering every failure mode hit during bring-up.

**中文** · 本仓库提供一套完整可用的 OpenCore 1.0.7 EFI 和经过实战验证的安装指南，用于在 ASUS ROG STRIX Z490-F GAMING（i9-10900K + RX 6900 XT）上安装 macOS Tahoe 26.6.2。**WiFi 与 AirDrop 双全通**——在 26.6.2 上所有现成 root patch 工具全部失效后，用"EFI 注入旧驱动栈 + OCLP 框架补丁 + SIP `0xFFFF` 下的 AMFIPass"混合配方达成。核心设计是"双 config 架构"（安装期最小化配置 + 日常全量配置），并附完整踩坑实录。

---

## Hardware / 硬件配置

| Component / 部件 | Model / 型号 | Compatibility / 兼容性 |
|---|---|---|
| Motherboard / 主板 | ASUS ROG STRIX Z490-F GAMING (BIOS 2094.80.5.0.0) | ✅ |
| CPU | Intel Core i9-10900K (Comet Lake, 10-core) | ✅ Native / 原生支持 |
| GPU / 显卡 | AMD Radeon RX 6900 XT 16GB (Navi 21, `0x73BF`) | ✅ Native / 原生支持 (MacPro7,1 ships W6900X — same silicon) |
| RAM / 内存 | 64GB DDR4 3600 | ✅ |
| Ethernet / 有线网卡 | Intel I225-V (onboard) | ✅ Injected AppleIntelI210Ethernet 2.3.1 + device-id spoof `15F3→15F2` + boot-arg `e1000=0` |
| WiFi / BT / 无线蓝牙 | Broadcom BCM94360 family (upgraded) | ✅ **WiFi + AirDrop** — EFI-injected legacy stack + OCLP root-patched frameworks + AMFIPass under SIP `0xFFFF` / EFI 注入旧驱动栈 + 框架补丁混合路线 |
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
| WiFi / BT | ✅ **WiFi + AirDrop** | EFI injection (Skywalk 1.0 + IO80211FamilyLegacy + AirPortBrcmNIC, native Skywalk blocked) + OCLP-Mod framework patches + AMFIPass + SIP `0xFFFF` + `ipc_control_port_options=0` — see INSTALL-LOG pit 14 / EFI 注入 + 框架补丁混合路线，见实录坑 14 |
| AirDrop | ✅ **Working** | Requires the WiFi route above **plus** BlueToolFixup + BlueWakeFixup (Bluetooth discovery) / 需上述路线 + 蓝牙双修复 |
| Sleep / 睡眠 | ❓ Untested / 未测试 | |

## Repository Layout / 仓库内容

```
├── EFI/                              ← Complete bootable EFI (OpenCore 1.0.7)
│   ├── BOOT/BOOTx64.efi
│   └── OC/
│       ├── config.plist              ← INSTALL config (minimal — field-tested through full install)
│       ├── config-postinstall.plist  ← DAILY config (swap in after install)
│       ├── ACPI/                     ← 4 SSDTs (PLUG / AWAC / EC-USBX / RHUB)
│       ├── Kexts/                    ← 16 kexts
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
| Kexts ON / 启用 | 5 (Lilu, VirtualSMC, USBMap, AppleIntelI210Ethernet, RestrictEvents) | 17 (all / 全部) |
| boot-args | `-v keepsyms=1 debug=0x100 revpatch=sbvmm e1000=0` | `+ agdpmod=pikera alcid=1 ipc_control_port_options=0 amfi=0x80 amfi_get_out_of_my_way=0x1 cs_enforcement_disable=1 -amfipassbeta` |
| SIP (`csr-active-config`) | `0x0000` (fully on / 全开) | `0xFFFF` (fully off — required by the WiFi+AirDrop route / 全关，WiFi+AirDrop 路线所需) |
| WhateverGreen | ❌ | ✅ |

## WiFi + AirDrop on Tahoe 26.6.2 — The Winning Formula / 制胜配方

**EN:** BCM94360-class cards are dead on Tahoe 26.6.2: every root patcher (OCLP-Plus, OCLP-Mod, WiFi Patcher Pro) fails because their shared enabler AMFIPass 1.4.1 loses a race against AMFI on every boot (pit 13), `amfi=*` boot-args no longer bypass library validation, and `kmutil` refuses to admit unsigned kexts into the kernel collection at all. The formula that finally works — **WiFi and AirDrop both fully functional, one of the very few confirmed AirDrop-working Tahoe 26.6.2 hackintosh builds**:

| Layer / 层 | What / 内容 |
|---|---|
| Kernel / 内核 | EFI-inject `IOSkywalkFamily` (v1.0) + `IO80211FamilyLegacy` + its **`AirPortBrcmNIC` plugin (must be listed as its own `Kernel→Add` entry — OC does NOT auto-load plugins inside a parent kext!)**, and **Block** native `com.apple.iokit.IOSkywalkFamily` (MinKernel 23.0.0) |
| Userspace / 用户态 | OCLP-Mod root-patched frameworks (IO80211Old.dylib, WiFiPeerToPeerOld.dylib, LibSystemShim) stay system-side |
| AMFI | `AMFIPass.kext` enabled + `-amfipassbeta` + `amfi=0x80 amfi_get_out_of_my_way=0x1 cs_enforcement_disable=1`, under **SIP `csr-active-config=0xFFFF`** — with SIP fully off the patched frameworks pass validation even when AMFIPass loses its race, which kills the boot lottery |
| Daemon compat / 守护进程 | `ipc_control_port_options=0` — patched daemons crash without it on Sonoma+ |
| AirDrop | `BlueToolFixup` + `BlueWakeFixup` (Bluetooth discovery/handoff) |

Why EFI injection + system-side patches don't double-load here (refining pit 12): on 26.6.2 `kmutil` never admits the patcher's unsigned kexts into the KC, so the system-side copies never load — EFI injection is the *only* kernel-side source. / 26.6.2 的 kmutil 根本不会把补丁工具装进系统侧的未签名 kext 收进 KC，系统侧副本永远不加载——EFI 注入是内核侧唯一来源，所以不冲突（坑 12 结论的修正版）。

**Alternative / 备选路线 (no AirDrop):** [AppleBCMWLANCompanion](https://github.com/0xFireWolf/AppleBCMWLANCompanion) drives the card through Apple's *native* stack — full-speed WiFi, **SIP fully on, no patches, no AMFIPass** — but no AirDrop. Kext stays bundled; disable the 4 EFI WiFi entries + AMFIPass, enable BCMC, set SIP `0x0000`, add `wlan.pcie.detectsabotage=0`, and skip the OCLP-Mod patch step. / BCMC 路线：满速 WiFi、SIP 全开、零补丁，但无 AirDrop。kext 仍在仓库中。

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
| AMFIPass | 1.4.1 | Final release (upstream repo deleted). Under SIP `0xFFFF` it loads reliably and its race (pit 13) stops being fatal / SIP 全关下稳定加载，竞态不再致命 |
| IO80211FamilyLegacy + AirPortBrcmNIC + IOSkywalkFamily | OCLP payload originals | EFI-injected kernel WiFi stack / EFI 注入的内核 WiFi 栈 |
| BlueToolFixup | 2.7.2 | Tahoe bluetoothd patches / 含 Tahoe 补丁 — **required for AirDrop / AirDrop 必需** |
| BlueWakeFixup | 2.7.2 | BT wake robustness / 蓝牙唤醒修复 — **required for AirDrop / AirDrop 必需** |
| USBMap | Custom / 定制 | Dual-format keys (Sequoia + Tahoe) / 双格式键名 |
| AppleBCMWLANCompanion | 1.1.0 | **Alternative route** (native driver, WiFi-only, no AirDrop) — bundled but unused by the daily config / **备选路线**（原生驱动、仅 WiFi、无 AirDrop）——已捆绑但日常配置未启用 |

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
- [0xFireWolf/AppleBCMWLANCompanion](https://github.com/0xFireWolf/AppleBCMWLANCompanion) — the alternative native-driver WiFi route (WiFi-only, SIP-on) / 备选原生 WiFi 路线
- [OCLP-Plus (YBronst)](https://github.com/YBronst/OCLP-Plus) / [OCLP-Mod (laobamac)](https://github.com/laobamac/OCLP-Mod) — Tahoe root patchers; OCLP-Mod's framework patches are one half of the final AirDrop formula / 其框架补丁是最终 AirDrop 配方的一半（另一半是 EFI 注入）
- [ChefKissInc/SMCRadeonSensors](https://github.com/ChefKissInc/SMCRadeonSensors) — AMD GPU sensors
- [corpnewt](https://github.com/corpnewt) — USBMap, GenSMBIOS

## License

Bundled Acidanthera kexts follow their original licenses. Documentation: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
