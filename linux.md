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
