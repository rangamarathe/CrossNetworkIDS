# Data Dictionary: CICIDS2017 (improved) and CSE-CIC-IDS2018 (improved)

This explains how the two datasets were collected and what one row means (§2), every one of the 91 columns and how it is computed (§3), data quality (§4), and how 2017 and 2018 differ (§5). For the attack labels themselves, see [ATTACK_LABELS.md](ATTACK_LABELS.md).

## 1. Source and files

Both datasets are the **error-corrected re-releases by DistriNet (KU Leuven)**, not the original CIC CSVs. DistriNet re-ran a patched CICFlowMeter over the original PCAPs and relabelled every flow:

- CICIDS2017 improved: Engelen, Rimmer & Joosen, *Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study*, IEEE Security & Privacy Workshops (WTMC) 2021.
- CSE-CIC-IDS2018 improved: Liu, Engelen, Lynar, Essam & Joosen, *Error Prevalence in NIDS datasets: A Case Study on CIC-IDS-2017 and CSE-CIC-IDS-2018*, IEEE CNS 2022.
- Download, labelling logic and tool changes: <https://intrusion-detection.distrinet-research.be/CNS2022/>

Because of this, column names, row counts and label names **do not match** the original CIC releases (78 to 84 columns, `Label` values like `DoS attacks-Hulk`). Results from papers that used the originals are not directly comparable.

| Dataset | Folder | Files | Rows | Size on disk |
|---|---|---|---|---|
| CICIDS2017 improved | `CICIDS2017_improved/` | 5 (`monday.csv` … `friday.csv`, 3 to 7 July 2017) | 2,099,976 | 1.15 GB |
| CSE-CIC-IDS2018 improved | `CSECICIDS2018_improved/` | 10 (`Wednesday-14-02-2018.csv` … `Friday-02-03-2018.csv`) | 63,195,145 | 36.0 GB |

Rows per file:

| 2017 file | Rows | 2018 file | Rows |
|---|---|---|---|
| monday.csv | 371,624 | Wednesday-14-02-2018.csv | 5,898,350 |
| tuesday.csv | 322,078 | Thursday-15-02-2018.csv | 5,410,102 |
| wednesday.csv | 496,641 | Friday-16-02-2018.csv | 7,390,266 |
| thursday.csv | 362,076 | Tuesday-20-02-2018.csv | 6,054,702 |
| friday.csv | 547,557 | Wednesday-21-02-2018.csv | 6,962,593 |
| | | Thursday-22-02-2018.csv | 6,071,153 |
| | | Friday-23-02-2018.csv | 5,976,481 |
| | | Wednesday-28-02-2018.csv | 6,568,726 |
| | | Thursday-01-03-2018.csv | 6,551,401 |
| | | Friday-02-03-2018.csv | 6,311,371 |

The project notebooks read these from an `archive (1)/` folder; the same files are in `Data/archive/` here.

### How this was checked (5 Oct 2026)

- All 15 files have a byte-identical header (91 columns, same order).
- `Label` and `Attempted Category` were counted over **every row of both datasets** (the 2018 counts match `label_counts_2018.csv`).
- Per-column statistics: **all 2.1M rows of 2017**, and a **deterministic 1% sample of 2018** (631,947 rows, every file, spread across the whole day). 2018 figures below are from that sample unless stated.
- Exact-equality checks between redundant columns were run on the full 2017 data.

## 2. How the data was collected and what a row is

### 2.1 Pipeline

```
 lab network ──► raw packets (PCAP) ──► flow meter ──► one row per flow ──► label
 (benign users      captured per day      CICFlowMeter     ~80 statistics      by IP + time
  + attackers)                            (DistriNet fix)                      + payload check)
```

1. **Generate traffic.** Scripted "users" produce benign traffic while a red team runs scheduled attacks.
2. **Capture packets.** Every packet is recorded to PCAP files, one set per day.
3. **Build flows.** CICFlowMeter (a Java tool from CIC) reads the PCAPs, groups packets into bidirectional flows by 5-tuple and computes statistics for each flow. These statistics are the columns.
4. **Label.** Each flow is matched to the attack schedule by attacker/victim IP and time window. DistriNet's relabelling also checks the content, which is how the `- Attempted` labels arise.

The CSVs never contain payload or packet contents. A model sees only the statistics in §3.

### 2.2 CICIDS2017: on-premises lab

| Item | Detail (CIC unless noted) |
|---|---|
| Who built it | Canadian Institute for Cybersecurity (CIC), University of New Brunswick |
| When | 9 a.m. Monday 3 July to 5 p.m. Friday 7 July 2017, working hours only |
| Network | A victim network (firewall, switches, routers, web servers, Windows 7/8.1/10/Vista, Ubuntu and Mac OS X clients) and a separate attacker network (Kali Linux plus Windows machines) |
| Benign traffic | **B-Profile** system: models of 25 users' behaviour over HTTP, HTTPS, FTP, SSH and email, replayed by agents to give realistic background traffic |
| Attack traffic | Open-source tools run from the attacker network on a published schedule (see ATTACK_LABELS.md) |
| Capture | Full packet capture of the victim network's traffic, 5 daily PCAPs |
| Flow extraction | Original release: CICFlowMeter, "more than 80" features. **These files:** DistriNet's corrected CICFlowMeter, 91 columns |

### 2.3 CSE-CIC-IDS2018: AWS

| Item | Detail (CIC unless noted) |
|---|---|
| Who built it | CIC together with the Communications Security Establishment (CSE), Canada |
| When | 10 weekdays between Wed 14 Feb and Fri 2 Mar 2018 |
| Network | Amazon Web Services. A simulated organisation of 5 departments (R&D, Management, Technician, Secretary/Operations, IT) plus a server room: 420 machines and 30 servers running Windows 8.1/10, Windows Server 2012/2016 and Ubuntu. The attacking side had 50 Kali Linux machines. |
| Benign traffic | **B-Profiles** learned from real users: distributions of packet sizes, packets per flow, payload patterns and request timing per protocol, replayed by agents |
| Attack traffic | **M-Profiles**: structured descriptions of seven attack scenarios, executed on a schedule |
| Capture | PCAPs **per machine**, plus Windows and Ubuntu event logs (the logs are not in these CSVs) |
| Flow extraction | Original release: CICFlowMeter-V3, about 80 features. **These files:** DistriNet's corrected CICFlowMeter, 91 columns |

2018 captured traffic on every machine of a much larger network (450 hosts vs. CIC's 12 machines for 2017), which is why it has 30× more rows.

### 2.4 What one row is

A row is **one bidirectional flow**: all packets sharing the same source IP, destination IP, source port, destination port and protocol, in both directions, until the flow ends. In the corrected tool a flow ends when:

- the TCP connection closes properly (both FINs, or an RST), or
- **120 s** have passed since the flow started; the connection continues as a new row with the same `Flow ID`.

UDP and ICMP have no connection, so their flows end only by timeout. In practice one DNS lookup, one HTTP connection or one SSH session is usually one row; a long download becomes several rows.

### 2.5 What DistriNet changed in the flow meter

Engelen et al. (2021) found that the original CICFlowMeter built many flows wrongly. The corrected tool used for these files:

- **Ends a TCP flow only when the connection really ends.** The original ended a flow at the *first* FIN, so the leftover ACK/FIN packets became a separate tiny flow (a "TCP appendix") carrying the same label. These appendices were **25.9% of the original 2017 dataset**.
- **Treats RST as ending a flow.** The original ignored RST.
- **Fixes TCP segmentation offload** packets that were split or counted wrongly.
- **Writes timestamps in UTC with microsecond precision.**
- **Adds 5 features:** `ICMP Code`, `ICMP Type`, `Total TCP Flow Time`, `Fwd RST Flags` and `Bwd RST Flags`. The relabelling also adds the `Attempted Category` column.

This is why these files have fewer rows than the original CSVs (2017 original: about 2.8M rows; improved: 2.1M).

### 2.6 Conventions used in the columns

- **One row = one bidirectional flow**, built by CICFlowMeter from the 5-tuple (src IP, dst IP, src port, dst port, protocol). "Forward" (Fwd) is the direction of the first packet seen (normally the client); "Backward" (Bwd) is the reply direction.
- **Flow timeout is 120 s** (`Flow Duration` never exceeds 119,999,998 µs). A long connection is split into several rows that share a `Flow ID`.
- **All times and durations are in microseconds (µs)**. Rates are per second.
- **Lengths are in bytes.** "Packet length" means payload bytes (TCP/UDP data), not the full frame, which is why many minima are 0.
- `-1` is a sentinel for "not applicable" in `ICMP Code`, `ICMP Type` and `Attempted Category`. Remember this before scaling or taking logs.
- `Timestamp` is UTC with microsecond precision (a DistriNet change). The CIC attack schedules on the UNB pages are in local lab time, so they are offset by several hours from these timestamps (2017 traffic starts at 11:55 UTC, about 08:55 lab time).

## 3. Column reference

Every column, in file order within each group. The **#** is the column position in the CSV (1 to 91).

- **Direction:** "Fwd" means packets from `Src IP` (whoever sent the first packet, normally the client); "Bwd" means packets from `Dst IP` back to it.
- **Medians:** 2017 is computed over all rows; 2018 over a 1% sample of every file. Medians are dominated by BENIGN traffic, especially DNS.
- **Role:** **ID** = identifier, drop before modelling; **F** = feature; **F\*** = feature with a caveat in the notes; **Label** = target.
- **Same in both years:** the computation of every column is identical in 2017 and 2018 (same DistriNet tool). Differences are in the *values*, flagged under "Notes and drift"; the full comparison is in §5.

### 3.1 Identifiers and metadata

These say *which* conversation a row is, not *how* it behaved. Drop them before modelling (except `Dst Port` and `Protocol`, which are judgment calls).

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 1 | `id` | — | Row counter written by the DistriNet pipeline. Starts at 1 in each CSV. | 210k / 3.2M | Not unique across files (2017 max 547,557 for 2.1M rows). Useless as a feature. | ID |
| 2 | `Flow ID` | — | Text key `SrcIP-DstIP-SrcPort-DstPort-Protocol` built from the first packet's direction. | — | Not unique: one long connection that times out every 120 s gives several rows with the same Flow ID (2017: 1.08M distinct for 2.10M rows). | ID |
| 3 | `Src IP` | — | IPv4 source address of the **first packet** of the flow. That host defines the "forward" direction for every Fwd/Bwd column. | — | Pure leakage: each attack comes from fixed attacker IPs. 2017 is a /24 lab plus NAT (174 distinct); 2018 is AWS `172.31.0.0/16` plus public attacker IPs (26k distinct in a 1% sample). | ID |
| 4 | `Src Port` | — | TCP/UDP source port of the first packet. Usually an ephemeral port chosen by the client's OS. 0 for ICMP and other protocols. | 53k / 53k | No semantic meaning; dropped in notebook 02. | ID |
| 5 | `Dst IP` | — | IPv4 destination address of the first packet: normally the server or victim. | — | Leakage, as for Src IP. | ID |
| 6 | `Dst Port` | — | TCP/UDP destination port of the first packet, i.e. the service being contacted (53 DNS, 80 HTTP, 443 HTTPS, 21 FTP, 22 SSH …). 0 for ICMP. | 80 / 80 | Strongest single shortcut: every attack except the port scans uses exactly one port (see ATTACK_LABELS §5.1). It is a category, not a number, so treat it that way if you keep it. The most common benign ports are 53 and 443 in both years. | F* |
| 7 | `Protocol` | — | IP protocol number from the IP header: 6 = TCP, 17 = UDP, 1 = ICMP, 0 = HOPOPT/other. | 6 / 6 | TCP / UDP split is 52/48% (2017) vs 60/39% (2018 sample). Only these 4 values occur. | F |
| 8 | `Timestamp` | UTC | Arrival time of the first packet, `YYYY-MM-DD HH:MM:SS.ffffff`, in UTC (DistriNet change; the original release used local time at lower precision). | — | Only for splitting or ordering. 2017 runs 2017-07-03 11:55 → 07-07 20:02 UTC; 2018 runs 2018-02-14 → 2018-03-03 00:31 UTC. | ID |

### 3.2 Flow size and duration

Counts and totals over the whole flow. *Payload* means TCP/UDP data bytes, excluding IP and TCP/UDP headers.

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 9 | `Flow Duration` | µs | Timestamp of the last packet − timestamp of the first packet, in both directions. | 62k / 139k | Capped near 120,000,000 µs by the flow timeout. A single-packet flow has duration 0, which is what produces the `inf` rates. | F |
| 10 | `Total Fwd Packet` | packets | Number of packets sent in the forward direction (from Src IP). | 2 / 5 |  | F |
| 11 | `Total Bwd packets` | packets | Number of packets sent in the backward direction (from Dst IP back to Src IP). | 2 / 4 | 0 means the other side never answered (scans, floods against dead hosts, one-way UDP). | F |
| 12 | `Total Length of Fwd Packet` | bytes | Sum of payload bytes over all forward packets. | 80 / 97 | 0 = the client sent no data at all. DistriNet uses this to find `Attempted` flows (code 0). | F |
| 13 | `Total Length of Bwd Packet` | bytes | Sum of payload bytes over all backward packets. | 218 / 228 | 2017 mean is 3× larger than 2018 (more large downloads in the lab). | F |
| 89 | `Total TCP Flow Time` | µs | **DistriNet feature.** Time from the first to the last packet of the *whole TCP connection*, even when the connection is split into several 120 s rows. 0 for UDP/ICMP. | 27 / 154k | Exceeds `Flow Duration` for long connections (max about 30,290 s in 2017). Highlights persistent sessions such as C&C or Heartbleed. Median 27 µs (2017) vs 154 ms (2018). | F |

### 3.3 Packet payload size statistics

Statistics over the payload length of each packet. Packets with no payload (pure ACK, SYN) count as length 0, which is why the minimum is usually 0. "Std" is the standard deviation over the packets in that direction (0 when there is only one packet).

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 14 | `Fwd Packet Length Max` | bytes | Largest forward payload in the flow. | 45 / 74 |  | F |
| 15 | `Fwd Packet Length Min` | bytes | Smallest forward payload, often 0 (SYN or ACK). | 0 / 0 | 0 in 52% (2017) and 61% (2018) of rows. | F |
| 16 | `Fwd Packet Length Mean` | bytes | `Total Length of Fwd Packet` / `Total Fwd Packet`. | 40.7 / 45 |  | F |
| 17 | `Fwd Packet Length Std` | bytes | Standard deviation of forward payload sizes. | 0 / 23.1 |  | F |
| 18 | `Bwd Packet Length Max` | bytes | Largest backward payload. | 128 / 155 |  | F |
| 19 | `Bwd Packet Length Min` | bytes | Smallest backward payload. | 0 / 0 |  | F |
| 20 | `Bwd Packet Length Mean` | bytes | `Total Length of Bwd Packet` / `Total Bwd packets`. | 114 / 118 |  | F |
| 21 | `Bwd Packet Length Std` | bytes | Standard deviation of backward payload sizes. | 0 / 20.6 |  | F |
| 48 | `Packet Length Min` | bytes | Smallest payload over **both** directions. | 0 / 0 |  | F |
| 49 | `Packet Length Max` | bytes | Largest payload over both directions. | 130 / 164 |  | F |
| 50 | `Packet Length Mean` | bytes | Mean payload over both directions. | 81 / 79 |  | F |
| 51 | `Packet Length Std` | bytes | Standard deviation of payload over both directions. | 52 / 72.8 |  | F |
| 52 | `Packet Length Variance` | bytes² | Variance of payload over both directions. | 2,700 / 5,299 | = `Packet Length Std`² in every row. Redundant. | F* |
| 62 | `Average Packet Size` | bytes | Meant as the mean packet size over the flow. | 81 / 79 | Identical to `Packet Length Mean` in every 2017 row. Redundant. | F* |
| 63 | `Fwd Segment Size Avg` | bytes | Meant as the average forward TCP segment size. | 40.7 / 45 | Identical to `Fwd Packet Length Mean`. Redundant. | F* |
| 64 | `Bwd Segment Size Avg` | bytes | Meant as the average backward TCP segment size. | 114 / 118 | Identical to `Bwd Packet Length Mean`. Redundant. | F* |

### 3.4 Rates

Each rate is a total divided by `Flow Duration` converted to seconds. If the duration is 0, the rate is infinite.

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 22 | `Flow Bytes/s` | bytes/s | (fwd + bwd payload bytes) / duration in seconds. | 3,864 / 2,178 | `inf` in 5 rows of 2017 (zero-duration BENIGN flows). Verified against the formula in every row. | F |
| 23 | `Flow Packets/s` | packets/s | (fwd + bwd packets) / duration in seconds. | 73.7 / 45.5 | Same `inf` rows. Floods and scans have very high values. | F |
| 46 | `Fwd Packets/s` | packets/s | Forward packets / duration in seconds. | 38.8 / 23.4 |  | F |
| 47 | `Bwd Packets/s` | packets/s | Backward packets / duration in seconds. | 32.6 / 20.6 |  | F |
| 61 | `Down/Up Ratio` | ratio | `Total Bwd packets` / `Total Fwd Packet`, as packet counts (not bytes). 0 if there are no forward packets. | 1 / 1 | Verified exactly. Around 1 for normal request-response; 0 for one-way traffic. | F |

### 3.5 Inter-arrival times (IAT)

An IAT is the gap between two consecutive packets. *Flow IAT* uses all packets in time order; *Fwd/Bwd IAT* only the packets of one direction. Each needs at least 2 packets; otherwise the value is 0.

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 24 | `Flow IAT Mean` | µs | Mean gap between consecutive packets of the flow (either direction). | 19k / 31k |  | F |
| 25 | `Flow IAT Std` | µs | Standard deviation of those gaps. Low values mean machine-regular timing. | 16k / 34k | Low std is a typical bot or flood signal. | F |
| 26 | `Flow IAT Max` | µs | Longest gap. | 51k / 95k | Always ≤ `Flow Duration`. Slow DoS attacks show gaps of several seconds. | F |
| 27 | `Flow IAT Min` | µs | Shortest gap. | 3 / 38 |  | F |
| 28 | `Fwd IAT Total` | µs | Sum of the gaps between forward packets = time from the first to the last forward packet. | 48 / 112k | Median 48 µs (2017) vs 112 ms (2018): a big shift. | F |
| 29 | `Fwd IAT Mean` | µs | Mean gap between forward packets. | 48 / 38k |  | F |
| 30 | `Fwd IAT Std` | µs | Standard deviation of forward gaps. | 0 / 18k |  | F |
| 31 | `Fwd IAT Max` | µs | Longest gap between forward packets. | 48 / 79k |  | F |
| 32 | `Fwd IAT Min` | µs | Shortest gap between forward packets. | 3 / 18 |  | F |
| 33 | `Bwd IAT Total` | µs | Time from the first to the last backward packet. | 48 / 77k |  | F |
| 34 | `Bwd IAT Mean` | µs | Mean gap between backward packets. | 48 / 24k |  | F |
| 35 | `Bwd IAT Std` | µs | Standard deviation of backward gaps. | 0 / 484 |  | F |
| 36 | `Bwd IAT Max` | µs | Longest gap between backward packets. | 48 / 48k |  | F |
| 37 | `Bwd IAT Min` | µs | Shortest gap between backward packets. | 3 / 1 |  | F |

### 3.6 TCP flags

Counts of packets with a given TCP header flag set; always 0 for UDP and ICMP. Flag meanings: **SYN** opens a connection, **ACK** acknowledges data, **PSH** asks the receiver to pass data to the application straight away, **FIN** closes politely, **RST** aborts, **URG** marks urgent data (practically unused), and **CWR/ECE** are used for Explicit Congestion Notification (ECN).

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 38 | `Fwd PSH Flags` | packets | Forward packets with PSH set. | 0 / 1 |  | F |
| 39 | `Bwd PSH Flags` | packets | Backward packets with PSH set. | 0 / 1 |  | F |
| 40 | `Fwd URG Flags` | packets | Forward packets with URG set. | 0 / 0 | Non-zero in < 0.03% of rows. Near-constant. | F* |
| 41 | `Bwd URG Flags` | packets | Backward packets with URG set. | 0 / 0 | Non-zero in only a handful of 2017 rows; constant 0 in the 2018 sample. | F* |
| 42 | `Fwd RST Flags` | packets | **DistriNet feature.** Forward packets with RST set: the client aborted. | 0 / 0 |  | F |
| 43 | `Bwd RST Flags` | packets | **DistriNet feature.** Backward packets with RST set: the server refused or aborted. Closed-port probes produce exactly this. | 0 / 0 | Key to `Attempted` code 1 (port closed) and to scans. | F |
| 53 | `FIN Flag Count` | packets | Packets with FIN in both directions. A clean close has 2. | 0 / 0 |  | F |
| 54 | `SYN Flag Count` | packets | Packets with SYN in both directions (SYN + SYN-ACK = 2 for a normal open). | 0 / 2 |  | F |
| 55 | `RST Flag Count` | packets | Packets with RST in both directions. | 0 / 0 | = Fwd + Bwd RST Flags in 99.9% of rows. | F |
| 56 | `PSH Flag Count` | packets | Packets with PSH in both directions. | 0 / 2 | = Fwd + Bwd PSH Flags in every row. Redundant. | F* |
| 57 | `ACK Flag Count` | packets | Packets with ACK in both directions; nearly every TCP packet after the first. | 1 / 8 |  | F |
| 58 | `URG Flag Count` | packets | Packets with URG in both directions. | 0 / 0 | = Fwd + Bwd URG. Near-constant. | F* |
| 59 | `CWR Flag Count` | packets | Packets with CWR (Congestion Window Reduced). | 0 / 0 | **Biggest drift**: non-zero in 0.04% of 2017 rows vs 29% of 2018. Most likely the 2018 Windows/Server hosts negotiate ECN (inferred). | F* |
| 60 | `ECE Flag Count` | packets | Packets with ECE (ECN-Echo). | 0 / 0 | Same drift as CWR (0.03% vs 29%). | F* |

### 3.7 Header and TCP-handshake fields

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 44 | `Fwd Header Length` | bytes | Sum of the transport-layer header bytes over all forward packets (20 bytes for a bare TCP header, more with options; 8 for UDP). | 32 / 108 | **Negative in about 0.02% of rows** in both years, most likely a 16-bit overflow (inferred). Median 32 (2017) vs 108 (2018). | F* |
| 45 | `Bwd Header Length` | bytes | Same for backward packets. | 20 / 88 | Same negative-value issue. | F* |
| 75 | `FWD Init Win Bytes` | bytes | TCP receive-window field of the first forward packet (normally the SYN): how much data the client offers to buffer. 0 for non-TCP. | 503 / 8,192 | OS-specific, like a fingerprint. Median 503 (2017) vs 8,192 (2018). | F* |
| 76 | `Bwd Init Win Bytes` | bytes | TCP window of the first backward packet (normally the SYN-ACK). 0 if the server never answered. | 0 / 0 | Mean 754 (2017) vs 10,233 (2018). | F* |
| 77 | `Fwd Act Data Pkts` | packets | Forward packets carrying at least 1 byte of payload. | 1 / 1 | 0 = connection with no data from the client. | F |
| 78 | `Fwd Seg Size Min` | bytes | Despite the name, the **smallest forward header length** seen (CICFlowMeter naming). | 20 / 20 | Values 0 to 44, median 20. | F |

### 3.8 Bulk transfer

CICFlowMeter's "bulk" is a run of at least 4 consecutive data packets in one direction with no data packet from the other side in between, i.e. a one-way transfer burst. The Avg columns divide the bulk totals by the number of bulks; the Rate divides bulk bytes by bulk time.

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 65 | `Fwd Bytes/Bulk Avg` | bytes | Average bytes per forward bulk. | 0 / 0 | Zero in 96% (2017) / 99% (2018) of rows. | F |
| 66 | `Fwd Packet/Bulk Avg` | packets | Average packets per forward bulk. | 0 / 0 |  | F |
| 67 | `Fwd Bulk Rate Avg` | bytes/s | Average forward bulk throughput. | 0 / 0 |  | F |
| 68 | `Bwd Bytes/Bulk Avg` | bytes | Average bytes per backward bulk (downloads). | 0 / 0 | Zero in 86% / 94%. | F |
| 69 | `Bwd Packet/Bulk Avg` | packets | Average packets per backward bulk. | 0 / 0 |  | F |
| 70 | `Bwd Bulk Rate Avg` | bytes/s | Average backward bulk throughput. | 0 / 0 |  | F |

### 3.9 Subflows (unreliable)

Intended as the flow split into sub-flows at idle gaps, with averages per sub-flow. The implementation is known to be buggy, and the values here confirm it.

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 71 | `Subflow Fwd Packets` | packets | Intended: average forward packets per subflow. | 0 / 0 | **Broken**: only 0 or 1. | F* |
| 72 | `Subflow Fwd Bytes` | bytes | Intended: average forward bytes per subflow. | 21 / 22 | Max 1,777 even for MB-sized flows. Unreliable. | F* |
| 73 | `Subflow Bwd Packets` | packets | Intended: average backward packets per subflow. | 0 / 0 | **Broken**: only 0 or 1, and 0 in >99.9% of rows. | F* |
| 74 | `Subflow Bwd Bytes` | bytes | Intended: average backward bytes per subflow. | 56 / 56 | Max 2,090. Unreliable. | F* |

### 3.10 Active and idle periods

CICFlowMeter splits a flow's timeline at gaps longer than an activity threshold (5 s by default in the tool). An *active* period is a stretch of packets without such a gap; an *idle* period is the gap itself. Flows that never pause have 0 in all eight columns (about 78% of rows).

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 79 | `Active Mean` | µs | Mean length of active periods. | 0 / 0 |  | F |
| 80 | `Active Std` | µs | Standard deviation of active-period lengths. | 0 / 0 |  | F |
| 81 | `Active Max` | µs | Longest active period. | 0 / 0 |  | F |
| 82 | `Active Min` | µs | Shortest active period. | 0 / 0 |  | F |
| 83 | `Idle Mean` | µs | Mean length of idle gaps. | 0 / 0 | Slowloris, keep-alive and C&C polling produce regular idle gaps. | F |
| 84 | `Idle Std` | µs | Standard deviation of idle gaps; near 0 means machine-regular pauses. | 0 / 0 |  | F |
| 85 | `Idle Max` | µs | Longest idle gap. | 0 / 0 |  | F |
| 86 | `Idle Min` | µs | Shortest idle gap. | 0 / 0 |  | F |

### 3.11 ICMP (DistriNet features)

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 87 | `ICMP Code` | — | ICMP code field (sub-type) for ICMP flows; `-1` otherwise. | -1 / -1 | -1 in 99.97% (2017) / 99.84% (2018) of rows. Treat -1 as "not ICMP", not as a number. | F* |
| 88 | `ICMP Type` | — | ICMP type: 0 echo reply, 3 destination unreachable, 8 echo request, 11 time exceeded; `-1` for non-ICMP. | -1 / -1 | Type 3 replies are what closed UDP ports send back, which helps with UDP floods and scans. | F* |

### 3.12 Labels

| # | Column | Unit | What it means and how it is computed | Median 2017 / 2018 | Notes and drift | Role |
|---|---|---|---|---|---|---|
| 90 | `Label` | — | Ground-truth class: `BENIGN`, an attack name, or `<attack> - Attempted`. | — | 27 values in 2017, 25 in 2018, with different names in places. See ATTACK_LABELS.md. | Label |
| 91 | `Attempted Category` | — | Why a flow is `Attempted`: 0 no payload, 1 port/system closed, 2 start-up/teardown artefact, 3 no malicious payload, 4 attack artefact, 5 attack implemented incorrectly, 6 target unresponsive. `-1` for every other row. | -1 / -1 | **Leaks the label.** Never use as a feature. | Label |

## 4. Data quality summary

| Check | 2017 (full) | 2018 (1% sample) |
|---|---|---|
| Missing values | none | none |
| `inf` values | 5 rows (`Flow Bytes/s`, `Flow Packets/s`; zero-duration BENIGN flows) | none seen |
| Negative values outside sentinels | `Fwd/Bwd Header Length`, ~0.02% | same, ~0.02% |
| Duplicate rows ignoring `id`, `Flow ID`, `Timestamp`, `Src Port` | 184,651 (8.8%) | not computed |
| Exact copies of another row (excluding `id`) | 0 | not computed |
| Constant or near-constant | `Bwd URG Flags`, `Fwd URG Flags`, `URG Flag Count`, `Subflow Bwd Packets` | same, plus `Bwd URG Flags` fully constant |
| Exact duplicates of another column | `Average Packet Size`, `Fwd/Bwd Segment Size Avg`, `Packet Length Variance` (= Std²), `PSH Flag Count` (= Fwd+Bwd PSH), `URG Flag Count` | same tool, same behaviour |

Notebook 02 removes constant columns with `nunique() <= 1`. On 2017 that removes nothing, because the URG and subflow columns have a handful of non-zero values. Dropping them explicitly is safer.

**Suggested modelling drop list** (in addition to notebook 02's `id, Flow ID, Src IP, Dst IP, Src Port, Timestamp, Attempted Category, Label`): `Average Packet Size`, `Fwd Segment Size Avg`, `Bwd Segment Size Avg`, `Packet Length Variance`, `Fwd URG Flags`, `Bwd URG Flags`, `URG Flag Count`, `Subflow Fwd Packets`, `Subflow Bwd Packets`, `Subflow Fwd Bytes`, `Subflow Bwd Bytes`. This is a recommendation, not something the notebooks currently do.

## 5. Drift between 2017 and 2018

The schema is identical, but the networks, operating systems and attack mix are not. A model trained on 2017 and tested on 2018 meets all of the following. Figures are 2017 (all rows) vs 2018 (1% sample) unless marked.

### 5.1 Environment

| Aspect | CICIDS2017 | CSE-CIC-IDS2018 |
|---|---|---|
| Testbed | On-premises lab at UNB | AWS (VPC `172.31.0.0/16`) |
| Victims | `192.168.10.0/24` (Windows 7/8.1/10/Vista, Ubuntu, one Mac) | `172.31.69.0/24` and others (Windows 8.1/10, Server 2012/2016, Ubuntu); 420 machines, 30 servers |
| Attackers | `205.174.165.73` etc., seen through NAT `172.16.0.1` | CIC documents `172.31.70.x` for brute force and DoS, but **in the flows every attack comes from public AWS addresses**: `18.221.219.4`, `13.58.98.64`, `18.219.211.138`, `18.217.165.70`, `18.218.115.60`, `13.58.225.34` (no `172.31.70.x` appears at all) |
| Distinct `Src IP` | 174 | 26,431 in a 1% sample |
| Days | 5 consecutive weekdays | 10 weekdays over 3 weeks |

IP-based features cannot transfer at all, which is one more reason to drop them.

### 5.2 Scale and class balance (full counts)

| | 2017 | 2018 |
|---|---|---|
| Rows | 2,099,976 | 63,195,145 (30× more) |
| BENIGN | 1,582,566 (75.4%) | 59,353,486 (93.9%) |
| Attack (not attempted) | 505,431 (24.1%) | 3,535,422 (5.6%) |
| `… - Attempted` | 11,979 (0.57%) | 306,237 (0.48%) |

The attack base rate differs by roughly 4×, so precision and any thresholds calibrated on 2017 will not hold on 2018.

### 5.3 Feature distributions

| Feature | 2017 | 2018 | Why it matters |
|---|---|---|---|
| Protocol mix TCP / UDP / ICMP | 52.3% / 47.6% / 0.03% | 60.4% / 39.4% / 0.16% | Shifts every aggregate statistic. |
| `CWR Flag Count` > 0 | 0.04% | 29.2% | **Largest shift.** Most likely the 2018 Windows/Server hosts negotiate ECN while the 2017 hosts almost never do (inferred from the data). A tree model can learn "ECE/CWR = 0" as a 2017-specific benign cue. |
| `ECE Flag Count` > 0 | 0.03% | 29.2% | Same as above. |
| `FWD Init Win Bytes` median | 503 | 8,192 | OS / TCP-stack difference. |
| `Bwd Init Win Bytes` mean | 754 | 10,233 | Same. |
| `Fwd Header Length` median | 32 | 108 | More packets per flow and TCP options. |
| `Flow Duration` median | 61.8 ms | 138.6 ms | |
| `Fwd IAT Total` median | 48 µs | 112.4 ms | |
| `Total TCP Flow Time` median | 27 µs | 153.7 ms | |
| `Fwd Act Data Pkts` mean | 2.2 | 7.5 | |
| `Total Length of Bwd Packet` mean | 21,920 B | 6,545 B | 2017 has more large downloads. |
| Rows with zero forward payload | 14.6% | 6.6% | |
| Fwd bulk features non-zero | 4.1% | 0.65% | |

### 5.4 Label drift

The class sets overlap but are not the same. Names differ (`FTP-Patator` vs `FTP-BruteForce`, `Web Attack - SQL Injection` vs `Web Attack - SQL`), some classes exist in only one year (Heartbleed, DoS Slowhttptest and external Portscan only in 2017; DDoS-HOIC and DDoS-LOIC-UDP only in 2018), and FTP brute force never succeeded in 2018 (every flow is `Attempted`). The full mapping is in [ATTACK_LABELS.md §4](ATTACK_LABELS.md#4-mapping-between-2017-and-2018-labels).

### 5.5 Practical consequences

- For cross-dataset tests, map both years onto a shared coarse label set first (BENIGN, DoS, DDoS, BruteForce, WebAttack, Botnet, Infiltration/Recon).
- Normalise per dataset or use tree models; standardising with 2017 statistics and applying them to 2018 shifts most features.
- Check feature importance for `CWR/ECE Flag Count`, `Init Win Bytes` and `Dst Port`. High importance there usually means the model learned the testbed, not the attack.

## 6. Sources

- Engelen, G., Rimmer, V., Joosen, W. (2021). *Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study.* IEEE S&P Workshops (WTMC). <https://intrusion-detection.distrinet-research.be/WTMC2021/Resources/wtmc2021_Engelen_Troubleshooting.pdf>
- Liu, L., Engelen, G., Lynar, T., Essam, D., Joosen, W. (2022). *Error Prevalence in NIDS datasets: A Case Study on CIC-IDS-2017 and CSE-CIC-IDS-2018.* IEEE CNS. <https://intrusion-detection.distrinet-research.be/CNS2022/>
- DistriNet CICFlowMeter changes (UTC timestamps, ICMP features, `Total TCP Flow Time`, Fwd/Bwd RST flags, TCP segmentation offload fix): <https://intrusion-detection.distrinet-research.be/CNS2022/Tools_Documentation.html>
- CIC dataset pages (network layout, attack schedules): <https://www.unb.ca/cic/datasets/ids-2017.html>, <https://www.unb.ca/cic/datasets/ids-2018.html>
- All counts and statistics in this file: computed directly from the CSVs on 5 Oct 2026.
