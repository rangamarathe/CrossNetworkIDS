# Attack Labels: CICIDS2017 (improved) and CSE-CIC-IDS2018 (improved)

Every value of `Label` and `Attempted Category` in both datasets, with exact flow counts, how the two years map onto each other, a technical explanation of each attack (§5), and which attacks are **not** labelled (§6). Column definitions are in [DATA_DICTIONARY.md](DATA_DICTIONARY.md).

All counts below were computed over **every row** of both datasets on 5 Oct 2026 (2,099,976 rows for 2017; 63,195,145 for 2018). The 2018 counts match `EDA/results/label_counts_2018.csv` from notebook 01.

## 1. How labelling works in the improved datasets

DistriNet (KU Leuven) relabelled both datasets by attacker/victim IP, time window **and** flow content. Two things follow from that:

1. **`BENIGN`** is everything not tied to an attack.
2. **`<attack> - Attempted`** marks a flow between attacker and victim during the attack that carries **no effective attack payload**: a refused connection, an empty TCP session, tool start-up noise and so on. A flow-level detector cannot tell these apart from failed benign connections, so DistriNet separated them instead of calling them attacks.

The `Attempted Category` column says why. It is `-1` on every BENIGN and normal attack row.

| Code | Meaning (DistriNet) | 2017 flows | 2018 flows |
|---|---|---|---|
| -1 | Not an attempted flow | 2,087,997 | 62,888,908 |
| 0 | No payload: TCP connection with no data in the forward direction | 4,845 | 2,634 |
| 1 | Port / system closed: the target refused or did not answer | 4,067 | 298,844 |
| 2 | Attack start-up / teardown artefact | 23 | 22 |
| 3 | No malicious payload: data was sent, but nothing malicious | 27 | 1 |
| 4 | Attack artefact: side traffic produced by the attack tool | 75 | 4,306 |
| 5 | Attack implemented incorrectly | 138 | 126 |
| 6 | Target system unresponsive: the victim was already down | 2,804 | 304 |

Notebook 02 folds every `- Attempted` flow into `BENIGN`. That follows DistriNet's own benchmark (Engelen et al. 2021, ML benchmark section), but it is a choice. The alternative is to drop them or keep them as a separate class.

## 2. CICIDS2017 improved: 27 labels

Week of 3 to 7 July 2017. The attacker is `205.174.165.73`, seen inside the network as the NAT address `172.16.0.1`; victims are in `192.168.10.0/24`.

| Label | Flows | Share | Day / file | What it is |
|---|---:|---:|---|---|
| BENIGN | 1,582,566 | 75.36% | all | Normal user traffic (Monday is benign only) |
| Portscan | 159,066 | 7.57% | Fri | nmap scans from outside (`172.16.0.1` → `192.168.10.50`), many scan types, all under one label |
| DoS Hulk | 158,468 | 7.55% | Wed | HTTP GET flood (HULK). DistriNet notes the tool is misimplemented (`Connection: close`) but kept the label |
| DDoS | 95,144 | 4.53% | Fri | LOIC HTTP flood against `192.168.10.50` |
| Infiltration - Portscan | 71,767 | 3.42% | Thu | Internal nmap scan **from a compromised victim** (`192.168.10.8`) across the LAN |
| DoS GoldenEye | 7,567 | 0.36% | Wed | HTTP keep-alive flood |
| FTP-Patator | 3,972 | 0.19% | Tue | FTP password brute force |
| DoS Slowloris | 3,859 | 0.18% | Wed | Slow partial HTTP headers |
| SSH-Patator | 2,961 | 0.14% | Tue | SSH password brute force |
| DoS Slowhttptest | 1,740 | 0.08% | Wed | Slow HTTP body/read attack |
| Botnet | 736 | 0.04% | Fri | Ares botnet command-and-control |
| Web Attack - Brute Force | 73 | 0.0035% | Thu | Login brute force on DVWA |
| Infiltration | 36 | 0.0017% | Thu | Reverse shell from infected victims (`.8` Vista, `.25` Mac) to the attacker on port 444 |
| Web Attack - XSS | 18 | 0.0009% | Thu | Cross-site scripting on DVWA |
| Web Attack - SQL Injection | 13 | 0.0006% | Thu | SQL injection on DVWA |
| Heartbleed | 11 | 0.0005% | Wed | OpenSSL Heartbleed exploit against `192.168.10.51` (few, very long flows) |
| Botnet - Attempted | 4,067 | 0.19% | Fri | Bot calling home after the C&C server was shut down (all code 1) |
| DoS Slowhttptest - Attempted | 3,368 | 0.16% | Wed | Mostly code 6 (server already down) |
| DoS Slowloris - Attempted | 1,847 | 0.09% | Wed | Mostly code 0 |
| Web Attack - Brute Force - Attempted | 1,292 | 0.06% | Thu | Mostly code 0 |
| Web Attack - XSS - Attempted | 655 | 0.03% | Thu | Mostly code 0 |
| DoS Hulk - Attempted | 581 | 0.03% | Wed | |
| DoS GoldenEye - Attempted | 80 | 0.004% | Wed | |
| Infiltration - Attempted | 45 | 0.002% | Thu | |
| SSH-Patator - Attempted | 27 | 0.001% | Tue | Code 3 (TLS handshake, no login attempt) |
| FTP-Patator - Attempted | 12 | 0.0006% | Tue | |
| Web Attack - SQL Injection - Attempted | 5 | 0.0002% | Thu | |

Totals: attacks 505,431 (24.07%), attempted 11,979 (0.57%).

## 3. CSE-CIC-IDS2018 improved: 25 labels

Ten weekdays, 14 Feb to 2 Mar 2018, on AWS. Victims are in `172.31.0.0/16`. In the flows, every attack comes from a public AWS address (listed per class), not from the `172.31.70.x` hosts given in CIC's schedule.

| Label | Flows | Share | Day(s) | Attacker seen in flows | What it is |
|---|---:|---:|---|---|---|
| BENIGN | 59,353,486 | 93.92% | all | | Normal traffic |
| DoS Hulk | 1,803,160 | 2.853% | 16 Feb | | HTTP GET flood |
| DDoS-HOIC | 1,082,293 | 1.713% | 21 Feb | multiple | HOIC HTTP flood |
| DDoS-LOIC-HTTP | 289,328 | 0.458% | 20 Feb | multiple | LOIC HTTP flood |
| Botnet Ares | 142,921 | 0.226% | 2 Mar | `18.219.211.138` | Ares C&C traffic from 10 infected hosts |
| SSH-BruteForce | 94,197 | 0.149% | 14 Feb | `13.58.98.64` | SSH password brute force |
| Infiltration - NMAP Portscan | 89,374 | 0.141% | 28 Feb, 1 Mar | (internal victim) | Internal scan from a compromised host |
| DoS GoldenEye | 22,560 | 0.036% | 15 Feb | `18.219.211.138` | HTTP keep-alive flood |
| DoS Slowloris | 8,490 | 0.013% | 15 Feb | `18.217.165.70` | Slow HTTP headers |
| DDoS-LOIC-UDP | 2,527 | 0.004% | 20, 21 Feb | multiple | LOIC UDP flood |
| Infiltration - Communication Victim Attacker | 204 | 0.0003% | 28 Feb, 1 Mar | `13.58.225.34` | Victim to attacker C&C session |
| Web Attack - Brute Force | 131 | 0.0002% | 22, 23 Feb | `18.218.115.60` | DVWA login brute force |
| Web Attack - XSS | 113 | 0.0002% | 22, 23 Feb | `18.218.115.60` | Cross-site scripting |
| Infiltration - Dropbox Download | 85 | 0.0001% | 28 Feb, 1 Mar | | Victim downloads the malicious file from Dropbox |
| Web Attack - SQL | 39 | 0.0001% | 22, 23 Feb | `18.218.115.60` | SQL injection |
| FTP-BruteForce - Attempted | 298,874 | 0.473% | 14, 16 Feb | `18.221.219.4` | **No FTP brute force ever succeeded**: port 21 was closed (code 1) |
| DoS GoldenEye - Attempted | 4,301 | 0.007% | 15 Feb | | Codes 4 and 6 |
| DoS Slowloris - Attempted | 2,280 | 0.004% | 15 Feb | | Code 0 |
| Botnet Ares - Attempted | 262 | 0.0004% | 2 Mar | | |
| DDoS-LOIC-UDP - Attempted | 251 | 0.0004% | 20, 21 Feb | | Code 6 |
| Web Attack - Brute Force - Attempted | 137 | 0.0002% | 22, 23 Feb | | Mostly code 5 |
| DoS Hulk - Attempted | 86 | 0.0001% | 16 Feb | | |
| Infiltration - Dropbox Download - Attempted | 28 | <0.0001% | 28 Feb, 1 Mar | | Code 4 |
| Web Attack - SQL - Attempted | 14 | <0.0001% | 22, 23 Feb | | |
| Web Attack - XSS - Attempted | 4 | <0.0001% | 22, 23 Feb | | |

Totals: attacks 3,535,422 (5.59%), attempted 306,237 (0.48%).

Per-file counts for 2018 are in `EDA/results/label_counts_2018.csv`.

## 4. Mapping between 2017 and 2018 labels

For cross-year experiments, map both onto a shared label. The family column is a suggestion.

| Family | 2017 label | 2018 label | Notes |
|---|---|---|---|
| BENIGN | BENIGN | BENIGN | |
| BruteForce | FTP-Patator | FTP-BruteForce *(Attempted only)* | **No successful 2018 example.** A 2017-trained FTP detector cannot be scored on 2018. |
| BruteForce | SSH-Patator | SSH-BruteForce | Same tool (Patator). |
| DoS | DoS Hulk | DoS Hulk | |
| DoS | DoS GoldenEye | DoS GoldenEye | |
| DoS | DoS Slowloris | DoS Slowloris | |
| DoS | DoS Slowhttptest | *(none)* | 2018 run misfired at port 21; see §6.1. |
| DDoS | DDoS | DDoS-LOIC-HTTP | Same tool (LOIC, HTTP mode). |
| DDoS | *(none)* | DDoS-LOIC-UDP | 2018 only. |
| DDoS | *(none)* | DDoS-HOIC | 2018 only, and the second-largest 2018 attack class. |
| WebAttack | Web Attack - Brute Force | Web Attack - Brute Force | |
| WebAttack | Web Attack - XSS | Web Attack - XSS | |
| WebAttack | Web Attack - SQL Injection | Web Attack - SQL | Name differs. |
| Botnet | Botnet | Botnet Ares | Both are Ares. |
| Infiltration | Infiltration | Infiltration - Communication Victim Attacker | Victim-to-attacker C&C. |
| Infiltration | *(not labelled; see §6.1)* | Infiltration - Dropbox Download | |
| Recon | Infiltration - Portscan | Infiltration - NMAP Portscan | Internal scan from a compromised host. |
| Recon | Portscan | *(none)* | External scan exists in 2017 only. |
| Exploit | Heartbleed | *(none)* | Documented for 2018 but never found. |

## 5. What each attack does

For each attack family: how it works, the tool used, what it looks like as flows in these files, and what changed between 2017 and 2018. The **measured** figures are medians over the non-attempted flows of that label, from every row of both datasets. Behaviour described as "expected" comes from how the tool works, not from the data.

### 5.1 Measured flow signatures

| Label (2017 / 2018) | Year | Flows | Dst port | Proto | Duration (median) | Fwd / Bwd packets | Fwd / Bwd payload bytes | Sources → targets |
|---|---|---:|---|---|---:|---|---|---|
| Portscan | 2017 | 159,066 | ~1,000 distinct | TCP | 0.05 ms | 1 / 1 | 0 / 0 | 1 → 1 |
| Infiltration - Portscan | 2017 | 71,767 | 1,102 distinct | TCP | 0.05 ms | 2 / 1 | 0 / 0 | 2 → 11 |
| Infiltration - NMAP Portscan | 2018 | 89,374 | 1,579 distinct | TCP | 0.23 ms | 1 / 1 | 0 / 0 | 2 → 22 |
| FTP-Patator | 2017 | 3,972 | 21 | TCP | 9.0 s | 11 / 17 | 120 / 188 | 1 → 1 |
| SSH-Patator / SSH-BruteForce | 2017 | 2,961 | 22 | TCP | 12.1 s | 22 / 33 | 2,008 / 2,745 | 1 → 1 |
| | 2018 | 94,197 | 22 | TCP | 0.38 s | 23 / 23 | 1,928 / 2,665 | 1 → 1 |
| DoS Hulk | 2017 | 158,468 | 80 | TCP | 163 ms | 8 / 6 | 353 / 11,595 | 1 → 1 |
| | 2018 | 1,803,160 | 80 | TCP | 139 ms | 5 / 5 | 351 / 935 | 1 → 1 |
| DoS GoldenEye | 2017 | 7,567 | 80 | TCP | 11.9 s | 9 / 5 | 424 / 11,632 | 1 → 1 |
| | 2018 | 22,560 | 80 | TCP | 11.6 s | 6 / 4 | 416 / 972 | 1 → 1 |
| DoS Slowloris | 2017 | 3,859 | 80 | TCP | 102.6 s | 5 / 3 | 247 / 0 | 1 → 1 |
| | 2018 | 8,490 | 80 | TCP | 106.9 s | 14 / 3 | 2,530 / 0 | 1 → 1 |
| DoS Slowhttptest | 2017 | 1,740 | 80 | TCP | 108.1 s | 4 / 3 | 530 / 0 | 1 → 1 |
| DDoS / DDoS-LOIC-HTTP | 2017 | 95,144 | 80 | TCP | 6.5 s | 8 / 5 | 20 / 11,595 | 1 → 1 (behind NAT) |
| | 2018 | 289,328 | 80 | TCP | 22.2 s | 5 / 4 | 20 / 964 | 10 → 1 |
| DDoS-HOIC | 2018 | 1,082,293 | 80 | TCP | 20 ms | 5 / 5 | 309 / 935 | 10 → 1 |
| DDoS-LOIC-UDP | 2018 | 2,527 | 80 | **UDP** | 119.8 s (timeout) | **122,682 / 0** | **3.9 MB / 0** | 10 → 2 |
| Web Attack - Brute Force | 2017 | 73 | 80 | TCP | 34.1 s | 204 / 105 | 43,906 / 72,190 | 1 → 1 |
| | 2018 | 131 | 80 | TCP | 56.5 s | 155 / 105 | 54,995 / 72,488 | 1 → 1 |
| Web Attack - XSS | 2017 | 18 | 80 | TCP | 67.7 s | 205 / 105 | 48,783 / 183,599 | 1 → 1 |
| | 2018 | 113 | 80 | TCP | 57.0 s | 205 / 105 | 56,083 / 189,977 | 1 → 1 |
| Web Attack - SQL Injection / SQL | 2017 | 13 | 80 | TCP | 5.0 s | 6 / 5 | 599 / 2,021 | 1 → 1 |
| | 2018 | 39 | 80 | TCP | 5.0 s | 6 / 5 | 618 / 1,873 | 1 → 1 |
| Botnet / Botnet Ares | 2017 | 736 | 8080 | TCP | 77 ms | 5 / 4 | 195 / 128 | 5 → 1 |
| | 2018 | 142,921 | 8080 | TCP | 12 ms | 5 / 5 | 326 / 129 | 10 → 1 |
| Infiltration / Comm. Victim Attacker | 2017 | 36 | 444 | TCP | 100.2 s | 34.5 / 34.5 | 12,051 / 0 | 2 → 1 |
| | 2018 | 204 | 31337 | TCP | 60.2 s | 4 / 4 | 320 / 256 | 2 → 1 |
| Infiltration - Dropbox Download | 2018 | 85 | 443 | TCP | 54.3 s | 14 / 12 | 949 / 3,795 | 2 → 5 |
| Heartbleed | 2017 | 11 | 444 | TCP | 119.3 s (timeout) | 2,791 / 2,070 | 7,920 / 7.9 MB | 1 → 1 |

"Sources → targets" counts distinct `Src IP` and `Dst IP`. Every attack class except the portscans hits a single port and, mostly, a single host. This is why `Dst Port` and the IP columns are such strong (and testbed-specific) shortcuts.

### 5.2 Reconnaissance: Portscan, Infiltration - Portscan, Infiltration - NMAP Portscan

- **What it is:** mapping which hosts are up and which TCP ports are open, as preparation for an attack.
- **Tool:** nmap. In 2017 the external scan cycled through many nmap scan types (SYN, connect, FIN, Xmas, Null, ACK, version, and others), all labelled `Portscan`.
- **How it works:** the scanner sends one probe (usually a SYN) per port. An open port answers SYN-ACK and a closed one answers RST; nmap then drops the connection.
- **In the flows:** about one row per probed port. Each row has 1 to 2 packets, **0 payload bytes** and a duration of a fraction of a millisecond. The number of distinct ports (about 1,000, nmap's default top-1000 list) gives it away.
- **External vs internal:** the 2017 `Portscan` comes from outside, through the NAT `172.16.0.1`, at one server. The `Infiltration - …Portscan` classes come from a **compromised inside host** scanning the LAN (2017: `192.168.10.8` → 11 hosts; 2018: → 22 hosts), so they are post-compromise lateral recon.
- **Findings:** DistriNet found two 2017 scans that the original labels had left as BENIGN (one before the documented window, one undocumented). They also found that part of the Friday scan ran with the firewall on, so only ports 21, 22, 80, 443 and 444 got through. In 2018 there is no external portscan class.

### 5.3 Brute force: FTP-Patator / FTP-BruteForce, SSH-Patator / SSH-BruteForce

- **What it is:** guessing passwords online against a login service with a wordlist.
- **Tool:** Patator, a multi-protocol brute-forcer.
- **How it works:** each connection attempts one or more username/password pairs. FTP sends `USER`/`PASS` in clear text. SSH negotiates keys first, then tries passwords inside the encrypted channel.
- **In the flows:** fixed port (21 or 22), and many near-identical flows from one source to one victim. The SSH flows are about 22 packets and 2 KB each way in both years, almost identical in size. In 2018 they are **30× shorter in time** (0.38 s vs 12 s), so timing features drift while size features do not.
- **2017 findings:** the first FTP flow was a successful login, not brute force. Some SSH flows only completed the handshake (`SSH-Patator - Attempted`, code 3).
- **2018 findings:** **FTP brute force never worked.** Port 21 on the victim was closed on both 14 and 16 Feb, so all 298,874 flows are SYN → RST (`- Attempted`, code 1). The 16 Feb run was not on the schedule at all. SSH brute force worked normally.

### 5.4 DoS (one attacker): Hulk, GoldenEye, Slowloris, Slowhttptest

All four attack one web server on port 80 from a single machine. They split into two kinds:

**Volume (flood) attacks: Hulk and GoldenEye**
- **How they work:** send many HTTP GET requests as fast as possible, with randomised URLs, user agents and referers so caches and simple filters don't help. GoldenEye also uses HTTP keep-alive to hold connections open.
- **In the flows:** many short HTTP connections, a few hundred bytes forward and a full page back. Hulk flows last about 0.15 s; GoldenEye flows last about 12 s because of keep-alive.
- **Findings:** the Hulk script used **sends `Connection: close`**, so it never holds connections as intended (Engelen et al. 2021). It is still labelled `DoS Hulk`, but it behaves like a fast GET flood. In 2018 the server's response is about 935 bytes instead of 11.6 KB (a different web page), so backward byte features differ between the years. About 16% of 2018 GoldenEye flows are `- Attempted` (codes 4 and 6: single transactions cut by RST, or the server already down).

**Slow (low-bandwidth) attacks: Slowloris and Slowhttptest**
- **How they work:** open many connections and keep each one alive by sending data extremely slowly. Slowloris sends partial HTTP headers (`X-a: b` every few seconds) and never finishes the request. Slowhttptest can also send a slow POST body or read the response slowly. The server's connection pool fills, and real users are locked out.
- **In the flows:** **long duration (100 s or more, close to the 120 s flow timeout), few packets, and 0 bytes back** because the server never replies to an unfinished request. This is the opposite of a flood.
- **Findings:** in 2017, Slowhttptest has more `- Attempted` flows (3,368) than real ones (1,740), mostly code 6: the server was already unresponsive, so later connections got nothing. Manual web browsing was mixed into the 2017 attack window. **In 2018 there is no Slowhttptest at all**: it was launched against port 21 by mistake and folded into `FTP-BruteForce - Attempted`.

### 5.5 DDoS (many attackers): LOIC-HTTP, LOIC-UDP, HOIC

- **What it is:** the same flood idea as DoS, but from many machines at once, so blocking one source doesn't help.
- **Tools:** LOIC (Low Orbit Ion Cannon) in HTTP or UDP mode; HOIC (High Orbit Ion Cannon), which floods HTTP with "booster" scripts that randomise headers.
- **How it works:** LOIC-HTTP repeats a fixed GET request over many TCP connections. LOIC-UDP blasts UDP datagrams at a port with no handshake. HOIC sends a high rate of varied HTTP requests.
- **In the flows:**
  - LOIC-HTTP: about 20 forward payload bytes per flow (a tiny fixed request) and long-lived connections (6.5 s in 2017, 22 s in 2018).
  - LOIC-UDP: **UDP to port 80**, one huge one-way flow per source. The median flow lasts the full 120 s timeout and carries about 122,000 packets (3.9 MB) with **nothing coming back**.
  - HOIC: very short (20 ms) HTTP flows in very large numbers. It is the second-largest 2018 class.
- **2017 vs 2018:** the 2017 `DDoS` is LOIC-HTTP from the Windows attacker machines (per CIC), but they all appear as one IP behind the NAT. In 2018, 10 public attacker IPs are visible.
- **Findings:** on 20 Feb 2018 the original release labelled every attack flow as LOIC-HTTP, including the LOIC-UDP ones. CICFlowMeter also processed ICMP packets as UDP. The DistriNet version fixes both, which is why `DDoS-LOIC-UDP` exists here. The `- Attempted` LOIC-UDP flows (code 6) are traffic sent after the target stopped answering. In 2017, DistriNet notes that LOIC did not take the server down, and that scripted look-alike flows outside the window remain BENIGN.

### 5.6 Web application attacks: Brute Force, XSS, SQL Injection

All target **DVWA** (Damn Vulnerable Web Application) on port 80, from one attacker (2017 through the NAT, 2018 `18.218.115.60`). They are driven by Selenium scripts written by CIC.

- **Web Attack - Brute Force:** repeated login form posts with password guesses. The flows are long (34 to 57 s), with about 200 forward packets and 44 to 55 KB each way: many form submissions inside one keep-alive connection.
- **Web Attack - XSS:** injects `<script>` payloads into input fields that are then reflected back. The flows look much like brute force (the same Selenium harness, about 205/105 packets), but return more bytes (about 185 KB): pages echoing the payload.
- **Web Attack - SQL Injection / SQL:** puts SQL syntax such as `' OR 1=1 --` into parameters to dump the database. The flows are short (5 s), about 600 bytes forward and 2 KB back.
- **Why they are hard:** the payload is inside the HTTP content, which flow statistics don't see. Size and timing differences come from the test harness more than from the attack. With 13 to 131 flows per class, per-class scores are noisy.
- **Findings:**
  - In 2017 most web brute-force and XSS flows (1,292 and 655) are `- Attempted` with code 0: the harness opened TCP connections and sent nothing. In the original release they counted as attacks.
  - On 23 Feb 2018, 41% of the original web brute-force labels were wrong (DistriNet): SSH, DHCP and port-500 UDP traffic had been included. Code 5 marks single login attempts that look exactly like a normal user logging in.
  - CIC's documented 2017 time windows were off by several minutes.

### 5.7 Botnet: Botnet (2017), Botnet Ares (2018)

- **What it is:** infected machines ("bots") that regularly contact a command-and-control (C&C) server for instructions: run commands, upload files, take screenshots.
- **Tool:** Ares, an open-source Python botnet that uses HTTP for C&C.
- **How it works:** each bot **polls** the C&C web server at a fixed interval. Most polls are short "anything for me?" requests.
- **In the flows:** many small, regular TCP flows from several victims to **one server on port 8080**: about 5 packets and a few hundred bytes each way, lasting tens of milliseconds. There are 5 bots in 2017 and 10 in 2018. The periodicity is the main signal, and it is only partly visible in per-flow features.
- **Findings:** in 2017 there are 4,067 `Botnet - Attempted` flows against 736 real ones, from bots that kept calling home after the C&C server was shut down (code 1). A model that counts those as Botnet learns "connection refused to 8080". CIC also lists Zeus for 2018, but only Ares is labelled.

### 5.8 Infiltration: Infiltration (2017), Dropbox Download, Communication Victim Attacker (2018)

- **What it is:** an outside attacker gets code running on an inside machine (through a malicious document or download), then uses that foothold to explore the internal network.
- **Scenario:**
  1. The victim downloads an infected file. In 2017 this was a Metasploit payload on Windows Vista via Dropbox, plus a "Cool Disk" attack on the Mac. In 2018 it was a Dropbox download.
  2. The payload opens a **reverse shell** (Meterpreter) back to the attacker.
  3. The attacker runs nmap from the victim (§5.2).
- **In the flows:**
  - Dropbox Download: HTTPS (443) to Dropbox servers, indistinguishable by port from normal browsing.
  - The C&C session is a long-lived TCP connection to an unusual port (**444** in 2017, **31337** in 2018). Total TCP connection time is 9 minutes (2017) to over an hour (2018), split into many 120 s flow rows.
- **Findings:**
  - The 2017 Dropbox download stays BENIGN: it is encrypted and cannot be pinned to one connection.
  - The Mac "Cool Disk" Meterpreter session was missed entirely by the original labels; it is now part of `Infiltration`.
  - The original 2018 release had a single `Infilteration` label that was mostly benign traffic, and the victim-attacker sessions were labelled BENIGN. The three-way split here is DistriNet's.

### 5.9 Heartbleed (2017 only)

- **What it is:** CVE-2014-0160, a bug in OpenSSL's TLS heartbeat extension. The attacker sends a heartbeat request that claims to carry far more data than it does, and the server replies with up to 64 KB of its own memory, which can include private keys, session cookies and passwords.
- **Tool:** a Heartbleed exploit script run against a vulnerable Ubuntu web server (`192.168.10.51`) on port 444.
- **In the flows:** 11 rows that are really one long TLS connection repeating the exploit, split at the 120 s timeout. There are thousands of packets each, and the server sends **about 7.9 MB back for about 8 KB in**: a huge download-to-upload ratio on a TLS port. The connection lasts about 12 minutes in total.
- **Findings:** DistriNet found no problems with it. For 2018, CIC lists Heartbleed (via the Heartleech tool), but it was never found in the traffic, so there is no 2018 example.

## 6. Attacks that are not labelled

There are two different gaps. §6.1 lists attacks that the dataset authors say they ran, or that are in the captures, but that have no (successful) label in these files. §6.2 lists common real-world attack types that neither dataset tried to cover.

### 6.1 Attacks in the documentation or the traffic, but not labelled

| Year | Attack | What happened | Effect in these files | Source |
|---|---|---|---|---|
| 2018 | **Heartbleed** (Heartleech) | Listed by CIC as one of the seven scenarios. CIC never gave attacker/victim IPs, and DistriNet found no evidence it was run. | No label. Nothing to detect. | CIC 2018 page; DistriNet 2018 notes |
| 2018 | **Zeus botnet** | CIC lists Zeus alongside Ares. | Only `Botnet Ares` exists. Any Zeus traffic, if present, is inside BENIGN. | CIC 2018 page |
| 2018 | **Command injection** and **unrestricted file upload** (DVWA) | CIC lists them under web attacks. | No labels; only Brute Force, XSS and SQL exist. Not documented in the schedule either, so may never have run. | CIC 2018 page |
| 2018 | **DoS SlowHTTPTest** (16 Feb) | Launched against port 21 (closed) instead of the web server. DistriNet found no successful SlowHTTPTest anywhere in 2018. | Folded into `FTP-BruteForce - Attempted`. We confirmed all 105,520 of that day's attempted flows target port 21. | DistriNet 2018 notes; Liu et al. 2022 |
| 2018 | **FTP brute force** (14 and 16 Feb) | Port 21 was closed, so no login was ever attempted. The 16 Feb run was not even scheduled. | 298,874 flows, all `- Attempted`. With notebook 02's folding they all become BENIGN. | DistriNet 2018 notes |
| 2018 | Original release labelled infiltration poorly | The original CSVs had one `Infilteration` label that was mostly benign; victim-attacker traffic was labelled BENIGN, and the first LOIC-UDP day had no LOIC-UDP labels. | **Fixed in these files**, but any paper using the original 2018 CSVs used different ground truth. | DistriNet 2018 notes; Liu et al. 2022 |
| 2017 | **Infiltration, Dropbox exploit download** (Thu) | The victim downloaded the exploit over HTTPS. It is encrypted, so DistriNet could not tell which connection carried it. | Left as BENIGN. Only the later reverse shell (`Infiltration`, 36 flows) and the internal scan are labelled. | DistriNet 2017 notes |
| 2017 | **Cool Disk (Mac) infiltration and two extra port scans** (Thu) | Completely missed in the original labels: a Meterpreter payload to `192.168.10.25`, a port scan at 17:33 before the listed window, and an undocumented scan. | **Labelled in these files** (`Infiltration`, `Infiltration - Portscan`), but BENIGN in the original release. | DistriNet 2017 notes; Engelen et al. 2021 |
| 2017 | **LOIC look-alike background flows** (Fri) | DistriNet saw identical, apparently scripted flows outside the DDoS window. | Left as BENIGN; their nature is unclear. | DistriNet 2017 notes |
| 2017 | **DoS Hulk** (Wed) | The HULK tool sends `Connection: close`, so it does not actually hold connections open. Strictly it should be `Attempted`, code 5. | Still labelled `DoS Hulk` (158,468 flows). Treat its "attack signature" with care. | Engelen et al. 2021, attack simulation section |

**Our own residual check.** We counted the labels on every flow to or from the documented attacker addresses:

- **2018:** of about 307,000 flows to or from the three public attacker addresses CIC documents (`18.218.115.60`, `18.219.211.138`, `13.58.225.34`), only 1 is BENIGN (one `13.58.225.34` flow on 28 Feb). Labelling looks close to complete for traffic touching those attackers. No flow at all involves the documented internal attackers `172.31.70.x`.
- **2017:** 4,128 such flows are BENIGN (0.2% of the dataset). Most are HTTPS from the web server `192.168.10.51` to the NAT address `172.16.0.1`, plus ICMP "destination unreachable" messages. We saw no clear attack pattern in them. Since `172.16.0.1` is also the firewall, ordinary traffic passes through it, so this is not evidence of missed attacks.

The internal infiltration traffic (Dropbox downloads, victim-initiated scans) does not involve attacker IPs, so this check cannot rule out unlabelled activity there.

### 6.2 Common attack types neither dataset covers

Neither year has labelled examples of the following. A model trained here will either miss them or flag them only by accident.

| Attack type | Closest label here | Why it matters | Public datasets that do include it |
|---|---|---|---|
| Reflection / amplification DDoS (DNS, NTP, SSDP, LDAP, memcached) | none (LOIC/HOIC are direct floods) | Most common volumetric DDoS in practice | CIC-DDoS2019 |
| TCP SYN flood, ICMP flood | none | Classic L3/L4 DoS | CIC-DDoS2019; UNSW-NB15 (`DoS`) |
| Ransomware and worm propagation | none | Lateral spread over SMB and similar | TON_IoT (`ransomware`); UNSW-NB15 (`Worms`) |
| Lateral movement (SMB, RDP, WinRM, pass-the-hash) | partly `Infiltration - Portscan` (recon only) | Post-compromise activity is the main enterprise risk | LANL Unified Host and Network data (host/auth logs, not flows) |
| Brute force on other services (RDP, SMB, Telnet, HTTP basic auth, credential stuffing) | FTP/SSH/DVWA only | Real brute force often targets RDP | TON_IoT (`password`); CIC-IoT2023 (dictionary brute force) |
| Data exfiltration and DNS tunnelling | none (Dropbox **download** only) | Data theft is the usual end goal | CIC-Bell-DNS-EXF-2021; CIC-DoHBrw-2020 (DNS over HTTPS tunnels) |
| Modern C2 (Cobalt Strike, Sliver, HTTPS/DoH beaconing) | `Botnet Ares` (plain HTTP) | Current malware hides in TLS | CIC-DoHBrw-2020 (partly) |
| Man-in-the-middle, ARP / DNS spoofing | none | Layer-2 attacks are mostly invisible to IP flow features | CIC-IoT2023; TON_IoT (`mitm`) |
| IoT botnets (Mirai etc.) and IoT protocol attacks (MQTT) | none | Large share of real scanning and DDoS sources | CIC-IoT2023; Bot-IoT; TON_IoT |
| Exploits, shellcode, fuzzing, backdoors (beyond Heartbleed) | `Heartbleed` (2017, 11 flows) | Remote code execution (e.g. Log4Shell) is a top entry point | UNSW-NB15 (`Exploits`, `Shellcode`, `Fuzzers`, `Backdoor`) |
| Other web attacks (command injection, file upload, SSRF, path traversal) | DVWA brute force, XSS, SQL only | OWASP Top 10 coverage is thin; SQL has only 13 to 39 flows | TON_IoT (`injection`, `xss`); CIC-IDS2018 lists them but has no labels |
| Vulnerability scanning (Nessus, OpenVAS), UDP-only scans | `Portscan` (nmap TCP-heavy) | Common, noisy recon | UNSW-NB15 (`Reconnaissance`, `Analysis`) |
| Cryptojacking, phishing / malicious email | none | Very common in practice | No standard flow-labelled dataset |
| Low-and-slow or evasive variants of covered attacks | none | Tests robustness rather than memorised signatures | Mostly in adversarial-ML papers, not standard datasets |

Coverage notes: the "datasets that include it" column is from each dataset's own published class list; we did not inspect those datasets.

## 7. What this means for the capstone

- **Rare classes.** Heartbleed (11), SQL injection (13 / 39), XSS (18 / 113) and 2017 Infiltration (36) are too small for stable per-class scores. Notebook 02 already flags this.
- **Cross-year testing.** Only 12 classes, BENIGN included, exist in both years (§4). Use the family mapping, and report FTP brute force, Slowhttptest, Heartbleed and external Portscan as "not testable on 2018".
- **Attempted flows.** Folding them into BENIGN (notebook 02) means 298,874 failed FTP brute-force flows become BENIGN in 2018. That is defensible for a flow-level detector, but say so in the write-up.
- **Out-of-scope attacks.** When describing results, state that "detection" covers the attack families in §2 and §3 only. §6.2 is the list of what the model has never seen.

## 8. Sources

- DistriNet, *CIC-IDS 2017: labelling logic and issues per day*: <https://intrusion-detection.distrinet-research.be/CNS2022/CICIDS2017.html>
- DistriNet, *CSE-CIC-IDS 2018: labelling logic and issues per day* (also the source of the `Attempted Category` code meanings): <https://intrusion-detection.distrinet-research.be/CNS2022/CSECICIDS2018.html>
- Engelen, G., Rimmer, V., Joosen, W. (2021). *Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study.* IEEE S&P Workshops (WTMC). <https://intrusion-detection.distrinet-research.be/WTMC2021/Resources/wtmc2021_Engelen_Troubleshooting.pdf>
- Liu, L., Engelen, G., Lynar, T., Essam, D., Joosen, W. (2022). *Error Prevalence in NIDS datasets: A Case Study on CIC-IDS-2017 and CSE-CIC-IDS-2018.* IEEE CNS. <https://intrusion-detection.distrinet-research.be/CNS2022/>
- Canadian Institute for Cybersecurity, CIC-IDS2017: <https://www.unb.ca/cic/datasets/ids-2017.html>
- Canadian Institute for Cybersecurity, CSE-CIC-IDS2018: <https://www.unb.ca/cic/datasets/ids-2018.html>
- Datasets cited in §6.2: CIC-DDoS2019 (<https://www.unb.ca/cic/datasets/ddos-2019.html>), CIC-IoT2023 (<https://www.unb.ca/cic/datasets/iotdataset-2023.html>), CIC-Bell-DNS-EXF-2021 (<https://www.unb.ca/cic/datasets/dns-exf-2021.html>), CIC-DoHBrw-2020 (<https://www.unb.ca/cic/datasets/dohbrw-2020.html>), UNSW-NB15 (<https://research.unsw.edu.au/projects/unsw-nb15-dataset>), TON_IoT (<https://research.unsw.edu.au/projects/toniot-datasets>), Bot-IoT (<https://research.unsw.edu.au/projects/bot-iot-dataset>), LANL Unified Host and Network (<https://csr.lanl.gov/data/2017/>).
- Flow counts, port and IP checks: computed directly from the CSVs on 5 Oct 2026.
