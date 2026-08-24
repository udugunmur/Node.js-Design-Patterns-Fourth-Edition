# Parte 1: Fundamentos de Node.js

## Capítulo 2: El sistema de módulos

En el [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1), La plataforma Node.js, presentamos brevemente la importancia de los módulos en Node.js. Discutimos cómo los módulos desempeñan un papel fundamental en la definición de algunos de los pilares de la filosofía de Node.js y su experiencia de programación. Sin embargo, ¿qué queremos decir cuando hablamos de módulos y por qué son tan importantes?

En términos generales, los módulos son los ladrillos para estructurar aplicaciones no triviales. Los módulos te permiten dividir la base de código en unidades pequeñas que pueden desarrollarse y probarse de forma independiente. Los módulos son también el mecanismo principal para hacer cumplir la ocultación de información (*information hiding*), manteniendo privadas todas las funciones y variables que no estén marcadas explícitamente para ser exportadas. Además, los módulos facilitan compartir y reutilizar código entre diferentes proyectos.

Por razones históricas, Node.js cuenta con dos sistemas de módulos diferentes: **módulos ECMAScript (módulos ES o ESM)** y **CommonJS**. Dado que los módulos ES son ahora el sistema de módulos principal en Node.js y se utilizan ampliamente en la web, nos centraremos en ellos en este capítulo. Analizaremos brevemente CommonJS, ya que todavía se encuentra en bases de código y bibliotecas más antiguas, y resulta útil entender cómo trabajar con ambos sistemas de módulos si es necesario. Al final de este capítulo, podrás tomar decisiones fundamentadas sobre cómo usar y escribir módulos de manera eficaz.

Adquirir una buena comprensión de los sistemas y patrones de módulos de Node.js es fundamental, ya que nos basaremos en este conocimiento en todos los demás capítulos de este libro.

En resumen, estos son los temas principales que trataremos a lo largo de este capítulo:
- Por qué son necesarios los módulos.
- Los diferentes sistemas de módulos disponibles en Node.js.
- El patrón de módulo revelador (*revealing module pattern*).
- Módulos ES en Node.js.
- Módulos CommonJS.
- CommonJS y la interoperabilidad con módulos ES.
- Cómo utilizar módulos en TypeScript.

Comencemos explorando por qué necesitamos módulos.

---

### Sección 1: La necesidad de los módulos

Comencemos aclarando la diferencia entre un módulo y un sistema de módulos. Un **módulo** es la pieza de software real, mientras que un **sistema de módulos** es la sintaxis y las herramientas que nos permiten definir y utilizar módulos en nuestros proyectos.

Sin importar el lenguaje de programación o la plataforma, un buen sistema de módulos debe dar respuesta a varias necesidades fundamentales de la ingeniería de software:
- **Organización del código:** Debe permitir dividir la base de código en múltiples archivos. Esto mantiene el código más organizado y fácil de entender, y permite el desarrollo y las pruebas independientes de diferentes partes.
- **Reutilización de código y gestión de dependencias:** Un buen sistema de módulos debe facilitar compartir y reutilizar funcionalidad entre proyectos, gestionando al mismo tiempo las dependencias de forma directa. Los desarrolladores deben poder construir sobre módulos existentes, incluidos los de terceros, y los usuarios deben poder importar un módulo junto con todo lo que este requiere sin fricción.
- **Encapsulación y ocultación de información:** Debe ayudar a ocultar los detalles de implementación y exponer únicamente una interfaz clara y simple. La mayoría de los sistemas de módulos permiten que ciertas partes del código permanezcan privadas mientras proporcionan funciones, clases u objetos públicos para los usuarios del módulo.

---

### Sección 2: Sistemas de módulos en JavaScript y Node.js

No todos los lenguajes de programación vienen con un sistema de módulos integrado, y JavaScript careció de esta característica durante mucho tiempo tras su creación.

Al escribir código JavaScript para el navegador, es posible dividir la base de código en múltiples archivos y luego importarlos utilizando diferentes etiquetas `<script>`. Durante muchos años, este enfoque fue suficiente para construir sitios web interactivos simples, y los desarrolladores de JavaScript lograron avanzar sin disponer de un sistema de módulos completo.

Solo cuando las aplicaciones JavaScript para el navegador se volvieron más avanzadas y frameworks como jQuery, Backbone y AngularJS conquistaron el ecosistema, la comunidad de JavaScript propuso varias iniciativas destinadas a definir un sistema de módulos que pudiera adoptarse eficazmente en proyectos JavaScript. Las más exitosas fueron la definición de módulos asíncronos (**AMD**), popularizada por RequireJS ([nodejsdp.link/requirejs](https://nodejsdp.link/requirejs)), y más tarde la definición universal de módulos (**UMD** — [nodejsdp.link/umd](https://nodejsdp.link/umd)).

Cuando se creó Node.js, fue concebido como un entorno de ejecución de servidor para JavaScript con acceso directo al sistema de archivos subyacente, por lo que surgió la oportunidad única de introducir una forma diferente de gestionar módulos. La idea no era depender de etiquetas HTML `<script>` y recursos accesibles a través de URLs. En su lugar, la elección fue depender puramente de archivos JavaScript disponibles en el sistema de archivos local. Para su sistema de módulos, Node.js propuso una implementación de la especificación **CommonJS** ([nodejsdp.link/commonjs](https://nodejsdp.link/commonjs)), diseñada para proporcionar un sistema de módulos para JavaScript en entornos fuera del navegador. Cabe señalar que CommonJS no formaba parte del estándar oficial de ECMAScript, sino que fue una iniciativa independiente para estandarizar JavaScript fuera del navegador.

CommonJS fue el sistema de módulos dominante en Node.js durante muchos años. Se volvió tan popular que la gente comenzó a utilizarlo para escribir módulos que pudieran ejecutarse incluso en aplicaciones de navegador gracias a empaquetadores de módulos (*module bundlers*) como Browserify ([nodejsdp.link/browserify](https://nodejsdp.link/browserify)) y Webpack ([nodejsdp.link/webpack](https://nodejsdp.link/webpack)).

En 2015, con el lanzamiento de ECMAScript 2015 (ES2015), hubo una propuesta oficial para definir un sistema de módulos estándar para JavaScript: los **módulos ES (ES modules)**. Los módulos ES aportan una gran innovación al ecosistema de JavaScript y, entre otras cosas, intentan cerrar la brecha entre cómo se gestionan los módulos en navegadores y en servidores.

ES2015 definió únicamente la especificación formal para los módulos ES en términos de sintaxis y semántica, pero no proporcionó detalles de implementación. A los desarrolladores de navegadores y a la comunidad de Node.js les llevó varios años desarrollar implementaciones sólidas; Node.js ofreció soporte estable para módulos ES a partir de la versión 13.2 (lanzada en 2019). Como resultado, la transición de CommonJS a módulos ES ha sido un tanto lenta y, durante los años de transición, los desarrolladores han estado utilizando diversas técnicas para publicar bibliotecas de modo dual (*dual-mode*) que puedan funcionar tanto con módulos ES como con CommonJS.

Hoy en día, los módulos ES son el sistema de módulos estándar ampliamente aceptado para JavaScript y Node.js. Aunque CommonJS sigue siendo común en bases de código heredadas y bibliotecas antiguas, la mayoría de los proyectos nuevos se escriben utilizando ESM, y se espera que el uso de CommonJS disminuya con el tiempo. Este libro utiliza módulos ES para todos los ejemplos de código, pero también examinaremos algo de código CommonJS para entender cómo pueden convivir ambos sistemas de módulos.

---

### Sección 3: El patrón de módulo revelador (*The revealing module pattern*)

Antes de sumergirnos directamente en los módulos ES, vale la pena hacer un breve desvío para explorar un patrón fundamental de JavaScript: el **patrón de módulo revelador** (*revealing module pattern*), un patrón que facilita la ocultación de información y resultará útil para construir un sistema de módulos primitivo. Estos antecedentes no solo profundizarán nuestro aprecio por sistemas de módulos completos como ES modules y CommonJS, sino que también brindarán una excelente oportunidad para ver cómo pueden implementarse estos sistemas entre bastidores. Como tal, este patrón se convertirá en una pieza de conocimiento fundamental que nos ayudará a comprender en profundidad los módulos ES.

Como hemos mencionado, los módulos son los ladrillos para estructurar aplicaciones no triviales y el mecanismo principal para hacer cumplir la ocultación de información manteniendo privadas todas las funciones y variables que no se marquen explícitamente para ser exportadas.

Un problema importante con JavaScript en el navegador es la falta de espacios de nombres (*namespacing*). Dado que cada script se ejecuta en el ámbito global, tanto el código interno de la aplicación como las dependencias de terceros pueden saturar el ámbito exponiendo sus propias funciones y variables. Esto puede resultar sumamente problemático. Por ejemplo, si una biblioteca de terceros crea una variable global llamada `utils` y otra biblioteca o el propio código de la aplicación sobrescribe o modifica accidentalmente `utils`, el código que depende de ella podría fallar de manera impredecible. Pueden surgir problemas similares si otras bibliotecas o el código de la aplicación invocan accidentalmente una función que estaba destinada solo para uso interno de otra biblioteca.

Depender del ámbito global es arriesgado, especialmente a medida que tu aplicación crece y dependes más del código escrito por otros.

Una técnica popular para resolver esta clase de problemas es el patrón de módulo revelador, y tiene este aspecto:

```javascript
const myModule = (() => {
  const privateFoo = () => {}
  const privateBar = []
  console.log('Inside:', privateFoo, privateBar)
  const exported = {
    publicFoo: () => {},
    publicBar: () => {},
  }
  return exported
})() // once the parenthesis here are parsed
// the function will be invoked
// and the returned value assigned to myModule
console.log('Outside:', myModule.privateFoo, myModule.privateBar)
console.log('Module:', myModule)
```

Este patrón aprovecha una función autoejecutable. Este tipo de función a veces también se denomina expresión de función invocada inmediatamente (**IIFE**, por sus siglas en inglés) y se utiliza para crear un ámbito privado, exportando solo las partes que están destinadas a ser públicas.

En JavaScript, las variables creadas dentro de una función no son accesibles desde el ámbito externo (fuera de la función). Las funciones pueden utilizar la sentencia `return` para propagar información selectivamente al ámbito exterior.

Este patrón esencialmente explota estas propiedades para mantener oculta la información privada y exportar únicamente una API pública.

En el código anterior, la variable `myModule` contiene únicamente la API exportada, mientras que el resto del contenido del módulo es prácticamente inaccesible desde el exterior.

La sentencia de registro imprimirá algo como esto:

```text
Inside: [Function: privateFoo] []
Outside: undefined undefined
Module: {
  publicFoo: [Function: publicFoo],
  publicBar: [Function: publicBar]
}
```

Esto demuestra varios detalles importantes:
- Las variables privadas `privateFoo` y `privateBar` son accesibles desde dentro de la función inmediatamente invocada.
- Al imprimir `myModule` no aparecen `privateFoo` ni `privateBar` entre los miembros del objeto y no tenemos forma de acceder a estos valores desde fuera de la función autoejecutable. Si intentamos acceder a `myModule.privateFoo` y `myModule.privateBar`, estos valores son `undefined`.
- Únicamente las propiedades exportadas `publicFoo` y `publicBar` son directamente accesibles desde `myModule`.

El patrón de módulo revelador se puede utilizar para crear un sistema de módulos básico, y este concepto es, de hecho, la base del sistema de módulos CommonJS. En este capítulo, el patrón de módulo revelador ayuda a demostrar cómo ciertas características intrínsecas de JavaScript pueden utilizarse para estructurar el código de forma modular e implementar la ocultación de información. Sin embargo, en la práctica, no se espera que construyas tu propio sistema de módulos desde cero; en su lugar, utilizarás módulos ES.

---

### Sección 4: Módulos ES

Los módulos ES se introdujeron como parte de la especificación ES2015 con el objetivo de dotar a JavaScript de un sistema de módulos oficial adecuado para diferentes entornos de ejecución. La especificación de módulos ES intenta conservar algunas buenas ideas de sistemas de módulos anteriores como CommonJS y AMD. La sintaxis es muy simple y compacta. Admite dependencias cíclicas y la posibilidad de cargar módulos de forma asíncrona.

Una **dependencia cíclica** en un sistema de módulos se produce cuando dos o más módulos dependen entre sí de forma circular. Por ejemplo, el Módulo A importa del Módulo B, y el Módulo B, a su vez, importa del Módulo A. Esto crea un bucle en las dependencias que puede provocar problemas como una carga incompleta del módulo, errores en tiempo de ejecución o comportamientos inesperados, ya que los módulos no pueden resolverse por completo debido a su dependencia mutua.

#### Uso de módulos ES en Node.js

Como se discutió anteriormente, cuando se introdujeron los módulos ES en Node.js, CommonJS ya había sido durante mucho tiempo el sistema de módulos predeterminado. En consecuencia, el soporte para módulos ES en Node.js tuvo que añadirse con cuidado para mantener la compatibilidad hacia atrás. Esto significa que los módulos ES no son el valor predeterminado y deben habilitarse explícitamente. Para adoptar módulos ES en un proyecto, existen pasos específicos que indican a Node.js que un archivo o módulo debe utilizar la sintaxis de módulos ES, lo que lo convierte en una característica opcional (*opt-in*) para los desarrolladores.

Node.js considerará por defecto que cada archivo `.js` está escrito utilizando la sintaxis de CommonJS. Por ejemplo, supongamos que tenemos un archivo llamado `index.js` con el siguiente contenido:

```javascript
import { someFeature } from './someModule.js'
console.log(someFeature)
```

Podríamos intentar ejecutarlo de la siguiente manera:

```bash
node index.js
```

Es posible que veas un error similar al siguiente:

```text
(node:69441) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
(Use `node --trace-warnings ...` to show where the warning was created)
index.js:1
import { someFeature } from './someModule.js'
^^^^^^

SyntaxError: Cannot use import statement outside a module
```

Este mensaje significa que nuestro archivo `index.js` no se reconoce como un módulo ES, por lo que no podemos usar la sintaxis de módulos ES. Node.js también proporciona sugerencias útiles sobre cómo solucionar esto: debemos indicar a Node.js que cargue este archivo como un módulo ES. Hay varias formas de hacerlo:
1. Dar al archivo del módulo la extensión `.mjs` (ten en cuenta que, alternativamente, puedes usar `.cjs` para forzar a que el módulo se interprete como un módulo CommonJS).
2. Añadir un campo llamado `"type"` con el valor `"module"` al `package.json` principal más cercano, como en el siguiente ejemplo:
   ```json
   {
     "name": "sample-esm-project",
     "version": "1.0.0",
     "main": "index.js",
     "type": "module"
   }
   ```
3. Usar el flag `--experimental-default-type="module"`. Este indicador establece el sistema de módulos predeterminado en módulos ES, lo que equivale a añadir `"type": "module"` en tu `package.json`. Sin embargo, deberás especificar este flag explícitamente cada vez que ejecutes `node` para ejecutar un módulo ES.
4. Usar el flag `--experimental-detect-module`. Este indicador instruye a Node.js para que analice archivos ambiguos (como aquellos con extensión `.js`) cuando el sistema de módulos no esté especificado explícitamente. Node.js intentará inferir el tipo de módulo escaneando el archivo en busca de palabras clave como `import` y `export`, que típicamente indican un módulo ES en lugar de CommonJS.

Nuestra recomendación actual es utilizar `"type": "module"` en el `package.json`, para que puedas seguir usando la extensión `.js`, que es la más ampliamente compatible en los editores de texto. Es probable que en el futuro Node.js adopte los módulos ES por defecto o que detecte automáticamente el sistema de módulos correcto para cargar tus archivos (al momento de escribir esto, Node.js 23 ha adoptado la detección de módulos por defecto, por lo que es probable que este se convierta en el comportamiento estándar en el futuro). A lo largo de este libro utilizaremos nuestro enfoque recomendado para la mayoría de los ejemplos, por lo que si estás copiando y pegando ejemplos directamente del libro, asegúrate de crear también un archivo `package.json` con la entrada `"type": "module"`.

Veamos ahora la sintaxis de los módulos ES.

#### La sintaxis de los módulos ES

En esta sección nos centraremos específicamente en la sintaxis de los módulos ES, cubriendo sus elementos principales como exportaciones con nombre y por defecto, exportaciones mixtas e identificadores de módulo. También analizaremos cómo la sintaxis de módulos ES admite tanto importaciones estáticas (cargadas al inicio de la ejecución) como importaciones dinámicas, que pueden cargarse condicional o asíncronamente.

##### Exportaciones e importaciones con nombre (*Named exports and imports*)

Los módulos ES nos permiten exportar constantes, funciones y clases desde un módulo a través de la palabra clave `export`.

En un módulo ES, todo es privado por defecto y solo las entidades exportadas son públicamente accesibles desde otros módulos.

La palabra clave `export` se puede utilizar delante de las entidades que queremos poner a disposición de los usuarios del módulo. Veamos un ejemplo:

```javascript
// logger.js
// exports a function as `log`
export function log(message) {
  console.log(message)
}
// exports a constant as `DEFAULT_LEVEL`
export const DEFAULT_LEVEL = 'info'
// exports an object as `LEVELS`
export const LEVELS = {
  error: 0,
  debug: 1,
  warn: 2,
  data: 3,
  info: 4,
  verbose: 5
}
// exports a class as `Logger`
export class Logger {
  constructor(name) {
    this.name = name
  }
  log(message) {
    console.log(`[${this.name}] ${message}`)
  }
}
```

Si queremos importar entidades de un módulo, podemos utilizar la palabra clave `import`. La sintaxis es bastante flexible y nos permite importar una o más entidades e incluso renombrar importaciones. Veamos algunos ejemplos:

```javascript
import * as loggerModule from './logger.js'
console.log(loggerModule)
```

En este ejemplo, estamos utilizando la sintaxis `*` (también llamada importación de espacio de nombres o *namespace import*) para importar todos los miembros del módulo y asignarlos a la variable local `loggerModule`. Este ejemplo generará una salida similar a esta:

```text
[Module] {
  DEFAULT_LEVEL: 'info',
  LEVELS: { error: 0, debug: 1, warn: 2, data: 3, info: 4, verbose: 5 },
  Logger: [Function: Logger],
  log: [Function: log]
}
```

Como podemos ver, todas las entidades exportadas en nuestro módulo ahora son accesibles en el espacio de nombres `loggerModule`. Por ejemplo, podríamos hacer referencia a la función `log()` mediante `loggerModule.log`.

Generalmente es mejor ser explícito y evitar importar un módulo completo, importando en su lugar únicamente las entidades específicas que se necesitan en el contexto actual:

```javascript
import { log } from './logger.js'
log('Hello World')
```

Si queremos importar más de una entidad, así es como lo haríamos:

```javascript
import { Logger, log } from './logger.js'
log('Hello World')
const logger = new Logger('DEFAULT')
logger.log('Hello world')
```

Cuando utilizamos este tipo de sentencia de importación, las entidades se importan en el ámbito actual, por lo que existe el riesgo de un conflicto de nombres (*name clash*). El siguiente código, por ejemplo, no funcionaría:

```javascript
import { log } from './logger.js'
const log = console.log
```

Si intentamos ejecutar el fragmento anterior, el intérprete falla con el siguiente error:

```text
SyntaxError: Identifier 'log' has already been declared
```

En situaciones como esta, podemos resolver el conflicto renombrando la entidad importada con la palabra clave `as`:

```javascript
import { log as log2 } from './logger.js'
const log = console.log
log('message from log')
log2('message from log2')
```

Este enfoque puede resultar particularmente útil cuando el conflicto se genera al importar dos entidades con el mismo nombre desde diferentes módulos y, por lo tanto, cambiar los nombres originales escapa al control del consumidor. También puede ser útil cuando se desea utilizar un nombre más corto o más limpio para las entidades que se están importando.

Ten en cuenta que si intentas importar un miembro de un módulo que no está exportado, se producirá un error de sintaxis en tiempo de ejecución. Por ejemplo, podríamos intentar ejecutar el siguiente código:

```javascript
import { something } from './logger.js'
```

Obtendremos entonces el siguiente error:

```text
SyntaxError: The requested module './logger.js' does not provide an export named 'something'
```

##### Exportaciones e importaciones por defecto (*Default exports and imports*)

En ocasiones, es posible que desees exportar una única entidad sin nombre. Con los módulos ES podemos hacer eso con una exportación por defecto (*default export*). Una exportación por defecto hace uso de las palabras clave `export default` y tiene este aspecto:

```javascript
// logger.js
export default class Logger {
  constructor(name) {
    this.name = name
  }
  log(message) {
    console.log(`[${this.name}] ${message}`)
  }
}
```

En este caso, se ignora el nombre `Logger` y la entidad exportada se registra bajo el nombre `default`. Este nombre exportado se puede importar de la siguiente manera:

```javascript
// main.js
import MyLogger from './logger.js'
const logger = new MyLogger('info')
logger.log('Hello World')
```

La diferencia con las importaciones con nombre de módulos ES es que aquí, dado que la exportación por defecto se considera anónima, podemos importarla y al mismo tiempo asignarle un nombre local de nuestra elección. En este ejemplo podemos reemplazar `MyLogger` por cualquier otra cosa que tenga sentido en nuestro contexto. Ten en cuenta también que no tenemos que envolver el nombre de importación entre llaves ni usar la palabra clave `as` al renombrar.

Internamente, una exportación por defecto equivale a una exportación con nombre donde el nombre es `default`. Podemos verificar fácilmente esta afirmación ejecutando el siguiente fragmento de código:

```javascript
// showDefault.js
import * as loggerModule from './logger.js'
console.log(loggerModule.default)
```

Al ejecutarse, el código anterior imprimirá algo como esto:

```text
[class Logger]
```

Sin embargo, una cosa que no podemos hacer es importar la entidad por defecto explícitamente entre llaves. De hecho, algo como lo siguiente fallará:

```javascript
import { default } from './logger.js'
```

La ejecución fallará con un error `SyntaxError: Unexpected reserved word`. Esto sucede porque la palabra clave `default` no puede utilizarse como nombre de variable. Es válida como atributo de un objeto, por lo que en el ejemplo anterior está bien usar `loggerModule.default`, pero no podemos tener una variable llamada `default` directamente en el ámbito.

##### Exportaciones mixtas (*Mixed exports*)

Es posible mezclar exportaciones con nombre y una exportación por defecto dentro de un mismo módulo ES. Veamos un ejemplo:

```javascript
// logger.js
export default function log(message) {
  console.log(message)
}
export function info(message) {
  log(`info: ${message}`)
}
```

El código anterior exporta la función `log()` como exportación por defecto y una exportación con nombre para una función llamada `info()`.

Ten en cuenta que `info()` puede hacer referencia internamente a `log()`. No sería posible reemplazar la llamada a `log()` por `default()` para hacer eso, ya que constituiría un error de sintaxis (`unexpected token default`).

Si queremos importar tanto la exportación por defecto como una o más exportaciones con nombre, podemos hacerlo utilizando el siguiente formato:

```javascript
import mylog, { info } from './logger.js'
```

En el ejemplo anterior, importamos la exportación por defecto de `logger.js` (como `mylog`) y la exportación con nombre `info`.

Ten en cuenta que, con esta sintaxis, el miembro por defecto siempre debe ir primero. Escribir el siguiente código daría lugar a un error de sintaxis:

```javascript
import { info }, mylog from './logger.js'
```

Analicemos ahora algunos detalles y diferencias clave entre la exportación por defecto y las exportaciones con nombre:
- La exportación por defecto es un mecanismo conveniente para comunicar cuál es la funcionalidad única más importante de un módulo. Asimismo, desde la perspectiva del usuario, puede ser más sencillo importar la pieza obvia de funcionalidad sin tener que conocer el nombre exacto de la vinculación (*binding*).
- Las exportaciones con nombre son explícitas. Tener nombres predeterminados permite a los IDEs asistir al desarrollador con importaciones automáticas, autocompletado y herramientas de refactorización. Por ejemplo, si escribimos `writeFileSync`, el editor podría añadir automáticamente `import { writeFileSync } from 'node:fs'` al principio del archivo actual. Las exportaciones por defecto, por el contrario, complican todas estas operaciones, ya que una funcionalidad dada podría recibir nombres diferentes en archivos distintos, por lo que es más difícil inferir qué módulo proporciona una funcionalidad determinada basándose únicamente en un nombre dado.
- En algunas circunstancias, las exportaciones por defecto pueden dificultar la aplicación de eliminación de código muerto (*tree shaking*). Por ejemplo, un módulo podría proporcionar únicamente una exportación por defecto que sea un objeto donde toda la funcionalidad se expone como propiedades de dicho objeto. Al importar este objeto por defecto, la mayoría de los empaquetadores de módulos considerarán que se está utilizando el objeto completo y no podrán eliminar el código no utilizado de la funcionalidad exportada.

Los inconvenientes de utilizar exportaciones por defecto en JavaScript superan a los beneficios, lo que ha llevado a la comunidad a inclinarse generalmente por evitarlas. Algunos linters de código incluyen ahora reglas para detectar y advertir contra el uso de exportaciones por defecto.

Biome ([nodejsdp.link/biome](https://nodejsdp.link/biome)), un linter y formateador de JavaScript, incluye una regla de linting para deshabilitar las exportaciones por defecto de forma predeterminada. La página de documentación de esta regla ofrece detalles adicionales sobre por qué esto se considera una buena práctica: [nodejsdp.link/no-default-export](https://nodejsdp.link/no-default-export).

Esta no es una regla estricta y existen notables excepciones a esta sugerencia. Por ejemplo, todos los módulos principales del núcleo de Node.js tienen tanto una exportación por defecto como varias exportaciones con nombre. Además, React ([nodejsdp.link/react](https://nodejsdp.link/react)) utiliza exportaciones mixtas.

Considera cuidadosamente cuál es el mejor enfoque para tu módulo específico y qué experiencia de desarrollo deseas brindar a los usuarios de tu módulo.

##### Identificadores de módulos (*Module identifiers*)

Los identificadores de módulo (también llamados especificadores de módulo o *module specifiers*) son los diferentes tipos de valores que podemos utilizar en nuestras sentencias `import` para especificar la ubicación del módulo que queremos cargar.

Hasta ahora solo hemos visto rutas relativas, pero existen otras posibilidades y algunos matices a tener en cuenta. Enumeremos todas las posibilidades:
- **Especificadores relativos:** como `./logger.js` o `../logger.js`, se utilizan para hacer referencia a una ruta relativa a la ubicación del archivo importador.
- **Especificadores absolutos:** como `file:///opt/nodejs/config.js` o `/opt/nodejs/config.js`, hacen referencia directa y explícita a una ruta completa.
- **Especificadores simples (*bare specifiers*):** son identificadores como `fastify` o `http`. Representan módulos disponibles en la carpeta `node_modules` y generalmente instalados mediante un gestor de paquetes (como npm) o disponibles como módulos centrales de Node.js.
- Para evitar la ambigüedad entre los módulos del núcleo de Node.js y los módulos de terceros, Node.js admite un prefijo opcional `node:` (por ejemplo, `node:http`). Esta es la forma recomendada de importar módulos del núcleo de Node.js.
- **Especificadores de importación profunda (*deep import specifiers*):** como `fastify/lib/logger.js`, hacen referencia a una ruta dentro de un paquete en `node_modules` (`fastify`, en este caso).
- En entornos de navegador es posible importar módulos directamente especificando la URL del módulo, por ejemplo, `https://unpkg.com/lodash`. Esta función no es compatible con Node.js.

##### Importaciones estáticas y dinámicas (*Static and dynamic imports*)

Los módulos ES son estáticos, lo que significa que las importaciones se describen en el nivel superior de cada módulo y fuera de cualquier sentencia de control de flujo. Además, el nombre de los módulos importados no se puede generar dinámicamente en tiempo de ejecución utilizando expresiones; solo se permiten cadenas constantes.

Por ejemplo, el siguiente código no sería válido al utilizar módulos ES:

```javascript
if (condition) {
  import module1 from 'module1'
} else {
  import module2 from 'module2'
}
```

A primera vista, estas características de los módulos ES pueden parecer limitaciones innecesarias, pero disponer de importaciones estáticas abre una serie de escenarios interesantes. Por ejemplo, las importaciones estáticas permiten el análisis estático del árbol de dependencias, lo que posibilita optimizaciones como la eliminación de código muerto (*tree shaking*).

¿Significa esto que con los módulos ES no podemos crear identificadores de módulos en tiempo de ejecución ni importar módulos condicionalmente? Por ejemplo, ¿qué sucede si queremos cargar un módulo específico —quizás uno pesado— solo cuando el usuario accede a la función que lo necesita? ¿O qué pasa si necesitamos importar un módulo de traducción particular según el idioma del usuario, o una versión de un módulo que dependa del sistema operativo del usuario?

Afortunadamente, los módulos ES incluyen una característica llamada **importaciones dinámicas** (o importaciones asíncronas) que da respuesta a estos escenarios.

Las importaciones asíncronas se pueden realizar en tiempo de ejecución utilizando el operador especial `import()`.

El operador `import()` es sintácticamente equivalente a una función que toma un identificador de módulo como argumento. Devuelve una promesa que se resuelve en un objeto de módulo.

Aprenderemos más sobre promesas en el [Capítulo 5](https://subscription.packtpub.com/book/web-development/9781803238944/5), Patrones de control de flujo asíncrono con Promesas y Async/Await, así que no te preocupes demasiado por comprender todos los matices de la sintaxis específica de las promesas por ahora.

El identificador del módulo puede ser cualquier identificador de módulo admitido por las importaciones estáticas, tal como se discutió en la sección anterior. Veamos ahora cómo usar importaciones dinámicas con un ejemplo sencillo.

Queremos crear una aplicación de línea de comandos que pueda imprimir "Hola Mundo" en diferentes idiomas. En el futuro, probablemente querremos admitir muchas más frases e idiomas, por lo que tiene sentido tener un archivo con las traducciones de todas las cadenas visibles para el usuario para cada idioma admitido.

Creemos algunos módulos de ejemplo para algunos de los idiomas que queremos admitir:

```javascript
// strings-el.js
export const HELLO = 'Γεια σου κόσμε'

// strings-en.js
export const HELLO = 'Hello World'

// strings-es.js
export const HELLO = 'Hola mundo'
```

Ahora creemos el script principal que toma un código de idioma de la línea de comandos e imprime "Hola Mundo" en el idioma seleccionado:

```javascript
// main.js
const SUPPORTED_LANGUAGES = ['el', 'en', 'es', 'it', 'pl'] // 1
const selectedLanguage = process.argv[2] // 2
if (!selectedLanguage) { // 3
  console.error(
    `Please specify a language
Usage: node ${process.argv[1]} <language_code>
Supported languages: ${SUPPORTED_LANGUAGES.join(', ')}`
  )
  process.exit(1)
}
if (!SUPPORTED_LANGUAGES.includes(selectedLanguage)) { // 4
  console.error('The specified language is not supported')
  process.exit(1)
}
const translationModule = `./strings-${selectedLanguage}.js` // 5
const strings = await import(translationModule) // 6
console.log(strings.HELLO) // 7
```

La primera parte del script es bastante simple. Lo que hacemos allí es lo siguiente:
1. Definir una lista de idiomas admitidos.
2. Leer el idioma seleccionado a partir del primer argumento pasado en la línea de comandos.
3. Validar que el usuario haya pasado el argumento y, si no es así, proporcionar un mensaje de error útil.
4. Gestionar el caso en el que el idioma seleccionado no esté admitido.

La segunda parte del código es donde realmente utilizamos importaciones dinámicas:
5. En primer lugar, construimos dinámicamente el nombre del módulo que queremos importar en función del idioma seleccionado. Ten en cuenta que el nombre del módulo debe ser una ruta relativa al archivo del módulo actual; por eso anteponemos `./` al nombre del archivo.
6. Utilizamos el operador `import()` para activar la importación dinámica del módulo. Esta importación se produce de forma asíncrona, devolviendo una promesa. Usamos `await` para esperar a que la promesa se resuelva y almacenamos el valor resuelto en la variable `strings`.
7. Ahora podemos acceder a `strings.HELLO` e imprimir su valor en la consola.

Podemos ejecutar este script de la siguiente manera:

```bash
node main.js it
```

Deberíamos ver `Ciao mondo` impreso en nuestra consola.

#### El algoritmo de resolución de módulos (*The module resolution algorithm*)

El término "infierno de dependencias" (*dependency hell*) describe un escenario en el que dos o más dependencias de un programa dependen de una biblioteca compartida pero requieren versiones diferentes e incompatibles de la misma. Este es un problema común en muchos lenguajes de programación y a menudo resulta difícil de resolver.

Node.js aborda este problema con elegancia cargando diferentes versiones de un módulo en función del lugar desde donde se carga el módulo. Esta capacidad se debe en gran medida a cómo los gestores de paquetes de Node.js (como npm o pnpm) organizan las dependencias de una aplicación y al algoritmo de resolución que asigna un especificador de módulo a un archivo en el sistema de archivos. Por ejemplo, este algoritmo traduce un identificador como `'fastify/lib/logger.js'` a una URL que representa un archivo que se puede cargar desde el sistema de archivos, como `file:///Users/luciano/projects/example_web_server/node_modules/fastify/lib/logger.js`, permitiendo que el archivo se cargue correctamente.

Node.js implementa la función de módulos ES `import.meta.resolve()`, que nos permite ver cómo se resuelve un especificador de módulo dado en el contexto actual. Intentemos tener una visión general de alto nivel de cómo funciona el algoritmo de resolución de módulos. Podemos dividir el algoritmo en las siguientes tres ramas principales:
- **Módulos de archivo (*File modules*):** Si el especificador del módulo comienza con `/`, se considera una ruta absoluta al archivo del módulo en el sistema de archivos. La ruta se normaliza y se convierte en una URL con el prefijo `file://`. Si comienza con `./` o `../`, el especificador del módulo se considera una ruta relativa, que se calcula a partir del directorio del módulo que lo requiere.
- **Módulos del núcleo de Node.js (*Node.js core modules*):** Si el especificador del módulo no tiene como prefijo `/`, `./` o `../`, el algoritmo primero intentará buscar dentro de los módulos centrales de Node.js. Si el especificador del módulo coincide con uno de los módulos del núcleo de Node.js, se le antepone el prefijo `node:` y se devuelve. Si el especificador de módulo proporcionado ya incluye el prefijo `node:`, se devuelve tal cual.
- **Módulos de paquetes (*Package modules*):** Si no se encuentra ningún módulo central que coincida con el especificador de módulo dado, la búsqueda continúa buscando un módulo coincidente en el primer directorio `node_modules` que se encuentre navegando hacia arriba en la estructura de directorios a partir del módulo importador. El algoritmo continúa buscando una coincidencia explorando el siguiente directorio `node_modules` hacia arriba en el árbol de directorios hasta alcanzar la raíz del sistema de archivos.

Se admiten algunos tipos adicionales de especificadores, como el prefijo `data:`. La documentación formal y completa del algoritmo de resolución se puede encontrar en [nodejsdp.link/resolve_esm](https://nodejsdp.link/resolve_esm).

Podemos ver este algoritmo en acción con el siguiente código:

```javascript
// relative import
console.log(import.meta.resolve('./utils/example.js'))
// -> file://<project_path>/utils/example.js

// Node.js core module import
console.log(import.meta.resolve('assert'))
// -> node:assert
console.log(import.meta.resolve('node:assert'))
// -> node:assert

// Third-party library import (with fastify@5.1.0 installed)
console.log(import.meta.resolve('fastify/lib/logger.js'))
// -> file://<project_path>/node_modules/fastify/lib/logger.js
```

El directorio `node_modules` es en realidad donde los gestores de paquetes instalan las dependencias de cada paquete. Esto significa que, según el algoritmo que acabamos de describir, cada paquete puede tener sus propias dependencias privadas. Por ejemplo, considera la siguiente estructura de directorios:

```text
myApp
├── foo.js
└── node_modules
    ├── depA
    │   └── index.js
    ├── depB
    │   ├── bar.js
    │   └── node_modules
    │       └── depA
    │           └── index.js
    └── depC
        ├── foobar.js
        └── node_modules
            └── depA
                └── index.js
```

En el ejemplo anterior, `myApp`, `depB` y `depC` dependen de `depA`. ¡Sin embargo, todos tienen su propia versión privada de la dependencia! Siguiendo las reglas del algoritmo de resolución, importar `depA` cargará un archivo diferente según el módulo que lo requiera:
- Importar `depA` desde `/myApp/foo.js` cargará `/myApp/node_modules/depA/index.js`.
- Importar `depA` desde `/myApp/node_modules/depB/bar.js` cargará `/myApp/node_modules/depB/node_modules/depA/index.js`.
- Importar `depA` desde `/myApp/node_modules/depC/foobar.js` cargará `/myApp/node_modules/depC/node_modules/depA/index.js`.

El algoritmo de resolución es la pieza central detrás de la solidez de la gestión de dependencias de Node.js, y hace posible tener cientos o incluso miles de paquetes en una aplicación sin que ocurran colisiones o problemas de compatibilidad de versiones entre los paquetes instalados.

#### Carga de módulos en profundidad (*Module loading in depth*)

Para entender cómo funcionan los módulos ES y cómo pueden gestionar eficazmente las dependencias e incluso las dependencias circulares, debemos profundizar un poco más en cómo se analiza y evalúa el código JavaScript cuando se utilizan módulos ES.

En esta sección aprenderemos cómo se cargan los módulos ES, presentaremos la idea de los enlaces dinámicos de solo lectura (*read-only live bindings*) y, finalmente, analizaremos un ejemplo con dependencias circulares.

##### Fases de carga (*Loading phases*)

Node.js utiliza grafos de dependencias para cargar módulos en el orden correcto. El intérprete construye este grafo compuesto por todos los módulos necesarios. Al respetar las dependencias, este grafo evita errores y garantiza que cada módulo esté disponible cuando se necesite.

**Figura 2.1:** Ejemplo de grafo de dependencias para una aplicación de servidor web que utiliza Fastify.

En la Figura 2.1 vemos un ejemplo de grafo de dependencias. Cada nodo en el grafo representa un módulo y las flechas indican cuándo un módulo depende de otro. El módulo principal, `server.js` (también llamado punto de entrada o *entry point*), depende de `fastify`. Fastify a su vez depende de varios módulos: `node:http`, `node:diagnostic_channel` y un módulo interno (`fastify/lib/server.js`). Este módulo interno, a su vez, depende de `node:os` y `node:dns`. Ten en cuenta que esta representación es solo parcial con fines ilustrativos; en un proyecto real basado en Fastify, probablemente habrá muchas más dependencias.

En términos generales, un grafo de dependencias se puede definir como un grafo dirigido ([nodejsdp.link/directed-graph](https://nodejsdp.link/directed-graph)) que representa las dependencias de un grupo de objetos. En el contexto de esta sección, cuando nos referimos a un grafo de dependencias, queremos indicar la relación de dependencia entre módulos ES. Como veremos, el uso de un grafo de dependencias nos permite determinar el orden en que se deben cargar todos los módulos necesarios en un proyecto determinado.

Básicamente, el intérprete necesita el grafo de dependencias para descubrir cómo dependen los módulos entre sí y en qué orden debe ejecutarse el código. Cuando se inicia el intérprete de `node`, se le pasa código para ejecutar, generalmente en forma de archivo JavaScript. Este archivo es el punto de partida para la resolución de dependencias y se denomina **punto de entrada** (*entry point*). Desde el punto de entrada, el intérprete encontrará y seguirá todas las sentencias de importación de forma recursiva en un recorrido en profundidad (*depth-first*), hasta que todo el código necesario sea explorado y luego evaluado.

Este proceso ocurre en tres fases separadas:
1. **Fase 1 — Construcción o Análisis (*Construction / Parsing*):** El intérprete identifica todas las importaciones y carga recursivamente el contenido de cada módulo desde sus respectivos archivos.
2. **Fase 2 — Instanciación (*Instantiation*):** Para cada entidad exportada en cada módulo, el intérprete crea una referencia con nombre en la memoria, pero aún no le asigna un valor. Se crean referencias para todas las sentencias de importación y exportación para rastrear las relaciones de dependencia entre ellas (*linking*). Durante esta fase no se ejecuta código JavaScript.
3. **Fase 3 — Evaluación (*Evaluation*):** El intérprete de Node.js ejecuta el código para que todas las entidades instanciadas previamente obtengan un valor real. Ahora es posible ejecutar el código a partir del punto de entrada porque todos los espacios en blanco se han completado.

Podríamos decir que la Fase 1 consiste en encontrar todos los puntos, la Fase 2 conecta esos puntos creando caminos y, finalmente, la Fase 3 recorre los caminos en el orden correcto.

Dado que estas tres fases son independientes, no se puede ejecutar ningún código hasta que el grafo de dependencias esté completamente construido. En consecuencia, las importaciones y exportaciones de módulos deben ser estáticas. Profundizaremos en este proceso cuando analicemos cómo los módulos ES gestionan las dependencias circulares, así que no te preocupes si los detalles aún no están del todo claros.

Este proceso difiere significativamente de cómo CommonJS gestiona las dependencias. CommonJS es dinámico; cuando se requiere un módulo, su contenido se carga y se ejecuta inmediatamente. Esta flexibilidad permite utilizar `require` dentro de sentencias `if` o bucles, y los identificadores de módulo se pueden crear a partir de variables. Sin embargo, este enfoque dinámico hace que para CommonJS sea difícil manejar dependencias cíclicas de manera eficaz, un desafío que, como veremos, los módulos ES gestionan de manera mucho más eficiente.

##### Enlaces dinámicos de solo lectura (*Read-only live bindings*)

Otra característica fundamental de los módulos ES, que resulta clave para gestionar dependencias cíclicas, es la idea de que los módulos importados son efectivamente **enlaces dinámicos de solo lectura** (*read-only live bindings*) a sus valores exportados.

Aclaremos qué significa esto con un ejemplo sencillo:

```javascript
// counter.js
export let count = 0
export function increment() {
  count++
}
```

Este módulo exporta dos valores: un simple contador entero llamado `count` y una función `increment` que incrementa el contador en uno.

Escribamos ahora código que utilice este módulo:

```javascript
// main.js
import { count, increment } from './counter.js'
console.log(count) // prints 0
increment()
console.log(count) // prints 1
count++ // TypeError: Assignment to constant variable!
```

Lo que podemos ver en este código es que podemos leer el valor de `count` en cualquier momento y cambiarlo usando la función `increment()`, pero tan pronto como intentamos mutar la variable `count` directamente, obtenemos un error como si estuviéramos intentando mutar una vinculación `const`.

Esto demuestra que cuando se importa una entidad en el ámbito, la vinculación a su valor original no se puede cambiar (vinculación de solo lectura) a menos que el valor vinculado cambie dentro del ámbito del propio módulo original (vinculación dinámica), lo cual queda fuera del control directo del código consumidor.

Este enfoque es fundamentalmente diferente de CommonJS. De hecho, en CommonJS, todo el objeto `exports` se copia (copia superficial o *shallow copy*) cuando se requiere desde un módulo. Esto significa que si el valor de variables primitivas como números o cadenas cambia posteriormente, el módulo que las requirió no podrá ver esos cambios.

##### Dependencias circulares (*Circular dependencies*)

Muchos consideran que las dependencias circulares son un problema de diseño intrínseco, pero es algo que realmente puede ocurrir en un proyecto real, por lo que resulta útil saber cómo funciona.

Repasemos juntos un ejemplo para ver cómo se comportan los módulos ES cuando manejan dependencias circulares. Supongamos que tenemos el escenario representado en la Figura 2.2:

**Figura 2.2:** Ejemplo de dependencia circular.

Un módulo llamado `main.js` importa `a.js` y `b.js`. A su vez, `a.js` importa `b.js`. ¡Sin embargo, `b.js` también depende de `a.js`! Es obvio que aquí tenemos una dependencia circular, ya que el módulo `a.js` requiere el módulo `b.js` y el módulo `b.js` requiere el módulo `a.js`.

Implementemos primero los módulos `a.js` y `b.js`:

```javascript
// a.js
import * as bModule from './b.js'
export let loaded = false
export const b = bModule
loaded = true

// b.js
import * as aModule from './a.js'
export let loaded = false
export const a = aModule
loaded = true
```

Observa que tanto `a.js` como `b.js` exportan una vinculación llamada `loaded`, que podemos usar para observar qué sucede durante la carga de cada módulo. El valor se establece inicialmente en `false`, y cuando el módulo se ejecuta, se actualiza inmediatamente a `true`. También es importante tener en cuenta que `a.js` importa `b.js`, y `b.js` importa `a.js` a su vez, creando la dependencia circular.

Veamos ahora cómo importar esos dos módulos en nuestro archivo `main.js` (el punto de entrada):

```javascript
// main.js
import * as a from './a.js'
import * as b from './b.js'
console.log('a ->', a)
console.log('b ->', b)
```

Cuando ejecutemos `main.js`, veremos la siguiente salida:

```text
a -> <ref *1> [Module: null prototype] {
  b: [Module: null prototype] { a: [Circular *1], loaded: true },
  loaded: true
}
b -> <ref *1> [Module: null prototype] {
  a: [Module: null prototype] { b: [Circular *1], loaded: true },
  loaded: true
}
```

La parte interesante aquí es que `a.js` y `b.js` tienen una visión completa el uno del otro. Podemos observar que `loaded` está establecido en `true` en todas las referencias tanto a `a` como a `b`. Esto tiene aún más sentido cuando nos damos cuenta de que `b` dentro de `a`, y `a` dentro de `b`, son referencias reales a los módulos respectivos, no copias. No hay duplicación de datos durante la importación y ejecución de los módulos. Si intercambiáramos el orden de las importaciones en `main.js`, la salida seguiría siendo la misma.

Si implementáramos este mismo ejemplo usando CommonJS, el resultado sería significativamente diferente: los valores internos de `loaded` serían `false`. Además, si intercambiáramos el orden de las importaciones en CommonJS, veríamos una salida diferente. Estas diferencias surgen de la naturaleza dinámica de CommonJS y de la copia superficial que se produce cuando se importan entidades.

Para comprender con claridad cómo los módulos ES hacen posible admitir dependencias circulares, vale la pena dedicar más tiempo a observar lo que sucede en las tres fases de la resolución de módulos (análisis, instanciación y evaluación) para este ejemplo específico.

###### Fase 1 — Análisis (*Parsing*)

Durante la fase de análisis, el código se explora a partir del punto de entrada (`main.js`). El intérprete busca únicamente sentencias `import` para encontrar todos los módulos necesarios y cargar el código fuente desde los archivos de los módulos. El grafo de dependencias se explora en profundidad (*depth-first*) y cada módulo se visita solo una vez. De esta manera, el intérprete construye una vista de las dependencias con una estructura en árbol, como se muestra en la Figura 2.3:

**Figura 2.3:** Análisis de dependencias cíclicas con módulos ES.

Dado el ejemplo de la Figura 2.3, analicemos los distintos pasos de la fase de análisis:
1. Desde `main.js`, la primera importación encontrada nos lleva directamente a `a.js`.
2. En `a.js`, encontramos una importación que apunta a `b.js`.
3. En `b.js`, también tenemos una importación de vuelta a `a.js` (nuestro ciclo), pero como `a.js` ya ha sido visitado, esta ruta no se explora nuevamente.
4. En este punto, la exploración comienza a retroceder: `b.js` no tiene otras importaciones, por lo que volvemos a `a.js`; `a.js` no tiene más sentencias de importación, por lo que volvemos a `main.js`. Aquí encontramos otra importación que apunta a `b.js`, pero una vez más, este módulo ya ha sido explorado, por lo que esta ruta se ignora.

Para clarificar el enfoque en profundidad (*depth-first*), imagina que `main.js` también importa un módulo llamado `c.js`, listado después de `a.js`. Con esta configuración, `c.js` solo se analizaría una vez que `a.js` y `b.js` estuvieran completamente procesados, momento en el que el proceso de análisis regresa a `main.js` para gestionar las importaciones restantes. La idea general detrás de un recorrido en profundidad es explorar cada rama lo más profundamente posible antes de retroceder al nodo padre para explorar otras rutas. Si este concepto aún no está claro, considera revisar este enlace sobre búsqueda en profundidad (DFS) para obtener más ejemplos e ilustraciones: [nodejsdp.link/dfs](https://nodejsdp.link/dfs).

Llegados a este punto, nuestro recorrido en profundidad del grafo de dependencias se ha completado y tenemos una vista lineal de los módulos, como se muestra en la Figura 2.4:

**Figura 2.4:** Vista lineal del grafo de módulos donde se han eliminado los ciclos.

Esta vista particular es bastante simple. En escenarios más realistas con muchos más módulos, la vista se parecerá más a una estructura de árbol.

###### Fase 2 — Instanciación (*Instantiation*)

En la fase de instanciación, el intérprete recorre la vista en árbol obtenida en la fase anterior de abajo hacia arriba. Para cada módulo, el intérprete buscará primero todas las propiedades exportadas y construirá un mapa de los nombres exportados en la memoria:

**Figura 2.5:** Representación visual de la fase de instanciación.

La Figura 2.5 describe el orden en el que se instancia cada módulo:
1. El intérprete comienza en `b.js` y descubre que el módulo exporta `loaded` y `a`.
2. Luego, el intérprete se traslada a `a.js`, que exporta `loaded` y `b`.
3. Finalmente, se desplaza a `main.js`, que no exporta ninguna funcionalidad.

Ten en cuenta que, en esta fase, el mapa de exportaciones solo realiza un seguimiento de los nombres exportados; sus valores asociados se consideran no inicializados por el momento.

Después de esta secuencia de pasos, el intérprete realizará otra pasada para vincular los nombres exportados con los módulos que los importan, como se muestra en la Figura 2.6:

**Figura 2.6:** Vinculación de exportaciones con importaciones entre módulos.

Podemos describir lo que vemos en la Figura 2.6 a través de los siguientes pasos:
1. El módulo `b.js` se vinculará a las exportaciones de `a.js`, refiriéndose a ellas como `aModule`.
2. A su vez, `a.js` se vinculará a todas las exportaciones de `b.js`, refiriéndose a ellas como `bModule`.
3. Finalmente, `main.js` importará todas las exportaciones de `b.js`, refiriéndose a ellas como `b`; del mismo modo, importará todo de `a.js`, refiriéndose a ello como `a`.

Nuevamente, es importante señalar que todos los valores siguen sin inicializarse. En esta fase solo estamos vinculando referencias a valores que estarán disponibles al final de la siguiente fase.

###### Fase 3 — Evaluación (*Evaluation*)

El último paso es la fase de evaluación. En esta fase, todo el código de cada archivo se ejecuta finalmente. El orden de ejecución es nuevamente de abajo hacia arriba, respetando el recorrido post-orden en profundidad de nuestro grafo de dependencias original. Con este enfoque, `b.js` se ejecuta primero, luego `a.js`, y `main.js` es el último archivo en ejecutarse. De esta manera, podemos estar seguros de que todos los valores exportados se han inicializado antes de comenzar a ejecutar nuestra lógica de negocio principal:

**Figura 2.7:** Representación visual de la fase de evaluación.

Siguiendo el diagrama de la Figura 2.7, esto es lo que sucede:
1. La ejecución comienza en `b.js` y la primera línea que se evalúa inicializa la exportación `loaded` en `false` para el módulo.
2. Del mismo modo, la propiedad exportada `a` se evalúa aquí. Esta vez, se evaluará como una referencia al objeto de módulo que representa al módulo `a.js`.
3. El valor de la propiedad `loaded` cambia a `true`. En este punto, hemos evaluado completamente el estado de las exportaciones para el módulo `b.js`.
4. Ahora la ejecución pasa a `a.js`. Nuevamente, comenzamos estableciendo `loaded` en `false`.
5. En este punto, la exportación `b` se evalúa como una referencia al módulo `b.js`.
6. Finalmente, la propiedad `loaded` se cambia a `true`. Ahora hemos evaluado finalmente todas las exportaciones para `a.js` también.

Tras todos estos pasos, el código en `main.js` puede ejecutarse y, en este punto, todas las propiedades exportadas están completamente evaluadas. Dado que los módulos importados se rastrean como referencias, podemos estar seguros de que cada módulo tiene una imagen actualizada de los demás módulos, incluso en presencia de dependencias circulares.

#### Módulos que modifican otros módulos (*Monkey patching*)

En ocasiones encontramos módulos que no proporcionan ninguna exportación. Si bien esto puede parecer inusual, es importante reconocer que un módulo aún puede ofrecer funcionalidad modificando el ámbito global o los objetos dentro de él, incluidos otros módulos. Aunque esta práctica generalmente se desaconseja, puede ser útil y segura en situaciones específicas, como en pruebas, y se emplea ocasionalmente en proyectos reales.

Esta técnica, en la que un módulo modifica otros módulos u objetos en el ámbito global, se conoce como **monkey patching**. Monkey patching se refiere a la práctica de alterar objetos existentes en tiempo de ejecución para cambiar o extender su comportamiento, o para aplicar soluciones temporales.

Consideremos un ejemplo de monkey patching. Supongamos que tenemos un módulo que proporciona una implementación básica de registro en consola (*logger*). Podríamos entonces usar otro módulo, tal vez mantenido por un tercero, para mejorar el logger original añadiendo códigos de color ASCII. Esta modificación colorearía la salida del logger en la terminal según el nivel de registro: mensajes informativos en verde, advertencias en amarillo y errores en rojo. Este módulo colorizador es opcional, lo que permite a los usuarios habilitar la salida coloreada solo si deciden importarlo:

```javascript
// logger.js
export const logger = {
  info(message) {
    console.log(`[INFO]\t${message}`)
  },
  error(message) {
    console.log(`[ERROR]\t${message}`)
  },
  warn(message) {
    console.log(`[WARN]\t${message}`)
  },
  debug(message) {
    console.log(`[DEBUG]\t${message}`)
  },
}
```

Esta es una implementación básica de un logger que exporta un objeto global `logger` con métodos como `info()`, `error()`, `warn()` y otros. Cada método imprime un mensaje con una etiqueta de prefijo que indica el nivel de registro específico. Este diseño ayuda a los usuarios a buscar y filtrar fácilmente los registros producidos por la aplicación.

Ahora implementemos el módulo `colorizeLogger.js` que mejora este logger añadiendo soporte para colores ASCII:

```javascript
// colorizeLogger.js
import { logger } from './logger.js'

const RED = '\x1b[31m'
const YELLOW = '\x1b[33m'
const GREEN = '\x1b[32m'
const WHITE = '\x1b[37m'
const RESET = '\x1b[0m'

const originalInfo = logger.info
const originalWarn = logger.warn
const originalError = logger.error
const originalDebug = logger.debug

logger.info = message => originalInfo(`${GREEN}${message}${RESET}`)
logger.warn = message => originalWarn(`${YELLOW}${message}${RESET}`)
logger.error = message => originalError(`${RED}${message}${RESET}`)
logger.debug = message => originalDebug(`${WHITE}${message}${RESET}`)
```

`colorizeLogger.js` importa el módulo `logger.js` original y sobrescribe las implementaciones de los métodos `info`, `warn`, `error` y `debug`. Sin embargo, estas nuevas implementaciones siguen llamando a los métodos originales para gestionar el registro real. El módulo `colorizeLogger.js` solo modifica el mensaje de entrada aplicando códigos de color ASCII, mientras que los métodos originales siguen siendo responsables de imprimir las entradas del registro. Este enfoque mantiene una clara separación de conceptos: el módulo logger es responsable de cómo se imprimen los registros en la terminal, mientras que el módulo colorizador es el único responsable de añadir color a los mensajes antes de que se envíen al logger.

Finalmente, veamos cómo podemos utilizar estos dos módulos juntos en una hipotética aplicación:

```javascript
// main.js
import { logger } from './logger.js'
import './colorizeLogger.js'

logger.info('Hello, World!')
logger.warn('Free disk space is running low')
logger.error('Failed to connect to database')
logger.debug('main() is starting')
```

En este archivo, importar `colorizeLogger.js` podría ser opcional. Podrías comentarlo para deshabilitar la salida coloreada sin afectar a la funcionalidad de la aplicación. Además, dado que `colorizeLogger.js` no exporta ninguna entidad, utilizamos una sintaxis de importación simplificada que omite la palabra clave `from`.

Este ejemplo ilustra cómo un módulo puede parchear a otro módulo utilizando módulos ES. Sin embargo, esta técnica puede ser arriesgada. Modificar el espacio de nombres global u otros módulos introduce efectos secundarios, impactando el estado de entidades fuera de su ámbito previsto. Esto puede llevar a consecuencias impredecibles, especialmente cuando múltiples módulos interactúan con las mismas entidades. Por ejemplo, si dos módulos diferentes intentan establecer la misma variable global o alterar la misma propiedad de un módulo, los resultados pueden ser inciertos (por ejemplo, ¿qué cambios prevalecerán?). Esta imprevisibilidad puede afectar a toda la aplicación.

Por lo tanto, es importante utilizar esta técnica con precaución y asegurarse de comprender todos los posibles efectos secundarios antes de implementarla.

Esta implementación del registrador tiene fines meramente demostrativos y no está pensada para su uso en producción. Si buscas un logger para tus aplicaciones de Node.js, considera usar **Pino** ([nodejsdp.link/pino](https://nodejsdp.link/pino)), una biblioteca de registro altamente eficiente, completa y extensible. Pino está diseñada teniendo en cuenta la extensibilidad, y existen extensiones como `pino-colada` ([nodejsdp.link/pino-colada](https://nodejsdp.link/pino-colada)) que proporcionan un formato y coloreado mejorados de la salida.

Aprendimos que las entidades importadas a través de módulos ES son enlaces dinámicos de solo lectura, lo que significa que no podemos reasignarlas desde un módulo externo. Sin embargo, hay una salvedad. Si bien no podemos cambiar las vinculaciones de la exportación por defecto o de las exportaciones con nombre de un módulo existente desde otro módulo, si una de estas vinculaciones es un objeto, aún podemos mutar el objeto en sí reasignando algunas de sus propiedades. Esto es precisamente lo que hicimos en nuestro ejemplo del registrador. Dado que `logger.js` exporta una vinculación llamada `logger` que es un objeto, podemos modificar las propiedades dentro de ese objeto, aunque no podríamos reasignar la vinculación `logger` por completo, como en el siguiente ejemplo:

```javascript
// replaceLogger.js
import { logger } from './logger.js'
const GREEN = '\x1b[32m'
// ...
const RESET = '\x1b[0m'
// replacing the logger binding with a new object
// this will fail with a TypeError: Assignment to constant variable.
logger = {
  info: message => {
    console.log(`INFO: ${GREEN}${message}${RESET}`)
  },
  // ...
}
```

Este ejemplo parcialmente implementado demuestra un enfoque alternativo potencial, pero defectuoso, para implementar nuestro colorizador de registros. Dado que esta implementación intenta anular la vinculación `logger` por completo, ejecutar este módulo generará un error: `TypeError: Assignment to constant variable`. Es importante señalar, no obstante, que aunque no podemos reasignar el objeto `logger` en sí, aún podríamos parchear los métodos individuales de la instancia del logger.

Un enfoque alternativo (aunque todavía defectuoso) para aplicar monkey patching a toda la vinculación de logger podría ser el siguiente:

```javascript
// replaceLogger2.js
import * as loggerModule from './logger.js'
const GREEN = '\x1b[32m'
// ...
const RESET = '\x1b[0m'
// replacing the logger binding inside the loggerModule with a new object
// this will fail with a TypeError: Cannot assign to read only property
loggerModule.logger = {
  info: message => {
    console.log(`INFO: ${GREEN}${message}${RESET}`)
  },
  // ...
}
```

La diferencia clave en este enfoque es que ahora estamos importando todo el módulo `logger.js` mediante una importación de espacio de nombres y luego intentando reasignar el miembro `logger` dentro de él. Sin embargo, este enfoque también falla porque los miembros del módulo son vinculaciones de solo lectura. La ejecución de este código daría como resultado un mensaje de error que confirma el problema: `TypeError: Cannot assign to read only property 'logger' of object '[object Module]'`. Ten en cuenta que incluso en este caso, si bien no podemos reasignar el objeto `logger` en sí, aún podríamos parchear los métodos individuales de la instancia del logger.

Existe otro enfoque que podríamos intentar, pero requiere realizar primero algunas modificaciones en el módulo `logger.js`:

```javascript
// logger.js
export const logger = {
  // …
}
export default {
  logger,
}
```

No hemos cambiado la implementación del módulo, pero hemos añadido una exportación por defecto que envuelve a todos los miembros del módulo —específicamente, la vinculación `logger`— dentro de un objeto. Al hacer esto, la exportación por defecto se convierte en un objeto, y aunque no podemos reasignar toda la vinculación por defecto, aún podemos modificar los miembros individuales del objeto. Este enfoque proporciona la flexibilidad necesaria para crear una alternativa funcional para nuestro módulo de reemplazo del registrador:

```javascript
// replaceLogger3.js
import loggerModule from './logger.js'
const GREEN = '\x1b[32m'
// …
const RESET = '\x1b[0m'
loggerModule.logger = {
  info: message => {
    console.log(`INFO: ${GREEN}${message}${RESET}`)
  },
  // …
}
```

Este enfoque se parece mucho a lo que hicimos con nuestra implementación del módulo `replaceLogger2.js`, pero con una diferencia crucial: en lugar de realizar una importación de espacio de nombres, ahora estamos importando la exportación por defecto del módulo `logger.js`. Con una importación de espacio de nombres, recibimos una instancia de módulo con miembros de solo lectura, lo que nos impide reasignar el miembro `logger`. Al utilizar una importación por defecto, sin embargo, obtenemos un objeto plano. Esto nos permite modificar sus miembros, incluido el miembro `logger`, según lo previsto. Como resultado, la ejecución de este código transcurrirá sin errores.

Usemos este nuevo módulo de monkey patching. Sin embargo, hay una salvedad. Supongamos que queremos usar el siguiente código:

```javascript
// main2Broken.js
import { logger } from './logger.js'
import './replaceLogger3.js'
logger.info('Hello, World!')
```

Esto significará que la salida no estará coloreada como se esperaba. Esto se debe a que estamos importando la exportación con nombre `logger` desde `logger.js`, pero `replaceLogger3.js` solo parchea el objeto exportado como exportación por defecto por `logger.js`. Podemos ilustrar el efecto del monkey patching aplicado por `replaceLogger3.js` en la Figura 2.8:

**Figura 2.8:** Cómo `replaceLogger3.js` aplica monkey patch a un miembro de la exportación por defecto.

En este ejemplo, `replaceLogger3.js` importa el objeto que `logger.js` exporta como exportación por defecto. Luego parchea la vinculación `logger` específica dentro de ese objeto. Esto significa que el parche solo afecta a la exportación por defecto y no a la exportación con nombre. Por lo tanto, para usar la versión parcheada del logger, debemos asegurarnos de que nuestro módulo principal importe la exportación por defecto.

Corrijamos nuestro código para hacer exactamente esto:

```javascript
// main2.js
import loggerModule from './logger.js'
import './replaceLogger3.js'
loggerModule.logger.info('Hello, World!')
loggerModule.logger.warn('Free disk space is running low')
loggerModule.logger.error('Failed to connect to database')
loggerModule.logger.debug('main() is starting')
```

En esta versión actualizada, importamos la exportación por defecto de `logger.js`. Al hacerlo, `replaceLogger3.js` parchea la misma referencia que se utiliza en el resto del código, lo que produce una salida coloreada según lo previsto.

Este enfoque es algo frágil porque requiere que se cumplan varias condiciones:
- El módulo objetivo (`logger.js` en nuestro ejemplo) debe exportar un objeto que contenga todos sus miembros como la exportación por defecto.
- El módulo que aplica el parche (`replaceLogger3.js` en nuestro ejemplo) debe importar la exportación por defecto y modificar sus miembros según sea necesario.
- El módulo consumidor (`main2.js` en nuestro ejemplo) debe importar y utilizar la exportación por defecto del módulo objetivo en lugar de sus exportaciones con nombre directamente.

Si el módulo objetivo o el módulo parcheador proceden de un tercero, es posible que no tengamos control sobre estas condiciones, lo que podría impedirnos aplicar esta técnica. Sin embargo, como veremos en el [Capítulo 10](https://subscription.packtpub.com/book/web-development/9781803238944/10), Pruebas: Patrones y mejores prácticas, esta técnica se puede utilizar con módulos centrales de Node.js y puede ser útil para escribir pruebas unitarias. Exploraremos ejemplos del uso de esta técnica de parcheo para simular (*mock*) el módulo del sistema de archivos en pruebas unitarias.

El monkey patching a veces se puede utilizar como un vector de ataque. Por ejemplo, si un atacante logra incluir código malicioso en una aplicación, este código podría aplicar monkey patching a módulos específicos para exfiltrar datos confidenciales, realizar escaladas de privilegios o alterar comportamientos importantes de la aplicación. Por esta razón, el equipo de Node.js ha asociado el monkey patching con la debilidad CWE-349. *Common Weakness Enumeration* (CWE) es una lista desarrollada por la comunidad de debilidades de software y hardware que pueden convertirse en vulnerabilidades. Los equipos de Node.js también ofrecen algunas sugerencias sobre cómo evitar el monkey patching por completo si deseas reforzar la seguridad de tu aplicación: [nodejsdp.link/cwe-349](https://nodejsdp.link/cwe-349).

##### Cómo afecta el Monkey Patching a la seguridad de tipos en proyectos de TypeScript

Si utilizas TypeScript, debes tener en cuenta que aplicar monkey patching con TypeScript conlleva desafíos adicionales que deben considerarse cuidadosamente. Antes de explorar algunos de ellos, ten en cuenta que TypeScript realiza comprobaciones de tipos y compila tu código a JavaScript plano antes de que pueda ejecutarse. El monkey patching es algo que ocurre en tiempo de ejecución, lo que significa que tiene lugar solo después de que TypeScript haya hecho su trabajo.

Veamos qué clase de desafíos crea esto:
- **Problemas de seguridad de tipos (*Type safety issues*):** TypeScript está diseñado para proporcionar seguridad de tipos asegurando que las variables, funciones y objetos se utilicen de acuerdo con sus tipos definidos. Con el monkey patching, generalmente estás alterando la estructura o el comportamiento de objetos o módulos existentes en tiempo de ejecución. Podrías estar cambiando algunos tipos de una manera que entre en conflicto con las definiciones de tipos originales. Imagina, por ejemplo, cambiar inadvertidamente el tipo de un miembro de un número a una cadena. Esto puede provocar errores de tipo inesperados en tiempo de ejecución. Si en algún momento del código se llama al método `toFixed()`, esto generaría un error en tiempo de ejecución porque el nuevo tipo (`string`) no tiene este método en particular. TypeScript podría advertirte sobre este problema potencial si realizas el monkey patching en el mismo proyecto TypeScript mostrando algunos errores en el lugar donde se realiza el parche (típicamente con un error como `"Type 'string' is not assignable to type 'number'"`). Sin embargo, si el parche se produce en una biblioteca de terceros o en un archivo JavaScript simple, en la mayoría de las circunstancias TypeScript no podrá detectar el problema de seguridad de tipos y correrás el riesgo de sufrir errores de tipo en tiempo de ejecución.
- **Desafíos con los archivos de declaración de tipos:** Trabajar con archivos de declaración de TypeScript (`.d.ts`) se vuelve más complejo cuando interviene el monkey patching. Es posible que necesites extender tipos o módulos existentes, lo cual puede ser complicado y propenso a errores, especialmente con estructuras de tipos complejas.
- **Pérdida de IntelliSense y autocompletado:** Uno de los mayores beneficios de TypeScript es proporcionar compleción de código y sugerencias inteligentes basadas en tipos. Cuando realizas monkey patching, puedes terminar cambiando la firma de funciones o los miembros de un objeto. Es posible que el IDE no capture correctamente estos cambios, por lo que es posible que no los veas representados adecuadamente cuando aparezca una sugerencia de autocompletado. Si estás añadiendo funcionalidad adicional, por ejemplo, agregando un nuevo método a una clase, es posible que este método no aparezca en absoluto en el cuadro de autocompletado. Todos estos pequeños problemas pueden crear fricción en el desarrollo, ralentizar el trabajo y aumentar la probabilidad de errores.

Dados estos desafíos, es importante pensar detenidamente antes de usar monkey patching en un proyecto de TypeScript. Si te preocupa la seguridad de tipos, también debes estar atento si estás importando módulos de terceros que puedan aplicar monkey patching, ya que esto podría acarrear los efectos secundarios no deseados que acabamos de comentar. Si bien el monkey patching puede resultar útil para la realización de pruebas, en la mayoría de los demás casos debe verse como un último recurso para extender módulos.

---

### Sección 5: Módulos CommonJS

Como ya hemos mencionado a lo largo de este capítulo, CommonJS está siendo reemplazado gradualmente por módulos ES, y ahora se recomienda utilizar módulos ES para nuevos proyectos y bibliotecas. Sin embargo, CommonJS no está oficialmente obsoleto (*deprecated*) y, dado su rol de larga data en el ecosistema de JavaScript, sigue siendo importante tener una comprensión básica del mismo. Este conocimiento te permitirá trabajar con confianza con bases de código y bibliotecas más antiguas.

Echemos un vistazo a la sintaxis básica de CommonJS.

Dos de los conceptos principales de la especificación CommonJS son los siguientes:
- `require`: una función que te permite importar un módulo desde el sistema de archivos local.
- `exports` y `module.exports`: variables especiales que se pueden utilizar para exportar funcionalidad pública desde el módulo actual.

Un módulo CommonJS simple podría tener este aspecto:

```javascript
// math.cjs
'use strict'

function add(a, b) {
  return a + b
}

module.exports = { add } // or `exports.add = add`
```

Podemos utilizar este módulo de la siguiente manera:

```javascript
'use strict'

const { add } = require('./math.cjs')
console.log(add(2, 3)) // 5
```

Estos fragmentos muestran lo sencillo que es exportar funcionalidad utilizando la variable `module.exports` (o simplemente `exports`) y cómo podemos importar fácilmente esa funcionalidad en otro módulo con la función `require()`.

Es importante señalar que habilitamos explícitamente el modo estricto utilizando `'use strict'`, ya que CommonJS no opera en modo estricto de forma predeterminada. Exploraremos este detalle más a fondo en la siguiente sección.

Otro detalle crucial a tener en cuenta es que la función `require()` en CommonJS es tanto sincrónica como dinámica. Opera de manera directa: cuando se ejecuta, `require()` resuelve el módulo especificado, carga el archivo asociado y ejecuta su contenido inmediatamente, sin necesidad de un callback. Esta naturaleza sincrónica influye en cómo definimos los módulos, ya que generalmente nos limita a utilizar código sincrónico durante la inicialización del módulo. Esta es también la razón por la que muchas bibliotecas centrales de Node.js proporcionan APIs sincrónicas junto con sus contrapartes asíncronas (por ejemplo, `readFile` y `readFileSync` en `node:fs`).

Si un módulo requiere inicialización asíncrona, un enfoque consiste en definir y exportar un módulo no inicializado que luego se inicializa asíncronamente. Sin embargo, este método no garantiza que el módulo esté listo para su uso inmediatamente después de cargarse. Exploraremos este problema y ofreceremos algunas soluciones elegantes en el [Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas (*Advanced Recipes*).

Node.js incluyó originalmente una versión asíncrona de `require()`, pero se eliminó rápidamente porque complicaba un proceso pensado para ser utilizado durante la inicialización, donde la E/S asíncrona a menudo introduce más desafíos que beneficios.

---

### Sección 6: Módulos ES y CommonJS: Diferencias e interoperabilidad

Analicemos ahora algunas diferencias importantes entre los módulos ES y CommonJS y cómo los dos sistemas de módulos pueden trabajar juntos cuando sea necesario.

#### Modo estricto (*Strict mode*)

A diferencia de CommonJS, los módulos ES se ejecutan implícitamente en modo estricto (*strict mode*). Esto significa que no tenemos que añadir explícitamente las sentencias `'use strict'` al principio de cada archivo. El modo estricto no se puede deshabilitar; por lo tanto, no podemos usar variables no declaradas o la sentencia `with`, ni contar con otras características que solo están disponibles en modo no estricto. Sin embargo, esto es definitivamente algo positivo, ya que el modo estricto es un modo de ejecución más seguro.

Si tienes curiosidad por conocer más acerca de las diferencias entre ambos modos, puedes consultar un artículo muy detallado en MDN Web Docs ([nodejsdp.link/strict-mode](https://nodejsdp.link/strict-mode)).

#### Top-level await

*Top-level await* permite a los desarrolladores utilizar `await` en el nivel superior de un módulo, simplificando el código asíncrono sin necesidad de envolverlo en una función `async`. En los módulos ES, `await` se puede utilizar directamente en el nivel superior, como se muestra aquí:

```javascript
// main.mjs
import { loadData } from 'someModule'
console.log(await loadData())
```

Sin embargo, en los módulos CommonJS, `await` no se puede utilizar directamente en el nivel superior. En su lugar, debes usarlo dentro de una función `async`:

```javascript
// main.cjs
'use strict'
const { loadData } = require('someModule')

async function main() {
  console.log(await loadData())
}

main()
```

Si intentas usar `await loadData()` fuera de una función `async`, obtendrás un `SyntaxError`. Esta limitación hace que sea más fácil trabajar con async/await en módulos ES en comparación con CommonJS.

#### Comportamiento de `this` (*Behavior of `this`*)

Una diferencia interesante entre los módulos ES y CommonJS es el comportamiento de la palabra clave `this`.

En el ámbito global de un módulo ES, `this` es `undefined`, mientras que en CommonJS, `this` es una referencia a `exports`:

```javascript
// main.mjs - ES modules
console.log(this) // undefined

// main.cjs – CommonJS
console.log(this === exports) // true
```

Ten en cuenta que en ambos sistemas de módulos, el comportamiento de la variable `globalThis` es consistente y hará referencia a un objeto que contiene utilidades globales de la plataforma como `setInterval` o `structuredClone`. Si deseas obtener más información sobre `globalThis`, visita [nodejsdp.link/global-this](https://nodejsdp.link/global-this).

#### Referencias ausentes en módulos ES (*Missing references in ES modules*)

Si estás acostumbrado a usar CommonJS, puede que te sorprenda la ausencia de ciertas referencias familiares en los módulos ES, como `require`, `exports`, `module.exports`, `__filename` y `__dirname`. Si intentamos usar cualquiera de ellas dentro de un módulo ES, dado que también se ejecuta en modo estricto, obtendremos un `ReferenceError`:

```javascript
console.log(exports) // ReferenceError: exports is not defined
console.log(module) // ReferenceError: module is not defined
console.log(__filename) // ReferenceError: __filename is not defined
console.log(__dirname) // ReferenceError: __dirname is not defined
```

`__filename` y `__dirname` representan la ruta absoluta al archivo del módulo actual y la ruta absoluta a su carpeta contenedora. Esas variables especiales pueden ser muy útiles cuando necesitamos construir una ruta relativa al archivo actual.

En los módulos ES, un objeto muy útil es `import.meta`. Si bien ya hemos discutido `import.meta.resolve()`, existen otras propiedades interesantes dentro de este objeto. Por ejemplo, puedes obtener los equivalentes a `__filename` y `__dirname` utilizando `import.meta.filename` e `import.meta.dirname`, respectivamente:

```javascript
// main.js
const __filename = import.meta.filename // /path/to/project/main.js
const __dirname = import.meta.dirname // /path/to/project
```

Estas propiedades son todavía relativamente nuevas (se introdujeron en Node.js v20.11.0, lanzado en enero de 2024). Si estás trabajando con una versión anterior, puedes usar `import.meta.url`, que es una referencia al archivo del módulo actual en un formato similar a `file:///path/to/current_module.js`. Veamos cómo se puede utilizar este valor para reconstruir la ruta del archivo actual y su directorio padre en forma de rutas absolutas:

```javascript
// main.js
import { fileURLToPath } from 'node:url'
import { dirname } from 'node:path'

const __filename = fileURLToPath(import.meta.url) // /path/to/project/main.js
const __dirname = dirname(__filename) // /path/to/project
```

Esto funciona porque `import.meta.url` nos proporcionará una URL que representa el archivo actual en la forma `"file:///path/to/project/main.js"`. La utilidad `fileURLToPath()` toma una URL `"file://..."` y la convierte en la ruta equivalente `"/path/to/project/main.js"`. Finalmente, `dirname()` puede tomar una ruta de archivo y devolver la ruta del directorio para ese archivo `"/path/to/project"`.

#### Interoperabilidad de importaciones (*Import interoperability*)

En ocasiones, es posible que necesites importar un módulo ES desde un módulo CommonJS o viceversa. Existen algunos detalles importantes y advertencias a considerar al manejar estas importaciones entre sistemas de módulos. Veamos rápidamente qué es posible y cómo gestionar estas importaciones en diferentes circunstancias.

##### Importar módulos CommonJS desde módulos ES

Se puede utilizar una sentencia `import` en un módulo ES para cargar un módulo CommonJS.

Veamos un ejemplo rápido:

```javascript
// someModule.cjs
'use strict'
module.exports = {
  someFeature: 'someFeature',
}
```

Este módulo CommonJS exporta un objeto con una propiedad `someFeature` utilizando `module.exports`. Ahora veamos cómo podemos importar este módulo desde un módulo ES:

```javascript
// main.js
import someModule from './someModule.cjs'
console.log(someModule)
```

Esto funciona según lo esperado y la salida de este archivo será la siguiente:

```text
{ someFeature: 'someFeature' }
```

Sin embargo, ¿qué pasa si solo queremos importar `someFeature`? Probemos:

```javascript
// main2.mjs
import { someFeature } from './someModule.cjs'
console.log(someFeature)
```

Si ejecutamos este código, veremos un mensaje de error bastante detallado:

```text
import { someFeature } from './someModule.cjs'
        ^^^^^^^^^^^
SyntaxError: Named export 'someFeature' not found. The requested module './someModule.cjs' is a CommonJS module, which may not support all module.exports as named exports.
CommonJS modules can always be imported via the default export, for example using:

import pkg from './someModule.cjs';
const { someFeature } = pkg;
```

El problema surge porque `someFeature` no se exportó directamente mediante `exports.someFeature`, por lo que no está disponible como una exportación con nombre al importarlo desde un módulo ES. Si puedes modificar el módulo CommonJS, podrías resolver esto asignando `someFeature` a `exports.someFeature`. Sin embargo, si no puedes cambiar el módulo de origen (por ejemplo, si se trata de un módulo de terceros instalado desde npm), existen un par de soluciones alternativas a las que puedes recurrir.

La primera solución es la recomendada por el mensaje de error anterior. Podemos importar el módulo completo (utilizando la sintaxis de importación por defecto) y luego utilizar la sintaxis de asignación por desestructuración para extraer `someFeature` del objeto importado:

```javascript
// main3.mjs
import someModule from './someModule.cjs'
const { someFeature } = someModule // destructuring assignment
console.log(someFeature)
```

En el fragmento anterior, la asignación por desestructuración es una forma compacta de extraer la propiedad `someFeature` de `someModule` y asignarla a una variable local llamada `someFeature`. Es prácticamente equivalente a escribir lo siguiente:

```javascript
const someFeature = someModule.someFeature
```

El otro enfoque consiste en recrear la función `require()` en módulos ES utilizando la función `createRequire()` proporcionada por el módulo central de Node.js `node:module`:

```javascript
// main4.mjs
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const { someFeature } = require('./someModule.cjs')
console.log(someFeature)
```

La función `require()` que creamos en este ejemplo se comporta exactamente como lo hace `require()` en un módulo CommonJS. Por lo tanto, podemos usarla sin preocuparnos demasiado por incompatibilidades. Veremos cómo se puede utilizar esto también para importar archivos JSON en el contexto de módulos ES más adelante en este capítulo.

##### Importar módulos ES desde CommonJS

Al momento de escribir este libro, al intentar importar un módulo ES en un módulo CommonJS, es posible que encuentres problemas. Veamos un ejemplo:

```javascript
// someModule.mjs
export const someFeature = 'someFeature'

// main.cjs
'use strict'
const { someFeature } = require('./someModule.mjs')
```

Dependiendo de tu versión de Node.js, cuando ejecutes `main.cjs` podrías ver el siguiente error:

```text
Error [ERR_REQUIRE_ESM]: require() of ES Module […]/someModule.mjs not supported.
Instead change the require of […]/someModule.mjs to a dynamic import() which is available in all CommonJS modules.
```

El mensaje de error es claro: no puedes usar `require()` para importar un módulo ES en un módulo CommonJS. Sin embargo, puedes utilizar una importación dinámica `import()` en su lugar, que es compatible en módulos CommonJS. He aquí cómo actualizar el ejemplo:

```javascript
// main2.cjs
'use strict'

async function main() {
  const { someFeature } = await import('./someModule.mjs')
  console.log(someFeature)
}

main()
```

Este código funciona según lo esperado. Cuando se ejecuta, `someModule.mjs` se importa dinámicamente y se imprime `"someFeature"` en la consola. Sin embargo, ten en cuenta que debido a que las importaciones dinámicas son asíncronas y estás trabajando dentro de un módulo CommonJS, no puedes usar *top-level await*. Debes envolver tu lógica en una función `async` y luego invocar esa función, lo que añade algo de código repetitivo (*boilerplate*).

Afortunadamente, Node.js ofrece un flag experimental, `--experimental-require-module`, que permite importaciones directas de módulos ES desde módulos CommonJS. Esto significa que nuestra implementación original puede funcionar si ejecutas el archivo de la siguiente manera:

```bash
node --experimental-require-module main.cjs
```

Esta característica simplifica la interoperabilidad entre CommonJS y los módulos ES, facilitando a los desarrolladores y mantenedores de bibliotecas la transición a módulos ES sin tener que preocuparse por la compatibilidad con CommonJS. Para cuando leas esto, es posible que este flag se haya vuelto estable, eliminando por completo la necesidad de soluciones alternativas.

Ten en cuenta que, al momento de escribir esto, el flag `--experimental-require-module` no admite módulos ES que utilicen *top-level await*.

#### Importación de archivos JSON (*Importing JSON files*)

JSON es un formato de datos ampliamente utilizado en el desarrollo web y es común necesitar cargar datos de un archivo JSON en tus programas. En CommonJS esto es directo:

```javascript
// main.cjs
'use strict'
const data = require('./sample.json')
console.log(data)
```

Usando `require()`, puedes cargar y parsear un archivo JSON con una sola línea de código. Esta función lee automáticamente el contenido del archivo, lo analiza como JSON y devuelve el resultado. Es equivalente a leer manualmente el archivo con `readFileSync()` y parsearlo con `JSON.parse()`, pero con mucho menos código redundante.

Veamos ahora cómo hacer lo mismo en un módulo ES:

```javascript
// main.mjs
import data from './sample.json'
console.log(data)
```

Esto parece lógico, pero no funcionará. La ejecución de este código dará como resultado un error:

```text
TypeError [ERR_IMPORT_ATTRIBUTE_MISSING]: Module "[…]/sample.json" needs an import attribute of "type: json"
```

Este error es en realidad muy informativo. Nos indica que debemos especificar que estamos importando un archivo JSON añadiendo un atributo de importación (*import attribute*). He aquí cómo hacerlo:

```javascript
// main2.mjs
import data from './sample.json' with { type: 'json' }
console.log(data)
```

Esto funciona, pero al momento de escribir esto verás una advertencia que indica que el soporte de módulos JSON todavía es experimental en Node.js.

La razón de esta sintaxis especial es la seguridad. Al declarar explícitamente el tipo, el cargador de módulos ES comprende que está tratando con JSON y no intentará ejecutar ningún código del módulo. Esto previene un escenario en el que, incluso si el archivo tiene una extensión `.json`, aún pudiera contener código JavaScript que luego pudiera ejecutarse, creando un posible vector de ataque. Si deseas conocer más sobre esta característica, debes consultar el repositorio de la propuesta de atributos de importación en [nodejsdp.link/proposal-import-attributes](https://nodejsdp.link/proposal-import-attributes).

Esta sintaxis, con algunas pequeñas diferencias sintácticas, también se puede utilizar con importaciones dinámicas de módulos ES:

```javascript
// main3.mjs
const { default: data } = await import('./sample.json', {
  with: { type: 'json' },
})
console.log(data)
```

Aquí la única diferencia es que el resultado de la importación está envuelto en una propiedad `default`, por lo que debes acceder a los datos JSON a través de ella. Al igual que la importación estática, este código también activará la advertencia de característica experimental.

Si no te sientes cómodo utilizando características experimentales, existen enfoques alternativos. El más obvio es leer y parsear manualmente el archivo JSON:

```javascript
// main4.js
import { readFile } from 'node:fs/promises'
import { join } from 'node:path'

const jsonPath = join(import.meta.dirname, 'sample.json')

try {
  const dataRaw = await readFile(jsonPath, 'utf-8')
  const data = JSON.parse(dataRaw)
  console.log(data)
} catch (error) {
  console.error(error)
}
```

Si bien este método funciona, requiere más código. Una opción más concisa es utilizar la función `require()` dentro de un módulo ES recurriendo a la utilidad `createRequire()` que vimos anteriormente:

```javascript
// main5.mjs
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const data = require('./sample.json')
console.log(data)
```

Importar archivos JSON en Node.js puede ser directo o un poco más complejo, dependiendo de si estás utilizando CommonJS o módulos ES. CommonJS lo hace fácil con la función `require()`, mientras que los módulos ES añaden un poco más de complejidad con una sintaxis especial para garantizar la seguridad. Aunque la nueva sintaxis para importar JSON en módulos ES sigue siendo experimental, está diseñada para proteger contra riesgos potenciales. Ya sea que prefieras la simplicidad de CommonJS o desees adoptar las funciones de seguridad de los módulos ES, dispones de opciones para elegir en función de lo que mejor funcione para tu proyecto.

---

### Sección 7: Uso de módulos en TypeScript

Al utilizar diferentes sistemas de módulos en TypeScript, es importante comprender que TypeScript es un superconjunto de JavaScript, diseñado para integrarse perfectamente con varios ecosistemas y plataformas, incluidos navegadores, Node.js y otros entornos JavaScript. Como compilador (o transpilador), TypeScript gestiona módulos en dos contextos principales: cómo estructuras tu código con módulos durante el desarrollo (módulos de entrada o detectados) y el formato de los módulos cuando tu código TypeScript se compila en JavaScript (módulos de salida o emitidos). TypeScript puede incluso convertir entre diferentes sistemas de módulos, permitiéndote escribir código utilizando módulos ES y compilarlo en CommonJS, por ejemplo.

La forma en que opera TypeScript puede variar de un proyecto a otro y viene determinada por el archivo de configuración `tsconfig.json`. Este archivo ofrece una amplia gama de opciones con una flexibilidad considerable, lo que a veces puede resultar abrumador y dificultar la consecución de los resultados deseados.

En esta sección entenderemos cómo funciona TypeScript en lo que respecta al manejo de módulos y cubriremos algunas de las opciones de configuración más importantes relacionadas con ellos. Estas opciones deberían ayudarte a crear una configuración que se adapte a tus necesidades o, si estás trabajando en una base de código TypeScript existente, a entender cómo se pretende utilizar los módulos en ese proyecto en particular.

#### El rol del compilador de TypeScript (*The role of the TypeScript compiler*)

El objetivo principal del compilador de TypeScript es detectar posibles errores en tiempo de ejecución durante el tiempo de compilación. Independientemente de si intervienen módulos o no, el compilador debe comprender las características específicas del entorno de ejecución esperado (el sistema anfitrión), incluyendo, por ejemplo, qué variables globales estarán disponibles. Cuando se introducen módulos, el compilador tiene que enfrentarse a desafíos adicionales.

Analicémoslos con un ejemplo:

```typescript
// hello.ts
import sayHello from 'greetings'
sayHello('world')
```

Para compilar `hello.ts` con precisión, el compilador de TypeScript debe determinar varios factores clave sobre la estructura del código de entrada y las características del entorno de destino que ejecutará el código compilado:
- **Carga de módulos (*Module loading*):** Suponiendo que el módulo `greetings` se escribió originalmente en TypeScript, ¿el sistema de módulos cargará un archivo TypeScript (por ejemplo, `greetings.ts`) o cargará un archivo JavaScript precompilado (por ejemplo, `greetings.js`) que podría estar disponible en el mismo paquete?
- **Tipo de módulo y resolución de módulos (*Module type and module resolution*):** ¿Qué tipo de formato de módulo espera el sistema de destino, en función del nombre y la ubicación del archivo? Esto se basa en convenciones. En nuestro ejemplo, el especificador del módulo es solo `"greetings"`, por lo que probablemente se refiere a una biblioteca de terceros instalada en la carpeta `node_modules`. Sin embargo, esta información por sí sola no determina explícitamente qué archivo exacto debe cargarse. TypeScript también admite alias de rutas (o reasignación de rutas / *path remapping*), lo que permite a los desarrolladores definir sus propios prefijos de ruta personalizados para importar archivos del proyecto actual ([nodejsdp.link/ts-path-aliases](https://nodejsdp.link/ts-path-aliases)). Esta puede ser una característica conveniente, pero también es otro factor que puede afectar a la resolución de módulos. Por lo tanto, debe configurarse y tenerse en cuenta adecuadamente durante la fase de compilación. Una vez que se ha identificado y cargado un archivo de módulo, la siguiente pregunta es: ¿qué tipo de módulo está utilizando el archivo cargado? Estamos utilizando la sintaxis de módulos ES al importar el módulo `greetings`, pero esto no implica necesariamente que el archivo cargado sea un módulo ES. De hecho, podría ser un módulo CommonJS.
- **Transformación de salida (*Output transformation*):** ¿Cómo se transformará la sintaxis del módulo durante el proceso de salida? Por ejemplo, ¿deben convertirse todas las importaciones y exportaciones de módulos ES a CommonJS?
- **Compatibilidad (*Compatibility*):** ¿Pueden los tipos de módulos detectados interactuar correctamente según la transformación de sintaxis?
- **Vinculación (*Binding*):** ¿Qué exportación específica del módulo `greetings` está vinculada a `sayHello`?

Estas preguntas dependen en gran medida de las características del sistema anfitrión que consume el JavaScript emitido o el TypeScript sin procesar (normalmente un entorno de ejecución como Node.js o un empaquetador como Webpack). Si bien la especificación ECMAScript describe cómo deben vincularse las importaciones y exportaciones de módulos ES, no define cómo se produce la resolución de módulos ni aborda otros sistemas de módulos como CommonJS. Esto significa que los entornos de ejecución y los empaquetadores tienen un margen significativo para diseñar sus propias reglas. Como resultado, el enfoque de TypeScript ante estas preguntas puede variar según dónde se ejecutará el código. No existe una única respuesta correcta, por lo que el compilador debe configurarse adecuadamente.

Por tanto, la responsabilidad de TypeScript en lo que respecta a los módulos se puede resumir de la siguiente manera:
- **Adaptarse al anfitrión:** Comprender las reglas del sistema anfitrión (por ejemplo, Node.js) lo suficientemente bien como para compilar archivos en un formato de módulo de salida válido.
- **Garantizar la compatibilidad:** Asegurar que las importaciones en los archivos de salida se resuelvan correctamente.
- **Asignación de tipos:** Asignar tipos precisos a las entidades importadas.

Para obtener más detalles y ejemplos, puedes consultar la sección oficial de Módulos (teoría) del sitio web de TypeScript en [nodejsdp.link/ts-modules-theory](https://nodejsdp.link/ts-modules-theory).

#### Configuración del formato de salida de módulos (*Configuring the module output format*)

La opción del compilador `module` informa al compilador sobre el formato de módulo deseado para el JavaScript emitido. Si bien su función principal es controlar el formato de salida, también guía al compilador sobre cómo detectar tipos de módulos, gestionar importaciones entre diferentes clases de módulos y manejar características como `import.meta` y *top-level await*. Incluso si tu proyecto TypeScript no emite archivos JavaScript (`noEmit`), seleccionar la configuración de módulo adecuada es crucial para una correcta comprobación de tipos e IntelliSense.

Para obtener una guía exhaustiva sobre cómo elegir la configuración de módulo adecuada para tu proyecto, consulta la sección Módulos en el manual oficial de TypeScript: [nodejsdp.link/ts-modules](https://nodejsdp.link/ts-modules).

Al trabajar con Node.js, recomendamos establecer la opción `module` en `NodeNext`, reflejando el sistema de módulos de Node.js más reciente.

#### Sintaxis de módulos de entrada y emisión de salida (*Input module syntax and output emission*)

Es importante comprender que la sintaxis del módulo de entrada en tus archivos TypeScript no siempre se corresponde directamente con la sintaxis del módulo de salida en los archivos JavaScript. Por ejemplo, un archivo con importaciones de módulos ES puede emitirse como módulos ES o transformarse en CommonJS, según la opción del compilador `module` y las reglas de detección pertinentes. Esta flexibilidad significa que simplemente inspeccionar un archivo de entrada no es suficiente para determinar si es un módulo ES o un módulo CommonJS: TypeScript puede convertir entre sistemas de módulos, una característica que, si bien es potente, a veces puede generar resultados inesperados y problemas de interoperabilidad.

En TypeScript 5.0 se introdujo la opción `verbatimModuleSyntax` para ayudar a los desarrolladores a comprender exactamente cómo se emitirán sus sentencias de importación y exportación. Cuando está habilitado, este flag exige que las importaciones y exportaciones en los archivos de entrada se escriban en la forma que experimentará la menor transformación antes de la emisión. Por ejemplo, si un archivo se emite como módulos ES, sus importaciones y exportaciones deben escribirse en la sintaxis de módulos ES. Cuando el objetivo es Node.js, recomendamos habilitar `verbatimModuleSyntax` para mantener los resultados consistentes y predecibles.

#### Resolución de módulos (*Module resolution*)

Si bien la especificación ECMAScript define el análisis y la interpretación de las sentencias de importación y exportación, deja los detalles de la resolución de módulos al sistema anfitrión. Para garantizar que TypeScript interprete estas sentencias de una manera que sea compatible con Node.js, debes establecer la opción `moduleResolution` en tu archivo `tsconfig.json`. Para Node.js, recomendamos establecer esta opción en `NodeNext`, que complementa la opción `module` y adopta el mismo valor de forma predeterminada si no se especifica. `NodeNext` está diseñado para admitir las próximas características de resolución de módulos de Node.js.

Escribir el archivo de configuración de TypeScript perfecto para tu proyecto requiere una comprensión adecuada de tu contexto específico y de cómo funciona el compilador de TypeScript. Confiamos en haberte proporcionado los conceptos básicos aquí. Si estás buscando configuraciones listas para usar en diferentes contextos, este repositorio puede ser un excelente recurso para consultar: [nodejsdp.link/total-typescript-tsconfig](https://nodejsdp.link/total-typescript-tsconfig).

---

### Sección 8: Resumen

En este capítulo hemos explorado la necesidad de módulos en JavaScript y la evolución de los sistemas de módulos en Node.js, centrándonos en CommonJS y los módulos ES. Cubrimos conceptos clave como exportaciones con nombre y por defecto, importaciones estáticas y dinámicas, y el algoritmo de resolución de módulos. También analizamos las implicaciones de las dependencias circulares, las fases de carga de módulos y cómo los módulos pueden modificar a otros (*monkey patching*).

Luego analizamos las diferencias entre CommonJS y los módulos ES, incluidos los desafíos de interoperabilidad, el modo estricto y las referencias no disponibles. Además, abordamos el impacto del monkey patching en la seguridad de tipos en proyectos de TypeScript.

Examinamos cómo importar archivos JSON en ambos sistemas de módulos y las soluciones alternativas a sus limitaciones. Finalmente, analizamos cómo aprovechar los módulos ES al utilizar TypeScript.

Con este conocimiento, ahora estás preparado para usar tanto módulos ES como CommonJS de manera efectiva, sentando las bases para el próximo capítulo sobre programación asíncrona en JavaScript. ¡Prepárate para explorar callbacks, eventos y sus patrones en profundidad!
