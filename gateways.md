###Detección de cortes en GW Dinstar FXO {#cortesdinstar}

Básicamente, se debe modificar el valor "busytone DeltaMS", el cual representa al tiempo de silencio entre los tonos de ocupado.
Para eso, se debe ingresar por telnet al GW, e ingresar los siguientes comandos:

    [root@alx2000 ~]# telnet 172.31.40.3
    Trying 172.31.40.3...
    Connected to 172.31.40.3 (172.31.40.3).
    Escape character is '^]'.

    Welcome to Command Shell! 
    Username:admin 
    Password:***** 
    ROS> 
    ROS>?
    enable Turn on privileged commands
    exit   Exit from the EXEC
    show   Show running system information
    ROS>enable
    ROS#?
    dbg               Show ada information
    dspconfigure      Configure device parameters
    exit              Exit from privelige mode
    menuconfigure     Configure system parameters
    ntp               Configure ntp_sntp parameters
    ping              Send echo messages
    show              Show running system information
    ROS#dspconfigure
    note: you can input <quit> to quit
```
-----------------------------------------------
-------------------Normal set-------------------
-----------------------------------------------
```
    PCM <- NET Port volume gain: (-32~32), default:0
        value = 0 :  
    PCM -> NET Port volume gain: (-32~32), default:0
        value = 0 :  
    FXO CODEC <- PHONE  gain: 0 ~ 10 (0 ~ 10 db), default:3
        value = 3 :  
    FXO CODEC -> PHONE  gain: 0 ~ 10 (0 ~ 10 db), default:4
        value = 4 :  
    FXS CODEC <-PHONE   gain: 0 ~ 2 (0: 0db; 1: -3.5db;2: +3.5db), default:0
        value = 0 :  
    FXS CODEC -> PHONE  gain: 0 ~ 2 (0: 0db; 1: -3.5db;2: +3.5db), default:0
        value = 0 :  
    Port SCE flag: (0, 1), default:1
        value = 0 :  
    busytone lo_cutoff_freq:  200hz~3000hz (if 0, use default value 300hz)
        value = 300 :  
    busytone hi_cutoff_freq:  200hz~3000hz (if 0, use default value 550hz)
        value = 700 :  
    busytone DeltaMs:  30ms~1000ms (Default 50ms)
        value = 50 : 200 <----------------------------------- Ésto lo pase de 50mseg a 200mseg
    set fxo max tone candence :2-10 default:4
         max tone candence= 4  :  
    Set Fxo detect cid sended before ring?   0 - 1 (0: No; 1: Yes) (default: 0)
        value = 0 :  
    Set hintvoice time for MockFxo incall & busy fxs:  (0~20S), (default: 10S; 0-3s: disable)
        value = 10 :  
    IpPhone AutoDialer enable: 0 - 1 (default: 0/Disable)
        value = 0 :  
    fax max rate:  2400,4800,7200,9600,12000,14400(unit:bps)(default : 14400)
        value = 14400 :  
    fax volume:  (-9 ~ 9)(default: 0)
        value = 0 :  
    fax pkt send internal:   (10 ~ 80), (default : 10ms)
        value = 10 :  
    Set fax mode ?   0 - 3 (0: T38; 1: T30; 2: Modem; 3: Adaptive) (default: 3)
        value = 3 :  
    fxo enable init periodically:0-disable 1-enable 
        fxo init periodically = 0 :  
    fxo init period:30 second~ 15 minute
        fxo init period = 180 s:  
    fxs enable init periodically:0-disable 1-enable 
        fxs init periodically = 0 :  
    fxs init period:30 second~ 15 minute
        fxs init period = 180 s:  
    set dtmf detect min duration: 10-100ms
        dtmf detect min duration = 40 ms :  
    set fxo min offhook voltage: 1-10V
        fxo min offhook voltage = 4 V :  
    set fxo min onhook voltage: 16-48V
        fxo min onhook voltage = 24 V :  
    set fxo min offhook current: 5-12 MA
        fxo min offhook current = 8 MA :  
    set fxo max offhook current: 18-160 MA
        fxo max offhook current = 45 MA :  
    set fxo DCR: 0:50,1:800 
        fxo DCR = 0  :  
    set fxo current limit: 0:disable,1:enable 
        fxo current limit = 0  :  
    send rtp keepalive: 0-do not send, 1-send
        value = 0 :  
    modem bypass  enable:0-disable 1-enable 
       modem bypass  enable = 0 :  
    suport fax:0-disable 1-enable 
       suport fax  = 1 :  
    fxo detect no ring timeout:(6000-160000ms default:6000 )
       fxo detect no ring timeout  = 6000 :  
    fxo acim match echo delta:(10-60 default:28 )
       fxo acim match echo delta  = 32 :  
        Apply the modified configuration(Y/N):  Y 
    ROS#^config
    ROS(config)#save
    ROS(config)#
    ROS(config)#reset 
    Are you sure to reset? (y/n):y 
    ROS(config)#Connection closed by foreign host.
---
###GW Grandstream FXO - Discado a la PSTN con tomando línea con 0 {#discadograndstream}

Si es necesario salir a la PSTN enviando el dígito 0 para que la Telco nos dé señal de marcado, y recién allí ingresar el número, enviamos el número completo con prefijo al GW, por ejemplo:

00351156375144

Luego en el GW Grandstream, vamos al menú FXO **Lines > Dialing > Dial DTMF Settings**, y configuramos el siguiente parámetro:

    Dial Pause Between Digit: ch1-8:d1p100;

De esa forma, estamos diciendo que en los 8 puertos, enviemos el primer dígito y hagamos una pausa de 1.000 mseg (100 x 10 mseg) antes de enviar el resto del número.

Aquí la explicación del foro de Grandstream:

*What is a pause in the dial plan and how do I configure this on my GXW410x?*

*Some offices may require employees to dial '9' then the phone number in order to call out . If the user dials '9' + phone number as one consistent string, the call will not go through and a generic prompt will be generated, e.g. "Your call cannot be completed as dialed. Please check the number and dial again.". If this is the case, this can easily be resolved by adding a pause in the dial-plan.*

*To add a pause in the dial plan you will need to configure the “DTMF Dial Pause Between Each Digit(X10ms)”, which can be found under “Dial Plan” tab located in the web-GUI. This is the syntax used to configure the pause: ‘ch1-4:d2p200, d4p400; ch5-8:
d1p100, d3p300’ dx/py means pause 10y-ms after d-th digit is dialed.*

*For example:*
*- if 100ms needs to be inserted after dialing the 4th digit, the syntax will be ch1-4:d4p10;*
*- if 300ms needs to be inserted after dialing the 3rd digit and a second pause of 400ms after the 6th digit the syntax will be: ch1-4:d3p30,d6p40;*

*Note: The dial plan pause ONLY works in Stage 1 dialing method.*

---
###Grandstream - Sincronizar el phonebook {#sincrophonebook}

1) Cronear el script `/root/Phonebook.php`:

```
######################
<?php

header("Content-type: text/xml");

$Host = "localhost";
$User = "root";
$Password = "Freetech123";
$Database = "asterisk";

$LinkID = mysql_connect($Host, $User, $Password) or die("Could not connect to host.");
mysql_select_db($Database, $LinkID) or die("Could not find database.");

$Query = "SELECT user, description FROM devices ORDER BY description ASC";
$ResultID = mysql_query($Query, $LinkID) or die("Data not found.");

$DB = new PDO('sqlite:/var/www/db/address_book.db') or die("Could not connect to host.");
$Results = $DB->query('SELECT * FROM contact')or die("Data not found");

$XMLOutput = "<?xml version=\"1.0\"?>\n";
$XMLOutput .= "<AddressBook>\n";
for($x = 0 ; $x < mysql_num_rows($ResultID) ; $x++){
	$Row = mysql_fetch_assoc($ResultID);
	$XMLOutput .= "\t<Contact>\n";
	$XMLOutput .= "\t\t<LastName>" . $Row['description'] . "</LastName>\n";
	$XMLOutput .= "\t\t<FirstName></FirstName>\n";
	$XMLOutput .= "\t\t\t<Phone>\n\t\t\t\t<phonenumber>" . $Row['user'] . "</phonenumber>\n";
	$XMLOutput .= "\t\t\t\t<accountindex>1</accountindex>\n";

	$XMLOutput .=  "\t\t\t</Phone>\n";
	$XMLOutput .= "\t</Contact>\n";
}

while ($Row = $Results->Fetch()) {
        $XMLOutput .= "\t<Contact>\n";
        $XMLOutput .= "\t\t<LastName>" . $Row['last_name'] . "</LastName>\n";
        $XMLOutput .= "\t\t<FirstName>" . $Row['name'] . "</FirstName>\n";
        $XMLOutput .= "\t\t\t<Phone>\n\t\t\t\t<phonenumber>" . $row['telefono'] . "</phonenumber>\n";
        $XMLOutput .= "\t\t\t\t<accountindex>1</accountindex>\n";
        $XMLOutput .=  "\t\t\t</Phone>\n";
        $XMLOutput .= "\t</Contact>\n";
}
$XMLOutput .= "</AddressBook>";

$Filename = "/tftpboot/phonebook.xml";

$FP = fopen($Filename, 'w');
fwrite($FP, $XMLOutput);
fclose($FP);

echo $XMLOutput;
?>
######################
```

El cron sería:

    [root@server ~]# cat /etc/cron.d/Phonebook 
    00 0,6,12,18 * * * root php /root/Phonebook.php

2) En el teléfono, acceder por Web al menú **Phonebook > Phonebook Management**, y cargar la siguiente configuración:

    Enable Phonebook XML Downlad: Enabled, use TFPT
    Phonebook XML server path: IP (Por ejemplo, 10.116.24.10)
    Phonebook Download Interval: 120
    Remove Manually-edited Entries on Download: Yes

---






