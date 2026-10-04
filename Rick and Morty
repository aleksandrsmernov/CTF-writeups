TryHackMe — Rick and Morty

Target: [target IP]

Reconnaissance

I started with an Nmap scan:

sudo nmap -sC -sV [target IP]

The scan showed SSH and HTTP open, so I started with the web server.

Web

I checked the website and then looked at robots.txt. It contained an interesting string:

Wubbalubbadubdub

This looked like a password, so I searched for the username used by the web application and found:

R1ckRul3s

Using these credentials, I was able to access the web portal.

First Ingredient

After getting access, I checked the files in the current directory. The first ingredient was there immediately in a file called:

first ingredient.txt

I read it and got the first flag.

Second Ingredient

There was also a clue pointing towards the next ingredient. I searched the system and found the second ingredient in Rick's home directory:

/home/rick/second ingredients

I read the file and got the second flag.

Privilege Escalation

Next I checked the sudo permissions:

sudo -l

The important part was that the current user could run commands with sudo without a password.

This allowed me to get a root shell.

Root

After getting root access, I checked the root user's files. There I found:

Sup3rS3cretPickl3Ingred.txt

I read the file and got the final ingredient.

Attack Path

Nmap → HTTP → robots.txt → Wubbalubbadubdub → R1ckRul3s → web portal → first ingredient.txt → clue → second ingredients → sudo -l → root → Sup3rS3cretPickl3Ingred.txt

Main Lesson

The main lesson for me was to keep following the clues instead of immediately trying more complicated techniques. The website gave the credentials, the first file pointed towards the next step, and sudo permissions provided the way to root.
