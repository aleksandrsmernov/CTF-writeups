TryHackMe — Dig Dug
Difficulty: Easy
Category: DNS / Recon
Goal: Extract hidden information from a DNS server that only exposes SSH externally.

Introduction
This room is built around the idea that DNS is an attack surface too.
A standard nmap -sC -sV scan can come back empty if the service only listens on 53/UDP and doesn't return banners.
The trick is to remember dig and use it correctly.

1. Recon
Start with a basic scan:

nmap -sC -sV <TARGET_IP>
Result: only 22/tcp (SSH) is open. No web services, nothing interesting in the scripts.

Expand to all ports:

nmap -p- <TARGET_IP>
No additional ports appear. A classic dead end for "web-first" thinking.

Takeaway: since the room description explicitly mentions a domain and the box is essentially a DNS server,
          we need to look at port 53 (UDP) — which -p- over TCP will never show.

2. DNS Inspection with dig
Check whether DNS responds:

dig @<TARGET_IP> <DOMAIN>
or, using the domain from the briefing:

dig @<TARGET_IP> <DOMAIN> ANY
The ANSWER SECTION of the response contains a record holding the flag. Copy it and you're done.

Why this works: dig queries the specified DNS server directly (@<IP>) instead of the system resolver. That way we see exactly what the target returns — including records that never show up in a browser.

Useful variants if the first attempt comes back empty:

dig @<TARGET_IP> <DOMAIN> TXT
dig @<TARGET_IP> <DOMAIN> MX
dig @<TARGET_IP> <DOMAIN> AXFR      # attempt a zone transfer

3. Flag
The flag was found in the ANSWER SECTION of the DNS response.
THM{...}


4. Lessons Learned
nmap over TCP is not the whole picture. DNS lives on 53/UDP, and -p- (TCP) will not see it. Always keep -sU in mind for UDP services, especially DNS/SNMP.

The room description is a hint. If it names a domain and the box is a DNS server, dig is the first tool to reach for.

dig @server > system resolver. Querying the target directly shows its real answers, not cached or filtered ones.

DNS records (TXT, ANY) are a common dumping ground for forgotten secrets and flags in CTFs.

