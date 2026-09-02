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

## Forensic Toolkit / 取证工具箱

| Evidence / 证据 | Location / 位置 | Shows / 能看到什么 |
|---|---|---|
| OC logs (`opencore-*.txt`) | ESP root; **filenames are UTC — Beijing +8** / 文件名是 UTC | Coverage ends at EXITBS; kernel-side invisible / 只到内核交接 |
| **`ia.log`** | Target Data volume `macOS Install Data/ia.log` | **The gold evidence for phase-2 hangs** / 第二阶段卡死的黄金证据 |
| ProgressMarkers | Target Preboot `/var/db/ProgressMarkers/` | Install progress milestones / 安装进度标记 |
| bless state / bless 状态 | `bless --info /Volumes/<target>` | Continuation boot target / 续装引导指向 |
| Volume groups / 卷组结构 | `diskutil apfs list <disk>` | System+Data+Preboot+Recovery+VM all present = phase-2 env ready / 五卷齐=续装环境就绪 |
| kext injection / 注入验证 | `kextstat` in running system | What actually loaded / 实际加载了什么 |
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
