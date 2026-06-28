Project Overview

This project demonstrates a hands-on Purple Team exercise conducted in an isolated virtual environment. By playing the role of both Attacker (Kali Linux) and Defender/Analyst (Wireshark on a Windows Host), I simulated an internal reconnaissance and credential harvesting attack vector — then analyzed its full packet-level footprint and documented strategic mitigations.


Purple Team = Red Team (Offense) + Blue Team (Defense) working together to expose and close gaps.




🛠️ Infrastructure & Lab Topology

All traffic was contained within an isolated Host-Only Network (VMnet1) using VMware Workstation, making it fully capturable by the host's packet sniffer.

RoleMachineNotes🖥️ Analysis Host (Sniffer)Windows 11 HostWireshark listening on VMnet1⚔️ Attacker NodeKali Linux 2025.3Offensive tooling🎯 Target NodeMetasploitable2Intentionally vulnerable VM


⚔️ Phase 1: Offensive Execution (Red Team)

Step 1 — Automated Network Reconnaissance

An aggressive SYN stealth port scan was executed from Kali Linux to enumerate open ports and identify available services on the target.
<img width="1600" height="832" alt="WhatsApp Image 2026-06-28 at 1 34 30 PM" src="https://github.com/user-attachments/assets/e46e8651-179d-4186-9c70-ad140dbc8883" />

bash:sudo nmap -sS -p 1-1000 [Target_IP]

Objective: Map the attack surface before selecting an exploitation vector.


Step 2 — Unencrypted Credential Harvesting

Upon discovering port 21 (FTP) was active, an unencrypted connection was initiated to simulate a real-world authentication attempt — passing credentials in cleartext across the wire.
<img width="546" height="283" alt="Screenshot 2026-06-28 143358" src="https://github.com/user-attachments/assets/8ae46cc3-826e-45c4-9dfc-bc90de8d208f" />

bash:ftp [Target_IP]

Objective: Demonstrate that legacy, unencrypted protocols expose sensitive data to any passive observer on the network.


🛡️ Phase 2: Defensive Analysis & Detection (Blue Team)

Switching to the role of a SOC Analyst, the packet capture (.pcap) recorded from the Windows Host via VMnet1 was examined in Wireshark to surface Indicators of Compromise (IoCs).


Finding 1 — Port Scan Signature Identified

Wireshark Display Filter:
<img width="1285" height="679" alt="Screenshot 2026-06-28 134939" src="https://github.com/user-attachments/assets/97b1b79e-3436-41a8-9870-f7b29f5e1275" />

tcp.flags.syn == 1 and tcp.flags.ack == 0

Technical Findings:

The capture revealed a massive, rapid influx of TCP SYN packets originating from the attacker IP, targeting sequential destination ports within milliseconds. This compressed burst pattern is a definitive indicator of automated scanning — no legitimate user interaction produces traffic at this velocity or across this many ports in sequence.


Finding 2 — Cleartext Credentials Extracted

Wireshark Display Filter:
<img width="1563" height="589" alt="Screenshot 2026-06-28 134907" src="https://github.com/user-attachments/assets/ba3cb0b5-ad5f-4394-ac32-23d9b6ebe268" />

ftp

Secondary filter used:

ftp.command == "USER" or ftp.command == "PASS"

Technical Findings:
<img width="841" height="487" alt="Screenshot 2026-06-28 135426" src="https://github.com/user-attachments/assets/23a8623a-2875-4ea2-89a1-afd6a1e93ada" />

Because legacy FTP transmits all data in plaintext with zero encryption, filtering for FTP packets immediately revealed the raw authentication exchange. Using Follow → TCP Stream, the complete session was reconstructed — exposing the exact username and password submitted during login. No decryption was needed; the credentials were fully readable by anyone on the same network segment.


📊 Incident Artifact Matrix

Lifecycle StageObserved Threat / ActionWireshark Display FilterKey Artifact CapturedReconnaissanceTCP SYN Port Sweeptcp.flags.syn == 1 and tcp.flags.ack == 0Rapid sequential connections to ports 1–1000Credential TheftPlaintext FTP Authenticationftp.command == "USER" or ftp.command == "PASS"Compromised credentials exposed in cleartext


🔒 Defensive Remediation & Hardening Recommendations

1. Enforce Transport Layer Encryption

Replace all legacy FTP usage with SFTP (SSH File Transfer Protocol). SFTP wraps the entire session — credentials and data payloads — inside an encrypted SSH tunnel, rendering passive sniffing attacks ineffective.

2. Implement Network Rate-Limiting

Configure firewall or managed switch policies to dynamically block or throttle source IPs that attempt connections to more than a defined threshold of distinct ports within a short time window. This disrupts automated scanning before full enumeration can complete.

3. Deploy a Network Intrusion Detection System (NIDS)

Establish detection signatures in Snort or Suricata to alert on:


Horizontal port sweep patterns from a single internal source IP
FTP authentication events on non-sanctioned hosts
Any cleartext protocol traffic carrying credential strings



🧰 Tools & Technologies Used

ToolPurposeKali Linux 2025.3Offensive operations platformNmapSYN stealth port scanning and service enumerationFTP ClientCleartext credential transmission simulationMetasploitable2Intentionally vulnerable target nodeWiresharkPacket capture, traffic analysis, IoC identificationVMware WorkstationIsolated lab environment (Host-Only Network)


📁 Repository Structure

purple-team-lab/
│
├── README.md               # This file — full lab documentation
├── captures/
│   └── lab_capture.pcap    # Wireshark packet capture file
└── screenshots/
    ├── nmap_scan.png        # Nmap scan output
    ├── syn_flood_filter.png # Wireshark SYN filter view
    └── ftp_stream.png       # Reconstructed FTP TCP stream


👤 Author

Ashif N
Cybersecurity Analyst | SOC |
