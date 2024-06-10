# INICIO BY d4l1

Muy buenas, hoy estaremos tratando la máquina Pandora de HTB.

Como siempre haremos un poco de resumen a lo que nos enfrentaremos, conocimientos y demás.

Empezaremos haciendo un escaner de sistema operativo, (tengo un repositorio con dicho escaneo).

Gracias a ello entendemos que nos estamos enfrentando a una máquina Linux, posteriormente toca un escaner de puertos, para ver si llegaramos a poder sacar alguna información.

```
sudo nmap -sCV -T4 --min-rate 5000 IP
```
Si queremos podemos poner un "-p-" o "-Pn" pero en este caso no se si hara falta.

Nos encontramos con los puertos "22" (SSH) y "80" (HTTP) abiertos.

Lo que haremos primero sera inspeccionar la página a ver que encontramos.

## PROCESO

Lo primero que encontramos nada más entrar es un dominio "panda.htb" lo añadimos
```
echo "ip panda.htb" | sudo tee -a /etc/hosts
```
En este momento, podemos hacer un escaner de directorios o dominios, pero no vamos a encontrar nada importante.

Probaremos a hacer un nuevo escaner de nmap pero en este caso enfocandonos en udp
```
sudo nmap -sU ip
```