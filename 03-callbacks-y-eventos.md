# Parte 1: Fundamentos de Node.js

## Capítulo 3: Callbacks y Eventos

En la programación síncrona, pensamos en el código como una serie de pasos consecutivos que colaboran para resolver un problema específico. Cada operación es bloqueante, lo que significa que una tarea debe completarse antes de que la siguiente pueda comenzar. Este enfoque hace que el código sea directo de leer, entender y depurar.

Por otro lado, la programación asíncrona opera de manera diferente. Ciertas operaciones, como leer de un archivo o realizar una petición de red, se ejecutan "en segundo plano". Cuando iniciamos una operación asíncrona, la siguiente instrucción se ejecuta de inmediato, incluso si la tarea anterior aún no ha finalizado. En este contexto, necesitamos una forma de recibir una notificación cuando la operación asíncrona finalice, de modo que podamos continuar nuestro flujo de ejecución con el resultado de dicha operación. La forma más fundamental de manejar esto en Node.js es a través de los **callbacks**: funciones invocadas por el entorno de ejecución una vez que se completa una operación asíncrona.

Los callbacks son los bloques de construcción fundamentales sobre los que se edifican todos los demás mecanismos asíncronos. Sin callbacks, no tendríamos promesas y, a su vez, no tendríamos async/await. Además, los streams y los eventos también dependen de los callbacks. Por esta razón, comprender cómo funcionan los callbacks es crucial.

Es fácil suponer que los callbacks y los emisores de eventos están algo anticuados u obsoletos en el JavaScript y Node.js modernos, pero nada más lejos de la realidad. Entender verdaderamente los callbacks y los emisores de eventos es esencial para asimilar cómo funcionan las promesas y async/await bajo el capó. Por lo tanto, no te saltes este capítulo: ¡es tu primer paso importante para dominar la programación asíncrona en JavaScript y Node.js!

En este capítulo profundizarás en el patrón Callback de Node.js y explorarás lo que significa escribir código asíncrono en la práctica. Cubriremos convenciones, patrones y trampas comunes. Al final de este capítulo, tendrás un conocimiento sólido de los conceptos básicos del patrón Callback.

También conocerás el patrón Observer, que está estrechamente relacionado con el patrón Callback. El patrón Observer, representado por `EventEmitter` en Node.js, utiliza callbacks para gestionar múltiples eventos heterogéneos y es uno de los componentes más utilizados en la programación con Node.js.

En resumen, esto es lo que aprenderás en este capítulo:
- El patrón Callback, cómo funciona, qué convenciones se utilizan en Node.js y cómo lidiar con sus errores más comunes.
- El patrón Observer y cómo implementarlo en Node.js utilizando la clase `EventEmitter`.
- La comparación entre `EventEmitter` y los callbacks, y cómo combinar ambos para obtener lo mejor de los dos mundos.

---

### Sección 1: El patrón Callback

Los callbacks son la materialización de los manejadores (*handlers*) del patrón Reactor (que cubrimos en el [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1), La plataforma Node.js), un componente central de la arquitectura de Node.js.

Los callbacks se pueden definir como funciones que se activan para gestionar el resultado de una operación, exactamente lo que necesitamos cuando trabajamos con tareas asíncronas. En un contexto asíncrono, los callbacks sustituyen a la típica sentencia `return`, que solo funciona de forma síncrona. JavaScript se adapta especialmente bien a los callbacks, ya que las funciones son objetos de primera clase (*first-class objects*). Esto permite asignarlas fácilmente a variables, pasarlas como argumentos, devolverlas desde otras funciones o almacenarlas en estructuras de datos.

Los callbacks desempeñaron un papel clave en la configuración del distintivo estilo de programación de Node.js durante muchos años. A pesar de la disminución de su popularidad en el código moderno orientado al usuario, siguen siendo un componente fundamental e indispensable tanto del ecosistema de Node.js como del de JavaScript. De hecho, los callbacks son el bloque de construcción para construir Promesas y Async/Await, y son esenciales para comprender estas abstracciones de nivel superior y utilizarlas correctamente. Además, muchas técnicas avanzadas que se explorarán en este capítulo y en los siguientes requieren una sólida comprensión de los callbacks. Es por ello que en este libro hemos optado por comenzar nuestra exploración del dominio de los patrones asíncronos desde la perspectiva de los callbacks.

Las clausuras (*closures*) también son un socio ideal para los callbacks. Permiten que una función conserve el acceso al entorno en el que fue creada, garantizando que el contexto de la operación asíncrona se preserve, sin importar cuándo o dónde se invoque finalmente el callback.

Si necesitas refrescar tus conocimientos sobre clausuras, puedes consultar el artículo en MDN Web Docs en [nodejsdp.link/mdn-closures](https://nodejsdp.link/mdn-closures).

En esta sección, examinaremos más de cerca este estilo de programación que se basa en callbacks en lugar de sentencias de retorno tradicionales.

#### El estilo de paso de continuación (*The continuation-passing style*)

En JavaScript (y por tanto en Node.js), un callback es una función que se pasa como argumento a una función $f$. Esta función callback se llamará eventualmente con el resultado de la operación cuando $f$ se complete. En programación funcional, esta forma de propagar el resultado se denomina **estilo de paso de continuación** (*Continuation-Passing Style* o **CPS**).

Es un concepto general y no siempre está asociado a operaciones asíncronas. De hecho, simplemente indica que un resultado se propaga pasándolo a otra función (el callback), en lugar de devolverlo directamente al invocador.

##### CPS Síncrono (*Synchronous CPS*)

Para aclarar el concepto, comencemos con una función síncrona simple:

```javascript
function add(a, b) {
  return a + b
}
```

Nada fuera de lo común aquí: solo una función de adición simple que toma dos números y devuelve su suma. El resultado se devuelve al invocador mediante una sentencia `return`. Esto se conoce como **estilo directo** (*direct style*) y es la forma más común de devolver resultados en la programación síncrona en la mayoría de los lenguajes de programación (incluido JavaScript).

Ahora, he aquí la función equivalente utilizando el Estilo de Paso de Continuación (CPS):

```javascript
function addCps(a, b, callback) {
  callback(a + b)
}
```

Si comparamos la función `addCps()` con la función `add()` descrita anteriormente, podemos observar 2 diferencias principales:
1. `addCps()` espera un argumento adicional: una función callback.
2. En lugar de usar una sentencia `return`, `addCps()` propaga el resultado de la operación invocando la función callback y pasando el resultado de la operación (`a + b`) como argumento.

La función `addCps()` es una función CPS síncrona. Sigue siendo síncrona porque durante su ejecución no ocurre nada asíncrono. Una vez que se llama a `addCps()`, se calcula `a + b` y el callback se invoca inmediatamente. Además, `addCps()` completa su ejecución solo cuando lo hace el callback.

En términos más generales, la idea de CPS es que cuando llamamos a una función, también pasamos una función callback. Una vez finalizado el cálculo, se invoca el callback con el resultado. En cierto sentido, le estamos diciendo a la función que "pase el testigo" al callback cuando haya terminado.

En este ejemplo particular, dado que esta operación es síncrona, el callback se llama de inmediato. Aclaremos esto aún más con un ejemplo:

```javascript
console.log('before') // 1
addCps(1, 2, result => console.log(`Result: ${result}`)) // 2
console.log('after') // 3
```

El código anterior imprimirá lo siguiente:

```text
before
Result: 3
after
```

Intentemos dar sentido a esta salida:
1. La primera línea de código realiza una llamada síncrona que registra la palabra `before` en la salida estándar.
2. En la segunda línea invocamos `addCps()` y le pasamos `1`, `2` y un callback. La función callback es una función flecha definida en línea. Si revisamos de nuevo la implementación de `addCps()`, podemos ver que la función callback se invocará inmediatamente con el resultado de la suma entre nuestros 2 primeros argumentos ($1 + 2 = 3$). El cuerpo de la función callback, por lo tanto, registra la cadena `Result: 3`.
3. En la última línea, simplemente registramos la palabra `after`.

Llegados a este punto, puede que todavía no estés 100% convencido de que todo lo que sucede aquí sea síncrono. Por ejemplo, podrías preguntarte: ¿cómo sabemos que `console.log()` es una función síncrona?

Esta es una pregunta legítima y podríamos responderla consultando la documentación de esta función, pero, en general, a medida que exploremos más funciones síncronas y asíncronas, aprenderemos a reconocer patrones e modismos comunes y desarrollaremos un instinto para saber cuándo el código se ejecuta de forma síncrona o asíncrona.

Veamos, pues, cómo funciona el CPS asíncrono.

##### CPS Asíncrono (*Asynchronous CPS*)

Veamos ahora una implementación alternativa de `addCps()` que se comporta de forma asíncrona. Llamémosla `addAsync()`:

```javascript
function addAsync(a, b, callback) {
  setTimeout(() => callback(a + b), 100)
}
```

En este ejemplo, introducimos un retraso artificial usando `setTimeout()` para simular un comportamiento asíncrono. La función `setTimeout()` programa una tarea en la cola de eventos para que se ejecute después de un número específico de milisegundos, convirtiendo esto en una operación asíncrona. Si bien este es un ejemplo simple destinado a demostrar el patrón, puedes imaginar escenarios del mundo real donde la función podría necesitar obtener datos externos, como una función de conversión de moneda que recupera los tipos de cambio más recientes antes de realizar la conversión.

Acabamos de aprender que `setTimeout()` es una función asíncrona y, al usarla, hemos transformado nuestra función `addAsync()` en una asíncrona también. Las funciones asíncronas a menudo se construyen mediante el uso de funciones asíncronas de nivel inferior como `setTimeout()`, `setInterval()`, `setImmediate()`, `process.nextTick()`, `fetch()`, `http.request()`, etc. La documentación de estas funciones puede revelar su naturaleza asíncrona, pero como regla general, siempre que tratemos con temporizadores o esperemos a que se completen operaciones de entrada/salida (como leer de un archivo o hacer una petición HTTP), nos encontraremos con alguna forma de abstracción asíncrona.

Probemos a usar `addAsync()` y observemos cómo cambia el orden de las operaciones:

```javascript
console.log('before')
addAsync(1, 2, result => console.log(`Result: ${result}`))
console.log('after')
```

El código anterior imprimirá lo siguiente:

```text
before
after
Result: 3
```

Dado que `setTimeout()` desencadena una operación asíncrona, no espera a que se ejecute el callback; en su lugar, regresa de inmediato, devolviendo el control a `addAsync()`, y luego de vuelta a su invocador. Esta propiedad en Node.js es crucial, ya que devuelve el control al bucle de eventos tan pronto como se envía una solicitud asíncrona, permitiendo así que se procese un nuevo evento de la cola.

**Figura 3.1:** Flujo de control de la invocación de una función asíncrona.

La Figura 3.1 muestra paso a paso cómo se ejecuta este código:
1. `console.log('before')` se ejecuta: Esto registra el mensaje `before` en la consola.
2. `addAsync()` se ejecuta: El control se transfiere a la función `addAsync()`, donde se pasan dos argumentos y un callback.
3. `setTimeout()` se invoca: Dentro de `addAsync()`, la función `setTimeout()` programa el callback para que se ejecute después de un retraso especificado (en este caso, 100 milisegundos).
4. El control se devuelve de `setTimeout()` a `addAsync()`: Como `setTimeout()` es asíncrono, no espera a que se complete el retraso antes de devolver el control.
5. El control vuelve a la función invocadora: Dado que `setTimeout()` es la última (y única) instrucción en `addAsync()`, `addAsync()` retorna. El control pasa a la siguiente línea de código, `console.log('after')`, que se ejecuta inmediatamente imprimiendo `after`.
6. El control pasa al bucle de eventos: No hay más instrucciones para ejecutar, por lo que el control vuelve al bucle de eventos. El bucle de eventos es consciente de que hay tareas pendientes (el temporizador asociado a la llamada `setTimeout()`), por lo que esperará a que se complete la operación asíncrona.
7. La tarea asíncrona se completa: Cuando el tiempo de espera concluye después de 100 milisegundos, se invoca el callback pasado a `setTimeout()`.
8. El callback se ejecuta y recibe como argumento el resultado de la expresión `a + b`.
9. `console.log('Result: ${result}')` se invoca: Este es el cuerpo de la función callback proporcionada. Llamamos al argumento `result`, por lo que esto imprimirá `"Result: 3"` (ya que `a` es 1 y `b` es 2) en la consola.
10. El control vuelve al bucle de eventos: Después de que el callback finaliza su ejecución, el control regresa al bucle de eventos, que ahora está listo para manejar otras tareas pendientes o, si no hay otras tareas, salir.

Es importante enfatizar que cuando se ejecuta un callback, comienza con una pila de llamadas (*call stack*) nueva, y comienza desde el bucle de eventos. Aquí es donde JavaScript verdaderamente destaca. Gracias a las clausuras (*closures*), resulta sencillo retener el contexto del invocador original, incluso si el callback se invoca más tarde o desde una parte diferente del código. Como exploraremos en los ejemplos posteriores de este capítulo, esto convierte al CPS asíncrono en un estilo de programación sumamente eficaz.

La **pila de llamadas** (*call stack*) es el mecanismo utilizado por el intérprete de JavaScript para realizar un seguimiento del contexto de ejecución. Hace posible saber dónde se encuentra actualmente el programa y qué funciones se están ejecutando. Cuando se llama a una función, se añade a la pila de llamadas. Si esa función llama a otra función, esta también se agrega a la pila. Una vez que una función finaliza, se retira de la pila y el intérprete reanuda la ejecución en el nivel anterior. Cuando una función callback se ejecuta a través del bucle de eventos, comienza con una pila de llamadas limpia. Este contexto de ejecución nuevo es fundamental para el comportamiento asíncrono de JavaScript. Las clausuras hacen posible retener el contexto original, asegurando que los datos y las variables sigan siendo accesibles cuando el callback finalmente se ejecute. Esta capacidad de retener el contexto hace que las clausuras sean indispensables para la programación en Estilo de Paso de Continuación, donde las funciones se pasan como argumentos y se ejecutan más tarde, a menudo de forma asíncrona. Las clausuras salvan la brecha entre la naturaleza temporal de la pila de llamadas y los datos persistentes necesarios para los callbacks.

##### Callbacks no-CPS (*Non-CPS callbacks*)

Existen muchas situaciones en las que la presencia de un callback podría llevarnos a asumir que una función es asíncrona o que utiliza CPS, pero no siempre es así. Tomemos como ejemplo el método `map()` de un array:

```javascript
const result = [1, 5, 7].map(element => element - 1)
console.log(result) // [0, 4, 6]
```

Aquí, el callback se utiliza simplemente para iterar sobre los elementos del array, no para gestionar el resultado de la operación. De hecho, el resultado se devuelve de forma síncrona mediante el estilo directo. No hay distinción sintáctica entre callbacks que no son CPS y los que sí lo son, razón por la cual el uso previsto de un callback debe explicarse claramente en la documentación de la API.

En la siguiente sección, profundizaremos en una de las trampas más comunes con los callbacks que todo desarrollador de Node.js debe conocer.

#### ¿Síncrono o asíncrono?

Has visto cómo el orden de ejecución de las instrucciones cambia radicalmente dependiendo de la naturaleza de una función: síncrona o asíncrona. Esto tiene fuertes repercusiones en el flujo de toda la aplicación, tanto en términos de corrección como de eficiencia. A continuación se presenta un análisis de estos dos paradigmas y sus dificultades. En general, lo que debe evitarse es crear inconsistencia y confusión en torno a la naturaleza de una API, ya que hacerlo puede dar lugar a un conjunto de problemas que pueden ser muy difíciles de detectar y reproducir. Para guiar nuestro análisis, tomaremos como ejemplo el caso de una función inconsistentemente asíncrona.

##### Escribir una función inconsistente

Una de las situaciones más peligrosas es tener una API que se comporta de forma síncrona bajo ciertas condiciones y de forma asíncrona bajo otras.

Imagina un escenario común en el que queremos crear una abstracción que obtenga el contenido de un archivo desde el disco y lo almacene en caché para agilizar futuras lecturas. He aquí una implementación potencial (pero defectuosa):

```javascript
import { readFile } from 'node:fs'

const cache = new Map()

function inconsistentRead(filename, cb) {
  if (cache.has(filename)) {
    // invoked synchronously
    cb(cache.get(filename))
  } else {
    // asynchronous function
    readFile(filename, 'utf8', (_err, data) => {
      cache.set(filename, data)
      cb(data)
    })
  }
}
```

Esta función utiliza la variable `cache` para almacenar el contenido de los archivos. Es un ejemplo muy básico con algunos problemas intrínsecos: carece de manejo de errores (por ejemplo, qué sucede si falla la lectura del archivo) y la lógica de almacenamiento en caché podría mejorarse significativamente (en el [Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas, aprenderás a gestionar el almacenamiento en caché asíncrono adecuadamente).

Sin embargo, el verdadero problema radica en su inconsistencia: la función es asíncrona la primera vez que se lee un archivo, pero se vuelve síncrona una vez que el archivo está en caché. Este comportamiento impredecible puede acarrear problemas inesperados, ¡creando una receta para todo tipo de caos! ¡Descubramos cómo las cosas pueden salir terriblemente mal con este enfoque!

##### Liberando a Zalgo (*Unleashing Zalgo*)

Veamos ahora cómo el uso de una función inconsistente, aquella que puede comportarse de forma síncrona en algunas llamadas y de forma asíncrona en otras, como la función `inconsistentRead()` definida anteriormente, puede romper una aplicación de formas sutiles y no evidentes. El siguiente ejemplo ilustra uno de tales efectos secundarios imprevistos:

```javascript
function createFileReader(filename) {
  const listeners = []
  inconsistentRead(filename, (value) => {
    for (const listener of listeners) {
      listener(value)
    }
  })
  return {
    onDataReady: listener => listeners.push(listener)
  }
}
```

Cuando se invoca la función anterior, crea un nuevo objeto que actúa como notificador, lo que nos permite configurar múltiples escuchadores (*listeners*) para una operación de lectura de archivos. Todos los escuchadores se invocarán a la vez cuando se complete la operación de lectura y los datos estén disponibles. La función anterior utiliza nuestra función `inconsistentRead()` para implementar esta funcionalidad. Veamos cómo usar la función `createFileReader()` para leer un archivo `data.txt` del directorio de trabajo actual que contiene el texto `some data`:

```javascript
const reader1 = createFileReader('data.txt')
reader1.onDataReady(data => {
  console.log(`First call data: ${data}`)
  // ...sometime later we try to read again from
  // the same file
  const reader2 = createFileReader('data.txt')
  reader2.onDataReady(data => {
    console.log(`Second call data: ${data}`)
  })
})
```

El código anterior imprimirá el siguiente texto:

```text
First call data: some data
```

... ¡Y nada más! ¿Qué pasó con nuestro segundo `console.log()`? ¿Por qué no lo vemos en nuestra salida? Revisemos el código con más atención para descubrir el motivo:
1. Durante la creación de `reader1`, nuestra función `inconsistentRead()` se comporta de forma asíncrona porque no hay ningún resultado almacenado en caché para `data.txt`. Esto significa que cualquier escuchador `onDataReady` se invocará más tarde en otro ciclo del bucle de eventos, por lo que disponemos de todo el tiempo necesario para registrar nuestro escuchador.
2. Luego, `reader2` se crea en un ciclo del bucle de eventos en el que la caché para `data.txt` ya existe. En este caso, la llamada interna a `inconsistentRead()` será síncrona. Por lo tanto, su callback se invocará de inmediato, lo que significa que todos los escuchadores de `reader2` se invocarán de forma síncrona también. Sin embargo, estamos registrando el escuchador *después* de la creación de `reader2`, por lo que nunca llegará a invocarse.

El comportamiento del callback de nuestra función `inconsistentRead()` es realmente impredecible, ya que depende de muchos factores, como la frecuencia de su invocación, el nombre de archivo pasado como argumento y la cantidad de tiempo necesaria para cargar el archivo.

Este tipo de comportamiento inconsistente puede dar lugar a errores sumamente complicados de identificar y reproducir en una aplicación real. Imagina utilizar una función similar en un servidor web, donde pueden concurrir múltiples solicitudes simultáneas. Imagina ver que algunas de esas solicitudes quedan colgadas, sin ningún motivo aparente y sin que se registre ningún error.

Isaac Z. Schlueter, el creador de npm y exlíder del proyecto Node.js, comparó una vez el uso de este tipo de función impredecible con "liberar a Zalgo" (*unleashing Zalgo*) en una de las entradas de su blog.

Zalgo es una leyenda de Internet sobre una entidad terrorífica que se dice que trae el caos, la locura e incluso el fin del mundo. Si no has oído hablar de Zalgo, te animo a que lo busques: ¡sin duda vale la pena leerlo!

Puedes encontrar el artículo original de Isaac Z. Schlueter en [nodejsdp.link/unleashing-zalgo](https://nodejsdp.link/unleashing-zalgo).

##### Uso de APIs síncronas

La lección que debemos aprender del ejemplo de "liberar a Zalgo" es que es imperativo que una API defina claramente su naturaleza: síncrona o asíncrona; mezclar ambas puede acarrear toda clase de problemas.

Una forma de solucionar nuestra función `inconsistentRead()` es haciéndola totalmente síncrona. Esto es viable porque Node.js ofrece APIs síncronas para la mayoría de las operaciones básicas de E/S. Por ejemplo, podemos usar la función `readFileSync()` en lugar de su versión asíncrona. He aquí el código actualizado:

```javascript
import { readFileSync } from 'node:fs'

const cache = new Map()

function consistentReadSync(filename) {
  if (cache.has(filename)) {
    return cache.get(filename)
  }
  const data = readFileSync(filename, 'utf8')
  cache.set(filename, data)
  return data
}
```

Notarás que la función ahora utiliza el estilo directo, lo que significa que devuelve los datos directamente en lugar de utilizar callbacks. Dado que es síncrona, no hay necesidad de un callback. De hecho, es una buena práctica utilizar el estilo directo para las APIs síncronas a fin de mantener las cosas simples y claras.

> **Patrón:**
> Elige siempre un estilo directo para funciones puramente síncronas.

Ten en cuenta que cambiar una API de estilo callback (CPS) a estilo directo, o de asíncrona a síncrona (o viceversa), es un cambio disruptivo (*breaking change*). Esto significa que cualquier código que utilice la API también deberá actualizarse.

Sin embargo, el uso de una API síncrona en lugar de una asíncrona tiene algunas advertencias:
- Es posible que no siempre esté disponible una API síncrona para una funcionalidad específica.
- Una API síncrona bloqueará el bucle de eventos y pondrá en espera cualquier solicitud concurrente. Esto entra en conflicto con el modelo de concurrencia de Node.js, ralentizando toda la aplicación. Discutiremos esto con mayor detalle en el [Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas.

En nuestra función `consistentReadSync()`, el riesgo de bloquear el bucle de eventos se mitiga parcialmente porque la API síncrona de E/S se invoca solo una vez por nombre de archivo, mientras que el valor almacenado en caché se utilizará para todas las invocaciones posteriores. Si tenemos un número limitado de archivos estáticos, el uso de `consistentReadSync()` no tendrá un gran impacto en nuestro bucle de eventos. Las cosas pueden cambiar rápidamente si tenemos que leer muchos archivos y solo una vez.

El uso de E/S síncrona en Node.js generalmente se desaconseja, pero en algunos casos puede ser la opción más fácil y eficiente. Si estás construyendo un servidor web que maneja muchas solicitudes concurrentes, es fundamental utilizar APIs asíncronas para evitar bloquear el bucle de eventos, asegurando que todas las solicitudes se procesen simultáneamente. Por otro lado, si estás creando un script de automatización que se ejecuta desde la línea de comandos y no necesitas concurrencia, es perfectamente aceptable utilizar APIs síncronas: pueden hacer que el código sea más simple y fácil de administrar.

A veces, incluso podrías utilizar una combinación de ambas. Por ejemplo, es posible que necesites cargar un archivo de configuración antes de iniciar un servidor web. En este caso, tiene sentido utilizar una API síncrona para cargar el archivo, pero la lógica de gestión de solicitudes debe utilizar código asíncrono y no bloqueante tanto como sea posible.

Evalúa siempre tu caso de uso específico para elegir el enfoque correcto.

> **Patrón:**
> Utiliza APIs bloqueantes con moderación y solo cuando no afecten la capacidad de la aplicación para gestionar operaciones asíncronas concurrentes.

##### Garantizar la asincronía mediante ejecución diferida

Otra alternativa para corregir nuestra función `inconsistentRead()` es hacerla puramente asíncrona. El truco aquí consiste en programar la invocación del callback síncrono para que se ejecute "en el futuro" en lugar de ejecutarse inmediatamente en el mismo ciclo del bucle de eventos. En Node.js, esto es posible con `process.nextTick()`, que difiere la ejecución de una función hasta después de que se complete la operación que se está ejecutando actualmente. Su funcionalidad es muy sencilla: toma un callback como argumento y lo coloca al principio de la cola de eventos, por delante de cualquier evento de E/S pendiente, y regresa inmediatamente. El callback se invocará tan pronto como la operación en ejecución ceda el control al bucle de eventos.

Apliquemos esta técnica para solucionar los problemas encontrados en la función `inconsistentRead()`:

```javascript
import { readFile } from 'node:fs'

const cache = new Map()

function consistentReadAsync(filename, callback) {
  if (cache.has(filename)) {
    // deferred callback invocation
    process.nextTick(() => callback(cache.get(filename)))
  } else {
    // asynchronous function
    readFile(filename, 'utf8', (_err, data) => {
      cache.set(filename, data)
      callback(data)
    })
  }
}
```

Ahora, gracias a `process.nextTick()`, se garantiza que nuestra función invocará su callback de forma asíncrona bajo cualquier circunstancia. Prueba a usarla en lugar de la función `inconsistentRead()` y verifica que, efectivamente, Zalgo ha sido erradicado.

> **Patrón:**
> Puedes garantizar que un callback se invoque de forma asíncrona difiriendo su ejecución mediante `process.nextTick()`.

Es posible que hayas visto otra API para diferir la ejecución de código: `setImmediate()`. Aunque su nombre sugiere que ejecuta callbacks de inmediato, esto puede resultar un poco engañoso. En realidad, `setImmediate()` otorga a los callbacks una prioridad más baja que los programados con `process.nextTick()` o incluso con `setTimeout(callback, 0)`. ¡Sí, nombrar cosas es difícil!

Los callbacks diferidos con `process.nextTick()` se denominan **microtareas** (*microtasks*) y se ejecutan justo después de que se completa la operación actual, incluso antes de que se dispare cualquier otro evento de E/S. Con `setImmediate()`, por otro lado, la ejecución se pone en cola en una fase del bucle de eventos que tiene lugar después de que se hayan procesado todos los eventos de E/S.

Esto significa que los callbacks programados con `process.nextTick()` siempre se ejecutarán antes, pero en ciertas situaciones, como al usar recursión, puede llevar a la **inanición de E/S** (*I/O starvation*), retrasando indefinidamente los callbacks de E/S. Por ejemplo, si programas recursivamente callbacks dentro de un callback de `process.nextTick()`, se seguirán acumulando más callbacks en la cola. Dado que el bucle de eventos prioriza esta cola sobre otros callbacks, no tendrá la oportunidad de procesar callbacks de eventos de E/S, como la lectura de un archivo. Podemos apreciar este concepto en el siguiente ejemplo:

```javascript
import { readFile } from 'node:fs'

readFile('data.txt', 'utf8', (_err, data) => {
  console.log(`Data from file: ${data}`)
})

let scheduledNextTicks = 0
function recursiveNextTick() {
  if (scheduledNextTicks++ >= 1000) {
    return
  }
  console.log('Keeping the event loop busy')
  process.nextTick(() => recursiveNextTick())
}

recursiveNextTick()
```

En este ejemplo, leemos el contenido de un archivo usando `readFile()` y proporcionamos un callback para registrar los datos una vez que estén disponibles. Después de eso, mantenemos artificialmente ocupado el bucle de eventos programando recursivamente 1000 tareas usando `process.nextTick()`. Cada tarea simplemente imprime `Keeping the event loop busy` en la consola.

Si ejecutamos este código, la salida tendrá este aspecto:

```text
Keeping the event loop busy
Keeping the event loop busy
Keeping the event loop busy
…
Data from file: some data
```

Hemos acortado la salida, pero en realidad verás `Keeping the event loop busy` impreso 1000 veces antes de que aparezca el contenido del archivo. Esto demuestra que, si bien `process.nextTick()` es útil para ejecutar callbacks de alta prioridad, puede resultar problemático si se utiliza con demasiada frecuencia.

Mencionamos que otra forma común de programar callbacks para que se ejecuten lo antes posible es `setTimeout(callback, 0)`, que se comporta de manera similar a `setImmediate()`. Sin embargo, en escenarios típicos, los callbacks programados con `setImmediate()` se ejecutan después de los programados con `setTimeout(callback, 0)`.

Para entender por qué, debemos observar cómo maneja el bucle de eventos los callbacks en diferentes fases. Para los eventos que estamos discutiendo, los temporizadores (`setTimeout()`) se ejecutan antes de los callbacks de E/S, que a su vez se procesan antes de los callbacks de `setImmediate()`. Podemos tomar prestado (con una ligera modificación) un excelente ejemplo del sitio web oficial de Node.js ([nodejsdp.link/next-tick](https://nodejsdp.link/next-tick)):

```javascript
setImmediate(() => {
  console.log('setImmediate(cb)')
})
setTimeout(() => {
  console.log('setTimeout(cb, 0)')
}, 0)
process.nextTick(() => {
  console.log('process.nextTick(cb)')
})
console.log('Sync operation')
```

Tómate un momento para pensar en la salida.

¿Listo? Esto es lo que verás:

```text
Sync operation
process.nextTick(cb)
setTimeout(cb, 0)
setImmediate(cb)
```

Esto nos da una imagen clara del orden de prioridad del bucle de eventos para diferentes tipos de eventos.

Si todavía te sientes confundido sobre cómo funcionan JavaScript y Node.js bajo el capó, o si te preguntas cómo se utilizan la pila, el bucle de eventos y las diferentes colas en distintos momentos de la ejecución de un programa, te recomendamos consultar este increíble simulador interactivo: [nodejsdp.link/js-visualizer](https://nodejsdp.link/js-visualizer).

Las diferencias entre estas APIs quedarán aún más claras más adelante en el libro, cuando analicemos la ejecución de tareas síncronas limitadas por CPU ([Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas).

A continuación, exploraremos las convenciones para definir callbacks en Node.js.

#### Convenciones de callbacks en Node.js

En Node.js, las APIs CPS y los callbacks siguen un conjunto de convenciones específicas. Estas convenciones no solo se encuentran en la biblioteca central, sino que la mayoría de los módulos y aplicaciones de terceros también las siguen. Para diseñar una API asíncrona que utilice callbacks de manera eficaz, es fundamental que comprendas estas pautas y te apegues a ellas.

##### El callback es el último argumento

En todas las funciones del núcleo de Node.js, la convención estándar es que cuando una función acepta un callback como entrada, este debe pasarse como el último argumento.

Tomemos como ejemplo la siguiente API central de Node.js:

```javascript
readFile(filename, [options], callback)
```

Como puedes ver en la firma de la función anterior, el callback siempre se coloca en la última posición, incluso en presencia de argumentos opcionales como `options` en este ejemplo. La razón de esta convención es que la llamada a la función resulta más legible en caso de que el callback se defina en el mismo lugar (*in place*).

##### Cualquier error siempre va primero

En CPS, los errores se propagan como cualquier otro tipo de resultado, lo que significa usar callbacks. En Node.js, cualquier error producido por una función CPS siempre se pasa como el primer argumento del callback, y cualquier resultado real se pasa a partir del segundo argumento. Si la operación tiene éxito sin errores, el primer argumento será `null` o `undefined`.

El siguiente código muestra cómo definir un callback que cumple con esta convención:

```javascript
readFile('foo.txt', 'utf8', (err, data) => {
  if (err) {
    handleError(err)
  } else {
    processData(data)
  }
})
```

Es una buena práctica verificar siempre la presencia de un error, ya que no hacerlo dificultará la depuración de tu código y la detección de posibles puntos de falla. Otra convención importante a considerar es que el error siempre debe ser una instancia de la clase `Error`. Esto significa que nunca deben pasarse cadenas o números simples como objetos de error.

##### Propagación de errores

La propagación de errores en funciones síncronas de estilo directo se realiza con la conocida sentencia `throw`, que hace que el error suba por la pila de llamadas hasta que sea capturado:

```javascript
throw new Error('Something went wrong')
```

En CPS asíncrono, sin embargo, la propagación adecuada de errores se realiza simplemente pasando el error al siguiente callback en la cadena. El patrón típico se estructura de la siguiente manera:

```javascript
import { readFile } from 'node:fs'

function readJson(filename, callback) {
  readFile(filename, 'utf8', (err, data) => {
    let parsed
    if (err) {
      // error reading the file
      // propagate the error and exit the current function
      return callback(err)
    }
    try {
      // parse the file contents
      parsed = JSON.parse(data)
    } catch (err) {
      // catch parsing errors
      return callback(err)
    }
    // no errors, propagate just the data
    callback(null, parsed)
  })
}
```

Observa que no lanzamos ni retornamos errores con `throw`. El primer posible error puede ocurrir cuando usamos la operación `readFile()`. Dentro de su callback, el primer argumento que recibimos es `err` y representa un error relacionado con la lectura del archivo dado. No lo lanzamos ni lo retornamos; en su lugar, simplemente invocamos el callback con el error para propagarlo de vuelta al invocador de `readJson()`. La sentencia `return` se usa únicamente para detener la ejecución de la función, no para devolver un valor específico.

El siguiente error potencial con el que debemos lidiar es cuando llamamos a `JSON.parse()`. Esta es una función síncrona y, por lo tanto, si intentamos parsear JSON no válido, utilizará la instrucción tradicional `throw` para propagar los errores al invocador. Esto significa que esta vez necesitamos usar un bloque `try/catch` para capturar los errores. Nuevamente, en la rama `catch` invocamos `callback(err)` para propagar el error al invocador y usamos `return` para detener la ejecución de la función. Finalmente, si todo salió bien, se invoca a `callback` con `null` como primer argumento para indicar que no hay errores.

También es interesante notar cómo nos abstuvimos de invocar al callback desde dentro del bloque `try`. Esto se debe a que hacerlo capturaría cualquier error lanzado por la ejecución del propio callback, lo cual no suele ser lo que deseamos.

##### Evitar excepciones no capturadas (*Avoiding uncaught exceptions*)

A veces puede ocurrir que se lance un error y no se capture dentro del callback de una función asíncrona. Esto podría suceder si, por ejemplo, olvidamos rodear `JSON.parse()` con un `try...catch` y luego nuestra función pasa a analizar JSON no válido en tiempo de ejecución.

Lanzar un error dentro de un callback asíncrono provocaría que el error salte hasta el bucle de eventos, por lo que nunca se propagaría al siguiente callback. En Node.js, este es un estado irrecuperable y la aplicación simplemente se cerrará con un código de salida distinto de cero, imprimiendo el seguimiento de la pila (*stack trace*) en la interfaz `stderr`.

Para ver esto en acción, intentemos eliminar el bloque `try...catch` que rodea a `JSON.parse()` de la función `readJson()` que definimos anteriormente:

```javascript
function readJsonThrows(filename, callback) {
  readFile(filename, 'utf8', (err, data) => {
    if (err) {
      return callback(err)
    }
    callback(null, JSON.parse(data))
  })
}
```

Ahora, en la función que acabamos de definir, no hay forma de capturar una eventual excepción proveniente de `JSON.parse()`. Intentemos parsear un archivo JSON no válido con el siguiente código:

```javascript
readJsonThrows('invalid_json.json', (err) => console.error(err))
```

Aunque hemos proporcionado un callback que debería poder manejar los errores, dado que la implementación de `readJsonThrows()` no llama al callback sino que simplemente lanza un error, esto provocará que la aplicación se interrumpa abruptamente, imprimiéndose en la consola un seguimiento de la pila similar al siguiente:

```text
SyntaxError: Unexpected token h in JSON at position 1
    at JSON.parse (<anonymous>)
    at file:///.../03-callbacks-and-events/10-uncaught-errors/index.js:8:25
    at FSReqCallback.readFileAfterClose [as oncomplete] (internal/fs/read/context.js:68:3)
```

Ahora bien, si observas el seguimiento de pila anterior de abajo hacia arriba, verás que comienza dentro del módulo integrado `fs`, y exactamente desde el punto en el que la API nativa completó la lectura y devolvió su resultado a la función `fs.readFile()`, a través del bucle de eventos. Esto muestra claramente que la excepción viajó desde nuestro callback, subió por la pila de llamadas y fue directa al bucle de eventos, donde finalmente fue capturada y arrojada a la consola.

Esto también significa que envolver la invocación de `readJsonThrows()` con un bloque `try...catch` no nos ayudará a capturar el error, porque la pila en la que opera el bloque es diferente de aquella en la que se invoca nuestro callback. El siguiente código muestra el antipatrón que se acaba de describir:

```javascript
try {
  readJsonThrows('invalid_json.json', (err) => console.error(err))
} catch (err) {
  console.log('This will NOT catch the JSON parsing exception')
}
```

La sentencia `catch` anterior nunca recibirá el error de parseo de JSON, ya que viajará por la pila de llamadas en la que se lanzó el error, es decir, en el bucle de eventos y no en la función que desencadenó la operación asíncrona.

Como se mencionó anteriormente, la aplicación se interrumpirá en el momento en que una excepción alcance el bucle de eventos. Sin embargo, todavía tenemos la oportunidad de realizar algunas tareas de limpieza o registro antes de que la aplicación finalice. De hecho, cuando una excepción termina en el bucle de eventos, Node.js emitirá un evento especial llamado `uncaughtException`, justo antes de salir del proceso. El siguiente código muestra un caso de uso de muestra:

```javascript
process.on('uncaughtException', (err) => {
  console.error(`This will catch at last the JSON parsing exception: ` +
    `${err.message}`)
  // Terminates the application with 1 (error) as exit code.
  // Without the following line, the application would continue
  process.exit(1)
})
```

Es importante comprender que una excepción no capturada deja a la aplicación en un estado que no se garantiza que sea consistente, lo que puede provocar problemas imprevisibles. Por ejemplo, es posible que todavía haya solicitudes de E/S incompletas en ejecución o que las clausuras se hayan vuelto inconsistentes. Es por eso que siempre se aconseja, especialmente en producción, no dejar nunca la aplicación ejecutándose después de recibir una excepción no capturada. En su lugar, el proceso debe salir inmediatamente, opcionalmente después de haber ejecutado algunas tareas de limpieza necesarias y, de forma ideal, un proceso de supervisión debe reiniciar la aplicación. Esto también se conoce como el enfoque **fail-fast** (*fallo rápido*) y es la práctica recomendada en Node.js; se discutirá con más detalle en el [Capítulo 12](https://subscription.packtpub.com/book/web-development/9781803238944/12), Patrones de arquitectura y escalabilidad.

Esto concluye nuestra introducción al patrón Callback. Ahora es el momento de conocer el patrón Observer, que es otro componente crítico de una plataforma orientada a eventos como Node.js.

---

### Sección 2: El patrón Observer

Otro patrón importante y fundamental utilizado en Node.js es el patrón Observer. Junto con el patrón Reactor y los callbacks, el patrón Observer es un requisito absoluto para dominar el mundo asíncrono de Node.js.

El patrón Observer es la solución ideal para modelar la naturaleza reactiva de Node.js y un complemento perfecto para los callbacks. Demos una definición formal:

> El **patrón Observer** define un objeto (llamado *sujeto*) que puede notificar a un conjunto de observadores (o escuchadores) cuando se produce un cambio en su estado.

La principal diferencia con respecto al patrón Callback es que el sujeto puede notificar a múltiples observadores, mientras que un callback CPS tradicional normalmente propagará su resultado a un solo escuchador: el callback.

#### El EventEmitter

En la programación tradicional orientada a objetos, el patrón Observer requiere interfaces, clases concretas y una jerarquía. En Node.js, todo esto se vuelve mucho más simple. El patrón Observer ya está integrado en el núcleo y está disponible a través de la clase `EventEmitter`. La clase `EventEmitter` nos permite registrar una o más funciones como escuchadores, que se invocarán cuando se dispare un tipo de evento particular. La Figura 3.2 explica visualmente este concepto:

**Figura 3.2:** Escuchadores que reciben eventos de un EventEmitter.

`EventEmitter` se exporta desde el módulo central `node:events`. El siguiente código muestra cómo podemos obtener una referencia al mismo:

```javascript
import { EventEmitter } from 'node:events'
const emitter = new EventEmitter()
```

Los métodos esenciales de `EventEmitter` son los siguientes:
- `on(event, listener)`: Este método nos permite registrar un nuevo escuchador (una función) para el tipo de evento dado (una cadena).
- `once(event, listener)`: Este método registra un nuevo escuchador, que luego se elimina después de que el evento se emite por primera vez.
- `emit(event, [arg1], [...])`: Este método produce un nuevo evento y proporciona argumentos adicionales para pasarlos a los escuchadores.
- `removeListener(event, listener)`: Este método elimina un escuchador para el tipo de evento especificado. Ten en cuenta que este método también tiene un alias: `off(event, listener)`.
- `removeAllListeners(event)`: Elimina todos los escuchadores registrados para el evento dado.

Todos estos métodos devolverán la misma instancia de `EventEmitter` para permitir el encadenamiento. La función del escuchador tiene la firma `function([arg1], [...])`, por lo que simplemente acepta los argumentos proporcionados en el momento en que se emite el evento.

Ya puedes apreciar que existe una gran diferencia entre un escuchador y un callback tradicional de Node.js. De hecho, el primer argumento no es un error, sino que puede ser cualquier dato pasado a `emit()` en el momento de su invocación.

#### Creación y uso del EventEmitter

Veamos ahora cómo podemos utilizar un `EventEmitter` en la práctica. La forma más sencilla es crear una nueva instancia y usarla de inmediato. El siguiente código nos muestra una función que utiliza un `EventEmitter` para notificar a sus suscriptores en tiempo real cuando se encuentra una coincidencia con una expresión regular (*regex*) particular en una lista de archivos:

```javascript
import { EventEmitter } from 'node:events'
import { readFile } from 'node:fs'

function findRegex(files, regex) {
  const emitter = new EventEmitter()
  for (const file of files) {
    readFile(file, 'utf8', (err, content) => {
      if (err) {
        return emitter.emit('error', err)
      }
      emitter.emit('fileread', file)
      const match = content.match(regex)
      if (match) {
        for (const elem of match) {
          emitter.emit('found', file, elem)
        }
      }
    })
  }
  return emitter
}
```

La función que acabamos de definir devuelve una instancia de `EventEmitter` que producirá tres eventos:
- `fileread`: cuando se está leyendo un archivo.
- `found`: cuando se ha encontrado una coincidencia.
- `error`: cuando ocurre un error durante la lectura del archivo.

Veamos ahora cómo se puede utilizar nuestra función `findRegex()`:

```javascript
findRegex(
  ['fileA.txt', 'fileB.json'],
  /hello [\w.]+/
)
  .on('fileread', file => console.log(`${file} was read`))
  .on('found', (file, match) => console.log(`Matched "${match}" in ${file}`))
  .on('error', err => console.error(`Error emitted ${err.message}`))
```

En el código que acabamos de definir, registramos un escuchador para cada uno de los tres tipos de eventos producidos por el `EventEmitter` creado por nuestra función `findRegex()`.

Mantuvimos este ejemplo intencionalmente simple para mostrar cómo usar la clase `EventEmitter`, pero hay margen de mejora, especialmente para su uso en producción. Un problema es que estamos cargando todos los archivos en la memoria, lo que podría generar problemas si los archivos son grandes, llenando potencialmente la memoria y bloqueando el programa. En el [Capítulo 6](https://subscription.packtpub.com/book/web-development/9781803238944/6), Programación con Streams, cubriremos técnicas para manejar estas situaciones de manera más confiable. Además, estamos utilizando un bucle `for...of` síncrono simple para leer cada archivo, desencadenando una operación `readFile()` para cada archivo a la vez. Este enfoque puede sobrecargar el sistema si hay muchos archivos que procesar, excediendo potencialmente los límites del sistema en descriptores de archivos abiertos. Una mejor solución sería procesar los archivos en lotes utilizando el patrón de ejecución paralela limitada, que discutiremos en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks.

#### Propagación de errores

Al igual que con los callbacks, el `EventEmitter` no puede simplemente lanzar una excepción cuando ocurre una condición de error. En su lugar, la convención es emitir un evento especial, llamado `error`, y pasar un objeto `Error` como argumento. Eso es exactamente lo que estamos haciendo en la función `findRegex()` que definimos anteriormente.

La clase `EventEmitter` trata el evento `error` de una manera especial. Lanzará automáticamente una excepción y saldrá de la aplicación si se emite dicho evento y no se encuentra ningún escuchador asociado. Por esta razón, se recomienda registrar siempre un escuchador para el evento `error`.

#### Hacer observable cualquier objeto

En el mundo de Node.js, el `EventEmitter` rara vez se usa de forma aislada, como viste en el ejemplo anterior. En su lugar, es más común verlo extendido por otras clases. En la práctica, esto permite que cualquier clase herede las capacidades de `EventEmitter`, convirtiéndose así en un objeto observable.

Para demostrar este patrón, intentemos implementar la funcionalidad de la función `findRegex()` en una clase, de la siguiente manera:

```javascript
import { EventEmitter } from 'node:events'
import { readFile } from 'node:fs'

class FindRegex extends EventEmitter {
  constructor(regex) {
    super()
    this.regex = regex
    this.files = []
  }

  addFile(file) {
    this.files.push(file)
    return this
  }

  find() {
    for (const file of this.files) {
      readFile(file, 'utf8', (err, content) => {
        if (err) {
          return this.emit('error', err)
        }
        this.emit('fileread', file)
        const match = content.match(this.regex)
        if (match) {
          for (const elem of match) {
            this.emit('found', file, elem)
          }
        }
      })
    }
    return this
  }
}
```

Este código define una clase `FindRegex` que extiende la clase `EventEmitter` del módulo `node:events`. Esta clase tiene un constructor que usa `super()` para inicializar los componentes internos de `EventEmitter`. También toma un argumento `regex` y crea un array de archivos. La clase tiene dos métodos: `addFile()` y `find()`. El método `addFile()` agrega un archivo al array `files` y devuelve la instancia actual de la clase `FindRegex` para permitir llamadas encadenadas. El método `find()` lee cada archivo en el array `files`, usa la función `readFile()` para leer el contenido del archivo y luego usa la función `match()` para buscar coincidencias en el contenido usando la expresión regular pasada como argumento. Si se encuentra una coincidencia, emite el evento `found` con la ruta del archivo y el texto coincidente.

El siguiente es un ejemplo de cómo usar la clase `FindRegex` que acabamos de definir:

```javascript
const findRegexInstance = new FindRegex(/hello [\w.]+/)
findRegexInstance
  .addFile('fileA.txt')
  .addFile('fileB.json')
  .find()
  .on('found', (file, match) => console.log(`Matched "${match}" in file ${file}`))
  .on('error', err => console.error(`Error emitted ${err.message}`))
```

Ahora notarás cómo el objeto `FindRegex` también proporciona el método `on()`, heredado de `EventEmitter`. Este es un patrón común en el ecosistema de Node.js. Por ejemplo, el objeto `Server` del módulo central HTTP hereda de la función `EventEmitter`, lo que le permite producir eventos como `request` (cuando se recibe una nueva solicitud), `connection` (cuando se establece una nueva conexión) o `close` (cuando se cierra el socket del servidor).

Otros ejemplos notables de objetos que extienden `EventEmitter` son los streams de Node.js. Analizaremos los streams con más detalle en el [Capítulo 6](https://subscription.packtpub.com/book/web-development/9781803238944/6), Programación con Streams.

#### El riesgo de fugas de memoria (*Memory Leaks*)

Una fuga de memoria (*memory leak*) es un defecto de software por el cual la memoria que ya no se necesita no se libera, lo que hace que el uso de memoria de una aplicación crezca indefinidamente.

Al suscribirse a observables con una larga vida útil, es extremadamente importante que cancelemos la suscripción de nuestros escuchadores una vez que ya no sean necesarios. Esto nos permite liberar la memoria utilizada por los objetos en el ámbito de un escuchador y evitar fugas de memoria. Los escuchadores de `EventEmitter` no liberados son la principal fuente de fugas de memoria en Node.js (y en JavaScript en general).

Por ejemplo, considera el siguiente código:

```javascript
const thisTakesMemory = 'A big string....'
const listener = () => {
  console.log(thisTakesMemory)
}
emitter.on('an_event', listener)
```

La variable `thisTakesMemory` se referencia en el escuchador y, por lo tanto, su memoria se retiene hasta que el escuchador se libere de `emitter`, o hasta que el propio `emitter` sea recolectado por el recolector de basura (*garbage collector*), lo que solo puede suceder cuando no haya más referencias activas a él, volviéndose inalcanzable.

Puedes encontrar una buena explicación sobre la recolección de basura en JavaScript y el concepto de accesibilidad en [nodejsdp.link/garbage-collection](https://nodejsdp.link/garbage-collection).

Esto significa que si un `EventEmitter` permanece accesible durante toda la duración de la aplicación, todos sus escuchadores también lo harán, y con ellos toda la memoria a la que hacen referencia. Si, por ejemplo, registramos un escuchador en un `EventEmitter` "permanente" en cada solicitud HTTP entrante y nunca lo liberamos, entonces estamos provocando una fuga de memoria. La memoria utilizada por la aplicación crecerá indefinidamente, a veces lentamente, a veces más rápido, pero eventualmente bloqueará la aplicación. Esto es algo que, bajo circunstancias específicas, un atacante podría aprovechar para desencadenar un ataque de Denegación de Servicio (DoS). Para evitar una situación tan peligrosa, podemos liberar el escuchador con el método `removeListener()` de `EventEmitter`:

```javascript
emitter.removeListener('an_event', listener)
```

Un `EventEmitter` tiene un mecanismo integrado muy simple para advertir al desarrollador sobre posibles fugas de memoria. Cuando el recuento de escuchadores registrados para un evento supera una cantidad específica (por defecto, 10), `EventEmitter` emitirá una advertencia. A veces, registrar más de 10 escuchadores es completamente normal, por lo que podemos ajustar este límite utilizando el método `setMaxListeners()` de `EventEmitter`.

Cuando solo queremos escuchar un evento una vez, podemos usar el método de conveniencia `once(event, listener)` en lugar de `on(event, listener)` para desregistrar automáticamente un escuchador después de que el evento se reciba por primera vez. Sin embargo, ten en cuenta que si el evento que especificamos nunca se emite, el escuchador nunca se libera, lo que provoca una fuga de memoria.

#### Eventos síncronos y asíncronos

Al igual que con los callbacks, los eventos también se pueden emitir de forma síncrona o asíncrona con respecto al momento en que se desencadenan las tareas que los producen. Es crucial que nunca mezclemos los dos enfoques en el mismo `EventEmitter`, pero aún más importante, nunca debemos emitir el mismo tipo de evento utilizando una mezcla de código síncrono y asíncrono, para evitar producir los mismos problemas descritos en la sección "Liberando a Zalgo". La principal diferencia entre emitir eventos síncronos y asíncronos radica en la forma en que se pueden registrar los escuchadores.

Cuando los eventos se emiten de forma asíncrona, podemos registrar nuevos escuchadores incluso después de que se inicie la tarea que produce los eventos, hasta el momento en que la pila actual ceda el control al bucle de eventos. Esto se debe a que se garantiza que los eventos no se dispararán hasta el siguiente ciclo del bucle de eventos, por lo que podemos estar seguros de que no nos perderemos ningún evento.

La clase `FindRegex()` que definimos anteriormente emite sus eventos de forma asíncrona después de invocar el método `find()`. Esta es la razón por la que podemos registrar los escuchadores después de que se invoque el método `find()`, sin perder ningún evento, como se muestra en el siguiente código:

```javascript
findRegexInstance
  .addFile(...)
  .find()
  .on('found', ...)
```

Por otro lado, si emitimos nuestros eventos de forma síncrona después de iniciar la tarea, tenemos que registrar todos los escuchadores antes de iniciar la tarea, o nos perderemos todos los eventos. Para ver cómo funciona esto, modifiquemos la clase `FindRegex` que definimos anteriormente y hagamos que el método `find()` sea síncrono:

```javascript
find() {
  for (const file of this.files) {
    let content
    try {
      content = readFileSync(file, 'utf8')
    } catch (err) {
      this.emit('error', err)
    }
    this.emit('fileread', file)
    const match = content.match(this.regex)
    if (match) {
      for (const elem of match) {
        this.emit('found', file, elem)
      }
    }
  }
  return this
}
```

Ahora intentemos registrar un escuchador antes de iniciar la tarea `find()`, y luego un segundo escuchador después de eso para ver qué sucede:

```javascript
const findRegexSyncInstance = new FindRegexSync(/hello [\w.]+/)
findRegexSyncInstance
  .addFile('fileA.txt')
  .addFile('fileB.json')
  // this listener is invoked
  .on('found', (file, match) => console.log(`[Before] Matched "${match}"`))
  .find()
  // this listener is never invoked
  .on('found', (file, match) => console.log(`[After] Matched "${match}"`))
```

Como era de esperar, el escuchador que se registró después de la invocación de la tarea `find()` nunca se llama; de hecho, el código anterior imprimirá:

```text
[Before] Matched "hello world"
[Before] Matched "hello Node.js"
```

Hay algunas situaciones (raras) en las que emitir un evento de forma síncrona tiene sentido, pero la naturaleza misma de `EventEmitter` radica en su capacidad para lidiar con eventos asíncronos. La mayoría de las veces, emitir eventos de forma síncrona es una señal reveladora de que no necesitamos `EventEmitter` en absoluto o de que, en algún otro lugar, el mismo observable está emitiendo otro evento de forma asíncrona, causando potencialmente una situación de tipo Zalgo.

La emisión de eventos síncronos se puede diferir con `process.nextTick()` para garantizar que se emitan de forma asíncrona.

---

### Sección 3: EventEmitter frente a callbacks

Al diseñar una API asíncrona, una pregunta común es si utilizar un `EventEmitter` o un callback. Una pauta simple es usar callbacks para devolver resultados de forma asíncrona, y eventos cuando desees notificar que algo ha sucedido, dejando en manos del consumidor la decisión de si manejar ese evento o no.

La confusión proviene del hecho de que ambos enfoques son bastante similares en la práctica y a menudo pueden producir los mismos resultados. Por ejemplo, comparemos 2 ejemplos de código diferentes.

El primero ilustra la API de eventos:

```javascript
import { EventEmitter } from 'node:events'

function helloEvents () {
  const eventEmitter = new EventEmitter()
  setTimeout(() => eventEmitter.emit('complete', 'hello world'), 100)
  return eventEmitter
}

helloEvents().on('complete', message => console.log(message))
```

El segundo ilustra el uso de callbacks:

```javascript
function helloCallback (cb) {
  setTimeout(() => cb(null, 'hello world'), 100)
}

helloCallback((_err, message) => console.log(message))
```

Las dos funciones `helloEvents()` y `helloCallback()` pueden considerarse equivalentes en términos de funcionalidad. Ambas usan `setTimeout()` para simular un retraso antes de producir un resultado (la cadena `hello world` en este caso). Se diferencian en la forma en que señalan la finalización del tiempo de espera. `helloEvents()` utiliza un evento, mientras que `helloCallback()` se basa en un callback. Las diferencias clave entre estos dos enfoques radican en la legibilidad, la semántica y la cantidad de código necesaria para implementarlos o utilizarlos.

Aunque no existen reglas estrictas para elegir un enfoque sobre el otro, he aquí algunos consejos para guiar tu decisión:
- Los callbacks pueden resultar limitantes a la hora de admitir diferentes tipos de eventos. Puedes pasar el tipo de evento como argumento al callback o utilizar múltiples callbacks para cada evento, pero este enfoque puede resultar en una API menos elegante. En tales casos, un `EventEmitter` proporciona una interfaz más limpia y un código más conciso.
- Utiliza `EventEmitter` cuando un evento pueda ocurrir varias veces o ninguna. Normalmente se espera que los callbacks se ejecuten una vez, independientemente del éxito o del fracaso. Si un evento puede repetirse, es mejor tratarlo como algo que se comunica en lugar de un resultado que se devuelve.
- Un `EventEmitter` permite registrar múltiples escuchadores para el mismo evento. Esta flexibilidad puede resultar útil en ciertos escenarios.

---

### Sección 4: Combinación de callbacks y eventos

En algunos casos, puedes usar un `EventEmitter` junto con un callback. Este patrón es muy potente y útil cuando necesitas gestionar el resultado final de una operación asíncrona con un callback, pero también deseas emitir eventos de progreso a medida que la operación está en curso. Este enfoque te brinda lo mejor de ambos mundos, permitiendo un seguimiento granular de eventos sin dejar de proporcionar una forma de propagar un resultado final.

Un ejemplo tradicional es descargar un archivo desde una URL. Podemos crear una función que use un callback para indicar cuándo se completó la descarga. Al mismo tiempo, la función puede devolver un `EventEmitter` para rastrear el progreso de la descarga. Esto nos permite mostrar actualizaciones en tiempo real, como el porcentaje de finalización, en la pantalla.

Antes de sumergirnos en el código, es importante tener en cuenta que crearemos un cliente HTTPS que sigue un flujo de solicitud/respuesta. He aquí un desglose de los pasos principales:
1. **Petición del cliente:** El cliente envía una petición al servidor, especificando un método (en nuestro caso, `GET`) y una ruta (la parte de la URL después del protocolo y el nombre de dominio). Esta es una operación asíncrona ya que se transmiten múltiples bytes a través de la red gradualmente.
2. **Respuesta del servidor:** El servidor procesa la petición y devuelve una respuesta. Esta respuesta normalmente tiene dos partes: una cabecera de respuesta (que incluye el código de estado y las cabeceras) y un cuerpo de respuesta, que, en este caso, será el contenido del archivo que se está descargando. Esta también es una operación asíncrona.
3. **Cierre de la conexión:** Una vez recibida la respuesta por completo, la conexión se cierra en ambos extremos, completando el flujo. También debemos tener en cuenta posibles errores, como no poder establecer una conexión o que el servidor interrumpa la respuesta a mitad de la transferencia.

Veamos ahora cómo podemos implementar este concepto:

```javascript
import { EventEmitter } from 'node:events'
import { get } from 'node:https'

function download(url, cb) { // 1
  const eventEmitter = new EventEmitter() // 2
  const req = get(url, resp => { // 3
    const chunks = [] // 4
    let downloadedBytes = 0
    const fileSize = Number.parseInt(resp.headers['content-length'], 10)
    resp
      .on('error', err => { // 5
        cb(err)
      })
      .on('data', chunk => { // 6
        chunks.push(chunk)
        downloadedBytes += chunk.length
        eventEmitter.emit('progress', downloadedBytes, fileSize)
      })
      .on('end', () => { // 7
        const data = Buffer.concat(chunks)
        cb(null, data)
      })
  })
  req.on('error', err => { // 8
    cb(err)
  })
  return eventEmitter // 9
}
```

Hay mucho contenido aquí, así que repasémoslo paso a paso:
1. Nuestra función `download()` toma una URL (`url`) desde la que descargar y un callback (`cb`) para gestionar el resultado una vez que se complete la descarga o si ocurre un error.
2. Crea una nueva instancia de `EventEmitter` para emitir eventos durante el proceso de descarga.
3. Utilizamos el módulo central de Node `node:https` para realizar una petición HTTPS, específicamente usamos la función `get` (`const req = get(url, resp => { ... })`). Esta función envía una solicitud HTTPS a la URL proporcionada y comienza a procesar la respuesta una vez que el servidor responde. Devuelve un objeto de solicitud (`req`), un `EventEmitter` que podemos utilizar para realizar un seguimiento del progreso del reenvío de la solicitud al servidor. El callback recibe un objeto de respuesta (`resp`), que es otro `EventEmitter` que puede notificarnos de varios eventos relacionados con la respuesta que estamos recibiendo del servidor. Esto incluye eventos como errores (`'error'`), nuevos datos recibidos (`'data'`) y finalización (`'end'`).
4. El objeto de respuesta se crea una vez que el servidor envía con éxito las cabeceras de respuesta, lo que nos permite acceder a esas cabeceras. En este punto, inicializamos el array `chunks` para almacenar los bytes que se recibirán a lo largo del tiempo (en fragmentos), junto con el contador `downloadedBytes` para rastrear el número total de bytes descargados. Al leer la cabecera `'Content-Length'`, podemos determinar el número total de bytes esperados en la respuesta.
5. Añadimos un escuchador para el evento `'error'` en el objeto de respuesta para detectar cualquier problema al recibir el cuerpo de la respuesta. Si ocurre un error, se lo pasamos al invocador a través del callback.
6. El seguimiento del progreso se realiza mediante el escuchador de eventos `'data'`, que se activa cada vez que se recibe un fragmento de datos. Cada fragmento se agrega al array `chunks` y `downloadedBytes` se actualiza con el tamaño del fragmento recibido. El `eventEmitter` emite un evento `'progress'` con el número actual de bytes descargados y el tamaño total del archivo, que se puede utilizar para calcular y mostrar el progreso de la descarga.
7. Cuando el cuerpo de la respuesta se ha enviado por completo, se dispara el evento `'end'`. En nuestro manejador de eventos, todos los fragmentos se concatenan en un único Buffer mediante `Buffer.concat(chunks)`, que representa los datos completos del archivo. También se llama al callback `cb(null, data)` para señalar la finalización y devolver los datos descargados al invocador.
8. Si ocurre un error durante la fase de solicitud, se activa el escuchador `req.on('error', err => { ... })` y se llama al callback con el error. Esto nos permite capturar y propagar los errores que puedan ocurrir durante la fase de solicitud.
9. La función devuelve el `eventEmitter`, lo que permite al invocador escuchar los eventos `'progress'` para rastrear la descarga.

Sí, fueron bastantes detalles, pero veamos ahora cómo usar la nueva función `download()`:

```javascript
download(
  'https://example.com/somefile.zip',
  (err, data) => {
    if (err) {
      return console.error(`Download failed: ${err.message}`)
    }
    console.log('Download completed', data)
  }
).on('progress', (downloaded, total) => {
  console.log(
    `${downloaded}/${total} ` +
    `(${((downloaded / total) * 100).toFixed(2)}%)`
  )
})
```

He aquí cómo funciona:
- **Llamada a `download()`:** Proporcionamos la URL del archivo para descargar y una función callback. Este callback maneja el resultado final de la descarga: ya sea recibiendo el buffer con los datos descargados o un error.
- **Escuchador de eventos Progress:** También podemos adjuntar opcionalmente un escuchador de eventos `'progress'`. Este escuchador imprime la cantidad de datos descargados hasta el momento y el porcentaje de finalización, proporcionando una agradable respuesta visual del progreso para el usuario en caso de que la descarga tarde algún tiempo.

Una salida de ejemplo que muestre actualizaciones de progreso en tiempo real y el resultado final podría verse así:

```text
280/127454 (0.22%)
1649/127454 (1.29%)
…
126747/127454 (99.45%)
127454/127454 (100.00%)
Download completed <Buffer ff d8 ff e0 00 10 4a 46 49 46 00 01 01 00 00 01 00 01 00 00 ff db 00 84 00 04 04 04 04 04 04 04 04 04 04 06 06 05 06 06 08 07 07 07 07 08 0c 09 09 09 ... 127404 more bytes>
```

Ten en cuenta que esta implementación tiene fines ilustrativos y no está lista para producción. Está simplificada para centrarse en demostrar el patrón de combinar un callback con un `EventEmitter`, pero carece de varias características importantes. No maneja transferencias fragmentadas (*chunked*), redirecciones ni peticiones HTTP simples, y almacena todos los datos en un solo buffer, lo que puede provocar problemas de memoria con archivos grandes. El manejo de errores para códigos de respuesta HTTP también está ausente y no hay soporte para la reanudación de descargas ni reintentos. Para escenarios de producción, es posible que desees utilizar una abstracción o biblioteca más completa como `fetch` u opciones de terceros como `axios` ([nodejsdp.link/axios](https://nodejsdp.link/axios)), que ofrecen funciones avanzadas y un mejor manejo de errores, lo que las hace más adecuadas para aplicaciones del mundo real.

Este ejemplo destaca la eficacia de combinar un callback con un `EventEmitter`. La conclusión clave es que este patrón es especialmente útil para gestionar tareas de larga duración en las que necesitas realizar un seguimiento tanto de la finalización como del progreso en tiempo real. También es aplicable a varios otros escenarios prácticos, incluidas las cargas de archivos, las canalizaciones de procesamiento de datos, las copias de seguridad y migraciones de bases de datos, el escaneo de puertos, las aplicaciones de fuerza bruta y más. Curiosamente, la función `get()` del módulo `node:https` también aprovecha este patrón: proporciona un callback para acceder a la respuesta al mismo tiempo que devuelve un `EventEmitter` para monitorizar la solicitud en curso.

Aunque la combinación de callbacks y emisores de eventos es común en el ecosistema de Node.js, este patrón está siendo reemplazado gradualmente por la combinación de promesas e iteradores asíncronos. Nos sumergiremos en estos enfoques modernos en el [Capítulo 9](https://subscription.packtpub.com/book/web-development/9781803238944/9), Patrones de diseño de comportamiento. Además, puedes combinar emisores de eventos con otros mecanismos asíncronos como promesas. Por ejemplo, puedes devolver un objeto que contenga tanto una promesa como un `EventEmitter`, que luego el invocador puede desestructurar, de esta manera: `{ promise, events } = foo()`.

---

### Sección 5: Resumen

En este capítulo hemos tomado nuestro primer contacto con los aspectos prácticos de la escritura de código asíncrono. Descubrimos los dos pilares de toda la infraestructura asíncrona de Node.js —el callback y el `EventEmitter`— y exploramos en detalle sus casos de uso, convenciones y patrones. También analizamos algunas de las dificultades que plantea el código asíncrono y aprendimos las formas de evitarlas. Dominar el contenido de este capítulo allana el camino hacia el aprendizaje de las técnicas asíncronas más avanzadas que se presentarán a lo largo del resto de este libro.

En el próximo capítulo, aprenderemos a gestionar flujos de control asíncronos complejos utilizando callbacks.

---

### Sección 6: Ejercicios

- **3.1 Un evento simple:** Modifica la clase asíncrona `FindRegex` para que emita un evento cuando comience el proceso de búsqueda, pasando la lista de archivos de entrada como argumento. *Pista: ¡cuidado con Zalgo!*
- **3.2 Ticker:** Escribe una función que acepte un número y un callback como argumentos. La función devolverá un `EventEmitter` que emite un evento llamado `tick` cada 50 milisegundos hasta que haya transcurrido el número de milisegundos desde la invocación de la función. La función también llamará al callback cuando haya transcurrido el número de milisegundos, proporcionando, como resultado, el recuento total de eventos `tick` emitidos. *Pista: puedes usar `setTimeout()` para programar otro `setTimeout()` de forma recursiva.*
- **3.3 Una modificación simple:** Modifica la función creada en el ejercicio 3.2 para que emita un evento `tick` inmediatamente después de invocar la función.
- **3.4 Jugando con errores:** Modifica la función creada en el ejercicio 3.3 para que produzca un error si la marca de tiempo (*timestamp*) en el momento de un `tick` (incluido el inicial que agregamos como parte del ejercicio 3.3) es divisible por 5. Propaga el error utilizando tanto el callback como el emisor de eventos. *Pista: usa `Date.now()` para obtener la marca de tiempo y el operador de resto (`%`) para verificar si la marca de tiempo es divisible por 5.*
- **3.5 Buscador de archivos pesados en disco (*Disk bloat finder*):** Crea una función que acepte la ruta a una carpeta en el sistema de archivos local e identifique el archivo más grande dentro de esa carpeta. Como crédito adicional, mejora la función para buscar recursivamente en subcarpetas. *Pista: puedes usar el módulo `node:fs` para esta tarea, específicamente la función `stats()` para determinar si una ruta es un directorio o un archivo y para obtener el tamaño del archivo en bytes, y la función `readdir()` para listar el contenido de un directorio.*
