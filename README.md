# Soporte Técnico Freetech

This file file serves as your book's preface, a great place to describe your book's content and ideas.Este material está orientado a sistematizar la información necesaria para las tareas que se desarrollan dentro del equipo de Soporte Técnico de Freetech Solutions.

Está organizado por capítulos que cuentan con material sobre cada una de las plataformas o herramientas necesarias para la asistencia a clientes, implementaciones, testeos y resolución de tickets.

---

### Procedimiento de inducción al equipo de Soporte

Cuando se incorpora un agente al equipo de Soporte Técnico, se le deben habilitar los siguientes usuarios/herramientas:

1. Usuario para acceso a Freshdesk \(plataforma de creación de tickets\)
2. Link a monitor vigente y las respectivas credenciales para verlo
3. Usuario en Plone y/o wikis de información que se utilicen al interior del equipo
4. Configuración de cuenta de correo corporativa e incorporación al grupo de correo "soporte\[at\]freetech"
5. Usuario en la plataforma de comunicación interna de la empresa con acceso a los respectivos canales creados para todo lo relacionado al funcionamiento interno \(servicios como Skype, Slack, Mattermost, Jitsi o cualquiera que se esté usando en ese momento\).
6. Configuración de softphone con número de interno propio, orientado a recibir consultas telefónicas y realizar llamadas a clientes \(en caso de ser estrictamente necesario\).

---

### Creación de tickets



Los clientes tienen diferentes vías de comunicación habilitadas según el Plan de Soporte que hayan contratado. Tienen la posibilidad de crear tickets a través de una cuenta de e-mail destinada a tal fin o mediante el uso de la plataforma de autogestión para clientes.

En caso de recibir comunicaciones de los clientes por otros medios que no sean tickets creados por ellos mismos, se debe seguir el siguiente proceso:

* Identificar el medio por el cual se recibió la comunicación para realizar alguna de las siguientes acciones: 
  a. Si es por chat: una vez finalizada la conversación indicar a Freshdesk que convierta el chat en un ticket
  b. Si es por otro medio, generar un ticket nuevo y completar los campos de acuerdo con el tutorial de Freshdesk.
* Proceder a conectarse a los recursos del cliente de acuerdo con los accesos que figuran en Plone para tratar de resolver el incidente.
* Una vez resuelto el incidente ingrear al ticket y cerrarlo agregando la siguiente iniformación: una nota pública -el texto del correo electrónico donde se comunica la resolución al cliente- y la especificación técnica acerca de cómo lo resolvió \(ver el tutorial sobre Freshdesk\).

---

### Herramientas de monitoreo

El equipo de soporte cuenta con una herramienta de monitoreo de infraestructura de los diferentes recursos de los clientes. 

Una vez que cuente con los datos de acceso a dicho servicio, es importante que esté siempre pendiente de las notificaciones de dicha herramienta o consulte con el equipo si se registran señales de advertencia o estados críticos en el monitor.

---

### Wiki de acceso a infraestructura de clientes

La información para poder acceder a los recursos que deben verificarse para resolver incidentes o pedidos de configuración está disponible en una wiki \(al estilo de Plone o cualquier otra que la empresa considere útil\) que el equipo de soporte debe mantener actualizada ya que es el único lugar donde se encuentra dicha información. Una vez generado el usuario para poder ingresar a la wiki, cada integrante debe recordar que es importante actualizar dicha información a medida que surgen modificaciones en la infraestructura de los clientes.

Para saber cómo editar, se pueden consultar los manuales de uso de la herramienta seleccionada, en la actualidad es Plone CMS.

---

### Uso de la cuenta de correo corporativa

Una vez que comiencen a realizarse comunicaciones por correo electrónico con clientes para poder resolver incidentes, deben tenerse en cuenta los siguientes puntos:

* Los correos electrónicos referidos a la resolución de un ticket deben tener incorporado en el asunto el número de dicho ticket con el siguiente formato \[\#XXXX\] \(donde "XXXX" es el número identificador del ticket\).
* Cuando se responda un e-mail, debe recordarse utilizar siempre la opción "Responder a todos", verificando especialmente que se encuentre la cuenta del grupo de soporte para que el thread le llegue a todo el equipo. Se debe solicitar al cliente que realice lo mismo a la hora de responder \(en caso que no lo hubiera hecho\).
* En términos comunicacionales, los correos deben mantener un tono amable y condescendiente hacia el cliente. En la medida de lo posible deben ser lo suficientemente explicativos pero breves. Siempre que deba refutarse alguna premisa establecida por el cliente deben aportarse pruebas \(captura de pantalla de logs, de servicios funcionando, etc.\) y a la vez consultar si se verificó correctamente o solicitar mayor información respecto del incidente. Es importante recordar que la cuenta de correo es corporativa y por lo tanto las comunicaciones establecidas representan tanto al/a la agente como a la empresa.







