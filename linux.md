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


