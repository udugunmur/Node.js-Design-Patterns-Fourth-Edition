# Parte 2: Patrones de diseño de Node.js

## Capítulo 9: Patrones de diseño de comportamiento

En los dos últimos capítulos, aprendimos sobre patrones que pueden ayudarnos en la creación de objetos y en la construcción de estructuras de objetos complejas. Ahora, estamos listos para cambiar nuestro enfoque hacia otro aspecto crítico del diseño de software: el comportamiento.

En este capítulo, aprenderemos cómo interactúan los objetos, cómo se pueden combinar y cómo se puede estructurar su comunicación para producir sistemas extensibles, modulares, reutilizables y adaptables. Abordaremos preguntas como: "¿Cómo podemos modificar partes de un algoritmo en tiempo de ejecución?", "¿Cómo puede un objeto cambiar su comportamiento según su estado?", y "¿Cómo podemos iterar sobre una colección sin conocer su implementación subyacente?". Estos son los tipos de desafíos que los patrones de diseño de comportamiento ayudan a resolver.

Ya hemos encontrado a un miembro notable de esta categoría de patrones, y ese es el patrón Observer, que presentamos en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos. El patrón Observer es uno de los patrones fundamentales de la plataforma Node.js, ya que nos proporciona una interfaz simple para tratar con eventos y suscripciones, que son la fuerza vital de la arquitectura basada en eventos de Node.

Si ya estás familiarizado con los patrones de diseño de la Banda de los Cuatro (*Gang of Four* o GoF), en este capítulo verás, una vez más, cómo la implementación de algunos de estos patrones puede ser radicalmente diferente en JavaScript en comparación con lenguajes puramente orientados a objetos como Java o C++. Un ejemplo sorprendente es el patrón Iterator, que exploraremos más adelante en el capítulo. A diferencia de la POO clásica, donde implementar un iterador a menudo requiere definir interfaces, extender clases y definir jerarquías complejas, JavaScript nos permite lograr la misma funcionalidad simplemente agregando un método especial a una clase o directamente a un objeto.

Además, un patrón cubierto en este capítulo, el patrón Middleware, se asemeja mucho al patrón Chain of Responsibility de la GoF. Sin embargo, su adopción generalizada en Node.js lo ha hecho tan fundamental que a menudo se considera un patrón distinto por derecho propio.

Ahora, es momento de arremangarse y ponerse manos a la obra con algunos patrones de diseño de comportamiento. En este capítulo, aprenderás sobre lo siguiente:
- El patrón **Strategy**, que nos ayuda a cambiar partes de un componente para adaptarlo a necesidades específicas.
- El patrón **State**, que nos permite cambiar el comportamiento de un componente en función de su estado.
- El patrón **Template**, que nos permite reutilizar la estructura de un componente para definir otros nuevos.
- El patrón **Iterator**, que nos proporciona una interfaz común para iterar sobre una colección.
- El patrón **Middleware**, que nos permite definir una cadena modular de pasos de procesamiento.
- El patrón **Command**, que materializa la información necesaria para ejecutar una rutina, permitiendo que dicha información se transfiera, almacene y procese fácilmente.

---

### Sección 1: Strategy (Estrategia)

El patrón **Strategy** permite que un objeto, conocido como el **contexto**, adapte su comportamiento delegando partes específicas de su lógica a objetos intercambiables llamados **estrategias**. El contexto define la estructura común de un algoritmo, mientras que cada estrategia encapsula una variación de sus aspectos mutables. Este enfoque permite al contexto modificar su comportamiento dinámicamente en función de factores como valores de entrada, configuraciones del sistema o preferencias del usuario.

He aquí una historia de la vida real (de Mario) sobre cómo el patrón Strategy me ayudó a optimizar una operación crítica para el rendimiento. Si bien este ejemplo no está directamente relacionado con Node.js, demuestra cuán fundamental y universal es este patrón.

> Hace años, estaba trabajando en un motor de juegos 3D como proyecto personal. Los motores de juegos son notoriamente intensivos en recursos, donde cada optimización cuenta para garantizar un rendimiento fluido. Un componente clave del renderizado 3D es la multiplicación de matrices y, en ese momento (a principios de la década de 2000), los fabricantes de CPU como Intel y AMD estaban introduciendo instrucciones especializadas para acelerar las operaciones matemáticas a través del paralelismo. Estas instrucciones SIMD (*single instruction, multiple data*) eran ideales para acelerar las multiplicaciones de matrices, pero había un inconveniente: diferentes procesadores admitían diferentes conjuntos de instrucciones, principalmente 3DNow! en AMD y SSE en Intel.
>
> Para maximizar el rendimiento en todos los procesadores convencionales, mi motor necesitaba admitir múltiples conjuntos de instrucciones. El desafío, sin embargo, era que aprovechar estas instrucciones requería escribir código ensamblador de bajo nivel. Mi enfoque inicial se parecía a esto (en pseudocódigo):

```javascript
function matrixMultiply(data) {
  if (is3DNowAvailable) {
    return matrixMultiply3DNow(data)
  }
  if (isSSEAvailable) {
    return matrixMultiplySSE(data)
  }
  return slowMatrixMultiplyFallback(data)
}
```

> A primera vista, parecía una solución razonable. Sin embargo, había un problema crítico: la función `matrixMultiply` se llamaba decenas de miles de veces por segundo. Las comprobaciones condicionales en cada llamada introducían una sobrecarga innecesaria, lejos de ser ideal cuando se optimiza para lograr la máxima eficiencia.
>
> Mientras buscaba un mejor enfoque, descubrí el patrón Strategy. El concepto de "definir una familia de algoritmos y hacerlos intercambiables" era exactamente lo que necesitaba. Refactoricé mi implementación para seleccionar la función óptima una vez al inicio:

```javascript
matrixMultiply = null
if (is3DNowAvailable) {
  matrixMultiply = matrixMultiply3DNow
} else if (isSSEAvailable) {
  matrixMultiply = matrixMultiplySSE
} else {
  matrixMultiply = matrixMultiplyFallback
}
```

> Con este enfoque, el algoritmo correcto se elegía solo una vez durante la inicialización, eliminando la necesidad de comprobaciones condicionales repetidas. La ganancia de rendimiento fue inmediata y me sorprendió la elegancia de la solución. No podía creer que no se me hubiera ocurrido antes, pero ese es el tipo de perspectiva que aporta la experiencia.
>
> Desde entonces, el patrón Strategy se ha convertido en una de mis técnicas de referencia, demostrando ser invaluable en muchos proyectos diferentes.
>
> Como nota al margen, es fascinante que, dos décadas después, las multiplicaciones de matrices sigan a la vanguardia de la innovación de hardware, impulsando todo, desde redes neuronales hasta aplicaciones modernas de IA.

Las estrategias suelen formar parte de una familia de soluciones y todas ellas implementan la misma interfaz esperada por el contexto. La siguiente figura muestra la situación que acabamos de describir:

**Figura 9.1:** Estructura general del patrón Strategy.

La Figura 9.1 te muestra cómo el objeto de contexto puede conectar diferentes estrategias en su estructura como si fueran partes reemplazables de una pieza de maquinaria. Imagina un coche; sus neumáticos pueden considerarse su estrategia para adaptarse a las diferentes condiciones de la carretera. Podemos poner neumáticos de invierno para circular por carreteras nevadas gracias a sus clavos, mientras que podemos decidir poner neumáticos de alto rendimiento para viajar principalmente por autopistas en un viaje largo. Por un lado, no queremos cambiar todo el coche para que esto sea posible y, por otro, no queremos un coche con ocho ruedas para que pueda circular por todas las carreteras posibles.

Comprendemos rápidamente lo poderoso que es este patrón. No solo ayuda a separar las responsabilidades dentro de un problema dado, sino que también permite que nuestra solución tenga una mayor flexibilidad y se adapte a diferentes variaciones del mismo problema.

El patrón Strategy es particularmente útil en todas aquellas situaciones en las que soportar variaciones en el comportamiento de un componente requiere una lógica condicional compleja (muchas sentencias `if...else` o `switch`) o mezclar diferentes componentes de la misma familia.

He aquí otro ejemplo: imagina un objeto llamado `Order` que representa un pedido online en un sitio web de comercio electrónico. El objeto tiene un método llamado `pay()` que, como dice, finaliza el pedido y transfiere los fondos del usuario a la tienda online.

Para admitir diferentes sistemas de pago, tenemos un par de opciones:
- Utilizar una sentencia `if...else` en el método `pay()` para completar la operación en función de la opción de pago elegida.
- Delegar la lógica del pago a un objeto de estrategia que implemente la lógica para la pasarela de pago específica seleccionada por el usuario.

En la primera solución, nuestro objeto `Order` no puede admitir otros métodos de pago a menos que se modifique su código. Además, esto puede volverse bastante complejo cuando crece el número de opciones de pago. En su lugar, el uso del patrón Strategy permite que el objeto `Order` admita un número virtualmente ilimitado de métodos de pago y mantiene su alcance limitado a gestionar únicamente los detalles del usuario, los artículos comprados y el precio relativo, mientras delega el trabajo de completar el pago a otro objeto.

Demostremos ahora este patrón con un ejemplo simple y realista.

#### Objetos de configuración multiformato

Consideremos un objeto llamado `Config` que contiene un conjunto de parámetros de configuración utilizados por un servidor de aplicaciones: el puerto y la dirección de escucha del servidor, la configuración del tiempo de espera (*timeout*), las variables de entorno, etc.

Para hacer este ejemplo un poco más estricto, vamos a utilizar TypeScript para definir cómo se ve la forma del objeto de configuración:

```typescript
// configData.ts
export type ConfigData = {
  listen: {
    port: number
    host: string
  }
  timeouts: {
    headersTimeoutMs: number
    keepAliveTimeoutMs: number
    requestTimeoutMs: number
  }
  env: Record<string, string>
}
```

Si prefieres no usar TypeScript, simplemente puedes ignorar las definiciones de tipos y anotaciones; el código funcionará de la misma manera. El principal beneficio de usar TypeScript en este contexto es que proporciona seguridad de tipos, lo que garantiza que una vez que se carga el objeto de configuración, solo puedas acceder y modificar las opciones de configuración admitidas. Esto ayuda a prevenir errores y mejora la mantenibilidad.

El objeto `Config` debe ser capaz de proporcionar una interfaz simple para acceder a estos parámetros, pero también una forma de importar y exportar la configuración utilizando almacenamiento persistente, como un archivo. Queremos poder admitir diferentes formatos para almacenar la configuración, por ejemplo, JSON, YAML o TOML.

Aplicando lo que aprendimos sobre el patrón Strategy, podemos identificar inmediatamente la parte variable del objeto `Config`, que es la funcionalidad que nos permite serializar y deserializar la configuración. Esto será implementado por nuestras estrategias.

Creemos un nuevo módulo llamado `config.ts` y definamos la parte genérica de nuestro gestor de configuración:

```typescript
import { readFile, writeFile } from 'node:fs/promises'
import type { FormatStrategy} from './strategies.ts'
import type { ConfigData } from './configData.ts'

export class Config {
  data?: ConfigData
  formatStrategy: FormatStrategy

  constructor(formatStrategy: FormatStrategy) { // 1
    this.data = undefined
    this.formatStrategy = formatStrategy
  }

  async load(filePath: string): Promise<void> { // 2
    console.log(`Deserializing from ${filePath}`)
    this.data = this.formatStrategy.deserialize(
      await readFile(filePath, 'utf-8')
    )
  }

  async save(filePath: string): Promise<void> { // 3
    if (!this.data) {
      throw new Error('No data to save')
    }
    console.log(`Serializing to ${filePath}`)
    await writeFile(filePath, this.formatStrategy.serialize(this.data))
  }
}
```

Esto es lo que sucede en el código anterior:
1. En el constructor, creamos una variable de instancia llamada `data` para almacenar los datos de configuración (inicialmente `undefined`). Esta variable contendrá la configuración una vez cargada desde un archivo y, por lo tanto, tiene el tipo `ConfigData`. Luego, también almacenamos `formatStrategy`, que representa el componente que utilizaremos para analizar y serializar los datos. Esta es la parte variable de nuestro módulo y debe pasarse al constructor. Ten en cuenta que esto está tipado como `FormatStrategy`; lo definiremos en breve.
2. Proporcionamos un método llamado `load()` que nos permite leer el contenido de un archivo de configuración determinado y analizarlo con nuestra estrategia actual. Una vez que se completa el análisis, el objeto resultante se almacenará en la variable de instancia `data`.
3. De manera similar, proporcionamos un método `save()` que, si se han cargado datos de configuración (y potencialmente modificado), nos permite persistir estos cambios en una ruta de archivo dada utilizando nuestra estrategia de configuración actual para la serialización.

Como podemos ver, este diseño tan simple y limpio permite que el objeto `Config` admita sin problemas diferentes formatos de archivo al cargar y guardar sus datos. La mejor parte es que la lógica para admitir esos diversos formatos no está codificada rígidamente (*hardcoded*) en ninguna parte, por lo que la clase `Config` puede adaptarse sin ninguna modificación a prácticamente cualquier formato de archivo, dada la estrategia correcta.

Para demostrar esta característica, creemos ahora un par de estrategias de formato en un archivo llamado `strategies.ts`.

Por simplicidad, estamos definiendo todas las estrategias para analizar archivos de configuración (JSON, YAML y TOML) dentro de un solo archivo. Sin embargo, en un escenario del mundo real donde se puedan introducir formatos adicionales con el tiempo, sería más práctico organizar la estrategia de cada formato en su propio archivo dedicado. Este enfoque mejora la mantenibilidad y facilita la extensión del sistema según sea necesario.

Comencemos proporcionando una definición de TypeScript para cómo se vería una `FormatStrategy` válida:

```typescript
// strategies.ts
export type FormatStrategy = {
  deserialize: (data: string) => ConfigData
  serialize: (data: ConfigData) => string
}
```

Cualquier objeto que implemente estos dos métodos califica como una estrategia de formato:
- `deserialize()`: Convierte una cadena de configuración sin procesar en un objeto `ConfigData` estructurado.
- `serialize()`: Realiza la operación inversa, transformando un objeto `ConfigData` en su representación de cadena serializada.

En este punto, estamos listos para implementar nuestra primera estrategia. Comencemos con el formato de archivo JSON:

```typescript
// strategies.ts
// ...
export const jsonStrategy: FormatStrategy = {
  deserialize(data): ConfigData {
    return JSON.parse(data)
  },
  serialize(data: ConfigData): string {
    return JSON.stringify(data, null, 2)
  },
}
```

¡Nada realmente complicado! Nuestra estrategia simplemente implementa la interfaz acordada, de modo que pueda ser utilizada por el objeto `Config`.

De manera similar, podemos implementar estrategias equivalentes para los formatos YAML y TOML. La diferencia clave es que, a diferencia de JSON, estos formatos no son compatibles de forma nativa en la biblioteca estándar de Node.js. Para manejar la serialización y deserialización correctamente para estos formatos, necesitaremos depender de bibliotecas externas:

```typescript
// strategies.ts
import YAML from 'yaml' // v2.7.0
import TOML from 'smol-toml' // v1.3.1
// ...
export const yamlStrategy: FormatStrategy = {
  deserialize(data): ConfigData {
    return YAML.parse(data)
  },
  serialize(data: ConfigData): string {
    return YAML.stringify(data, { indent: 2 })
  },
}

export const tomlStrategy: FormatStrategy = {
  deserialize(data): ConfigData {
    return TOML.parse(data) as ConfigData
  },
  serialize(data: ConfigData): string {
    return TOML.stringify(data)
  },
}
```

Desde una perspectiva de seguridad de tipos, vale la pena señalar que hemos utilizado algunas aserciones de tipo `as ConfigData`. Si bien esto puede ser conveniente, no es el enfoque más seguro, ya que esencialmente le dice a TypeScript: "Confía en mí, estos datos coinciden con el tipo esperado", aunque nada impide que un archivo de configuración se modifique manualmente de una manera que rompa esta suposición. Para mayor seguridad, un mejor enfoque sería validar los datos después de la deserialización para asegurarse de que se adhieran al esquema esperado. Una biblioteca como `zod` ([nodejsdp.link/zod](https://nodejsdp.link/zod)) puede ayudar con esto proporcionando validación en tiempo de ejecución, evitando errores inesperados causados por archivos de configuración mal formados.

También es importante tener en cuenta que nuestra implementación actual no maneja excepciones. Si la deserialización de una cadena falla (por ejemplo, si el archivo dado no se ajusta a la especificación de formato), la excepción de la biblioteca de deserialización específica (JSON, YAML o TOML) simplemente se propagará hacia arriba. Estas excepciones pueden diferir significativamente, lo que hace que el manejo de errores aguas arriba sea más engorroso. Una implementación más robusta podría introducir un `DeserializationError` estándar, obligando a cada formateador a capturar excepciones específicas del formato y lanzar un error unificado en su lugar.

Ahora, para mostrarte cómo se combina todo, creemos un archivo llamado `index.ts` e intentemos cargar y guardar una configuración de muestra utilizando diferentes formatos:

```typescript
import { join } from 'node:path'
import { Config } from './config.ts'
import { jsonStrategy, tomlStrategy, yamlStrategy } from './strategies.ts'

const SAMPLES = join(import.meta.dirname, 'samples')

const jsonConfig = new Config(jsonStrategy)
await jsonConfig.load(join(SAMPLES, 'config.json'))
if (jsonConfig.data?.env) {
  jsonConfig.data.env.NODE_ENV = 'production'
  jsonConfig.data.env.NODE_OPTIONS = '--enable-source-maps'
}
await jsonConfig.save(join(SAMPLES, 'config_mod.json'))

const yamlConfig = new Config(yamlStrategy)
await yamlConfig.load(join(SAMPLES, 'config.yaml'))
if (yamlConfig.data?.env) {
  yamlConfig.data.env.NODE_ENV = 'production'
  yamlConfig.data.env.NODE_OPTIONS = '--enable-source-maps'
}
await yamlConfig.save(join(SAMPLES, 'config_mod.yaml'))

const tomlConfig = new Config(tomlStrategy)
await tomlConfig.load(join(SAMPLES, 'config.toml'))
if (tomlConfig.data?.env) {
  tomlConfig.data.env.NODE_ENV = 'production'
  tomlConfig.data.env.NODE_OPTIONS = '--enable-source-maps'
}
await tomlConfig.save(join(SAMPLES, 'config_mod.toml'))
```

Nuestro módulo de prueba revela las propiedades centrales del patrón Strategy. Definimos solo una clase `Config`, que implementa las partes comunes de nuestro gestor de configuración. Luego, al usar diferentes estrategias para serializar y deserializar datos, creamos diferentes instancias de la clase `Config` que admiten diferentes formatos de archivo.

El ejemplo que acabamos de ver nos mostró solo una de las posibles alternativas que teníamos para seleccionar una estrategia. Otros enfoques válidos podrían haber sido los siguientes:
- **Crear dos familias de estrategias diferentes, una para la deserialización y la otra para la serialización:** Esto habría permitido leer desde un formato y guardar en otro.
- **Seleccionar dinámicamente la estrategia:** Dependiendo de la extensión del archivo proporcionado, el objeto `Config` podría haber mantenido un mapa de extensión -> estrategia y haberlo utilizado para seleccionar el algoritmo correcto para la extensión dada.

Como podemos ver, tenemos varias opciones para seleccionar la estrategia a utilizar, y la correcta solo depende de tus requisitos y del equilibrio (*trade-off*) en términos de características y la simplicidad que deseas obtener.

Además, la implementación del patrón en sí también puede variar mucho. Por ejemplo, en su forma más simple, el contexto y la estrategia pueden ser funciones simples:

```javascript
function context(strategy) {...}
```

Aunque esto pueda parecer insignificante, no debe subestimarse en un lenguaje de programación como JavaScript, donde las funciones son ciudadanos de primera clase y se utilizan tanto como los objetos completos.

Entre todas estas variaciones, sin embargo, lo que no cambia es la idea detrás del patrón; como siempre, la implementación puede cambiar ligeramente, pero los conceptos centrales que impulsan el patrón son siempre los mismos.

La estructura del patrón Strategy puede parecer similar a la del patrón Adapter. Sin embargo, existe una diferencia sustancial entre los dos. El objeto adaptador no añade ningún comportamiento al adaptado; simplemente lo pone a disposición bajo otra interfaz. Esto también puede requerir que se implemente cierta lógica adicional para convertir una interfaz en otra, pero esta lógica se limita solo a esta tarea. En el patrón Strategy, sin embargo, el contexto y la estrategia implementan dos partes diferentes de un algoritmo y, por lo tanto, ambos implementan algún tipo de lógica, y ambos son esenciales para construir el algoritmo final (cuando se combinan).

#### En la práctica

Passport.js ([nodejsdp.link/passportjs](https://nodejsdp.link/passportjs)) es una popular biblioteca de autenticación para Node.js. Está diseñada para simplificar la integración de varios métodos de autenticación en tus aplicaciones web. Emplea hábilmente el patrón Strategy para ofrecer un sistema de autenticación flexible y extensible. En su núcleo, Passport.js separa el flujo de trabajo de autenticación principal (gestión de sesiones y datos de usuario) de los pasos específicos requeridos para los diferentes métodos de autenticación. Por ejemplo, una estrategia podría aprovechar OAuth para recuperar un token de acceso para acceder a perfiles de Facebook o LinkedIn. Otra podría simplemente validar una combinación de usuario/contraseña contra una base de datos local. Passport.js trata a cada uno de estos como una estrategia distinta para lograr la autenticación.

Este diseño permite a Passport.js admitir una amplia gama de enfoques de autenticación. De hecho, al momento de escribir este artículo, su sitio web enumera más de 500 estrategias diferentes ([nodejsdp.link/passport-strategies](https://nodejsdp.link/passport-strategies)), cada una disponible en npm como un plugin instalable. Esto facilita la adición de nuevos métodos de autenticación a tu aplicación sin modificar la biblioteca central de Passport.js.

---

### Sección 2: State (Estado)

El patrón **State** es una especialización del patrón Strategy donde la estrategia cambia según el estado del contexto.

Hemos visto en la sección anterior cómo se puede seleccionar una estrategia en función de diferentes variables, como una propiedad de configuración o un parámetro de entrada, y una vez realizada esta selección, la estrategia permanece sin cambios durante el resto de la vida útil del objeto de contexto. En el patrón State, en cambio, la estrategia (también llamada el **estado** en esta circunstancia) es dinámica y puede cambiar durante la vida útil del contexto, lo que permite que su comportamiento se adapte en función de su estado interno.

La siguiente figura nos muestra una representación del patrón:

**Figura 9.2:** El patrón State.

La Figura 9.2 muestra cómo un objeto de contexto pasa por tres estados (A, B y C). Con el patrón State, en cada estado de contexto diferente, seleccionamos una estrategia diferente. Esto significa que el objeto de contexto adoptará un comportamiento diferente en función del estado en el que se encuentre.

Para que esto sea más fácil de entender, consideremos un ejemplo: imagina que tenemos un sistema de reservas de hotel y un objeto llamado `Reservation` que modela la reserva de una habitación. Esta es una situación típica en la que debemos adaptar el comportamiento de un objeto en función de su estado.

Considera la siguiente serie de eventos:
- Cuando la reserva se crea inicialmente, el usuario puede confirmar (usando un método llamado `confirm()`) la reserva. Por supuesto, no pueden cancelarla (usando `cancel()`), porque todavía no está confirmada (el invocador recibiría una excepción, por ejemplo). Sin embargo, pueden eliminarla (usando `delete()`) si cambian de opinión antes de comprar.
- Una vez confirmada la reserva, usar el método `confirm()` de nuevo no tiene ningún sentido; sin embargo, ahora debería ser posible cancelar la reserva pero ya no eliminarla, porque debe conservarse para los registros.
- El día antes de la fecha de la reserva, ya no debería ser posible cancelar la reserva; es demasiado tarde para eso.

Ahora, imagina que tenemos que implementar el sistema de reservas que acabamos de describir en un solo objeto monolítico. Ya podemos imaginarnos todas las sentencias `if...else` o `switch` que tendríamos que escribir para habilitar/deshabilitar cada acción según el estado de la reserva.

**Figura 9.3:** Un ejemplo de aplicación del patrón State.

Como se ilustra en la Figura 9.3, el patrón State es perfecto en esta situación: habría tres estrategias, todas implementando los tres métodos descritos (`confirm()`, `cancel()` y `delete()`) y cada una implementando solo un comportamiento: el correspondiente al estado modelado. Al utilizar este patrón, debería ser muy fácil para el objeto `Reservation` cambiar de un comportamiento a otro; esto simplemente requeriría la activación de una estrategia diferente (objeto de estado) en cada cambio de estado.

La transición de estado puede ser iniciada y controlada por el objeto de contexto, por el código del cliente o por los propios objetos de estado. Esta última opción suele proporcionar los mejores resultados en términos de flexibilidad y desacoplamiento, ya que el contexto no tiene que conocer todos los estados posibles ni cómo realizar la transición entre ellos.

Trabajemos ahora en un ejemplo más concreto para que podamos aplicar lo que aprendimos sobre el patrón State.

#### Implementación de un socket básico a prueba de fallos (failsafe socket)

Construyamos un socket de cliente TCP que no falle cuando se pierda la conexión con el servidor; en su lugar, queremos poner en cola todos los datos enviados durante el tiempo en que el servidor esté fuera de línea y luego intentar enviarlos nuevamente tan pronto como se restablezca la conexión. Queremos aprovechar este socket en el contexto de un sistema de monitorización simple, donde un conjunto de máquinas envía algunas estadísticas sobre su utilización de recursos a intervalos regulares. Si el servidor que recopila estos recursos se cae, nuestro socket continuará encolando los datos localmente hasta que el servidor vuelva a estar en línea.

Comencemos creando un nuevo módulo llamado `failsafeSocket.js` que defina nuestro objeto de contexto:

```javascript
import { OfflineState } from './offlineState.js'
import { OnlineState } from './onlineState.js'

export class FailsafeSocket {
  constructor(options) { // 1
    this.options = options
    this.queue = []
    this.currentState = null
    this.socket = null
    this.states = {
      offline: new OfflineState(this),
      online: new OnlineState(this),
    }
    this.changeState('offline')
  }

  changeState(state) { // 2
    console.log(`Activating state: ${state}`)
    this.currentState = this.states[state]
    this.currentState.activate()
  }

  send(data) { // 3
    this.currentState.send(data)
  }
}
```

La clase `FailsafeSocket` se compone de tres elementos principales:
1. El constructor inicializa varias estructuras de datos, incluida la cola que contendrá los datos enviados mientras el socket esté fuera de línea. Además, crea un conjunto de dos estados: uno para implementar el comportamiento del socket mientras está fuera de línea (*offline*) y otro cuando el socket está en línea (*online*).
2. El método `changeState()` es responsable de la transición de un estado a otro. Simplemente actualiza la variable de instancia `currentState` y llama a `activate()` en el estado de destino.
3. El método `send()` contiene la funcionalidad principal de la clase `FailsafeSocket`. Aquí es donde queremos tener un comportamiento diferente según el estado fuera de línea/en línea. Como podemos ver, esto se hace delegando la operación al estado actualmente activo.

Veamos ahora cómo se ven los dos estados, comenzando por el módulo `offlineState.js`:

```javascript
import { createConnection } from 'node:net'

export class OfflineState {
  constructor(failsafeSocket) {
    this.failsafeSocket = failsafeSocket
  }

  send(data) { // 1
    this.failsafeSocket.queue.push(data)
  }

  activate() { // 2
    const retry = () => {
      setTimeout(() => this.activate(), 1000)
    }
    console.log(
      `Trying to connect (${this.failsafeSocket.queue.length} queued `+
      `messages)`
    )
    this.failsafeSocket.socket = createConnection(
      this.failsafeSocket.options,
      () => {
        console.log('Connection established')
        this.failsafeSocket.socket.removeListener('error', retry)
        this.failsafeSocket.changeState('online')
      }
    )
    this.failsafeSocket.socket.once('error', retry)
  }
}
```

El módulo que acabamos de crear es responsable de gestionar el comportamiento del socket mientras está fuera de línea. Así es como funciona:
1. El método `send()` solo es responsable de encolar cualquier dato que reciba. Asumimos que estamos fuera de línea, por lo que guardaremos esos objetos de datos para más adelante. Eso es todo lo que tenemos que hacer aquí.
2. El método `activate()` intenta establecer una conexión con el servidor utilizando la función `createConnection()` del módulo central `node:net`. Si la operación falla, vuelve a intentarlo después de un segundo. Continúa intentándolo hasta que se establece una conexión válida, en cuyo caso el estado de `failsafeSocket` pasa a estar en línea (*online*).

A continuación, creemos el módulo `onlineState.js`, que es donde implementaremos la clase `OnlineState`:

```javascript
export class OnlineState {
  constructor(failsafeSocket) {
    this.failsafeSocket = failsafeSocket
  }

  send(data) { // 1
    this.failsafeSocket.queue.push(data)
    this._tryFlush()
  }

  async _tryFlush() { // 2
    try {
      let success = true
      while (this.failsafeSocket.queue.length > 0) {
        const data = this.failsafeSocket.queue[0]
        const flushed = await this._tryWrite(data)
        if (flushed) {
          this.failsafeSocket.queue.shift()
        } else {
          success = false
          break
        }
      }
      if (!success) {
        this.failsafeSocket.changeState('offline')
      }
    } catch (err) {
      console.error('Error during flush', err.message)
      this.failsafeSocket.changeState('offline')
    }
  }

  _tryWrite(data) { // 3
    return new Promise(resolve => {
      this.failsafeSocket.socket.write(data, err => {
        if (err) {
          console.error('Error writing data', err.message)
          resolve(false)
        } else {
          resolve(true)
        }
      })
    })
  }

  activate() { // 4
    this._tryFlush()
  }
}
```

La clase `OnlineState` modela el comportamiento de `FailsafeSocket` cuando hay una conexión activa con el servidor. Así es como funciona:
1. El método `send()` encola los datos y luego intenta inmediatamente vaciar (*flush*) todos los datos acumulados en el socket, ya que asume que estamos en línea. Utilizará el método interno `_tryFlush()` para hacer eso.
2. El método `_tryFlush()` intenta enviar todos los mensajes que se encuentran actualmente en la cola. Es la lógica central para garantizar que los mensajes se entreguen eventualmente cuando el socket esté en línea. Este método itera mientras haya mensajes en la cola; recupera el primer mensaje de la cola (FIFO: *first-in, first-out*) y luego intenta escribir el mensaje en el socket utilizando el método interno `_tryWrite()`. Este método devuelve una promesa que se resuelve en `true` (éxito) o `false` (fallo al escribir). Si la operación es exitosa, el mensaje actual se elimina de la cola; de lo contrario, establecemos la bandera local `success` en `false` y rompemos el bucle. Después de que se completa el bucle, si la bandera `success` es `false` (lo que significa que al menos un mensaje no se pudo enviar), se llama al método `this.failsafeSocket.changeState('offline')`. Esto hace que `failsafeSocket` pase al estado fuera de línea si no pudimos escribir debido a que perdimos la conexión con el servidor. En caso de que haya una excepción en esta lógica, la capturamos y también asumimos que el socket se ha desconectado. Esto evita posibles rechazos no controlados (*uncaught rejections*).
3. El método `_tryWrite()` encapsula la escritura real de datos en el socket subyacente, manejando posibles errores y resolviendo una promesa para indicar éxito (`true`) o fallo (`false`).
4. Finalmente, tenemos la función `activate()`. Este método se llama cuando el estado en línea se convierte en el estado activo para `failsafeSocket`. Asegura que cualquier mensaje encolado se intente enviar de inmediato llamando al método interno `_tryFlush()`.

Eso es todo para `FailsafeSocket`. Ahora que hemos establecido la estructura central, construyamos un cliente y un servidor de muestra para demostrar el socket a prueba de fallos en acción. Sin embargo, debido a que estamos trabajando con sockets TCP sin procesar (*raw TCP sockets*), necesitamos definir explícitamente un protocolo de comunicación. A diferencia de protocolos de nivel superior como HTTP, los sockets sin formato transmiten solo un flujo de bytes. Para enviar mensajes estructurados y significativos, debemos acordar cómo se codifican esos mensajes en un flujo de bytes en el lado del emisor y cómo se decodifican de nuevo en datos estructurados en el lado del receptor.

Nuestro objetivo es construir un sistema de monitorización simple donde las máquinas cliente envíen periódicamente estadísticas de utilización de recursos a un servidor central. Para estructurar estas transmisiones de datos, necesitamos definir la forma de los datos contenidos en cada mensaje. Para nuestro caso de uso, podemos utilizar los siguientes campos:
- `ts`: Una marca de tiempo Unix (*timestamp*) que indica la hora de generación del mensaje en el cliente.
- `client`: Una cadena que identifica de forma única al cliente. Generaremos esto concatenando el nombre de host del cliente (*hostname*) y el ID del proceso (PID).
- `mem`: Un objeto que representa la salida de `process.memoryUsage()`. Esto proporciona información detallada sobre el uso de la memoria, incluyendo:
  - `rss` (*resident set size*): La cantidad de memoria ocupada por el proceso en la memoria RAM.
  - `heapTotal`: El tamaño total del heap del motor de JavaScript V8 para este proceso.
  - `heapUsed`: La cantidad del heap de V8 actualmente en uso.
  - Otras métricas relacionadas con la memoria, todas medidas en bytes.

Si bien podríamos transmitir esta información como un mensaje simple codificado en JSON, ese enfoque presenta un desafío: el servidor no podría distinguir fácilmente mensajes individuales dentro del flujo continuo de bytes, ya que cada mensaje podría tener una longitud diferente.

Para resolver esto, utilizaremos la delimitación por prefijo de longitud (*length-prefix framing*). Esta técnica implica estructurar cada mensaje en dos partes:
- **Campo de longitud (4 bytes):** Un campo de tamaño fijo (4 bytes, codificado en orden Big Endian) que indica la longitud, en bytes, del campo de datos subsiguiente.
- **Campo de datos (longitud variable):** Los datos reales del mensaje, codificados como una cadena JSON UTF-8.

Esto significa que el servidor, al recibir un flujo de bytes de un cliente, primero leerá los 4 bytes iniciales para determinar la longitud del siguiente mensaje. Luego consumirá esa cantidad de bytes para obtener el mensaje JSON completo, que luego puede deserializarse. El proceso luego se repite para el siguiente mensaje entrante.

La delimitación por prefijo de longitud proporciona una forma sencilla y confiable de delimitar mensajes de longitud variable a través de una conexión de red, y se utiliza ampliamente. Algunos ejemplos notables son gRPC, el protocolo de Redis (RESP) y BitTorrent.

Ahora que hemos definido el protocolo que queremos utilizar, estamos listos para comenzar a trabajar en la implementación de nuestro servidor. Queremos mantener esta implementación simple, por lo que, por ahora, lo único que haremos será aceptar la conexión, analizar los mensajes entrantes e imprimirlos en la salida estándar. Coloquemos el código del servidor en un módulo llamado `server.js`:

```javascript
import { createServer } from 'node:net'

const server = createServer(socket => {
  socket.on('error', err => {
    console.error('Server error', err.message)
  })

  // Accumulate incoming data
  let buffer = Buffer.alloc(0)

  // When a chunk of data is received
  socket.on('data', data => {
    // Append new data to the buffer
    buffer = Buffer.concat([buffer, data])

    // Ensure we have enough bytes for the length prefix
    while (buffer.length >= 4) {
      // Read the message length (Big Endian)
      const messageLength = buffer.readUInt32BE(0)
      if (buffer.length < 4 + messageLength) {
        // Not enough data yet; wait for more
        break
      }

      // Check if we have the complete message
      const message = buffer.subarray(4, 4 + messageLength).toString('utf8')

      // Process the message (just log it)
      console.log('Received message:', JSON.parse(message))

      // Remove the processed message from the buffer
      buffer = buffer.subarray(4 + messageLength)
    }
  })
})

// Start the server and listen on port 4545
server.listen(4545, () => console.log('Server started'))
```

El código anterior implementa un servidor TCP utilizando la función `createServer()` del módulo central `node:net`. Se establece un socket para cada conexión, lo que permite la comunicación con los clientes. El servidor utiliza estos sockets para recibir y procesar mensajes JSON, adhiriéndose al protocolo de delimitación por prefijo de longitud que definimos. Los comentarios dentro del código deberían ofrecer una explicación exhaustiva de cómo se implementa este protocolo.

La última pieza del rompecabezas es el código del lado del cliente. Este código es especialmente interesante porque muestra cómo podemos aprovechar el patrón State implementado dentro de nuestro socket a prueba de fallos para manejar la gestión de conexiones y el envío de mensajes. Este código va en `client.js`:

```javascript
import { hostname } from 'node:os'
import { FailsafeSocket } from './failsafeSocket.js'

const clientId = `${hostname()}@${process.pid}` // 1
console.log(`Starting client ${clientId}`)
const failsafeSocket = new FailsafeSocket({ port: 4545 }) // 2

setInterval(() => { // 3
  // constructs the message
  const messageData = Buffer.from(
    JSON.stringify({
      ts: Date.now(),
      client: clientId,
      mem: process.memoryUsage(),
    }),
    'utf-8'
  )
  // creates a 4-byte buffer to store the message length
  const messageLength = Buffer.alloc(4)
  messageLength.writeUInt32BE(messageData.length, 0)
  // concatenates the message length and message data
  const message = Buffer.concat([messageLength, messageData])
  // sends the message
  failsafeSocket.send(message)
}, 5000)
```

Este código es, como era de esperar, simple. Dado que hemos abstraído toda la lógica de conectividad en nuestra implementación de socket a prueba de fallos (usando el patrón State), aquí solo tenemos que preocuparnos por algunas cosas básicas:
1. Generar e imprimir el ID de cliente que se utilizará para identificar a este cliente ante el servidor.
2. Crear una nueva instancia de nuestro socket a prueba de fallos (que intentará conectarse inmediatamente al servidor en el puerto dado).
3. Cada 5 segundos, creamos un mensaje que contiene la información de memoria para el proceso de cliente actual y, utilizando nuestro protocolo de prefijo de longitud, lo enviamos a través del socket a prueba de fallos.

Para probar el pequeño sistema que construimos, debemos ejecutar tanto el cliente como el servidor, y luego podemos probar las características de `failsafeSocket` deteniendo y reiniciando el servidor. Deberíamos ver que el estado del cliente cambia entre en línea y fuera de línea, y que cualquier medición de memoria recopilada mientras el servidor está fuera de línea se encola y luego se reenvía tan pronto como el servidor vuelve a estar en línea.

Esta muestra debería ser una clara demostración de cómo el patrón State puede ayudar a aumentar la modularidad y legibilidad de un componente que tiene que adaptar su comportamiento en función de su estado.

La clase `FailsafeSocket` que construimos en esta sección es solo para demostrar el patrón State y no pretende ser una solución completa y 100% confiable para manejar problemas de conectividad con sockets TCP. Por ejemplo, no estamos verificando que todos los datos escritos en el flujo del socket sean recibidos por el servidor, lo que requeriría algo más de código que no está estrictamente relacionado con el patrón que queríamos describir. Para una alternativa de producción, puedes contar con ZeroMQ ([nodejsdp.link/zeromq](https://nodejsdp.link/zeromq)). Hablaremos sobre algunos patrones que usan ZeroMQ más adelante en el libro en el [Capítulo 13](https://subscription.packtpub.com/book/web-development/9781803238944/13), Patrones de mensajería e integración. Otro atajo que tomamos es que no hemos incluido un mecanismo para gestionar el crecimiento de la cola durante desconexiones prolongadas. Si un cliente permanece desconectado durante un período prolongado, la cola continuará acumulando mensajes, aumentando la huella de memoria del proceso. Si esto persiste, el cliente eventualmente agotará toda la memoria disponible y se bloqueará (*crash*), lo que resultará en la pérdida de todos los mensajes encolados. Hay varias estrategias que se podrían implementar para mitigar este problema, como persistir los datos de la cola en un archivo (o una base de datos local) o implementar un tamaño de cola limitado (*capped queue*), descartando los mensajes más antiguos una vez que se alcance la capacidad máxima. Sin embargo, implementar estas soluciones está fuera del alcance de esta sección y se deja como un ejercicio para el lector interesado. Si necesitas una pista, echa un vistazo a la biblioteca `cbuffer` ([nodejsdp.link/cbuffer](https://nodejsdp.link/cbuffer)) o `tracked-queue` ([nodejsdp.link/tracked-queue](https://nodejsdp.link/tracked-queue)).

#### En la práctica

XState ([nodejsdp.link/xstate](https://nodejsdp.link/xstate)) es una biblioteca popular para crear máquinas de estados (un modelo matemático de computación) en JavaScript y TypeScript. Si bien las máquinas de estados son un concepto más amplio que el patrón de diseño State, XState proporciona una biblioteca potente que también se puede utilizar para implementar el patrón State. Aunque XState se usa más comúnmente en proyectos frontend debido a sus características reactivas, XState también se puede usar en el contexto de proyectos de Node.js. Por ejemplo, Mastra ([nodejsdp.link/mastra](https://nodejsdp.link/mastra)), un framework de agentes de IA escrito en TypeScript, utiliza XState para modelar el comportamiento de un flujo de trabajo de IA a medida que pasa por los distintos estados de su ciclo de vida (pendiente, ejecutando, completado, fallido, etc.).

---

### Sección 3: Template (Plantilla)

El siguiente patrón que vamos a analizar se llama **Template**, y tiene mucho en común con el patrón Strategy. El patrón Template define una clase abstracta que implementa el esqueleto (que representa las partes comunes) de un componente, donde algunos de sus pasos se dejan sin definir. Las subclases pueden luego llenar los vacíos en el componente implementando las partes faltantes, llamadas **métodos de plantilla** (*template methods*). La intención de este patrón es hacer posible definir una familia de clases que son todas variaciones de una familia de componentes. El siguiente diagrama UML muestra la estructura que acabamos de describir:

**Figura 9.4:** Diagrama UML del patrón Template.

Las tres clases concretas que se muestran en la Figura 9.4 extienden la clase plantilla y proporcionan una implementación para `templateMethod()`, que es abstracto o virtual puro, para usar la terminología de C++. En JavaScript plano, no tenemos una forma formal de definir clases abstractas, por lo que todo lo que podemos hacer es dejar el método sin definir o asignarlo a una función que siempre lance una excepción en tiempo de ejecución, indicando que el método debe ser implementado.

TypeScript te permite anotar una clase como `abstract`. Cuando haces esto, TypeScript verificará en tiempo de compilación que todas las implementaciones concretas de la clase estén implementando todos los métodos necesarios. En esencia, JavaScript se basa en descubrir estos errores durante la ejecución, mientras que TypeScript los previene de manera proactiva antes de que el código se ejecute. Hay más detalles disponibles en la documentación oficial en [nodejsdp.link/ts-abstract](https://nodejsdp.link/ts-abstract).

El patrón Template se puede considerar un patrón más tradicionalmente orientado a objetos que los otros patrones que hemos visto hasta ahora, porque la herencia es una parte fundamental de su implementación.

El propósito de los patrones Template y Strategy es muy similar, pero la principal diferencia entre los dos radica en su estructura e implementación. Ambos nos permiten cambiar las partes variables de un componente mientras reutilizamos las partes comunes. Sin embargo, mientras que Strategy nos permite hacerlo dinámicamente en tiempo de ejecución, con Template, el componente completo se determina en el momento en que se define la clase concreta.

Bajo estos supuestos, el patrón Template podría ser más adecuado en aquellas circunstancias en las que deseamos crear variaciones preempaquetadas de un componente. Como siempre, la elección entre un patrón y el otro depende del desarrollador, quien debe considerar los diversos pros y contras para cada caso de uso.

Trabajemos ahora en un ejemplo.

#### Una plantilla de gestor de configuración

Para tener una mejor idea de las diferencias entre Strategy y Template, reimplementemos ahora el objeto `Config` que definimos en la sección del patrón Strategy, pero esta vez, usando Template. Como en la versión anterior del objeto `Config`, queremos tener la capacidad de cargar y guardar un conjunto de propiedades de configuración utilizando diferentes formatos de archivo.

Comencemos definiendo la clase de plantilla. La llamaremos `ConfigTemplate`:

```javascript
import { readFile, writeFile } from 'node:fs/promises'

export class ConfigTemplate {
  async load(file) {
    console.log(`Deserializing from ${file}`)
    this.data = this._deserialize(await readFile(file, 'utf-8'))
  }

  async save(file) {
    console.log(`Serializing to ${file}`)
    await writeFile(file, this._serialize(this.data))
  }

  _serialize() {
    throw new Error('_serialize() must be implemented')
  }

  _deserialize() {
    throw new Error('_deserialize() must be implemented')
  }
}
```

La clase `ConfigTemplate` implementa las partes comunes de la lógica de gestión de configuración, a saber, establecer y obtener propiedades, además de cargarlas y guardarlas en el disco. Sin embargo, deja abierta la implementación de `_serialize()` y `_deserialize()`; esos son de hecho nuestros métodos de plantilla, que permitirán la creación de clases `Config` concretas que admitan formatos de configuración específicos. El guion bajo al principio de los nombres de los métodos de plantilla indica que son solo para uso interno, una forma fácil de marcar métodos protegidos. Dado que en JavaScript no podemos declarar un método como abstracto, simplemente los definimos como funciones vacías (*stubs*), lanzando un error si se invocan (en otras palabras, si no son sobrescritos por una subclase concreta).

Creemos ahora una clase concreta utilizando nuestra plantilla como ejemplo, una que nos permita cargar y guardar la configuración utilizando el formato JSON:

```javascript
// jsonConfig.js
import { ConfigTemplate } from './configTemplate.js'

export class JsonConfig extends ConfigTemplate {
  _deserialize(data) {
    return JSON.parse(data)
  }

  _serialize(data) {
    return JSON.stringify(data, null, ' ')
  }
}
```

La clase `JsonConfig` extiende nuestra clase plantilla, `ConfigTemplate`, y proporciona una implementación concreta para los métodos `_deserialize()` y `_serialize()`.

De manera similar, podemos implementar una clase `YamlConfig` que admita el formato de archivo `.yaml` utilizando la misma clase de plantilla:

```javascript
// yamlConfig.js
import { ConfigTemplate } from './configTemplate.js'
import YAML from 'yaml' // v2.7.0

export class YamlConfig extends ConfigTemplate {
  _deserialize(data) {
    return YAML.parse(data)
  }

  _serialize(data) {
    return YAML.stringify(data)
  }
}
```

Ahora, podemos usar nuestras clases de gestión de configuración concretas para cargar y guardar algunos datos de configuración:

```javascript
// index.js
import { join } from 'node:path'
import { JsonConfig } from './jsonConfig.js'
import { YamlConfig } from './yamlConfig.js'

const SAMPLES = join(import.meta.dirname, 'samples')

const jsonConfig = new JsonConfig()
await jsonConfig.load(join(SAMPLES, 'config.json'))
jsonConfig.data.env.NODE_ENV = 'production'
jsonConfig.data.env.NODE_OPTIONS = '--enable-source-maps'
await jsonConfig.save(join(SAMPLES, 'config_mod.json'))

const yamlConfig = new YamlConfig()
await yamlConfig.load(join(SAMPLES, 'config.yaml'))
yamlConfig.data.env.NODE_ENV = 'production'
yamlConfig.data.env.NODE_OPTIONS = '--enable-source-maps'
await yamlConfig.save(join(SAMPLES, 'config_mod.yaml'))
```

Observa la diferencia con el patrón Strategy: la lógica para formatear y analizar los datos de configuración está integrada en la propia clase, en lugar de elegirse en tiempo de ejecución.

Con un esfuerzo mínimo, el patrón Template nos permitió obtener un nuevo gestor de configuración completamente funcional reutilizando la lógica y la interfaz heredadas de la clase plantilla principal y proporcionando solo la implementación de unos pocos métodos abstractos.

En este punto, como un ejercicio divertido, debería ser muy fácil para ti implementar una plantilla que pueda manejar la configuración en formato TOML. Deberías poder tomar prestada la mayor parte del código de nuestro ejemplo del patrón Strategy y, si te quedas atascado, puedes encontrar una solución en nuestro repositorio de código.

#### En la práctica

Este patrón no debería resultarnos del todo nuevo. Ya lo encontramos en el [Capítulo 6](https://subscription.packtpub.com/book/web-development/9781803238944/6), Programación con Streams, cuando estábamos extendiendo las diferentes clases de stream para implementar nuestros streams personalizados. En ese contexto, los métodos de plantilla eran los métodos `_write()`, `_read()`, `_transform()` o `_flush()`, según la clase de stream que quisiéramos implementar. Para crear un nuevo stream personalizado, necesitábamos heredar de una clase de stream abstracta específica, proporcionando una implementación para los métodos de plantilla.

A continuación, vamos a aprender sobre un patrón muy importante y omnipresente que también está integrado en el propio lenguaje JavaScript: el patrón Iterator.

---

### Sección 4: Iterator (Iterador)

El patrón **Iterator** es un patrón fundamental, y es tan importante y comúnmente utilizado que generalmente está integrado en el propio lenguaje de programación. Todos los principales lenguajes de programación implementan el patrón de una forma u otra, incluido, por supuesto, JavaScript (a partir de la especificación ECMAScript 2015).

El patrón Iterator define una interfaz o protocolo común para iterar sobre los elementos de un contenedor, como un array, una pila o una estructura de datos de árbol. Por lo general, el algoritmo para iterar sobre los elementos de un contenedor es diferente según la estructura real de los datos. Piensa en iterar sobre un array frente a recorrer un árbol. En la primera situación, solo necesitamos un bucle secuencial simple; en la segunda, se requiere un algoritmo de recorrido de árbol más complejo ([nodejsdp.link/tree-traversal](https://nodejsdp.link/tree-traversal)). Con el patrón Iterator, ocultamos los detalles sobre el algoritmo que se está utilizando o la estructura de los datos y proporcionamos una interfaz común para iterar sobre cualquier tipo de contenedor. En esencia, el patrón Iterator nos permite desacoplar la implementación del algoritmo de recorrido de la forma en que consumimos los resultados (los elementos) de la operación de recorrido.

En JavaScript, sin embargo, los iteradores funcionan muy bien incluso con otros tipos de construcciones que no son necesariamente contenedores, como emisores de eventos y streams. Por lo tanto, podemos decir en términos más generales que el patrón Iterator define una interfaz para iterar sobre elementos producidos o recuperados en secuencia.

#### El protocolo iterador

En JavaScript, el patrón Iterator se implementa a través de protocolos en lugar de mediante construcciones formales, como la herencia. Esto significa esencialmente que la interacción entre el implementador y el consumidor del patrón Iterator se comunicará utilizando interfaces y objetos cuya forma se acuerda de antemano.

Lo primero que debemos aprender es sobre el protocolo iterador (*iterator protocol*). Este protocolo define una interfaz para producir una secuencia de valores. El protocolo iterador define un objeto iterador como cualquier objeto que implemente un método `next()` con el siguiente comportamiento: cada vez que se llama al método, la función devuelve el siguiente elemento en la iteración envuelto en un objeto, llamado el **resultado del iterador** (*iterator result*), que tiene dos propiedades: `done` y `value`:
- `done` se establece en `true` cuando se completa la iteración, o en otras palabras, cuando no hay más elementos que devolver. De lo contrario, `done` será `undefined` o `false`.
- `value` contiene el elemento actual de la iteración y puede dejarse `undefined` si `done` es `true`. Si `value` se establece incluso cuando `done` es `true`, entonces se dice que `value` contiene el valor de retorno de la iteración, un valor que no forma parte de los elementos que se están iterando, pero que está relacionado con la iteración en sí como un todo (por ejemplo, el tiempo empleado en iterar todos los elementos o el promedio de todos los elementos iterados si los elementos son números).

Nada nos impide agregar propiedades adicionales al objeto devuelto por un iterador. Sin embargo, esas propiedades serán simplemente ignoradas por las construcciones o APIs integradas que consumen el iterador (las discutiremos en un momento).

Si queremos representar el tipo `Iterator` en TypeScript, podríamos definirlo de la siguiente manera:

```typescript
type Iterator<T> = {
  next(): {
    done: boolean
    value: T
  }
}
```

Esta definición de tipo es genérica sobre `T`, lo que significa que un iterador puede producir valores de cualquier tipo específico `T`. En otras palabras, podrías tener un iterador que produce cadenas, otro que produce números o uno que produce objetos más complejos. El punto clave es que el tipo permanece consistente a través de sucesivas invocaciones del método `next()` en el mismo iterador.

Ten en cuenta que el tipo real de TypeScript para el `Iterator` es un poco más complejo, ya que captura varias características avanzadas que no son particularmente relevantes a este nivel.

Enfoquémonos ahora en un ejemplo rápido para demostrar cómo implementar el protocolo iterador. Implementemos una función factoría llamada `createAlphabetIterator()`, que crea un iterador que nos permite recorrer todas las letras del alfabeto inglés. Dicha función se vería así:

```javascript
const A_CHAR_CODE = 'A'.charCodeAt(0)
const Z_CHAR_CODE = 'Z'.charCodeAt(0)

function createAlphabetIterator() {
  let currCode = A_CHAR_CODE
  return {
    next() {
      const currChar = String.fromCodePoint(currCode)
      if (currCode > Z_CHAR_CODE) {
        return { done: true }
      }
      currCode++
      return { value: currChar, done: false }
    },
  }
}
```

La lógica de la iteración es realmente muy sencilla; en cada invocación del método `next()`, simplemente incrementamos un número que representa el código de carácter de la letra, lo convertimos en un carácter y luego lo devolvemos utilizando la forma de objeto definida por el protocolo iterador.

No es un requisito que un iterador devuelva alguna vez `done: true`. De hecho, puede haber muchas situaciones en las que un iterador sea infinito. Un ejemplo es un iterador que devuelve un número aleatorio en cada iteración. Otro ejemplo es un iterador que calcula una serie matemática, como la serie de Fibonacci o los dígitos de la constante pi (como ejercicio, puedes intentar convertir el siguiente algoritmo para usar iteradores: [nodejsdp.link/pi-js](https://nodejsdp.link/pi-js)). Ten en cuenta que incluso si un iterador es teóricamente infinito, no significa que no tendrá límites computacionales o espaciales. Por ejemplo, el número devuelto por la secuencia de Fibonacci se volverá muy grande muy pronto.

El aspecto importante a tener en cuenta es que un iterador es muy a menudo un objeto con estado (*stateful*), ya que debemos realizar un seguimiento de alguna manera de la posición actual de la iteración. En el ejemplo anterior, logramos mantener el estado en una clausura (la variable `currCode`), pero esta es solo una de las formas en que podemos hacerlo. Podríamos haber mantenido, por ejemplo, el estado en una variable de instancia. Esto suele ser mejor en términos de depurabilidad (*debuggability*), ya que podemos leer el estado de la iteración desde el propio iterador en cualquier momento, pero por otro lado, no impide que el código externo modifique la variable de instancia y, por lo tanto, altere el estado de la iteración. Depende de ti decidir los pros y los contras de cada opción.

Los iteradores también pueden ser completamente sin estado (*stateless*). Ejemplos de esto son los iteradores que devuelven elementos aleatorios y se completan aleatoriamente o nunca se completan, y los iteradores que se detienen en la primera iteración.

Ahora, veamos cómo podemos usar el iterador que acabamos de construir. Considera el siguiente fragmento de código:

```javascript
const iterator = createAlphabetIterator()
let iterationResult = iterator.next()
while (!iterationResult.done) {
  console.log(iterationResult.value)
  iterationResult = iterator.next()
}
```

Como podemos ver en el código anterior, el código que consume un iterador puede considerarse un patrón en sí mismo. Es efectivamente un bucle que itera hasta que no haya más elementos disponibles. Sin embargo, podríamos llamar a esto la forma de bajo nivel de usar iteradores. Como veremos más adelante en esta sección, esta no es la única forma que tenemos de consumir un iterador. De hecho, JavaScript ofrece formas mucho más convenientes y elegantes de utilizar iteradores.

Los iteradores pueden especificar opcionalmente dos métodos adicionales: `return([value])` y `throw(error)`. El primero se utiliza por convención para señalar al iterador que el consumidor ha detenido la iteración antes de su finalización, mientras que el segundo se utiliza para comunicar al iterador que se ha producido una condición de error. Ambos métodos rara vez son utilizados por los iteradores integrados.

#### El protocolo iterable

El protocolo iterable (*iterable protocol*) define un medio estandarizado para que un objeto devuelva un iterador. Este tipo de objetos se denominan **iterables**. Por lo general, un iterable es un contenedor de elementos, por ejemplo, una lista, un conjunto, una estructura de árbol, etc. También puede ser un objeto que represente virtualmente un conjunto de elementos, como un objeto `Directory`, que nos permitiría iterar sobre los archivos de un directorio.

En JavaScript, podemos definir un iterable asegurándonos de que implemente el método `@@iterator`, o en otras palabras, un método accesible a través del símbolo integrado `Symbol.iterator`.

> La convención `@@name` indica un símbolo bien conocido (*well-known symbol*) según la especificación ES6. Para obtener más información, puedes consultar la sección correspondiente de la especificación ES6 en [nodejsdp.link/es6-well-known-symbols](https://nodejsdp.link/es6-well-known-symbols).

Dicho método `@@iterator` debe devolver un objeto iterador, que se puede utilizar para iterar sobre los elementos representados por el iterable. Por ejemplo, si nuestro iterable es una clase, tendríamos algo como lo siguiente:

```javascript
class MyIterable {
  // other methods...
  [Symbol.iterator] () {
    // return an iterator
  }
}
```

O si queremos intentar proporcionar un tipo `Iterable` en TypeScript, podría verse así:

```typescript
type Iterable<T> = {
  [Symbol.iterator](): Iterator<T>
}
```

Como puedes ver, esto aprovecha el tipo `Iterator` que definimos en la sección anterior. Esto debería darte una idea clara de cómo el protocolo iterable se basa en el protocolo iterador.

Una vez más, hemos simplificado un poco las cosas. La definición de tipo real de TypeScript para `Iterable` es un poco más compleja de lo que se presenta aquí, ya que captura varias características avanzadas que no son particularmente relevantes en este momento.

Para hacer que esta definición sea un poco más concreta, trabajemos a través de un ejemplo. Construiremos una clase para gestionar datos organizados en una estructura de matriz bidimensional. Queremos que esta clase implemente el protocolo iterable, de modo que podamos examinar todos los elementos de la matriz utilizando un iterador. Creemos un archivo llamado `matrix.js` que contenga el siguiente contenido:

```javascript
export class Matrix {
  constructor(inMatrix) {
    this.data = inMatrix
  }

  get(row, column) {
    if (row >= this.data.length || column >= this.data[row].length) {
      throw new RangeError('Out of bounds')
    }
    return this.data[row][column]
  }

  set(row, column, value) {
    if (row >= this.data.length || column >= this.data[row].length) {
      throw new RangeError('Out of bounds')
    }
    this.data[row][column] = value
  }

  [Symbol.iterator]() {
    let nextRow = 0
    let nextCol = 0

    return {
      next: () => {
        if (nextRow === this.data.length) {
          return { done: true }
        }

        const currVal = this.data[nextRow][nextCol]

        if (nextCol === this.data[nextRow].length - 1) {
          nextRow++
          nextCol = 0
        } else {
          nextCol++
        }

        return { value: currVal }
      },
    }
  }
}
```

Como podemos ver, la clase contiene los métodos básicos para obtener y establecer valores en la matriz, así como el método `@@iterator`, que implementa nuestro protocolo iterable. El método `@@iterator` devolverá un iterador, como se especifica en el protocolo iterable, y dicho iterador se adhiere al protocolo iterador. La lógica del iterador es muy sencilla: simplemente estamos recorriendo las celdas de la matriz desde la parte superior izquierda hasta la parte inferior derecha, escaneando cada columna de cada fila. Hacemos eso aprovechando dos índices: `nextRow` y `nextCol`.

Ahora, es el momento de probar nuestra clase `Matrix` iterable. Podemos hacerlo en un archivo llamado `index.js`:

```javascript
import { Matrix } from './matrix.js'

const matrix2x2 = new Matrix([
  ['11', '12'],
  ['21', '22']
])

const iterator = matrix2x2[Symbol.iterator]()
let iterationResult = iterator.next()
while (!iterationResult.done) {
  console.log(iterationResult.value)
  iterationResult = iterator.next()
}
```

Todo lo que estamos haciendo en el código anterior es crear una instancia de `Matrix` de muestra y luego obtener un iterador usando el método `@@iterator`. Lo que viene a continuación, como ya sabemos, es solo código repetitivo que itera sobre los elementos devueltos por el iterador. La salida de la iteración debería ser `'11'`, `'12'`, `'21'` y `'22'`.

#### Iteradores e iterables como una interfaz nativa de JavaScript

En este punto, te preguntarás: "¿Cuál es el punto de tener todos estos protocolos para definir iteradores e iterables?". Bueno, tener una interfaz estandarizada permite que el código de terceros, así como el propio lenguaje, se modelen alrededor de los dos protocolos que acabamos de ver. De esta manera, podemos tener APIs (incluso nativas), así como construcciones sintácticas que acepten iterables como entrada.

Por ejemplo, la construcción de sintaxis más obvia que acepta un iterable es el bucle `for...of`. Acabamos de ver en el último ejemplo de código que iterar sobre un iterador de JavaScript es una operación bastante estándar y su código es en su mayoría texto repetitivo. De hecho, siempre tendremos una invocación a `next()` para recuperar el siguiente elemento y una comprobación para verificar si la propiedad `done` del resultado de la iteración está establecida en `true`, lo que indica el final de la iteración. Pero no te preocupes, simplemente pasa un iterable a la instrucción `for...of` para recorrer sin problemas los elementos devueltos por su iterador. Esto nos permite procesar la iteración con una sintaxis intuitiva y compacta:

```javascript
for (const element of matrix2x2) {
  console.log(element)
}
```

Otra construcción compatible con iterables es el operador de propagación (*spread operator*):

```javascript
const flattenedMatrix = [...matrix2x2]
console.log(flattenedMatrix)
```

De manera similar, podemos usar un iterable con la operación de asignación por desestructuración:

```javascript
const [oneOne, oneTwo, twoOne, twoTwo] = matrix2x2
console.log(oneOne, oneTwo, twoOne, twoTwo)
```

Las siguientes son algunas APIs integradas de JavaScript que aceptan iterables:
- `Map([iterable])`: [nodejsdp.link/map-constructor](https://nodejsdp.link/map-constructor)
- `WeakMap([iterable])`: [nodejsdp.link/weakmap-constructor](https://nodejsdp.link/weakmap-constructor)
- `Set([iterable])`: [nodejsdp.link/set-constructor](https://nodejsdp.link/set-constructor)
- `WeakSet([iterable])`: [nodejsdp.link/weakset-constructor](https://nodejsdp.link/weakset-constructor)
- `Promise.all(iterable)`: [nodejsdp.link/promise-all](https://nodejsdp.link/promise-all)
- `Promise.race(iterable)`: [nodejsdp.link/promise-race](https://nodejsdp.link/promise-race)
- `Array.from(iterable)`: [nodejsdp.link/array-from](https://nodejsdp.link/array-from)

Del lado de Node.js, una API notable que acepta un iterable es `stream.Readable.from(iterable, [options])` ([nodejsdp.link/readable-from](https://nodejsdp.link/readable-from)), que crea un stream legible a partir de un objeto iterable.

JavaScript en sí define muchos iterables que se pueden usar con las APIs y construcciones que acabamos de ver. El iterable más notable es `Array`, pero también otras estructuras de datos, como `Map`, `Set` e incluso `String`, implementan el método `@@iterator`. En el mundo de Node.js, `Buffer` es otro objeto iterable notable.

Un truco simple para crear una copia de un array sin elementos duplicados es el siguiente: `const uniqArray = Array.from(new Set(arrayWithDuplicates))`. Este fragmento nos muestra cómo los iterables ofrecen una forma para que diferentes componentes se comuniquen entre sí utilizando una interfaz compartida.

#### Implementación del protocolo iterable en iteradores

Las funciones descritas en la sección anterior aceptan un objeto iterable como entrada, no un iterador. Esto plantea una pregunta: ¿Qué podemos hacer si tenemos una función que devuelve un iterador, como nuestro ejemplo `createAlphabetIterator()`? ¿Hay alguna manera de hacer que dicho objeto iterador sea interoperable con las APIs integradas y las construcciones de sintaxis diseñadas para iterables? ¡Bueno, sí y no! No, porque para ser directamente interoperable, el objeto debe implementar el protocolo iterable. Sí, porque nada nos impide implementar también el protocolo iterable en un objeto que ya es un iterador (como los producidos por `createAlphabetIterator()`). De hecho, los dos protocolos no son mutuamente excluyentes; pueden coexistir en el mismo objeto de JavaScript. Exploremos cómo lograr esto actualizando nuestro ejemplo `createAlphabetIterator()`:

```javascript
const A_CHAR_CODE = 'A'.charCodeAt(0)
const Z_CHAR_CODE = 'Z'.charCodeAt(0)

function createAlphabetIterableIterator() {
  let currCode = A_CHAR_CODE
  return {
    next() {
      const currChar = String.fromCodePoint(currCode)
      if (currCode > Z_CHAR_CODE) {
        return { done: true }
      }
      currCode++
      return { value: currChar, done: false }
    },
    [Symbol.iterator]() {
      return this
    },
  }
}
```

¿Puedes detectar lo que ha cambiado? Sí, solo agregamos tres líneas de código en el objeto devuelto:

```javascript
[Symbol.iterator]() {
  return this
}
```

¿Por qué funciona esto? ¿Recuerdas la definición del protocolo iterable? Un objeto es iterable si implementa el método `@@iterator` para devolver un objeto iterador. Bueno, originalmente diseñamos `createAlphabetIterator()` para devolver un objeto iterador, por lo que implementar el método `@@iterator` es tan simple como devolver una referencia a sí mismo (usando `this`).

Para demostrar que este cambio funciona, podemos intentar usar nuestra nueva función con un bucle `for...of` para imprimir todas las letras del alfabeto, cada una en una línea separada:

```javascript
for (const letter of createAlphabetIterableIterator()) {
  console.log(letter)
}
```

O usarlo con la sintaxis de propagación para acumular las letras en un array:

```javascript
const letters = [...createAlphabetIterableIterator()]
console.log(letters)
```

Si todavía tienes dificultades para comprender la diferencia conceptual entre un iterable y un iterador, puedes pensar en ellos de esta manera:
- **Iterable:** Un objeto que representa una colección de elementos sobre los que puedes iterar.
- **Iterator:** Un objeto que te permite moverte de un elemento al siguiente en una colección.

¿Puedes ver la correlación natural entre estas dos definiciones? Un objeto es iterable si puedes obtener un iterador de él. Si ya es un iterador, puedes hacerlo iterable simplemente devolviendo una referencia a sí mismo en el método `@@iterator`.

Implementar ambos protocolos suele ser una buena práctica porque permite a los usuarios de nuestro código elegir la interfaz que sea más adecuada para ellos.

#### Utilidades de iterador

¿Recuerdas cómo discutimos que, en JavaScript, el patrón Iterator se implementa principalmente a través de protocolos (el método `next()`) en lugar de construcciones formales como la herencia? Bueno, si bien eso sigue siendo fundamentalmente cierto, las versiones recientes de JavaScript han introducido un prototipo `Iterator`. Este prototipo se puede extender para crear clases de iteradores que ofrecen beneficios adicionales.

Para ver un ejemplo práctico, implementemos una clase `RangeIterator` que nos permita iterar sobre un rango de valores enteros. Este tipo de utilidad existe en la biblioteca estándar de muchos lenguajes de programación, pero no en JavaScript ni en Node.js (aunque hay una propuesta de ECMAScript para introducirlo como un ciudadano de primera clase en el lenguaje, actualmente en la etapa 2 del proceso de aceptación: [nodejsdp.link/proposal-range](https://nodejsdp.link/proposal-range)). Aquí está el código:

```javascript
class RangeIterator extends Iterator {
  #start
  #end
  #step
  #current

  constructor(start, end, step = 1) {
    super()
    this.#start = start
    this.#end = end
    this.#step = step
    this.#current = undefined
  }

  next() {
    this.#current = this.#current === undefined
      ? this.#start
      : this.#current + this.#step

    if (
      this.#step > 0
        ? this.#current < this.#end
        : this.#current > this.#end
    ) {
      return { done: false, value: this.#current }
    }

    return { done: true }
  }
}
```

Si miras de cerca la implementación del método `next()`, deberías poder apreciar que cada instancia de esta clase es un objeto iterador.

Veamos cómo podemos usar esta nueva clase para enumerar todos los números positivos del 1 al 5:

```javascript
const range = new RangeIterator(1, 6)
let iterationResult = range.next()
while (!iterationResult.done) {
  console.log(iterationResult.value)
  iterationResult = range.next()
}
```

Como era de esperar, esto imprime:

```text
1
2
3
4
5
```

Ten en cuenta que nuestra implementación es inclusiva en el lado izquierdo del rango (1 está incluido) y exclusiva en el lado derecho (6 no está incluido). Este es solo un detalle de implementación común. Te invitamos a intentar hacer que esta clase sea más configurable para que también admita rangos totalmente inclusivos si lo deseas.

El detalle de implementación importante es que, además de implementar el protocolo iterador (al definir el método `next()`), también estamos extendiendo el prototipo `Iterator`. Esto viene con algunas ventajas interesantes:
- **Comprobación de tipos:** Puedes comprobar fácilmente si un objeto es una instancia de `Iterator` utilizando `instanceof`. Esto proporciona una forma confiable de verificar el tipo del iterador en tiempo de ejecución (`range instanceof Iterator` se evalúa como `true`).
- **Métodos auxiliares (Helper methods):** El prototipo `Iterator` proporciona una ubicación estándar para agregar métodos auxiliares integrados y personalizados a todos tus iteradores. Discutiremos algunos de los integrados más adelante en esta sección.
- **Conversión e interoperabilidad:** Puedes convertir fácilmente un iterador basado en protocolos (un objeto que solo implementa `next()`) en una instancia de `Iterator` utilizando el método estático `Iterator.from()`, obteniendo acceso a cualquier método auxiliar que pueda estar disponible.
- **Iteradores iterables:** El prototipo `Iterator` implementa convenientemente el protocolo iterable entre bastidores. Ahora, deberías entender que esto simplemente significa que la clase base `Iterator` tiene un método `@@iterator` que simplemente devuelve `this`. Por lo tanto, todas las instancias de nuestra clase `Iterator` no son solo iteradores sino también objetos iterables. Así podemos usar la sintaxis `for...of`, el operador de propagación y todas las demás utilidades estándar que funcionan con objetos iterables. Si necesitas un ejemplo, intenta ejecutar `const numbers = [...new RangeIterator(1,6)]`.

Ahora, exploremos las utilidades de iterador proporcionadas directamente por el prototipo `Iterator`. Actualmente, las más frecuentes son probablemente `Iterator.prototype.map()`, `Iterator.prototype.filter()` e `Iterator.prototype.reduce()`, que pueden sonarte familiares si has trabajado con sus contrapartes en el prototipo `Array`. Sin embargo, existen diferencias significativas entre los dos conjuntos de métodos, como demostraremos en breve.

Para obtener una lista completa de los métodos disponibles, consulta la documentación de MDN en [nodejsdp.link/iterator](https://nodejsdp.link/iterator).

Para ilustrar el comportamiento de estos métodos, usemos nuestra clase `RangeIterator` para crear un iterador de números del 0 al 10. Luego filtraremos los números impares y duplicaremos los números pares restantes:

```javascript
const zeroToTen = new RangeIterator(0, 10)
const doubledEven = zeroToTen
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
  .toArray()
console.log(doubledEven)
```

Como era de esperar, este ejemplo imprime `[ 0, 4, 8, 12, 16 ]`. Pero, ¿por qué necesitamos llamar explícitamente al asistente `toArray()` al final?

Si intentamos eliminar la llamada `toArray()` y ejecutar nuestro código nuevamente, generará:

```text
Object [Iterator Helper] {}
```

Esto se debe a que `filter()` y `map()` (así como la mayoría de los otros métodos auxiliares de `Iterator`) son **perezosos** (*lazy*). No realizan ningún cálculo hasta que intentas consumir datos del iterador resultante. En su lugar, construyen una canalización de procesamiento (*processing pipeline*, otro iterador) que realiza los cálculos necesarios solo cuando se solicitan sus valores. En esencia, estos métodos te permiten definir una serie de transformaciones que se ejecutan bajo demanda, en lugar de inmediatamente.

Otra forma de apreciar este comportamiento perezoso es examinar la ejecución paso a paso de la canalización del iterador. Considera este código:

```javascript
const zeroToTenIt = new RangeIterator(0, 10)
const doubledEvenIt = zeroToTenIt
  .filter(n => n % 2 === 0)
  .map(n => n * 2)

console.log(doubledEvenIt.next()) // { done: false, value: 0 }
console.log(doubledEvenIt.next()) // { done: false, value: 4 }
```

Examinemos qué sucede después de que se ejecuta cada línea:

```javascript
const zeroToTenIt = new RangeIterator(0, 10);
```

Esto crea una nueva instancia de `RangeIterator` que generará números del 0 hasta (pero sin incluir) el 10. En este punto no se genera ningún valor; el iterador simplemente se inicializa.

```javascript
const doubledEvenIt = zeroToTenIt
  .filter(n => n % 2 === 0)
  .map(n => n * 2);
```

Esto encadena los métodos `filter()` y `map()` en el iterador `zeroToTenIt`. Una vez más, en esta etapa no se realizan operaciones de filtrado ni de mapeo. En su lugar, se crea un nuevo iterador (`doubledEvenIt`), que representa la composición de estas operaciones. Este nuevo iterador sabe cómo extraer valores del iterador `zeroToTenIt`, filtrarlos y luego mapearlos, pero solo lo hace cuando se le solicita explícitamente un valor.

```javascript
console.log(doubledEvenIt.next()); // { done: false, value: 0 }
```

Aquí es donde la magia comienza a suceder. Llamar a `next()` en el iterador `doubledEvenIt` desencadena la ejecución de la canalización de procesamiento para un solo valor:
1. El iterador `doubledEvenIt` solicita el primer valor de `zeroToTenIt`.
2. `zeroToTenIt` genera el valor `0`.
3. El método de filtro de `doubledEvenIt` comprueba si `0` es par (`0 % 2 === 0` es `true`).
4. Dado que `0` es par, el método `map()` de `doubledEvenIt` lo duplica (`0 * 2 = 0`).
5. El método `next()` devuelve `{ done: false, value: 0 }`.

```javascript
console.log(doubledEvenIt.next()); // { done: false, value: 4 }
```

La segunda llamada a `next()` repite el proceso:
1. `doubledEvenIt` solicita el siguiente valor de `zeroToTenIt`.
2. `zeroToTenIt` genera el valor `1`.
3. El método de filtro de `doubledEvenIt` comprueba si `1` es par (`1 % 2 === 0` es `false`).
4. Dado que `1` no es par, el método de filtro lo descarta y solicita el siguiente valor de `zeroToTenIt`.
5. `zeroToTenIt` genera el valor `2`.
6. El método de filtro de `doubledEvenIt` comprueba si `2` es par (`2 % 2 === 0` es `true`).
7. Dado que `2` es par, el método de mapeo de `doubledEvenIt` lo duplica (`2 * 2 = 4`).
8. El método `next()` devuelve `{ done: false, value: 4 }`.

Con suerte, ahora tienes una comprensión clara de lo que queremos decir con métodos auxiliares de iterador "perezosos" y cómo se pueden usar para construir canalizaciones de procesamiento expresivas y eficientes. Es importante recordar que no necesitamos consumir todo el iterador (a menos que decidamos hacerlo explícitamente, por ejemplo, llamando a `toArray()`). Esto es precisamente lo que hace que este enfoque perezoso sea tan conveniente: no se realiza ningún cálculo a menos que lo solicitemos explícitamente intentando consumir el iterador.

Ahora bien, ¿cómo se compara esto con los ayudantes de `Array` similares? Contrastemos la evaluación perezosa de los ayudantes del prototipo `Iterator` con la evaluación impaciente (*eager evaluation*) de sus contrapartes de `Array`. Considera el siguiente ejemplo, que refleja el anterior:

```javascript
const numbersArray = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
const doubledEvenArray = numbersArray
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
console.log(doubledEvenArray)
```

Veamos qué sucede en cada paso:

```javascript
const numbersArray = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

Se crea el array `numbersArray`, que contiene los números enteros del 0 al 9. Todos estos números se cargan en la memoria.

```javascript
const doubledEvenArray = numbersArray
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
```

Aquí es donde tiene lugar la evaluación impaciente (*eager*):
1. El método `filter()` itera sobre cada elemento en `numbersArray`. Para cada elemento `n`, evalúa la condición `n % 2 === 0` (¿es `n` par?). Si la condición es verdadera, el elemento se incluye en un nuevo array (también cargado en la memoria). Si la condición es falsa, el elemento se excluye. Esto da como resultado un nuevo array que contiene solo los números pares: `[0, 2, 4, 6, 8]`.
2. El método `map()` itera luego sobre cada elemento en el nuevo array creado por `filter` (es decir, `[0, 2, 4, 6, 8]`). Para cada elemento `n`, aplica la función de mapeo `n * 2` (duplica el número). Esto da como resultado otro array nuevo, `[0, 4, 8, 12, 16]`, donde cada elemento es el doble del número par correspondiente.
3. Finalmente, este segundo nuevo array, `[0, 4, 8, 12, 16]`, se asigna a la variable `doubledEvenArray`.

En el paso final, simplemente imprimimos el array resultante en la consola.

Si bien el resultado de este enfoque basado en arrays puede ser el mismo que el del enfoque basado en iteradores, existen algunas diferencias sustanciales (y muy importantes) que debemos destacar:
- **Evaluación impaciente, ejecutada inmediatamente:** Con los métodos `filter()` y `map()` de `Array`, en el momento en que el motor de JavaScript encuentra esa línea de código, realiza el cálculo. Las operaciones de filtrado y mapeo se realizan en ese mismo momento. Esto es lo que queremos decir con evaluación "impaciente": el trabajo se realiza por adelantado.
- **Arrays intermedios:** Una consecuencia crítica de la evaluación impaciente es la creación de arrays intermedios. En nuestro ejemplo, `filter()` produce un nuevo array que contiene los números pares y luego `map()` crea otro nuevo array que contiene los números pares duplicados. Estos arrays se mantienen en la memoria, exigiendo un espacio directamente proporcional a su tamaño. Pero, ¿y si, en lugar de unos pocos números, tuviéramos millones? Esta canalización aparentemente simple asignaría tres copias (ligeramente diferentes) del array, cada una con millones de registros. Esto no es simplemente una pequeña ineficiencia de la CPU; puede convertirse rápidamente en un problema importante, agotando potencialmente toda la memoria disponible y haciendo que el programa falle. La saturación de la memoria puede provocar fácilmente errores de falta de memoria (*out-of-memory*), e incluso si la memoria está disponible, copiar millones de registros varias veces requiere una cantidad significativa de procesamiento de CPU. Esto hace que la evaluación impaciente sea inadecuada para grandes conjuntos de datos o situaciones donde la memoria está restringida.
- **Todos los elementos procesados:** Quizás el inconveniente más significativo es que `filter()` y `map()` de `Array` siempre procesan cada elemento del array de entrada, independientemente de si realmente necesitas todos los valores resultantes. Imagina que tuvieras un array con un millón de valores, pero solo necesitaras los primeros 10 elementos del resultado transformado. El enfoque de `Array` seguiría realizando las operaciones de filtrado y mapeo en todo el millón de valores, desperdiciando potencialmente una cantidad significativa de tiempo de CPU y memoria.

La buena noticia es que puedes convertir fácilmente una canalización basada en arrays en una basada en iteradores utilizando `Iterator.from()`:

```javascript
Iterator.from([0, 1, 2, 3, 4, 5, 6, 7, 8, 9])
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
```

Este código generará una eficiente canalización basada en iteradores perezosos que puedes consumir según sea necesario. `Iterator.from()` acepta cualquier objeto iterable (y los arrays son iterables por defecto).

Si estás trabajando con arrays, también puedes usar `Array.prototype.values()` para obtener un iterador sobre los valores del array:

```javascript
const arr = [10, 20, 30]
const iterator = arr.values()
console.log(iterator.next()) // { value: 10, done: false }
```

Esta es una forma conveniente y directa de obtener un iterador a partir de un array. Sin embargo, `Iterator.from()` es una solución más genérica y flexible. Funciona no solo con arrays sino también con cualquier objeto que implemente el protocolo iterable o iterador, incluidos los iteradores personalizados devueltos por bibliotecas u otras estructuras de datos.

#### Generadores

La especificación ES2015 introdujo una construcción de sintaxis estrechamente relacionada con los iteradores: los **generadores**, también conocidos como semicorrutinas (*semicoroutines*) o funciones generadoras. Los generadores generalizan las funciones estándar de JavaScript al permitir múltiples puntos de entrada.

En una función estándar de JavaScript, solo hay un punto de entrada: la invocación de la función en sí. Una vez que se invoca una función, el código de la función se ejecuta secuencialmente hasta el final (o hasta que se alcanza una declaración `return` o se lanza un error).

La ejecución de un generador, sin embargo, se puede suspender en cualquier punto utilizando la declaración `yield` y reanudarse más tarde, comenzando desde la línea siguiente a la última declaración `yield` que suspendió la ejecución. Los generadores son particularmente adecuados para implementar colecciones iterables. De hecho, como veremos en breve, el objeto generador devuelto por una función generadora es tanto un iterador como un iterable.

#### Generadores en teoría

Para definir una función generadora, necesitamos usar la declaración `function*` (la palabra clave `function` seguida de un asterisco):

```javascript
function* myGenerator() {
  // generator body
}
```

Invocar una función generadora no ejecuta su cuerpo inmediatamente. En su lugar, devuelve un objeto generador, que, como hemos mencionado, es tanto un iterador como un iterable. Sin embargo, eso es solo el comienzo. Llamar a `next()` en el objeto generador inicia o reanuda la ejecución del generador hasta que encuentra una declaración `yield` o llega al final (ya sea implícitamente o explícitamente con una declaración `return`).

Dentro de la función generadora, usar `yield x` equivale a devolver `{ done: false, value: x }` desde un iterador. Si la función generadora devuelve un valor `x`, equivale a devolver `{ done: true, value: x }`.

#### Una función generadora simple

Para demostrar lo que acabamos de aprender, veamos un generador simple llamado `fruitGenerator()`, que producirá (*yield*) dos nombres de frutas y devolverá su temporada de maduración:

```javascript
function* fruitGenerator () {
  yield 'peach'
  yield 'watermelon'
  return 'summer'
}

const fruitGeneratorObj = fruitGenerator()
console.log(fruitGeneratorObj.next()) // 1
console.log(fruitGeneratorObj.next()) // 2
console.log(fruitGeneratorObj.next()) // 3
```

El código anterior imprimirá el siguiente texto:

```text
{ value: 'peach', done: false }
{ value: 'watermelon', done: false }
{ value: 'summer', done: true }
```

Esta es una breve explicación de lo que sucedió:
1. La primera vez que se invocó `fruitGeneratorObj.next()`, el generador inició su ejecución hasta que alcanzó el primer comando `yield`, lo que puso el generador en pausa y devolvió el valor `'peach'` al invocador.
2. En la segunda invocación de `fruitGeneratorObj.next()`, el generador se reanudó, comenzando desde el segundo comando `yield`, lo que, a su vez, volvió a poner la ejecución en pausa, mientras devolvía el valor `'watermelon'` al invocador.
3. La última invocación de `fruitGeneratorObj.next()` hizo que la ejecución del generador se reanudara desde su última instrucción, una declaración `return`, que finaliza el generador, devuelve el valor `'summer'` y establece la propiedad `done` en `true` en el objeto resultante.

Dado que un objeto generador también es un iterable, podemos usarlo en un bucle `for...of`, como en este ejemplo:

```javascript
for (const fruit of fruitGenerator()) {
  console.log(fruit)
}
```

El bucle anterior imprimirá:

```text
peach
watermelon
```

¿Por qué no se imprime `summer`? Bueno, `summer` no es producido por `yield` por nuestro generador, sino que es retornado con `return`, lo que indica que la iteración está completa con `summer` como valor de retorno (no como un elemento).

#### Control de un iterador generador

Una característica menos conocida de los iteradores, que aún no hemos discutido, es que el método `next()` puede aceptar un argumento opcional: un valor que pasa a estar disponible dentro del cuerpo de la función `next()` y que se puede utilizar para alterar el comportamiento original del método. Esto permite pasar contexto adicional entre llamadas sucesivas a `next()`, lo que permite una forma de comunicación bidireccional entre el iterador y su consumidor.

Por ejemplo, piensa en nuestro `RangeIterator` anterior. En teoría, podríamos modificar su función `next()` para que acepte un valor que anule el parámetro `step`, lo que permitiría al consumidor omitir una cantidad de elementos dinámicamente. Esto crearía un mecanismo donde cada llamada a `next()` no solo recupera un valor del iterador sino que también influye en su comportamiento. Si bien esto agrega flexibilidad, rara vez se usa en la práctica.

Vale la pena señalar que los generadores (que son efectivamente una abstracción sobre los iteradores) proporcionan la misma funcionalidad. El método `next()` puede aceptar un valor opcional, que se convierte en el resultado de la expresión `yield` correspondiente dentro de la función generadora.

Para ilustrar esto, considera el siguiente ejemplo:

```javascript
function* twoWayGenerator() {
  const who = yield null
  yield `Hello ${who}`
}

const twoWay = twoWayGenerator()
twoWay.next()
console.log(twoWay.next('world'))
```

Cuando se ejecuta, el código anterior imprime `Hello world`. Esto significa que ha sucedido lo siguiente:
1. La primera vez que se invocó el método `next()`, el generador alcanzó la primera declaración `yield` y luego se puso en pausa.
2. Cuando se invocó `next('world')`, el generador se reanudó desde el punto donde se puso en pausa, que está en la instrucción `yield`, pero esta vez, teníamos un valor que se pasó de regreso al generador. Este valor se asignó luego a la variable `who`. Luego, el generador añadió la variable `who` a la cadena `'Hello '` y produjo (*yield*) el resultado.

Otras dos características adicionales proporcionadas por los objetos generadores son los métodos de iterador opcionales `throw()` y `return()`. El primero se comporta como `next()` pero también lanzará una excepción dentro del generador como si se hubiera lanzado en el punto del último `yield`, y devuelve el objeto iterador canónico con las propiedades `done` y `value`. El segundo, el método `return()`, fuerza al generador a terminar y devuelve un objeto como el siguiente: `{done: true, value: returnArgument}`, donde `returnArgument` es el argumento pasado al método `return()`.

El siguiente código muestra una demostración de estos dos métodos:

```javascript
function* twoWayGenerator() {
  try {
    const who = yield null
    yield `Hello ${who}`
  } catch (err) {
    yield `Hello error: ${err.message}`
  }
}

console.log('Using throw():')
const twoWayException = twoWayGenerator()
twoWayException.next()
console.log(twoWayException.throw(new Error('Boom!')))

console.log('Using return():')
const twoWayReturn = twoWayGenerator()
console.log(twoWayReturn.return('myReturnValue'))
```

La ejecución del código anterior imprimirá lo siguiente en la consola:

```text
Using throw():
{ value: 'Hello error: Boom!', done: false }
Using return():
{ value: 'myReturnValue', done: true }
```

Como podemos ver, la función `twoWayGenerator()` recibirá una excepción tan pronto como retorne la primera instrucción `yield`. Esto funciona exactamente como si se lanzara una excepción desde el interior del generador, y esto significa que se puede capturar y manejar como cualquier otra excepción utilizando un bloque `try...catch`. El método `return()`, en cambio, simplemente detendrá la ejecución del generador, haciendo que el valor dado sea proporcionado como un valor de retorno por el generador.

#### Cómo usar generadores en lugar de iteradores

Los objetos generadores también son iteradores. Esto significa que las funciones generadoras se pueden usar para implementar el método `@@iterator` de objetos iterables. Para demostrar esto, convirtamos nuestro ejemplo de iteración `Matrix` anterior a generadores. Actualicemos nuestro archivo `matrix.js` de la siguiente manera:

```javascript
export class Matrix {
  // ... rest of the methods (unchanged)
  *[Symbol.iterator]() {
    for (const row of this.data) {
      for (const cell of row) {
        yield cell
      }
    }
  }
}
```

Hay algunos aspectos interesantes en el fragmento de código que acabamos de ver. Analicémoslos con más detalle:
- Lo primero que se nota es que el método `@@iterator` ahora es un generador (observa el asterisco, `*`, antes del nombre del método).
- Las variables que usamos para mantener el estado de la iteración ahora son solo variables locales (`row` y `cell`) para el generador, mientras que en la versión anterior de la clase `Matrix`, esas dos variables formaban parte de una clausura. Esto es posible porque cuando se invoca un generador, su estado local se conserva entre reentradas.
- Estamos usando dos bucles anidados para iterar sobre los elementos de la matriz (filas y celdas, respectivamente). Esto es ciertamente más intuitivo que intentar imaginar un bucle que invoca el método `next()` de un iterador, y no necesitamos hacer todo el seguimiento con índices que estábamos haciendo en la implementación anterior.

Como podemos ver, los generadores son una excelente alternativa a escribir objetos iteradores e iterables desde cero. Mejorarán la legibilidad de nuestra rutina de iteración al tiempo que ofrecen el mismo nivel de funcionalidad.

¡Espera! Hay un truco más que los generadores nos ofrecen para hacer este código aún más conciso. Estamos hablando de la sintaxis de delegación de generador: `yield* iterable`. Este es otro ejemplo de una sintaxis integrada de JavaScript que acepta un iterable como argumento. La instrucción recorrerá los elementos del iterable y producirá cada elemento uno por uno. Es efectivamente un azúcar sintáctico conveniente para lidiar con iteradores anidados. Entonces, si queremos aprovechar esto, podríamos simplificar la lógica de iteración de nuestra clase `Matrix` aún más:

```javascript
export class Matrix {
  // ... rest of the methods (unchanged)
  *[Symbol.iterator]() {
    for (const row of this.data) {
      yield* row
    }
  }
}
```

¡Mira eso! ¡No más bucles anidados! Ahora bien, ¿podemos hacerlo aún mejor y eliminar incluso el bucle restante? Mira este código:

```javascript
export class Matrix {
  // ... rest of the methods (unchanged)
  *[Symbol.iterator]() {
    yield* this.data.flat()
  }
}
```

¡Excelente, sin bucles!

Ten en cuenta que aquí estamos aprovechando la función `Array.flat()` para convertir un array bidimensional en un array plano. Luego, usamos la sintaxis de delegación de generador para producir todos sus elementos, uno por uno.

#### Iteradores asíncronos

Los iteradores que hemos visto hasta ahora devuelven un valor sincrónicamente desde su método `next()`. Sin embargo, en JavaScript (y especialmente en Node.js), es muy común tener iteraciones sobre colecciones que requieren que se complete una operación asíncrona antes de que se pueda producir un elemento.

Imagina, por ejemplo, iterar sobre las solicitudes recibidas por un servidor HTTP, o sobre los resultados de una consulta SQL, o sobre los elementos de una API REST paginada. En todas esas situaciones, sería útil poder devolver una promesa desde el método `next()` de un iterador, o mejor aún, usar la construcción `async/await`.

Bueno, eso es exactamente lo que son los iteradores asíncronos (*async iterators*); son iteradores que devuelven una promesa cuando llamas a `next()`. Dado que ese es el único requisito adicional, significa que simplemente podemos definir `next()` como una función asíncrona (`async`). De manera similar, los **iterables asíncronos** (*async iterables*) son objetos que implementan un método `@@asyncIterator`, o en otras palabras, un método accesible a través de la clave `Symbol.asyncIterator`, que devuelve (sincrónicamente) un iterador asíncrono.

Puedes recorrer los elementos de iterables asíncronos utilizando la sintaxis especial `for await...of`, que solo se puede utilizar dentro de un contexto `async`. Con la sintaxis `for await...of`, esencialmente estamos implementando un flujo de ejecución asíncrono secuencial sobre el patrón Iterator. Esencialmente, si tenemos el siguiente código:

```javascript
for await (const value of iterable) {
  console.log(value);
}
```

Es solo azúcar sintáctico para el siguiente bucle:

```javascript
const asyncIterator = iterable[Symbol.asyncIterator]()
let iterationResult = await asyncIterator.next()
while (!iterationResult.done) {
  console.log(iterationResult.value)
  iterationResult = await asyncIterator.next()
}
```

Esto significa que la sintaxis `for await...of` también se puede usar para iterar sobre un iterable simple (no solo iterables asíncronos), como sobre un array de promesas. Funcionará incluso si no todos (o ninguno de) los elementos del iterador son promesas.

Para ver un caso de uso más práctico para iterables asíncronos y la sintaxis `for await...of`, construyamos una clase que tome una lista de URLs como entrada y nos permita iterar sobre su estado de disponibilidad (*up/down*). Llamemos a la clase `CheckUrls`:

```javascript
// checkUrls.js
export class CheckUrls {
  #urls

  constructor(urls) {
    this.#urls = urls // 1
  }

  [Symbol.asyncIterator]() {
    const urlsIterator = Iterator.from(this.#urls) // 2
    return {
      async next() { // 3
        const iteratorResult = urlsIterator.next() // 4
        if (iteratorResult.done) {
          return { done: true }
        }
        const url = iteratorResult.value
        try {
          const checkResult = await fetch(url, { // 5
            method: 'HEAD',
            redirect: 'follow',
            signal: AbortSignal.timeout(5000),
          })
          if (!checkResult.ok) {
            return {// 5a
              done: false,
              value: `${url} is down, error: ${checkResult.status} `+
                `${checkResult.statusText}`,
            }
          }
          return {// 5b
            done: false,
            value: `${url} is up, status: ${checkResult.status}`,
          }
        } catch (err) {
          return {// 5c
            done: false,
            value: `${url} is down, error: ${err.message}`,
          }
        }
      },
    }
  }
}
```

Analicemos las partes más importantes del código anterior:
1. El constructor de la clase `CheckUrls` toma como entrada una lista de URLs. Esta lista puede ser un array o cualquier otro objeto iterable.
2. En nuestro método `@@asyncIterator`, obtenemos un iterador del objeto `this.#urls`, que, como acabamos de decir, debería ser un iterable. Podemos hacer eso simplemente usando el asistente `Iterator.from()`.
3. Observa cómo el método `next()` es ahora una función `async`. Esto significa que siempre devolverá una promesa, como lo requiere el protocolo iterable asíncrono.
4. En el método `next()`, usamos `urlsIterator` para obtener la siguiente URL en la lista, a menos que no haya más, en cuyo caso simplemente devolvemos `{done: true}`.
5. Aquí es donde la naturaleza asíncrona de la verificación cobra vida. La palabra clave `await` pausa la ejecución hasta que se completa la llamada `fetch()`, enviando una solicitud `HEAD` a la URL actual y recuperando la respuesta. La función `fetch()` está configurada para seguir redirecciones (`redirect: 'follow'`) y para abortar después de 5 segundos (`signal: AbortSignal.timeout(5000)`). La lógica maneja entonces tres resultados potenciales:
   - **Fallo de solicitud (error):** Si la operación `fetch()` encuentra un error (por ejemplo, un problema de conexión de red o un tiempo de espera agotado), lanzará una excepción. Esta excepción es capturada por el bloque `catch`. Luego, el código construye un resultado de error que indica que la URL está inactiva (*down*), incluido el mensaje de error específico de la excepción (`err.message`).
   - **Respuesta exitosa (OK):** Si `fetch()` recibe una respuesta exitosa y `checkResult.ok` es `true`, el código construye un resultado exitoso, indicando que la URL está activa (*up*) y proporcionando el código de estado HTTP.
   - **Respuesta exitosa (pero no OK):** Si `fetch()` recibe con éxito una respuesta del servidor, el código verifica la propiedad `checkResult.ok`. Esta propiedad es `true` si el código de estado HTTP está en el rango 200–299 (lo que indica éxito) y `false` en caso contrario. Si `checkResult.ok` es `false`, el código interpreta esto como un problema (por ejemplo, un 404 Not Found o un 500 Internal Server Error). Construye un resultado que indica que la URL está inactiva, incluido el código de estado HTTP y el texto de estado de la respuesta.

Ahora, usemos la sintaxis `for await...of` que mencionamos anteriormente para iterar sobre un objeto `CheckUrls`:

```javascript
import { CheckUrls } from './checkUrls.js'

const checkUrls = new CheckUrls([
  'https://nodejsdesignpatterns.com',
  'https://example.com',
  'https://mustbedownforsurehopefully.com',
  'https://loige.co',
  'https://mario.fyi',
  'https://httpstat.us/200',
  'https://httpstat.us/301',
  'https://httpstat.us/404',
  'https://httpstat.us/500',
  'https://httpstat.us/200?sleep=6000',
])

for await (const status of checkUrls) {
  console.log(status)
}
```

Al momento de escribir este artículo, este es el resultado que vemos si ejecutamos este código:

```text
https://nodejsdesignpatterns.com is up, status: 200
https://example.com is up, status: 200
https://mustbedownforsurehopefully.com is down, error: fetch failed
https://loige.co is up, status: 200
https://mario.fyi is up, status: 200
https://httpstat.us/200 is up, status: 200
https://httpstat.us/301 is up, status: 200
https://httpstat.us/404 is down, error: 404 Not Found
https://httpstat.us/500 is down, error: 500 Internal Server Error
https://httpstat.us/200?sleep=6000 is down, error: The operation was aborted due to timeout
```

Por supuesto, podrías obtener una respuesta diferente según tu conectividad y la disponibilidad actual de las URLs dadas.

Como podemos ver, la sintaxis `for await...of` es una forma muy intuitiva de iterar sobre un iterable asíncrono y, como veremos en un momento, se puede utilizar junto con algunos iterables integrados interesantes para obtener nuevas formas alternativas de acceder a la información asíncrona.

El bucle `for await...of` (así como su versión sincrónica) llamará al método opcional `return()` del iterador si se interrumpe prematuramente con un `break`, un `return` o una excepción. Esto se puede utilizar para realizar inmediatamente cualquier tarea de limpieza que normalmente se realizaría cuando se completa la iteración.

#### Generadores asíncronos

Además de los iteradores asíncronos, también podemos tener **generadores asíncronos** (*async generators*). Para definir una función generadora asíncrona, simplemente antepón la palabra clave `async` a la definición de la función:

```javascript
async function* generatorFunction() {
  // ...generator body
}
```

Como puedes imaginar, los generadores asíncronos permiten el uso de la instrucción `await` dentro de su cuerpo, y el valor de retorno de su método `next()` es una promesa que se resuelve en un objeto que tiene las propiedades canónicas `done` y `value`. De esta manera, los objetos generadores asíncronos también son iteradores asíncronos válidos. También son iterables asíncronos válidos, por lo que se pueden usar en bucles `for await...of`.

Para demostrar cómo los generadores asíncronos pueden simplificar la implementación de iteradores asíncronos, convirtamos la clase `CheckUrls` que vimos en el ejemplo anterior para usar un generador asíncrono:

```javascript
export class CheckUrls {
  constructor(urls) {
    this.urls = urls
  }

  async *[Symbol.asyncIterator]() {
    for (const url of this.urls) {
      try {
        const checkResult = await fetch(url, {
          method: 'HEAD',
          redirect: 'follow',
          signal: AbortSignal.timeout(5000),
        })
        checkResult.ok
          ? yield `${url} is up, status: ${checkResult.status}`
          : yield `${url} is down, error: ${checkResult.status} `+
            `${checkResult.statusText}`
      } catch (err) {
        yield `${url} is down, error: ${err.message}`
      }
    }
  }
}
```

Curiosamente, el uso de un generador asíncrono en lugar de un iterador asíncrono básico nos permitió ahorrar bastantes líneas de código, y la lógica resultante también es más legible y explícita.

#### Iteradores asíncronos y streams de Node.js

Si nos detenemos por un segundo y pensamos en la relación entre los iteradores asíncronos y los streams legibles de Node.js, nos sorprendería lo similares que son tanto en propósito como en comportamiento. De hecho, podemos decir que los iteradores asíncronos son una construcción de stream, ya que se pueden utilizar para procesar los datos de un recurso asíncrono pieza por pieza, exactamente como sucede con los streams legibles.

No es una coincidencia que `stream.Readable` implemente el método `@@asyncIterator`, convirtiéndolo en un iterable asíncrono. Esto nos proporciona un mecanismo adicional, y probablemente aún más intuitivo, para leer datos de un stream legible, gracias a la construcción `for await...of`.

Para demostrar esto rápidamente, considera el siguiente ejemplo, donde tomamos el stream `stdin` del proceso actual y lo canalizamos al stream de transformación `split()`, que emitirá un nuevo fragmento cuando encuentre un carácter de nueva línea. Luego, iteramos sobre cada línea usando el bucle `for await...of`:

```javascript
import split from 'split2' // v4.2.0

const stream = process.stdin.pipe(split())
for await (const line of stream) {
  console.log(`You wrote: ${line}`)
}
```

Este código de muestra imprimirá de nuevo lo que hayamos escrito en la entrada estándar solo después de haber presionado la tecla Intro (*Return*). Para salir del programa, simplemente puedes presionar `Ctrl + C`.

Como podemos ver, esta forma alternativa de consumir un stream legible es ciertamente muy intuitiva y compacta. El ejemplo anterior también nos muestra cuán similares son los dos paradigmas: iteradores y streams. Son tan similares que pueden interoperar casi a la perfección. Para demostrar este punto aún más, considera que la función `stream.Readable.from(iterable, [options])` toma un iterable como argumento, que puede ser tanto síncrono como asíncrono. La función devolverá un stream legible que envuelve el iterable proporcionado, "adaptando" su interfaz a la de un stream legible (este también es un buen ejemplo del patrón Adapter, que ya conocimos en el [Capítulo 8](https://subscription.packtpub.com/book/web-development/9781803238944/8), Patrones de diseño estructurales).

Entonces, si los streams y los iteradores asíncronos están tan estrechamente relacionados, ¿cuál deberías usar realmente? Como siempre, depende de tu caso de uso, pero aquí hay algunas diferencias clave que pueden ayudarte a guiar tu elección:
- Los **streams** (en modo fluido / *flowing mode*) utilizan un **modelo de empuje** (*push model*): los datos se producen y se empujan a los buffers internos a medida que están disponibles. Los **iteradores asíncronos** utilizan un **modelo de extracción** (*pull model*): el consumidor solicita explícitamente cada fragmento de datos, lo que le otorga un control total sobre el ritmo.
- Los **streams** son ideales para datos binarios y escenarios de alto rendimiento (*high-throughput*), gracias a los mecanismos integrados de almacenamiento en búfer y contrapresión (*backpressure*). Son especialmente útiles cuando se trabaja con archivos, sockets de red o cualquier flujo continuo de datos.
- Los **iteradores asíncronos** brillan por su capacidad de composición y claridad, especialmente en flujos de trabajo asíncronos donde los datos se producen de forma perezosa o bajo demanda. Se integran naturalmente con `for await...of`, lo que los hace perfectos para APIs, datos paginados o lógica personalizada.

En resumen: si necesitas rendimiento, almacenamiento en búfer o integración con las partes internas de Node.js, opta por los streams. Si deseas simplicidad, legibilidad y control basado en extracción, los iteradores asíncronos podrían ser la mejor opción.

Ten en cuenta que también puedes mezclar y combinar streams e iteradores asíncronos. Por ejemplo, el asistente `pipeline()` (discutido en el [Capítulo 6](https://subscription.packtpub.com/book/web-development/9781803238944/6), Programación con Streams) te permite crear una canalización de procesamiento concatenando un array de streams. ¡Pues bien, este asistente admite más que solo streams! En la entrada que proporcionas al asistente, puedes mezclar y combinar streams, iterables (síncronos) e iterables asíncronos.

También podemos iterar sobre un `EventEmitter`. Con la función de utilidad `events.on(emitter, eventName)`, podemos, de hecho, obtener un iterable asíncrono cuyo iterador devolverá todos los eventos que coincidan con el `eventName` especificado.

#### Utilidades de iteradores asíncronos

Como hemos visto, los iteradores sincrónicos en JavaScript se benefician del prototipo dedicado `Iterator` que se puede extender, junto con un rico conjunto de funciones auxiliares integradas como `map()`, `filter()` y `reduce()`. Desafortunadamente, al momento de escribir este artículo, los iteradores asíncronos aún no tienen el mismo nivel de conveniencia integrada.

Sin embargo, la situación está evolucionando. Una propuesta abierta del TC39 ([nodejsdp.link/async-iterator-helpers](https://nodejsdp.link/async-iterator-helpers)) está explorando activamente cómo implementar mejor métodos auxiliares similares para iteradores asíncronos. Un enfoque principal de la conversación gira en torno a la gestión de la concurrencia entre operaciones, una consideración compleja pero crucial para los flujos de datos asíncronos.

Esto significa que, por ahora, careces de la facilidad de uso que ofrecen los ayudantes de iteradores sincrónicos. Sin embargo, esto podría cambiar rápidamente a medida que avanza la propuesta del TC39, ¡así que mantente atento a su desarrollo!

Mientras tanto, existe una solución alternativa simple pero poderosa: los streams de Node.js. Mencionamos en la sección anterior que puedes transformar fácilmente un iterador asíncrono en un stream legible usando `Readable.from()`, y fundamentalmente, los streams legibles ya admiten métodos auxiliares como `map()`, `filter()` y `reduce()`.

Por lo tanto, al combinar estas dos técnicas, puedes crear canalizaciones de procesamiento asíncronas elegantes y eficientes basadas en iteradores asíncronos.

Para ver esta idea en acción, actualicemos nuestra aplicación de verificación de estado de URLs anterior. El iterador asíncrono `CheckUrls` permanece sin cambios, pero esta vez, queremos hacer un poco de análisis sobre los datos que provienen del iterador. Digamos que queremos contar cuántos enlaces están activos (*up*) y cuántos están caídos (*down*). He aquí una posible solución:

```javascript
import { Readable } from 'node:stream'
import { CheckUrls } from './checkUrls.js'

const checkUrls = new CheckUrls([
  'https://nodejsdesignpatterns.com',
  'https://loige.co',
  'https://mario.fyi',
  'https://httpstat.us/200',
  'https://httpstat.us/200?sleep=6000',
])

const stats = await Readable.from(checkUrls)
  .map(status => {
    console.log(status)
    return status
  })
  .reduce(
    (acc, status) => {
      if (status.includes(' is up,')) {
        acc.up++
      } else {
        acc.down++
      }
      return acc
    },
    { up: 0, down: 0 }
  )

console.log(stats)
```

Como puedes ver, creamos un stream legible con `Readable.from()` y luego usamos `map()` para imprimir todos los mensajes de estado que pasan por el stream. Luego, usamos `reduce()` y realizamos algunas coincidencias de cadenas para verificar si el estado actual es activo o inactivo. Finalmente, incrementamos el campo respectivo en el objeto acumulador. Ten en cuenta que el asistente `reduce()` consume el stream legible y devuelve una promesa que se resuelve con el resultado de la operación `reduce()` una vez que finaliza el stream.

Si ejecutas este código, deberías obtener una salida similar a esta:

```text
https://nodejsdesignpatterns.com is up, status: 200
https://loige.co is up, status: 200
https://mario.fyi is up, status: 200
https://httpstat.us/200 is up, status: 200
https://httpstat.us/200?sleep=6000 is down, error: The operation was aborted due to timeout
{ up: 4, down: 1 }
```

#### En la práctica

Los iteradores (en particular, los iteradores asíncronos) están ganando popularidad rápidamente en el ecosistema de Node.js. De hecho, en muchas circunstancias, se están convirtiendo en una alternativa preferida a los streams y están reemplazando los mecanismos de iteración personalizados.

Por ejemplo, los paquetes `@databases/pg`, `@databases/mysql` y `@databases/sqlite` son bibliotecas populares para acceder a bases de datos Postgres, MySQL y SQLite, respectivamente (más información en [nodejsdp.link/atdatabases](https://nodejsdp.link/atdatabases)).

Todos ellos exponen una función llamada `queryStream()`, que devuelve un iterable asíncrono que se puede utilizar para iterar fácilmente sobre los resultados de una consulta, como en este ejemplo:

```javascript
for await (const record of db.queryStream(sql`SELECT * FROM my_table`)) {
  // do something with record
}
```

Internamente, el iterador manejará automáticamente el cursor para el resultado de una consulta, por lo que todo lo que tenemos que hacer es simplemente un bucle con la construcción `for await...of`.

Otro ejemplo de una biblioteca que depende en gran medida de los iteradores para su API es el paquete `zeromq` ([nodejsdp.link/npm-zeromq](https://nodejsdp.link/npm-zeromq)). Veremos un ejemplo detallado de ella en la siguiente sección, sobre el patrón Middleware, a medida que avanzamos hacia otros patrones de comportamiento.

---

### Sección 5: Middleware

Uno de los patrones más distintivos en Node.js es el patrón **Middleware**. Desafortunadamente, también es uno de los más confusos, especialmente para los desarrolladores que provienen del mundo de la programación empresarial. La razón de la desorientación probablemente esté conectada con el significado tradicional del término *middleware*, que, en la jerga de la arquitectura empresarial, representa los diversos paquetes de software que ayudan a abstraer mecanismos de nivel inferior, como las APIs del sistema operativo, las comunicaciones de red, la gestión de memoria, etc., permitiendo al desarrollador centrarse solo en el caso de negocio de la aplicación. En este contexto, el término middleware recuerda temas como CORBA, bus de servicios empresariales (*enterprise service bus* o ESB), Spring, JBoss y WebSphere, pero en su significado más genérico, también puede definir cualquier tipo de capa de software que actúe como pegamento entre los servicios de nivel inferior y la aplicación (literalmente, el software en el medio).

#### Middleware en Express

Express ([nodejsdp.link/express](https://nodejsdp.link/express)) popularizó el término *middleware* en el mundo de Node.js, vinculándolo a un patrón de diseño muy específico. En Express, de hecho, el middleware representa un conjunto de servicios, típicamente funciones, que se organizan en una canalización (*pipeline*) y son responsables de procesar las solicitudes HTTP entrantes y las respuestas relacionadas.

Express es famoso por ser un framework web muy poco obstinado (*non-opinionated*) y minimalista, y el patrón Middleware es la razón principal de ello. El middleware de Express es, de hecho, una estrategia eficaz para permitir a los desarrolladores crear y distribuir fácilmente nuevas funciones que se pueden añadir fácilmente a una aplicación, sin la necesidad de hacer crecer el núcleo minimalista del framework.

Un middleware de Express tiene la siguiente firma:

```javascript
function (req, res, next) { ... }
```

Aquí, `req` es la solicitud HTTP entrante, `res` es la respuesta y `next` es el callback que se invocará cuando el middleware actual haya completado sus tareas, y eso, a su vez, activa el siguiente middleware en la canalización.

Los ejemplos de las tareas realizadas por el middleware de Express incluyen lo siguiente:
- Analizar el cuerpo de la solicitud (*body parsing*).
- Comprimir/descomprimir solicitudes y respuestas.
- Producir registros de acceso (*access logs*).
- Gestionar sesiones.
- Gestionar cookies cifradas.
- Proporcionar protección contra la falsificación de solicitudes entre sitios (CSRF).

Si lo pensamos bien, todas estas son tareas que no están estrictamente relacionadas con la lógica de negocio principal de una aplicación, ni son partes esenciales del núcleo mínimo de un servidor web. Son accesorios, componentes que brindan soporte al resto de la aplicación y permiten que los manejadores de solicitudes reales se concentren solo en su lógica de negocio principal. Esencialmente, esas tareas son "software en el medio".

#### Middleware como patrón

La técnica utilizada para implementar middleware en Express no es nueva; de hecho, puede considerarse la encarnación en Node.js del patrón Intercepting Filter y el patrón Chain of Responsibility. En términos más genéricos, también representa una canalización de procesamiento, lo que nos recuerda a los streams. Hoy en día, en Node.js, la palabra middleware se utiliza mucho más allá de los límites del framework Express e indica un patrón particular mediante el cual un conjunto de unidades de procesamiento, filtros y manejadores, bajo la forma de funciones, se conectan para formar una secuencia asíncrona con el fin de realizar el preprocesamiento y posprocesamiento de cualquier tipo de datos. La principal ventaja de este patrón es la flexibilidad. El patrón Middleware nos permite obtener una infraestructura de plugins con un esfuerzo increíblemente pequeño, proporcionando una forma discreta de extender un sistema con nuevos filtros y manejadores.

Si deseas obtener más información sobre el patrón Intercepting Filter, el siguiente artículo es un buen punto de partida: [nodejsdp.link/intercepting-filter](https://nodejsdp.link/intercepting-filter). De manera similar, una buena descripción general del patrón Chain of Responsibility está disponible en esta URL: [nodejsdp.link/chain-of-responsibility](https://nodejsdp.link/chain-of-responsibility).

El siguiente diagrama muestra los componentes del patrón Middleware:

**Figura 9.5:** La estructura del patrón Middleware.

El componente esencial del patrón es el gestor de middleware (*middleware manager*), que es responsable de organizar y ejecutar las funciones de middleware. Los detalles de implementación más importantes del patrón son los siguientes:
- Se puede registrar un nuevo middleware invocando la función `use()` (el nombre de esta función es una convención común en muchas implementaciones del patrón Middleware, pero podemos elegir cualquier nombre). Por lo general, el nuevo middleware solo se puede agregar al final de la canalización, pero esta no es una regla estricta.
- Cuando se reciben nuevos datos para su procesamiento, el middleware registrado se invoca en un flujo de ejecución secuencial asíncrono. Cada unidad de la canalización recibe como entrada el resultado de la ejecución de la unidad anterior.
- Cada pieza de middleware puede decidir detener el procesamiento posterior de los datos. Esto se puede hacer invocando una función especial, no invocando el callback (en caso de que el middleware use callbacks) o propagando un error. Una situación de error generalmente desencadena la ejecución de otra secuencia de middleware que está específicamente dedicada al manejo de errores.

No existe una regla estricta sobre cómo se procesan y propagan los datos en la canalización. Las estrategias para propagar las modificaciones de datos en la canalización incluyen:
- Aumentar los datos recibidos como entrada con propiedades o funciones adicionales.
- Mantener la inmutabilidad de los datos y devolver siempre copias nuevas como resultado del procesamiento.

El enfoque correcto depende de la forma en que se implemente el gestor de middleware y del tipo de procesamiento realizado por el propio middleware.

#### Creación de un framework de middleware para ZeroMQ

Demostremos ahora el patrón construyendo un framework de middleware alrededor de la biblioteca de mensajería ZeroMQ ([nodejsdp.link/zeromq](https://nodejsdp.link/zeromq)). ZeroMQ (también conocida como ZMQ o ØMQ) proporciona una interfaz simple para intercambiar mensajes atómicos a través de la red utilizando una variedad de protocolos. Destaca por su rendimiento y su conjunto básico de abstracciones está específicamente diseñado para facilitar la implementación de arquitecturas de mensajería personalizadas. Por esta razón, a menudo se elige ZeroMQ para construir sistemas distribuidos complejos.

En el [Capítulo 13](https://subscription.packtpub.com/book/web-development/9781803238944/13), Patrones de mensajería e integración, tendremos la oportunidad de analizar las características de ZeroMQ con más detalle.

La interfaz de ZeroMQ es de bastante bajo nivel, ya que solo nos permite usar cadenas y buffers binarios para los mensajes. Por lo tanto, cualquier codificación o formato personalizado de datos debe ser implementado por los usuarios de la biblioteca.

En el siguiente ejemplo, vamos a construir una infraestructura de middleware para abstraer el preprocesamiento y posprocesamiento de los datos que pasan a través de un socket ZeroMQ, de modo que podamos trabajar de forma transparente con objetos JSON, pero también comprimir sin problemas los mensajes que viajan a través del cable.

#### El gestor de middleware

El primer paso hacia la construcción de una infraestructura de middleware alrededor de ZeroMQ es crear un componente que sea responsable de ejecutar la canalización de middleware cuando se recibe o envía un nuevo mensaje. Para este propósito, creemos un nuevo módulo llamado `zmqMiddlewareManager.js` y definámoslo:

```javascript
export class ZmqMiddlewareManager {
  #socket
  #inboundMiddleware = []
  #outboundMiddleware = []

  constructor(socket) { // 1
    this.#socket = socket
    this.#handleIncomingMessages()
  }

  async #handleIncomingMessages() { // 2
    for await (const [message] of this.#socket) {
      await this.#executeMiddleware(this.#inboundMiddleware, message).catch(
        err => {
          console.error('Error while processing the message', err)
        }
      )
    }
  }

  async send(message) { // 3
    const finalMessage = await this.#executeMiddleware(
      this.#outboundMiddleware,
      message
    )
    return this.#socket.send(finalMessage)
  }

  use(middleware) { // 4
    if (middleware.inbound) {
      this.#inboundMiddleware.push(middleware.inbound)
    }
    if (middleware.outbound) {
      this.#outboundMiddleware.unshift(middleware.outbound)
    }
  }

  async #executeMiddleware(middlewares, initialMessage) { // 5
    let message = initialMessage
    for await (const middlewareFunc of middlewares) {
      message = await middlewareFunc.call(this, message)
    }
    return message
  }
}
```

Analicemos en detalle cómo implementamos `ZmqMiddlewareManager`:
1. En la primera parte de la clase, definimos el constructor que acepta un socket ZeroMQ como argumento, gestionado como una propiedad privada de la clase. También tenemos dos propiedades privadas que contienen dos listas vacías que contendrán nuestras funciones de middleware: una para mensajes entrantes (*inbound*) y otra para mensajes salientes (*outbound*). A continuación, comenzamos a procesar inmediatamente los mensajes procedentes del socket. Hacemos eso en el método privado `handleIncomingMessages()`.
2. En el método `handleIncomingMessages()`, usamos el socket ZeroMQ como un iterable asíncrono y, con un bucle `for await...of`, procesamos cualquier mensaje entrante y lo pasamos por la lista `inboundMiddleware` utilizando la función `executeMiddleware()`.
3. De manera similar, el método `send()` pasará el mensaje recibido como argumento por la canalización `outboundMiddleware` utilizando la función `executeMiddleware()`. El resultado del procesamiento se almacena en la variable `finalMessage` y luego se envía a través del socket.
4. El método `use()` se utiliza para agregar nuevas funciones de middleware a nuestras canalizaciones internas. En nuestra implementación, cada middleware viene en pares; es un objeto que contiene dos propiedades, `inbound` y `outbound`. Cada propiedad se puede utilizar para definir la función de middleware que se agregará a la lista respectiva. Es importante observar aquí que el middleware entrante se inserta al final de la lista `inboundMiddleware`, mientras que el middleware saliente se inserta (usando `unshift()`) al principio de la lista `outboundMiddleware`. Esto se debe a que las funciones de middleware entrantes/salientes complementarias generalmente deben ejecutarse en orden inverso. Por ejemplo, si queremos descomprimir y luego deserializar un mensaje entrante usando JSON, significa que para el saliente, primero debemos serializar y luego comprimir. Esta convención para organizar el middleware en pares no es estrictamente parte del patrón general, sino solo un detalle de implementación de nuestro ejemplo específico.
5. El último método, `executeMiddleware()`, representa el núcleo de nuestro componente, ya que es la parte responsable de ejecutar las funciones de middleware. Cada función en el array de middleware recibido como entrada se ejecuta una tras otra, y el resultado de la ejecución de una función de middleware se pasa a la siguiente. Ten en cuenta que estamos usando la instrucción `await` en cada resultado devuelto por cada función de middleware; esto permite que la función de middleware devuelva un valor sincrónicamente, así como asincrónicamente mediante una promesa. Finalmente, el resultado de la última función de middleware se devuelve al invocador.

Por brevedad, no estamos implementando una canalización de middleware de errores. Normalmente, cuando una función de middleware propaga un error, se ejecuta otro conjunto de funciones de middleware específicamente dedicadas al manejo de errores. Esto se puede implementar fácilmente utilizando la misma técnica que estamos demostrando aquí. Por ejemplo, podríamos aceptar una función `errorMiddleware` adicional (opcional) además de `inboundMiddleware` y `outboundMiddleware`.

#### Implementación del middleware para procesar mensajes

Ahora que hemos implementado nuestro gestor de middleware, podemos crear nuestro primer par de funciones de middleware para demostrar cómo procesar mensajes entrantes y salientes. Como dijimos, uno de los objetivos de nuestra infraestructura de middleware es tener un filtro que serialice y deserialice mensajes JSON. Entonces, creemos un nuevo middleware para encargarnos de esto. En un nuevo módulo llamado `jsonMiddleware.js`, incluyamos el siguiente código:

```javascript
export function jsonMiddleware() {
  return {
    inbound(message) {
      return JSON.parse(message.toString())
    },
    outbound(message) {
      return Buffer.from(JSON.stringify(message))
    },
  }
}
```

La parte `inbound` de nuestro middleware deserializa el mensaje recibido como entrada, mientras que la parte `outbound` serializa los datos en una cadena, que luego se convierte en un buffer.

De manera similar, podemos implementar un par de funciones de middleware en un archivo llamado `zlibMiddleware.js`, para inflar/desinflar (*inflate/deflate*) el mensaje utilizando el módulo central `zlib` ([nodejsdp.link/zlib](https://nodejsdp.link/zlib)):

```javascript
import { inflateRaw, deflateRaw } from 'node:zlib'
import { promisify } from 'node:util'

const inflateRawAsync = promisify(inflateRaw)
const deflateRawAsync = promisify(deflateRaw)

export function zlibMiddleware() {
  return {
    inbound(message) {
      return inflateRawAsync(Buffer.from(message))
    },
    outbound(message) {
      return deflateRawAsync(message)
    },
  }
}
```

En comparación con el middleware JSON, nuestras funciones de middleware zlib son asíncronas y devuelven una promesa como resultado. Como ya sabemos, esto es perfectamente compatible con nuestro gestor de middleware.

Puedes notar cómo el middleware utilizado por nuestro framework es bastante diferente del utilizado en Express. Esto es totalmente normal y una demostración perfecta de cómo podemos adaptar este patrón para satisfacer nuestras necesidades específicas.

#### Uso del framework de middleware de ZeroMQ

Ahora estamos listos para usar la infraestructura de middleware que acabamos de crear. Para hacer eso, vamos a construir una aplicación muy simple, con un cliente que envía un ping a un servidor a intervalos regulares y el servidor responde con un eco del mensaje recibido.

Desde una perspectiva de implementación, vamos a confiar en un patrón de mensajería de solicitud/respuesta (*request/reply*) utilizando el par de sockets `req`/`rep` proporcionado por ZeroMQ ([nodejsdp.link/zmq-req-rep](https://nodejsdp.link/zmq-req-rep)). Luego envolveremos los sockets con nuestro `ZmqMiddlewareManager` para obtener todas las ventajas de la infraestructura de middleware que construimos, incluido el middleware para serializar/deserializar mensajes JSON.

Analizaremos el patrón de solicitud/respuesta y otros patrones de mensajería en el [Capítulo 13](https://subscription.packtpub.com/book/web-development/9781803238944/13), Patrones de mensajería e integración.

#### El servidor

Comencemos creando el lado del servidor de nuestra aplicación en un archivo llamado `server.js`:

```javascript
import zeromq from 'zeromq' // v6.3.0 // 1
import { ZmqMiddlewareManager } from './zmqMiddlewareManager.js'
import { jsonMiddleware } from './jsonMiddleware.js'
import { zlibMiddleware } from './zlibMiddleware.js'

const socket = new zeromq.Reply() // 2
await socket.bind('tcp://127.0.0.1:5000')

const zmqm = new ZmqMiddlewareManager(socket) // 3
zmqm.use(zlibMiddleware())
zmqm.use(jsonMiddleware())
zmqm.use({ // 4
  async inbound(message) {
    console.log('Received', message)
    if (message.action === 'ping') {
      await this.send({ action: 'pong', echo: message.echo })
    }
    return message
  },
})

console.log('Server started')
```

El lado del servidor de nuestra aplicación funciona de la siguiente manera:
1. Primero cargamos las dependencias necesarias. El paquete `zeromq` es esencialmente una interfaz de JavaScript sobre la biblioteca nativa ZeroMQ (consulta [nodejsdp.link/npm-zeromq](https://nodejsdp.link/npm-zeromq)).
2. A continuación, creamos un nuevo socket de respuesta (*Reply*) de ZeroMQ y lo vinculamos al puerto 5000 en localhost.
3. Luego viene la parte donde envolvemos ZeroMQ con nuestro gestor de middleware y luego agregamos los middlewares zlib y JSON.
4. Finalmente, estamos listos para manejar una solicitud proveniente del cliente. Haremos esto simplemente agregando otro middleware, esta vez, usándolo como un manejador de solicitudes.

Dado que nuestro manejador de solicitudes viene después de los middlewares zlib y JSON, recibiremos una versión descomprimida y deserializada del mensaje recibido. Por otro lado, cualquier dato pasado a `send()` será procesado por el middleware saliente, que, en nuestro caso, serializará y luego comprimirá los datos.

#### El cliente

Del lado del cliente de nuestra pequeña aplicación, en un archivo llamado `client.js`, tendremos el siguiente código:

```javascript
import zeromq from 'zeromq' // v6.3.0
import { ZmqMiddlewareManager } from './zmqMiddlewareManager.js'
import { jsonMiddleware } from './jsonMiddleware.js'
import { zlibMiddleware } from './zlibMiddleware.js'

const socket = new zeromq.Request() // 1
await socket.connect('tcp://127.0.0.1:5000')

const zmqm = new ZmqMiddlewareManager(socket)
zmqm.use(zlibMiddleware())
zmqm.use(jsonMiddleware())
zmqm.use({
  inbound(message) {
    console.log('Echoed back', message)
    return message
  },
})

setInterval(() => { // 2
  zmqm
    .send({ action: 'ping', echo: Date.now() })
    .catch(err => console.error(err))
}, 1000)

console.log('Client connected')
```

La mayor parte del código de la aplicación cliente es muy similar al del servidor. Las diferencias notables son:
1. Creamos un socket de solicitud (*Request*) en lugar de un socket de respuesta (*Reply*), y lo conectamos a un host remoto (o local) en lugar de vincularlo a un puerto local. El resto de la configuración del middleware es la misma que en el servidor, excepto por el hecho de que nuestro manejador de solicitudes ahora solo imprime cualquier mensaje que recibe. Esos mensajes deberían ser la respuesta pong a nuestras solicitudes ping.
2. La lógica central de la aplicación cliente es un temporizador que envía un mensaje ping cada segundo.

Ahora estamos listos para probar nuestro par cliente/servidor y ver la aplicación en acción. Primero, inicia el servidor:

```bash
node server.js
```

Luego podemos iniciar el cliente en otra terminal con el siguiente comando:

```bash
node client.js
```

En este punto, deberíamos ver al cliente enviando mensajes y al servidor respondiéndolos como un eco.

Nuestro framework de middleware hizo su trabajo. Nos permitió descomprimir/comprimir y deserializar/serializar nuestros mensajes de forma transparente, dejando a los manejadores libres para centrarse en su lógica de negocio.

#### En la práctica

Abrimos esta sección diciendo que la biblioteca que popularizó el patrón Middleware en Node.js es Express ([nodejsdp.link/express](https://nodejsdp.link/express)). Por lo tanto, podemos decir fácilmente que Express es también el ejemplo más notable del patrón Middleware que existe.

Otros tres ejemplos interesantes son los siguientes:
- **Koa** ([nodejsdp.link/koa](https://nodejsdp.link/koa)) es otra alternativa a Express y ciertamente se inspira en él.
- **Middy** ([nodejsdp.link/middy](https://nodejsdp.link/middy)) es un ejemplo clásico del patrón Middleware aplicado a algo diferente a un framework web. Middy es, de hecho, un motor de middleware para funciones AWS Lambda.
- **Hono** ([nodejsdp.link/hono](https://nodejsdp.link/hono)) es un framework web emergente construido sobre estándares web que promete compatibilidad con cualquier entorno de ejecución de JavaScript (no solo Node.js). Un detalle interesante es que el motor de middleware de Hono ofrece un middleware combinado integrado ([nodejsdp.link/hono-combine](https://nodejsdp.link/hono-combine)), que permite combinar múltiples funciones de middleware en un solo middleware. Proporciona tres funciones: `some` (ejecuta solo uno de los middlewares dados), `every` (ejecuta todos los middlewares dados) y `except` (ejecuta todos los middlewares dados solo si no se cumple una condición). Esto puede permitirte crear un comportamiento condicional potente con muy poco esfuerzo. Por ejemplo, podrías implementar "omitir la limitación de tasa (*rate limiting*) si el usuario tiene una IP determinada" sin tener que crear un middleware personalizado para eso.

A continuación, vamos a explorar el patrón Command, que, como veremos en breve, es un patrón muy flexible y multiforme.

---

### Sección 6: Command (Comando)

Otro patrón de diseño de gran importancia en Node.js es **Command**. En su definición más genérica, podemos considerar un comando como cualquier objeto que encapsula toda la información necesaria para realizar una acción más tarde. Entonces, en lugar de invocar un método o una función directamente, creamos un objeto que representa la intención de realizar dicha invocación. Luego será responsabilidad de otro componente materializar la intención, transformándola en una acción real. Tradicionalmente, este patrón se basa en cuatro componentes principales, como se muestra en la Figura 9.6:

**Figura 9.6:** Los componentes del patrón Command.

La configuración típica del patrón Command se puede describir de la siguiente manera:
- **Command (Comando):** Es el objeto que encapsula la información necesaria para invocar un método o función.
- **Client (Cliente):** Es el componente que crea el comando y lo proporciona al invocador.
- **Invoker (Invocador):** Es el componente responsable de ejecutar el comando en el destino.
- **Target (Destino o receptor):** Es el sujeto de la invocación. Puede ser una función solitaria o un método de un objeto.

Como veremos, estos cuatro componentes pueden variar mucho según la forma en que queramos implementar el patrón. Esto no debería sonar nuevo en este punto. Por ejemplo, no es raro ver que las partes del cliente y del invocador se implementan juntas en la misma clase.

El uso del patrón Command en lugar de ejecutar directamente una operación tiene varias aplicaciones:
- Un comando se puede programar para su ejecución posterior.
- Un comando se puede serializar y enviar fácilmente a través de la red. Esta simple propiedad nos permite distribuir trabajos entre máquinas remotas, transmitir comandos desde el navegador al servidor, crear sistemas de llamada a procedimiento remoto (RPC), etc.
- Los comandos facilitan mantener un historial de todas las operaciones ejecutadas en un sistema.
- Los comandos son una parte importante de algunos algoritmos para la sincronización de datos y la resolución de conflictos.
- Un comando programado para su ejecución se puede cancelar si aún no se ha ejecutado. También se puede revertir (deshacer / *undo*), llevando el estado de la aplicación al punto anterior a la ejecución del comando.
- Se pueden agrupar varios comandos. Esto se puede utilizar para crear transacciones atómicas o para implementar un mecanismo mediante el cual todas las operaciones del grupo se ejecutan a la vez.
- Se pueden realizar diferentes tipos de transformaciones en un conjunto de comandos, como la eliminación de duplicados, la unión y división, o la aplicación de algoritmos más complejos como la transformación operacional (*operational transformation* u OT) ([nodejsdp.link/operational-transformation](https://nodejsdp.link/operational-transformation)), que es la base de la mayoría del software colaborativo en tiempo real de la actualidad, como la edición colaborativa de texto.

La lista anterior nos muestra claramente cuán importante es este patrón, especialmente en una plataforma como Node.js, donde las redes y la ejecución asíncrona son actores esenciales.

Ahora, vamos a explorar con más detalle un par de implementaciones diferentes del patrón Command, solo para darte una idea de su alcance.

#### El patrón Task

Podemos comenzar con la implementación más básica y trivial del patrón Command: el patrón **Task**. La forma más fácil en JavaScript de crear un objeto que represente una invocación es creando una clausura alrededor de una definición de función o una función vinculada (*bound function*):

```javascript
function createTask(target, ...args) {
  return () => {
    target(...args)
  }
}
```

Esto es (en su mayor parte) equivalente a hacer:

```javascript
const task = target.bind(null, ...args)
```

Esto no debería parecer nada nuevo. De hecho, ya hemos utilizado este patrón muchas veces a lo largo del libro y, en particular, en el [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781803238944/4), Patrones de control de flujo asíncrono con callbacks. Esta técnica nos permitió utilizar un componente separado para controlar y programar la ejecución de nuestras tareas, lo cual es esencialmente equivalente al componente invocador del patrón Command.

#### Un comando más complejo

Trabajemos ahora en un ejemplo más articulado aprovechando el patrón Command. Esta vez, queremos admitir la función de deshacer (*undo*) y la serialización. Comencemos con el objetivo de nuestros comandos, un pequeño objeto que es responsable de enviar actualizaciones de estado a un servicio como una plataforma social. Usaremos una maqueta (*mockup*) de dicho servicio por simplicidad (el archivo `statusUpdateService.js`):

```javascript
const statusUpdates = new Map()

export const statusUpdateService = {
  postUpdate(status) {
    const id = Math.floor(Math.random() * 1000000)
    statusUpdates.set(id, status)
    console.log(`Status posted: ${status} (${id})`)
    return id
  },
  destroyUpdate(id) {
    statusUpdates.delete(id)
    console.log(`Status removed: ${id}`)
  },
}
```

El `statusUpdateService` que acabamos de crear representa el destino de nuestro patrón Command. Ahora, implementemos una función factoría que cree un comando para representar la publicación de una nueva actualización de estado. Lo haremos en un archivo llamado `createPostStatusCmd.js`:

```javascript
export function createPostStatusCmd(service, status) {
  let postId = null

  return {
    run() {
      postId = service.postUpdate(status)
    },
    undo() {
      if (postId) {
        service.destroyUpdate(postId)
        postId = null
      }
    },
    serialize() {
      return {
        type: 'status',
        action: 'post',
        status
      }
    },
  }
}
```

La función anterior es una función factoría que produce comandos para modelar intenciones de "publicar estado". Cada comando implementa las siguientes tres funcionalidades:
- Un método `run()` que, cuando se invoca, activará la acción. En otras palabras, implementa el patrón Task que hemos visto antes. El comando, cuando se ejecuta, publicará una nueva actualización de estado utilizando los métodos del servicio de destino.
- Un método `undo()` que revierte los efectos de la operación de publicación. En nuestro caso, simplemente estamos invocando el método `destroyUpdate()` en el servicio de destino.
- Un método `serialize()` que construye un objeto JSON que contiene toda la información necesaria para reconstruir el mismo objeto de comando.

Después de esto, podemos construir un `Invoker`. Podemos comenzar implementando su método `run()` (en el archivo `invoker.js`):

```javascript
export class Invoker {
  #history = []

  run(cmd) {
    this.#history.push(cmd)
    cmd.run()
    console.log('Command executed', cmd.serialize())
  }

  // ...rest of the class
}
```

El método `run()` es la funcionalidad básica de nuestro `Invoker`. Es responsable de guardar el comando en la variable de instancia `history` y luego desencadenar la ejecución del comando en sí.

A continuación, podemos añadir al `Invoker` un nuevo método que retrase la ejecución de un comando:

```javascript
delay(cmd, delay) {
  setTimeout(() => {
    console.log('Executing delayed command', cmd.serialize())
    this.run(cmd)
  }, delay)
}
```

Luego, podemos implementar un método `undo()` que revierta el último comando:

```javascript
undo() {
  const cmd = this.#history.pop()
  if (cmd) {
    cmd.undo()
    console.log('Command undone', cmd.serialize())
  }
}
```

Finalmente, también queremos poder ejecutar un comando en un servidor remoto, serializándolo y luego transfiriéndolo a través de la red utilizando un servicio web:

```javascript
async runRemotely(cmd) {
  await fetch('http://localhost:3000/cmd', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(cmd.serialize()),
  })
  console.log('Command executed remotely', cmd.serialize())
}
```

Ahora que tenemos el comando, el invocador y el destino, el único componente que falta es el cliente, que implementaremos en un archivo llamado `client.js`. Comencemos importando todas las dependencias necesarias e instanciando `Invoker`:

```javascript
import { createPostStatusCmd } from './createPostStatusCmd.js'
import { statusUpdateService } from './statusUpdateService.js'
import { Invoker } from './invoker.js'

const invoker = new Invoker()
```

Luego, podemos crear un comando usando la siguiente línea de código:

```javascript
const command = createPostStatusCmd(statusUpdateService, 'HI!')
```

Ahora tenemos un comando que representa la publicación de un mensaje de estado. Podemos entonces decidir despacharlo inmediatamente:

```javascript
invoker.run(command)
```

¡Vaya, cometimos un error! Revirtamos nuestra línea de tiempo al estado en que estaba antes de publicar el último mensaje:

```javascript
invoker.undo()
```

También podemos decidir programar el mensaje para que se envíe dentro de 3 segundos:

```javascript
invoker.delay(command, 1000 * 3)
```

Alternativamente, podemos distribuir la carga de la aplicación migrando la tarea a otra máquina:

```javascript
invoker.runRemotely(command)
```

El ejemplo que acabamos de revisar demuestra cómo encapsular una operación dentro de un comando puede abrir un mundo de posibilidades. Si bien este simple ejemplo apenas araña la superficie, insinúa las numerosas aplicaciones del mundo real de este patrón de diseño. Por ejemplo, considera una herramienta de colaboración en tiempo real donde los comandos se utilizan para rastrear y aplicar las acciones del usuario en múltiples clientes remotos. Este enfoque garantiza que cada acción realizada por cualquier cliente se registre y se replique en tiempo real, fomentando una colaboración fluida entre los usuarios. Otra aplicación convincente es en un entorno de procesamiento por lotes distribuido. Aquí, cada trabajo se puede poner en cola como un comando, lo que permite a múltiples trabajadores recuperar tareas de la cola y ejecutarlas de forma independiente y en paralelo. Esta configuración no solo mejora la escalabilidad sino que también garantiza una utilización eficiente de los recursos en una red de trabajadores.

En ambos escenarios, el patrón Command demuestra su versatilidad al permitir soluciones modulares y escalables. Transforma las operaciones en unidades manejables que se pueden rastrear, ejecutar y coordinar a través de diversos sistemas.

Por otra parte, vale la pena señalar que un patrón Command completo debe usarse solo cuando sea estrictamente necesario. Vimos cuánto código adicional tuvimos que escribir para simplemente invocar un método de `statusUpdateService`. Si todo lo que necesitamos es solo una invocación, entonces un comando complejo sería excesivo. Si, por el contrario, necesitamos programar la ejecución de una tarea o ejecutar una operación asíncrona, entonces el patrón Task más simple ofrece el mejor compromiso. Si, en cambio, necesitamos funciones más avanzadas, como soporte para deshacer, transformaciones, resolución de conflictos o uno de los otros casos de uso sofisticados que describimos anteriormente, usar una representación más compleja para el comando es casi obligatorio.

#### En la práctica

Un excelente ejemplo de la vida real que utiliza el patrón Command es el SDK de AWS para JavaScript v3 ([nodejsdp.link/aws-sdk](https://nodejsdp.link/aws-sdk)). En versiones anteriores del SDK de AWS para JavaScript, la interacción con los servicios de AWS a menudo implicaba llamar directamente a métodos en objetos de servicio. Aunque funcional, este enfoque a menudo conducía a código estrechamente acoplado y dificultaba la gestión de problemas transversales (*cross-cutting concerns*) como los reintentos de solicitudes, el registro y la autenticación. Con el lanzamiento de la versión 3, el SDK de AWS para JavaScript ha adoptado el patrón Command, ofreciendo una forma más estructurada y flexible de interactuar con los servicios de AWS. Este patrón encapsula cada llamada a la API de AWS como un objeto de comando independiente, desacoplando el código del cliente de los detalles específicos de cómo se ejecuta la llamada a la API.

He aquí un ejemplo de cómo cargar un archivo en el servicio de almacenamiento de objetos S3 utilizando el patrón Command en el SDK de AWS para JavaScript v3:

```javascript
import { PutObjectCommand, S3Client} from '@aws-sdk/client-s3' // v3.750.0
import { createReadStream } from 'node:fs'
import { basename } from 'node:path'

const [bucketName, filePath] = process.argv.slice(2)
if (!(bucketName && filePath)) {
  console.error('Usage: node index.js <bucketName> <filePath>')
  process.exit(1)
}

const s3Client = new S3Client()
const fileStream = createReadStream(filePath)
const key = basename(filePath)

// Set the parameters for the PutObject command.
const params = {
  Bucket: bucketName,
  Key: key,
  Body: fileStream,
}

try {
  // Create a PutObject command object.
  const putObjectCommand = new PutObjectCommand(params)
  // Send the command to S3.
  const data = await s3Client.send(putObjectCommand)
  console.log('Successfully uploaded file to S3', data)
} catch (err) {
  console.log('Error', err)
}
```

Este ejemplo implementa una utilidad CLI simple que permite cargar un archivo determinado desde el disco local a un bucket de S3 remoto. Observa cómo la operación `PutObject` (carga) se abstrae a través del patrón de comando; de hecho, necesitamos crear un `PutObjectCommand` que contenga todos los detalles de la operación que queremos realizar. Luego, podemos enviar este comando usando una instancia de la clase `S3Client`.

---

### Sección 7: Resumen

Abrimos este capítulo con tres patrones estrechamente relacionados: Strategy, State y Template.

- **Strategy** nos permite extraer las partes comunes de una familia de componentes estrechamente relacionados en un componente llamado el contexto y nos permite definir objetos de estrategia que el contexto puede usar para implementar comportamientos específicos.
- El patrón **State** es una variación del patrón Strategy, donde las estrategias se utilizan para modelar el comportamiento de un componente cuando se encuentra bajo diferentes estados.
- El patrón **Template** se puede considerar la versión "estática" del patrón Strategy, donde los diferentes comportamientos específicos se implementan como subclases de la clase plantilla, que modela las partes comunes del componente.

A continuación, aprendimos sobre lo que ahora se ha convertido en un patrón central en Node.js, que es el patrón **Iterator**. Aprendimos cómo JavaScript ofrece soporte nativo para el patrón (con los protocolos de iterador e iterable), y cómo los iteradores asíncronos se pueden utilizar como una alternativa a los patrones complejos de iteración asíncrona e incluso a los streams de Node.js.

Luego, examinamos **Middleware**, que es un patrón muy distintivo nacido dentro del ecosistema de Node.js. Aprendimos cómo se puede utilizar para preprocesar y posprocesar datos y solicitudes.

Finalmente, tuvimos una muestra de las posibilidades que ofrece el patrón **Command**, que se puede utilizar para implementar una gran cantidad de funcionalidades, desde simples funciones de deshacer/rehacer y serialización hasta algoritmos de transformación operacional más complejos.

Hemos llegado ahora al final del último capítulo dedicado a los patrones de diseño "tradicionales". A estas alturas, deberías haber añadido a tu cinturón de herramientas una serie de patrones que te serán enormemente útiles en tus tareas cotidianas de programación.

Habiendo llegado al final de nuestra exploración de los patrones de diseño "tradicionales", es hora de dirigir nuestra atención a un aspecto igualmente crítico del desarrollo de software: las pruebas (*testing*). En el próximo capítulo, cambiaremos de marcha y nos sumergiremos profundamente en los principios y prácticas que aseguran que estos patrones (y tus aplicaciones en su conjunto) funcionen de manera confiable. Exploraremos los diferentes tipos de pruebas, desde pruebas unitarias hasta pruebas de integración (mapeando la pirámide de pruebas), y profundizaremos en el mundo de los dobles de prueba (*test doubles*), aprendiendo a crear stubs, mocks y espías para aislar y verificar tu código. También presentaremos el ejecutor de pruebas (*test runner*) integrado de Node.js, mostrándote cómo escribir y ejecutar pruebas de manera efectiva. Finalmente, demostraremos cómo combinar dobles de prueba con inyección de dependencias para escribir código altamente testeable, y abordaremos brevemente otros enfoques de prueba más especializados. ¡Así que prepárate para subir de nivel en tu juego de pruebas y construir aplicaciones Node.js más robustas!

---

### Sección 8: Ejercicios

- **9.1 Registro con Strategy (Logging with Strategy):** Implementa un componente de registro que tenga al menos los siguientes métodos: `debug()`, `info()`, `warn()` y `error()`. El componente de registro también debe aceptar una estrategia que defina hacia dónde se envían los mensajes de registro. Por ejemplo, podríamos tener `ConsoleStrategy` para enviar los mensajes a la consola, o `FileStrategy` para guardar los mensajes de registro en un archivo.
- **9.2 Registro con Template (Logging with Template):** Implementa el mismo componente de registro que definimos en el ejercicio anterior, pero esta vez, utilizando el patrón Template. Obtendríamos entonces una clase `ConsoleLogger` para registrar en la consola o una clase `FileLogger` para registrar en un archivo. Aprecia las diferencias entre los enfoques Template y Strategy.
- **9.3 Artículo de almacén (Warehouse item):** Imagina que estamos trabajando en un programa de gestión de almacén. Nuestra siguiente tarea es crear una clase para modelar un artículo de almacén y ayudar a rastrearlo. Dicha clase `WarehouseItem` tiene un constructor, que acepta un `id` y el estado inicial del artículo (que puede ser uno de `arriving`, `stored` o `delivered`). Tiene tres métodos públicos:
  - `store(locationId)` mueve el artículo al estado `stored` y registra el `locationId` donde se almacena.
  - `deliver(address)` cambia el estado del artículo a `delivered`, establece la dirección de entrega y borra el `locationId`.
  - `describe()` devuelve una representación en cadena del estado actual del artículo (por ejemplo, "Item 5821 is on its way to the warehouse", o "Item 3647 is stored in location 1ZH3", o "Item 3452 was delivered to John Smith, 1st Avenue, New York").
  - El estado `arriving` solo se puede establecer cuando se crea el objeto, ya que no se puede transicionar a él desde los otros estados. Un artículo no puede volver al estado `arriving` una vez que está `stored` o `delivered`, no se puede volver a mover a `stored` una vez que está `delivered`, y no se puede entregar si no se almacena primero. Usa el patrón State para implementar la clase `WarehouseItem`.
- **9.4 Registro con Middleware (Logging with Middleware):** Reescribe el componente de registro que implementaste para los ejercicios 9.1 y 9.2, pero esta vez, usa el patrón Middleware para posprocesar cada mensaje de registro, permitiendo que diferentes middlewares personalicen cómo manejar los mensajes y cómo emitirlos. Podríamos, por ejemplo, agregar un middleware `serialize()` para convertir los mensajes de registro en una representación de cadena lista para ser enviada a través de la red o guardada en algún lugar. Luego, podríamos agregar un middleware `saveToFile()` que guarde cada mensaje en un archivo. Este ejercicio debería resaltar la flexibilidad y universalidad del patrón Middleware.
- **9.5 Colas con iteradores (Queues with iterators):** Implementa una clase `AsyncQueue` similar a una de las clases `TaskQueue` que definimos en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con promesas y async/await, pero con un comportamiento e interfaz ligeramente diferentes. Dicha clase `AsyncQueue` tendrá un método llamado `enqueue()` para agregar nuevos elementos a la cola y luego expondrá un método `@@asyncIterator`, que debería brindar la capacidad de procesar los elementos de la cola de forma asíncrona, uno a la vez (por lo tanto, con una concurrencia de 1). El iterador asíncrono devuelto por `AsyncQueue` debe terminar solo después de que se invoque el método `done()` de `AsyncQueue` y solo después de que se consuman todos los elementos de la cola. Considera que el método `@@asyncIterator` podría invocarse en más de un lugar, devolviendo así un iterador asíncrono adicional, lo que te permitiría aumentar la concurrencia con la que se consume la cola.
