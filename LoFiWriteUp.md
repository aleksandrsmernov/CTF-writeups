Lo-Fi — LFI / Path Traversal

First I ran a basic nmap scan:

sudo nmap -sC -sV 10.114.156.223

I found 2 open ports:

22/tcp ssh
80/tcp http

SSH is running on port 22 and Apache 2.2.22 on port 80.

So I checked the website.

Web

The website is called Lo-Fi Music and has a few playlists.

When I clicked on one of them I noticed this in the URL:

http://10.114.156.223/?page=coffee.php

The page parameter looked interesting because it was taking a filename.

So I thought maybe the backend is just taking whatever is in page and loading that file.

I tested it with:

http://10.114.156.223/?page=/etc/passwd

And it worked. The server returned /etc/passwd.

So this was an LFI.

Path Traversal

Since I could read local files, I tried using ../ to move up through the directories.

I tried:

http://10.114.156.223/?page=../../../../../../flag.txt

This worked and the server returned the contents of flag.txt.

Got the flag.

Attack path

Nmap
Port 80
?page=coffee.php
LFI
/etc/passwd
../ Path Traversal
flag.txt
Flag
