###Hotline en los Sendos {#hotlinesendos}

Se suele necesitar que cuando se descuelgue un cierto teléfono X, inmediatamanete se dispare una llamada desde ese teléfono a otros internos hasta que alguien atienda. La forma de hacerlo, es la siguiente:

1. Se crea un ringroup con los internos que deben sonar. En lo posible, en loop. Es decir, que el ringroup tiene como destino en caso de fallo el mismo ringroup.
2. Se ingresa a la configuración del teléfono X, en Phone ---> Call service, y como HotLine se apunta al ringroup.

###Upgrade de firmware a través de POST MODE {#upgradepostmode}

- Apagar y prender el teléfono, y ni bien encienda, presionar #. Va a aparecer en la pantalla, la frase "POST MODE".

- Levantar una interfaz local en la PC, con la dirección 192.168.10.2/24. Debiéramos tener ping a la 192.168.10.1, que es la IP que toma el teléfono cuando se pone en POST MODE.

- Levantar un servidor FTP en la máquina local. En mi caso, el directorio FTP es `/srv/ftp`, y allí dejé el firmware con el nombre 2.z. El usuario/password para transferencias es por defecto ftp/ftp.

- Conectarse por telnet al teléfono y realizar el upgrade:

```
 root@Fenix:/srv/ftp# telnet 192.168.10.1
 Trying 192.168.10.1...
 Connected to 192.168.10.1.
 Escape character is '^]'.
                        Voip   Phone   System
                        Post   Version:2.4(SDRAM:8m FLASH:2m)
                        Date:Jan 18 2011   14:05:38

 	1  ----  Show Mac Address
 	2  ----  FTP Update Image
 	3  ----  Clear Configuration
 	4  ----  Exit and Reboot
3
 	1  ----  Show Mac Address
 	2  ----  FTP Update Image
 	3  ----  Clear Configuration
 	4  ----  Exit and Reboot
2
	Input server address:
	192.168.10.2
	Input image name:
	2.z
	Input user name:
	ftp
	Input password:
	ftp
Update start...
The correct length is 1094455
post write mmiset1 addr=0xbf1f0000
Updating sys img,please wait,chipID=0x1c14
Not enough buffer for size 0x1021dc
................Already new!.finish with no mem
Update success!
 	1  ----  Show Mac Address
 	2  ----  FTP Update Image
 	3  ----  Clear Configuration
 	4  ----  Exit and Reboot
4
Connection closed by foreign host.
root@Fenix:/srv/ftp#
```

