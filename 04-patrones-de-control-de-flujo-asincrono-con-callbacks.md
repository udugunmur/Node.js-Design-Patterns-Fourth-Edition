# Parte 1: Fundamentos de Node.js

## Capítulo 4: Patrones de control de flujo asíncrono con Callbacks

La transición de un estilo de programación síncrono a una plataforma como Node.js, donde el estilo de paso de continuación (*continuation-passing style* o CPS) y las APIs asíncronas son el estándar, puede resultar todo un desafío. El código asíncrono dificulta predecir el orden en el que se ejecutarán las instrucciones. Tareas sencillas como iterar a través de un conjunto de archivos, ejecutar tareas de forma secuencial o esperar a que se completen múltiples operaciones requieren que los desarrolladores adopten nuevos enfoques y técnicas para evitar escribir código ineficiente y difícil de leer.

Al utilizar callbacks para gestionar el flujo de control asíncrono, el error más común es caer en la trampa del "infierno de los callbacks" (*callback hell*), donde el código crece horizontalmente con un anidamiento excesivo, lo que hace que incluso las rutinas más simples sean difíciles de leer y mantener. Sin embargo, aplicando disciplina y utilizando ciertos patrones, es posible dominar los callbacks y escribir código asíncrono limpio y manejable.

En este capítulo profundizaremos en los callbacks: qué tienen de bueno, qué tienen de malo y cómo manejarlos de manera eficaz. Específicamente, cubriremos:
- Los desafíos de la programación asíncrona.
- Buenas prácticas para evitar el callback hell y escribir código asíncrono eficiente.
- Patrones asíncronos comunes: ejecución secuencial (ejecutar tareas una tras otra), iteración secuencial (procesar elementos en secuencia sin paralelizar tareas), ejecución concurrente (ejecutar múltiples tareas concurrentemente) y ejecución concurrente limitada (controlar el número de operaciones concurrentes).

Al dominar estos conceptos, estarás bien encaminado para escribir código asíncrono eficiente y legible.

---

### Sección 1: Los desafíos de la programación asíncrona

La programación asíncrona en JavaScript puede parecer sencilla, pero no está exenta de dificultades. El uso de clausuras (*closures*) y la definición en línea (*in-place*) de funciones anónimas permite una experiencia de codificación fluida que no requiere que el desarrollador salte a otros puntos del código base, un enfoque alineado con el principio KISS (*Keep It Simple, Stupid* o, como preferimos llamarlo, "Mantenlo Súper Simple").

El principio KISS es un recordatorio amigable para priorizar la simplicidad en el diseño de software. Acuñado por el ingeniero Kelly Johnson en Lockheed, la idea era que las cosas deberían ser fáciles de reparar, incluso con herramientas limitadas. Para nosotros, los desarrolladores, significa escribir código claro y comprensible en lugar de soluciones complejas. Apunta a la legibilidad, usa nombres claros y apégate a patrones conocidos. Esto no solo reduce los errores, sino que también facilita el trabajo en equipo y hace que el código sea más fácil de mantener.

Sin embargo, esta simplicidad a menudo se logra a expensas de la modularidad, la reutilización y la mantenibilidad. El riesgo es terminar con código compuesto por múltiples callbacks anidados, funciones extensas y, en general, una mala organización del código.

Si bien no existe una única solución "perfecta" para afrontar los desafíos del código asíncrono, te dotaremos de las habilidades necesarias para evaluar las ventajas y desventajas (*trade-offs*) y elegir enfoques que equilibren la simplicidad con la mantenibilidad. Aprenderemos a reconocer problemas potenciales antes de que surjan, anticipar dificultades e implementar soluciones que promuevan la modularidad, la reutilización y la mantenibilidad. Al dominar estos conceptos, podrás escribir código asíncrono eficiente, legible y bien estructurado que satisfaga las demandas del desarrollo moderno en JavaScript, no eliminando todas las concesiones, sino comprendiendo cómo tomar decisiones informadas sobre cuándo priorizar la simplicidad sobre la complejidad.

#### Creación de un web spider simple

Para trabajar con un ejemplo práctico y realista que pueda ilustrar algunos de los desafíos de la programación asíncrona, crearemos un pequeño *web spider* (rastreador web): una aplicación de línea de comandos que toma una URL web como entrada y descarga su contenido localmente en uno o más archivos. Este tipo de software es algo que puede resultar útil si deseas poder navegar por un sitio web sin conexión o tomar una instantánea del mismo en un momento determinado.

¡En una ocasión tuve que construir algo muy similar para el trabajo! Necesitaba un programa que pudiera recorrer un sitio web y encontrar los enlaces rotos (errores 404). Utilizar la programación asíncrona fue la clave para manejar la escala de los sitios en los que se utilizó, compuestos a menudo por cientos o incluso miles de páginas.

Este tipo de programas son muy comunes cuando se trabaja con sitios web, y el web spider que vamos a construir será un excelente punto de partida para tus propios proyectos. Puedes cambiar fácilmente su funcionamiento o agregar nuevas funciones si necesitas construir algo similar. Por ejemplo, podrías querer crear un programa que rastree y extraiga información estructurada de múltiples sitios de forma concurrente (es decir, un *web scraper*).

En nuestra implementación, nos referiremos a menudo a un módulo local llamado `./utils.js`, que contiene algunas funciones auxiliares que utilizaremos en nuestra aplicación. Omitiremos el contenido de este archivo por brevedad, pero puedes encontrar la implementación completa, junto con un archivo `package.json` que contiene la lista completa de dependencias, en el repositorio oficial en [nodejsdp.link/repo4](https://nodejsdp.link/repo4).

La funcionalidad principal de nuestra aplicación está contenida dentro del módulo `spider.js`. Echemos un vistazo a su interior.

Primero, importamos las dependencias necesarias:

```javascript
import { writeFile } from 'node:fs'
import { dirname } from 'node:path'
import { exists, get, recursiveMkdir, urlToFilename } from './utils.js'
```

He aquí una breve explicación de cada función de utilidad importada desde `./utils.js`:
- `exists`: Una función basada en callbacks que comprueba si un archivo existe en el sistema de archivos. Toma una ruta de archivo e invoca el callback con un valor booleano que indica si el archivo existe.
- `get`: Una función basada en callbacks que recupera el cuerpo de una respuesta HTTP para una URL determinada. Toma la URL como entrada y devuelve un `Buffer` que representa el cuerpo de la respuesta.
- `recursiveMkdir`: Otra función basada en callbacks que crea recursivamente todos los directorios necesarios dentro de una ruta especificada.
- `urlToFilename`: Una función síncrona que convierte una URL en una ruta válida del sistema de archivos.

A continuación, definimos una nueva función llamada `spider()`, que toma dos parámetros: la URL a descargar y una función callback que se invoca cuando se completa el proceso de descarga:

```javascript
export function spider(url, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => { // 1
    if (err) { // 1.1
      cb(err)
    } else if (alreadyExists) { // 1.2
      cb(null, filename, false)
    } else { // 1.3
      console.log(`Downloading ${url} into ${filename}`)
      get(url, (err, content) => { // 2
        if (err) {
          cb(err)
        } else {
          recursiveMkdir(dirname(filename), err => { // 3
            if (err) {
              cb(err)
            } else {
              writeFile(filename, content, err => { // 4
                if (err) {
                  cb(err)
                } else {
                  cb(null, filename, true) // 5
                }
              })
            }
          })
        }
      })
    }
  })
}
```

Hay mucho contenido aquí, así que analicemos con más detalle qué sucede en cada paso:
1. El código utiliza la función `exists()` para comprobar si la URL ya se había descargado, verificando si el archivo correspondiente ya está presente en el disco. Podemos tener 3 posibles resultados aquí:
   - **Resultado 1.1:** Obtenemos un error (`err` está definido). Esto significa que hubo un error al acceder al sistema de archivos. En este caso propagamos el error invocando `cb(err)`.
   - **Resultado 1.2:** El archivo ya existe en el disco (`alreadyExists` es `true`). No queremos descargarlo de nuevo; simplemente podemos llamar al callback con el nombre del archivo y `false` (lo que indica al invocador que el archivo no fue descargado activamente).
   - **Resultado 1.3:** El archivo no existe (`alreadyExists` es `false`). Necesitamos descargar el archivo desde la URL.
2. Para descargar el archivo usamos la función `get()`.
3. Debemos asegurarnos de que la carpeta de destino exista. Podemos hacerlo utilizando la función `recursiveMkdir()`.
4. Finalmente, estamos listos para escribir el contenido descargado en el disco utilizando la función `writeFile()` de Node.js.
5. Si todo salió bien, podemos llamar al callback con el nombre del archivo y `true` (indicando esta vez que el archivo sí se descargó).

Para utilizar nuestra aplicación web spider, podemos crear una sencilla aplicación de interfaz de línea de comandos (CLI) que lea una URL como argumento. Esto se puede lograr creando un nuevo archivo llamado `spider-cli.js`:

```javascript
import { spider } from './spider.js'

spider(process.argv[2], (err, filename, downloaded) => {
  if (err) {
    console.error(err)
    process.exit(1)
  }
  if (downloaded) {
    console.log(`Completed the download of "${filename}"`)
  } else {
    console.log(`"${filename}" was already downloaded`)
  }
})
```

Si has copiado los archivos `utils.js` y `package.json` de nuestro repositorio de ejemplos de código, ejecuta `npm install` para descargar todas las dependencias necesarias. Ya estás listo para probar tu aplicación web spider.

Para ejecutarla, navega hasta el directorio de tu proyecto en tu terminal o símbolo del sistema y ejecuta:

```bash
node spider-cli.js https://www.nodejsdesignpatterns.com/
```

Esto debería crear un archivo llamado `www.nodejsdesignpatterns.com/index.html` a partir del directorio de trabajo actual. Siéntete libre de revisar el contenido de este archivo, compararlo con el código HTML fuente del sitio web remoto y experimentar con otras URLs.

Ten en cuenta que nuestra sencilla aplicación web spider requiere que siempre incluyamos el protocolo (por ejemplo, `http://`) en la URL que proporcionamos. Además, no esperes que los enlaces HTML se reescriban ni que se descarguen recursos como imágenes, vídeos, scripts y hojas de estilo. Mantenemos intencionalmente este ejemplo austero porque queremos centrarnos en demostrar cómo funciona la programación asíncrona. ¡Siéntete libre de intentar agregar estas características adicionales por tu cuenta como un divertido desafío!

En la siguiente sección, aprenderás cómo mejorar la legibilidad de este código y, en general, cómo mantener el código basado en callbacks lo más limpio y legible posible.

#### El infierno de los callbacks (*Callback hell*)

Observa más de cerca la función `spider()` que implementamos en la sección anterior. Verás que, incluso con un algoritmo simple, el código se vuelve profundamente anidado y difícil de seguir. Este es un problema común con el código asíncrono.

Ahora, imagina si pudiéramos usar una API bloqueante en su lugar. El código sería mucho más simple y fácil de leer. Para darte una mejor idea, he aquí cómo podría verse la implementación asumiendo que tuviéramos APIs bloqueantes equivalentes para todas nuestras funciones auxiliares:

```javascript
export function spider(url) {
  const filename = urlToFilename(url)
  if (exists(filename)) {
    return false
  } else {
    const content = get(url)
    recursiveMkdir(dirname(filename))
    writeFile(filename, content)
  }
}
```

Observa cómo ni siquiera necesitamos manejar los errores explícitamente; cualquier excepción síncrona se propagará automáticamente por la pila de llamadas.

Sin embargo, las cosas son diferentes cuando se trata de utilizar CPS asíncrono. Definir callbacks en el mismo lugar puede conducir rápidamente a un código desordenado e ilegible.

Este problema, donde una sobrecarga de clausuras y callbacks en línea convierte el código en una maraña inmanejable, se conoce comúnmente como **callback hell** (el infierno de los callbacks). Es uno de los antipatrones más notorios en el desarrollo con Node.js y JavaScript. El código atrapado en esta trampa normalmente tiene un aspecto similar a este:

```javascript
asyncFoo(err => {
  asyncBar(err => {
    asyncFooBar(err => {
      //...
    })
  })
})
```

Puedes ver cómo el código escrito de esta manera adopta la forma de una pirámide (apuntando hacia la derecha) debido al anidamiento profundo, y es por eso que también se le conoce coloquialmente como la **pirámide de la perdición** (*pyramid of doom*).

El problema más obvio con un código como este es su escasa legibilidad. Los niveles profundos de anidamiento pueden dificultar enormemente saber dónde termina una función y dónde comienza otra.

Otro problema surge de la superposición de nombres de variables en diferentes ámbitos (*scopes*). A menudo, terminamos usando nombres similares, o incluso idénticos, para las variables en cada ámbito. Un ejemplo común es el argumento de error pasado a cada callback. Algunos desarrolladores intentan diferenciarlos utilizando ligeras variaciones, como `err`, `error`, `err1`, `err2`, y así sucesivamente. Otros prefieren usar el mismo nombre, como `err`, en todos los ámbitos, sombreando (*shadowing*) la variable del ámbito exterior. Ninguno de estos enfoques es ideal, ya que ambos aumentan la confusión y el riesgo de introducir errores.

Además, las clausuras conllevan cierta sobrecarga de rendimiento y memoria. También pueden provocar fugas de memoria, que no siempre son fáciles de rastrear. De hecho, cualquier contexto referenciado por una clausura activa se retiene y el recolector de basura no lo limpiará.

Recuerda que, cuando se crea una clausura, se "cierra sobre" su entorno léxico circundante, lo que significa que retiene el acceso a las variables y los datos en ese entorno, incluso después de que la función externa haya finalizado. Este es el núcleo de cómo funcionan las clausuras, pero también significa que el contexto referenciado no se puede recolectar mediante el recolector de basura mientras la clausura todavía esté en uso. En esencia, cualquier variable, objeto u otro dato dentro del ámbito de la clausura permanecerá en la memoria.

Si miras nuevamente nuestra función `spider()`, verás que es un claro ejemplo de callback hell, mostrando todos los problemas que acabamos de discutir. Afortunadamente, abordaremos estos problemas utilizando los patrones y técnicas cubiertos en las próximas secciones de este capítulo.

---

### Sección 2: Buenas prácticas con callbacks

Ahora que has conocido tu primer ejemplo de callback hell, sabes qué evitar. Pero gestionar código asíncrono conlleva más desafíos que simplemente prevenir callbacks profundamente anidados. De hecho, existen muchas situaciones en las que controlar el flujo de múltiples tareas asíncronas requiere patrones y técnicas específicos, especialmente si estás trabajando con JavaScript puro sin ninguna biblioteca externa.

Por ejemplo, considera el escenario común en el que tienes un array de URLs y necesitas obtener el contenido de cada URL en secuencia, una tras otra. En la vida real, una razón por la que podrías querer hacer algo como esto secuencialmente podría ser debido a restricciones de límite de tasa (*rate limiting*) (y evitar hacer más de una solicitud a la vez). Para resolver este problema, podrías tener la tentación de usar un bucle `forEach()` simple, como el siguiente:

```javascript
const urls = ['url1', 'url2', 'url3']
urls.forEach(url => {
  fetch(url, response => {
    console.log(`Fetched ${url}`)
    // ... process the response
  })
})
```

Sin embargo, este código no funcionará como cabría esperar. El bucle `forEach()` se ejecutará de forma síncrona, iniciando todas las llamadas a `fetch` con gran rapidez, una tras otra. Debido a que `fetch` es asíncrono, los callbacks se ejecutarán en un punto no especificado en el futuro sin tener en cuenta el orden en el array. Esto daría lugar a que las solicitudes se realicen de forma concurrente y a que los resultados no estén en orden, sin un control claro sobre la ejecución.

Para procesar estas operaciones asíncronas secuencialmente, necesitamos un mecanismo que espere la finalización de cada operación antes de pasar a la siguiente. Una técnica común para este tipo de problema es implementar un patrón similar a la recursión, que es algo que exploraremos más adelante en este capítulo.

En esta sección, no solo aprenderás a mantenerte alejado del callback hell, sino también a implementar algunos de los patrones de flujo de control más comunes, utilizando únicamente JavaScript simple y puro.

Implementar varios flujos de control utilizando callbacks puede volverse engorroso rápidamente si no tenemos cuidado. Antes de profundizar en los detalles de cómo podemos usar callbacks para implementar diferentes flujos de control, tomemos un breve descanso para explorar algunas técnicas que nos ayudarán a escribir código basado en callbacks de mayor calidad y más fácil de mantener. La aplicación de estas técnicas es clave para hacer que el trabajo con callbacks sea más manejable y agradable.

#### Disciplina de callbacks (*Callback discipline*)

Al escribir código asíncrono, la primera regla es evitar el uso excesivo de definiciones de funciones en línea para los callbacks. Si bien puede parecer conveniente, ya que te ahorra preocuparte por la modularidad o la reutilización, has visto cómo a menudo genera más problemas de los que resuelve. En la mayoría de los casos, solucionar el callback hell no requiere bibliotecas, técnicas complejas ni un cambio de paradigmas; solo requiere seguir algunas ideas simples.

He aquí algunos principios básicos que pueden ayudar a reducir el anidamiento y mejorar la organización general del código:
- **Salir temprano (*Exit early*):** Utiliza `return`, `continue` o `break` según sea necesario para salir de una sentencia de inmediato, en lugar de anidar bloques `if...else` completos. Esto mantiene tu código poco profundo y más fácil de seguir.
- **Usar funciones con nombre para los callbacks (*Use named functions for callbacks*):** Mueve los callbacks fuera de las clausuras y pasa los resultados intermedios como argumentos. Las funciones con nombre también proporcionan seguimientos de pila (*stack traces*) más claros, lo cual es útil para la depuración.
- **Modularizar el código (*Modularize your code*):** Divide tu código en funciones más pequeñas y reutilizables siempre que sea posible para promover la claridad y la mantenibilidad.

Ahora, apliquemos estos principios en la práctica.

#### Aplicación de la disciplina de callbacks

Para ilustrar la eficacia de los principios de callbacks que mencionamos, apliquémoslos para resolver el callback hell en nuestra aplicación web spider.

Primero, podemos refactorizar nuestro patrón de verificación de errores eliminando la sentencia `else`. Esto funciona porque podemos retornar de la función inmediatamente cuando se detecta un error. En lugar de escribir:

```javascript
if (err) {
  cb(err)
} else {
  // code to execute when there are no errors
}
```

Podemos optimizar el código de esta manera:

```javascript
if (err) {
  return cb(err)
}
// code to execute when there are no errors
```

Esto se conoce a menudo como el **principio de retorno temprano** (*early return principle*). Con este sencillo truco, logramos de inmediato una reducción en el nivel de anidamiento de nuestras funciones. Es fácil y no requiere ninguna refactorización compleja.

Un error común al ejecutar la optimización que acabamos de describir es olvidar terminar la función después de invocar el callback. Para un escenario de manejo de errores, el siguiente código es una fuente típica de defectos:

```javascript
if (err) {
  cb(err)
}
// code to execute when there are no errors.
```

Nunca debemos olvidar que la ejecución de nuestra función continuará incluso después de invocar el callback. Por lo tanto, es importante insertar una instrucción `return` para bloquear la ejecución del resto de la función. Además, ten en cuenta que en realidad no importa qué valor devuelve la función; el resultado real (o error) se produce de forma asíncrona y se pasa al callback. El valor de retorno de la función asíncrona generalmente se ignora. Esta propiedad nos permite escribir atajos como el siguiente:

```javascript
return cb(...)
```

De lo contrario, tendríamos que escribir código ligeramente más detallado, como el siguiente:

```javascript
cb(...)
return
```

Como segunda optimización para nuestra función `spider()`, podemos intentar identificar piezas de código reutilizables. Por ejemplo, la funcionalidad que escribe una cadena dada en un archivo (asegurándose también de que se cree la ruta del directorio si es necesario) se puede extraer fácilmente en una función separada, de la siguiente manera:

```javascript
function saveFile(filename, content, cb) {
  recursiveMkdir(dirname(filename), err => {
    if (err) {
      return cb(err)
    }
    writeFile(filename, content, cb)
  })
}
```

Siguiendo el mismo principio, podemos crear una función genérica llamada `download()` que tome una URL y un nombre de archivo como entrada, y descargue la URL en el archivo dado. Internamente, podemos usar la función `saveFile()` que creamos anteriormente:

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

Ahora, nuestro código también es más modular y explícito. Para finalizar la refactorización, modificamos la función `spider()`, que ahora se verá de la siguiente manera:

```javascript
export function spider(url, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    if (err) {
      return cb(err)
    }
    if (alreadyExists) {
      return cb(null, filename, false)
    }
    download(url, filename, err => {
      if (err) {
        return cb(err)
      }
      cb(null, filename, true)
    })
  })
}
```

La funcionalidad y la interfaz de la función `spider()` permanecen sin cambios; lo que hemos mejorado es cómo está organizado el código. Al aplicar el principio de retorno temprano y otras técnicas de disciplina de callbacks, redujimos significativamente el anidamiento del código al tiempo que aumentamos tanto la reutilización como la capacidad de prueba (*testability*). De hecho, podríamos considerar exportar tanto `saveFile()` como `download()` y moverlas a nuestra biblioteca de utilidades compartida para hacerlas reutilizables en diferentes módulos. Esto también nos permitiría probar estas funciones de forma aislada, mejorando la calidad de nuestro código.

Esta refactorización demuestra que, en la mayoría de los casos, todo lo que se necesita es cierta disciplina para evitar el uso excesivo de clausuras y funciones anónimas. Es un enfoque simple y eficaz que requiere un esfuerzo mínimo y no depende de bibliotecas externas.

Ahora que sabes cómo escribir código asíncrono limpio con callbacks, estamos listos para sumergirnos en los patrones asíncronos comunes, como la ejecución secuencial y la concurrente.

---

### Sección 3: Patrones de control de flujo

En esta sección, examinaremos los patrones de flujo de control asíncrono y comenzaremos analizando el flujo de ejecución secuencial.

#### Ejecución secuencial (*Sequential execution*)

Ejecutar un conjunto de tareas en secuencia significa ejecutarlas una a la vez, una tras otra. El orden de ejecución importa y debe preservarse, porque el resultado de una tarea en la lista puede afectar la ejecución de la siguiente. La Figura 4.1 ilustra este concepto:

**Figura 4.1:** Ejemplo de un flujo de ejecución secuencial con tres tareas.

Existen diferentes variaciones de este flujo:
- Ejecutar un conjunto de tareas conocidas en secuencia, sin propagar datos entre ellas.
- Utilizar la salida de una tarea como entrada para la siguiente (también conocido como cadena, tubería o cascada / *chain, pipeline, or waterfall*).
- Iterar sobre una colección mientras se ejecuta una tarea asíncrona en cada elemento, uno tras otro.

La ejecución secuencial suele ser la causa principal del problema del callback hell cuando se utiliza CPS asíncrono.

##### Ejecución de un conjunto conocido de tareas en secuencia

Quizás no te hayas dado cuenta, pero en la sección anterior ya profundizamos en el concepto de flujo de ejecución secuencial mientras implementábamos la función `spider()`. Si te tomas un momento para pensarlo, verás que nuestro spider realiza varias tareas asíncronas en un orden específico, y cada tarea se completa antes de que comience la siguiente: verificar si el archivo ya existe, descargar contenido desde una URL remota y guardar ese contenido en un archivo. Siguiendo algunas reglas simples, pudimos organizar estas tareas en un flujo secuencial. Ahora, utilizando ese código como referencia, podemos generalizar nuestra solución con el siguiente patrón:

```javascript
function task1(cb) {
  asyncOperation(() => {
    task2(cb) // calls task2 with the current callback
  })
}
function task2(cb) {
  asyncOperation(() => {
    task3(cb) // calls task3 with the current callback
  })
}
function task3(cb) {
  asyncOperation(() => {
    cb() // finally completes and executes the callback
  })
}
task1(() => {
  // executed when task1, task2, and task3 are completed
  console.log('tasks 1, 2, and 3 executed')
})
```

El patrón anterior demuestra cómo cada tarea desencadena la siguiente tras la finalización de una operación asíncrona genérica. Destaca la modularización de las tareas y demuestra que las clausuras no siempre son necesarias para gestionar el código asíncrono.

##### Iteración secuencial (*Sequential iteration*)

El patrón de ejecución secuencial funciona muy bien cuando sabemos de antemano qué tareas deben ejecutarse y cuántas son, especialmente si el número de tareas es relativamente pequeño. Esto nos permite codificar directamente la invocación de cada tarea posterior en la secuencia. Pero, ¿qué sucede cuando queremos realizar una operación asíncrona para cada elemento de una colección? En situaciones como esta, ya no podemos codificar de forma fija la secuencia de tareas; en su lugar, necesitamos construirla dinámicamente.

##### Web spider versión 2

Para ilustrar la iteración secuencial, introduzcamos una nueva característica en nuestra aplicación web spider: la capacidad de descargar todos los enlaces de una página web de forma recursiva, siempre que los enlaces permanezcan dentro del mismo dominio. Después de todo, es un spider: ¡puede rastrear la web, siguiendo enlaces hacia sus profundidades! Para lograr esto, extraeremos todos los enlaces de la página actual y haremos que nuestro web spider siga cada uno, descargándolos recursivamente en secuencia.

El primer paso es modificar nuestra función `spider()` para desencadenar una descarga recursiva de los enlaces de la página mediante el uso de una nueva función llamada `spiderLinks()`, que crearemos en breve.

Una decisión de diseño clave aquí es garantizar que el spider no se quede atascado en un bucle sin fin. Para evitar esto, introduciremos un parámetro `maxDepth` que limite la profundidad de la recursión. He aquí el código actualizado para la función `spider()`:

```javascript
export function spider(url, maxDepth, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    if (err) {
      // error checking the file
      return cb(err)
    }
    if (alreadyExists) {
      if (!filename.endsWith('.html')) {
        // ignoring non-HTML resources
        return cb()
      }
      // If the page was already downloaded, read the contents and download the links
      return readFile(filename, 'utf8', (err, fileContent) => {
        if (err) {
          // error reading the file
          return cb(err)
        }
        return spiderLinks(url, fileContent, maxDepth, cb)
      })
    }
    // The file does not exist, download it
    download(url, filename, (err, fileContent) => {
      if (err) {
        // error downloading the file
        return cb(err)
      }
      // if the file is an HTML file, spider it
      if (filename.endsWith('.html')) {
        return spiderLinks(url, fileContent.toString('utf8'), maxDepth, cb)
      }
      // otherwise, stop here
      return cb()
    })
  })
}
```

Hemos añadido comentarios en todo el código, por lo que debería ser fácil seguir lo que hace esta nueva versión.

En la siguiente sección, exploraremos cómo se implementa la función `spiderLinks()`.

##### Rastreo secuencial de enlaces (*Sequential crawling of links*)

Ahora, podemos construir el núcleo de esta nueva versión de nuestro web spider: la función `spiderLinks()`. Esta función descarga todos los enlaces de una página HTML utilizando un algoritmo de iteración asíncrona secuencial. Observa con atención cómo la definimos en el siguiente bloque de código:

```javascript
function spiderLinks(currentUrl, body, maxDepth, cb) {
  if (maxDepth === 0) { // 1
    return process.nextTick(cb)
  }
  const links = getPageLinks(currentUrl, body) // 2
  if (links.length === 0) {
    return process.nextTick(cb)
  }
  function iterate(index) { // 3
    if (index === links.length) {
      return cb()
    }
    spider(links[index], maxDepth - 1, err => { // 4
      if (err) {
        return cb(err)
      }
      iterate(index + 1)
    })
  }
  iterate(0) // 5
}
```

He aquí los pasos clave para comprender esta nueva función:
1. **Caso base:** Si la variable `maxDepth` ha llegado a 0, detenemos el rastreo. Observa cómo llamamos al callback de forma asíncrona para evitar al infame Zalgo.
2. Extraemos todos los enlaces de la página utilizando la función `getPageLinks()`, que es síncrona y devuelve solo enlaces que apuntan al mismo nombre de host (esta función está implementada en el archivo `utils.js`). Si no hay enlaces, detenemos el proceso.
3. Usamos una función local llamada `iterate()` para recorrer los enlaces. Esta función toma el índice del siguiente enlace a procesar. Lo primero que hace es comprobar si el índice es igual a la longitud del array `links`. Si es así, significa que hemos procesado todos los elementos y podemos detenernos invocando `cb()`.
4. En esta etapa, todo está listo para procesar el enlace. Llamamos a la función `spider()`, reduciendo la profundidad máxima de recursión (`maxDepth - 1`), y continuamos con el siguiente paso de la iteración una vez que la operación se complete.
5. Como paso final en `spiderLinks()`, iniciamos la iteración llamando a `iterate(0)`.

El algoritmo que acabamos de presentar nos permite iterar sobre un array ejecutando una operación asíncrona de forma secuencial, que, en nuestro caso, es la función `spider()`.

Ahora, actualicemos `spider-cli.js` para permitirnos especificar el nivel de anidamiento como un argumento adicional de la interfaz de línea de comandos (CLI):

```javascript
import { spider } from './spider.js'

const url = process.argv[2]
const maxDepth = Number.parseInt(process.argv[3], 10) || 1

spider(url, maxDepth, err => {
  if (err) {
    console.error(err)
    process.exit(1)
  }
  console.log('Downloaded complete')
})
```

Ahora podemos probar esta nueva versión de nuestra aplicación spider y ver cómo descarga todos los enlaces de una página web de forma recursiva, uno tras otro. Para detener el proceso —ya que puede llevar un tiempo si hay muchos enlaces—, recuerda que siempre puedes presionar `Ctrl + C`. Si deseas reanudarlo más tarde, puedes reiniciar el spider con la misma URL que utilizaste en la primera ejecución.

Ahora que nuestro web spider tiene el potencial de descargar un sitio web completo, úsalo con precaución. Por ejemplo, evita establecer un nivel de profundidad máxima alto o dejar el spider ejecutándose durante demasiado tiempo. Sobrecargar un servidor con miles de solicitudes no solo es de mala educación, sino que, en algunos casos, incluso puede ser ilegal. Por ejemplo, podría considerarse un ataque DoS (*Denial of Service*). ¡Rastrea con responsabilidad!

##### El patrón (*The pattern*)

El código de la función `spiderLinks()` de la sección anterior es un ejemplo claro de cómo es posible iterar sobre una colección mientras se aplica una operación asíncrona. También puedes notar que es un patrón que se puede adaptar a cualquier otra situación en la que necesitemos iterar asíncronamente sobre los elementos de una colección o, en general, sobre una lista de tareas. Este patrón se puede generalizar de la siguiente manera:

```javascript
function iterate (index) {
  if (index === tasks.length) {
    return finish()
  }
  const task = tasks[index]
  task(() => iterate(index + 1))
}
function finish () {
  // iteration completed
}
iterate(0)
```

Es importante tener en cuenta que este tipo de algoritmos se vuelven recursivos si `task()` es una operación síncrona. En tal caso, la pila no se desbobinará (*unwind*) en cada ciclo y podría existir el riesgo de alcanzar el límite de tamaño máximo de la pila de llamadas (*maximum call stack size*).

El patrón que se acaba de presentar es muy potente y puede ampliarse o adaptarse para abordar varias necesidades comunes. Solo por mencionar algunos ejemplos:
- Podemos mapear los valores de un array de forma asíncrona.
- Podemos pasar los resultados de una operación a la siguiente en la iteración para implementar una versión asíncrona del algoritmo `reduce`.
- Podemos salir del bucle prematuramente si se cumple una condición particular (implementación asíncrona del auxiliar `Array.some()`).
- Incluso podemos iterar sobre un número infinito de elementos.

También podríamos optar por generalizar la solución aún más envolviéndola en una función con una firma como la siguiente:

```javascript
iterateSeries(collection, iteratorCallback, finalCallback)
```

Aquí, `collection` es el conjunto de datos real sobre el que deseas iterar, `iteratorCallback` es la función a ejecutar sobre cada elemento y `finalCallback` es la función que se ejecuta cuando se procesan todos los elementos o en caso de error. La implementación de esta función auxiliar se deja como ejercicio.

> **El patrón Iterador Secuencial (*The Sequential Iterator pattern*):**
> Permite la ejecución de tareas dentro de una colección, una tras otra, asegurando un flujo controlado. Esto se logra mediante el uso de una función iteradora que activa automáticamente la siguiente tarea en la secuencia tan pronto como se completa la actual.

En la siguiente sección, exploraremos el patrón de ejecución concurrente, que resulta más conveniente cuando el orden de las distintas tareas no es importante.

---

### Sección 4: Ejecución concurrente

Existen algunas situaciones en las que el orden de ejecución de un conjunto de tareas asíncronas no es importante y no existe una correlación lógica ni ninguna dependencia de datos entre estas tareas. Todo lo que queremos es que se nos notifique cuando se hayan completado todas esas tareas en ejecución. Tales situaciones se manejan mejor utilizando un flujo de ejecución concurrente.

Llegados a este punto del libro, hemos utilizado las palabras "paralelo" y "concurrente" unas cuantas veces. Pueden sonar similares, pero la diferencia entre ellas es un concepto importante de entender, especialmente en el contexto de Node.js.

Como comentamos en el [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1), La plataforma Node.js, Node.js es de un solo hilo (*single-threaded*), lo que hace que comprender el concepto de concurrencia sea especialmente relevante. Antes de continuar, intentemos definir estos dos conceptos con una analogía.

Imagina que diriges un restaurante y recibes dos pedidos al mismo tiempo. En la cocina, hay dos formas de gestionar esto:
- **Paralelismo (*Parallelism*):** Si tienes dos chefs, cada uno puede preparar un pedido simultáneamente, trabajando de forma completamente independiente. Esto es como tener múltiples núcleos de CPU ejecutando tareas en paralelo.
- **Concurrencia (*Concurrency*):** Si solo hay un chef, debe dividir su tiempo de manera eficiente. Mientras espera a que hierva el agua para un plato, puede empezar a picar ingredientes para el otro. El chef no está trabajando en ambas tareas simultáneamente, pero está progresando en ambas. Así es como funciona un bucle de eventos, alternando entre tareas de manera eficiente mientras espera operaciones más lentas como las de E/S.

Node.js está construido sobre JavaScript, que se ejecuta en un bucle de eventos de un solo hilo. Esto significa que, por defecto, Node.js destaca en la ejecución concurrente, gestionando múltiples tareas sin bloquear el hilo principal.

Si bien el paralelismo se puede lograr utilizando hilos de trabajo (*worker threads*) o generando múltiples procesos, la concurrencia suele ser más eficiente y ligera para muchas aplicaciones del mundo real, como el manejo de múltiples solicitudes de red o consultas a bases de datos.

Una ejecución paralela de múltiples tareas podría representarse como en la Figura 4.2:

**Figura 4.2:** Ejemplo de ejecución paralela con tres tareas.

Aunque esta imagen se utiliza generalmente para representar la ejecución paralela de múltiples tareas, también puede ayudar a describir la concurrencia a un alto nivel, donde múltiples tareas progresan activamente dentro del bucle de eventos. La distinción clave entre la ejecución paralela y la concurrente radica en cómo se progresa: la ejecución paralela ejecuta tareas simultáneamente, mientras que la concurrencia implica alternar eficientemente entre tareas para mantener las cosas en movimiento.

Para aclarar aún más nuestra comprensión, veamos otro diagrama que muestra cómo dos tareas asíncronas pueden ejecutarse concurrentemente en un programa de Node.js:

**Figura 4.3:** Ejemplo de cómo se ejecutan concurrentemente las tareas asíncronas.

En la Figura 4.3, tenemos una función `Main` que ejecuta dos tareas asíncronas:
1. La función `Main` desencadena la ejecución de la `Tarea 1` y la `Tarea 2`. Como desencadenan una operación asíncrona, devuelven inmediatamente el control a la función `Main`, que luego lo devuelve al bucle de eventos.
2. Cuando se completa la operación asíncrona de la `Tarea 1`, el bucle de eventos le cede el control. Cuando la `Tarea 1` completa también su procesamiento síncrono interno, notifica a la función `Main`.
3. Cuando se completa la operación asíncrona desencadenada por la `Tarea 2`, el bucle de eventos invoca su callback, devolviendo el control a la `Tarea 2`. Al final de la `Tarea 2`, se notifica a la función `Main` una vez más. En este punto, la función `Main` sabe que tanto la `Tarea 1` como la `Tarea 2` están completas, por lo que puede continuar su ejecución o devolver los resultados de las operaciones a otro callback.

En resumen, esto significa que en Node.js generalmente ejecutamos operaciones asíncronas de forma concurrente, porque su concurrencia es manejada internamente por las APIs no bloqueantes. En Node.js, las operaciones síncronas (bloqueantes) no se pueden paralelizar fácilmente ni ejecutar de forma concurrente a menos que su ejecución se entrelace con una operación asíncrona, o se entrelace con `setTimeout()` o `setImmediate()`. Verás estas técnicas con más detalle en el [Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas.

#### Web spider versión 3

Nuestra aplicación web spider parece una candidata perfecta para aplicar el concepto de ejecución concurrente. Hasta ahora, nuestra aplicación ejecuta la descarga recursiva de las páginas enlazadas de manera secuencial. Podemos mejorar fácilmente el rendimiento de este proceso descargando todas las páginas enlazadas simultáneamente de forma concurrente.

Para hacer eso, solo necesitamos modificar la función `spiderLinks()` para asegurarnos de generar todas las tareas `spider()` a la vez y luego invocar el callback final solo cuando todas ellas hayan completado su ejecución. Por lo tanto, modifiquemos nuestra función `spiderLinks()` de la siguiente manera:

```javascript
function spiderLinks(currentUrl, body, maxDepth, cb) {
  if (maxDepth === 0) {
    return process.nextTick(cb)
  }
  const links = getPageLinks(currentUrl, body)
  if (links.length === 0) {
    return process.nextTick(cb)
  }
  let completed = 0
  let hasErrors = false // 3
  function done(err) { // 2
    if (err) {
      hasErrors = true
      return cb(err)
    }
    if (++completed === links.length && !hasErrors) {
      return cb()
    }
  }
  for (const link of links) { // 1
    spider(link, maxDepth - 1, done)
  }
}
```

Analicemos qué hemos cambiado:
1. Como se mencionó anteriormente, las tareas `spider()` ahora se inician todas a la vez. Esto es posible simplemente iterando sobre el array `links` e iniciando cada tarea sin esperar a que termine la anterior.
2. Luego, el truco para hacer que nuestra aplicación espere a que se completen todas las tareas es proporcionar a la función `spider()` un callback especial, al que llamamos `done()`. La función `done()` incrementa un contador cuando se completa una tarea del spider. Cuando el número de descargas completadas alcanza el tamaño del array `links`, se invoca el callback final (`cb`).
3. La variable `hasErrors` es necesaria porque si una tarea concurrente falla, queremos llamar inmediatamente al callback con el error dado. Además, debemos asegurarnos de que otras tareas concurrentes que aún puedan estar ejecutándose no vuelvan a invocar el callback.

Con estos cambios implementados, si ahora intentamos ejecutar nuestro spider contra una página web, notaremos una gran mejora en la velocidad del proceso general, ya que cada descarga se llevará a cabo de forma concurrente, sin esperar a que se procese el enlace anterior.

#### El patrón (*The pattern*)

Finalmente, podemos extraer nuestro bonito patrón para el flujo de ejecución concurrente. Representemos una versión genérica del patrón con el siguiente código:

```javascript
const tasks = [ /* ... */ ]
let completed = 0
for (const task of tasks) {
  task(() => {
    if (++completed === tasks.length) {
      finish()
    }
  })
}
function finish () {
  // all the tasks completed
}
```

Con pequeñas modificaciones, podemos adaptar el patrón para acumular los resultados de cada tarea en una colección, filtrar o mapear los elementos de un array, o invocar el callback `finish()` tan pronto como se completen una o un número determinado de tareas (esta última situación en particular se denomina **carrera competitiva** o *competitive race*).

> **El patrón de Ejecución Concurrente Ilimitada (*The Unlimited Concurrent Execution pattern*):**
> Este patrón implica ejecutar un conjunto de tareas asíncronas de forma concurrente lanzándolas todas a la vez y esperando su finalización. Todas las tareas se inician de inmediato y la finalización se rastrea contando cuántas veces se invocan sus callbacks.

Cuando tenemos múltiples tareas ejecutándose de forma concurrente, podemos tener condiciones de carrera (*race conditions*), es decir, contienda para acceder a recursos externos (por ejemplo, archivos o registros en una base de datos). En la siguiente sección, hablaremos sobre las condiciones de carrera en Node.js y exploraremos algunas técnicas para identificarlas y abordarlas.

---

### Sección 5: Solución de condiciones de carrera con tareas concurrentes

Ejecutar un conjunto de tareas en paralelo puede causar problemas al utilizar E/S bloqueante en combinación con múltiples hilos. Sin embargo, acabas de ver que, en Node.js, la historia es totalmente diferente. Ejecutar múltiples tareas asíncronas de forma concurrente es, de hecho, directo y económico en términos de recursos.

Esta es una de las fortalezas más importantes de Node.js, porque hace que ejecutar múltiples tareas a través de la concurrencia sea una práctica común en lugar de una técnica compleja que solo se utiliza cuando es estrictamente necesario.

Otra característica importante del modelo de concurrencia de Node.js es cómo maneja la sincronización de tareas y las condiciones de carrera. En un entorno multihilo, la gestión de recursos compartidos normalmente requiere mecanismos de sincronización como bloqueos (*locks*), exclusiones mutuas (*mutexes*), semáforos y monitores. Estas construcciones ayudan a coordinar el acceso a los datos compartidos, pero pueden introducir una complejidad considerable y una sobrecarga de rendimiento.

Para retomar nuestra analogía de la cocina, imagina dos chefs trabajando en paralelo, cada uno preparando su propio plato. Si ambos necesitan usar el mismo fregadero al mismo tiempo para lavar ingredientes, no pueden proceder simultáneamente; deben turnarse o arriesgarse a estorbarse mutuamente. En la programación multihilo, mecanismos como los bloqueos actúan como una forma de gestionar esta contienda, asegurando que solo una tarea acceda al recurso compartido a la vez. Sin embargo, una sincronización mal administrada puede provocar ineficiencias, como que un chef bloquee el fregadero durante demasiado tiempo, ralentizando toda la cocina.

En Node.js, normalmente no necesitamos un mecanismo de sincronización complejo, ya que todo se ejecuta en un solo hilo. Sin embargo, esto no significa que no podamos tener condiciones de carrera; al contrario, pueden ser bastante comunes. La raíz del problema es el retraso entre la invocación de una operación asíncrona y la notificación de su resultado.

> Una **condición de carrera** (*race condition*) es una situación en la que el comportamiento de un programa depende de la sincronización o el orden temporal de las operaciones concurrentes que acceden a recursos compartidos.

Para ver un ejemplo concreto, nos referiremos nuevamente a nuestra aplicación web spider y, en particular, a la última versión que creamos, que contiene una condición de carrera. ¿Puedes detectarla? Tómate unos minutos para pensarlo y ver si puedes adivinar cuál es el problema. ¡Estaremos aquí esperándote!

El problema del que estamos hablando radica en la función `spider()`, donde verificamos si un archivo ya existe antes de comenzar a descargar la URL correspondiente:

```javascript
export function spider(url, maxDepth, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    // ...
    if (alreadyExists) {
      // ...
    } else {
      download(url, filename, (err, fileContent) => {
        // ...
```

El problema es que dos tareas del spider que operan sobre la misma URL pueden invocar `exists()` sobre el mismo archivo antes de que una de las dos tareas complete la descarga y cree un archivo, lo que provoca que ambas tareas inicien una descarga. La Figura 4.4 explica esta situación:

**Figura 4.4:** Ejemplo de una condición de carrera en nuestra función spider().

La Figura 4.4 muestra cómo la Tarea 1 y la Tarea 2 se entrelazan en el único hilo de Node.js, así como la forma en que una operación asíncrona puede introducir en la práctica una condición de carrera. En nuestro caso, las dos tareas del spider terminan descargando el mismo archivo.

¿Cómo podemos solucionar esto? La respuesta es mucho más simple de lo que podrías pensar. De hecho, todo lo que necesitamos es una variable para excluir mutuamente múltiples tareas `spider()` que se ejecutan sobre la misma URL. Esto se puede lograr con un código como el siguiente:

```javascript
const spidering = new Set()

function spider (url, nesting, cb) {
  if (spidering.has(url)) {
    return process.nextTick(cb)
  }
  spidering.add(url)
  // ...
```

Simplemente salimos de la función de inmediato si la URL ya está en el conjunto (*set*) global `spidering` (sé que *spidering* no es técnicamente una palabra formal, pero suena genial, ¿verdad?). De lo contrario, agregamos la URL al conjunto y procedemos con la descarga. Dado que no queremos descargar una URL más de una vez, no necesitamos eliminar las URLs del conjunto.

Si estás construyendo un spider que podría tener que descargar cientos de miles de páginas web, eliminar la URL descargada del conjunto una vez que se descarga un archivo te ayudará a evitar que la cardinalidad del conjunto y, por lo tanto, el consumo de memoria, crezcan indefinidamente.

Las condiciones de carrera pueden causar todo tipo de problemas, incluso en un entorno de un solo hilo como Node.js. En algunos casos, pueden provocar daños en los datos (*data corruption*) y son notoriamente difíciles de depurar debido a su naturaleza fugaz. Es por eso que siempre es una buena idea verificar dos veces si existen posibles condiciones de carrera al ejecutar tareas de forma concurrente.

Hablando de tareas concurrentes, ejecutar demasiadas tareas concurrentes a la vez suele ser una mala idea: podrías agotar rápidamente la memoria del sistema o encontrarte con límites como el número máximo de descriptores de archivos abiertos. En la siguiente sección, profundizaremos en por qué esto puede ser un problema y cómo gestionar el número de tareas concurrentes de forma eficaz.

---

### Sección 6: Ejecución concurrente limitada

Generar tareas concurrentes sin ningún control puede provocar fácilmente una sobrecarga excesiva. Imagina intentar leer miles de archivos, acceder a múltiples URLs o ejecutar numerosas consultas a bases de datos, todo a la vez. El problema más común en tales casos es quedarse sin recursos. Por ejemplo, una aplicación podría intentar abrir demasiados archivos al mismo tiempo, agotando rápidamente los descriptores de archivos disponibles.

Un servidor que lanza un número ilimitado de tareas concurrentes en respuesta a las solicitudes de los usuarios también puede volverse vulnerable a un ataque de denegación de servicio (DoS). En este tipo de ataque, un actor malintencionado puede diseñar una o más solicitudes que de algún modo obliguen al servidor a consumir todos sus recursos y dejar de responder. Limitar el número de tareas concurrentes es una buena práctica que ayuda a construir aplicaciones más resilientes.

Actualmente, la Versión 3 de nuestro web spider no impone ningún límite a las tareas concurrentes, lo que lo hace propenso a fallar en varios escenarios. Por ejemplo, si lo ejecutamos en un sitio web grande, podría ejecutarse durante unos segundos antes de fallar con un error `ECONNREFUSED`. Esto sucede porque, cuando se descargan demasiadas páginas de forma concurrente, el servidor web puede comenzar a rechazar nuevas conexiones desde la misma IP. En tales casos, nuestro spider se bloquearía y tendríamos que reiniciar el proceso para continuar rastreando el sitio. Si bien podríamos manejar el error `ECONNREFUSED` para evitar el bloqueo, aún correríamos el riesgo de sobrecargar el sistema al asignar demasiadas tareas concurrentes, lo que provocaría otros problemas.

En esta sección, exploraremos cómo hacer que nuestro spider sea más resiliente limitando la concurrencia.

El siguiente diagrama muestra un escenario donde tenemos 5 tareas para ejecutar concurrentemente, pero hemos establecido un límite de concurrencia de 2:

**Figura 4.5:** Ejemplo de cómo se puede limitar la concurrencia a un máximo de dos tareas al mismo tiempo.

Al observar la Figura 4.5, podemos ver que, con este ejemplo particular, el algoritmo funciona de la siguiente manera:
1. Primero, iniciamos tantas tareas como sea posible sin exceder el límite de concurrencia. Dado que el límite está establecido en dos, el nodo Inicio (*Start*) lanza la `Tarea 1` y la `Tarea 2`.
2. Tan pronto como finaliza una tarea, lanzamos la siguiente en la fila, teniendo siempre en cuenta el límite. Por ejemplo, cuando finaliza la `Tarea 2`, se inicia la `Tarea 3`. Cuando finaliza la `Tarea 1`, se inicia la `Tarea 4`. Una vez que finaliza la `Tarea 3`, pasamos a la tarea final, la `Tarea 5`. Cuando la `Tarea 5` se completa y no hay más tareas para ejecutar, todo el proceso termina.

En la siguiente sección, exploraremos una posible implementación del patrón de ejecución concurrente limitada.

#### Limitación de concurrencia (*Limiting concurrency*)

Examinaremos ahora un patrón que ejecutará un conjunto de tareas determinadas de forma concurrente con concurrencia limitada:

```javascript
const tasks = [
  // ...
]
const concurrency = 2
let running = 0
let completed = 0
let nextTaskIndex = 0

function next() { // 1
  while (running < concurrency && nextTaskIndex < tasks.length) {
    const task = tasks[nextTaskIndex++]
    task(() => { // 2
      if (++completed === tasks.length) {
        return finish()
      }
      running--
      next()
    })
    running++
  }
}
next()

function finish() {
  // all the tasks completed
}
```

Este algoritmo puede considerarse una mezcla de ejecución secuencial y ejecución concurrente. De hecho, puedes notar similitudes con ambos patrones:
1. Tenemos una función llamada `next()` que actúa como iterador. En su interior, hay un bucle que inicia tantas tareas como sea posible, manteniendo el número de tareas en ejecución dentro del límite de concurrencia establecido.
2. Cada vez que se inicia una tarea, le pasamos un callback. En ese callback, decidimos qué hacer cuando la tarea se completa. Si quedan más tareas, llamamos a `next()` nuevamente para mantener las cosas en marcha. Si todas las tareas han finalizado, llamamos a `finish()` para indicar que todo está listo. En el camino, nos aseguramos de actualizar adecuadamente nuestros contadores: incrementando el recuento de tareas completadas y ajustando el número de tareas en ejecución (que se incrementa cada vez que `next()` inicia una nueva tarea).

Bastante simple, ¿verdad?

En este patrón, controlamos la concurrencia rastreando cuántas tareas se están ejecutando en un momento dado mediante la variable `running`. Dado que Node.js ejecuta JavaScript en un bucle de eventos de un solo hilo, no hay riesgo de que múltiples tareas modifiquen `running` al mismo tiempo; cada actualización ocurre en secuencia, lo que hace innecesaria la sincronización. A medida que las tareas se inician, `running` aumenta, y a medida que se completan, disminuye antes de activar `next()` para lanzar más tareas si es necesario. Debido a que la ejecución siempre está ordenada dentro del bucle de eventos, `running` siempre es precisa, sin la complejidad de bloqueos o condiciones de carrera. Esta es una ventaja clave del modelo de concurrencia de Node.js: gestión eficiente de tareas sin los inconvenientes del multihilo.

#### Limitación global de concurrencia (*Globally limiting concurrency*)

Nuestra aplicación web spider es un excelente ejemplo de dónde podemos aplicar lo que acabamos de aprender sobre cómo limitar el número de tareas que se ejecutan a la vez. Sin dicho límite, podríamos terminar rastreando miles de enlaces simultáneamente, lo que podría saturar nuestro sistema. Al controlar la cantidad de descargas concurrentes, podemos agregar cierta previsibilidad muy necesaria al proceso.

Ahora bien, podríamos aplicar el patrón de concurrencia limitada a nuestra función `spiderLinks()`. Sin embargo, esto solo limitaría la cantidad de tareas para los enlaces que se encuentran en una sola página. Por ejemplo, si establecemos el límite de concurrencia en dos, tendríamos como máximo dos enlaces descargándose al mismo tiempo por página. Pero dado que cada página podría generar dos descargas más, esto aún podría provocar que el número total de descargas crezca rápidamente, potencialmente fuera de control.

En general, esta implementación particular del patrón de concurrencia limitada funciona bien cuando se tiene un conjunto predefinido de tareas, o cuando las tareas aumentan a un ritmo manejable. Sin embargo, en casos como nuestro web spider —donde una tarea puede generar múltiples tareas nuevas— no es eficaz para limitar la concurrencia general. Para solucionar esto, necesitamos introducir un mecanismo que nos permita controlar la concurrencia a nivel global.

#### Colas al rescate (*Queues to the rescue*)

Lo que realmente queremos es limitar el número total de operaciones de descarga que se ejecutan de forma concurrente. Una buena forma de lograrlo es introduciendo colas para gestionar la concurrencia de múltiples tareas. Veamos cómo funciona esto.

Ahora implementaremos una clase simple llamada `TaskQueue`, que combina una cola con el algoritmo de concurrencia limitada que acabamos de comentar. Creemos un nuevo módulo llamado `taskQueue.js`:

```javascript
export class TaskQueue {
  constructor(concurrency) {
    this.concurrency = concurrency
    this.running = 0
    this.queue = []
  }

  pushTask(task) {
    this.queue.push(task)
    process.nextTick(this.next.bind(this))
    return this
  }

  next() {
    while (
      this.running < this.concurrency &&
      this.queue.length > 0
    ) {
      const task = this.queue.shift()
      task(() => {
        this.running--
        process.nextTick(this.next.bind(this))
      })
      this.running++
    }
  }
}
```

El constructor de esta clase toma como entrada únicamente el límite de concurrencia, pero además de eso, inicializa las variables de instancia `running` y `queue`. La primera variable es un contador utilizado para realizar un seguimiento de todas las tareas en ejecución, mientras que la segunda es el array que se utilizará como cola para almacenar las tareas pendientes.

El método `pushTask()` simplemente añade una nueva tarea a la cola y luego arranca la ejecución del trabajador invocando asíncronamente `this.next()`.

El método `next()` genera un conjunto de tareas desde la cola, asegurando que no exceda el límite de concurrencia.

Puedes notar que este método tiene algunas similitudes con el patrón presentado al principio de la sección *Limitación de concurrencia*. Básicamente inicia tantas tareas de la cola como sea posible, sin exceder el límite de concurrencia. Cuando se completa cada tarea, actualiza el recuento de tareas en ejecución y luego inicia otra ronda de tareas invocando asíncronamente `next()` nuevamente. La propiedad interesante de la clase `TaskQueue` es que nos permite agregar dinámicamente nuevos elementos a la cola. La otra ventaja es que, ahora, tenemos una entidad central responsable de limitar la concurrencia de nuestras tareas, que se puede compartir entre todas las instancias de ejecución de una función. En nuestro caso, es la función `spider()`, como verás en un momento.

#### Perfeccionamiento de TaskQueue (*Refining the TaskQueue*)

La implementación anterior de `TaskQueue` es suficiente para demostrar el patrón de cola, pero para ser utilizada en proyectos de la vida real, necesita un par de características adicionales. Por ejemplo, ¿cómo podemos saber cuándo ha fallado una de las tareas? ¿Cómo sabemos si se ha completado todo el trabajo en la cola?

Recuperemos algunos de los conceptos que discutimos en el [Capítulo 3](https://subscription.packtpub.com/book/web-development/9781803238944/3), Callbacks y Eventos, y convirtamos `TaskQueue` en un `EventEmitter` para que podamos emitir eventos para propagar fallos de tareas e informar a cualquier observador cuando la cola esté vacía.

El primer cambio que debemos hacer es importar la clase `EventEmitter` y hacer que nuestro `TaskQueue` la extienda:

```javascript
import { EventEmitter } from 'node:events'

export class TaskQueue extends EventEmitter {
  constructor (concurrency) {
    super()
    // ...
  }
  // ...
}
```

En este punto, podemos usar `this.emit` para disparar eventos desde dentro del método `next()` de `TaskQueue`:

```javascript
next () {
  if (this.running === 0 && this.queue.length === 0) { // 1
    return this.emit('empty')
  }

  while (this.running < this.concurrency && this.queue.length) {
    const task = this.queue.shift()
    task((err) => { // 2
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

Comparando esta implementación con la anterior, hay dos adiciones aquí:
1. Cada vez que se llama a la función `next()`, comprobamos que no haya ninguna tarea en ejecución y si la cola está vacía. En tal caso, significa que la cola se ha vaciado por completo y podemos disparar el evento `empty`.
2. El callback de finalización de cada tarea ahora se puede invocar pasando un error. Comprobamos si efectivamente se pasa un error, lo que indica que la tarea ha fallado, y en ese caso, propagamos dicho error con un evento `error`.

Ten en cuenta que, en caso de error, mantenemos deliberadamente la cola en funcionamiento. No detenemos otras tareas en curso ni eliminamos ninguna tarea pendiente. Esto es bastante común en los sistemas basados en colas. Se espera que ocurran errores y, en lugar de permitir que el sistema se bloquee en esas ocasiones, generalmente es mejor identificar los errores y pensar en estrategias de reintento o recuperación. Discutiremos estos conceptos un poco más en el [Capítulo 13](https://subscription.packtpub.com/book/web-development/9781803238944/13), Patrones de mensajería e integración.

Hipotéticamente, si quisieras detener todas las tareas con elegancia al encontrar un error, podrías considerar introducir un mecanismo que indique a `TaskQueue` que deje de aceptar y procesar nuevas tareas. Imagina un escenario en el que añadimos un indicador `this.stopped` a la clase `TaskQueue`. En caso de error, este indicador se establece inmediatamente en `true` y se limpia la cola. Esto evitaría que se inicien más tareas que estén esperando en la cola. También podríamos agregar lógica para dejar de aceptar nuevas tareas (por ejemplo, lanzando un error en el método `pushTask()` cuando `this.stopped` sea `true`). Si bien este enfoque evitaría que la cola inicie nuevas tareas, es importante tener en cuenta que no podría detener las tareas que ya se están ejecutando o que están esperando a que se complete alguna llamada asíncrona.

#### Web spider versión 4

Ahora que tenemos nuestra cola genérica para ejecutar tareas en un flujo concurrente limitado, usémosla de inmediato para refactorizar nuestra aplicación web spider.

Vamos a utilizar una instancia de `TaskQueue` como registro de trabajo (*backlog*); cada URL que queramos rastrear debe agregarse a la cola como una tarea. La URL inicial se agregará como la primera tarea, luego también se agregará cualquier otra URL descubierta durante el proceso de rastreo. La cola gestionará toda la programación por nosotros, asegurándose de que la cantidad de tareas en curso (es decir, la cantidad de páginas que se están descargando o leyendo desde el sistema de archivos) en cualquier momento dado nunca sea mayor que el límite de concurrencia configurado para la instancia dada de `TaskQueue`.

Ya hemos definido la lógica para rastrear una URL determinada dentro de nuestra función `spider()`. Podemos considerar que esta es nuestra tarea genérica de rastreo. Para mayor claridad, es mejor cambiar el nombre de esta función a `spiderTask`:

```javascript
function spiderTask(url, maxDepth, queue, cb) { // 1
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    if (err) { // error checking the file
      return cb(err)
    }
    if (alreadyExists) {
      if (!filename.endsWith('.html')) { // ignoring non-HTML resources
        return cb()
      }
      return readFile(filename, 'utf8', (err, fileContent) => {
        if (err) { // error reading the file
          return cb(err)
        }
        spiderLinks(url, fileContent, maxDepth, queue) // 2
        return cb()
      })
    }
    // The file does not exist, download it
    download(url, filename, (err, fileContent) => {
      if (err) { // error downloading the file
        return cb(err)
      }
      // if the file is an HTML file, spider it
      if (filename.endsWith('.html')) {
        spiderLinks(url, fileContent.toString('utf8'), maxDepth, queue) // 2
        return cb()
      }
      // otherwise, stop here
      return cb()
    })
  })
}
```

Aparte de cambiar el nombre de la función, habrás notado que aplicamos algunos otros pequeños cambios:
1. La firma de la función ahora acepta un nuevo parámetro llamado `queue`. Esta es una instancia de `TaskQueue` que debemos transferir para poder agregar nuevas tareas cuando sea necesario.
2. La función responsable de agregar nuevos enlaces para rastrear es `spiderLinks()`, por lo que debemos asegurarnos de pasar la instancia de la cola cuando llamemos a esta función después de descargar una nueva página.

En un momento veremos que la función `spiderLinks()` se simplificará a una función síncrona, ya que solo necesitará agregar tareas a la cola. Ya no necesitaremos pasarle el callback; en su lugar, simplemente llamaremos a `return cb()` después de invocarla.

Ahora, echemos un vistazo a la función `spiderLinks()`. Esta función se puede simplificar significativamente ya que ya no tiene que rastrear la finalización de las tareas: la cola ahora se encarga de eso. Su ejecución se volverá efectivamente síncrona; simplemente necesita llamar a la nueva función `spider()` (que definiremos en breve) para enviar una nueva tarea a la cola por cada enlace descubierto:

```javascript
function spiderLinks(currentUrl, body, maxDepth, queue) {
  if (maxDepth === 0) {
    return
  }
  const links = getPageLinks(currentUrl, body)
  if (links.length === 0) {
    return
  }
  for (const link of links) {
    spider(link, maxDepth - 1, queue)
  }
}
```

Revisemos ahora la función `spider()`, que debe actuar como el punto de entrada para la primera URL; también se utilizará para agregar cada nueva URL descubierta a la cola:

```javascript
const spidering = new Set() // 1

export function spider(url, maxDepth, queue) {
  if (spidering.has(url)) {
    return
  }
  spidering.add(url)
  queue.pushTask(done => { // 2
    spiderTask(url, maxDepth, queue, done)
  })
}
```

Como puedes ver, esta función ahora tiene dos responsabilidades principales:
1. Gestiona el registro de las URLs ya visitadas o en curso utilizando el conjunto `spidering`.
2. Envía una nueva tarea a la cola. Una vez ejecutada, esta tarea invocará la función `spiderTask()`, iniciando efectivamente el rastreo de la URL dada.

Finalmente, podemos actualizar el script `spider-cli.js`, que nos permite invocar nuestro spider desde la línea de comandos:

```javascript
import { spider } from './spider.js'
import { TaskQueue } from './TaskQueue.js'

const url = process.argv[2] // 1
const maxDepth = Number.parseInt(process.argv[3], 10) || 1
const concurrency = Number.parseInt(process.argv[4], 10) || 2

const spiderQueue = new TaskQueue(concurrency) // 2
spiderQueue.on('error', console.error)
spiderQueue.on('empty', () => console.log('Download complete'))

spider(url, maxDepth, spiderQueue) // 3
```

Este script ahora se compone de tres partes principales:
1. Análisis de argumentos de la CLI. Observa que el script ahora acepta un tercer parámetro adicional que se puede usar para personalizar el nivel de concurrencia.
2. Se crea un objeto `TaskQueue` y se adjuntan escuchadores a los eventos `error` y `empty`. Cuando ocurre un error, simplemente queremos imprimirlo. Cuando la cola está vacía, significa que hemos terminado de rastrear el sitio web.
3. Finalmente, iniciamos el proceso de rastreo invocando la función `spider`.

Después de haber aplicado estos cambios, podemos intentar ejecutar el módulo spider nuevamente. Cuando ejecutamos el siguiente comando:

```bash
node spider-cli.js https://loige.co 1 4
```

Deberíamos notar que no habrá más de cuatro descargas activas al mismo tiempo.

Con este último ejemplo, hemos concluido nuestra exploración de los patrones basados en callbacks.

---

### Sección 7: Resumen

Al comienzo de este capítulo, mencionamos que Node.js puede ser un desafío debido a su naturaleza asíncrona, especialmente para los desarrolladores que provienen de otras plataformas. Sin embargo, como has visto, puedes hacer que las APIs asíncronas jueguen a tu favor. Las herramientas que has aprendido son flexibles y proporcionan soluciones sólidas a muchos problemas comunes, al mismo tiempo que permiten diferentes estilos de programación para adaptarse a las preferencias individuales.

También hemos continuado refactorizando y mejorando nuestro ejemplo de rastreador web a lo largo del capítulo. Al trabajar con código asíncrono, a veces puede resultar complicado encontrar el equilibrio adecuado entre simplicidad y eficacia, así que tómate tu tiempo para asimilar los conceptos tratados y experimentar con ellos.

Nuestro viaje con la programación asíncrona en Node.js no ha hecho más que empezar. En los próximos capítulos, te sumergirás en otras técnicas ampliamente utilizadas como las promesas y async/await. Una vez que estés familiarizado con todos estos enfoques, podrás elegir el mejor para tus necesidades, o incluso combinar múltiples técnicas dentro del mismo proyecto.

Antes de continuar, te recomendamos encarecidamente que pruebes los siguientes ejercicios para consolidar tu comprensión. Te ayudarán a practicar los conceptos clave y te prepararán para los capítulos venideros.

---

### Sección 8: Ejercicios

- **4.1 Concatenación de archivos (*File concatenation*):** Escribe la implementación de `concatFiles()`, una función de estilo callback que toma dos o más rutas a archivos de texto en el sistema de archivos y un archivo de destino:
  ```javascript
  function concatFiles (srcFile1, srcFile2, srcFile3, ... , dest, cb) {
    // ...
  }
  ```
  Esta función debe copiar el contenido de cada archivo fuente en el archivo de destino, respetando el orden de los archivos según lo proporciona la lista de argumentos. Por ejemplo, dados dos archivos, si el primer archivo contiene `foo` y el segundo contiene `bar`, la función debe escribir `foobar` (y no `barfoo`) en el archivo de destino. Ten en cuenta que la firma de ejemplo anterior no es sintaxis de JavaScript válida: debes encontrar una forma diferente de manejar un número arbitrario de argumentos. Por ejemplo, podrías usar la sintaxis de parámetros rest ([nodejsdp.link/rest-parameters](https://nodejsdp.link/rest-parameters)).
- **4.2 Listar archivos recursivamente (*List files recursively*):** Escribe `listNestedFiles()`, una función de estilo callback que toma como entrada la ruta a un directorio en el sistema de archivos local y que itera de forma asíncrona sobre todos los subdirectorios para devolver finalmente una lista de todos los archivos descubiertos. Así es como debería verse la firma de la función:
  ```javascript
  function listNestedFiles (dir, cb) {
    /* ... */
  }
  ```
  Puntos extra si logras evitar el callback hell. Siéntete libre de crear funciones auxiliares adicionales si es necesario.
- **4.3 Búsqueda recursiva (*Recursive find*):** Escribe `recursiveFind()`, una función de estilo callback que toma una ruta a un directorio en el sistema de archivos local y una palabra clave, según la siguiente firma:
  ```javascript
  function recursiveFind(dir, keyword, cb) {
    /* ... */
  }
  ```
  La función debe encontrar todos los archivos de texto dentro del directorio dado que contengan la palabra clave en el contenido del archivo. La lista de archivos coincidentes debe devolverse utilizando el callback cuando se complete la búsqueda. Si no se encuentra ningún archivo coincidente, se debe invocar el callback con un array vacío. Como caso de prueba de ejemplo, si tienes los archivos `foo.txt`, `bar.txt` y `baz.txt` en `myDir` y la palabra clave `'batman'` está contenida en los archivos `foo.txt` y `baz.txt`, deberías poder ejecutar el siguiente código:
  ```javascript
  recursiveFind('myDir', 'batman', console.log) // should print ['foo.txt', 'baz.txt']
  ```
  Puntos extra si haces que la búsqueda sea recursiva (que también busque archivos de texto en cualquier subdirectorio). Puntos de bonificación adicionales si logras realizar la búsqueda en diferentes archivos y subdirectorios de forma concurrente, ¡pero ten cuidado de mantener bajo control el número de tareas concurrentes!
- **4.4 Verificador de enlaces rotos (*Broken links checker*):** Escribe una función `checkBrokenLinks()` que tome una URL y un nivel de profundidad máxima y reporte cualquier enlace roto (enlaces que devuelvan un código de estado 404) que encuentre durante el proceso de rastreo. Puedes partir del código que escribimos para el web crawler y modificarlo para que, en lugar de descargar el contenido de las páginas, solo verifique el estado HTTP de cada enlace. Si el código de estado es 404, debe registrar la URL del enlace roto y continuar rastreando otros enlaces. Consejo: puedes intentar usar el método `HEAD` en lugar de `GET` al verificar los enlaces, ya que solo recupera las cabeceras, lo que puede acelerar el proceso y reducir el uso de ancho de banda.
