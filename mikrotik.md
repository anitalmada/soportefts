###Configuración inicial {#config}

Para configurar un router Mikrotik, conectarlo mediante cable de red y seguir las siguientes instrucciones:

1. Descargar la herramienta [winbox (es un archivo .exe)](https://mikrotik.com/download) y en la terminal ejecutar

        wine winbox.exe

  La documentación de winbox está disponible en https://wiki.mikrotik.com/wiki/Manual:Winbox
2. Una vez que pudimos abrir winbox, en la pestaña `Neighbors` buscar el dispositivo a conectar a través de la MAC. Si no lo permite, se pueden hacer dos cosas: levantar una interfaz virtual o bien cambiar la IP de la interfaz ethernet a una dirección que se encuentre en la misma subred que el router.

3. Agregar las IP correspondientes a las interfaces que necesitamos (LAN y WAN). 
La interfaz ethernet debe tener la IP que provea el cliente en cuya red se instalará el router. 

4. Verificar que ninguna de las conexiones quede como "Master". Chequear que todas tengan seteado el `Master Port: none`

5. Cambiar la password de administrador yendo a System -> users -> doble click en admin -> editar password

6. Verificar que funciona el acceso web al router, modificando el puerto http a 8081.

7. Para finalizar la configuración agregar la IP pública a la interfaz eth5 del router.

8. Verificar que esté desactivado el servidor DHCP.

9. Configurar el firewall para que natee por eth5, siguiendo los pasos del siguiente apartado.

###Cargar las reglas generales del firewall {#loadfw}

1) Editar el archivo Firewall-General.rsc, y cargar las redes internas que correspondan.
2) Subir el archivo vía web desde el menú Files.
3) Abrir un terminal e importar el archivo:

    [admin@MikroTik] > import file-name=Firewall-General.rsc
    
---
###Crear usuarios para VPN {#addusersvpn}

Paso 1:
![](/assets/mkt1.png)

Paso 2:
![](/assets/mkt2.png)

Paso 3:
![](/assets/mkt3.png)

Paso 4:
![](/assets/mkt4.png)

Paso 5:
![](/assets/mkt5.png)

###Crear VPN para OpenVPN {#openvpn}

1) Crear una regla de Firewall para aceptar el tráfico entrante al puerto 1194.
2) Subir los archivos vía web, desde el menú Files: CA.crt, Client.crt, Client.key.
3) Agregar los certificados, desde el menú System > Certificates.

    Import > Only File: CA.crt > Import (sin passphrase)
    Import > Only File: Client.crt > Import (sin passphrase)
    Import > Only File: Client.key > Import (sin passphrase)

4) Crear un pool para VPN, desde el menú IP > Pool.
5) Crear un profile para VPN, desde el menú PPP > Profiles:

    Local Addres: IP del default GW
    Remote Address: Pool para VPN
    Bridge: Bridge creado para puertos LAN

6) Levantar el OpenVPN server, desde el menú PPP > Interface > OVPN Server:

    Port: 1194
    Mode: ethernet
    Default Profile: Profile para VPN
    Certificate: Certificado importado con iniciales KT

7) Crear los usuarios VPN, desde el menú PPP > Secrets.

---
###DynDNS - Script para actualización automática de IP {#iprefresh}

Ir a `System > Scripts`, y crear un nuevo script.
En la sección `Source` pegar el siguiente código, con las credenciales que correspondan:

```
####################
# DATOS PERSONALES #
####################
:global DDNSUser "nombreuser"
:global DDNSPassword "passwduser"
:global DDNSHost "dirección.dyndns.org"
:global Interface "WAN-FIBERTEL"
####################

:log info ("DynDNS: Inicio del script...")
:global DDNSIp [:resolve server=8.8.8.8 $DDNSHost];
:global CurrentInterfaceIp [/ip address get [/ip address find interface=$Interface ] address]
:if ([ :typeof $CurrentInterfaceIp ] = nil ) do={
  :log info ("DynDNS: La interfaz $Interface no tiene IP asignada.")
} else={
  :for i from=( [:len $CurrentInterfaceIp] - 1) to=0 do={
    :if ( [:pick $CurrentInterfaceIp $i] = "/") do={
      :set CurrentInterfaceIp [:pick $CurrentInterfaceIp 0 $i];
    }
  }
  :if ($DDNSIp != $CurrentInterfaceIp) do={
    :log info ("DynDNS: La direccion $DDNSHost se resuelve en la IP $DDNSIp.")
    :log info ("DynDNS: La IP actual en la interface $Interface es $CurrentInterfaceIp.")
    :log info ("DynDNS: Se necesita actualizar la IP. Actualizando...")
    :global String "/nic/update\?hostname=$DDNSHost&myip=$CurrentInterfaceIp&wildcard=NOCHG&mx=NOCHG&backmx=NOCHG"
    :global DDNSMembersIp [:resolve server=8.8.8.8 members.dyndns.org];
    :log info ("DynDNS: La direccion members.dyndns.org se resuelve en la IP $DDNSMembersIp.")
    /tool fetch address=$DDNSMembersIp port=8245 src-path="$String" mode=http user=$DDNSUser password=$DDNSPassword dst-path=("/DynDNS.".$DDNSHost)
    :delay 1
    :global String [/file find name="DynDNS.$DDNSHost"];
    /file remove $String
    :global DDNSIp $CurrentInterfaceIp
    :log info ("DynDNS: La IP del host $DDNSHost ha sido actualizada a $DDNSIp.")
  } else={
    :log info ("DynDNS: No se necesita actualizar la IP.")
  }
}
```

Finalmente, ir a `System > Scheduler`, y crear un nuevo cron.
En la sección `On Event`, colocar el mismo nombre que se le puso al script.

---
###Estadísticas de tráfico {#stats}

    Tools ---> Torch

Elegir la interfaz, y darle Start.
Ordenar por tasa de Tx o Rx.

---
###Redirección de puertos {#portredirect}

    IP ---> Firewall
    NAT ---> Add

    Chain: dstnat
    Dst Port: 80
    Protocol: 6 (tcp)
    Action: dst-nat
    To address: 192.168.0.150
    To port: 80
    Comment: Web redirect

