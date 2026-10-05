# Resumen

En el Obligatorio 3 se nos solicita explotar, de distintas formas, las vulnerabilidades presentes en los sitios instalados durante el Obligatorio 1: **Juice Shop** y **crAPI**.

A continuación se detallan las vulnerabilidades que debemos explotar.

## Desafíos Juice Shop

| ID  | Nombre             | Descripción                                                                                                   |
| --- | ------------------ | ------------------------------------------------------------------------------------------------------------- |
| J1  | Zero Stars         | Give a devastating zero-star feedback to the store. Utilizar la palabra **evidentemente** en la respuesta.    |
| J2  | Login Admin        | Log in with the administrator's user account. Usar `or 22=22` en la respuesta.                                |
| J3  | Login Bjoern       | Log in with Bjoern's OWASP user account (bjoern@owasp.org) with the original answer to his security question. |
| J4  | Admin Registration | Register as a user with administrator privileges. El usuario se debe llamar **ernesto**.                      |
| J5  | User Credentials   | Retrieve a list of all user credentials via SQL Injection.                                                    |
| J6  | View Basket        | View another user's shopping basket.                                                                          |
| J7  | Christmas Special  | Order the Christmas special offer of 2014.                                                                    |

## Desafíos crAPI

| ID  | Nombre                                      | Descripción                                                                    |
| --- | ------------------------------------------- | ------------------------------------------------------------------------------ |
| C1  | Find the location of another user's vehicle | Leak sensitive information of another user's vehicle.                          |
| C2  | Reset the password of a different user      | Abuse the password recovery flow process.                                      |
| C3  | Get an item for free                        | Return a product.                                                              |
| C4  | Increase your balance by $1.000 or more     | Find a way to get refunded over 1.000.                                         |
| C5  | SSRF                                        | Make crAPI send an HTTP call to "www.google.com" and return the HTTP response. |
| C6  | NoSQL Injection                             | Find a way to get free coupons without knowing the coupon code.                |
| C7  | DoS                                         | Perform a layer 7 DoS using 'contact mechanic' feature.                        |

## 1. Levantar los contenedores

Lo primero que debemos hacer es levantar Juice Shop y crAPI.

Para crAPI ingresamos a la carpeta `crAPI-main/deploy/docker` y ejecutamos el siguiente comando:

```bash
docker compose -f docker-compose.yml --compatibility up -d
```

![img1](/imagenes/image1.png)

Con el comando

```bash
docker ps
```

verificamos que quedó corriendo correctamente.

![img2](/imagenes/image2.png)

Luego levantamos el contenedor de Juice Shop. Para ello ejecutamos:

```bash
docker run -d --rm --name juice-shop -p 127.0.0.1:3000:3000 bkimminich/juice-shop


docker ps
```

![img3](/imagenes/image3.png)

Aquí podemos ver que el contenedor queda levantado.

## 2. Configurar ZAP

También usaremos **ZAP** para interceptar los paquetes. Para abrirlo ejecutamos:

```bash
zaproxy
```

![img4](/imagenes/image4.png)

## 3. Resolución del práctico

Resolveré los ejercicios en un orden distinto al de la tabla, por lo que iré indicando qué ejercicio corresponde a cada caso.

### Ejercicio 1 — J2: Login Admin (Juice Shop)

Como se trata de un desafío relacionado con el login, voy a la sección de inicio de sesión. En el Práctico 2 ya vimos que para SQL Injection es muy común utilizar `' OR 1=1 --`; además, el enunciado da la pista de usar `' OR 22=22 --`. Por lo tanto, escribo un correo cualquiera y en la contraseña coloco `' OR 22=22 --`.

![img6](/imagenes/image6.png)

Lo que haré entonces será probar ingresando `' OR 22=22 --` directamente en el campo de correo, de la siguiente manera:

![img7](/imagenes/image7.png)

Como vemos a continuación, quedamos logueados como administradores, cumpliendo así con el objetivo.

![img8](/imagenes/image8.png)

**Mitigación:** parametrizar los campos como el de Email, ya que de lo contrario la inyección seguirá siendo posible y se podrá ingresar como cualquier usuario.

### Ejercicio 2 — J3: Login Bjoern

En este ejercicio se nos pide ingresar al sitio con la cuenta `bjoern@owasp.org`. Para ello intentaré nuevamente una SQL Injection, pero esta vez desde ZAP, ya que desde la propia aplicación vimos que no lo permite. Mi idea es colocar en el correo una condición que fuerce a que el usuario autenticado sea el deseado (`bjoern@owasp.org`), algo como: `' OR email = 'bjoern@owasp.org' AND 1=1 --`.

Por error encontré otra vulnerabilidad:

![img9](/imagenes/image9.png)

Aquí vemos que me faltó indicar el email, pero en el proceso descubrí otra vulnerabilidad. Tras investigarla un poco, determiné que se trata de un **manejo indebido de errores**: el sistema está mostrando un error crudo. Me pareció interesante comentarlo como dato curioso. Ahora voy a intentar lo que había planteado antes, con los siguientes datos:

![img10](/imagenes/image10.png)

Luego de enviar esta request verificamos y vemos que logramos ingresar como ese usuario:

![img11](/imagenes/image11.png)

Se cumple así con el objetivo de la consigna.

**Mitigación:** igual que en el ejercicio anterior, parametrizar los campos para evitar la SQL Injection.

### Ejercicio 3 — J4: Admin Registration

En este ejercicio se nos pide registrar a un usuario llamado **ernesto** con privilegios de administrador. Mi hipótesis es que, si realizamos un registro cualquiera e interceptamos los paquetes con ZAP, podremos ver lo que se envía a la base de datos y quizá identificar el campo que define el tipo de usuario, lo que nos permitiría modificarlo. Por lo tanto, lo primero que haré es crear un usuario cualquiera con correo `pepe@gmail.com`, contraseña `123pepe`, pregunta de seguridad _"Your favorite book?"_ y respuesta _"no se"_.

![img12](/imagenes/image12.png)

Ahora observamos la petición en ZAP; esto fue lo que encontré:

![img13](/imagenes/image13.png)

Entonces probaré agregar la nueva línea, de esta forma:

![img14](/imagenes/image14.png)

Esto nos devolvió un _bad request_. Pensé que posiblemente el nombre del rol no fuera `administrator`, sino `admin`.

![img15](/imagenes/image15.png)

Al probar con `admin`, obtuve como resultado que el usuario se creó correctamente.

![img16](/imagenes/image16.png)
![img17](/imagenes/image17.png)

**Mitigación:** la asignación del rol debería manejarse internamente y no pasarse por parámetro. No es un valor que deba viajar en la request, por lo que solo genera una vulnerabilidad innecesaria.

### Ejercicio 4 — J6: View another user's shopping basket

En este ejercicio se nos pide acceder al carrito de otro usuario. Lo primero que se me ocurre es observar qué sucede al entrar a mi propio carrito, para ver si hay alguna URL que pueda aprovechar.

![img18](/imagenes/image18.png)

Encontré que, si bien la página no muestra en qué `id` de basket estoy, al revisar ZAP sí lo encontramos:

![img19](/imagenes/image19.png)

Lo que se me ocurre aquí es copiar esa URL, cambiar el número y ver qué ocurre.

![img20](/imagenes/image20.png)

Después de buscar bastante sin éxito, pasé a intentar otro de los desafíos. Casi sin querer, al entrar al carrito noté que se llamaba a otro endpoint que traía el carrito del usuario que estaba comprando. Decidí entonces reenviar esa petición desde ZAP.

![img21](/imagenes/image21.png)

Luego cambié el número de basket por `1`, con lo cual obtuve el siguiente resultado:

![img22](/imagenes/image22.png)
![img23](/imagenes/image23.png)

Logrando así el resultado esperado: obtener el carrito de otra persona.

**Mitigación:** aplicar control de autorización del actor; es decir, verificar que quien realiza la acción sea efectivamente el dueño de ese carrito.

### Ejercicio 5 — J5: User Credentials

En este ejercicio se nos pide conseguir las credenciales de todos los usuarios mediante SQL Injection. Mi primera idea es realizar, desde el login, una consulta en el campo de correo que, a través de otro `SELECT`, nos permita obtener todos los usuarios. Para ello voy a una request de login y, en ZAP, la abro en la pestaña _Requester_.

![img24](/imagenes/image24.png)

En la pestaña donde se puede modificar la consulta, probé lo siguiente:

![img25](/imagenes/image25.png)

Probé ingresar un `OR 1=1 --` cambiando el método a GET, pero no existe un método así, por lo que no funcionó. Otra idea: cuando me logueo necesito obtener el usuario de alguna manera, así que quizás ahí pueda realizar algo. Decidí loguearme con el usuario `ernesto` creado anteriormente y buscar alguna consulta hacia el back. Por ejemplo, la consulta de `products`, ya que tiene un parámetro `=q`, lo que indica que puede haber una query.

![img29](/imagenes/image29.png)

Lo que se me ocurre es realizar un `UNION` con los campos de usuario que me interesan, lo que resultó en algo como esto:

![img26](/imagenes/image26.png)

Aquí noté que faltaba cerrar dos paréntesis; los agregué y llegué al siguiente resultado:

![img30](/imagenes/image30.png)

El mensaje indica que la primera consulta no tiene la misma cantidad de campos que la que está después del `UNION`, lo cual tiene sentido porque así funciona el operador. Entonces fui agregando `1,1,1,1` hasta igualar la cantidad de columnas, obteniendo el siguiente resultado:

![img27](/imagenes/image27.png)
![img28](/imagenes/image28.png)

Allí podemos ver un usuario camuflado entre los campos de los productos, consiguiendo así el objetivo de obtener a todos los usuarios del sistema.

**Mitigación:** como ya mencioné varias veces, parametrizar los campos que se pasan a la base de datos para impedir la inyección de código SQL.

Con esto concluyo los ejercicios de Juice Shop. A continuación comienzo con los ejercicios de crAPI.

---

Antes de comenzar, abrimos la aplicación en ZAP; en el navegador colocamos `http://localhost:8888`, que es el puerto donde levanté crAPI.

![img31](/imagenes/image31.png)

### Ejercicio 1 — C1: Find the location of another user's vehicle

Primero recorreré un poco la página para entender de qué trata crAPI. Lo primero que encuentro al entrar es un login, por lo que debo registrarme para acceder. Creé el siguiente usuario:

![img32](/imagenes/image32.png)

Luego inicié sesión:

![img33](/imagenes/image33.png)

Al iniciar sesión vemos la pantalla con nuestros vehículos, por lo que debe existir una petición a la base de datos para solicitarlos. Reviso ZAP para ver qué hay.

![img34](/imagenes/image34.png)

No había mucho; solo encontré la request donde se envía el token. Ahí hay algo mal, pero para este ejercicio no es relevante.

Se me ocurre agregar un vehículo y ver qué ocurre.

![img35](/imagenes/image35.png)

Aquí vi algo raro: aunque creé el vehículo, con valores anteriores que había probado el sistema me indicaba que ya existía un vehículo con ese código, lo que podría permitirme encontrar otros a partir de esos PIN. Al final eso no tuvo relación. Mientras exploraba toda la página entré en la sección _Community_; abrí un comentario y, al revisar ZAP, encontré que se muestra la información del usuario que publicó el comentario.

![img36](/imagenes/image36.png)
![img37](/imagenes/image37.png)

Ahora debía encontrar dónde colocar el `id` para obtener la información del vehículo. Probé cambiando la URL donde se ven todos los vehículos por `vehicle/id_encontrado/location` y obtuve el siguiente resultado:

![img38](/imagenes/image38.png)
![img39](/imagenes/image39.png)

Aquí conseguí la ubicación del vehículo de la persona. Cabe destacar que la ruta la obtuve por prueba y error, con rutas comunes que se suelen utilizar.

**Mitigación:** esos datos son privados del usuario y no deberían exponerse. Solo deberían mostrarse los datos no sensibles y estrictamente necesarios para la funcionalidad; en este caso, los correspondientes al comentario del usuario.

Así concluyo el primer ejercicio de crAPI.

### Ejercicio 2 — C2: Reset the password of a different user

En este ejercicio tenemos que cambiar la contraseña de otro usuario. Lo intentaré por la funcionalidad de cambio de contraseña, aprovechando que en el ejercicio anterior el comentario nos reveló, entre otros datos, el correo del usuario.

Primero, en mi propio usuario, realicé un cambio de contraseña para observar qué sucede, pero no encontré nada relevante. Por lo tanto, voy a probar cambiarla desde el login: luego de ingresar el correo `robot001@example.com`, el sistema solicita un código OTP. Puse datos de prueba y un amigo me recomendó investigar qué era un _fuzzer_. Lo busqué y entendí que sirve para enviar una misma request muchas veces con distintos valores. Probé eso en el siguiente endpoint:

![img41](/imagenes/image41.png)

Luego hice clic derecho → _Atacar_ → _Fuzzer_.

![img42](/imagenes/image42.png)

Al abrirlo encontré lo siguiente: hay que marcar el valor que queremos variar, seleccionar _Añadir_ y colocar la secuencia que queremos que pruebe.

![img43](/imagenes/image43.png)

Para la secuencia le pedí a Claude que me generara, en columna, los números del 1 al 1000.

![img44](/imagenes/image44.png)

Ahora lo ejecuto.

![img45](/imagenes/image45.png)

Vi que todas las respuestas eran 500, por lo que quizás el código era un número mayor; probaré entonces hasta 10000. Además, un compañero me comentó que había que cambiar una parte del endpoint: reemplazar `v3` por `v2` (ya que `v3` bloquea luego de 3 intentos, lo que constituye la mitigación).

![img46](/imagenes/image46.png)

Aquí vemos una respuesta 200, que correspondía al código solicitado, logrando el objetivo de cambiar la contraseña.

**Mitigación:** limitar la cantidad de intentos para ingresar el código, como ya se contemplaba en la versión `v3` del endpoint.

### Ejercicio 3 — C4: Increase your balance by $1000 or more

En este ejercicio se nos pide incrementar nuestro balance en 1000 o más. Para ello busco en la sección _Shop_, ya que allí está todo lo relacionado con el balance. En un principio pensé en cambiar el precio de los productos a un valor negativo, pero ningún endpoint envía esa información por la request. Tuve que buscar otras opciones: al comprar un producto vi que sí se enviaban un `id` de producto y una cantidad.

![img47](/imagenes/image47.png)

Probé entonces cambiar la cantidad a un valor negativo.

![img48](/imagenes/image48.png)
![img49](/imagenes/image49.png)

Aquí vemos que tenemos más saldo del que deberíamos.

**Mitigación:** no permitir valores negativos.

### Ejercicio 4 — C3: Get an item for free

En este ejercicio se nos pide obtener un producto gratis. Probé algo parecido al ejercicio anterior: vi que los productos se agregan con saldo negativo, lo que suma dinero a mi cuenta, pero si coloco la cantidad en `0` el producto se registra como comprado a $0.

![img50](/imagenes/image50.png)
![img51](/imagenes/image51.png)

Allí vemos el producto 33, con cantidad 0: no descontó dinero pero sí quedó registrado como comprado.

**Mitigación:** la cantidad debe ser siempre mayor a 0, validándolo en el back para que no pueda ser alterada.

### Ejercicio 5 — C6: NoSQL Injection

En este ejercicio se nos pide realizar una NoSQL Injection para obtener un descuento sin conocer ningún código. Para ello entro en la sección _Add Coupons_ a ver qué encuentro. Al insertar un cupón y darle a _Validar_, encontré este endpoint en ZAP:

![img52](/imagenes/image52.png)

Luego investigué cómo se realiza una NoSQL Injection y encontré que se puede lograr algo parecido a una SQL Injection insertando `{ "$ne": null }`. Lo probé en el campo donde se envía el cupón:

![img53](/imagenes/image53.png)

La respuesta fue:

![img54](/imagenes/image54.png)

Consiguiendo así el código de descuento.

**Mitigación:** validar estrictamente la entrada del usuario, parseándola para que sea un valor primitivo y no un JSON.

Con esto doy por finalizado el Práctico 3. :)

---

> _Nota: la redacción de este documento fue mejorada con la ayuda de Claude._
