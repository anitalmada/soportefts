###Crear usuario nuevo para gestión {#crearuser}

    [root@mysql-cliente ~]# su - ftsender

    [ftsender@mysql-cliente ~]$ /home/ftsender/deploy/bin/manage.sh create_ftsender_user
    Username (leave blank to use 'ftsender'): VentasPA02
    Email address: 
    Password: 
    Password (again): 
    User created successfully.

---
###Error en reportes de campaña sobre horario recibida {#bugreporte}

Se detectó un "bug" a la hora de querer ver los reportes SMS de mensaje recibido con respuesta que fue iniciado por un proceso que depuró las bases de contactos de determinada campaña a nivel de DB, por lo que es necesario entrar a la base y agregar los contactos manualmente.

1. En la URL ver el numero de la campaña: http://una.ip.determinada:puerto/campana_sms/1180/reportes_sms_recibido_repuesta/1/. El numero de campaña es 1118
2. Ingresar a la base ftsender
	- su postgres -
	- psql -U ftsender -W -h 127.0.0.1 ftsender

3. Ver a cuál base de datos pertenece a esa campaña
    
    `select * from fts_web_campanasms where id=1180;`

4. Ver si existen contactos de esa base de datos:

    `select * from fts_web_contacto where bd_contacto_id =1298 (este numero se saca de la columna bd_contacto_id del query anterior)`

5. ver si existe la base de datos (se puede hacer via web o sql):
    
    `select * from fts_web_basedatoscontacto where id=1298; (esta query deberia arrojar 0 resultados)`

6. Si no existen, ver la tabla que se genera por la creación de la campaña sms:
    
    `select * from fts_web_contacto_1180;`

7. Ingresar cada contacto manualmente con el comando:
    
    `insert into fts_web_contacto(id, datos, bd_contacto_id) values(ID, ' ["3516419329", "2264802", "4934755", "", "CORTES, RAUL ALFREDO", "", "Lunes 4 Diciembre 10:00", " ", " "]', bd_contacto_id);`

8. Esta query ya debería arrojar los contactos:

    `select * from fts_web_basedatoscontacto where id=1298;`

---
###Instalación de ICS {#install}

1) Instalar Centos 6 versión minimal, sin interfaz gráfica para evitar sobrecarga.
2)Pre-requisitos:

    service iptables stop
    chkconfig iptables off
    Deshabilitar selinux
    yum update

3)Primeros pasos

Instalar paquetes requeridos

    root@new-server $ yum install libselinux-python

Crear usuario ftsender

    root@new-server $ adduser ftsender

Configurar sudo para que el usuario ftsender pueda ejecutar cualquier comando sin que se le requiera el password:

    root@new-server $ visudo

Cuando aparezca el editor, agregar la linea:

    ftsender ALL=(ALL)       NOPASSWD: ALL

4)Segundos pasos

Configurar acceso ssh:

* Agregar el certificado de deployer a `~/.ssh/authorized_keys`, para que pueda iniciar sesión sin requerir password.

    ```
    ftsender@new-sever $ mkdir .ssh
    ftsender@new-sever $ chmod 0700 .ssh
    ftsender@new-sever $ vi ~/.ssh/authorized_keys
    ## AGREGAR el certificado publico de deployer
    ftsender@new-sever $ chmod 0600 ~/.ssh/authorized_keys
    ftsender@new-sever $ restorecon -R ~/.ssh
    ```

Para verificar que el usuario deployer puede acceder al nuevo servidor, ejecutar:

    deployer@ftsender-deployer $ ssh ftsender@192.168.99.222

5)Terceros pasos

Creacion de usuarios para acceder al sistema:

Para crear usuarios, es necesario loguearse en el servidor con el usuario ftsender y ejecutar `/home/ftsender/deploy/bin/manage.sh create_ftsender_user`:

    $ host> ssh ftsender@server-or-ip
    $ server> /home/ftsender/deploy/bin/manage.sh create_ftsender_user

6)Pasos finales

En el deployer, editar el inventario en `/home/deployer`. Copiar de cualquier otro cliente y editarlo.

    deployer@ftsender-deployer$ ./deploy.sh agrega_archivo_modem /home/deployer/"inventario"
    ENTER
    yes


Archivos a tener en cuenta:

    /var/log
    /root/.gammurc (archivo para envio)
    /etc/gammu-smsrc (archivo para recepcion)

---
### Evitar banneo por IMEI {#unbanIMEI}

**1) A nivel del GW GSM**

Se selecciona el menú "Mobile Configuration > IMEI".

![](/assets/imei1.png)

Se selecciona el checkbox _"I have read and I accept the agreement"_, y luego _"IMEI Auto Set"_.


Se selecciona la opción _"Enable IMEI Auto Set - Yes"_, y luego _"Save"_.

![](/assets/imei2.png)

Se deja únicamente la policy _"By time"_ con el valor 720, y luego _"Save"_.

![](/assets/imei3.png)

Con éste cambio, se está haciendo que el GW cambie el IMEI de todos sus puertos cada 12 hs.

**2) A nivel del ICS**



```
[root@mysql-cliente ~]# vim /home/ftsender/deploy/apidinstar/dinstar/dwgc.py

```


Se cambia el valor de la variable "check_timer" de 5 seg. a 40 seg.:

`check_timer = 5.0` ---> **check_timer = 40.0**

Se guardan los cambios y se reinicia el demonio:



```
[root@mysql-cliente ~]# /home/ftsender/deploy/apidinstar/dinstar/dwg.py stop
[root@mysql-cliente ~]# /home/ftsender/deploy/apidinstar/dinstar/dwg.py start
```



Con éste cambio, se está haciendo que los SMSs salgan cada 40 seg. en lugar de cada 5 seg. Es decir, se baja la tasa de envío.


