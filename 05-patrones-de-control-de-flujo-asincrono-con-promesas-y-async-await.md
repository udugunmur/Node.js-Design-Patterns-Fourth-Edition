# Parte 1: Fundamentos de Node.js

## Capítulo 5: Patrones de control de flujo asíncrono con Promesas y Async/Await

Los callbacks son los bloques de construcción de bajo nivel de la programación asíncrona en Node.js y, como tales, son uno de los conceptos más importantes que debemos dominar, pero distan mucho de ser cómodos para el desarrollador. No es de extrañar que, en el capítulo anterior, hayamos dedicado una cantidad significativa de tiempo a aprender varias técnicas para aplicar la disciplina de callbacks y mantener nuestro código manejable. También aprendimos sobre diferentes construcciones de flujo de control utilizando callbacks, y debemos reconocer que pueden ser complejas y detalladas hasta el punto de que necesitamos crear abstracciones de nivel superior para mantener esta complejidad bajo control y simplificar la reutilización de estos patrones. Merece una mención especial el flujo de ejecución serial, que es la estructura de control de flujo predominante en la mayor parte del código que escribimos. Intentar implementar este flujo de control puede llevar fácilmente a un desarrollador no entrenado a escribir código afectado por el problema del callback hell. Además de eso, incluso si se implementa correctamente, un flujo de ejecución serial parece innecesariamente complicado y propenso a errores. Recordemos también lo frágil que es la gestión de errores con callbacks: si olvidamos reenviar un error, simplemente se pierde, y si olvidamos capturar cualquier excepción lanzada por código síncrono, el programa se bloquea y termina abruptamente. Y todo esto sin considerar que Zalgo siempre está respirándonos en la nuca.

Node.js y JavaScript han sido criticados durante años por la falta de una solución nativa para un problema tan común y ubicuo. Afortunadamente, a lo largo de los años, la comunidad ha trabajado en nuevas soluciones al problema y, finalmente, tras muchas iteraciones, discusiones y años de espera, hoy contamos con una solución adecuada para el "problema de los callbacks".

El primer paso hacia una mejor experiencia con el código asíncrono es la **promesa** (*promise*), un objeto que "transporta" el estado y el resultado eventual de una operación asíncrona. Una promesa se puede encadenar fácilmente para implementar flujos de ejecución seriales y se puede mover como cualquier otro objeto. Las promesas simplifican enormemente el código asíncrono; sin embargo, aún había margen de mejora. Así que, en un intento de hacer que el omnipresente flujo de ejecución serial fuera lo más simple posible, se introdujo una nueva construcción llamada **async/await**, que finalmente puede hacer que el código asíncrono parezca tan simple y fácil de razonar como el código síncrono.

En la programación moderna con Node.js, async/await es la construcción preferida al tratar con código asíncrono. Sin embargo, async/await se construye sobre promesas, del mismo modo que las promesas se construyen sobre callbacks. Por lo tanto, es importante que conozcamos y dominemos todos ellos en orden. Para comprender verdaderamente cómo funciona async/await y usarlo correctamente, es esencial tener primero un conocimiento sólido de los callbacks y las promesas. Saltarse estos conceptos fundamentales puede llevar a un grave malentendido de cómo opera async/await y dar como resultado la escritura de código defectuoso o ineficiente.

Si repasaste con atención el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks, ya no deberías temer a los callbacks. Ahora, en este capítulo, cubriremos los siguientes temas:
- Cómo funcionan las promesas y cómo utilizarlas eficazmente para implementar las principales construcciones de flujo de control que ya conocemos.
- La sintaxis async/await, que se convertirá en nuestra herramienta principal para trabajar con código asíncrono en Node.js.
- Patrones prácticos con async/await, errores comunes y cómo evitarlos.

Al final del capítulo, habrás aprendido sobre los dos componentes más importantes que tenemos en JavaScript para dominar el código asíncrono. Comencemos, pues, descubriendo las promesas.

---

### Sección 1: Promesas

Las promesas forman parte del estándar ECMAScript 2015 (o ES6, por lo que también se denominan promesas ES6) y están disponibles de forma nativa en Node.js desde la versión 4. Pero la historia de las promesas se remonta a unos años antes, cuando existían decenas de implementaciones, inicialmente con diferentes características y comportamientos. Con el tiempo, la mayoría de esas implementaciones se establecieron en un estándar llamado **Promises/A+**.

Las promesas constituyen un gran paso hacia proporcionar una alternativa sólida a los callbacks de estilo de paso de continuación (CPS) para propagar un resultado asíncrono. Como veremos, el uso de promesas hará que las construcciones de flujo de control asíncrono sean más fáciles de leer, menos detalladas y más robustas en comparación con sus alternativas basadas en callbacks.

#### ¿Qué es una promesa?

Una promesa es un objeto que representa el resultado eventual (o error) de una operación asíncrona. En la jerga de las promesas, decimos que una Promise está **pendiente** (*pending*) cuando la operación asíncrona aún no se ha completado, está **cumplida** (*fulfilled*) cuando la operación se completa con éxito y está **rechazada** (*rejected*) cuando la operación termina con un error. Una vez que una Promise se cumple o se rechaza, se considera **resuelta / liquidada** (*settled*).

Para recibir el valor de cumplimiento o el error (motivo) asociado con el rechazo, podemos usar el método `then()` de una instancia de Promise. La siguiente es su firma:

```javascript
promise.then(onFulfilled, onRejected)
```

En la firma anterior, `onFulfilled` es un callback que eventualmente recibirá el valor de cumplimiento de la Promise, y `onRejected` es otro callback que recibirá el motivo del rechazo (si lo hay). Ambos son opcionales.

Para tener una idea de cómo las promesas pueden transformar nuestro código, consideremos el siguiente código basado en callbacks:

```javascript
asyncOperation(arg, (err, result) => {
  if(err) {
    // handle the error
  }
  // do stuff with the result
})
```

Las promesas nos permiten transformar este típico código de estilo de paso de continuación en un código mejor estructurado y más elegante, como el siguiente:

```javascript
asyncOperationPromise(arg)
  .then(result => {
    // do stuff with result
  }, err => {
    // handle the error
  })
```

En el código anterior, `asyncOperationPromise()` devuelve una Promise, que luego podemos usar para recibir el valor de cumplimiento o el motivo del rechazo del resultado eventual de la función. Hasta ahora, parece que esto es solo una pequeña diferencia sintáctica en comparación con el código basado en callbacks, pero una propiedad crucial del método `then()` es que **devuelve sincrónicamente otra Promise**.

Además, si cualquiera de las funciones `onFulfilled` u `onRejected` devuelve un valor `x`, la Promise devuelta por el método `then()` hará lo siguiente:
- Se cumplirá con `x` si `x` es un valor.
- Se cumplirá con el valor de cumplimiento de `x` si `x` es una Promise en estado cumplido (*fulfilled*).
- Se rechazará con el motivo de rechazo eventual de `x` si `x` es una Promise en estado rechazado (*rejected*).

Este comportamiento —el retorno síncrono de otra Promise y el manejo de los valores devueltos— nos permite construir cadenas de promesas (*promise chains*), lo que permite una fácil agregación y disposición de operaciones asíncronas en varias configuraciones. Además, si no especificamos un manejador `onFulfilled` u `onRejected`, el valor de cumplimiento o el motivo del rechazo se reenvía automáticamente a la siguiente promesa de la cadena. Esto nos permite, por ejemplo, propagar errores automáticamente a lo largo de toda la cadena hasta que sean capturados por un manejador `onRejected`. Con una cadena de promesas, la ejecución secuencial de tareas se convierte de repente en una operación sencilla:

```javascript
asyncOperationPromise(arg)
  .then(result1 => {
    // returns another promise
    return asyncOperationPromise(arg2)
  })
  .then(result2 => {
    // returns a value
    return 'done'
  })
  .then(undefined, err => {
    // any error in the chain is caught here
  })
```

El siguiente diagrama proporciona otra perspectiva sobre cómo funciona una cadena de promesas:

**Figura 5.1:** Flujo de ejecución de una cadena de promesas.

La Figura 5.1 muestra cómo fluye nuestro programa cuando usamos una cadena de promesas. Cuando invocamos `then()` en la Promesa A, recibimos sincrónicamente la Promesa B como resultado, y cuando invocamos `then()` en la Promesa B, recibimos sincrónicamente la Promesa C como resultado. Con el tiempo, cuando la Promesa A se resuelva, se cumplirá o se rechazará, lo que resulta en la invocación del callback `onFulfilled()` o `onRejected()`, respectivamente. El resultado de la ejecución de dicho callback cumplirá o rechazará a la Promesa B y dicho resultado se propaga, a su vez, al callback `onFulfilled()` o `onRejected()` pasado a la invocación de `then()` en la Promesa B. La ejecución continúa de manera similar para la Promesa C y cualquier otra promesa que siga en la cadena.

Desde la perspectiva del usuario, esto significa que una vez que se evalúa la expresión completa (por ejemplo, `PromiseA.then(...).then(...)`), el resultado es la última promesa de la cadena; en este caso, la Promesa C. Este valor se devuelve sincrónicamente en el momento en que se evalúa la expresión, aunque las operaciones asíncronas subyacentes aún no se hayan completado. Supongamos que asignamos este valor a una variable llamada `promiseC`; ahora podemos adjuntar algunos manejadores de finalización utilizando `promiseC.then(onFulfilled, onRejected)` para recibir una notificación asíncrona cuando `promiseC` finalmente se resuelva, lo que nos permite manejar el resultado final de toda la cadena de promesas.

Una propiedad importante de las promesas es que se garantiza que los callbacks `onFulfilled()` y `onRejected()` se invocarán de forma asíncrona y como máximo una vez, incluso si la promesa ya está resuelta cuando se llama a `.then()`. He aquí un ejemplo:

```javascript
Promise.resolve(someValue).then(onFulfilled, onRejected)
```

En este caso, `Promise.resolve(someValue)` devuelve una promesa que ya está cumplida. Sin embargo, `onFulfilled` no se llamará inmediatamente. En su lugar, se programará para ejecutarse de forma asíncrona, después de que se vacíe la pila de llamadas actual.

Esta programación se maneja utilizando la cola de microtareas (*microtask queue*), un mecanismo que garantiza que los callbacks de las promesas se ejecuten lo antes posible, pero solo después de que el código síncrono actual haya terminado de ejecutarse. Este comportamiento ayuda a prevenir el entrelazado inesperado de lógica síncrona y asíncrona, un problema común conocido como Zalgo (ver [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos). Gracias a esto, las promesas proporcionan un modelo consistente y predecible para el flujo de control asíncrono, facilitando el razonamiento sobre los tiempos de ejecución del código.

Ahora viene la mejor parte: si se lanza una excepción (usando la sentencia `throw`) en el manejador `onFulfilled()` o `onRejected()`, la Promise devuelta por el método `then()` se rechazará automáticamente con la excepción que se lanzó. Esta es una tremenda ventaja sobre CPS, ya que significa que con las promesas las excepciones se propagarán automáticamente a través de la cadena, y la sentencia `throw` finalmente se vuelve utilizable en código asíncrono.

#### Promises/A+ y thenables

Históricamente, han existido muchas implementaciones diferentes de promesas, y la mayoría de ellas no eran compatibles entre sí, lo que significaba que no era posible crear cadenas entre objetos Promise provenientes de bibliotecas que utilizaban diferentes implementaciones.

La comunidad de JavaScript trabajó muy duro para abordar esta limitación, y esos esfuerzos llevaron a la creación de la especificación **Promises/A+**. Esta especificación detalla el comportamiento del método `then()`, proporcionando una base interoperable que hace que los objetos Promise de diferentes bibliotecas puedan funcionar entre sí sin configuración adicional. Hoy en día, la gran mayoría de las implementaciones de Promise utilizan este estándar, incluido el objeto Promise nativo de JavaScript y Node.js.

Para obtener una descripción detallada de la especificación Promises/A+, puedes consultar el sitio web oficial en [nodejsdp.link/promises-aplus](https://nodejsdp.link/promises-aplus).

Como resultado de la adopción del estándar Promises/A+, muchas implementaciones de Promise, incluida la API nativa de JavaScript, considerarán cualquier objeto con un método `then()` como un objeto similar a una Promise, también llamado **thenable**. Este comportamiento permite que diferentes implementaciones de Promise interactúen entre sí sin problemas.

La técnica de reconocer (o tipar) objetos en función de su comportamiento externo, en lugar de su tipo real, se denomina **tipado de pato** (*duck typing*, [nodejsdp.link/duck](https://nodejsdp.link/duck)) y se utiliza ampliamente en JavaScript.

#### La API de promesas

Echemos ahora un vistazo rápido a la API nativa de JavaScript Promise. Esta es solo una descripción teórica para darte una idea de lo que podemos hacer con las promesas. Haz un esfuerzo por comprender la semántica, pero no te preocupes si las cosas aún no están tan claras en este punto; tendremos la oportunidad de usar la mayoría de estas APIs a lo largo del libro, y la práctica suele ser el mejor maestro.

El constructor de Promise (`new Promise((resolve, reject) => {})`) crea una nueva instancia de Promise que se cumple o se rechaza según el comportamiento de la función proporcionada como argumento. La función proporcionada al constructor recibirá dos argumentos:
- `resolve(obj)`: Esta es una función que, al ser invocada, cumplirá la Promise con el valor de cumplimiento proporcionado, que será `obj` si `obj` es un valor. Será el valor de cumplimiento de `obj` si `obj` es una Promise o un thenable.
- `reject(err)`: Esto rechaza la Promise con el motivo `err`. Es una convención que `err` sea una instancia de `Error`.

Veamos ahora los métodos estáticos más importantes del objeto Promise:
- `Promise.resolve(obj)`: Este método crea una nueva Promise a partir de otra Promise, un thenable o un valor. Si se pasa una Promise, esa Promise se devuelve tal como está. Si se proporciona un thenable, se convierte a la implementación de Promise en uso. Si se proporciona un valor, la Promise se cumplirá con ese valor.
- `Promise.reject(err)`: Este método crea una Promise que se rechaza con `err` como motivo.
- `Promise.all(iterable)`: Este método crea una Promise que se cumple con un array de valores de cumplimiento cuando cada elemento en el objeto iterable de entrada (como un `Array`) se cumple. Si alguna Promise en el objeto iterable se rechaza, la Promise devuelta por `Promise.all()` se rechazará con el primer motivo de rechazo. Ten en cuenta que si alguna promesa en el iterable se rechaza, la promesa devuelta por `Promise.all()` se rechazará de inmediato, pero cualquier otra promesa pendiente en el iterable continuará ejecutándose y resolviéndose de forma independiente; no se cancelan ni abortan automáticamente.
- `Promise.allSettled(iterable)`: Este método espera a que todas las promesas de entrada se cumplan o se rechacen y luego devuelve un array de objetos que contienen el valor de cumplimiento o el motivo del rechazo para cada Promise de entrada. Cada objeto de salida tiene una propiedad `status`, que puede ser igual a `'fulfilled'` o `'rejected'`, y una propiedad `value` que contiene el valor de cumplimiento, o una propiedad `reason` que contiene el motivo del rechazo. La diferencia con `Promise.all()` es que `Promise.allSettled()` siempre esperará a que cada Promise se cumpla o se rechace, en lugar de rechazarse inmediatamente cuando una de las promesas falla.
- `Promise.race(iterable)`: Este método devuelve una Promise que es equivalente a la primera Promise en `iterable` que se resuelva (incluso si la primera Promise es rechazada; en ese caso, toda la operación da como resultado un rechazo). Si bien `Promise.race()` se usa con menos frecuencia, tiene casos de uso prácticos. Por ejemplo, si tienes varios servidores disponibles, puedes hacer ping a todos a la vez y usar `Promise.race()` para elegir el primero que responda. El ganador es el servidor con la latencia observada más baja en ese momento, por lo que puedes preferirlo para solicitudes posteriores.
- `Promise.withResolvers()`: Devuelve un objeto que contiene un nuevo objeto Promise y dos funciones para resolverlo o rechazarlo, correspondientes a los dos parámetros pasados al ejecutor del constructor `Promise()`. Es aproximadamente equivalente a la siguiente función:
  ```javascript
  function withResolvers() {
    let resolve
    let reject
    const promise = new Promise((res, rej) => {
      resolve = res
      reject = rej
    })
    return { promise, resolve, reject }
  }
  ```
  Cuando llamas a esta función, te da acceso a las funciones `resolve()` y `reject()` de la promesa, lo que te permite controlar el resultado de la promesa externamente. Esto es especialmente útil cuando se integra con código basado en callbacks o guiado por eventos, donde necesitas resolver o rechazar manualmente una promesa. Veremos un caso de uso práctico para esto en el [Capítulo 10](https://subscription.packtpub.com/book/web-development/9781803238944/10), Pruebas: Patrones y mejores prácticas.

Los siguientes son los métodos principales disponibles en una instancia de Promise:
- `promise.then(onFulfilled, onRejected)`: Este es el método esencial de una Promise para manejar la continuación. Su comportamiento es compatible con el estándar Promises/A+ que mencionamos anteriormente.
- `promise.catch(onRejected)`: Este método es simplemente azúcar sintáctico (*syntactic sugar*, [nodejsdp.link/syntactic-sugar](https://nodejsdp.link/syntactic-sugar)) para `promise.then(undefined, onRejected)`.
- `promise.finally(onFinally)`: Este método nos permite configurar un callback `onFinally`, que se invoca cuando la Promise se resuelve (ya sea cumplida o rechazada). A diferencia de `onFulfilled` y `onRejected`, el callback `onFinally` no recibirá ningún argumento como entrada y cualquier valor devuelto por él será ignorado. La Promise devuelta por `finally` se resolverá con el mismo valor de cumplimiento o motivo de rechazo de la instancia de Promise actual. Solo hay una excepción a todo esto, que es el caso en el que lanzamos una excepción con `throw` dentro del callback `onFinally` o devolvemos una Promise rechazada. En este caso, la Promise devuelta se rechazará con el error lanzado o el motivo de rechazo de la Promise rechazada devuelta.

Veamos ahora un ejemplo de cómo podemos crear una Promise desde cero utilizando su constructor.

#### Creación de una promesa

Crear una Promise desde cero es una operación de bajo nivel y, por lo general, se requiere cuando necesitamos convertir una API que utiliza otro estilo asíncrono (como un estilo basado en callbacks). La mayor parte del tiempo trabajarás con promesas producidas por otras bibliotecas, y la mayoría de las promesas que creemos provendrán del método `then()`. No obstante, en algunos escenarios avanzados, necesitamos crear manualmente una Promise utilizando su constructor.

Para demostrar cómo usar el constructor de Promise, creemos una función que devuelva una Promise que se cumpla con la marca de tiempo Unix actual después de un número específico de milisegundos. Una marca de tiempo Unix se define como la cantidad de segundos transcurridos desde el 1 de enero de 1970 a las 00:00:00 UTC (la época Unix). En JavaScript, podemos obtener la marca de tiempo Unix actual con `Date.now()`, pero ten en cuenta que devolverá el valor en milisegundos en lugar de en segundos.

Veamos una posible implementación de esta función:

```javascript
function delay(milliseconds) {
  return new Promise((resolve, _reject) => {
    setTimeout(() => {
      resolve(Date.now())
    }, milliseconds)
  })
}
```

Como habrás adivinado, usamos `setTimeout()` para invocar la función `resolve()` del constructor de Promise.

Ten en cuenta que la función ejecutora (la función pasada al constructor de Promise) se ejecuta sincrónicamente tan pronto como se crea la promesa. Volveremos a esto más adelante en el capítulo cuando analicemos cómo crear promesas diferidas (*lazy promises*).

Podemos notar cómo todo el cuerpo de la función está envuelto por el constructor de Promise; este es un patrón de código frecuente que verás al crear una Promise desde cero.

Ahora podemos usar la función `delay()`. Veamos un ejemplo:

```javascript
console.log(`Delaying... (${Date.now()})`)
delay(1000).then(newDate => {
  console.log(`Done (${newDate})`)
})
```

El `console.log()` dentro del manejador `then()` se ejecutará aproximadamente 1 segundo después de la invocación de `delay()`. Puedes ver que las dos marcas de tiempo Unix diferirán en aproximadamente (¡pero no exactamente!) 1000 milisegundos.

Este sencillo ejemplo ilustra una propiedad a menudo sorprendente de los temporizadores de Node.js: la programación de tareas con `setTimeout()` se considera de mejor esfuerzo (*best effort*), no de temporización precisa garantizada. De hecho, es posible que el callback no se ejecute exactamente después de que haya expirado el tiempo de espera especificado. Esta discrepancia surge porque puede haber retrasos entre el momento en que se completa un temporizador y el momento en que el bucle de eventos recoge y ejecuta su callback de finalización.

Ten en cuenta también que aquí creamos nuestra función personalizada basada en promesas para poder controlar dinámicamente la creación del valor con el que se resuelve la promesa. Si solo deseas esperar una cierta cantidad de tiempo, simplemente puedes usar la función integrada `setTimeout()` de Node.js del módulo `'node:timers/promises'`, de la siguiente manera:

```javascript
import { setTimeout } from 'node:timers/promises'
await setTimeout(100) // or setTimeout(100).then(/* … */)
```

La especificación Promises/A+ establece que los callbacks `onFulfilled` y `onRejected` del método `then()` deben invocarse solo una vez y de manera exclusiva (solo se invoca uno u otro). Una implementación de promesas compatible garantiza que incluso si llamamos a `resolve` o `reject` varias veces, la Promise se cumpla o se rechace solo una vez.

#### Promisificación (*Promisification*)

Cuando se conocen de antemano algunas características de una función basada en callbacks, es posible crear una función que transforme dicha función basada en callbacks en una función equivalente que devuelva una Promise. Esta transformación se llama **promisificación** (*promisification*).

Por ejemplo, consideremos las convenciones utilizadas en las funciones basadas en callbacks al estilo Node.js:
- El callback es el último argumento de la función.
- El error (si lo hay) es siempre el primer argumento pasado al callback.
- Cualquier valor de retorno se pasa después del error al callback.

En base a estas reglas, podemos crear fácilmente una función genérica que promisifique una función basada en callbacks al estilo de Node.js. Veamos cómo es esta función:

```javascript
function promisify(callbackBasedFn) {
  return function promisifiedFn(...args) {
    return new Promise((resolve, reject) => { // 1
      const newArgs = [ // 2
        ...args, // 3
        (err, result) => { // 4
          if (err) {
            return reject(err)
          }
          resolve(result)
        },
      ]
      callbackBasedFn(...newArgs) // 5
    })
  }
}
```

La función anterior devuelve otra función llamada `promisifiedFn()`, que representa la versión promisificada de `callbackBasedFn` dada como entrada. Así es como funciona:
1. La función `promisifiedFn()` crea una nueva Promise usando el constructor de Promise y la devuelve inmediatamente al invocador.
2. En la función pasada al constructor de Promise, construimos un nuevo conjunto de argumentos para pasarlos a la función original basada en callbacks (`callbackBasedFn`).
3. Esta lista de argumentos incluye todos los argumentos recibidos por el invocador de `promisifiedFn` (`...args`).
4. La lista de argumentos también incluye un callback que se utiliza para capturar el resultado de `callbackBasedFn` y resolver la promesa en consecuencia. Si esta función callback ha recibido un error (`err`), entonces llamamos a `reject()`; de lo contrario, llamamos a `resolve()` con el resultado recibido.
5. Finalmente, simplemente invocamos `callbackBasedFn` con la lista de argumentos que hemos creado para desencadenar la ejecución de la lógica de negocio original que ahora hemos envuelto en una interfaz basada en promesas.

Ahora, promisifiquemos una función de Node.js utilizando nuestra función `promisify()` recién creada. Podemos usar la función `randomBytes()` del módulo central `crypto`, que produce un buffer que contiene el número especificado de bytes aleatorios. La función `randomBytes()` acepta un callback como último argumento y sigue las convenciones de callbacks que ya conocemos muy bien (el callback se llamará con dos argumentos: un error y un valor).

Veamos cómo se ve esto:

```javascript
import { randomBytes } from 'node:crypto'

const randomBytesP = promisify(randomBytes)
randomBytesP(32)
  .then(buffer => {
    console.log(`Random bytes: ${buffer.toString()}`)
  })
```

El código anterior debería imprimir una secuencia aleatoria en la consola; esto se debe a que la mayoría de los bytes generados aleatoriamente no corresponden a un carácter imprimible.

La función de promisificación que creamos aquí es solo para fines educativos y carece de algunas funciones, como la capacidad de lidiar con callbacks que devuelven más de un resultado. En la vida real, usaríamos la función `promisify()` del módulo central `util` para promisificar nuestras funciones basadas en callbacks al estilo Node.js. Puedes consultar su documentación en [nodejsdp.link/promisify](https://nodejsdp.link/promisify).

#### Ejecución e iteración secuencial

En este punto, ya sabemos lo suficiente como para convertir la aplicación web spider que creamos en el capítulo anterior para que utilice promesas. Comencemos directamente desde la versión 2, la que descarga los enlaces de una página web en secuencia.

Podemos acceder a una versión ya promisificada de la API central `fs` a través del objeto `promises` del módulo `fs` —por ejemplo: `import { promises } from 'node:fs'`. Alternativamente, también puedes importar desde `'node:fs/promises'`. Este patrón está presente en muchos otros módulos centrales de Node.js que se escribieron originalmente pensando solo en callbacks y luego se ampliaron con una versión promisificada alternativa de sus funciones. Algunos otros ejemplos notables son `'node:dns/promises'`, `'node:timers/promises'`, `'node:inspector/promises'`, `'node:readline/promises'` y `'node:stream/promises'`.

Ten en cuenta que en esta versión del código, hemos promisificado manualmente todas las funciones auxiliares que puedes encontrar en el archivo `utils.js` en el repositorio de código; por lo tanto, ya no estamos usando callbacks. Esto significa, por ejemplo, que ahora podemos llamar a `get()` sin una función callback y obtener a cambio una promesa que rastrea el progreso de la operación asíncrona.

Ahora, comencemos convirtiendo la función `download()` para usar promesas:

```javascript
function download(url, filename) {
  console.log(`Downloading ${url} into ${filename}`)
  return get(url)
    .then(content => saveFile(filename, content))
}
```

A modo de comparación, revisemos la versión anterior basada en callbacks de esta función tal como se implementó en el capítulo anterior:

```javascript
function download(url, filename, cb) {
  console.log(`Downloading ${url} into ${filename}`)
  get(url, (err, content) => {
    if (err) {
      return cb(err)
    }
    saveFile(filename, content, err => {
      if (err) {
        return cb(err)
      }
      cb(null, content)
    })
  })
}
```

¡Lo primero que deberíamos apreciar es que redujimos el código a casi un tercio! El nuevo código no solo es más corto, sino que también debería resultar más fácil de seguir. Casi se lee en lenguaje natural:
`get` (el contenido de) `url`, `then` (toma el) `content` (y) `saveFile` (con) `filename` (y) `content`.

Además, observa cómo estamos devolviendo la cadena de promesas al invocador. Esto hace que el invocador reciba otra promesa que eventualmente se resolverá con el resultado de la llamada a `saveFile()` (que es, a su vez, una promesa que se resuelve con el contenido que se escribió en el archivo). En otras palabras, esta función no solo descarga el contenido de una URL determinada en un archivo, sino que también expone ese mismo contenido al invocador. Esto es para garantizar que la semántica de nuestra nueva función `download()` no haya cambiado respecto a la versión anterior.

En la función `download()` que acabamos de ver, hemos ejecutado un conjunto conocido de operaciones asíncronas en secuencia: obtener el contenido, guardarlo en un archivo. Sin embargo, en la función `spiderLinks()`, tendremos que lidiar con una iteración secuencial sobre un conjunto dinámico de tareas asíncronas. Veamos cómo podemos lograrlo:

```javascript
function spiderLinks(currentUrl, body, maxDepth) {
  let promise = Promise.resolve() // 1
  if (maxDepth === 0) {
    return promise
  }
  const links = getPageLinks(currentUrl, body)
  for (const link of links) {
    promise = promise.then(() => spider(link, maxDepth - 1)) // 2
  }
  return promise
}
```

Te ahorraremos la comparación esta vez, pero si volvieras al capítulo anterior y compararas esta implementación con la anterior, verías que esta también es significativamente más corta.

La nueva e interesante pieza de conocimiento aportada por esta versión es que, para iterar sobre todos los enlaces de una página web de forma asíncrona, tuvimos que construir dinámicamente una cadena de promesas. Esto se logra de la siguiente manera:
1. Primero, definimos una Promise "vacía", que se resuelve en `undefined`. Esta Promise se utiliza simplemente como punto de partida para nuestra cadena.
2. Luego, en un bucle, actualizamos la variable `promise` con una nueva Promise obtenida invocando `then()` en la promesa anterior de la cadena. Esta es efectivamente una implementación de nuestro patrón de iteración asíncrona utilizando promesas.
3. Al final del bucle `for`, la variable `promise` contendrá la promesa de la última invocación de `then()`, por lo que se resolverá solo cuando todas las promesas de la cadena se hayan resuelto.

> **Patrón (Iteración secuencial con promesas):**
> Construye dinámicamente una cadena de promesas usando un bucle.

Una alternativa al patrón de iteración secuencial con promesas hace uso de la función `reduce()`, para una implementación aún más compacta:

```javascript
const promise = tasks.reduce((prev, task) => {
  return prev.then(() => {
    return task()
  })
}, Promise.resolve())
```

Ahora, finalmente podemos convertir la función `spider()`:

```javascript
export function spider(url, maxDepth) {
  const filename = urlToFilename(url)
  return exists(filename).then(alreadyExists => {
    if (alreadyExists) {
      if (!filename.endsWith('.html')) { // ignoring non-HTML resources
        return
      }
      return readFile(filename, 'utf8').then(fileContent =>
        spiderLinks(url, fileContent, maxDepth)
      )
    }
    // if file does not exist, download it
    return download(url, filename).then(fileContent => {
      // if the file is an HTML file, spider it
      if (filename.endsWith('.html')) {
        return spiderLinks(url, fileContent.toString('utf8'), maxDepth)
      }
      // otherwise, stop here
      return
    })
  })
}
```

En esta nueva versión de la función `spider()`, estamos aprovechando una nueva versión basada en promesas de la función auxiliar `exists()`. Esta función ahora devuelve una promesa que se resuelve con el valor booleano: `true` si la ruta del archivo dada existe, `false` en caso contrario. Podemos capturar este valor con `then()` y luego manejar las dos ramas de código diferentes: si el archivo existe (y es un archivo HTML), podemos leer su contenido y continuar rastreando desde allí; de lo contrario, tenemos que descargarlo antes de poder continuar con el rastreo. La lógica básica de la función no ha cambiado, pero ahora hemos logrado eliminar todos los callbacks del código. Un detalle importante a subrayar es que la naturaleza condicional de este código nos obliga a mantener cierto nivel de anidamiento. De hecho, no podemos simplemente poner todo en una única y agradable cadena de eventos, porque efectivamente tenemos que lidiar con dos rutas de código distintas. Veremos en la segunda parte de este capítulo cómo async/await hace que manejar la lógica condicional asíncrona sea tan fácil como manejarla en código síncrono.

Ahora que también hemos convertido nuestra función `spider()`, finalmente podemos modificar el módulo `spider-cli.js`:

```javascript
spider(url, maxDepth)
  .then(() => console.log('Downloaded complete'))
  .catch(err => {
    console.error(err)
    process.exit(1)
  })
```

Este debería ser bastante directo, pero es importante tener en cuenta que este es el primer lugar en esta nueva versión donde manejamos explícitamente los errores con un manejador `catch()`. Este manejador interceptará cualquier error originado en todo el proceso `spider()`, lo que destaca cómo las promesas simplifican la propagación de errores y reducen la cantidad de código necesario para el manejo de errores. Claramente, esta es una ventaja enorme, ya que reduce en gran medida el código repetitivo en nuestro código y las posibilidades de perder cualquier error asíncrono.

Esto completa la implementación de la versión 2 de nuestra aplicación web spider con promesas.

#### Ejecución concurrente

Otro flujo de ejecución que se simplifica con las promesas es el flujo de ejecución concurrente. De hecho, todo lo que tenemos que hacer es usar el método integrado `Promise.all()`. Esta función auxiliar crea otra Promise que se cumple solo cuando todas las promesas recibidas como entrada se cumplen. Si no hay relación causal entre esas promesas (por ejemplo, no forman parte de la misma cadena de promesas), se ejecutarán de forma concurrente.

Para demostrar esto, consideremos la versión 3 de nuestra aplicación web spider, que descarga todos los enlaces de una página concurrentemente. Actualicemos la función `spiderLinks()` nuevamente para implementar un flujo de ejecución concurrente utilizando promesas:

```javascript
function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) {
    return Promise.resolve()
  }
  const links = getPageLinks(currentUrl, body)
  const promises = links.map(link => spider(link, maxDepth - 1))
  return Promise.all(promises)
}
```

El patrón aquí consiste en iniciar las tareas todas a la vez en el bucle `links.map()`. Al mismo tiempo, cada Promise devuelta al invocar `spider()` se recopila en el array final `promises`. La diferencia crítica en este bucle —en comparación con el bucle de iteración secuencial— es que no estamos esperando a que se complete la tarea `spider()` anterior en la lista antes de comenzar una nueva. Todas las tareas `spider()` se inician en el bucle a la vez, en el mismo ciclo del bucle de eventos.

Una vez que tenemos todas las promesas, las pasamos al método `Promise.all()`, que devuelve una nueva Promise que se cumplirá cuando todas las promesas en el array se cumplan. En otras palabras, se cumple cuando todas las tareas de descarga se han completado. Además de eso, la Promise devuelta por `Promise.all()` se rechazará inmediatamente si alguna de las promesas en el array de entrada se rechaza. Esto es exactamente lo que queríamos para esta versión de nuestro web spider.

#### Ejecución concurrente limitada

Hasta ahora, las promesas no han defraudado nuestras expectativas. Pudimos mejorar en gran medida nuestro código tanto para la ejecución serial como para la concurrente. Ahora, con la ejecución concurrente limitada, las cosas no deberían ser tan diferentes, considerando que este flujo es solo una combinación de ejecución serial y concurrente.

En esta sección, pasaremos directamente a implementar una solución que nos permita limitar globalmente la concurrencia de nuestras tareas de web spider.

Si solo estás interesado en una solución simple para limitar localmente la ejecución concurrente de un conjunto de tareas, aún puedes aplicar los mismos principios que veremos en esta sección para implementar una versión asíncrona especial de `Array.map()`. Dejamos esto como ejercicio; puedes encontrar más detalles y pistas al final de este capítulo (Ejercicio 5.4 Un map() asíncrono).

Para una implementación lista para usar y preparada para producción de una función `map()` que admita promesas y concurrencia limitada, puedes confiar en el paquete `p-map`. Obtén más información en [nodejsdp.link/p-map](https://nodejsdp.link/p-map).

##### Implementación de la clase TaskQueue con promesas

Para limitar globalmente la concurrencia de nuestras tareas de descarga del spider, reutilizaremos la clase `TaskQueue` que implementamos originalmente en la versión 4 de nuestro rastreador, descrita en el capítulo anterior. En aquel entonces, la cola estaba diseñada para manejar tareas usando callbacks. En esta versión, sin embargo, queremos que maneje tareas que devuelvan promesas. Para admitir esto, necesitamos actualizar el método `next()`, que es responsable de despachar tareas mientras no hayamos alcanzado el límite de concurrencia.

Para una referencia rápida, he aquí la versión anterior de la función `next()`:

```javascript
next() {
  if (this.running === 0 && this.queue.length === 0) {
    return this.emit('empty')
  }
  while (this.running < this.concurrency && this.queue.length > 0) {
    const task = this.queue.shift()
    task(err => {
      if (err) {
        this.emit('error', err)
      }
      this.running--
      process.nextTick(this.next.bind(this))
    })
    this.running++
  }
}
```

Y aquí está la versión actualizada:

```javascript
next() {
  if (this.running === 0 && this.queue.length === 0) {
    return this.emit('empty')
  }
  while (this.running < this.concurrency && this.queue.length > 0) {
    const task = this.queue.shift()
    task() // 1
      .catch(err => { // 4
        this.emit('error', err)
      })
      .finally(() => { // 2
        this.running--
        this.next() // 3
      })
    this.running++
  }
}
```

Analicemos qué sucede aquí comparándolo con la implementación anterior basada en callbacks:
1. En la versión original, cada tarea aceptaba un callback y lo llamaba cuando terminaba. Ahora, cada tarea es una función que devuelve una promesa. Esto nos permite esperar operaciones asíncronas dentro de cada tarea sin necesidad de gestionar manualmente los callbacks.
2. Anteriormente, decrementábamos manualmente el contador `running` dentro del callback. Ahora, usamos el método `.finally()` de la promesa para asegurarnos de que `running` se decremente una vez que finalice la tarea, independientemente de si se resolvió o se rechazó.
3. Después de que finaliza una tarea, programamos la siguiente iteración de `next()`, al igual que antes.
4. En la versión original, necesitábamos verificar si había errores inspeccionando el argumento `err` pasado a la función callback, lo que requería que usáramos una sentencia `if` dedicada. Ahora, podemos usar `.catch()` para escuchar los rechazos de promesas y emitir un evento de error si algo sale mal. Esto hace que el manejo de errores sea más limpio y consistente.

Estos son los únicos cambios que necesitamos para adaptar nuestra clase `TaskQueue` para que use promesas en lugar de depender de callbacks. A continuación, utilizaremos esta nueva versión de la clase `TaskQueue` para implementar la versión 4 de nuestro web spider.

##### Actualización del web spider

Ahora es el momento de adaptar nuestro web spider para implementar un flujo de ejecución concurrente limitado utilizando la clase `TaskQueue` que acabamos de crear. También aprovecharemos esta oportunidad para aplicar algunas optimizaciones. Por ejemplo, incluiremos un conjunto que nos permita rastrear qué URLs se están procesando o ya se han procesado, para que podamos omitirlas en caso de que las encontremos nuevamente durante el proceso de rastreo.

Con esta idea en mente, comencemos copiando el código de `spider.js` de la versión 3 de nuestro spider basado en promesas y actualicemos primero la función `spiderLinks()`:

```javascript
const spidering = new Set() // 1

function spiderLinks(currentUrl, body, maxDepth, queue) { // 2
  if (maxDepth === 0) {
    return
  }
  const links = getPageLinks(currentUrl, body)
  for (const link of links) { // 3
    if (!spidering.has(link)) {
      queue.pushTask(() => spider(link, maxDepth - 1, queue))
      spidering.add(link)
    }
  }
}
```

He aquí los cambios:
1. En primer lugar, creamos el conjunto `spidering` para realizar un seguimiento de qué URLs están en curso o ya se han completado.
2. Añadimos `queue` (una instancia de nuestra clase `TaskQueue`) a la lista de argumentos en la función `spiderLinks()`.
3. En lugar de usar `Array.map()` y `Promise.all()` como hicimos en la versión anterior, ahora usamos un bucle `for` normal para recorrer todos los enlaces que se encuentran en la URL actual. Para cada enlace que aún no esté en el conjunto `spidering`, creamos una nueva tarea, la añadimos a la cola y actualizamos el conjunto `spidering` con este enlace. La tarea que creamos es una función flecha que llama a la función `spider()` con los parámetros correctos (observa cómo `maxDepth` disminuye en uno). Este enfoque pospone la ejecución de `spider()` hasta que la tarea se extrae de la cola. Este método nos permite controlar la concurrencia, ya que podemos seguir agregando tareas sin iniciarlas inmediatamente; se ejecutan solo cuando se extraen de la cola.

En esta versión, hemos recuperado el concepto del conjunto `spidering`, que introdujimos por primera vez en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks. Esta es una técnica de optimización que ayuda a reducir la cantidad de tareas y evita el trabajo duplicado innecesario. Los lectores observadores pueden notar que esta vez lo implementamos de manera ligeramente diferente. En el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks, verificamos si había duplicados dentro de la función `spider()`, pero ahora hemos movido esa verificación a la función `spiderLinks()`. Esta optimización adicional nos permite evitar la creación de tareas que de otro modo serían inútiles al extraerse de la cola. Esto mantiene el sistema más eficiente al limitar el crecimiento de la cola, algo que puede ser importante para sitios web con una gran cantidad de enlaces.

Ahora podemos actualizar la función `spider()`:

```javascript
export function spider(url, maxDepth, queue) { // 1
  const filename = urlToFilename(url)
  return exists(filename).then(alreadyExists => {
    if (alreadyExists) {
      if (!filename.endsWith('.html')) {
        return
      }
      return readFile(filename, 'utf8').then(fileContent =>
        spiderLinks(url, fileContent, maxDepth, queue) // 2
      )
    }
    return download(url, filename).then(fileContent => {
      if (filename.endsWith('.html')) {
        return spiderLinks(url, fileContent.toString('utf8'), maxDepth, queue) // 3
      }
      return
    })
  })
}
```

No ha cambiado mucho aquí en comparación con la versión anterior de nuestra función `spider()`. Solo necesitábamos introducir nuestra instancia de `queue` en la firma de la función (1) y luego asegurarnos de propagarla cuando llamamos a `spiderLinks()` (2, 3).

Lo último que debemos hacer es actualizar nuestro script de CLI (`spider-cli.js`):

```javascript
const url = process.argv[2]
const maxDepth = Number.parseInt(process.argv[3], 10) || 1
const concurrency = Number.parseInt(process.argv[4], 10) || 2 // 1

const queue = new TaskQueue(concurrency) // 2
queue.pushTask(() => spider(url, maxDepth, queue)) // 3
queue.on('error', console.error) // 4
queue.on('empty', () => {
  console.log('Download complete')
})
```

Este código no es muy diferente de lo que hicimos en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks cuando implementamos el patrón de ejecución concurrente limitada usando callbacks. He aquí un resumen rápido de lo que cambió respecto a nuestro web spider v3 basado en promesas:
1. Ahora aceptamos un argumento de CLI adicional para permitir al usuario especificar la cantidad máxima deseada de tareas concurrentes.
2. Creamos una instancia de `TaskQueue` con la concurrencia dada.
3. Añadimos la primera tarea a la cola, que iniciará el rastreo de la URL dada. Ten en cuenta que debemos asegurarnos de pasar la instancia de la cola que acabamos de crear a la función `spider` para que la use internamente y continúe programando nuevas tareas en esa cola.
4. Finalmente, adjuntamos algunos escuchadores de eventos a la cola para saber si hay un error (y registrarlo en la salida de error estándar) y para saber cuándo se han procesado todas las tareas.

¡Eso es todo! Ahora finalmente puedes probar esta nueva versión. Intenta usarla en un sitio web significativamente grande y observa cómo se comporta. Puedes jugar con diferentes valores para los parámetros `concurrency` y `maxDepth` para ver qué tan rápido puedes descargar páginas del sitio web. Por supuesto, recuerda siempre las implicaciones legales del scraping, por lo que es mejor si puedes realizar estas pruebas contra un sitio web que poseas o, mejor aún, un sitio web que puedas ejecutar en `localhost`.

En el código de producción, puedes usar el paquete `p-limit` (disponible en [nodejsdp.link/p-limit](https://nodejsdp.link/p-limit)) para limitar la concurrencia de un conjunto de tareas. El paquete esencialmente implementa el patrón que acabamos de mostrar, pero envuelto en una API ligeramente diferente. Si tienes curiosidad, puedes consultar la implementación. Probablemente no te sorprenderá demasiado descubrir que esta biblioteca también utiliza internamente una cola para gestionar la programación de tareas.

#### Promesas diferidas (*Lazy promises*)

Hasta ahora, hemos aprendido que podemos crear una instancia de Promise utilizando su constructor canónico:

```javascript
new Promise((resolve, reject) => {
  // promise execution logic …
})
```

La función que pasamos al constructor de Promise se llama **función ejecutora** (*executor function*), y es donde definimos el ciclo de vida de una Promise personalizada. Es importante saber que la función ejecutora se invoca de inmediato (sincrónicamente) tan pronto como se crea el objeto Promise.

Este detalle importa porque, a veces, es posible que necesitemos ejecutar operaciones que consumen muchos recursos (como leer un archivo grande o acceder a un recurso remoto), o que necesitemos crear una gran cantidad de instancias de Promise que solo necesitamos usar más adelante en el código. En otros casos, es posible que deseemos crear una Promise pero tengamos una lógica condicional que podría terminar sin necesitarla (por ejemplo, cargar un archivo de configuración predeterminado si el usuario no ha proporcionado una configuración explícita en tiempo de ejecución).

En tales casos, puede ser beneficioso posponer la creación de la instancia de Promise (y, por tanto, la ejecución de su función ejecutora) hasta que realmente necesitemos interactuar con ella, como al llamar a `then()`, `catch()` o `finally()`. Al diferir la ejecución de la función ejecutora hasta este punto, podemos evitar el uso innecesario de recursos, lo que ayuda a mantener nuestro código más eficiente.

La clase Promise no admite este tipo de instanciación diferida de forma nativa, pero podemos implementarla nosotros mismos de varias maneras. De hecho, ya hemos implementado una forma de Promise diferida o perezosa en la versión 4 de nuestro raspador web basado en promesas. Allí, necesitábamos una forma de programar una tarea (rastrear una URL) sin iniciarla de inmediato. Para hacer esto, simplemente envolvimos la instanciación de la Promise en una función flecha, un patrón que podemos generalizar de la siguiente manera:

```javascript
const lazyPromise = () => new Promise((resolve, reject) => {
  // …
})
```

Este código no crea la instancia de Promise de inmediato. En su lugar, crea una función que, cuando se llame, creará la instancia de Promise. Ahora podemos usar esta función de la siguiente manera:

```javascript
lazyPromise().then((value) => {
  // …
})
```

Observa que debemos invocar explícitamente `lazyPromise()` antes de poder usar `then()` en ella. Si bien este enfoque funciona, no crea verdaderos objetos Promise perezosos; en su lugar, simplemente nos da una función que necesita ser llamada para producir una Promise.

Técnicamente hablando, este enfoque debería considerarse una implementación del patrón **función factoría** (*factory function*, cubierto con más detalle en el [Capítulo 7](https://subscription.packtpub.com/book/web-development/9781803238944/7), Patrones de diseño creacionales) en lugar de un verdadero objeto de promesa perezosa.

Al trabajar con bibliotecas de terceros que esperan objetos Promise simples, este enfoque para diferir la inicialización de la Promise no funcionará. El código de terceros eventualmente intentará llamar a `then()` directamente en nuestra función sin invocarla primero, lo que provocará el siguiente error:

```text
Uncaught TypeError: lazyPromise.then is not a function
```

Un enfoque alternativo podría ser implementar una extensión de la clase Promise que se comporte de forma perezosa. Llamémosla `LazyPromise`:

```javascript
export class LazyPromise extends Promise { // 1
  #resolve // 2
  #reject
  #executor
  #promise
  constructor(executor) { // 3
    let _resolve
    let _reject
    super((resolve, reject) => {
      _resolve = resolve
      _reject = reject
    })
    this.#resolve = _resolve
    this.#reject = _reject
    this.#executor = executor
    this.#promise = null
  }
  #ensureInit() { // 4
    if (!this.#promise) {
      this.#promise = new Promise(this.#executor)
      this.#promise.then(
        v => this.#resolve(v),
        e => this.#reject(e)
      )
    }
  }
  then(onFulfilled, onRejected) { // 5
    this.#ensureInit()
    return this.#promise.then(onFulfilled, onRejected)
  }
  catch(onRejected) {
    this.#ensureInit()
    return this.#promise.catch(onRejected)
  }
  finally(onFinally) {
    this.#ensureInit()
    return this.#promise.finally(onFinally)
  }
}
```

Veamos cómo funciona esto:
1. La clase `LazyPromise` extiende la clase `Promise` integrada, heredando todas sus funcionalidades. Esto también permite que las instancias de esta clase se consideren instancias de la clase `Promise` original, lo que significa que escribir algo como `lazyPromise instanceof Promise` (donde `lazyPromise` es una instancia de `LazyPromise`) se evaluará como `true`.
2. Definimos algunas propiedades privadas de la clase. Estas se utilizarán para realizar un seguimiento del ciclo de vida de la promesa perezosa, pero no serán accesibles desde fuera de la clase. (La notación `#` es una sintaxis relativamente nueva introducida en ECMAScript 2022. Si deseas obtener más información sobre ella, consulta [nodejsdp.link/private-properties](https://nodejsdp.link/private-properties)).
3. Redefinimos el constructor original de Promise. Aquí es donde ocurre la mayor parte de la magia. Necesitamos respetar la misma interfaz que en el constructor de Promise original, lo que significa que recibimos solo un argumento: la función ejecutora. Pero en esta implementación, no invocamos inmediatamente la función ejecutora que recibimos, sino que simplemente la almacenamos como una propiedad privada, junto con sus funciones `resolve` y `reject`. También creamos una propiedad `promise` que inicialmente se establece en `null`. Este es el valor que completaremos solo cuando finalmente ejecutemos la función ejecutora para crear la instancia real de Promise.
4. Definimos una función privada llamada `ensureInit()`. Esta función es responsable de inicializar la instancia interna de la promesa cuando sea necesario y conectarla a las funciones `resolve()` y `reject()` originales del ejecutor. Ten en cuenta que esta función realizará la inicialización solo si la promesa interna aún no está inicializada.
5. Finalmente, reimplementamos los métodos `then()`, `catch()` y `finally()`. Estos son los métodos que llama el usuario cuando está interesado en el resultado de la promesa. Por lo tanto, aquí es donde necesitamos inicializar la promesa interna llamando a `ensureInit()` y conectar el callback recibido a la instancia de promesa interna.

Veamos un ejemplo simple que ilustra cómo podemos usar esta nueva clase:

```javascript
const lazyPromise = new LazyPromise(resolve => { // 1
  console.log('Executor Started!')
  // simulate some async work to be done
  setTimeout(() => {
    resolve('Completed!')
  }, 1000)
})
console.log('Lazy Promise instance created!')
console.log(lazyPromise)
lazyPromise.then(value => { // 2
  console.log(value)
  console.log(lazyPromise)
})
```

Este código es bastante sencillo. Simplemente estamos creando una instancia de `LazyPromise` y proporcionando una función ejecutora que resolverá la promesa después de 1 segundo (1). Más adelante, usamos `lazyPromise` llamando a `then()` en ella (2). Esta acción desencadena la ejecución de la función ejecutora que definimos previamente. Para demostrar que esto funciona como se esperaba, observemos los registros producidos por este simple fragmento:

```text
Lazy Promise instance created!
LazyPromise [Promise] { <pending> }
Executor Started!
< 1 second pause... >
Completed!
LazyPromise [Promise] { 'Completed!' }
```

Como podemos ver en los registros, `Lazy Promise instance created!` se registra antes de `Executor Started!`. Esto demuestra que la función ejecutora no se invoca de inmediato, sino solo más tarde cuando llamamos a `then()`.

También es interesante observar cómo, cuando llamamos a `console.log(lazyPromise)`, el estado de la promesa realiza un seguimiento correcto del estado interno de nuestra promesa perezosa. Una vez creada, la promesa se informa como `<pending>`. Más adelante, cuando la promesa se resuelve, su estado se informa correctamente como resuelta al valor `'Completed!'`.

Ahora que sabes cómo escribir verdaderas promesas perezosas, podrías asumir un pequeño desafío e intentar reescribir la versión 4 de nuestro rastreador web basado en promesas utilizando la clase `LazyPromise` como una forma de definir tareas.

Una biblioteca popular que implementa la idea de promesas perezosas es `p-lazy`. Consúltala en [nodejsdp.link/p-lazy](https://nodejsdp.link/p-lazy) y siéntete libre de comparar su implementación con la que proporcionamos aquí. Encontrarás muchas similitudes, pero también notarás que `p-lazy` ofrece algunas utilidades adicionales, por lo que probablemente sea una mejor opción para su uso en producción.

Esto concluye nuestra exploración de las promesas de JavaScript. A continuación, aprenderemos sobre el par async/await, que revolucionará por completo la forma en que manejamos el código asíncrono.

---

### Sección 2: Async/await

Como acabamos de ver, las promesas son un salto cuántico respecto a los callbacks. Nos permiten escribir código asíncrono limpio y legible y proporcionan un conjunto de salvaguardas que solo se pueden lograr con código repetitivo al trabajar con código asíncrono basado en callbacks. Sin embargo, las promesas siguen siendo subóptimas cuando se trata de escribir código asíncrono secuencial. La cadena de promesas es ciertamente mucho mejor que tener callback hell, pero aun así, tenemos que invocar un `then()` y crear una nueva función para cada tarea en la cadena. Esto sigue siendo demasiado para un flujo de control que definitivamente es el más utilizado en la programación diaria. JavaScript necesitaba una forma adecuada de abordar el omnipresente flujo de ejecución secuencial asíncrono, y la respuesta llegó con la introducción en el estándar ECMAScript de las funciones `async` y la expresión `await` (async/await para abreviar).

El par async/await nos permite escribir funciones que parecen bloquearse en cada operación asíncrona, esperando los resultados antes de continuar con la siguiente instrucción. Como veremos, cualquier código asíncrono que use async/await tiene una legibilidad comparable a la del código síncrono tradicional.

Hoy en día, async/await es la construcción recomendada para trabajar con código asíncrono tanto en Node.js como en JavaScript. Sin embargo, async/await no reemplaza todo lo que hemos aprendido hasta ahora sobre los patrones de flujo de control asíncrono; al contrario, como veremos, async/await se apoya en gran medida en las promesas.

#### Funciones async y la expresión await

Una función `async` es un tipo especial de función en la que es posible utilizar la expresión `await` para "pausar" la ejecución en una Promise determinada hasta que se resuelva. Consideremos un ejemplo simple y usemos la función `delay()` que implementamos en la subsección *Creación de una promesa*. La Promise devuelta por `delay()` se resuelve con la fecha actual como valor después del número de milisegundos indicado. Usemos esta función con el par async/await:

```javascript
async function playingWithDelays() {
  console.log('Delaying...', Date.now())
  const timeAfterOneSecond = await delay(1000)
  console.log(timeAfterOneSecond)
  const timeAfterThreeSeconds = await delay(3000)
  console.log(timeAfterThreeSeconds)
  return 'done'
}
```

Como podemos ver en la función anterior, async/await parece funcionar como por arte de magia. El código ni siquiera parece contener ninguna operación asíncrona. Sin embargo, no te equivoques; esta función no se ejecuta sincrónicamente (¡se llaman funciones async por una razón!). En cada expresión `await`, la ejecución de la función se pone en espera, se guarda su estado y se devuelve el control al bucle de eventos. Una vez que se resuelve la Promise que se ha esperado, el control se devuelve a la función async, devolviendo el valor de cumplimiento de la Promise.

La expresión `await` se puede utilizar con una Promise, un objeto thenable (un objeto con un método `then()`) o cualquier otro valor. Si el valor no es un thenable, JavaScript lo envuelve automáticamente en una Promise resuelta utilizando `Promise.resolve()`. Por ejemplo, `await 5` es equivalente a `await Promise.resolve(5)`.

Veamos ahora cómo podemos invocar nuestra nueva función async:

```javascript
playingWithDelays()
  .then(result => {
    console.log(`After 4 seconds: ${result}`)
  })
```

A partir del código anterior, queda claro que las funciones async se pueden invocar como cualquier otra función. Sin embargo, los más observadores de ustedes ya habrán detectado otra propiedad importante de las funciones async: **siempre devuelven una Promise**. Es como si el valor de retorno de una función async se pasara a `Promise.resolve()` y luego se devolviera al invocador.

Llamar a una función async ocurre de inmediato, al igual que cualquier otra llamada a una función normal. Sin embargo, en lugar de devolver un valor directamente, devuelve una Promise de inmediato. Esa Promise se resolverá más tarde, ya sea cumpliéndose con el valor de retorno de la función o rechazándose si se lanza un error.

Desde este primer encuentro con async/await, podemos ver cuán dominantes siguen siendo las promesas en nuestra discusión. De hecho, podemos considerar async/await simplemente como azúcar sintáctico para un consumo más simple de promesas. Como veremos, todos los patrones de flujo de control asíncrono con async/await utilizan promesas y su API para la mayoría de las operaciones pesadas.

#### Await en el nivel superior (*Top-level await*)

A partir de Node.js 14, obtenemos acceso a una potente característica llamada **top-level await** cuando trabajamos con Módulos ECMAScript (ESM). Esta característica nos permite usar la palabra clave `await` directamente en el nivel superior de nuestro módulo, fuera de cualquier función async.

Recuerda que, para habilitar ESM, todo lo que necesitamos hacer es asegurarnos de que nuestro archivo tenga una extensión `.mjs` o debemos establecer `"type": "module"` en `package.json`. Esto se discutió en el [Capítulo 2](https://subscription.packtpub.com/book/web-development/9781803238944/2), El sistema de módulos.

Con top-level await, podemos hacer que nuestro código asíncrono sea más simple y legible. Ya no es necesario llamar a `then()` en promesas ni crear envoltorios complejos; solo acceso directo y limpio a operaciones asíncronas. Supongamos que queremos llamar a la función asíncrona `playingWithDelays()` que escribimos en la sección anterior. En la sección anterior, vimos cómo podemos invocar la función y usar `then()` en la promesa devuelta, pero también podríamos haberla llamado de esta manera:

```javascript
(async () => {
  const result = await playingWithDelays()
  console.log(`After 4 seconds: ${result}`)
})()
```

Esto es lo que llamamos una IIFE asíncrona (*Immediately Invoked Function Expression*). Dentro del primer par de paréntesis, definimos una función flecha async en línea y la invocamos inmediatamente con el par final de paréntesis.

Ahora, con top-level await, podemos eliminar estos pasos adicionales y simplificar nuestro código aún más, así:

```javascript
const result = await playingWithDelays()
console.log(`After 4 seconds: ${result}`)
```

Este código es mucho más limpio, menos detallado y más fácil de leer.

Esta técnica puede ser muy útil en algunos casos de uso prácticos cuando necesitas realizar una inicialización asíncrona antes de que se pueda ejecutar tu lógica de negocio principal. He aquí algunos ejemplos:
- **Conexión a una base de datos:** Si nuestra aplicación necesita conectarse a una base de datos, ahora podemos configurar esa conexión desde el principio, asegurándonos de que todo esté listo antes de gestionar las solicitudes.
- **Obtención de configuraciones:** Es posible que debas obtener la configuración de un archivo o secretos de un servicio de administración de secretos remoto. Podrías usar top-level await para obtener esta configuración antes de que se ejecute la lógica de la aplicación.
- **Inicialización de otras dependencias remotas:** Cuando nuestro módulo depende de servicios de terceros o APIs externas, podemos inicializarlos directamente en el nivel superior, lo que hace que sea claro y fácil ver de qué depende nuestra aplicación antes de que se ejecute.

#### Manejo de errores con async/await

Async/await no solo mejora la legibilidad del código asíncrono en condiciones estándar, sino que también ayuda a la hora de gestionar errores. De hecho, una de las mayores ventajas de async/await es la capacidad de normalizar el comportamiento del bloque `try...catch`, para que funcione a la perfección tanto con excepciones síncronas lanzadas con `throw` como con rechazos de Promise asíncronos. Demostrémoslo con un ejemplo.

##### Una experiencia try...catch unificada

Definamos una función que devuelva una Promise que se rechace con un error después de un número determinado de milisegundos. Esto es muy similar a la función `delay()` que ya conocemos muy bien:

```javascript
function delayError(milliseconds) {
  return new Promise((_resolve, reject) => {
    setTimeout(() => {
      reject(new Error(`Error after ${milliseconds}ms`))
    }, milliseconds)
  })
}
```

A continuación, implementemos una función async que pueda tanto lanzar un error sincrónicamente como esperar una Promise que se rechazará. Esta función demuestra cómo tanto el error síncrono como el rechazo de la Promise son capturados por el mismo bloque `catch`:

```javascript
async function playingWithErrors(throwSyncError) {
  try {
    if (throwSyncError) {
      throw new Error('This is a synchronous error')
    }
    await delayError(1000)
  } catch (err) {
    console.error(`We have an error: ${err.message}`)
  } finally {
    console.log('Done')
  }
}
```

Ahora, invoca la función de esta manera:

```javascript
playingWithErrors(true)
```

Esto imprimirá lo siguiente en la consola:

```text
We have an error: This is a synchronous error
Done
```

Invoca la función con `false` como entrada, así:

```javascript
playingWithErrors(false)
```

Esto producirá la siguiente salida:

```text
We have an error: Error after 1000ms
Done
```

Si recordamos cómo tuvimos que lidiar con los errores en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con Callbacks, sin duda apreciaremos las gigantescas mejoras introducidas tanto por las promesas como por async/await. Ahora, el manejo de errores es justo como debería ser: simple, legible y, lo más importante, funciona de la misma manera tanto para errores síncronos como asíncronos.

##### La trampa de "return" frente a "return await"

Un antipatrón común al lidiar con errores con async/await es devolver una Promise que se rechaza al invocador y esperar que el error sea capturado por el bloque local `try...catch` de la función async.

Por ejemplo, considera el siguiente código:

```javascript
async function errorNotCaught() {
  try {
    return delayError(1000)
  } catch (err) {
    console.error('Error caught by the async function: ' + err.message)
  }
}

errorNotCaught()
  .catch(err => console.error('Error caught by the caller: ' + err.message))
```

La Promise devuelta por `delayError()` no se espera localmente, lo que significa que se devuelve tal como está al invocador. En consecuencia, el bloque `catch` local nunca se invocará. De hecho, el código anterior generará lo siguiente:

```text
Error caught by the caller: Error after 1000ms
```

Si nuestra intención es capturar localmente cualquier error generado por la operación asíncrona que produce el valor que queremos devolver, entonces debemos usar la expresión `await` en esa Promise antes de devolver el valor al invocador. El siguiente código demuestra esto:

```javascript
async function errorCaught() {
  try {
    return await delayError(1000)
  } catch (err) {
    console.error('Error caught by the async function: ' + err.message)
  }
}

errorCaught()
  .catch(err => console.error('Error caught by the caller: ' + err.message))
```

Todo lo que hicimos fue agregar un `await` después de la palabra clave `return`. Esto es suficiente para hacer que la función async "gestione" la Promise localmente y, por lo tanto, también capture cualquier rechazo de forma local. Como confirmación, cuando ejecutamos el código anterior, deberíamos ver la siguiente salida:

```text
Error caught by the async function: Error after 1000ms
```

Otro escenario donde `return await` puede ser útil es durante la depuración. Cuando usas `return await` dentro de una función async, mantiene el marco (*frame*) de esa función en la pila de llamadas hasta que la Promise esperada se resuelva. Aunque esto introduce un costo de rendimiento muy leve (una microtarea adicional antes de que se resuelva la Promise externa), puede brindarte un seguimiento de la pila más completo, facilitando rastrear el origen de la Promise. Si tienes curiosidad por profundizar en este balance, consulta esta excelente publicación del equipo de V8: [nodejsdp.link/fast-async](https://nodejsdp.link/fast-async).

#### Ejecución e iteración secuencial

Nuestra exploración de los patrones de flujo de control con async/await comienza con la ejecución y la iteración secuencial. Ya mencionamos varias veces que la fortaleza central de async/await radica en su capacidad de hacer que la ejecución serial asíncrona sea fácil de escribir y directa de leer. Esto ya era evidente en todas las muestras de código que hemos escrito hasta ahora; sin embargo, se volverá aún más obvio ahora que comenzaremos a convertir nuestra versión 2 del web spider. Async/await es tan simple de usar y comprender que realmente no hay patrones aquí para estudiar. Iremos directo al código, sin ningún preámbulo.

Comencemos, pues, con nuestras funciones `saveFile()` y `download()` de nuestro web spider; así es como se ven con async/await:

```javascript
async function saveFile(filename, content) {
  await recursiveMkdir(dirname(filename))
  return writeFile(filename, content)
}

async function download(url, filename) {
  console.log(`Downloading ${url} into ${filename}`)
  const content = await get(url)
  await saveFile(filename, content)
  return content
}
```

Apreciemos por un momento lo simples y compactas que se han vuelto estas dos funciones. Consideremos que la misma funcionalidad se implementó con callbacks en dos funciones diferentes utilizando más de 20 líneas de código. Ahora solo tenemos 10. Además, el código ahora es completamente plano, sin ningún anidamiento. Esto nos dice mucho sobre el enorme impacto positivo que async/await tiene en nuestro código.

Ahora, veamos cómo podemos iterar asíncronamente sobre un array usando async/await. Esto se ejemplifica en la función `spiderLinks()`:

```javascript
async function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) {
    return
  }
  const links = getPageLinks(currentUrl, body)
  for (const link of links) {
    await spider(link, maxDepth - 1)
  }
}
```

Incluso aquí, no hay ningún patrón que aprender. Solo tenemos una simple iteración sobre una lista de enlaces y, para cada elemento, esperamos la Promise devuelta por `spider()`.

El siguiente fragmento de código muestra la función `spider()` implementada usando async/await:

```javascript
export async function spider(url, maxDepth) {
  const filename = urlToFilename(url)
  let content
  if (!(await exists(filename))) {
    // if the file does not exist, download it
    content = await download(url, filename)
  }
  // if the file is not an HTML file, stop here
  if (!filename.endsWith('.html')) {
    return
  }
  // if file content is not already loaded, load it from disk
  if (!content) {
    content = await readFile(filename)
  }
  // spider the links in the file
  return spiderLinks(url, content.toString('utf8'), maxDepth)
}
```

Si comparas esta implementación con la anterior usando promesas, puedes ver que aquí pudimos reestructurar la lógica un poco. Con promesas, es difícil expresar lógica condicional, por lo que teníamos cierta repetición en nuestro código (estábamos llamando a `spiderLinks()` en dos ramas diferentes). Aquí, dado que async/await facilita el manejo de condiciones, pudimos lograr una mejor estructura de código y eliminar la llamada duplicada. Los comentarios en el código deberían aclarar exactamente cuál es el comportamiento previsto. Esto hace que el código sea más fácil de mantener a largo plazo.

Con la función `spider()`, hemos completado la conversión de nuestra aplicación web spider a async/await. Como puedes ver, ha sido un proceso muy fluido, pero los resultados son impresionantes.

##### Antipatrón: uso de async/await con Array.forEach para ejecución serial

Vale la pena mencionar que existe un antipatrón común mediante el cual los desarrolladores intentan usar `Array.forEach()` o `Array.map()` para implementar una iteración asíncrona secuencial con async/await, lo cual no funcionará como se espera.

Para ver por qué, echemos un vistazo a la siguiente implementación alternativa (¡que es incorrecta!) de la iteración asíncrona en la función `spiderLinks()`:

```javascript
links.forEach(async function iteration(link) {
  await spider(link, nesting - 1)
})
```

En el código anterior, la función `iteration` se invoca una vez para cada elemento del array `links`. Luego, en la función `iteration`, usamos la expresión `await` en la Promise devuelta por `spider()`. Sin embargo, `forEach()` simplemente ignora la Promise devuelta por la función `iteration`. El resultado es que todas las funciones `spider()` se invocan en la misma ronda del bucle de eventos, lo que significa que se inician de forma concurrente, y la ejecución continúa inmediatamente después de invocar `forEach()`, sin esperar a que se completen todas las operaciones `spider()`.

#### Ejecución concurrente

Hay principalmente dos formas de ejecutar un conjunto de tareas concurrentemente usando async/await: una usa puramente la expresión `await` y la otra se basa en `Promise.all()`. Ambas son muy sencillas de implementar; sin embargo, ten en cuenta que el método que depende de `Promise.all()` es el recomendado (y óptimo) para usar.

Veamos un ejemplo de ambos. Consideremos la función `spiderLinks()` de nuestro web spider (v3). Si quisiéramos utilizar puramente la expresión `await` para implementar un flujo de ejecución asíncrono concurrente ilimitado, lo haríamos con un código como el siguiente:

```javascript
async function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) {
    return
  }
  const links = getPageLinks(currentUrl, body)
  const promises = links.map(link => spider(link, maxDepth - 1))
  return Promise.all(promises)
}
```

¡Eso es todo! Muy sencillo. En el código anterior, primero iniciamos todas las tareas `spider()` de forma concurrente, recopilando sus promesas con un `map()`. Luego, podemos esperar (`await`) cada una de esas promesas o pasarlas a `Promise.all()`.

Si intentamos hacer un bucle esperando cada una de las promesas del array individualmente, parecería limpio y funcional; sin embargo, tiene un pequeño efecto no deseado: si una Promise en el array se rechaza, tenemos que esperar a que todas las promesas anteriores en el array se resuelvan antes de que la Promise devuelta por `spiderLinks()` también se rechace. Esto no es óptimo en la mayoría de las situaciones, ya que normalmente queremos saber si una operación ha fallado lo antes posible.

Afortunadamente, ya tenemos una función integrada que se comporta exactamente de la manera que queremos, y esa es `Promise.all()`. De hecho, `Promise.all()` se rechazará tan pronto como cualquiera de las promesas proporcionadas en el array de entrada se rechace. Por lo tanto, simplemente podemos confiar en este método incluso para todo nuestro código async/await. Y, dado que `Promise.all()` devuelve simplemente otra Promise, podemos invocar un `await` en ella para obtener los resultados de múltiples operaciones asíncronas. El siguiente código muestra un ejemplo:

```javascript
const results = await Promise.all(promises)
```

Por lo tanto, para resumir, nuestra implementación recomendada de la función `spiderLinks()` con ejecución concurrente y async/await se verá casi idéntica a la que usa promesas. La única diferencia visible es el hecho de que ahora estamos usando una función async, que siempre devuelve una Promise:

```javascript
async function spiderLinks (currentUrl, content, nesting) {
  if (nesting === 0) {
    return
  }
  const links = getPageLinks(currentUrl, content)
  const promises = links.map(link => spider(link, nesting - 1))
  return Promise.all(promises)
}
```

Al comienzo de este capítulo, mencionamos la función `Promise.allSettled()`. Esta función es similar a `Promise.all()`, pero en lugar de rechazarse cuando una promesa en la secuencia falla, continuará procesando todas las demás promesas y se resolverá solo cuando todas las promesas dadas estén resueltas (*settled*). El resultado de esta operación es un array que describe el estado de cada promesa (cumplida o rechazada) y el valor o error respectivo en el que se resolvió cada promesa. Esta es una excelente opción cuando deseas realizar una ejecución concurrente mientras toleras errores. Si deseas que nuestro rastreador continúe en presencia de fallas ocasionales, puedes intercambiar `Promise.all()` por `Promise.allSettled()` en el fragmento de código anterior.

Lo que acabamos de aprender sobre la ejecución concurrente y async/await simplemente reitera el hecho de que async/await es inseparable de las promesas. La mayoría de las utilidades que funcionan con promesas también funcionarán a la perfección con async/await, y nunca debemos dudar en aprovecharlas en nuestras funciones async.

#### Ejecución concurrente limitada

Para implementar un patrón de ejecución concurrente limitada con async/await, simplemente podemos reutilizar la clase `TaskQueue` que creamos en la subsección *Ejecución concurrente limitada* dentro de la sección *Promesas*. Podemos usarla tal como está o convertir sus componentes internos a async/await (como ya hicimos para la versión 3 de nuestro rastreador web basado en async/await). Convertir la clase `TaskQueue` a async/await es una operación simple, y te dejamos esto como un desafío rápido. De cualquier manera, la interfaz externa de `TaskQueue` no debería cambiar.

Puedes consultar el repositorio de código para ver nuestra implementación completa, y no debería ser demasiado sorprendente. Una cosa que podría valer la pena destacar es nuestro uso de la función `once()` del módulo central `'node:events'` en el archivo `spider-cli.js`.

En la versión 4 de nuestra versión del web spider basada en promesas, estábamos haciendo algo como esto:

```javascript
// ...
const queue = new TaskQueue(concurrency)
queue.pushTask(() => spider(url, maxDepth, queue))
queue.on('error', console.error)
queue.on('empty', () => {
  console.log('Download complete')
})
```

Observa el uso de `queue.on('empty', ...)` al final. Esperamos que este evento ocurra solo una vez, cuando se hayan procesado todas las tareas.

En esta nueva versión basada en async/await, hemos hecho algo ligeramente diferente que se ve así:

```javascript
import { once } from 'node:events'

// ...
const queue = new TaskQueue(concurrency)
queue.pushTask(() => spider(url, maxDepth, queue))
queue.on('taskError', console.error)
await once(queue, 'empty')
console.log('Download complete')
```

Reemplazamos la llamada `queue.on('empty', ...)` con `await once(queue, 'empty')`.

La función `once()` crea una instancia de Promise que se cumple cuando el `EventEmitter` dado (`queue` en nuestro caso) emite el evento dado (`'empty'` en nuestro caso) o se rechaza si el `EventEmitter` emite `'error'` mientras espera.

Este enfoque nos permite utilizar top-level await para esperar la finalización del rastreo desde nuestro ayudante CLI. Este no es un cambio fundamental, sino un ayudante muy práctico que vale la pena conocer cuando se trabaja con eventos y async/await.

Además, observa cómo tuvimos que cambiar el nombre del evento `'error'` a `'taskError'`, ya que el evento `'error'` tiene un significado especial. De hecho, se considera un error global irrecuperable (y, por lo tanto, rechaza la promesa devuelta por `once()`). En nuestro caso, queremos continuar el rastreo incluso si falla una URL, por lo que tuvimos que introducir un nombre diferente para manejarlo.

---

### Sección 3: El problema de las cadenas infinitas de resolución recursiva de promesas

En este punto del capítulo, deberías tener una sólida comprensión de cómo funcionan las promesas y cómo usarlas para implementar las construcciones de flujo de control más comunes. Este es el momento adecuado para discutir un tema avanzado que todo desarrollador profesional de Node.js debe conocer y comprender. Este tema avanzado trata sobre una fuga de memoria (*memory leak*) causada por cadenas infinitas de resolución de promesas. El error afecta a la especificación Promises/A+ en sí, por lo que ninguna implementación compatible es inmune.

Es bastante común en programación tener tareas que no tienen un final predefinido o que toman como entrada un flujo potencialmente infinito de datos. Podemos incluir en esta categoría cosas como la codificación/decodificación de transmisiones de audio/video en vivo, el procesamiento de datos del mercado de criptomonedas en tiempo real y la monitorización de sensores IoT. Pero no necesitas escenarios tan complejos para encontrarte con los mismos desafíos. De hecho, estas situaciones pueden surgir incluso en código más ordinario. Por ejemplo, al utilizar patrones de programación funcional de forma exhaustiva, es fácil crear operaciones recursivas o que se repiten a sí mismas y que nunca terminan a menos que se detengan explícitamente.

Para tomar un ejemplo simple, consideremos el siguiente código, que define una operación infinita simple utilizando promesas:

```javascript
function leakingLoop() {
  return delay(1)
    .then(() => {
      console.log(`Tick ${Date.now()}`)
      return leakingLoop()
    })
}
```

La función `leakingLoop()` que acabamos de definir usa la función `delay()` (que creamos al principio de este capítulo) para simular una operación asíncrona. Cuando ha transcurrido el número dado de milisegundos, imprimimos la marca de tiempo actual e invocamos `leakingLoop()` recursivamente para comenzar la operación de nuevo. La parte interesante es que la Promise devuelta por `leakingLoop()` nunca se resuelve porque su estado depende de la siguiente invocación de `leakingLoop()`, que a su vez depende de la siguiente invocación de `leakingLoop()`, y así sucesivamente. Esta situación crea una cadena de promesas que nunca se resuelven (*never settle*), y provocará una fuga de memoria en las implementaciones de Promise que siguen estrictamente la especificación Promises/A+, incluidas las promesas ES6 de JavaScript.

Para demostrar la fuga, podemos intentar ejecutar la función `leakingLoop()` muchas veces para acentuar sus efectos:

```javascript
for (let i = 0; i < 1e6; i++) {
  leakingLoop()
}
```

Luego podemos echar un vistazo a la huella de memoria del proceso usando nuestro inspector de procesos favorito y notar cómo crece indefinidamente hasta que (después de unos minutos) el proceso se bloquea por completo.

La solución al problema es romper la cadena de resolución de las promesas. Podemos hacer eso asegurándonos de que el estado de la Promise devuelta por `leakingLoop()` no dependa de la promesa devuelta por la siguiente invocación de `leakingLoop()`.

Podemos asegurarnos de ello simplemente eliminando la instrucción `return`:

```javascript
function nonLeakingLoop() {
  delay(1)
    .then(() => {
      console.log(`Tick ${Date.now()}`)
      nonLeakingLoop()
    })
}
```

Ahora, si usamos esta nueva función en nuestro programa de muestra, deberíamos ver que la huella de memoria del proceso subirá y bajará, siguiendo el cronograma de las diversas ejecuciones del recolector de basura, lo que significa que no hay fuga de memoria.

Sin embargo, la solución que acabamos de proponer cambia radicalmente el comportamiento de la función `leakingLoop()` original. En particular, esta nueva función no propagará eventuales errores producidos en lo profundo de la recursión, ya que no hay vínculo entre el estado de las distintas promesas. Este inconveniente puede mitigarse añadiendo algún registro (*logging*) adicional dentro de la función. Pero a veces el nuevo comportamiento en sí mismo puede no ser una opción admisible. Por lo tanto, una posible solución implica envolver la función recursiva con un constructor de Promise, como en el siguiente ejemplo de código:

```javascript
function nonLeakingLoopWithErrors() {
  return new Promise((_resolve, reject) => {
    (function internalLoop () {
      delay(1)
        .then(() => {
          console.log(`Tick ${Date.now()}`)
          internalLoop()
        })
        .catch(err => {
          reject(err)
        })
    })()
  })
}
```

En este caso, todavía no tenemos ningún vínculo entre las promesas creadas en las distintas etapas de la recursión; sin embargo, la Promise devuelta por la función `nonLeakingLoopWithErrors()` aún se rechazará si falla alguna operación asíncrona, sin importar a qué profundidad de la recursión ocurra eso.

Una tercera solución hace uso de async/await. De hecho, con async/await, podemos simular una cadena de promesas recursiva con un bucle `while` infinito simple, como el siguiente:

```javascript
async function nonLeakingLoopAsync() {
  while (true) {
    await delay(1)
    console.log(`Tick ${Date.now()}`)
  }
}
```

En esta función también preservamos el comportamiento de la función recursiva original, por lo que cualquier error lanzado por la tarea asíncrona (en este caso, `delay()`) se propaga al invocador de la función original.

Debemos tener en cuenta que todavía tendríamos una fuga de memoria si, en lugar de un bucle `while`, decidiéramos implementar la solución async/await con un paso recursivo asíncrono real, como el siguiente:

```javascript
async function leakingLoopAsync() {
  await delay(1)
  console.log(`Tick ${Date.now()}`)
  return leakingLoopAsync()
}
```

El código anterior aún crearía una cadena infinita de promesas que nunca se resuelven y, por lo tanto, todavía se ve afectado por el mismo problema de fuga de memoria de la implementación equivalente basada en promesas.

Si estás interesado en saber más sobre la fuga de memoria discutida en esta sección, puedes consultar el problema relacionado de Node.js en [nodejsdp.link/node-6673](https://nodejsdp.link/node-6673) o el problema relacionado en el repositorio de GitHub de Promises/A+ en [nodejsdp.link/promisesaplus-memleak](https://nodejsdp.link/promisesaplus-memleak).

Por lo tanto, la próxima vez que construyas una cadena de promesas infinita, recuerda verificar dos veces si existen las condiciones para crear una fuga de memoria, como aprendiste en esta sección. Si ese es el caso, puedes aplicar una de las soluciones propuestas, asegurándote de elegir la que mejor se adapte a tu contexto.

---

### Sección 4: Resumen

En este capítulo, aprendimos a utilizar las promesas y la sintaxis async/await para escribir código asíncrono más conciso, limpio y fácil de leer.

Como hemos visto, las promesas y async/await simplifican en gran medida el flujo de ejecución serial, que es el flujo de control más utilizado. De hecho, con async/await, escribir una secuencia de operaciones asíncronas es casi tan fácil como escribir código síncrono. Ejecutar algunas operaciones asíncronas concurrentemente también es muy fácil gracias a `Promise.all()` y `Promise.allSettled()`.

Pero las ventajas de usar promesas y async/await no se detienen aquí. Aprendimos que proporcionan un escudo transparente contra situaciones engañosas como el código con comportamiento mixto síncrono/asíncrono (es decir, Zalgo, que discutimos en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos). Además de eso, la gestión de errores con promesas y async/await es mucho más intuitiva y deja menos espacio para errores (como olvidar reenviar errores, que es una fuente grave de defectos en el código que usa callbacks).

En términos de patrones y técnicas, definitivamente debemos tener en cuenta la cadena de promesas (para ejecutar tareas en serie), la promisificación y el patrón Productor-Consumidor (con nuestro ejemplo de cola de tareas). Además, presta atención al usar `Array.forEach()` con async/await (probablemente lo estés haciendo mal), y ten en cuenta la diferencia entre un `return` simple y `return await` en funciones async.

Los callbacks todavía se utilizan ampliamente en el mundo de Node.js y JavaScript. Los encontramos en APIs heredadas, en código que interactúa con bibliotecas nativas o cuando existe la necesidad de microoptimizar rutinas particulares. Por eso siguen siendo relevantes para nosotros, desarrolladores de Node.js; sin embargo, para la mayoría de nuestras tareas de programación cotidianas, las promesas y async/await son un gran paso adelante en comparación con los callbacks y, por lo tanto, ahora son el estándar de facto para trabajar con código asíncrono en Node.js. Es por eso que utilizaremos promesas y async/await a lo largo del resto del libro para escribir nuestro código asíncrono.

En el próximo capítulo, exploraremos otro tema fascinante relativo a la ejecución de código asíncrono, que también es otro bloque de construcción fundamental en todo el ecosistema de Node.js: los streams.

---

### Sección 5: Ejercicios

- **5.1 Diseccionando Promise.all():** Implementa tu propia versión de `Promise.all()` aprovechando promesas, async/await o una combinación de ambos. La función debe ser funcionalmente equivalente a su contraparte original.
- **5.2 TaskQueue con promesas:** Migra los componentes internos de la clase `TaskQueue` de promesas a async/await donde sea posible. *Pista: no podrás usar async/await en todas partes.*
- **5.3 Productor-consumidor con promesas:** Actualiza los métodos internos de la clase `TaskQueuePC` para que utilicen únicamente promesas, eliminando cualquier uso de la sintaxis async/await. *Pista: el bucle infinito debe convertirse en una recursión asíncrona. ¡Cuidado con la fuga de memoria por resolución recursiva de promesas!*
- **5.4 Un map() asíncrono:** Implementa una versión asíncrona concurrente de `Array.map()` que admita promesas y un límite de concurrencia. La función no debe aprovechar directamente las clases `TaskQueue` o `TaskQueuePC` que presentamos en este capítulo, pero puede utilizar los patrones subyacentes. La función, que definiremos como `mapAsync(iterable, callback, concurrency)`, aceptará lo siguiente como entradas:
  - Un iterable, como un array.
  - Un callback, que recibirá como entrada cada elemento del iterable (exactamente como en el `Array.map()` original) y puede devolver una Promise o un valor simple.
  - Una concurrencia, que define cuántos elementos del iterable pueden ser procesados por el callback de forma concurrente en cada momento dado.
