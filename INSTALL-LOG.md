# Installation Chronicle / 安装实录（完整踩坑记录）

> **EN** · A complete, evidence-based record of a 5-day bring-up (2026-08-24 → 2026-08-28): every failure, its symptom, forensic evidence, root cause, fix, and lesson. Written for anyone debugging this platform — the clean install path in [INSTALL.md](INSTALL.md) already incorporates all these fixes.
>
> **中文** · 2026-08-24 至 08-28 五天安装调试的完整实录：每个故障的症状、取证、根因、解法与教训。[INSTALL.md](INSTALL.md) 的干净步骤已吸收全部修复，本文为排障者而写。

---

## Meta / 元信息

| | |
|---|---|
| Goal / 目标 | macOS Tahoe 26.6.2 on P3600 800GB NVMe, Sequoia system on PM1735 untouched / Sequoia 系统盘全程不动 |
| Outcome / 结果 | ✅ Success / 成功 |
| Effort / 工作量 | 5 days, 30+ boot attempts, 13 distinct root causes / 五天，30+ 次启动尝试，13 个独立根因 |
| Ending config / 最终配置 | Dual-config architecture (see README) / 双 config 架构 |

## Timeline Overview / 时间线总览

| Date / 日期 | Phase / 阶段 |
|---|---|
| Aug 24 | EFI audit & unified upgrade (OC 1.0.7) → pits 1-3 (empty ACPI / WEG freeze / amfipassbeta) |
| Aug 25 | USB hell (pit 4 a/b/c) → GPU block white rectangle (pit 5) → "damaged installer" (pit 6) → phantom volume (pit 7) |
| Aug 28 | Move installer to internal NVMe (pit 9 decision) → ESP mount chaos (pit 8) → phase-2 5% deadlock (pit 10) → AEA network freeze (pit 11) → **SUCCESS** |

---

## Pit 1: Empty ACPI Directory / 空 ACPI 目录

**Symptom / 症状**: After the first EFI rebuild, nothing boots — installer or existing system. OC log starts with 4 × `OC: Failed to find ACPI SSDT-*.aml`. / 重建 EFI 后全部无法引导，OC 日志开头 4 行 ACPI 找不到。

**Root cause / 根因**: A `a && b && c` shell chain broke midway (HfsPlus path typo); the subsequent SSDT copies were silently skipped. Directory existed — empty. / 命令链中途失败，后面的 SSDT 拷贝被静默跳过；目录存在但为空。

**Fix / 解法**: Restore the 4 SSDTs (PLUG-DRTNIA / AWAC / EC-USBX-DESKTOP / RHUB — stock Comet Lake desktop set).

**Lesson / 教训**: Verify copies by listing directory **contents**, never directory existence. / 验证拷贝要列目录**内容**，不是验证目录存在。

## Pit 2: WhateverGreen Freezes the Tahoe Installer / WEG 冻结 Tahoe 安装器

**Symptom / 症状**: Kernel-stage freeze at random points (sometimes at WiFi driver log lines, sometimes at sensor lines). No panic, no panic file, Caps Lock dead. Sequoia boots fine with the same EFI. / 内核阶段随机冻结，无 panic 无日志，Caps Lock 死。Sequoia 正常。

**Root cause / 根因**: Tahoe's installer/recovery uses `BaseSystemKernelExtensions.kc`; WEG freezes the instant it hooks AMDSupport under that KC ([WhateverGreen PR #124](https://github.com/acidanthera/WhateverGreen/pull/124) — reporter also had a Z-board + 6900 XT). The upstream root cause is a Lilu address-slot overrun ([Lilu PR #102](https://github.com/acidanthera/Lilu/pull/102), fixed in Lilu 1.7.2) — but WEG 1.7.0 still froze for us even on Lilu 1.7.2.

**Fix / 解法**: Disable WEG entirely for installs. MacPro7,1 + Navi 21 renders the installer GUI natively (real Mac Pro 2019 shipped W6800X/W6900X — same silicon). The AGDP board-id→board-ix kernel patch is optional (it logs "Not Found" on 26.6.2 = harmless no-op). Installed systems use PrelinkedKernel KC where WEG works fine. / 安装期彻底禁用 WEG；正式系统正常使用。

## Pit 3: `-amfipassbeta` Corrupts Boot / amfipassbeta 引导破坏

**Symptom / 症状**: Adding the boot-arg breaks boot on BOTH Sequoia 15.7.9 and Tahoe 26.6.2. / 加入后双系统都无法引导。

**Root cause / 根因**: AMFIPass 1.4.1 is a 2024-era kext (upstream repo deleted). It has no symbol offsets for these kernels; the beta flag force-patches stale offsets = memory corruption. / 不认识新内核，强制打旧偏移=内存破坏。

**Fix / 解法**: Remove during install; enable only for the OCLP root-patch stage (post-install). / 安装期移除；仅 OCLP root patch 阶段启用。

## Pit 4: The USB Triple Pit / USB 三连坑（最大的坑群）

### 4a — XhciPortLimit patch corrupts under load / 补丁高压崩溃
- Symptom: GUI works, then hard freeze at ~20% file copy (sustained 18GB read from USB stick). / 能进 GUI，持续大流量读盘时全系统冻结。
- Root cause: the quirk kernel-patches Tahoe's brand-new USB stack; sustained I/O corrupts it. Community long warned "unstable". / 补丁打的旧结构在新 USB 栈上崩。
- Fix: disable it, use USBMap. / 关掉，用 USBMap。

### 4b — Old-format USBMap silently dead on Tahoe / 老格式 USBMap 静默失效
- Symptom: enabling USBMap → the stick can't even read BaseSystem.dmg. / 开启后连 BaseSystem 都读不到。
- Root cause: Apple renamed port-property keys in macOS 26 (`UsbConnector`/`port` → `usb-port-type`/`usb-port-number`); old maps don't merge — no error, just dead. / Apple 改了键名，老映射不报错直接不合并。
- Fix: dual-format keys (both sets in parallel) — works on Sequoia AND Tahoe. / 双格式键名并行写入。

### 4c — Hub contention + motherboard's hidden hub / Hub 竞争 + 主板暗坑
- Symptom: keyboard+mouse receivers + stick on one hub → completely random behavior per boot; later, keyboard/mouse die mid-install. / Hub 上设备一多现象完全随机。
- Root cause: the installer environment ships a stripped-down USB stack; composite HID devices + mass storage contention = enumeration races. Bonus discovery via ioreg: **this board's two black rear USB2.0 ports are fed by an internal hub, not direct PCH**. / 裁剪版 USB 栈 + 复合设备竞争 = 枚举竞态；黑色 USB2.0 口是内部 Hub 汇聚的。
- Fix: install-time rule — keyboard direct in a rear port, mouse unplugged, storage never on USB. / 键盘直插后置口，鼠标拔掉，存储不走 USB。

## Pit 5: GPU Blocks = White Rectangle / 屏蔽 GPU = 大白块

**Symptom / 症状**: With AMDSupport excluded (a debugging step), the GUI shows a giant white rectangle covering 2/3 of the screen; clicking makes the cursor vanish. / 排查法屏蔽显卡驱动后 GUI 崩坏。

**Root cause / 根因**: Blocking the GPU driver falls back to EFI-framebuffer software rendering, which glitches on a 6K display. / 软件渲染在 6K 屏上渲染错乱。

**Fix / 解法**: Don't block — native drivers (see pit 2: MacPro7,1+Navi 21 is native). / 不屏蔽，用原生驱动。

**Lesson / 教训**: When bisecting by disabling drivers, consider display-output path dependencies. / 排障屏蔽驱动时要想显示输出路径的依赖。

## Pit 6: Hand-Cloned Installer Won't Boot / 手工克隆的安装器不能引导

**Symptom / 症状**: rsync/ditto-cloned installer volume fails to boot; separately, an install aborted mid-way with "installer is damaged, download again". / 克隆的安装器卷无法引导；另一次安装中途报"安装器已损坏"。

**Root cause / 根因**: createinstallmedia does more than file copy (boot structures, bless bookkeeping). / createinstallmedia 做的不止拷文件。

**Fix / 解法**: Always `createinstallmedia --volume <target>` onto the installer partition. / 永远用官方工具制作。

## Pit 7: Phantom Boot Entry / 幽灵启动项

**Symptom / 症状**: OC picker shows a "macOS" entry that boot-loops forever — loading a Sequoia 15.2-era boot.efi (582~2898) from a stale Preboot. / 同名条目无限循环，加载的是旧 Preboot 里的老引导文件。

**Root cause / 根因**: Orphaned APFS volume group from a previous macOS install still had a complete boot chain in Preboot; OC enumerates it as legit. / 孤儿卷组残留完整引导链。

**Fix / 解法**: `diskutil apfs deleteVolume` the orphan + remove its Preboot directory. / 删孤儿卷及其 Preboot 目录。

## Pit 8: ESP Dirty Flag & Mount-Point Drift / ESP 脏标记与挂载点漂移

**Symptom / 症状**: macOS-created 200MB ESPs won't mount without sudo (dirty flag); with multiple ESPs mounted, `/Volumes/EFI` vs `/Volumes/EFI 1` naming drifts. / ESP 挂不上；多 ESP 时挂载点命名漂移。

**Consequence / 后果**: Config written to the WRONG disk once (P3600's stale 2024 ESP instead of the stick). / 曾把 config 写错盘。

**Fix / 解法**: Always `sudo diskutil mount <device>`; **before writing, `diskutil info <mountpoint>` to confirm the device**. / 写前先确认设备号。

## Pit 9: USB-Stick Installs Are Fundamentally Unstable / U 盘安装路线根本性不稳（架构决策）

**Decision / 决策**: After fixing pit 4 and still hitting new random freezes, we abandoned USB entirely. Target disk became self-hosting: ESP + JHFS+ installer volume + APFS target on the same internal NVMe. All storage-layer freezes ceased to exist — USB now carries only the keyboard. / 放弃 U 盘，安装器搬到内置 NVMe。此后所有"存储层冻结"绝迹。

**Portability note / 可移植性说明**: The installer config is boot-device-agnostic and would boot from a USB stick's ESP — but sustained-18GB-read stability under the final fix set was never proven. Internal-NVMe is the golden path. / 配置本身与启动介质无关，但 U 盘持续大流量读取在最终修复组合下未验证。内置盘是金路径。

## Pit 10: Phase-2 Freeze at 5% (MSU Deadlock) / 第二阶段 5% 卡死

**Symptom / 症状**: First reboot into install continuation: white Apple logo + progress bar frozen at ~5% for 30+ minutes. Target volume: zero writes, no install.log. / 第一次重启后白苹果卡 5%，目标卷零写入。

**Forensics / 取证**: Preboot `com.apple.Boot.plist` kernel flags confirmed `lca-boot-mode=autoinstall-msu`; zero ProgressMarkers updates. / 内核参数确认走 MSU 模式，进度标记零更新。

**Root cause / 根因**: Tahoe's install continuation runs an MSU UpdateBrain hardware-compatibility check that **deadlocks** (not errors — hangs) on hackintoshes. / MSU 硬件兼容检查在黑苹果上死锁。

**Fix / 解法**: **RestrictEvents.kext 1.1.6 + boot-arg `revpatch=sbvmm`** — makes the flow believe it's in a VM, skipping the check. Same fix as the community's working case (SchmockLord repo issue #274 — also i9-10900K + Z490 + 6900 XT). / 让流程以为自己在 VM 里跳过检查。

## Pit 11: Phase-2 Freeze at "N minutes remaining" (AEA Personalization) / 第二阶段"剩余N分钟"冻结

**Symptom / 症状**: Past 5%, UI shows a time estimate, then keyboard+mouse die, progress frozen 20 minutes. / 过了 5%，显示剩余时间后键鼠全死。

**Forensics / 取证（本案的破案关键）**: The target Data volume carries **`macOS Install Data/ia.log`** — the installer's own log. Last line before death: `MSU: Starting preflight personalize`. / 目标 Data 卷里的 ia.log 显示死前最后一行是"开始个性化"。

**Root cause / 根因**: **Tahoe's install payload is AEA-encrypted; personalization must contact Apple's servers (albert.apple.com) to obtain per-machine decryption keys.** The installer environment had NO ethernet driver → infinite wait. (Earlier USB-era runs reached 20% because that was GUI phase-1 local copying — personalize never ran.) / 载荷是 AEA 加密的，个性化必须联网；安装环境没有网卡驱动就死等。

**Fix / 解法**: Enable AppleIntelI210Ethernet.kext + `e1000=0` (I225-V additionally needs device-id spoof 15F3→15F2 — all three required, the user's own empirical finding). Verify connectivity first: `curl https://albert.apple.com` from the working system before rebooting. Plug the cable. → **SUCCESS**. / 启用网卡 + 伪装 + 参数，验证出网后重启——成功。

---

## Pit 12: WiFi Double-Load After Root Patching / root patch 后的 WiFi 双载冲突

**Symptom / 症状**: After applying OCLP-Mod root patches (WiFi works!), boots become a lottery — sometimes freezing at the verbose text stage, sometimes at graphics init (Caps Lock dead), sometimes booting fine. No pattern. / 打完 root patch 后（WiFi 正常了），启动变成抽签——有时死在滚字符阶段，有时死在图形界面阶段（Caps Lock 无反应），有时正常。无规律。

**Forensics / 取证**: All 5 OC logs from the mixed good/bad boots are byte-identical through EXITBS — the freezes are purely kernel-side. The randomness across freeze *locations* points to a load-order race, not a deterministic offset mismatch. / 五次混合成败的 OC 日志在 EXITBS 之前逐字节一致——冻结全在内核侧。冻结位置随机漂移指向加载顺序竞态，而非确定的偏移错配。

**Root cause / 根因**: The daily config still injected the WiFi stack (IOSkywalkFamily + IO80211FamilyLegacy + AirPortBrcmNIC) AND blocked `com.apple.iokit.IOSkywalkFamily` — while the root patch had installed the same kexts system-side. Two copies of the same drivers competing per boot; the bundle-ID block also killed the root-patched system version. Whichever copy won the race determined whether the boot survived. / 日常 config 还在注入 WiFi 三件套并屏蔽原生 IOSkywalkFamily——而 root patch 已把同一套装进了系统。双份竞争 + 屏蔽误杀 root patch 版本，谁赢谁定生死。

**Fix / 解法**: Disable all EFI-injected WiFi entries + the IOSkywalkFamily block in the daily config. Root patches own the WiFi stack from now on. (This repo's daily config ships pre-fixed.) / 禁用日常 config 里全部 WiFi 注入条目和 IOSkywalk 屏蔽，WiFi 交给 root patch。

**Lesson / 教训**: **Root patches and EFI kext injection are mutually exclusive — pick one per subsystem.** This is documented OCLP guidance that's easy to miss when a config was built before patching. / **root patch 和 EFI 注入互斥——每个子系统二选一。**这是 OCLP 文档里容易漏看的准则。

---

## Pit 13: AMFIPass Lottery on Tahoe 26.6.2 / AMFIPass 抽签机（最终大案）

**Symptom / 症状**: Random boot failures with no pattern — sometimes freezing at verbose text, sometimes at graphics init (black screen, Caps Lock dead), sometimes booting fine. / 随机启动失败无规律——有时死在滚字符，有时死在图形初始化，有时正常。

**Forensics / 取证**: A failed boot's verbose screenshot showed the hang at launchd's `[Event: EndpointSecurity] Doing boot task` — the EndpointSecurity subsystem depends on AMFI being in a consistent state. With AMFIPass **disabled**, the boot died deterministically with `AMFI: code signature validation failed` on every OCLP-patched WiFi framework dylib (IO80211Old.dylib, WiFiPeerToPeer, LibSystemShim). / 失败启动的 verbose 截图显示挂在 launchd 的 EndpointSecurity 启动任务（依赖 AMFI 状态一致）；关闭 AMFIPass 后则确定性地死在 root patch 框架的签名校验上。

**Root cause / 根因**: **Race condition in parallel kext loading.** AMFIPass is a Lilu plugin that patches AMFI via byte-signature matching. The patch either matches or it doesn't — this is deterministic and cannot explain "sometimes works, sometimes doesn't." The non-determinism comes from **when** the patch lands relative to AMFI starting to enforce checks. Tahoe loads kexts in parallel across multiple threads; if AMFI's initialization thread completes before AMFIPass's patch callback fires, enforcement has already started → boot dies at EndpointSecurity. If the patch lands first → boot succeeds. Thread scheduling varies per boot → lottery. An alternating success/failure pattern suggests that after a failed boot, the system may fall back to a more conservative loading path (which works), then restore normal parallel loading on the next boot (which races again). On Sequoia, AMFIPass 1.4.1 was built for that exact OS — its timing is calibrated for Sequoia's load order, so the race never manifests. (Credit: this analysis was refined by the user during debugging — the initial "stale offsets" hypothesis was insufficient to explain the lottery pattern.) / **并行 kext 加载的竞态条件**。AMFIPass 用字节签名匹配打补丁——匹配与否是确定性的，解释不了抽签。非确定性来自补丁**何时**落地与 AMFI **何时**开始执行的竞争：Tahoe 并行加载 kext，如果 AMFI 的初始化线程先于 AMFIPass 的补丁回调完成 → 检查已开始 → 死在 EndpointSecurity；反之则正常启动。线程调度每次不同 → 抽签。（此分析由用户在调试过程中提出，修正了最初的"过期偏移"假说。）

**Fix / 解法**: Revert the OCLP root patches entirely (OCLP-Mod → Revert Root Patches), remove AMFIPass + `-amfipassbeta`, and switch WiFi to **[AppleBCMWLANCompanion (BCMC)](https://github.com/0xFireWolf/AppleBCMWLANCompanion)** — a Lilu plugin + chip configurator that makes Apple's *native* Tahoe Broadcom driver accept the legacy BCM43602/BCM4350 chips. No root patches, no AMFI bypass, no SIP lowering. / 彻底回滚 OCLP root patch、移除 AMFIPass，WiFi 改用 BCMC（让 Apple 原生 Tahoe 驱动认老卡），无 root patch、无 AMFI 绕过、无需降 SIP。

**Lesson / 教训**: **When a root patch's enabler (AMFIPass) is frozen upstream and the OS has moved past it, the whole root-patch approach is dead — find the native-driver path instead.** Also: a component that "works" on one macOS version because it was built for it will become a per-boot lottery on any newer version it doesn't recognize. / **root patch 的使能器（AMFIPass）上游冻结而 OS 已走远时，整条 root patch 路线即死——去找原生驱动路线。** 另外：一个组件"能用"只是因为它是为当前版本写的；版本一更新它就变成每次启动的抽签。

---

## Pit 14: The AirDrop Endgame — Every Patcher Dead, One Config Entry Away / AirDrop 终局战：全工具阵亡，差一条配置

**Symptom / 症状**: BCMC route (pit 13's fix) gives stable full-speed WiFi but **no AirDrop**. Chasing AirDrop: OCLP-Plus, OCLP-Mod and WiFi Patcher Pro all apply "successful" patches, yet the WiFi toggle won't even turn on. / BCMC 路线稳定满速但无 AirDrop。追击 AirDrop 的过程中：三家 root patch 工具全部"打补丁成功"，但 WiFi 开关根本打不开。

**Forensics / 取证**（each dead end eliminated one by one / 每个死胡同逐一排除）:

1. **WiFi Patcher Pro "missing Sparkle"** — red herring. The app bundle was complete (Sparkle 2.9.3, XPC services, universal binary); the launch failure was just Gatekeeper quarantine on browser downloads. A curl-downloaded copy ran fine. / 红鲱鱼：app 完整，是浏览器下载的 quarantine 标记导致。
2. **Root patches ARE applied** — patched frameworks (IO80211Old.dylib, WiFiPeerToPeerOld.dylib, LibSystemShim) verified system-side, and `BootKernelExtensions.kc` rebuilt containing `IOSkywalkFamily 1.0` + `IO80211Family 1200.13.1`. The patches land; they just can't *load*. / 补丁确实打上了（文件在、KC 里有），只是加载不了。
3. **`amfi=0x80 amfi_get_out_of_my_way=0x1 cs_enforcement_disable=1` (AMFIPass removed)** — deterministic boot death: every patched dylib fails `local signing public key not initialized` → `code signature validation failed` → launchd never completes. Apple has closed the boot-arg AMFI bypass on 26.6.2. / 纯 boot-arg 旁路在 26.6.2 上已被 Apple 封死：补丁框架全部校验失败，系统起不来。
4. **Recovery saga / 恢复拉锯**: with patches installed but unloadable, Tahoe boot-loops. `bless --last-sealed-snapshot` fails ("Read-only file system" — SSV); APFS snapshot forensics show the clean `com.apple.os.update` snapshot (XID 420) still intact next to the broken patched one (XID 476238, "Will root to"). Blessing the clean snapshot from another macOS fails; **the pragmatic fix was reinstalling Tahoe**. / 打了补丁却加载不了的 Tahoe 陷入启动循环；bless 回退快照被 SSV 只读挡死。快照取证显示干净快照还在，但无法切回——最终务实解法：重装。
5. **Fresh install + OCLP-Mod again** — patches still dead, but for a NEW reason: **Tahoe 26.6.2 `kmutil` refuses unsigned kexts entirely.** `sudo kmutil load -b com.apple.driver.AirPortBrcmNIC` → `not found`. Manually copying kexts to `/tmp`, `chown -R 0:0`, and loading them one by one *works at the kext level* — but the NIC stays dead: PCI personality matching must happen at boot, not post-hoc. / 重装后再打补丁仍死——新原因：26.6.2 的 kmutil 全面收紧，未签名 kext 既进不了 KC 也 load 不到。手动逐个加载可行但网卡无反应：PCI 匹配必须发生在启动阶段。
6. **SIP `0xFFFF0000` breakthrough / SIP 全关突破口**: with `csr-active-config=0xFFFF`, unsigned EFI-injected kexts load again — AMFIPass included. The pit-13 lottery also stops being fatal: even when AMFIPass loses its race, fully-disabled SIP lets the patched frameworks pass validation anyway. / SIP 全关后未签名 kext 恢复可加载，且即便 AMFIPass 竞态输了，框架校验也能过——抽签机拆了。

**Root cause / 根因**（the actual missing piece / 真正缺的那一块）: **`AirPortBrcmNIC.kext` — the WiFi driver itself — was `Enabled=false` in `Kernel→Add`.** OpenCore does **not** auto-load plugins bundled inside a parent kext; every plugin needs its own explicit `Add` entry (`IO80211FamilyLegacy.kext/Contents/PlugIns/AirPortBrcmNIC.kext`). IOSkywalkFamily and IO80211FamilyLegacy were injected — the stack loaded — but the driver that binds to the actual PCI device was never in the boot set. Compounding it: `BlueToolFixup` was off (AirDrop needs Bluetooth discovery) and `ipc_control_port_options=0` had been dropped from boot-args (patched daemons crash without it on Sonoma+). / **WiFi 驱动本体 AirPortBrcmNIC 在 Kernel→Add 里是 Enabled=false。** OC 不会自动加载父 kext 内的插件——每个插件必须有独立条目。栈加载了、驱动没加载。雪上加霜：BlueToolFixup 关着（AirDrop 靠蓝牙发现）、boot-args 丢了 ipc_control_port_options=0（打补丁的守护进程会崩）。

**Fix / 解法**: The shipped daily config = **EFI-inject the legacy kernel stack (Skywalk 1.0 + IO80211FamilyLegacy + AirPortBrcmNIC plugin, all `MinKernel=23.0.0`) + Block native `com.apple.iokit.IOSkywalkFamily`** so it can't conflict, **OCLP-Mod framework patches stay system-side** (they own userspace), **AMFIPass + `-amfipassbeta` + the amfi boot-args + SIP `0xFFFF`** so validation passes regardless of the race, **BlueToolFixup + BlueWakeFixup** for AirDrop, **`ipc_control_port_options=0`** for the daemons. Result: WiFi connects at full speed, **AirDrop fully working** — one of the very few confirmed AirDrop-working Tahoe 26.6.2 hackintosh builds. / 见仓库日常 config：EFI 注入内核栈（含 AirPortBrcmNIC 插件）+ 屏蔽原生 Skywalk + OCLP 框架补丁留在系统侧 + AMFIPass/SIP 全关保校验 + 蓝牙双修复 + IPC 参数。WiFi 满速、AirDrop 全通。

**Lesson / 教训**:
1. **A kext stack loading ≠ the driver loading.** Verify with `sudo kmutil inspect | grep -iE "skywalk|80211|brcm"` that the *leaf* driver (the one with the PCI personality) is in the loaded list — parent/child `Enabled` states are independent in OpenCore. / 栈加载了不等于驱动加载了：OC 里父/子条目的 Enabled 互相独立，必须验证叶子驱动。
2. **Pit 12's "EFI injection vs root patches are mutually exclusive" needs refining on 26.6.2**: they conflict only when both provide the *kernel-side* kexts. Since 26.6.2 `kmutil` never admits the patcher's unsigned kexts into the KC, EFI injection becomes the only kernel-side source — and the hybrid (EFI kernel + patched userspace) is not just possible but **the** working formula. / 坑 12 的"互斥"结论在 26.6.2 上要修正：只有内核侧双载才冲突；kmutil 收紧后 EFI 注入成了内核侧唯一来源，混合路线反而是唯一正解。
3. **When the enabler is racy, remove the stakes**: SIP fully off made AMFIPass's race irrelevant — the lottery stopped being a lottery because both outcomes now boot. Debugging tip: change the environment so failures can't be fatal, then fix the actual gap. / 让竞态变得无关紧要：SIP 全关后输赢都能启动。先把环境改成"失败不致命"，再补真正的缺口。
4. **A poisoned system volume is often cheaper to reinstall than to surgically revert** — SSV read-only + bless limitations + snapshot selector buried in APFS metadata make in-place recovery a multi-hour trap. Snapshots (clean `os.update` vs patched `bless`) at least let you *diagnose* precisely what state you're in. / 被污染的系统卷，重装往往比手术式回滚便宜。快照对比至少能精确诊断现状。

---

## Pit 15: The amfi Trio Kills DRM / amfi 三件套杀死 DRM

**Symptom / 症状**: Netflix in Chrome fails with "Chrome may not have protected content playback enabled"; AirDrop/WiFi fine. / Chrome 放 Netflix 报"未启用受保护内容播放"；WiFi/AirDrop 正常。

**Root cause / 根因**: The boot-args `amfi=0x80 amfi_get_out_of_my_way=0x1 cs_enforcement_disable=1` (carried from the debugging era as belt-and-suspenders next to AMFIPass) disable AMFI's code-signing enforcement entirely — and DRM stacks (Widevine in Chrome, FairPlay in Safari) refuse to run in that environment. This is precisely why AMFIPass exists: it patches only the library-validation path so root-patched frameworks load while AMFI's enforcement (which DRM needs) stays alive. Running BOTH means the trio silently voids AMFIPass's whole point. / 三件套把 AMFI 整个关掉，而 DRM（Widevine/FairPlay）拒绝在无签名校验环境运行。AMFIPass 的设计目标就是"只开框架校验的口子、保住其余 AMFI 功能"——和三件套并用等于白装。

**Fix / 解法**: Remove the three amfi args; keep `AMFIPass.kext` + `-amfipassbeta` + SIP `0xFFFF`. Under SIP-off, the patched frameworks pass even when AMFIPass loses its race — the pit-13 lottery stays dead AND Netflix plays. Bonus: removing them also restored AirPods audio routing visibility in DRM playback. / 删三件套、保 AMFIPass + SIP 全关。抽签机依然死、Netflix 能放。

**Lesson / 教训**: **AMFIPass and the amfi boot-args are alternatives, not layers.** If you need root patches to load, AMFIPass-only preserves DRM/Apple-ID/iCloud; the blunt boot-args preserve nothing. Community wisdom already said "incompatible" — the mechanism is that the trio destroys what AMFIPass is carefully keeping alive. / **二者是替代关系不是叠加关系。** 要 DRM 就只能 AMFIPass 单飞。

---

## Pit 16: `revpatch=sbvmm` Silently Disables AirDrop Receiving / VMM 伪装静默关闭 AirDrop 接收

**Symptom / 症状**: AirDrop send (Mac→iPhone) works, but the iPhone never lists the Mac in its share sheet. Universal Clipboard (copy on iPhone → paste on Mac) works, Bluetooth works, same machine's Sequoia install IS visible. / 发送正常、手机分享列表里就是没有这台 Mac；通用剪贴板正常；同机 Sequoia 能被发现。

**Forensics / 取证**（three rounds, narrowing each time / 三轮逐步收窄）:
1. bluetoothd logs decode the Apple Continuity manufacturer data as TLVs. The working Sequoia advertises types `9 16 22`; the broken Tahoe advertises `5 9 15 16 18` — same machinery (`status=0` everywhere), **different service set**. AirDrop's presence is one gated decision, not a radio problem.
2. sharingd's boot log shows the gate: `AirDrop not ready: wifi=?, bluetooth=?, carplay=?, receivingEnabled=?, isVirtualMachine=?, isMulticastAdvertisementsDisabled=?` (values privacy-redacted) — six suspects, one of which is `isVirtualMachine`.
3. Differential on the same machine: Sequoia's boot-args vs Tahoe's boot-args differ in exactly one meaningful token — **`revpatch=sbvmm` (present only on Tahoe)**. That's RestrictEvents' VMM spoof, needed during *install* to pass update checks — and macOS disables AirDrop on virtual machines.

**Fix / 解法**: Delete `revpatch=sbvmm` from the **daily** config's boot-args (the install config keeps it). Reboot → iPhone lists the Mac immediately; photo received over `fe80::…%awdl0`. Note: `kern.hv_support` stays `1` on this OCLP-patched system either way — the gate reads the spoofed platform identity, not that sysctl. / 从日常 config 删掉 revpatch=sbvmm（安装 config 保留）。重启即被列出、照片经 awdl0 收到。注意 hv_support 与本案无关。

**Lesson / 教训**:
1. **Install-time crutches must not leak into daily configs.** `revpatch=sbvmm` is install-only; in daily use it costs AirDrop receiving for a "software update" convenience that a natively-supported SMBIOS doesn't even need. / **安装期的拐杖不能带进日常配置**——代价是静默砍掉整个接收方向。
2. **When a feature is silently missing (not erroring), audit the feature's *capability gates* — daemons publish them in boot logs** (`Device Capabilities (… AirDrop: …)`, `AirDrop not ready: …`). Privacy redaction hides the values, but a two-OS differential of the inputs (boot-args) can still isolate the culprit. / 功能"静默缺席"时，去读守护进程开机时的能力门禁日志；值被隐私遮蔽没关系，两系统输入差分一样能定位。
3. **Decode the BLE TLVs**: `log show --predicate 'process == "bluetoothd"' | grep "manufacturer data"` gives you the literal advertisement bytes — decoding the Continuity TLV types tells you which services are actually on air. / 抓 BLE 厂商数据解码 TLV，广播里有什么服务一目了然。

---

## Bonus Fix: Hidden-Network Auto-Join Priority / 附：隐藏网络自动加入优先级

**Symptom / 症状**: Mac always auto-joins the visible `BlizzardNew-5G` instead of the hidden preferred SSID, even after manually joining the hidden one (the join log even records `UserPreferredNetworkNames`). / 明明手动连过隐藏网络（日志都记了偏好），还是总连可见网络。

**Root cause / 根因**: On Sonoma+ known networks live in `/Library/Preferences/com.apple.wifi.known-networks.plist`; entries default to auto-join when no `AutoJoin` key exists. A broadcasting visible network wins the race against a hidden SSID that requires directed probes. / 可见网络的广播帧永远比隐藏网络的定向探测先到。

**Fix / 解法**: Set `AutoJoin=false` on the visible network's entry and `AutoJoin=true` on the hidden one (editable from any other macOS install with the target Data volume mounted — file is `root:wheel 600`; verify with a key-count diff that the plistlib round-trip lost nothing). GUI equivalent: System Settings → Wi-Fi → ⓘ on the visible network → uncheck "Automatically join". Watch out: networks added via `Cloud Sync` can get their auto-join re-synced by iCloud. / 给可见网络条目写 AutoJoin=false、隐藏网络写 true（可从另一套 macOS 直接改目标卷上的文件；注意 iCloud 同步可能覆盖）。

---

## Forensic Toolkit / 取证工具箱

| Evidence / 证据 | Location / 位置 | Shows / 能看到什么 |
|---|---|---|
| OC logs (`opencore-*.txt`) | ESP root; **filenames are UTC — Beijing +8** / 文件名是 UTC | Coverage ends at EXITBS; kernel-side invisible / 只到内核交接 |
| **`ia.log`** | Target Data volume `macOS Install Data/ia.log` | **The gold evidence for phase-2 hangs** / 第二阶段卡死的黄金证据 |
| ProgressMarkers | Target Preboot `/var/db/ProgressMarkers/` | Install progress milestones / 安装进度标记 |
| bless state / bless 状态 | `bless --info /Volumes/<target>` | Continuation boot target / 续装引导指向 |
| Volume groups / 卷组结构 | `diskutil apfs list <disk>` | System+Data+Preboot+Recovery+VM all present = phase-2 env ready / 五卷齐=续装环境就绪 |
| kext injection / 注入验证 | `kextstat` in running system; on 26.x `sudo kmutil inspect | grep <id>` | What actually loaded / 实际加载了什么 |
| APFS boot snapshots / 启动快照 | `diskutil apfs listSnapshots <vol>` | Which snapshot "Will root to"; clean `os.update` vs patched `bless` / 干净更新快照 vs 被 patch 的 bless 快照 |
| Known WiFi networks / 已知网络 | `<Data>/Library/Preferences/com.apple.wifi.known-networks.plist` | `AutoJoin`/`Hidden` per SSID (root:wheel 600; edit from another macOS) / 每网络自动加入与隐藏标志 |
| USB topology / USB 拓扑 | `ioreg -p IOUSB -w 0` | Hub chains, port addresses / Hub 链与端口地址 |
| Config diff / 配置对比 | `plistlib` structural walk | Field-level drift between configs / 字段级差异 |

## Meta-Lessons / 元教训

1. **Two configs, two worlds / 两套 config 两个世界**: installer env (BaseSystem KC) ≠ installed system (Prelinked KC). Components that help one can kill the other. / 帮一个的组件可能杀死另一个。
2. **Freeze ≠ panic / 冻结不等于 panic**: no panic text often means a userspace deadlock waiting on something external (network!) — check what the installer is *waiting for*, not what crashed. / 无 panic 常意味着用户态在等外部资源——查它在等什么。
3. **Read the installer's own diary / 读安装器自己的日记**: `ia.log` answers "where did it die" in one file read. / 一个文件回答"死在哪"。
4. **Log timestamps lie about timezone / 日志时间戳有时区**: ESP log names are UTC. We chased ghosts for an hour before the +8 correction. / 文件名是 UTC，注意换算。
5. **Silent skip is the deadliest failure mode / 静默跳过是最致命的失败模式**: the empty-ACPI pit (pit 1) cost a full day because directories "existed". / 目录"存在"但为空，坑了一整天。
6. **Move the problem off the failing bus / 把问题挪出故障总线**: when USB kept failing in every variant, hosting the installer on internal NVMe eliminated the entire failure class. / 内置盘方案消灭了整类故障。

## WhateverGreen: Keep or Drop? / WEG 去留分析

For this exact combo (native 0x73BF + MacPro7,1), the GUI works **without** WEG — proven empirically through the whole install. What you lose without it: **Safari/TV-app DRM** (FairPlay 2/3 — Netflix 4K, Apple TV+; Chrome/Firefox work via Widevine at reduced quality), the **AGDP safety net** against future OS updates changing policy tables, and some sleep/wake robustness. Recommendation: keep WEG in the daily config (already so in this repo) — the installer-only freeze doesn't apply to installed systems. / 本配置无 WEG 也能显示（全程实证），但损失 DRM、AGDP 保险与唤醒稳健性。建议日常 config 保留（本仓库已如此）。

## Acknowledgements / 致谢

Same as README — plus everyone who documented Tahoe quirks in the forums cited throughout this chronicle. / 同 README——并感谢各论坛记录 Tahoe 怪癖的每一个人。
