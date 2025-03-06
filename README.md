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

Antes de empezar, hay que comentar que Debian no viene con **sudo** instalado de forma predeterminada, si lo desea puede instalarlo y configurarlo (https://www.cyberciti.biz/faq/how-to-install-and-configure-sudo-on-debian-linux/), en mi caso opté por correr todos los comandos desde root. Lo primero siempre es actualizar el sistema:  

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