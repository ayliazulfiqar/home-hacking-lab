# home-lab
An isolated lab for practising penetration testing legally.

## Tools chosen
- Linux Mint: host operating system.
- VirtualBox: runs the virtual machines.
- Kali Linux: security-testing machine.
- Metasploitable 2: intentionally vulnerable practice target.

## Safety plan
- Keep the target on a private host-only network.
- Never use bridged networking for the target.
- Keep the target disconnected from the internet.
- Take snapshots before practical exercises.
- Test only systems I own or have permission to test.

## Lab diagram
[Home lab diagram](lab-diagram.drawio.png)

## Problems I hit
- None yet.

## Network Setup

| Machine | Role | Network adapters |
|---|---|---|
| Kali Linux | Testing machine | Adapter 1: NAT; Adapter 2: Host-only (`vboxnet0`) |
| Metasploitable 2 | Target machine | Adapter 1: Host-only (`vboxnet0`) |

- **Host-only network:** `vboxnet0`
- **Network range:** `192.168.56.x`
- **Kali IP:** `192.168.56.103`
- **Metasploitable IP:** `192.168.56.102`

Kali has a NAT adapter for internet access when needed and a host-only adapter for communicating with the target. Metasploitable uses only the host-only adapter, keeping it separate from the normal network.

## Safety Plan

- Keep Metasploitable on the private host-only network.
- Never use bridged networking for the vulnerable target.
- Keep the target disconnected from the internet.
- Take VM snapshots before practical exercises.
- Only test systems I own or have permission to test.
- Keep scanning and testing inside this lab.

## Verification

| Test | Expected result | Actual result |
|---|---|---|
| Kali discovers the target | Target is reachable | Nmap confirmed `192.168.56.102` was up |
| Metasploitable pings `8.8.8.8` | Internet connection unavailable | `Network is unreachable` |

The Nmap scan confirmed that Kali could reach the target. The target's ping test showed that it had no available network route to `8.8.8.8`.

The target's failed ping is a useful check, but I should also verify its VirtualBox adapter settings and routing configuration to confirm the intended isolation.

![Ping test](ping-tes.png)

## Snapshots

The planned snapshot name for both virtual machines is `clean-baseline`. VirtualBox snapshots belong to individual VMs, so both VMs can use the same snapshot name.

**Status:** Confirm that the snapshot has been taken for each VM before stating that both snapshots are complete.

## First Scan

The initial service-discovery scan was:

```bash
nmap -sV 192.168.56.102 -oN scan1.txt
```

The extended scan was:

```bash
nmap -sV -sC -p- 192.168.56.102 -oN full-scan.txt
```

The extended scan checked all 65,535 TCP ports and completed in approximately 153 seconds.

### Scan Results

- **Target:** `192.168.56.102`
- **Host status:** Up
- **Open TCP ports:** 29
- **Closed TCP ports:** 65,505
- **Total TCP ports scanned:** 65,535
- **Saved output:** `full-scan.txt`

### Services Identified

The scan identified several services, including:

- FTP — port 21 (`vsftpd 2.3.4`); anonymous FTP login was allowed.
- SSH — port 22 (`OpenSSH 4.7p1`).
- Telnet — port 23.
- HTTP — port 80 (`Apache 2.2.8`, with DAV/WebDAV reported).
- Samba — ports 139 and 445 (`Samba 3.0.20`).
- MySQL — port 3306.
- PostgreSQL — port 5432.
- VNC — port 5900.
- Tomcat — port 8180.
- Additional services included NFS, distccd, IRC, and a service identified on port 1524 as a Metasploitable root shell.

These versions and services are old and may contain known security weaknesses. A scan identifies potential risks; further controlled testing would be needed to confirm which vulnerabilities are exploitable.

### Three Services That Stood Out

- **FTP (port 21):** Anonymous login was allowed. This could expose files to users who should not have access. Traditional FTP also does not encrypt credentials or data.
- **HTTP (port 80):** The server reported an old Apache version and WebDAV support. Outdated software and misconfigured file-authoring features can increase the risk of unauthorized access.
- **Telnet (port 23):** Telnet does not encrypt login credentials or session traffic, so information may be exposed if someone can observe the network.

![Nmap results](full-nmap-scan.png)

## Problems I Hit

- **Understanding the VM network setup:** I needed to distinguish the NAT adapter from the host-only adapter. I configured Kali with both adapters and kept Metasploitable on host-only networking.
- **Target internet connectivity:** Running `ping -c 4 8.8.8.8` on Metasploitable returned `Network is unreachable`. This indicates that no usable route was available for that destination.
- **Finding the target's services:** I ran an extended Nmap scan using `-sV`, `-sC`, and `-p-` to identify service versions, run default scripts, and scan all TCP ports.

## What I Learned

I learned how to identify the IP addresses of my lab machines, check their network configuration, and use Nmap to discover open ports and service versions. I also learned that services such as anonymous FTP, unencrypted Telnet, and outdated web servers can introduce security risks.

## Next Steps

- Session 3: capture a scan in Wireshark and write up the results
- Practice attacks on DVWA inside the lab

## Wireshark capture of the scan

**Goal:** see what an Nmap scan actually looks like as network traffic.

**Setup:** I started a capture on Kali's host-only interface ([eth1]), then ran `nmap -sS [target-ip]` against the target. I stopped the capture when the scan finished.

**Filters I used:**
- `ip.addr == [target-ip]` to show only lab traffic
- `tcp.flags.syn == 1 && tcp.flags.ack == 0` to show only the first packet of each connection attempt
- `tcp.flags.reset == 1` to show closed ports

**What I observed:**
- **Open port:** Kali sent a SYN, the target replied with SYN-ACK, and Kali sent a RST instead of completing the handshake. That's why it's called a "half-open" or SYN scan.
- **Closed port:** Kali sent a SYN and the target replied with RST-ACK.
- **Volume and speed:** [e.g. about X packets in Y seconds, sent to many different ports in rapid order]

![Wireshark: SYN scan](screenshots/wireshark-syn-scan.png)

**Why this matters:** a pattern of SYN packets to many ports from one source, in a short time, is a recognisable sign of a port scan. A defender could spot it in logs, which is something I plan to explore in Project 5.

**What I learned:**
- [e.g. How Nmap decides a port is open or closed from the reply it gets]
- [e.g. How the TCP handshake I studied in week one appears in real traffic]

## Web Attack Practice

### 1. SQL Injection
I tested the SQL Injection page in DVWA and learned how unsafe user input can affect database queries. Developers can prevent this by using parameterised queries.
- Vulnerable input: User ID field.
- What I typed: A SQL injection test input in the user ID field.
- What happened: The application returned user information from the database.
- Developer fix: Use parameterised queries to prevent user input from changing SQL commands.
  
**Screenshot:** `screenshots/sql-injection.png`

### 2. Reflected XSS
I tested the Reflected XSS page and learned how unsafe input can cause JavaScript to execute in the browser. Developers should use proper output encoding.
- Vulnerable input: Name input field.
- What I typed: A harmless JavaScript alert test.
- What happened: The browser displayed an alert box because the input was handled unsafely.
- Developer fix: Use context-aware output encoding and safely handle user input.

**Screenshot:** `screenshots/reflected-xss.png`

### 3. Stored XSS
I tested the Stored XSS page and learned that unsafe input can be saved and affect users who visit the page later. Developers should safely handle user-generated content.
- Vulnerable input: Guestbook message field.
- What I typed: A harmless alert test as a message.
- What happened: The message was saved and triggered an alert when the page was loaded again.
- Developer fix: Safely encode user-generated content before displaying it.

**Screenshot:** `screenshots/stored-xss.png`

### 4. Brute Force
I tested the DVWA login page using Burp Suite Intruder with a small list of test credentials. I learned how to compare HTTP responses to identify potentially successful login attempts. Rate limiting and multi-factor authentication can help protect login pages.
- Vulnerable input: DVWA login form.
- What I typed: A small list of test usernames and passwords in Burp Intruder.
- What happened: I compared HTTP response messages and lengths to identify potentially successful login attempts.
- Developer fix: Implement rate limiting, multi-factor authentication, and monitoring for repeated failed logins.

**Screenshot:** `screenshots/brute-force.png`

## Results
- Built an isolated lab (Kali attacker, Metasploitable target) on a host-only network.
- Verified isolation: Kali reaches the target, the target has no internet access.
- Scanned the target with Nmap and captured the traffic in Wireshark.
- Exploited SQL injection and XSS on DVWA, and tried brute force with Burp Intruder, all inside the lab.

## Skills demonstrated
Virtualization and network isolation, Nmap scanning, packet analysis in Wireshark, Burp Suite, web vulnerability testing, safe lab practice, technical documentation.

## What I learned
- [2-3 sentences in your own words, such as why isolation matters or what surprised you]

## Limitations and next improvements
- [e.g. Only one target machine so far]
- [e.g. No logging or detection yet, which I'll add in Project 5 with Wazuh]
