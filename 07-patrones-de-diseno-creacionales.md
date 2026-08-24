# Parte 2: Patrones de diseño de Node.js

## Capítulo 7: Patrones de diseño creacionales

Esto puede parecer un poco inesperado para un libro con "Patrones de diseño" (*Design Patterns*) en su título, pero se necesitaron seis capítulos para llegar a un punto en el que podamos comenzar a hablar un poco más en profundidad sobre los patrones de diseño, al menos en el sentido convencional. Entonces, ¿qué es un patrón de diseño? En términos simples, un patrón de diseño es una solución reutilizable a un problema recurrente. El término es realmente amplio en su definición y puede abarcar múltiples dominios de una aplicación. Sin embargo, el término a menudo se asocia con un conjunto bien conocido de patrones orientados a objetos que fueron popularizados en los años 90 por el libro *Design Patterns: Elements of Reusable Object-Oriented Software*, Pearson Education, por la legendaria Banda de los Cuatro (*Gang of Four* o GoF): Erich Gamma, Richard Helm, Ralph Johnson y John Vlissides. A menudo nos referiremos a estos conjuntos específicos de patrones como patrones de diseño tradicionales o patrones de diseño de GoF.

Aplicar este conjunto de patrones de diseño orientados a objetos en JavaScript no es tan lineal y formal como lo sería en un lenguaje orientado a objetos clásico. Como sabemos, JavaScript está orientado a objetos, basado en prototipos y tiene tipado dinámico. También trata a las funciones como ciudadanos de primera clase y permite estilos de programación funcional. Estas características hacen de JavaScript un lenguaje muy versátil, que otorga un enorme poder al desarrollador, pero al mismo tiempo provoca la fragmentación de estilos de programación, convenciones, técnicas y, en última instancia, los patrones de su ecosistema. Con JavaScript, hay tantas formas de lograr el mismo resultado que cada desarrollador tiene su propia opinión sobre cuál es la mejor manera de abordar un problema. Una clara demostración de este fenómeno es la abundancia de frameworks y bibliotecas con opiniones firmes (*opinionated libraries*) en el ecosistema de JavaScript; probablemente ningún otro lenguaje haya visto tantos jamás, especialmente desde que Node.js ha brindado nuevas y asombrosas posibilidades a JavaScript y ha creado tantos escenarios nuevos.

En este contexto, la naturaleza de JavaScript también afecta a los patrones de diseño tradicionales. Hay tantas formas en las que los patrones de diseño tradicionales se pueden implementar en JavaScript que la implementación tradicional, fuertemente orientada a objetos, deja de ser relevante.

En algunos casos, la implementación tradicional de estos patrones de diseño ni siquiera es posible porque JavaScript, como sabemos, no tiene clases abstractas ni interfaces reales. Lo que no cambia, sin embargo, es la idea original en la base de cada patrón, el problema que resuelve y los conceptos en el corazón de la solución.

Si estás familiarizado con TypeScript, probablemente sepas que añade características del lenguaje como clases abstractas e interfaces. Estas pueden hacer que sea más fácil expresar algunos patrones de diseño de una manera que se sienta más cercana a las implementaciones originales de GoF. En este libro, sin embargo, nos centramos en JavaScript puro para mostrar cómo funcionan estos patrones utilizando las características centrales del lenguaje en sí. TypeScript es una abstracción en tiempo de compilación. Sus tipos, interfaces y clases abstractas —aunque útiles— desaparecen una vez que el código se transpila, dejando únicamente a JavaScript ejecutándose en el entorno. Es por eso que los autores creen que es más valioso comprender cómo se comportan estos patrones en tiempo de ejecución en lugar de depender demasiado de herramientas que no existen durante la ejecución. Una vez que hayas dominado los fundamentos, estarás en una excelente posición para superponer cualquier abstracción que prefieras, ya sea TypeScript, anotaciones JSDoc o algo completamente diferente.

En este capítulo y en los dos siguientes, veremos cómo algunos de los patrones de diseño de GoF más importantes se aplican a Node.js y su filosofía, redescubriendo así su importancia desde otra perspectiva. Entre estos patrones tradicionales, también echaremos un vistazo a algunos patrones de diseño "menos tradicionales" nacidos dentro del propio ecosistema de JavaScript.

En este capítulo, en particular, analizaremos una clase de patrones de diseño llamados **patrones creacionales**. Como su nombre indica, estos patrones abordan problemas relacionados con la creación de objetos. Por ejemplo, el patrón Factory nos permite encapsular la creación de un objeto dentro de una función. El patrón Revealing Constructor nos permite exponer propiedades y métodos privados de un objeto únicamente durante la creación del mismo, mientras que el patrón Builder simplifica la creación de objetos complejos. Finalmente, el patrón Singleton y el patrón Dependency Injection nos ayudan con la conexión (*wiring*) de módulos dentro de nuestras aplicaciones.

En este capítulo, así como en los dos siguientes, asumimos que tienes alguna noción de cómo funciona la herencia en JavaScript. Ten en cuenta también que a menudo utilizaremos diagramas genéricos y más intuitivos para describir un patrón en lugar del UML estándar. Esto se debe a que muchos patrones pueden tener una implementación basada no solo en clases, sino también en objetos e incluso en funciones.

---

### Sección 1: Factory (Factoría)

Comenzaremos nuestro viaje a partir de uno de los patrones de diseño más comunes en Node.js: **Factory**. Como verás, el patrón Factory es muy versátil y sirve para más de un propósito. Su principal ventaja es su capacidad para desacoplar la creación de un objeto de una implementación específica. Esto nos permite, por ejemplo, crear un objeto cuya clase o forma se determina en tiempo de ejecución.

Una factoría también nos permite exponer una superficie mucho más reducida (*smaller surface area*) que una clase. Mientras que una clase se puede extender, instanciar directamente o incluso parchear sobre la marcha (*monkey-patched*), una factoría —al ser solo una función— ofrece menos formas para que los consumidores hagan un mal uso o interfieran con los aspectos internos de la implementación. Esto hace que la API resultante sea menos propensa a errores y más fácil de mantener.

Finalmente, las factorías también pueden ayudar a forzar la encapsulación aprovechando las clausuras (*closures*), manteniendo los detalles internos verdaderamente privados y ocultos del mundo exterior.

#### Desacoplamiento de la creación de objetos y su implementación

Ya hemos enfatizado el hecho de que, en JavaScript, a menudo se prefiere el paradigma funcional a un diseño puramente orientado a objetos por su simplicidad, usabilidad y reducida superficie expuesta. Esto es especialmente cierto al crear nuevas instancias de objetos. De hecho, invocar una factoría en lugar de crear directamente un nuevo objeto a partir de una clase usando el operador `new` u `Object.create()` es mucho más conveniente y flexible en varios aspectos.

En primer lugar, una factoría nos permite separar la creación de un objeto de su implementación. Esencialmente, una factoría envuelve la creación de una nueva instancia, dándonos más flexibilidad y control en la forma en que lo hacemos. Dentro de la factoría, podemos optar por crear una nueva instancia de una clase utilizando el operador `new`, o aprovechar las clausuras para construir dinámicamente un objeto literal con estado, o incluso devolver un tipo de objeto diferente según una condición particular. El consumidor de la factoría es totalmente agnóstico sobre cómo se lleva a cabo la creación de la instancia. La verdad es que al usar `new`, estamos vinculando nuestro código a una forma específica de crear un objeto, mientras que con una factoría, podemos tener mucha más flexibilidad, casi de forma gratuita. Como ejemplo rápido, consideremos una factoría simple que crea un objeto `Image`:

```javascript
function createImage(name) {
  return new Image(name)
}
const image = createImage('photo.jpeg')
```

La factoría `createImage()` puede parecer totalmente innecesaria; ¿por qué no instanciar la clase `Image` utilizando el operador `new` directamente? ¿Por qué no escribir algo como lo siguiente:

```javascript
const image = new Image(name)
```

Como ya mencionamos, usar `new` vincula nuestro código a un tipo particular de objeto, que, en el caso anterior, es el tipo de objeto `Image`. Una factoría, por otro lado, nos da mucha más flexibilidad. Imagina que queremos refactorizar la clase `Image`, dividiéndola en clases más pequeñas, una para cada formato de imagen que admitimos.

Si expusiéramos una factoría como el único medio para crear nuevas imágenes, podríamos simplemente reescribirla de la siguiente manera, sin romper nada del código existente:

```javascript
const jpgRgx = /\.jpe?g$/
const gifRgx = /\.gif$/
const pngRgx = /\.png$/

function createImage(name) {
  if (name.match(jpgRgx)) {
    return new ImageJpeg(name)
  }
  if (name.match(gifRgx)) {
    return new ImageGif(name)
  }
  if (name.match(pngRgx)) {
    return new ImagePng(name)
  }
  throw new Error('Unsupported format')
}
```

Nuestra factoría también nos permite mantener las clases ocultas y evita que se extiendan o modifiquen (¿recuerdas el principio de reducida superficie expuesta?). En JavaScript, esto se puede lograr exportando únicamente la factoría mientras se mantienen las clases privadas.

#### Un mecanismo para forzar la encapsulación

Una factoría también se puede utilizar como un mecanismo de encapsulación, gracias a las clausuras.

La encapsulación se refiere a controlar el acceso a algunos detalles internos de un componente evitando que el código externo los manipule directamente. La interacción con el componente ocurre solo a través de su interfaz pública, aislando el código externo de los cambios en los detalles de implementación del componente. La encapsulación es un principio fundamental del diseño orientado a objetos, junto con la herencia, el polimorfismo y la abstracción.

En JavaScript, una de las principales formas de forzar la encapsulación es a través de los ámbitos de función (*function scopes*) y las clausuras (*closures*). Una factoría hace que sea sencillo imponer variables privadas. Considera lo siguiente, por ejemplo:

```javascript
function createPerson(name) {
  const privateProperties = {}

  const person = {
    setName(name) {
      if (!name) {
        throw new Error('A person must have a name')
      }
      privateProperties.name = name
    },
    getName() {
      return privateProperties.name
    }
  }

  person.setName(name)
  return person
}
```

En el código anterior, aprovechamos una clausura para crear dos objetos: un objeto `person`, que representa la interfaz pública devuelta por la factoría, y un grupo de `privateProperties` que son inaccesibles desde el exterior y que se pueden manipular solo a través de la interfaz proporcionada por el objeto `person`. Por ejemplo, en el código anterior, nos aseguramos de que el nombre de una persona nunca esté vacío; esto sería imposible de imponer si `name` fuera simplemente una propiedad normal del objeto `person`.

El uso de clausuras no es la única técnica que tenemos para forzar la encapsulación. De hecho, otros posibles enfoques son:
- Usar campos de clase privados (la sintaxis de prefijo almohadilla o hashtag, `#`). Más información sobre esto en [nodejsdp.link/tc39-private-fields](https://nodejsdp.link/tc39-private-fields).
- Usar WeakMaps. Más información sobre esto en [nodejsdp.link/weakmaps-private](https://nodejsdp.link/weakmaps-private).
- Usar símbolos, como se explica en el siguiente artículo: [nodejsdp.link/symbol-private](https://nodejsdp.link/symbol-private).
- Definir variables privadas en un constructor (como recomienda Douglas Crockford: [nodejsdp.link/crockford-private](https://nodejsdp.link/crockford-private)). Este es el enfoque tradicional (*legacy*), pero también el más conocido.
- Utilizar convenciones, por ejemplo, anteponer el nombre de una propiedad con un guion bajo, `_`. Sin embargo, esto técnicamente no impide que un miembro sea leído o modificado desde el exterior.

#### Construcción de un perfilador de código simple

Ahora, trabajemos en un ejemplo completo utilizando una factoría. Construyamos un perfilador de código (*code profiler*) simple: un objeto que ayuda a medir cuánto tiempo tarda en ejecutarse un fragmento de código. Los perfiladores se utilizan a menudo para identificar cuellos de botella de rendimiento mediante el seguimiento de la duración de operaciones o funciones específicas. Nuestro perfilador expondrá los siguientes métodos:
- Un método `start()` que activa el inicio de una sesión de perfilado.
- Un método `end()` para terminar la sesión y registrar su tiempo de ejecución en la consola.

Comencemos creando un archivo llamado `profiler.js`, que tendrá el siguiente contenido:

```javascript
class Profiler {
  constructor (label) {
    this.label = label
    this.lastTime = null
  }

  start() {
    this.lastTime = process.hrtime()
  }

  end() {
    const diff = process.hrtime(this.lastTime)
    console.log(`Timer "${this.label}" took ${diff[0]} seconds ` +
      `and ${diff[1]} nanoseconds.`)
  }
}
```

La clase `Profiler` que acabamos de definir utiliza el temporizador de alta resolución predeterminado de Node.js para guardar la hora actual cuando se invoca `start()`, y luego calcular el tiempo transcurrido cuando se ejecuta `end()`, imprimiendo el resultado en la consola.

Ahora, si vamos a utilizar dicho perfilador en una aplicación del mundo real para calcular el tiempo de ejecución de diferentes rutinas, podemos imaginar fácilmente la enorme cantidad de información de perfilado impresa en la consola, especialmente en un entorno de producción. Lo que podríamos querer hacer en su lugar es redirigir la información de perfilado a otra fuente, por ejemplo, un archivo de registro dedicado, o deshabilitar el perfilador por completo si la aplicación se está ejecutando en modo de producción. Está claro que si fuéramos a instanciar un objeto `Profiler` directamente utilizando el operador `new`, necesitaríamos algo de lógica adicional en el código cliente o en el propio objeto `Profiler` para cambiar entre las diferentes lógicas.

Alternativamente, podemos usar una factoría para abstraer la creación del objeto `Profiler` de modo que, dependiendo de si la aplicación se ejecuta en modo de producción o de desarrollo, podamos devolver una instancia de `Profiler` completamente funcional o un objeto simulado (*mock object*) con la misma interfaz pero con métodos vacíos. Esto es exactamente lo que vamos a hacer en nuestro módulo `profiler.js`. En lugar de exportar la clase `Profiler`, exportaremos únicamente nuestra factoría. El siguiente es su código:

```javascript
const noopProfiler = {
  start () {},
  end () {}
}

export function createProfiler (label) {
  if (process.env.NODE_ENV === 'production') {
    return noopProfiler
  }

  return new Profiler(label)
}
```

La función `createProfiler()` es nuestra factoría, y su función es abstraer la creación de un objeto `Profiler` de su implementación. Si la aplicación se ejecuta en modo de producción, devolvemos `noopProfiler`, que esencialmente no hace nada, deshabilitando efectivamente cualquier perfilado. Si la aplicación no se ejecuta en modo de producción, creamos y devolvemos una nueva instancia de `Profiler` completamente funcional.

Gracias al tipado dinámico de JavaScript, pudimos devolver un objeto instanciado con el operador `new` en una circunstancia y un simple objeto literal en la otra (esto también se conoce como *duck typing*, y puedes leer más sobre ello en [nodejsdp.link/duck-typing](https://nodejsdp.link/duck-typing)). Esto confirma cómo podemos crear objetos de la forma que queramos dentro de la función factoría. También podríamos ejecutar pasos de inicialización adicionales o devolver un tipo diferente de objeto según condiciones particulares, todo esto mientras aislamos al consumidor del objeto de todos estos detalles. Podemos comprender fácilmente el poder de este patrón tan simple.

En el ejemplo anterior, usamos `NODE_ENV` para construir una factoría inteligente que cambia entre las versiones de producción y desarrollo de nuestro perfilador. Esta puede ser una técnica útil, pero úsala con cuidado. Depender demasiado de `NODE_ENV` para cambiar el comportamiento del código puede hacer que tu código sea más difícil de probar, depurar y mantener, especialmente si ciertas ramas solo se ejecutan en producción. Úsalo para configuración, no para el flujo de control.

Ahora, juguemos un poco con nuestra factoría de perfiladores. Creemos un algoritmo para calcular todos los factores de un número dado y usemos nuestro perfilador para registrar su tiempo de ejecución:

```javascript
// index.js
import { createProfiler } from './profiler.js'

function getAllFactors(n) {
  let intNumber = n
  const profiler = createProfiler(`Finding all factors of ${intNumber}`)
  profiler.start()
  const factors = []
  for (let factor = 2; factor <= intNumber; factor++) {
    while (intNumber % factor === 0) {
      factors.push(factor)
      intNumber /= factor
    }
  }
  profiler.end()
  return factors
}

const myNumber = process.argv[2]
const myFactors = getAllFactors(myNumber)
console.log(`Factors of ${myNumber} are: `, myFactors)
```

La variable `profiler` contiene nuestro objeto `Profiler`, cuya implementación será decidida por la factoría `createProfiler()` en tiempo de ejecución, en función de la variable de entorno `NODE_ENV`.

Por ejemplo, si ejecutamos el módulo en modo de producción, no obtendremos información de perfilado:

```bash
NODE_ENV=production node index.js 2201307499
```

Sin embargo, si ejecutamos el módulo en modo de desarrollo, veremos la información de perfilado impresa en la consola:

```bash
node index.js 2201307499
```

Salida:

```text
Timer "Finding all factors of 2201307499" took 0 seconds and 6010750 nanoseconds.
Factors of 2201307499 are: [ 38737, 56827 ]
```

El ejemplo que acabamos de presentar es solo una aplicación simple del patrón de función Factoría, pero muestra claramente las ventajas de separar la creación de un objeto de su implementación.

#### En la práctica

Como dijimos, las factorías son muy comunes en Node.js. Podemos encontrar un ejemplo en el popular paquete Knex ([nodejsdp.link/knex](https://nodejsdp.link/knex)). Knex es un constructor de consultas SQL que admite múltiples bases de datos. Su paquete exporta solo una función, que es una factoría. La factoría realiza varias comprobaciones, selecciona el objeto de dialecto correcto a utilizar en función del motor de base de datos y, finalmente, crea y devuelve el objeto Knex. Echa un vistazo al código en [nodejsdp.link/knex-factory](https://nodejsdp.link/knex-factory).

---

### Sección 2: Builder (Constructor)

**Builder** es un patrón de diseño creacional que simplifica la creación de objetos complejos al proporcionar una interfaz fluida (*fluent interface*), lo que nos permite construir el objeto paso a paso. Esto mejora en gran medida la legibilidad y la experiencia general del desarrollador al crear tales objetos complejos.

La situación más evidente en la que podríamos beneficiarnos del patrón Builder es una clase con un constructor que tiene una larga lista de argumentos o toma muchos parámetros complejos como entrada. Por lo general, este tipo de clases requieren tantos parámetros por adelantado porque todos ellos son necesarios para construir una instancia que esté completa y en un estado consistente, por lo que es necesario tener esto en cuenta al considerar posibles soluciones.

Veamos entonces la estructura general del patrón. Imagina que estás construyendo el backend para un videojuego sobre carreras de barcos. Podrías tener una clase `Boat` con un constructor como el siguiente:

```javascript
class Boat {
  constructor (hasMotor, motorCount, motorBrand, motorModel, hasSails, sailsCount, sailsMaterial, sailsColor, hullColor, hasCabin) {
    // ...
  }
}
```

Invocar un constructor de este tipo crearía un código difícil de leer, que es fácilmente propenso a errores. Mirando tu código después de unos días, podrías preguntarte: "¿Qué argumento es qué?". Peor aún si necesitas refactorizar el orden de los argumentos o agregar nuevos argumentos.

Toma el siguiente código, por ejemplo:

```javascript
const myBoat = new Boat(true, 2, 'Best Motor Co. ', 'OM123', true, 1, 'fabric', 'white', 'blue', false)
```

Un primer paso para mejorar el diseño de este constructor es agregar todos los argumentos en un solo objeto literal (generalmente llamado objeto de configuración), como el siguiente:

```javascript
class Boat {
  constructor(allParameters) {
    // ...
  }
}

const myBoat = new Boat({
  hasMotor: true,
  motorCount: 2,
  motorBrand: 'Best Motor Co. ',
  motorModel: 'OM123',
  hasSails: true,
  sailsCount: 1,
  sailsMaterial: 'fabric',
  sailsColor: 'white',
  hullColor: 'blue',
  hasCabin: false
})
```

Como podemos notar en el código anterior, este nuevo constructor es de hecho mucho mejor que el original, ya que nos permite ver claramente qué parámetro recibe cada valor. Además, es fácil agregar nuevos parámetros o cambiar su orden con un impacto limitado en el código existente. Sin embargo, podemos hacerlo incluso mejor que esto. Una desventaja de usar un solo objeto literal para pasar todas las entradas a la vez es que la única forma de saber cuáles son las entradas reales es mirar la documentación de la clase o, peor aún, en el código de la clase. Además de eso, no existe un protocolo impuesto que guíe a los desarrolladores hacia la creación de una clase coherente. Por ejemplo, si especificamos `hasMotor: true`, entonces estamos obligados a especificar también un `motorCount`, un `motorBrand` y un `motorModel`, pero no hay nada en esta interfaz que nos transmita esta información.

El uso de TypeScript podría ayudar a que sea fácil entender qué parámetros están disponibles y cuál es su tipo, pero es posible que tengas que crear una definición de tipo más sofisticada para el objeto de configuración si deseas expresar dependencias entre parámetros. Por ejemplo, podrías crear propiedades anidadas para motor y velas y marcarlas como opcionales, como en el siguiente fragmento:

```typescript
export type BoatConfig = {
  motor?: {
    count: number
    brand: string
    model: string
  }
  sails?: {
    count: number
    material: string
    color: string
  }
  hullColor: string
  hasCabin: boolean
}

class Boat {
  constructor(config: BoatConfig) {
    // ...
  }
}

const myBoat = new Boat({
  motor: {
    count: 2,
    brand: 'Best Motor Co. ',
    model: 'OM123',
  },
  sails: {
    count: 1,
    material: 'fabric',
    color: 'white',
  },
  hullColor: 'blue',
  hasCabin: false,
})
```

Con este enfoque, el sistema de tipos te ayudará a obtener la configuración correcta. Si deseas crear un barco que tenga motor o velas, deberás proporcionar toda la configuración necesaria para esas propiedades como objetos anidados. Si bien este enfoque funciona en casos simples, veremos cómo el patrón Builder ofrece una alternativa más flexible que funciona bien incluso con JavaScript simple, y cómo nos permite mantener plana la lista de parámetros.

El patrón Builder puede ser una excelente solución para estos problemas, ofreciendo una interfaz fluida que es simple de leer, autodocumentada y que guía la creación de un objeto coherente. Veamos la clase `BoatBuilder`, que implementa el patrón Builder para la clase `Boat`:

```javascript
// boat.js
export class BoatBuilder {
  withMotors (count, brand, model) {
    this.hasMotor = true
    this.motorCount = count
    this.motorBrand = brand
    this.motorModel = model
    return this
  }

  withSails (count, material, color) {
    this.hasSails = true
    this.sailsCount = count
    this.sailsMaterial = material
    this.sailsColor = color
    return this
  }

  hullColor (color) {
    this.hullColor = color
    return this
  }

  withCabin () {
    this.hasCabin = true
    return this
  }

  build() {
    return new Boat({
      hasMotor: this.hasMotor,
      motorCount: this.motorCount,
      motorBrand: this.motorBrand,
      motorModel: this.motorModel,
      hasSails: this.hasSails,
      sailsCount: this.sailsCount,
      sailsMaterial: this.sailsMaterial,
      sailsColor: this.sailsColor,
      hullColor: this.hullColor,
      hasCabin: this.hasCabin
    })
  }
}
```

Para apreciar plenamente el impacto positivo que tiene el patrón Builder en la forma en que creamos nuestros objetos `Boat`, veamos un ejemplo de ello:

```javascript
// index.js
const myBoat = new BoatBuilder()
  .withMotors(2, 'Best Motor Co. ', 'OM123')
  .withSails(1, 'fabric', 'white')
  .withCabin()
  .hullColor('blue')
  .build()
```

Como podemos ver, la función de nuestra clase `BoatBuilder` es recopilar todos los parámetros necesarios para crear un `Boat` utilizando algunos métodos auxiliares. Por lo general, tenemos un método para cada parámetro o conjunto de parámetros relacionados, pero no existe una regla exacta para eso. Depende del diseñador de la clase Builder decidir el nombre y el comportamiento de cada método responsable de recopilar los parámetros de entrada.

En su lugar, podemos resumir algunas reglas generales para implementar el patrón Builder, de la siguiente manera:
- El objetivo principal es descomponer un constructor complejo en múltiples pasos más legibles y manejables.
- Intenta crear métodos constructores que puedan establecer múltiples parámetros relacionados a la vez.
- Deduce y establece parámetros implícitamente en función de los valores recibidos como entrada por un método setter y, en general, intenta encapsular la mayor cantidad posible de lógica relacionada con el establecimiento de parámetros en los métodos setter para que el consumidor de la interfaz constructora quede libre de hacerlo.
- Si es necesario, es posible manipular aún más los parámetros (por ejemplo, conversión de tipos, normalización, compatibilidad con valores predeterminados o validación adicional) antes de pasarlos al constructor de la clase que se está construyendo para simplificar aún más el trabajo que le queda por hacer al consumidor de la clase builder.
- Considera proporcionar valores predeterminados razonables para los parámetros que no se establecen explícitamente. Esto hace que el constructor sea más fácil de usar, especialmente cuando algunas opciones de configuración son opcionales o rara vez se personalizan.

En JavaScript, el patrón Builder también se puede aplicar para invocar funciones, no solo para construir objetos utilizando su constructor. De hecho, desde un punto de vista técnico, los dos escenarios son casi idénticos. La principal diferencia al tratar con funciones es que, en lugar de tener un método `build()`, tendríamos un método `invoke()` que invoca la función compleja con los parámetros recopilados por el objeto constructor y devuelve cualquier resultado eventual a quien lo llama.

A continuación, trabajaremos en un ejemplo más concreto que hace uso del patrón Builder que acabamos de aprender.

#### Implementación de un constructor de objetos URL

Queremos implementar una clase `Url` que pueda contener todos los componentes de una URL estándar, validarlos y formatearlos nuevamente en una cadena. Esta clase va a ser intencionalmente simple y mínima, por lo que para un uso estándar en producción, recomendamos la clase `URL` incorporada ([nodejsdp.link/docs-url](https://nodejsdp.link/docs-url)).

Ahora, implementemos nuestra clase `Url` personalizada en un archivo llamado `url.js`:

```javascript
export class Url {
  constructor(protocol, username, password, hostname, port, pathname, search, hash) {
    this.protocol = protocol
    this.username = username
    this.password = password
    this.hostname = hostname
    this.port = port
    this.pathname = pathname
    this.search = search
    this.hash = hash
    this.validate()
  }

  validate() {
    if (!(this.protocol && this.hostname)) {
      throw new Error('Must specify at least a protocol and a hostname')
    }
  }

  toString() {
    let url = ''
    url += `${this.protocol}://`
    if (this.username && this.password) {
      url += `${this.username}:${this.password}@`
    }
    url += this.hostname
    if (this.port) {
      url += this.port
    }
    if (this.pathname) {
      url += this.pathname
    }
    if (this.search) {
      url += `?${this.search}`
    }
    if (this.hash) {
      url += `#${this.hash}`
    }
    return url
  }
}
```

Una URL estándar se compone de varios componentes, por lo que para incorporarlos a todos, el constructor de la clase `Url` es inevitablemente grande. Invocar dicho constructor puede ser un desafío, ya que debemos realizar un seguimiento de la posición del argumento para saber qué componente de la URL estamos pasando. Mira el siguiente ejemplo para tener una idea de esto:

```javascript
return new Url('https', null, null, 'example.com', null, null, null, null)
```

Esta es la situación perfecta para aplicar el patrón Builder sobre el que acabamos de aprender. Hagámoslo ahora. El plan es crear una clase `UrlBuilder`, que tenga un método setter para cada parámetro (o conjunto de parámetros relacionados) necesario para instanciar la clase `Url`. Finalmente, el constructor tendrá un método `build()` para recuperar una nueva instancia de `Url` que se haya creado utilizando todos los parámetros que se han establecido en el constructor. Entonces, implementemos el constructor en un archivo llamado `urlBuilder.js`:

```javascript
export class UrlBuilder {
  setProtocol(protocol) {
    this.protocol = protocol
    return this
  }

  setAuthentication(username, password) {
    this.username = username
    this.password = password
    return this
  }

  setHostname(hostname) {
    this.hostname = hostname
    return this
  }

  setPort(port) {
    this.port = port
    return this
  }

  setPathname(pathname) {
    this.pathname = pathname
    return this
  }

  setSearch(search) {
    this.search = search
    return this
  }

  setHash(hash) {
    this.hash = hash
    return this
  }

  build() {
    return new Url(
      this.protocol,
      this.username,
      this.password,
      this.hostname,
      this.port,
      this.pathname,
      this.search,
      this.hash
    )
  }
}
```

Esto debería ser directo. Simplemente observa la forma en que acoplamos los parámetros de nombre de usuario y contraseña en un único método `setAuthentication()`. Esto transmite claramente el hecho de que si queremos especificar alguna información de autenticación en la URL, debemos proporcionar tanto el nombre de usuario como la contraseña.

Ahora, estamos listos para probar nuestro `UrlBuilder` y presenciar sus beneficios frente al uso directo de la clase `Url`. Podemos hacerlo en un archivo llamado `index.js`:

```javascript
import { UrlBuilder } from './urlBuilder.js'

const url = new UrlBuilder()
  .setProtocol('https')
  .setAuthentication('user', 'pass')
  .setHostname('example.com')
  .build()

console.log(url.toString())
```

Como podemos ver, la legibilidad del código ha mejorado drásticamente. Cada método setter nos da claramente una pista de qué parámetro estamos configurando y, además de eso, proporcionan cierta orientación sobre cómo deben configurarse esos parámetros (por ejemplo, el nombre de usuario y la contraseña deben configurarse juntos).

Ten en cuenta que esta implementación es muy básica y carece de algunas características importantes. Por ejemplo, la funcionalidad de búsqueda (`search`) podría ser más fácil de usar al permitir un array de pares clave-valor (o un mapa u objeto) en lugar de una simple cadena de texto. Esto facilitaría el escape de caracteres especiales automáticamente, de modo que el usuario no tendría que preocuparse por ello.

El patrón Builder también se puede implementar directamente en la clase de destino. Por ejemplo, podríamos haber refactorizado la clase `Url` agregando un constructor vacío (y, por lo tanto, sin validación en el momento de la creación del objeto) y los métodos setter para los diversos componentes, en lugar de crear una clase `UrlBuilder` separada. Sin embargo, este enfoque tiene un gran inconveniente. Usar un constructor que esté separado de la clase de destino tiene la ventaja de producir siempre instancias que se garantiza que estén en un estado consistente. Por ejemplo, podríamos agregar fácilmente alguna validación en el método `UrlBuilder.build()` para asegurarnos de que se hayan establecido el protocolo y el nombre de host. En este punto, se garantiza que cada objeto `Url` devuelto por `UrlBuilder.build()` sea válido y esté en un estado consistente; llamar a `toString()` en dichos objetos siempre devolverá una URL válida. No se puede decir lo mismo si implementamos el patrón Builder en la clase `Url` directamente. De hecho, en este caso, si invocamos `toString()` mientras todavía estamos configurando los distintos componentes de la URL, su valor de retorno puede no ser válido.

#### En la práctica

El patrón Builder es bastante común en Node.js y JavaScript, ya que proporciona una solución muy elegante al problema de crear objetos complejos o invocar funciones complejas. Un ejemplo perfecto es crear nuevas solicitudes de cliente HTTP(S) con la API `request()` de los módulos integrados `http` y `https`.

Si miramos su documentación (disponible en [nodejsdp.link/docs-http-request](https://nodejsdp.link/docs-http-request)), podemos ver inmediatamente que acepta una gran cantidad de opciones, lo cual es la señal habitual de que el patrón Builder puede proporcionar potencialmente una mejor interfaz. De hecho, un popular envoltorio de solicitudes HTTP(S), `superagent` ([nodejsdp.link/superagent](https://nodejsdp.link/superagent)), tiene como objetivo simplificar la creación de nuevas solicitudes implementando el patrón Builder, proporcionando así una interfaz fluida para crear nuevas solicitudes paso a paso. Consulta el siguiente fragmento de código para ver un ejemplo:

```javascript
import superagent from 'superagent' // v10.1.1

superagent
  .post('https://example.com/api/person')
  .send({ name: 'John Doe', role: 'user' })
  .set('accept', 'json')
  .then((response) => {
    // deal with the response
  })
```

A partir del código anterior, podemos notar que este es un constructor inusual; de hecho, no tenemos un método `build()` o `invoke()` (o cualquier otro método con un propósito similar). Lo que desencadena la solicitud en su lugar es una invocación al método `then()`. Es interesante notar que el objeto de solicitud de superagent no es una promesa sino un *thenable* personalizado, donde el método `then()` desencadena la ejecución de la solicitud que hemos construido con el objeto builder. Debido a que JavaScript trata a cualquier *thenable* como una promesa, el mismo comportamiento se aplica cuando se usa `await` en lugar de llamar explícitamente a `.then()`. Por ejemplo, `await superagent.post(...)` seguirá desencadenando la solicitud; de hecho, entre bastidores, la palabra clave `await` simplemente llama al método `.then()` del objeto. Este diseño inteligente permite a la biblioteca proporcionar una API fluida de estilo constructor que se integra a la perfección tanto con el encadenamiento de promesas como con la sintaxis moderna de `async/await`.

Ya discutimos los *thenables* en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con Promesas y Async/Await.

Si echas un vistazo al código de la biblioteca, verás el patrón Builder en acción en la clase `Request` ([nodejsdp.link/superagent-src-builder](https://nodejsdp.link/superagent-src-builder)).

Otro caso de uso común para el patrón Builder es cuando deseas proporcionar una interfaz de JavaScript a un archivo de configuración complejo o un lenguaje de dominio específico (DSL). Un gran ejemplo de esto es una biblioteca de mapeo objeto-relacional (ORM). Los ORMs te permiten interactuar con bases de datos usando objetos JavaScript en lugar de escribir consultas SQL sin procesar. Muchos de ellos incluyen un generador de consultas SQL, a menudo implementado utilizando el patrón Builder, para ayudarte a construir consultas a través de una API fluida y componible. Ya hemos mencionado a Knex como un buen ejemplo del patrón Factory. Knex también ofrece un generador de consultas que aprovecha el patrón Builder (documentación: [nodejsdp.link/knex-query-builder](https://nodejsdp.link/knex-query-builder)).

Para demostrar que este es un caso de uso muy común, veamos también otra biblioteca de ORM SQL famosa: Drizzle ([nodejsdp.link/drizzle](https://nodejsdp.link/drizzle)). Con Drizzle (v0.38.2), dada una conexión de base de datos (`db`) y dos esquemas (`posts` y `comments`) que representan dos tablas de base de datos, podemos construir y ejecutar una consulta SQL de la siguiente manera:

```javascript
await db
  .select()
  .from(posts)
  .leftJoin(comments, eq(posts.id, comments.post_id))
  .where(eq(posts.id, 10))
```

Esto construirá y ejecutará efectivamente la siguiente sentencia SQL:

```sql
SELECT * FROM posts LEFT JOIN comments ON posts.id = comments.post_id WHERE posts.id = 10
```

Ten en cuenta que también en este caso, el patrón Builder aprovecha un *thenable* personalizado, por lo que la construcción solo ocurre cuando llamamos a `await` (que llama implícitamente a `.then()` en el objeto builder).

Quizás un ejemplo un poco más tradicional del uso del patrón Builder en la práctica sea el análisis de argumentos de la CLI. La popular biblioteca `commander.js` ([nodejsdp.link/commander](https://nodejsdp.link/commander)) utiliza otra variación del patrón Builder para permitirte definir cómo analizar los argumentos de la CLI y varias otras opciones de configuración que son típicas de las aplicaciones CLI. Veamos un ejemplo, directamente de la documentación:

```javascript
// cli.js
import { Command } from 'commander' // v14.0.0

const program = new Command()

program
  .name('string-util')
  .description('CLI to some JavaScript string utilities')
  .version('0.8.0')
  .command('split')
  .description('Split a string into substrings and display as an array')
  .argument('<string>', 'string to split')
  .option('--first', 'display just the first substring')
  .option('-s, --separator <char>', 'separator character', ',')
  .action((str, options) => {
    const limit = options.first ? 1 : undefined
    console.log(str.split(options.separator, limit))
  })
  .parse()
```

Este ejemplo implementa una utilidad CLI completa que permite dividir cadenas en la terminal. Por ejemplo, puedes llamarlo con:

```bash
node cli.js --separator "," "Hello, World"
```

Y verás el siguiente resultado en la terminal:

```text
[ 'Hello', ' World' ]
```

Deberías poder apreciar cómo se utiliza aquí el patrón Builder para especificar todos los detalles de este programa: qué opciones se admiten y qué argumentos posicionales, pero también la versión, la descripción y la función específica que debe ejecutarse para el subcomando `split`. La construcción finaliza cuando se llama al método `parse()`. La peculiaridad de esta implementación es que `parse()` no devuelve un nuevo objeto, sino que inicia el análisis de los argumentos, que luego se almacenan internamente en la instancia de `program` y se hacen accesibles a través del objeto `program`. La construcción también evalúa qué comando debe ejecutarse, activando la acción apropiada.

Esto concluye nuestra exploración del patrón Builder. A continuación, veremos el patrón Revealing Constructor.

---

### Sección 3: Revealing Constructor (Constructor revelador)

El patrón **Revealing Constructor** es uno de esos patrones que no encontrarás en el libro de GoF, ya que se originó directamente de JavaScript y la comunidad de Node.js. Resuelve un problema muy complicado, que es: ¿Cómo podemos "revelar" alguna funcionalidad privada de un objeto solo en el momento de la creación del objeto? Esto es particularmente útil cuando queremos permitir que los componentes internos de un objeto se manipulen solo durante su fase de creación. Esto permite algunos escenarios interesantes, como:
- Crear objetos que se pueden modificar solo en el momento de la creación (es decir, se vuelve inmutable después).
- Crear objetos cuyo comportamiento personalizado se puede definir solo en el momento de la creación.
- Crear objetos que se pueden inicializar solo una vez en el momento de la creación.

Estas son solo algunas de las posibilidades que ofrece el patrón Revealing Constructor. Pero para comprender mejor todos los posibles casos de uso, veamos de qué se trata el patrón observando el siguiente fragmento de código:

```javascript
// (1) (2) (3)
const object = new SomeClass(function executor(revealedMembers) {
  // manipulation code ...
})
```

Como podemos ver en el código anterior, el patrón Revealing Constructor consta de tres elementos fundamentales: un constructor (1) que toma una función como entrada (el ejecutor o `executor` (2)), que se invoca en el momento de la creación y recibe un subconjunto de los componentes internos del objeto como entrada (`revealedMembers` (3)).

Para que el patrón funcione, la funcionalidad revelada no debe ser accesible de otro modo para los usuarios del objeto una vez creado. Esto se puede lograr con una de las técnicas de encapsulación que mencionamos en la sección anterior sobre el patrón Factory.

Domenic Denicola fue el primero en identificar y nombrar el patrón en una de las publicaciones de su blog, que se puede encontrar en [nodejsdp.link/domenic-revealing-constructor](https://nodejsdp.link/domenic-revealing-constructor).

Ahora, veamos un par de ejemplos para comprender mejor cómo funciona el patrón Revealing Constructor.

#### Construcción de un buffer inmutable

Los objetos y las estructuras de datos inmutables tienen muchas propiedades excelentes que los hacen ideales para usar en innumerables situaciones en lugar de sus contrapartes mutables (o cambiables). Inmutable se refiere a la propiedad de un objeto por la cual sus datos o estado se vuelven inmodificables una vez creados.

Con los objetos inmutables, no necesitamos crear copias defensivas antes de pasarlos a otras bibliotecas o funciones. Simplemente tenemos una fuerte garantía, por definición, de que no se modificarán, incluso cuando se pasen a código que no conocemos ni controlamos.

La modificación de un objeto inmutable solo se puede realizar mediante la creación de una nueva copia, lo que puede hacer que el código sea más fácil de mantener y razonar. Hacemos esto para que sea más fácil realizar un seguimiento de los cambios de estado.

Otro caso de uso común para objetos inmutables es la detección eficiente de cambios. Dado que cada cambio requiere una copia, y si asumimos que cada copia corresponde a una modificación, detectar un cambio es tan simple como usar el operador de igualdad estricta (o triple igual, `===`). Esta técnica se utiliza ampliamente en la programación frontend para detectar eficientemente si la interfaz de usuario necesita actualizarse.

En este contexto, creemos ahora una versión inmutable simple del componente Buffer de Node.js ([nodejsdp.link/docs-buffer](https://nodejsdp.link/docs-buffer)) utilizando el patrón Revealing Constructor. El patrón nos permite manipular un buffer inmutable solo en el momento de su creación.

Implementemos nuestro buffer inmutable en un nuevo archivo llamado `immutableBuffer.js`, de la siguiente manera:

```javascript
const MODIFIER_NAMES = ['swap', 'write', 'fill']

export class ImmutableBuffer {
  constructor(size, executor) {
    const buffer = Buffer.alloc(size) // 1
    const modifiers = {} // 2
    for (const prop in buffer) { // 3
      if (typeof buffer[prop] !== 'function') {
        continue
      }
      if (MODIFIER_NAMES.some(m => prop.startsWith(m))) { // 4
        modifiers[prop] = buffer[prop].bind(buffer)
      } else {
        this[prop] = buffer[prop].bind(buffer) // 5
      }
    }
    executor(modifiers) // 6
  }
}
```

Veamos ahora cómo funciona nuestra nueva clase `ImmutableBuffer`:
1. Primero, asignamos un nuevo buffer de Node.js (`buffer`) del tamaño especificado en el argumento del constructor `size`.
2. Luego, creamos un objeto literal (`modifiers`) para contener todos los métodos que pueden mutar el buffer.
3. Después de eso, iteramos sobre todas las propiedades (propias y heredadas) de nuestro buffer interno, asegurándonos de omitir todas aquellas que no sean funciones.
4. A continuación, intentamos identificar si la propiedad actual (`prop`) es un método que nos permite modificar el buffer. Hacemos esto intentando hacer coincidir su nombre con una de las cadenas en el array `MODIFIER_NAMES`. Si tenemos dicho método, lo vinculamos a la instancia del buffer y luego lo agregamos al objeto `modifiers`.
5. Si nuestro método no es un método modificador, lo agregamos directamente a la instancia actual (`this`).
6. Finalmente, invocamos la función `executor` recibida como entrada en el constructor y pasamos el objeto `modifiers` como argumento, lo que permitirá a `executor` mutar nuestro buffer interno.

En la práctica, nuestro `ImmutableBuffer` actúa como un proxy entre sus consumidores y el objeto buffer interno. Algunos de los métodos de la instancia del buffer se exponen directamente a través de la interfaz de `ImmutableBuffer` (los métodos de solo lectura), mientras que otros se proporcionan a la función `executor` (los métodos modificadores).

Analizaremos el patrón Proxy con más detalle en el [Capítulo 8](https://subscription.packtpub.com/book/web-development/9781803238944/8), Patrones de diseño estructurales.

Ten en cuenta que esto es solo una demostración del patrón Revealing Constructor, por lo que la implementación del buffer inmutable se mantiene intencionalmente simple. Por ejemplo, no estamos exponiendo el tamaño del buffer ni proporcionando otros medios para inicializar el buffer. Dejaremos esto para ti como ejercicio.

Ahora, escribamos algo de código para demostrar cómo usar nuestra nueva clase `ImmutableBuffer`. Creemos un nuevo archivo, `index.js`, que contenga el siguiente código:

```javascript
import { ImmutableBuffer } from './immutableBuffer.js'

const hello = 'Hello!'
const immutable = new ImmutableBuffer(hello.length, ({ write }) => { // 1
  write(hello)
})

console.log(String.fromCharCode(immutable.readInt8(0))) // 2
// the following line will throw
// "TypeError: immutable.write is not a function"
// immutable.write('Hello?') // 3
```

Lo primero que podemos notar del código anterior es cómo la función `executor` utiliza la función `write()` (que forma parte de los métodos modificadores) para escribir una cadena en el buffer (1). De manera similar, la función `executor` podría haber usado `fill()`, `writeInt8()`, `swap16()` o cualquier otro método expuesto en el objeto `modifiers`.

El código que acabamos de ver también demuestra cómo la nueva instancia de `ImmutableBuffer` expone solo los métodos que no mutan el buffer, como `readInt8()` (2), mientras que no proporciona ningún método para cambiar el contenido del buffer (3).

#### En la práctica

El patrón Revealing Constructor ofrece garantías muy sólidas y, por esta razón, se utiliza principalmente en contextos donde necesitamos proporcionar una encapsulación infalible. Una aplicación perfecta del patrón sería en componentes utilizados por cientos de miles de desarrolladores que tienen que proporcionar interfaces sin opiniones firmes (*unopinionated interfaces*) y una encapsulación estricta. Sin embargo, también podemos usar el patrón en nuestros proyectos para mejorar la confiabilidad y simplificar el intercambio de código con otras personas y equipos (ya que puede hacer que un objeto sea más seguro de usar por parte de terceros).

Una aplicación popular del patrón Revealing Constructor está en la clase `Promise` de JavaScript. Algunos de ustedes ya lo habrán notado. Cuando creamos una nueva `Promise` desde cero, su constructor acepta como entrada una función `executor` que recibirá las funciones `resolve()` y `reject()` utilizadas para mutar el estado interno de la `Promise`. Recordemos cómo se ve esto:

```javascript
return new Promise((resolve, reject) => {
  // ...
})
```

Una vez creada, el estado de la Promise no puede ser alterado por ningún otro medio. Todo lo que podemos hacer es recibir su valor de cumplimiento o motivo de rechazo utilizando los métodos que ya aprendimos en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con Promesas y Async/Await.

Otro ejemplo, una vez más de Domenic Denicola, es una implementación de emisor de eventos donde el comportamiento del publicador de eventos se define en el momento de la construcción a través de un constructor revelador. La idea es que una vez que se construye el emisor, no es posible cambiar la lógica e introducir nuevos eventos. Si tienes curiosidad, puedes consultar la implementación: [nodejsdp.link/revealing-ee](https://nodejsdp.link/revealing-ee).

---

### Sección 4: Singleton

Ahora, vamos a dedicar unas palabras a un patrón que se encuentra entre los más utilizados en la programación orientada a objetos: el patrón **Singleton**. Como veremos, Singleton es uno de esos patrones que tiene una implementación trivial en Node.js que casi no vale la pena discutir. Sin embargo, existen algunas advertencias y limitaciones que todo buen desarrollador de Node.js debe conocer.

El propósito del patrón Singleton es forzar la presencia de una sola instancia de una clase y centralizar su acceso. Hay algunas razones para usar una sola instancia en todos los componentes de una aplicación:
- Para compartir información con estado (*stateful information*).
- Para optimizar el uso de recursos.
- Para sincronizar el acceso a un recurso.

Como puedes imaginar, esos son escenarios bastante comunes. Toma, por ejemplo, una clase `Database` típica, que proporciona acceso a una base de datos:

```javascript
// Database.js
export class Database {
  constructor (dbName, connectionDetails) {
    // ...
  }
  // ...
}
```

Las implementaciones típicas de dicha clase suelen mantener un grupo (*pool*) de conexiones de bases de datos, por lo que no tiene sentido crear una nueva instancia de `Database` para cada solicitud. Además, una instancia de `Database` puede almacenar información con estado, como la lista de transacciones pendientes. Por lo tanto, nuestra clase `Database` cumple con dos criterios para justificar el patrón Singleton. Por ende, lo que normalmente queremos es configurar e instanciar una única instancia de `Database` al inicio de nuestra aplicación y permitir que cada componente use esa única instancia de `Database` compartida.

Muchas personas nuevas en Node.js se confunden sobre cómo implementar el patrón Singleton correctamente; sin embargo, la respuesta es más fácil de lo que podríamos pensar. Simplemente exportar una instancia desde un módulo ya es suficiente para obtener algo muy similar al patrón Singleton. Considera, por ejemplo, las siguientes líneas de código:

```javascript
// dbInstance.js
import { Database } from './Database.js'

export const dbInstance = new Database('my-app-db', {
  url: 'localhost:5432',
  username: 'user',
  password: 'password'
})
```

Al exportar simplemente una nueva instancia de nuestra clase `Database`, ya podemos asumir que dentro del paquete actual (que fácilmente puede ser el código completo de nuestra aplicación), vamos a tener solo una referencia compartida al valor `dbInstance`. Esto es posible porque, como sabemos por el [Capítulo 2](https://subscription.packtpub.com/book/web-development/9781803238944/2), El Sistema de Módulos, Node.js almacenará en caché el módulo, asegurándose de no ejecutar su código en cada importación.

Por ejemplo, podemos obtener fácilmente una instancia compartida del módulo `dbInstance`, que definimos anteriormente, con la siguiente línea de código:

```javascript
import { dbInstance } from './dbInstance.js'
```

Pero hay una advertencia. El módulo se almacena en caché utilizando su ruta completa como clave de búsqueda, por lo que solo se garantiza que sea un singleton dentro del paquete actual. De hecho, cada paquete puede tener su propio conjunto de dependencias privadas dentro de su directorio `node_modules`, lo que podría resultar en múltiples instancias del mismo paquete y, por lo tanto, del mismo módulo, ¡haciendo que nuestro singleton ya no sea realmente único! Este es, por supuesto, un escenario poco común, pero es importante comprender cuáles son sus consecuencias.

Considera, por ejemplo, el caso en el que los archivos `Database.js` y `dbInstance.js` que vimos anteriormente se empaquetan en un paquete llamado `mydb`. Las siguientes líneas de código estarían en su archivo `package.json`:

```json
{
  "name": "mydb",
  "version": "2.0.0",
  "type": "module",
  "main": "dbInstance.js"
}
```

A continuación, considera dos paquetes (`package-a` y `package-b`), los cuales tienen un solo archivo llamado `index.js` que contiene el siguiente código:

```javascript
import { dbInstance } from 'mydb'

export function getDbInstance () {
  return dbInstance
}
```

Tanto `package-a` como `package-b` tienen una dependencia del paquete `mydb`. Sin embargo, `package-a` depende de la versión 1.0.0 del paquete `mydb`, mientras que `package-b` depende de la versión 2.0.0 del mismo paquete (que, para nuestro ejemplo, tiene una implementación mayoritariamente compatible, pero solo una versión diferente especificada en su archivo `package.json`).

Dada la estructura que acabamos de describir, terminaríamos con el siguiente árbol de dependencias de paquetes:

```text
app/
`-- node_modules
    |-- package-a
    |   `-- node_modules
    |       `-- mydb
    `-- package-b
        `-- node_modules
            `-- mydb
```

Terminamos con una estructura de directorios como esta porque `package-a` y `package-b` requieren dos versiones diferentes del módulo `mydb` (por ejemplo, 1.0.0 frente a 2.0.0). En este caso, un administrador de paquetes típico como npm o yarn no "elevará" (*hoist*) la dependencia al directorio superior de `node_modules`, sino que instalará una copia privada de cada paquete.

Con la estructura de directorios que acabamos de ver, tanto `package-a` como `package-b` tienen una dependencia del paquete `mydb`; a su vez, el paquete `app`, que es nuestro paquete raíz, depende tanto de `package-a` como de `package-b`.

El escenario que acabamos de describir romperá la suposición sobre la unicidad de la instancia de la base de datos. De hecho, considera el siguiente archivo (`index.js`) ubicado en la carpeta raíz del paquete `app`:

```javascript
import { getDbInstance as getDbFromA } from 'package-a'
import { getDbInstance as getDbFromB } from 'package-b'

const isSame = getDbFromA() === getDbFromB()
console.log('Is the db instance in package-a the same ' +
  `as package-b? ${isSame ? 'YES' : 'NO'}`)
```

Si ejecutas el archivo anterior, notarás que la respuesta a "¿Es la instancia de base de datos en package-a la misma que en package-b?" es NO. De hecho, `package-a` y `package-b` cargarán dos instancias diferentes del objeto `dbInstance` porque el módulo `mydb` se resolverá en un directorio diferente, dependiendo del paquete desde el que se requiera. Esto rompe claramente las suposiciones del patrón Singleton.

Si, en cambio, tanto `package-a` como `package-b` requirieran dos versiones del paquete `mydb` compatibles entre sí, por ejemplo, `^2.0.1` y `^2.0.7`, entonces el administrador de paquetes instalaría el paquete `mydb` en el directorio de nivel superior de `node_modules` (una práctica conocida como elevación de dependencias o *dependency hoisting*), compartiendo efectivamente la misma instancia con `package-a`, `package-b` y el paquete raíz.

En este punto, podemos decir fácilmente que el patrón Singleton, tal como se describe en la literatura, no existe en Node.js, a menos que usemos una variable global real para almacenarlo, como la siguiente:

```javascript
global.dbInstance = new Database('my-app-db', {/*...*/})
```

Esto garantiza que la instancia sea la única compartida en toda la aplicación y no solo en el mismo paquete. Sin embargo, ten en cuenta que la mayoría de las veces no necesitamos un singleton puro. De hecho, generalmente creamos e importamos singletons dentro del paquete principal de una aplicación o, en el peor de los casos, en un subcomponente de la aplicación que se ha modularizado en una dependencia.

Si estás creando un paquete que va a ser utilizado por terceros, intenta mantenerlo sin estado (*stateless*) para evitar los problemas que hemos discutido en esta sección.

A lo largo de este libro, por simplicidad, utilizaremos el término singleton para describir una instancia de clase o un objeto con estado exportado por un módulo, incluso si esto no representa un singleton real en la definición estricta del término.

Ten en cuenta que esta implementación no es una implementación estricta del patrón Singleton. Si bien exportar una sola instancia (`dbInstance`) desde un módulo te brinda una referencia compartida en toda tu aplicación, no impone estrictamente el patrón Singleton como se definió originalmente en el libro de GoF. De hecho, todavía es posible eludir este mecanismo importando la clase `Database` original y creando una nueva instancia manualmente. Dependiendo de tu caso de uso, este nivel de flexibilidad puede estar perfectamente bien, o puede ser una laguna que preferirías evitar. Si deseas hacer las cosas más estrictas, considera definir la clase `Database` directamente dentro del módulo donde la instancias y exportar solo la instancia singleton. De esa manera, ninguna otra parte de tu base de código puede acceder fácilmente al constructor ni crear instancias adicionales. También vale la pena señalar que, debido a la naturaleza dinámica de JavaScript, incluso ocultar la definición de clase dentro de un módulo solo hace que sea más difícil —no imposible— crear instancias adicionales. JavaScript te permite acceder al constructor original de un objeto a través de la propiedad `constructor`. Esto significa que incluso si la clase `Database` no se exporta, alguien con acceso a la instancia singleton aún podría llamar a `new dbInstance.constructor()` para crear un nuevo objeto. Este es un buen ejemplo de cómo los patrones de diseño tradicionales, como Singleton, a menudo adoptan una forma más relajada y flexible en lenguajes dinámicos como JavaScript. La imposición absoluta es difícil y, en muchos casos, innecesaria. Lo que más importa es establecer límites claros y seguir las convenciones de manera consistente en toda tu base de código.

A continuación, veremos los dos enfoques principales para gestionar las dependencias entre módulos, uno basado en el patrón Singleton y el otro aprovechando el patrón Dependency Injection.

---

### Sección 5: Conexión de módulos (Wiring modules)

Cada aplicación es el resultado de la agregación de varios componentes y, a medida que la aplicación crece, la forma en que conectamos estos componentes se convierte en un factor decisivo para la mantenibilidad y el éxito del proyecto.

Cuando el componente A necesita el componente B para cumplir una funcionalidad determinada, decimos que "A depende de B" o, a la inversa, que "B es una dependencia de A". Para apreciar este concepto, presentemos un ejemplo.

Digamos que queremos escribir una API para un sistema de blogs que usa una base de datos para almacenar sus datos. Podemos tener un módulo genérico que implemente una conexión a la base de datos (`db.js`) y un módulo de blog que exponga la funcionalidad principal para crear y recuperar publicaciones de blog de la base de datos (`blog.js`).

La siguiente figura ilustra la relación entre el módulo de base de datos y el módulo de blog:

**Figura 7.1:** Grafo de dependencias entre el módulo de blog y el módulo de base de datos.

En esta sección, veremos cómo podemos modelar esta dependencia utilizando dos enfoques diferentes, uno basado en el patrón Singleton y el otro utilizando el patrón Dependency Injection.

#### Dependencias Singleton

La forma más sencilla de conectar dos módulos es aprovechando el sistema de módulos de Node.js. Las dependencias con estado conectadas de esta manera son singletons de facto, como analizamos en la sección anterior.

Para ver cómo funciona esto en la práctica, vamos a implementar la sencilla aplicación de blogs que describimos anteriormente utilizando una instancia singleton para la conexión a la base de datos. Veamos una posible implementación de este enfoque en el archivo `db.js`:

```javascript
import { join } from 'node:path'
import sqlite3 from 'sqlite3' // v5.1.7
import { open } from 'sqlite' // v5.1.1

export const db = await open({
  filename: join(import.meta.dirname, 'data.sqlite'),
  driver: sqlite3.Database,
})
```

En el código anterior, estamos usando SQLite ([nodejsdp.link/sqlite](https://nodejsdp.link/sqlite)) como base de datos para almacenar nuestras publicaciones. Para interactuar con SQLite, estamos usando el paquete `sqlite3` ([nodejsdp.link/sqlite3](https://nodejsdp.link/sqlite3)) de npm. SQLite es un sistema de base de datos que guarda todos los datos en un solo archivo local. En nuestro módulo de base de datos, decidimos usar un archivo llamado `data.sqlite` guardado en la misma carpeta que el módulo.

El código anterior crea una nueva instancia de la base de datos que apunta a nuestro archivo de datos y exporta el objeto de conexión de base de datos como un singleton con el nombre `db`.

Ahora, veamos cómo podemos implementar el módulo `blog.js`:

```javascript
import { db } from './db.js'

export class Blog {
  initialize() {
    const initQuery = `CREATE TABLE IF NOT EXISTS posts (
      id TEXT PRIMARY KEY,
      title TEXT NOT NULL,
      content TEXT,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );`
    return db.run(initQuery)
  }

  createPost(id, title, content, createdAt) {
    return db.run(
      'INSERT INTO posts VALUES (?, ?, ?, ?)',
      id,
      title,
      content,
      createdAt
    )
  }

  getAllPosts() {
    return db.all('SELECT * FROM posts ORDER BY created_at DESC')
  }
}
```

El módulo `blog.js` exporta una clase llamada `Blog` que contiene tres métodos:
- `initialize()`: Crea la tabla `posts` si no existe. La tabla se utilizará para almacenar los datos de las publicaciones del blog.
- `createPost()`: Toma todos los parámetros necesarios para crear una publicación. Ejecutará una sentencia `INSERT` para agregar la nueva publicación a la base de datos.
- `getAllPosts()`: Recupera todas las publicaciones disponibles en la base de datos y las devuelve como un array.

Ahora, creemos un módulo (`index.js`) para probar la funcionalidad del módulo de blog que acabamos de crear:

```javascript
import { Blog } from './blog.js'

const blog = new Blog()
await blog.initialize()
const posts = await blog.getAllPosts()
if (posts.length === 0) {
  console.log(
    'No post available. Run `node import-posts.js`' +
    ' to load some sample posts'
  )
}

for (const post of posts) {
  console.log(post.title)
  console.log('-'.repeat(post.title.length))
  console.log(`Published on ${new Date(post.created_at).toISOString()}`)
  console.log(post.content)
}
```

El módulo anterior es muy simple. Recuperamos el array con todas las publicaciones usando `blog.getAllPosts()`, y luego iteramos sobre él y mostramos los datos de cada publicación individual, dándole un poco de formato.

Puedes usar el módulo provisto `import-posts.js` para cargar algunas publicaciones de muestra en la base de datos antes de ejecutar `index.js`. Puedes encontrar `import-posts.js` en el repositorio de código de este libro, junto con el resto de los archivos de este ejemplo.

Como ejercicio divertido, podrías intentar modificar el módulo `index.js` para generar archivos HTML: uno para el índice del blog y luego un archivo dedicado para cada publicación del blog. ¡De esta manera, construirías tu propio generador de sitios web estáticos minimalista!

Como podemos ver en el código anterior, podemos implementar un sistema de gestión de blogs de línea de comandos muy simple aprovechando el patrón Singleton para pasar la instancia `db`. La mayoría de las veces, así es como gestionamos las dependencias con estado en nuestra aplicación; sin embargo, hay situaciones en las que esto puede no ser suficiente.

Usar un singleton, como hemos hecho en el ejemplo anterior, es sin duda la solución más simple, inmediata y legible para pasar dependencias con estado. Sin embargo, ¿qué sucede si queremos simular (*mock*) nuestra base de datos durante nuestras pruebas? ¿Qué podemos hacer si queremos permitir que el usuario de la CLI del blog o de la API del blog seleccione otro backend de base de datos, en lugar del backend estándar de SQLite que proporcionamos de forma predeterminada? Para estos casos de uso, un singleton puede ser un obstáculo para implementar una solución adecuadamente estructurada.

Podríamos introducir sentencias `if` en nuestro módulo `db.js` para elegir diferentes implementaciones según alguna condición ambiental o alguna configuración. Alternativamente, podríamos manipular el sistema de módulos de Node.js para interceptar la importación del archivo de la base de datos y reemplazarlo con otra cosa (simulación de importaciones o *mocking imports*). Pero, como puedes imaginar, estas soluciones distan mucho de ser elegantes.

En la siguiente sección, aprenderemos sobre otra estrategia para conectar módulos, que puede ser la solución ideal para algunos de los problemas que discutimos aquí.

#### Inyección de dependencias (Dependency Injection)

El sistema de módulos de Node.js y el patrón Singleton pueden servir como excelentes herramientas para organizar y conectar los componentes de una aplicación. Sin embargo, estos no siempre garantizan el éxito. Si, por un lado, son fáciles de usar y muy prácticos, por el otro, pueden introducir un acoplamiento más estrecho entre los componentes.

En el ejemplo anterior, podemos ver que el módulo `blog.js` está estrechamente acoplado con el módulo `db.js`. De hecho, nuestro módulo `blog.js` no puede funcionar sin el módulo `database.js` por diseño, ni puede utilizar un módulo de base de datos diferente si es necesario. Podemos solucionar fácilmente este acoplamiento estrecho entre los dos módulos aprovechando el patrón **Dependency Injection (DI)**.

DI es un patrón muy simple en el que las dependencias de un componente son proporcionadas como entrada por una entidad externa, a menudo denominada **inyector** (*injector*).

El inyector inicializa los diferentes componentes y une sus dependencias. Puede ser un simple script de inicialización o un contenedor global más sofisticado que mapea todas las dependencias y centraliza la conexión de todos los módulos del sistema. La principal ventaja de este enfoque es un desacoplamiento mejorado, especialmente para módulos que dependen de instancias con estado (por ejemplo, una conexión a una base de datos). Usando DI, en lugar de estar codificada de forma fija (*hardcoded*) en el módulo, cada dependencia se recibe desde el exterior. Esto significa que el módulo dependiente se puede configurar para usar cualquier dependencia compatible y, por lo tanto, el módulo en sí se puede reutilizar en diferentes contextos con un esfuerzo mínimo.

El siguiente diagrama ilustra esta idea:

**Figura 7.2:** Esquema de DI.

En la Figura 7.2, podemos ver que un servicio genérico espera una dependencia con una interfaz predeterminada. Es responsabilidad del inyector recuperar o crear una instancia concreta real que implemente dicha interfaz y pasarla (o "inyectarla") en el servicio. En otras palabras, el inyector tiene el objetivo de proporcionar una instancia que cumpla con la dependencia del servicio.

Para demostrar este patrón en la práctica, refactoricemos el sencillo sistema de blogs que construimos en la sección anterior utilizando DI para conectar sus módulos. Comencemos refactorizando el módulo `blog.js`:

```javascript
export class Blog {
  constructor(db) {
    this.db = db
  }

  initialize() {
    const initQuery = `CREATE TABLE IF NOT EXISTS posts (
      id TEXT PRIMARY KEY,
      title TEXT NOT NULL,
      content TEXT,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );`
    return this.db.run(initQuery)
  }

  createPost(id, title, content, createdAt) {
    return this.db.run(
      'INSERT INTO posts VALUES (?, ?, ?, ?)',
      id,
      title,
      content,
      createdAt
    )
  }

  getAllPosts() {
    return this.db.all('SELECT * FROM posts ORDER BY created_at DESC')
  }
}
```

Si comparas la nueva versión con la anterior, son casi idénticas. Solo hay dos diferencias pequeñas pero importantes:
- Ya no estamos importando el módulo de la base de datos.
- El constructor de la clase `Blog` toma `db` como argumento.

El nuevo argumento del constructor `db` es la dependencia esperada que debe proporcionar en tiempo de ejecución el componente cliente de la clase `Blog`. Este componente cliente será el inyector de la dependencia. Dado que JavaScript no tiene ninguna forma de representar interfaces abstractas, se espera que la dependencia proporcionada implemente los métodos `db.run()` y `db.all()`. Esto se llama *duck typing*, como se mencionó anteriormente en este libro.

Reescribamos ahora nuestro módulo `db.js`. El objetivo aquí es deshacerse del patrón Singleton y llegar a una implementación que sea más reutilizable y configurable:

```javascript
import { open } from 'sqlite'
import sqlite3 from 'sqlite3'

export function createDb(filename) {
  return open({
    filename,
    driver: sqlite3.Database,
  })
}
```

Esta nueva implementación del módulo `db` proporciona una función factoría llamada `createDb()`, que nos permite crear nuevas instancias de la base de datos en tiempo de ejecución. También nos permite pasar la ruta al archivo de la base de datos en el momento de la creación para que podamos crear instancias independientes que puedan escribir en diferentes archivos de base de datos si es necesario.

En este punto, tenemos casi todos los bloques de construcción en su lugar; solo nos falta el inyector. Daremos un ejemplo del inyector reimplementando el módulo `index.js`:

```javascript
import { join } from 'node:path'
import { Blog } from './blog.js'
import { createDb } from './db.js'

const db = await createDb( // 1
  join(import.meta.dirname, 'data.sqlite'))
const blog = new Blog(db) // 2
await blog.initialize()
const posts = await blog.getAllPosts()
if (posts.length === 0) {
  console.log(
    'No post available. Run `node import-posts.js`' +
    ' to load some sample posts'
  )
}

for (const post of posts) {
  console.log(post.title)
  console.log('-'.repeat(post.title.length))
  console.log(`Published on ${new Date(post.created_at).toISOString()}`)
  console.log(post.content)
}
```

Este código es bastante similar a la implementación anterior, excepto por dos cambios importantes (resaltados en el código anterior):
1. Creamos la dependencia de la base de datos (`db`) usando la función factoría `createDb()`.
2. "Inyectamos" explícitamente la instancia de la base de datos cuando instanciamos la clase `Blog`.

En esta implementación de nuestro sistema de blogs, el módulo `blog.js` está totalmente desacoplado de la implementación real de la base de datos, lo que lo hace más componible y fácil de probar de forma aislada.

Vimos cómo inyectar dependencias como argumentos del constructor (*constructor injection*), pero las dependencias también se pueden pasar al invocar una función o método (*function injection*) o inyectarse explícitamente asignando las propiedades relevantes de un objeto (*property injection*).

Desafortunadamente, las ventajas en términos de desacoplamiento y reutilización que ofrece el patrón DI tienen un precio que pagar. En general, la incapacidad de resolver una dependencia en el momento de la codificación hace que sea más difícil comprender la relación entre los distintos componentes de un sistema. Esto es especialmente cierto en aplicaciones grandes donde podríamos tener una cantidad significativa de servicios con un grafo de dependencias complejo.

Además, si observamos la forma en que instanciamos nuestra dependencia de base de datos en nuestro script de ejemplo anterior, podemos ver que tuvimos que asegurarnos de que la instancia de la base de datos se creara antes de poder invocar cualquier función de nuestra instancia de `Blog`. Esto significa que, cuando se usa en su forma básica, DI nos obliga a construir el grafo de dependencias de toda la aplicación a mano, asegurándonos de hacerlo en el orden correcto. Esto puede volverse inmanejable cuando la cantidad de módulos a conectar es demasiado alta.

Otro patrón, llamado **Inversión de Control** (*Inversion of Control* o IoC), nos permite trasladar la responsabilidad de conectar los módulos de una aplicación a una entidad externa. Esta entidad puede ser un localizador de servicios (*service locator*, un componente simple utilizado para recuperar una dependencia, por ejemplo, `serviceLocator.get('db')`) o un contenedor DI (*DI container*, un sistema que inyecta las dependencias en un componente basado en algunos metadatos especificados en el código mismo o en un archivo de configuración). Puedes encontrar más información sobre estos componentes en el blog de Martin Fowler en [nodejsdp.link/ioc-containers](https://nodejsdp.link/ioc-containers). Aunque estas técnicas se desvían un poco de la forma habitual de hacer las cosas en Node.js, algunas de ellas han ganado popularidad recientemente. Consulta `inversify` ([nodejsdp.link/inversify](https://nodejsdp.link/inversify)) y `awilix` ([nodejsdp.link/awilix](https://nodejsdp.link/awilix)) para obtener más información.

---

### Sección 6: Resumen

En este capítulo, se te presentó un conjunto de patrones de diseño tradicionales relacionados con la creación de objetos. Algunos de esos patrones son tan básicos y, al mismo tiempo, esenciales que probablemente ya los hayas utilizado de una forma u otra.

Patrones como Factory y Singleton son, por ejemplo, dos de los más omnipresentes en la programación orientada a objetos en general. Sin embargo, en JavaScript, su implementación e importancia son muy diferentes de lo propuesto por el libro de GoF. Por ejemplo, Factory se convierte en un patrón muy versátil que funciona en perfecta armonía con la naturaleza híbrida del lenguaje JavaScript, es decir, mitad orientado a objetos y mitad funcional. Por otro lado, Singleton se vuelve tan trivial de implementar que es casi un "no-patrón", pero conlleva una serie de advertencias que deberías haber aprendido a considerar.

Entre los patrones que has aprendido en este capítulo, el patrón Builder puede parecer el que ha conservado la mayor parte de su forma tradicional orientada a objetos. Sin embargo, te hemos demostrado que también se puede utilizar para invocar funciones complejas y no solo para construir objetos.

El patrón Revealing Constructor, por otro lado, merece una categoría propia. Nacido de las necesidades que surgen del propio lenguaje JavaScript, proporciona una solución elegante al problema de "revelar" ciertas propiedades privadas de un objeto únicamente en el momento de la construcción. Proporciona fuertes garantías en un lenguaje que es relajado por naturaleza.

Finalmente, aprendiste sobre las dos técnicas principales para conectar componentes: Singleton y DI. Hemos visto cómo la primera es el enfoque más simple y práctico, mientras que la segunda es más poderosa pero también potencialmente más compleja de implementar.

Como ya mencionamos, este fue el primero de una serie de tres capítulos dedicados enteramente a los patrones de diseño tradicionales. En estos capítulos, intentaremos enseñar el equilibrio adecuado entre creatividad y rigor. Queremos mostrar no solo que existen patrones que se pueden reutilizar para mejorar nuestro código, sino también que su implementación no es el detalle más importante; de hecho, puede variar mucho o incluso superponerse con otros patrones. Lo que realmente importa, sin embargo, es el plano (*blueprint*), las directrices y la idea en la base de cada patrón. Esta es la verdadera información reutilizable que podemos aprovechar para diseñar mejores aplicaciones de Node.js de una manera divertida.

En el próximo capítulo, aprenderás sobre otra categoría de patrones de diseño tradicionales, llamados **patrones estructurales**. Como sugiere el nombre, estos patrones están dirigidos a mejorar la forma en que combinamos objetos para construir estructuras más complejas, pero flexibles y reutilizables.

---

### Sección 7: Ejercicios

- **7.1 Factoría de colores para la consola:** Crea una clase llamada `ColorConsole` que tenga solo un método vacío llamado `log()`. Luego, crea tres subclases: `RedConsole`, `BlueConsole` y `GreenConsole`. El método `log()` de cada subclase de `ColorConsole` aceptará una cadena como entrada e imprimirá esa cadena en la consola utilizando el color que da nombre a la clase. Luego, crea una función factoría que tome un color como entrada, como `'red'`, y devuelva la subclase de `ColorConsole` correspondiente. Finalmente, escribe un pequeño script de línea de comandos para probar la nueva factoría de colores de consola. Puedes utilizar esta respuesta de Stack Overflow como referencia para usar colores en la consola: [nodejsdp.link/console-colors](https://nodejsdp.link/console-colors).
- **7.2 Constructor de solicitudes (Request builder):** Crea tu propia clase Builder alrededor de la función incorporada `http.request()`. El constructor debe poder proporcionar al menos facilidades básicas para especificar el método HTTP, la URL, el componente de consulta (*query*) de la URL, los parámetros de cabecera y los datos del cuerpo eventuales que se enviarán. Para enviar la solicitud, proporciona un método `invoke()` que devuelva una Promise para la invocación. Puedes encontrar la documentación de `http.request()` en la siguiente URL: [nodejsdp.link/docs-http-request](https://nodejsdp.link/docs-http-request).
- **7.3 Una cola a prueba de manipulaciones:** Crea una clase `Queue` que tenga solo un método accesible públicamente llamado `dequeue()`. Dicho método devuelve una Promise que se resuelve con un nuevo elemento extraído de una estructura de datos de cola interna. Si la cola está vacía, la Promise se resolverá cuando se agregue un nuevo elemento. La clase `Queue` también debe tener un constructor revelador que proporcione una función llamada `enqueue()` al ejecutor que empuje un nuevo elemento al final de la cola interna. La función `enqueue()` se puede invocar de forma asíncrona y también debe encargarse de "desbloquear" cualquier Promise eventual devuelta por el método `dequeue()`. Para probar la clase `Queue`, podrías construir un pequeño servidor HTTP dentro de la función ejecutora. Dicho servidor recibiría mensajes o tareas de un cliente y los empujaría a la cola. Luego, un bucle consumiría todos esos mensajes utilizando el método `dequeue()`.
