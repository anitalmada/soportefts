###Elastix - Actualización de repositorios para que funcionen los updates {#actualizarepos}

Actualizar los siguientes archivos:

En el archivo /etc/yum.repos.d/commercial-addons.repo, actualizar el bloque completo de commercial-addons, y dejarlo tal como se muestra a continuación:

```
[commercial-addons]
name=Commercial-Addons RPM Repository for Issabel
mirrorlist=http://mirror.issabel.org/?release=2.5&arch=$basearch&repo=commercial_addons
#baseurl=http://repo.issabel.org/elastix/2/commercial_addons/$basearch/
gpgcheck=1
enabled=1
gpgkey=http://repo.issabel.org/elastix/RPM-GPG-KEY-Issabel
```


En los archivos /etc/yum.repos.d/Centos-Base.repo y /etc/yum.repos.d/elastix.repo, actualizar los enlaces de los bloques de base, updates y extras, y dejarlo tal como se muestra a continuación:

```
baseurl=http://vault.centos.org/5.11/os/x86_64/

baseurl=http://vault.centos.org/5.11/updates/x86_64/

baseurl=http://vault.centos.org/5.11/extras/x86_64/
```

Finalmente, comentar el mirrorlist en los 3 bloques.

---

###Analizar colas de mails {#colasmails}

Inspecting Postfix’s email queue.

This post explains how to view messages in the postfix queue, another post on this blog explains how to delete or selectively delete from the postfix queue

Postfix maintains two queues, the pending mails queue, and the deferred mail queue, the differed mail queue has the mail that has soft-fail and should be retried (Temporary failure), Postfix retries the deferred queue on set intervals (configurable, and by default 5 minutes).

In any case, the following commands should be useful.

**Display a list of queued mail, deferred and pending**

`mailq`

or

`postqueue -p`

**To save the output to a text file you can run**

`mailq > myfile.txt`

or

`postqueue -p > myfile.txt`

The above commands display all queued messages (Not the message itself but the sender and recipients and ID), The ID is particularly useful if you want to inspect the message itself.
2- View message (contents, header and body) in Postfix queue

Assuming the message has the ID XXXXXXX (you can see the ID form the QUEUE)

`postcat -vq XXXXXXXXXX`

Or to save it in a file

`postcat -vq XXXXXXXXXX > themessage.txt`

3- Tell Postfix to process the Queue now

`postqueue -f`

OR

`postfix flush`

**Delete all queued mail**

`postsuper -d ALL`

**Delete differed mail queue messages** (the ones the system intends to retry later)

`postsuper -d ALL deferred`

**Delete from queue selectively**

To delete from the queue all emails that have a certain address in them, we can use [this program](http://www.buildingcubes.com/wp-content/files/postfix-queue-delete.zip "Perl Script v1") (perl script).

NOTE: This perl script seems to be free, and is all over the internet, i could not find out where it originates or who wrote it.

Download [this file](http://www.buildingcubes.com/wp-content/files/postfix-queue-delete.zip "Perl Script v1"), unzip, and upload the file to your server, then from your bash command line, Change Directory to wherever you uploaded this file, for example cd /root (Just an example, You can upload it wherever you wish)

NOTE: A second script here works differently, i have not yet tested it, [download it here](http://www.buildingcubes.com/wp-content/files/second_postfix-queue-delete.zip "Perl Script v2")

Now, from within that directory, execute

`./postfix-queue-delete.pl anyaddress@example.com`

Any mail that has this email address in it’s IN or OUT list will be deleted.

The script uses the postqueue -p then looks for your string, once found, it deletes the email by ID, this means that this script can delete messages using any text that appears when you run mailq (or postqueue -p), so if you run it with the parameter joe all mail with addresses such as joefriend@example.com and

Other moethods exist, like executing directly

`mailq | tail +2 | grep -v '^ *(' | awk  'BEGIN { RS = "" } { if ($8 == "email@address.com" && $9 == "") print $1 } ' | tr -d '*!' | postsuper -d -`

——————————–

Sample Messages in a differed mail queue

——————————–
```
SOME282672ID 63974 Mon Nov 29 05:12:30 someaddresss@yahoo.com (temporary failure. Command output: maildrop: maildir over quota.)
localuser@exmple.com
```
———————————-
```
SOME282672ID 9440 Wed Jun 30 05:30:11 MAILER-DAEMON (SomeHostName [xxx.xxx.xxx.xxx] said: 452  Mailbox size limit exceeded (in reply to RCPT TO command)) 
username@example.org
```

———————————-
```
SOME282672ID 4171 Thu Nov 25 13:22:03 MAILER-DAEMON (host inbound.somedomain.net [yyy.yyy.yyy.yyy] refused to talk to me: 550 Rejected: 188.xx.179.46, listed at http://csi.cloudmark.com/reset-request for remediation.)
someuser@example.com
```
———————————
```
SOME282672ID 37031 Thu Nov 25 08:53:36 someuser@example.net
(Host or domain name not found. Name service error for name=example.com type=MX: Host not found, try again)
someuser@example.com 
```

---

###Arreglar los faxes virtuales cuando dejan de funcionar {#arreglarfax}

Hay veces que al parar iaxmodem y hylafax, los procesos faxgetty quedan corriendo y no pueden pararse con kill.
Entonces, se debe comentar las líneas referidas a los faxgetty en `/etc/inittab`, correr `init q` para forzar que se re-lea dicho archivo, y después killear los faxgetty.
Luego, levantar iaxmodem, hylafax y los procesos faxgetty de forma manual:

`service iaxmodem start`
`service hylafax start`

```
/usr/sbin/faxgetty ttyIAX1 &
/usr/sbin/faxgetty ttyIAX2 &
...
...
...
```

---
###Backup and restore de Elastix {#bckuprestore}

1) Ambos servers donde se realizará el backup and restore, deben estar exactamente en las mismas versiones. En este caso, corrí un "yum update" en ambos servers, y sólamente el ALX tenía paquetes disponibles a ser actualizados (entre ellos, FreePBX).

2) En el ALX procedí a realizar un backup, desde el menú "Sistema > Respaldar/Restaurar". Allí, seleccioné la opción "Crear un respaldo...", seleccioné todas las opciones, y luego des-tildé "Monitoreos (Contenido Pesado)", "Correo de Voz (Contenido Pesado)", y "Archivos de configuración del panel de operaciones".
Las 2 opciones de contenido pesado, es para evitar que se realice un backup de las grabaciones y de los voicemails de los internos, agregando en vano varios gigas al archivo de backup. Y la tercer opción, es para evitar un backup de FOP, que al tratarse de un addon desarrollado por un tercero, puede traer inconvenientes.

3) El backup hecho en la página, crea un archivo en el ALX en el directorio `/var/www/backup`, del tipo "elastixbackup-20150724154014-c2.tar". Dicho archivo, fue copiado al MiniUCS vía SSH, en el mismo directorio `/var/www/backup`.

4) En el MiniUCS procedí a realizar el restore, desde el menú "Sistema > Respaldar/Restaurar". En el listado de respaldos, ya aparece el archivo copiado en el paso anterior, por lo que se procede a seleccionarlo y a la opción "Restaurar".

5) Una vez finalizado el restore desde la web, accedí vía SSH al MiniUCS para hacer un restore de las contraseñas. Para ello, se ejecuta el siguiente comando:

`[root@elxcliente ~]# elastix-admin-passwords --change`

Se vuelven a setear las contraseñas que se solicitan (web, freepbx, mysql, etc.).

6) Reboot.

---

###Cambiar pass de admin web, por consola

```
sqlite3 /var/www/db/acl.db "UPDATE acl_user SET md5_password = '`echo -n NUEVO-PASSWORD|md5sum|cut -d ' ' -f 1`' WHERE name = 'admin'"
```
---

###Cluster de 64bits {#cluster}

**En ambos nodos:**

* Particionar: 1024 x N ----> / ext3. 1024 x N ----> Swap 
Reboot

`yum update`
`yum install heartbeat.x86_64 drbd83.x86_64 kmod-drbd83.x86_64`

- Editar el archivo `/etc/hosts` en cada nodo:

```
xxx.xxx.xxx.xxx pbxa.example.com    pbxa
xxx.xxx.xxx.xxx pbxb.example.com    pbxb
```

Donde XXX.XXX.XXX.XXX es la IP de las placas de red de réplica (eth1).
* Comprobar que ambos nodos se "vean" a través de su IP ETH1.

**Trabajar sobre la partición DRBD (en cada nodo)**

Es importante que las particiones tengan el mismo tamaño. Si los discos y la tabla de partición de ambos nodos son idénticos simplemente creamos la partición utilizando el resto del espacio disponible. 
Por el contrario, debemos dimensionar esta nueva partición utilizando un tamaño en Megabytes (por ejemplo).

Nueva particion "n", primaria "p", numero "3", tamaño "+(1024 * X)M".
Ingresar "t" para cambiar formato, elegir particion "3", indicar formato "83", imprimir la tabla "p".

`fdisk /dev/sda`
`1024 x N`

Corroborar que queden iguales con `fdisk -l`

Si todo está ok, guardar cambios.

- Formatear la nueva particion en cada server: 

    `mke2fs -j /dev/sda3`
    `dd if=/dev/zero bs=1M count=1 of=/dev/sda3; sync`

* Configurar /etc/drbd.conf en el nodo 1.

```
global { usage-count no; }
resource r0 {
protocol C;
startup { wfc-timeout 10; degr-wfc-timeout 30; }
disk { on-io-error detach; }
net {
        after-sb-0pri discard-younger-primary; 
        after-sb-1pri discard-secondary;
        after-sb-2pri call-pri-lost-after-sb;
        cram-hmac-alg "sha1";
        shared-secret "31asT1X666";
        }

syncer { rate 100M; }
on pbxa.example.com {
        device /dev/drbd0;
        disk /dev/sda3;
        address xxx.xxx.xxx.xx1:7788;
        meta-disk internal;
        }
on pbxb.example.com {
        device /dev/drbd0;
        disk /dev/sda3;
        address xxx.xxx.xxx.xx2:7788;
        meta-disk internal;
        } 
}
```

Donde xxx.xxx.xx1 es la IP del la interfaz de cluster del server 1 y xxx.xxx.xxx.xx2 es la del server 2.

* Copiar el archivo generado desde el nodo 1 al nodo 2.

    `scp /etc/drbd.conf root@pbxb.example.com:/etc/`

* Inicializar el sector del disco DRBD (metadata). En ambos nodos ejecutar:

    `drbdadm create-md r0`

* Levantar el servicio DRBD. En ambos nodos: 

    `service drbd start`
    
* Comprobar si ambos servers están como "secundarios". En ambos nodos: 

    `cat /proc/drbd`

Deberíamos observar los dos en secondary/secondary

Ahora vamos a iniciar como nodo primario al pbxa.

**Solo en el pbxa:** 

    drbdadm -- --overwrite-data-of-peer primary r0

En el nodo pbxb se puede comprobar la sincronización: 

    cat /proc/drbd
    
* Ahora vamos a formatear el nuevo device /dev/drbd0 y vamos a montarlo en un directorio /replica que vamos a crear.

**En ambos nodos:**

    mkdir /replica

**Solo en el pbxa:**

```
    mkfs.ext3 /dev/drbd0 
    tune2fs -c 0 -i 0 /dev/drbd0
    mount /dev/drbd0 /replica
```
    
* Comprobamos el rol de cada nodo, el pbxa debería arrojar: `Primary/Secondary` y el pbxb: `Secundary/Primary`

    `drbdadm role r0`

- Replicar directorios y archivos del sistema en el nodo a

```
cd /replica
tar czvf etc-asterisk.tgz /etc/asterisk
tar czvf etc-asterisk.elastix.tgz /etc/asterisk.elastix
tar czvf opt-lcdelastix.tgz /opt/lcdelastix
tar czvf usr-lib-asterisk.tgz /usr/lib64/asterisk
tar czvf var-lib-asterisk.tgz /var/lib/asterisk
tar czvf var-lib-mysql.tgz /var/lib/mysql
tar czvf var-www.tgz /var/www
tar czvf var-spool-asterisk.tgz /var/spool/asterisk
tar czvf tftpboot.tgz /tftpboot

tar xzvf etc-asterisk.tgz
tar xzvf etc-asterisk.elastix.tgz
tar xzvf opt-lcdelastix.tgz
tar xzvf usr-lib-asterisk.tgz
tar xzvf var-lib-asterisk.tgz
tar xzvf var-lib-mysql.tgz
tar xzvf var-www.tgz
tar xzvf var-spool-asterisk.tgz
tar xzvf tftpboot.tgz
cp -p /etc/odbc.ini /replica
cp -p /etc/amportal.conf /replica
cp -p /etc/freepbx.conf /replica

rm -rf /etc/asterisk
rm -rf /etc/asterisk.elastix
rm -rf /opt/lcdelastix
rm -rf /usr/lib64/asterisk
rm -rf /var/lib/asterisk
rm -rf /var/lib/mysql
rm -rf /var/www
rm -rf /var/spool/asterisk
rm -rf /tftpboot
rm -rf /etc/odbc.ini 
rm -rf /etc/amportal.conf 
rm -rf /etc/freepbx.conf 

ln -s /replica/etc/asterisk /etc/asterisk
ln -s /replica/etc/asterisk.elastix /etc/asterisk.elastix
ln -s /replica/opt/lcdelastix /opt/lcdelastix
ln -s /replica/usr/lib64/asterisk /usr/lib64/asterisk
ln -s /replica/var/lib/asterisk /var/lib/asterisk
ln -s /replica/var/lib/mysql /var/lib/mysql
ln -s /replica/var/www /var/www
ln -s /replica/var/spool/asterisk /var/spool/asterisk
ln -s /replica/tftpboot /tftpboot
ln -s /replica/odbc.ini /etc/odbc.ini 
ln -s /replica/amportal.conf /etc/amportal.conf 
ln -s /replica/freepbx.conf /etc/freepbx.conf 
```

* Bajar servicios

    `service httpd stop`
    `service asterisk stop`
    `service mysqld stop`
    
Cambiar el nodo b a primario y trabajar sobre la réplica
    
**En pbxa:** 

    umount /replica
    drbdadm secondary r0
    
**En pbxb:**

    drbdadm primary r0
    mount /dev/drbd0 /replica
    ls /replica/
    
* En ambos comprobar roles: 

    `drbdadm role r0`
    
* Trabajar sobre la replicación de archivos.

```
rm -rf /etc/asterisk
rm -rf /etc/asterisk.elastix
rm -rf /opt/lcdelastix
rm -rf /usr/lib64/asterisk
rm -rf /var/lib/asterisk
rm -rf /var/lib/mysql
rm -rf /var/www
rm -rf /var/spool/asterisk
rm -rf /tftpboot
rm -rf /etc/odbc.ini 
rm -rf /etc/amportal.conf 
rm -rf /etc/freepbx.conf 

ln -s /replica/etc/asterisk /etc/asterisk
ln -s /replica/etc/asterisk.elastix /etc/asterisk.elastix
ln -s /replica/opt/lcdelastix /opt/lcdelastix
ln -s /replica/usr/lib64/asterisk /usr/lib64/asterisk
ln -s /replica/var/lib/asterisk /var/lib/asterisk
ln -s /replica/var/lib/mysql /var/lib/mysql
ln -s /replica/var/www /var/www
ln -s /replica/var/spool/asterisk /var/spool/asterisk
ln -s /replica/tftpboot /tftpboot
ln -s /replica/odbc.ini /etc/odbc.ini 
ln -s /replica/amportal.conf /etc/amportal.conf 
ln -s /replica/freepbx.conf /etc/freepbx.conf 
```

* Bajar servicios

    `service mysqld stop`
    `service httpd stop`
    `service asterisk stop`
    
* Volver al pbxa como primario

**En pbxb:**

    umount /replica
    drbdadm secondary r0
    
**En pbxa**

    drbdadm primary r0
    mount /dev/drbd0 /replica
    ls /replica/
    
**En ambos:**
 
    drbdadm role r0


* Configuración de Heartbeat

Desactivo y bajo los servicios que va a manejar HA (en **ambos nodos**) 

    chkconfig asterisk off 
    chkconfig mysqld off 
    chkconfig httpd off 
    service mysqld stop 
    service asterisk stop 
    service httpd stop

* Crear el archivo `/etc/ha.d/ha.cf` con el siguiente contenido:

```
debugfile /var/log/ha-debug
logfile /var/log/ha-log
logfacility local0 
keepalive 2
deadtime 30
warntime 10
initdead 120
udpport 694
bcast eth1 
auto_failback off
node pbxa.example.com
node pbxb.example.com
```

* Crear el archivo `/etc/ha.d/authkeys` con el siguiente contenido:

    `auth 1`
    `1 sha1 MySecret` 
    

* Asignar permisos: 

    `chmod 600 /etc/ha.d/authkeys`

* Crear el archivo /etc/ha.d/haresources con el sgte contenido:

```
pbxa.example.com drbddisk::r0 Filesystem::/dev/drbd0::/replica::ext3 IPaddr::xxx.xxx.xxx.xxx/24/eth0/xxx.xxx.xxx.255 mysqld asterisk httpd
```

Donde xxx.xxx.xxx.xxx es la IP flotante del cluster y xxx.xxx.xxx.255 es la IP de broadcast.

* Copiar los archivos de HA en el nodo pbxb

    `scp /etc/ha.d/ha.cf /etc/ha.d/authkeys /etc/ha.d/haresources root@pbxb.example.com:/etc/ha.d/`
    
* Levantar heartbeat en ambos nodos y dejarlo para que levante en el runlevel 3

    `service heartbeat start`
    `chkconfig heartbeat on`

Reboot en ambos servidores y el cluster debería comenzar a funcionar.

*NOTA:* ojo con el archivo: `/etc/ha.d/ha.cf` y `/etc/ha.d/haresources` ya que llevan el nombre de la interfaz `eth` (eth0, eth1, etc.). Debe llevar la eth de replicación.

*NOTA2:* tener el cuenta el parámetro BINDADDR en Freepbx, debe tener la IP flotante.- En ambos nodos:

* Particionar: 1024 x N ----> / ext3. 1024 x N ----> Swap 

    `Reboot`
    `yum update`
    `yum install heartbeat.x86_64 drbd83.x86_64 kmod-drbd83.x86_64`

- Editar el archivo /etc/hosts en cada nodo:

```
xxx.xxx.xxx.xxx pbxa.example.com    pbxa
xxx.xxx.xxx.xxx pbxb.example.com    pbxb
```

Donde XXX.XXX.XXX.XXX es la IP de las placas de red de réplica (eth1).
Comprobar que ambos nodos se "vean" a través de su IP ETH1.

* Trabajar sobre la partición DRBD (en cada nodo)

Es importante que las particiones tengan el mismo tamaño. Si los discos y la tabla de partición de ambos nodos son idénticos simplemente creamos la partición utilizando el resto del espacio disponible. 
Por el contrario, debemos dimensionar esta nueva partición utilizando un tamaño en Megabytes (por ejemplo).

Nueva particion "n", primaria "p", numero "3", tamaño "+(1024 * X)M".
Ingresar "t" para cambiar formato, elegir particion "3", indicar formato "83", imprimir la tabla "p".

    fdisk /dev/sda
    1024 x N

Corroborar que queden iguales con

    fdisk -l

Si todo está ok, guardar cambios.

* Formatear la nueva particion en cada server: 

    `mke2fs -j /dev/sda3`
    `dd if=/dev/zero bs=1M count=1 of=/dev/sda3; sync`

- Configurar `/etc/drbd.conf` en el nodo 1.

```
global { usage-count no; }
resource r0 {
protocol C;
startup { wfc-timeout 10; degr-wfc-timeout 30; }
disk { on-io-error detach; }
net {
        after-sb-0pri discard-younger-primary; 
        after-sb-1pri discard-secondary;
        after-sb-2pri call-pri-lost-after-sb;
        cram-hmac-alg "sha1";
        shared-secret "31asT1X666";
        }

syncer { rate 100M; }
on pbxa.example.com {
        device /dev/drbd0;
        disk /dev/sda3;
        address xxx.xxx.xxx.xx1:7788;
        meta-disk internal;
        }
on pbxb.example.com {
        device /dev/drbd0;
        disk /dev/sda3;
        address xxx.xxx.xxx.xx2:7788;
        meta-disk internal;
        } 
}
```

Donde xxx.xxx.xx1 es la IP del la interfaz de cluster del server 1 y xxx.xxx.xxx.xx2 es la del server 2.

* Copiar el archivo generado desde el nodo 1 al nodo 2.

    `scp /etc/drbd.conf root@pbxb.example.com:/etc/`

* Inicializar el sector del disco DRBD (metadata). En ambos nodos ejecutar:

    `drbdadm create-md r0`

* Levantar el servicio DRBD. En ambos nodos: 

    `service drbd start`
    
* Comprobar si ambos servers están como "secundarios". En ambos nodos: 

    `cat /proc/drbd`

Deberíamos observar los dos en secondary/secondary

* Ahora vamos a iniciar como nodo primario al pbxa.

**Solo en el pbxa:** 

    drbdadm -- --overwrite-data-of-peer primary r0

En el **nodo pbxb** se puede comprobar la sincronización: 

    cat /proc/drbd
    
* Ahora vamos a formatear el nuevo device `/dev/drbd0` y vamos a montarlo en un directorio `/replica` que vamos a crear.

**En ambos nodos:**

    mkdir /replica

**Solo en el pbxa:**

    mkfs.ext3 /dev/drbd0
    tune2fs -c 0 -i 0 /dev/drbd0
    mount /dev/drbd0 /replica
    
* Comprobamos el rol de cada nodo, el pbxa debería arrojar: `Primary/Secondary` y el pbxb: `Secundary/Primary`

    `drbdadm role r0`

* Replicar directorios y archivos del sistema en el nodo a

```
cd /replica
tar czvf etc-asterisk.tgz /etc/asterisk
tar czvf etc-asterisk.elastix.tgz /etc/asterisk.elastix
tar czvf opt-lcdelastix.tgz /opt/lcdelastix
tar czvf usr-lib-asterisk.tgz /usr/lib64/asterisk
tar czvf var-lib-asterisk.tgz /var/lib/asterisk
tar czvf var-lib-mysql.tgz /var/lib/mysql
tar czvf var-www.tgz /var/www
tar czvf var-spool-asterisk.tgz /var/spool/asterisk
tar czvf tftpboot.tgz /tftpboot

tar xzvf etc-asterisk.tgz
tar xzvf etc-asterisk.elastix.tgz
tar xzvf opt-lcdelastix.tgz
tar xzvf usr-lib-asterisk.tgz
tar xzvf var-lib-asterisk.tgz
tar xzvf var-lib-mysql.tgz
tar xzvf var-www.tgz
tar xzvf var-spool-asterisk.tgz
tar xzvf tftpboot.tgz
cp -p /etc/odbc.ini /replica
cp -p /etc/amportal.conf /replica
cp -p /etc/freepbx.conf /replica

rm -rf /etc/asterisk
rm -rf /etc/asterisk.elastix
rm -rf /opt/lcdelastix
rm -rf /usr/lib64/asterisk
rm -rf /var/lib/asterisk
rm -rf /var/lib/mysql
rm -rf /var/www
rm -rf /var/spool/asterisk
rm -rf /tftpboot
rm -rf /etc/odbc.ini 
rm -rf /etc/amportal.conf 
rm -rf /etc/freepbx.conf 

ln -s /replica/etc/asterisk /etc/asterisk
ln -s /replica/etc/asterisk.elastix /etc/asterisk.elastix
ln -s /replica/opt/lcdelastix /opt/lcdelastix
ln -s /replica/usr/lib64/asterisk /usr/lib64/asterisk
ln -s /replica/var/lib/asterisk /var/lib/asterisk
ln -s /replica/var/lib/mysql /var/lib/mysql
ln -s /replica/var/www /var/www
ln -s /replica/var/spool/asterisk /var/spool/asterisk
ln -s /replica/tftpboot /tftpboot
ln -s /replica/odbc.ini /etc/odbc.ini 
ln -s /replica/amportal.conf /etc/amportal.conf 
ln -s /replica/freepbx.conf /etc/freepbx.conf 
```

* Bajar servicios

```
service httpd stop
service asterisk stop
service mysqld stop
```
    
* Cambiar el nodo b a primario y trabajar sobre la réplica
    
**En pbxa:**

    umount /replica
    drbdadm secondary r0
    
**En pbxb:**
 
    drbdadm primary r0
    mount /dev/drbd0 /replica
    ls /replica/
    
**En ambos comprobar roles:** 

    drbdadm role r0
    
* Trabajar sobre la replicación de archivos.

```
rm -rf /etc/asterisk
rm -rf /etc/asterisk.elastix
rm -rf /opt/lcdelastix
rm -rf /usr/lib64/asterisk
rm -rf /var/lib/asterisk
rm -rf /var/lib/mysql
rm -rf /var/www
rm -rf /var/spool/asterisk
rm -rf /tftpboot
rm -rf /etc/odbc.ini 
rm -rf /etc/amportal.conf 
rm -rf /etc/freepbx.conf 

ln -s /replica/etc/asterisk /etc/asterisk
ln -s /replica/etc/asterisk.elastix /etc/asterisk.elastix
ln -s /replica/opt/lcdelastix /opt/lcdelastix
ln -s /replica/usr/lib64/asterisk /usr/lib64/asterisk
ln -s /replica/var/lib/asterisk /var/lib/asterisk
ln -s /replica/var/lib/mysql /var/lib/mysql
ln -s /replica/var/www /var/www
ln -s /replica/var/spool/asterisk /var/spool/asterisk
ln -s /replica/tftpboot /tftpboot
ln -s /replica/odbc.ini /etc/odbc.ini 
ln -s /replica/amportal.conf /etc/amportal.conf 
ln -s /replica/freepbx.conf /etc/freepbx.conf 
```

* Bajar servicios

    `service mysqld stop`
    `service httpd stop`
    `service asterisk stop`
    
* Volver al pbxa como primario

**En pbxb:**

    umount /replica
    drbdadm secondary r0
    
**En pbxa**

    drbdadm primary r0
    mount /dev/drbd0 /replica
    ls /replica/
    
**En ambos: **

    drbdadm role r0


* Configuración de Heartbeat

Desactivo y bajo los servicios que va a manejar HA (en ambos nodos) 

    chkconfig asterisk off 
    chkconfig mysqld off 
    chkconfig httpd off 
    service mysqld stop 
    service asterisk stop 
    service httpd stop

* Crear el archivo `/etc/ha.d/ha.cf` con el siguiente contenido:

    ```
    debugfile /var/log/ha-debug
    logfile /var/log/ha-log
    logfacility local0 
    keepalive 2
    deadtime 30
    warntime 10
    initdead 120
    udpport 694
    bcast eth1 
    auto_failback off
    node pbxa.example.com
    node pbxb.example.com
    ```

* Crear el archivo `/etc/ha.d/authkeys` con el sgte contenido:

    `auth 1`
    `1 sha1 MySecret`    

* Asignar permisos: 

    `chmod 600 /etc/ha.d/authkeys`

* Crear el archivo /etc/ha.d/haresources con el sgte contenido:

    `pbxa.example.com drbddisk::r0 `
    `dev/drbd0::/replica::ext3`
    `IPaddr::xxx.xxx.xxx.xxx/24/eth0/xxx.xxx.xxx.255 mysqld asterisk httpd`

Donde xxx.xxx.xxx.xxx es la IP flotante del cluster y xxx.xxx.xxx.255 es la IP de broadcast.

* Copiar los archivos de HA en el nodo pbxb

    `scp /etc/ha.d/ha.cf /etc/ha.d/authkeys /etc/ha.d/haresources root@pbxb.example.com:/etc/ha.d/`
    
* Levantar heartbeat en ambos nodos y dejarlo para que levante en el runlevel 3

    `service heartbeat start`
    `chkconfig heartbeat on`

Reboot en ambos servidores y el cluster debería comenzar a funcionar.

*NOTA:* ojo con el archivo: `/etc/ha.d/ha.cf` y `/etc/ha.d/haresources` ya que llevan el nombre de la interfaz eth (eth0, eth1, etc.)
Debe llevar la eth de replicacion.

*NOTA2:* tener el cuenta el parámetro BINDADDR en Freepbx, debe tener la IP flotante.

---
###Cluster HA - Particionado de disco {#clusterparticionado}

*Particionado:*
En la guía de instalación, se parte de un diseño predeterminado seleccionando el disco (en el ejemplo, se usa un disco chico):
![](/assets/ClusterHA_Particionado0.png)


Luego se acepta la modificación de la capa de particiones:
![](/assets/ClusterHA_Particionado1.png)


Se clickea en NUEVO, sobre el Espacio Libre:

![](/assets/ClusterHA_Particionado2.png)

BOOT: se crea la partición /boot del tipo ext3 con un tamaño de 100MB:

![](/assets/ClusterHA_Particionado3.png)


SWAP: se crea una partición del tipo swap con un tamaño en megas igual al doble de RAM (ej: para 8 GB de RAM pondríamos 16000)
![](/assets/ClusterHA_Particionado4.png)

TMP: Seteamos una partición /tmp con un tamaño de 50 gigas:

![](/assets/ClusterHA_Particionado5.png)

RAÍZ: Se crea un partición raíz (/) con un tamaño igual a 40 gigas (40000):
![](/assets/ClusterHA_Particionado6.png)

Finalmente se deja todo el espacio libre restante sin usar, y se procede con la instalación normal del sistema.
Cuando se llegue a la parte de Direccionamiento IP, deberíamos encontrarnos con 2 interfaces:

- eth0: que se usara como IP de producción 1
- eth1: que se usará como IP de heartbeat (el direccionamiento aquí es interno, ya que esta placa se usará para conectar físicamente en punto a punto el otro nodo para enlace de heartbeat: ej. 192.168.100.1/255.255.255.0).

Mismo procedimiento de particionado debe seguirse en el otro nodo, especificando una IP de heartbeat dentro del rango (ejemplo: 192.168.100.2/255.255.255.0).

La capa de particiones debe quedar "exacta" en ambos nodos, con mismo valor de espacio libre.

---

###Comandos útiles en Elastix {#comandosutiles}

`asterisk -rx ' sip show peers'`
`asterisk -rx 'dahdi show channels'`
`asterisk -rx 'core show channels'`	---> `asterisk -rx 'core show channels'|awk -F"/" '{print $1}'|grep "DAHDI"|wc -l`
`lsdahdi`	---> `lsdahdi|grep " ACTIVE"|wc -l`				| Correspondencia
`asterisk -rx 'queue show 9082'`

Asterisk Manager esucha en el puerto 5038
El proceso op_server.pl corresponde al FOP.

```
; The next three params are for debuging, you can disable when in production

mfcr2_call_files=yes
mfcr2_logdir=oi
mfcr2_logging=all 
```

El cliente también puede ingresar a la interfaz web y gestionar sus mensajes de voz de alli y por supuesto desde un teléfono interno (como todas las centrales) discando al *98

`asterisk -rx 'module reload app_queue.so'` ---> Recarga la configuración de colas

---
###Consulta sqlite para ver los vendors soportados {#consultavendors}

    sqlite3 /var/www/db/endpoint.db "select vendor.name , model.name from model , vendor where model.id_vendor = vendor.id order by vendor.name;"
    
---
###DUNDi - Arreglar conexionaes a nivel sqlite {#dundisqlite}

En el servidor central del cliente, eliminé el peer en la base de datos:

```
[root@pbxcliente ~]# cd /var/www/db/
[root@pbxcliente db]# sqlite3 elastixconnection.db
SQLite version 3.3.6
Enter ".help" for instructions

sqlite> .databases
seq  name             file
---  ---------------  ----------------------------------------------------------
0    main             /var/www/db/elastixconnection.db

sqlite> .tables
general    parameter  peer

sqlite> select * from peer where host='10.4.70.250';
46|a0:98:05:01:d7:13|symmetric|10.4.xxx.xxx|CERxxxxxxxxxxxx|CERxxxxxxxxxxxx|connected|-----BEGIN PUBLIC KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END PUBLIC KEY-----
|Nombre de Cliente|sigla|connected

sqlite> delete from peer where id='46';
```

Una vez eliminado en base de datos, deja de aparecer a nivel web el nodo. Recién ahí, me voy al nodo remoto y genero una nueva conexión, y acepto en el servidor central.
Sin embargo, queda en estado "Conectando...", por lo que vuelvo a editar el peer en base de datos, en el nodo remoto:

```
[root@elx keys]# cd /var/www/db/
[root@elx db]# sqlite3 elastixconnection.db
SQLite version 3.3.6
Enter ".help" for instructions

sqlite> .databases
seq  name             file
---  ---------------  ----------------------------------------------------------
0    main             /var/www/db/elastixconnection.db

sqlite> .tables
general    parameter  peer

sqlite> select * from general;
1|XXX|CABA|BsAs|Argentina|tecnologia@example.com.ar|011XXXXXXX|Tecnologia|CERXXXXXXXXXXXX|xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

sqlite> select * from peer;
1||symmetric|10.4.Xx.xX||CERXXXXXXXXXXXX|waiting response||||Requesting connection

sqlite> .schema
CREATE TABLE general(
       id           INTEGER PRIMARY KEY,
       organization varchar(100),
       locality     varchar(100),
       stateprov    varchar(150),
       country      varchar(150),
       email        varchar(150),
       phone        varchar(20),
       department   varchar(255),
       certificate  varchar(255)
, secret text);
CREATE TABLE parameter(
       id           INTEGER PRIMARY KEY,
       name         varchar(255),
       value        varchar(255),
       id_peer      integer,
       foreign key(id_peer) references peer(id)
);
CREATE TABLE peer(
       id           INTEGER PRIMARY KEY,
       mac          varchar(255),
       model        varchar(255),
       host         varchar(255),
       inkey        varchar(255),
       outkey       varchar(255),
       status       varchar(25)  default 'connect',
       key          text,
       comment      text,
       company      varchar(250),
       his_status   varchar(255) default 'disconnected'
);

sqlite> update peer set mac = 'Xc:Xe:8X:Xd:2X:XX' where id = 1;
sqlite> update peer set inkey = 'CERXXXXXXXXXXXXX' where id = 1;
sqlite> update peer set status = 'connected' where id = 1;
sqlite> update peer set his_status = 'connected' where id = 1;

sqlite> select * from peer;
1|Xc:Xe:8X:Xd:2X:XX|symmetric|10.4.XX.XX|CER6XXXXXXXXXXX|CERXXXXXXXXXXXXX|connected||||connected

sqlite> .exit

[root@elx ~]# cd /var/lib/asterisk/keys/
[root@elx keys]# ls -l
total 12
-rw-r--r-- 1 asterisk asterisk 891 sep 14 07:51 CERXXXXXXXXXXXXX.key
-rw-r--r-- 1 asterisk asterisk 272 sep 14 07:51 CERXXXXXXXXXXXXX.pub
```

Me traje la key pública de Casa Central y lo pegué en éste directorio:

```
[root@elx keys]# ls -l
total 12
-rw-r--r-- 1 asterisk asterisk 272 sep 14 11:53 CERXXXXXXXXXXXXX.pub
-rw-r--r-- 1 asterisk asterisk 891 sep 14 07:51 CERXXXXXXXXXXXXX.key
-rw-r--r-- 1 asterisk asterisk 272 sep 14 07:51 CERXXXXXXXXXXXXX.pub

[root@elx keys]# cat CERXXXXXXXXX.pub
-----BEGIN PUBLIC KEY-----
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
-----END PUBLIC KEY-----
```

---
###Extensión de prueba {#extensiontest}

Ir a PBX ---> Misc Applications
Agregar una descripción
Poner un código, que va a ser el que desde una extensión se disque para acceder a esa aplicación
Setear el destination.

---
###FXOtune {#fxotune}

    #amportal stop

    ~#fxotune -d -b 1
    d=dump results
    i=calibrate echo settings
    b= device (cat /proc/dahdi/X) --> ojo que suele ser /dev/dahdi/X (X=ports)

    Dumping module /dev/dahdi/1

    echo ratio = 0.2886 (1305.9 / 4557.0)
    Done!

    Dumping module /dev/dahdi/2
    echo ratio = 0.3889 (1544.3 / 4557.0)
    Done!


    ~#fxotune -i 5
    i = calibrate echo settings
    5= string to dial to clear the line, default is 5

    ~#fxotune -s
    s=set calibrated settings

Volvemos a cargar los resultados para ver si estos bajaron los niveles de eco.

    Dumping module /dev/dahdi/1
    echo ratio = 0.0096 (51.9 / 4457.0)
    Done!

    Dumping module /dev/dahdi/2
    echo ratio = 0.0068 (43.7 / 4457.0)
    Done!

    ~#amportal start

Nota: Es necesario ejecutar `fxotune -s` después de cada test

---
###Grabar todas las llamadas entrantes por un DID particular {#grabarllamxdid}

Se tiene una ruta entrante, con el DID 9342, y sin importar a dónde vaya, se debe grabar la conversación completa y enviar un mail con el audio adjunto.

En el archivo `/etc/asterisk/extensions_custom.conf`:

```
[from-pstn-custom]
exten => 9342,1,Set(Fecha=${STRFTIME(${EPOCH},,%Y%m%d-%H%M%S)})
exten => 9342,n,Set(Grabacion=/tmp/Recording-${Fecha}-${EXTEN}-${CALLERID(number)}.wav)
exten => 9342,n,MixMonitor(${Grabacion},bm(3999@default),/var/lib/asterisk/bin/ModoGrabarOn.sh ${Grabacion})
exten => 9342,n,Goto(ext-did-0002,9342,1)
```

Creamos el script `/var/lib/asterisk/bin/ModoGrabarOn.sh`:

```
#!/bin/bash

# Inicialización de variables
Lame="`which lame`"
Mimesend="`which mimesend`"
Rm="`which rm`"

File=$1

Fecha="`echo ${File}|awk -F"-" '{print $2}'`"
Hora="`echo ${File}|awk -F"-" '{print $3}'`"
DID="`echo ${File}|awk -F"-" '{print $4}'`"
CallerID="`echo ${File}|awk -F"-" '{print $5}'| awk -F"." '{print $1}'`"

Filemp3="`echo ${File}| awk -F"." '{print $1}'`"

To="rodrigo.montiel@freetechsolutions.com.ar"
Subject="Recorded call '$CallerID'"
#############################

touch /tmp/${CallerID}.txt
echo "From: $CallerID" >> /tmp/${CallerID}.txt
echo "To: $DID" >> /tmp/${CallerID}.txt
echo "Date: $Fecha $Hora" >> /tmp/${CallerID}.txt

${Lame} --silent -m m -b 8 --tt ${File} --add-id3v2 ${File} ${Filemp3}.mp3

sleep 5

${Mimesend} -t ${To} -s "${Subject}" -f /tmp/${CallerID}.txt -f ${Filemp3}.mp3

${Rm} -f /tmp/${CallerID}.txt
${Rm} -f ${File}
```
---
###Habilitar detección de cortes en placas con puertos FXO {#deteccióncortefxo}

1. Descomentar las siguientes 2 líneas en `/etc/asterisk/chan_dahdi.conf`:

    ;Uncomment these lines if you have problems with the     disconection of your analog lines
    busydetect=yes
    busycount=3

2. Hacer un reload del módulo de dahdi:

    asterisk -r
    module reload chan_dahdi.so

---
###Habilitar log en Elastix 2.5 {#logelastix25}

For the log work after upgrade you must put `VERBOSITY=3` in `/etc/sysconfig/asterisk` file

    echo 'VERBOSITY=3' >> /etc/sysconfig/asterisk

After restart asterisk with `/etc/init.d/asterisk restart`

For amportal restart works too, you must edit the file `/var/lib/asterisk/bin/freepbx_engine`

```
/usr/sbin/safe_asterisk -U $AMPASTERISKUSER -G $AMPASTERISKGROUP 2>&1 >/dev/null
```

by

```
/usr/sbin/safe_asterisk -U $AMPASTERISKUSER -G $AMPASTERISKGROUP -v -v -v 2>&1 >/dev/null
```

`core restart` now don't do anything

---
###Habilitar transferencia en llamadas salientes {#habilitartransfer}

En PBX ---> Configuración PBX ---> Configuración general, agregar la opción T en "Asterisk Outbound Dial Command options".

---
###Habilitar usuario para ver todas las llamadas {#habuserver}

```
rwxr-xr-x 1 root root  53K Jan 30 13:39 paloSantoACL.class.php
-rwxr-xr-x 1 root root  53K May 24 10:38 paloSantoACL.class.php_24May2013_bkp
```

    [root@user libs]# pwd
    /var/www/html/libs

```
/**
     * Procedimiento para saber si un usuario (login) pertenece al grupo administrador
     *
     * @param string   $username  Username del usuario
     *
     * @return boolean true or false
     */
    function isUserAdministratorGroup($username)
    {
        $is=false;
        $idUser = $this->getIdUser($username);
        if($idUser){
            $arrGroup = $this->getMembership($idUser);
            //$is = array_key_exists('administrator',$arrGroup);
            $is = array_search('1', $arrGroup);
            if ($username == "jpalotes") $is=1; //aca agregue el usuario de Juan de los Palotes   <--------- Esta línea se debe agregar
        }
        return $is;
    }
```
---
###HA-Recuperación de splitbrain {#splitbrain}

http://www.ipserverone.info/dedicated-server/linux-2/how-to-fix-drbd-recovery-from-split-brain/

---
###Instalar custom context en Elastix {#customcontext}

FreePBX
Module Admin ---> Check for updates online ---> Custom contexts ---> Action ---> Download and install ---> Process ---> Confirm ---> Return

---
###Insertar botón Manual Call en el módulo call center {#btnmanualcall}


1. Moverse a `/var/www/html/modules/agent_console` con cd.

2. Editar el `index.php` bajo el comentario     `// Acciones para mostrar la pantalla principal, fuera de cualquier acción AJAX` (en linea 552 aprox.)
 y agrega " `'BTN_LLAMADA_MANUAL'            =>  _tr('Manual Call')," dentro de $smarty->assing(array( )). Quedando $smarty->assing(array('BTN_LLAMADA_MANUAL' => _tr('Manual Call'),));`

3. Editar `themes/default/agent_console.tpl` y agrega 
    `<button id="btn_manualcall" class="elastix-callcenter-boton-activo ui-button ui-widget ui-state-default ui-corner-all ui-button-text-only" role="button" aria-disabled="false"><span class="ui-button-text">{$BTN_LLAMADA_MANUAL}</span></button>`

4. Agregar las siguientes lineas en `themes/default/js/javascript.js`

```
    // Pongo en pausa al agente
 $('#btn_manualcall').click(function() {
     if($('div[dialog=click2Dial]').length == 0) {
         $.ajax({
             url: '/freetech_ccenter/getForm.php',
             success: function(data) {
                          $('<div />').attr('dialog', 'click2Dial').html(data).dialog({
                          title: 'Freetech Solutions | click2Dial',
                          width: 'auto',
                          height: 'auto',
                          close: function(event, ui) {
                                     $(this).dialog('destroy').remove();
                                 }
                          });
                     }
         });
     }
});
```

5. Agregar los archivos php en `/var/www/html/freetech_ccenter/`

6. Setear las credenciales de MySQL y AMI en el archivo `config.php`

---







