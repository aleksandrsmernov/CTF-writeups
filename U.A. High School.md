# TryHackMe — U.A. High School

**Room:** U.A. High School  
**Difficulty:** Easy  
**Category:** Linux, Web Exploitation, Privilege Escalation

## 1. Reconnaissance

Start with a full Nmap scan to identify open ports, running services and their versions.

```bash
nmap -sC -sV -p- 10.112.191.150
```

The goal is to identify potential entry points and determine which services are available.

## 2. Web Enumeration

After discovering the HTTP service, investigate the website and search for hidden directories using Gobuster.

```bash
gobuster dir -u http://10.112.191.150 -w /usr/share/wordlists/dirb/common.txt
```

During enumeration, investigate interesting endpoints and files.

## 3. Command Injection

The web application contains a vulnerable parameter named `cmd` in:

```text
/assets/index.php
```

The vulnerability allows system commands to be executed through the web application.

A reverse shell is used to obtain remote access.

Start a listener on the attacking machine:

```bash
nc -lvnp 4444
```

After establishing the connection, we obtain a shell as:

```text
www-data
```

## 4. Steganography — Finding Credentials

Continue enumerating the filesystem and search for interesting files.

An image named `oneforall.jpg` is discovered.

Download it:

```bash
wget http://10.112.191.150/assets/images/oneforall.jpg
```

Inspect the file:

```bash
file oneforall.jpg
```

The image initially has an incorrect file signature. After repairing its header, it is recognized as a JPEG image.

Check for embedded data:

```bash
steghide info oneforall.jpg
```

Extract the hidden file:

```bash
steghide extract -sf oneforall.jpg
```

Use the passphrase:

```text
AllmightForever!!!
```

The extraction reveals `creds.txt`.

Inside the file, we find the following credentials:

```text
deku:One?For?All_!!one1/A
```

## 5. SSH Access

Use the discovered credentials to connect to the machine through SSH.

```bash
ssh deku@10.112.191.150
```

After logging in, enumerate the current user and available privileges.

```bash
id
sudo -l
```

The output shows:

```text
User deku may run the following commands:
(ALL) /opt/NewComponent/feedback.sh
```

This means that `deku` is allowed to execute the specified script with elevated privileges, including root.

## 6. Analyzing feedback.sh

Inspect the script:

```bash
cat /opt/NewComponent/feedback.sh
```

The important part is:

```bash
read feedback

if [[ "$feedback" != *"\`"* &&
      "$feedback" != *")"* &&
      "$feedback" != *"\$("* &&
      "$feedback" != *"|"* &&
      "$feedback" != *"&"* &&
      "$feedback" != *";"* &&
      "$feedback" != *"?"* &&
      "$feedback" != *"!"* &&
      "$feedback" != *"\\"* ]]; then

    echo "It is This:"
    eval "echo $feedback"

    echo "$feedback" >> /var/log/feedback.txt
fi
```

### Why is this vulnerable?

The main issue is:

```bash
eval "echo $feedback"
```

The `eval` command evaluates a constructed string as a new shell command.

For example, the input:

```text
hello world
```

becomes:

```bash
echo hello world
```

However, specially crafted input may cause shell operators to be interpreted during the second parsing stage.

The script attempts to block several potentially dangerous characters, but this blacklist does not provide reliable protection against command injection.

### Why can this lead to privilege escalation?

The script is executed using:

```bash
sudo /opt/NewComponent/feedback.sh
```

Therefore, the vulnerable `eval` operates with the privileges of the script process.

The combination of unrestricted sudo execution and unsafe input handling creates a privilege escalation vulnerability.

## 7. Flags

**User flag:**

```text
THM{W3lC0m3_D3kU_1A_0n3f0rAll??}
```

**Root flag:**

```text
THM{Y0U_4r3_7h3_NUm83r_1_H3r0}
```

## 8. What I Learned

- Nmap — port scanning and service enumeration.
- Gobuster — directory and file discovery.
- Command Injection — executing system commands through vulnerable web applications.
- Reverse Shell — obtaining remote command-line access.
- Steganography — discovering hidden information inside image files.
- Steghide — extracting embedded files.
- SSH — remote access using discovered credentials.
- Linux permissions and sudo — understanding privilege boundaries.
- Bash eval — understanding unsafe command evaluation.
- Privilege Escalation — identifying and exploiting misconfigured permissions.

## Conclusion

This room demonstrates how multiple security weaknesses can be combined to compromise a Linux system.

The most important lesson is that user input must never be passed directly to `eval`. A weak character blacklist does not make this practice safe.

Understanding how shell commands are parsed, how sudo permissions work and how applications handle user input is essential when investigating Linux privilege escalation vulnerabilities.
