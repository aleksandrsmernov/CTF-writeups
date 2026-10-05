TryHackMe — RootMe

Target: 10.113.151.10

Reconnaissance

I started with an Nmap scan:

sudo nmap -sC -sV 10.113.151.10

I found two open ports: 22 SSH and 80 HTTP. I checked the website first.

Web

The website was a simple page called HackIT. I then used Gobuster to find hidden directories:

gobuster dir -u http://10.113.151.10 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html

This revealed /panel/ and /uploads/.

At first I spent quite some time looking at the file upload functionality. I found that HTML and JavaScript files were accepted and tried some XSS payloads. I also used Burp Suite to inspect and modify the requests, but this did not lead anywhere useful.

The important part was the upload functionality itself.

The panel blocked .php files, so I started testing other PHP extensions. I found that .phtml files were accepted and executed as PHP.

I uploaded a PHP reverse shell and started a listener on my Kali machine:

nc -lvnp 4444

The connection worked and I got a shell as www-data.

Privilege Escalation

I checked for SUID files:

find / -type f -perm -4000 2>/dev/null

One interesting result was:

/usr/bin/python2.7

I checked its permissions and found that it had the SUID bit:

-rwsr-xr-x 1 root root ... /usr/bin/python2.7

I first tried to spawn a normal shell, but it stayed as www-data because the shell dropped the elevated privileges.

I then used:

/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh","sh","-p")'

This preserved the privileges and gave me a root shell.

I checked with whoami and got root. I then found and read the user and root flags.

Attack Path

Nmap → HTTP → Gobuster → /panel/ → XSS → Burp Suite → dead end → file upload → PHP extension bypass → .phtml → reverse shell → www-data → SUID Python → root → flags

Main Lesson

The main lesson for me was not to get stuck on the first interesting thing I find. XSS and Burp Suite looked promising at first, but the real path was much simpler: the file upload functionality, a different PHP extension, and then the SUID Python binary for privilege escalation.
