###Agregar rtptimeout en FreePBX {#rtptimeout}

1. Habilitar FreePBX desde Security -> Advanced settings.
2. Entrar al módulo de FreePBX sin embeber.
3. Hacer un backup de /etc/asterisk/sip_general_custom.conf y luego dejar vacío dicho archivo.
4. Ir a Tools ---> Asterisk SIP settings, y agregar al final regla => rtptimeout = 30.
5. Submitir los cambios y reload.

---

###Cambiar contraseña de administrador por web {#passweb}

Seguir los pasos descriptos en la  [wiki oficial de FreePBX que aparecen aquí](https://wiki.freepbx.org/pages/viewpage.action?pageId=37912685#fwconsolecommands\(13+\)-Unlock "Unlock fwconsole")

---
###Conversion tool {#conversiontool}

1) Registrar el freepbx en el portal de Sangoma usando la cuenta de soporte (solicitar datos al equipo). Se registra yendo a Administrator -> System Admin -> Activate

2) En el FreePBX se ejecuta el siguiente script:

    `curl -s https://convert.freepbx.org | bash`
    `Enter Conversion ID (leave blank if this is NEW): (dejar en blanco ya que es nueva instalacion)`
    Mostrará un DEPLOYMENT ID, copiarlo ya que este ID se tiene que poner en el Elastix


3) En el Elastix, aka DONOR, ejecutar el mismo comando: 

    `curl -s https://convert.freepbx.org | bash`

    `Enter Conversion ID (leave blank if this is NEW): (en esta parte ingresar el DEPLOYMENT ID copiado del freepbx)`

---
###Crear múltiples extensiones desde FreePBX {#bulkextensions}

Ir al FreePBX a Module Admin e instalar Bulk extensions. Para ello:

**Module Admin ---> Check for updates online ---> Bulk extensions**

Click en el nombre, y luego:

**Action ---> Download and install**

Finalmente:

**Process ---> Confirm**

Una vez instalado, ir a:

**Tools ---> Bulk extensions ---> Export extensions**

Editar el CSV: En la primer columna poner `add` y crear las extensiones necesarias.

**Tools ---> Bulk extensions ---> Examinar ---> Load file**

---
###Detección de placas por comando {#deteccionplacas}

Si tienen a sus ojos una PBX nueva (puede ser PBX ACT) y necesitan configurar placas DAHDI y no le funciona el DAHDI config, ejecutar este comando:

    setup-sangoma

Les despliega un menu de configuración de la o las placas.

---
###Fail2ban {#fail2ban}

En FreePBX viene instalado por defecto fail2ban por lo que solo hace falta configurarlo.

1) Configurar fail2ban para que entienda los ataques contra asterisk. Revisar el archivo `/etc/fail2ban/filter.d/asterisk.conf`: este archivo ya está creado por defecto por fail2ban y en el muestra los parámetros que va a mostrar en el log al haber intentos fallidos de registro, no editar, solo revisar que tenga contenido.
2) Añadir en el `/etc/fail2ban/jail.conf` lo siguiente:


    [asterisk-iptables]

    enabled  = true
    filter   = asterisk
    action   = iptables-allports[name=ASTERISK, protocol=all]
           sendmail-whois[name=ASTERISK, dest=root,         sender=fail2ban@example.org]
    logpath  = /var/log/asterisk/full
    maxretry = 5
    bantime = 259200

Aquí le añadimos parámetros de configuración de la jaula en la que va a entrar la IP que se banea. Se puede configurar la cantidad de intentos fallidos con `maxretry` y el tiempo de baneo está en segundos.

3) Cerciorarse que el timestamp del log de asterisk sea del formato `%F %T` y que incluya el notice en el `/var/log/asterisk/full`. Si no está asi fail2ban no escribirá en el log.

4) Iniciar IPtables y fail2ban. 
    service iptables start
    service fail2ban start

---
###Recargar FreePBX por consola {#descargabash}

    /var/lib/asterisk/bin/retrieve_conf

---
###Unable to generate MOTD on new install {#motd}

Esto me ocurrió instalando el FreePBX desde ISO, al instalar y loguearse aparece el mensaje "unable to generate MOTD on new install CRITICAL", si pasa eso es necesario primero hacer estas modificaciones en la BIOS:

*1) change boot order settings in BIOS and startup to 'Legacy' and not 'AHCI' mode.*
*2) change storage device (ie- hard drive) type from SATA RAID1 emulation to IDE in the bios.*

After those changes, FreePBX made a RAID1 installation via their software and was error free.

Luego volver a instalar.

Otra opción a tener en cuenta es la que desarrollan en [este thread](https://community.freepbx.org/t/critical-error-from-fresh-distro-install-on-multiple-systems/43496/5 "Critical Error MOTD") del foro de FreePBX.
---








