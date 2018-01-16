## Asternic
Es un módulo de estadísticas orientado a las necesidades de los call centers. 

### Contenido
1. [Agregar salientes a Asternic](#aggsalientes)
2. [Call Center Asternic](#callcenter)
3. [Eliminar datos de qstats](#eliminarqstats)
4. [Enviar llamadas entrantes por líneas FXO a colas, por consola](#llamadafxocolas)


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
