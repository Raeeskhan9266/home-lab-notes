# Home Lab - Week 1

## Setup
- Installed VirtualBox and Kali Linux
- Installed Metasploitable2 as a vulnerable target
- Configured both VMs on the same Internal Network in VirtualBox

## Step 1: Found target IP
Ran `ifconfig` on Metasploitable2 to find its IP address.
<img width="720" height="400" alt="VirtualBox_Metasploitable 2_04_07_2026_01_06_10" src="https://github.com/user-attachments/assets/9b70d4f1-d4a5-4e9a-8127-d763c33fd323" />

Target IP: 192.168.100.150

## Step 2: Confirmed connectivity
Ran from Kali:
ping -c 4 192.168.100.150
<img width="1920" height="478" alt="VirtualBox_kali-linux-2026 2-virtualbox-amd64_04_07_2026_01_07_29" src="https://github.com/user-attachments/assets/f1bf1396-ced4-4c20-9679-6a11b08b07fb" />

Result: Got replies, confirming both machines can communicate.

## Step 3: Ran initial scan
Command used:
nmap -sV 192.168.100.150
<img width="1920" height="936" alt="nmap" src="https://github.com/user-attachments/assets/3f8e182d-9012-4c51-afa1-767a23812f2e" />

## Step 4: Findings
Nmap identified 23 open TCP ports on the target, including several with known, well-documented vulnerabilities:

| Port | Service | Version |
|------|---------|---------|
| 21   | ftp     | vsftpd 2.3.4 |
| 22   | ssh     | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 23   | telnet  | Linux telnetd |
| 25   | smtp    | Postfix smtpd |
| 53   | domain  | ISC BIND 9.4.2 |
| 80   | http    | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 111  | rpcbind | RPC #100000 |
| 139/445 | netbios-ssn | Samba smbd 3.X - 4.X |
| 512  | exec    | netkit-rsh rexecd |
| 1099 | java-rmi | GNU Classpath grmiregistry |
| 1524 | bindshell | Metasploitable root shell |
| 2049 | nfs     | RPC #100003 |
| 2121 | ftp     | ProFTPD 1.3.1 |
| 3306 | mysql   | MySQL 5.0.51a-3ubuntu5 |
| 5432 | postgresql | PostgreSQL DB 8.3.0 - 8.3.7 |
| 5900 | vnc     | VNC (protocol 3.3) |
| 6667 | irc     | UnrealIRCd |
| 8009 | ajp13   | Apache Jserv (Protocol v1.3) |
| 8180 | http    | Apache Tomcat/Coyote JSP engine 1.1 |

## Step 5: What I learned
The scan revealed a large number of outdated and often intentionally vulnerable services. A few stand out for further research:
- **vsftpd 2.3.4** (port 21) — a version with a known public backdoor vulnerability
- **UnrealIRCd** (port 6667) — a known backdoored version exists
- **Port 1524** labeled directly as "Metasploitable root shell" — suggests a pre-configured open shell for practice

This confirms why reconnaissance is the first phase of any penetration test: before touching anything, you identify what's running and cross-reference versions against known weaknesses.
