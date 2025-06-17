
# **# THM-NMAP**

**Advanced Nmap reconnaissance lab demonstrating systematic host discovery, TCP SYN (-sS), UDP (-sU), OS (-O), and service/version (-sV) fingerprinting, timing control (-T0–T5), and multi-format output (-oA) for high-fidelity network enumeration and adversarial simulation.**
**By Ramyar Daneshgar**

---

## **Task 1: Host Discovery**

### **Scenario**

To emulate an adversary’s first step in the Cyber Kill Chain (Reconnaissance), I initiated an assessment of both local and remote network environments. The goal was to identify live hosts that could potentially expose open services. In operational environments, enumerating responsive IPs is critical to scope lateral movement and target prioritization.

### **Execution Logic**

For **local discovery**, I executed:

```bash
nmap -sn 192.168.66.0/24
```

The `-sn` flag triggers a "ping scan" that suppresses port probing. On local layer-2 networks, Nmap intelligently defaults to **ARP requests** instead of ICMP, bypassing firewall-level ICMP filtering and directly querying MAC addresses via Ethernet.

For **remote subnets** beyond the first hop, I used:

```bash
nmap -sn 192.168.11.0/24
```

Because ARP is not routable, Nmap adapted by sending a sequence of:

* **ICMP Echo Requests**
* **TCP SYN packets to port 443**
* **TCP ACK packets to port 80**

These probes increase response likelihood even when ICMP is blocked, mimicking attacker persistence in hostile environments.

---

## **Task 2: Port Scanning**

### **Connect Scan (-sT)**

```bash
nmap -sT 192.168.1.1
```

This scan performs a full TCP three-way handshake to test if a port is open. It's used when raw sockets are unavailable (non-root). However, it's **highly detectable**, as it completes the connection and may generate logs in host-level auditing tools like auditd or SIEM platforms.

### **SYN Scan (-sS)**

```bash
sudo nmap -sS 192.168.1.1
```

With root privileges, I triggered a **half-open scan** using SYN packets. If the target replies with a SYN/ACK, Nmap sends an immediate RST, avoiding full connection establishment. This mimics common red team stealth behavior — useful in evading connection logs, IDS sensors, and host-based firewalls.

### **UDP Scan (-sU)**

```bash
sudo nmap -sU 192.168.1.1
```

Unlike TCP, UDP lacks session state. I sent payload-less UDP packets to common service ports. When the port was closed, the host returned **ICMP Port Unreachable (Type 3, Code 3)**. Silence implied a possible open or filtered port. UDP scanning is resource-heavy but necessary for discovering DNS, SNMP, TFTP, and other stateless services.

---

## **Task 3: Port Selection Strategy**

To tailor scan scope and reduce noise or increase thoroughness:

```bash
nmap -sS -p1-1024 192.168.1.1
```

* `-F`: scans top 100 ports based on frequency.
* `-p-`: full range (1–65535).
* Custom ranges allow targeted enumeration of known service ports.

In operational use, I would scan privileged ports first (1–1024), then schedule exhaustive scans asynchronously.

---

## **Task 4: Version and OS Detection**

### **Service Fingerprinting**

```bash
nmap -sV 10.10.100.85
```

This probe initiated **banner grabbing**, TLS handshakes, and service-specific probes to fingerprint running software and protocol versions. Returned output included:

```
22/tcp open  ssh  OpenSSH 8.9p1
```

Crucial for vulnerability matching using CVEs or security advisories.

### **OS Detection**

```bash
nmap -O 10.10.100.85
```

Nmap compared TCP/IP stack behavior (e.g., TCP window size, TTL, DF bit, sequence predictability) against its internal fingerprint database to infer kernel versions and OS family.

### **Aggressive Detection (-A)**

```bash
nmap -A 10.10.100.85
```

Bundled `-O`, `-sV`, traceroute, and default NSE scripts. I used this when needing a broad fingerprint in a single pass, typically during rapid target profiling.

---

## **Task 5: Timing and Performance Tuning**

### **Scan Timing Templates**

```bash
nmap -sS -T0 10.10.100.85
```

Nmap timing levels (T0–T5) control:

* Delay between probes
* Number of parallel threads
* Retransmission behavior

Used T0 (paranoid) to simulate evasion scenarios and T4 (aggressive) for rapid scans in controlled or cooperative environments.

### **Rate and Parallelism Control**

```bash
nmap --min-rate 100 --max-rate 1000 10.10.100.85
```

Tuning packet-per-second transmission rate helped simulate DDoS-style reconnaissance or throttle load during scans on latency-sensitive links.

```bash
nmap --host-timeout 60s 10.10.100.85
```

This ensured the scan abandoned unresponsive hosts quickly — essential when scanning large subnets to avoid delays.

---

## **Task 6: Output & Reporting**

### **Real-Time Visibility**

```bash
nmap -vv -d3 192.168.1.1
```

Verbose and debug levels gave visibility into:

* ARP/DNS phases
* Port state transitions
* Probing retries

Ideal for SOC visibility or forensics validation.

### **Structured Output**

```bash
nmap -sS -oA scan_results 192.168.1.1
```

Saved output as:

* `.nmap` → human-readable
* `.xml` → machine-readable for parsing
* `.gnmap` → grepable (for use with `awk`, `grep`, `cut`)

Supports automation pipelines for red team reporting or integration with SIEM/SOAR.

---

## **Task 7: Default Behavior Based on Privilege**

```bash
nmap 10.10.100.85
```

Without elevated privileges, Nmap used a TCP Connect scan (`-sT`), relying on the OS's socket API. This is verbose on target logs. With `sudo`, Nmap preferred the raw packet SYN scan (`-sS`) due to its flexibility and stealth.

---


| Flag        | Description                                                               |
| ----------- | ------------------------------------------------------------------------- |
| `-sn`       | Host discovery only (ping scan)                                           |
| `-sS`       | Stealth TCP SYN scan                                                      |
| `-sT`       | Full TCP Connect scan                                                     |
| `-sU`       | UDP port scan                                                             |
| `-p-`       | All ports (1–65535)                                                       |
| `-F`        | Fast scan (top 100 ports)                                                 |
| `-sV`       | Service and version detection                                             |
| `-O`        | Operating system detection                                                |
| `-A`        | Aggressive scan (OS + service + script + traceroute)                      |
| `-T0`–`-T5` | Timing profiles (paranoid to insane)                                      |
| `-oA`       | Output in all formats: .nmap, .gnmap, .xml                                |
| `-v`, `-d`  | Verbosity and debug mode for real-time scan monitoring                    |
| `-Pn`       | Skip host discovery; assume all IPs are live (bypasses ICMP restrictions) |

---

## **Final Lessons Learned**

1. **Network topography matters** — different discovery techniques are required depending on whether you’re scanning local (layer-2) or remote (routed) networks. ARP fails on remote networks, while ICMP and TCP probes may be blocked.
2. **Privilege level dictates scan stealth** — non-root scans generate more noise and are limited in technique.
3. **Use timing templates and rate controls to evade detection or tune performance** — slower scans minimize IDS alerts, faster scans expedite recon in time-sensitive environments.
4. **Structured output is key for reporting and automation** — especially in professional pentest or compliance engagements.
5. **Nmap is a foundational tool not just for enumeration, but for understanding target architecture and shaping post-recon strategy.**

