# Enumeración

<img width="867" height="672" alt="image" src="https://github.com/user-attachments/assets/8b0ba605-7450-43fd-91dd-12434df7597c" />

````
gobuster vhost -u http://editor.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain

````
<img width="915" height="337" alt="image" src="https://github.com/user-attachments/assets/dc0e899c-f5de-4234-b1d5-028775bbd1c1" />
````
https://github.com/dollarboysushil/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC/blob/main/CVE-2025-24893-dbs.py
````
<img width="965" height="653" alt="image" src="https://github.com/user-attachments/assets/eb65552f-ed46-47e6-87a2-23460535b23c" />
<img width="790" height="304" alt="image" src="https://github.com/user-attachments/assets/1c49ea15-98b4-4eaa-9392-54123c66c857" />
Mejoramos shell:
````
python3 -c 'import pty; pty.spawn("/bin/bash")'
````
Dentro del etc xwiki tenemos un password:
<img width="704" height="169" alt="image" src="https://github.com/user-attachments/assets/8cb5ec13-499e-4c4a-b31b-83cc46a8a981" />
Miramos users con sh
<img width="388" height="26" alt="image" src="https://github.com/user-attachments/assets/f4443ed2-8ae1-40e9-a422-128946ddd360" />
````
theEd1t0rTeam99
````
Pillamos oliver y ya

ESCALADA:

<img width="938" height="305" alt="image" src="https://github.com/user-attachments/assets/17ff7f15-a679-49a2-8299-47cb0809ff69" />

Vulnerable:
https://github.com/dollarboysushil/CVE-2024-32019-Netdata-ndsudo-PATH-Vulnerability-Privilege-Escalation/blob/main/CVE-2024-32019-dbs.py
<img width="888" height="694" alt="image" src="https://github.com/user-attachments/assets/c570afeb-7ff7-460a-9455-3ca1d2e206a6" />
