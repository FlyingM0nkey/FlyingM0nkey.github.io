---
title: Blackfield [HTB] 
date: 2020-11-07 18:08:00 +/- 0000
categories: [Writeup, Hack_The_Box]
tags: [Walkthrough, HTB, AD, Active Directory, Kerberos, Crackmapexec, Evil-Winrm, BloodHound, JTR, Hashcat, rpcclient, GetNPUsers, Kerbrute, pypykatz, psexec, wmiexec, SMB, LSASS, impacket, Windows]
headline: HTB Blackfield machine.
image: /assets/img/blk/blk1.png
---

Blackfield was a really fun Active Directory machine with many steps required to be able to read the root flag. The writeup and the video differ slightly as I learned a few more things after I had initially rooted the machine. Encrypting the root flag so that NT Authority\System couldn't read it was a dick move... ;)

![pic](/assets/img/blk/blk2.png)

This machine is supposed to be maxed out on enumeration, so let's enumerate!

## Enumeration

```shell
nmap $ip -p-
nmap -sC -sV -p 53,88,389,445,593,3268,5985 $ip
```

![pic](/assets/img/blk/blk3.png)

One of the scans from Autorecon shows me smb info.

![pic](/assets/img/blk/blk4.png)

The only share of interest that I can access at the moment is the profiles$ share.

![pic](/assets/img/blk/blk5.png)

Each of these directories is empty, but the directory list itself will make a nice user list to continue enumerating. There are many ways to do this, but the first idea that popped into my head was to mount the share and copy/paste/cut until I had a usable list.

I mounted the share with the ```mount -t cifs //$ip/profiles$ /mnt``` command. Once mounted, I copied the output to a users.txt file and used the ```cat users.txt | cut -d " " -f 10 > user_list``` command to output my list to a file called user_list.

![pic](/assets/img/blk/blk6.png)

This is a fairly long list, and being a HTB machine, there will likely only be a few actual users. I use a tool called kerbrute to check for valid users.

![pic](/assets/img/blk/blk7.png)

The tool returns 3 valid users, so I just make a file with the 3 names in it. To check for kerberos tickets, I use a tool called GetNPUsers.py. A simple *for loop* one-liner automates the process. We find a ticket for user ***support***.

```shell
for user in $(cat names.txt); do GetNPUsers -no-pass -dc-ip $ip blackfield.local/$user | grep krb5asrep; done
```

![pic](/assets/img/blk/blk8.png)

The ticket is copied to a file and *John The Ripper* is used to crack it.

```shell
john hash.txt --wordlist=rockyou.txt
```

![pic](/assets/img/blk/blk9.png)

*Hashcat* can also be used to crack this hash, but, since it's Hashcat, we need to find the mode. You can look for it manually on the [example page](https://hashcat.net/wiki/doku.php?id=example_hashes):

![pic](/assets/img/blk/blk10.png)

or use a one-liner in your terminal.

```shell
hashcat --example-hashes | grep -B5 'krb5asrep'
```

![pic](/assets/img/blk/blk11.png)

I'm sure you can probably use hashid.py as well. Hachcat cracks the ticket in 2 seconds.

![pic](/assets/img/blk/blk12.png)

With a valid set of credentials, I can use *crackmapexec* to try and log in.

```shell
crackmapexec smb $ip -u support -p '#00^BlackKnight'
```

![pic](/assets/img/blk/blk13.png)

I cannot log in with this user. I can, however, read more smb shares. Unfortunately, I find no more useful information. I really want to read the *forensic* share, partly because it is also called *audit* and we know there is an audit2020 user. Since I am at an impasse, I will employ *BloodHound*. 

The BloodHound ingestors can now be run from a Linux machine with *bloodhound-python*, which is cool and saves time/hassle. I don't have screenshots for this, but it's in the video.

```shell
bloodhound-python -c ALL -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns $ip
```

Start the neo4j console: ```neo4j console``` and the BloodHound program: ```bloodhound --no-sandbox```, then drop the 4 json files from bloodhound-python into the app. Pulling up our support user and I can see that this user can change audit2020's password.

![pic](/assets/img/blk/blk15.png)
![pic](/assets/img/blk/blk16.png)

There's an excellent, and short, blog post written by Mubix explaining how to change the password using *rpcclient*.

<https://room362.com/post/2017/reset-ad-user-password-with-linux/>

```shell
setuserinfo2 audit2020 23 'Unic0rn'
```

The audit2020 password changed to Unic0rn.

![pic](/assets/img/blk/blk17.png)

Using crackmapexec again, we can see that the password was changed succesfully, but I still can't log in. This time I can read the forensic share, though.

![pic](/assets/img/blk/blk18.png)

Inside the forensic share is a directory called *memory_analysis* which contains an *lsass.zip* file. LSASS stans for ***Local Security Authority Subsystem Service*** so it stands to reason that there will be some juicy intel inside. 

![pic](/assets/img/blk/blk19.png)
![pic](/assets/img/blk/blk20.png)

Transfer the file back to Kali with *mget* and unzip to reveal a dump (DMP) file.

![pic](/assets/img/blk/blk21.png)
![pic](/assets/img/blk/blk22.png)

Using a mimikatz-like program called *pypykatz* gets us a hash for the svc_backup user.

```shell
pypykatz lsa minidump lsass.DMP
```

![pic](/assets/img/blk/blk23.png)

Crackmapexec tells me that the hash is valid, but still doesn't show me the orang **Pwned!** I was expecting to see.

![pic](/assets/img/blk/blk24.png)

I'm all out of users, so I'm hoping that this is a bug and I will be able to log in with *Evil-Winrm*.

```shell
evil-winrm -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d -i $ip
```

![pic](/assets/img/blk/blk25.png)

Huzzah!! After 3 users, I finally have a shell. Checking the privs show that my user has *SeBackupPrivilege*. Using [these slides](https://hackinparis.com/data/slides/2019/talks/HIP2019-Andrea_Pierini-Whoami_Priv_Show_Me_Your_Privileges_And_I_Will_Lead_You_To_System.pdf) shows how to abuse this privilege.

![pic](/assets/img/blk/blk27.png)

Other links that proved helpful:

* <https://pentestlab.blog/2018/07/04/dumping-domain-password-hashes/>
* <https://pentestlab.blog/tag/diskshadow/>
* <https://bohops.com/2018/03/26/diskshadow-the-return-of-vss-evasion-persistence-and-active-directory-database-extraction/>

Shadow script made, converted to dos and uploaded to the target.

```shell
diskshadow /s C:\programdata\shadow3.dsh
```

![pic](/assets/img/blk/blk28.png)
![pic](/assets/img/blk/blk29.png)

After far too much trial and error with the whole process, I've finally managed to create the shadow drive.

![pic](/assets/img/blk/blk30.png)

Two dll's from [this repository](https://github.com/giuliano108/SeBackupPrivilege) need to be uploaded to the target and imported into the PS session.

![pic](/assets/img/blk/blk31.png)

I need the ntds.dit file back on my Kali machine. It takes a good few minutes, but I get it in the end.

![pic](/assets/img/blk/blk32.png)

I will also need the SYSTEM file to extract credentials. Again, this takes a few minutes to download.

```shell
REG SAVE HKLM\system system
```

![pic](/assets/blk/blk33.png)

With both files on my machine, I can now run *secretsdump.py*.

```shell
python3 secretsdump.py -system system -ntds ntds.dit LOCAL
```

![pic](/assets/img/blk/blk34.png)

Awesome! We have the admin hash. I can use *psexec.py* to get a system shell.

```shell
python3 /usr/share/doc/python3-impacket/examples/psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee administrator@$ip
```

![pic](/assets/blk/blk35.png)

Just when I thought I was done with this machine, I can't read the root flag for some reason! WTF?? There is also a note about the file being encrypted.

![pic](/assets/img/blk/blk36.png)
![pic](/assets/img/blk/blk44.png)

I check the permissions with *icacls* and see that the Administrator has a slightly different permission to me. Psexec will always give a system shell, so I log out and log back in with Evil-Winrm (I use wmiexec on the video) and I can finally read the root flag.

![pic](/assets/img/blk/blk37.png)

Link to video: <https://youtu.be/HZJGQm9iYUY>

Thanks to the creators for a fun machine.

M0nkey out.
