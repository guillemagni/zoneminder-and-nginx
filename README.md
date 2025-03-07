# Guía de instalación: ZoneMinder con Nginx  

De forma estándar, ZoneMinder está diseñado para trabajar con [Apache](https://zoneminder.readthedocs.io/en/latest/installationguide/index.html). Sin embargo, en una situación donde era necesario optimizar el consumo de memoria debido a picos de carga, decidimos optar por **Nginx** en lugar de **Apache**.  

Como no existe documentación oficial sobre esta configuración, he decidido crear esta guía como referencia para quienes necesiten implementar ZoneMinder con Nginx.  

Toda la instalación y los comandos se realizarán para ejecutar **ZoneMinder 1.36.33** con **Nginx 1.22.1** en un servidor **Debian 12 Bookworm**.  

---

## **Requisitos**  
* [ZoneMinder](https://zoneminder.com/)  
* [Nginx](https://nginx.org/) como servidor web  
* [MariaDB](https://mariadb.org/) para la base de datos  
* [Let's Encrypt](https://letsencrypt.org/) para los certificados SSL y configuración HTTPS  

---

## **Guía de instalación**  

Antes de empezar, hay que comentar que Debian no viene con **sudo** instalado de forma predeterminada, si lo desea puede instalarlo y configurarlo [Install sudo on Debian](https://www.cyberciti.biz/faq/how-to-install-and-configure-sudo-on-debian-linux/), en mi caso opté por correr todos los comandos desde root. Lo primero siempre es actualizar el sistema:  

```bash
apt update && apt upgrade
```

También es recomendable configurar el direccionamiento IP para gestionar el servidor:

``` bash
nano /etc/network/interfaces
```

A modo de ejemplo, esta puede ser una configuración para una IP estática:

``` bash
iface enp2s0 inet static
        address 192.168.1.5
        netmask 255.255.255.0
```

Para aplicar los cambios en la configuración de red, es necesario reiniciar el servicio:
``` bash
/etc/init.d/networking restart
```

Una vez realizados estos cambios, procedemos con la instalación.

``` bash
apt install -y zoneminder nginx mariadb-server mariadb-client php-fpm php-mysql php-gd php-xml php-mbstring php-curl php-zip ffmpeg curl net-tools certbot python3-certbot-nginx fcgiwrap
```

Los paquetes que se van a instalar: 
* **zoneminder**: software principal.
* **nginx**: servidor web.
* **mariadb-server** y **mariadb-client**: base de datos compatible con MySQL.
* **php-fpm**: para ejecutar PHP con FastCGI en Nginx.
* **php-mysql**: conector PHP para MariaDB/MySQL.
* **php-gd**: biblioteca de gráficos utilizada por zoneminder.
* **php-xml**, **php-mbstring**, **php-curl**, **php-sip**: extensiones PHP requeridas.
* **ffmpeg**: necesario para trabajar con flujos de video.
* **curl**: utilidades generales.
* **net-tools**: comandos de red útilies (netstat, por ejemplo).
* **certbot** y **python3-certbot-nginx**: para certificados SSL de Let's Encrypt.
* **fcgiwrap**: ejecuta CGI en FastCGI para ZoneMinder en Nginx.

Al instalar ZoneMinder, de forma predeterminada Debian utilizará la última version publicada estable. Puede instalar la versión más nueva mediante backports, se puede leer más [aquí](https://zoneminder.readthedocs.io/en/latest/installationguide/debian.html#debian-12-bookworm). 

## **Configuración de la Base de Datos**
Una vez instalado MariaDB, puede hacer los ajustes de seguridad por medio de `mariadb-secure-installation`. Se tiene que crear el usuario y la base de datos para poder correr el script que genere las tablas necesarias para el ZM (ingresar al gestor de base de datos por medio de `mariadb`):

``` bash
CREATE DATABASE zm;
CREATE USER zmuser@localhost IDENTIFIED BY 'zmpass'; 
GRANT ALL ON zm.* TO zmuser@localhost;
FLUSH PRIVILEGES;
exit;
```

Puede modificar **zmpass** por la contraseña que desee, pero deberá modificar el archivo **zm.conf** (/etc/zm/zm.conf) para que ZoneMinder puede acceder a la DB. 

``` bash
# ZoneMinder database password
ZM_DB_PASS=contraseñaDB
```

Realizado estos cambios, se procede a ejectuar el script para darle forma a la base de datos:

``` bash
mariadb -u zmuser -p zm < /usr/share/zoneminder/db/zm_create.sql
```

Por último cambiamos los permisos para que zoneminder pueda leer la configuración 

``` bash
chgrp -c www-data /etc/zm/zm.conf
```

##  **Configuración de Nginx**
Creamos un archivo para dar las configuraciones para que Nginx pueda dar de alta la web de ZoneMinder:

``` bash
nano /etc/nginx/sites-avalibre/zoneminder.conf
```

En **example.conf** podrá ver una ejemplo de configuración para vincular ZoneMinder con Nginx, con los comentarios correspondiente para cada linea. Podría también generar el zoneminder.conf en /etc/nginx/ y desde el archivo **default** incluir la configuración dedicada para ZoneMinder por medio de `include /etc/nginx/zoneminder.conf;`, queda a elección del administrador. Por último, habilitar la configuración en el servidor:

``` bash
ln -s /etc/nginx/sites-available/zoneminder.conf /etc/nginx/sites-enabled/
```


## **Cambios extras**
Por último, unos cambios para la configuración de ZoneMinder:
* Modificar el archivo **01-system-paths.conf** (/etc/zm/conf.d/01-system-paths.conf) y modificar el path **ZM_PATH_ZMS**, dejando el path directo con **/cgi-bin/nph-zms**, eliminar /zm/ al comienzo del path. 
* Modificar archivo php.ini (/etc/php/x.x/fpm/php.ini) y modificar **cgi.fix_pathinfo**, dejando la variable con el valor 0 (cero).

## **Certificados SSL y configuración HTTPS con Let's Encrypt**
Obtener los certificados para el dominio que vaya a utilizar:

``` bash
certbot --nginx -d su.dominio.com -d www.sudominio.com
```

Certbot debería modificar automáticamente el archivo de configuración del sitio en /etc/nginx/sites-avalibles/. Si todo está en orden, configure la renovación automática (los certificados de Let's Encrypt caducan cada 90 días):

``` bash
certbot renew --dry-run
```

Hechos todos los cambios, reinicie los servicios y habilitilos para que inicien al arrancar el servidor con `systemctl restart servicio` y `systemctl enable servicio`.