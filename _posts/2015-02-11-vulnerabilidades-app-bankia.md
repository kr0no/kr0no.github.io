---
title: Vulnerabilidades en APP móvil Bankia
layout: post
date:   2015-05-29
---
Cada día el número de aplicaciones que precisan almacenar algún tipo de dato en la nube, o que necesitan obtener la información o el contenido de la aplicación a través de un endpoint aumenta considerablemente. En la mayoría de ocasiones, estas aplicaciones han pasado meticulosos controles de calidad y seguridad (o así debería ser), pero a los endpoints no se les suele dar la misma importancia, por lo que un atacante podría encontrarse alguna puerta medio abierta que le podría permitir comprometer el servidor en cuestión y por ejemplo, obtener información sensible que puede ser muy beneficiosa para él (por ejemplo una BD).

Es usual encontrar servidores endpoint vulnerables cuando el desarrollador de la aplicación es una única persona en su casa, haciéndolo por hobby o por sacarse un pequeño sobresueldo a base de publicidad, o en menor medida, de ventas de la app, y tiene su parte de lógica ya que salvo casos excepcionales, este tipo de aplicaciones no pasan los controles de calidad y seguridad que se mencionaban antes. Sin embargo, estos fallos también se encuentran en aplicaciones de mayor envergadura las cuales tienen un equipo de desarrollo detrás, que junto a las pruebas de seguridad, deberían garantizar la seguridad de los usuarios y de los datos almacenados en sus propios servidores, y esto ya no tiene tanta lógica.

Hace unos días empecé a testear las apps bancarias a las cuales tengo acceso, empezando por Bankia. Esta aplicación nos ofrece una oficina virtual para realizar prácticamente cualquier tipo de gestión que podríamos realizar a través de la propia web, accediendo para ello a una API alojada en "m.bankia.es". Las peticiones relacionadas con la sesión del usuario viajan siempre por HTTPS, pero para obtener contenido estándar (noticias, productos, últimos tweets, etc), los datos se envían y reciben a través de texto plano (HTTP).

Al ver las peticiones que viajaban a través de HTTP (del estilo "contenido.json?id=XXX", casi por inercia metí una comilla, por si acaso. En la primera prueba no salió nada, pero en la segunda, me quedé sorprendido al ver lo siguiente:

![BlindSQL](/images/vulnerabilidades-app-bankia/BlindSQL.png)

Una Blind SQL Injection de manual, sin ningún tipo de filtro o protección, ni un triste WAF en el servidor. No se a ciencia cierta hasta que punto podría llegar a comprometer los datos almacenados en la DB, ni la confidencialidad de estos.

Siguiendo con el pequeño análisis de los endpoints, vi que el acceso al portal de la Bolsa era simplemente un WebView que mostraba la URL "http://asp.infobolsanet.com/BankiaMobile/".
En esta web tan solo hay una página ("Default.aspx") que se encarga de mostrar el resto del contenido en base al valor de los parámetros "PageID, key, source, date", y el que nos interesa: "SessionID", el cual no filtra correctamente el contenido y provoca una vulnerabilidad de XSS reflejado:

![XSS](/images/vulnerabilidades-app-bankia/XSS.png)

Esto en principio no representa una vulnerabilidad en la aplicación como tal (recordemos que estos datos viajan a través de HTTP, por lo que un atacante podría modificar la respuesta que el servidor manda al usuario legítimo sin necesidad de ninguna vulnerabilidad más), pero podría facilitar mucho más un ataque de, por ejemplo, phising, solicitando al usuario legítimo sus datos personales, todo esto sin salir de la propia app, sin ver la URL y sin ver a donde se están enviando esos datos.

![MITM](/images/vulnerabilidades-app-bankia/MITM.png)

Como curiosidad, otra de las cosas que se pueden encontrar en el código fuente es una clase llamada "TrustAllSSLFactory.class", cuya función es evadir la comprobación de que el certificado SSL del endpoint es válido, y permitiría a un atacante utilizar un certificado self-signed, viendo así todos los datos sensibles del usuario (DNI, clave, números de tarjeta, transacciones, etc). Sin embargo, por lo que he podido ver y en las pruebas que he realizado, la aplicación ya no hace uso de esta clase, y comprueba satisfactoriamente que el certificado SSL sea correcto (pero si está ahí, será por que en algún momento se ha utilizado, ¿no? ;) ).

Esto lo he encontrado haciendo un análisis muy superficial de la aplicación, sin ser ningún experto en este tema, por lo que un usuario malintencionado con suficientes conocimientos podría llegar muy lejos con estos fallos (o incluso ya podría haber llegado).

Las vulnerabilidades fueron notificadas en su momento, y a día de hoy, se encuentran correctamente parcheadas.