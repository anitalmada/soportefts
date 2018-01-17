## Asternic
Es un módulo de estadísticas orientado a las necesidades de los call centers. 

### Contenido
1. [Agregar salientes a Asternic](#aggsalientes)
2. [Call Center Asternic](#callcenter)
3. [Eliminar datos de qstats](#eliminarqstats)
4. [Enviar llamadas entrantes por líneas FXO a colas, por consola](#llamadafxocolas)
5. [Forma para que desaparezca Agent con la barra](#desapareceragent)
6. [Asternic PRO - Instalación](#asternicpro)
7. [Reinserción de datos en la base qstats](#reinserciondatos)




### Agregar salientes a Asternic {#aggsalientes}
1.- Agregar AccountCode a las extensiones donde se quiera registrar sus salientes en Asternic.
2.- Agregar el siguiente archivo en /etc/asterisk:

`extensions_custom_asternic_outbound_freepbx.conf`

```
; This macro will override dialout in freepbx and will track in Asternic outbound
; calls only if the accountcode is set for the Agent device. The Accountcode will
; be the queue name for the outobund call.

[macro-dialout-trunk-predial-hook]
exten => s,1,Noop(Test Track Outbound)
exten => s,n,Noop(Trunk is ${OUT_${DIAL_TRUNK}})
exten => s,n,Noop(Dialout number is ${OUTNUM})
exten => s,n,Noop(Dial options are ${DIAL_TRUNK_OPTIONS})
exten => s,n,GotoIf($["${CDR(accountcode)}" != ""]?bypass)
exten => s,n,Noop(NO ACCOUNTCODE, exit normally with no tracking outbound)
exten => s,n,MacroExit()
exten => s,n(bypass),Set(PREDIAL_HOOK_RET=BYPASS)
exten => s,n,Goto(queuedial,${OUTNUM},1)
exten => s,n,MacroExit()

;; Dialplan for storing OUTBOUND campaing in queue_log
;; Goto(queuedial,YYYXXXXXXXX,1) where YYY is the queue-campaign code
;; and XXXXXXXX is the number to dial.
;; The queuedial context has the outobound trunk hardcoded

[queuedial]
; this piece of dialplan is just a calling hook into the [qlog-queuedial] context that actually does the
; outbound dialing - replace as needed - just fill in the same variables.
exten => _X.,1,Set(QDIALER_QUEUE=${CDR(accountcode)})
;exten => _X.,n,Set(QDIALER_AGENT=Agent/${AMPUSER})
exten => _X.,n,Set(QDIALER_AGENT=${DB(AMPUSER/${AMPUSER}/cidname)})
exten => _X.,n,Set(QDIALER_CHANNEL=${OUT_${DIAL_TRUNK}}/${EXTEN})
exten => _X.,n,Set(QueueName=${QDIALER_QUEUE})
exten => _X.,n,MixMonitor(${QDIALER_QUEUE}-${UNIQUEID}.wav,b,/usr/local/parselog/update_mix_mixmonitor.pl ${UNIQUEID} ${QDIALER_QUEUE}-${UNIQUEID}.wav)
exten => _X.,n,Goto(qlog-queuedial,${EXTEN},1)

[qlog-queuedial]
; We use a global variable to pass values back from the answer-detect macro.
; STATUS = U unanswered
;        = A answered    (plus CAUSECOMPLETE=C when callee hung up)
; The 'g' dial parameter must be used in order to track callee disconnecting.
; Note that we'll be using the 'h' hook in any case to do the logging when channels go down.
;
exten => _X.,1,NoOp(Outbound call -> A:${QDIALER_AGENT} N:${EXTEN} Q:${QDIALER_QUEUE} Ch:${QDIALER_CHANNEL})
exten => _X.,n,Set(ST=${EPOCH})
;exten => _X.,n,Set(GM=${QDIALER_AGENT})
exten => _X.,n,Set(GM=${REPLACE(QDIALER_AGENT, ,_))
exten => _X.,n,Set(GLOBAL(${GM})=U)
exten => _X.,n,Set(GLOBAL(${GM}ans)=0)
exten => _X.,n,Macro(queuelog,${ST},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},ENTERQUEUE,-,${EXTEN})
exten => _X.,n,Dial(${QDIALER_CHANNEL},300,gM(queuedial-answer^${UNIQUEID}^${GM}^${QDIALER_QUEUE}^${QDIALER_AGENT}^${ST}))
exten => _X.,n,Set(CAUSECOMPLETE=${IF($["${DIALSTATUS}" = "ANSWER"]?C)})

; Trapping call termination here
exten => h,1,NoOp( "Call exiting: status ${GLOBAL(${GM})} answered at: ${GLOBAL(${GM}ans)} DS: ${DIALSTATUS}"  )
exten => h,n,Set(DB(LASTDIAL/${QDIALER_AGENT})=${EPOCH})
exten => h,n,Goto(case-${GLOBAL(${GM})})
exten => h,n,Hangup()

; Call unanswered
exten => h,n(case-U),Set(WT=$[${EPOCH} - ${ST}])
exten => h,n,Macro(queuelog,${EPOCH},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},ABANDON,1,1,${WT})
exten => h,n,Hangup()

; call answered: agent/callee hung
exten => h,n(case-A),Set(COMPLETE=${IF($["${CAUSECOMPLETE}" = "C"]?COMPLETECALLER:COMPLETEAGENT)})
exten => h,n,Set(WT=$[${GLOBAL(${GM}ans)} - ${ST}])
exten => h,n,Set(CT=$[${EPOCH} - ${GLOBAL(${GM}ans)}])
exten => h,n,Macro(queuelog,${EPOCH},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},${COMPLETE},${WT},${CT})
exten => h,n,Hangup()

[macro-queuedial-answer]
; Expecting $ARG1: uniqueid of the caller channel
;           $ARG2: global variable to store the answer results
;           $ARG3: queue name
;           $ARG4: agent name
;           $ARG5: enterqueue
;
exten => s,1,NoOp("Macro: queuedial-answer UID:${ARG1} GR:${ARG2} Q:${ARG3} A:${ARG4} E:${ARG5}")
exten => s,n,Set(NOW=${EPOCH})
exten => s,n,Set(WD=$[${NOW} - ${ARG5}])
exten => s,n,Macro(queuelog,${NOW},${ARG1},${ARG3},${ARG4},CONNECT,${WD})
exten => s,n,Set(GLOBAL(${ARG2})=A)
exten => s,n,Set(GLOBAL(${ARG2}ans)=${NOW})
exten => s,n,NoOp("Macro queuedial-answer terminating" )

[macro-queuelog]
; The advantage of using this macro is that you can choose whether to use the Shell version
; (where you have complete control of what gets written) or the Application version (where you
; do not need a shellout, so it's way faster).
;
; Expecting  $ARG1: Timestamp
;            $ARG2: Call-id
;            $ARG3: Queue
;            $ARG4: Agent
;            $ARG5: Verb
;            $ARG6: Param1
;            $ARG7: Param2
;            $ARG8: Param3
;
;exten => s,1,System( echo "${ARG1},${ARG2},${ARG3,${ARG4},${ARG5},${ARG6},${ARG7},${ARG8}" >> /var/log/asterisk/queue_log )
exten => s,1,QueueLog(${ARG3},${ARG2},${ARG4},${ARG5},${ARG6}|${ARG7}|${ARG8})
```

3.- Incluirlo a este archivo en extensions_custom.conf. Al final del archivo, agregar:

`#include extensions_custom_asternic_outbound_freepbx.conf`

---
### Call Center Asternic {#callcenter}

#### Call Center Asternic, con manejo de login/logoff desde SIP Phones

1 - Configurar FreePBX en modo USER/DEVICE
2 - Crear Dispositivos y Usuarios de acuerdo al relevamiento
3 - Agregar al final del archivo "extensions_custom.conf" las siguientes 2 lineas:

```
include => fts-pausas
#include extensions_custom_fts.conf
```

4 - Mover el archivo "extensions_custom_fts.conf" al directorio /etc/asterisk/

```
------------------------ archivo extensions_custom_fts.conf ------------------------ 
;  FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS
;  FTS FTS FTS FTS FTS FTS     LLAMADAS SALIENTES ASIGNADAS A AGENTES CON LOGIN    FTS FTS FTS FTS FTS FTS FTS FTS FTS
;  FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS

[macro-dialout-trunk-predial-hook]
exten => s,1,Noop(Test Track Outbound)
; Aqui se saca el numero de extension y el nombre del usuario
; Aprovechamos que el modo de trabajo DEVICE/USER de FreePBX setea varias cosas dentro de la familia DEVICE/$EXTENSION/
; Entre las cosas que setea es el nombre de la persona que hizo un loguin sobre la extension
exten => s,n,Set(REALCHAN=${CUT(CHANNEL,-,1)})
exten => s,n,Set(VIRTUAL=${DB(DEVICE/${REALCHAN:4}/user)})
; Si no tiene device definido en astdb, asumimos que no esta logueado asi que no llama un carajo !
exten => s,n,GotoIf($["${VIRTUAL}" = ""]?nosesigue)
exten => s,n,GotoIf($["${CDR(accountcode)}" != ""]?okbypass)
exten => s,n(nosesigue),Noop(NO HAY ACCOUNTCODE, salgo de aqui y sigo por el extensions_additional.conf)
exten => s,n,MacroExit()
exten => s,n(okbypass),Set(PREDIAL_HOOK_RET=BYPASS)
; preparo las variables de canal para avanzar sobre los tipicos dialplans de asternic_oubound_freepbx 
exten => s,n,Set(QDIALER_AGENT=${DB(AMPUSER/${VIRTUAL}/cidname)})
exten => s,n,Set(QDIALER_QUEUE=${CDR(accountcode)})
exten => s,n,Set(DIAL_PREFIX=${DB(dialprefix/${QDIALER_QUEUE})})
exten => s,n,Noop(Trunk is ${OUT_${DIAL_TRUNK}})
exten => s,n,Noop(Dialout number is ${OUTNUM})
exten => s,n,Noop(Dial options are ${DIAL_TRUNK_OPTIONS})
exten => s,n,GotoIf($["${QDIALER_AGENT}" != ""]?nextcheck)
exten => s,n,Noop(NO AGENT?, exit normally with no tracking outbound)
exten => s,n,MacroExit()
exten => s,n(nextcheck),GotoIf($["${QDIALER_QUEUE}" != ""]?bypass)
exten => s,n,Noop(NO QUEUE, exit normally with no tracking outbound)
exten => s,n,MacroExit()
exten => s,n(bypass),Set(PREDIAL_HOOK_RET=BYPASS)
exten => s,n,Goto(queuedial,${DIAL_PREFIX}${OUTNUM},1)
exten => s,n,MacroExit()

[queuedial]
; this piece of dialplan is just a calling hook into the [qlog-queuedial] context that actually does the
; outbound dialing - replace as needed - just fill in the same variables.
exten => _X.,1,Noop(QDIALER_QUEUE ${QDIALER_QUEUE})
exten => _X.,n,Noop(QDIALER_AGENT ${QDIALER_AGENT})
exten => _X.,n,Set(QDIALER_CHANNEL=${OUT_${DIAL_TRUNK}}/${EXTEN})
exten => _X.,n,Set(MIXAG=${REPLACE(QDIALER_AGENT, ,_)})
exten => _X.,n,Set(MIXQ=${REPLACE(QDIALER_QUEUE, ,_)})
exten => _X.,n,Set(MIXDEST=${EXTEN})
exten => _X.,n,MixMonitor(${MIXQ}-${MIXAG}-${MIXDEST}-${UNIQUEID}.wav,b,/usr/local/parselog/update_mix_mixmonitor.pl ${UNIQUEID} ${MIXQ}-${MIXAG}-${MIXDEST}-${UNIQUEID}.wav)
exten => _X.,n,Goto(qlog-queuedial,${EXTEN},1)

[qlog-queuedial]
; We use a global variable to pass values back from the answer-detect macro.
; STATUS = U unanswered
;        = A answered    (plus CAUSECOMPLETE=C when callee hung up)
; The 'g' dial parameter must be used in order to track callee disconnecting.
; Note that we'll be using the 'h' hook in any case to do the logging when channels go down.
;
exten => _X.,1,NoOp(Outbound call -> A:${QDIALER_AGENT} N:${EXTEN} Q:${QDIALER_QUEUE} Ch:${QDIALER_CHANNEL})
exten => _X.,n,Set(ST=${EPOCH})
exten => _X.,n,Set(GM=${REPLACE(QDIALER_AGENT, ,_)})
exten => _X.,n,Set(GLOBAL(${GM})=U)
exten => _X.,n,Set(GLOBAL(${GM}ans)=0)
exten => _X.,n,Macro(queuelog,${ST},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},ENTERQUEUE,-,${EXTEN})
exten => _X.,n,Dial(${QDIALER_CHANNEL},300,gM(queuedial-answer^${UNIQUEID}^${GM}^${QDIALER_QUEUE}^${QDIALER_AGENT}^${ST})${DIAL_TRUNK_OPTIONS})
exten => _X.,n,Set(CAUSECOMPLETE=${IF($["${DIALSTATUS}" = "ANSWER"]?C)})

; Trapping call termination here
exten => h,1,NoOp( "Call exiting: status ${GLOBAL(${GM})} answered at: ${GLOBAL(${GM}ans)} DS: ${DIALSTATUS}"  )
exten => h,n,Set(DB(LASTDIAL/${QDIALER_AGENT})=${EPOCH})
exten => h,n,Goto(case-${GLOBAL(${GM})})
exten => h,n,Hangup()

; Call unanswered
exten => h,n(case-U),Set(WT=$[${EPOCH} - ${ST}])
exten => h,n,Macro(queuelog,${EPOCH},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},ABANDON,1,1,${WT})
exten => h,n,Hangup()

; call answered: agent/callee hung
exten => h,n(case-A),Set(COMPLETE=${IF($["${CAUSECOMPLETE}" = "C"]?COMPLETECALLER:COMPLETEAGENT)})
exten => h,n,Set(WT=$[${GLOBAL(${GM}ans)} - ${ST}])
exten => h,n,Set(CT=$[${EPOCH} - ${GLOBAL(${GM}ans)}])
exten => h,n,Macro(queuelog,${EPOCH},${UNIQUEID},${QDIALER_QUEUE},${QDIALER_AGENT},${COMPLETE},${WT},${CT})

; fire userevent, find real channel when Local
exten => h,n,GotoIf($["${CHANNEL:0:5}" != "Local"]?notlocal)
exten => h,n(islocal),Set(CHAN=${CUT(CHANNEL,@,1)})
exten => h,n,Set(CHAN=${CUT(CHAN,/,2)})
exten => h,n,Set(VIRTUAL=${CHAN})
exten => h,n,Goto(doevent)
exten => h,n(notlocal),Set(REALCHAN=${CUT(CHANNEL,-,1)})
exten => h,n,Set(VIRTUAL=${DB(hotdeskmap/${REALCHAN}})
exten => h,n(doevent),UserEvent(POPUPAGENTCOMPLETE,Channel: VIRTUAL/${VIRTUAL},Value: ${UNIQUEID},Family: POPUPAGENTCOMPLETE)
exten => h,n(doevent),UserEvent(POPUPAGENTCOMPLETE,Channel: CUSTOM/${VIRTUAL},Value: ${UNIQUEID},Family: POPUPAGENTCOMPLETE)

exten => h,n,Hangup()

[macro-queuedial-answer]
; Expecting $ARG1: uniqueid of the caller channel
;           $ARG2: global variable to store the answer results
;           $ARG3: queue name
;           $ARG4: agent name
;           $ARG5: enterqueue
;
exten => s,1,NoOp("Macro: queuedial-answer UID:${ARG1} GR:${ARG2} Q:${ARG3} A:${ARG4} E:${ARG5}")
exten => s,n,Set(NOW=${EPOCH})
exten => s,n,Set(WD=$[${NOW} - ${ARG5}])
exten => s,n,Macro(queuelog,${NOW},${ARG1},${ARG3},${ARG4},CONNECT,${WD})
exten => s,n,Set(GLOBAL(${ARG2})=A)
exten => s,n,Set(GLOBAL(${ARG2}ans)=${NOW})
exten => s,n,NoOp("Macro queuedial-answer terminating" )

[macro-queuelog]
; The advantage of using this macro is that you can choose whether to use the Shell version
; (where you have complete control of what gets written) or the Application version (where you
; do not need a shellout, so it's way faster).
;
; Expecting  $ARG1: Timestamp
;            $ARG2: Call-id
;            $ARG3: Queue
;            $ARG4: Agent
;            $ARG5: Verb
;            $ARG6: Param1
;            $ARG7: Param2
;            $ARG8: Param3
;
;exten => s,1,System( echo "${ARG1},${ARG2},${ARG3,${ARG4},${ARG5},${ARG6},${ARG7},${ARG8}" >> /var/log/asterisk/queue_log )
exten => s,1,QueueLog(${ARG3},${ARG2},${ARG4},${ARG5},${ARG6}|${ARG7}|${ARG8})


;  FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS
;  FTS FTS FTS FTS FTS FTS     MANEJO DE LOGIN/LOGOFF Y PAUSAS     FTS FTS FTS FTS FTS FTS FTS FTS FTS
;  FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS FTS
[fts-pausas]
; Login de agentes
exten => *100,1,NoOp(login de agente FTS)
same => n,Set(DEVICETYPE=${DB(DEVICE/${CALLERID(number)}/type)})
same => n,Answer()
same => n,Wait(1)
same => n,GotoIf($["${DEVICETYPE}" = "fixed"]?s-FIXED,1)
; get user's extension
same => n,Set(AMPUSER=${ARG1})
same => n,GotoIf($["${AMPUSER}" != ""]?gotpass)
same => n(playagain),Read(AMPUSER,please-enter-your-extension-then-press-pound,,,4)
; get user's password and authenticate
same => n,GotoIf($["${AMPUSER}" = ""]?s-MAXATTEMPTS,1)
same => n(gotpass),GotoIf($["${DB_EXISTS(AMPUSER/${AMPUSER}/password)}" = "0"]?s-NOUSER,1)
same => n,Set(AMPUSERPASS=${DB_RESULT})
same => n,GotoIf($[${LEN(${AMPUSERPASS})} = 0]?s-NOPASSWORD,1)
; do not continue if the user has already logged onto this device
same => n,Set(DEVICEUSER=${DB(DEVICE/${CALLERID(number)}/user)})
same => n,GotoIf($["${DEVICEUSER}" = "${AMPUSER}"]?s-ALREADYLOGGEDON,1)
same => n,Authenticate(${AMPUSERPASS})
same => n,AGI(user_login_out.agi,login,${CALLERID(number)},${AMPUSER})
same => n,Goto(agente-colas,s,1)

exten => s-FIXED,1,NoOp(Device is FIXED and cannot be logged into)
exten => s-FIXED,n,Playback(ha/phone)
exten => s-FIXED,n,SayDigits(${CALLERID(number)})
exten => s-FIXED,n,Playback(is-curntly-unavail&vm-goodbye)
exten => s-FIXED,n,Hangup ;TODO should play msg indicated device cannot be logged into 

exten => s-ALREADYLOGGEDON,1,NoOp(This device has already been logged into by this user)
exten => s-ALREADYLOGGEDON,n,Playback(vm-goodbye)
exten => s-ALREADYLOGGEDON,n,Hangup ;TODO should play msg indicated device is already logged into 

exten => s-NOPASSWORD,1,NoOp(This extension does not exist or no password is set)
exten => s-NOPASSWORD,n,Playback(pbx-invalid)
exten => s-NOPASSWORD,n,Goto(s,playagain)

exten => s-MAXATTEMPTS,1,NoOp(Too many login attempts)
exten => s-MAXATTEMPTS,n,Playback(vm-goodbye)
exten => s-MAXATTEMPTS,n,Hangup

exten => s-NOUSER,1,NoOp(Invalid extension ${AMPUSER} entered)
exten => s-NOUSER,n,Playback(pbx-invalid)
exten => s-NOUSER,n,Goto(s,playagain)

; Logoff de agentes
exten => *111,1,NoOp(logoff de usuario FTS)
same => n,Set(__TIPOACCION=LOGOFF)
same => n,Goto(agente-colas,s,1)
same => n(logoffdevice),Set(DEVICEFTS=${CUT(CHANNEL,-,1)})
same => n,Set(DEVICETYPE=${DB(DEVICE/${DEVICEFTS:4}/type)})
same => n,GotoIf($["${DEVICETYPE}" = "fixed"]?s-FIXED,1)
same => n,AGI(user_login_out.agi,logout,${DEVICEFTS:4})

exten => s-FIXED,1,NoOp(Device is FIXED and cannot be logged out of)
same => n,Playback(an-error-has-occured&vm-goodbye)
same => n,Hangup() 

; Pausa de agentes
exten => _*10X,1,Answer
same => n,Set(REASON1=Break)
same => n,Set(REASON2=Capacitacion)
same => n,Set(REASON3=Sanitario)
same => n,Set(REASON4=Gestion)
same => n,Set(REALCHAN=${CUT(CHANNEL,-,1)})
same => n,Set(AGENT=${DB(DEVICE/${REALCHAN:4}/user)})
same => n,Set(REASONNUMBER=${EXTEN:-1})
same => n,Set(REASONTEXT=${REASON${REASONNUMBER}})
same => n,Noop(Agente ${AGENT} Number ${REASONNUMBER} Text ${REASONTEXT})
same => n,PauseQueueMember(,Local/${AGENT}@from-queue/n,,${REASONTEXT})
;same => n,Set(DB(DND/${AMPUSER})=YES)
same => n,Noop(Pause Status ${PQMSTATUS})
same => n,GotoIf($["${PQMSTATUS}" = "PAUSED"]?log)
same => n,Hangup()
same => n(log),QueueLog(NONE,${UNIQUEID},Local/${AGENT}@from-queue/n,PAUSE,${REASONTEXT})
same => n,Set(DB(PAUSECUSTOM/AGENT/${AGENT})=${REASONTEXT}:${EPOCH})
same => n,Set(DB(fop2state/SIP/${REALCHAN:4})=${REASONTEXT})
same => n,UserEvent(FOP2ASTDB,Family: fop2state,Channel: SIP/${REALCHAN:4},Value: ${REASONTEXT})
same => n,GotoIf($["${REASONNUMBER}" == "6"]?salta)
same => n,Playback(do-not-disturb&activated)
same => n(salta),Hangup

; Salir de pausa
exten => _*666,1,Answer()
same => n,Set(REALCHAN=${CUT(CHANNEL,-,1)})
same => n,Set(AGENT=${DB(DEVICE/${REALCHAN:4}/user)})
same => n,UnPauseQueueMember(,Local/${AGENT}@from-queue/n)
same => n,NoOp(${DB_DELETE(PAUSECUSTOM/AGENT/${AGENT})})
same => n,NoOp(${DB_DELETE(fop2state/SIP/${RALCHAN:4})})
same => n,UserEvent(FOP2ASTDB,Family: fop2state,Channel: SIP/${RALCHAN:4},Value: )
;same => n,Macro(user-callerid,)
;same => n,Noop(Deleting: DND/${AMPUSER} ${DB_DELETE(DND/${AMPUSER})})
same => n(hook_1),Playback(do-not-disturb&de-activated)
same => n,Hangup()

; Se usa para login y logoff de agentes a las colas configuradas en FreePBX 
[agente-colas]
exten => s,1,NoOp(Ahora sigo con el login de gestor a colas)
same => n(start),NoOp(este contexto esta en el override porque lo personalizo FTS)
same => n,Macro(user-callerid,)
same => n,AGI(queue_devstate.agi,getall,${AMPUSER})
same => n,GotoIf($["${QUEUESTAT}" = "NOQUEUES"]?skip)
same => n,Set(TOGGLE_MACRO=${IF($["${QUEUESTAT}"="LOGGEDOUT"]?toggle-add-agent:toggle-del-agent)})
same => n,Set(LOOPCNTALL=${FIELDQTY(USERQUEUES,-)})
same => n,Set(ITERALL=1)
same => n(begin),Set(QUEUENO=${CUT(USERQUEUES,-,${ITERALL})})
same => n,Set(ITERALL=$[${ITERALL}+1])
same => n,Macro(${TOGGLE_MACRO},)
same => n,GotoIf($[${ITERALL} <= ${LOOPCNTALL}]?begin)
same => n(skip),ExecIf($["${QUEUESTAT}"="LOGGEDIN" | "${QUEUESTAT}"="NOQUEUES"]?Playback(agent-loggedoff))
same => n,ExecIf($["${QUEUESTAT}"="LOGGEDOUT"]?Playback(agent-loginok))
same => n,ExecIf($["${QUEUESTAT}"="LOGGEDOUT"]?SayDigits(${AMPUSER}))
same => n,ExecIf($["${TIPOACCION}"="LOGOFF"]?Goto(fts-pausas,*111,logoffdevice))
same => n,Macro(hangupcall,)
```
---
### Eliminar datos de qstats {#eliminarqstats}
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema | 
| asterisk           | 
| asteriskcdrdb      | 
| meetme             | 
| mya2billing        | 
| mysql              | 
| qstats             | 
| roundcubedb        | 
| test               | 
| vtigercrm521       | 
+--------------------+
10 rows in set (0.00 sec)

mysql> use qstats;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------+
| Tables_in_qstats |
+------------------+
| acl              | 
| designer         | 
| language         | 
| qagent           | 
| qevent           | 
| qname            | 
| queue_stats      | 
| recordings       | 
| sched            | 
| secure_level     | 
| setup            | 
| sqlrealtime      | 
| user_filter      | 
| userqagent       | 
| userqname        | 
| users            | 
+------------------+
16 rows in set (0.00 sec)

mysql> delete from queue_stats where datetime between '2012-12-01' and '1012-12-30';
mysql> delete from queue_stats where datetime between '2012-11-01' and '1012-11-30';
mysql> delete from queue_stats where datetime between '2012-10-01' and '1012-10-30';
...

mysql > delete from qagent where agent_id='10';
```
---
### Enviar llamadas entrantes por líneas FXO a colas, por consola {#llamadafxocolas}

Para enviar llamadas entrantes por líneas FXO a colas, y que las mismas sean tratadas por Asternic, se deben seguir los siguientes pasos:

1. Asignar un determinado contexto a los puertos, en /etc/asterisk/dahdi-channels.conf:

```
; Span 1: WCTDM/0 "Wildcard AEX410" (MASTER) 
;;; line="1 WCTDM/0/0 FXSKS"
signalling=fxs_ks
callerid=asreceived
group=0
context=from-harris
channel => 1-4
```

2. Configurar el contexto, en /etc/asterisk/extensions_custom.conf:

```
[from-harris]
exten => s,1,Goto(ext-queues,9001,1)
```

###### Nota:
En un principio habíamos utilizado la siguiente configuración:

```
[from-harris]
exten => s,1,Queue(9001)
```

Sin embargo, de esa forma la llamada no es tratada por Asternic, de forma tal que no se encuentra el audio en .mp3 ni tampoco se lista en el CDR.

---
### Forma para que desaparezca Agent con la barra {#desapareceragent}

Ir a Asternic, Configuración ---> Preferencias.
Agregar registros por agente, de la siguiente forma:

keyword: dict_agent
parameter: Agent/101
value: nick-del-agente

---
### Asternic PRO - Instalación {#asternicpro}

1.Descargar la última versión de Asternic PRO y descomprimir:

```
wget http://download.asternic.net/asternic-stats-pro-2.0.7.tgz
tar -xvfz asternic-stats-pro-2.0.7.tgz
```

Para saber cuál es la última versión, directamente voy probando qué tgz logro descargar. Actualmente, la 2.0.7 es la última.

2.Crear y cargar base de datos:

```
mysqladmin -u root -p create qstats
cd asternic-stats-pro-2.0.7
mysql -uroot -p qstats < ./sql/mysql-tables.sql
GRANT ALL PRIVILEGES ON qstats.* TO 'asternic'@'localhost' IDENTIFIED BY '4st3rNIC123' WITH GRANT OPTION;
```


3.Configurar conexión a base de datos y manager de Asterisk:

- Editar /html/config.php, y poner user y password MySQL, y user y password manager asterisk.

```
// Connection details
$DBHOST = ‘localhost’;
$DBNAME = ‘qstats’;
$DBUSER = ‘root’;
$DBPASS = ‘tupassword’;

// Manager details (for realtime tab)
$MANAGER_HOST   = “localhost”;
$MANAGER_USER   = “admin”;
$MANAGER_SECRET = “tupassword”;
```

`mv html /var/www/html/stats` 
`chown asterisk.asterisk /var/www/html/stats`
`chown asterisk.asterisk /var/www/html/stats/* -R`
`mv parselog /usr/local`

4.Ioncube

- Descargar el paquete que corresponda:
```
wget http://downloads2.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz (para 64 bits)
wget http://downloads2.ioncube.com/loader_downloads/ioncube_loaders_lin_x86.tar.gz (para 32 bits)
```
`tar -xzvf ioncube_loaders_lin_x86-64.tar.gz`
`mv ioncube /usr/local`
`echo "zend_extension=/usr/local/ioncube/ioncube_loader_lin_5.1.so" > /etc/php.d/ioncube.ini`
`service httpd restart`

##### Nota
Acá me pasó que al intentar acceder a Asternic por web, me tiraba un error que el .so seleccionado no era el apropiado para la versión instalada. De acuerdo al error que tiraba, indicaba que la versión apropiada era la X. Entonces reemplacé la misma en /etc/php.d/ioncube.ini.

5.Activar la licencia

- Agregar la siguiente línea al /etc/rc.local:

`cd /usr/local/parselog ; ./tailqueuelogFreePBX28 -u asternic -p 4st3rNIC123 -d qstats -l /var/log/asterisk/queue_log &`

- Aplicar el patch, de acuerdo a la versión de FreePBX:

`cd /var/www/html/admin/modules/queues`
`patch < /usr/src/asternic-stats-pro-2.0.7/FreePBX/fpbxmonitor.diff (version 2.8 de FreePBX)`

- Editar el archivo /usr/local/parselog/update_mix_mixmonitor.pl y cambiar el usuario y password de MySQL, y el path de grabaciones y mp3:

```
# CONFIGURATION
# You have to set the proper database credentials
$config{'dbhost'} = 'localhost';
$config{'dbname'} = 'qstats';
$config{'dbuser'} = 'asternic';
$config{'dbpass'} = '4st3rNIC123';

# Destination directory for recordings
$config{'asterisk_spool'}  = "/var/spool/asterisk/monitor";
$config{'destination_dir'} = "/var/spool/asterisk/asternic";
```


A continuación, el mismo instructivo pero desde el README:

**REQUIREMENTS**

Minimum software versions:

PHP version 5.1 or higher
MYSQL version 5.0 or higher

**INSTALLING**

1) Untar de file (you already done that). It is best to extract
the file OUTSIDE your web root.

2) Populate the database and tables. It will use the database
name "qstats". If you want to change that, modify mysql-tables.sql accordingly. If you do not have the database created, do a:

  `#> mysqladmin -u root -p create qstats`

3) Then populate the tables:

   `#> mysql -uroot -p qstats < ./sql/mysql-tables.sql`

4) Edit /html/config.php to set your mysql db user and password, and also the Asterisk Manager credentials as set in /etc/asterisk/manager.conf. 
For using scheduled events, you will also have to set the rest user and password.
The later can be any user in the Asternic system that has access to the reports you want to schedule.

5) Copy or move /html to a suitable place on your web root

   `#> mv html /var/www/html/stats`

6) To activate the software, the destination directory must be owned by the same user apache is running as. So you might need to issue the appropiate chown command, for example, in FreePBX based distributions you might need to do:

   `#> chown asterisk.asterisk /var/www/html/stats`
   `#> chown asterisk.asterisk /var/www/html/stats/* -R`

Another option is to create an empty license file and give 666 permissions to it. For example:

   `#> touch /var/www/html/stats/asterniclic.txt`
   `#> chmod 666 /var/www/html/stats/asterniclic.txt` 


7) Finally you might want to copy or move /parselog to a nice place,
   for example inside /usr/local

   `#> mv parselog /usr/local`


**IONCUBE LOADER**

Asternic Stats Pro requires the free ioncube loaders for php. You can get them from : http://www.ioncube.com/loaders.php. Be sure to download the correct tarball for your platform. 

Installation is quite simple: extract the downloaded ioncube tarball, it will create an ioncube directory with all the loaders for different php versions, move that directory to its permanent location (we recommend to install them in /usr/local with the command:

 `#> wget http://downloads2.ioncube.com/loader_downloads/ioncube_loaders_lin_x86.tar.gz`
 `#> tar zxvf ioncube_loaders_lin_x86.tar.gz`
 `#> mv ioncube /usr/local`

In a centos system you can write a special ini file to load the extension with this command:

 `#> echo "zend_extension=/usr/local/ioncube/ioncube_loader_lin_5.1.so" >/etc/php.d/ioncube.ini`

If you do not use Centos, you can modify the php.ini file, (but please do not do it if you preformed the above actions or you will have problems with duplicate loading of the library, which causes segfaults and strange errors. 
So, if you did not create the ioncube.ini file as told above, you can add the following line to php.ini, (usually located in the /etc directory):

```
  !! ONLY ADD THIS IF YOU DID NOT RUN THE PREVIOUS COMMAND !!
  zend_extension=/usr/local/ioncube/ioncube_loader_lin_5.1.so
```


Where 5.1 is your PHP version. You will have to select the correct file depending on your PHP version, that you can find out by running in the command line the command "php -v"


**ACTIVATING THE PRO LICENSE**

After the installation, you need to activate the software. The license is tied to the MAC address of your server network interface. You will also need internet access from the server in order to activate the copy.

If you do not have internet connection, contact us to do an offline activation.

Once you load the web page for the first time, it will prompt you for the activation code (You have received that code via email), and a Licensee Name. The Licensee Name can be anything you want, it will be displayed on the footer of every report.

Once you enter both fields, you can press the "Generate Request" button.
When you do that it will contact our registration servers, validate your code, send the license request and retrieve the final license. If all goes well, you will be prompted with a login box. The default admin user is "admin" with password "admin".

If you get an error, you will have to contact us. If your server crashes or if you need to move licenses around, you will also have to contact us. 

The license is stored in the asterniclic.php file on the same directory as the reports themselves. Be sure to back the directory up just in case.

**PARSING LOGS**

Requirements: perl, perl-Time-HiRes

In /parselog there is a script: ./tailqueuelog

It accepts several parameters:

       -u <name>     - Connect to mysql as username <name> [root]
       -p <pw>       - Connect to mysql with password <pw>
       -h <host>     - Connect to mysql host <host> [localhost]
       -d <dbname>   - Connect to mysql database <dbname> [qstats]
       -l <queuelog> - Path and name for queue_log [/var/log/asterisk/queue_log]
       -w            - Write configuration to disk
       --purge       - Purge all data from tables

You can run it manually as a background job, it will read events in realtime from the queue_log file and feed it into mysql. There is no need for cron jobs. You can start it from rc.local. Eg:

`/usr/local/parselog/tailqueuelog -u root -p password -d qstats -l /var/log/asterisk/queue_log &`

If you have GNU screen installed, it is a good idea to run the process inside it, so you can attach to the console to see information about the process, you can do it with the following command:

`screen -dmS tailqueuelog /usr/local/parselog/tailqueuelog -u root -p password -d qstats -l /var/log/asterisk/queue_log`

You can put the above command inside the rc.local file so the daemon starts at boot time. In both cases you will have to use the full path that might change if you do not move the tailqueuelog directory into /usr/local


**RECORDINGS**

In parselog there is a script to convert and organize queues recordings.
You have to configure the DB access details and recording directory at the top of the file. It uses some tools that might not be installed in your linux distro (sox, lame), so you might need to tweak it a little bit. You also need to add some tidbits in the dialplan before launching the Queue command, like:

```
extensions.conf

exten => s,1,Answer
exten => s,n,Set(__MONITOR_FILENAME=/var/spool/asterisk/monitor/q${EXTEN}-${STRFTIME(${EPOCH},,%Y%m%d-%H%M%S)}-${UNIQUEID})
exten => s,n,Set(__MONITOR_EXEC=/usr/local/parselog/update_mix_mixmonitor.pl ^{UNIQUEID} ^{MIXMONITOR_FILENAME})
exten => s,n,Queue(myqueue)
```

This will instruct asterisk to call the script with some parameters after a call is recorded. The script task is to update the mysql tables to relate filenames with call ids, and optionally convert them to mp3. 
(for mp3 convertion to work, recording format must be set to wav and lame must be installed)

You queue configuration should be set to record calls and use mixmonitor for it:

```
queues.conf

[myqueue]
eventmemberstatus=no
eventwhencalled=yes
monitor-type=mixmonitor
monitor-format=wav
..
```

**RECORDINGS AND FreePBX**

If you use FreePBX, you will find a freepbx patch and a custom dialplan in the FreePBX directory of this tarball.

There are different patches depending on your FreePBX version:

For version 2.8 or older: fpbxmonitor.diff
For version 2.9: fpbxmonitor29.diff
For version 2.10: fpbxmonitor210.diff
For version 2.11: no need to patch. You need to configure FreePBX manually


You will need to apply that patch to FreePBX in order to have the queue recordings integrated in the reports. You can do so with the following commands (change the path to the directory where you extracted the tarball):

For version 2.8:

`cd /var/www/html/admin/modules/queues`
`patch < /path/to/FreePBX/fpbxmonitor.diff`

For version 2.9:

`cd /var/www/html/admin/modules/queues`
`patch < /path/to/FreePBX/fpbxmonitor29.diff`

For version 2.10:

`cd /var/www/html/admin/modules/core`
`patch < /path/to/FreePBX/fpbxmonitor210.diff`

For version 2.11:

You will need to open SETTINGS - Advanced Settings and under the 
option DEVELOPER AND CUSTOMIZATION you need to set the field 
POST CALL RECORDING SCRIPT to:

`/usr/local/parselog/update_mix_mixmonitor.pl ^{UNIQUEID} ^{MIXMONITOR_FILENAME}`

If you do not see that option, you will need to enable the options to allow modification of read only settings at the top of Advanced Settings.

After the patch is applied or the configuration is made, you will have to make a change on FreePBX web UI and apply changes so the dialplan is regenerated. The above patch will call the following script:

`/usr/local/parselog/update_mix_mixmonitor.pl`

Be sure to update the script and tweak it to your needs. Basically, be sure to change the following parameters if needed:

`$config{'asterisk_spool'} = "/var/spool/asterisk/monitor";`
`$config{'destination_dir'} = "/var/spool/asterisk/asternic";`

The later is the directory where you want to store mp3 recordings. The script will archive recordings in subdirectories using the YYYYMMDD scheme, inside it.
You will have to create that directory if it does not exists, and you must be sure the owner of the directory is the "asterisk" user.

You can also convert scripts to mp3 within the script, in that case be sure you have:

`$config{'convertmp3'} = true;`

mp3 convertion requires the utility "lame" to be installed in /usr/bin. If you do not have the utility or if its installed in some other directory, modify the script accordingly. Also, be sure the recording format in FreePBX queue config is set to "wav" (not wav49 or gsm or any other format).

You can also set convertmp3 to false and the original recording format as set in FreePBX queue config will be preserved. Asternic 2.0 supports audio streaming for .wav and .gsm files.


**OUTBOUND CALL TRACKING AND FreePBX**

The other file in the FreePBX directory is a dialplan to use with asterisk to track outbound calls. The included one will only track calls if you set the account code to something before performing the dial (it could be added to the extension configuration for example). The filename is:

`extensions_custom_asternic_outbound_freepbx.conf`

You have to include that file from your dialplan, you can do so by adding "#include xtensions_custom_asternic_outbound_freepbx.conf" at the end
of /etc/asterisk/extensions_custom.conf


**OUTBOUND CALL TRACKING (no FreePBX)**

In the docs directory you have the README.outbound with a sample dialplan.
You will need to tweak it heavily in order to suit your needs. The example uses a 3 digit prefix and fixed trunk.

**MAINTENANCE**

If you want to clear/purge the tables run tailqueuelog with
the purge parameter:

` ./tailqueuelog -u root --purge`

Attention! It will remove all queue activity from the logs so you can start afresh.


**ACCESING STATS**

Point your browser to the new url. There are two default users:

admin, password admin
user, password user

You should login as admin first and set user permissions.

_IMPORTANT!!!!_
_You should go to USER ACCESS tab and at least select ALL QUEUES 
and ALL AGENTS for the admin user. After you have some data in the tables you can refine the user access as you wish. But the default install DOES NOT permit viewing ANY queues or agents, so you will be stuck in the HOME page until you set them up._

Agent and Queue data are not displayed until you start populating mysql with the parselogscripts.  


**UPDATING THE queue_stats TABLE FROM ASTERNIC 1.6 OR PREVIOUS**

Transfers now are logged with a forth parameter in queue\_log, you have to alter the queue_stats table and then update records in order to store information correctly:

```
ALTER TABLE queue_stats ADD info4 varchar(40) default '';

UPDATE queue_stats SET info4=substring_index(info3,'|',-1),info3=substring_index(info3,'|',1) WHERE info3 like '%|%';
```

**UPDATING THE users TABLE FOR ASTERNIC 1.9 AND ENCRYPTED PASSWORDS**

```
ALTER TABLE users CHANGE password password varchar(100);

UPDATE users SET password=sha1(password);
```

**UPDATING THE queue_stats TABLE FROM ASTERNIC TO VERSION 1.9.2**

```
ALTER TABLE queue_stats ADD info5 varchar(40) default '';

UPDATE queue_stats SET info5=substring_index(info4,'|',-1),info4=substring_index(info4,'|',1) WHERE info4 like '%|%';
```

**UPDATING THE queue_stats TABLE FROM ASTERNIC TO VERSION 1.9.3**

```
ALTER TABLE setup CHANGE value value varchar(255);
```


**UPGRADE TO 2.0**

```
CREATE TABLE IF NOT EXISTS `userqname` (
  `users_id` int(6) default NULL,
  `qname_queue_id` int(6) default NULL
) DEFAULT CHARSET=utf8;

CREATE TABLE IF NOT EXISTS `userqagent` (
  `users_id` int(6) default NULL,
  `qagent_agent_id` int(6) default NULL
) DEFAULT CHARSET=utf8;

CREATE TABLE IF NOT EXISTS `sqlrealtime` (
  `id` int(6) NOT NULL,
  `lastupdate` timestamp NOT NULL default CURRENT_TIMESTAMP on update CURRENT_TIMESTAMP,
  `data` longtext,
  PRIMARY KEY  (`id`)
) DEFAULT CHARSET=utf8;

CREATE TABLE IF NOT EXISTS `designer` (
  `id` int(11) NOT NULL auto_increment,
  `keyword` varchar(50) default NULL,
  `parameter` varchar(50) default NULL,
  `value` varchar(255) default NULL,
  PRIMARY KEY  (`id`),
  UNIQUE KEY `keypar` (`keyword`,`parameter`)
) AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE IF NOT EXISTS `language` (
  `id` int(6) NOT NULL auto_increment,
  `iso_code` varchar(5) NOT NULL default 'en',
  `string` text,
  `translation` text character set utf8 collate utf8_bin,
  `pending` tinyint(1) default '1',
  PRIMARY KEY  (`id`),
  KEY `stindex` (`string`(200)),
  KEY `idx` (`iso_code`,`string`(200))
) AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE IF NOT EXISTS `sched` (
  `schid` int(6) NOT NULL auto_increment,
  `destinos` text,
  `reportes` text,
  `ultimos` tinyint(4) default NULL,
  `campo` varchar(40) default '',
  `valor` varchar(40) default '',
  `crondiames` varchar(40) default '*',
  `crondiasem` varchar(40) default '*',
  `cronhora` varchar(20) default '*',
  `cronminu` varchar(20) default '*',
  `lastrun` datetime default '0000-00-00 00:00:00',
  `activo` tinyint(4) default '1',
  PRIMARY KEY  (`schid`)
) AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

REPLACE INTO qagent VALUES (-1,'ALL');

REPLACE INTO qname  VALUES (-1,'ALL');

INSERT INTO userqagent SELECT users.id,agent_id FROM user_filter INNER JOIN users ON user_filter.user=users.login INNER JOIN qagent ON qagent.agent=value WHERE param='agent';

INSERT INTO userqname SELECT users.id,queue_id FROM user_filter INNER JOIN users ON user_filter.user=users.login INNER JOIN qname ON qname.queue=value WHERE param='queue';

ALTER TABLE qname CHANGE qname_id queue_id int(6);

ALTER TABLE user_filter DROP id;

UPDATE setup SET keyword='avg_time_format' WHERE keyword='avg_duration_format';
UPDATE setup SET keyword='realtime_custom_pauses' WHERE keyword='custom_pauses';
UPDATE setup SET keyword='realtime_group_queues' WHERE keyword='group_queues';
UPDATE setup SET keyword='realtime_spy_device' WHERE keyword='spychannel';
UPDATE setup SET keyword='short_abandon_threshold' WHERE keyword='minimum_abandon_duration';
UPDATE setup SET keyword=concat('realtime_',keyword) WHERE keyword like 'alarm_%';

// Find out if queue_stats unico index has the uniqueid set or not
SHOW INDEX FROM queue_stats WHERE Key_name='unico' AND Column_name='uniqueid';
ALTER TABLE queue_stats DROP INDEX unico;
ALTER TABLE queue_stats ADD unique unico (`queue_stats_id`,`uniqueid`,`qagent`,`qevent`,`qname`);

Import data from sql/lang.sql
```

**UPGRADE TO 2.0.6**

`ALTER TABLE sched add queues varchar(200) default NULL;`


**How to Import queue Names for FreePBX in one query:**

insert into setup (keyword,parameter,value) select 'dict_queue',extension,descr from asterisk.queues_config;

---
### Reinserción de datos en la base qstats {#reinserciondatos}

El tailqueuelog no va a tratar de insertar registros anteriores al más reciente registrado (por motivos de eficiencia). Ahora bien, eso impide que se reparseen los eventos viejos que hayan podido perderse por un crash o similar.

Una opción es comentar la linea de tailqueuelog que realiza dicho chequeo por fecha (las versiones más recientes de asternic tienen un proceso de parseo que se llama asterniclog al que se le puede pasar el parámetro -r para que haga el reparseo, sin necesidad de modificar código).
Si se está usando tailqueuelog, se debe modificar el código para hacer el reparseo, solo una linea que hay que comentar (alrededor de la línea 434):

```
&reconecta();
&last_event();
&initial_load();```

Tenés que comentar la línea que llama a last_event():

&reconecta();
#&last_event();
&initial_load();

luego de eso, tendrás que detener con kill el tailqueuelog que esté corriendo, y ejecutalo nuevamente a mano desde consola como hiciste anteriormente. Esta vez si va a insertar o intentar insertar todos los registros del log, incluído registros viejos (por lo que va a demorar un buen rato y vas a ver montones de errores de RECORD NOT INSERTED, DUPLICATE o parecido. Eso es normal, ya que va a tratar de insertar lo que ya existe.

Cuando termine, revisá que esté ok corriendo un reporte de ese día. Luego edita el archivo nuevamente, sacale el # a la línea e iniciá tailqueuelog normalmente.