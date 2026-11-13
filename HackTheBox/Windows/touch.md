# Enumeración

````
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-13 09:39 +0100
Nmap scan report for 10.129.99.69
Host is up (0.044s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
3389/tcp open  ms-wbt-server Microsoft Terminal Service
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8443/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-cors: GET POST PUT OPTIONS
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-server-header: Microsoft-HTTPAPI/2.0
| http-title: Nexion DeviceHub - Login
|_Requested resource was /login
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 185.16 seconds

````

````
└─$ gobuster dir -u http://10.129.99.69:8443/api -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,aspx,asp -b 302,404
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.99.69:8443/api
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   302,404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,txt,bak,aspx,asp
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
scan                 (Status: 405) [Size: 30]
status               (Status: 200) [Size: 116]
Progress: 27678 / 27678 (100.00%)
===============================================================
Finished
==========
````

Tenemos password 
<img width="706" height="322" alt="image" src="https://github.com/user-attachments/assets/c141191f-970e-4e89-8374-84a5e6e76132" />

<img width="954" height="898" alt="image" src="https://github.com/user-attachments/assets/70168258-33ea-4bc3-999b-0eb28032fb92" />


Apagamos scaner y scaneamos
Abrimos error y abrimos cmd

Tenemos password:
<img width="1223" height="218" alt="image" src="https://github.com/user-attachments/assets/be3a8568-7cec-41a6-81ef-fc0b6155a6e6" />
````
Usuario: kiosk_app
Contraseña: K!0sk_R3pl1ca#DB
````
<img width="957" height="100" alt="image" src="https://github.com/user-attachments/assets/53b7fb6e-8209-496e-9cc3-a3f8abf63e47" />

Nuevas credenciales de root dfe la mysql

