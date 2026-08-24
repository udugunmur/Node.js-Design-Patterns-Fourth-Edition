# Parte 2: Patrones de diseño de Node.js

## Capítulo 8: Patrones de diseño estructurales

En este capítulo, nos sumergiremos en algunos de los patrones de diseño estructurales más utilizados y veremos cómo se aplican en el mundo de Node.js. Los patrones estructurales nos ayudan a definir relaciones claras entre componentes, lo que permite arquitecturas flexibles y eficientes.

Nos centraremos en tres patrones clave:
- **Proxy:** Controla el acceso a un objeto sustituyéndolo (*standing in for it*).
- **Decorator (Decorador):** Extiende o modifica dinámicamente el comportamiento de un objeto.
- **Adapter (Adaptador):** Conecta interfaces incompatibles para permitir una colaboración fluida.

A lo largo del camino, también abordaremos la programación reactiva (*reactive programming* o RP) y exploraremos Level, un almacén de clave-valor (*key-value store*) rápido y ligero que encaja muy bien en el ecosistema de Node.js. Aprenderás a usarlo e incluso a crear tu propio plugin.

Al final del capítulo, no solo comprenderás cómo funcionan estos patrones estructurales, sino que también sabrás cuándo y cómo aplicarlos eficazmente en aplicaciones de Node.js del mundo real.

---

### Sección 1: Proxy

Un **proxy** es un objeto que controla el acceso a otro objeto, llamado **sujeto** (*subject*). El proxy y el sujeto tienen una interfaz idéntica, y esto nos permite intercambiar uno por el otro de forma transparente; de hecho, el nombre alternativo para este patrón es **sustituto** (*surrogate*).

Un proxy intercepta todas o algunas de las operaciones que están destinadas a ejecutarse en el sujeto, aumentando o complementando su comportamiento. La Figura 8.1 muestra una representación esquemática de este patrón:

**Figura 8.1:** Esquema del patrón Proxy.

La Figura 8.1 nos muestra cómo el proxy y el sujeto tienen la misma interfaz, y cómo esto es transparente para el cliente, quien puede usar uno u otro indistintamente. El proxy reenvía cada operación al sujeto, mejorando su comportamiento con preprocesamiento o posprocesamiento adicional.

Es importante observar que no estamos hablando de crear proxies entre clases; el patrón Proxy implica envolver una instancia real del sujeto, preservando así su estado interno.

Un proxy puede ser útil en varias circunstancias, por ejemplo:
- **Validación de datos:** El proxy valida la entrada antes de reenviarla al sujeto.
- **Seguridad:** El proxy verifica que el cliente esté autorizado para realizar la operación, y pasa la solicitud al sujeto solo si el resultado de la comprobación es positivo.
- **Almacenamiento en caché (Caching):** El proxy mantiene una caché interna para que las operaciones a través del proxy se ejecuten en el sujeto solo si los datos aún no están presentes en la caché.
- **Inicialización perezosa (Lazy initialization):** Si la creación del sujeto es costosa, el proxy puede retrasarla hasta que sea realmente necesaria.
- **Observabilidad:** El proxy intercepta las invocaciones a métodos y los parámetros relativos, registrándolos a medida que ocurren.
- **Objetos remotos:** El proxy puede tomar un objeto remoto y hacer que parezca local.

Hay más aplicaciones del patrón Proxy, pero estas deberían darnos una idea de su propósito.

#### Técnicas para implementar proxies

Al crear un proxy para un objeto, podemos decidir interceptar todos sus métodos o solo algunos de ellos, delegando el resto directamente al sujeto. Hay varias formas de lograr esto y, en esta sección, presentaremos algunas de ellas.

Trabajaremos en un ejemplo simple, una clase `StackCalculator` que se ve así:

```javascript
class StackCalculator {
  constructor() {
    this.stack = []
  }

  putValue(value) {
    this.stack.push(value)
  }

  getValue() {
    return this.stack.pop()
  }

  peekValue() {
    return this.stack[this.stack.length - 1]
  }

  clear() {
    this.stack = []
  }

  divide() {
    const divisor = this.getValue()
    const dividend = this.getValue()
    const result = dividend / divisor
    this.putValue(result)
    return result
  }

  multiply() {
    const multiplicand = this.getValue()
    const multiplier = this.getValue()
    const result = multiplier * multiplicand
    this.putValue(result)
    return result
  }
}
```

Esta clase implementa una versión simplificada de una calculadora basada en pila (*stack calculator*). La idea de esta calculadora es mantener todos los operandos (valores) en una pila. Cuando realizas una operación, por ejemplo una multiplicación, el multiplicando y el multiplicador se extraen de la pila y el resultado de la multiplicación se vuelve a insertar en la pila. Esto no es muy diferente de cómo se implementa la aplicación de calculadora en tu teléfono móvil.

He aquí un ejemplo de cómo podríamos usar `StackCalculator` para realizar algunas multiplicaciones y divisiones:

```javascript
const calculator = new StackCalculator()
calculator.putValue(3)
calculator.putValue(2)
console.log(calculator.multiply()) // 3*2 = 6
calculator.putValue(2)
console.log(calculator.multiply()) // 6*2 = 12
```

También hay algunos métodos de utilidad, como `peekValue()`, que nos permite echar un vistazo al valor en la parte superior de la pila (el último valor insertado o el resultado de la última operación), y `clear()`, que nos permite reiniciar la pila.

Dato curioso: En JavaScript, cuando realizas una división por 0, obtienes un valor misterioso llamado `Infinity`. En muchos otros lenguajes de programación, dividir por 0 es una operación ilegal que hace que el programa entre en pánico (*panic*) o lance una excepción en tiempo de ejecución.

Nuestra tarea en las próximas secciones será aprovechar el patrón Proxy para mejorar una instancia de `StackCalculator` proporcionando un comportamiento más conservador para la división por 0: en lugar de devolver `Infinity`, lanzaremos un error explícito.

#### Composición de objetos (Object composition)

La composición es una técnica mediante la cual un objeto se combina con otro objeto con el propósito de extender o utilizar su funcionalidad. En el caso específico del patrón Proxy, se crea un nuevo objeto con la misma interfaz que el sujeto y una referencia al sujeto se almacena internamente en el proxy en forma de una variable de instancia o una variable de clausura. El sujeto puede ser inyectado desde el cliente en el momento de la creación o creado por el propio proxy.

El siguiente ejemplo implementa una calculadora segura utilizando la composición de objetos:

```javascript
class SafeCalculator {
  constructor(calculator) {
    this.calculator = calculator
  }

  // proxied method
  divide() {
    // additional validation logic
    const divisor = this.calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    // if valid delegates to the subject
    return this.calculator.divide()
  }

  // delegated methods
  putValue(value) {
    return this.calculator.putValue(value)
  }

  getValue() {
    return this.calculator.getValue()
  }

  peekValue() {
    return this.calculator.peekValue()
  }

  clear() {
    return this.calculator.clear()
  }

  multiply() {
    return this.calculator.multiply()
  }
}

const calculator = new StackCalculator()
const safeCalculator = new SafeCalculator(calculator)

calculator.putValue(3)
calculator.putValue(2)
console.log(calculator.multiply()) // 3*2 = 6

safeCalculator.putValue(2)
console.log(safeCalculator.multiply()) // 6*2 = 12

calculator.putValue(0)
console.log(calculator.divide()) // 12/0 = Infinity

safeCalculator.clear()
safeCalculator.putValue(4)
safeCalculator.putValue(0)
console.log(safeCalculator.divide()) // 4/0 -> Error
```

El objeto `safeCalculator` es un proxy para la instancia original de `calculator`. Al invocar `multiply()` en `safeCalculator`, terminaremos llamando al mismo método en `calculator`. Lo mismo ocurre con `divide()`, pero en este caso podemos ver que, si intentamos dividir por cero, obtendremos resultados diferentes dependiendo de si realizamos la división en el sujeto o en el proxy.

Para implementar este proxy mediante composición, tuvimos que interceptar los métodos que nos interesaba manipular (`divide()`), mientras simplemente delegamos el resto de ellos al sujeto (`putValue()`, `getValue()`, `peekValue()`, `clear()` y `multiply()`).

Ten en cuenta que el estado de la calculadora (los valores en la pila) aún lo mantiene la instancia `calculator`; `safeCalculator` solo invocará métodos en `calculator` para leer o mutar el estado según sea necesario. En el método `divide()`, necesitamos acceder al estado para verificar si el divisor es 0. Hacemos esto utilizando el método `peekValue()` directamente en la instancia interna `calculator`. Aparte de usar `peekValue()`, la instancia `safeCalculator` no tiene forma de acceder a todos los demás valores en la pila.

Una implementación alternativa del proxy presentado en el fragmento de código anterior podría usar simplemente un objeto literal y una función factoría:

```javascript
function createSafeCalculator(calculator) {
  return {
    // proxied method
    divide() {
      // additional validation logic
      const divisor = calculator.peekValue()
      if (divisor === 0) {
        throw new Error('Division by 0')
      }
      // if valid delegates to the subject
      return calculator.divide()
    },
    // delegated methods
    putValue(value) {
      return calculator.putValue(value)
    },
    getValue() {
      return calculator.getValue()
    },
    peekValue() {
      return calculator.peekValue()
    },
    clear() {
      return calculator.clear()
    },
    multiply() {
      return calculator.multiply()
    },
  }
}

const calculator = new StackCalculator()
const safeCalculator = createSafeCalculator(calculator)
// ...
```

Esta implementación es más simple y concisa que la basada en clases, pero, una vez más, nos obliga a delegar todos los métodos al sujeto explícitamente.

Tener que delegar muchos métodos para clases complejas puede ser muy tedioso y dificultar la implementación de estas técnicas. Una forma de crear un proxy que delegue la mayoría de sus métodos es utilizar una biblioteca que genere todos los métodos por nosotros, como `delegates` ([nodejsdp.link/delegates](https://nodejsdp.link/delegates)).

`delegates` es una gran biblioteca y te recomendamos que leas su código fuente para ver cómo implementa la delegación de métodos. Pero al mismo tiempo, es importante reconocer que es una biblioteca muy antigua, utiliza algunas características obsoletas de JavaScript y no se ha actualizado en mucho tiempo, por lo que debes evitar usarla en producción.

Otra forma sencilla de reducir la duplicación de código podría ser crear los diversos métodos en el proxy realizando un bucle:

```javascript
function createSafeCalculator(calculator) {
  const safeCalculator = {
    // proxied method
    divide() {
      // ...hidden for brevity
    },
  }
  // delegated methods
  for (const fn of [
    'putValue',
    'getValue',
    'peekValue',
    'clear',
    'multiply'
  ]) {
    safeCalculator[fn] = calculator[fn].bind(calculator)
  }
  return safeCalculator
}
```

Una alternativa más moderna y nativa es utilizar el objeto `Proxy`, que analizaremos más adelante en este capítulo.

#### Aumento de objetos (Object augmentation)

El aumento de objetos (o parcheo dinámico / *monkey patching*) es probablemente la forma más sencilla y común de crear un proxy para unos pocos métodos de un objeto. Implica modificar el sujeto directamente reemplazando un método con su implementación con proxy.

En el contexto de nuestro ejemplo de calculadora, esto podría hacerse de la siguiente manera:

```javascript
function patchToSafeCalculator(calculator) {
  const divideOrig = calculator.divide
  calculator.divide = () => {
    // additional validation logic
    const divisor = calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    // if valid delegates to the subject
    return divideOrig.apply(calculator)
  }
  return calculator
}

const calculator = new StackCalculator()
const safeCalculator = patchToSafeCalculator(calculator)
// ...
```

Esta técnica es definitivamente conveniente cuando necesitamos crear un proxy para uno o unos pocos métodos. ¿Notaste que aquí no tuvimos que volver a implementar el método `multiply()` ni todos los demás métodos delegados?

Desafortunadamente, la simplicidad tiene el costo de tener que mutar el objeto sujeto directamente, lo que puede ser peligroso.

Las mutaciones deben evitarse a toda costa cuando el sujeto se comparte con otras partes de la base de código. De hecho, hacer "monkey patching" en el sujeto podría crear efectos secundarios indeseables que afecten a otros componentes de nuestra aplicación. Utiliza esta técnica solo cuando el sujeto exista en un contexto controlado o en un ámbito privado. Si deseas apreciar por qué el "monkey patching" es una práctica peligrosa, podrías intentar invocar una división por cero en la instancia original `calculator`. Si lo haces, verás que la instancia original ahora arrojará un error en lugar de devolver `Infinity`. El comportamiento original ha sido alterado y esto podría tener efectos inesperados en otras partes de la aplicación.

En la siguiente sección, exploraremos el objeto `Proxy` integrado, que es una poderosa alternativa para implementar el patrón Proxy y más.

#### El objeto Proxy integrado

La especificación ES2015 introdujo una forma nativa de crear potentes objetos proxy.

Estamos hablando del objeto `Proxy` de ES2015, que consta de un constructor `Proxy` que acepta `target` y `handler` como argumentos:

```javascript
const proxy = new Proxy(target, handler)
```

Aquí, `target` representa el objeto sobre el que se aplica el proxy (el sujeto en nuestra definición canónica), mientras que `handler` es un objeto especial que define el comportamiento del proxy.

El objeto `handler` contiene una serie de métodos opcionales con nombres predefinidos llamados **métodos trampa** (*trap methods*, por ejemplo, `apply`, `get`, `set` y `has`) que se llaman automáticamente cuando se realizan las operaciones correspondientes en la instancia del proxy.

Para comprender mejor cómo funciona esta API, veamos cómo podemos usar el objeto `Proxy` para implementar nuestro proxy de calculadora segura:

```javascript
const safeCalculatorHandler = {
  get: (target, property) => {
    if (property === 'divide') {
      // proxied method
      return () => {
        // additional validation logic
        const divisor = target.peekValue()
        if (divisor === 0) {
          throw new Error('Division by 0')
        }
        // if valid delegates to the subject
        return target.divide()
      }
    }
    // delegated methods and properties
    return target[property]
  },
}

const calculator = new StackCalculator()
const safeCalculator = new Proxy(
  calculator,
  safeCalculatorHandler
)
// ...
```

En esta implementación del proxy de calculadora segura utilizando el objeto `Proxy`, adoptamos la trampa `get` para interceptar el acceso a las propiedades y métodos del objeto original, incluidas las llamadas al método `divide()`. Cuando se intercepta el acceso a `divide()`, el proxy devuelve una versión modificada de la función que implementa la lógica adicional para verificar posibles divisiones por cero. Ten en cuenta que simplemente podemos devolver todos los demás métodos y propiedades sin cambios mediante `target[property]`.

Finalmente, es importante mencionar que el objeto `Proxy` hereda el prototipo del sujeto; por lo tanto, ejecutar `safeCalculator instanceof StackCalculator` devolverá `true`.

Con este ejemplo, debería quedar claro que el objeto `Proxy` nos permite evitar mutar el sujeto mientras nos brinda una manera fácil de aplicar proxies solo a las partes que necesitamos mejorar, sin tener que delegar explícitamente todas las demás propiedades y métodos.

#### Capacidades y limitaciones adicionales del objeto Proxy

El objeto `Proxy` es una característica profundamente integrada en el propio lenguaje JavaScript, que permite a los desarrolladores interceptar y personalizar muchas operaciones que se pueden realizar en objetos. Esta característica abre escenarios nuevos e interesantes que antes no eran fácilmente alcanzables, como la metaprogramación, la sobrecarga de operadores y la virtualización de objetos.

Veamos otro ejemplo para aclarar este concepto:

```javascript
const evenNumbers = new Proxy([], {
  get: (target, index) => index * 2,
  has: (target, number) => number % 2 === 0
})

console.log(2 in evenNumbers) // true
console.log(5 in evenNumbers) // false
console.log(evenNumbers[7]) // 14
```

En este ejemplo, estamos creando un array virtual que contiene todos los números pares. Se puede usar como un array normal, lo que significa que podemos acceder a los elementos del array con la sintaxis de array habitual (por ejemplo, `evenNumbers[7]`) o verificar la existencia de un elemento en el array con el operador `in` (por ejemplo, `2 in evenNumbers`). El array se considera virtual porque nunca almacenamos datos en él.

Es muy importante tener en cuenta que, si bien el fragmento de código anterior es un ejemplo muy interesante que tiene como objetivo mostrar algunas de las capacidades avanzadas del objeto `Proxy`, no implementa el patrón Proxy. Este ejemplo nos permite ver que, aunque el objeto `Proxy` se utiliza comúnmente para implementar el patrón Proxy (de ahí su nombre), también se puede utilizar para implementar otros patrones y casos de uso. Como ejemplo, veremos más adelante en este capítulo cómo usar el objeto `Proxy` para implementar el patrón Decorator.

Al observar la implementación, este proxy usa un array vacío como destino (`target`) y luego define las trampas `get` y `has` en el `handler`:
- La trampa `get` intercepta el acceso a los elementos del array, devolviendo el número par para el índice dado.
- La trampa `has` intercepta el uso del operador `in` y comprueba si el número dado es par o no.

El objeto `Proxy` admite varias otras trampas interesantes, como `set`, `delete` y `construct`, y nos permite crear proxies que se pueden revocar a petición, deshabilitando todas las trampas y restaurando el comportamiento original del objeto de destino.

Analizar todas estas características va más allá del alcance de este capítulo; lo importante aquí es comprender que el objeto `Proxy` proporciona una base sólida para implementar el patrón de diseño Proxy.

Si tienes curiosidad por descubrir todas las capacidades y métodos trampa que ofrece el objeto `Proxy`, puedes leer más en el artículo correspondiente de MDN en [nodejsdp.link/mdn-proxy](https://nodejsdp.link/mdn-proxy). Otra buena fuente es este detallado artículo de Google en [nodejsdp.link/intro-proxy](https://nodejsdp.link/intro-proxy).

Si bien el objeto `Proxy` es una potente funcionalidad del lenguaje JavaScript, sufre una limitación muy importante: el objeto `Proxy` no se puede transpilar ni aplicar mediante un polyfill por completo. Esto se debe a que algunas de las trampas del objeto `Proxy` solo se pueden implementar a nivel de entorno de ejecución (*runtime*) y no se pueden reescribir simplemente en JavaScript plano. Esto es algo a tener en cuenta si estás trabajando con navegadores muy antiguos o versiones antiguas de Node.js que no admiten el objeto `Proxy` directamente.

> **Transpilación (*Transpilation*):** Abreviatura de transcompilación. Es la acción de compilar código fuente traduciéndolo de un lenguaje de programación de origen a otro. En el caso de JavaScript, esta técnica se utiliza para convertir un programa que utiliza nuevas capacidades del lenguaje en un programa equivalente que también puede ejecutarse en entornos de ejecución más antiguos que no admiten estas nuevas capacidades.
>
> **Polyfill:** Código que proporciona una implementación para una API estándar en JavaScript puro y que se puede importar en entornos donde esta API no está disponible (generalmente navegadores o entornos de ejecución más antiguos). `core-js` ([nodejsdp.link/corejs](https://nodejsdp.link/corejs)) es una de las bibliotecas de polyfill más completas para JavaScript.

#### Comparación de las diferentes técnicas de creación de proxies

La **composición** se puede considerar una forma sencilla y segura de crear un proxy porque deja al sujeto intacto sin mutar su comportamiento original. Su único inconveniente es que debemos delegar manualmente todos los métodos, incluso si solo queremos crear un proxy para uno de ellos. Además, es posible que tengamos que delegar el acceso a las propiedades del sujeto.

El **aumento de objetos** (*object augmentation*), por otro lado, modifica al sujeto, lo que puede no ser siempre ideal, pero no sufre de los diversos inconvenientes relacionados con la delegación. Por esta razón, entre estos dos enfoques, el aumento de objetos es generalmente la técnica preferida en todas aquellas circunstancias en las que modificar al sujeto sea una opción válida.

Sin embargo, hay al menos un escenario donde la composición se vuelve no solo útil sino necesaria: cuando necesitas controlar la inicialización del sujeto. Esto es especialmente importante en los casos en que el objeto es costoso de crear o puede que no sea necesario en absoluto, lo que comúnmente se conoce como **inicialización perezosa** (*lazy initialization*). Con la composición, puedes posponer la creación del sujeto hasta que realmente se llame a uno de sus métodos. Esta es una distinción clave con respecto al enfoque del objeto `Proxy`, que requiere una instancia existente para envolverla. Dado que `Proxy` opera sobre un objetivo completamente construido, no permite este tipo de configuración perezosa de fábrica. La composición te brinda un control total sobre cuándo y cómo cobra vida el objeto subyacente, lo que la convierte en la solución ideal en estas situaciones.

Finalmente, el objeto `Proxy` es el enfoque de referencia si necesitas interceptar llamadas a funciones o tener diferentes tipos de acceso a los atributos del objeto, incluso los dinámicos. El objeto `Proxy` proporciona un nivel avanzado de control de acceso que simplemente no está disponible con las otras técnicas. Por ejemplo, el objeto `Proxy` nos permite interceptar la eliminación de una clave en un objeto y realizar comprobaciones de existencia de propiedades.

Una vez más, vale la pena destacar que el objeto `Proxy` no muta al sujeto, por lo que se puede utilizar de forma segura en contextos donde el sujeto se comparte entre diferentes componentes de la aplicación. También vimos que con el objeto `Proxy` podemos realizar fácilmente la delegación de todos los métodos y atributos que queremos dejar sin cambios.

En la siguiente sección, presentamos un ejemplo más realista que aprovecha el patrón Proxy y lo usamos para comparar las diferentes técnicas que hemos discutido hasta ahora para implementar este patrón.

#### Creación de un stream Writable con registro (logging)

Para ver el patrón Proxy aplicado a un ejemplo real, ahora construiremos un objeto que actúa como proxy para un stream Writable, el cual intercepta todas las llamadas al método `write()` y registra un mensaje cada vez que esto sucede. Usaremos el objeto `Proxy` para implementar nuestro proxy. Escribamos nuestro código en un archivo llamado `logging-writable.js`:

```javascript
export function createLoggingWritable(writable) {
  return new Proxy(writable, { // 1
    get(target, propKey, _receiver) { // 2
      if (propKey === 'write') { // 3
        return (...args) => { // 4
          const [chunk] = args
          console.log('Writing', chunk)
          return writable.write(...args)
        }
      }
      return target[propKey] // 5
    },
  })
}
```

En el código anterior, creamos una factoría que devuelve una versión con proxy del objeto `writable` pasado como argumento. Veamos cuáles son los puntos principales de la implementación:
1. Creamos y devolvemos un proxy para el objeto `writable` original utilizando el constructor `Proxy` de ES2015.
2. Usamos la trampa `get` para interceptar el acceso a las propiedades del objeto.
3. Comprobamos si la propiedad a la que se accede es el método `write`. Si ese es el caso, devolvemos una función para aplicar un proxy al comportamiento original.
4. La lógica de implementación del proxy aquí es simple: extraemos el fragmento actual de la lista de argumentos pasados a la función original, registramos el contenido del fragmento y, finalmente, invocamos el método original con la lista de argumentos dada.
5. Devolvemos sin cambios cualquier otra propiedad.

Ahora podemos usar esta función recién creada y probar nuestra implementación de proxy:

```javascript
import { createWriteStream } from 'node:fs'
import { createLoggingWritable } from './logging-writable.js'

const writable = createWriteStream('test.txt')
const writableProxy = createLoggingWritable(writable)

writableProxy.write('First chunk')
writableProxy.write('Second chunk')
writable.write('This is not logged')
writableProxy.end()
```

El proxy no cambió la interfaz original del stream ni su comportamiento externo, pero si ejecutamos el código anterior, ahora veremos que cada fragmento que se escribe en el stream `writableProxy` se registra de forma transparente en la consola.

#### Observador de cambios (Change Observer) con Proxy

El patrón **Change Observer** es un patrón de diseño en el que un objeto (el sujeto) notifica a uno o más observadores de cualquier cambio de estado, para que puedan "reaccionar" a los cambios tan pronto como ocurran.

Aunque muy similar, el patrón Change Observer no debe confundirse con el patrón Observer discutido en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos. El patrón Change Observer se centra en permitir la detección de cambios de propiedades, mientras que el patrón Observer es un patrón más genérico que adopta un emisor de eventos para propagar información sobre los eventos que ocurren en el sistema.

Los proxies resultan ser una herramienta muy eficaz para crear objetos observables. Veamos una posible implementación con `create-observable.js`:

```javascript
export function createObservable(target, observer) {
  const observable = new Proxy(target, {
    set(obj, prop, value) {
      if (value !== obj[prop]) {
        const prev = obj[prop]
        obj[prop] = value
        observer({ prop, prev, curr: value })
      }
      return true
    },
  })
  return observable
}
```

En el código anterior, `createObservable()` acepta un objeto de destino (el objeto a observar en busca de cambios) y un observador (una función para invocar cada vez que se detecta un cambio).

Aquí, creamos la instancia observable a través de un proxy ES2015. El proxy implementa la trampa `set`, que se activa cada vez que se establece una propiedad. La implementación compara el valor actual con el nuevo y, si son diferentes, el objeto de destino se muta y se notifica al observador. Cuando se invoca al observador, pasamos un objeto literal que contiene información relacionada con el cambio (el nombre de la propiedad, el valor anterior y el valor actual).

Esta es una implementación simplificada del patrón Change Observer. Las implementaciones más avanzadas admiten múltiples observadores y utilizan más trampas para detectar otros tipos de mutación, como eliminaciones de campos o cambios de prototipo. Además, nuestra implementación no crea proxies recursivamente para objetos o arrays anidados; una implementación más avanzada también se encarga de estos casos.

Veamos ahora cómo podemos aprovechar los objetos observables con una sencilla aplicación de facturas donde el total de la factura se actualiza automáticamente en función de los cambios observados en los distintos campos de la factura:

```javascript
import { createObservable } from './create-observable.js'

function calculateTotal(invoice) { // 1
  return invoice.subtotal - invoice.discount + invoice.tax
}

const invoice = {
  subtotal: 100,
  discount: 10,
  tax: 20,
}
let total = calculateTotal(invoice)
console.log(`Starting total: ${total}`)

const obsInvoice = createObservable( // 2
  invoice,
  ({ prop, prev, curr }) => {
    total = calculateTotal(invoice)
    console.log(`TOTAL: ${total} (${prop} changed: ${prev} -> ${curr})`)
  }
)

// 3
obsInvoice.subtotal = 200 // TOTAL: 210
obsInvoice.discount = 20 // TOTAL: 200
obsInvoice.discount = 20 // no change: doesn't notify
obsInvoice.tax = 30 // TOTAL: 210
console.log(`Final total: ${total}`)
```

En el ejemplo anterior, una factura se compone de un valor de subtotal, un valor de descuento y un valor de impuestos. El importe total se puede calcular a partir de estos tres valores. Analicemos la implementación con mayor detalle:
1. Declaramos una función que calcula el total para una factura determinada, luego creamos un objeto de factura y un valor para mantener el total de la misma.
2. Aquí, creamos una versión observable del objeto de factura. Cada vez que hay un cambio en el objeto de factura original, recalculamos el total y también imprimimos algunos registros para realizar un seguimiento de los cambios.
3. Finalmente, aplicamos algunos cambios a la factura observable. Cada vez que mutamos el objeto `obsInvoice`, se activa la función observadora, se actualiza el total y se imprimen algunos registros en la pantalla.

Si ejecutamos este ejemplo, veremos la siguiente salida en la consola:

```text
Starting total: 110
TOTAL: 210 (subtotal changed: 100 -> 200)
TOTAL: 200 (discount changed: 10 -> 20)
TOTAL: 210 (tax changed: 20 -> 30)
Final total: 210
```

En este ejemplo, podríamos hacer que la lógica de cálculo total sea arbitrariamente complicada, por ejemplo, introduciendo nuevos campos en el cálculo (costos de envío, otros impuestos, etc.). En este caso, será bastante trivial introducir los nuevos campos en el objeto de factura y actualizar la función `calculateTotal()`. Una vez que hagamos eso, se observará cada cambio en las nuevas propiedades y el total se mantendrá actualizado con cada cambio.

Los observables son la piedra angular de la programación reactiva (*reactive programming* o RP) y la programación reactiva funcional (*functional reactive programming* o FRP). Si tienes curiosidad por saber más sobre estos estilos de programación, consulta el Manifiesto Reactivo en [nodejsdp.link/reactive-manifesto](https://nodejsdp.link/reactive-manifesto).

#### En la práctica

El patrón Proxy y, más específicamente, el patrón Change Observer son patrones ampliamente adoptados que se pueden encontrar tanto en proyectos y bibliotecas de backend como en el mundo frontend. Algunos proyectos populares que aprovechan estos patrones incluyen los siguientes:
- **LoopBack** ([nodejsdp.link/loopback](https://nodejsdp.link/loopback)) es un popular framework web de Node.js que utiliza el patrón Proxy para brindar la capacidad de interceptar y mejorar las llamadas a métodos en controladores. Esta capacidad se puede utilizar para construir mecanismos de validación o autenticación personalizados.
- **Vue.js** ([nodejsdp.link/vue](https://nodejsdp.link/vue)), un framework de interfaz de usuario reactivo de JavaScript muy popular, ha reimplementado propiedades observables utilizando el patrón Proxy con el objeto `Proxy`.
- **MobX** ([nodejsdp.link/mobx](https://nodejsdp.link/mobx)) es una famosa biblioteca reactiva de gestión de estado comúnmente utilizada en aplicaciones frontend en combinación con React o Vue.js. Al igual que Vue.js, MobX implementa observables reactivos utilizando el objeto `Proxy`.

---

### Sección 2: Decorator (Decorador)

**Decorator** es un patrón de diseño estructural que consiste en aumentar dinámicamente el comportamiento de un objeto existente. Es diferente de la herencia clásica porque el comportamiento no se añade a todos los objetos de la misma clase, sino solo a las instancias que se decoran explícitamente.

En cuanto a la implementación, es muy similar al patrón Proxy, pero en lugar de mejorar o modificar el comportamiento de la interfaz existente de un objeto, la aumenta con nuevas funcionalidades, como se muestra en la Figura 8.2:

**Figura 8.2:** Esquema del patrón Decorator.

En la Figura 8.2, el objeto `Decorator` está extendiendo el objeto `Component` agregando la operación `methodC()`. Los métodos existentes generalmente se delegan al objeto decorado sin procesamiento adicional, pero en algunos casos, también pueden interceptarse y aumentarse con comportamientos adicionales.

#### Técnicas para implementar decoradores

Aunque Proxy y Decorator son conceptualmente dos patrones diferentes con intenciones distintas, prácticamente comparten las mismas estrategias de implementación. Las revisaremos en breve. Esta vez, queremos usar el patrón Decorator para poder tomar una instancia de nuestra clase `StackCalculator` y "decorarla" para que también exponga un nuevo método llamado `add()`, que podamos usar para realizar sumas entre dos números. También usaremos el decorador para interceptar todas las llamadas al método `divide()` e implementar la misma comprobación de división por cero que ya vimos en nuestro ejemplo `SafeCalculator`.

#### Composición

Utilizando la composición, el componente decorado se envuelve alrededor de un nuevo objeto que normalmente hereda de él. El decorador en este caso simplemente necesita definir los nuevos métodos, mientras delega los existentes al componente original:

```javascript
class EnhancedCalculator {
  constructor(calculator) {
    this.calculator = calculator
  }

  // new method
  add() {
    const addend2 = this.getValue()
    const addend1 = this.getValue()
    const result = addend1 + addend2
    this.putValue(result)
    return result
  }

  // modified method
  divide() {
    // additional validation logic
    const divisor = this.calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    // if valid delegates to the subject
    return this.calculator.divide()
  }

  // delegated methods
  putValue(value) {
    return this.calculator.putValue(value)
  }

  getValue() {
    return this.calculator.getValue()
  }

  peekValue() {
    return this.calculator.peekValue()
  }

  clear() {
    return this.calculator.clear()
  }

  multiply() {
    return this.calculator.multiply()
  }
}

const calculator = new StackCalculator()
const enhancedCalculator = new EnhancedCalculator(calculator)

enhancedCalculator.putValue(4)
enhancedCalculator.putValue(3)
console.log(enhancedCalculator.add()) // 4+3 = 7
enhancedCalculator.putValue(2)
console.log(enhancedCalculator.multiply()) // 7*2 = 14
```

Si recuerdas nuestra implementación de composición para el patrón Proxy, probablemente puedas ver que el código aquí se ve bastante similar.

Creamos el nuevo método `add()` y mejoramos el comportamiento del método `divide()` original (replicando efectivamente la característica que vimos en el ejemplo anterior de `SafeCalculator`). Finalmente, delegamos los métodos `putValue()`, `getValue()`, `peekValue()`, `clear()` y `multiply()` al sujeto original.

#### Decoración de objetos (Object decoration)

La decoración de objetos también se puede lograr simplemente adjuntando nuevos métodos directamente al objeto decorado (*monkey patching*), de la siguiente manera:

```javascript
function patchCalculator(calculator) {
  // new method
  calculator.add = () => {
    const addend2 = calculator.getValue()
    const addend1 = calculator.getValue()
    const result = addend1 + addend2
    calculator.putValue(result)
    return result
  }

  // modified method
  const divideOrig = calculator.divide
  calculator.divide = () => {
    // additional validation logic
    const divisor = calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    // if valid delegates to the subject
    return divideOrig.apply(calculator)
  }

  return calculator
}

const calculator = new StackCalculator()
const enhancedCalculator = patchCalculator(calculator)
// ...
```

Ten en cuenta que en este ejemplo, `calculator` y `enhancedCalculator` hacen referencia al mismo objeto (`calculator == enhancedCalculator`). Esto se debe a que `patchCalculator()` está mutando el objeto `calculator` original y luego devolviéndolo. Puedes confirmar esto invocando `calculator.add()` o `calculator.divide()`.

#### Decoración con el objeto Proxy

Es posible implementar la decoración de objetos mediante el uso del objeto `Proxy`. Un ejemplo genérico podría verse así:

```javascript
const enhancedCalculatorHandler = {
  get(target, property) {
    if (property === 'add') {
      // new method
      return function add() {
        const addend2 = target.getValue()
        const addend1 = target.getValue()
        const result = addend1 + addend2
        target.putValue(result)
        return result
      }
    }
    if (property === 'divide') {
      // modified method
      return () => {
        // additional validation logic
        const divisor = target.peekValue()
        if (divisor === 0) {
          throw new Error('Division by 0')
        }
        // if valid delegates to the subject
        return target.divide()
      }
    }
    // delegated methods and properties
    return target[property]
  },
}

const calculator = new StackCalculator()
const enhancedCalculator = new Proxy(
  calculator,
  enhancedCalculatorHandler
)
// ...
```

Si tuviéramos que comparar estas diferentes implementaciones, las mismas advertencias discutidas durante el análisis del patrón Proxy también se aplicarían al decorador. ¡Enfoquémonos en su lugar en practicar el patrón con un ejemplo de la vida real!

#### Funciones regulares frente a funciones flecha en proxies

Si has estado prestando mucha atención, habrás notado que nuestro manejador de proxy utiliza una función regular para el método `add()` y una función flecha para `divide()`. Esto no tiene ninguna implicación práctica en nuestro ejemplo, pero es algo que se deja allí para hacerte pensar. En JavaScript, el valor de `this` se comporta de manera diferente según el tipo de función:
- Las **funciones regulares** establecen `this` en función de cómo se llaman. En el contexto de una trampa de proxy, devolver una función regular significa que `this` dentro de esa función se vinculará al objetivo del proxy.
- Las **funciones flecha**, por otro lado, no tienen su propio `this`. En su lugar, heredan `this` del ámbito léxico circundante. En este caso, la propia trampa de proxy.

En la práctica, esto rara vez importa cuando se trabaja con proxies, porque generalmente se tiene acceso directo al destino desde los parámetros de la trampa (como hicimos en nuestro ejemplo). Pero sigue siendo una distinción sutil que vale la pena recordar, especialmente si alguna vez necesitas confiar en `this` dentro del cuerpo de la función.

En resumen, las funciones flecha no son un reemplazo directo de las funciones regulares cuando `this` importa. ¡Tenlo en cuenta para evitar errores sutiles y frustrantes!

#### Decoración de una base de datos Level

Antes de comenzar a codificar el siguiente ejemplo, digamos algunas palabras sobre Level, el módulo con el que ahora vamos a trabajar.

##### Presentación de Level y LevelDB

Level es un ecosistema potente y flexible de módulos de JavaScript para crear bases de datos clave-valor. Diseñado pensando en la transparencia y la modularidad, proporciona una base sólida de primitivas de bajo nivel que te permiten crear bases de datos personalizadas, adaptadas a tus necesidades específicas. Estas bases de datos pueden ser embebidas o en red, persistentes o en memoria.

En su núcleo, Level se basa en los principios de LevelDB ([nodejsdp.link/leveldb](https://nodejsdp.link/leveldb)), un almacén de clave-valor de alto rendimiento desarrollado en Google. Al igual que LevelDB, las bases de datos Level admiten claves y valores binarios, escrituras por lotes eficientes e iteración bidireccional rápida sobre entradas ordenadas. Este orden lexicográfico, combinado con consultas de rango y capturas instantáneas (*snapshots*), permite operaciones de lectura potentes y coherentes. Lo que hace que Level sea especialmente adecuado para Node.js es su integración con interfaces familiares: streams, eventos, buffers e iteradores asíncronos. También admite funciones avanzadas como codificaciones personalizadas y subniveles (*sublevels*), que te permiten organizar tus datos en secciones modulares impulsadas por eventos dentro de una sola base de datos. Level se puede utilizar tanto en Node.js en el servidor como en el navegador en el cliente, lo que lo convierte en una excelente opción para aplicaciones JavaScript full-stack.

Hoy en día, existe un vasto ecosistema alrededor de Level compuesto por plugins y módulos que amplían el núcleo diminuto para implementar características como replicación, índices secundarios, actualizaciones en vivo, motores de consulta y más. También se crearon bases de datos completas sobre Level, incluidos clones de CouchDB como PouchDB ([nodejsdp.link/pouchdb](https://nodejsdp.link/pouchdb)), e incluso una base de datos de grafos, LevelGraph ([nodejsdp.link/levelgraph](https://nodejsdp.link/levelgraph)), que puede funcionar tanto en Node.js como en el navegador.

Obtén más información sobre el ecosistema de Level en [nodejsdp.link/awesome-level](https://nodejsdp.link/awesome-level).

##### Implementación de un plugin de Level

En el siguiente ejemplo, te mostraremos cómo podemos crear un plugin simple para Level utilizando el patrón Decorator y, en particular, la técnica de aumento de objetos, que es la forma más simple pero también la más pragmática y efectiva de decorar objetos con capacidades adicionales.

Lo que queremos construir es un plugin para Level que nos permita recibir notificaciones cada vez que se guarda un objeto con un determinado patrón en la base de datos. Por ejemplo, si nos suscribimos a un patrón como `{a: 1}`, queremos recibir una notificación cuando se guarden en la base de datos objetos como `{a: 1, b: 3}` o `{a: 1, c: 'x'}`.

Comencemos a construir nuestro pequeño plugin creando un nuevo módulo llamado `level-subscribe.js`. Luego insertaremos el siguiente código:

```javascript
export function levelSubscribe(db) {
  db.subscribe = (pattern, listener) => { // 1
    db.on('write', docs => { // 2
      for (const doc of docs) {
        const match = Object.keys(pattern).every(
          k => pattern[k] === doc.value[k] // 3
        )
        if (match) {
          listener(doc.key, doc.value) // 4
        }
      }
    })
  }
  return db
}
```

Eso es todo para nuestro plugin; es extremadamente simple. Analicemos brevemente el código anterior:
1. Decoramos el objeto `db` con un nuevo método llamado `subscribe()`. Simplemente adjuntamos el método directamente a la instancia `db` proporcionada (aumento de objetos).
2. Escuchamos cualquier operación de escritura realizada en la base de datos.
3. Realizamos un algoritmo de coincidencia de patrones muy simple, que verifica que todas las propiedades del patrón proporcionado también estén disponibles en los datos que se insertan.
4. Si tenemos una coincidencia, notificamos al escuchador (*listener*).

Escribamos ahora algo de código para probar nuestro nuevo plugin:

```javascript
import { join } from 'node:path'
import { Level } from 'level' // v9.0.0
import { levelSubscribe } from './level-subscribe.js'

const dbPath = join(import.meta.dirname, 'db')
const db = new Level(dbPath, { valueEncoding: 'json' }) // 1

levelSubscribe(db) // 2

db.subscribe( // 3
  { doctype: 'message', language: 'en' },
  (_k, val) => console.log(val)
)

await db.put('1', { // 4
  doctype: 'message',
  text: 'Hi',
  language: 'en',
})

await db.put('2', {
  doctype: 'company',
  name: 'ACME Co.',
})
```

Así es como funciona el código anterior:
1. Primero, inicializamos nuestra base de datos Level, eligiendo el directorio donde se almacenan los archivos y la codificación predeterminada para los valores.
2. Luego, adjuntamos nuestro plugin, que decora el objeto `db` original.
3. En este punto, estamos listos para usar la nueva funcionalidad proporcionada por nuestro plugin, que es el método `subscribe()`, donde especificamos que estamos interesados en todos los objetos con `doctype: 'message'` y `language: 'en'`.
4. Finalmente, guardamos algunos valores en la base de datos usando `put`. La primera llamada activa el callback asociado con nuestra suscripción de escritura, y deberíamos ver el objeto almacenado impreso en la consola. Esto se debe a que, en este caso, el objeto coincide con la suscripción. La segunda llamada no genera ninguna salida porque el objeto almacenado no coincide con los criterios de suscripción; de hecho, al segundo registro le falta el campo `language`.

Este ejemplo muestra una aplicación real del patrón Decorator en su implementación más simple, que es el aumento de objetos. Puede parecer un patrón trivial, pero tiene un poder innegable si se usa apropiadamente.

#### En la práctica

Para ver más ejemplos de cómo se utilizan los decoradores en el mundo real, puedes inspeccionar el código de algunos plugins más de Level:
- `levelgraph` ([nodejsdp.link/levelgraph](https://nodejsdp.link/levelgraph)): un plugin que añade capacidades de base de datos de grafos a Level.
- `search-index` ([nodejsdp.link/search-index](https://nodejsdp.link/search-index)): un plugin que añade capacidades de búsqueda de texto completo a Level.
- `level-ttl` ([nodejsdp.link/level-ttl](https://nodejsdp.link/level-ttl)): añade un TTL (*Time To Live*) para expirar claves automáticamente en una instancia de Level.

Aparte de los plugins de Level, los siguientes proyectos también son buenos ejemplos de la adopción del patrón Decorator:
- `json-socket` ([nodejsdp.link/json-socket](https://nodejsdp.link/json-socket)): Este módulo facilita el envío de datos JSON a través de un socket TCP (o Unix). Está diseñado para decorar una instancia existente de `net.Socket`, que se enriquece con métodos y comportamientos adicionales.
- `fastify` ([nodejsdp.link/fastify](https://nodejsdp.link/fastify)) es un framework de aplicaciones web que expone una API para decorar una instancia de servidor Fastify con funcionalidad o configuración adicional. Con este enfoque, la funcionalidad adicional se hace accesible a diferentes partes de la aplicación. Esta es una implementación bastante generalizada del patrón Decorator. Consulta la página de documentación dedicada para obtener más información en [nodejsdp.link/fastify-decorators](https://nodejsdp.link/fastify-decorators).

#### La propuesta de decoradores para ECMAScript

Es importante distinguir el patrón de diseño Decorator, como se describe aquí, de la propuesta de Decoradores para ECMAScript ([nodejsdp.link/proposal-decorators](https://nodejsdp.link/proposal-decorators)), al momento de escribir este artículo, en la Etapa 3 (*Stage 3*) de estandarización. Si bien ambos conceptos comparten el nombre "Decorator", operan en diferentes contextos y tienen diferentes propósitos.

El patrón de diseño Decorator es un patrón estructural para añadir responsabilidades dinámicamente a objetos individuales en tiempo de ejecución. Por el contrario, la propuesta de decoradores de ECMAScript tiene como objetivo proporcionar una nueva sintaxis declarativa para modificar o extender clases y sus miembros, y, para darte un ejemplo concreto, se ve así:

```javascript
@defineElement("my-class")
class C extends HTMLElement {
  @reactive accessor clicked = false;
}
```

`@defineElement` y `@reactive` son ejemplos de la nueva sintaxis propuesta que te permite decorar una clase o un elemento de clase. Una sintaxis similar existe en otros lenguajes (por ejemplo, Java y Python).

La propuesta de decoradores de ECMAScript ya se adopta en entornos JavaScript transpilados y ha ganado un interés significativo en la estandarización. Frameworks como Angular ([nodejsdp.link/angular](https://nodejsdp.link/angular)) y NestJS ([nodejsdp.link/nestjs](https://nodejsdp.link/nestjs)) dependen en gran medida de esta sintaxis de decoradores, que requiere transpilación para su uso en Node.js y el navegador.

Incluso TypeScript ha añadido soporte para la sintaxis de decoradores propuesta en la propuesta de decoradores de ECMAScript. Al utilizar el compilador de TypeScript, el código de polyfill necesario se genera automáticamente para ti, lo que garantiza la compatibilidad con entornos de JavaScript que no admiten la sintaxis de forma nativa.

La distinción central entre la sintaxis de Decorator y el patrón Decorator radica en su propósito y aplicación. El patrón Decorator es una técnica de diseño general utilizada para mejorar instancias de objetos individuales con funcionalidad adicional o para modificar su comportamiento. Por el contrario, la sintaxis de Decorator propuesta por el comité TC39 se centra en extender la definición de clases de JavaScript a través de modificadores a nivel de clase.

Aunque el nombre pueda sugerir una conexión, los dos conceptos abordan diferentes casos de uso. La última propuesta del TC39 se basa en años de iteración y experimentos previos, pero sigue centrada en la extensión de clases en lugar del aumento de instancias, como se ve en el patrón de diseño.

El comité TC39 ha estado trabajando en la especificación de decoradores a nivel de clase durante varios años. Sin embargo, actualmente no está claro cuándo o si esta propuesta alcanzará la Etapa 4 (*Stage 4*), la etapa final y estable, lo que indica que está lista para su implementación y amplia adopción. Aunque no es raro ver que esta sintaxis ya se adopte en la práctica, recomendamos precaución al confiar en esta característica en el código de producción hasta que esté finalizada, ya que cualquier cambio en su diseño final podría resultar en cambios importantes que rompan el código (*breaking changes*) y afecten negativamente a las bases de código existentes. Además, debido a que los decoradores en TypeScript se implementan como transformaciones del compilador que producen código en tiempo de ejecución, la eliminación de tipos integrada de Node.js (*type stripping*) no es suficiente. Todavía necesitas un paso de transpilación, lo cual es otra razón para evitar este enfoque por ahora.

---

### Sección 3: La delgada línea entre Proxy y Decorator

En este punto del libro, es posible que tengas algunas dudas legítimas sobre las diferencias entre los patrones Proxy y Decorator. Estos dos patrones son de hecho muy similares y, a veces, se pueden utilizar indistintamente.

En su encarnación clásica, el patrón Decorator se define como un mecanismo que nos permite mejorar un objeto existente con un nuevo comportamiento, mientras que el patrón Proxy se utiliza para controlar el acceso a un objeto concreto o virtual.

Existe una diferencia conceptual entre los dos patrones, y se basa principalmente en la forma en que se utilizan en tiempo de ejecución.

Puedes ver el patrón Decorator como un envoltorio (*wrapper*); puedes tomar diferentes tipos de objetos y decidir envolverlos con un decorador para mejorar sus capacidades con funcionalidad adicional. Un proxy, en cambio, se utiliza para controlar el acceso a un objeto y no cambia la interfaz original. Por esta razón, una vez que has creado una instancia de proxy, puedes pasarla a un contexto que espera el objeto original.

Cuando se trata de la implementación, estas diferencias son generalmente mucho más obvias con lenguajes fuertemente tipados donde el tipo de los objetos que pasas se comprueba en tiempo de compilación. En el ecosistema de Node.js, dada la naturaleza dinámica del lenguaje JavaScript, la línea entre los patrones Proxy y Decorator es bastante borrosa y, a menudo, los dos nombres se usan indistintamente. También hemos visto cómo se pueden utilizar las mismas técnicas para implementar ambos patrones.

Al tratar con JavaScript y Node.js, nuestro consejo es evitar atascarse con la nomenclatura y la definición canónica de estos dos patrones. Te recomendamos que consideres la clase de problemas que resuelven proxy y decorator como un todo y trates estos dos patrones como herramientas complementarias y, a veces, intercambiables.

---

### Sección 4: Adapter (Adaptador)

El patrón **Adapter** nos permite acceder a la funcionalidad de un objeto utilizando una interfaz diferente.

Un ejemplo de la vida real de un adaptador sería un dispositivo que te permite conectar un cable USB Tipo A a un puerto USB Tipo C. En un sentido genérico, un adaptador convierte un objeto con una interfaz determinada para que pueda usarse en un contexto donde se espera una interfaz diferente.

En software, el patrón Adapter se utiliza para tomar la interfaz de un objeto (el adaptado o *adaptee*) y hacerla compatible con otra interfaz que espera un cliente determinado. Echemos un vistazo a la Figura 8.3 para aclarar esta idea:

**Figura 8.3:** Esquema del patrón Adapter.

En la Figura 8.3, podemos ver cómo el adaptador es esencialmente un envoltorio para el adaptado (*adaptee*), exponiendo una interfaz diferente. El diagrama también resalta el hecho de que las operaciones del adaptador también pueden ser una composición de una o más invocaciones de métodos en el adaptado. Desde una perspectiva de implementación, la técnica más común es la composición, donde los métodos del adaptador proporcionan un puente hacia los métodos del adaptado. Este patrón es sencillo, así que trabajemos inmediatamente en un ejemplo.

#### Uso de Level a través de la API del sistema de archivos

Ahora vamos a construir un adaptador alrededor de la API de Level, transformándola en una interfaz que sea vagamente compatible con algunas funciones proporcionadas por el módulo central `node:fs/promises`. En particular, nos aseguraremos de que cada llamada a `readFile()` y `writeFile()` se traduzca en llamadas a `db.get()` y `db.put()`. De esta manera, podremos usar una base de datos Level como backend de almacenamiento para operaciones simples del sistema de archivos.

Si bien la implementación de un adaptador Level completo para `node:fs/promises` es definitivamente posible, hacerlo nos llevaría más allá del enfoque de esta sección. Una vez que hayas completado esta sección, no dudes en asumir ese desafío como un ejercicio para ayudarte a consolidar tu comprensión de los conceptos que hemos cubierto.

Comencemos creando un nuevo módulo llamado `fs-adapter.js`. Comenzaremos cargando las dependencias y exportando la factoría `createFSAdapter()` que vamos a utilizar para construir el adaptador:

```javascript
import { resolve } from 'node:path'

export function createFSAdapter (db) {
  return ({
    async readFile (filename, options = undefined) {
      // ...
    },
    async writeFile (filename, contents, options = undefined) {
      // ...
    }
  })
}
```

A continuación, implementaremos la función `readFile()` dentro de la factoría y nos aseguraremos de que su interfaz sea compatible con la función original del módulo `fs`:

```javascript
async readFile(filename, options = undefined) {
  const valueEncoding = typeof options === 'string' ? options : options?.encoding
  const opt = valueEncoding ? { valueEncoding } : undefined // 1
  const value = await db.get(resolve(filename), opt) // 2
  if (typeof value === 'undefined') { // 3
    const e = new Error(
      `ENOENT: no such file or directory, open '${filename}'`
    )
    e.code = 'ENOENT'
    e.errno = 34
    e.path = filename
    throw e
  }
  return value // 4
}
```

En el código anterior, tuvimos que hacer un trabajo adicional para asegurarnos de que el comportamiento de nuestra nueva función sea lo más cercano posible a la función `readFile()` original del módulo `node:fs/promises`. Los pasos realizados por la función se describen a continuación:
1. La única opción que nos interesa es `encoding`. Esto se puede pasar como una cadena simple (por ejemplo, `options = 'utf8'`) o como un objeto con una clave llamada `encoding` (por ejemplo, `options = { encoding: 'utf8' }`). Si no se especifica ningún valor, se utiliza la opción predeterminada (`undefined`), que se asigna a una codificación binaria sin procesar. La codificación se utiliza para decodificar automáticamente el valor leído de la base de datos. Si se proporciona `'utf8'` (u otra codificación de texto), el valor devuelto será una cadena; de lo contrario, será un `Buffer`. Esto es consistente con cómo funciona la función `readFile()` original.
2. Para recuperar un archivo de la instancia `db`, invocamos `db.get()`, usando `filename` como clave, asegurándonos de usar siempre su ruta completa (usando `resolve()`). Establecemos el valor de la opción `valueEncoding` utilizada por la base de datos para que sea igual a cualquier opción de codificación eventual recibida como entrada.
3. Si la clave no se encuentra en la base de datos, creamos y lanzamos un error con `ENOENT` como código de error, que es el código utilizado por el módulo `fs` original para indicar un archivo que falta.
4. Si el par clave-valor se recupera con éxito de la base de datos, devolveremos el valor a quien llama.

La función que creamos no pretende ser un reemplazo perfecto para la función `readFile()`, pero definitivamente cumple su función en las situaciones más comunes.

Para completar nuestro pequeño adaptador, veamos ahora cómo implementar la función `writeFile()`:

```javascript
async writeFile(filename, contents, options = undefined) {
  const valueEncoding = typeof options === 'string' ? options : options?.encoding
  const opt = valueEncoding ? { valueEncoding } : undefined
  await db.put(resolve(filename), contents, opt)
}
```

Como podemos ver, en este caso tampoco tenemos un envoltorio perfecto. Estamos ignorando algunas opciones, como los permisos de archivo (`options.mode`), y estamos reenviando cualquier error que recibamos de la base de datos tal como está.

Nuestro nuevo adaptador ya está listo, pero antes de usarlo, escribamos un pequeño módulo de prueba que solo use la API original de `node:fs/promises`:

```javascript
import fs from 'node:fs/promises'

await fs.writeFile('file.txt', 'Hello!', 'utf8')
const res = await fs.readFile('file.txt', 'utf8')
console.log(res)

// try to read a missing file (throws an error)
await fs.readFile('missing.txt')
```

El código anterior utiliza la API original de `fs` para realizar algunas operaciones de lectura y escritura en el sistema de archivos, y debería imprimir algo como lo siguiente en la consola:

```text
Hello!
Error: ENOENT: no such file or directory, open 'missing.txt'
```

La primera línea muestra el contenido del archivo que creamos con `writeFile()` y luego leímos con `readFile()`, mientras que la segunda línea muestra el error que se lanzó al intentar acceder al archivo inexistente `missing.txt`.

Ahora, podemos intentar reemplazar el módulo `fs` con nuestro adaptador, de la siguiente manera:

```javascript
import { join } from 'node:path'
import { Level } from 'level' // v9.0.0
import { createFsAdapter } from './fs-adapter.js'

const db = new Level(join(import.meta.dirname, 'db'), {
  valueEncoding: 'binary',
})
const fs = createFsAdapter(db)
// ...
```

Ejecutar nuestro programa nuevamente debería producir la misma salida, excepto por el hecho de que ninguna parte del archivo que especificamos se lee o escribe utilizando la API del sistema de archivos directamente. En su lugar, cualquier operación realizada con nuestro adaptador se convertirá en una operación realizada en una base de datos Level.

El adaptador que acabamos de crear puede parecer redundante; ¿cuál es el propósito de usar una base de datos en lugar del sistema de archivos real? Sin embargo, debemos recordar que Level en sí tiene adaptadores que permiten que la base de datos también se ejecute en el navegador. Uno de estos adaptadores es `browser-level` ([nodejsdp.link/browser-level](https://nodejsdp.link/browser-level)). Ahora nuestro adaptador tiene mucho sentido. Podríamos usar algo similar para permitir que el código que aprovecha el módulo `fs` se ejecute tanto en Node.js como en un navegador.

#### En la práctica

Hay muchos ejemplos del mundo real del patrón Adapter. Hemos enumerado algunos de los ejemplos más notables aquí para que los explores y analices:
- Ya sabemos que Level puede ejecutarse con diferentes backends de almacenamiento, desde el LevelDB predeterminado hasta IndexedDB en el navegador. Esto es posible gracias a los diversos adaptadores que se crean para replicar la API interna (privada) de Level. Echa un vistazo a algunos de ellos para ver cómo se implementan en [nodejsdp.link/level-stores](https://nodejsdp.link/level-stores).
- **Prisma ORM** ([nodejsdp.link/prisma](https://nodejsdp.link/prisma)), un famoso ORM de Node.js, admite varias bases de datos diferentes (PostgreSQL, MySQL, MongoDB, etc.) utilizando el patrón Adapter para implementar controladores específicos de bases de datos.
- El complemento perfecto para el ejemplo que creamos es `level-filesystem` ([nodejsdp.link/level-filesystem](https://nodejsdp.link/level-filesystem)), que es una implementación más completa de la API `fs` sobre Level.

---

### Sección 5: Resumen

Los patrones de diseño estructurales son definitivamente algunos de los patrones de diseño más ampliamente adoptados en la ingeniería de software y es importante tener confianza con ellos. En este capítulo, exploramos los patrones Proxy, Decorator y Adapter, y discutimos diferentes formas de implementarlos en el contexto de Node.js.

Vimos cómo el patrón Proxy puede ser una herramienta muy valiosa para controlar el acceso a objetos existentes. En este capítulo, también mencionamos cómo el patrón Proxy puede habilitar diferentes paradigmas de programación, como la programación reactiva mediante el patrón Change Observer.

En la segunda parte del capítulo, descubrimos que el patrón Decorator es una herramienta invaluable para poder añadir funcionalidad adicional a objetos existentes. Vimos que su implementación no difiere mucho del patrón Proxy y exploramos algunos ejemplos construidos alrededor del ecosistema LevelDB.

Finalmente, discutimos el patrón Adapter, que nos permite envolver un objeto existente y exponer su funcionalidad a través de una interfaz diferente. Vimos que este patrón puede ser útil para exponer una pieza de funcionalidad existente a un componente que espera una interfaz diferente. En nuestros ejemplos, vimos cómo este patrón se puede utilizar para implementar una capa de almacenamiento alternativa que sea compatible con la interfaz proporcionada por el módulo `fs` para interactuar con archivos.

Proxy, Decorator y Adapter son muy similares; la diferencia entre ellos se puede apreciar desde la perspectiva del consumidor de la interfaz: **Proxy** proporciona la misma interfaz que el objeto envuelto, **Decorator** proporciona una interfaz mejorada y **Adapter** proporciona una interfaz diferente.

En el próximo capítulo, completaremos nuestro viaje a través de los patrones de diseño tradicionales en Node.js explorando la categoría de los patrones de diseño de comportamiento (*behavioral design patterns*). Esta categoría incluye patrones importantes como el patrón Strategy, el patrón Middleware y el patrón Iterator. ¿Estás listo para descubrir los patrones de diseño de comportamiento?

---

### Sección 6: Ejercicios

- **8.1 Caché de cliente HTTP:** Escribe un proxy para tu biblioteca de cliente HTTP favorita que almacene en caché la respuesta de una solicitud HTTP determinada, de modo que si realizas la misma solicitud nuevamente, la respuesta se devuelva inmediatamente desde la caché local, en lugar de recuperarse de la URL remota. Si necesitas inspiración, puedes consultar el módulo `superagent-cache` ([nodejsdp.link/superagent-cache](https://nodejsdp.link/superagent-cache)).
- **8.2 Registros con marca de tiempo:** Crea un proxy para el objeto `console` que mejore cada función de registro (`log()`, `error()`, `debug()` e `info()`) anteponiendo la marca de tiempo actual al mensaje que deseas imprimir en los registros. Por ejemplo, ejecutar `consoleProxy.log('hello')` debería imprimir algo como `2020-02-18T15:59:30.699Z hello` en la consola.
- **8.3 Salida de consola coloreada:** Escribe un decorador para la consola que agregue los métodos `red(message)`, `yellow(message)` y `green(message)`. Estos métodos tendrán que comportarse como `console.log(message)` excepto que imprimirán el mensaje en rojo, amarillo o verde, respectivamente. En uno de los ejercicios del capítulo anterior, ya te indicamos algunos paquetes útiles para crear salidas de consola coloreadas. Si quieres probar algo diferente esta vez, echa un vistazo a `ansi-styles` ([nodejsdp.link/ansi-styles](https://nodejsdp.link/ansi-styles)).
- **8.4 Sistema de archivos virtual:** Modifica nuestro ejemplo de adaptador de sistema de archivos LevelDB para escribir los datos del archivo en la memoria en lugar de en LevelDB. Puedes usar un objeto o una instancia de `Map` para almacenar los pares clave-valor de nombres de archivo y los datos asociados.
- **8.5 El buffer perezoso (lazy buffer):** ¿Puedes implementar `createLazyBuffer(size)`, una función factoría que genera un proxy virtual para un `Buffer` del tamaño dado? La instancia del proxy debe instanciar un objeto `Buffer` (asignando efectivamente la cantidad de memoria dada) solo cuando se invoque `write()` por primera vez. Si no se intenta escribir en el buffer, no se debe crear ninguna instancia de `Buffer`.
