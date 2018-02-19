###Agregar usuario a un grupo {#agguser}

    adduser -G wheel -m -s /bin/bash fts
---
###Arreglar Wireshark en Ubuntu {#wiresharkubuntu}

Para tener permisos de usar las interfaces para capturas, correr el siguiente comando:
				
	setcap 'CAP_NET_RAW+eip CAP_NET_ADMIN+eip' /usr/bin/dumpcap

Para solucionar el crash al realizar capturas o abrir capturas, editar el archivo `/usr/share/applications/wireshark.desktop`:

Cambiar la siguiente línea:
	
	Exec=wireshark %f

Por lo siguiente:

	Exec=env LIBOVERLAY_SCROLLBAR=0 wireshark %f
	
---
###Backup del NFS {#bckpnfs}

La idea es hacer un backup de todas las carpetas que están en la carpeta `/home/fts`, excepto la carpeta ISOs. La forma de hacerlo, sería la siguiente:

	cd /home
	tar -pczvf Backup.tar.gz fts/ --exclude fts/ISOs

Luego, este backup se debe copiar a la máquina huésped. Ya fue implementado el acceso por SSH sin autenticación del usuario root en la máquina central al usuario root en la máquina huésped.

---
###Captura y análisis de tráfico {#capturatrafico}

- Capturar con tcpdump desde la central (`tcpdump -s0 -w file.pcap udp port 5060 / tcpdump -s0 -w /root/fritura.pcap`);
- Hacer un llamado que se necesite, por ejemplo desde afuera entrar por trama y de alli a un interno;
- Cortar la captura;
- Copiar la captura desde la central a la máquina local y abrirla con wireshark;
- Ir a **Telephony ---> VoIP Calls**. Seleccionar la llamada en cuestión e ir a **Flow**. También se puede ir a **Player** y de ahí a **Decode** y escuchar ambos canales.
- Ir a **Telephony ---> RTP ---> Show All Streams**. Acá se pueden ver parámetros de pérdida de paquetes, jitter, etc. Del **Flow** se puede sacar el SSRC de una conversación, y buscar el reverso, para analizar lo mismo.

	ssh -p8765 freetechsol@192.168.10.126 sudo /usr/sbin/tcpdump -s0 -nn -U -w - udp port 5060 | wireshark -k -i -

---
###Chef paso a paso {#chef}

1. Instalar chef todas las máquinas no importa si es el server o el cliente: 
  
  `curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P chefdk -c stable -v 2.0.28`

2. Instalar el server en la máquina que va a oficiar de server, para eso ejecutar el script siguiente: 

```
##############################################################################################################
# create staging directories
if [ ! -d /tmp/chef-server/drop ]; then
          mkdir -p /tmp/chef-server/drop
  fi
  if [ ! -d /tmp/chef-server/downloads ]; then
            mkdir -p /tmp/chef-server/downloads
    fi

    # download the Chef server package
    if [ ! -f /downloads/chef-server-core_12.16.2_amd64.deb ]; then
              echo "Downloading the Chef server package..."
                wget -nv -P /tmp/chef-server/downloads https://packages.chef.io/files/stable/chef-server/12.16.2/ubuntu/16.04/chef-server-core_12.16.2-1_amd64.deb
        fi

        # install Chef server
        if [ ! $(which chef-server-ctl) ]; then
                  echo "Installing Chef server..."
                    dpkg -i /tmp/chef-server/downloads/chef-server-core_12.16.2-1_amd64.deb
                      chef-server-ctl reconfigure

                        echo "Waiting for services..."
                          until (curl -D - http://localhost:8000/_status) | grep "200 OK"; do sleep 15s; done
                            while (curl http://localhost:8000/_status) | grep "fail"; do sleep 15s; done

                              echo "Creating initial user and organization..."
                                chef-server-ctl user-create chefadmin Chef Admin soporte@freetechsolutions.com.ar 098098ZZZ --filename /tmp/chef-server/drop/chefadmin.pem
                                  chef-server-ctl org-create Freetech Solutions "Freetech Solutions" --association_user chefadmin --filename Freetech-validator.pem
                          fi

                          echo "Your Chef server is ready!"
#################################################################################################
```

Este script instala el servidor levanta la interfaz `localhost:8000`, crea el usuario chefadmin y crea dos certificados, uno para chefadmin y otro para Freetech.

---
###Concatenar audios {#uniraudios}

En [este posteo](http://es.ccm.net/faq/2654-unir-canciones-gratis-mp3-wav-wma#q=unir+canciones&cur=1&url=%2F "Post unir audios audacity") se encuentran las indicaciones para concatenar audios utilizando Audacity.

---
###Conexión por SSH sin autenticación {#sshsinauth}

A veces es necesario obviar el paso de introducir la contraseña cada vez que nos conectamos a otro equipo vía SSH, principalmente para aquellos casos en que necesitamos automatizar un proceso o tarea. Evitar introducir la contraseña es totalmente posible, donde para ciertos casos de automatización se puede utilizar el intérprete "expect", pero también se puede configurar el servicio SSH para que acepte llaves conocidas.
El concepto básico tras la autenticación SSH sin introducción de contraseñas consiste en crear llaves públicas y privadas con un usuario en un "Equipo A", donde la llave pública es la que se comparte con un usuario de otro "Equipo B" al cual nos queremos conectar, y la llave privada es la que queda como clave de autenticación. Cuando el usuario del "Equipo A" intente conectarse mediante el protocolo SSH con un usuario del "Equipo B", el primero le enviará su llave privada al segundo, y éste al recibirla evaluará si la llave coincide con su contrapartida llave pública recibida anteriormente (por otros canales o por el mismo), y si el cálculo matemático de comprobación entre las 2 llaves coincide, entonces se confirma la autenticación, mientras que en caso contrario la misma se rechaza.
A continuación, se muestran las configuraciones necesarias en ambos equipos para lograr una conexión por SSH sin autenticación, suponiendo que el usuario "admin" en el "Equipo A" quiere conectarse con el usuario "backup" en el "Equipo B":

**Configuraciones en el "Equipo A"**
1) El usuario "admin" debe generar sus llaves pública y privada, ejecutando el siguiente comando:

	admin@equipo_a:~> ssh-keygen -t dsa
	Generating public/private dsa key pair.
	Enter file in which to save the key (/home/admin/.ssh/id_dsa):
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /home/admin/.ssh/id_dsa.
	Your public key has been saved in /home/admin/.ssh/id_dsa.pub.
	The key fingerprint is:
	9c:8d:21:1d:a6:27:d9:d9:01:11:9f:55:b0:09:4c:ec admin@equipo_a.testnet.local
		The key's randomart image is:
		+--[ DSA 1024]----+
		|        *B+ oo.  |
		|       * =o= o   |
		|      = *.+ o    |
		|       = =E      |
		|        S .      |
		|                 |
		|                 |
		|                 |
		|                 |
		+-----------------+

Durante la ejecución del comando "ssh-keygen", éste solicitará introducir información adicional para generar las llaves, pero esto es alternativo por lo que se puede obviar presionando la tecla enter para avanzar. En el directorio oculto .ssh del home del usuario en cuestión se generarán 2 archivos conteniendo las 2 llaves, el archivo id_dsa que contendrá la llave privada y el archivo id_dsa.pub que contendrá la llave pública (la cual puede ser compartida). Esto se puede verificar ejecutando el siguiente comando:

	admin@equipo_a:~> ls -l ~/.ssh
	total 3
	-rw------- 1 admin users  672 sep  8 21:27 id_dsa
	-rw-r--r-- 1 admin users  618 sep  8 21:27 id_dsa.pub
	-rw-r--r-- 1 admin users 2312 ago 13 17:40 known_hosts

2) El usuario "admin" del "Equipo A" debe pasar su llave pública al usuario "backup" del "Equipo B". Esto se puede hacer de varias formas: copiarla en un pendrive, enviarla por mail, o a través del mismo protocolo SSH. Éste último método se muestra a continuación:

	admin@equipo_a:~> scp ~/.ssh/id_dsa.pub backup@equipo_b:/home/backup/id_dsa.pub.equipo_a
	Password:
	id_dsa.pub	100%	618	0.6KB/s	00:00

**Configuraciones en el "Equipo B"**
1) Verificar que el usuario "backup" cuente con la estructura necesaria en su directorio home para albergar las configuraciones requeridas. Para ello se debe verificar que el directorio .ssh exista en el directorio home, y si no existe aún se lo crea de la siguiente manera:

	backup@equipo_b:~> mkdir ~/.ssh

2) Copiar la llave pública del usuario "admin" contenida en el archivo id_dsa.pub.equipo_a al archivo authorized_keys del usuario "backup":

	backup@equipo_b:~> cat ~/id_dsa.pub.equipo_a >> ~/.ssh/authorized_keys

3) Aplicar los permisos correspondientes al directorio ~/.ssh y eliminar el archivo id_dsa.pub.equipo_a que ya no será necesario:

	backup@equipo_b:~> chmod 700 ~/.ssh
	backup@equipo_b:~> rm ~/id_dsa.pub.equipo_a

---
###Configurar zona horaria por consola {#zonahoraria}

	mv /etc/localtime /etc/localtime.bkp
	cd /etc
	ln -s /usr/share/zoneinfo/America/Argentina/Cordoba localtime

	[root@localhost etc]# ls -l localtime
	lrwxrwxrwx 1 root root 45 abr  6 11:25 localtime -> /usr/share/zoneinfo/America/Argentina/Cordoba

---
###Crear un USB booteable {#usbbooteable}

	dd if=Archivo.iso of=/dev/sdX

	/dev/sdX = Device donde montó el USB

---
###Eliminar GPT de un disco USB {#eliminargpt}

No matter what I do, I can't get rid of this error message that everything except parted displays when dealing with this disk:

	# sfdisk -l /dev/sdb

	WARNING: GPT (GUID Partition Table) detected on '/dev/sdb'! The util sfdisk doesn't support GPT. Use GNU Parted.


I have even tried `dd if=/dev/zero of=/dev/sdb bs=1` and letting it write 50 MB of data. I tried `bs=4096` and it wrote GB's of data.

I have tried doing this and rebooting in between, and I still see this (error?) message.

--
*darkod*
*May 4th, 2012, 08:02 PM*
That is actually not an error message, just a warning that those tools don't work with gpt disks (or don't work correctly which is worse).

If you want to create msdos table, simply do:
	
	sudo parted /dev/sdX
	mklabel msdos
	quit

That's it.

If you want to keep using it as gpt, start investigating gpt partitioning tools, like gdisk, cgdisk, etc.

---
###Función de Excel para dejar en segundos el reporte {#excelreporte}

=MID(I2;1;HALLAR("s";I2;1)-1)

---
###History con fecha y hora {#historydate}

	root@Fenix:~# export HISTTIMEFORMAT="%d/%m/%y %T "
	root@Fenix:~# history|grep 14:45
	2022  29/10/15 14:45:54 export HISTTIMEFORMAT="%d/%m/%y %T "
	2023  29/10/15 14:45:56 history 
	2024  29/10/15 14:46:09 history|grep 14:45

---
###Inicio de Ubuntu server en recovery mode {#ubuntuserver}

1. Se enciende el server y, cuando aparece el menú del grub, se selecciona la entrada de Ubuntu en "recovery mode" y se pulsa [ENTER]. De esta manera, se accede a un menú donde se puede seleccionar entre distintos tipos de arranque.
2. Se selecciona el tipo de arranque `root   Drop to root shell prompt`. Cuando pida el password de root, se introduce el mismo.
3. Se remonta el directorio raíz en modo lectura-escritura con el comando `mount -o remount,rw /`.
4. Se edita el `/etc/fstab` para remover la línea que no deja bootear.
5. Reboot.

---
###Setting Up An NFS Server And Client On CentOS 6.3 {#nfscentos}

Version 1.0
Author: Falko Timme
Last edited 12/11/2012

This guide explains how to set up an NFS server and an NFS client on CentOS 6.3. NFS stands for Network File System; through NFS, a client can access (read, write) a remote share on an NFS server as if it was on the local hard disk.

I do not issue any guarantee that this will work for you!

 
1) Preliminary Note

I'm using two CentOS systems here:

    NFS Server: server.example.com, IP address: 192.168.0.100
    NFS Client: client.example.com, IP address: 192.168.0.101

 
2) Installing NFS

**server:**

On the NFS server we run:

	yum install nfs-utils nfs-utils-lib

Then we create the system startup links for the NFS server and start it:

	chkconfig --levels 235 nfs on
	/etc/init.d/nfs start

**client:**

On the client we can install NFS as follows (this is actually the same as on the server):

	yum install nfs-utils nfs-utils-lib

 
3) Exporting Directories On The Server

**server:**

I'd like to make the directories /home and /var/nfs accessible to the client; therefore we must "export" them on the server.

When a client accesses an NFS share, this normally happens as the user nobody. Usually the /home directory isn't owned by nobody (and I don't recommend to change its ownership to nobody!), and because we want to read and write on /home, we tell NFS that accesses should be made as root (if our /home share was read-only, this wouldn't be necessary). The `/var/nfs` directory doesn't exist, so we can create it and change its ownership; in my tests the user and group nobody both had the ID 99 on both my CentOS test systems (server and client); when I tried to write to `/var/nfs` from the NFS client, I got a Permission denied error, so I did a `chmod 777 /var/nfs` so that everyone could write to that directory; writing to `/var/nfs` from the client worked then, and on the client the files written to `/var/nfs` appeared to be owned by the user and group nobody, but on the server they were owned by the (nonexistant) user and group with the ID 65534; so I changed ownership of `/var/nfs` to the user/group 65534 on the server and changed permissions of `/var/nfs` back to `755`, and voilà, the client was allowed to write to `/var/nfs`:

	mkdir /var/nfs
	chown 65534:65534 /var/nfs
	chmod 755 /var/nfs

Now we must modify `/etc/exports` where we "export" our NFS shares. We specify /home and `/var/nfs` as NFS shares and tell NFS to make accesses to `/home` as root. To learn more about `/etc/exports`, its format and available options, take a look at

	man 5 exports


	vi /etc/exports

	/home           192.168.0.101(rw,sync,no_root_squash,no_subtree_check)
	/var/nfs        192.168.0.101(rw,sync,no_subtree_check)

(The no_root_squash option makes that /home will be accessed as root.)

Whenever we modify /etc/exports, we must run

	exportfs -a

afterwards to make the changes effective.

 
4) Mounting The NFS Shares On The Client

**client:**

First we create the directories where we want to mount the NFS shares, e.g.:

	mkdir -p /mnt/nfs/home
	mkdir -p /mnt/nfs/var/nfs

Afterwards, we can mount them as follows:

	mount 192.168.0.100:/home /mnt/nfs/home
	mount 192.168.0.100:/var/nfs /mnt/nfs/var/nfs

You should now see the two NFS shares in the outputs of

	df -h

	[root@client ~]# df -h
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/mapper/vg_server2-LogVol00 9.7G  1.7G  7.5G  18% 
	/tmpfs                 499M     0  499M   0% /dev/shm
	/dev/sda1             504M   39M  440M   9% /boot
	192.168.0.100:/home   9.7G  1.7G  7.5G  19% 
	/mnt/nfs/home
	192.168.0.100:/var/nfs9.7G  1.7G  7.5G  19% 
	/mnt/nfs/var/nfs
	[root@client ~]#

and

	mount
```
[root@client ~]# mount
/dev/mapper/vg_server2-LogVol00 on / type ext4 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
tmpfs on /dev/shm type tmpfs (rw)
/dev/sda1 on /boot type ext4 (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
192.168.0.100:/home on /mnt/nfs/home type nfs (rw,vers=4,addr=192.168.0.100,clientaddr=192.168.0.101)
192.168.0.100:/var/nfs on /mnt/nfs/var/nfs type nfs (rw,vers=4,addr=192.168.0.100,clientaddr=192.168.0.101)
[root@client ~]#
```
 
5) Testing

On the client, you can now try to create test files on the NFS shares:

**client:**

	touch /mnt/nfs/home/test.txt
	touch /mnt/nfs/var/nfs/test.txt

Now go to the server and check if you can see both test files:

**server:**

	ls -l /home/

	[root@server ~]# ls -l /home/
	total 0
	-rw-r--r-- 1 root root 0 Dec 11 16:58 test.txt
	[root@server ~]#

	ls -l /var/nfs

	[root@server ~]# ls -l /var/nfs
	total 0
	-rw-r--r-- 1 nfsnobody nfsnobody 0 Dec 11 16:58 test.txt
	[root@server ~]#

(Please note the different ownerships of the test files: the `/home` NFS share gets accessed as root, therefore `/home/test.txt` is owned by root; the `/var/nfs` share gets accessed as `nobody/65534`, therefore `/var/nfs/test.txt` is owned by 65534.)

 
6) Mounting NFS Shares At Boot Time

Instead of mounting the NFS shares manually on the client, you could modify /etc/fstab so that the NFS shares get mounted automatically when the client boots.

**client:**

Open `/etc/fstab` and append the following lines:

	vi /etc/fstab
```
[...]
192.168.0.100:/home  /mnt/nfs/home   nfs      rw,sync,hard,intr  0     0
192.168.0.100:/var/nfs  /mnt/nfs/var/nfs   nfs      rw,sync,hard,intr  0     0
```

Instead of rw,sync,hard,intr you can use different mount options. To learn more about available options, take a look at

	man nfs

To test if your modified /etc/fstab is working, reboot the client:

	reboot

After the reboot, you should find the two NFS shares in the outputs of

	df -h
```
[root@client ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/vg_server2-LogVol00
                      9.7G  1.7G  7.5G  18% /
tmpfs                 499M     0  499M   0% /dev/shm
/dev/sda1             504M   39M  440M   9% /boot
192.168.0.100:/home   9.7G  1.7G  7.5G  19% /mnt/nfs/home
192.168.0.100:/var/nfs
                      9.7G  1.7G  7.5G  19% /mnt/nfs/var/nfs
[root@client ~]#
```

and

	mount
```
[root@client ~]# mount
/dev/mapper/vg_server2-LogVol00 on / type ext4 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
tmpfs on /dev/shm type tmpfs (rw)
/dev/sda1 on /boot type ext4 (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
192.168.0.100:/home on /mnt/nfs/home type nfs (rw,sync,hard,intr,vers=4,addr=192.168.0.100,clientaddr=192.168.0.101)
192.168.0.100:/var/nfs on /mnt/nfs/var/nfs type nfs (rw,sync,hard,intr,vers=4,addr=192.168.0.100,clientaddr=192.168.0.101)
[root@client ~]#
```

---
###IPtables {#iptables}

```
iptables -F
iptables -F -t nat
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -i eth0 -s 10.0.0.254/32 -p all -j DROP
iptables -A INPUT -i eth0 -s 10.0.0.0/24 -d 0/0 -j ACCEPT
#iptables -A INPUT -s 172.20.0.10/32 -d 0/0 -p udp --dport 4569 -j DROP
iptables -A INPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i tun0 -s 10.9.0.1/32 -d 10.9.0.34/32 -p tcp --dport 5666 -j ACCEPT
iptables -A INPUT -s 10.9.0.1/32 -d 0/0 -p icmp -j ACCEPT
iptables -A INPUT -p all -i eth0 -j DROP
```

---
###Irqbalance {#irqbalance}

	yum install irqbalance

	chkconfig --list|grep irqbalance
	chkconfig irqbalance on

	cat /proc/interrupts
	service irqbalance start
	cat /proc/interrupts

---
###Abrir el KVM y crear un host virtual {#kvmhost}

1. Ingresar por consola al KVM con las siguientes opciones:


	ssh -C -X root@192.168.xx.xxx

2. Una vez dentro del KVM, ejecutar virt-manager:


	[root@KVM ~]# virt-manager

3.  Una vez que se abra la interfaz gráfica, click en "Crear una máquina virtual nueva" y seguir el paso a paso

---
###Levantar un host virtual {#virtualhost}

- Crear el sitio en `/etc/apache2/sites-available`, por ejemplo `/etc/apache2/sites-available/plone`:

```
<VirtualHost *:80> 
	ProxyPreserveHost On
	ProxyRequests Off
	ServerName documentacion.freetechsolutions.com.ar
	ServerAlias documentacion.freetechsolutions.com.ar
	ProxyPass / http://192.168.99.241:8080/
	ProxyPassReverse / http://192.168.99.241:8080/
</VirtualHost>
```

- Habilitar el site:


	cd /etc/apache2/sites-enabled
	ln -s ../sites-available/plone

- Habilitar los módulos de proxy en apache:


	a2enmod proxy
	a2enmod proxy_http

- Restart de apache

---
###Memoria superior a 4G {#mem4g}

Para que el SO vea los 12G por ejemplo, se debe instalar el siguiente paquete:

`yum install kernel-PAE`

---
###Montado CIFS {#cifs}

En `/etc/fstab`:

`//192.168.0.2/Grabaciones	/var/spool/asterisk/monitor/ cifs	defaults,password=''	0 0`

`//192.168.0.205/Central 	/mnt/Backups   cifs rw,username=elusername,password=lapassword       0 0`

Luego montarla con `mount -a`

---
###Montar servidor FTP {#ftplinux}

Guía para instalar `vsftpd` en distribuciones basadas en Debian.

**Actualizar repositorios es instalar los paquetes para vsftpd**

`root@server:~#aptget update`
`root@server:~#aptget install vsftpd`

**Crear un grupo**

`root@server:~#groupadd ftp`

**Crear el home**

Cuando vsftpd termina de instalarse crea una carpeta en home, pero se pueden tener los usuarios en otras rutas. Una vez que se determine cuál será, se crea así:

`root@server:~#mkdir /home/ftp/username`

**Creación de shell FTP**

Se crea un shell virtual o fantasma con el objetivo de que los usuarios no puedan conectarse a una sesión del sistema operativo:

	root@server:~#mkdir /bin/ftp

	root@server:~#vim /etc/shells

Agregar al final del archivo `/etc/ftp`

	/bin/bash
	/bin/rbash
	/bin/ftp

**Crear usuarios FTP**

	root@server:~#useradd g ftp d /home/ftp/username s /bin/ftp username

Agregarle una pass:

	root@server:~#passwd username
	Enter new UNIX password:
	Retype new UNIX password:
	passwd: password updated successfully

Otorgar permisos al usuario:

	root@server:~#cd /home/ftp
	root@server:~#chown username.ftp username/ R

**Ficheros de configuración**

Configuración de vsftpd

Backup del archivo de configuración:

	root@server:~#cd /etc
	root@server:/etc# cp vsftpd.conf vsftpd.confori

Editar el archivo de configuración:

	root@server:/etc# vim vsftpd.conf

Cambiar los siguientes parámetros:

- Desactivar acceso a usuarios anónimos, para mayor seguridad.

	anonymous_enable=NO

- Permitir a los usuarios autenticados tener sus propias carpetas locales:

	local_enable=YES

- Permitir el modo de escritura en su carpeta:

	write_enable=YES

- Permitir enmascarar con algún permiso la subida de información:

	local_umask=037

(En este caso estamos indicándole al parámetro, va tener permisos de rwx para el usuario, r-- para el grupo y --- otro ningún permiso)

- Habilitamos a los usuarios dentro de su directorio personal y también para acceder a sus carpetas por FTP.

	chroot_local_user=YES
	chroot_list_enable=YES

Editar el siguiente archivo para listar los usuarios con acceso local y grupal al FTP:

	chroot_list_file=/etc/vsftpd.chroot_list

Al terminar de configurar el archivo:

	root@server:/etc# touch vsftpd.chroot_list
	root@server:/etc#echo “username”>> vsftpd.chroot_list

**Control del ancho de banda**

Se pueden agregar mas opciones al final del archivo como por ejemplo:

	anon_max_rate=5100 #Ancho de banda para usuario anónimo 5kb.
	local_max_rate=5100 #Ancho de banda por usuario local 5kb.
	max_clients=3 #Numero máximo clientes conectados.
	max_per_ip=2 #Numero máximo de conexiones por ip.

**Reinicio del servidor FTP**

	root@server:/etc# /etc/init.d/vsftpd restart
	• Stopping FTP server: vsftpd
	• Starting FTP server: vsftpd

**Comandos FTP**

COMANDOS DESCRIPCION

	cd [rutaRemota] //Cambia de directorio dentro del servidor remoto
	lcd [rutaLocal] //Cambia de directorio en el equipo local
	chgrp [grp] [rutaRemota] //Cambia el grupo de trabajo de un fichero remoto.
	chmod [rutaRemota] //Cambia los permisos de Lectura, Escritura o de Ejecución a un fichero remoto
	chown [own] [rutaRemota] //Cambia el grupo de trabajo de un fichero remoto.
	get [rutaRemota] [rutaLocal] //Copia un recurso remoto en un equipo local
	lmkdir [rutaLocal] //Crea una carpeta en el equipo local
	lpwd //Imprime la ruta local en la cual estamos trabajando
	mkdir [rutaRemota] //Crea una carpeta en el equipo remoto
	put [rutaLocal] [rutaRemota] //Sube un fichero o archivo desde una ruta local hasta una ruta remota
	pwd //Imprime la ruta remota en la cual estamos trabajando
	exit //Salimos de SFTP
	rename [rutaLocal] [rutaRemota] //Renombra un un fichero remoto
	rmdir [rutaRemota] //Borra una carpeta remota
	rm [rutaRemota] //Borra un fichero remoto
---
###Mover grabaciones en modo masivo en Centos {#mvgrabacentos}

En CentOS, al intentar mover muchas grabaciones con un solo comando, el comando `mv` suele arrojar el siguiente error:

	bash: /bin/mv: Argument list too long

Para solventarlo a la hora de mover grabaciones, se puede recurrir al siguiente comando:

	[root@Elastix ~]# for File in /var/spool/asterisk/monitor.bkp/*.wav;do mv $File /var/spool/asterisk/monitor/;done

En EMI hice lo siguiente:

	for File in /var/spool/asterisk/monitor/201601*.wav;do mv $File /mnt/monitor-temporal/;done;for File in /var/spool/asterisk/monitor/OUT*-201601*-*-*.*.wav;do mv $File /mnt/monitor-temporal/;done;for File in /var/spool/asterisk/monitor/g*-201601*-*-*.*.wav;do mv $File /mnt/monitor-temporal/;done;for File in /var/spool/asterisk/monitor/q*-201602*-*-*.*.wav;do mv $File /mnt/monitor-temporal/;done

---
###NRPE - Arreglar NRPE en Debian {#nrpe}

https://www.claudiokuenzler.com/blog/626/nrpe-debian-jessie-command-ags-arguments-not-working-error#.WO0Rc0e1vV0

---
###Ping con fecha {#pingfecha}

	ping www.google.com | while read pong; do echo "$(date): $pong"; done

---
###Rsyslog {#rsyslog}

```
--- /etc/rsyslog.conf	2016-03-07 11:40:44.320539619 -0300
+++ /etc/rsyslog.conf.dpkg-new	2015-09-01 03:56:48.000000000 -0300
@@ -5,25 +5,22 @@
 #
 #  Default logging rules can be found in /etc/rsyslog.d/50-default.conf
 
-#$template IpTemplate,"/var/log/%FROMHOST-IP%.log" 
-#*.*  ?IpTemplate 
-#& ~ 
 
 #################
 #### MODULES ####
 #################
 
-$ModLoad imuxsock # provides support for local system logging
-$ModLoad imklog   # provides kernel logging support
-#$ModLoad immark  # provides --MARK-- message capability
+module(load="imuxsock") # provides support for local system logging
+module(load="imklog")   # provides kernel logging support
+#module(load="immark")  # provides --MARK-- message capability
 
 # provides UDP syslog reception
-$ModLoad imudp
-$UDPServerRun 514
+#module(load="imudp")
+#input(type="imudp" port="514")
 
 # provides TCP syslog reception
-#$ModLoad imtcp
-#$InputTCPServerRun 514
+#module(load="imtcp")
+#input(type="imtcp" port="514")
 
 # Enable non-kernel facility klog messages
 $KLogPermitNonKernelFacility on
```
---
###Screen {#screen}

Para crear un screen con un nombre particular:

	[root@elx-pbx ~]# screen -S test
	[root@elx-pbx ~]#

Para salir del screen en el que me encuentro:

	<CTRL>+A, D

Listar screens:

	[root@elx-pbx ~]# screen -ls
	There are screens on:
	7632.test	(Detached)
	7318.pts-0.elx-pbx	(Detached)
	3 Sockets in /var/run/screen/S-root.

Killear un screen en particular:

- Entras al screen en particular y haces `<CTRL>+D`
- `[root@elx-pbx ~]# screen -S test -X quit`

Reatachar al screen:

	[root@elx-pbx ~]# screen -r 7318.pts-0.elx-pbx

---
###Sngrep {#sngrep}

Fuente: https://voztovoice.org/?q=node/804

700 .ssh
600 privada
644 publica

SNGREP es un programa que nos permite ver, de forma grafica, toda una serie de informaciones acerca del flujo de la señalización SIP; esto desde una ventana terminal de nuestro servidor Linux. Otra funcionalidad brindada, es la posibilidad de guardar/abrir archivos en formado PCAP.

Desarrollado por la empresa IRONTEC, su instalación en CentOS es bastante sencilla.

Se añaden los repositorios IRONTEC:

	nano /etc/yum.repos.d/irontec.repo

se pegan las siguientes líneas:

	[irontec]
	name=Irontec RPMs repository
	baseurl=http://packages.irontec.com/centos/$releasever/$basearch/

Se guardan los cambios y se añade la clave publica de los repositorios IRONTEC:

	rpm --import http://packages.irontec.com/public.key

Se instala el programa:

	yum install sngrep

Para ver los comandos disponibles:

	sngrep --help

Algunos ejemplos:

para abrir un archivo pcap:

	sngrep -I archivo.pcap

para guardar la captura en un archivo pcap:

	sngrep –O archivo.pcap

para activar la captura en una determinada interfaz de red (en lugar de la predefinida):

	sngrep -d eth1

---
###Habilitar syslog de equipos remotos en Centos 5 {#syslogremoto}

Agregar la opción `-r` en `/etc/sysconfig/syslog`:

	SYSLOGD_OPTIONS="-m 0 -r"

Reiniciar syslog:

	service syslog restart
---
###Tiempo máximo de ejecución en PHP {#phpmaxtime}

En Asternic, suele pasar que una query tarda más de 30 segundos en ejecutarse, razón por la cual la distribución se muestra de manera parcial.
En ese momento, se ve en el log lo siguiente:

	[Thu May 12 11:32:41 2016] [error] [client 192.168.0.81] PHP Fatal error:  Maximum execution time of 30 seconds exceeded in /var/www/html/stats/misc.php on line 0, referer: https://192.168.1.201/stats/index.php

Para solucionarlo, se debe editar el archivo `/etc/php.ini`, en la parte de límites de recursos:

```
;;;;;;;;;;;;;;;;;;;
; Resource Limits ;
;;;;;;;;;;;;;;;;;;;

max_execution_time = 30     ; Maximum execution time of each script, in seconds
max_input_time = 60     ; Maximum amount of time each script may spend parsing request data
memory_limit = 2048M      ; Maximum amount of memory a script may consume
```

Se debe aumentar el tiempo máximo de ejecución.

---
###Túneles SSH {#tunelssh}

Fuente: http://www.vilecha.com/hellguest/ssh_tuneles.asp

**Crear túneles con SSH**

OpenSSH nos permite crear dos clases de túneles: locales y remotos. En los locales se redirecciona un puerto de la máquina local (cliente) hacia un puerto en una máquina remota a la que el servidor tenga acceso. En los túneles remotos, lo que se hace es redireccionar un puerto desde una máquina remota a la que el servidor tenga acceso hacia un puerto de la máquina local.

**Túneles locales**

La forma de crear túneles locales con OpenSSH es mediante la opción **-L**, cuya sintaxis es:

	-L [dirección_escucha:]puerto_escucha:máquina_remota:puerto_máquina_remota

En caso de emplear direcciones IPv6, se puede utilizar la siguiente sintaxis:

	-L [dirección_escucha/]puerto_escucha/máquina_remota/puerto_máquina_remota

Los túneles locales se establecen de la siguiente forma: primero se crea un conector (socket) de escucha en la máquina local, asociado al puerto y, opcionalmente, a la dirección. Cuando se realice una conexión al puerto en el que está escuchando el conector, OpenSSH encauzará la conexión a través del canal seguro hacia la máquina remota a la que el servidor tenga acceso, indicada por la IP y el puerto.
Para demostrar como funcionan los túneles locales, me basaré en el siguiente diagrama ilustrativo:

![](/assets/tunelSSH.png)

En el diagrama tenemos que nos encontramos en el equipo llamado *alpha.local.net* y lo que queremos es acceder al servidor web (puerto 80) que hay en el equipo *web.remoto.net*. El problema es que entre nosotros y el servidor hay un router denominado *router.remoto.net* que nos impide acceder al servidor. Obviamente, el router ha de tener un servidor de SSH funcionando al que nosotros tengamos acceso. Cumpliéndose esta premisa, lo que vamos a hacer es redireccionar el puerto 80 (web) del servidor hacia, por ejemplo, el puerto 8080 de nuestro equipo:

	[hell@alpha.local.net] $ ssh-agent -L 8080:web.remoto.net:80 router.remoto.net
	Enter passphrase for key '/home/hell/.ssh/id_dsa':
	[hell@router.remoto.net] $

Hemos iniciado una sesión SSH interactiva en el router, pero además, ahora, en nuestro equipo, se habrá abierto el puerto 8080:

	[hell@alpha.local.net] $ netstat -an | grep LISTEN
	tcp   0   0  127.0.0.1.8080  *.*   LISTEN
	tcp   0   0  *.22            *.*   LISTEN
	[hell@alpha.local.net] $

El túnel está creado, ahora, mientras no cerremos la sesión SSH con el router, cada vez que nos conectemos al puerto 8080 de nuestro equipo (*localhost* o *alpha.local.net*), nuestra conexión estará siendo reenviada al puerto 80 del servidor *web.remoto.net*. Por ejemplo, para acceder a el desde un navegador, tendríamos que usar una URL del estilo: *http://localhost:8080/*
Hasta aquí, bien, pero... si nos fijamos, en la salida del comando que netstat que hay arriba, el túnel sólo acepta conexiones desde *alpha.local.net*, porque el conector (socket) que creo está asociado a la IP 127.0.0.1 (también conocida como *localhost*) que sirve para referirse al propio equipo, así es que, ¿qué pasaría si lo que buscásemos fuese que otro equipo se conecte al servidor *web.remoto.net* a través de nuestro equipo? Añadamos ese nuevo equipo a nuestro diagrama:

![](/assets/tunelSSH2.png)

Para permitir que beta.local.net también pueda beneficiarse de nuestro túnel, tenemos que hacer que el socket no se asocie con la IP 127.0.0.1, esto se consigue poniendo un asterisco (*), o, la IP del propio *alpha.local.net* en el parámetro opcional dirección_escucha, ejemplo:

	[hell@alpha.local.net] $ ssh-agent -L *:8080:web.remoto.net:80 router.remoto.net
	Enter passphrase for key '/home/hell/.ssh/id_dsa':
	[hell@router.remoto.net] $

Si ahora ejecutásemos otra vez el comando `netstat` en  *alpha.local.net* veríamos lo siguiente:

	[hell@alpha.local.net] $ netstat -an | grep LISTEN
	tcp   0   0  *.8080          *.*   LISTEN
	tcp   0   0  *.22            *.*   LISTEN
	[hell@alpha.local.net] $

El conector del túnel ahora acepta conexiones de cualquier equipo, perfectamente, ahora podríamos abrir un navegador en *beta.local.net* y poner la siguiente URL: *alpha.local.net:8080*, la conexión sería canalizada a través del canal seguro establecido entre *alpha.local.net* y *router.remoto.net*, hasta llegar al puerto 80 de *web.remoto.net*.

Túneles remotos

La forma de crear túneles remotos con OpenSSH es mediante la opción -R, que tiene la siguiente sintaxis:

	-R [dirección_escucha:]puerto_escucha:máquina_remota:puerto_máquina_remota

Pero si se quieren emplear direcciones IPv6, también se permite usar la siguiente sintaxis:

	-R [dirección_escucha/]puerto_escucha/máquina_remota/puerto_máquina_remota

El mecanismo empleado para establecer los túneles remotos es: se crea un conector (socket) en el servidor asociado al puerto indicado por puerto_escucha y, opcionalmente, a la dirección IP dirección_escucha. Posteriormente, cuando se realice una conexión a dicho conector, la conexión será encauzada a través del canal seguro hacia una máquina a la que el equipo local tenga acceso, indicada por la IP máquina_remota y el puerto puerto_máquina_remota.

A modo de ejemplo sobre como se establecen los túneles remotos, basemonos en los equipos del siguiente diagrama:

![](/assets/tunelSSH3.png)

Para ponernos en situación, imaginemos que nos encontramos en el equipo *alpha.local.net* y que lo que queremos es que alguien desde el equipo *terminal.remoto.com* se pueda conectar a un servidor web (puerto 80) que tenemos en nuestro equipo, el problema está en que el router deja que nosotros podamos conectarnos a *terminal.remoto.com*, pero impide que el pueda conectarse a nosotros. Entonces, creamos un túnel remoto de la siguiente forma:

	[hell@alpha.local.net] $ ssh-agent -R 8080:localhost:80 terminal.remoto.net
	Enter passphrase for key '/home/hell/.ssh/id_dsa':
	[hell@terminal.remoto.net] $

Ahora, se habrá creado un conector (socket) en terminal.remoto.net que estará escuchando en el puerto 8080:

	[hell@terminal.remoto.net] $ netstat -an | grept LISTEN
	tcp   0   0  127.0.0.1.8080  *.*   LISTEN
	tcp   0   0  *.22            *.*   LISTEN
	[hell@terminal.remoto.net] $

Cada vez que se establezca una conexión al puerto 8080 en *terminal.remoto.net*, esta conexión, será canalizada por el canal seguro hasta llegar al puerto 80 del equipo *alpha.local.net*, de esta forma, conseguimos que un equipo ajeno a nuestra red, pueda acceder a nuestro equipo. Pero, ¿qué pasaría si en lugar de a nuestro equipo lo que quisiéramos es que *terminal.remoto.com* pueda acceder a otro?, usemos el siguiente diagrama para orientarnos:

![](/assets/tunelSSH4.png)

De nuevo, nos encontramos en *alpha.local.net*, pero ahora, el servidor está en *web.local.net*, y lo que queremos es que *terminal.remoto.net* pueda acceder a el, pero el router sigue impidiendo que *terminal.remoto.net* se pueda conectar, y sin embargo, si que deja que nosotros desde *alpha.local.net* podamos conectarnos a *terminal.remoto.net*. Entonces, un comando como el siguiente sería suficiente:

	[hell@alpha.local.net] $ ssh-agent -R 8080:web.local.net:80 terminal.remoto.net
	Enter passphrase for key '/home/hell/.ssh/id_dsa':
	[hell@terminal.remoto.net] $

Esto indica que un extremo del túnel se corresponde con el puerto 8080 del equipo al que hemos conectado (*terminal.remoto.net*), y, el otro extremo, es el puerto 80 del equipo *web.local.net*, así de simple. Al establezcer una conexión en el puerto 8080 de *terminal.remoto.net*, la conexión viajará por el canal seguro establecido entre el cliente y el servidor de SSH, y después, se establecerá una nueva conexión entre *alpha.local.net* y *web.local.net* para que los datos puedan fluir hasta su destino.

---
###Vlan {#vlan}

```
[root@example ~]# cd /etc/sysconfig/network-scripts/
[root@example network-scripts]# cp ifcfg-eth0 ifcfg-eth0.21
[root@example network-scripts]# vim ifcfg-eth0.21 

# Realtek Semiconductor Co., Ltd. RTL8111/8168B PCI Express Gigabit Ethernet controller
DEVICE=eth0.21
BOOTPROTO=none
#BROADCAST=192.168.6.
#DHCPCLASS=
#HWADDR=90:2B:34:DD:DA:79
ONBOOT=yes
IPADDR=154.1.47.170
NETMASK=255.255.255.248
#NETWORK=192.168.0.0
VLAN=yes

[root@example ~]# yum install vconfig
[root@example ~]# vconfig add eth0 21
[root@example ~]# service network restart
[root@example ~]# ifup eth0.21
```
---
