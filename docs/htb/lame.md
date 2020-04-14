---
layout: default
title: Home
permalink: /htb/lame
---

# Lame

> **Name:**      Lame
> **Creator:**   [ch4p](https://www.hackthebox.eu/home/users/profile/1)
> **Difficulty** Easy

## Intro

[Lame](https://www.hackthebox.eu/home/machines/profile/1) was released on 14 Mar 2017 and among the first boxes released on the platform. It is one of the easier linux boxes available, which makes it a good place to start for beginners. I originally completed this when it had already retired and I was making my way through some of the older boxes to help further my understanding. As it was so simple I never made notes so I've had to fully go back and re-do the box to make this write-up.

## Getting Started

As is always the case we start off with performing a full port scan against the box to  get an idea of what services are available. My preferred command is shown below:

``` nmap -p- 10.10.10.3 -sC -sV -oA full-scan ```

Which gives the following results:

![Scan Results](./assets/lame/scan-results.png)

From here it is best to go through each service in turn, and check for any known vulnerabilities for the versions in use and for any interesting results from the nmap scripts that were run. Straight away searchsploit delivers the goods.

![Searchsploit Results](./assets/lame/vsftpd-searchsploit.png)

The nmap scripts also show that anonymous access is enabled, so a quick connect and check for any files is performed.

![Anonymous FTP](./assets/lame/ftp-anonymous.png)

No luck this time! It is all empty. Rather than diving straight into using the obvious vsftpd exploit available in metasploit it is better (in my opinion) to continue checking those other services from the initial results. It can help to have a backup plan rather than getting stuck chasing an exploit that may not be the only way forward. This is especially important on this box, as it appears that exploiting vsftpd is a dead end. I could not get the exploit to work. Most likely explanation is firewall rules. 

The only other significant finding was a vulnerability for the version of Samba in use, as shown when checking searchsploit.

![Searchsploit Results](./assets/lame/samba-searchsploit.png)

Checking the Rapid7 site for this [exploit](https://www.rapid7.com/db/modules/exploit/multi/samba/usermap_script) gives the following description:

> This module exploits a command execution vulnerability in Samba versions 3.0.20 through 3.0.25rc3 when using the non-default "username map script" configuration option. By specifying a username containing shell meta characters, attackers can execute arbitrary commands. No authentication is needed to exploit this vulnerability since this option is used to map usernames prior to authentication!

Checking the ruby script shows that it is as easy as sending a crafted username with a payload after it. 

![Samba Exploit](./assets/lame/msf-script.png)

First of all there needs to be a share we can connect to, so smbclient is used to get a list of available shares. 

![Shares](./assets/lame/smb-shares.png)

You may have noticed an additional option in that command. On newer versions of SMBClient it is configured to prevent connection to older versions of SMB, so this setting just needs to be set so we can connect properly. 

It is possible to null authenticate to the /tmp share and use the logon command to submit the crafted username to get a shell

![Exploit](./assets/lame/smb-exploit.png)

Yay! Straight to root. 
