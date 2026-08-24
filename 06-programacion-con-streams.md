# Parte 1: Fundamentos de Node.js

## Capítulo 6: Programación con Streams

Mencionamos brevemente los streams en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos, y en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con Promesas y Async/Await, como una opción para hacer que parte de nuestro código sea un poco más robusto. ¡Ahora, finalmente es el momento de profundizar! Estamos aquí para hablar de streams: uno de los componentes y patrones más importantes de Node.js. Hay un lema en la comunidad que dice: "¡aplica streams a todas las cosas!" (*stream all the things!*), y esto por sí solo debería ser suficiente para describir el papel de los streams en Node.js. Dominic Tarr, uno de los primeros colaboradores de la comunidad de Node.js, definió los streams como "la mejor idea de Node y la más incomprendida". Hay diferentes razones que hacen que los streams de Node.js sean tan atractivos; no solo se trata de propiedades técnicas, como el rendimiento o la eficiencia, sino más bien de su elegancia y de la forma en que encajan perfectamente en la filosofía de Node.js. Sin embargo, a pesar de su potencial, los streams siguen estando infrautilizados en la comunidad de desarrolladores en general. A muchos les resultan intimidantes y prefieren evitarlos por completo. Este capítulo está aquí para cambiar eso. Exploraremos los streams en profundidad, destacaremos sus ventajas y los presentaremos de una manera clara y accesible, poniendo su poder al alcance de todos los desarrolladores.

Pero antes de sumergirnos, tomemos un breve descanso para una nota del autor (Luciano por aquí). Los streams son uno de mis temas favoritos en Node.js, y no puedo evitar compartir una historia de mi carrera en la que los streams realmente salvaron el día.

Estaba trabajando para una empresa de seguridad de redes en un equipo que desarrollaba una aplicación en la nube. El propósito de la aplicación era recopilar metadatos de red de dispositivos físicos que monitorizaban el tráfico en entornos corporativos. Imagina registrar todas las conexiones entre hosts en la red, qué protocolos están usando y cuántos datos están transfiriendo. Estos datos podrían ayudar a detectar el movimiento de un atacante en la red o descubrir intentos de exfiltración de datos. La idea era simple pero poderosa: en caso de un incidente de seguridad, nuestros clientes podían iniciar sesión en nuestra plataforma, navegar a través de los metadatos registrados y descubrir exactamente qué sucedió, lo que les permitía tomar medidas rápidamente.

Como podrás imaginar, esto requería transmitir continuamente una cantidad significativa de datos desde los dispositivos en las instalaciones del cliente a nuestro servidor web basado en la nube. Con el espíritu de mantener las cosas simples y entregar rápido, nuestra implementación inicial del recopilador de datos (el servidor HTTP que recibía y almacenaba metadatos) utilizó un enfoque basado en buffers (*buffered approach*).

Los dispositivos enviaban metadatos de red en tramas (*frames*) cada minuto, y cada trama contenía todas las observaciones de los 60 segundos anteriores.

Así es como funcionaba: cargábamos la trama completa en memoria a medida que llegaba, y solo después de recibir la trama completa la escribíamos en el almacenamiento persistente. Esto funcionó bien al principio porque solo atendíamos a clientes pequeños que generaban cantidades relativamente modestas de metadatos, incluso durante los picos de tráfico.

Pero cuando implementamos la solución para un cliente más grande, las cosas comenzaron a fallar. Notamos fallas ocasionales en el recopilador y, lo que es peor, lagunas en los datos almacenados. Después de investigar el problema, descubrimos que el recopilador se estaba bloqueando debido al uso excesivo de memoria. Si un cliente generaba una trama particularmente grande, el sistema no podía manejarla, lo que provocaba la pérdida de datos.

Este era un problema grave. Toda nuestra propuesta de valor dependía de poder almacenar y recuperar metadatos de red de manera confiable para el análisis forense. Si los clientes no podían confiar en nosotros para preservar sus datos, la plataforma era efectivamente inútil.

Necesitábamos una solución, y rápido. La raíz del problema estaba clara: almacenar tramas enteras en buffers en memoria fue un error de novato. ¿La solución? Mantener baja la huella de memoria procesando los datos en fragmentos (*chunks*) más pequeños y escribiéndolos en el almacenamiento de forma incremental.

Aquí entran los streams de Node.js. Con los streams, pudimos procesar los datos pieza por pieza a medida que llegaban, en lugar de esperar la trama completa. Después de refactorizar nuestro código para usar streams, pudimos manejar terabytes de datos diariamente sin despeinarnos. La latencia del sistema mejoró drásticamente: los clientes podían ver sus datos en la nube en menos de dos minutos. También redujimos costos al usar máquinas más pequeñas con menos memoria, y la nueva implementación fue mucho más elegante y fácil de mantener, gracias a la naturaleza componible de la API de streams de Node.js.

Si bien esto puede sonar como un caso de uso específico, las lecciones aquí se aplican ampliamente. Cada vez que mueves datos de A a B, especialmente cuando se trata de volúmenes impredecibles o cuando los resultados tempranos son valiosos, los streams de Node.js son una herramienta invaluable.

¡Te prometo que una vez que aprendas los fundamentos de los streams, apreciarás su poder y verás muchas oportunidades para aprovecharlos en tus aplicaciones!

Este capítulo tiene como objetivo proporcionar una comprensión completa de los streams de Node.js. La primera mitad de este capítulo sirve como introducción a las ideas principales, la terminología y las bibliotecas detrás de los streams de Node.js. En la segunda mitad, cubriremos temas más avanzados y, lo más importante, exploraremos patrones de streaming útiles que pueden hacer que tu código sea más elegante y efectivo en muchas circunstancias.

En este capítulo, aprenderás sobre los siguientes temas:
- Por qué los streams son tan importantes en Node.js.
- Comprensión, uso y creación de streams.
- Los streams como paradigma de programación: aprovechar su poder en muchos contextos diferentes y no solo para E/S (I/O).
- Patrones de streaming y conexión de streams en diferentes configuraciones.

Sin más preámbulos, descubramos juntos por qué los streams son una de las piedras angulares de Node.js.

---

### Sección 1: Descubriendo la importancia de los streams

En una plataforma basada en eventos como Node.js, la forma más eficiente de gestionar la E/S es en tiempo real, consumiendo la entrada tan pronto como esté disponible y enviando la salida tan pronto como la aplicación la produzca.

En esta sección, te daremos una introducción inicial a los streams de Node.js y sus puntos fuertes. Ten en cuenta que esto es solo una descripción general, ya que más adelante en este capítulo se realizará un análisis más detallado sobre cómo usar y componer streams.

#### Buffering frente a streaming

Casi todas las APIs asíncronas que hemos visto hasta ahora en este libro funcionan mediante el modo buffer (*buffer mode*). Para una operación de entrada, el modo buffer hace que todos los datos provenientes de un recurso se recopilen en un buffer hasta que se complete la operación; luego se devuelve al invocador como un único bloque (*blob*) de datos. El siguiente diagrama muestra un ejemplo visual de este paradigma:

**Figura 6.1:** Buffering.

En la Figura 6.1, nuestro objetivo es transferir datos que contienen la cadena "Hello Node.js" desde un recurso a un consumidor. Este proceso ilustra el concepto del modo buffer, donde todos los datos se acumulan en un buffer antes de ser consumidos. En el instante t1, el primer fragmento de datos, "Hello N", se recibe del recurso y se almacena en el buffer. En t2, llega el segundo fragmento, "ode.js", completando la operación de lectura. Con la cadena completa ahora totalmente acumulada en el buffer, se envía al consumidor en t3.

Los streams proporcionan un enfoque diferente, permitiendo que los datos se procesen incrementalmente a medida que llegan del recurso. Esto se muestra en el siguiente diagrama:

**Figura 6.2:** Streaming.

Esta vez, la Figura 6.2 muestra que, tan pronto como se recibe cada nuevo fragmento de datos del recurso, se pasa inmediatamente al consumidor, quien ahora tiene la oportunidad de procesarlo de inmediato, sin esperar a que todos los datos se recopilen en el buffer.

Pero, ¿cuáles son las diferencias entre estos dos enfoques? Desde una perspectiva puramente de eficiencia, los streams son generalmente más eficientes en términos de espacio (uso de memoria) y, a veces, incluso en términos de tiempo de cálculo de reloj. Sin embargo, los streams de Node.js tienen otra ventaja importante: la **composabilidad**. Veamos ahora qué impacto tienen estas propiedades en la forma en que diseñamos y escribimos nuestras aplicaciones.

#### Eficiencia espacial

En primer lugar, los streams nos permiten hacer cosas que no serían posibles almacenando datos en buffers y procesándolos todos a la vez. Por ejemplo, considera el caso en el que tenemos que leer un archivo muy grande, digamos, del orden de cientos de megabytes o incluso gigabytes. Claramente, usar una API que devuelva un buffer grande cuando el archivo se lea por completo no es una buena idea. Imagina leer algunos de estos archivos grandes de forma concurrente; nuestra aplicación se quedaría sin memoria fácilmente. Además de eso, los buffers en V8 tienen un tamaño limitado. No puedes asignar más de unos pocos gigabytes de datos, por lo que podríamos chocar contra un muro mucho antes de quedarnos sin memoria física.

El tamaño máximo real de un buffer cambia entre plataformas y versiones de Node.js. Si tienes curiosidad por saber cuál es el límite en bytes en una plataforma determinada, puedes ejecutar este código:

```javascript
import buffer from 'node:buffer'
console.log(buffer.constants.MAX_LENGTH)
```

##### Gzipping usando una API con buffering

Para poner un ejemplo concreto, consideremos una aplicación de línea de comandos simple que comprime un archivo utilizando el formato GZIP. Usando una API con buffering, dicha aplicación se verá de la siguiente manera en Node.js (el manejo de errores se omite por brevedad):

```javascript
// gzip-buffer.js
import { readFile, writeFile } from 'node:fs/promises'
import { gzip } from 'node:zlib'
import { promisify } from 'node:util'

const gzipPromise = promisify(gzip) // note: gzip is a callback-based function

const filename = process.argv[2]

const data = await readFile(filename)
const gzippedData = await gzipPromise(data)
await writeFile(`${filename}.gz`, gzippedData)
console.log('File successfully compressed')
```

Ahora, podemos intentar ejecutarlo con el siguiente comando:

```bash
node gzip-buffer.js <path to file>
```

Si elegimos un archivo que sea lo suficientemente grande (por ejemplo, 8 GB o más), lo más probable es que recibamos un mensaje de error que dice que el archivo que estamos intentando leer es más grande que el tamaño de buffer máximo permitido:

```text
RangeError [ERR_FS_FILE_TOO_LARGE]: File size is greater than possible Buffer
```

Eso es exactamente lo que esperábamos, y es un síntoma del hecho de que estamos utilizando el enfoque equivocado.

Ten en cuenta que el error ocurre cuando ejecutamos `readFile()`. Aquí es donde tomamos todo el contenido del archivo y lo cargamos en un buffer en memoria. Node.js verificará el tamaño del archivo antes de comenzar a cargar su contenido. Si el archivo es demasiado grande para caber en un buffer, se nos presentará el error `ERR_FS_FILE_TOO_LARGE`.

##### Gzipping usando streams

La forma más sencilla que tenemos de arreglar nuestra aplicación Gzip y hacer que funcione con archivos grandes es utilizar una API de streaming. Veamos cómo se puede lograr esto. Escribamos un nuevo módulo con el siguiente código:

```javascript
// gzip-stream.js
import { createReadStream, createWriteStream } from 'node:fs'
import { createGzip } from 'node:zlib'

const filename = process.argv[2]

createReadStream(filename)
  .pipe(createGzip())
  .pipe(createWriteStream(`${filename}.gz`))
  .on('finish', () => console.log('File successfully compressed'))
```

"¿Eso es todo?", te preguntarás. ¡Sí! Como dijimos, los streams son increíbles debido a su interfaz y composabilidad, lo que permite un código limpio, elegante y conciso. Veremos esto dentro de un momento con más detalle, pero por ahora, lo importante es darse cuenta de que el programa se ejecutará sin problemas contra archivos de cualquier tamaño y con una utilización de memoria constante. Pruébalo tú mismo (pero ten en cuenta que comprimir un archivo grande puede llevar un tiempo).

Ten en cuenta que, en el ejemplo anterior, omitimos el manejo de errores por brevedad. Analizaremos los matices del manejo adecuado de errores con streams más adelante en este capítulo. Hasta entonces, ten en cuenta que la mayoría de los ejemplos carecerán de un manejo adecuado de errores.

#### Eficiencia temporal

Podríamos hablar de la eficiencia temporal de los streams en términos abstractos, pero probablemente sea mucho más fácil entender por qué los streams son tan ventajosos viéndolos en acción. Trabajemos en algo práctico para apreciar cómo los streams ahorran tiempo y recursos en escenarios del mundo real.

¡Construyamos una nueva aplicación cliente-servidor! Nuestro objetivo es crear un cliente que lea un archivo del sistema de archivos, lo comprima y lo envíe a un servidor a través de HTTP. Luego, el servidor recibirá el archivo, lo descomprimirá y lo guardará en una carpeta local. De esta manera, ¡estamos creando nuestra propia utilidad casera de transferencia de archivos!

Para lograr esto, tenemos dos opciones: podemos usar una API basada en buffers o aprovechar los streams. Si no esperamos transferir archivos grandes, ambos enfoques cumplirán con el trabajo, pero difieren significativamente en cómo se procesan y transfieren los datos.

Si fuéramos a utilizar una API con buffering para esto, el cliente primero necesitaría cargar el archivo completo en memoria como un buffer. Una vez que el archivo esté completamente cargado, comprimirá los datos, creando un segundo buffer que contendrá la versión comprimida. Solo después de estos pasos el cliente podrá enviar los datos comprimidos al servidor.

En el lado del servidor, un enfoque con buffering implicaría acumular todos los datos entrantes de la solicitud HTTP en un buffer. Una vez que se hayan recibido todos los datos, el servidor los descomprimirá en otro buffer que contendrá los datos sin comprimir, que luego se guardarán en el disco.

Si bien esto funciona, un mejor enfoque utiliza streams. Con los streams, el cliente puede comenzar a comprimir y enviar fragmentos de datos tan pronto como se lean del sistema de archivos. De manera similar, el servidor puede descomprimir cada fragmento de datos tan pronto como llegue, eliminando la necesidad de esperar por el archivo completo. Como ventaja adicional, ya hemos visto cómo los streams nos brindan la capacidad de manejar archivos arbitrariamente grandes.

Veamos cómo podemos construir una versión simple de este enfoque basado en streams, comenzando con el servidor:

```javascript
// gzip-receive.js
import { createServer } from 'node:http'
import { createWriteStream } from 'node:fs'
import { createGunzip } from 'node:zlib'
import { basename, join } from 'node:path'

const server = createServer((req, res) => {
  const filename = basename(req.headers['x-filename'])
  const destFilename = join(import.meta.dirname, 'received_files', filename)
  console.log(`File request received: ${filename}`)
  req
    .pipe(createGunzip())
    .pipe(createWriteStream(destFilename))
    .on('finish', () => {
      res.writeHead(201, { 'content-type': 'text/plain' })
      res.end('OK\n')
      console.log(`File saved: ${destFilename}`)
    })
})

server.listen(3000, () => console.log('Listening on http://localhost:3000'))
```

En el ejemplo anterior, estamos configurando un servidor HTTP que escucha las cargas de archivos entrantes, las descomprime y las guarda en el disco. La parte clave de este servidor es la función controladora (la que se pasa a la función `createServer()`), donde entran en juego dos objetos importantes: `req` (la solicitud) y `res` (la respuesta). Estos objetos son ambos streams:
- `req` representa la solicitud entrante del cliente al servidor. En este caso, transporta los datos del archivo comprimido que envía el cliente.
- `res` representa la respuesta saliente del servidor hacia el cliente.

El enfoque aquí está en `req`, que actúa como el stream de origen. El código procesa `req`:
1. Descomprimiéndolo mediante `createGunzip()`.
2. Guardándolo en disco con `createWriteStream()` en un directorio llamado `received_files` (en la misma carpeta que este ejemplo de código).

Las llamadas a `pipe()` vinculan estos pasos, creando un flujo fluido de datos desde la solicitud entrante, a través de la descompresión, hasta el archivo en disco. No te preocupes demasiado por la sintaxis de `pipe()` por ahora; la cubriremos con más detalle más adelante en el capítulo.

Cuando todos los datos se hayan escrito en el disco, se activa el evento `finish`. En este punto, el servidor responde al cliente con un código de estado de 201 (Created) y un simple mensaje "OK", lo que indica que el archivo se ha recibido y guardado correctamente.

Finalmente, el servidor escucha conexiones en el puerto 3000 y se registra un mensaje para confirmar que se está ejecutando.

En nuestra aplicación de servidor, usamos `basename()` para eliminar cualquier ruta del nombre de un archivo recibido (por ejemplo, `basename("/path/to/file")` nos daría `"file"`). Esta es una medida de seguridad importante para garantizar que los archivos se guarden dentro de nuestra carpeta `received_files`. Sin `basename()`, un usuario malintencionado podría crear una solicitud que escape de la carpeta de la aplicación, lo que podría tener graves consecuencias, como poder sobrescribir archivos del sistema e inyectar código malicioso. Por ejemplo, imagina que el nombre de archivo proporcionado fuera algo como `../../../usr/bin/node`. Un atacante podría adivinar una ruta relativa para sobrescribir `/usr/bin/node`, reemplazando el intérprete de Node.js con cualquier archivo ejecutable que desee. Aterrador, ¿verdad? Este tipo de ataque se denomina ataque de salto de directorio (*path traversal attack* o *directory traversal*). Puedes leer más sobre esto aquí: [nodejsdp.link/path-traversal](https://nodejsdp.link/path-traversal).

Ten en cuenta que aquí no estamos siguiendo la forma más convencional de realizar cargas de archivos a través de HTTP. De hecho, generalmente, esta función se implementa utilizando un protocolo un poco más avanzado y estándar que requiere codificar los datos de origen utilizando la especificación `multipart/form-data` ([nodejsdp.link/multipart](https://nodejsdp.link/multipart)). Esta especificación le permite enviar uno o más archivos y sus respectivos nombres de archivo utilizando campos codificados en el cuerpo. En nuestra implementación más simple, el cuerpo de la solicitud no contiene metadatos, sino solo los bytes comprimidos con gzip del archivo original; por lo tanto, debemos especificar el nombre del archivo en otro lugar. Por eso proporcionamos una cabecera personalizada llamada `x-filename`.

Ahora que hemos terminado con el servidor, escribamos el código de cliente correspondiente:

```javascript
// gzip-send.js
import { request } from 'node:http'
import { createGzip } from 'node:zlib'
import { createReadStream } from 'node:fs'
import { basename } from 'node:path'

const filename = process.argv[2]
const serverHost = process.argv[3]

const httpRequestOptions = {
  hostname: serverHost,
  port: 3000,
  path: '/',
  method: 'POST',
  headers: {
    'content-type': 'application/octet-stream',
    'content-encoding': 'gzip',
    'x-filename': basename(filename),
  },
}

const req = request(httpRequestOptions, res => {
  console.log(`Server response: ${res.statusCode}`)
})

createReadStream(filename)
  .pipe(createGzip())
  .pipe(req)
  .on('finish', () => {
    console.log('File successfully sent')
  })
```

En el código anterior, implementamos el lado cliente de nuestro sistema de transferencia de archivos. Su objetivo es leer un archivo del sistema de archivos local, comprimirlo y enviarlo al servidor mediante una solicitud HTTP POST. Así es como funciona:
1. El cliente lee el nombre del archivo (que se enviará) y el nombre de host del servidor (`serverHost`) de los argumentos de la línea de comandos. Estos valores se utilizan luego para configurar el objeto `httpRequestOptions`, que define los detalles de la solicitud HTTP, incluyendo:
   - El nombre de host y el puerto del servidor.
   - La ruta y el método de la solicitud.
   - Las cabeceras, incluida la información sobre el nombre del archivo (`x-filename`), el tipo de contenido y el hecho de que el contenido está comprimido con gzip.
2. La solicitud HTTP real (`req`) que se crea mediante la función `request()`. Este objeto es un stream que representa una solicitud HTTP que va del cliente al servidor.
3. El archivo de origen se lee mediante `createReadStream()`, se comprime con `createGzip()` y luego se envía al servidor conectando mediante `pipe()` el stream resultante a `req`. Esto crea un flujo continuo de datos desde el archivo en el disco, a través de la compresión y finalmente al servidor.
4. Cuando se hayan enviado todos los datos, se activa el evento `finish` en el stream de solicitud. En este punto, se registra un mensaje de confirmación ("File successfully sent").
5. Mientras tanto, la respuesta del servidor se maneja en el callback proporcionado a `request()`. Una vez que el servidor responde, su código de estado se registra en la consola, lo que permite al cliente confirmar que la operación se completó con éxito.

Ahora, para probar la aplicación, iniciemos primero el servidor usando el siguiente comando:

```bash
node gzip-receive.js
```

Luego, podemos iniciar el cliente especificando el archivo a enviar y la dirección del servidor (por ejemplo, `localhost`):

```bash
node gzip-send.js <path to file> localhost
```

Si elegimos un archivo lo suficientemente grande, podemos observar cómo fluyen los datos desde el cliente al servidor. El archivo de destino aparecerá en la carpeta `received_files` antes de que se muestre el mensaje "File successfully sent" en el cliente. Esto se debe a que, a medida que el archivo comprimido se envía por HTTP, el servidor ya lo está descomprimiendo y guardando en el disco.

Sin embargo, todavía no hemos abordado por qué este paradigma, con su flujo continuo de datos, es más eficiente que utilizar una API con buffering. La Figura 6.3 debería hacer que este concepto sea más fácil de comprender:

**Figura 6.3:** Comparación entre buffering y streaming.

Cuando se procesa un archivo, pasa por una serie de pasos secuenciales:
1. `[Cliente]` Leer del sistema de archivos
2. `[Cliente]` Comprimir los datos
3. `[Cliente]` Enviarlo al servidor
4. `[Servidor]` Recibir del cliente
5. `[Servidor]` Descomprimir los datos
6. `[Servidor]` Escribir los datos en el disco

Para completar el procesamiento, tenemos que pasar por cada etapa como en una línea de montaje, en secuencia, hasta el final. En la Figura 6.3, podemos ver que, utilizando una API con buffering, el proceso es completamente secuencial. Para comprimir los datos, primero debemos esperar a que se lea todo el archivo; luego, para enviar los datos, tenemos que esperar a que todo el archivo sea leído y comprimido, y así sucesivamente.

Usando streams, la línea de montaje se pone en marcha tan pronto como recibimos el primer fragmento de datos, sin esperar a que se lea todo el archivo. Pero lo más sorprendente es que cuando el siguiente fragmento de datos está disponible, no es necesario esperar a que se complete el conjunto anterior de tareas; en su lugar, se lanza otra línea de montaje de forma concurrente. Esto funciona a la perfección porque cada tarea que ejecutamos es asíncrona, por lo que Node.js puede ejecutarla de forma concurrente. La única restricción es que se debe preservar el orden en que los fragmentos llegan a cada etapa. La implementación interna de los streams de Node.js se encarga de mantener el orden por nosotros.

Como podemos ver en la Figura 6.3, el resultado de utilizar streams es que todo el proceso lleva menos tiempo, porque no perdemos tiempo esperando a que todos los datos se lean y procesen de una sola vez.

Esta sección podría hacer que parezca que los streams siempre son más rápidos que utilizar un enfoque con buffering. Si bien eso suele ser cierto (como en el ejemplo que acabamos de cubrir), no está garantizado. Los streams están diseñados para la eficiencia de la memoria, no necesariamente para la velocidad. La abstracción que proporcionan puede agregar sobrecarga, lo que podría ralentizar las cosas. Si todos los datos que necesitas caben en la memoria, ya están cargados y no es necesario transferirlos entre procesos o sistemas, procesarlos directamente sin streams probablemente te brindará resultados más rápidos.

#### Composabilidad

El código que hemos visto hasta ahora demuestra cómo se pueden componer los streams mediante el método `pipe()`. Este método nos permite conectar diferentes unidades de procesamiento, cada una responsable de una única funcionalidad, al más puro estilo Node.js. Los streams pueden hacer esto porque comparten una interfaz consistente, lo que los hace compatibles entre sí a nivel de API. El único requisito es que el siguiente stream en la canalización (*pipeline*) admita el tipo de datos producido por el stream anterior (datos binarios u objetos, como exploraremos más adelante en este capítulo).

Para demostrar aún más la composabilidad de los streams de Node.js, intentemos agregar una capa de cifrado a la aplicación gzip-send/gzip-receive que construimos anteriormente. Esto requerirá solo unos pequeños cambios tanto en el cliente como en el servidor.

##### Añadiendo cifrado en el lado del cliente

Comencemos con el cliente:

```javascript
// crypto-gzip-send.js
// ...
import { createCipheriv, randomBytes } from 'node:crypto' // 1
// ...
const secret = Buffer.from(process.argv[4], 'hex') // 2
const iv = randomBytes(16) // 3
// ...
```

Revisemos lo que cambiamos aquí:
1. En primer lugar, importamos el stream Transform `createCipheriv()` y la función `randomBytes()` del módulo `node:crypto`.
2. Obtenemos el secreto de cifrado del servidor desde la línea de comandos. Esperamos que la cadena se pase como una cadena hexadecimal, por lo que leemos este valor y lo cargamos en memoria utilizando un buffer configurado en modo hexadecimal.
3. Finalmente, generamos una secuencia aleatoria de bytes que usaremos como vector de inicialización (*initialization vector*, [nodejsdp.link/iv](https://nodejsdp.link/iv)) para el cifrado de archivos.

Un vector de inicialización (IV) es un poco como barajar una baraja de cartas de manera diferente antes de repartirlas, incluso si siempre estás usando la misma baraja. Al comenzar cada ronda con una mezcla diferente, se vuelve mucho más difícil para alguien que observa tus manos de cerca predecir las cartas que tienes. En criptografía, el IV establece el estado inicial para el cifrado. Por lo general, es aleatorio o único, lo que garantiza que cifrar el mismo mensaje dos veces con la misma clave produzca resultados diferentes. Esto ayuda a evitar que los atacantes identifiquen patrones. Ten en cuenta que el IV es necesario para el descifrado posterior. El destinatario del mensaje debe conocer tanto la clave como el IV para descifrar el mensaje, y solo la clave debe permanecer en secreto (generalmente, el IV se transfiere junto con el mensaje cifrado, mientras que la clave se intercambia de alguna otra forma segura). La analogía de barajar cartas no es perfecta, pero ayuda a ilustrar cómo comenzar con una configuración diferente cada vez puede aumentar significativamente la seguridad.

Ahora, podemos actualizar la porción de código responsable de crear la solicitud HTTP:

```javascript
const httpRequestOptions = {
  hostname: serverHost,
  headers: {
    'content-type': 'application/octet-stream',
    'content-encoding': 'gzip',
    'x-filename': basename(filename),
    'x-initialization-vector': iv.toString('hex') // 1
  }
}
// ...
const req = request(httpRequestOptions, (res) => {
  console.log(`Server response: ${res.statusCode}`)
})

createReadStream(filename)
  .pipe(createGzip())
  .pipe(createCipheriv('aes192', secret, iv)) // 2
  .pipe(req)
// ...
```

Los cambios principales aquí son:
1. Pasamos el vector de inicialización al servidor como una cabecera HTTP.
2. Ciframos los datos, justo después de la fase de Gzip.

Eso es todo para el lado cliente.

##### Añadiendo descifrado en el lado del servidor

Refactoricemos ahora el servidor. Lo primero que debemos hacer es importar algunas funciones de utilidad del módulo central `node:crypto`, que podemos usar para generar una clave de cifrado aleatoria (el secreto):

```javascript
// crypto-gzip-receive.js
// ...
import { createDecipheriv, randomBytes } from 'node:crypto'

const secret = randomBytes(24)
console.log(`Generated secret: ${secret.toString('hex')}`)
```

El secreto generado se imprime en la consola como una cadena hexadecimal para que podamos compartirlo con nuestros clientes.

Ahora, necesitamos actualizar la lógica de recepción de archivos:

```javascript
const server = createServer((req, res) => {
  const filename = basename(req.headers['x-filename'])
  const iv = Buffer.from(
    req.headers['x-initialization-vector'], 'hex') // 1
  const destFilename = join('received_files', filename)
  console.log(`File request received: ${filename}`)
  req
    .pipe(createDecipheriv('aes192', secret, iv)) // 2
    .pipe(createGunzip())
    .pipe(createWriteStream(destFilename))
  // ...
```

Aquí aplicamos dos cambios:
1. Tenemos que leer el vector de inicialización de cifrado enviado por el cliente.
2. El primer paso de nuestra canalización de streaming ahora es responsable de descifrar los datos entrantes utilizando el stream Transform `createDecipheriv` del módulo `crypto`.

Con muy poco esfuerzo (solo unas pocas líneas de código), agregamos una capa de cifrado a nuestra aplicación; simplemente tuvimos que usar algunos streams Transform ya disponibles (`createCipheriv` y `createDecipheriv`) e incluirlos en las canalizaciones de procesamiento de streams para el cliente y el servidor. De manera similar, podemos agregar y combinar otros streams, como si estuviéramos jugando con piezas de LEGO.

La principal ventaja de este enfoque es la reutilización, pero como podemos ver en el código hasta ahora, los streams también permiten un código más limpio y modular. Por estas razones, los streams a menudo se usan no solo para lidiar con E/S pura, sino también para simplificar y modularizar el código.

Ahora que has probado un aperitivo de cómo se siente usar streams, estamos listos para explorar, de una manera más estructurada, los diferentes tipos de streams disponibles en Node.js.

En esta implementación, utilizamos el cifrado como ejemplo para demostrar la composabilidad de los streams. Dado que nuestra comunicación cliente-servidor se basa en el protocolo HTTP, un enfoque más estándar y posiblemente más simple habría sido utilizar HTTPS simplemente cambiando del módulo `node:http` al módulo `node:https`. Independientemente de la implementación que decidas utilizar, asegúrate de que, si estás transfiriendo datos a través de una red, utilices algún tipo de cifrado sólido. ¡Nunca transfieras datos no cifrados a través de una red!

---

### Sección 2: Primeros pasos con los streams

En la sección anterior, aprendimos por qué los streams son tan poderosos, pero también que están en todas partes en Node.js, comenzando por sus módulos centrales. Por ejemplo, hemos visto que el módulo `fs` tiene `createReadStream()` para leer de un archivo y `createWriteStream()` para escribir en un archivo, los objetos de solicitud y respuesta HTTP son esencialmente streams, el módulo `zlib` nos permite comprimir y descomprimir datos mediante una interfaz de streaming y, finalmente, incluso el módulo `crypto` expone algunas primitivas de streaming útiles como `createCipheriv` y `createDecipheriv`.

Ahora que sabemos por qué los streams son tan importantes, demos un paso atrás y comencemos a explorarlos con más detalle.

#### Anatomía de los streams

Cada stream en Node.js es una implementación de una de las cuatro clases abstractas base disponibles en el módulo central `stream`:
- **Readable**
- **Writable**
- **Duplex**
- **Transform**

Cada clase de stream es también una instancia de `EventEmitter`. De hecho, los streams pueden producir varios tipos de eventos, como `end` cuando un stream Readable ha terminado de leer, `finish` cuando un stream Writable ha completado la escritura (ya hemos visto este en algunos de los ejemplos anteriores) o `error` cuando algo sale mal.

Una de las razones por las que los streams son tan flexibles es el hecho de que pueden manejar no solo datos binarios, sino casi cualquier valor de JavaScript. De hecho, admiten dos modos de funcionamiento:
- **Modo binario (*Binary mode*):** Para transmitir datos en forma de fragmentos (*chunks*), como buffers o cadenas.
- **Modo objeto (*Object mode*):** Para transmitir datos como una secuencia de objetos discretos (lo que nos permite utilizar casi cualquier valor de JavaScript).

Estos dos modos de funcionamiento nos permiten utilizar streams no solo para E/S, sino también como una herramienta para componer elegantemente unidades de procesamiento de forma funcional, como veremos más adelante en este capítulo.

Comencemos nuestra inmersión profunda en los streams de Node.js presentando la clase de streams Readable.

#### Readable streams (Streams de lectura)

Un stream Readable representa una fuente de datos. En Node.js, se implementa mediante la clase abstracta `Readable`, que está disponible en el módulo `stream`.

##### Leyendo de un stream

Hay dos enfoques para recibir datos de un stream Readable: **no fluido (o pausado)** (*non-flowing* o *paused*) y **fluido** (*flowing*). Analicemos estos modos con más detalle.

##### El modo non-flowing (pausado)

El modo non-flowing o pausado es el patrón predeterminado para leer de un stream Readable. Implica adjuntar un escuchador al stream para el evento `readable`, que señala la disponibilidad de nuevos datos para leer. Luego, en un bucle, leemos los datos continuamente hasta que se vacía el buffer interno. Esto se puede hacer utilizando el método `read()`, que lee sincrónicamente del buffer interno y devuelve un objeto `Buffer` que representa el fragmento de datos. El método `read()` tiene la siguiente firma:

```javascript
readable.read([size])
```

Con este enfoque, los datos se extraen (*pull*) del stream bajo demanda.

Para mostrar cómo funciona esto, creemos un nuevo módulo llamado `read-stdin.js`, que implementa un programa simple que lee de la entrada estándar (que también es un stream Readable) y repite todo de regreso a la salida estándar:

```javascript
process.stdin
  .on('readable', () => {
    let chunk
    console.log('New data available')
    while ((chunk = process.stdin.read()) !== null) {
      console.log(
        `Chunk read (${chunk.length} bytes): "${chunk.toString()}"`
      )
    }
  })
  .on('end', () => console.log('End of stream'))
```

El método `read()` es una operación síncrona que extrae un fragmento de datos de los buffers internos del stream Readable. El fragmento devuelto es, por defecto, un objeto `Buffer` si el stream está funcionando en modo binario.

En un stream Readable que funciona en modo binario, podemos leer cadenas en lugar de buffers llamando a `setEncoding(encoding)` en el stream y proporcionando un formato de codificación válido (por ejemplo, `utf8`). Este enfoque se recomienda al transmitir datos de texto UTF-8, ya que el stream manejará adecuadamente los caracteres multibyte, realizando el buffering necesario para asegurarse de que ningún carácter termine dividido en fragmentos separados. En otras palabras, cada fragmento producido por el stream será una secuencia de bytes UTF-8 válida.

Ten en cuenta que puedes llamar a `setEncoding()` tantas veces como desees en un stream Readable, incluso después de haber comenzado a consumir los datos del stream. La codificación se cambiará dinámicamente en el siguiente fragmento disponible. Los streams son intrínsecamente binarios; la codificación es solo una vista sobre los datos binarios emitidos por el stream.

Los datos se leen únicamente desde dentro del escuchador `readable`, que se invoca tan pronto como hay nuevos datos disponibles. El método `read()` devuelve `null` cuando no hay más datos disponibles en los buffers internos; en tal caso, tenemos que esperar a que se dispare otro evento `readable`, indicándonos que podemos leer de nuevo, o esperar al evento `end` que señala el final del stream. Cuando un stream está funcionando en modo binario, también podemos especificar que estamos interesados en leer una cantidad específica de datos pasando un valor de `size` al método `read()`. Esto es particularmente útil al implementar protocolos de red o al analizar formatos de datos específicos.

Ahora estamos listos para ejecutar el módulo `read-stdin.js` y experimentar con él. Escribamos algunos caracteres en la consola y luego presionemos Enter para ver los datos reflejados en la salida estándar. Para terminar el stream y, por tanto, generar un evento `end` limpio, debemos insertar un carácter EOF (*end-of-file*) (usando `Ctrl + Z` en Windows o `Ctrl + D` en Linux y macOS).

También podemos intentar conectar nuestro programa con otros procesos. Esto es posible utilizando el operador de tubería (`|`), que redirige la salida estándar de un programa a la entrada estándar de otro. Por ejemplo, podemos ejecutar un comando como el siguiente:

```bash
cat <path to a file> | node read-stdin.js
```

Esta es una demostración asombrosa de cómo el paradigma de streaming es una interfaz universal que permite que nuestros programas se comuniquen, independientemente del lenguaje en el que estén escritos.

##### Modo flowing (fluyente)

Otra forma de leer de un stream es adjuntando un escuchador al evento `data`. Esto cambiará el stream al modo flowing, donde los datos no se extraen mediante `read()`, sino que se envían (*push*) al escuchador `data` tan pronto como llegan. Por ejemplo, la aplicación `read-stdin.js` que creamos anteriormente se verá así usando el modo flowing:

```javascript
process.stdin
  .on('data', (chunk) => {
    console.log('New data available')
    console.log(
      `Chunk read (${chunk.length} bytes): "${chunk.toString()}"`
    )
  })
  .on('end', () => console.log('End of stream'))
```

El modo flowing ofrece menos flexibilidad para controlar el flujo de datos en comparación con el modo non-flowing. El modo de funcionamiento predeterminado para los streams es non-flowing, por lo que para habilitar el modo flowing, es necesario adjuntar un escuchador al evento `data` o invocar explícitamente el método `resume()`. Para evitar temporalmente que el stream emita eventos `data`, podemos invocar el método `pause()`, lo que hace que los datos entrantes se almacenen en caché en el buffer interno. Llamar a `pause()` volverá a cambiar el stream al modo non-flowing.

##### Iteradores asíncronos (*Async iterators*)

Los streams Readable también son iteradores asíncronos; por lo tanto, podríamos reescribir nuestro ejemplo `read-stdin.js` de la siguiente manera:

```javascript
for await (const chunk of process.stdin) {
  console.log('New data available')
  console.log(`Chunk read (${chunk.length} bytes): "${chunk.toString()}"`)
}
console.log('End of stream')
```

Analizaremos los iteradores asíncronos con mayor detalle en el [Capítulo 9](https://subscription.packtpub.com/book/web-development/9781803238944/9), Patrones de diseño de comportamiento, así que no te preocupes demasiado por la sintaxis del ejemplo anterior por ahora. Lo importante que debes saber es que también puedes consumir datos de un stream Readable utilizando esta conveniente sintaxis `for await ... of`.

##### Implementación de streams Readable

Ahora que sabemos cómo leer de un stream, el siguiente paso es aprender a implementar un nuevo stream Readable personalizado. Para hacer esto, es necesario crear una nueva clase heredando el prototipo `Readable` del módulo `stream`. El stream concreto debe proporcionar una implementación del método `_read()`, que tiene la siguiente firma:

```javascript
readable._read(size)
```

Los componentes internos de la clase `Readable` llamarán al método `_read()`, que, a su vez, comenzará a llenar el buffer interno utilizando `push()`:

```javascript
readable.push(chunk)
```

Ten en cuenta que `read()` es un método llamado por los consumidores del stream, mientras que `_read()` es un método que debe implementar una subclase de stream y nunca debe llamarse directamente. El prefijo de guion bajo se utiliza para indicar que el método no debe considerarse público y no debe llamarse directamente.

Para demostrar cómo implementar nuevos streams Readable, podemos intentar implementar un stream que genere cadenas aleatorias. Creemos un nuevo módulo que contenga el código de nuestro generador de cadenas aleatorias:

```javascript
// random-stream.js
import { Readable } from 'node:stream'
import Chance from 'chance' // v1.1.12

const chance = new Chance()

export class RandomStream extends Readable {
  constructor(options) {
    super(options)
    this.emittedBytes = 0
  }

  _read(size) {
    const chunk = chance.string({ length: size }) // 1
    this.push(chunk, 'utf8') // 2
    this.emittedBytes += chunk.length
    if (chance.bool({ likelihood: 5 })) { // 3
      this.push(null)
    }
  }
}
```

Para este ejemplo, estamos utilizando un módulo de terceros de npm llamado `chance` ([nodejsdp.link/chance](https://nodejsdp.link/chance)), que es una biblioteca para generar todo tipo de valores aleatorios, desde números hasta cadenas y oraciones completas.

Ten en cuenta que `chance` no es criptográficamente seguro, lo que significa que se puede utilizar para pruebas o simulaciones, pero no para generar tokens, contraseñas u otros fines relacionados con la seguridad.

Comenzamos definiendo una nueva clase llamada `RandomStream`, que especifica `Readable` como su padre. En el constructor de esta clase, tenemos que invocar `super(options)`, que llamará al constructor de la clase padre (`Readable`), inicializando el estado interno del stream.

Si tienes un constructor que solo invoca `super(options)`, puedes eliminarlo, ya que la clase heredará el constructor padre de forma predeterminada. Solo ten cuidado de recordar llamar a `super(options)` cada vez que necesites escribir un constructor personalizado.

Los posibles parámetros que se pueden pasar a través del objeto `options` incluyen los siguientes:
- El argumento `encoding`, que se utiliza para convertir buffers en cadenas (por defecto es `null`).
- Un indicador para habilitar el modo objeto (`objectMode`, por defecto es `false`).
- El límite superior de los datos almacenados en el buffer interno, después del cual no se debe realizar más lectura desde la fuente (`highWaterMark`, por defecto es `16KB`).

Dentro del constructor, inicializamos una variable de instancia: `emittedBytes`. Usaremos esta variable para realizar un seguimiento de cuántos bytes se han emitido hasta ahora desde el stream. Esto será útil para la depuración, pero no es un requisito al crear streams Readable.

Bien, ahora analicemos la implementación del método `_read()`:
1. El método genera una cadena aleatoria de longitud igual a `size` utilizando `chance`.
2. Empuja la cadena al buffer interno. Ten en cuenta que, dado que estamos empujando cadenas, también debemos especificar la codificación, `utf8` (esto no es necesario si el fragmento es simplemente un `Buffer` binario).
3. Termina el stream aleatoriamente, con una probabilidad del 5 por ciento, empujando `null` al buffer interno para indicar una situación de EOF o, en otras palabras, el final del stream. Este es solo un detalle de implementación que estamos adoptando para forzar que el stream termine eventualmente. Sin esta condición, nuestro stream estaría produciendo datos aleatorios indefinidamente.

> **Streams Readable finitos frente a infinitos:**
> Depende de nosotros determinar si un stream Readable debe terminar. Puedes señalar el final de un stream invocando `this.push(null)` en el método `_read()`. Algunos streams son naturalmente finitos. Por ejemplo, al leer de un archivo, el stream terminará una vez que se hayan leído todos los bytes porque el archivo tiene un tamaño definido. En otros casos, podríamos crear streams que proporcionen datos indefinidamente. Por ejemplo, un stream legible podría entregar lecturas continuas de temperatura de un sensor o una transmisión de video en vivo de una cámara de seguridad. Estos streams seguirán produciendo datos mientras la fuente permanezca activa y no ocurran errores de comunicación.

Ten en cuenta que el argumento `size` en la función `_read()` es un parámetro consultivo. Es bueno respetarlo y empujar solo la cantidad de datos solicitada por el invocador, aunque no es obligatorio hacerlo.

Cuando invocamos `push()`, debemos verificar si devuelve `false`. Cuando eso sucede, significa que el buffer interno del stream receptor ha alcanzado el límite de `highWaterMark` y debemos dejar de agregarle más datos. Esto se llama **contrapresión** (*backpressure*), y lo discutiremos con más detalle en la siguiente sección de este capítulo. Por ahora, solo ten en cuenta que esta implementación no es perfecta porque no maneja la contrapresión.

Eso es todo para `RandomStream`; ahora estamos listos para usarlo. Veamos cómo instanciar un objeto `RandomStream` y extraer algunos datos de él (usando el modo flowing):

```javascript
// index.js
import { RandomStream } from './random-stream.js'

const randomStream = new RandomStream()
randomStream
  .on('data', chunk => {
    console.log(`Chunk received (${chunk.length} bytes): ${chunk.toString()}`)
  })
  .on('end', () => {
    console.log(`Produced ${randomStream.emittedBytes} bytes of random data`)
  })
```

Ahora todo está listo para que probemos nuestro nuevo stream personalizado. Simplemente ejecuta el módulo `index.js` como de costumbre y observa un conjunto aleatorio de cadenas fluir en la pantalla.

##### Construcción simplificada

Para streams personalizados simples, podemos evitar la creación de una clase personalizada utilizando el enfoque de construcción simplificada del stream Readable. Con este enfoque, solo necesitamos invocar `new Readable(options)` y pasar un método llamado `read()` en el conjunto de opciones. El método `read()` aquí tiene exactamente la misma semántica que el método `_read()` que vimos en el enfoque de extensión de clase. Reescribamos `RandomStream` utilizando el enfoque de constructor simplificado:

```javascript
// simplified-construction.js
import { Readable } from 'node:stream'
import Chance from 'chance' // v1.1.12

const chance = new Chance()
let emittedBytes = 0

const randomStream = new Readable({
  read(size) {
    const chunk = chance.string({ length: size })
    this.push(chunk, 'utf8')
    emittedBytes += chunk.length
    if (chance.bool({ likelihood: 5 })) {
      this.push(null)
    }
  },
})

// now you can read data from the randomStream instance directly ...
```

Este enfoque puede ser particularmente útil cuando no necesitas administrar un estado complicado y te permite aprovechar una sintaxis más sucinta. En el ejemplo anterior, creamos una única instancia de nuestro stream personalizado. Si queremos adoptar el enfoque de constructor simplificado, pero necesitamos crear múltiples instancias del stream personalizado, podemos envolver la lógica de inicialización en una función factoría que podemos invocar varias veces para crear esas instancias.

Analizaremos el patrón Factory y otros patrones de diseño creacionales en el [Capítulo 7](https://subscription.packtpub.com/book/web-development/9781803238944/7), Patrones de diseño creacionales.

##### Streams Readable a partir de iterables

Puedes crear fácilmente instancias de streams Readable a partir de arrays u otros objetos iterables (es decir, generadores, iteradores e iteradores asíncronos) utilizando el ayudante `Readable.from()`.

Para acostumbrarnos a este ayudante, veamos un ejemplo simple donde convertimos datos de un array en un stream Readable:

```javascript
import { Readable } from 'node:stream'

const mountains = [
  { name: 'Everest', height: 8848 },
  { name: 'K2', height: 8611 },
  { name: 'Kangchenjunga', height: 8586 },
  { name: 'Lhotse', height: 8516 },
  { name: 'Makalu', height: 8481 }
]

const mountainsStream = Readable.from(mountains)
mountainsStream.on('data', (mountain) => {
  console.log(`${mountain.name.padStart(14)}\t${mountain.height}m`)
})
```

Como podemos ver en este código, el método `Readable.from()` es bastante simple de usar: el primer argumento es una instancia iterable (en nuestro caso, el array `mountains`). `Readable.from()` acepta un argumento opcional adicional que se puede usar para especificar opciones de stream como `objectMode`.

Ten en cuenta que no tuvimos que establecer explícitamente `objectMode` en `true`. Por defecto, `Readable.from()` establecerá `objectMode` en `true`, a menos que se opte explícitamente por no hacerlo configurándolo en `false`. Las opciones del stream se pueden pasar como segundo argumento a la función.

La ejecución del código anterior producirá la siguiente salida:

```text
       Everest	8848m
            K2	8611m
 Kangchenjunga	8586m
        Lhotse	8516m
        Makalu	8481m
```

Debes evitar instanciar arrays grandes en memoria. Imagina si, en el ejemplo anterior, quisiéramos enumerar todas las montañas del mundo. Hay alrededor de 1 millón de montañas, por lo que si fuéramos a cargarlas todas en un array por adelantado, asignaríamos una cantidad bastante significativa de memoria. Incluso si luego consumimos los datos en el array a través de un stream Readable, todos los datos ya han sido precargados, por lo que efectivamente estamos anulando la eficiencia de memoria de los streams. Siempre es preferible cargar y consumir los datos en fragmentos, y podrías hacerlo utilizando streams nativos como `fs.createReadStream`, creando un stream personalizado o utilizando `Readable.from` con iterables perezosos como generadores, iteradores o iteradores asíncronos. Veremos algunos ejemplos de este último enfoque en el [Capítulo 9](https://subscription.packtpub.com/book/web-development/9781803238944/9), Patrones de diseño de comportamiento.

#### Writable streams (Streams de escritura)

Un stream Writable representa un destino de datos. Imagina, por ejemplo, un archivo en el sistema de archivos, una tabla de base de datos, un socket, la salida estándar o la interfaz de error estándar. En Node.js, este tipo de abstracciones se pueden implementar mediante la clase abstracta `Writable`, que está disponible en el módulo `stream`.

##### Escribiendo en un stream

Enviar datos a través de un stream Writable es un asunto sencillo; todo lo que tenemos que hacer es usar el método `write()`, que tiene la siguiente firma:

```javascript
writable.write(chunk, [encoding], [callback])
```

El argumento `encoding` es opcional y se puede especificar si `chunk` es una cadena (por defecto es `utf8` y se ignora si `chunk` es un buffer). La función `callback`, por otro lado, se llama cuando el fragmento se vuelca en el recurso subyacente y también es opcional.

Para indicar que no se escribirán más datos en el stream, tenemos que usar el método `end()`:

```javascript
writable.end([chunk], [encoding], [callback])
```

Podemos proporcionar un fragmento final de datos a través del método `end()`; en este caso, la función callback es equivalente a registrar un escuchador para el evento `finish`, que se activa cuando todos los datos escritos en el stream se han volcado en el recurso subyacente.

Ahora, mostremos cómo funciona esto creando un pequeño servidor HTTP que genere una secuencia aleatoria de cadenas:

```javascript
// entropy-server.js
import { createServer } from 'node:http'
import Chance from 'chance' // v1.1.12

const chance = new Chance()

const server = createServer((_req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' }) // 1
  do { // 2
    res.write(`${chance.string()}\n`)
  } while (chance.bool({ likelihood: 95 }))
  res.end('\n\n') // 3
  res.on('finish', () => console.log('All data sent')) // 4
})

server.listen(3000, () => {
  console.log('listening on http://localhost:3000')
})
```

El servidor HTTP que creamos escribe en el objeto `res`, que es una instancia de `http.ServerResponse` y también un stream Writable. Lo que sucede se explica a continuación:
1. Primero escribimos el encabezado de la respuesta HTTP. Ten en cuenta que `writeHead()` no es parte de la interfaz `Writable`; de hecho, es un método auxiliar expuesto por la clase `http.ServerResponse` y es específico del protocolo HTTP. Este método escribe en el stream una cabecera HTTP formateada correctamente, que contendrá el código de estado 200 y una cabecera que especifica el tipo de contenido del cuerpo de la respuesta que estamos a punto de transmitir.
2. Iniciamos un bucle que termina con una probabilidad del 5% (indicamos a `chance.bool()` que devuelva `true` el 95% de las veces). Dentro del bucle, escribimos una cadena aleatoria en el stream. Ten en cuenta que usamos un bucle `do ... while` aquí porque queremos asegurarnos de producir al menos una cadena aleatoria.
3. Una vez que salimos del bucle, llamamos a `end()` en el stream, indicando que no se escribirán más datos. Además, proporcionamos una cadena final que contiene dos caracteres de nueva línea para escribirlos en el stream antes de finalizarlo.
4. Finalmente, registramos un escuchador para el evento `finish`, que se disparará cuando todos los datos se hayan volcado en el socket subyacente.

Para probar el servidor, podemos abrir un navegador en la dirección `http://localhost:3000` o usar `curl` desde la terminal de la siguiente manera:

```bash
curl -i --raw localhost:3000
```

En este punto, el servidor debería comenzar a enviar cadenas aleatorias al cliente HTTP que elijas. Si estás utilizando un navegador web, ten en cuenta que el hardware moderno puede procesar las cosas muy rápidamente y que algunos navegadores pueden almacenar los datos en un buffer, por lo que el comportamiento de streaming puede no ser evidente.

Al utilizar los indicadores `-i --raw` en el comando `curl`, podemos echar un vistazo a muchos detalles del protocolo HTTP. Específicamente, `-i` incluye cabeceras de respuesta en la salida. Esto nos permite ver los datos exactos transferidos en la parte de la cabecera de la respuesta cuando invocamos `res.writeHead()`. El módulo `node:http` simplifica el trabajo con el protocolo HTTP formateando automáticamente las cabeceras de respuesta y aplica valores predeterminados sensatos, como agregar cabeceras estándar como `Connection: keep-alive` y `Transfer-Encoding: chunked`. Esta última cabecera es particularmente interesante. Informa al cliente cómo interpretar el cuerpo de la respuesta entrante. La codificación fragmentada (*chunked encoding*) es especialmente útil al enviar grandes cantidades de datos cuyo tamaño total no se conoce hasta que la solicitud se haya procesado por completo. Esto lo convierte en un ajuste perfecto para los streams Writable de Node.js. Con la codificación fragmentada, se omite la cabecera `Content-Length`. En su lugar, cada fragmento comienza con su longitud en formato hexadecimal, seguido de `\r\n`, los datos del fragmento y otro `\r\n`. El stream termina con un fragmento de terminación, que es idéntico a un fragmento regular excepto que su longitud es cero. En nuestro código, no necesitamos manejar estos detalles manualmente. El stream Writable `ServerResponse` proporcionado por el módulo `node:http` se encarga de codificar los fragmentos correctamente por nosotros. Simplemente proporcionamos fragmentos llamando a `write()` o `end()` en el stream de respuesta, y el stream se encarga del resto. Esta es una de las fortalezas de los streams de Node.js: abstraen detalles de implementación complejos, haciéndolos fáciles de trabajar. Si deseas obtener más información sobre la codificación fragmentada, consulta: [nodejsdp.link/transfer-encoding](https://nodejsdp.link/transfer-encoding). Al utilizar la opción `--raw`, que desactiva toda la decodificación HTTP interna de contenido o codificaciones de transferencia, podemos ver que estos terminadores de fragmentos (`\r\n`) están presentes en los datos recibidos del servidor.

##### Contrapresión (*Backpressure*)

Los streams de datos de Node.js, como los líquidos en un sistema de tuberías, pueden sufrir cuellos de botella cuando los datos se escriben más rápido de lo que el stream puede manejar. Para gestionar esto, los datos entrantes se almacenan en un buffer en la memoria. Sin embargo, sin retroalimentación del stream al escritor, el buffer podría seguir creciendo, lo que podría llevar a un uso excesivo de memoria.

Los streams de Node.js están diseñados para mantener un uso de memoria constante y predecible, incluso durante grandes transferencias de datos. Los streams Writable incluyen un mecanismo de señalización incorporado para alertar a la aplicación cuando el buffer interno ha acumulado demasiados datos. Esta señal indica que es mejor pausar y esperar a que los datos almacenados en el buffer se vuelquen en el destino del stream antes de enviar más datos. El método `writable.write()` devuelve `false` una vez que el tamaño del buffer excede el límite de `highWaterMark`.

En los streams Writable, el valor de `highWaterMark` establece el tamaño máximo del buffer en bytes. Cuando se excede este límite, `write()` devuelve `false`, señalando a la aplicación que detenga la escritura. Una vez que se borra el buffer, se emite un evento `drain`, indicando que es seguro reanudar la escritura. Este proceso se conoce como **contrapresión** (*backpressure*).

La contrapresión es un mecanismo consultivo. Incluso si `write()` devuelve `false`, podríamos ignorar esta señal y continuar escribiendo, haciendo que el buffer crezca indefinidamente. El stream no se bloqueará automáticamente cuando se alcance el umbral de `highWaterMark`; por lo tanto, se recomienda ser siempre consciente y respetar la contrapresión.

El mecanismo descrito en esta sección es igualmente aplicable a los streams Readable. De hecho, la contrapresión también existe en los streams Readable, y se activa cuando el método `push()`, que se invoca dentro de `_read()`, devuelve `false`. Sin embargo, ese es un problema específico de los implementadores de streams, por lo que generalmente tenemos que lidiar con él con menos frecuencia.

Podemos demostrar rápidamente cómo tener en cuenta la contrapresión de un stream Writable modificando el módulo `entropy-server.js` que creamos anteriormente:

```javascript
// ...
const CHUNK_SIZE = 16 * 1024 - 1
const chance = new Chance()

const server = createServer((_req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' })
  let backPressureCount = 0
  let bytesSent = 0
  function generateMore() { // 1
    do {
      const randomChunk = chance.string({ length: CHUNK_SIZE }) // 2
      const shouldContinue = res.write(`${randomChunk}\n`) // 3
      bytesSent += CHUNK_SIZE
      if (!shouldContinue) { // 4
        console.warn(`back-pressure x${++backPressureCount}`)
        return res.once('drain', generateMore)
      }
    } while (chance.bool({ likelihood: 95 }))
    res.end('\n\n')
  }

  generateMore()
  res.on('finish', () => console.log(`Sent ${bytesSent} bytes`))
})
// ...
```

Los pasos más importantes del código anterior se pueden resumir de la siguiente manera:
1. Envolvimos la lógica principal de generación de datos en una función llamada `generateMore()`.
2. Para aumentar las posibilidades de recibir algo de contrapresión, aumentamos el tamaño del fragmento de datos a 16 KB menos 1 byte, lo cual está muy cerca del límite predeterminado de `highWaterMark`.
3. Después de escribir un fragmento de datos, verificamos el valor de retorno de `res.write()`. Si recibimos `false`, significa que el buffer interno está lleno y debemos dejar de enviar más datos.
4. Cuando esto sucede, salimos de la función y registramos otro ciclo de escrituras utilizando `generateMore()` para cuando se emita el evento `drain`.

Si ahora intentamos ejecutar el servidor nuevamente y luego generamos una solicitud de cliente con `curl` (o con un navegador), existe una alta probabilidad de que haya algo de contrapresión, ya que el servidor produce datos a una velocidad muy alta, más rápido de lo que el socket subyacente puede manejar. Este ejemplo también imprime cuántos eventos de contrapresión ocurren y cuántos datos se transfieren para cada solicitud. Te animamos a probar diferentes solicitudes, revisar los registros e intentar comprender qué sucede debajo del capó.

##### Implementación de streams Writable

Podemos implementar un nuevo stream Writable heredando la clase `Writable` y proporcionando una implementación para el método `_write()`. Intentemos hacerlo de inmediato mientras analizamos los detalles en el camino.

Construyamos un stream Writable que reciba objetos en el siguiente formato:

```javascript
{
  path: <path to a file>
  content: <string or buffer>
}
```

Para cada uno de estos objetos, nuestro stream guardará la propiedad `content` en un archivo creado en la ruta indicada. Podemos ver inmediatamente que las entradas de nuestro stream son objetos, y no cadenas o buffers. Esto significa que nuestro stream debe funcionar en modo objeto, lo que nos da una gran excusa para explorar también el modo objeto con este ejemplo:

```javascript
// to-file-stream.js
import { Writable } from 'node:stream'
import { promises as fs } from 'node:fs'
import { dirname } from 'node:path'
import { mkdirp } from 'mkdirp' // v3.0.1

export class ToFileStream extends Writable {
  constructor(options) {
    super({ ...options, objectMode: true })
  }

  _write(chunk, _encoding, cb) {
    mkdirp(dirname(chunk.path))
      .then(() => fs.writeFile(chunk.path, chunk.content))
      .then(() => cb())
      .catch(cb)
  }
}
```

Creamos una nueva clase para nuestro nuevo stream, que extiende `Writable` del módulo `stream`.

Tuvimos que invocar al constructor padre para inicializar su estado interno; también necesitábamos asegurarnos de que el objeto `options` especifique que el stream funciona en modo objeto (`objectMode: true`). Otras opciones aceptadas por `Writable` son las siguientes:
- `highWaterMark` (el valor predeterminado es 16 KB): Controla el límite de contrapresión.
- `decodeStrings` (por defecto es `true`): Habilita la decodificación automática de cadenas en buffers binarios antes de pasarlos al método `_write()`. Esta opción se ignora en modo objeto.

Finalmente, proporcionamos una implementación para el método `_write()`. Como puedes ver, el método acepta un fragmento de datos y una codificación (que tiene sentido solo si estamos en modo binario y la opción de stream `decodeStrings` está configurada en `false`). Además, el método acepta una función callback (`cb`), que debe invocarse cuando se completa la operación; no es necesario pasar el resultado de la operación pero, si es necesario, aún podemos pasar un error que hará que el stream emita un evento `error`.

Ahora, para probar el stream que acabamos de construir, podemos crear un nuevo módulo y realizar algunas operaciones de escritura contra el stream:

```javascript
// index.js
import { join } from 'node:path'
import { ToFileStream } from './to-file-stream.js'

const tfs = new ToFileStream()
const outDir = join(import.meta.dirname, 'files')

tfs.write({ path: join(outDir, 'file1.txt'), content: 'Hello' })
tfs.write({ path: join(outDir, 'file2.txt'), content: 'Node.js' })
tfs.write({ path: join(outDir, 'file3.txt'), content: 'streams' })
tfs.end(() => console.log('All files created'))
```

Aquí creamos y usamos nuestro primer stream Writable personalizado. Ejecuta el nuevo módulo como de costumbre y verifica su salida. Verás que después de la ejecución, se crearán tres archivos nuevos dentro de una nueva carpeta llamada `files`.

##### Construcción simplificada

Como vimos para los streams Readable, los streams Writable también ofrecen un mecanismo de construcción simplificado. Si fuéramos a reescribir `ToFileStream` usando la construcción simplificada para streams Writable, se vería así:

```javascript
// ...
const tfs = new Writable({
  objectMode: true,
  write(chunk, _encoding, cb) {
    mkdirp(dirname(chunk.path))
      .then(() => fs.writeFile(chunk.path, chunk.content))
      .then(() => cb())
      .catch(cb)
  },
})
// ...
```

Con este enfoque, simplemente estamos usando el constructor `Writable` y pasando una función `write()` que implementa la lógica personalizada de nuestra instancia de `Writable`. Ten en cuenta que con este enfoque, la función `write()` no tiene un guion bajo en el nombre. También podemos pasar otras opciones de construcción como `objectMode`.

#### Streams Duplex

Un stream Duplex es un stream que es a la vez Readable y Writable. Es útil cuando queremos describir una entidad que es tanto una fuente de datos como un destino de datos, como los sockets de red, por ejemplo. Los streams Duplex heredan los métodos de `stream.Readable` y `stream.Writable`, por lo que esto no es nada nuevo para nosotros. Esto significa que podemos leer (`read()`) o escribir (`write()`) datos, o escuchar tanto eventos `readable` como `drain`.

Para crear un stream Duplex personalizado, tenemos que proporcionar una implementación tanto para `_read()` como para `_write()`. El objeto `options` pasado al constructor `Duplex()` se reenvía internamente a los constructores `Readable` y `Writable`. Las opciones son las mismas que ya discutimos en las secciones anteriores, con la adición de una nueva llamada `allowHalfOpen` (por defecto es `true`) que, si se establece en `false`, hará que ambas partes (Readable y Writable) del stream finalicen si solo una de ellas lo hace.

Si necesitamos tener un stream Duplex funcionando en modo objeto en un lado y en modo binario en el otro, podemos usar las opciones `readableObjectMode` y `writableObjectMode` de forma independiente.

#### Streams Transform

Los streams Transform son un tipo especial de stream Duplex que están diseñados específicamente para manejar transformaciones de datos. Solo para darte algunos ejemplos concretos, las funciones `zlib.createGzip()` y `crypto.createCipheriv()` que analizamos al comienzo de este capítulo crean streams Transform para compresión y cifrado, respectivamente.

En un stream Duplex simple, no existe una relación inmediata entre los datos leídos del stream y los datos escritos en él (al menos, el stream es agnóstico a dicha relación). Piensa en un socket TCP, que simplemente envía y recibe datos hacia y desde el par remoto; el socket no es consciente de ninguna relación entre la entrada y la salida. La Figura 6.4 ilustra el flujo de datos en un stream Duplex:

**Figura 6.4:** Representación esquemática de un stream Duplex.

Por otro lado, los streams Transform aplican algún tipo de transformación a cada fragmento de datos que reciben de su lado Writable, y luego hacen que los datos transformados estén disponibles en su lado Readable. La Figura 6.5 muestra cómo fluyen los datos en un stream Transform:

**Figura 6.5:** Representación esquemática de un stream Transform.

Volviendo a nuestro ejemplo de compresión, un stream Transform se puede visualizar de la siguiente manera: cuando se escriben datos sin comprimir en el stream, su implementación interna comprime los datos y los almacena en un buffer interno. Cuando los datos se leen del otro extremo, se recupera la versión comprimida de los datos. Así es como la transformación ocurre sobre la marcha: los datos entran, se transforman y luego salen.

Desde la perspectiva del usuario, la interfaz programática de un stream Transform es exactamente como la de un stream Duplex. Sin embargo, cuando queremos implementar un nuevo stream Duplex, tenemos que proporcionar los métodos `_read()` y `_write()`, mientras que para implementar un nuevo stream Transform, tenemos que completar otro par de métodos: `_transform()` y `_flush()`.

Veamos cómo crear un nuevo stream Transform con un ejemplo.

##### Implementación de streams Transform

Implementemos un stream Transform que reemplace todas las apariciones de una cadena determinada:

```javascript
// replaceStream.js
import { Transform } from 'node:stream'

export class ReplaceStream extends Transform {
  constructor(searchStr, replaceStr, options) {
    super({ ...options })
    this.searchStr = searchStr
    this.replaceStr = replaceStr
    this.tail = ''
  }

  _transform(chunk, _encoding, cb) {
    const pieces = (this.tail + chunk).split(this.searchStr) // 1
    const lastPiece = pieces[pieces.length - 1] // 2
    const tailLen = this.searchStr.length - 1
    this.tail = lastPiece.slice(-tailLen)
    pieces[pieces.length - 1] = lastPiece.slice(0, -tailLen)
    this.push(pieces.join(this.replaceStr)) // 3
    cb()
  }

  _flush(cb) {
    this.push(this.tail)
    cb()
  }
}
```

En este ejemplo, creamos una nueva clase que extiende la clase base `Transform`. El constructor de la clase acepta tres argumentos: `searchStr`, `replaceStr` y `options`. Como puedes imaginar, nos permiten definir el texto que coincide y la cadena que se utilizará como reemplazo, además de un objeto `options` para la configuración avanzada del stream Transform subyacente. También inicializamos una variable interna `tail`, que será utilizada más adelante por el método `_transform()`.

Ahora, analicemos el método `_transform()`, que es el núcleo de nuestra nueva clase. El método `_transform()` tiene prácticamente la misma firma que el método `_write()` del stream Writable, pero en lugar de escribir datos en un recurso subyacente, los empuja al buffer de lectura interno usando `this.push()`, exactamente como lo haríamos en el método `_read()` de un stream Readable. Esto muestra cómo están conectados los dos lados de un stream Transform.

El método `_transform()` de `ReplaceStream` implementa el núcleo de nuestro algoritmo. Buscar y reemplazar una cadena en un buffer es una tarea fácil; sin embargo, es una historia totalmente diferente cuando los datos se transmiten en streaming y las posibles coincidencias pueden distribuirse en múltiples fragmentos. El procedimiento seguido por el código se puede explicar de la siguiente manera:
1. Nuestro algoritmo divide los datos en memoria (los datos de la cola `tail` y el fragmento actual) utilizando `searchStr` como separador.
2. Luego, toma el último elemento del array generado por la operación y extrae los últimos `searchString.length - 1` caracteres. El resultado se guarda en la variable `tail` y se antepondrá al siguiente fragmento de datos.
3. Finalmente, todas las piezas resultantes de `split()` se unen usando `replaceStr` como separador y se empujan al buffer interno.

Cuando el stream finaliza, es posible que todavía tengamos algo de contenido en la variable `tail` que no se haya empujado al buffer interno. Para eso sirve exactamente el método `_flush()`; se invoca justo antes de que finalice el stream, y aquí es donde tenemos una última oportunidad de finalizar el stream o empujar cualquier dato restante antes de finalizar completamente el stream.

El método `_flush()` solo recibe un callback, que debemos asegurarnos de invocar cuando se completen todas las operaciones, lo que provocará la terminación del stream. Con esto, hemos completado nuestra clase `ReplaceStream`.

##### ¿Por qué es necesaria la variable tail?

Los streams procesan datos en fragmentos, y estos fragmentos no siempre se alinean con los límites de la cadena de búsqueda de destino. Por ejemplo, si la cadena que intentamos hacer coincidir se divide en dos fragmentos, la operación `split()` en un solo fragmento no la detectará, lo que podría hacer que parte de la coincidencia pase desapercibida. La variable `tail` garantiza que la última porción de un fragmento —potencialmente parte de una coincidencia— se conserve y se concatene con el siguiente fragmento. De esta manera, el stream puede manejar adecuadamente las coincidencias que abarcan los límites de los fragmentos, evitando reemplazos incorrectos o la pérdida total de coincidencias.

En los streams Transform, no es raro que la lógica implique almacenar en un buffer datos de múltiples fragmentos antes de que haya suficiente información para realizar la transformación. Por ejemplo, la criptografía a menudo funciona en bloques de datos de tamaño fijo. Si un fragmento no proporciona suficientes datos para formar un bloque completo, el stream Transform acumula múltiples fragmentos hasta que tiene suficiente para procesar la transformación. Este comportamiento de buffering garantiza que las transformaciones sean precisas y consistentes, incluso cuando los datos de entrada llegan en tamaños impredecibles.

Esto también debería aclarar por qué existe el método `_flush()`. Se proporciona para manejar cualquier dato restante que aún esté almacenado en el buffer del stream Transform cuando el escritor haya terminado de escribir. Esto garantiza que los datos sobrantes (como la cola `tail` en este ejemplo) se procesen y se emitan, evitando una salida incompleta o perdida.

Ahora es el momento de probar el nuevo stream. Creemos un script que escriba algunos datos en el stream y luego lea el resultado transformado:

```javascript
// index.js
import { ReplaceStream } from './replace-stream.js'

const replaceStream = new ReplaceStream('World', 'Node.js')
replaceStream.on('data', chunk => process.stdout.write(chunk.toString()))

replaceStream.write('Hello W')
replaceStream.write('orld!')
replaceStream.end('\n')
```

Para hacerle la vida un poco más difícil a nuestro stream, distribuimos el término de búsqueda (que es `World`) en dos fragmentos diferentes; luego, utilizando el modo flowing, leemos del mismo stream, registrando cada fragmento transformado. La ejecución del programa anterior debería producir la siguiente salida:

```text
Hello Node.js!
```

##### Construcción simplificada

Como era de esperar, incluso los streams Transform admiten la construcción simplificada. A estas alturas, deberíamos haber desarrollado un instinto sobre cómo podría verse esta API, así que pongámonos manos a la obra de inmediato y reescribamos el ejemplo anterior utilizando este enfoque:

```javascript
// simplified-construction.js
// ...
const searchStr = 'World'
const replaceStr = 'Node.js'
let tail = ''

const replaceStream = new Transform({
  defaultEncoding: 'utf8',
  transform(chunk, _encoding, cb) {
    const pieces = (tail + chunk).split(searchStr)
    const lastPiece = pieces[pieces.length - 1]
    const tailLen = searchStr.length - 1
    tail = lastPiece.slice(-tailLen)
    pieces[pieces.length - 1] = lastPiece.slice(0, -tailLen)
    this.push(pieces.join(replaceStr))
    cb()
  },
  flush(cb) {
    this.push(tail)
    cb()
  },
})

// now write to replaceStream ...
```

Como era de esperar, la construcción simplificada funciona instanciando directamente un nuevo objeto `Transform` y pasando nuestra lógica de transformación específica a través de las funciones `transform()` y `flush()` directamente mediante el objeto de opciones. Ten en cuenta que `transform()` y `flush()` no tienen un guion bajo antepuesto aquí.

##### Filtrado y agregación de datos con streams Transform

Como comentamos anteriormente, los streams Transform son una gran herramienta para crear canalizaciones de transformación de datos. En un ejemplo anterior, mostramos cómo usar un stream Transform para reemplazar palabras en un stream de texto. También mencionamos otros casos de uso, como la compresión y el cifrado. Pero los streams Transform no se limitan a esos ejemplos. A menudo se utilizan para tareas como filtrar y agregar datos.

Para que esto sea más concreto, imagina que una empresa de Fortune 500 nos pide que analicemos un archivo grande que contiene todos sus datos de ventas para 2024. El archivo, `data.csv`, es un informe de ventas en formato CSV y quieren que calculemos el beneficio total de las ventas realizadas en Italia. Claro, podríamos usar una aplicación de hoja de cálculo para hacer esto, pero ¿dónde está la diversión en eso?

En su lugar, usemos streams de Node.js. Los streams son adecuados para este tipo de tareas porque pueden procesar grandes conjuntos de datos de forma incremental, sin cargar todo en la memoria. Esto los hace eficientes y escalables. Además, crear una solución con streams sienta las bases para la automatización; perfecto si necesitas generar informes similares con regularidad o procesar otros conjuntos de datos grandes en el futuro.

Imaginemos que los datos de ventas que se almacenan en el archivo CSV contienen tres campos por línea: tipo de artículo, país de venta y beneficio. Por lo tanto, dicho archivo podría verse así:

```csv
type,country,profit
Household,Namibia,597290.92
Baby Food,Iceland,808579.10
Meat,Russia,277305.60
Meat,Italy,413270.00
Cereal,Malta,174965.25
Meat,Indonesia,145402.40
Household,Italy,728880.54
[... many more lines]
```

Ahora queda claro que debemos encontrar todos los registros que tengan "Italy" en el campo `country` y, en el proceso, sumar el valor de `profit` de las líneas coincidentes en un solo número.

Para procesar un archivo CSV en streaming, podemos usar el excelente módulo de terceros `csv-parse` ([nodejsdp.link/csv-parse](https://nodejsdp.link/csv-parse)).

Si asumimos por un momento que ya hemos implementado nuestros streams personalizados para filtrar y agregar los datos, una posible solución a esta tarea podría verse así:

```javascript
// index.js
import { createReadStream } from 'node:fs'
import { Parser } from 'csv-parse' // v5.6.0
import { FilterByCountry } from './filter-by-country.js'
import { SumProfit } from './sum-profit.js'

const csvParser = new Parser({ columns: true })

createReadStream('data.csv.gz') // 1
  .pipe(csvParser) // 2
  .pipe(new FilterByCountry('Italy')) // 3
  .pipe(new SumProfit()) // 4
  .pipe(process.stdout) // 5
```

La canalización de streaming propuesta aquí consta de cinco pasos:
1. Leemos el archivo CSV de origen como un stream.
2. Usamos la biblioteca `csv-parse` para analizar cada línea del documento como un registro CSV. Para cada línea, este stream emitirá un objeto que contiene las propiedades `type`, `country` y `profit`. Con la opción `columns: true`, la biblioteca leerá los nombres de las columnas disponibles en la primera fila del archivo CSV.
3. Filtramos todos los registros por país, reteniendo solo los registros que coinciden con el país "Italy". Todos los registros que no coinciden con "Italy" se descartan, lo que significa que no se pasarán a los otros pasos de la canalización. Ten en cuenta que este es uno de los streams Transform personalizados que tenemos que implementar.
4. Para cada registro, acumulamos el beneficio. Este stream emitirá eventualmente una única cadena, que representa el valor del beneficio total para los productos vendidos en Italia. Este valor será emitido por el stream solo cuando todos los datos del archivo original hayan sido completamente procesados. Ten en cuenta que este es el segundo stream Transform personalizado que tenemos que implementar para completar este proyecto.
5. Finalmente, los datos emitidos desde el paso anterior se muestran en la salida estándar.

Ahora, implementemos el stream `FilterByCountry`:

```javascript
// filter-by-country.js
import { Transform } from 'node:stream'

export class FilterByCountry extends Transform {
  constructor(country, options = {}) {
    options.objectMode = true
    super(options)
    this.country = country
  }

  _transform(record, _enc, cb) {
    if (record.country === this.country) {
      this.push(record)
    }
    cb()
  }
}
```

`FilterByCountry` es un stream Transform personalizado. Podemos ver que el constructor acepta un argumento llamado `country`, que nos permite especificar el nombre del país que queremos hacer coincidir. En el constructor, también configuramos el stream para que se ejecute en `objectMode` porque sabemos que se usará para procesar objetos (registros provenientes del archivo CSV).

En el método `_transform`, verificamos si el país del registro actual coincide con el país especificado en el momento de la construcción. Si coincide, pasamos el registro a la siguiente etapa de la canalización llamando a `this.push()`. Independientemente de si el registro coincide o no, debemos invocar `cb()` para indicar que el registro actual se ha procesado con éxito y que el stream está listo para recibir otro registro.

> **Patrón (Filtro Transform):**
> Invoca `this.push()` de forma condicional para permitir que solo algunos datos lleguen a la siguiente etapa de la canalización.

Finalmente, implementemos el filtro `SumProfit`:

```javascript
// sum-profit.js
import { Transform } from 'node:stream'

export class SumProfit extends Transform {
  constructor(options = {}) {
    options.objectMode = true
    super(options)
    this.total = 0
  }

  _transform(record, _enc, cb) {
    this.total += Number.parseFloat(record.profit)
    cb()
  }

  _flush(cb) {
    this.push(this.total.toString())
    cb()
  }
}
```

Este stream también debe ejecutarse en `objectMode`, porque recibirá objetos que representan registros del archivo CSV. Ten en cuenta que, en el constructor, también inicializamos una variable de instancia llamada `total` y establecemos su valor en 0.

En el método `_transform()`, procesamos cada registro y usamos el valor de beneficio actual para aumentar el total. Es importante tener en cuenta que esta vez no estamos llamando a `this.push()`. Esto significa que no se emite ningún valor mientras los datos fluyen a través del stream. Sin embargo, aún debemos llamar a `cb()` para indicar que el registro actual ha sido procesado y que el stream está listo para recibir otro.

Para emitir el resultado final cuando se hayan procesado todos los datos, tenemos que definir un comportamiento de vaciado personalizado utilizando el método `_flush()`. Aquí finalmente llamamos a `this.push()` para emitir una representación en cadena del valor total resultante. Recuerda que `_flush()` se invoca automáticamente antes de que se cierre el stream.

> **Patrón (Agregación en streaming):**
> Usa `_transform()` para procesar los datos y acumular el resultado parcial, luego llama a `this.push()` solo en el método `_flush()` para emitir el resultado cuando se hayan procesado todos los datos.

Esto completa nuestro ejemplo. Ahora puedes tomar el archivo CSV del repositorio de código y ejecutar este programa para ver cuál es el beneficio total para Italia. ¡No es ninguna sorpresa que vaya a ser mucho dinero, ya que estamos hablando de los beneficios de una empresa de Fortune 500!

Podrías combinar el filtrado y la agregación en un único stream Transform. Si bien este enfoque puede ser menos reutilizable, puede ofrecer una ligera mejora en el rendimiento, ya que se transfieren menos datos entre los pasos de la canalización de streams. Si estás preparado para el desafío, ¡intenta implementar esto como un ejercicio!

La biblioteca de streams de Node.js incluye un conjunto de métodos auxiliares para streams Readable (experimentales al momento de escribir este artículo). Entre ellos se encuentran `Readable.map()` y `Readable.reduce()`, que podrían resolver el problema que acabamos de explorar de una manera más concisa y simplificada. Nos sumergiremos en los ayudantes de streams Readable más adelante en este capítulo.

#### Streams PassThrough

Hay un quinto tipo de stream que vale la pena mencionar: **PassThrough**. Este tipo de stream es un tipo especial de stream Transform que emite cada fragmento de datos sin aplicar ninguna transformación.

PassThrough es posiblemente el tipo de stream más subestimado, pero hay varias circunstancias en las que puede ser una herramienta muy valiosa en nuestro cinturón de herramientas. Por ejemplo, los streams PassThrough pueden ser útiles para la **observabilidad** o para implementar patrones de **conexión tardía** (*late piping*) y **streams perezosos** (*lazy streams*).

##### Observabilidad

Si queremos observar cuántos datos fluyen a través de uno o más streams, podríamos hacerlo adjuntando un escuchador de eventos `data` a una instancia de PassThrough y luego conectando mediante `pipe()` esta instancia en un punto determinado de una canalización de streams. Veamos un ejemplo simplificado para poder apreciar este concepto:

```javascript
import { PassThrough } from 'node:stream'

let bytesWritten = 0
const monitor = new PassThrough()
monitor.on('data', chunk => {
  bytesWritten += chunk.length
})
monitor.on('finish', () => {
  console.log(`${bytesWritten} bytes written`)
})

monitor.write('Hello!')
monitor.end()
```

En este ejemplo, estamos creando una nueva instancia de `PassThrough` y usando el evento `data` para contar cuántos bytes fluyen a través del stream. También usamos el evento `finish` para volcar la cantidad total en la consola. Finalmente, escribimos algunos datos directamente en el stream usando `write()` y `end()`. Este es solo un ejemplo ilustrativo; en un escenario más realista, conectaríamos nuestra instancia de monitor mediante `pipe()` en un punto determinado de una canalización de streams. Por ejemplo, si quisiéramos monitorizar cuántos bytes se escriben en el disco en nuestro primer ejemplo de compresión de archivos de este capítulo, podríamos lograrlo fácilmente haciendo algo como esto:

```javascript
createReadStream(filename)
  .pipe(createGzip())
  .pipe(monitor)
  .pipe(createWriteStream(`${filename}.gz`))
```

La belleza de este enfoque es que no tuvimos que tocar ninguno de los otros streams existentes en la canalización, por lo que si necesitamos observar otras partes de la canalización (por ejemplo, imagina que queremos saber el número de bytes de los datos sin comprimir), podemos mover `monitor` con muy poco esfuerzo. Incluso podríamos tener múltiples streams PassThrough para monitorizar diferentes partes de una canalización al mismo tiempo.

Ten en cuenta que podrías implementar una versión alternativa del stream monitor utilizando un stream Transform personalizado en su lugar. En tal caso, tendrías que asegurarte de que los fragmentos recibidos se empujen sin ninguna modificación o retraso, lo cual es algo que un stream PassThrough haría automáticamente por ti. Ambos enfoques son igualmente válidos, así que elige el que te parezca más natural.

##### Conexión tardía (*Late piping*)

En algunas circunstancias, es posible que tengamos que utilizar APIs que acepten un stream como parámetro de entrada. En general, esto no es un gran problema porque ya sabemos cómo crear y usar streams. Sin embargo, puede volverse un poco más complicado si los datos que queremos leer o escribir a través del stream solo están disponibles después de haber llamado a la API en cuestión.

Para visualizar este escenario en términos más concretos, imaginemos que tenemos que utilizar una API que nos proporciona la siguiente función para cargar un archivo en un servicio de almacenamiento de datos:

```javascript
function upload (filename, contentStream) {
  // ...
}
```

Esta función es efectivamente una versión simplificada de lo que comúnmente está disponible en el SDK de servicios de almacenamiento de archivos como Amazon Simple Storage Service (S3) o el servicio Azure Blob Storage. A menudo, esas bibliotecas proporcionarán al usuario una función más flexible que puede recibir los datos de contenido en diferentes formatos (por ejemplo, una cadena, un buffer o un stream Readable).

Ahora bien, si queremos subir un archivo desde el sistema de archivos, esta es una operación trivial y podemos hacer algo como esto:

```javascript
import { createReadStream } from 'fs'
upload('a-picture.jpg', createReadStream('/path/to/a-picture.jpg'))
```

Pero, ¿qué sucede si queremos realizar algún procesamiento en el stream del archivo antes de la carga? Por ejemplo, digamos que queremos comprimir o cifrar los datos. Además, ¿qué sucede si necesitamos realizar esta transformación de forma asíncrona después de que se haya llamado a la función de carga?

En tales casos, podemos proporcionar un stream PassThrough a la función `upload()`, que actuará efectivamente como un marcador de posición (*placeholder*). La implementación interna de `upload()` intentará inmediatamente consumir datos de él, pero no habrá datos disponibles en el stream hasta que realmente escribamos en él. Además, el stream no se considerará completo hasta que lo cerremos, por lo que la función `upload()` tendrá que esperar a que los datos fluyan a través de la instancia de PassThrough para iniciar la carga.

Veamos un posible script de línea de comandos que utiliza este enfoque para cargar un archivo desde el sistema de archivos y lo comprime mediante la compresión Brotli. Asumiremos que la función de terceros `upload()` se proporciona en un archivo llamado `upload.js`.

```javascript
// upload-cli.js
import { createReadStream } from 'node:fs'
import { createBrotliCompress } from 'node:zlib'
import { PassThrough } from 'node:stream'
import { basename } from 'node:path'
import { upload } from './upload.js'

const filepath = process.argv[2] // 1
const filename = basename(filepath)
const contentStream = new PassThrough() // 2

upload(`${filename}.br`, contentStream) // 3
  .then(response => {
    console.log(`Server response: ${response.data}`)
  })
  .catch(err => {
    console.error(err)
    process.exit(1)
  })

createReadStream(filepath) // 4
  .pipe(createBrotliCompress())
  .pipe(contentStream)
```

En el repositorio de este libro encontrarás una implementación completa de este ejemplo que te permite cargar archivos a un servidor HTTP que puedes ejecutar localmente.

Revisemos lo que sucede en el ejemplo anterior:
1. Obtenemos la ruta al archivo que queremos cargar desde el primer argumento de la línea de comandos y usamos `basename` para extrapolar el nombre del archivo de la ruta dada.
2. Creamos un marcador de posición para nuestro stream de contenido como una instancia de `PassThrough`.
3. Ahora invocamos la función de carga pasando nuestro nombre de archivo (con el sufijo `.br` agregado, lo que indica que está usando la compresión Brotli) y el stream de contenido de marcador de posición.
4. Finalmente, creamos una canalización encadenando un stream Readable del sistema de archivos, un stream Transform de compresión Brotli y, por último, nuestro stream de contenido como destino.

Cuando se ejecuta este código, la carga se iniciará tan pronto como invoquemos la función `upload()` (posiblemente estableciendo una conexión con el servidor remoto), pero los datos comenzarán a fluir solo más tarde, cuando se inicialice nuestra canalización. Ten en cuenta que nuestra canalización también cerrará `contentStream` cuando se complete el procesamiento, lo que indicará a la función `upload()` que todo el contenido se ha consumido por completo.

> **Patrón:**
> Usa un stream PassThrough cuando necesites proporcionar un marcador de posición para datos que se leerán o escribirán en el futuro.

También podemos usar este patrón para transformar la interfaz de la función `upload()`. En lugar de aceptar un stream Readable como entrada, podemos hacer que devuelva un stream Writable, que luego se puede usar para proporcionar los datos que queremos cargar:

```javascript
function createUploadStream (filename) {
  // ...
  // returns a writable stream that can be used to upload data
}
```

Si tuviéramos la tarea de implementar esta función, podríamos lograrlo de una manera muy elegante usando una instancia de PassThrough, como en el siguiente ejemplo de implementación:

```javascript
function createUploadStream (filename) {
  const connector = new PassThrough()
  upload(filename, connector)
  return connector
}
```

En el código anterior, estamos utilizando un stream PassThrough como conector. Este stream se convierte en una abstracción perfecta para un caso en el que el consumidor de la biblioteca puede escribir datos en cualquier etapa posterior.

La función `createUploadStream()` se puede utilizar de la siguiente manera:

```javascript
const upload = createUploadStream('a-file.txt')
upload.write('Hello World')
upload.end()
```

El repositorio de este libro también contiene un ejemplo de carga HTTP que adopta este patrón alternativo.

##### Streams perezosos (*Lazy streams*)

A veces, necesitamos crear una gran cantidad de streams al mismo tiempo, por ejemplo, para pasarlos a una función para su posterior procesamiento. Un ejemplo típico es cuando se usa `archiver` ([nodejsdp.link/archiver](https://nodejsdp.link/archiver)), un paquete para crear archivos comprimidos como TAR y ZIP. El paquete archiver te permite crear un archivo a partir de un conjunto de streams, que representan los archivos a agregar. El problema es que si queremos pasar una gran cantidad de streams, como por ejemplo de archivos en el sistema de archivos, probablemente obtendríamos un error `EMFILE, too many open files`. Esto se debe a que funciones como `createReadStream()` del módulo `fs` abrirán un descriptor de archivo cada vez que se crea un nuevo stream, incluso antes de que comiences a leer de esos streams.

En términos más genéricos, la creación de una instancia de stream puede inicializar operaciones costosas de inmediato (por ejemplo, abrir un archivo o un socket, inicializar una conexión a una base de datos, etc.), incluso antes de que comencemos a usar dicho stream. Esto podría no ser deseable si estás creando una gran cantidad de instancias de stream para su posterior consumo.

En estos casos, es posible que desees retrasar la inicialización costosa hasta que realmente necesites consumir datos del stream.

Es posible lograr esto mediante el uso de una biblioteca como `lazystream` ([nodejsdp.link/lazystream](https://nodejsdp.link/lazystream)). Esta biblioteca te permite crear proxies para instancias de stream reales, donde la creación de la instancia de stream se pospone hasta que alguna porción de código comienza a consumir datos del proxy.

En el siguiente ejemplo, `lazystream` nos permite crear un stream Readable perezoso para el archivo especial de Unix `/dev/urandom`:

```javascript
import lazystream from 'lazystream'

const lazyURandom = new lazystream.Readable(function (options) {
  return fs.createReadStream('/dev/urandom')
})
```

La función que pasamos como parámetro a `new lazystream.Readable()` es efectivamente una función factoría que genera el stream proxied cuando es necesario.

Entre bastidores, `lazystream` se implementa utilizando un stream PassThrough que, solo cuando su método `_read()` se invoca por primera vez, crea la instancia proxied invocando la función factoría y conecta mediante `pipe()` el stream generado en el propio PassThrough. El código que consume el stream es totalmente agnóstico del proxy que está ocurriendo aquí y consumirá los datos como si fluyeran directamente desde el stream PassThrough. `lazystream` implementa una utilidad similar para crear también un stream Writable perezoso.

Crear streams Readable y Writable perezosos desde cero podría ser un ejercicio interesante que te dejamos a ti. Si te quedas atascado, echa un vistazo al código fuente de `lazystream` para inspirarte sobre cómo implementar este patrón.

En la siguiente sección, analizaremos el método `.pipe()` con mayor detalle y también veremos otras formas de conectar diferentes streams para formar una canalización de procesamiento.

---

### Sección 3: Conexión de streams mediante pipes

El concepto de tuberías de Unix (*pipes*) fue inventado por Douglas McIlroy. Esto permitió que la salida de un programa se conectara a la entrada del siguiente. Echa un vistazo al siguiente comando:

```bash
echo Hello World! | sed s/World/Node.js/g
```

En el comando anterior, `echo` escribirá `Hello World!` en su salida estándar, que luego se redirige a la entrada estándar del comando `sed` (gracias al operador de tubería `|`). Luego, `sed` reemplaza cualquier aparición de `World` con `Node.js` e imprime el resultado en su salida estándar (que, esta vez, es la consola).

De manera similar, los streams de Node.js se pueden conectar utilizando el método `pipe()` del objeto de stream Readable, que tiene la siguiente interfaz:

```javascript
readable.pipe(writable, [options])
```

Ya hemos utilizado el método `pipe()` en algunos ejemplos, pero finalmente profundicemos en lo que hace por nosotros bajo el capó.

De forma muy intuitiva, el método `pipe()` toma los datos que se emiten desde el stream readable y los bombea hacia el stream writable proporcionado. Además, el stream writable finaliza automáticamente cuando el stream readable emite un evento `end` (a menos que especifiquemos `{end: false}` como opciones). El método `pipe()` devuelve el stream writable pasado en el primer argumento, lo que nos permite crear invocaciones encadenadas si dicho stream también es Readable (como un stream Duplex o Transform).

Conectar dos streams mediante una tubería creará succión, lo que permite que los datos fluyan automáticamente hacia el stream writable, por lo que no es necesario llamar a `read()` o `write()`, pero lo más importante es que ya no es necesario controlar la contrapresión, porque se gestiona automáticamente.

Para proporcionar un ejemplo rápido, podemos crear un nuevo módulo que tome un stream de texto de la entrada estándar, aplique la transformación de reemplazo analizada anteriormente cuando construimos nuestro `ReplaceStream` personalizado y luego envíe los datos de regreso a la salida estándar:

```javascript
// replace.js
import { ReplaceStream } from './replace-stream.js'

process.stdin
  .pipe(new ReplaceStream(process.argv[2], process.argv[3]))
  .pipe(process.stdout)
```

El programa anterior canaliza los datos que provienen de la entrada estándar hacia una instancia de `ReplaceStream` y luego de regreso a la salida estándar. Ahora, para probar esta pequeña aplicación, podemos aprovechar una tubería de Unix para redirigir algunos datos a su entrada estándar, como se muestra en el siguiente ejemplo:

```bash
echo Hello World! | node replace.js World Node.js
```

Esto debería producir la siguiente salida:

```text
Hello Node.js!
```

Este sencillo ejemplo demuestra que los streams (y en particular, los streams de texto) son una interfaz universal y que las tuberías son la forma de componer e interconectar todas estas interfaces de forma casi mágica.

#### Pipes y manejo de errores

El método `pipe()` es muy poderoso, pero hay un problema importante: **los eventos de error no se propagan automáticamente a través de la canalización cuando se utiliza `pipe()`**. Toma, por ejemplo, este fragmento de código:

```javascript
stream1
  .pipe(stream2)
  .on('error', () => {})
```

En la canalización anterior, capturaremos solo los errores provenientes de `stream2`, que es el stream al que le adjuntamos el escuchador. Esto significa que, si queremos capturar cualquier error generado por `stream1`, tenemos que adjuntarle otro escuchador de errores directamente, lo que hará que nuestro ejemplo se vea así:

```javascript
stream1
  .on('error', () => {})
  .pipe(stream2)
  .on('error', () => {})
```

Esto claramente no es ideal, especialmente cuando se trata de canalizaciones con una cantidad significativa de pasos. Para empeorar las cosas, en caso de error, el stream que falla solo se desconecta (*unpiped*) de la canalización. El stream defectuoso no se destruye adecuadamente, lo que podría dejar recursos colgando (por ejemplo, descriptores de archivo, conexiones, etc.) y provocar fugas de memoria. Una implementación más robusta (aunque poco elegante) del fragmento anterior podría verse así:

```javascript
function handleError (err) {
  console.error(err)
  stream1.destroy()
  stream2.destroy()
}

stream1
  .on('error', handleError)
  .pipe(stream2)
  .on('error', handleError)
```

En este ejemplo, registramos un controlador para el evento `error` tanto para `stream1` como para `stream2`. Cuando ocurre un error, se invoca nuestra función `handleError()` y podemos registrar el error y destruir cada stream en la canalización. Esto nos permite garantizar que todos los recursos asignados se liberen adecuadamente y que el error se maneje de forma elegante.

#### Mejor manejo de errores con pipeline()

Manejar errores manualmente en una canalización no solo es engorroso, sino también propenso a errores, ¡algo que debemos evitar si podemos!

Afortunadamente, el paquete central `node:stream` nos ofrece una excelente función de utilidad que puede hacer que la construcción de canalizaciones sea una práctica mucho más segura y agradable: la función auxiliar `pipeline()`.

En pocas palabras, puedes usar la función `pipeline()` de la siguiente manera:

```javascript
pipeline(stream1, stream2, stream3, ... , cb)
```

El último argumento es un callback opcional que se llamará cuando finalice el stream. Si finaliza debido a un error, el callback se invocará con el error dado como primer argumento.

Si prefieres evitar los callbacks y utilizar una Promise, existe una alternativa basada en promesas en el paquete `node:stream/promises`:

```javascript
pipeline(stream1, stream2, stream3, ...) // returns a promise
```

Esta alternativa devuelve una Promise que se resolverá cuando se complete la canalización o se rechazará en caso de error.

Ambos ayudantes conectan mediante `pipe()` cada stream pasado en la lista de argumentos con el siguiente. Para cada stream, también registrarán escuchadores de `error` y `close` adecuados. De esta manera, todos los streams se destruyen correctamente cuando la canalización se completa con éxito o cuando se interrumpe por un error.

Para practicar con estos ayudantes, escribamos un script de línea de comandos simple que implemente la siguiente canalización:
1. Lee un stream de datos Gzip de la entrada estándar
2. Descomprime los datos
3. Pasa todo el texto a mayúsculas
4. Comprime con Gzip los datos resultantes
5. Envía los datos de vuelta a la salida estándar

```javascript
// uppercasify-gzipped.js
import { createGzip, createGunzip } from 'node:zlib' // 1
import { Transform } from 'node:stream'
import { pipeline } from 'node:stream/promises'

const uppercasify = new Transform({ // 2
  transform(chunk, _enc, cb) {
    this.push(chunk.toString().toUpperCase())
    cb()
  },
})

await pipeline( // 3
  process.stdin,
  createGunzip(),
  uppercasify,
  createGzip(),
  process.stdout
)
```

En este ejemplo:
1. Estamos importando las dependencias necesarias de los módulos `zlib`, `stream` y `stream/promises`.
2. Creamos un stream Transform simple que pone en mayúsculas cada fragmento.
3. Definimos nuestra canalización, donde listamos todas las instancias de stream en orden. Ten en cuenta que usamos `await` para esperar a que se complete la canalización. En este ejemplo, esto no es obligatorio porque no hacemos nada después de que se complete la canalización, pero es una buena práctica tener esto ya que podríamos decidir hacer evolucionar nuestro script en el futuro, o podríamos querer agregar un `try...catch` alrededor de esta expresión para manejar posibles errores.

La canalización se iniciará automáticamente consumiendo datos de la entrada estándar y produciendo datos para la salida estándar.

Podríamos probar nuestro script con el siguiente comando:

```bash
echo 'Hello World!' | gzip | node uppercasify-gzipped.js | gunzip
```

Esto debería producir la siguiente salida:

```text
HELLO WORLD!
```

Si intentamos eliminar el paso `gzip` de la secuencia de comandos anterior, nuestro script fallará con un error no detectado. Este error lo genera el stream creado con la función `createGunzip()`, que es responsable de descomprimir los datos. Si los datos no están realmente comprimidos con gzip, el algoritmo de descompresión no podrá procesar los datos y fallará. En tal caso, `pipeline()` se encargará de limpiar después del error y destruirá todos los streams en la canalización.

Ahora que hemos adquirido un conocimiento sólido de los streams de Node.js, estamos listos para pasar a algunos patrones de streams más complejos, como el flujo de control y los patrones de tuberías avanzados.

---

### Sección 4: Patrones de control de flujo asíncrono con streams

Al revisar los ejemplos que hemos presentado hasta ahora, debería quedar claro que los streams pueden ser útiles no solo para manejar E/S, sino también como un patrón de programación elegante que se puede utilizar para procesar cualquier tipo de datos. Pero las ventajas no terminan en su simple apariencia; los streams también se pueden aprovechar para convertir el "flujo de control asíncrono" en "control de flujo", como veremos en esta sección.

#### Ejecución secuencial

Por defecto, los streams manejarán los datos en secuencia. Por ejemplo, la función `_transform()` de un stream Transform nunca se invocará con el siguiente fragmento de datos hasta que la invocación anterior se complete llamando a `callback()`. Esta es una propiedad importante de los streams, crucial para procesar cada fragmento en el orden correcto, pero también se puede explotar para convertir los streams en una elegante alternativa a los patrones tradicionales de flujo de control.

Veamos un poco de código para aclarar a qué nos referimos. Trabajaremos en un ejemplo para demostrar cómo podemos usar streams para ejecutar tareas asíncronas en secuencia. Creemos una función que concatene un conjunto de archivos recibidos como entrada, asegurándonos de respetar el orden en que se proporcionan. Creemos un nuevo módulo llamado `concat-files.js` y definamos su contenido de la siguiente manera:

```javascript
import { createReadStream, createWriteStream } from 'node:fs'
import { Readable, Transform } from 'node:stream'

export function concatFiles(dest, files) {
  return new Promise((resolve, reject) => {
    const destStream = createWriteStream(dest)
    Readable.from(files) // 1
      .pipe(
        new Transform({ // 2
          objectMode: true,
          transform(filename, _enc, done) {
            const src = createReadStream(filename)
            src.pipe(destStream, { end: false })
            // same as ((err) => done(err)) // propagates the error
            src.on('error', done)
            // same as (() => done()) // propagates correct completion
            src.on('end', done) // 3
          },
        })
      )
      .on('error', err => {
        destStream.end()
        reject(err)
      })
      .on('finish', () => { // 4
        destStream.end()
        resolve()
      })
  })
}
```

La función anterior implementa una iteración secuencial sobre el array `files` transformándolo en un stream. El algoritmo se puede explicar de la siguiente manera:
1. Primero, usamos `Readable.from()` para crear un stream Readable a partir del array `files`. Este stream opera en modo objeto (la configuración predeterminada para streams creados con `Readable.from()`) y emitirá nombres de archivo: cada fragmento es una cadena que indica la ruta a un archivo. El orden de los fragmentos respeta el orden de los archivos en el array `files`.
2. A continuación, creamos un stream Transform personalizado para manejar cada archivo en la secuencia. Dado que estamos recibiendo cadenas, establecemos la opción `objectMode` en `true`. En nuestra lógica de transformación, para cada archivo creamos un stream Readable para leer el contenido del archivo y conectarlo mediante `pipe()` a `destStream` (un stream Writable para el archivo de destino). Nos aseguramos de no cerrar `destStream` después de que el archivo de origen termine de leerse especificando `{ end: false }` en las opciones de `pipe()`.
3. Cuando todo el contenido del archivo de origen se haya canalizado a `destStream`, invocamos la función `done` para comunicar la finalización del procesamiento actual, lo cual es necesario para desencadenar el procesamiento del siguiente archivo.
4. Cuando se hayan procesado todos los archivos, se activa el evento `finish`; finalmente podemos finalizar `destStream` e invocar la función `resolve()` de `concatFiles()`, que señala la finalización de toda la operación.

Ahora podemos intentar usar el pequeño módulo que acabamos de crear:

```javascript
// concat.js
import { concatFiles } from './concat-files.js'

try {
  await concatFiles(process.argv[2], process.argv.slice(3))
} catch (err) {
  console.error(err)
  process.exit(1)
}

console.log('All files concatenated successfully')
```

Ahora podemos ejecutar el programa anterior pasando el archivo de destino como primer argumento de la línea de comandos, seguido de una lista de archivos para concatenar; por ejemplo:

```bash
node concat.js all-together.txt file1.txt file2.txt
```

Esto debería crear un nuevo archivo llamado `all-together.txt` que contendrá, en orden, el contenido de `file1.txt` y `file2.txt`.

Con la función `concatFiles()`, pudimos obtener una iteración secuencial asíncrona utilizando únicamente streams. Esta es una solución elegante y compacta que enriquece nuestro cinturón de herramientas, junto con las técnicas que ya exploramos en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks, y el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con Promesas y Async/Await.

> **Patrón:**
> Usa un stream, o una combinación de streams, para iterar fácilmente sobre un conjunto de tareas asíncronas en secuencia.

En la siguiente sección, descubriremos cómo utilizar los streams de Node.js para implementar la ejecución de tareas concurrentes no ordenadas.

#### Ejecución concurrente no ordenada

Acabamos de ver que los streams procesan fragmentos de datos en secuencia, pero a veces esto puede ser un cuello de botella, ya que no aprovecharíamos al máximo la concurrencia de Node.js. Si tenemos que ejecutar una operación asíncrona lenta para cada fragmento de datos, puede ser ventajoso hacer que la ejecución sea concurrente y acelerar el proceso general. Por supuesto, este patrón solo se puede aplicar si no existe una relación causal entre cada fragmento de datos, lo que puede ocurrir con frecuencia en los streams de objetos, pero muy raramente en los streams binarios.

> [!CAUTION]
> Los streams concurrentes no ordenados no se pueden usar cuando el orden en el que se procesan los datos es importante.

Para hacer que la ejecución de un stream Transform sea concurrente, podemos aplicar los mismos patrones que aprendimos en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks, pero con algunas adaptaciones para que funcionen con streams. Veamos cómo funciona esto.

##### Implementación de un stream concurrente no ordenado

Demostremos inmediatamente cómo implementar un stream concurrente no ordenado con un ejemplo. Creemos un módulo llamado `concurrent-stream.js` y definamos un stream Transform genérico que ejecute una función de transformación dada de forma concurrente:

```javascript
import { Transform } from 'node:stream'

export class ConcurrentStream extends Transform {
  constructor(userTransform, opts) { // 1
    super({ objectMode: true, ...opts })
    this.userTransform = userTransform
    this.running = 0
    this.terminateCb = null
  }

  _transform(chunk, enc, done) { // 2
    this.running++
    this.userTransform(
      chunk,
      enc,
      this.push.bind(this),
      this._onComplete.bind(this)
    )
    done()
  }

  _flush(done) { // 3
    if (this.running > 0) {
      this.terminateCb = done
    } else {
      done()
    }
  }

  _onComplete(err) { // 4
    this.running--
    if (err) {
      return this.emit('error', err)
    }
    if (this.running === 0) {
      this.terminateCb?.()
    }
  }
}
```

Analicemos esta nueva clase paso a paso:
1. Como puedes ver, el constructor acepta una función `userTransform()`, que luego se guarda como una variable de instancia. Esta función implementará la lógica de transformación que debe ejecutarse para cada objeto que fluya a través del stream. En este constructor, invocamos al constructor padre para inicializar el estado interno del stream y habilitamos el modo objeto de forma predeterminada.
2. A continuación está el método `_transform()`. En este método, ejecutamos la función `userTransform()` y luego incrementamos el recuento de tareas en ejecución. Finalmente, notificamos al stream Transform que el paso de transformación actual está completo invocando `done()`. El truco para activar el procesamiento de otro elemento simultáneamente es exactamente este: no estamos esperando a que se complete la función `userTransform()` antes de invocar a `done()`; en su lugar, lo hacemos inmediatamente. Por otro lado, proporcionamos un callback especial a `userTransform()`, que es el método `this._onComplete()`. Esto nos permite recibir una notificación cuando se completa la ejecución de `userTransform()`.
3. El método `_flush()` se invoca justo antes de que termine el stream, por lo que si todavía hay tareas en ejecución, podemos poner en espera la emisión del evento `finish` no invocando el callback `done()` inmediatamente. En su lugar, lo asignamos a la variable `this.terminateCb`.
4. Para comprender cómo se finaliza correctamente el stream, debemos observar el método `_onComplete()`. Este último método se invoca cada vez que se completa una tarea asíncrona. Comprueba si hay más tareas en ejecución y, si no las hay, invoca la función `this.terminateCb()`, lo que provocará que el stream finalice, liberando el evento `finish` que se puso en espera en el método `_flush()`. Ten en cuenta que `_onComplete()` es un método que introdujimos por conveniencia como parte de la implementación de nuestro `ConcurrentStream`; no es un método que estemos sobrescribiendo de la clase base `Transform`.

La clase `ConcurrentStream` que acabamos de construir nos permite crear fácilmente un stream Transform que ejecuta sus tareas de forma concurrente, pero hay una advertencia: no preserva el orden de los elementos tal como se reciben. De hecho, si bien inicia cada tarea en orden, las operaciones asíncronas pueden completarse y empujar datos en cualquier momento, independientemente de cuándo se iniciaron. Esta propiedad no funciona bien con streams binarios donde el orden de los datos suele ser importante, pero ciertamente puede ser útil con algunos tipos de streams de objetos.

##### Implementación de una aplicación de monitorización del estado de URLs

Ahora, apliquemos nuestro `ConcurrentStream` a un ejemplo concreto. Imaginemos que queremos crear un servicio simple para monitorizar el estado de una gran lista de URLs. Imaginemos que todas estas URLs están contenidas en un solo archivo y están separadas por saltos de línea.

Los streams pueden ofrecer una solución muy eficiente y elegante a este problema, especialmente si usamos nuestra clase `ConcurrentStream` para verificar las URLs de manera concurrente.

```javascript
// check-urls.js
import { createInterface } from 'node:readline'
import { createReadStream, createWriteStream } from 'node:fs'
import { pipeline } from 'node:stream/promises'
import { ConcurrentStream } from './concurrent-stream.js'

const inputFile = createReadStream(process.argv[2]) // 1
const fileLines = createInterface({ // 2
  input: inputFile,
})

const checkUrls = new ConcurrentStream( // 3
  async (url, _enc, push, done) => {
    if (!url) {
      return done()
    }
    try {
      await fetch(url, {
        method: 'HEAD',
        timeout: 5000,
        signal: AbortSignal.timeout(5000),
      })
      push(`${url} is up\n`)
    } catch (err) {
      push(`${url} is down: ${err}\n`)
    }
    done()
  }
)

const outputFile = createWriteStream('results.txt') // 4

await pipeline(fileLines, checkUrls, outputFile) // 5
console.log('All urls have been checked')
```

Como podemos ver, con los streams nuestro código se ve muy elegante y directo: inicializamos los diversos componentes de nuestra canalización de streaming y luego los combinamos. Pero analicemos algunos detalles importantes:
1. Primero, creamos un stream Readable a partir del archivo dado como entrada.
2. Aprovechamos la función `createInterface()` del módulo `node:readline` para crear un stream que envuelve el stream de entrada y proporciona el contenido del archivo original línea por línea. Este es un ayudante conveniente que es muy flexible y nos permite leer líneas de varias fuentes.
3. En este punto, creamos nuestra instancia de `ConcurrentStream`. En nuestra lógica de transformación personalizada, esperamos recibir una URL a la vez. Si la URL está vacía (por ejemplo, si hay una línea vacía en el archivo de origen), simplemente ignoramos la entrada actual. De lo contrario, realizamos una solicitud HEAD a la URL indicada con un tiempo de espera de 5 segundos. Si la solicitud es exitosa, el stream emite una cadena que describe el resultado positivo; de lo contrario, emite una cadena que describe un error. De cualquier manera, llamamos al callback `done()`, que le dice a `ConcurrentStream` que hemos completado el procesamiento de la tarea actual. Ten en cuenta que, dado que manejamos los fallos de manera elegante, el stream puede continuar procesando tareas incluso si una de ellas falla. Además, observa que estamos usando tanto `timeout` como un `AbortSignal` porque `AbortSignal` garantiza que la solicitud fallará si toma más de 5 segundos, independientemente de si los datos se están transfiriendo activamente. Algunas herramientas de prevención de bots mantienen deliberadamente las conexiones abiertas enviando respuestas a velocidades muy lentas, lo que hace que los bots se cuelguen indefinidamente. Al implementar este mecanismo, nos aseguramos de que las solicitudes se traten como fallidas si superan los 5 segundos por cualquier motivo.
4. El último stream que necesitamos crear es nuestro stream de salida: un archivo llamado `results.txt`.
5. ¡Finalmente tenemos todas las piezas juntas! Solo necesitamos combinar los streams en una canalización para permitir que los datos fluyan entre ellos. Y, una vez que se completa la canalización, imprimimos un mensaje de éxito.

Ahora, podemos ejecutar el módulo `check-urls.js` con un comando como este:

```bash
node check-urls.js urls.txt
```

Aquí, el archivo `urls.txt` contiene una lista de URLs (una por línea); por ejemplo:

```text
https://mario.fyi
https://loige.co
http://thiswillbedownforsure.com
```

Cuando el comando termine de ejecutarse, veremos que se creó un archivo, `results.txt`. Contiene los resultados de la operación; por ejemplo:

```text
http://thiswillbedownforsure.com is down
https://mario.fyi is up
https://loige.co is up
```

Existe una gran probabilidad de que el orden en que se escriben los resultados sea diferente del orden en que se especificaron las URLs en el archivo de entrada. Esta es una clara evidencia de que nuestro stream ejecuta sus tareas de forma concurrente y no impone ningún orden entre los diversos fragmentos de datos en el stream.

Por curiosidad, podríamos intentar reemplazar `ConcurrentStream` con un stream `Transform` normal y comparar el comportamiento y el rendimiento de ambos (puedes hacerlo como un ejercicio). Usar `Transform` directamente es mucho más lento, porque cada URL se verifica en secuencia, pero por otro lado, se preserva el orden de los resultados en el archivo `results.txt`.

En la siguiente sección, veremos cómo extender este patrón para limitar el número de tareas concurrentes que se ejecutan en un momento dado.

#### Ejecución concurrente no ordenada limitada

Si intentamos ejecutar la aplicación `check-urls.js` contra un archivo que contiene miles o millones de URLs, seguramente nos encontraremos con problemas. Nuestra aplicación creará una cantidad descontrolada de conexiones a la vez, enviando una cantidad considerable de datos concurrentemente y socavando potencialmente la estabilidad de la aplicación y la disponibilidad de todo el sistema. Como ya sabemos, la solución para mantener la carga y el uso de recursos bajo control es limitar la cantidad de tareas concurrentes que se ejecutan en un momento dado.

Veamos cómo funciona esto con streams creando un módulo `limited-concurrent-stream.js`, que es una adaptación de `concurrent-stream.js`, que creamos en la sección anterior.

Veamos cómo se ve, comenzando desde su constructor (resaltaremos las partes modificadas):

```javascript
export class LimitedConcurrentStream extends Transform {
  constructor (concurrency, userTransform, opts) {
    super({ ...opts, objectMode: true })
    this.concurrency = concurrency
    this.userTransform = userTransform
    this.running = 0
    this.continueCb = null
    this.terminateCb = null
  }
  // ...
```

Necesitamos un límite de concurrencia como entrada, y esta vez, guardaremos dos callbacks, uno para cualquier método `_transform` pendiente (`continueCb`, más sobre esto a continuación) y otro para el callback del método `_flush` (`terminateCb`).

A continuación está el método `_transform()`:

```javascript
  _transform (chunk, enc, done) {
    this.running++
    this.userTransform(
      chunk,
      enc,
      this.push.bind(this),
      this._onComplete.bind(this)
    )
    if (this.running < this.concurrency) {
      done()
    } else {
      this.continueCb = done
    }
  }
```

Esta vez, en el método `_transform()`, debemos verificar si tenemos algún espacio de ejecución libre antes de poder invocar `done()` y activar el procesamiento del siguiente elemento. Si ya hemos alcanzado el número máximo de streams en ejecución concurrente, guardamos el callback `done()` en la variable `continueCb` para que pueda invocarse tan pronto como finalice una tarea.

El método `_flush()` permanece exactamente igual que en la clase `ConcurrentStream`, así que pasemos directamente a implementar el método `_onComplete()`:

```javascript
  _onComplete (err) {
    this.running--
    if (err) {
      return this.emit('error', err)
    }
    const tmpCb = this.continueCb
    this.continueCb = null
    tmpCb?.()
    if (this.running === 0) {
      this.terminateCb && this.terminateCb()
    }
  }
```

Cada vez que se completa una tarea, invocamos cualquier `continueCb()` guardado que hará que el stream se desbloquee, activando el procesamiento del siguiente elemento.

Eso es todo para la clase `LimitedConcurrentStream`. Ahora podemos usarla en el módulo `check-urls.js` en lugar de `ConcurrentStream` y tener la concurrencia de nuestras tareas limitada al valor que establezcamos (consulta el código en el repositorio del libro para ver un ejemplo completo).

#### Ejecución concurrente ordenada

Los streams concurrentes que creamos anteriormente pueden alterar el orden de los datos emitidos, pero hay situaciones en las que esto no es aceptable. A veces, de hecho, es necesario que cada fragmento se emita en el mismo orden en que se recibió. Sin embargo, no todo está perdido. Aún podemos ejecutar la función de transformación de forma concurrente; todo lo que debemos hacer es ordenar los datos emitidos por cada tarea para que sigan el mismo orden en que se recibieron los datos. Es importante aquí distinguir claramente entre la lógica de procesamiento interna aplicada a cada fragmento recibido, que puede ocurrir de forma segura concurrentemente y, por lo tanto, en cualquier orden arbitrario, y cómo el stream Transform emite finalmente los datos procesados, que podrían necesitar preservar el orden original de los fragmentos.

La técnica que vamos a utilizar implica el uso de un buffer para reordenar los fragmentos mientras son emitidos por cada tarea en ejecución. Por brevedad, no proporcionaremos una implementación de dicho stream, ya que es bastante detallada para el alcance de este libro. Lo que vamos a hacer en su lugar es reutilizar uno de los paquetes disponibles en npm creados para este propósito específico, es decir, `parallel-transform` ([nodejsdp.link/parallel-transform](https://nodejsdp.link/parallel-transform)).

Podemos comprobar rápidamente el comportamiento de una ejecución concurrente ordenada modificando nuestro módulo `check-urls` existente. Digamos que queremos que nuestros resultados se escriban en el mismo orden que las URLs en el archivo de entrada, mientras ejecutamos nuestras comprobaciones de forma concurrente. Podemos hacer esto usando `parallel-transform`:

```javascript
//...
import parallelTransform from 'parallel-transform' // v1.2.0

const inputFile = createReadStream(process.argv[2])
const fileLines = createInterface({
  input: inputFile,
})

const checkUrls = parallelTransform(8, async function (url, done) {
  if (!url) {
    return done()
  }
  try {
    await fetch(url, { method: 'HEAD', timeout: 5 * 1000 })
    this.push(`${url} is up\n`)
  } catch (err) {
    this.push(`${url} is down: ${err}\n`)
  }
  done()
})

const outputFile = createWriteStream('results.txt')

await pipeline(fileLines, checkUrls, outputFile)
console.log('All urls have been checked')
```

En el ejemplo aquí, `parallelTransform()` crea un stream Transform en modo objeto que ejecuta nuestra lógica de transformación con una concurrencia máxima de 8. Si intentamos ejecutar esta nueva versión de `check-urls.js`, ahora veremos que el archivo `results.txt` enumera los resultados en el mismo orden en que aparecen las URLs en el archivo de entrada. Es importante ver que, aunque el orden de la salida es el mismo que el de la entrada, las tareas asíncronas todavía se ejecutan de forma concurrente y pueden completarse en cualquier orden.

Al utilizar el patrón de ejecución concurrente ordenada, debemos tener cuidado con los elementos lentos que bloquean la canalización o hacen que la memoria crezca indefinidamente. De hecho, si hay un elemento que requiere mucho tiempo para completarse, dependiendo de la implementación del patrón, provocará que el buffer que contiene los resultados ordenados pendientes crezca indefinidamente o que todo el procesamiento se bloquee hasta que se complete el elemento lento. Con la primera estrategia, optimizamos el rendimiento, mientras que con la segunda, obtenemos un uso de memoria predecible. La implementación de `parallel-transform` opta por una utilización predecible de la memoria y mantiene un buffer interno que no crecerá más que la concurrencia máxima especificada.

Con esto, concluimos nuestro análisis de los patrones de flujo de control asíncrono con streams. A continuación, nos centraremos en algunos patrones de tuberías.

---

### Sección 5: Patrones de tuberías (Piping patterns)

Al igual que en la fontanería de la vida real, los streams de Node.js también se pueden canalizar juntos siguiendo diferentes patrones. De hecho, podemos fusionar el flujo de dos streams diferentes en uno, dividir el flujo de un stream en dos o más tuberías, o redirigir el flujo según una condición. En esta sección, exploraremos los patrones de fontanería más importantes que se pueden aplicar a los streams de Node.js.

#### Combinación de streams

En este capítulo, hemos enfatizado el hecho de que los streams proporcionan una infraestructura simple para modularizar y reutilizar nuestro código, pero falta una última pieza de este rompecabezas: ¿qué sucede si queremos modularizar y reutilizar una canalización completa? ¿Qué pasa si queremos combinar múltiples streams para que parezcan uno solo desde el exterior? La siguiente figura muestra lo que esto significa:

**Figura 6.6:** Combinación de streams.

A partir de la Figura 6.6, ya deberíamos tener una pista de cómo funciona esto:
- Cuando escribimos en el stream combinado, estamos escribiendo en el primer stream de la canalización.
- Cuando leemos del stream combinado, estamos leyendo del último stream de la canalización.

Un stream combinado suele ser un stream Duplex, que se construye conectando el primer stream a su lado Writable y el último stream a su lado Readable.

Para crear un stream Duplex a partir de dos streams diferentes, uno Writable y uno Readable, podemos usar un módulo de npm como `duplexer3` ([nodejsdp.link/duplexer3](https://nodejsdp.link/duplexer3)) o `duplexify` ([nodejsdp.link/duplexify](https://nodejsdp.link/duplexify)).

Pero eso no es suficiente. De hecho, otra característica importante de un stream combinado es que debe capturar y propagar todos los errores que se emitan desde cualquier stream dentro de la canalización. Como ya mencionamos, cualquier evento de error no se propaga automáticamente por la canalización cuando usamos `pipe()`, y deberíamos adjuntar explícitamente un escuchador de errores a cada stream. Vimos que podíamos usar la función auxiliar `pipeline()` para superar las limitaciones de `pipe()` con la gestión de errores, pero el problema tanto con `pipe()` como con el ayudante `pipeline()` es que las dos funciones devuelven solo el último stream de la canalización, por lo que solo obtenemos el (último) componente Readable y no el (primer) componente Writable.

Podemos verificar esto muy fácilmente con el siguiente fragmento de código:

```javascript
import { createReadStream, createWriteStream } from 'node:fs'
import { Transform, pipeline } from 'node:stream'
import assert from 'node:assert/strict'

const streamA = createReadStream('package.json')
const streamB = new Transform({
  transform(chunk, _enc, done) {
    this.push(chunk.toString().toUpperCase())
    done()
  },
})
const streamC = createWriteStream('package-uppercase.json')

const pipelineReturn = pipeline(streamA, streamB, streamC, () => {
  // handle errors here
})
assert.equal(streamC, pipelineReturn) // valid

const pipeReturn = streamA.pipe(streamB).pipe(streamC)
assert.equal(streamC, pipeReturn) // valid
```

A partir del código anterior, debería quedar claro que con solo `pipe()` o `pipeline()`, no sería trivial construir un stream combinado.

Para recapitular, un stream combinado tiene dos ventajas principales:
- Podemos redistribuirlo como una caja negra ocultando su canalización interna.
- Tenemos una gestión de errores simplificada, ya que no tenemos que adjuntar un escuchador de errores a cada stream en la canalización, sino solo al propio stream combinado.

La combinación de streams es común en Node.js, y `node:stream` expone `compose()` para que sea limpio. Fusiona dos o más streams en un único Duplex: las escrituras que realizas en el compuesto entran en el primer stream de la cadena, las lecturas provienen del último. La contrapresión se preserva de un extremo a otro, y si algún stream interno arroja un error, el compuesto emite un error y toda la cadena se destruye.

```javascript
import { compose } from 'node:stream'

// ... define streamA, streamB, streamC
const combinedStream = compose(streamA, streamB, streamC)
```

Cuando hacemos algo como esto, `compose` creará una canalización a partir de nuestros streams, devolverá un nuevo stream combinado que abstrae la complejidad de nuestra canalización y proporcionará las ventajas analizadas anteriormente.

A diferencia de `.pipe()` o `pipeline()`, `compose()` es perezoso (*lazy*): simplemente construye la cadena y no inicia ningún flujo de datos, por lo que aún necesitas conectar mediante `pipe()` el Duplex devuelto a una fuente y/o destino para mover datos. Úsalo cuando quieras empaquetar una canalización de procesamiento reutilizable como un solo stream; usa `pipeline()` cuando quieras conectar una fuente a un destino y esperar a que se complete.

##### Implementación de un stream combinado

Para ilustrar un ejemplo simple de combinación de streams, consideremos el caso de los dos streams Transform siguientes:
- Uno que comprime y cifra los datos
- Uno que descifra y descomprime los datos

Usando `compose`, podemos construir fácilmente estos streams (en un archivo llamado `combined-streams.js`) combinando algunos de los streams que ya tenemos disponibles en las bibliotecas centrales:

```javascript
import { createGzip, createGunzip } from 'node:zlib'
import { createCipheriv, createDecipheriv, scryptSync } from 'node:crypto'
import { compose } from 'node:stream'

function createKey(password) {
  return scryptSync(password, 'salt', 24)
}

export function createCompressAndEncrypt(password, iv) {
  const key = createKey(password)
  const combinedStream = compose(
    createGzip(),
    createCipheriv('aes192', key, iv)
  )
  combinedStream.iv = iv
  return combinedStream
}

export function createDecryptAndDecompress(password, iv) {
  const key = createKey(password)
  return compose(createDecipheriv('aes192', key, iv), createGunzip())
}
```

Ahora podemos usar estos streams combinados como si fueran cajas negras, por ejemplo, para crear una pequeña aplicación que archive un archivo comprimiéndolo y cifrándolo. Hagamos eso en un nuevo módulo llamado `archive.js`:

```javascript
import { createReadStream, createWriteStream } from 'node:fs'
import { pipeline } from 'node:stream'
import { randomBytes } from 'node:crypto'
import { createCompressAndEncrypt } from './combined-streams.js'

const [, , password, source] = process.argv
const iv = randomBytes(16)
const destination = `${source}.gz.enc`

pipeline(
  createReadStream(source),
  createCompressAndEncrypt(password, iv),
  createWriteStream(destination),
  err => {
    if (err) {
      console.error(err)
      process.exit(1)
    }
    console.log(`${destination} created with iv: ${iv.toString('hex')}`)
  }
)
```

Observa cómo no tenemos que preocuparnos por cuántos pasos hay dentro de `archive.js`. De hecho, simplemente lo tratamos como un único stream dentro de nuestra canalización actual. Esto hace que nuestro stream combinado sea fácilmente reutilizable en otros contextos.

Ahora, para ejecutar el módulo de archivo, simplemente especifica una contraseña y un archivo en los argumentos de la línea de comandos:

```bash
node archive.js mypassword /path/to/a/file.txt
```

Este comando creará un archivo llamado `/path/to/a/file.txt.gz.enc` e imprimirá el vector de inicialización generado en la consola.

Ahora, como ejercicio, podrías usar la función `createDecryptAndDecompress()` para crear un script similar que tome una contraseña, un vector de inicialización y un archivo archivado y lo desarchive. No te preocupes, si te quedas atascado, tendremos una solución implementada en el repositorio de código de este libro bajo el archivo `unarchive.js`.

En aplicaciones de la vida real, generalmente es preferible incluir el vector de inicialización como parte de los datos cifrados, en lugar de requerir que el usuario lo pase de un lado a otro. Una forma de implementar esto es hacer que los primeros 16 bytes emitidos por el stream de archivo representen el vector de inicialización. La utilidad de desarchivado tendría que actualizarse en consecuencia para consumir los primeros 16 bytes antes de comenzar a procesar los datos en streaming. Este enfoque agregaría cierta complejidad adicional, que está fuera del alcance de este ejemplo; por lo tanto, optamos por una solución más simple. Una vez que te sientas cómodo con los streams, te recomendamos que intentes implementar, como ejercicio, una solución donde el usuario no tenga que pasar el vector de inicialización.

Con este ejemplo, hemos demostrado claramente lo importante que es combinar streams. Por un lado, nos permite crear composiciones reutilizables de streams y, por el otro, simplifica la gestión de errores de una canalización.

#### Bifurcación de streams (Forking streams)

Podemos realizar una bifurcación de un stream conectando un único stream Readable a múltiples streams Writable mediante `pipe()`. Esto es útil cuando queremos enviar los mismos datos a diferentes destinos; por ejemplo, dos sockets diferentes o dos archivos diferentes. También se puede utilizar cuando queremos realizar diferentes transformaciones sobre los mismos datos, o cuando queremos dividir los datos según algún criterio. Si estás familiarizado con el comando de Unix `tee` ([nodejsdp.link/tee](https://nodejsdp.link/tee)), ¡este es exactamente el mismo concepto aplicado a los streams de Node.js!

La Figura 6.7 nos ofrece una representación gráfica de este patrón:

**Figura 6.7:** Bifurcación de un stream.

Bifurcar un stream en Node.js es bastante fácil, pero hay algunas advertencias a tener en cuenta. Comencemos analizando este patrón con un ejemplo. Será más fácil apreciar las advertencias de este patrón una vez que tengamos un ejemplo a mano.

##### Implementación de un generador de múltiples sumas de comprobación (checksums)

Creemos una pequeña utilidad que genere los hashes `sha1` y `md5` de un archivo determinado. Llamemos a este nuevo módulo `generate-hashes.js`:

```javascript
import { createReadStream, createWriteStream } from 'node:fs'
import { createHash } from 'node:crypto'

const filename = process.argv[2]
const sha1Stream = createHash('sha1').setEncoding('hex')
const md5Stream = createHash('md5').setEncoding('hex')

const inputStream = createReadStream(filename)
inputStream.pipe(sha1Stream).pipe(createWriteStream(`${filename}.sha1`))
inputStream.pipe(md5Stream).pipe(createWriteStream(`${filename}.md5`))
```

Muy simple, ¿verdad? La variable `inputStream` se canaliza hacia `sha1Stream` por un lado y hacia `md5Stream` por el otro. Hay algunas cosas a tener en cuenta que suceden detrás de escena:
- Tanto `md5Stream` como `sha1Stream` finalizarán automáticamente cuando finalice `inputStream`, a menos que especifiquemos `{ end: false }` como opción al invocar `pipe()`.
- Las dos ramas de la bifurcación recibirán una referencia a los mismos fragmentos de datos, por lo que debemos tener mucho cuidado al realizar operaciones con efectos secundarios en los datos, ya que eso afectaría a todos los streams a los que enviamos datos.
- La contrapresión funcionará de inmediato; el flujo proveniente de `inputStream` irá tan rápido como la rama más lenta de la bifurcación. En otras palabras, si un destino pausa el stream de origen para manejar la contrapresión durante mucho tiempo, todos los demás destinos también estarán esperando. Además, ¡un destino que se bloquee indefinidamente bloqueará toda la canalización!
- Si canalizamos a un stream adicional después de haber comenzado a consumir los datos en la fuente (canalización asíncrona), el nuevo stream solo recibirá nuevos fragmentos de datos. En esos casos, podemos usar una instancia de PassThrough como marcador de posición para recopilar todos los datos desde el momento en que comenzamos a consumir el stream. Luego, el stream PassThrough se puede leer en cualquier momento futuro sin riesgo de perder ningún dato. Solo ten en cuenta que este enfoque podría generar contrapresión y bloquear toda la canalización, como se discutió en el punto anterior.

#### Fusión de streams (Merging streams)

La fusión es la operación opuesta a la bifurcación e implica canalizar un conjunto de streams Readable en un único stream Writable, como se muestra en la Figura 6.8:

**Figura 6.8:** Fusión de streams.

Fusionar múltiples streams en uno es, en general, una operación simple; sin embargo, tenemos que prestar atención a la forma en que manejamos el evento `end`, ya que la canalización utilizando las opciones predeterminadas (donde `{ end: true }`) hace que el stream de destino finalice tan pronto como finaliza una de las fuentes. Esto a menudo puede provocar un error, ya que las otras fuentes activas continúan escribiendo en un stream ya terminado.

La solución a este problema es usar la opción `{ end: false }` al canalizar múltiples fuentes hacia un único destino y luego invocar `end()` en el destino solo cuando todas las fuentes hayan completado la lectura.

##### Fusión de archivos de texto

Para poner un ejemplo sencillo, implementemos un pequeño programa que tome una ruta de salida y un número arbitrario de archivos de texto, y luego fusione las líneas de cada archivo en el archivo de destino. Nuestro nuevo módulo se llamará `merge-lines.js`. Definamos su contenido, comenzando desde algunos pasos de inicialización:

```javascript
import { createReadStream, createWriteStream } from 'node:fs'
import { Readable, Transform } from 'node:stream'
import { createInterface } from 'node:readline'

const [, , dest, ...sources] = process.argv
```

En el código anterior, simplemente estamos cargando todas las dependencias e inicializando las variables que contienen el nombre del archivo de destino (`dest`) y todos los archivos de origen (`sources`).

A continuación, crearemos el stream de destino:

```javascript
const destStream = createWriteStream(dest)
```

Ahora es el momento de inicializar los streams de origen:

```javascript
let endCount = 0
for (const source of sources) {
  const sourceStream = createReadStream(source, { highWaterMark: 16 })
  const linesStream = Readable.from(createInterface({ input: sourceStream }))
  const addLineEnd = new Transform({
    transform(chunk, _encoding, cb) {
      cb(null, `${chunk}\n`)
    },
  })

  sourceStream.on('end', () => {
    if (++endCount === sources.length) {
      destStream.end()
      console.log(`${dest} created`)
    }
  })

  linesStream
    .pipe(addLineEnd)
    .pipe(destStream, { end: false })
}
```

En este código, inicializamos un stream de origen para cada archivo en el array `sources`. Cada fuente se lee mediante `createReadStream()`.

La función `createInterface()` del módulo `node:readline` se utiliza para procesar cada archivo de origen línea por línea, produciendo un `linesStream` que emite líneas individuales del archivo de origen.

Para garantizar que cada línea emitida termine con un carácter de nueva línea, usamos un stream Transform simple (`addLineEnd`). Esta transformación añade `\n` a cada fragmento de datos.

También adjuntamos un escuchador de eventos `end` a cada stream de origen. Este escuchador incrementa un contador (`endCount`) cada vez que finaliza un stream de origen. Cuando se han leído todos los streams de origen, se asegura de que el stream de destino (`destStream`) esté cerrado, señalando la finalización de la canalización de streaming.

Finalmente, cada `linesStream` se canaliza a través de la transformación `addLineEnd` y hacia el stream de destino. Durante este último paso, usamos la opción `{ end: false }` para mantener abierto el stream de destino incluso cuando finaliza una de las fuentes. El stream de destino solo se cierra cuando todos los streams de origen han finalizado, lo que garantiza que no se pierdan datos durante la fusión. Este último paso es donde ocurre la fusión, porque efectivamente estamos canalizando múltiples streams independientes hacia el mismo stream de destino.

Ahora podemos ejecutar este código con el siguiente comando:

```bash
node merge-lines.js <destination> <source1> <source2> <source3> ...
```

Si ejecutas este código con archivos lo suficientemente grandes, notarás que el archivo de destino contendrá líneas intercaladas aleatoriamente de todos los archivos de origen (un `highWaterMark` bajo de 16 bytes hace que esta propiedad sea aún más evidente). Este tipo de comportamiento puede ser aceptable en algunos tipos de streams de objetos y algunos streams de texto divididos por líneas (como en nuestro ejemplo actual), pero a menudo no es deseable cuando se trata de la mayoría de los streams binarios.

Existe una variación del patrón que nos permite fusionar streams en orden; consiste en consumir los streams de origen uno tras otro. Cuando termina el anterior, el siguiente comienza a emitir fragmentos (es como concatenar la salida de todas las fuentes). Como siempre, en npm podemos encontrar algunos paquetes que también se ocupan de esta situación. Uno de ellos es `multistream` ([https://npmjs.org/package/multistream](https://npmjs.org/package/multistream)).

#### Multiplexación y desmultiplexación

Existe una variación particular del patrón de fusión de streams en la que no queremos simplemente unir múltiples streams, sino utilizar un canal compartido para entregar los datos de un conjunto de streams. Esta es una operación conceptualmente diferente porque los streams de origen permanecen lógicamente separados dentro del canal compartido, lo que nos permite dividir el stream nuevamente una vez que los datos llegan al otro extremo del canal compartido. La Figura 6.9 aclara esta situación:

**Figura 6.9:** Multiplexación y desmultiplexación de streams.

La operación de combinar múltiples streams (en este caso, también conocidos como canales) para permitir la transmisión a través de un único stream se llama **multiplexación**, mientras que la operación opuesta, es decir, reconstruir los streams originales a partir de los datos recibidos de un stream compartido, se llama **desmultiplexación**. Los dispositivos que realizan estas operaciones se denominan multiplexor (o mux) y desmultiplexor (o demux), respectivamente. Esta es un área ampliamente estudiada en ciencias de la computación y telecomunicaciones en general, ya que es uno de los fundamentos de casi cualquier tipo de medio de comunicación, como la telefonía, la radio, la televisión y, por supuesto, la propia Internet. Para el alcance de este libro, no iremos demasiado lejos con las explicaciones, ya que este es un tema muy amplio.

Lo que queremos demostrar en esta sección es cómo es posible utilizar un stream compartido de Node.js para transmitir múltiples streams lógicamente separados que luego se separan nuevamente en el otro extremo del stream compartido.

##### Creación de un logger remoto

Usemos un ejemplo para guiar nuestra discusión. Queremos un pequeño programa que inicie un proceso secundario y redirija tanto su salida estándar como su error estándar a un servidor remoto, que, a su vez, guarde los dos streams en dos archivos separados. Entonces, en este caso, el medio compartido es una conexión TCP, mientras que los dos canales a multiplexar son el `stdout` y el `stderr` de un proceso secundario. Aprovecharemos una técnica llamada conmutación de paquetes (*packet switching*), la misma técnica que utilizan protocolos como IP, TCP y UDP. La conmutación de paquetes implica envolver los datos en paquetes, lo que nos permite especificar diversa metainformación que es útil para multiplexar, enrutar, controlar el flujo, verificar datos corruptos, etc. El protocolo que estamos implementando en nuestro ejemplo es muy minimalista. Envolvemos nuestros datos en paquetes simples, como se ilustra en la Figura 6.10:

**Figura 6.10:** Estructura de bytes del paquete de datos para nuestro registrador remoto.

Como se muestra en la Figura 6.10, el paquete contiene los datos reales, pero también una cabecera (Channel ID + Data length), lo que permitirá diferenciar los datos de cada stream y permitirá que el desmultiplexor enrute el paquete al canal correcto.

##### Lado del cliente – multiplexación

Comencemos a construir nuestra aplicación desde el lado del cliente. Con mucha creatividad, llamaremos al módulo `client.js`. Esto representa la parte de la aplicación que se encarga de iniciar un proceso secundario y multiplexar sus streams.

Entonces, comencemos por definir el módulo. Primero, necesitamos algunas dependencias:

```javascript
import { fork } from 'node:child_process'
import { connect } from 'node:net'
```

Ahora, implementemos una función que realice la multiplexación de una lista de fuentes:

```javascript
function multiplexChannels(sources, destination) {
  let openChannels = sources.length
  for (let i = 0; i < sources.length; i++) {
    sources[i]
      .on('readable', function () { // 1
        let chunk
        while ((chunk = this.read()) !== null) {
          const outBuff = Buffer.alloc(1 + 4 + chunk.length) // 2
          outBuff.writeUInt8(i, 0)
          outBuff.writeUInt32BE(chunk.length, 1)
          chunk.copy(outBuff, 5)
          console.log(`Sending packet to channel: ${i}`)
          destination.write(outBuff) // 3
        }
      })
      .on('end', () => { // 4
        if (--openChannels === 0) {
          destination.end()
        }
      })
  }
}
```

La función `multiplexChannels()` acepta los streams de origen que se multiplexarán y el canal de destino como entrada, y luego realiza los siguientes pasos:
1. Para cada stream de origen, registra un escuchador para el evento `readable`, donde leemos los datos del stream utilizando el modo non-flowing (el uso del modo non-flowing nos dará más flexibilidad al leer un número específico de bytes, a medida que lleguemos a escribir el código de desmultiplexación).
2. Cuando se lee un fragmento, lo envolvemos en un paquete llamado `outBuff` que contiene, en orden, 1 byte (`UInt8`) para el ID del canal (desplazamiento 0), 4 bytes (`UInt32BE`) para el tamaño del paquete (desplazamiento 1) y luego los datos reales (desplazamiento 5).
3. Cuando el paquete está listo, lo escribimos en el stream de destino.
4. Finalmente, registramos un escuchador para el evento `end` para que podamos terminar el stream de destino cuando todos los streams de origen hayan terminado.

Nuestro protocolo es capaz de multiplexar hasta 256 streams de origen diferentes porque tenemos 1 byte para identificar el canal. Esto probablemente sea suficiente para la mayoría de los casos de uso, pero si necesitas más, puedes usar más bytes para identificar el canal.

Ahora, la última parte de nuestro cliente se vuelve muy fácil:

```javascript
const socket = connect(3000, () => { // 1
  const child = fork( // 2
    process.argv[2],
    process.argv.slice(3),
    { silent: true }
  )
  multiplexChannels([child.stdout, child.stderr], socket) // 3
})
```

En este último fragmento de código, realizamos las siguientes operaciones:
1. Creamos una nueva conexión de cliente TCP a la dirección `localhost:3000`.
2. Iniciamos el proceso secundario utilizando el primer argumento de la línea de comandos como ruta, mientras proporcionamos el resto del array `process.argv` como argumentos para el proceso secundario. Especificamos la opción `{silent: true}` para que el proceso secundario no herede `stdout` y `stderr` del padre.
3. Finalmente, tomamos `stdout` y `stderr` del proceso secundario y los multiplexamos en el stream Writable del socket usando la función `multiplexChannels()`.

##### Lado del servidor – desmultiplexación

Ahora podemos encargarnos de crear el lado del servidor de la aplicación (`server.js`), donde desmultiplexamos los streams de la conexión remota y los canalizamos hacia dos archivos diferentes.

Comencemos creando una función llamada `demultiplexChannel()`:

```javascript
import { createWriteStream } from 'node:fs'
import { createServer } from 'node:net'

function demultiplexChannel(source, destinations) {
  let currentChannel = null
  let currentLength = null
  source
    .on('readable', () => { // 1
      let chunk
      if (currentChannel === null) { // 2
        chunk = source.read(1)
        currentChannel = chunk?.readUInt8(0)
      }
      if (currentLength === null) { // 3
        chunk = source.read(4)
        currentLength = chunk?.readUInt32BE(0)
        if (currentLength === null) {
          return null
        }
      }
      chunk = source.read(currentLength) // 4
      if (chunk === null) {
        return null
      }
      console.log(`Received packet from: ${currentChannel}`)
      destinations[currentChannel].write(chunk) // 5
      currentChannel = null
      currentLength = null
    })
    .on('end', () => { // 6
      for (const destination of destinations) {
        destination.end()
      }
      console.log('Source channel closed')
    })
}
```

El código anterior puede parecer complicado, pero no lo es. Gracias a las características de los streams Readable de Node.js, podemos implementar fácilmente la desmultiplexación de nuestro pequeño protocolo de la siguiente manera:
1. Comenzamos a leer del stream usando el modo non-flowing (como puedes ver, ahora podemos leer fácilmente tantos bytes como necesitemos para cada parte del mensaje recibido).
2. Primero, si aún no hemos leído el ID del canal, intentamos leer 1 byte del stream y luego transformarlo en un número.
3. El siguiente paso es leer la longitud de los datos. Necesitamos 4 bytes para eso, por lo que es posible (aunque poco probable) que no tengamos suficientes datos en el buffer interno, lo que hará que la invocación de `this.read()` devuelva `null`. En tal caso, simplemente interrumpimos el análisis y volvemos a intentarlo en el siguiente evento `readable`.
4. Cuando finalmente también podemos leer el tamaño de los datos, sabemos cuántos datos extraer del buffer interno, por lo que intentamos leerlos todos. Nuevamente, si esta operación devuelve `null`, todavía no tenemos todos los datos en el buffer, por lo que devolvemos `null` y volvemos a intentarlo en el siguiente evento `readable`.
5. Cuando leemos todos los datos, podemos escribirlos en el canal de destino correcto, asegurándonos de restablecer las variables `currentChannel` y `currentLength` (estas se usarán para analizar el siguiente paquete).
6. Por último, nos aseguramos de finalizar todos los canales de destino cuando finalice el canal de origen.

Ahora que podemos desmultiplexar el stream de origen, pongamos a trabajar nuestra nueva función:

```javascript
const server = createServer(socket => {
  const stdoutStream = createWriteStream('stdout.log')
  const stderrStream = createWriteStream('stderr.log')
  demultiplexChannel(socket, [stdoutStream, stderrStream])
})

server.listen(3000, () => console.log('Server started'))
```

En el código anterior, primero iniciamos un servidor TCP en el puerto 3000; luego, para cada conexión que recibimos, creamos dos streams Writable que apuntan a dos archivos diferentes: uno para la salida estándar y el otro para el error estándar. Estos son nuestros canales de destino. Finalmente, usamos `demultiplexChannel()` para desmultiplexar el stream del socket en `stdoutStream` y `stderrStream`.

##### Ejecución de la aplicación mux/demux

Ahora estamos listos para probar nuestra nueva aplicación mux/demux, pero primero, creemos un pequeño programa de Node.js para producir una salida de muestra:

```javascript
// generate-data.js
console.log('out1')
console.log('out2')
console.error('err1')
console.log('out3')
console.error('err2')
```

Bien, ahora estamos listos para probar nuestra aplicación de registro remoto. Primero, iniciemos el servidor:

```bash
node server.js
```

Luego, iniciaremos el cliente proporcionando el archivo que queremos iniciar como un proceso secundario:

```bash
node client.js generateData.js
```

El cliente se ejecutará casi de inmediato, pero al final del proceso, la entrada estándar y la salida estándar de la aplicación `generate-data.js` habrán viajado a través de una única conexión TCP y se habrán desmultiplexado en el servidor en dos archivos separados.

Ten en cuenta que, dado que estamos utilizando `child_process.fork()` ([nodejsdp.link/fork](https://nodejsdp.link/fork)), nuestro cliente solo podrá iniciar otros módulos de Node.js.

##### Multiplexación y desmultiplexación de streams de objetos

El ejemplo que acabamos de mostrar demuestra cómo multiplexar y desmultiplexar un stream binario/de texto, pero vale la pena mencionar que las mismas reglas se aplican a los streams de objetos. La mayor diferencia es que al usar objetos, ya tenemos una forma de transmitir los datos usando mensajes atómicos (los objetos), por lo que multiplexar sería tan fácil como configurar una propiedad `channelID` en cada objeto. Desmultiplexar simplemente implicaría leer la propiedad `channelID` y enrutar cada objeto hacia el stream de destino correcto.

Otro patrón que involucra solo la desmultiplexación es enrutar los datos que provienen de una fuente según alguna condición. Con este patrón, podemos implementar flujos complejos, como el que se muestra en la Figura 6.11:

**Figura 6.11:** Desmultiplexación de un stream de objetos.

El desmultiplexor utilizado en el sistema de la Figura 6.11 toma un stream de objetos que representan animales y distribuye cada uno de ellos al stream de destino correcto según la clase del animal: reptiles, anfibios o mamíferos.

Usando el mismo principio, también podemos implementar una sentencia `if...else` para streams. Para inspirarte, echa un vistazo al paquete `ternary-stream` ([nodejsdp.link/ternary-stream](https://nodejsdp.link/ternary-stream)), que nos permite hacer exactamente eso.

---

### Sección 6: Utilidades para streams Readable

En este capítulo, hemos explorado cómo funcionan los streams de Node.js, cómo crear streams personalizados y cómo componerlos en canalizaciones de procesamiento de datos eficientes y elegantes. Para completar el panorama, veamos algunas utilidades proporcionadas por el módulo `node:stream` que simplifican el trabajo con streams Readable. Estas utilidades están diseñadas para agilizar el procesamiento de datos en streaming y aportar un toque de programación funcional a las operaciones con streams.

Todas estas utilidades son métodos disponibles para cualquier stream Readable, incluidos los streams Duplex, PassThrough y Transform. Dado que la mayoría de estos métodos devuelven un nuevo stream Readable, se pueden encadenar para crear un código expresivo similar a una canalización. Como era de esperar, muchos de estos métodos reflejan operaciones comunes disponibles en el prototipo de `Array`, pero están optimizados para manejar datos en streaming.

He aquí un resumen de los métodos clave:

#### Mapeo y transformación
- `readable.map(fn)`: Aplica una función de transformación (`fn`) a cada fragmento en el stream, devolviendo un nuevo stream con los datos transformados. Si `fn` devuelve una Promise, el resultado se espera con `await` antes de pasarlo al stream de salida.
- `readable.flatMap(fn)`: Similar a `map`, pero permite que `fn` devuelva streams, iterables o iterables asíncronos, que luego se aplanan y se fusionan en el stream de salida.

#### Filtrado e iteración
- `readable.filter(fn)`: Filtra el stream aplicando `fn` a cada fragmento. Solo los fragmentos para los que `fn` devuelve un valor verdadero (*truthy*) se incluyen en el stream de salida. Admite funciones `fn` asíncronas.
- `readable.forEach(fn)`: Invoca `fn` para cada fragmento en el stream. Esto se usa típicamente para efectos secundarios en lugar de producir un nuevo stream. Si `fn` devuelve una Promise, se esperará antes de procesar el siguiente fragmento.

#### Búsqueda y evaluación
- `readable.some(fn)`: Comprueba si al menos un fragmento satisface la condición en `fn`. Una vez que se encuentra un valor verdadero, el stream se destruye y la Promise devuelta se resuelve en `true`. Si ningún fragmento satisface la condición, se resuelve en `false`.
- `readable.every(fn)`: Verifica si todos los fragmentos satisfacen la condición en `fn`. Si algún fragmento no cumple la condición, el stream se destruye y la Promise devuelta se resuelve en `false`. De lo contrario, se resuelve en `true` cuando termina el stream.
- `readable.find(fn)`: Devuelve una Promise que se resolverá con el valor del primer fragmento que satisfaga la condición en `fn`. Si ningún fragmento cumple la condición, la Promise devuelta se resolverá en `undefined` una vez que finalice el stream.

#### Limitación y reducción
- `readable.drop(n)`: Omite los primeros `n` fragmentos en el stream, devolviendo un nuevo stream que comienza desde el fragmento `(n+1)`-ésimo.
- `readable.take(n)`: Devuelve un nuevo stream que incluye, como máximo, los primeros `n` fragmentos. Una vez que se alcanzan `n` fragmentos, el stream se termina.
- `readable.reduce(fn, initialValue)`: Reduce el stream aplicando `fn` a cada fragmento, acumulando un resultado que se devuelve como una Promise. Si no se proporciona `initialValue`, el primer fragmento se utiliza como valor inicial.

La documentación oficial tiene muchos ejemplos para todos estos métodos y hay otros métodos menos comunes que no hemos explorado por brevedad. Te recomendamos que consultes la documentación ([nodejsdp.link/stream-iterators](https://nodejsdp.link/stream-iterators)) si alguno de estos todavía te parece confuso y no estás seguro de cuándo usarlos.

Solo para darte una descripción más práctica, volvamos a implementar la canalización de procesamiento que ilustramos antes para explicar el filtrado y la reducción con un stream Transform personalizado, pero esta vez vamos a usar solo utilidades de streams Readable. Como recordatorio, en este ejemplo, estamos analizando un archivo CSV que contiene datos de ventas. Queremos calcular el importe total del beneficio obtenido de las ventas en Italia. Cada línea del archivo CSV tiene 3 campos: tipo, país y beneficio. La primera línea contiene las cabeceras CSV.

```javascript
import { createReadStream } from 'node:fs'
import { createInterface } from 'node:readline'
import { Readable, compose } from 'node:stream'
import { createGunzip } from 'node:zlib'

const uncompressedData = compose( // 1
  createReadStream('data.csv.gz'),
  createGunzip()
)

const byLine = Readable.from( // 2
  createInterface({ input: uncompressedData })
)

const totalProfit = await byLine // 3
  .drop(1) // 4
  .map(chunk => { // 5
    const [type, country, profit] = chunk.toString().split(',')
    return { type, country, profit: Number.parseFloat(profit) }
  })
  .filter(record => record.country === 'Italy') // 6
  .reduce((acc, record) => acc + record.profit, 0) // 7

console.log(totalProfit)
```

He aquí un desglose paso a paso de lo que hace el código anterior:
1. Los datos provienen de un archivo CSV comprimido con gzip, por lo que inicialmente componemos un stream de lectura de archivos y un stream de descompresión para crear un stream de origen que proporcione datos CSV sin comprimir.
2. Queremos leer los datos línea por línea, por lo que usamos la utilidad `createInterface()` del módulo `node:readline` para envolver nuestro stream de origen y darnos un nuevo stream Readable (`byLine`) que produce líneas a partir del stream original.
3. Aquí es donde comenzamos a usar algunos de los ayudantes que analizamos en esta sección. Dado que el último ayudante es `.reduce()`, que devuelve una Promise, usamos `await` aquí para esperar a que se resuelva la Promise devuelta y capturar el resultado final en la variable `totalProfit`.
4. El primer ayudante que usamos es `.drop(1)`, que nos permite omitir la primera línea de los datos de origen sin comprimir. Esta línea contendrá la cabecera CSV (`"type,country,profit"`) y ningún dato útil, por lo que tiene sentido omitirla. Esta operación devuelve un nuevo stream Readable, por lo que podemos encadenar otros métodos auxiliares.
5. El siguiente ayudante que usamos en la cadena es `.map()`. En la función de mapeo, proporcionamos toda la lógica necesaria para analizar una línea del archivo CSV original y convertirla en un objeto de registro que contiene los campos `type`, `country` y `profit`. Esta operación devuelve otro stream Readable, por lo que podemos seguir encadenando más funciones auxiliares para continuar construyendo nuestra lógica de procesamiento.
6. El siguiente paso es `.filter()`, que usamos para retener solo los registros que representan el beneficio asociado con el país `Italy`. Una vez más, esta operación nos da un nuevo stream Readable.
7. El último paso de la canalización de procesamiento es `.reduce()`. Usamos este ayudante para agregar todos los registros filtrados sumando sus beneficios. Como mencionamos antes, esta operación nos dará una Promise que se resolverá con el beneficio total una vez que se complete el stream.

Este ejemplo muestra cómo crear canalizaciones de procesamiento de streams utilizando un enfoque más directo. En este enfoque, encadenamos métodos auxiliares y tenemos toda la lógica de transformación claramente visible en el mismo contexto (asumiendo que definimos todas las funciones de transformación en línea). Este enfoque puede ser particularmente conveniente en situaciones donde la lógica de transformación es muy simple y no necesitas construir streams Transform personalizados altamente especializados y reutilizables.

Ten en cuenta que, en este ejemplo, creamos nuestra propia forma básica de analizar registros a partir de líneas CSV en lugar de utilizar una biblioteca dedicada para ello. Hicimos esto solo para tener una excusa para mostrar cómo usar los métodos `.drop()` y `.map()`. Nuestra implementación es muy rudimentaria y no maneja todos los casos extremos posibles. Esto está bien porque sabemos que no hay casos extremos (por ejemplo, campos entre comillas) en nuestros datos de entrada, pero en proyectos del mundo real, recomendaríamos utilizar una biblioteca de análisis de CSV confiable en su lugar.

---

### Sección 7: Web Streams

El estándar WHATWG Streams ([nodejsdp.link/web-streams](https://nodejsdp.link/web-streams)) proporciona una API estandarizada para trabajar con datos en streaming, conocida como "Web Streams". Si bien está inspirada en los streams de Node.js, tiene su propia implementación distinta y está diseñada para ser un estándar universal para el ecosistema de JavaScript más amplio, incluidos los navegadores.

Aproximadamente una década después del desarrollo inicial de los streams de Node.js, surgieron los Web Streams para abordar la falta de una API de streaming nativa en los entornos de navegador, algo que dificultaba el trabajo eficiente con grandes conjuntos de datos en el frontend.

Hoy en día, la mayoría de los navegadores modernos admiten el estándar Web Streams de forma nativa, lo que lo convierte en una opción ideal para crear canalizaciones de streaming dentro del navegador. Por el contrario, los streams de Node.js no están disponibles de forma nativa en los navegadores. Podrías llevar los streams de Node.js al navegador instalándolos como una biblioteca en tu proyecto, pero su utilidad es limitada ya que las APIs nativas como `fetch` usan Web Streams para enviar solicitudes o leer respuestas de forma incremental. Dado esto, el uso de Web Streams en el navegador es la opción recomendada.

Los Web Streams también se han implementado en Node.js, lo que efectivamente nos brinda dos APIs en competencia para lidiar con datos en streaming. Sin embargo, al momento de escribir este artículo, Web Streams todavía es relativamente nuevo y aún no ha alcanzado el mismo nivel de adopción que los streams nativos de Node.js dentro del gran ecosistema de Node.js. Es por eso que este capítulo se centró principalmente en los streams de Node.js, pero comprender los Web Streams sigue siendo una pieza importante de conocimiento y esperamos que se vuelva más relevante en los próximos años.

Afortunadamente, comenzar con Web Streams debería ser fácil si has estado siguiendo este capítulo. La mayoría de los conceptos están alineados y las diferencias principales radican en los nombres de las funciones y los argumentos, que es algo que se puede aprender fácilmente consultando la documentación de la API de Web Streams.

Un aspecto que vale la pena explorar aquí es la interoperabilidad entre los streams de Node.js y los Web Streams. Afortunadamente, es posible convertir objetos de stream de Node.js a objetos de Web Stream equivalentes y viceversa. Esto facilita la transición o el trabajo con bibliotecas de terceros que utilizan Web Streams en el contexto de Node.js.

Analicemos brevemente cómo funciona esta interoperabilidad.

En el estándar Web Streams, tenemos 3 tipos principales de objetos:
- `ReadableStream`: Fuente de datos en streaming y prácticamente equivalente a un stream Readable de Node.js.
- `WritableStream`: Destino para datos en streaming; equivalente a un stream Writable de Node.js.
- `TransformStream`: Permite transformar datos en una canalización de streaming. Equivalente a un stream Transform de Node.js.

Observa cómo estos conceptos coinciden casi a la perfección. También ten en cuenta cómo, gracias al sufijo `Stream` de las clases de Web Streams, no tenemos conflictos de nombres entre abstracciones de streaming equivalentes.

#### Conversión de streams de Node.js a Web Streams

Puedes convertir fácilmente streams de Node.js en objetos de Web Streams equivalentes utilizando el método `.toWeb(sourceNodejsStream)` disponible respectivamente en las clases `Readable`, `Writable` y `Transform`.

Veamos cómo se ve la sintaxis:

```javascript
import { Readable, Writable, Transform } from 'node:stream'

const nodeReadable = new Readable({/*...*/}) // Readable
const webReadable = Readable.toWeb(nodeReadable) // ReadbleStream
const nodeWritable = new Writable({/*...*/}) // Writable
const webWritable = Writable.toWeb(nodeWritable) // WritableStream
const nodeTransform = new Transform({/*...*/}) // Transform
const webTransform = Transform.toWeb(nodeTransform) // TransformStream
```

#### Conversión de Web Streams a streams de Node.js

Las clases `Readable`, `Writable` y `Transform` también exponen métodos para convertir un Web Stream en un stream equivalente de Node.js. Estos métodos, como era de esperar, tienen la siguiente firma: `.fromWeb(sourceWebStream)`.

Veamos un ejemplo rápido para aclarar la sintaxis:

```javascript
import { Readable, Writable, Transform } from 'node:stream'
import {
  ReadableStream,
  WritableStream,
  TransformStream,
} from 'node:stream/web'

const webReadable = new ReadableStream({/*...*/}) // ReadableStream
const nodeReadable = Readable.fromWeb(webReadable) // Readable
const webWritable = new WritableStream({/*...*/}) // WritableStream
const nodeWritable = Writable.fromWeb(webWritable) // Writable
const webTransform = new TransformStream({/*...*/}) // TransformStream
const nodeTransform = Transform.fromWeb(webTransform) // Transform
```

Los dos últimos fragmentos ilustran lo fácil que es convertir tipos de stream entre streams de Node.js y Web Streams.

Un detalle importante a tener en cuenta es que estas conversiones no destruyen el stream de origen, sino que lo envuelven en un nuevo objeto que cumple con la API de destino. Por ejemplo, cuando convertimos un stream Readable de Node.js en un `ReadableStream` web, todavía podemos leer del stream de origen mientras también leemos del nuevo Web Stream. El siguiente ejemplo debería ayudar a aclarar esta idea:

```javascript
import { Readable } from 'node:stream'

const nodeReadable = new Readable({
  read() {
    this.push('Hello, ')
    this.push('world!')
    this.push(null)
  },
})

const webReadable = Readable.toWeb(nodeReadable)
nodeReadable.pipe(process.stdout)
webReadable.pipeTo(Writable.toWeb(process.stdout))
```

En el ejemplo anterior, estamos definiendo un stream de Node.js que emite la cadena "Hello, world!" en 2 fragmentos antes de completarse. Convertimos este stream en un Web Stream equivalente, luego canalizamos tanto el stream de Node.js de origen como el Web Stream recién creado a la salida estándar.

Este código producirá la siguiente salida:

```text
Hello, Hello, world!world!
```

Esto se debe a que, cada vez que el stream de Node.js de origen emite un fragmento, el mismo fragmento también es emitido por el Web Stream asociado.

Los métodos `.fromWeb()` y `.toWeb()` son implementaciones del patrón Adapter que analizaremos con más detalle en el [Capítulo 8](https://subscription.packtpub.com/book/web-development/9781803238944/8), Patrones de diseño estructurales.

---

### Sección 8: Utilidades para el consumo de streams

Como hemos repetido innumerables veces a lo largo de este capítulo, los streams están diseñados para transferir y procesar grandes cantidades de datos en pequeños fragmentos. Sin embargo, hay situaciones en las que necesitas consumir todo el contenido de un stream y acumularlo en la memoria. Esto es más común de lo que parece, en gran parte porque muchas abstracciones en el ecosistema de Node.js utilizan streams como bloque de construcción fundamental para la transferencia de datos. Este diseño proporciona una gran flexibilidad, pero también significa que a veces es necesario manejar datos fragmento por fragmento manualmente. En tales casos, es importante comprender cómo convertir un stream de fragmentos discretos en una única porción de datos en buffer que se pueda procesar en su totalidad.

Un buen ejemplo de esto es el módulo de bajo nivel `node:http`, que te permite realizar solicitudes HTTP. Al manejar una respuesta HTTP, Node.js representa el cuerpo de la respuesta como un stream Readable. Esto significa que se espera que proceses los datos de la respuesta de forma incremental, a medida que llegan los fragmentos.

Pero, ¿qué sucede si sabes de antemano que el cuerpo de la respuesta contiene un objeto serializado en JSON? En ese caso, no puedes procesar los fragmentos de forma independiente; debes esperar hasta que se haya recibido toda la respuesta para poder analizarla como una cadena completa mediante `JSON.parse()`.

Una implementación simple de este patrón podría verse como el siguiente código:

```javascript
import { request } from 'node:http'

const req = request('http://example.com/somefile.json', res => { // 1
  let buffer = '' // 2
  res.on('data', chunk => {
    buffer += chunk
  })
  res.on('end', () => { // 3
    console.log(JSON.parse(buffer))
  })
})

req.end() // 4
```

Para comprender mejor este ejemplo, analicemos sus puntos principales:
1. Aquí se realiza una solicitud a `http://example.com/somefile.json`. El segundo argumento es un callback que recibe el objeto de respuesta (`res`), que es un stream Readable. Este stream emite fragmentos de datos a medida que llegan a través de la red.
2. Dentro del callback de respuesta, inicializamos una cadena vacía llamada `buffer`. A medida que llega cada fragmento de datos (a través del evento `'data'`), lo concatenamos a la cadena `buffer`. Esto almacena efectivamente todo el cuerpo de la respuesta en la memoria. Este enfoque es necesario cuando necesitas manejar toda la respuesta como una unidad completa; por ejemplo, al analizar JSON, ya que `JSON.parse()` solo funciona en cadenas completas.
3. Una vez que se ha recibido toda la respuesta y no llegarán más datos (evento `'end'`), usamos `JSON.parse()` para deserializar la cadena acumulada en un objeto de JavaScript. Luego, el objeto resultante se registra en la consola.
4. Finalmente, se llama a `req.end()` para señalar que no se enviará ningún cuerpo de solicitud (nuestra solicitud está completa y se puede reenviar). Dado que se trata de una solicitud GET sin cuerpo, es necesario finalizar explícitamente la solicitud.

Un punto final que vale la pena señalar es que este código no requiere `async/await` porque se basa enteramente en callbacks basados en eventos, que es la forma tradicional de manejar operaciones asíncronas en los streams de Node.js.

Esta solución funciona, pero tiene bastante código repetitivo (*boilerplate*). Afortunadamente, existe una mejor solución, gracias al módulo `node:stream/consumers`.

Esta biblioteca integrada se introdujo en la versión 16 de Node.js para exponer varias utilidades que facilitan el consumo de todo el contenido de una instancia Readable de Node.js o una instancia `ReadableStream` de Web Streams.

Este módulo expone el objeto `consumers`, que implementa los siguientes métodos estáticos:
- `consumers.arrayBuffer(stream)`
- `consumers.blob(stream)`
- `consumers.buffer(stream)`
- `consumers.text(stream)`
- `consumers.json(stream)`

Cada uno de estos métodos consume el stream dado y devuelve una Promise que se resuelve solo cuando el stream se ha consumido por completo.

Es fácil adivinar que cada método acumula los datos en un tipo diferente de objeto. `arrayBuffer()`, `blob()` y `buffer()` acumularán fragmentos como datos binarios en una instancia de `ArrayBuffer`, `Blob` o `Buffer`, respectivamente. `text()` acumula datos en un objeto de cadena, mientras que `json()` acumula datos en un objeto de cadena y también intentará deserializar los datos mediante `JSON.parse()` antes de resolver la Promise correspondiente.

Esto significa que podemos reescribir el ejemplo anterior de la siguiente manera:

```javascript
import { request } from 'node:https'
import consumers from 'node:stream/consumers'

const req = request(
  'http://example.com/somefile.json',
  async res => {
    const buffer = await consumers.json(res)
    console.log(buffer)
  }
)

req.end()
```

Mucho más conciso y elegante, ¿verdad?

Si usas `fetch` para realizar solicitudes HTTP(S), el objeto de respuesta proporcionado por la API `fetch` tiene varios consumidores integrados. Podrías reescribir el ejemplo anterior de la siguiente manera:

```javascript
const res = await fetch('http://example.com/somefile.json')
const buffer = await res.json()
console.log(buffer)
```

El objeto de respuesta (`res`) también expone `.blob()`, `.arrayBuffer()` y `.text()` si deseas acumular los datos de respuesta como un buffer binario o como texto. Sin embargo, ten en cuenta que falta el método `.buffer()`. Esto se debe a que la clase `Buffer` no forma parte del estándar web, sino que existe únicamente en Node.js.

---

### Sección 9: Resumen

En este capítulo, arrojamos algo de luz sobre los streams de Node.js y algunos de sus casos de uso más comunes. Aprendimos por qué los streams son tan aclamados por la comunidad de Node.js y dominamos su funcionalidad básica, lo que nos permite descubrir más y navegar cómodamente en este nuevo mundo. Analizamos algunos patrones avanzados y comenzamos a comprender cómo conectar streams en diferentes configuraciones, captando la importancia de la interoperabilidad, que es lo que hace que los streams sean tan versátiles y poderosos.

Si no podemos hacer algo con un solo stream, probablemente podamos hacerlo conectando otros streams juntos, y esto funciona muy bien con la filosofía de una sola cosa por módulo (*one thing per module*). En este punto, debería quedar claro que los streams no son solo una función conveniente de Node.js; son una parte esencial: un patrón crucial para manejar datos binarios, cadenas y objetos. No es casualidad que les hayamos dedicado un capítulo entero.

En los próximos capítulos, nos centraremos en los patrones de diseño orientados a objetos tradicionales. Pero no te dejes engañar; aunque JavaScript es, hasta cierto punto, un lenguaje orientado a objetos, en Node.js a menudo se prefiere el enfoque funcional o híbrido. Deshazte de todo prejuicio antes de leer los próximos capítulos.

---

### Sección 10: Ejercicios

- **6.1 Eficiencia en la compresión de datos:** Escribe un script de línea de comandos que tome un archivo como entrada y lo comprima utilizando los diferentes algoritmos disponibles en el módulo `zlib` (Brotli, Deflate, Gzip). Debes producir una tabla de resumen que compare el tiempo de compresión del algoritmo y la eficiencia de compresión en el archivo dado. *Pista: este podría ser un buen caso de uso para el patrón fork, pero recuerda que hicimos algunas consideraciones importantes sobre el rendimiento cuando lo discutimos anteriormente en este capítulo.*
- **6.2 Procesamiento de datos con streams:** En Kaggle, puedes encontrar muchos conjuntos de datos interesantes, como London Crime Data ([nodejsdp.link/london-crime](https://nodejsdp.link/london-crime)). Puedes descargar los datos en formato CSV y crear un script de procesamiento de streams que analice los datos e intente responder a las siguientes preguntas:
  - ¿El número de delitos aumentó o disminuyó a lo largo de los años?
  - ¿Cuáles son las zonas más peligrosas de Londres?
  - ¿Cuál es el delito más común por zona?
  - ¿Cuál es el delito menos común?
  *Pista: Puedes utilizar una combinación de streams Transform y streams PassThrough para analizar y observar los datos a medida que fluyen. Luego, puedes crear agregaciones en memoria para los datos, lo que puede ayudarte a responder a las preguntas anteriores. Además, no necesitas hacer todo en una sola canalización; podrías crear canalizaciones muy especializadas (por ejemplo, una por pregunta) y usar el patrón fork para distribuir los datos analizados entre ellas.*
- **6.3 Compartir archivos sobre TCP:** Crea un cliente y un servidor para transferir archivos a través de TCP. Puntos extra si agregas una capa de cifrado encima de eso y si puedes transferir varios archivos a la vez. Una vez que tengas tu implementación lista, ¡dale el código del cliente y tu dirección IP a un amigo o colega y pídele que te envíe algunos archivos! *Pista: podrías usar mux/demux para recibir varios archivos a la vez.*
- **6.4 Animaciones con streams Readable:** ¿Sabías que puedes crear animaciones de terminal increíbles solo con streams Readable? Bueno, para entender de qué estamos hablando aquí, ¡intenta ejecutar `curl parrot.live` en tu terminal y mira qué sucede! Si crees que esto es genial, ¿por qué no intentas crear algo similar? *Pista: si necesitas ayuda para descubrir cómo implementar esto, puedes consultar el código fuente real de `parrot.live` simplemente accediendo a su URL a través de tu navegador.*
