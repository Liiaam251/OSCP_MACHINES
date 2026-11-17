# Enumeración




## HTTP:

<img width="1919" height="303" alt="image" src="https://github.com/user-attachments/assets/a58dda02-09b0-48b5-a313-8f0ea1a343c4" />

Ahora tenemos esto:
<img width="1345" height="154" alt="image" src="https://github.com/user-attachments/assets/c6fcfa4b-d614-421d-b0a9-f570c663bcea" />
Lo buscamos con path traversal
<img width="1739" height="323" alt="image" src="https://github.com/user-attachments/assets/41def1ff-a59d-4034-ba7a-99f54bae619c" />
````
Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO Ukd4RVdub3dPVU5uUFQwSwo= 
````
<img width="342" height="575" alt="image" src="https://github.com/user-attachments/assets/8854d576-c7ea-48cd-915f-a1418140e348" />
Tenemos password
<img width="342" height="575" alt="image" src="https://github.com/user-attachments/assets/dadba111-93a3-4b5c-afc8-95004748dc63" />
````
Charix!2#4%6&8(0
````

````
eaacdfb2d141b72a589233063604209c
````
<img width="737" height="697" alt="image" src="https://github.com/user-attachments/assets/df0bdddc-04a7-4f1a-8399-2d69aa4d5a43" />
# Escalada de priv
<img width="943" height="378" alt="image" src="https://github.com/user-attachments/assets/ec2c0035-693f-48ce-ba78-281fd12a01db" />
````
echo "=== PROCESOS ROOT RAROS ==="; ps aux | awk '$1=="root"' | grep -vE "\[|/sbin/|/usr/sbin/|/usr/libexec/|httpd|sshd|csh|vmtoolsd"; echo; echo "=== PUERTOS ESCUCHANDO ==="; sockstat -4 -l 2>/dev/null || ss -tulnp
````
HAcemos puente ssh
````
ssh -L 5901:127.0.0.1:5901 charix@10.129.1.254
````
En kali nos conectamos

<img width="731" height="335" alt="image" src="https://github.com/user-attachments/assets/c0914052-0c48-46c1-8895-0436a174091e" />

<img width="1211" height="930" alt="image" src="https://github.com/user-attachments/assets/81d09767-719e-49f1-ba71-380add5bf1ce" />

````

````
