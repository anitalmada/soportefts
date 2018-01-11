# Asterisk

Es un framework de código abierto orientado a desarrollar aplicaciones para comunicaciones, principalmente Voz sobre IP y administración de centrales PBX.

Es el núcleo detrás de FreePBX \(una distro/GUI para Asterisk\) y por lo tanto detrás de Elastix \(un FreePBX embebido\).

Para las configuraciones cotidianas es necesario manejar el funcionamiento de la Command Line Interface \(CLI\) de Asterisk. En este capítulo se listan algunas herramientas que solucionan problemas o configuraciones que son solicitadas por los clientes de forma recurrente.

---

**Contenidos del capítulo:**  
1. [Agregar CF por consola](#agregarcf)  
2. [Aumentar el tiempo de espera de la transferencia](#aumentartirmpo)  
3. [Borrar entradas de la base de datos de Asterisk](#removerentdb)  
4. Callback  
5. Deshabilitar CHOWN en amportal  
6. Dumpear variables de canal  
7. Entrar por telnet y loguearse  
8. Extensiones en DND  
9. Instalación de G729  
10. Logueo/deslogueo dinámico en colas  
11. Matar canal de agente colgado  
12. Ocultar caller-ID en llamadas internas  
13. Originar llamada vía AMI  
14. Probar llamadas por consola  
15. SIPP

---

### Agregar CF por consola {#agregarcf}

```
[root@elx-cliente ~]# asterisk -rx 'database show' |grep CF |grep 2999

[root@elx-cliente ~] # asterisk -rx 'database put CF 2999 123456789'

Updated database successfully

[root@elx-cliente ~] # asterisk -rx 'database show' |grep CF |grep 2999

CF 2999                                          : 123456789                

[root@elx-cliente ~] # asterisk -rx 'database del CF 2999'

Database entry removed.

[root@elx-cliente ~] # asterisk -rx 'database show' |grep CF\|grep 2999

[root@elx-rapela ~]#
```

---

### Aumentar el tiempo de espera por transferencia {#aumentartiempo}

1. Agregar las siguientes opciones en /etc/asterisk/features\_general\_custom.conf


```
atxfernoanswertimeout = 30     ; Timeout for answer on attended transfer default is 15 seconds.

transferdigittimeout = 10      ; Number of seconds to wait between digits when transferring a call
```


2. Reload de las features:


`Elastixi5\*CLI&gt; features reload`

### Borrar entradas de la base de datos de asterisk {#removerentdb}
```
[root@elx-cliente asterisk]# asterisk -rx 'database show'|grep CF
/CF/1224                                          : 1532507692

[root@elx-cliente asterisk]# asterisk -r
Asterisk 11.4.0, Copyright (C) 1999 - 2012 Digium, Inc. and others.
Created by Mark Spencer <markster@digium.com>
Asterisk comes with ABSOLUTELY NO WARRANTY; type 'core show warranty' for details.
This is free software, with components licensed under the GNU General Public
License version 2 and other licenses; you are welcome to redistribute it under
certain conditions. Type 'core show license' for details.
=========================================================================
Connected to Asterisk 11.4.0 currently running on elx-diagnosis (pid = 2773)
elx-cliente*CLI> database deltree CF
1 database entries removed.
```

### Callback {#callback}

En la cola de atención X, se configura un IVR breakout con un audio donde se le da la opción al llamante de dejar su número para ser contactado más tarde, ya que todos los operadores están ocupados.
El IVR breakout, tiene la opción 1 apuntando a la extensión 0204.
La extensión 0204 es una extensión custom que hace el Dial al siguiente string: "Local/s@custom-ivr-2015-callback".
En el archivo /etc/asterisk/extensions_custom.conf se hace un include del archivo definitivo donde irá el código:

```
#include extensions_cliente.conf

En el archivo incluido (en éste caso, /etc/asterisk/extensions_conci.conf), se coloca el código correspondiente al dial anterior:

; Manejo de CALL BACK
[custom-ivr-2015-callback]
exten => s,1,NoOp(***** Pedido de CallBack *****)
same => n,Read(CALLBACKNUM,custom/ivr-2014/conci_callback_201410_02&beep)
same => n,Playback(custom/ivr-2014/cliente_comodines_201410_01)
same => n,SayDigits(${CALLBACKNUM})
same => n,Read(CONFIRMACION-CB,custom/ivr-2014/cliente_comodines_201410_02)
same => n,Goto(custom-ivr-2015-callback,${CONFIRMACION-CB},1)

exten => 1,1,NoOp(Se confirmo el numero de CallBack)
same => n,Set(COLA=${CUT(BLKVM_OVERRIDE,/,2)})
same => n,NoOp(El pedido de CallBack proviene de la cola ${COLA})
same => n,ExecIf($["${COLA}" = "7001"]?Set(_COLACALLBACK=22))
same => n,ExecIf($["${COLA}" = "7007"]?Set(_COLACALLBACK=24))
same => n,ExecIf($["${COLA}" = "7009"]?Set(_COLACALLBACK=23))
same => n,NoOp(El ID de la campaña donde vamos a insertar los datos es ${COLACALLBACK})
same => n,AGI(agi_callback.php,${CALLBACKNUM},${COLACALLBACK})
same => n,Playback(custom/ivr-2014/cliente_comodines_201410_04A)
same => n,Hangup()

exten => 2,1,NoOp(Se solicito ingresar nuevamente el numero)
same => n,Goto(s,2)

exten => _X.,1,Playback(pbx-invalid)
same => n,Goto(custom-ivr-2015-callback,s,1)

exten => _X,1,Playback(pbx-invalid)
same => n,Goto(custom-ivr-2015-callback,s,1)
```

Por otro lado, se crea al agi que utiliza el código anterior (agi_callback.php):
```
[root@cliente-elx ~]# cat /var/lib/asterisk/agi-bin/agi_callback.php 
#!/usr/bin/php -q
<?php

set_time_limit(30);
error_reporting(E_ALL);

include_once("/var/www/html/freetech_ccenter/constantes.php");

require "phpagi.php";

$agi = new AGI();
ob_implicit_flush(true) ;

$numero=$argv[1];
$campaign=$argv[2];

$cnn = mysql_connect(IP_MySQL, USER_MySQL, PASS_MySQL) or die(mysql_error());

$SQL1="INSERT INTO call_center.calls (id_campaign,phone) VALUES ('$campaign','$numero')";
$SQL2="UPDATE call_center.campaign SET estatus='A' WHERE id='$campaign'";
$result1 = mysql_query($SQL1) or die(mysql_error()); 
$result2 = mysql_query($SQL2) or die(mysql_error()); 
mysql_close($cnn);
```

Finalmente, a nivel módulo de call center, se crean las campañas cuyos IDs están referenciados en el bloque de código anterior (22, 23, 24).
A modo de ejemplo, la campaña 22 tiene un rango de fechas con una fecha límite bien alta, en el rango horario que se desee contactar a los pacientes, con el formulario que se desee, con el troncal (Por Plan Marcado), apuntando a la cola que corresponda (una cola hecha puntualmente para CallBack) y con algún guión "Llamada de CallBack". Finalmente, se adjunta un csv con algún dato en la fila 2, columna 1, con algún número de teléfono de ejemplo.

El funcionamiento sería el siguiente:

* Si llama una persona y cae en la cola 7007 por ejemplo, y accede a dejar su número para ser contactado más tarde, el número de teléfono de la persona se guarda en la base de datos correspondiente a la campaña que corresponda, para ser contactado. En el horario definido en la campaña, y siempre y cuando haya un agente logueado en la cola 7107, se llama automáticamente al número y se lo asigna a un agente de la cola 7107.
Como todavía no hay agentes en dicha cola, las llamadas NO se disparan, pero quedan los números guardados.
