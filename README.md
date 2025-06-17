# THM-NMAP
Advanced Nmap reconnaissance lab showcasing host discovery, TCP SYN (-sS), UDP (-sU), OS (-O) and service/version (-sV) fingerprinting, timing control (-T0–T5), and multi-format output (-oA) for network enumeration.

By Ramyar Daneshgar 

## **Task 1: Host Discovery**

### **Scenario**

I began by simulating a reconnaissance phase against both local and remote network segments. My objective was to identify which hosts are online, using ICMP echo requests, ARP scans, and TCP SYN/ACK probes.

### **Execution**

I issued the following command:

```bash
nmap -sn 192.168.66.0/24
```

This is a *ping scan* using the `-sn` flag, which tells `nmap` not to perform port scanning but to detect live hosts. On a local subnet, `nmap` uses **ARP requests** to identify hosts at Layer 2 of the OSI model.

For remote scans:

```bash
nmap -sn 192.168.11.0/24
```

Since ARP is not routable, `nmap` falls back to sending **ICMP echo requests**, **TCP SYN to port 443**, and **TCP ACK to port 80** to infer host responsiveness.

### **Lessons Learned**

* ARP scanning is only viable on directly connected subnets.
* Firewalls or ACLs dropping ICMP packets can cause false negatives.
* Use `-Pn` to bypass host discovery when firewall evasion is required.

---

## **Task 2: Port Scanning**

### **Connect Scan**

```bash
nmap -sT 192.168.1.1
```

This scan completed the **full TCP three-way handshake** on each port, suitable when raw packet crafting is not possible (non-root users). This increases detectability in IDS/IPS logs.

### **SYN Scan (Stealth Scan)**

```bash
sudo nmap -sS 192.168.1.1
```

I ran this using root privileges. It sent only the initial **SYN** packet and analyzed the response:

* SYN/ACK → port open
* RST → port closed

This method avoids completing the handshake, making it stealthier and less likely to be logged.

### **UDP Scan**

```bash
sudo nmap -sU 192.168.1.1
```

UDP lacks built-in handshakes. I used this to scan for stateless services like DNS or SNMP. Responses like **ICMP port unreachable** helped determine closed ports.

### **Lessons Learned**

* SYN scan is preferred for stealth.
* UDP scans are slow and imprecise but necessary for completeness.
* Non-root users default to TCP connect scan, limiting stealth.

---

## **Task 3: Port Selection**

To optimize for speed or coverage:

* `-F` scans the top 100 common ports
* `-p-` scans all 65535 ports
* `-p1-1024` or custom ranges narrow the scan window

```bash
nmap -sS -p1-1024 192.168.1.1
```

---

## **Task 4: Version and OS Detection**

### **Version Detection**

```bash
nmap -sV 10.10.100.85
```

This returned:

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
```

### **OS Fingerprinting**

```bash
nmap -O 10.10.100.85
```

This matched TCP/IP stack behavior to an internal signature database, showing:

```
Linux 4.15 - 5.8
```

### **Comprehensive Scan**

```bash
nmap -A 10.10.100.85
```

Combined OS detection, version scanning, and traceroute.

### **Lessons Learned**

* Combine `-sV`, `-O`, and `-A` for rich fingerprinting.
* False positives are possible; interpret results cautiously.
* OS detection can assist in vulnerability prioritization.

---

## **Task 5: Timing and Performance**

### **Timing Templates**

```bash
nmap -sS -T0 10.10.100.85
```

The scan took several minutes. I incrementally tested:

* `-T1` → slow, evades IDS
* `-T3` → default, balanced
* `-T4` → aggressive, faster scans
* `-T5` → fastest, risky on noisy networks

### **Rate Control**

```bash
nmap -sS --min-rate 100 --max-rate 1000 10.10.100.85
```

### **Host Timeout**

```bash
nmap --host-timeout 60s 10.10.100.85
```

This terminated scanning if the host did not respond within 60 seconds.

### **Lessons Learned**

* Use slower timing when evasion is key.
* Control bandwidth usage with rate limiting.
* Timing impacts reliability, especially on unstable networks.

---

## **Task 6: Output and Reporting**

### **Verbose Output**

```bash
nmap -v 192.168.1.1
nmap -vv -d3 192.168.1.1
```

Increasing verbosity helped trace each phase of the scan in real time, from ARP ping to DNS resolution and TCP response parsing.

### **Output Formats**

```bash
nmap -sS -oA scan_results 192.168.1.1
```

This saved:

* `scan_results.nmap` (normal)
* `scan_results.xml` (machine-readable)
* `scan_results.gnmap` (grepable)

These formats are valuable for automated parsing, incident documentation, or integration with SIEM/SOAR pipelines.

---

## **Task 7: Default Scan Behavior (User Privilege)**

When I ran:

```bash
nmap 10.10.100.85
```

without `sudo`, Nmap defaulted to a **TCP Connect scan (-sT)**. This is less stealthy and limited in customization. With root, it defaulted to **SYN scan (-sS)**, offering more control and flexibility in packet crafting.

---

## **Key Nmap Flags Summary**

| Option         | Purpose                                    |
| -------------- | ------------------------------------------ |
| `-sn`          | Ping scan for host discovery               |
| `-sS`          | SYN scan (stealth)                         |
| `-sT`          | TCP connect scan                           |
| `-sU`          | UDP scan                                   |
| `-p-`          | Scan all 65535 ports                       |
| `-F`           | Fast scan (100 ports)                      |
| `-sV`          | Service and version detection              |
| `-O`           | OS fingerprinting                          |
| `-A`           | Aggressive scan with all detection modules |
| `-T0` to `-T5` | Scan timing templates                      |
| `-oA`          | Save output in all formats                 |
| `-v`, `-d`     | Verbose/debug output                       |
| `-Pn`          | Treat all hosts as online                  |

---

## **Final Lessons Learned**

1. **Host discovery** is context-dependent—use ARP for local, ICMP/TCP probes for remote.
2. **Scan stealth** and accuracy depend on technique (`-sS` vs `-sT`) and privilege level.
3. **Output formatting** is vital for integrating into larger threat intelligence workflows.
4. **Performance tuning** matters in evasive red-team ops and high-latency environments.
5. **Nmap is not just a port scanner**—it's a fingerprinting and recon framework when used fully.

