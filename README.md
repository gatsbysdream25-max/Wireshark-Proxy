# Wireshark-Proxy
Home Security lab
DevSecOps Home Lab
Proxychains4 + Tor + Wireshark
Anonymous Network Scanning & Traffic Analysis
Overview
This document covers the setup and use of Proxychains4 with Tor for anonymous network reconnaissance, and Wireshark for real-time traffic verification — core skills in DevSecOps and penetration testing.

Tool	Purpose
Tor	Routes traffic through 3 encrypted relays, masking real IP
Proxychains4	Forces command-line tools (nmap, curl) through Tor
Wireshark	Captures and inspects network packets in real time
Nmap	Network scanner used for port discovery and recon
Ubuntu 64-bit VM	Isolated environment for all security tooling

1. Installing & Configuring Tor
Installation
sudo apt update
sudo apt install tor -y

Start & Enable Tor Service
sudo systemctl start tor
sudo systemctl enable tor
sudo systemctl status tor

Tor runs as a background service on port 9050 (localhost). No manual interaction is needed after this — it runs silently and restarts automatically on reboot.

2. Installing & Configuring Proxychains4
Installation
sudo apt install proxychains4 -y

Edit Configuration
sudo nano /etc/proxychains4.conf

Make the following changes in the config file:

1.	Uncomment dynamic_chain (remove the #)
2.	Comment out strict_chain (add a # in front)
3.	Ensure the proxy list at the bottom contains:

dynamic_chain
#strict_chain
socks5  127.0.0.1 9050

NOTE: dynamic_chain skips dead proxies automatically. strict_chain fails the entire connection if any proxy in the chain is unavailable — less reliable through Tor relays.

3. Verifying Tor Routing
Check Real IP vs Proxychains IP
curl ifconfig.me
proxychains4 curl https://icanhazip.com

The two IP addresses should be completely different. The second command routes through Tor and will show a Tor exit node IP — not your real Spectrum/ISP IP.

NOTE: Some sites like ifconfig.me actively block known Tor exit nodes and return a 403 Forbidden. This is actually confirmation that Tor is working — your real IP is hidden and the Tor exit node was detected.

Verify with Tor Project
proxychains4 curl https://check.torproject.org | grep -i congratulations

A successful result will return text confirming your traffic is routed through Tor.

4. Anonymous Nmap Scanning
Requirements
TCP connect scan (-sT) is required because SYN scans use raw packets that cannot be forwarded through Tor. ICMP ping must also be disabled (-Pn) since ICMP does not route through Tor.

Basic Scan — Legal Practice Target
proxychains4 nmap -sT -Pn scanme.nmap.org

Faster Scan — Top 100 Ports with Verbose Output
proxychains4 nmap -sT -Pn --top-ports 100 scanme.nmap.org -v

Specific Port Scan
proxychains4 nmap -sT -Pn -p 22,80,443,8080 scanme.nmap.org

Flag	Explanation
-sT	TCP connect scan — works through Tor (SYN scan does not)
-Pn	Skip ping — ICMP does not route through Tor
--top-ports 100	Scan 100 most common ports (faster than full scan)
-v	Verbose — shows results as discovered rather than at the end
scanme.nmap.org	Legal scan target maintained by the Nmap project

NOTE: Scans through Tor are significantly slower than direct scans due to the 3-relay routing. Expect 5-20 minutes for a full scan vs seconds for a direct scan. This is normal.

Scan Results — scanme.nmap.org
Completed Connect Scan at 07:18, 59.31s elapsed (100 total ports)
Nmap scan report for scanme.nmap.org (224.0.0.1)
Host is up (0.37s latency)
Not shown: 98 closed tcp ports (conn-refused)
PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  open   http
Nmap done: 1 IP address (1 host up) scanned in 59.34 seconds

Wireshark confirmed all traffic routed through 127.0.0.1:9050 (Tor). TCP SYN/ACK/FIN handshakes visible in packet capture but content unreadable due to Tor encryption — confirming anonymity. Closed ports returned conn-refused via Tor dynamic chain as expected.

5. Wireshark — Traffic Verification
Installation
sudo apt install wireshark -y

When prompted: 'Should non-superusers be able to capture packets?' — select Yes.

Add User to Wireshark Group
sudo usermod -aG wireshark $USER
newgrp wireshark

After adding yourself to the wireshark group, fully log out and back in to make it permanent. You can then run Wireshark without sudo.

Launch Wireshark
wireshark &

The & runs Wireshark in the background, keeping your terminal free for running proxychains and nmap simultaneously.

Interface Selection
Interface	What It Shows
lo (loopback)	Traffic between your machine and Tor — use this to see proxychains activity
enp0s3	External network traffic — encrypted Tor relay packets leaving your VM

Key Filters
Filter	What It Captures
tcp.port == 9050	All traffic to/from Tor (use on lo interface)
tcp	All TCP traffic on selected interface
dns	All DNS queries — verify they route through Pi-hole
ip.addr == 127.0.0.1	Loopback traffic only

What You Should See
•	On lo with filter tcp.port == 9050: packets flowing during nmap scan — confirming traffic routes through Tor locally
•	On enp0s3: encrypted packets — Wireshark cannot read inside the Tor tunnel, confirming encryption is working
•	Real destination IP (scanme.nmap.org) is NOT visible in enp0s3 captures — only Tor relay IPs appear

6. Full Workflow — Scan + Capture Together
Run all three together for real-time anonymous scanning with live traffic verification:

Terminal 1 — Verify Tor is Running
sudo systemctl status tor

Terminal 2 — Launch Wireshark
wireshark &
Select lo interface, apply filter: tcp.port == 9050

Terminal 3 — Run Anonymous Scan
proxychains4 nmap -sT -Pn --top-ports 100 scanme.nmap.org -v

Watch Terminal 3 output while Wireshark shows the corresponding packet flow on lo — this is real-time confirmation that your scan traffic is anonymized through Tor.

7. DevSecOps Skills Demonstrated
Skill	Application
Anonymous Reconnaissance	Routing nmap through Tor via Proxychains — core red team technique
DNS Leak Prevention	Verifying traffic routes correctly without exposing real IP
Packet Analysis	Using Wireshark to verify encryption and traffic flow
Linux Administration	Service management, group permissions, CLI tooling
Network Security	Understanding tunneling, proxy chains, and traffic isolation
OPSEC Fundamentals	Separating identities through VM isolation + Tor routing

All scanning performed on legal targets only (scanme.nmap.org). Built for DevSecOps learning and portfolio development.
