# Asterisk

Es un framework de código abierto orientado a desarrollar aplicaciones para comunicaciones, principalmente Voz sobre IP y administración de centrales PBX.

Es el núcleo detrás de FreePBX \(una distro/GUI para Asterisk\) y por lo tanto detrás de Elastix \(un FreePBX embebido\).

Para las configuraciones cotidianas es necesario manejar el funcionamiento de la Command Line Interface \(CLI\) de Asterisk. En este capítulo se listan algunas herramientas que solucionan problemas o configuraciones que son solicitadas por los clientes de forma recurrente.

---

**Contenidos del capítulo:**  
1. [Agregar CF por consola](#agregarcf)  
2. [Aumentar el tiempo de espera de la transferencia](#aumentartirmpo)  
3. [Borrar entradas de la base de datos de Asterisk](#removerentdb)  
4. [Callback](#callback)  
5. [Deshabilitar CHOWN en amportal](#chownamportal)   
6. [Dumpear variables de canal](#dumpvarcanal)  
7. [Entrar por telnet y loguearse](#telnet)  
8. [Extensiones en DND](#dnd)  
9. [ Instalación de G729](#g729)  
10. [Logueo/deslogueo dinámico en colas](#logincolas)  
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

1. Reload de las features:

`Elastixi5\*CLI&gt; features reload`

---

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

---

### Callback {#callback}

En la cola de atención X, se configura un IVR breakout con un audio donde se le da la opción al llamante de dejar su número para ser contactado más tarde, ya que todos los operadores están ocupados.  
El IVR breakout, tiene la opción 1 apuntando a la extensión 0204.  
La extensión 0204 es una extensión custom que hace el Dial al siguiente string: "Local/s@custom-ivr-2015-callback".  
En el archivo /etc/asterisk/extensions\_custom.conf se hace un include del archivo definitivo donde irá el código:

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

Por otro lado, se crea al agi que utiliza el código anterior \(agi\_callback.php\):

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

Finalmente, a nivel módulo de call center, se crean las campañas cuyos IDs están referenciados en el bloque de código anterior \(22, 23, 24\).  
A modo de ejemplo, la campaña 22 tiene un rango de fechas con una fecha límite bien alta, en el rango horario que se desee contactar a los pacientes, con el formulario que se desee, con el troncal \(Por Plan Marcado\), apuntando a la cola que corresponda \(una cola hecha puntualmente para CallBack\) y con algún guión "Llamada de CallBack". Finalmente, se adjunta un csv con algún dato en la fila 2, columna 1, con algún número de teléfono de ejemplo.

El funcionamiento sería el siguiente:

* Si llama una persona y cae en la cola 7007 por ejemplo, y accede a dejar su número para ser contactado más tarde, el número de teléfono de la persona se guarda en la base de datos correspondiente a la campaña que corresponda, para ser contactado. En el horario definido en la campaña, y siempre y cuando haya un agente logueado en la cola 7107, se llama automáticamente al número y se lo asigna a un agente de la cola 7107.
  Como todavía no hay agentes en dicha cola, las llamadas NO se disparan, pero quedan los números guardados.

---

### Deshabilitar CHOWN en amportal {#chownamportal}

Cambiar el archivo /var/lib/asterisk/bin/freepbx\_engine, líneas 218 y 219.

---

### Dumpear variables de canal {#dumpvarcanal}

```
; Manejo de CALL BACK
[custom-ivr-2015-callback]
exten => s,1,NoOp(***** Pedido de CallBack *****)
same => n,Read(CALLBACKNUM,custom/ivr-2014/cliente_callback_201410_02&beep)
same => n,Playback(custom/ivr-2014/cliente_comodines_201410_01)
same => n,SayDigits(${CALLBACKNUM})
same => n,Read(CONFIRMACION-CB,custom/ivr-2014/cliente_comodines_201410_02)
same => n,Goto(custom-ivr-2015-callback,${CONFIRMACION-CB},1)

exten => 1,1,NoOp(Se confirmo el numero de CallBack)
same => n,DumpChan() ; <-- DUMPEO DE VARIABLES PARA VER DÓNDE ESTÁ LA INFORMACIÓN QUE NECESITO!!!!
same => n,Set(COLA=${CUT(CALLFILENAME,-,2)})
same => n,NoOp(El pedido de CallBack proviene de la cola ${COLA})
same => n,ExecIf($["${COLA}" = "7001"]?Set(_COLACALLBACK=1))
same => n,ExecIf($["${COLA}" = "7007"]?Set(_COLACALLBACK=3))
same => n,ExecIf($["${COLA}" = "7009"]?Set(_COLACALLBACK=2))
same => n,ExecIf($["${COLA}" = "7999"]?Set(_COLACALLBACK=2))
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

En el log, se ve algo parecido a lo siguiente:

```
[2016-12-28 23:57:24] VERBOSE[32109][C-00001d29] pbx.c:     -- Executing [1@custom-ivr-2015-callback:2] DumpChan("Local/s@custom-ivr-2015-callback-00000d51;2", "") in n
ew stack
[2016-12-28 23:57:24] VERBOSE[32109][C-00001d29] app_dumpchan.c: 
Dumping Info For Channel: Local/s@custom-ivr-2015-callback-00000d51;2:
================================================================================
Info:
Name=               Local/s@custom-ivr-2015-callback-00000d51;2
Type=               Local
UniqueID=           1482980220.16404
LinkedID=           1482980204.16402
CallerIDNum=        199
CallerIDName=       FTS testing
ConnectedLineIDNum= 0204
ConnectedLineIDName=Sistema de CallBack
DNIDDigits=         (N/A)
RDNIS=              (N/A)
Parkinglot=         
Language=           en
State=              Up (6)
Rings=              0
NativeFormat=       (ulaw)
WriteFormat=        ulaw
ReadFormat=         ulaw
RawWriteFormat=     ulaw
RawReadFormat=      ulaw
WriteTranscode=     No 
ReadTranscode=      No 
1stFileDescriptor=  -1
Framesin=           1125 
Framesout=          1167 
TimetoHangup=       0
ElapsedTime=        0h0m24s
DirectBridge=       <none>
IndirectBridge=     <none>
Context=            custom-ivr-2015-callback
Extension=          1
Priority=           2
CallGroup=          
PickupGroup=        
Application=        DumpChan
Data=               (Empty)
Blocking_in=        (Not Blocking)

Variables:
READSTATUS=TIMEOUT
CONFIRMACION-CB=1
PLAYBACKSTATUS=SUCCESS
CALLBACKNUM=156375144
DIALEDPEERNUMBER=s@custom-ivr-2015-callback
KEEPCID=TRUE
CWIGNORE=
MON_FMT=wav
PICKUPMARK=0204
EXTTOCALL=0204
TTL=63
RINGTIMER=15
NODEST=
CALLFILENAME=q-7999-199-20161228-235644-1482980204.16402
FROMEXTEN=199
TIMESTR=20161228-235644
YEAR=2016
MONTH=12
DAY=28
REC_STATUS=INITIALIZED
REC_POLICY_MODE=dontcare
MOHCLASS=2017-MOH
DIAL_OPTIONS=trM(auto-blkvm)
BLKVM_CHANNEL=SIP/199-00001a29
================================================================================
[2016-12-28 23:57:24] VERBOSE[32109][C-00001d29] app_dumpchan.c: 
Dumping Info For Channel: Local/s@custom-ivr-2015-callback-00000d51;2:
================================================================================
Info:
Name=               Local/s@custom-ivr-2015-callback-00000d51;2
Type=               Local
UniqueID=           1482980220.16404
LinkedID=           1482980204.16402
CallerIDNum=        199
CallerIDName=       FTS testing
ConnectedLineIDNum= 0204
ConnectedLineIDName=Sistema de CallBack
DNIDDigits=         (N/A)
RDNIS=              (N/A)
Parkinglot=         
Language=           en
State=              Up (6)
Rings=              0
NativeFormat=       (ulaw)
WriteFormat=        ulaw
ReadFormat=         ulaw
RawWriteFormat=     ulaw
RawReadFormat=      ulaw
WriteTranscode=     No 
ReadTranscode=      No 
1stFileDescriptor=  -1
Framesin=           1125 
Framesout=          1167 
TimetoHangup=       0
ElapsedTime=        0h0m24s
DirectBridge=       <none>
IndirectBridge=     <none>
Context=            custom-ivr-2015-callback
Extension=          1
Priority=           2
CallGroup=          
PickupGroup=        
Application=        DumpChan
Data=               (Empty)
Blocking_in=        (Not Blocking)

Variables:
READSTATUS=TIMEOUT
CONFIRMACION-CB=1
PLAYBACKSTATUS=SUCCESS
CALLBACKNUM=156375144
DIALEDPEERNUMBER=s@custom-ivr-2015-callback
KEEPCID=TRUE
CWIGNORE=
MON_FMT=wav
PICKUPMARK=0204
EXTTOCALL=0204
TTL=63
RINGTIMER=15
NODEST=
CALLFILENAME=q-7999-199-20161228-235644-1482980204.16402
FROMEXTEN=199
TIMESTR=20161228-235644
YEAR=2016
MONTH=12
DAY=28
REC_STATUS=INITIALIZED
REC_POLICY_MODE=dontcare
MOHCLASS=2017-MOH
DIAL_OPTIONS=trM(auto-blkvm)
BLKVM_CHANNEL=SIP/199-00001a29
================================================================================
```

Gracias a ésta función, se determinó que el número de la cola estaba en la variable CALLFILENAME, y se procedió a hacer el CUT.

---

### Entrar por telnet y loguearse {#telnet}

```
telnet 192.168.0.20 5038
...
Asterisk Call Manager/1.0
Action: Login
UserName: *******
Secret: ********

Response: Success
Message: Authentication accepted
```

---

#### Extensiones en DND {#dnd}

```
asterisk -rx 'database show'|grep DND
```

---

### Instalación de G729 {#g729}

Ir a la siguiente página y seleccionar el binario que corresponda, según la versión de Asterisk y de sistema operativo:

[http://asterisk.hosting.lv/\#bin](http://asterisk.hosting.lv/#bin)

```
[root@nlxmu modules]# pwd
/usr/lib/asterisk/modules
[root@nlxmu modules]# wget "http://asterisk.hosting.lv/bin/codec_g729-ast110-gcc4-glibc-pentium.so"
[root@nlxmu modules]# chmod 755 codec_g729-ast110-gcc4-glibc-pentium.so
```

Renombrar el archivo

`mv codec_g729-ast110-gcc4-glibc-pentium.so codec_g729.so`

Cargar el módulo:

```
asterisk -r

nlxmu*CLI> module load codec_g729.so

nlxmu*CLI> core show translation
```

Ahí debe aparecer el nuevo codec.

---

### Logueo/deslogueo dinámico en colas {#logincolas}

En extensions\_custom.conf:

```
[from-internal-custom]
...
...
...
include => login-operadora
...
...
...
[login-operadora]
exten => *551,1,NoOp(Logueando interno: ${CALLERID(num)})
same => n,GotoIf($[$["${CALLERID(num)}" == "101"]|$["${CALLERID(num)}" == "113"]|$["${CALLERID(num)}" == "121"]|$["${CALLERID(num)}" == "125"]|$["${CALLERID(num)}" == "126"]|$["${CALLERID(num)}" == "136"]|$["${CALLERID(num)}" == "137"]|$["${CALLERID(num)}" == "139"]|$["${CALLERID(num)}" == "163"]|$["${CALLERID(num)}" == "182"]|$["${CALLERID(num)}" == "183"]|$["${CALLERID(num)}" == "184"]|$["${CALLERID(num)}" == "201"]|$["${CALLERID(num)}" == "226"]|$["${CALLERID(num)}" == "232"]|$["${CALLERID(num)}" == "267"]]?SetearPrioridad)
same => n,Playback(pbx-invalid)
same => n,Hangup()
same => n(SetearPrioridad),ExecIf($[$["${CALLERID(num)}" == "101"]|$["${CALLERID(num)}" == "201"]]?Set(Prioridad=1):Set(Prioridad=2))
same => n,Goto(LoginAgente)
same => n,Hangup()
same => n(LoginAgente),AddQueueMember(3002,Local/${CALLERID(num)}@from-internal,${Prioridad})
same => n,Playback(agent-loginok)
same => n,Hangup()

exten => *552,1,NoOp(Deslogueando interno: ${CALLERID(num)})
same => n,GotoIf($[$["${CALLERID(num)}" == "101"]|$["${CALLERID(num)}" == "113"]|$["${CALLERID(num)}" == "121"]|$["${CALLERID(num)}" == "125"]|$["${CALLERID(num)}" == "126"]|$["${CALLERID(num)}" == "136"]|$["${CALLERID(num)}" == "137"]|$["${CALLERID(num)}" == "139"]|$["${CALLERID(num)}" == "163"]|$["${CALLERID(num)}" == "182"]|$["${CALLERID(num)}" == "183"]|$["${CALLERID(num)}" == "184"]|$["${CALLERID(num)}" == "201"]|$["${CALLERID(num)}" == "226"]|$["${CALLERID(num)}" == "232"]|$["${CALLERID(num)}" == "267"]]?LogoffAgente)
same => n,Playback(pbx-invalid)
same => n,Hangup()
same => n(LogoffAgente),RemoveQueueMember(3002,Local/${CALLERID(num)}@from-internal)
same => n,Playback(agent-loggedoff)
same => n,Hangup()
```

---
### Matar canal de agente colgado {#matarcanal}

```
[root@cliente-elx eccp-examples]# asterisk -rx 'queue show 9129'
9129 has 0 calls (max 1) in 'ringall' strategy (2s holdtime, 32s talktime), W:10, C:77, A:4, SL:100.0% within 60s
   Members: 
      Agent/129 (ringinuse enabled) (Busy) has taken 67 calls (last was 2609 secs ago)
   No Callers

[root@cliente-elx eccp-examples]# asterisk -rx 'core show channels'|grep 129
Agent/129            (None)               Down    (None)
```

El comando request hangup no hace efecto alguno, por lo que procedemos con un redirect:

```
[root@cliente-elx eccp-examples]# asterisk -rx 'channel redirect Agent/129 from-internal,199,1'
Channel 'Agent/129' successfully redirected to from-internal,199,1
```

Atendí la llamada desde el 199 y corté.

```
[root@cliente-elx eccp-examples]# asterisk -rx 'core show channels'|grep 129
AsyncGoto/Agent/129< (None)               Down    (None)

[root@cliente-elx eccp-examples]# asterisk -rx 'queue show 9129'
9129 has 0 calls (max 1) in 'ringall' strategy (2s holdtime, 32s talktime), W:10, C:77, A:5, SL:100.0% within 60s
   Members: 
      Agent/129 (ringinuse enabled) (Unavailable) has taken no calls yet
   No Callers
```
---

### Ocultar CallerID en llamadas internas {#ocultarID}

Cargar la siguiente información en el archivo /etc/asterisk/sip_custom_post.conf:

```
[715](+)
setvar=CALLERID(all)="Numero privado"

```

De ésta manera, todas las llamadas del interno 715 a otros internos, les ringueará con ID desconocido.

---

### Originar llamadas vía AMI {#llamadasAMI}

```
Action: Originate
Channel: SIP/gsm-gateway/0351156375144
Exten: s
Context: from-pstn-custom
Priority: 1
Timeout: 30000
```
---

### Probar llamada por consola {#llamadaconsola}

Ingresar por ssh al server y en un asterisk cli ejecutar el siguiente comando:

`console dial INTERNO@CONTEXTO`

Por ejemplo:

`console dial 72140@from-internal`

---

### SIPP {#sipp}

`sipp -sn uac -d 200000 -s 1234 127.0.0.1 -l 200 -r 10  -trace_err -error_file sipperror`

1. Instalar primero `yum install sin`

2. Luego habilitar las llamadas SIP anónimas desde Elastix Security

3. Luego en extensions.conf modificar from-sip-external y comentar descomentar como sigue:

```
;exten => s,n,Playback(ss-noservice)
exten => s,n,Playback(demo-congrats)
;exten => s,n,Playtones(congestion)
;exten => s,n,Congestion(5)
```

`sipp –h |less`
```
               Handy switches of SIPp command line:
               -sn                                          : Scenario e.g uac, uas
               -sf                                           : Open customized scenario; usually XML file
               -d                                            : Duration of each call (in milliseconds)
               -s                                            : Extension to dial
               -l                                             : Call limit (Maximum concurrent calls)
               -r                                             : Calling rate (calls/sec)
               -trace_err                               : Enable error tracing
               -error_file  filename             : Dump error in specified file

cat /proc/`pidof asterisk`/limits
```












