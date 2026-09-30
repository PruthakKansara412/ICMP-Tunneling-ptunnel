# ICMP-Tunneling-ptunnel
ICMP tunneling demo with ptunnel, firewall bypass, and Wireshark traffic analysis.


# Exploring ICMP Tunneling with ptunnel

A practical demonstration of **ICMP tunneling** — a technique that hides data inside ICMP (ping) packets to bypass firewall restrictions and covertly forward traffic (in this case, SSH) between two machines.

>  **Disclaimer:** This project was built in an isolated lab for educational purposes only. ICMP tunneling can be used to bypass security controls and exfiltrate data — never use it on networks you don't own or have permission to test.

---

##  Objectives

- Set up two virtual machines: Kali Linux (attacker) and Ubuntu Server (victim)
- Configure the victim's firewall to allow only ICMP and SSH traffic
- Install and configure `ptunnel` on both machines
- Establish an ICMP tunnel to forward SSH traffic from attacker to victim
- Simulate data exfiltration over the tunnel and open an SSH session
- Capture and analyse the ICMP traffic using Wireshark

---

##  Tools & Technologies

| Tool | Purpose |
|------|---------|
| VirtualBox | Virtualization platform |
| Kali Linux 2024.3 | Attacker machine (VM1) |
| Ubuntu Server 22.04 LTS | Victim machine (VM2) |
| ptunnel | Tool for tunneling TCP traffic over ICMP |
| iptables | Firewall configuration on the victim |
| Wireshark | Packet capture and traffic analysis |
| Nmap | Verifying which ports are reachable |

---

##  Lab Setup

| Machine | Role | IP Address |
|---------|------|------------|
| Ubuntu Server (VM2) | Victim | 10.0.2.4 |
| Kali Linux (VM1) | Attacker | 10.0.2.5 |

Both VMs were set to **NAT network** mode in VirtualBox.

---

##  Firewall Configuration (Victim Machine)

To simulate a restricted network where only ICMP and SSH are allowed:

```bash
# Drop all incoming/forwarded traffic by default, allow all outgoing
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow outgoing traffic
sudo iptables -A OUTPUT -j ACCEPT

# Allow incoming ICMP (ping)
sudo iptables -A INPUT -p icmp -j ACCEPT

# Allow incoming SSH (needed for ptunnel)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow established/related connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Verified from the attacker machine with `ping` and `nmap`, confirming that only ICMP and port 22 (SSH) were reachable.

---

##  Ptunnel Installation

Installed on **both** machines:

```bash
sudo apt install ptunnel
```

---

##  Execution

**On the victim machine (VM2)** — start ptunnel in listening/proxy mode:
```bash
sudo ptunnel
```

**On the attacker machine (VM1)** — start the tunnel toward the victim:
```bash
sudo ptunnel -p 10.0.2.4 -lp 2222 -da 127.0.0.1 -dp 22
```

| Flag | Meaning |
|------|---------|
| `-p 10.0.2.4` | Target IP (victim machine) |
| `-lp 2222` | Local port on the attacker that ptunnel listens on |
| `-da 127.0.0.1` | Destination address on the target (localhost) |
| `-dp 22` | Destination port on the target (SSH) |

This forwards any traffic sent to local port `2222` through the ICMP tunnel to port `22` (SSH) on the victim, using ICMP echo request/reply packets as the carrier.

**Connecting over the tunnel:**
```bash
ssh -p 2222 abcd123@localhost
```
This opens a real SSH session with the victim machine, even though the firewall only permits ICMP and SSH traffic directly — demonstrating how ICMP tunneling can smuggle other protocols through a restrictive firewall.

---

##  Traffic Analysis (Wireshark)

Wireshark captured the ICMP Echo Request/Reply packets exchanged during the tunnel session.

**Observed packet details:**
- **Source:** 10.0.2.5 (attacker)
- **Destination:** 10.0.2.4 (victim)
- **Protocol:** ICMP (ping)
- **Packet length:** 70 bytes
- **Example:** Echo Request with ID `0x421c`, sequence `577`, TTL `64`, matched by a corresponding Echo Reply with the same ID and sequence number

This confirmed a steady, request/reply pattern consistent with tunneled data riding inside what looks like ordinary ping traffic.

---

##  Key Findings

1. **ICMP as a covert channel** — Tunneling data inside ICMP echo packets lets traffic slip past firewalls that only inspect protocol type, not payload content.
2. **Traffic analysis matters** — Wireshark made it possible to see the ICMP request/reply pattern and identify the tunneled session.
3. **Detection is hard** — Standard IDS/IPS rules that just flag "ICMP traffic" won't catch this; detecting ICMP tunneling requires deeper packet/payload inspection and behavioural analysis (e.g., abnormal packet sizes, timing, or volume).
4. **Practical takeaway** — This reinforced why network defenders need more than basic protocol-based firewall rules to catch covert channels and data exfiltration attempts.

---

##  Future Improvements

- Write Snort/Suricata rules to flag anomalous ICMP payload sizes or request/reply timing that suggest tunneling
- Compare normal ping traffic vs. tunneled traffic side-by-side in Wireshark
- Extend the tunnel to exfiltrate a file and measure throughput/detection difficulty
- Test detection using an IDS (e.g., Snort) alongside this setup

---
