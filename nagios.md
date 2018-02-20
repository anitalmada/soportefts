###Instalación en CentOS {#nagiosinstall}

Fuentes:

http://fraterneo.blogspot.com.ar/2013/03/instalando-y-configurando-nagios-en.html
https://www.digitalocean.com/community/articles/how-to-install-nagios-on-centos-6
http://docs.cslabs.clarkson.edu/wiki/Install_Nagios_on_CentOS_5

    yum install gd gd-devel httpd php gcc glibc glibc-common bash-completion wget nano

    wget http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm
    rpm -ivh epel-release-5-4.noarch.rpm


    [root@x015vm05 ~] # yum install nagios nagios-plugins-all nagios-plugins-nrpe nrpe php httpd

    [root@x015vm05 /etc/nagios] # htpasswd -c /etc/nagios/htpasswd.users nagiosadmin
    New password: 
    Re-type new password: 
    Adding password for user nagiosadmin
    [root@x015vm05 /etc/nagios] #

---
###Instalación de pnp4nagios en Centos {#php4nagios}

    yum install rrdtool perl-rrdtool php-gd

- Descargar el tar desde la página, la última versión estable
- Destarear y acceder a la carpeta


    ./configure
    make all
    make fullinstall
```
*** Configuration summary for pnp4nagios-0.6.21 03-24-2013 ***

  General Options:
  -------------------------         -------------------
  Nagios user/group:                nagios nagios
  Install directory:                /usr/local/pnp4nagios
  HTML Dir:                         /usr/local/pnp4nagios/share
  Config Dir:                       /usr/local/pnp4nagios/etc
  Location of rrdtool binary:       /usr/bin/rrdtool Version 1.2.27
  RRDs Perl Modules:                FOUND (Version 1.2027)
  RRD Files stored in:              /usr/local/pnp4nagios/var/perfdata
  process_perfdata.pl Logfile:      /usr/local/pnp4nagios/var/perfdata.log
  Perfdata files (NPCD) stored in:  /usr/local/pnp4nagios/var/spool

  Web Interface Options:
  -------------------------         -------------------
  HTML URL:                         http://localhost/pnp4nagios
  Apache Config File:               /etc/httpd/conf.d/pnp4nagios.conf



*** Main program, Scripts and HTML files installed ***

Enjoy.
```

Se configuró el Pnp4Nagios para que funcione en modo masivo con NPCD:

**Modo Masivo con NPCD**

La configuración es idéntica al modo masivo, excepto por el comanado usado. Se debe habilitar el procesado de datos de rendimiento en nagios.cfg

     process_performance_data=1

Además, otras directivas son necesarias

```
#
# service performance data
#
service_perfdata_file=/usr/local/pnp4nagios/var/service-perfdata
service_perfdata_file_template=DATATYPE::SERVICEPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tSERVICEDESC::$SERVICEDESC$\tSERVICEPERFDATA::$SERVICEPERFDATA$\tSERVICECHECKCOMMAND::$SERVICECHECKCOMMAND$\tHOSTSTATE::$HOSTSTATE$\tHOSTSTATETYPE::$HOSTSTATETYPE$\tSERVICESTATE::$SERVICESTATE$\tSERVICESTATETYPE::$SERVICESTATETYPE$
service_perfdata_file_mode=a
service_perfdata_file_processing_interval=15
service_perfdata_file_processing_command=process-service-perfdata-file

#
# host performance data starting with Nagios 3.0
# 
host_perfdata_file=/usr/local/pnp4nagios/var/host-perfdata
host_perfdata_file_template=DATATYPE::HOSTPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tHOSTPERFDATA::$HOSTPERFDATA$\tHOSTCHECKCOMMAND::$HOSTCHECKCOMMAND$\tHOSTSTATE::$HOSTSTATE$\tHOSTSTATETYPE::$HOSTSTATETYPE$
host_perfdata_file_mode=a
host_perfdata_file_processing_interval=15
host_perfdata_file_processing_command=process-host-perfdata-file
```

Atención: ¡Tenga en cuenta que la definición de esta plantilla puede diferir de las suministradas en nagios.cfg!

Las directivas y su significado:

    service_perfdata_file ruta al fichero temporal que debería alojar los datos de rendimiento.
    service_perfdata_file_template formato del fichero temporal. Los datos se definen usando macros de Nagios.
    service_perfdata_file_mode opción “a” especifica los datos que se van a añadir al fichero.
    service_perfdata_file_processing_interval el intervalo es 15 segundos
    service_perfdata_file_processing_command el comando que se invocará en el intervalo.

Los comandos que se utilizan deben ser configurados en Nagios. Si ha usado las guías de instalación rápida de Nagios, puede modificar las definiciones en commands.cfg.

    define command{
       command_name    process-service-perfdata-file
       command_line    /bin/mv /usr/local/pnp4nagios/var/service-perfdata /usr/local/pnp4nagios/var/spool/service-perfdata.$TIMET$
    }

    define command{
       command_name    process-host-perfdata-file
       command_line    /bin/mv /usr/local/pnp4nagios/var/host-perfdata /usr/local/pnp4nagios/var/spool/host-perfdata.$TIMET$
    }

Al usar estos comandos, el fichero service-perfdata, es movido a `var/spool/` depués del intervalo especificado en `service_perfdata_file_processing_interval`. La macro de Nagios `$TIMET$` se añade al nombre del fichero para evitar la sobreescritura accidental de ficheros anteriores. La macro `$TIMET$` contiene el timestamp actual en formaro `time_t` (segundos desde la época UNIX).

En el directorio `/usr/local/pnp4nagios/var/spool/` los ficheros son recuperados y procesados por NPCD.

NPCD monitoriza el directorio spool y le pasa los nombres de fichero a `process_perfdata.pl`. De esta forma el procesado de los datos de rendimiento está totalmente desacoplado de Nagios.

Antes de iniciar NPCD debe comprobar las rutas al directorio de spool y a `process_perfdata.pl` en el fichero de configuración `npcd.cfg`. Lo único que queda es iniciar NPCD.

     /usr/local/pnp4nagios/bin/npcd -d -f /usr/local/pnp4nagios/etc/npcd.cfg

La opción -d inicia NPCD como un demonio en segundo plano.

Restart de httpd
Restart de Nagios

Si todo anda bien, se debe poder acceder a http://IP/pnp4nagios

En mi caso no anduvo, porque en `/etc/httpd/conf.d/pnp4nagios.conf`, estaba mal referenciado el archivo de autenticación. Apuntaba a `/usr/local/nagios/etc/htpasswd.users`. Se lo cambió, restart de apache y arrancó bien.

Tuve que hacer tambien:

    chown -R root:apache /var/lib/php/session
---
###