# Enumeración:

<img width="840" height="388" alt="image" src="https://github.com/user-attachments/assets/47ce241c-f8c6-417f-afcc-6ee841fe8ade" />

# FASE WEB:

<img width="476" height="343" alt="image" src="https://github.com/user-attachments/assets/9174f4b2-868c-4b9e-a8ab-ff739c454078" />

Poniendo localhost sale el siteisup.htb, lo metemos en localhost

<img width="946" height="462" alt="image" src="https://github.com/user-attachments/assets/ad2c636f-3ac7-4f96-b942-20d51a95f73b" />
````
ffuf -u http://siteisup.htb -H "Host: FUZZ.siteisup.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fs 1131
````
<img width="734" height="365" alt="image" src="https://github.com/user-attachments/assets/02112103-50c4-4988-b70f-ce0c692c6101" />
````
git-dumper http://siteisup.htb/dev/.git dev
````
<img width="591" height="378" alt="image" src="https://github.com/user-attachments/assets/3b2aa4a4-c9b7-4ba8-b120-5b7ff3d87e5d" />

Hace falta ese header

