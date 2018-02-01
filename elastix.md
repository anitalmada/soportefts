###Elastix - Actualización de repositorios para que funcionen los updates {#actualizarepos}

Actualizar los siguientes archivos:

En el archivo /etc/yum.repos.d/commercial-addons.repo, actualizar el bloque completo de commercial-addons, y dejarlo tal como se muestra a continuación:

```
[commercial-addons]
name=Commercial-Addons RPM Repository for Issabel
mirrorlist=http://mirror.issabel.org/?release=2.5&arch=$basearch&repo=commercial_addons
#baseurl=http://repo.issabel.org/elastix/2/commercial_addons/$basearch/
gpgcheck=1
enabled=1
gpgkey=http://repo.issabel.org/elastix/RPM-GPG-KEY-Issabel
```


En los archivos /etc/yum.repos.d/Centos-Base.repo y /etc/yum.repos.d/elastix.repo, actualizar los enlaces de los bloques de base, updates y extras, y dejarlo tal como se muestra a continuación:

```
baseurl=http://vault.centos.org/5.11/os/x86_64/

baseurl=http://vault.centos.org/5.11/updates/x86_64/

baseurl=http://vault.centos.org/5.11/extras/x86_64/
```

Finalmente, comentar el mirrorlist en los 3 bloques.

---

###Analizar colas de mails {#colasmails}

Inspecting Postfix’s email queue.

This post explains how to view messages in the postfix queue, another post on this blog explains how to delete or selectively delete from the postfix queue

Postfix maintains two queues, the pending mails queue, and the deferred mail queue, the differed mail queue has the mail that has soft-fail and should be retried (Temporary failure), Postfix retries the deferred queue on set intervals (configurable, and by default 5 minutes).

In any case, the following commands should be useful.

**Display a list of queued mail, deferred and pending**

`mailq`

or

`postqueue -p`

**To save the output to a text file you can run**

`mailq > myfile.txt`

or

`postqueue -p > myfile.txt`

The above commands display all queued messages (Not the message itself but the sender and recipients and ID), The ID is particularly useful if you want to inspect the message itself.
2- View message (contents, header and body) in Postfix queue

Assuming the message has the ID XXXXXXX (you can see the ID form the QUEUE)

`postcat -vq XXXXXXXXXX`

Or to save it in a file

`postcat -vq XXXXXXXXXX > themessage.txt`

3- Tell Postfix to process the Queue now

`postqueue -f`

OR

`postfix flush`

**Delete all queued mail**

`postsuper -d ALL`

**Delete differed mail queue messages** (the ones the system intends to retry later)

`postsuper -d ALL deferred`

**Delete from queue selectively**

To delete from the queue all emails that have a certain address in them, we can use [this program](http://www.buildingcubes.com/wp-content/files/postfix-queue-delete.zip "Perl Script v1") (perl script).

NOTE: This perl script seems to be free, and is all over the internet, i could not find out where it originates or who wrote it.

Download [this file](http://www.buildingcubes.com/wp-content/files/postfix-queue-delete.zip "Perl Script v1"), unzip, and upload the file to your server, then from your bash command line, Change Directory to wherever you uploaded this file, for example cd /root (Just an example, You can upload it wherever you wish)

NOTE: A second script here works differently, i have not yet tested it, [download it here](http://www.buildingcubes.com/wp-content/files/second_postfix-queue-delete.zip "Perl Script v2")

Now, from within that directory, execute

`./postfix-queue-delete.pl anyaddress@example.com`

Any mail that has this email address in it’s IN or OUT list will be deleted.

The script uses the postqueue -p then looks for your string, once found, it deletes the email by ID, this means that this script can delete messages using any text that appears when you run mailq (or postqueue -p), so if you run it with the parameter joe all mail with addresses such as joefriend@example.com and

Other moethods exist, like executing directly

`mailq | tail +2 | grep -v '^ *(' | awk  'BEGIN { RS = "" } { if ($8 == "email@address.com" && $9 == "") print $1 } ' | tr -d '*!' | postsuper -d -`

——————————–

Sample Messages in a differed mail queue

——————————–
```
SOME282672ID 63974 Mon Nov 29 05:12:30 someaddresss@yahoo.com (temporary failure. Command output: maildrop: maildir over quota.)
localuser@exmple.com
```
———————————-
```
SOME282672ID 9440 Wed Jun 30 05:30:11 MAILER-DAEMON (SomeHostName [xxx.xxx.xxx.xxx] said: 452  Mailbox size limit exceeded (in reply to RCPT TO command)) 
username@example.org
```

———————————-
```
SOME282672ID 4171 Thu Nov 25 13:22:03 MAILER-DAEMON (host inbound.somedomain.net [yyy.yyy.yyy.yyy] refused to talk to me: 550 Rejected: 188.xx.179.46, listed at http://csi.cloudmark.com/reset-request for remediation.)
someuser@example.com
```
———————————
```
SOME282672ID 37031 Thu Nov 25 08:53:36 someuser@example.net
(Host or domain name not found. Name service error for name=example.com type=MX: Host not found, try again)
someuser@example.com 
```

---



