# Enumeración

````
└─$ sudo nmap -sVC -p- -n -T4 --min-rate=5000 -sS 10.129.84.148
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-15 12:26 -0500
Nmap scan report for 10.129.84.148
Host is up (0.041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:be:60:d2:95:f6:28:c1:b7:e9:e8:17:06:f1:68:f3 (RSA)
|   256 0e:b6:a6:a8:c9:9b:41:73:74:6e:70:18:0d:5f:e0:af (ECDSA)
|_  256 d1:4e:29:3c:70:86:69:b4:d7:2c:c8:0b:48:6e:98:04 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://pilgrimage.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.50 seconds

````
<img width="1914" height="915" alt="image" src="https://github.com/user-attachments/assets/44a67f7c-b980-4224-9298-9e2454d1c7b1" />
````
cd ~/Desktop/GitTools/Dumper
./gitdumper.sh http://pilgrimage.htb/.git/ /tmp/dump
###########
# GitDumper is part of https://github.com/internetwache/GitTools
#
# Developed and maintained by @gehaxelt from @internetwache
#
# Use at your own risk. Usage might be illegal in certain circumstances. 
# Only for educational purposes!
###########
````
