---
Title: Forest HTB
Rating: Easy
OS: Windows
---
# Forest HTB
![Logo](Images/Forest_HTB/forest_logo.png)

*This writeup is for the Forst Hack the Box Machine*

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
The domain name lookup script from our nmap commnad enumerated the machine to have the net bios name of FOREST, but the domain name of local.htb:
```bash
Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2025-01-24T23:07:32-08:00
```

The FQDN for the domain shows the forest.htb.local domain. We'll add these additions to our host file to ensure we can perform accurate enumeration. 
`10.10.10.161    htb.local FOREST.htb.local`

I'll start by enumerating the LDAP services as that seems to be the best starting point.
Running the below ldapsearch command produces a slew of information (and confirms that unauthenticated searches are enabled):
`└─$ ldapsearch -x -H ldap://htb.local -b "dc=htb,dc=local" >> ldapsearch.txt`

Adding the `cn=Users` flag to this we can get information regarding active users on the box. 
Next We'll use a tool like Rubeus to see if AS-REP Roasting is possible.

AS-REP Roasting is a methodology to scan for user hashes that have Kerberos pre-authenitcation disabled.  
Kerberos pre-auth is a Windows security mechanism that protects against password-guessing attacks. 
It works by responding to an asynchronous server request with a timestamp enabled with the users password hash.
The Keey Distribution Center then validates this hash by confirming the timestamp.

If Pre-authentication is disabled, then the AS-REQ will be directly responded to with the users password hash.
For more information, see the [LDAP WiKi](https://ldapwiki.com/wiki/Wiki.jsp?page=Kerberos%20Pre-Authentication).




