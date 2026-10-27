# Enumeración 

<img width="633" height="406" alt="image" src="https://github.com/user-attachments/assets/4395e8a7-458c-4709-904c-ea240025a464" />

<img width="1919" height="886" alt="image" src="https://github.com/user-attachments/assets/7d6258c5-6be4-44c9-9b0f-b9998e50d9ce" />

Enumeramos subdominios:
````
curl -s -o /dev/null -w "%{size_download}\n" -H "Host: noexiste.devvortex.htb" http://devvortex.htb
# 154
````
````
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt:FUZZ \
     -u http://devvortex.htb \
     -H 'Host: FUZZ.devvortex.htb' \
     -fs 154 \
     -t 40 \
     -rate 100

````
dev

Vemos que tiene joomla

<img width="816" height="142" alt="image" src="https://github.com/user-attachments/assets/aa5e4db1-191d-4fcd-8b81-57d54be833f7" />
<img width="641" height="381" alt="image" src="https://github.com/user-attachments/assets/70dd1d8d-1841-49fa-9f00-85e2db5b2f03" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/86033a42-29aa-4aa8-9207-ce979c1248f6" />
Tenemos acceso al joomla, ahora la idea es hacer reverseshell

