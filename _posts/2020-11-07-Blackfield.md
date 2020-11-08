---
title: Blackfield [HTB] 
date: 2020-11-07 18:08:00 +/- 0000
categories: [Writeup, Hack_The_Box]
tags: [Walkthrough, HTB, AD, Active Directory, Kerberos, Crackmapexec, Evil-Winrm, BloodHound, JTR, Hashcat, rpcclient, GetNPUsers, Kerbrute, pypykatz, psexec, wmiexec, SMB, LSASS, impacket, Windows]
headline: HTB Blackfield machine.
image: /assets/img/blk/blk1.png
---

Blackfield was a really fun Active Directory machine with many steps required to be able to read the root flag. The writeup and the video differ slightly as I learned a few more things after I had initially rooted the machine. Encrypting the root flag so that NT Authority\System couldn't read it was a dick move... ;)

## Enumeration

```shell
nmap $ip -p-
