# RTT — Round Trip Time

**Latency Analysis · Network Measurement · Ping · Traceroute · TCP Handshake · Red Team OPSEC**

![RTT](https://img.shields.io/badge/RTT-ROUND%20TRIP%20TIME-ff0000?style=for-the-badge&logo=cloudflare&logoColor=white&labelColor=1a0000)
![Networking](https://img.shields.io/badge/NETWORKING-LATENCY-8b0000?style=for-the-badge&logo=cisco&logoColor=white&labelColor=1a0000)
![Ping](https://img.shields.io/badge/PING-TRACEROUTE-cc0000?style=for-the-badge&logo=wireshark&logoColor=white&labelColor=1a0000)
![OPSEC](https://img.shields.io/badge/OPSEC-C2%20TIMING-990000?style=for-the-badge&logo=linux&logoColor=white&labelColor=1a0000)
![Platform](https://img.shields.io/badge/PLATFORM-LINUX%20%7C%20WINDOWS-333333?style=for-the-badge&labelColor=1a0000)

---

```
[*] RTT = Time for a packet to travel from source → destination → back
[*] Measured in milliseconds (ms)
[*] Used for: Network diagnosis, recon, C2 timing, firewall detection
[*] Formula: RTT = Propagation + Transmission + Processing + Queuing
```

---

## Table of Contents

1. [What is RTT?](#1-what-is-rtt)
2. [RTT Formula & Components](#2-rtt-formula--components)
3. [RTT in the OSI Model](#3-rtt-in-the-osi-model)
4. [Measuring RTT](#4-measuring-rtt)
5. [RTT in TCP Handshake](#5-rtt-in-tcp-handshake)
6. [RTT in HTTP / HTTPS](#6-rtt-in-http--https)
7. [RTT vs Latency vs Jitter](#7-rtt-vs-latency-vs-jitter)
8. [RTT Benchmarks](#8-rtt-benchmarks)
9. [RTT in Red Teaming & OPSEC](#9-rtt-in-red-teaming--opsec)
10. [RTT-Based Network Recon](#10-rtt-based-network-recon)
11. [RTT Manipulation & Evasion](#11-rtt-manipulation--evasion)
12. [Tools & Commands](#12-tools--commands)

---

## 1. What is RTT?

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROUND TRIP TIME                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   [Attacker]                              [Target]             │
│       │                                      │                 │
│       │ ─────── PACKET (REQUEST) ──────────► │                 │
│       │                                      │                 │
│       │ ◄────── PACKET (RESPONSE) ────────── │                 │
│       │                                      │                 │
│   RTT = Time from SEND → RECEIVE response                      │
│                                                                 │
│   RTT ≠ One-Way Latency                                        │
│   One-Way Latency = RTT / 2  (approximate)                     │
└─────────────────────────────────────────────────────────────────┘
```

RTT measures the **total time** for a packet to:
```
[1] Leave the source
[2] Travel through the network to the destination
[3] Be processed by the destination
[4] Travel back to the source
```

---

## 2. RTT Formula & Components

```
RTT = Propagation Delay × 2
    + Transmission Delay × 2
    + Processing Delay
    + Queuing Delay
```

### Component Breakdown

| Component | Description | Affected By |
|---|---|---|
| **Propagation Delay** | Time for signal to travel the physical medium | Distance, medium (fiber/copper/wireless) |
| **Transmission Delay** | Time to push all bits onto the wire | Bandwidth, packet size |
| **Processing Delay** | Time for routers/hosts to process the packet | CPU load, routing table size |
| **Queuing Delay** | Time spent waiting in router buffers | Network congestion, traffic load |

### Propagation Speed Reference

```
Speed of light in vacuum  : 299,792 km/s
Speed in fiber optic      : ~200,000 km/s  (≈ 2/3 of light)
Speed in copper cable     : ~200,000 km/s

Theoretical minimum RTT:
  New York → London (~5,570 km fiber)
  = (5,570 / 200,000) × 2 × 1000 ms
  = ~55.7 ms  (theoretical)
  Real-world: 70–90 ms (due to routing hops + processing)
```

---

## 3. RTT in the OSI Model

```
┌─────────────────────────────────────────────────────────────┐
│                  RTT ACROSS OSI LAYERS                      │
├───────┬──────────────┬──────────────────────────────────────┤
│ Layer │ Name         │ RTT Contribution                     │
├───────┼──────────────┼──────────────────────────────────────┤
│   7   │ Application  │ Server processing time, app logic    │
│   6   │ Presentation │ TLS handshake overhead               │
│   5   │ Session      │ Session setup, keepalive             │
│   4   │ Transport    │ TCP handshake (1.5× RTT overhead)    │
│   3   │ Network      │ Routing hops, IP forwarding          │
│   2   │ Data Link    │ MAC lookup, switching delay          │
│   1   │ Physical     │ Signal propagation, cable length     │
└───────┴──────────────┴──────────────────────────────────────┘
```

---

## 4. Measuring RTT

### ping — ICMP RTT

```bash
# Basic ping (Linux)
ping TARGET_IP

# Specify count
ping -c 10 TARGET_IP

# Flood ping (requires root — stress test)
sudo ping -f TARGET_IP

# Set packet size
ping -s 1400 TARGET_IP

# Set TTL
ping -t 64 TARGET_IP

# Windows ping
ping TARGET_IP
ping -n 20 TARGET_IP          # 20 packets
ping -l 1400 TARGET_IP        # packet size 1400 bytes
ping -i 1 TARGET_IP           # TTL = 1 (hop-by-hop)
```

### Sample ping Output

```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=118 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=11.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
round-trip min/avg/max/stddev = 11.9/12.1/12.4/0.2 ms
                                 │    │    │    │
                                 │    │    │    └── Jitter
                                 │    │    └─────── Max RTT
                                 │    └──────────── Average RTT
                                 └───────────────── Min RTT
```

### traceroute / tracert — Per-Hop RTT

```bash
# Linux traceroute (UDP by default)
traceroute TARGET_IP

# ICMP traceroute (bypass firewalls)
traceroute -I TARGET_IP

# TCP traceroute on specific port
traceroute -T -p 443 TARGET_IP

# Windows tracert
tracert TARGET_IP
tracert -d TARGET_IP           # No DNS resolution (faster)
tracert -h 30 TARGET_IP        # Max 30 hops
```

### Sample traceroute Output

```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max
 1  192.168.1.1      1.2 ms   1.1 ms   1.0 ms   ← Local gateway
 2  10.0.0.1         5.4 ms   5.2 ms   5.1 ms   ← ISP hop
 3  172.16.0.1      10.3 ms  10.1 ms  10.2 ms   ← Transit
 4  * * *                                         ← Filtered (ICMP blocked)
 5  209.85.240.1    12.1 ms  11.9 ms  12.0 ms   ← Google edge
 6  8.8.8.8         12.4 ms  12.2 ms  12.3 ms   ← Destination

[*] * * * = Router drops ICMP TTL-exceeded → not necessarily a firewall
[*] RTT jump between hops = congestion or geographic distance
```

### hping3 — Advanced RTT Measurement

```bash
# TCP RTT on port 80
hping3 -S -p 80 -c 5 TARGET_IP

# TCP RTT on port 443
hping3 -S -p 443 -c 5 TARGET_IP

# UDP RTT
hping3 --udp -p 53 -c 5 TARGET_IP

# Measure RTT while varying packet size
hping3 -S -p 443 -c 10 -d 500 TARGET_IP    # 500 byte payload
```

### nping (Nmap suite)

```bash
# TCP RTT
nping --tcp -p 80,443 TARGET_IP

# ICMP RTT
nping --icmp TARGET_IP

# UDP RTT
nping --udp -p 53 TARGET_IP

# Sample output:
# SENT: 5 | RCVD: 5 | LOST: 0 (0%)
# Max rtt: 12.45ms | Min rtt: 11.89ms | Avg rtt: 12.10ms
```

---

## 5. RTT in TCP Handshake

```
┌─────────────────────────────────────────────────────────────────┐
│                   TCP 3-WAY HANDSHAKE RTT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                                    Server              │
│    │                                          │                │
│    │ ──────────── SYN ──────────────────────► │  t=0           │
│    │                                          │                │
│    │ ◄──────── SYN-ACK ──────────────────── │  t=0.5 RTT     │
│    │                                          │                │
│    │ ──────────── ACK ──────────────────────► │  t=1.0 RTT     │
│    │                                          │                │
│    │ ──────────── DATA ─────────────────────► │                │
│    │ ◄──────── RESPONSE ──────────────────── │  t=1.5 RTT     │
│                                                                 │
│  Connection established at: 1 RTT                              │
│  First data received at:    1.5 RTT                            │
└─────────────────────────────────────────────────────────────────┘
```

### Measure TCP Handshake RTT

```bash
# Using curl — time connection
curl -o /dev/null -s -w "TCP Connect: %{time_connect}s\nTotal: %{time_total}s\n" http://TARGET_IP

# Full timing breakdown
curl -o /dev/null -s -w "
DNS Lookup:      %{time_namelookup}s
TCP Connect:     %{time_connect}s
TLS Handshake:   %{time_appconnect}s
First Byte:      %{time_starttransfer}s
Total:           %{time_total}s
" https://TARGET_IP
```

---

## 6. RTT in HTTP / HTTPS

```
HTTP  connection cost  = 1 RTT (TCP) + 1 RTT (request/response)
HTTPS connection cost  = 1 RTT (TCP) + 2 RTT (TLS 1.2) + 1 RTT (request)
HTTPS TLS 1.3          = 1 RTT (TCP) + 1 RTT (TLS 1.3) + 1 RTT (request)
HTTP/2                 = 1 RTT (TCP) + 1 RTT (TLS)  [multiplexed requests]
HTTP/3 (QUIC)          = 1 RTT total  [0-RTT resumption]
```

### RTT Cost per Protocol

| Protocol | RTTs to First Byte | Notes |
|---|---|---|
| HTTP/1.1 | 2 RTT | TCP + Request |
| HTTPS TLS 1.2 | 4 RTT | TCP + TLS 1.2 + Request |
| HTTPS TLS 1.3 | 3 RTT | TCP + TLS 1.3 + Request |
| HTTP/2 | 3 RTT | TCP + TLS + Multiplexed |
| HTTP/3 QUIC | 1–2 RTT | QUIC 0-RTT possible |

---

## 7. RTT vs Latency vs Jitter

```
┌─────────────────────────────────────────────────────────────┐
│                  KEY DIFFERENCES                            │
├─────────────────┬───────────────────────────────────────────┤
│  Term           │  Definition                               │
├─────────────────┼───────────────────────────────────────────┤
│  RTT            │  Full round-trip time (send + receive)    │
│  Latency        │  One-way delay (RTT / 2 approx.)         │
│  Jitter         │  Variation in latency over time           │
│  Throughput     │  Data transferred per unit time           │
│  Bandwidth      │  Max capacity of the link                 │
│  Packet Loss    │  % of packets not delivered               │
└─────────────────┴───────────────────────────────────────────┘
```

### Jitter Calculation

```
Jitter = | RTT_current - RTT_previous |

Example:
  RTT values: 10ms, 12ms, 9ms, 15ms, 11ms
  Jitter:     |12-10|, |9-12|, |15-9|, |11-15|
            = 2ms, 3ms, 6ms, 4ms
  Avg Jitter = (2+3+6+4) / 4 = 3.75ms
```

---

## 8. RTT Benchmarks

```
[*] Use these to identify network segment, detect VPNs, geo-locate targets
```

| Connection Type | Expected RTT |
|---|---|
| Loopback (127.0.0.1) | < 0.1 ms |
| Same LAN (Ethernet) | 0.1 – 1 ms |
| Same LAN (WiFi) | 1 – 5 ms |
| Same City (ISP) | 5 – 15 ms |
| Same Country | 10 – 40 ms |
| Cross-continent (US → EU) | 70 – 120 ms |
| Cross-continent (US → Asia) | 150 – 250 ms |
| Satellite (Geostationary) | 500 – 700 ms |
| Satellite (LEO / Starlink) | 20 – 60 ms |
| VPN (adds overhead) | +5 – 30 ms |
| Tor (3 hops) | 200 – 500 ms |

### OS Fingerprinting via TTL (Initial TTL)

```
[*] TTL in ping response reveals OS family

TTL = 64   → Linux / Unix / macOS
TTL = 128  → Windows
TTL = 255  → Cisco / Network devices / Solaris

[*] Actual received TTL = Initial TTL - number of hops

Example:
  Received TTL = 118
  Windows initial TTL = 128
  Hops = 128 - 118 = 10 hops away
```

---

## 9. RTT in Red Teaming & OPSEC

### C2 Beacon Timing via RTT

```
[*] RTT tells you how fast your C2 comms will be
[*] High RTT target = slow beacon response = increase timeout

# Cobalt Strike — adjust based on measured RTT
set sleeptime "60000";   # 60s sleep if RTT > 200ms
set jitter     "30";     # 30% jitter variation
```

### Detect Proxy / Sandbox via RTT Anomaly

```python
import subprocess, re

def get_rtt(host):
    result = subprocess.run(['ping', '-c', '3', host],
                            capture_output=True, text=True)
    match = re.search(r'avg.*?(\d+\.\d+)', result.stdout)
    return float(match.group(1)) if match else None

# Sandbox/VM has unusually low RTT to external (simulated network)
rtt = get_rtt('8.8.8.8')
if rtt is not None and rtt < 2.0:
    # Suspiciously fast → likely sandbox
    exit(0)
```

### Geo-Locate Target via RTT

```
[*] Combine RTT with known datacenter RTTs to approximate location

RTT to target = 12ms from US East Coast
→ Target is likely within 1,200 km of measurement point
→ Narrow down datacenter region

RTT reference:
  US-East → EU-West  ≈ 80ms
  US-East → US-West  ≈ 60ms
  US-East → Asia     ≈ 200ms
```

### Detect IDS/Firewall via RTT Spike

```bash
# Gradual RTT increase when firewall starts throttling
# Send packets and watch for RTT spike = throttling detected
hping3 -S -p 80 --flood TARGET_IP 2>&1 | grep rtt

# If RTT doubles/triples mid-scan → stateful firewall / IPS active
```

---

## 10. RTT-Based Network Recon

### Map Network Topology via RTT

```bash
# Identify routers and hops via RTT jumps in traceroute
traceroute -I TARGET_IP

# Large RTT jump between hops = geographic boundary or slow link
# * * * = filtered hop (firewall dropping ICMP TTL-exceeded)

# TCP traceroute (harder to filter)
traceroute -T -p 443 TARGET_IP
```

### Infer Network Segment via RTT

```
RTT 0.1 – 1ms    → Same broadcast domain / VLAN
RTT 1 – 5ms      → Adjacent VLAN / subnet (one router hop)
RTT 5 – 15ms     → Regional network / multiple hops
RTT > 50ms       → WAN / cross-region link
```

### Detect Load Balancers via RTT Variance

```bash
# Send multiple pings and look for different RTTs
# Load balancer routes to different backend servers
for i in {1..10}; do
  ping -c 1 TARGET_IP | grep time= | awk '{print $7}'
done

# Varying RTT across requests → load balancer detected
# Consistent RTT → single server
```

### OS TTL Fingerprinting via Ping

```bash
# Linux
ping -c 1 TARGET_IP | grep ttl

# Windows
ping TARGET_IP | findstr TTL

# Output:
# ttl=118 → Windows (128 - 10 hops)
# ttl=54  → Linux (64 - 10 hops)
# ttl=245 → Cisco (255 - 10 hops)
```

---

## 11. RTT Manipulation & Evasion

### Slow Scan to Avoid IDS Threshold

```bash
# IDS triggers on scan speed — use RTT-based delays
nmap -T1 --scan-delay 2000ms TARGET_IP

# Add delay equal to 2× measured RTT
RTT=$(ping -c 3 TARGET_IP | awk -F'/' 'END{print $5}')
DELAY=$(echo "$RTT * 2" | bc)
nmap --scan-delay ${DELAY}ms TARGET_IP
```

### Use RTT to Time Exploit Delivery

```bash
# Measure RTT before sending exploit
# Ensure target is reachable and responsive

RTT=$(ping -c 5 -q TARGET_IP | awk -F'/' 'END{print $5}')
echo "[*] Target RTT: ${RTT}ms"

if (( $(echo "$RTT < 100" | bc -l) )); then
    echo "[+] Low latency — proceed with exploit"
else
    echo "[!] High latency — adjust timeouts"
fi
```

### Tor RTT Consideration

```
[*] Tor adds 200–500ms RTT due to 3-hop onion routing
[*] Never use time-sensitive exploits over Tor
[*] Use for recon only — not for interactive shells
[*] Use proxychains for TCP-based tools over Tor
```

---

## 12. Tools & Commands

### Quick Reference Commands

```bash
# ICMP RTT
ping -c 10 TARGET_IP

# Per-hop RTT
traceroute -I TARGET_IP               # Linux (ICMP)
tracert TARGET_IP                     # Windows

# TCP RTT
hping3 -S -p 443 -c 5 TARGET_IP
nping --tcp -p 443 TARGET_IP

# HTTP RTT timing
curl -o /dev/null -s -w "Connect: %{time_connect}s | TTFB: %{time_starttransfer}s\n" https://TARGET

# Continuous RTT monitor
ping -i 0.2 TARGET_IP | awk '/time=/{print $7}' | sed 's/time=//'

# RTT to multiple targets
for ip in 10.10.10.1 10.10.10.2 10.10.10.3; do
  rtt=$(ping -c 3 -q $ip 2>/dev/null | awk -F'/' 'END{print $5}')
  echo "$ip → ${rtt}ms"
done
```

### Tool Summary

| Tool | Protocol | Use Case |
|---|---|---|
| `ping` | ICMP | Basic RTT measurement |
| `traceroute` / `tracert` | ICMP/UDP/TCP | Per-hop RTT, path discovery |
| `hping3` | TCP/UDP/ICMP | Custom RTT, firewall bypass |
| `nping` | Any | Nmap-integrated RTT test |
| `curl` | HTTP/S | Application-layer RTT |
| `mtr` | ICMP | Real-time per-hop RTT monitor |
| `wireshark` | Any | Capture & analyse RTT in pcap |
| `iperf3` | TCP/UDP | Throughput + jitter test |
| `netcat` | TCP | Manual TCP RTT test |

### mtr — Real-Time RTT Monitor

```bash
# Install
apt install mtr -y

# Run
mtr TARGET_IP

# Report mode (10 packets)
mtr --report --report-cycles 10 TARGET_IP

# Output shows per-hop RTT, loss, jitter in real time
```

---

## Quick Reference

```
[+] RTT formula     → Propagation + Transmission + Processing + Queuing (×2)
[+] Measure ICMP    → ping -c 10 TARGET
[+] Measure TCP     → hping3 -S -p 443 TARGET
[+] Measure HTTP    → curl -w "%{time_connect}" TARGET
[+] OS fingerprint  → TTL 64=Linux, 128=Windows, 255=Cisco
[+] Geo estimate    → RTT < 15ms=local, 70ms=cross-continent
[+] C2 timing       → Set beacon sleep > RTT, add jitter
[+] Sandbox detect  → RTT < 2ms to 8.8.8.8 = suspicious
[+] IDS evasion     → scan-delay = 2× RTT
[+] Topology map    → RTT jump in traceroute = router boundary
```

---

## ⚠️ Disclaimer

```
[!] For EDUCATIONAL and AUTHORIZED TESTING purposes only.
[!] Use only on systems you own or have explicit permission to test.
[!] Author is NOT responsible for any misuse or illegal activity.
```

---

## License

MIT
