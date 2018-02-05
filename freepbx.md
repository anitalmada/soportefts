###Agregar rtptimeout en FreePBX {#rtptimeout}

1. Habilitar FreePBX desde Security -> Advanced settings.
2. Entrar al módulo de FreePBX sin embeber.
3. Hacer un backup de /etc/asterisk/sip_general_custom.conf y luego dejar vacío dicho archivo.
4. Ir a Tools ---> Asterisk SIP settings, y agregar al final regla => rtptimeout = 30.
5. Submitir los cambios y reload.
