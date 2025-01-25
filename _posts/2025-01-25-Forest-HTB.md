---
Title: Forest HTB
Rating: Easy
OS: Windows
---
# Forest HTB
*This writeup is for the Forst Hack the Box Machine*
![Logo](Images/Forest_HTB/forest_logo.png)

We'll start by enumerating this box using a quick nmap scan running the default scripts:
```bash
└─$ nmap -sC -sV 10.10.10.161 -v -oN nmap_short.txt
Starting Nmap 7.95 ( https://nmap.org ) at 2025-01-25 15:00 AWST
Scanning 10.10.10.161 [1000 ports]
---snip---
Discovered open port 53/tcp on 10.10.10.161
Discovered open port 135/tcp on 10.10.10.161
Discovered open port 139/tcp on 10.10.10.161
Discovered open port 445/tcp on 10.10.10.161
Discovered open port 593/tcp on 10.10.10.161
Discovered open port 636/tcp on 10.10.10.161
Discovered open port 389/tcp on 10.10.10.161
Discovered open port 3269/tcp on 10.10.10.161
Discovered open port 3268/tcp on 10.10.10.161
Discovered open port 88/tcp on 10.10.10.161
Discovered open port 5985/tcp on 10.10.10.161
Discovered open port 464/tcp on 10.10.10.161
---snip---
```

The boave shows varios open ports that are associated with Windows Domain services, indicating that this machine is possibly a domain controller
