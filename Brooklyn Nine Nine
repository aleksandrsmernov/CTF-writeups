TryHackMe — Brooklyn Nine Nine

Target: 10.112.169.49

The target IP was changed during the CTF. At the beginning it was 10.112.143.37.

Reconnaissance

I started with:

sudo nmap -sV -sS 10.112.143.37

The scan showed three open ports: 21 FTP, 22 SSH and 80 HTTP.

Web

I checked the website and inspected the source. There was an interesting clue: "Have you ever heard of steganography?" I also found the image brooklyn99.jpg, so I downloaded it with wget and started checking it.

I tried exiftool and binwalk first, but nothing useful was found. Since the page specifically mentioned steganography, I tried steghide:

steghide info brooklyn99.jpg

It showed that the image could contain embedded data, but a passphrase was required.

I used stegseek with rockyou.txt:

stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt

The passphrase was admin and the hidden file was note.txt. It contained Holt's password:

fluffydog12@ninenine

SSH

Since SSH was open, I tried the credentials:

ssh holt@10.112.169.49

The login worked. In Holt's home directory I found user.txt and got the user flag.

Privilege Escalation

Next I checked sudo permissions:

sudo -l

Holt could run /bin/nano as root without a password:

(ALL) NOPASSWD: /bin/nano

I started Nano with sudo and used its shell functionality to get a root shell. whoami confirmed that I was root.

I then went to /root and read root.txt to get the root flag.

Attack Path

Nmap → FTP / SSH / HTTP → steganography clue → brooklyn99.jpg → steghide → stegseek → admin → Holt's password → SSH → user.txt → sudo -l → NOPASSWD: nano → root → root.txt

Main Lesson

The main lesson for me was to follow the clues from one step to the next instead of focusing on a single service. The website led to the image, the image gave credentials, the credentials gave SSH access, and sudo -l revealed the privilege escalation.
