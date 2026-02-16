# sqm-scripts-nss: Hardware-Accelerated SQM for Qualcomm NSS (IPQ807x)

> Eliminate bufferbloat on OpenWrt with **zero CPU overhead** using Qualcomm NSS hardware offload. Achieve A+ bufferbloat grades at 300+ Mbps while your router's CPU stays idle.

This is a fork of [qosmio/sqm-scripts-nss](https://github.com/qosmio/sqm-scripts-nss) with a flattened qdisc hierarchy, automatic overhead detection, tighter queue limits, and dead code removal. Tested on IPQ807x (Qualcomm IPQ8074) with kernel 6.12, PPPoE, VLAN 7, and 300 Mbps FTTH.

**Maintainer:** Julius Bairaktaris — [julius@bairaktaris.de](mailto:julius@bairaktaris.de)

## Why NSS SQM?

Standard SQM (fq_codel, CAKE) runs on the router's CPU. On a Qualcomm IPQ807x, the **NSS (Network Subsystem)** co-processor handles packet scheduling in dedicated hardware, freeing the CPU entirely. This means:

- **No CPU load** from traffic shaping, even at gigabit speeds
- **Consistent latency** regardless of what else the router is doing (WiFi, NAT, firewall)
- **A+ bufferbloat grades** on [Waveform](https://www.waveform.com/tools/bufferbloat) with zero added latency under download load

The tradeoff: NSS only supports `nssfq_codel` (no CAKE, no HFSC, no multi-class traffic classification). But for most users, a well-tuned fq_codel with hardware offload outperforms a feature-rich software qdisc that competes with the CPU for resources.

## What This Fork Changes

The upstream script creates a deep 7-qdisc hierarchy per direction with priority bands that **never receive any traffic** (verified with real packet counters). This fork strips it down to the minimum effective configuration: 2 qdiscs per direction, auto-detected overhead, and tighter queue limits.

See [Fork Changes](#fork-changes-v20260215) for the full technical breakdown with before/after measurements.

## Requirements

- OpenWrt with NSS support (IPQ807x / IPQ8074 platform)
- `sqm-scripts` package
- `kmod-qca-nss-drv-qdisc` and `kmod-qca-nss-drv-igs` kernel modules
- `luci-app-sqm` (optional, for GUI configuration)

## Installation

### Quick Install (scp)

Copy the script directly to your router:

```bash
scp nss-zk.qos root@192.168.1.1:/usr/lib/sqm/nss-zk.qos
```

### OpenWrt Build System

Add to your feeds and build:

```bash
echo "src-git sqm_scripts_nss https://github.com/JuliusBairaktaris/sqm-scripts-nss.git" >> feeds.conf
./scripts/feeds update && ./scripts/feeds install sqm-scripts-nss
```

Then select `sqm-scripts-nss` in menuconfig under Network.

## Configuration

1. Go to **Network -> SQM QoS** in LuCI
2. Set your **WAN interface** (e.g. `wan`)
3. Set **download/upload speeds** to ~95% of your actual line speed
4. Under **Queue Discipline**, select `fq_codel` and `nss-zk.qos`
5. Click **Save & Apply**

That's it for basic setup. The script auto-detects your overhead based on your connection type.

### Verifying It Works

```bash
tc -s qdisc show dev wan
tc -s qdisc show dev ifb@wan
```

You should see exactly **2 qdiscs per direction** (nsstbl + nssfq_codel), both with `accel_mode 0` (NSS offload active):

```
qdisc nsstbl 1: root refcnt 5 rate 145Mbit mtu 1536b accel_mode 0
 Sent 4316052 bytes 6563 pkt (dropped 0, overlimits 182 requeues 0)
qdisc nssfq_codel 10: parent 1: target 4ms limit 354p interval 50ms
 flows 1024 quantum 304 set_default accel_mode 0
 Sent 4316052 bytes 6563 pkt (dropped 0, overlimits 0 requeues 0)
 maxpacket 1518 drop_overlimit 0 new_flow_count 2258 ecn_mark 0
```

If you see `nssprio`, `nsspfifo`, or `nssred` in the output, the old upstream script is still loaded.

### Overhead (Auto-Detected)

When `OVERHEAD` is not set in the SQM config, the script **automatically detects** the correct value by reading your network interface configuration:

| Component | Bytes | How detected |
|---|---|---|
| Ethernet (header + FCS) | 18 | Always present |
| NSS internal framing | 4 | Always present |
| PPPoE (header + PPP) | +10 | `network.<iface>.proto = pppoe` |
| DS-Lite (IPv6 encap) | +40 | `network.<iface>.proto = dslite` |
| MAP-E (IPv4-in-IPv6) | +20 | `network.<iface>.proto = map` |
| VLAN 802.1Q tag | +4 | `network.<iface>.device` contains `.` |

Auto-detected values per connection type:

| Connection Type | Overhead |
|---|---|
| Ethernet (direct) | 22 |
| PPPoE | 32 |
| PPPoE + VLAN (e.g. Telekom DE) | 36 |
| DS-Lite (IPv4-in-IPv6 tunnel) | 62 |
| MAP-E (IPv4-in-IPv6 mapping) | 42 |

Check what was detected:

```bash
logread | grep auto_detect_overhead
# SQM: auto_detect_overhead: proto=pppoe device=wan.7 -> overhead=36 bytes
```

Setting any non-zero `OVERHEAD` value in LuCI overrides auto-detection.

### Gaming-Optimized Tuning

For low-latency gaming, set these under **Advanced Options** in LuCI:

| Setting | Value | Why |
|---|---|---|
| Target (egress + ingress) | `4ms` | Aggressive CoDel — detects bloat faster |
| Advanced option string | `interval 50ms quantum 304` | 50ms reaction time, 5x priority for small packets (<304 bytes like game/VoIP packets) |

The script auto-calculates queue limits based on a 30ms window. Don't set manual limits unless you know what you're doing.

### Example: Telekom Germany FTTH (300/150 Mbps)

Deutsche Telekom uses PPPoE over VLAN 7. Minimal SQM config — overhead is auto-detected:

```
Interface:  wan
Download:   285000 kbps  (300 * 0.95)
Upload:     142500 kbps  (150 * 0.95)
Script:     nss-zk.qos
Qdisc:      fq_codel
Overhead:   (leave empty — auto-detects 36)

Advanced options:  interval 50ms quantum 304
Target:            4ms
```

## Fork Changes (v20260215)

### Before: Upstream v20240502

The upstream script creates **7 qdiscs per direction** (14 total). Real packet counters from an IPQ807x router running 140/285 Mbps:

```
=== EGRESS (wan) — 7 qdiscs ===
nsstbl 1: root           rate 140Mbit    219,785 pkt
nssprio 10: parent 1:    bands 3         219,785 pkt
nsstbl 100: parent 10:1  rate 14Mbit           0 pkt  (dead)
nsspfifo 1000:                                 0 pkt  (dead)
nsstbl 200: parent 10:2  rate 56Mbit           0 pkt  (dead)
nssred 2000:                                   0 pkt  (dead)
nssfq_codel 300: parent 10:3 set_default 220,011 pkt  (all traffic)

=== INGRESS (ifb@wan) — 7 qdiscs ===
nsstbl 1: root           rate 285Mbit    829,512 pkt
nssprio 10: parent 1:    bands 3         829,512 pkt
nsstbl 100: parent 10:1  rate 28.5Mbit         0 pkt  (dead)
nsspfifo 1000:                                 0 pkt  (dead)
nsstbl 200: parent 10:2  rate 114Mbit          0 pkt  (dead)
nssred 2000:                                   0 pkt  (dead)
nssfq_codel 300: parent 10:3 set_default 829,569 pkt  (all traffic)
```

10 out of 14 qdiscs process **zero packets**. The `nssprio` DSCP classification is ineffective because NSS ECM (Enhanced Connection Manager) accelerates flows after the first packet, bypassing Linux TC filters entirely.

### After: Fork v20260215

**2 qdiscs per direction** (4 total):

```
=== EGRESS (wan) — 2 qdiscs ===
nsstbl 1: root              rate 145Mbit  mtu 1536b  accel_mode 0
nssfq_codel 10: parent 1:   target 4ms  limit 354p  interval 50ms  quantum 304

=== INGRESS (ifb@wan) — 2 qdiscs ===
nsstbl 1: root              rate 295Mbit  mtu 1536b  accel_mode 0
nssfq_codel 10: parent 1:   target 4ms  limit 720p  interval 50ms  quantum 304
```

### Bufferbloat Test Results (Waveform)

Both upstream and fork achieve **A+** grades. Latency is identical — the fork just removes dead weight:

| Metric | Upstream | Fork |
|---|---|---|
| **Bufferbloat grade** | A+ | A+ |
| Unloaded latency | 11 ms | 11 ms |
| Download +latency | +0 ms | +0 ms |
| Upload +latency | +5 ms | +5 ms |
| Download speed | 249 Mbps | 267 Mbps |
| Upload speed | 121 Mbps | 125 Mbps |
| Qdiscs (per direction) | 7 | 2 |
| Queue limit | 3000p (manual) | 354-720p (auto) |

### Technical Changes

#### 1. Flattened qdisc hierarchy

Removed the `nssprio` subtree (nssprio, 2x nsstbl, nsspfifo, nssred) — 5 dead qdiscs per direction. `nssfq_codel` is now a direct child of the root `nsstbl`.

The old hierarchy existed for DSCP-based priority classification, but it never worked: `set_default` routed all traffic to band 2, and no TC filters were configured. Even if filters existed, NSS ECM accelerates flows after the first packet — only the initial SYN ever hits TC classification.

#### 2. Tighter queue limits

`MAXQLIMIT_MS` reduced from 100ms to 30ms. Queue depth is auto-calculated based on speed and MTU:

- 300 Mbps → ~720 packets (30ms)
- 150 Mbps → ~354 packets (30ms)
- Below ~130 Mbps → 200 packets (MINLIMIT floor, sufficient for CoDel)

The old 100ms default allowed initial bufferbloat before CoDel's detection kicked in.

#### 3. Automatic overhead detection

New `auto_detect_overhead()` function reads UCI network config to determine L2 framing overhead. Detects PPPoE (+10 bytes) and VLAN tagging (+4 bytes) automatically. Falls back to 22 bytes (Ethernet + NSS) for plain ethernet connections.

The upstream script hardcoded 18 bytes when overhead wasn't configured — wrong for PPPoE (needs 32-36) and gave no indication that configuration was needed.

#### 4. ECN passthrough (firmware limitation documented)

ECN (Explicit Congestion Notification) allows CoDel to **mark** packets instead of **dropping** them during congestion. The sender sees the CE (Congestion Experienced) mark and slows down — same effect as a drop, but no retransmissions needed. This is especially useful for ingress shaping where drops trigger expensive TCP retransmits.

**Source code analysis** — ECN appears fully wired from tc → kernel → firmware:

| Layer | Source | ECN support |
|---|---|---|
| Firmware interface | [`nss_shaper.h`](https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-drv/-/blob/NHSS.QSDK.12.5.0.6/exports/nss_shaper.h) | `uint32_t ecn` field in `nss_shaper_config_codel_param` |
| Kernel module | [`nss_codel.c`](https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-clients/-/blob/NHSS.QSDK.12.5.0.6/nss_qdisc/nss_codel.c) | `q->ecn = qopt->ecn` — wires tc parameter through to firmware |
| tc userspace | `q_nss.c` (iproute2 patch) | `ecn`/`noecn` keywords parsed, `opt.ecn` set |
| tc stats output | `q_nss.c` | `ecn_mark %u` counter printed in `tc -s` |

**However, testing confirms the NSS firmware does NOT actually perform ECN marking:**

| Test condition | Result |
|---|---|
| ECN negotiated end-to-end | Yes — `ss -teni` shows `ecn ecnseen` on all TCP flows |
| iperf3 `-4 -R -P 4` sustained 265 Mbps, 15 sec | ~1200 CoDel drops (`drop_overlimit`) |
| `ecn_mark` counter after test | **0** (never incremented) |
| Firmware / kernel | NHSS.QSDK.12.5, kernel 6.12.68 |
| Test date | February 2026 |

The firmware accepts the `ecn` parameter and exposes the `ecn_mark` counter, but the CoDel implementation always drops rather than marks. The counter is dead code in the current firmware.

The script still forwards SQM's ECN settings (`EECN`/`IECN` from `defaults.sh`) to `nssfq_codel` for forward-compatibility — if Qualcomm fixes the firmware, ECN marking will start working without script changes. If the tc binary rejects the parameter (stock iproute2 without the [ECN patch](https://github.com/qosmio/openwrt-ipq/pull/86)), the script retries without it and logs a warning.

#### 5. Code cleanup

- Added ECN passthrough with graceful fallback (forwards SQM ECN settings; retries without if tc rejects)
- Removed `cake_egress()` / `cake_ingress()` stubs (CAKE not supported by NSS)
- Extracted `calc_effective_mtu()` to DRY up duplicated overhead+MTU logic in `egress()`/`ingress()`
- Replaced bash-only `[[ ]]` tests with POSIX `case` patterns (shebang is `#!/bin/sh`)
- Simplified `sqm_stop()` for flat hierarchy (single `tc qdisc del root` instead of nested teardown)
- Added `ipt_log_rewind` in `sqm_stop()` for proper iptables rule cleanup
- Fixed `exit $?` → `return $?` in `sqm_start()` (no longer kills the shell on `check_addr` failure)
- Fixed inconsistent raw `tc` vs `$TC` wrapper usage in `sqm_stop()`
- Proper `local` scoping for all function variables (`burst`, `quantum`, `mtu`, etc.)
- Removed unused ECN parameter from `add_nsstbl()`
- Fixed `local IFACE` vs `IF` variable name mismatch

## FAQ

### Does this work with CAKE?

No. NSS hardware only supports `nssfq_codel`. CAKE, HFSC, and HTB are software-only qdiscs that run on the CPU. If you need CAKE features (per-host fairness, tin-based prioritization), use the standard `piece_of_cake.qos` script — but expect CPU usage.

### What routers are supported?

Any OpenWrt router with a **Qualcomm IPQ807x / IPQ8074** SoC and NSS-enabled firmware. Common models include the Dynalink DL-WRX36, Xiaomi AX9000, and Netgear WAX630. The NSS co-processor must be active with `kmod-qca-nss-drv-qdisc` loaded.

### How is this different from qosmate?

[qosmate](https://github.com/hudra0/qosmate) is a CPU-based QoS system using HFSC/HTB + nftables for DSCP classification. It offers multi-class traffic prioritization but cannot offload to NSS. This script trades classification features for **zero CPU overhead** via hardware offload — better for high-throughput links where CPU is the bottleneck.

### Do I need to set overhead manually?

No. The script auto-detects PPPoE (+10 bytes) and VLAN (+4 bytes) overhead from your UCI network config. You only need to set it manually if auto-detection gets it wrong (check with `logread | grep auto_detect_overhead`).

### Why is my bufferbloat grade A instead of A+?

A vs A+ is within Waveform's test-to-test variance (~1ms). Both grades indicate effectively zero added latency. If you see B or below, verify your shaper rates are set to ~95% of your actual line speed.

## Known Limitations

- **Only fq_codel** is supported as a queue discipline (NSS firmware limitation)
- **No traffic classification** — DSCP marking, squashing, and multi-class prioritization are not possible with NSS qdiscs. All flows get equal treatment within fq_codel's fair queuing.
- **ECN marking does not work** — Despite full source-level support from tc → kernel module ([`nss_codel.c`](https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-clients/-/blob/NHSS.QSDK.12.5.0.6/nss_qdisc/nss_codel.c): `q->ecn = qopt->ecn`) → firmware interface ([`nss_shaper.h`](https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-drv/-/blob/NHSS.QSDK.12.5.0.6/exports/nss_shaper.h): `nss_shaper_config_codel_param.ecn`), the NSS firmware does not perform ECN marking. The `ecn_mark` counter in `tc -s` always reads 0 even with ECN-negotiated flows and active CoDel drops. See [ECN passthrough](#4-ecn-passthrough-firmware-limitation-documented) for full test methodology.
- **peakrate (dual-rate shaping) untested** — The `nsstbl` kernel module ([`nss_tbl.c`](https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-clients/-/blob/NHSS.QSDK.12.5.0.6/nss_qdisc/nss_tbl.c)) supports CIR + PIR via `lap_cir`/`lap_pir`, and tc accepts the `peakrate` keyword with the [iproute2 patch](https://github.com/qosmio/openwrt-ipq/pull/86). However, `tc -s` does not display peakrate in output, making it unclear whether the firmware actually applies dual-rate shaping.
- **No `stab` (Link Layer Adaptation)** — NSS hardware qdiscs don't support tc's statistics-based overhead accounting. The auto-detected fixed overhead is sufficient for Ethernet/FTTH; DSL with ATM framing may see minor inaccuracy for small packets.
- On kernel 5.15, the router may crash on `ip link up` of the IFB interface under high load, especially when triggered by hotplug. A workaround in the script prevents execution from hotplug events.

## Credits

- Maintainer: Julius Bairaktaris ([@JuliusBairaktaris](https://github.com/JuliusBairaktaris)) — [julius@bairaktaris.de](mailto:julius@bairaktaris.de)
- Upstream: [@qosmio](https://github.com/qosmio/sqm-scripts-nss)
- Original script: [@rickkdotnet](https://github.com/rickkdotnet/sqm-scripts-nss)
- Derived from work by [@michaelchen644](https://forum.openwrt.org/u/michaelchen644)
- NSS Packages: [qosmio/nss-packages](https://github.com/qosmio/nss-packages)

---

*If this script helped you eliminate bufferbloat, consider starring the repo — it helps others find it.*
