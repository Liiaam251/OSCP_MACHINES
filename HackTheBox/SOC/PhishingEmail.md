# Phishing Email

## What is the return path of the email?
<img width="514" height="33" alt="image" src="https://github.com/user-attachments/assets/5b032e86-3aa5-4271-a2a2-6d03a3f27b62" />

## What is the domain name of the url in this mail?
<img width="325" height="62" alt="image" src="https://github.com/user-attachments/assets/0c5598e1-cd2d-4360-bb36-cb200d90f0ee" />

## What is the sender IP address listed in the Received-SPF header?
<img width="937" height="18" alt="image" src="https://github.com/user-attachments/assets/da9137af-98e5-44de-8f3c-a88f88228fb4" />

1. El remitente real. El From: que ves es texto libre y se falsifica sin esfuerzo. El Return-Path viene del sobre SMTP y es mucho más difícil de manipular. Si uno dice PayPal y el otro dice un dominio random, ahí tienes la primera señal.

2. El destino real del enlace. El texto del enlace puede poner lo que quiera. Lo que importa es el href. Si el botón dice “continuar con la entrega” pero apunta a un dominio que no es paypal.com, señal número dos.

3. La reputación del dominio. Pasa el dominio por VirusTotal y URLScan.io, y mira el WHOIS. Lo que buscas: dominios registrados hace pocos días o semanas, registradores baratos, datos de registro ocultos. Las marcas legítimas tienen dominios con años de antigüedad.

4. La autenticación del correo. Aquí entran tres cabeceras que vas a ver todos los días en el SOC:

SPF comprueba si la IP que envió el correo está autorizada a enviar en nombre de ese dominio. El Received-SPF que te piden en la tarea 4 lleva el resultado y la IP de origen.
DKIM es una firma criptográfica que acredita que el contenido no se ha alterado.
DMARC define qué hacer cuando SPF o DKIM fallan.
