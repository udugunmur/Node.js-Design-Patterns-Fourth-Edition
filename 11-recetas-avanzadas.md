# Parte 2: Patrones de diseño de Node.js

## Capítulo 11: Recetas avanzadas

Si alguna vez has intentado cocinar una cena elegante mientras los amigos siguen llegando temprano, ya conoces el caos de la programación asíncrona. Algunas cosas necesitan hornearse, otras necesitan enfriarse y, de alguna manera, debes mantener todo en movimiento sin quemar la cocina. A veces, trabajar con Node.js puede sentirse igual. En este capítulo, adoptamos un enfoque de problema-solución y, al igual que un libro de cocina de confianza, te ofrecemos recetas listas para usar para navegar por los desafíos comunes de Node.js.

No debería sorprenderte que la mayoría de los problemas que exploramos aquí surjan cuando intentamos realizar tareas de forma asíncrona. De hecho, como hemos visto repetidamente en los capítulos anteriores, las tareas que son triviales en la programación sincrónica tradicional pueden volverse más complicadas cuando se aplican a la programación asíncrona. Un ejemplo típico es el uso de un componente que requiere un paso de inicialización asíncrono. En tales casos, nos enfrentamos al inconveniente de tener que retrasar cualquier intento de usar el componente hasta que se complete su inicialización. Más adelante, te mostraremos cómo manejar esto elegantemente.

Sin embargo, este capítulo no trata solo de recetas relacionadas con la programación asíncrona. También aprenderás las mejores formas de ejecutar tareas intensivas de CPU en Node.js.

Para hacer esto más concreto, aquí hay un ejemplo del mundo real de Mario:

> Hace unos años, me encargaron agregar una función para exportar un registro de auditoría a PDF. Como podrás adivinar, estos registros podían crecer enormemente, a veces abarcando miles de páginas. El plan era sencillo: generar una página HTML lista para imprimir y luego usar Puppeteer ([nodejsdp.link/puppeteer](https://nodejsdp.link/puppeteer)) para convertirla automáticamente en un PDF. Supuse que, dado que Puppeteer se ejecuta en un proceso externo, incluso los archivos masivos no afectarían a la aplicación principal. Pero mis primeras pruebas demostraron lo contrario. El motor de plantillas HTML que usamos era sincrónico y, para registros muy grandes, bloqueaba el bucle de eventos de Node.js durante varios segundos. De repente, el servidor se congelaba el tiempo suficiente para ser perceptible, y no había anticipado eso.
>
> La solución fue descargar la generación de PDF a un proceso de Node.js independiente y usar un grupo (*pool*) de workers para las solicitudes de impresión. Esto mantuvo la aplicación principal receptiva independientemente del tamaño del registro. Esa experiencia fue un claro recordatorio de lo fácil que es toparse con operaciones intensivas de CPU en Node.js y por qué es tan importante saber cómo manejarlas.

Aquí están las recetas que aprenderás en este capítulo:
- **Gestión de componentes inicializados asíncronamente**
- **Procesamiento por lotes y almacenamiento en caché de solicitudes asíncronas**
- **Cancelación de operaciones asíncronas**
- **Ejecución de tareas intensivas de CPU**

¡Empecemos!

---

### Sección 1: Gestión de componentes inicializados asíncronamente

A lo largo de este libro, hemos enfatizado cuán importante es la programación asíncrona en JavaScript y Node.js, y hemos aprendido mucho sobre diferentes formas y patrones para utilizar la programación asíncrona de manera efectiva. En este punto, quizás te preguntes: ¿por qué todavía hay tantas APIs sincrónicas en Node.js, como `readFileSync()` de `node:fs`?

Una de las razones por las que existen APIs sincrónicas en los módulos centrales de Node.js y en muchos paquetes npm es que son convenientes para manejar tareas de inicialización. En programas simples, el uso de APIs sincrónicas durante la inicialización puede simplificar las cosas significativamente, y los inconvenientes típicos del código sincrónico permanecen contenidos, ya que solo afectan al programa una vez, al inicio o cuando se inicializa un componente particular.

Sin embargo, no siempre es posible confiar en APIs sincrónicas. En algunos casos, es posible que no haya una versión sincrónica disponible, especialmente para componentes que necesitan realizar operaciones de red durante su fase de inicialización, como ejecutar protocolos de enlace (*handshake*) o recuperar parámetros de configuración. Esto es común con muchos controladores de bases de datos y clientes para sistemas de middleware como colas de mensajes, donde la mayoría de las bibliotecas están diseñadas para ofrecer solo APIs asíncronas por naturaleza.

En otros casos, es posible que necesites inicializar algo dentro de una ruta de código asíncrona (por ejemplo, en un controlador de ruta de una aplicación web). En situaciones como estas, usar una API sincrónica no es ideal porque corres el riesgo de bloquear el bucle de eventos durante un período prolongado, lo que podría hacer que todo el servidor web no responda durante la operación sincrónica. ¡Lejos de ser ideal!

#### El problema con los componentes inicializados asíncronamente

Veamos un ejemplo que involucra un módulo llamado `db`, que se utiliza para interactuar con una base de datos remota. El módulo `db` solo puede aceptar solicitudes después de haber completado la conexión y el protocolo de enlace con el servidor de base de datos. Hasta que finalice esta fase de inicialización, no se pueden enviar consultas ni otros comandos.

Ten en cuenta que, para simplificar las cosas, este módulo `db` no implementa un conector de base de datos real, sino que imita fielmente la estructura general y el comportamiento de uno realista. Aquí está el código de este módulo de muestra (`db.js`):

```javascript
// db.js
import { setTimeout } from 'node:timers/promises'

class Database {
  connected = false
  #pendingConnection = null

  async connect() {
    if (!this.connected) {
      if (this.#pendingConnection) {
        return this.#pendingConnection
      }
      // simulate the delay of the connection
      this.#pendingConnection = setTimeout(500)
      await this.#pendingConnection
      this.connected = true
      this.#pendingConnection = null
    }
  }

  async query(queryString) {
    if (!this.connected) {
      throw new Error('Not connected yet')
    }
    // simulate the delay of the query execution
    await setTimeout(100)
    console.log(`Query executed: ${queryString}`)
  }
}

export const db = new Database()
```

Este código define una clase simple `Database` que simula el comportamiento de un cliente de base de datos. El método `connect()` simula la creación de una conexión a la base de datos. Comprueba si es necesario establecer una conexión y luego imita un retraso de conexión esperando 500 milisegundos. Una vez establecida la conexión, establece el indicador `connected` en `true`.

Observa que, aquí, el miembro privado `#pendingConnection` contiene una promesa que representa un intento de conexión en curso. Antes de iniciar una nueva conexión, comprobamos si esta promesa ya existe. Si existe, significa que hay una conexión en curso, por lo que en lugar de crear una nueva promesa (e iniciar otra conexión), simplemente reutilizamos la existente. Esto garantiza que múltiples llamadas a `connect()` antes de que se establezca la primera conexión se monten en la misma promesa (*promise piggybacking*), evitando conexiones duplicadas. Esta técnica a menudo se denomina **promise piggybacking** y constituye la base de otro patrón útil llamado **procesamiento por lotes de solicitudes** (*request batching*). Exploraremos ambos con más detalle más adelante en este capítulo.

El método `query()` simula el envío de una consulta a la base de datos. Antes de ejecutar la consulta, comprueba si la conexión se ha establecido. Si no es así, lanza un error para indicar que la base de datos aún no está lista. Si la conexión está lista, simula un breve retraso (100 milisegundos) y luego registra la cadena de consulta en la consola. Al final del archivo, se crea y exporta una instancia de esta clase `Database` como `db`, lo que la deja lista para su uso en otras partes de la aplicación (como un singleton).

Este es un ejemplo típico de un componente que requiere inicialización asíncrona. Sin embargo, la implementación actual es propensa a un mal uso porque el código de la aplicación podría intentar ejecutar una consulta antes de que el cliente haya terminado de conectarse.

Hay algunas formas en que esto puede suceder:
- Un usuario puede olvidar llamar al método `connect()` antes de ejecutar una consulta.
- Un usuario puede llamar a `connect()` pero olvidar usar `await` en la llamada, sin esperar de forma efectiva a que se establezca la conexión.

Así es como podrían verse estos errores en el código:

Olvidar llamar a `connect()`:

```javascript
import { db } from './db.js'

const users = await db.query('SELECT * FROM users')
```

Olvidar usar `await` en la llamada a `connect()`:

```javascript
import { db } from './db.js'

db.connect() // no await before this call!
const users = await db.query('SELECT * FROM users')
```

En ambos casos, la ejecución del código resultará en el lanzamiento de un error (`Not connected yet`).

Por lo general, tenemos dos soluciones rápidas y sencillas para situaciones como estas, que podemos llamar **comprobación de inicialización local** e **inicio retrasado**. Analicémoslas con más detalle.

#### Comprobación de inicialización local

La primera solución, quizás la más sencilla, es asegurarse de que el módulo esté inicializado antes de invocar cualquiera de sus APIs. Si no es así, esperamos a su inicialización antes de continuar. Esta comprobación debe realizarse cada vez que queramos llamar a una operación en el módulo asíncrono.

Así es como se ve en el código:

```javascript
import { db } from './db.js'

async function getUsers() {
  if (!db.connected) {
    await db.connect()
  }

  await db.query('SELECT * FROM users')
}
```

Como puedes ver, antes de ejecutar una consulta, comprobamos si la instancia `db` ya está conectada. Si no es así, esperamos explícitamente a que se complete la conexión llamando y esperando a `connect()`.

Podríamos haber evitado agregar la condición `if` aquí, ya que nuestro cliente de base de datos ya comprueba internamente si está conectado antes de realizar una conexión. Agregar la comprobación en este punto puede verse como una pequeña optimización de rendimiento: si el cliente ya está conectado, podemos omitir llamar y esperar una operación asíncrona, evitando así un aplazamiento innecesario de la ejecución de la función a un ciclo posterior del bucle de eventos.

En el ejemplo anterior, la responsabilidad de garantizar que la conexión esté lista recae enteramente en el consumidor del módulo `db`.

Una variación común de esta técnica es mover la comprobación de conexión dentro del propio método `query()`. Esto traslada la carga del consumidor al proveedor: el módulo se encargaría de asegurarse de que esté inicializado antes de ejecutar cualquier consulta, ahorrando a los usuarios escribir código repetitivo de comprobación de conexión cada vez que interactúan con la base de datos.

#### Inicio retrasado

La segunda solución rápida (y algo sucia) al problema de los componentes inicializados asíncronamente es retrasar la ejecución de cualquier código que dependa del componente hasta después de que se haya completado su rutina de inicialización.

Si pensamos en nuestro ejemplo anterior de cliente de base de datos, una forma sencilla de implementar esta idea es crear una función de fábrica que devuelva un cliente de base de datos que ya esté conectado. Al hacer esto, nos aseguramos de obtener un cliente conectado una vez y luego podemos usarlo de forma segura sin tener que preocuparnos por administrar el estado de la conexión en el resto de nuestro código.

Aquí tienes un ejemplo:

```javascript
import { db } from './db.js'

async function getConnectedDb() {
  await db.connect()
  return db
}

async function getUsers(db) {
  await db.query('SELECT * FROM users')
}

const connectedDb = await getConnectedDb()
await getUsers(connectedDb)
```

Como puedes ver, introdujimos una función de fábrica llamada `getConnectedDb()`, que asegura que el cliente de la base de datos esté conectado antes de devolverlo. También actualizamos ligeramente la función `getUsers()` de nuestro ejemplo anterior: esta vez, toma una instancia de base de datos (`db`) como argumento.

Con esta configuración, primero podemos obtener una instancia de base de datos conectada (`connectedDb`) y luego usarla de forma segura en el resto de la aplicación. Idealmente, la función `getConnectedDb()` podría incluso trasladarse al módulo `db.js`, ya que actúa como una utilidad común para trabajar con la base de datos.

La principal desventaja de esta técnica es que nos obliga a saber, con anticipación, qué componentes o funciones dependerán del recurso inicializado asíncronamente. Esto hace que el código sea más frágil y más propenso a errores. Este inconveniente se vuelve más tangible cuando nuestra aplicación tiene una cadena más compleja de dependencias. En tales casos, debemos asegurarnos en cada paso de que el cliente de la base de datos esté conectado, en lugar de poder asumir que ya tenemos un cliente conectado disponible como dependencia.

Por ejemplo, supongamos que nuestra aplicación envía un correo electrónico de bienvenida cuando se registra un nuevo usuario, y esa función consulta internamente la base de datos para recuperar la dirección de correo electrónico y el nombre del usuario. Si olvidamos pasar el cliente de base de datos ya conectado a la función de envío de correo electrónico, y esa función intenta usar el cliente directamente sin asegurarse de que se haya establecido la conexión, fallará. Esto sucede porque no previmos que esta nueva función también dependía de que la base de datos estuviera conectada.

Una forma de abordar esto es retrasar el inicio de toda la aplicación hasta que todos los servicios asíncronos estén inicializados. Este enfoque es simple y efectivo, pero puede agregar un retraso significativo al tiempo de inicio de la aplicación. Además, no tiene en cuenta los casos en los que un componente inicializado asíncronamente necesita reinicializarse después del inicio inicial (imagina el caso en que nuestro cliente de base de datos pierde la conexión con la base de datos y es necesario crear una nueva conexión). La inicialización previa de todos los recursos también asume que todos estos recursos siempre se utilizarán, lo que a menudo no es cierto en grandes aplicaciones web con muchas características, diferentes rutas de código o aplicaciones CLI con múltiples subcomandos. Dependiendo de cómo interactúen los usuarios con la aplicación, es posible que nunca activen ciertas rutas de código, lo que significa que habrás pagado el costo de inicializar recursos que finalmente no se utilizan.

Como veremos en la siguiente sección, existe una tercera alternativa que nos permite retrasar de forma transparente y eficiente las operaciones hasta que se haya completado el paso de inicialización asíncrono.

#### Colas de pre-inicialización

Otro enfoque para garantizar que los servicios de un componente se invoquen solo después de que esté completamente inicializado es combinando una cola simple con el patrón **Command**.

La idea es encolar las invocaciones de métodos (específicamente, aquellas que requieren que el componente esté inicializado) mientras el componente aún no está listo y luego ejecutarlas automáticamente una vez que se complete la inicialización.

Apliquemos esta técnica a nuestro componente `db` de muestra:

```javascript
// db.js
import { setTimeout } from 'node:timers/promises'

class Database {
  connected = false
  #pendingConnection = null
  commandsQueue = []

  async connect() {
    if (!this.connected) {
      if (this.#pendingConnection) {
        return this.#pendingConnection
      }
      // simulate the delay of the connection
      this.#pendingConnection = setTimeout(500)
      await this.#pendingConnection
      this.connected = true
      this.#pendingConnection = null
      // once connected executes all the queued commands
      while (this.commandsQueue.length > 0) { // 2
        const command = this.commandsQueue.shift()
        command()
      }
    }
  }

  async query(queryString) {
    if (!this.connected) { // 1
      console.log(`Request queued: ${queryString}`)
      return new Promise((resolve, reject) => {
        const command = () => {
          this.query(queryString).then(resolve, reject)
        }
        this.commandsQueue.push(command)
      })
    }

    // simulate the delay of the query execution
    await setTimeout(100)
    console.log(`Query executed: ${queryString}`)
  }
}

export const db = new Database()
```

La técnica aquí consta de dos partes principales:
1. Si el componente aún no está inicializado (en nuestro caso, cuando `connected` es `false`), creamos un comando que captura los parámetros de la invocación actual y lo agregamos a `commandsQueue`. Cuando el comando se ejecuta posteriormente, vuelve a invocar el método `query()` y reenvía el resultado al llamador original a través de una promesa. Observa cómo el método `query()` implementa efectivamente dos rutas de código distintas: una para cuando el componente aún no está inicializado (donde la consulta se encola) y otra para cuando está inicializado (donde la consulta se ejecuta). El retorno temprano dentro de la sentencia `if` separa limpiamente estos dos comportamientos.
2. Una vez completada la inicialización (después de que se establece la conexión con la base de datos), procesamos `commandsQueue` ejecutando todos los comandos encolados en orden.

Con este diseño, los consumidores del componente `db` ya no necesitan comprobar si está inicializado. Toda la lógica de encolado y ejecución diferida se maneja internamente, y la instancia `db` se puede usar de forma transparente como si siempre estuviera lista. Veamos un ejemplo de uso:

```javascript
// index.js
import { setTimeout } from 'node:timers/promises'
import { db } from './db.js'

db.connect()

async function updateLastAccess() {
  await db.query(`INSERT (${Date.now()}) INTO "LastAccesses"`)
}

updateLastAccess()
await setTimeout(600)
updateLastAccess()
```

En este ejemplo, estamos simulando la inserción de una marca de tiempo que representa el último acceso a una página determinada (imagínalo como una tabla de análisis web simple). Lo único que tenemos que hacer es llamar a `db.connect()`. No tenemos que esperar a que se establezca la conexión antes de emitir consultas a la base de datos. Dependiendo del estado del cliente (si la conexión está establecida o no), la consulta se encolará o se ejecutará de inmediato.

En este caso, estamos ejecutando dos consultas: la primera se emite de inmediato (y se encolará), y la segunda se emite después de 600 milisegundos (que se ejecutará de inmediato, ya que nuestro retraso de conexión simulado es de 500 milisegundos y, para ese momento, el cliente ya estará conectado).

Ten en cuenta que con esta estrategia, si olvidamos llamar a `connect()`, cualquier consulta que ejecutemos devolverá promesas que nunca se resolverán, a menos que `connect()` se llame eventualmente más tarde.

La ejecución de este código debería producir la siguiente salida:

```
Request queued: INSERT (1745693136392) INTO "LastAccesses"
<pause for ~600ms>
Query executed: INSERT (1745693136392) INTO "LastAccesses"
Query executed: INSERT (1745693136998) INTO "LastAccesses"
```

#### Uso del patrón State

Podemos mejorar la modularidad y reducir el código repetitivo de la clase `Database` aún más aplicando el patrón **State**, que exploramos en el [Capítulo 9](https://subscription.packtpub.com/book/web-development/9781803238944/9), Patrones de diseño de comportamiento.

En este enfoque, el comportamiento del componente depende de su estado interno, cambiando automáticamente entre dos estados:
- **Initialized state (Estado inicializado):** Este estado maneja todas las operaciones asumiendo que el componente está listo. Cada método implementa directamente su lógica de negocio sin preocuparse por la inicialización.
- **Queuing state (Estado de encolado):** Este estado está activo antes de que se complete la inicialización. Implementa los mismos métodos, pero su único propósito es encolar las operaciones solicitadas para su posterior ejecución.

Primero implementemos `InitializedState`, que contiene la lógica de negocio para el estado listo:

```javascript
class InitializedState {
  constructor(db) {
    this.db = db
  }

  async query(queryString) {
    // simulate the delay of the query execution
    await setTimeout(100)
    console.log(`Query executed: ${queryString}`)
  }
}
```

Como puedes ver, `query()` simplemente simula la ejecución de una consulta introduciendo un pequeño retraso y luego registrando la consulta ejecutada.

A continuación, implementamos `QueuingState`, que maneja las solicitudes recibidas mientras la base de datos aún no está conectada:

```javascript
const deactivate = Symbol('deactivate')

class QueuingState {
  constructor(db) {
    this.db = db
    this.commandsQueue = []
  }

  async query(queryString) { // 1
    console.log(`Request queued: ${queryString}`)
    return new Promise((resolve, reject) => {
      const command = () => {
        this.db.query(queryString).then(resolve, reject)
      }
      this.commandsQueue.push(command)
    })
  }

  [deactivate]() { // 2
    while (this.commandsQueue.length > 0) {
      const command = this.commandsQueue.shift()
      command()
    }
  }
}
```

Hay algunos detalles importantes a tener en cuenta aquí:
1. Cuando se llama a `query()` antes de que se establezca la conexión, encola un comando en lugar de intentar ejecutarse de inmediato.
2. El método `[deactivate]` (nombrado con un `Symbol` para evitar futuros conflictos de nombres y mantener la función privada) vacía y ejecuta todos los comandos encolados una vez que la base de datos está lista.

Ahora, reimplementemos la clase `Database` para hacer uso de estos dos estados:

```javascript
// db.js
import { setTimeout } from 'node:timers/promises'

const deactivate = Symbol('deactivate')

class InitializedState {
  constructor(db) {
    this.db = db
  }

  async query(queryString) {
    // simulate the delay of the query execution
    await setTimeout(100)
    console.log(`Query executed: ${queryString}`)
  }
}

class QueuingState {
  constructor(db) {
    this.db = db
    this.commandsQueue = []
  }

  async query(queryString) {
    console.log(`Request queued: ${queryString}`)
    return new Promise((resolve, reject) => {
      const command = () => {
        this.db.query(queryString).then(resolve, reject)
      }
      this.commandsQueue.push(command)
    })
  }

  [deactivate]() {
    while (this.commandsQueue.length > 0) {
      const command = this.commandsQueue.shift()
      command()
    }
  }
}

class Database {
  connected = false
  #pendingConnection = null

  constructor() {
    this.state = new QueuingState(this) // 1
  }

  async query(queryString) {
    return this.state.query(queryString) // 2
  }

  async connect() {
    if (!this.connected) {
      if (this.#pendingConnection) {
        return this.#pendingConnection
      }
      // simulate the delay of the connection
      this.#pendingConnection = setTimeout(500)
      await this.#pendingConnection
      this.connected = true
      this.#pendingConnection = null
      // once connected update the state
      const oldState = this.state // 3
      this.state = new InitializedState(this)
      oldState[deactivate]?.()
    }
  }
}

export const db = new Database()
```

Destaquemos los aspectos clave de esta nueva implementación:
1. En el constructor, inicializamos el componente en `QueuingState` porque la inicialización asíncrona (la conexión a la base de datos) aún no se ha completado.
2. El método `query()` simplemente delega la llamada al método `query()` del estado actual, sin comprobaciones adicionales.
3. Una vez establecida la conexión, cambiamos el estado interno de `QueuingState` a `InitializedState` y desactivamos el estado anterior para ejecutar cualquier operación encolada.

Al aplicar el patrón State aquí, reducimos significativamente el código repetitivo, encapsulamos las preocupaciones de inicialización limpiamente dentro de los estados y permitimos que la lógica de negocio (en `InitializedState`) permanezca libre de comprobaciones de conexión de bajo nivel.

Este enfoque, sin embargo, solo funciona cuando tienes control sobre la implementación del componente. Si no puedes modificar el componente directamente, necesitarías crear un wrapper externo o proxy para lograr un comportamiento similar, pero los principios y la estructura seguirían siendo prácticamente los mismos.

#### En el mundo real

El patrón que acabamos de presentar es utilizado por muchos controladores de bases de datos y bibliotecas ORM. El más notable es **Mongoose** ([nodejsdp.link/mongoose](https://nodejsdp.link/mongoose)), que es un ORM para MongoDB. Con Mongoose, no es necesario esperar a que se abra la conexión de la base de datos para poder enviar consultas. Esto se debe a que cada operación se encola y luego se ejecuta más tarde cuando la conexión con la base de datos está completamente establecida, exactamente como hemos descrito en esta sección. Claramente, esto es imprescindible para cualquier API que quiera proporcionar una buena experiencia de desarrollo (DX).

Mira el código de Mongoose para ver cómo cada método en el controlador nativo tiene un proxy para agregar la cola de pre-inicialización. Esto también demuestra una forma alternativa de implementar la receta que presentamos en esta sección. Puedes encontrar el fragmento de código relevante en [nodejsdp.link/mongoose-init-queue](https://nodejsdp.link/mongoose-init-queue).

De manera similar, el paquete `pg` ([nodejsdp.link/pg](https://nodejsdp.link/pg)), que es un cliente para la base de datos PostgreSQL, aprovecha las colas de pre-inicialización, pero de una manera ligeramente diferente. `pg` encola cada consulta, independientemente del estado de inicialización de la base de datos, y luego intenta ejecutar inmediatamente todos los comandos en la cola. La línea de código relevante se puede leer aquí: [nodejsdp.link/pg-queue](https://nodejsdp.link/pg-queue).

---

### Sección 2: Procesamiento por lotes y almacenamiento en caché de solicitudes asíncronas

En aplicaciones de alta carga, el almacenamiento en caché es esencial. Impulsa gran parte de la web moderna, ayudando a servir desde activos estáticos como páginas web, imágenes y hojas de estilo hasta datos dinámicos como resultados de consultas de bases de datos. En esta sección, nos sumergiremos en cómo se aplican las estrategias de almacenamiento en caché a las operaciones asíncronas y cómo, con las técnicas adecuadas, un aumento repentino de solicitudes puede convertirse en una oportunidad en lugar de un problema. Exploraremos patrones poderosos como el **procesamiento por lotes de solicitudes asíncronas** (*asynchronous request batching*), que pueden ayudarte a optimizar el rendimiento y el uso de recursos aún más.

#### ¿Qué es el procesamiento por lotes de solicitudes asíncronas?

Cuando se trata de operaciones asíncronas, el nivel más básico de almacenamiento en caché se puede lograr agrupando por lotes un conjunto de invocaciones a la misma API. La idea es muy simple: si invocamos una función asíncrona mientras todavía hay otra pendiente, podemos montarnos en la operación que ya se está ejecutando en lugar de crear una solicitud completamente nueva.

Podemos apreciar esta idea con una analogía de la vida real. En una fábrica con línea de montaje (por ejemplo, una planta de automóviles), el personal de producción, mantenimiento y compras a menudo necesita la misma información: si hay una pieza de repuesto disponible y cuántas unidades hay en stock. La Persona 1 pregunta en el mostrador de piezas y el empleado llama al almacén cercano para que un colega pueda revisar el estante y leer el recuento actual. Momentos después, la Persona 2 de otro departamento pregunta por esa misma pieza. En lugar de hacer una segunda llamada al almacén (lo que desencadenaría otra caminata hacia los estantes), el empleado responde: *"Un colega ya está comprobando. Espera un momento y compartiremos el resultado"*. El colega completa la comprobación una vez, regresa con el número y ambas personas usan esa lectura única. En resumen: un viaje en progreso puede atender múltiples solicitudes que llegan durante el mismo; esto mantiene las cosas eficientes y evita el trabajo duplicado innecesario.

El procesamiento por lotes de solicitudes en un servidor funciona de manera similar: si llegan múltiples solicitudes para los mismos datos mientras la primera aún se está procesando, todas pueden "montarse" (*piggyback*) en la misma operación en lugar de iniciar un trabajo nuevo e idéntico.

Echemos un vistazo al siguiente diagrama:

```
Cliente 1 ---> [ Iniciar Operación A ] ------------> [ Resultado A ] ---> Cliente 1
Cliente 2 ---------> [ Iniciar Operación B ] ----------> [ Resultado B ] ---> Cliente 2
```

*Figura 11.1 – Dos solicitudes asíncronas sin procesamiento por lotes.*

El diagrama muestra dos clientes que invocan la misma operación asíncrona con exactamente la misma entrada. Naturalmente, sin ningún mecanismo de procesamiento por lotes implementado, cada cliente inicia su propia operación por separado, lo que lleva a dos procesos asíncronos independientes que se completan en momentos diferentes.

Si la operación es particularmente costosa (por ejemplo, realizar una consulta pesada a la base de datos o renderizar una página compleja), terminamos pagando el costo total dos veces, aunque el resultado sea idéntico para ambas solicitudes. ¿No parece un desperdicio? ¿Y qué pasa si el sistema está bajo una carga alta y, en lugar de dos solicitudes concurrentes, tenemos docenas o incluso cientos? ¿Empiezas a ver la oportunidad de optimización aquí?

Ahora, considera el siguiente escenario:

```
Cliente 1 ---> [ Iniciar Operación A ] ------------> [ Resultado A ] ---> Cliente 1
Cliente 2 ---------> (Se une a Operación A) --------^                 ---> Cliente 2
```

*Figura 11.2 – Procesamiento por lotes de dos solicitudes asíncronas.*

La Figura 11.2 muestra cómo se pueden agrupar dos solicitudes idénticas (que invocan la misma API con la misma entrada), lo que significa que ambas se adjuntan a la misma operación en ejecución.

Al hacer esto, cuando se completa la operación asíncrona, se notifica a ambos clientes, aunque la operación en sí se ejecutó solo una vez. Esto representa una forma simple, pero extremadamente poderosa, de optimizar la carga en una aplicación sin introducir la complejidad de los mecanismos de almacenamiento en caché tradicionales, que generalmente requieren una gestión cuidadosa de la memoria y estrategias de invalidación.

#### Almacenamiento en caché óptimo de solicitudes asíncronas

El procesamiento por lotes de solicitudes se vuelve menos efectivo cuando las operaciones son lo suficientemente rápidas o cuando las solicitudes coincidentes se distribuyen durante un período más prolongado.

Además, en muchos casos, podemos asumir con seguridad que el resultado de dos invocaciones idénticas a la API no cambiará con frecuencia. En tales situaciones, el simple procesamiento por lotes de solicitudes por sí solo no ofrecerá el mejor rendimiento.

Cuando esto sucede, un mecanismo de almacenamiento en caché más agresivo se convierte en el mejor candidato para reducir la carga en una aplicación y mejorar su capacidad de respuesta.

La idea es sencilla: tan pronto como se completa una solicitud, almacenamos su resultado en una memoria caché, que podría ser una variable en memoria o una entrada en un sistema de almacenamiento en caché dedicado como Redis. Luego, la próxima vez que se invoque la API con la misma entrada, el resultado se puede recuperar inmediatamente de la memoria caché, evitando la necesidad de realizar la operación nuevamente.

El concepto de almacenamiento en caché ya debería ser familiar para la mayoría de los desarrolladores experimentados. Sin embargo, lo que hace que el almacenamiento en caché sea particularmente interesante en el contexto de la programación asíncrona es que se puede combinar con el procesamiento por lotes de solicitudes para ser verdaderamente óptimo.

La razón es simple: mientras la caché aún no está poblada, pueden llegar múltiples solicitudes concurrentes para la misma entrada. Sin procesamiento por lotes, cada una de estas solicitudes desencadenaría la misma operación de forma independiente y, una vez completadas, todas intentarían establecer la caché por separado, lo que generaría un trabajo duplicado innecesario.

Aquí es donde entra el procesamiento por lotes de solicitudes. Veamos eso en acción en la siguiente figura:

```
Fase 1 (Caché vacía - Batching):
Cliente A ---> [ Iniciar Operación ] ------> [ Guardar en Caché ] ---> Cliente A
Cliente B ---------> (Se une a Operación) --^                      ---> Cliente B

Fase 2 (Caché lista - Cache Hit):
Cliente C -----------------------------> [ Leer de Caché ] -----------> Cliente C
```

*Figura 11.3 – Procesamiento por lotes y almacenamiento en caché combinados.*

La figura anterior muestra las dos fases de un algoritmo óptimo de almacenamiento en caché asíncrono:
- La primera fase es idéntica al patrón de procesamiento por lotes. Cualquier solicitud recibida mientras no se haya establecido la caché se agrupará (este es el caso del Cliente A y el Cliente B en la figura). Cuando se completa la solicitud, la caché se establece una sola vez.
- Cuando la caché finalmente se establece, cualquier solicitud posterior se servirá directamente desde la caché. En la figura, este es el caso del Cliente C.

Otro detalle crucial a recordar es el antipatrón de **Zalgo** (como se discutió en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos). Dado que estamos tratando con APIs asíncronas, debemos asegurarnos de devolver siempre el valor en caché de forma asíncrona, incluso si el acceso a la caché implica solo una operación sincrónica, como en el caso en que el valor en caché se recupera de una variable en memoria.

#### Un servidor de API sin caché ni procesamiento por lotes

Antes de comenzar a sumergirnos en este nuevo desafío, implementemos un pequeño servidor de demostración que usaremos como referencia para medir el impacto de las diversas técnicas que vamos a implementar.

Consideremos un servidor de API que administra las ventas de una empresa de comercio electrónico. En particular, queremos consultar a nuestro servidor la suma de todas las transacciones de un tipo particular de mercancía. Para este propósito, vamos a utilizar una base de datos Level a través del paquete npm `level` ([nodejsdp.link/level](https://nodejsdp.link/level)). El modelo de datos que vamos a utilizar es una lista simple de transacciones almacenadas en la base de datos de ventas, que está organizada en el siguiente formato:

```
transactionId {amount, product}
```

La clave está representada por `transactionId`, y el valor es un objeto JSON que contiene el importe de la venta (`amount`) y el tipo de producto (`product`).

Los datos a procesar son bastante básicos, así que implementemos una consulta simple sobre la base de datos que podamos usar para nuestros experimentos. Digamos que queremos obtener el importe total de ventas de un producto en particular. La rutina se vería de la siguiente manera (archivo `totalSales.js`):

```javascript
// totalSales.js
import { Level } from 'level'

const db = new Level('sales', { valueEncoding: 'json' })

export async function totalSales(product) {
  const now = Date.now()
  let sum = 0
  for await (const [_transactionId, transaction] of db.iterator()) {
    if (!product || transaction.product === product) {
      sum += transaction.amount
    }
  }
  console.log(`totalSales() took: ${Date.now() - now}ms`)
  return sum
}
```

La función `totalSales()` itera sobre todas las transacciones de la base de datos de ventas y calcula la suma de los importes de un producto en particular. El algoritmo es intencionalmente lento, ya que queremos resaltar el efecto del procesamiento por lotes y el almacenamiento en caché más adelante.

En una aplicación del mundo real, se debe evitar escanear una tabla de base de datos completa. En su lugar, deberías considerar usar un índice (por ejemplo, para filtrar por producto) o, mejor aún, implementar una canalización basada en eventos que calcule continuamente el total de cada producto en una tabla dedicada a medida que se registran nuevas transacciones.

Ahora podemos exponer la API `totalSales()` a través de un servidor HTTP simple (el archivo `server.js`):

```javascript
// server.js
import { createServer } from 'node:http'
import { totalSales } from './totalSales.js'

createServer(async (req, res) => {
  const url = new URL(req.url, 'http://localhost')
  const product = url.searchParams.get('product')
  console.log(`Processing query: ${url.search}`)

  const sum = await totalSales(product)

  res.setHeader('Content-Type', 'application/json')
  res.writeHead(200)
  res.end(
    JSON.stringify({
      product,
      sum,
    })
  )
}).listen(8000, () => console.log('Server started'))
```

Para mantener la configuración ligera y centrada en la lógica esencial, elegimos utilizar el módulo integrado `node:http` de Node.js en lugar de un marco web completo como Fastify o Express.

Cuando un cliente envía una solicitud, el servidor extrae el parámetro de consulta `product` de la URL y registra la consulta entrante en la consola. Luego llama a la función `totalSales()`, pasando el producto solicitado, y espera el resultado. Una vez calculada la suma, el servidor responde con un objeto JSON que contiene tanto el nombre del producto como el importe total de las ventas.

Antes de iniciar el servidor por primera vez, necesitamos llenar la base de datos con algunos datos de muestra. Podemos hacer esto usando el script `populateDb.js`, que puedes encontrar en el repositorio de código del libro bajo la carpeta dedicada a esta sección ([nodejsdp.link/batch-cache](https://nodejsdp.link/batch-cache)). Este script genera 100.000 transacciones de ventas aleatorias en la base de datos, asegurando que nuestras consultas tarden algún tiempo en procesar los datos. Puedes llenar la base de datos ejecutando esto:

```bash
node populateDb.js
```

Una vez que los datos estén en su lugar, estamos listos para iniciar el servidor:

```bash
node server.js
```

Para consultar el servidor, simplemente abre una ventana del navegador y navega a:

```
http://localhost:8000?product=book
```

Deberías ver una salida como esta:

```json
{"product":"book","sum":1008193}
```

Ten en cuenta que tu suma real puede ser diferente, ya que el conjunto de datos se genera aleatoriamente.

Esto confirma que nuestra implementación funciona. Pero, ¿qué tan rápida es?

Para comprender mejor el rendimiento de nuestro servidor, necesitamos más que una sola solicitud. Para esto, utilizaremos **Autocannon** ([nodejsdp.link/autocannon](https://nodejsdp.link/autocannon)), una herramienta de evaluación comparativa (*benchmarking*) HTTP escrita en Node.js, que podemos instalar desde npm:

```bash
npm i -g autocannon
```

Una vez instalado Autocannon, podemos ejecutar una prueba de rendimiento ejecutando esto:

```bash
autocannon 'http://localhost:8000?product=book'
```

Por defecto, Autocannon crea 10 conexiones concurrentes, cada una de las cuales envía solicitudes continuamente, una tras otra, durante unos 10 segundos.

Cuando se completa el benchmark, Autocannon imprime un informe detallado que contiene métricas de rendimiento clave para el servidor bajo carga. Estas incluyen estadísticas de latencia (cuánto tiempo lleva cada solicitud), tasa de solicitudes (solicitudes por segundo) y rendimiento de datos (bytes por segundo). También proporciona estadísticas de distribución como promedios, percentiles y valores máximos, ofreciendo una imagen completa de cómo se comporta el servidor bajo acceso concurrente.

En mi máquina, esta prueba resultó en 150 solicitudes durante 10 segundos, con un promedio de alrededor de 15 solicitudes por segundo.

Esto nos da una buena línea de base para comenzar. A medida que experimentemos con diferentes optimizaciones, como el procesamiento por lotes y el almacenamiento en caché de solicitudes, podemos ejecutar el benchmark nuevamente para ver si realmente estamos mejorando el rendimiento y en qué medida.

Las pruebas comparativas son complejas porque los resultados varían según el hardware, la red y la carga. Ejecuta tus propias pruebas y compara enfoques en condiciones similares en lugar de perseguir números absolutos. El objetivo es mostrar que técnicas como el procesamiento por lotes y el almacenamiento en caché pueden mejorar significativamente el rendimiento, y las ganancias exactas dependerán de tu servidor, los patrones de tráfico y la duración de la caché.

Ahora, vamos a aplicar nuestras optimizaciones y medir cuánto tiempo podemos ahorrar. Comenzaremos implementando tanto el procesamiento por lotes como el almacenamiento en caché aprovechando las propiedades de las promesas.

#### Procesamiento por lotes y almacenamiento en caché con promesas

Las promesas son una gran herramienta para implementar el procesamiento por lotes y el almacenamiento en caché asíncronos de solicitudes. Veamos por qué.

Si recordamos lo que aprendimos sobre las promesas en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con promesas y async/await, hay dos propiedades que pueden ser explotadas a nuestro favor en esta circunstancia:
- Se pueden adjuntar múltiples escuchadores `then()` a la misma promesa.
- Se garantiza que el escuchador `then()` se invocará (solo una vez), y funciona incluso si se adjunta después de que la promesa ya se haya resuelto. Además, se garantiza que `then()` siempre se invoque de forma asíncrona.

En resumen, la primera propiedad es exactamente lo que necesitamos para procesar solicitudes por lotes, mientras que la segunda significa que una promesa ya es una memoria caché para el valor resuelto y ofrece un mecanismo natural para devolver un valor almacenado en caché de una manera consistente y asíncrona. En otras palabras, esto significa que el procesamiento por lotes y el almacenamiento en caché se vuelven extremadamente simples y concisos con promesas.

##### Procesamiento por lotes de solicitudes

Agreguemos ahora una capa de procesamiento por lotes sobre nuestra API `totalSales`.

El patrón que usaremos es muy sencillo: si ya hay una solicitud idéntica en curso cuando se invoca la API, simplemente esperaremos a que se complete esa solicitud en lugar de iniciar una nueva. Como veremos, esto se puede implementar fácilmente con promesas.

La idea es simple: cada vez que lanzamos una nueva solicitud, guardamos la promesa correspondiente en un mapa, asociándola con los parámetros de la solicitud (en nuestro caso, el tipo de producto). Luego, para cada solicitud posterior, primero verificamos si ya existe una promesa para ese producto. Si existe, devolvemos la promesa existente; si no, creamos y almacenamos una nueva.

Ahora, veamos cómo se traduce esto en código.

Crearemos un nuevo módulo llamado `totalSalesBatch.js`, donde implementamos una capa de procesamiento por lotes sobre la API `totalSales()` original:

```javascript
// totalSalesBatch.js
import { totalSales as totalSalesRaw } from './totalSales.js'

const runningRequests = new Map()

export function totalSales(product) {
  if (runningRequests.has(product)) { // 1
    console.log('Batching')
    return runningRequests.get(product)
  }

  const resultPromise = totalSalesRaw(product) // 2
  runningRequests.set(product, resultPromise)
  resultPromise.finally(() => { // 3
    runningRequests.delete(product)
  })

  return resultPromise
}
```

La función `totalSales()` del módulo `totalSalesBatch` es efectivamente un proxy para la API original `totalSales()`, y funciona de la siguiente manera:
1. Si ya existe una promesa para el producto dado, simplemente la devolvemos. Aquí es donde nos montamos (*piggyback*) en una solicitud que ya se está ejecutando.
2. Si no hay ninguna solicitud en ejecución para el producto dado, ejecutamos la función original `totalSales()` y guardamos la promesa resultante en el mapa `runningRequests`.
3. A continuación, nos aseguramos de eliminar la misma promesa del mapa `runningRequests` tan pronto como se complete la solicitud.

El comportamiento de la nueva función `totalSales()` es idéntico al de la API original `totalSales()`, con la diferencia de que, ahora, múltiples llamadas a la API que usan la misma entrada se agrupan por lotes, lo que potencialmente nos ahorra tiempo y recursos.

¿Tienes curiosidad por saber cuál es la mejora de rendimiento en comparación con la versión sin procesar y sin lotes de la API `totalSales()`? Reemplacemos entonces el módulo `totalSales` utilizado por el servidor HTTP por el que acabamos de crear (el archivo `server.js`):

```javascript
// import { totalSales } from './totalSales.js'
import { totalSales } from './totalSalesBatch.js'

createServer(async (req, res) => {
  // ...
```

Ahora podemos reiniciar el servidor y volver a ejecutar nuestro benchmark de Autocannon.

En mi máquina, el servidor ahora puede manejar alrededor de 1.000 solicitudes en 10 segundos, con un promedio de alrededor de 100 solicitudes por segundo.

Esto es aproximadamente **6,5 veces más solicitudes por segundo** en comparación con nuestra implementación anterior utilizando la API `totalSales()` directamente sin procesamiento por lotes de solicitudes. ¡Nada mal!

Este resultado demuestra claramente el aumento significativo del rendimiento que podemos lograr simplemente agregando una capa de procesamiento por lotes, sin la complejidad de administrar una memoria caché completa o los desafíos de lidiar con estrategias de invalidación de caché.

El patrón Request Batching alcanza su mejor potencial en aplicaciones de alta carga y con APIs lentas. Esto se debe a que es exactamente en estas circunstancias donde podemos agrupar una gran cantidad de solicitudes.

Veamos ahora cómo podemos implementar tanto el procesamiento por lotes como el almacenamiento en caché utilizando una ligera variación de la técnica que acabamos de explorar.

##### Procesamiento por lotes y almacenamiento en caché de solicitudes

Agregar una capa de almacenamiento en caché a nuestra API de procesamiento por lotes es sencillo, gracias a las promesas. Todo lo que debemos hacer es dejar la promesa en nuestro mapa de solicitudes, incluso después de que se haya completado la solicitud.

Implementemos el módulo `totalSalesCache.js` de inmediato:

```javascript
// totalSalesCache.js
import { totalSales as totalSalesRaw } from './totalSales.js'

const CACHE_TTL = 30 * 1000 // 30 seconds TTL
const cache = new Map()

export function totalSales(product) {
  if (cache.has(product)) {
    console.log('Cache hit')
    return cache.get(product)
  }

  const resultPromise = totalSalesRaw(product)
  cache.set(product, resultPromise)
  resultPromise.then(
    () => {
      setTimeout(() => {
        cache.delete(product)
      }, CACHE_TTL)
    },
    err => {
      cache.delete(product)
      throw err
    }
  )

  return resultPromise
}
```

Todo lo que debemos hacer es eliminar la promesa de la caché después de un cierto tiempo (`CACHE_TTL`) tras completarse la solicitud, o inmediatamente si la solicitud ha fallado. Esta es una técnica de invalidación de caché muy básica, pero funciona perfectamente para nuestra demostración.

Ahora, estamos listos para probar el wrapper de almacenamiento en caché `totalSales()` que acabamos de crear. Para hacer eso, solo necesitamos actualizar el módulo `server.js`, de la siguiente manera:

```javascript
// import { totalSales } from './totalSales.js'
// import { totalSales } from './totalSalesBatch.js'
import { totalSales } from './totalSalesCache.js'

createServer(async (req, res) => {
  // ...
```

Ahora, podemos iniciar el servidor nuevamente y compararlo con Autocannon.

Esta vez, en mi máquina, ¡el servidor puede manejar alrededor de **376.000 solicitudes en unos 10 segundos**!

Eso es más de 350 veces la cantidad de solicitudes manejadas en el ejemplo de procesamiento por lotes y aproximadamente **2.500 veces más** que nuestra versión original sin procesamiento por lotes. ¡Un incremento de rendimiento increíble para un cambio tan simple!

Por supuesto, estos resultados dependen en gran medida de muchos factores, como la cantidad de solicitudes entrantes y el retraso entre ellas. Las ventajas del almacenamiento en caché sobre el procesamiento por lotes se vuelven mucho más sustanciales cuando el volumen de solicitudes es alto y se extiende durante un período más largo.

En aplicaciones del mundo real, pueden ser necesarias estrategias más avanzadas de invalidación y almacenamiento en caché:
- Para limitar el uso de memoria al almacenar en caché muchos valores, utiliza políticas como **Least Recently Used (LRU)** o **First In First Out (FIFO)**.
- En despliegues distribuidos, las memorias caché en memoria pueden producir resultados inconsistentes entre instancias. Un almacén compartido como **Redis** ([nodejsdp.link/redis](https://nodejsdp.link/redis)), **Valkey** ([nodejsdp.link/valkey](https://nodejsdp.link/valkey)) o **Memcached** ([nodejsdp.link/memcached](https://nodejsdp.link/memcached)) mantiene los datos consistentes y puede ser más eficiente.
- La invalidación manual (activada cuando cambian los datos subyacentes) permite tiempos de vida de caché más largos mientras se mantienen los resultados actualizados, pero es más compleja de administrar. Como dijo célebremente Phil Karlton: *"Solo hay dos cosas difíciles en Ciencias de la Computación: la invalidación de la caché y ponerle nombre a las cosas"*.

---

### Sección 3: Cancelación de operaciones asíncronas

Poder detener una operación de larga duración es particularmente útil si la operación ha sido cancelada por el usuario o si se ha vuelto redundante. En la programación multiproceso (*multithreaded*), simplemente podemos terminar el hilo, pero en una plataforma de un solo hilo como Node.js, las cosas pueden complicarse un poco más.

En esta sección, nos centraremos en cancelar operaciones asíncronas, no en cancelar las promesas en sí, lo cual es un tema diferente. Vale la pena señalar que la especificación Promises/A+ no define una API para cancelar promesas. Sin embargo, si necesitas esta funcionalidad, puedes usar una biblioteca de promesas de terceros como Bluebird (consulta más en [nodejsdp.link/bluebird-cancelation](https://nodejsdp.link/bluebird-cancelation)). Ten en cuenta que cancelar una promesa no cancela automáticamente la operación asíncrona subyacente. De hecho, Bluebird ofrece un callback `onCancel` en su constructor de promesas, junto con `resolve` y `reject`, que se puede usar para cancelar explícitamente la operación subyacente cuando se cancela la promesa. Y eso es exactamente lo que exploraremos en esta sección.

#### Una receta básica para crear funciones cancelables

En la programación asíncrona, el principio básico para cancelar la ejecución de una función es muy simple: comprobamos si la operación ha sido cancelada después de cada llamada asíncrona y, si es así, salimos prematuramente de la operación. Considera, por ejemplo, el siguiente código:

```javascript
// index.js
import { asyncRoutine } from './asyncRoutine.js'
import { CancelError } from './cancelError.js'

async function cancelable(cancelObj) {
  const resA = await asyncRoutine('A')
  console.log(resA)
  if (cancelObj.cancelRequested) {
    throw new CancelError()
  }

  const resB = await asyncRoutine('B')
  console.log(resB)
  if (cancelObj.cancelRequested) {
    throw new CancelError()
  }

  const resC = await asyncRoutine('C')
  console.log(resC)
}
```

La función `cancelable()` recibe, como entrada, un objeto (`cancelObj`) que contiene una única propiedad llamada `cancelRequested`. En la función, comprobamos la propiedad `cancelRequested` después de cada llamada asíncrona y, si es `true`, lanzamos una excepción especial `CancelError` para interrumpir la ejecución de la función.

La función `asyncRoutine()` es solo una función de demostración que imprime una cadena en la consola y devuelve otra cadena después de 100 ms. Encontrarás su implementación completa, junto con la de `CancelError`, en el repositorio de código de este libro ([nodejsdp.link/canceling-async-simple](https://nodejsdp.link/canceling-async-simple)).

Es importante tener en cuenta que cualquier código externo a la función `cancelable()` podrá establecer la propiedad `cancelRequested` solo después de que la función `cancelable()` devuelva el control al bucle de eventos, lo que generalmente ocurre cuando se espera una operación asíncrona con `await`. Es por eso que vale la pena comprobar la propiedad `cancelRequested` solo después de la finalización de una operación asíncrona y no con más frecuencia.

Podrías pensar que comprobar un indicador `cancelRequested` después de cada `await` es poco elegante y no sigue el principio DRY. Eso es cierto, pero refleja un principio fundamental de la programación asíncrona: el entorno de ejecución no detendrá una función asíncrona de forma arbitraria, solo en los puntos de `await` donde el control vuelve al bucle de eventos (consulta la sección *La filosofía de Node.js* del [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1)). JavaScript utiliza **multitarea cooperativa**, donde las tareas ceden el control voluntariamente (por ejemplo, con `await`), a diferencia de la multitarea apropiativa (*pre-emptive*) en entornos multiproceso, donde el sistema operativo puede interrumpir o terminar hilos en cualquier momento. El punto clave es que en el modelo asíncrono tenemos el control, y con ese control viene la responsabilidad: si queremos la cancelación, nuestro código debe comprobarlo periódicamente y actuar en consecuencia.

El siguiente código demuestra cómo podemos cancelar la función `cancelable()`:

```javascript
const cancelObj = {
  cancelRequested: false,
}

// schedule to set the cancel flag to true after 100ms
setTimeout(() => {
  cancelObj.cancelRequested = true
}, 100)

try {
  // starts the processing
  await cancelable(cancelObj)
} catch (err) {
  if (err instanceof CancelError) {
    console.log('Function canceled')
  } else {
    console.error(err)
  }
}
```

Como podemos ver, todo lo que debemos hacer para cancelar la función es establecer la propiedad `cancelObj.cancelRequested` en `true`. Esto hará que la función se detenga y lance un `CancelError`.

#### Envolver invocaciones asíncronas

Crear y usar una función cancelable asíncrona básica es muy fácil, pero hay mucho código repetitivo (*boilerplate*) involucrado. De hecho, implica tanto código adicional que se vuelve difícil identificar la lógica de negocio real de la función.

Podemos reducir el código repetitivo incluyendo la lógica de cancelación dentro de una función envoltorio (*wrapper*), que podemos usar para invocar rutinas asíncronas.

Dicho wrapper se vería de la siguiente manera (el archivo `cancelWrapper.js`):

```javascript
// cancelWrapper.js
import { CancelError } from './cancelError.js'

export function createCancelWrapper() {
  let cancelRequested = false

  function cancel() {
    cancelRequested = true
  }

  function callIfNotCanceled(func, ...args) {
    if (cancelRequested) {
      return Promise.reject(new CancelError())
    }
    return func(...args)
  }

  return { callIfNotCanceled, cancel }
}
```

La fábrica devuelve dos funciones:
- `callIfNotCanceled()`: Una función envoltorio que nos permite llamar a una función asíncrona solo si la operación asíncrona general aún no ha sido cancelada.
- `cancel()`: Una función que activa la cancelación.

Esta configuración nos permite envolver múltiples invocaciones asíncronas con la misma función contenedora y luego usar una sola llamada `cancel()` para cancelarlas todas a la vez.

La función `callIfNotCanceled()` toma, como entrada, una función a invocar (`func`) y un conjunto de parámetros para pasar a la función (`args`). El wrapper simplemente comprueba si se ha solicitado una cancelación y, en caso afirmativo, devolverá una promesa rechazada con un objeto `CancelError` como motivo del rechazo; de lo contrario, invocará `func()` con los argumentos dados (`args`) y devolverá su valor de retorno.

Veamos ahora cómo nuestra fábrica de wrappers puede mejorar en gran medida la legibilidad y la modularidad de nuestra función `cancelable()`:

```javascript
// index.js
import { asyncRoutine } from './asyncRoutine.js'
import { createCancelWrapper } from './cancelWrapper.js'
import { CancelError } from './cancelError.js'

async function cancelable(callIfNotCanceled) {
  const resA = await callIfNotCanceled(asyncRoutine, 'A')
  console.log(resA)
  const resB = await callIfNotCanceled(asyncRoutine, 'B')
  console.log(resB)
  const resC = await callIfNotCanceled(asyncRoutine, 'C')
  console.log(resC)
}

const { callIfNotCanceled, cancel } = createCancelWrapper()

setTimeout(cancel, 100)

try {
  await cancelable(callIfNotCanceled)
} catch (err) {
  if (err instanceof CancelError) {
    console.log('Function canceled')
  } else {
    console.error(err)
  }
}
```

Podemos ver de inmediato los beneficios de usar una función wrapper para implementar nuestra lógica de cancelación. De hecho, la función `cancelable()` ahora es mucho más concisa y legible.

#### Funciones asíncronas cancelables con AbortController

Las soluciones anteriores funcionan, pero dependen de una API personalizada, lo que limita la interoperabilidad con código de terceros. Por ejemplo, si publicas una biblioteca con una función asíncrona cancelable, ¿quién proporciona el objeto de cancelación? ¿Tu biblioteca o el código del usuario? Si cada biblioteca define su propio mecanismo, la interoperabilidad se resiente. La solución es utilizar un estándar, y ahí es donde entra **AbortController** ([nodejsdp.link/abort-controller](https://nodejsdp.link/abort-controller)). AbortController es una interfaz estándar en JavaScript (disponible en navegadores y en Node.js) que te permite abortar una o más operaciones asíncronas según sea necesario.

Si alguna vez has cancelado una solicitud `fetch()`, ya has utilizado AbortController, posiblemente sin siquiera darte cuenta.

Para crear una instancia de AbortController, simplemente llamas a:

```javascript
const ac = new AbortController()
```

Ten en cuenta que la clase `AbortController` está disponible en el ámbito global (no se requiere importación explícita).

El `AbortController` expone dos partes importantes: el propio controlador (que puede activar la cancelación) y la señal asociada (`signal`, que las operaciones asíncronas utilizan para escuchar solicitudes de cancelación).

Para activar una cancelación, llamas al método `abort()` en la instancia del controlador:

```javascript
ac.abort()
```

Cuando llamas a `ac.abort()`, opcionalmente puedes pasar un objeto de error como argumento. Si no proporcionas un error, se crea uno predeterminado internamente utilizando la clase `DOMException` con la propiedad `name` establecida en `'AbortError'`.

Esto envía una señal a todas las operaciones que están escuchando el objeto `AbortSignal` asociado.

Para conectar una función asíncrona al mecanismo de cancelación, puedes pasar la propiedad `signal` del controlador (una instancia de `AbortSignal`) como argumento:

```javascript
someAsyncFunction(ac.signal)
```

Dentro de la función asíncrona, puedes comprobar si se ha solicitado la cancelación de dos maneras:

Una forma es llamando a `abortSignal.throwIfAborted()`, que lanza inmediatamente un error si la señal ha sido abortada e interrumpe el flujo:

```javascript
abortSignal.throwIfAborted()
```

Esto lanzará el error personalizado que pasaste al llamar a `ac.abort()` o un error de cancelación predeterminado.

Alternativamente, puedes verificar la propiedad `abortSignal.aborted`, un valor booleano que te dice si la operación ha sido cancelada, lo que permite un manejo más personalizado:

```javascript
if (abortSignal.aborted) {
  // Handle cancellation manually (throw or return)
}
```

Además del método `throwIfAborted()` y la propiedad `aborted`, un `AbortSignal` también proporciona una interfaz basada en eventos. Puedes escuchar el evento `'abort'` utilizando `abortSignal.addEventListener()`. Por ejemplo:

```javascript
ac.signal.addEventListener(
  'abort',
  event => {
    console.log(event.type) // Prints 'abort'
  },
  { once: true }
)
```

Esto te permite reaccionar a los eventos de cancelación de una manera más flexible o desacoplada. La opción `{ once: true }` asegura que el escuchador se elimine automáticamente después de que se active el evento.

Es importante comprender claramente la diferencia entre el controlador y la señal:
- El **AbortController** es la parte responsable de activar la cancelación; normalmente es propiedad de quien inicia la operación asíncrona y quiere tener la capacidad de cancelarla más adelante.
- El **AbortSignal** es la parte responsable de recibir y reaccionar a la solicitud de cancelación; se pasa a la operación asíncrona y se comprueba internamente para decidir cómo comportarse cuando se produce la cancelación.

Esta separación deja claro quién tiene la autoridad para solicitar una cancelación y quién necesita escucharla y reaccionar en consecuencia.

Ahora que estamos familiarizados con el AbortController y su API, veamos cómo podemos reimplementar nuestro ejemplo anterior para lograr un enfoque más estandarizado e interoperable:

```javascript
// index.js
import { asyncRoutine } from './asyncRoutine.js'

async function cancelable(abortSignal) {
  abortSignal.throwIfAborted()
  const resA = await asyncRoutine('A')
  console.log(resA)

  abortSignal.throwIfAborted()
  const resB = await asyncRoutine('B')
  console.log(resB)

  abortSignal.throwIfAborted()
  const resC = await asyncRoutine('C')
  console.log(resC)
}

const ac = new AbortController()
setTimeout(() => ac.abort(), 100)

try {
  await cancelable(ac.signal)
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Function canceled')
  } else {
    console.error(err)
  }
}
```

Este ejemplo pone en práctica lo que hemos discutido sobre AbortController y AbortSignal.

La función `cancelable()` acepta un `AbortSignal` y comprueba periódicamente si se ha solicitado una cancelación llamando a `abortSignal.throwIfAborted()` antes de cada paso asíncrono.

Mientras tanto, se crea una instancia de `AbortController` (`ac`) y, después de 100 milisegundos, se llama a `ac.abort()` para activar la cancelación.

Cuando se solicita la cancelación, la siguiente llamada a `abortSignal.throwIfAborted()` lanza una excepción, que se captura en el bloque `try...catch`. Si el error es un `AbortError`, el programa registra que la función fue cancelada. De lo contrario, cualquier otro error inesperado se registra por separado.

Si deseas hacer que el código anterior sea un poco más compacto y evitar un poco de repetición, puedes crear una función wrapper `callIfNotAborted()`. Esta función tomaría un `AbortSignal`, otra función para llamar y una lista de argumentos. Primero verificaría si la señal ha sido abortada (por ejemplo, llamando a `abortSignal.throwIfAborted()`) y, si no, invocaría la función proporcionada con los argumentos dados y devolvería su resultado. Esta utilidad nos permitiría ahorrar una línea en cada punto de control, convirtiendo algo como esto:

```javascript
abortSignal.throwIfAborted()
const resA = await asyncRoutine('A')
```

en algo como esto:

```javascript
const resA = await callIfNotAborted(abortSignal, asyncRoutine, 'A')
```

Esta idea es muy similar a la función `callIfNotCanceled()` que implementamos anteriormente, pero en este caso, está estandarizada mediante el uso de un `AbortSignal` en lugar de un objeto de cancelación personalizado.

Ahora que has visto cómo funciona el estándar AbortController, este es el enfoque que deberías preferir al implementar operaciones asíncronas cancelables en tu propio código. Pasar algún tiempo antes construyendo nuestro propio mecanismo de cancelación no fue un esfuerzo en vano: debería haberte dado una comprensión más profunda de cómo funciona la cancelación bajo el capó y por qué AbortController está diseñado de la forma en que lo está. Con esta base, ahora deberías poder utilizar AbortController de manera más eficaz y correcta en escenarios del mundo real.

---

### Sección 4: Ejecución de tareas intensivas de CPU

La API `totalSales()` que implementamos en la sección *Procesamiento por lotes y almacenamiento en caché de solicitudes asíncronas* era (intencionalmente) costosa en términos de recursos y tardaba unos cientos de milisegundos en ejecutarse. No obstante, invocar la función `totalSales()` no afectó la capacidad de la aplicación para procesar solicitudes entrantes concurrentes. Lo que aprendimos sobre el bucle de eventos en el [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1), La plataforma Node.js, debería explicar este comportamiento: invocar una operación asíncrona siempre hace que la pila se desenrolle de nuevo al bucle de eventos, dejándolo libre para manejar otras solicitudes.

Pero, ¿qué sucede cuando ejecutamos una tarea sincrónica costosa que tarda mucho tiempo en completarse y que no devuelve el control al bucle de eventos hasta que haya finalizado? Este tipo de tarea también se conoce como **intensiva de CPU** (*CPU-bound*), porque su característica principal es que hace un uso intensivo de la CPU en lugar de operaciones de I/O. Trabajemos de inmediato en un ejemplo para ver cómo se comportan este tipo de tareas en Node.js.

#### Resolver el problema de la suma de subconjuntos

Para dar inicio a esta sección, elijamos un problema computacionalmente costoso que sea fácil de entender e ideal para la experimentación. Un candidato perfecto es el **problema de la suma de subconjuntos** (*subset sum problem*), que pregunta si un conjunto (o multiconjunto) de enteros contiene un subconjunto no vacío cuya suma sea igual a cero. Por ejemplo, dada la entrada `[1, 2, -4, 5, -3]`, las soluciones válidas incluyen `[1, 2, -3]` y `[2, -4, 5, -3]`.

El enfoque de fuerza bruta para resolver este problema implica comprobar cada combinación posible de subconjuntos, una operación con un costo de $\mathcal{O}(2^n)$. Eso significa que incluso una modesta entrada de 20 enteros requeriría comprobar más de un millón de combinaciones (específicamente, 1.048.576). Ese es exactamente el tipo de carga de trabajo que necesitamos para poner a prueba nuestras ideas.

El problema de la suma de subconjuntos no es solo una curiosidad teórica. Desempeña un papel crucial en la resolución de desafíos del mundo real en una variedad de campos. En criptografía, se ha utilizado para diseñar esquemas de cifrado de clave pública, aprovechando su complejidad computacional para proteger datos confidenciales. En finanzas, ayuda a optimizar las carteras de inversión seleccionando combinaciones de activos que cumplen con criterios específicos de rendimiento y riesgo. En la asignación de recursos y la programación de proyectos, ayuda a distribuir eficientemente los recursos limitados entre tareas en competencia. En bioinformática, los investigadores lo utilizan para identificar secuencias genéticas o estructuras de proteínas que coinciden con propiedades específicas. Incluso en juegos de rompecabezas y resolución de problemas impulsada por IA, los algoritmos de suma de subconjuntos están detrás de escena, identificando configuraciones o movimientos válidos.

Para nuestra versión del problema, iremos un paso más allá: en lugar de simplemente buscar subconjuntos que sumen cero, buscaremos **todos los subconjuntos cuya suma coincida con un valor objetivo determinado**. Esta variación generalizada no solo amplía el alcance, sino que nos brinda el entorno de pruebas perfecto para explorar cuellos de botella de rendimiento, cancelación y control sobre cálculos de larga duración.

Ahora, trabajemos para implementar dicho algoritmo. Primero, creemos un nuevo módulo llamado `subsetSum.js`. Comenzaremos creando una clase llamada `SubsetSum`:

```javascript
// subsetSum.js
import { EventEmitter } from 'node:events'

export class SubsetSum extends EventEmitter {
  constructor(sum, set) {
    super()
    this.sum = sum
    this.set = set
    this.totalSubsets = 0
  }

  //...
```

La clase `SubsetSum` extiende `EventEmitter`. Esto nos permite producir un evento cada vez que encontramos un nuevo subconjunto que coincide con la suma recibida como entrada. Como veremos, esto nos dará mucha flexibilidad.

Un enfoque alternativo sería usar generadores y producir (*yield*) una nueva solución cada vez que se encuentre una. Esta puede ser una forma limpia y eficiente de generar resultados perezosamente (*lazily*), especialmente cuando estás interesado en iterar sobre las soluciones una a la vez en lugar de manejarlas a través de eventos.

A continuación, veamos cómo podemos generar todas las combinaciones posibles de subconjuntos:

```javascript
  _combine(set, subset) {
    for (let i = 0; i < set.length; i++) {
      const newSubset = subset.concat(set[i])
      this._combine(set.slice(i + 1), newSubset)
      this._processSubset(newSubset)
    }
  }
```

No entraremos en demasiados detalles sobre el algoritmo, pero hay dos cosas importantes a tener en cuenta:
- El método `_combine()` es completamente sincrónico. Genera recursivamente todos los subconjuntos posibles sin devolver nunca el control al bucle de eventos.
- Cada vez que se genera una nueva combinación, se la proporcionamos al método `_processSubset()` para su posterior procesamiento.

El método `_processSubset()` es responsable de verificar que la suma de los elementos del subconjunto dado sea igual al número que estamos buscando:

```javascript
  _processSubset(subset) {
    console.log('Subset', ++this.totalSubsets, subset)
    const res = subset.reduce((prev, item) => (prev + item), 0)
    if (res === this.sum) {
      this.emit('match', subset)
    }
  }
```

El método `_processSubset()` aplica una operación `reduce` al subconjunto para calcular la suma de sus elementos. Luego, emite un evento del tipo `match` cuando la suma resultante es igual a la que nos interesa encontrar (`this.sum`).

Finalmente, el método `start()` une todas las piezas anteriores:

```javascript
  start() {
    this._combine(this.set, [])
    this.emit('end')
  }
```

El método `start()` desencadena la generación de todas las combinaciones invocando `_combine()` y, por último, emite un evento `end`, señalando que se comprobaron todas las combinaciones y que ya se ha emitido cualquier posible coincidencia. Esto es posible porque `_combine()` es sincrónico; por lo tanto, el evento `end` se emite tan pronto como la función retorna, lo que significa que se han calculado todas las combinaciones.

A continuación, debemos exponer el algoritmo que acabamos de crear a través de la red. Como siempre, podemos usar un servidor HTTP simple para esta tarea. Queremos crear un endpoint que acepte solicitudes en el formato `/subsetSum?data=<Array>&sum=<Integer>` que invoque el algoritmo `SubsetSum` con el array dado de enteros y la suma a coincidir.

Implementemos este servidor simple en un módulo llamado `index.js`:

```javascript
// index.js
import { createServer } from 'node:http'
import { SubsetSum } from './subsetSum.js'

createServer((req, res) => {
  const url = new URL(req.url, 'http://localhost')
  if (url.pathname !== '/subsetSum') {
    res.writeHead(200)
    return res.end("I'm alive!
")
  }

  const data = JSON.parse(url.searchParams.get('data'))
  const sum = JSON.parse(url.searchParams.get('sum'))

  res.writeHead(200)
  const subsetSum = new SubsetSum(sum, data)
  subsetSum.on('match', match => {
    res.cork()
    res.write(`Match: ${JSON.stringify(match)}
`)
    res.uncork()
  })
  subsetSum.on('end', () => res.end())
  subsetSum.start()
}).listen(8000, () => console.log('Server started'))
```

Gracias al hecho de que el objeto `SubsetSum` devuelve sus resultados mediante eventos, podemos transmitir los subconjuntos coincidentes tan pronto como el algoritmo los genera, en tiempo real. Otro detalle a mencionar es que nuestro servidor responde con el texto `I'm alive!` cada vez que accedemos a una URL diferente de `/subsetSum`. Usaremos esto para comprobar la capacidad de respuesta de nuestro servidor, como veremos en un momento.

En el código anterior, usamos `res.cork()` y `res.uncork()` para controlar cómo se escriben los datos en el flujo de respuesta HTTP. Normalmente, los streams de escritura en Node.js pueden almacenar en búfer pequeñas escrituras internamente antes de enviarlas realmente. Al llamar explícitamente a `res.cork()` antes de escribir y a `res.uncork()` inmediatamente después, nos aseguramos de que cada coincidencia se envíe al cliente tan pronto como esté disponible ([nodejsdp.link/cork](https://nodejsdp.link/cork)). Este truco nos permite ver aparecer cada nueva coincidencia en tiempo real, lo que nos ayuda a apreciar cuánto tiempo puede pasar entre un resultado y otro, especialmente cuando se trata de cálculos costosos donde los resultados pueden no llegar de inmediato.

Ahora estamos listos para probar nuestro algoritmo de suma de subconjuntos. ¿Tienes curiosidad por saber cómo lo manejará nuestro servidor? Vamos a encenderlo, entonces:

```bash
node index.js
```

Tan pronto como se inicia el servidor, estamos listos para enviar nuestra primera solicitud. Probémoslo con un multiconjunto de 17 números aleatorios, lo que resultará en la generación de 131.071 combinaciones, una buena cantidad para mantener ocupado a nuestro servidor por un tiempo:

```bash
curl -G http://localhost:8000/subsetSum --data-urlencode "data=[16, 19,1,1,-16,9,1,-5,-2,17,-15,-97,19,-16,-4,-5,15]" --data-urlencode "sum=0"
```

Deberíamos ver los resultados provenientes del servidor. Pero si probamos el siguiente comando en otra terminal mientras la primera solicitud aún se está ejecutando, detectaremos un gran problema:

```bash
curl -G http://localhost:8000
```

Veremos inmediatamente que esta última solicitud se bloquea hasta que haya finalizado el algoritmo de suma de subconjuntos de la primera solicitud: ¡el servidor no responde! Esto era de esperar. El bucle de eventos de Node.js se ejecuta en un solo hilo y, si este hilo está bloqueado por un cálculo sincrónico largo, no podrá ejecutar ni un solo ciclo para responder con un simple `I'm alive!`.

Entendemos rápidamente que este comportamiento no funciona para ningún tipo de aplicación destinada a procesar múltiples solicitudes concurrentes. Pero no te desesperes. En Node.js, podemos abordar este tipo de situaciones de varias maneras. Analicemos los tres métodos más populares: **intercalado con setImmediate**, **uso de procesos externos** y **uso de worker threads**.

> [!WARNING]
> Tener un endpoint que pueda bloquear el bucle de eventos durante un período prolongado no es solo un problema de rendimiento; también puede exponer tu aplicación a ataques de **Denegación de Servicio (DoS)**. El objetivo de un ataque DoS es agotar los recursos de un sistema e inutilizarlo para usuarios legítimos, a menudo explotando vulnerabilidades o inundándolo con cantidades masivas de tráfico (como en los ataques DDoS, o Denegación de Servicio Distribuida). En el contexto de Node.js, un atacante podría desencadenar deliberadamente cálculos pesados para bloquear el bucle de eventos y evitar que el servidor responda a cualquier otra solicitud. Incluso sin intenciones maliciosas, un servidor bajo una carga alta normal podría experimentar problemas similares si las operaciones de larga duración no están debidamente aisladas. Por esta razón, es fundamental evitar bloquear el bucle de eventos para construir aplicaciones Node.js confiables, seguras y escalables.

#### Intercalado con setImmediate

Por lo general, un algoritmo intensivo de CPU se basa en un conjunto de pasos. Esto puede ser un conjunto de invocaciones recursivas, un bucle o cualquier variación/combinación de estos. Por lo tanto, una solución simple a nuestro problema sería devolver el control al bucle de eventos después de que se complete cada uno de estos pasos (o después de un cierto número de ellos). De esta manera, el bucle de eventos aún puede procesar cualquier I/O pendiente en aquellos intervalos en los que el algoritmo de larga duración cede la CPU. Una forma sencilla de lograr esto es programar el siguiente paso del algoritmo para que se ejecute después de cualquier solicitud de I/O pendiente. Este parece el caso de uso perfecto para la función `setImmediate()` (ya presentamos esta API en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos).

##### Intercalar los pasos del algoritmo de suma de subconjuntos

Veamos ahora cómo se aplica esta técnica a nuestro algoritmo de suma de subconjuntos. Todo lo que debemos hacer es modificar ligeramente el módulo `subsetSum.js`. Por comodidad, vamos a crear un nuevo módulo llamado `subsetSumDefer.js`, tomando como punto de partida el código de la clase `SubsetSum` original.

El primer cambio que vamos a hacer es agregar un nuevo método llamado `_combineInterleaved()`, que es el núcleo de la técnica que estamos implementando:

```javascript
  _combineInterleaved(set, subset) {
    this.runningCombine++
    setImmediate(() => {
      this._combine(set, subset)
      if (--this.runningCombine === 0) {
        this.emit('end')
      }
    })
  }
```

Como podemos ver, todo lo que tuvimos que hacer fue aplazar la invocación del método original (sincrónico) `_combine()` con `setImmediate()`. Sin embargo, ahora resulta más difícil saber cuándo la función ha terminado de generar todas las combinaciones, porque el algoritmo ya no es sincrónico.

Para solucionar esto, debemos realizar un seguimiento de todas las instancias en ejecución del método `_combine()` utilizando un patrón muy similar al flujo de ejecución paralelo asíncrono que vimos en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con callbacks. Cuando todas las instancias del método `_combine()` hayan terminado de ejecutarse, podemos emitir el evento `end`, notificando a cualquier escuchador que el proceso se ha completado.

Para terminar de refactorizar el algoritmo de suma de subconjuntos, necesitamos hacer un par de ajustes más. Primero, debemos reemplazar el paso recursivo en el método `_combine()` con su contraparte diferida:

```javascript
  _combine(set, subset) {
    for (let i = 0; i < set.length; i++) {
      const newSubset = subset.concat(set[i])
      this._combineInterleaved(set.slice(i + 1), newSubset)
      this._processSubset(newSubset)
    }
  }
```

Con el cambio anterior, nos aseguramos de que cada paso del algoritmo se encole en el bucle de eventos usando `setImmediate()` y, por lo tanto, se ejecute después de cualquier solicitud de I/O pendiente en lugar de ejecutarse sincrónicamente.

El otro pequeño ajuste está en el método `start()`:

```javascript
  start() {
    this.runningCombine = 0
    this._combineInterleaved(this.set, [])
  }
```

En el código anterior, inicializamos el número de instancias en ejecución del método `_combine()` a 0. También reemplazamos la llamada a `_combine()` con una llamada a `_combineInterleaved()` y eliminamos la emisión del evento `end` porque ahora se maneja de forma asíncrona en `_combineInterleaved()`.

Aquí está el módulo completo `subsetSumDefer.js`:

```javascript
// subsetSumDefer.js
import { EventEmitter } from 'node:events'

export class SubsetSum extends EventEmitter {
  constructor(sum, set) {
    super()
    this.sum = sum
    this.set = set
    this.totalSubsets = 0
  }

  _combineInterleaved(set, subset) {
    this.runningCombine++
    setImmediate(() => {
      this._combine(set, subset)
      if (--this.runningCombine === 0) {
        this.emit('end')
      }
    })
  }

  _combine(set, subset) {
    for (let i = 0; i < set.length; i++) {
      const newSubset = subset.concat(set[i])
      this._combineInterleaved(set.slice(i + 1), newSubset)
      this._processSubset(newSubset)
    }
  }

  _processSubset(subset) {
    console.log('Subset', ++this.totalSubsets, subset)
    const res = subset.reduce((prev, item) => prev + item, 0)
    if (res === this.sum) {
      this.emit('match', subset)
    }
  }

  start() {
    this.runningCombine = 0
    this._combineInterleaved(this.set, [])
  }
}
```

La última pieza que falta es actualizar el módulo `index.js` para que pueda usar la nueva versión de la API `SubsetSum`:

```javascript
import { createServer } from 'node:http'
// import { SubsetSum } from './subsetSum.js'
import { SubsetSum } from './subsetSumDefer.js'

createServer((req, res) => {
  // ...
```

Ahora estamos listos para probar esta nueva versión del servidor de suma de subconjuntos. Inicia el servidor nuevamente y luego intenta enviar una solicitud para calcular todos los subconjuntos que coincidan con una suma determinada:

```bash
curl -G http://localhost:8000/subsetSum --data-urlencode "data=[16, 19,1,1,-16,9,1,-5,-2,17,-15,-97,19,-16,-4,-5,15]" --data-urlencode "sum=0"
```

Mientras se ejecuta la solicitud, comprueba nuevamente si el servidor responde:

```bash
curl -G http://localhost:8000
```

¡Genial! La segunda solicitud debería regresar casi de inmediato, incluso mientras la tarea `SubsetSum` aún se está ejecutando, lo que confirma que nuestra técnica funciona bien.

##### Consideraciones sobre el enfoque de intercalado

Como vimos, ejecutar una tarea intensiva de CPU mientras se preserva la capacidad de respuesta de una aplicación no es tan complicado; solo requiere el uso de `setImmediate()` para programar el siguiente paso de un algoritmo para que se ejecute después de cualquier I/O pendiente. Sin embargo, esta no es necesariamente la mejor receta en términos de eficiencia. De hecho, aplazar una tarea introduce una pequeña sobrecarga que, multiplicada por todos los pasos que tiene que ejecutar un algoritmo, puede tener un impacto significativo en el tiempo de ejecución general. Esto suele ser lo último que queremos cuando ejecutamos una tarea intensiva de CPU. Una posible solución para mitigar este problema sería usar `setImmediate()` solo después de un cierto número de pasos (en lugar de usarlo en cada paso), pero aún así, esto no resolvería la raíz del problema.

Además, esta técnica no funciona muy bien si cada paso tarda mucho tiempo en ejecutarse. En este caso, de hecho, el bucle de eventos perdería capacidad de respuesta y toda la aplicación comenzaría a retrasarse, lo cual no es deseable en un entorno de producción.

Ten en cuenta que esto no significa que la técnica que acabamos de ver deba evitarse a toda costa. En ciertas situaciones, en las que la tarea sincrónica se ejecuta esporádicamente y no tarda demasiado en ejecutarse, usar `setImmediate()` para intercalar su ejecución es a veces la forma más simple y efectiva de evitar bloquear el bucle de eventos.

> [!NOTE]
> Recuerda que `process.nextTick()` **no** se puede utilizar para intercalar una tarea de larga duración. Como vimos en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos, `nextTick()` programa una tarea para que se ejecute antes de cualquier I/O pendiente, y esto causará inanición de I/O (*I/O starvation*) en caso de llamadas repetidas. Puedes verificarlo tú mismo reemplazando `setImmediate()` con `process.nextTick()` en el ejemplo anterior.

---

#### Uso de procesos externos

Aplazar los pasos de un algoritmo no es la única opción que tenemos para ejecutar tareas intensivas de CPU. Otro patrón para evitar que el bucle de eventos se bloquee es utilizar **procesos hijos** (*child processes*).

Ya sabemos que Node.js rinde al máximo cuando ejecuta aplicaciones con uso intensivo de I/O, como servidores web, donde su arquitectura asíncrona ayuda a optimizar la utilización de recursos. Para mantener la capacidad de respuesta de una aplicación, la mejor estrategia es evitar por completo la ejecución de tareas costosas de CPU en el contexto de la aplicación principal. El objetivo es eliminar los cálculos pesados del bucle de eventos principal y moverlos a otra parte, por ejemplo, descargándolos a un proceso separado, para que el bucle de eventos permanezca libre para manejar rápidamente las solicitudes entrantes. Esto tiene tres ventajas principales:
- La tarea sincrónica puede ejecutarse a toda velocidad, sin la necesidad de intercalar los pasos de su ejecución.
- Trabajar con procesos en Node.js es simple, probablemente más fácil que modificar un algoritmo para usar `setImmediate()`, y nos permite usar fácilmente múltiples procesadores sin la necesidad de escalar la aplicación principal en sí.
- Si realmente necesitamos el máximo rendimiento, el proceso externo podría crearse en lenguajes de nivel inferior, como el buen C, o lenguajes compilados más modernos como Go, Rust o Zig. ¡Utiliza siempre la mejor herramienta para el trabajo!

Node.js tiene un amplio conjunto de herramientas y APIs para interactuar con procesos externos. Podemos encontrar todo lo que necesitamos en el módulo `node:child_process`. Además, cuando el proceso externo es solo otro programa de Node.js, conectarlo a la aplicación principal es sumamente fácil y permite una comunicación fluida con la aplicación local. Esta magia ocurre gracias a la función `child_process.fork()`, que crea un nuevo proceso hijo de Node.js y crea automáticamente un canal de comunicación con él, lo que nos permite intercambiar información utilizando una interfaz muy similar a la de `EventEmitter`. Veamos cómo funciona esto refactorizando nuestro servidor de suma de subconjuntos nuevamente.

##### Delegar la tarea de suma de subconjuntos a un proceso externo

El objetivo de refactorizar la tarea `SubsetSum` es crear un proceso hijo separado responsable de manejar el procesamiento sincrónico, dejando el bucle de eventos del servidor principal libre para manejar las solicitudes provenientes de la red. Esta es la receta que vamos a seguir para hacerlo posible:
1. Crearemos un nuevo módulo llamado `processPool.js` que nos permitirá crear un grupo (*pool*) de procesos en ejecución. Iniciar un nuevo proceso es costoso y requiere tiempo, por lo que mantenerlos en ejecución constante y listos para manejar solicitudes nos permite ahorrar tiempo y ciclos de CPU. Además, el grupo nos ayudará a limitar el número de procesos que se ejecutan al mismo tiempo para evitar exponer la aplicación a ataques DoS.
2. A continuación, crearemos un módulo llamado `subsetSumFork.js`, responsable de abstraer una tarea `SubsetSum` que se ejecuta en un proceso hijo. Su función será comunicarse con el proceso hijo y reenviar los resultados de la tarea como si provinieran de la aplicación actual.
3. Finalmente, necesitamos un **worker** (nuestro proceso hijo), un nuevo programa de Node.js con el único objetivo de ejecutar el algoritmo de suma de subconjuntos y reenviar sus resultados al proceso padre.

##### Implementar un pool de procesos

Comencemos construyendo el módulo `processPool.js` pieza por pieza:

```javascript
// processPool.js
import { fork } from 'node:child_process'

export class ProcessPool {
  constructor(file, poolMax) {
    this.file = file
    this.poolMax = poolMax
    this.pool = []
    this.active = []
    this.waiting = []
  }

  //...
```

En la primera parte del módulo, importamos la función `fork()` del módulo `node:child_process`, que usaremos para crear nuevos procesos. Luego, definimos el constructor `ProcessPool`, que acepta un parámetro `file` que representa el programa Node.js a ejecutar y el número máximo de instancias en ejecución en el grupo (`poolMax`). A continuación, definimos tres variables de instancia:
- `pool` es el conjunto de procesos en ejecución listos para ser utilizados.
- `active` contiene la lista de los procesos que se están utilizando actualmente.
- `waiting` contiene una cola de callbacks para todas aquellas solicitudes que no se pudieron cumplir de inmediato debido a la falta de un proceso disponible.

La siguiente pieza de la clase `ProcessPool` es el método `acquire()`, que se encarga de devolver eventualmente un proceso listo para ser utilizado cuando haya uno disponible:

```javascript
  acquire() {
    return new Promise((resolve, reject) => {
      let worker
      if (this.pool.length > 0) { // 1
        worker = this.pool.pop()
        this.active.push(worker)
        return resolve(worker)
      }

      if (this.active.length >= this.poolMax) { // 2
        return this.waiting.push({ resolve, reject })
      }

      worker = fork(this.file) // 3
      worker.once('message', message => {
        if (message === 'ready') {
          this.active.push(worker)
          return resolve(worker)
        }
        worker.kill()
        reject(new Error('Improper process start'))
      })
      worker.once('exit', code => {
        console.log(`Worker exited with code ${code}`)
        this.active = this.active.filter(w => worker !== w)
        this.pool = this.pool.filter(w => worker !== w)
      })
    })
  }
```

La lógica de `acquire()` se explica de la siguiente manera:
1. Si tenemos un proceso en el grupo listo para ser utilizado, lo movemos a la lista activa y luego lo usamos para cumplir la promesa externa con `resolve()`.
2. Si no hay procesos disponibles en el grupo y ya hemos alcanzado el número máximo de procesos en ejecución, debemos esperar a que haya uno disponible. Lo logramos encolando los callbacks `resolve()` y `reject()` de la promesa externa, para su uso posterior.
3. Si aún no hemos alcanzado el número máximo de procesos en ejecución, creamos uno nuevo usando `child_process.fork()`. Luego, esperamos el mensaje `ready` proveniente del proceso recién lanzado, que indica que el proceso ha comenzado y está listo para aceptar nuevos trabajos. Este canal basado en mensajes se proporciona automáticamente con todos los procesos iniciados con `child_process.fork()`.

El último método de la clase `ProcessPool` es `release()`, cuyo propósito es volver a colocar un proceso en el grupo una vez que hayamos terminado con él:

```javascript
  release(worker) {
    if (this.waiting.length > 0) { // 1
      const { resolve } = this.waiting.shift()
      return resolve(worker)
    }
    this.active = this.active.filter(w => worker !== w) // 2
    this.pool.push(worker)
  }
```

Así es como funciona el método `release()`:
1. Si hay una solicitud en la lista de espera, simplemente reasignamos el worker que estamos liberando pasándolo al callback `resolve()` al principio de la cola de espera.
2. De lo contrario, eliminamos el worker que estamos liberando de la lista activa y lo volvemos a colocar en el grupo (`pool`).

Como podemos ver, los procesos nunca se detienen, sino que simplemente se reasignan, lo que nos permite ahorrar tiempo al no reiniciarlos en cada solicitud. Sin embargo, es importante observar que esta podría no ser siempre la mejor opción, y esto depende en gran medida de los requisitos de tu aplicación.

Otros posibles ajustes para reducir el uso de memoria a largo plazo y agregar resiliencia a nuestro grupo de procesos son:
- Terminar los procesos inactivos para liberar memoria después de una cierta cantidad de inactividad.
- Agregar un mecanismo para finalizar procesos que no responden o reiniciar aquellos que se han bloqueado (*crashed*).

En este ejemplo, mantendremos la implementación de nuestro pool de procesos simple y fácil de entender.

##### Comunicación con un proceso hijo

Ahora que nuestra clase `ProcessPool` está lista, podemos usarla para implementar la clase `SubsetSumFork`, cuya función es comunicarse con el worker y reenviar los resultados que produce. Como ya mencionamos, iniciar un proceso con `child_process.fork()` también nos proporciona un canal de comunicación simple basado en mensajes, así que veamos cómo funciona esto implementando el módulo `subsetSumFork.js`:

```javascript
// subsetSumFork.js
import { EventEmitter } from 'node:events'
import { join } from 'node:path'
import { ProcessPool } from './processPool.js'

const workerFile = join(
  import.meta.dirname,
  'workers',
  'subsetSumProcessWorker.js'
)
const workers = new ProcessPool(workerFile, 2)

export class SubsetSum extends EventEmitter {
  constructor(sum, set) {
    super()
    this.sum = sum
    this.set = set
  }

  async start() {
    const worker = await workers.acquire() // 1
    worker.send({ sum: this.sum, set: this.set })

    const onMessage = msg => {
      if (msg.event === 'end') { // 3
        worker.removeListener('message', onMessage)
        workers.release(worker)
      }

      this.emit(msg.event, msg.data) // 4
    }

    worker.on('message', onMessage) // 2
  }
}
```

Lo primero a tener en cuenta es que creamos un nuevo objeto `ProcessPool` usando el archivo `./workers/subsetSumProcessWorker.js` como worker hijo. También establecemos la capacidad máxima del grupo en 2.

Otro punto que vale la pena mencionar es que intentamos mantener la misma API pública de la clase `SubsetSum` original. De hecho, `SubsetSumFork` es un `EventEmitter` cuyo constructor acepta `sum` y `set`, mientras que el método `start()` desencadena la ejecución del algoritmo, que, esta vez, se ejecuta en un proceso separado. Esto es lo que sucede cuando se invoca el método `start()`:
1. Intentamos adquirir un nuevo proceso hijo del grupo. Cuando se completa la operación, usamos inmediatamente el identificador del worker para enviar un mensaje al proceso hijo con los datos del trabajo a ejecutar. La API `send()` es proporcionada automáticamente por Node.js a todos los procesos iniciados con `child_process.fork()`. Este es esencialmente el canal de comunicación del que estábamos hablando.
2. Luego comenzamos a escuchar cualquier mensaje enviado por el proceso de trabajo usando el método `on()` para adjuntar un nuevo escuchador (esto también es parte del canal de comunicación proporcionado por todos los procesos iniciados con `child_process.fork()`).
3. En el escuchador `onMessage`, primero verificamos si recibimos un evento `end`, lo que significa que la tarea `SubsetSum` ha finalizado, en cuyo caso eliminamos el escuchador `onMessage` y liberamos el worker, volviéndolo a colocar en el grupo.
4. El worker produce mensajes en el formato `{event, data}`, lo que nos permite reenviar (*re-emit*) sin problemas cualquier evento producido por el proceso hijo.

Eso es todo para el wrapper `SubsetSumFork`. Implementemos ahora el worker (nuestro proceso hijo).

> [!TIP]
> Es bueno saber que el método `send()` disponible en una instancia de proceso hijo también se puede utilizar para propagar un descriptor de socket desde la aplicación principal a un proceso hijo (consulta la documentación en [nodejsdp.link/childprocess-send](https://nodejsdp.link/childprocess-send)). Esta es la técnica utilizada por el módulo `cluster` para distribuir la carga de un servidor HTTP a través de múltiples procesos. Veremos esto con más detalle en el próximo capítulo.

##### Implementar el worker

Creemos ahora el módulo `workers/subsetSumProcessWorker.js`, nuestro proceso de trabajo:

```javascript
// workers/subsetSumProcessWorker.js
import { SubsetSum } from '../subsetSum.js'

process.on('message', msg => { // 1
  const subsetSum = new SubsetSum(msg.sum, msg.set)

  subsetSum.on('match', data => { // 2
    process.send({ event: 'match', data: data })
  })

  subsetSum.on('end', data => {
    process.send({ event: 'end', data: data })
  })

  subsetSum.start()
})

process.send('ready')
```

Podemos ver de inmediato que estamos reutilizando el `SubsetSum` original (y sincrónico) tal como está. Ahora que estamos en un proceso separado, ya no tenemos que preocuparnos por bloquear el bucle de eventos; todas las solicitudes HTTP continuarán siendo manejadas por el bucle de eventos de la aplicación principal sin interrupciones.

Cuando el worker se inicia como un proceso hijo, esto es lo que sucede:
1. Inmediatamente comienza a escuchar los mensajes provenientes del proceso padre. Esto se puede hacer fácilmente con la función `process.on()` (una parte de la API de comunicación proporcionada cuando el proceso se inicia con `child_process.fork()`). El único mensaje que esperamos del proceso padre es el que proporciona la entrada a una nueva tarea `SubsetSum`. Tan pronto como se recibe dicho mensaje, creamos una nueva instancia de la clase `SubsetSum` y registramos los escuchadores para los eventos `match` y `end`. Por último, comenzamos el cómputo con `subsetSum.start()`.
2. Cada vez que el algoritmo en ejecución encuentra una solución (`match`), la envolvemos en un objeto que tiene el formato `{event, data}` y la enviamos al proceso padre. Estos mensajes luego se manejan en el módulo `subsetSumFork.js`, como hemos visto en la sección anterior.

Como podemos ver, solo tuvimos que envolver el algoritmo que ya creamos, sin modificar sus componentes internos. Esto muestra claramente que cualquier parte de una aplicación se puede colocar fácilmente en un proceso externo simplemente utilizando la técnica que acabamos de ver.

Cuando el proceso hijo no es un programa Node.js, el canal de comunicación simple que acabamos de describir (`on()`, `send()`) no está disponible. En estas situaciones, aún podemos establecer una interfaz con el proceso hijo implementando nuestro propio protocolo sobre los flujos de entrada estándar (*standard input*) y salida estándar (*standard output*), que se exponen al proceso padre. Para obtener más información sobre todas las capacidades de la API `child_process`, puedes consultar la documentación oficial de Node.js en [nodejsdp.link/child_process](https://nodejsdp.link/child_process).

##### Consideraciones para el enfoque multiproceso

Como siempre, para probar esta nueva versión del algoritmo de suma de subconjuntos, simplemente debemos reemplazar el módulo utilizado por el servidor HTTP (el archivo `index.js`):

```javascript
import { createServer } from 'node:http'
// import { SubsetSum } from './subsetSum.js'
// import { SubsetSum } from './subsetSumDefer.js'
import { SubsetSum } from './subsetSumFork.js'

createServer((req, res) => {
  //...
```

Ahora podemos iniciar el servidor nuevamente e intentar enviar una solicitud de muestra:

```bash
curl -G http://localhost:8000/subsetSum --data-urlencode "data=[16, 19,1,1,-16,9,1,-5,-2,17,-15,-97,19,-16,-4,-5,15]" --data-urlencode "sum=0"
```

Al igual que en el enfoque de intercalado que vimos anteriormente, con esta nueva versión del módulo `SubsetSum`, el bucle de eventos no se bloquea mientras se ejecuta la tarea intensiva de CPU. Esto se puede confirmar enviando otra solicitud concurrente, de la siguiente manera:

```bash
curl -G http://localhost:8000
```

El comando anterior debería devolver inmediatamente el texto `I'm alive!`.

Aún más interesante, podemos intentar iniciar dos tareas `SubsetSum` al mismo tiempo. Si tu sistema tiene más de un procesador, verás que cada tarea se ejecuta en un núcleo de CPU diferente, aprovechando al máximo el hardware.

Si luego intentamos ejecutar tres tareas `SubsetSum` simultáneamente, la tercera tarea no comenzará de inmediato. Esto no se debe a que el bucle de eventos del proceso principal esté bloqueado, sino a que hemos establecido un límite de concurrencia de dos procesos para la tarea `SubsetSum`. Como resultado, la tercera solicitud se encolará y solo comenzará una vez que uno de los dos procesos en ejecución esté disponible.

Node.js proporciona una forma de obtener información sobre los núcleos de CPU disponibles en la máquina actual. Puedes hacer esto utilizando el módulo `node:os`, y específicamente la función `cpus()`, que devuelve un array de objetos que describen cada núcleo lógico de CPU. Esta información puede ser útil para establecer un límite de concurrencia predeterminado óptimo siempre que estés trabajando en tareas que impliquen la creación de múltiples procesos, como estamos haciendo con nuestra implementación actual de `SubsetSum`. Por ejemplo:

```javascript
import { cpus } from 'node:os'

console.log(cpus().length) // prints the number of logical CPU cores in the system
```

Como vimos, el enfoque multiproceso tiene muchas ventajas en comparación con el enfoque de intercalado. Primero, no introduce ninguna penalización computacional al ejecutar el algoritmo. En segundo lugar, puede aprovechar al máximo una máquina con múltiples procesadores.

Ahora, veamos un enfoque alternativo que utiliza hilos (*threads*) en lugar de procesos.

---

#### Uso de worker threads

Desde Node.js 10.5.0, tenemos un nuevo mecanismo para ejecutar algoritmos intensivos de CPU fuera del bucle de eventos principal llamado **worker threads**. Los worker threads pueden verse como una alternativa ligera a `child_process.fork()` con algunas ventajas adicionales. En comparación con los procesos, los worker threads tienen una huella de memoria más pequeña y un tiempo de inicio más rápido, ya que se ejecutan dentro del proceso principal pero dentro de diferentes hilos.

Los worker threads son esencialmente hilos que, por defecto, no comparten nada con el hilo principal de la aplicación; se ejecutan dentro de su propia instancia de V8, con un entorno de ejecución de Node.js y un bucle de eventos independientes. La comunicación con el hilo principal es posible gracias a canales de comunicación basados en mensajes, la transferencia de objetos `ArrayBuffer` y el uso de objetos `SharedArrayBuffer` cuya sincronización es administrada por el usuario (generalmente con la ayuda de `Atomics`).

Puedes leer más sobre `SharedArrayBuffer` y `Atomics` en el siguiente artículo: [nodejsdp.link/shared-array-buffer](https://nodejsdp.link/shared-array-buffer). Aunque el artículo se centra en los web workers, muchos conceptos son idénticos a los worker threads de Node.js.

Este amplio nivel de aislamiento de los worker threads del hilo principal preserva la integridad del lenguaje. Al mismo tiempo, las facilidades de comunicación básicas y las capacidades de intercambio de datos son más que suficientes para el 99% de los casos de uso.

Ahora, usemos worker threads en nuestro ejemplo de `SubsetSum`.

##### Ejecutar la tarea de suma de subconjuntos en un worker thread

La API de worker threads tiene mucho en común con la de `ChildProcess`, por lo que los cambios en nuestro código serán mínimos.

Primero, necesitamos crear una nueva clase llamada `ThreadPool`, que es nuestro `ProcessPool` adaptado para operar con worker threads en lugar de procesos. El siguiente código muestra las diferencias entre la nueva clase `ThreadPool` y la clase `ProcessPool`. Solo hay unas pocas diferencias en el método `acquire()`; el resto del código es idéntico:

```javascript
// threadPool.js
import { Worker } from 'node:worker_threads'

export class ThreadPool {
  constructor(file, poolMax) {
    this.file = file
    this.poolMax = poolMax
    this.pool = []
    this.active = []
    this.waiting = []
  }

  acquire() {
    return new Promise((resolve, reject) => {
      let worker
      if (this.pool.length > 0) {
        worker = this.pool.pop()
        this.active.push(worker)
        return resolve(worker)
      }

      if (this.active.length >= this.poolMax) {
        return this.waiting.push({ resolve, reject })
      }

      worker = new Worker(this.file)
      worker.once('online', () => {
        this.active.push(worker)
        resolve(worker)
      })
      worker.once('exit', code => {
        console.log(`Worker exited with code ${code}`)
        this.active = this.active.filter(w => worker !== w)
        this.pool = this.pool.filter(w => worker !== w)
      })
    })
  }

  release(worker) {
    if (this.waiting.length > 0) {
      const { resolve } = this.waiting.shift()
      return resolve(worker)
    }
    this.active = this.active.filter(w => worker !== w)
    this.pool.push(worker)
  }
}
```

A continuación, debemos adaptar el worker y colocarlo en un nuevo archivo llamado `subsetSumThreadWorker.js`. La principal diferencia con nuestro antiguo worker es que en lugar de usar `process.send()` y `process.on()`, tendremos que usar `parentPort.postMessage()` y `parentPort.on()`:

```javascript
// workers/subsetSumThreadWorker.js
import { parentPort } from 'node:worker_threads'
import { SubsetSum } from '../subsetSum.js'

parentPort.on('message', msg => {
  const subsetSum = new SubsetSum(msg.sum, msg.set)

  subsetSum.on('match', data => {
    parentPort.postMessage({ event: 'match', data: data })
  })

  subsetSum.on('end', data => {
    parentPort.postMessage({ event: 'end', data: data })
  })

  subsetSum.start()
})
```

De manera similar, el módulo `subsetSumThreads.js` es esencialmente el mismo que el módulo `subsetSumFork.js` excepto por un par de líneas de código:

```javascript
// subsetSumThreads.js
import { EventEmitter } from 'node:events'
import { join } from 'node:path'
import { ThreadPool } from './threadPool.js'

const workerFile = join(
  import.meta.dirname,
  'workers',
  'subsetSumThreadWorker.js'
)
const workers = new ThreadPool(workerFile, 2)

export class SubsetSum extends EventEmitter {
  constructor(sum, set) {
    super()
    this.sum = sum
    this.set = set
  }

  async start() {
    const worker = await workers.acquire()
    worker.postMessage({ sum: this.sum, set: this.set })

    const onMessage = msg => {
      if (msg.event === 'end') {
        worker.removeListener('message', onMessage)
        workers.release(worker)
      }

      this.emit(msg.event, msg.data)
    }

    worker.on('message', onMessage)
  }
}
```

Como hemos visto, adaptar una aplicación existente para usar worker threads en lugar de procesos bifurcados (*forked processes*) es una operación trivial. Esto se debe a que la API de los dos componentes es muy similar, pero también a que un worker thread tiene mucho en común con un proceso Node.js completo.

Finalmente, necesitamos actualizar el módulo `index.js` para que pueda usar el nuevo módulo `subsetSumThreads.js`, como hemos visto para las otras implementaciones del algoritmo:

```javascript
import { createServer } from 'node:http'
// import { SubsetSum } from './subsetSum.js'
// import { SubsetSum } from './subsetSumDefer.js'
// import { SubsetSum } from './subsetSumFork.js'
import { SubsetSum } from './subsetSumThreads.js'

createServer((req, res) => {
  // ...
```

Ahora, puedes probar la nueva versión del servidor de suma de subconjuntos utilizando worker threads. Al igual que en las dos implementaciones anteriores, el bucle de eventos de la aplicación principal no se bloquea con el algoritmo de suma de subconjuntos, ya que se ejecuta en un hilo separado.

El ejemplo que hemos visto utiliza solo un pequeño subconjunto de todas las capacidades que ofrecen los worker threads. Para temas más avanzados, como la transferencia de objetos `ArrayBuffer` u objetos `SharedArrayBuffer`, puedes leer la documentación oficial de la API en [nodejsdp.link/worker-threads](https://nodejsdp.link/worker-threads).

#### Ejecución de tareas intensivas de CPU en producción

Los ejemplos que hemos visto hasta ahora deberían darte una idea de las herramientas a nuestra disposición para ejecutar operaciones intensivas de CPU en Node.js. Sin embargo, componentes como los pools de procesos y los pools de hilos son piezas complejas de maquinaria que requieren mecanismos adecuados para hacer frente a tiempos de espera, errores y otros tipos de fallos, que, por brevedad, dejamos fuera de nuestra implementación. Por lo tanto, a menos que tengas requisitos especiales, es mejor confiar en bibliotecas más probadas en batalla para su uso en producción. Dos de esas bibliotecas son **workerpool** ([nodejsdp.link/workerpool](https://nodejsdp.link/workerpool)) y **piscina** ([nodejsdp.link/piscina](https://nodejsdp.link/piscina)), que se basan en los mismos conceptos que hemos visto en esta sección. Nos permiten coordinar la ejecución de tareas intensivas de CPU utilizando procesos externos o worker threads.

Una última observación es que debemos considerar que si tenemos algoritmos particularmente complejos para ejecutar o si la cantidad de tareas intensivas de CPU excede la capacidad de un solo nodo, es posible que tengamos que pensar en **escalar horizontalmente** el cómputo a través de múltiples nodos. Este es un problema completamente diferente, y aprenderemos más sobre esto en los próximos dos capítulos.

---

### Sección 5: Resumen

Este capítulo agregó algunas herramientas nuevas y excelentes a nuestro cinturón, y como puedes ver, nuestro viaje se está enfocando más en problemas avanzados. Debido a esto, hemos comenzado a profundizar en soluciones más complejas. Este capítulo nos brindó no solo un conjunto de recetas para reutilizar y personalizar según nuestras necesidades, sino también excelentes demostraciones de cómo dominar unos pocos principios y patrones puede ayudarnos a abordar los problemas más complejos en el desarrollo de Node.js.

Los próximos dos capítulos representan la cúspide de nuestro viaje. Después de estudiar las diversas tácticas del desarrollo de Node.js, ahora estamos listos para pasar a las estrategias y explorar los patrones arquitectónicos para escalar y distribuir nuestras aplicaciones Node.js.

---

### Sección 6: Ejercicios

- **11.1 Proxy con colas de pre-inicialización:** Usando un Proxy de JavaScript, crea un wrapper para agregar colas de pre-inicialización a cualquier objeto. Debes permitir que el consumidor del wrapper decida qué métodos aumentar y el nombre de la propiedad/evento que indica si el componente está inicializado.
- **11.2 Procesamiento por lotes y almacenamiento en caché con callbacks:** Implementa el procesamiento por lotes y el almacenamiento en caché para los ejemplos de la API `totalSales` utilizando únicamente callbacks, streams y eventos (sin utilizar promesas ni async/await). Pista: ¡Presta atención a Zalgo al devolver valores almacenados en caché!
- **11.3 Cancelable asíncrono profundo:** Extiende la función `createAsyncCancelable()` para que sea posible invocar otras funciones cancelables desde dentro de la función cancelable principal. Cancelar la operación principal también debería cancelar todas las operaciones anidadas. Pista: Permite producir (*yield*) el resultado de un `asyncCancelable()` desde dentro de la función generadora.
- **11.4 Granja de cálculo (Compute farm):** Crea un servidor HTTP con un endpoint `POST` que reciba, como entrada, el código de una función (como una cadena) y un array de argumentos, ejecute la función con los argumentos dados en un worker thread o en un proceso separado, y devuelva el resultado al cliente. Pista: Puedes usar `eval()`, `vm.runInContext()`, o ninguno de los dos. Nota: Independientemente del código que produzcas para este ejercicio, ten en cuenta que permitir que los usuarios ejecuten código arbitrario en un entorno de producción puede plantear graves riesgos de seguridad, y nunca debes hacerlo a menos que sepas exactamente cuáles son las implicaciones.

