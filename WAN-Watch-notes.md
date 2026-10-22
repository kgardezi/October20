# WAN Watch — commands, logs and devices

> Hostnames, site codes, domains, server name, IPs and MAC addresses in this file are placeholders that keep the real format (e.g. `uaaa100dl0109` = NY5 DL). The real values are kept outside this repository.

Working notes for the WAN Watch dashboard: monitoring WAN circuits (interface up/down/flap) and OSPF adjacencies (down/flap) between branch/campus sites and the three data centers, driven by syslog.

Prototype dashboard (sample data, simulated feed): https://claude.ai/artifact/1WdgAi1uH7fAfuk7L1QX4k

---

## 1. Syslog server and files

| Item | Value |
|---|---|
| Server | `<syslog-server>` |
| Directory | `<syslog-dir>` |
| Devices sending syslog | ~600–700 |
| Live file | `log` |
| Rotation | `log` → `logYYYYMMDD` (daily, uncompressed, e.g. `log20261021`) → `logYYYYMMDD.gz` |
| Retention seen | `log20260101.gz` … `log20261020.gz`, plus `log20261021` (not yet compressed) |

How the collector will use the three stages:

- `log` — followed live by **name** (like `tail -F`, capital F), so it reopens the new `log` after midnight rotation and finishes the old file first.
- `logYYYYMMDD` — read once at startup together with `log`, to rebuild current state (a circuit that went down before a restart still shows red).
- `.gz` — never read live; only for history/reports (e.g. outages per circuit per month).

---

## 2. Log line format

```
Oct 21 21:05:21 ujjj110as1001.us.corp.example Oct 21 2026 21:05:20.578 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet1/0/40, changed state to down
└ server time ┘ └──── device FQDN ───────┘ └──── device time ─────────┘ └──────────────── message ─────────────────────────────────────┘
```

- Hostname = 4th field; strip the domain (`.us.corp.example`, `.ca.corp.example`, `.br.corp.example`, `.co.corp.example`). Lowercase in the log.
- IOS/IOS-XE device time: `Oct 21 2026 21:05:20.578 EDST:`
- NX-OS device time: `2026 Oct 21 21:05:21.275060 EDST:`
- Some devices repeat their name: `ceee500as0402.ca.corp.example CEEE500AS0402: Oct 21 2026 21:05:27.874 EDST: ...`
- Long messages wrap; `(message repeated 1 time)` suffix appears.

### Sample lines (from `tail -f log`, 21 Oct 2026)

```
Oct 21 21:04:35 ubbb200sf0929.us.corp.example 2026 Oct 21 21:04:35.946002 EDST: %L2FM-4-L2FM_MAC_MOVE2: Mac 0000.0000.0000 in vlan 1400 has moved from Po2 to Po1523
Oct 21 21:04:36 ceee500as1901.ca.corp.example Oct 21 2026 21:04:35.084 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet1/0/29, changed state to up
Oct 21 21:04:37 bhhh800as0802.br.corp.example Oct 21 2026 21:04:35.968 EDST: %ILPOWER-5-IEEE_DISCONNECT: Interface Gi2/0/7: PD removed
Oct 21 21:04:37 ceee500as1901.ca.corp.example Oct 21 2026 21:04:36.085 EDST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/29, changed state to up
Oct 21 21:04:37 uiii900as0302.us.corp.example Oct 21 2026 21:04:36.004 EDST: %DOT1X-5-SUCCESS: Switch 1 R0/0: sessmgrd: Authentication successful for client (0000.0000.0000) on Interface Gi1/0/33 ...
Oct 21 21:04:39 ceee500as1101.ca.corp.example Oct 21 2026 21:04:38.495 EDST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet3/0/27, changed state to down
Oct 21 21:04:39 ubbb200vs0204.us.corp.example 2026 Oct 21 21:04:39.713198 EDST: %VPC-2-L3_VPC_UNEQUAL_WEIGHT: Layer3 peer-router is enabled. Please make sure both vPC peers have the same L3 routing configuration (message repeated 1 time)
Oct 21 21:04:40 ceee500as1101.ca.corp.example Oct 21 2026 21:04:39.498 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet3/0/27, changed state to down
Oct 21 21:04:40 ukkk120es0202.us.corp.example 2026 Oct 21 21:04:40.683406 EDST: %USER-SLOT1-5-SYSTEM_MSG: 312524 bytes written to file /bootflash/BufferMonitor-1HourData (message repeated 1 time)
Oct 21 21:04:42 ceee500as0901.ca.corp.example Oct 21 2026 21:04:40.994 EDST: %CDP-4-DUPLEX_MISMATCH: duplex mismatch discovered on GigabitEthernet5/0/31 (not full duplex), with SEP000000000000 Port 1 (full duplex).
Oct 21 21:05:27 cfff600as0902.co.corp.example Oct 21 2026 21:05:25.971 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet3/0/9, changed state to up
Oct 21 21:05:28 uaaa100sf0190.us.corp.example 2026 Oct 21 21:05:28.787749 EDST: %ETHPORT-5-IF_TRUNK_DOWN: Interface Ethernet1/45, vlan 3040 down
Oct 21 21:05:28 uaaa100sf0190.us.corp.example 2026 Oct 21 21:05:28.787902 EDST: %ETHPORT-5-IF_DOWN_LINK_FAILURE: Interface Ethernet1/45 is down (Link failure)
Oct 21 21:05:28 ceee500as0402.ca.corp.example CEEE500AS0402: Oct 21 2026 21:05:27.874 EDST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet3/0/22, changed state to down
Oct 21 21:06:06 uaaa100dl0109.us.corp.example 2026 Oct 21 21:06:06.576843 EDST: %L2FM-4-L2FM_MAC_MOVE2: Mac 0000.0000.0000 in vlan 508 has moved from Po10 to Po6
Oct 21 21:06:07 mccc300cl0201.us.corp.example Oct 21 2026 21:06:06.145 EDST: %SNMP-3-AUTHFAIL: Authentication failure for host 10.0.0.50
```

Observation: in these samples there are **no** WAN circuit or OSPF events. Almost all traffic is access-switch (`as`) port flaps, MAC moves, dot1x/MAB and PoE. Filtering by device type is essential.

### Messages the dashboard acts on

| Mnemonic | Platform | Meaning |
|---|---|---|
| `%LINK-3-UPDOWN` | IOS / IOS-XE | Interface up/down |
| `%LINEPROTO-5-UPDOWN` | IOS / IOS-XE | Line protocol up/down |
| `%ETHPORT-5-IF_DOWN_*` / `%ETHPORT-5-IF_UP` | NX-OS | Interface down/up |
| `%OSPF-5-ADJCHG` | IOS / IOS-XE | `... Nbr x.x.x.x on <intf> from FULL to DOWN` / `from LOADING to FULL` |
| `%OSPF-5-ADJCHANGE` | NX-OS | `... Nbr x.x.x.x on <intf> went DOWN` / `went FULL` |

Everything else is ignored.

---

## 3. Hostname standard

`UIII900CL0801` = 13 characters:

| Pos | Example | Field |
|---|---|---|
| 1 | `U` | Country |
| 2–4 | `III` | City |
| 5–7 | `900` | Site |
| 8–9 | `CL` | Device type |
| 10–11 | `08` | Floor |
| 12–13 | `01` | Device number |

- Site key = first 7 characters (`UIII900`).
- Some device types are 3+ letters (`ACS`, `ALM`, `SNF`, `SRV`, `UAT`, `VDC`, `Corvil`) — parse as "all letters between site digits and last 4 digits".
- The country letter is not unique (two different countries can share a letter; the domain region `.us` / `.ca` / `.br` / `.co` differs). City codes keep site keys unique.

### Device types

| Code | Device | Code | Device |
|---|---|---|---|
| ACS | Access Control Server (TACACS) | IS | Internet DMZ Switch |
| AG | Aggregation Switch | LL | LAN Loadbalancer |
| AR | Aggregate Router (Voice/Video) | ML | External L3 Distribution Switch |
| AS | Access Switch | MP | Merge Point device (DCF) |
| ALM | Optical Advanced Link Monitoring | MR | Multicast router |
| BK | Backup Switch | MS | Management Switch |
| **BR** | **Branch office router** | MX | Matrix Switch |
| **CL** | **Core Layer device** | OL | Optical Layer Switch |
| Corvil | Corvil application monitor | PM | Netscout polling machines |
| **DL** | **Distribution layer device** | SF | ServerFarm Switch |
| ER | Extranet router | SW | ServerFarm Switch in COLO |
| ES | Extranet Switch | SNF | Sniffer |
| GS | Global Trading System (3rd party exchange switches) | SRV | Server |
| IA | Internet Application Layer Switch | **TL** | **Transport layer device** |
| IF | Internet Front End Switch | UAT | User Acceptance Testing / Development |
| IL | ICP2 Loadbalancer | VC | Video Conference |
| IP | Internet Presentation Layer Switch | VDC | Virtual Data Center / Staging |
| IR | Internet Router Edge / ISP router | VG | Voice Gateway |
| | | VS | Voice Switch |

**Watched device types: TL, CL, DL (and BR where a site still has one).** All others are ignored.

---

## 4. WAN design (how circuits and OSPF relate)

```
 SITE (e.g. site MCCC300)                               DATA CENTER (NY5 / SDR / Ashburn)

 mccc300cl0201 ── mccc300tl.. ══ circuit A (carrier) ══ uaaa100tl.. ── uaaa100dl01..09
   (CL: OSPF)     (TL: circuit)  ══ circuit B ═════════  (TL: circuit)   (DL: OSPF)
        └──────────────── OSPF adjacency CL ↔ DL ─────────────────────────┘
```

- **Circuits** terminate on the **TL** (site and DC). TL interface up/down = circuit status.
- **OSPF** runs between the site **CL** (behind its TL) and the DC **DL** (behind the DC TL).
- Site MCCC300 was recently upgraded: no BR any more, now TL + CL.
- Interface descriptions on DC, branch and campus devices include **carrier and circuit ID**.
- Status rules:
  - TL interface down → **INTF DOWN** (OSPF expected down too)
  - TL interface up, CL↔DL adjacency down → **OSPF DOWN**
  - ≥3 drops in 10 minutes → **FLAPPING**
  - all paths to a site down → **SITE ISOLATED**

---

## 5. Devices

### Data centers

| DC | Site code | Known devices |
|---|---|---|
| NY5 | `UAAA100` (also `UAAA101` seen) | `uaaa100tl0101`, `uaaa100dl0101` … `uaaa100dl0109`, plus `ag`/`sf` (ignored) |
| SDR | `UBBB200` | `ubbb200dl0209`, plus `ag`/`sf`/`vs` (ignored) |
| Ashburn | `UASH???` | TBD |

### Sites

| Site | Site code | Known devices |
|---|---|---|
| Site C | `MCCC300` | `mccc300cl0201` (CL, OSPF), TL TBD, `mccc300as0201` (ignored) |
| Site D | `CDDD400` | `cddd400cl3101` (example from naming standard) |

### All hostnames seen in the log samples (21 Oct 2026)

| Site code | Hostnames | Watched? |
|---|---|---|
| UAAA100 | uaaa100dl0109 | **DL — yes** |
| UBBB200 | ubbb200dl0209 | **DL — yes** |
| MCCC300 | mccc300cl0201 | **CL — yes** |
| UAAA100 | uaaa100ag0102, uaaa100ag0104, uaaa100sf0101, uaaa100sf0102, uaaa100sf0106, uaaa100sf0124, uaaa100sf0127, uaaa100sf0129, uaaa100sf0190 | no |
| UAAA101 | uaaa101sf0162 | no |
| UBBB200 | ubbb200ag0903, ubbb200ag0904, ubbb200sf0220, ubbb200sf0221, ubbb200sf0242, ubbb200sf0262, ubbb200sf0922, ubbb200sf0923, ubbb200sf0926, ubbb200sf0927, ubbb200sf0928, ubbb200sf0929, ubbb200vs0204 | no |
| MCCC300 | mccc300as0201 | no |
| CEEE500 | ceee500as0402, ceee500as0901, ceee500as1002, ceee500as1101, ceee500as1402, ceee500as1901 | no |
| CFFF600 | cfff600as0902 | no |
| CGGG700 | cggg700as1201 | no |
| BHHH800 | bhhh800as0802 | no |
| UIII900 | uiii900as0302, uiii900as0804, uiii900as0901, uiii900vs0303 | no |
| UIII901 | uiii901as3001 | no |
| UJJJ110 | ujjj110as1001 | no |
| UKKK120 | ukkk120es0202 | no |
| ULLL130 | ulll130as0302 | no |

---

## 6. Commands

### On the syslog server

Live view (follows the file across rotation):
```bash
tail -F log
```

WAN interface/OSPF events from watched device types only (today):
```bash
grep -E '%(LINK-3-UPDOWN|LINEPROTO-5-UPDOWN|OSPF-5-ADJCHG|OSPF-5-ADJCHANGE|ETHPORT-5-IF_DOWN|ETHPORT-5-IF_UP)' log \
 | awk '{h=$4; sub(/\..*/,"",h)} h ~ /^[a-z][a-z][a-z][a-z][0-9][0-9][0-9](br|cl|tl|dl)[0-9][0-9][0-9][0-9]$/' \
 | tail -100
```

OSPF changes over the last week of rotated files:
```bash
zcat log202610{14..20}.gz | grep -E '%OSPF-5-ADJCH' \
 | awk '{h=$4; sub(/\..*/,"",h)} h ~ /^[a-z][a-z][a-z][a-z][0-9][0-9][0-9](br|cl|tl|dl)[0-9][0-9][0-9][0-9]$/' \
 | tail -100
```

Which WAN devices send syslog, with message counts:
```bash
awk '{h=$4; sub(/\..*/,"",h)} h ~ /^[a-z][a-z][a-z][a-z][0-9][0-9][0-9](br|cl|tl|dl)[0-9][0-9][0-9][0-9]$/ {print h}' log \
 | sort | uniq -c | sort -rn
```

### On the devices (to build the circuit inventory)

| Device | Commands | Gives |
|---|---|---|
| Site TL (e.g. MCCC300 TL) | `show interfaces description` | Circuit interfaces, carrier, circuit ID |
| Site CL (`mccc300cl0201`) | `show ip ospf neighbor`, `show ip ospf interface brief` | OSPF neighbors (DC DLs), OSPF interfaces |
| DC TL (`uaaa100tl0101`) | `show interfaces description` | DC side of each circuit |
| DC DL (`uaaa100dl01xx`) | `show ip ospf neighbor` (NX-OS: `show ip ospf neighbors`) | OSPF neighbors (site CLs) |

---

## 6a. Real WAN events (21 Oct 2026, filtered with the grep/awk above)

The filter works: only interface and OSPF changes from BR/TL/CL/DL devices come through. Hostnames and IPs below are placeholders.

**OSPF-only flap on two BR routers at one location (13:25:10–13:25:24).** No interface went down, so the circuits stayed up while OSPF re-formed. This is the "circuit up but not passing OSPF" case.

```
Oct 21 13:25:10 ummm140br0302.us.corp.example Oct 21 2026 13:25:09.843 EDST: %OSPF-5-ADJCHG: Process 106, Nbr 10.0.71.23 on GigabitEthernet0/0/0.106 from INIT to 2WAY, 2-Way Received
Oct 21 13:25:10 ummm140br0302.us.corp.example Oct 21 2026 13:25:10.066 EDST: %OSPF-5-ADJCHG: Process 104, Nbr 10.0.69.23 on GigabitEthernet0/0/0.104 from DOWN to INIT, Received Hello
Oct 21 13:25:10 ummm140br0302.us.corp.example Oct 21 2026 13:25:10.068 EDST: %OSPF-5-ADJCHG: Process 104, Nbr 10.0.69.23 on GigabitEthernet0/0/0.104 from EXSTART to EXCHANGE, Negotiation Done
Oct 21 13:25:11 ummm140br0302.us.corp.example Oct 21 2026 13:25:10.070 EDST: %OSPF-5-ADJCHG: Process 104, Nbr 10.0.69.23 on GigabitEthernet0/0/0.104 from LOADING to FULL, Loading Done
Oct 21 13:25:11 ummm140br0302.us.corp.example Oct 21 2026 13:25:11.421 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.64.23 on GigabitEthernet0/0/0.5 from DOWN to INIT, Received Hello
Oct 21 13:25:12 ummm141br0802.us.corp.example Oct 21 2026 13:25:12.640 EDST: %OSPF-5-ADJCHG: Process 103, Nbr 10.0.68.20 on GigabitEthernet0/0/0.103 from INIT to 2WAY, 2-Way Received
Oct 21 13:25:23 ummm141br0802.us.corp.example Oct 21 2026 13:25:22.235 EDST: %OSPF-5-ADJCHG: Process 103, Nbr 10.0.68.20 on GigabitEthernet0/0/0.103 from LOADING to FULL, Loading Done
```

**Short interface flaps on TL devices (7–8 seconds each):**

```
Oct 21 13:28:51 cfff601tl0101.us.corp.example Oct 21 2026 13:28:51.202 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/11, changed state to up
Oct 21 14:19:38 uooo150tl0302.us.corp.example Oct 21 2026 14:19:38.393 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/0, changed state to down
Oct 21 14:19:40 uooo150tl0302.us.corp.example Oct 21 2026 14:19:39.093 EDST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/0, changed state to down
Oct 21 14:19:45 uooo150tl0302.us.corp.example Oct 21 2026 14:19:45.393 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/0, changed state to up
Oct 21 14:19:47 uooo150tl0302.us.corp.example Oct 21 2026 14:19:46.094 EDST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/0, changed state to up
Oct 21 16:16:15 uppp160tl0701.us.corp.example Oct 21 2026 16:16:15.098 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/8, changed state to down
Oct 21 16:16:23 uppp160tl0701.us.corp.example Oct 21 2026 16:16:23.095 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/8, changed state to up
Oct 21 17:18:16 uppp160tl0701.us.corp.example Oct 21 2026 17:18:16.381 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/12, changed state to down
Oct 21 17:18:24 uppp160tl0701.us.corp.example Oct 21 2026 17:18:24.403 EDST: %LINK-3-UPDOWN: Interface GigabitEthernet0/0/12, changed state to up
```

### What these logs mean for the design

1. **Some sites still run on BR routers.** The BR holds both the circuit and the OSPF adjacencies.
2. **One OSPF adjacency per VRF, on a subinterface of the circuit.** On a BR, `GigabitEthernet0/0/0.100`…`.107` each run their own OSPF process (100–107), and `.5` runs process 7. That is about 9 adjacencies per circuit.
   - Subinterface `Gi0/0/0.104` is mapped to its parent circuit `Gi0/0/0`.
   - The circuit's OSPF is **Full only when every VRF adjacency on it is Full**. Otherwise it shows e.g. "OSPF DOWN 3/9 VRF".
   - A flap counts as **one** drop per circuit, not one per VRF.
3. **The recovery sequence is DOWN → INIT → 2WAY → EXSTART → EXCHANGE → LOADING → FULL.** Only `→ FULL` turns a VRF green, and only `FULL →` or `→ DOWN` turns it red. The intermediate steps are shown in the log but change nothing. An adjacency stuck in EXSTART/2WAY (for example an MTU mismatch) stays red.
4. **Volume during a flap.** One circuit with 9 VRFs produces ~50+ lines within a few seconds. The dashboard groups them into one circuit event.
5. **TL flaps are short (7–8 s).** They still count toward flapping (3+ drops in 10 minutes).

## 6b. OSPF history, 14–20 Oct 2026 (`zcat log202610{14..20}.gz | grep %OSPF-5-ADJCH | awk …`)

Placeholders for hostnames and IPs as above.

**CL devices: one OSPF process per VRF on `TwentyFiveGigE` subinterfaces, plus process 7 on `.5`:**
```
Oct 17 17:11:35 uiii902cl0202.us.corp.example Oct 17 2026 17:11:35.947 EDST: %OSPF-5-ADJCHG: Process 101, Nbr 10.0.181.35 on TwentyFiveGigE1/0/3.101 from INIT to 2WAY, 2-Way Received
Oct 17 17:11:35 uiii902cl0202.us.corp.example Oct 17 2026 17:11:35.954 EDST: %OSPF-5-ADJCHG: Process 101, Nbr 10.0.181.35 on TwentyFiveGigE1/0/3.101 from LOADING to FULL, Loading Done
Oct 17 17:11:36 uiii902cl0202.us.corp.example Oct 17 2026 17:11:36.854 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.248.141 on TwentyFiveGigE1/0/3.5 from INIT to 2WAY, 2-Way Received
```

**CL pair at one site: OSPF on an SVI (`Vlan2108`) and on a subinterface:**
```
Oct 17 17:54:52 uqqq170cl0601.us.corp.example Oct 17 2026 17:54:52.594 EDST: %OSPF-5-ADJCHG: Process 108, Nbr 10.0.91.44 on Vlan2108 from LOADING to FULL, Loading Done
Oct 17 17:54:53 uqqq170cl0602.us.corp.example Oct 17 2026 17:54:52.598 EDST: %OSPF-5-ADJCHG: Process 108, Nbr 10.0.91.43 on Vlan2108 from LOADING to FULL, Loading Done
Oct 17 17:59:48 uqqq170cl0601.us.corp.example Oct 17 2026 17:59:47.509 EDST: %OSPF-5-ADJCHG: Process 108, Nbr 10.0.91.4 on TwentyFiveGigE1/0/1.108 from LOADING to FULL, Loading Done
```

**TL devices run OSPF process 7 directly on the circuit interface:**
```
Oct 17 19:10:07 cfff601tl0101.us.corp.example Oct 17 2026 19:10:07.205 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.104.49 on TenGigabitEthernet0/0/25 from FULL to DOWN, Neighbor Down: Dead timer expired
Oct 17 19:10:07 cfff602tl0701.us.corp.example Oct 17 2026 19:10:07.309 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.104.51 on TenGigabitEthernet0/0/25 from FULL to INIT, 1-Way
Oct 17 19:10:08 cfff601tl0101.us.corp.example Oct 17 2026 19:10:07.334 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.104.49 on TenGigabitEthernet0/0/25 from LOADING to FULL, Loading Done
Oct 18 04:56:16 mccc301tl1001.us.corp.example Oct 18 2026 04:56:16.063 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.249.98 on GigabitEthernet0/0/0 from LOADING to FULL, Loading Done
Oct 18 04:56:17 mccc302tl0101.us.corp.example Oct 18 2026 04:56:16.058 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.249.100 on GigabitEthernet0/0/0 from LOADING to FULL, Loading Done
```

**A full outage at one site (19 Oct 20:23), then the DC TLs dropping it about a minute later:**
```
Oct 19 20:23:05 urrr180cl1001.us.corp.example Oct 19 2026 20:23:05.937 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.248.140 on TwentyFiveGigE1/0/3.5 from FULL to DOWN, Neighbor Down: Interface down or detached
Oct 19 20:23:05 urrr180cl1001.us.corp.example Oct 19 2026 20:23:05.937 EDST: %OSPF-5-ADJCHG: Process 100, Nbr 10.0.181.1 on TwentyFiveGigE1/0/3.100 from FULL to DOWN, Neighbor Down: Interface down or detached
   … processes 101–107 on .101–.107, same second …
Oct 19 20:23:06 urrr181cl1001.us.corp.example Oct 19 2026 20:23:05.939 EDST: %OSPF-5-ADJCHG: Process 108, Nbr 10.0.91.4 on TwentyFiveGigE1/0/3.108 from FULL to DOWN, Neighbor Down: Interface down or detached
Oct 19 20:24:03 usss190tl0102.us.corp.example Oct 19 2026 20:24:02.717 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.249.80 on TenGigabitEthernet0/0/10 from FULL to DOWN, Neighbor Down: Dead timer expired
Oct 19 20:24:03 uiii900tl0201.us.corp.example Oct 19 2026 20:24:02.952 EDST: %OSPF-5-ADJCHG: Process 7, Nbr 10.0.249.253 on TenGigabitEthernet0/1/0 from FULL to DOWN, Neighbor Down: Dead timer expired
```

### What this adds to the design

1. **Two OSPF layers.**
   - **Transport:** process 7, TL ↔ TL, directly on the circuit interface (`Gi0/0/0`, `Te0/0/25`, `Te0/0/10`). This is the most direct "is the circuit passing OSPF" signal.
   - **Service:** processes 100–108 per VRF on the CL (`TwentyFiveGigE1/0/3.100`…`.108`, or SVIs like `Vlan2108`), plus process 7 on `.5`.
2. **Interface names normalised:** `TwentyFiveGigE` ↔ `Twe`, `Vlan` ↔ `Vl`. Subinterfaces map to their parent.
3. **`FULL to INIT, 1-Way` counts as down**, same as `FULL to DOWN`.
4. **DC-side detection lags.** "Interface down or detached" is instant at the site. The DC TLs only log "Dead timer expired" ~40–60 s later. The dashboard should go red on whichever end reports first.
5. **The Ashburn site code appears in the logs** (`usss190` placeholder). **`uiii900tl0201`** (NYC site) is also a TL peering with sites. Is it a hub?

## 7. Open questions

1. One OSPF adjacency **per circuit** (separate VLAN/subinterface per circuit), or one **per site**? If per circuit, which interface type carries it?
2. Ashburn site code (`UASH???`) and the TL hostnames at each DC; one or two TLs per DC?
3. Do all sites have TL + CL now, or do some still have a single BR?
4. Is `UAAA101` part of NY5?
5. Country/city code list for map labels.

## 8. Next steps

1. Get the device outputs in section 6 for one worked example (MCCC300 ↔ NY5).
2. Rebuild the dashboard model around TL (circuit) + CL/DL (OSPF); rename STR → SDR.
3. Write the collector: follow `log`, pre-filter, device-type filter, map to circuits, keep state, rebuild on startup from `log` + `logYYYYMMDD`, push live to the dashboard.
