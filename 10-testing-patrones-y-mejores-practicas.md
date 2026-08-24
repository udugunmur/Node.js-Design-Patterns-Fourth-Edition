# Parte 2: Patrones de diseño de Node.js

## Capítulo 10: Testing: Patrones y mejores prácticas

Ha sido un largo viaje; ¡felicitaciones por llegar tan lejos en este libro! A lo largo de los primeros nueve capítulos de este libro, hemos explorado el intrincado mundo de JavaScript y Node.js. Entendimos cómo funciona el bucle de eventos (*event loop*) y cómo aprovechar al máximo la naturaleza asíncrona de JavaScript en Node.js a través de eventos, callbacks, promesas, async/await y streams. Aprendimos diversas técnicas y patrones de diseño para construir software escalable, mantenible y confiable. Cubrimos patrones creacionales para gestionar la instanciación de objetos, patrones estructurales para ensamblar objetos y clases en estructuras más grandes, y patrones de comportamiento para definir cómo interactúan los objetos. Esta sólida base nos ha preparado para el siguiente paso crítico: asegurar que nuestro código se comporte como se espera mediante pruebas exhaustivas.

Incluso el código diseñado con más cuidado no es inmune a los errores (*bugs*). Como dijo célebremente Edsger W. Dijkstra, una de las figuras más influyentes en la historia de la ingeniería de software y la informática: *"¡Las pruebas de programas pueden usarse para demostrar la presencia de errores, pero nunca para demostrar su ausencia!"*

Esta sincera observación nos recuerda que, aunque las pruebas nunca pueden garantizar la perfección, son una herramienta indispensable en nuestro arsenal para minimizar riesgos y construir software confiable. Los patrones de diseño que hemos explorado hasta ahora nos permiten escribir código elegante y mantenible, pero las pruebas son lo que realmente puede brindarnos la confianza de que nuestro código es resiliente y puede soportar las incertidumbres del uso en el mundo real.

En este capítulo, nos sumergiremos en el mundo de las pruebas en Node.js. Exploraremos preguntas esenciales como: ¿Cómo estructuramos nuestras pruebas de manera efectiva? ¿Cómo podemos aislar las dependencias para asegurar que nuestras pruebas sean significativas? ¿Qué herramientas y técnicas podemos usar para automatizar y optimizar nuestros procesos de prueba? Al responder a estas preguntas, te equiparemos con las habilidades necesarias para verificar con confianza que tu código se comporta según lo previsto en una variedad de escenarios.

Más específicamente, en este capítulo, aprenderás lo siguiente:
- Las definiciones y los principios de las pruebas, y los diversos tipos de pruebas, incluyendo pruebas unitarias, de integración y de extremo a extremo (*End-to-End* o E2E).
- La pirámide de pruebas, que brinda orientación sobre cómo equilibrar los diferentes tipos de pruebas dentro de tu proyecto.
- Cómo usar el ejecutor de pruebas (*test runner*) integrado de Node.js, una herramienta ligera y eficiente para escribir y ejecutar pruebas sin depender de dependencias externas.
- Dobles de prueba (*Test doubles*): Comprender stubs, mocks y espías para reemplazar dependencias reales y simular comportamientos en un entorno controlado.
- La sinergia entre los dobles de prueba y la Inyección de Dependencias (*Dependency Injection* o DI), una estrategia clave para mejorar la capacidad de prueba y la modularidad.
- Cómo escribir pruebas de integración (con el test runner integrado de Node.js) y pruebas E2E.

Las pruebas son un campo vasto y en constante evolución, con libros enteros dedicados a dominar sus complejidades. Aunque intentaremos proporcionar algunos conocimientos fundamentales, nuestro enfoque aquí se centra en los patrones de prueba prácticos que son más relevantes para el desarrollo en Node.js. A medida que continúes tu viaje, recuerda que las pruebas no solo mejoran la calidad de tu código, sino que también aumentan tu confianza como desarrollador, permitiéndote entregar software que realmente resista el paso del tiempo. Como tal, es una habilidad que te animamos a seguir explorando y perfeccionando incluso más allá de los límites de este libro para continuar creciendo como ingeniero de software.

¡Arremanguémonos y sumerjámonos en el mundo de las pruebas en Node.js!

---

### Sección 1: Una introducción al testing de software

Las pruebas son uno de los temas más debatidos entre los desarrolladores. ¿Deberías seguir el Desarrollo Guiado por Pruebas (*Test-Driven Development* o TDD) o el Desarrollo Guiado por el Comportamiento (*Behavior-Driven Development* o BDD)? ¿Qué califica exactamente como una prueba unitaria? ¿Qué nivel de cobertura de código es aceptable? ¿Es la cobertura de código realmente una métrica relevante? ¿Son necesarias las pruebas E2E? ¿Las pruebas deben realizarse localmente o en un entorno remoto? Y lo más importante, ¿cómo puedes asegurarte de que tus entornos de prueba repliquen de manera confiable la producción?

Si alguna vez has lidiado con estas preguntas, no estás solo. Las pruebas pueden ser un tema controvertido, pero siguen siendo una parte fundamental del desarrollo de software.

Para entender su importancia, considera una realidad alternativa donde los sistemas se despliegan sin ninguna prueba. Escribes algo de código, lo subes a producción y sigues adelante. Puede parecer eficiente... hasta que recibes una llamada a las 3 A.M. porque algo está roto. Ahora tienes que levantarte de la cama para diagnosticar y solucionar un problema que podría haberse detectado antes. Depurar bajo presión, desplegar correcciones y tal vez incluso revertir código... todo eso consume mucho más tiempo que si simplemente hubieras escrito algunas pruebas en primer lugar. Ese "atajo" termina siendo un camino muy largo.

Los beneficios de las pruebas van más allá de detectar errores temprano. Una de las ventajas más significativas de las pruebas es la retroalimentación rápida (*fast feedback*). Muchos desarrolladores trabajan en sistemas complejos, cambiando con frecuencia entre diferentes partes del código base. Cada cambio de contexto requiere tiempo para recuperar la concentración. Cuanto más tiempo se tarde en verificar un cambio, más difícil será mantenerse productivo. Si desplegar y luego verificar un cambio lleva 90 minutos, es probable que te distraigas, lo que dificulta volver a sumergirte en tu trabajo. Las suites de pruebas automatizadas bien estructuradas ayudan a resolver esto proporcionando retroalimentación inmediata, asegurando que puedas iterar rápida y eficientemente con un enfoque total.

Otro efecto positivo, quizás sorprendente, que las pruebas pueden tener en un código base es maximizar la colaboración. Tener un código base bien probado significa que el flujo de trabajo de desarrollo es más uniforme y predecible, lo que, a su vez, facilita la reproducción de problemas y la documentación de casos de prueba, aliviando la colaboración entre diferentes desarrolladores o equipos y acortando el tiempo necesario para entregar cambios. Esto es especialmente importante en proyectos de código abierto donde personas de los orígenes más diversos y con las expectativas más dispares vienen a contribuir: las pruebas son uno de los aspectos clave que permiten tanto a los colaboradores como a los mantenedores tener la confianza de que los cambios propuestos no afectarán la confiabilidad del software. Además de esto, las pruebas bien escritas también pueden servir como documentación. Por ejemplo, alguien que se une a un proyecto podría revisar las pruebas para comprender sus capacidades implementadas, los errores relevantes y los casos extremos (*edge cases*). Del mismo modo, adoptar una biblioteca de código abierto bien probada permite comprender cómo se puede usar y sus casos de uso admitidos simplemente leyendo las pruebas.

Ya hemos mencionado algunos términos específicos de pruebas. Si te estás acercando a este tema por primera vez, algunos de ellos pueden resultar confusos. Por lo tanto, antes de profundizar más, proporcionemos algunas definiciones.

#### Definiciones

Las definiciones de esta sección están destinadas a ayudarte a navegar por el resto de este capítulo y construir un modelo mental más estructurado sobre el amplio mundo del testing de software. Como ya hemos dicho, no vamos a profundizar en exceso, pero deberías poder utilizar estas definiciones como punto de partida si es necesario.

##### Sistema bajo prueba (System Under Test - SUT)

El Sistema Bajo Prueba (*System Under Test* o **SUT**) es el componente, módulo, función o aplicación completa específica que se evalúa o prueba durante un caso de prueba particular. Es el punto focal de la prueba: la parte del sistema cuyo comportamiento se está verificando. El SUT puede variar en alcance desde una sola línea de código hasta una aplicación compleja de varios niveles.

##### Arrange, Act, Assert (AAA)

El patrón **Arrange-Act-Assert (AAA)** es una forma ampliamente utilizada de estructurar pruebas unitarias y otros tipos de pruebas automatizadas. Proporciona un enfoque claro y consistente para escribir pruebas, haciéndolas más fáciles de entender, mantener y depurar.

Básicamente, divides el código de tu prueba en tres partes diferentes:
- **Arrange (Organizar/Preparar):** Configuras las condiciones previas para el SUT.
- **Act (Actuar/Ejecutar):** Ejecutas tu sistema con esas condiciones previas.
- **Assert (Afirmar/Verificar):** Verificas los resultados para confirmar que tu sistema se comportó como se esperaba.

Cada prueba que escribas, independientemente del tipo, probablemente seguirá estos tres pasos de alto nivel.

Veremos este enfoque en acción una vez que comencemos a trabajar en algunos ejemplos de pruebas unitarias.

##### Cobertura de código (Code coverage)

La cobertura de código (*code coverage*) es una métrica que mide el porcentaje de tu código base que se ejecuta cuando corres tu suite de pruebas. Proporciona información sobre cuán exhaustivamente tus pruebas ejercitan tu código. Dependiendo del marco de pruebas utilizado, la cobertura de código se puede medir en diferentes niveles, incluidos los siguientes:
- **Line coverage (Cobertura de líneas):** Porcentaje de líneas de código ejecutadas.
- **Branch coverage (Cobertura de ramas):** Porcentaje de puntos de decisión (por ejemplo, sentencias `if`, casos de `switch`) ejecutados.
- **Statement coverage (Cobertura de sentencias):** Porcentaje de sentencias ejecutables ejecutadas (una sola línea puede contener múltiples sentencias).
- **Class coverage (Cobertura de clases):** Porcentaje de clases cubiertas por la suite de pruebas.
- **Function coverage (Cobertura de funciones):** Porcentaje de funciones invocadas por la suite de pruebas.

Si bien la cobertura de código puede ser una herramienta útil para identificar áreas de tu código que carecen de pruebas, es crucial comprender sus limitaciones. Un alto porcentaje de cobertura de código no garantiza automáticamente un código de alta calidad ni que tus pruebas estén verificando de manera significativa el comportamiento de tu sistema.

La cobertura de código solo te dice qué código es ejecutado por tus pruebas, no cómo se prueba. Puedes lograr un 100% de cobertura de código con pruebas que no tengan aserciones, lo que significa que tus pruebas simplemente están ejecutando el código sin verificar que se comporte correctamente. Estos son ejemplos de pruebas triviales, y son malas: no aportan valor a un código base.

En lugar de centrarte únicamente en lograr un alto nivel de cobertura de código, prioriza escribir pruebas enfocadas que verifiquen minuciosamente el comportamiento de los componentes individuales bajo diferentes circunstancias significativas.

La cobertura de código debe considerarse como un subproducto de estas prácticas en lugar del objetivo principal. Es un indicador útil, pero no debería ser la única métrica que impulse tu estrategia de prueba o que mida la calidad de tus pruebas.

##### Dobles de prueba: Stubs, spies y mocks

Los dobles de prueba (*test doubles*) son componentes sustitutos que se utilizan para aislar el SUT de sus dependencias. Al igual que un doble de acción en una película, imitan implementaciones reales pero están adaptados para entornos de prueba controlados. Las dependencias (por ejemplo, bases de datos, APIs o servicios) se reemplazan con dobles de prueba para garantizar que el espectáculo continúe sin problemas, incluso cuando estas dependencias no estén listas, no sean confiables o no sea práctico usarlas. Desglosemos los tres tipos más comunes de dobles de prueba, sus propósitos y cómo usarlos de manera efectiva.

###### Stubs

Los **stubs** son la base del aislamiento en las pruebas. Imagina reemplazar un procesador de tarjetas de crédito en vivo con un actor con guión que siempre entrega la misma línea: "¡Aprobado!" o "¡Rechazado!". Eso es un stub.

Los stubs proporcionan respuestas estáticas y predeterminadas a llamadas de métodos. Ignoran el contexto: no importa qué entrada reciban, se apegan a su guión. Por ejemplo, un stub de API meteorológica siempre podría devolver "soleado" para probar la representación de la interfaz de usuario de una aplicación, evitando la imprevisibilidad del mundo real.

Los stubs son ideales para:
- Probar cómo el SUT maneja respuestas predecibles (por ejemplo, devolver datos de usuario simulados desde un servicio de autenticación).
- Simular escenarios de error (por ejemplo, forzar a un stub de base de datos a lanzar una excepción de tiempo de espera para probar la lógica de reintento).
- Acelerar las pruebas omitiendo operaciones lentas (por ejemplo, evitar llamadas de red reales o consultas pesadas a la base de datos).

Los stubs no realizan un seguimiento de cuántas veces se les llama ni de los argumentos que reciben: simplemente responden con lo que se les ha programado para devolver.

###### Spies (Espías)

Los **espías** (*spies*) van un paso más allá de los stubs. Son stubs con memoria: conservan el comportamiento similar al de un stub, pero registran las interacciones para su posterior inspección.

Un espía rastrea lo siguiente:
- Cuántas veces se llamó a un método.
- Qué argumentos se pasaron cada vez.
- El orden de las llamadas en una secuencia.

Los espías son particularmente útiles cuando necesitas verificar que el SUT interactuó con sus dependencias de la manera esperada sin alterar necesariamente el comportamiento de la dependencia. Por ejemplo, es posible que desees confirmar que se llamó a una función de registro con el mensaje de error correcto después de que fallara una operación. A diferencia de los stubs, los espías pueden envolver métodos reales, lo que permite que se ejecute la lógica original mientras se registra la actividad.

###### Mocks

Los **mocks** van un paso más allá que los espías al combinar respuestas preprogramadas con expectativas estrictas sobre cómo deben ser utilizados. Un mock no solo observa; hace cumplir un contrato.

Al configurar un mock, defines tanto el comportamiento esperado (lo que debe devolver) como el uso esperado (cómo debe interactuarse con él). Si el SUT no interactúa con el mock exactamente como se especificó, la prueba falla.

Por ejemplo, un servicio de autenticación mock podría exigir lo siguiente:
- La función `checkPermissions()` debe llamarse exactamente una vez con derechos de "admin".
- El método `logAccess()` debe invocarse una vez con el ID de usuario administrador actual.

Si se llama a `checkPermissions()` dos veces, o si nunca se llama a `logAccess()`, el mock fallará la prueba automáticamente, incluso si la lógica del SUT parece correcta en la superficie. Los mocks son ideales para probar la corrección del protocolo en sistemas complejos, donde la secuencia y la naturaleza exacta de las interacciones importan tanto como el resultado final.

##### Desarrollo guiado por pruebas (Test-Driven Development - TDD)

El **Desarrollo Guiado por Pruebas** (*Test-Driven Development* o **TDD**) es una metodología de desarrollo de software donde las pruebas se escriben antes del código que se supone deben validar. Fue popularizado por Kent Beck a fines de la década de 1990 como parte de la metodología *Extreme Programming* (XP).

El proceso de TDD sigue un ciclo estricto y repetitivo conocido como el ciclo **Rojo-Verde-Refactorización** (*Red-Green-Refactor*):
1. **Rojo (Red):** Escribe una prueba pequeña y enfocada para una característica o comportamiento específico que aún no existe. Ejecuta la prueba y observa cómo falla (de ahí el término "Rojo"). Esto confirma que la prueba está funcionando correctamente y que la característica realmente falta.
2. **Verde (Green):** Escribe la cantidad mínima de código necesaria para que la prueba pase. El objetivo aquí es la simplicidad y la velocidad, no la elegancia. Una vez que la prueba pasa (se vuelve "Verde"), sabes que el comportamiento deseado está implementado.
3. **Refactorizar (Refactor):** Limpia el código que acabas de escribir, mejorando su estructura, legibilidad y rendimiento, mientras te aseguras de que todas las pruebas sigan pasando. Este paso te permite mantener un código base limpio sin temor a romper la funcionalidad existente.

TDD fomenta un diseño modular y desacoplado, ya que el código debe ser testeable por naturaleza para adaptarse a este flujo de trabajo. También ayuda a prevenir el exceso de ingeniería, asegurando que los desarrolladores solo escriban el código necesario para cumplir con los requisitos. Si bien TDD requiere disciplina y puede ralentizar el desarrollo inicial, a menudo conduce a menos errores y menores costos de mantenimiento a largo plazo.

##### Desarrollo guiado por el comportamiento (Behavior-Driven Development - BDD)

El **Desarrollo Guiado por el Comportamiento** (*Behavior-Driven Development* o **BDD**) es una evolución de TDD que enfatiza la colaboración entre desarrolladores, evaluadores (*testers*) y partes interesadas (*stakeholders*) del negocio. BDD fue introducido por Dan North a principios de la década de 2000 como una forma de abordar algunos de los desafíos y confusiones que a menudo rodean a TDD, como qué probar, dónde comenzar y cómo estructurar las pruebas.

En BDD, los requisitos se expresan en un formato legible por humanos utilizando un lenguaje ubicuo que tanto los miembros técnicos como los no técnicos del equipo puedan entender. Este formato a menudo sigue la estructura **Dado-Cuando-Entonces** (*Given-When-Then*):
- **Dado (Given):** El contexto o estado inicial del sistema antes de que ocurra una acción.
- **Cuando (When):** El evento o acción desencadenada por el usuario o el sistema.
- **Entonces (Then):** El resultado esperado o consecuencia de la acción.

Por ejemplo, un escenario BDD para un proceso de inicio de sesión podría verse así:
- **Dado** que el usuario está en la página de inicio de sesión
- **Cuando** ingresa credenciales válidas y hace clic en "Iniciar sesión"
- **Entonces** debería ser redirigido a su panel de control y ver un mensaje de bienvenida

Al centrarse en el comportamiento del sistema desde la perspectiva del usuario, BDD ayuda a garantizar que el equipo de desarrollo esté creando las funciones correctas que se alinean con los objetivos comerciales. También sirve como documentación viva que se mantiene actualizada a medida que evoluciona el sistema.

##### Integración continua (Continuous Integration - CI)

La **Integración Continua** (*Continuous Integration* o **CI**) es una práctica de desarrollo de software donde los desarrolladores integran con frecuencia sus cambios de código en un repositorio central compartido (generalmente varias veces al día). Cada integración es verificada automáticamente por una compilación automatizada y la ejecución de la suite de pruebas automatizadas.

El objetivo principal de CI es detectar errores de integración de manera temprana y rápida, facilitando la localización y resolución de conflictos antes de que se conviertan en problemas mayores. Los aspectos clave de CI incluyen:
- Un repositorio de código fuente compartido (por ejemplo, Git en GitHub o GitLab).
- Un proceso de compilación y prueba automatizado activado por cada *push* o *pull request* (usando herramientas como GitHub Actions, GitLab CI, CircleCI, Jenkins, etc.).
- Comentarios rápidos para los desarrolladores si la compilación o las pruebas fallan.

Al automatizar el proceso de prueba en cada cambio, los equipos pueden mantener una rama principal siempre estable y lista para ser entregada.

##### Entrega continua y despliegue continuo (Continuous Delivery y Continuous Deployment - CD)

La **Entrega Continua** (*Continuous Delivery*) y el **Despliegue Continuo** (*Continuous Deployment*) son extensiones de CI que automatizan las etapas posteriores del ciclo de vida del lanzamiento del software:
- **Entrega Continua (Continuous Delivery):** Garantiza que cada cambio de código que pasa las pruebas automatizadas de CI se compila y empaqueta automáticamente para su lanzamiento en producción. Sin embargo, el despliegue real en producción requiere una aprobación manual.
- **Despliegue Continuo (Continuous Deployment):** Lleva la automatización un paso más allá al desplegar automáticamente cada cambio aprobado directamente a producción sin intervención humana, siempre que supere todas las pruebas y comprobaciones automatizadas.

Ambos enfoques reducen la fricción y el riesgo asociados con los lanzamientos de software, lo que permite a las organizaciones entregar nuevas características, mejoras y correcciones de errores a los usuarios de manera rápida, confiable y con un tiempo de inactividad mínimo.

---

#### Tipos de pruebas

Existen muchos tipos diferentes de pruebas de software, cada uno con un propósito específico y operando a un nivel diferente de granularidad. En el desarrollo moderno de Node.js, las pruebas se clasifican comúnmente en tres niveles principales: pruebas unitarias, pruebas de integración y pruebas de extremo a extremo (*End-to-End* o E2E).

##### Pruebas unitarias (Unit tests)

Las **pruebas unitarias** se centran en verificar la unidad más pequeña y aislada de código, normalmente una sola función, método o clase. Se ejecutan de forma completamente aislada de las dependencias externas (como bases de datos, sistemas de archivos o llamadas de red), las cuales se reemplazan mediante dobles de prueba (stubs, mocks o spies).

Ejemplos de pruebas unitarias:
- Una función `calculateTotal()` que suma los precios de los artículos y aplica impuestos.
- Una función `validateCoupon()` que comprueba si un código de descuento está activo.

Aquí están algunas de las principales cualidades de las pruebas unitarias:
- **Rápidas:** Al no involucrar I/O ni dependencias externas, se ejecutan en milisegundos.
- **Aisladas:** Un fallo en una prueba unitaria apunta directamente a la función o método específico con el error.
- **Económicas de escribir y mantener:** Requieren poca configuración de infraestructura.

##### Pruebas de integración (Integration tests)

Las **pruebas de integración** verifican cómo interactúan múltiples unidades, módulos o servicios entre sí. A diferencia de las pruebas unitarias, las pruebas de integración evalúan el flujo de datos y el comportamiento combinado a través de los límites de los componentes, lo que a menudo implica interactuar con recursos reales o semi-reales (como una base de datos SQLite en memoria, un servidor HTTP local o un sistema de archivos real).

Ejemplos de pruebas de integración:
- Verificar que una función `createOrder()` descuente correctamente el inventario de la base de datos e inserte un nuevo registro de pedido.
- Comprobar que un endpoint HTTP de Fastify responda con el código de estado y el cuerpo JSON correctos cuando se le envían datos válidos.

Aunque son muy potentes, las pruebas de integración conllevan una complejidad inherente:
- **Más lentas que las pruebas unitarias:** Involucran operaciones de I/O, configuración de bases de datos o red local.
- **Configuración más compleja:** Requieren preparar y limpiar el estado de la base de datos o los servicios antes y después de cada ejecución.
- **Diagnóstico más amplio:** Un fallo puede deberse a la lógica de negocio, a la capa de base de datos o a la comunicación entre componentes.

##### Pruebas de extremo a extremo (End-to-End tests o E2E)

Las **pruebas de extremo a extremo (E2E)** evalúan la aplicación completa desde la perspectiva del usuario final, abarcando toda la pila tecnológica (frontend, backend, base de datos y servicios de terceros). Automatizan las interacciones reales del usuario a través de un navegador web o un cliente de API.

Tomemos el siguiente ejemplo:
- Un usuario abre el navegador, navega a la página de registro, crea una cuenta, inicia sesión, busca un evento disponible, reserva un asiento y verifica que el evento aparezca en su panel de reservas.

Las pruebas E2E proporcionan el mayor nivel de confianza de que el sistema funciona correctamente como un todo, pero son las más costosas de escribir, las más lentas de ejecutar y las más propensas a la inestabilidad (*flakiness*) debido a tiempos de espera de red, animaciones de interfaz o dependencias de terceros.

##### Otros tipos de pruebas

Además de las tres categorías principales, existen otros tipos de pruebas especializadas:
- **Pruebas de regresión (Regression tests):** Garantizan que las nuevas funciones o correcciones no rompan el comportamiento existente.
- **Pruebas de rendimiento y carga (Performance & Load tests):** Evalúan cómo responde el sistema bajo diferentes volmenes de tráfico y estrés.
- **Pruebas de seguridad (Security tests):** Buscan vulnerabilidades como inyecciones SQL, XSS o fugas de datos.
- **Pruebas de mutación (Mutation tests):** Introducen pequeños cambios aleatorios ("mutaciones") en tu código para comprobar si tus pruebas fallan adecuadamente, evaluando así la eficacia de tus pruebas.

---

#### La pirámide de pruebas

La **pirámide de pruebas** (*testing pyramid*), introducida originalmente por Mike Cohn y popularizada por Martin Fowler, es un modelo visual que guía la distribución ideal de los diferentes tipos de pruebas en una suite de pruebas saludable.

```
       / \
      /   \     Pruebas E2E (Pocas, lentas, alta confianza)
     / E2E \
    /-------\
   /         \   Pruebas de integración (Moderadas, equilibradas)
  / Integración\
 /---------------\
/    Unitarias    \  Pruebas unitarias (Muchas, rápidas, bajo costo)
/-------------------\
```

*Figura 10.1 – Una representación moderna de la pirámide de pruebas.*

En resumen, la pirámide de pruebas sugiere lo siguiente:
- **Una base amplia de pruebas unitarias:** La gran mayoría de tus pruebas deben ser unitarias, ya que son ultrarrápidas, económicas y fáciles de ejecutar durante el desarrollo diario.
- **Una capa media de pruebas de integración:** Un número moderado de pruebas de integración para validar que los componentes individuales se ensamblen y comuniquen correctamente con la base de datos y los servicios.
- **Un puñado de pruebas E2E:** Un número reducido de pruebas de extremo a extremo de alto nivel para validar los flujos de usuario más críticos del negocio antes de enviar cambios a producción.

Mantener este equilibrio evita el antipatrón del "cono de helado" (*ice-cream cone*), donde un exceso de pruebas E2E lentas e inestables ahoga la velocidad de desarrollo del equipo.

---

### Sección 2: Escribir pruebas con Node.js

Hemos cubierto extensamente el amplio mundo de las pruebas, discutido los diferentes tipos de pruebas y destacado los principios rectores que ayudan a moldear nuestra mentalidad de prueba. ¡Ahora, finalmente es hora de arremangarse y escribir algo de código!

#### Nuestra primera prueba unitaria

Comencemos con un ejemplo simple utilizando una función de comercio electrónico. Construiremos una función llamada `calculateBasketTotal()` que toma un objeto `basket` (cesta) como entrada. La cesta contiene un array de artículos (`items`), donde cada artículo tiene un `name`, un `unitPrice` y una `quantity`. Un pequeño adelanto: nuestra implementación inicial contiene un error sutil que nuestra prueba expondrá más adelante:

```javascript
// calculateBasketTotal.js
export function calculateBasketTotal(basket) {
  let total = 0
  for (const item of basket.items) {
    total += item.unitPrice
  }
  return total
}
```

Para nuestra prueba inicial, vamos a escribir un script simple que utiliza la biblioteca integrada `assert` de Node.js para validar nuestro código:

```javascript
// test.js
import { equal } from 'node:assert/strict'
import { calculateBasketTotal } from './calculateBasketTotal.js'

// arrange
const basket = {
  items: [
    { name: 'Croissant', unitPrice: 2, quantity: 2 },
    { name: 'Olive bread', unitPrice: 3, quantity: 1 },
  ],
}

// act
const result = calculateBasketTotal(basket)

// assert
const expectedTotal = 7 // (2 * 2) + (3 * 1) = 7
equal(
  result,
  expectedTotal,
  `Expected total to be ${expectedTotal}, but got ${result}`
)

console.log('Test passed!')
```

El comportamiento general de este código debería ser relativamente fácil de entender, pero destaquemos algunos detalles importantes:
- Organizamos nuestras pruebas en tres fases distintas siguiendo la metodología **AAA**. Primero, la fase **Arrange** configura el entorno de prueba, en este caso, inicializando una cesta con algunos artículos. A continuación, la fase **Act** ejecuta el SUT (la función `calculateBasketTotal()`) contra la cesta preparada. Finalmente, la fase **Assert** valida si el resultado real coincide con el resultado esperado, asegurando que la lógica se comporte según lo previsto. Esta estructura mantiene las pruebas enfocadas, legibles y alineadas con el comportamiento probado.
- Para las aserciones, utilizamos la función integrada `equal()` de Node.js del módulo `node:assert/strict`. Esta función compara dos valores y lanza un error si difieren, deteniendo el script de prueba por completo (ya que los errores no se capturan). Si los dos valores son equivalentes, no ocurre nada. En otras palabras: si nuestro script sale con éxito, nuestra prueba pasó; de lo contrario, falló. El tercer parámetro que pasamos a la función `equal()` es opcional y representa un mensaje de error personalizado. Es una buena práctica proporcionar uno descriptivo para que sea más fácil entender qué falló cuando se lanza un error.
- La función `equal()` del módulo `node:assert/strict` considera que los dos valores dados son iguales si son dos valores primitivos (`undefined`, `null`, números, bigIntegers, cadenas y símbolos) con el mismo contenido. Si los dos valores son objetos, deben hacer referencia exactamente al mismo objeto para que la aserción pase. Si son dos objetos que son estructuralmente equivalentes (diferentes objetos en memoria pero con los mismos campos y los mismos valores para cada campo), no se considerarán iguales. Si necesitas comparar dos objetos por equivalencia estructural, puedes usar la función `deepEqual()`. En resumen: usa `equal()` para primitivas o comprobaciones estrictas de identidad de objetos; usa `deepEqual()` para comparar objetos por su contenido.
- Al importar `equal()` desde `node:assert/strict`, quizás te preguntes qué hace el submódulo `strict`. En resumen, impone el modo de aserción estricto, alterando el comportamiento de los métodos de aserción para evitar la coerción de tipos implícita, una fuente común de errores sutiles. Por ejemplo, bajo `strict`, `deepEqual()` se comporta automáticamente como `deepStrictEqual()`, lo que significa que verifica tanto el valor como el tipo (por ejemplo, `5 !== '5'`). Las aserciones no estrictas (como el `deepEqual()` original) están en desuso porque permiten la conversión automática de tipos (*type juggling*), lo que puede ocultar posibles errores. Al usar `node:assert/strict`, te aseguras de que todas las aserciones sigan reglas estrictas, lo que se considera una mejor práctica. Un beneficio adicional del modo estricto es una mejor depuración: cuando las aserciones fallan para los objetos, los mensajes de error incluyen un diff que resalta las discrepancias entre los valores esperados y los reales. Esto es especialmente útil al solucionar problemas de estructuras de datos complejas, ya que señala los desajustes y reduce la necesidad de inspección manual.

Para ejecutar este script de prueba, simplemente podemos ejecutar el siguiente comando en la terminal:

```bash
node test.js
```

Cuando ejecutes este archivo, verás un error. El error indica que el total calculado no coincide con nuestra expectativa:

```
AssertionError [ERR_ASSERTION]: Expected total to be 7, but got 5
5 !== 7
```

¡Te dijimos que había un error en nuestra función `calculateBasketTotal()`! Echemos un segundo vistazo a nuestra función... ¿Puedes ver el error?

¡Sí, olvidamos multiplicar por la cantidad del artículo! Vamos a arreglar eso:

```javascript
// calculateBasketTotal.js
export function calculateBasketTotal(basket) {
  let total = 0
  for (const item of basket.items) {
    total += item.unitPrice * item.quantity // <- FIX
  }
  return total
}
```

Después de actualizar el código, si ejecutamos la prueba nuevamente, la prueba debería pasar e imprimir `Test passed!` indicando que el error ha sido corregido. ¡Felicitaciones, has escrito tu primera prueba y también has corregido un error! Uf... este era peligroso. Imagina el daño monetario que podría haber causado en producción. Afortunadamente, ¡lo detectamos a tiempo con esta prueba!

---

### Sección 3: El test runner de Node.js

Hasta ahora, hemos ejecutado nuestro script de prueba manualmente invocando el comando `node test.js`. Si bien este enfoque funciona para un script simple, rápidamente se vuelve insostenible a medida que crece una aplicación. ¿Cómo ejecutamos cientos de pruebas organizadas en múltiples archivos? ¿Cómo recopilamos informes de ejecución y métricas de cobertura?

Históricamente, los desarrolladores de Node.js recurrían a bibliotecas de terceros como **Mocha**, **Jest** o **Vitest**. Sin embargo, a partir de Node.js 18 (y estabilizado en versiones posteriores), Node.js incluye un **test runner** nativo integrado directamente en el núcleo a través del módulo `node:test`.

El test runner de Node.js ofrece varias ventajas clave:
- **Cero dependencias:** No requiere instalar paquetes npm externos ni herramientas pesadas de compilación.
- **Rendimiento ultra rápido:** Construido directamente sobre el motor V8 de Node.js, iniciándose al instante.
- **Soporte de estándares modernos:** Soporte nativo para módulos ES (ESM), TypeScript (a partir de Node.js 22/23), async/await, promesas y streams.

#### Nuestra primera prueba con el test runner de Node.js

Reescribamos nuestra prueba anterior para utilizar el módulo `node:test`:

```javascript
// calculateBasketTotal.test.js
import { equal } from 'node:assert/strict'
import { test } from 'node:test'
import { calculateBasketTotal } from './calculateBasketTotal.js'

test('Calculates basket total with multiple items correctly', () => {
  // arrange
  const basket = {
    items: [
      { name: 'Croissant', unitPrice: 2, quantity: 2 },
      { name: 'Olive bread', unitPrice: 3, quantity: 1 },
    ],
  }

  // act
  const result = calculateBasketTotal(basket)

  // assert
  equal(result, 7)
})
```

Para ejecutar la prueba, ejecuta este comando en tu terminal:

```bash
node --test
```

Esto produce una salida como la siguiente:

```
✔ Calculates basket total with multiple items correctly (0.613417ms)
ℹ tests 1
ℹ suites 0
ℹ pass 1
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 43.0805
```

¡Excelente, todo en verde!

La función `test()` admite pruebas síncronas, funciones asíncronas que devuelven promesas y funciones con estilo de callback pasando un parámetro `done`. Veamos un ejemplo exhaustivo:

```javascript
// example.test.js
import { test } from 'node:test'

test('passing sync test', _t => {})
test('failing sync test', _t => {
  throw new Error('fail')
})

test('passing async test', async _t => {})
test('failing async test', async _t => {
  throw new Error('fail')
})

test('failing promise test', _t => {
  return Promise.reject(new Error('fail'))
})

test('passing callback test', (_t, done) => {
  setImmediate(done)
})

test('failing callback test', (_t, done) => {
  setImmediate(() => done(new Error('fail')))
})
```

Si ejecutamos `node --test`, obtenemos una salida estructurada que detalla exactamente qué pruebas pasaron y cuáles fallaron con sus trazas de pila:

```
✔ passing sync test (0.697458ms)
✖ failing sync test (0.916834ms)
  Error: fail
      at TestContext.<anonymous> (file:///.../example.test.js:5:9)
      ...

✔ passing async test (0.134542ms)
✖ failing async test (0.245833ms)
  Error: fail
      at TestContext.<anonymous> (file:///.../example.test.js:10:9)
      ...

✖ failing promise test (0.169125ms)
  Error: fail
      at file:///.../example.test.js:14:25
      ...

✔ passing callback test (0.33925ms)
✖ failing callback test (0.432667ms)
  Error: fail
      at file:///.../example.test.js:22:27
      ...

ℹ tests 7
ℹ suites 0
ℹ pass 3
ℹ fail 4
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 50.043583
```

---

#### Organización de pruebas

A medida que tu conjunto de pruebas crece, estructurar tus pruebas de manera clara y lógica se vuelve esencial.

##### Subtests

El objeto de contexto de prueba `t` que se pasa a la función de prueba permite definir **subtests** anidados, creando una jerarquía clara:

```javascript
// example.test.js
import { test } from 'node:test'

test('Top level test', t => {
  t.test('Subtest 1', _t => {
    // ...
  })

  t.test('Subtest 2', _t => {
    // ...
  })
})
```

Al ejecutar esta prueba, veremos la siguiente salida anidada:

```
▶ Top level test
  ✔ Subtest 1 (0.505417ms)
  ✔ Subtest 2 (0.119792ms)
✔ Top level test (1.464625ms)
ℹ tests 3
ℹ suites 0
ℹ pass 3
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 42.977416
```

> [!IMPORTANT]
> Cuando utilices subtests dentro de una función de prueba asíncrona (`async t =>`), **debes** usar `await` para cada subtest:
> ```javascript
> test('Top level test', async t => {
>   await t.test('Subtest 1', async _t => { /* ... */ })
>   await t.test('Subtest 2', async _t => { /* ... */ })
> })
> ```
> Si olvidas el `await`, la prueba principal terminará antes de que se completen los subtests, lo que provocará el error `'test did not finish before its parent and was cancelled'`.

##### Concurrencia de subtests

Por defecto, los subtests se ejecutan de forma secuencial. Si deseas que los subtests se ejecuten en paralelo para acelerar la ejecución, puedes pasar la opción `{ concurrency: true }`:

```javascript
// example.test.js
import { test } from 'node:test'

test('Top level test', { concurrency: true }, async t => {
  await Promise.all([
    t.test('Subtest 1', async _t => {
      // ...
    }),
    t.test('Subtest 2', async _t => {
      // ...
    }),
  ])
})
```

##### Casos de prueba parametrizados

A menudo querrás ejecutar la misma lógica de prueba con diferentes conjuntos de datos de entrada y salidas esperadas. En lugar de duplicar código, puedes iterar programáticamente sobre un array de casos de prueba:

```javascript
// calculateBasketTotal.test.js
import { equal } from 'node:assert/strict'
import { test } from 'node:test'
import { calculateBasketTotal } from './calculateBasketTotal.js'

const testCases = [
  {
    name: 'Empty basket',
    basket: { items: [] },
    expectedTotal: 0,
  },
  {
    name: 'Single item with quantity 1',
    basket: {
      items: [{ name: 'Croissant', unitPrice: 2, quantity: 1 }],
    },
    expectedTotal: 2,
  },
  {
    name: 'Single item with quantity > 1',
    basket: {
      items: [{ name: 'Croissant', unitPrice: 2, quantity: 3 }],
    },
    expectedTotal: 6,
  },
  {
    name: 'Multiple items',
    basket: {
      items: [
        { name: 'Croissant', unitPrice: 2, quantity: 2 },
        { name: 'Olive bread', unitPrice: 3, quantity: 1 },
      ],
    },
    expectedTotal: 7,
  },
]

for (const { name, basket, expectedTotal } of testCases) {
  test(name, () => {
    equal(calculateBasketTotal(basket), expectedTotal)
  })
}
```

##### Suites de prueba

Node.js también proporciona la función `suite()` (o su alias `describe()`) del módulo `node:test` para agrupar pruebas relacionadas de una manera similar a otros marcos como Mocha o Jest:

```javascript
// example.test.js
import { suite, test } from 'node:test'

suite('Top level suite', { concurrency: true }, () => {
  test('Test 1', () => {})
  test('Test 2', () => {})

  suite('Nested suite', () => {
    test('Nested test 1', () => {})
    test('Nested test 2', () => {})
  })
})
```

Salida:

```
▶ Top level suite
  ✔ Test 1 (0.505708ms)
  ✔ Test 2 (0.165416ms)
  ▶ Nested suite
    ✔ Nested test 1 (0.245875ms)
    ✔ Nested test 2 (0.125958ms)
  ✔ Nested suite (0.707541ms)
✔ Top level suite (2.057375ms)
ℹ tests 4
ℹ suites 2
ℹ pass 4
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 44.5385
```

---

#### Consejos y trucos del test runner

##### Modo watch

El test runner incluye un modo de observación (*watch mode*) que vuelve a ejecutar automáticamente las pruebas cada vez que se modifican los archivos de origen o de prueba:

```bash
node --test --watch
```

##### Ejecución de un subconjunto de todas tus pruebas

###### Descubrimiento de pruebas por defecto

Cuando ejecutas `node --test` sin argumentos, Node.js busca archivos que coincidan con las siguientes convenciones de nombres estándar:
- `**/*.test.{cjs,mjs,js}`
- `**/*-test.{cjs,mjs,js}`
- `**/*_test.{cjs,mjs,js}`
- `**/test-*.{cjs,mjs,js}`
- `**/test.{cjs,mjs,js}`
- `**/test/**/*.{cjs,mjs,js}`

###### Ejecución de pruebas dirigidas con patrones glob personalizados

Puedes especificar patrones de archivos o rutas exactas para ejecutar únicamente ciertas pruebas:

```bash
node --test "shoppingCart/**/*.test.js" "checkout/**/*.test.js"
```

###### Filtrar pruebas por nombre

Puedes filtrar qué pruebas ejecutar según su nombre utilizando expresiones regulares con el indicador `--test-name-pattern`:

```bash
node --test --test-name-pattern="suite 2 Test 2"
```

O para omitir pruebas que coincidan con un patrón:

```bash
node --test --test-skip-pattern="suite 1"
```

###### Filtrar pruebas con skip y only

Directamente en tu código, puedes usar `test.skip()` o la opción `{ skip: true }` para omitir pruebas temporalmente, y `test.only()` o `{ only: true }` para ejecutar exclusivamente una prueba específica (pasando `--test-only` al comando CLI):

```javascript
import { describe, it, suite, test } from 'node:test'

test.skip('skipped test', () => {})
it.skip('skipped it', () => {})
suite.skip('skipped suite', () => {})
describe.skip('skipped describe', () => {})

test('another skipped test', { skip: true }, () => {})
test('skipped test with a reason', { skip: 'Reason for skipping' }, () => {})

test.only('the only test that will run', () => {})
test('another only test', { only: true }, () => {})
```

##### Test reporters

Node.js admite varios formateadores de salida (*reporters*):
- `spec` (por defecto): Salida legible por humanos con marcas de verificación.
- `tap`: Formato estándar Test Anything Protocol.
- `dot`: Salida minimalista con puntos para pruebas rápidas.
- `lcov`: Para herramientas de cobertura de código.

Ejemplo:

```bash
node --test --test-reporter=dot
```

También puedes enviar diferentes formatos a diferentes destinos simultáneamente:

```bash
node --test --test-reporter=spec --test-reporter-destination=stdout --test-reporter=lcov --test-reporter-destination=lcov.info
```

##### Recopilación de cobertura de código

Node.js incluye soporte integrado para recopilar métricas de cobertura de código:

```bash
node --test --experimental-test-coverage
```

Puedes incluir o excluir archivos específicos de la cobertura:

```bash
# Solo recopilar cobertura para archivos *.js en src
node --test --experimental-test-coverage --test-coverage-include="src/*.js"

# Excluir todos los archivos *.js en someModule
node --test --experimental-test-coverage --test-coverage-exclude="someModule/*.js"
```

##### Visualización de la cobertura en la consola y el navegador

Para generar un informe HTML detallado e interactivo que puedas abrir en tu navegador web, puedes exportar a formato `lcov` y usar herramientas como `@lcov-viewer/cli` o `c8`:

```bash
node --test --experimental-test-coverage --test-reporter=lcov --test-reporter-destination=lcov.info
npx lcov-viewer lcov lcov.info
```

O usando `c8`:

```bash
NODE_V8_COVERAGE=./coverage npx c8 -r html node --test --experimental-test-coverage
```

El informe HTML generado estará disponible en `coverage/index.html`.

##### Uso del test runner con TypeScript

En las versiones modernas de Node.js (Node.js 22.6+ con soporte experimental de tipo de módulo o Node.js 23+), puedes ejecutar pruebas escritas en TypeScript directamente:

```typescript
// calculateBasketTotal.ts
export type BasketItem = {
  name: string
  unitPrice: number
  quantity: number
}

export type Basket = {
  items: BasketItem[]
}

export function calculateBasketTotal(basket: Basket): number {
  let total = 0
  for (const item of basket.items) {
    total += item.unitPrice * item.quantity
  }
  return total
}
```

```typescript
// calculateBasketTotal.test.ts
import { equal } from 'node:assert/strict'
import { test } from 'node:test'
import { calculateBasketTotal } from './calculateBasketTotal.ts'

test('Calculates basket total with multiple items correctly', () => {
  // arrange
  const basket = {
    items: [
      { name: 'Croissant', unitPrice: 2, quantity: 2 },
      { name: 'Olive bread', unitPrice: 3, quantity: 1 },
    ],
  }

  // act
  const result = calculateBasketTotal(basket)

  // assert
  equal(result, 7)
})
```

Para ejecutar pruebas TypeScript:

```bash
node --test '**/*.test.ts'
```

Y para recopilar cobertura excluyendo los propios archivos de prueba:

```bash
node --test --experimental-test-coverage --test-coverage-exclude='**/*.test.ts' '**/*.test.ts'
```

---

### Sección 4: Escribir pruebas unitarias

Ahora que dominamos los conceptos básicos del test runner de Node.js, profundicemos en las técnicas avanzadas para escribir pruebas unitarias de nivel profesional.

#### Probar código asíncrono

Una de las características más críticas de Node.js es su naturaleza asíncrona. Veamos cómo probar una clase compleja basada en eventos y promesas: `TaskQueue`.

```javascript
// TaskQueue.js
import { EventEmitter } from 'node:events'

export class TaskQueue extends EventEmitter {
  constructor(concurrency) {
    super()
    this.concurrency = concurrency
    this.running = 0
    this.queue = []
  }

  runTask(task) {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          return resolve(await task())
        } catch (err) {
          return reject(err)
        }
      })
      process.nextTick(this.next.bind(this))
    })
  }

  async next() {
    while (this.running < this.concurrency && this.queue.length) {
      const task = this.queue.shift()
      this.running++
      try {
        await task()
      } finally {
        this.running--
        this.next()
      }
    }
    if (this.running === 0 && this.queue.length === 0) {
      this.emit('empty')
    }
  }
}
```

Para probar esta clase, crearemos una suite completa que verifique:
1. La ejecución de tareas y la emisión del evento `empty`.
2. Que se respete el límite de concurrencia.
3. El manejo adecuado de errores sin detener la cola.
4. La emisión inmediata de `empty` si no se agregan tareas.

```javascript
// TaskQueue.test.js
import assert from 'node:assert/strict'
import { once } from 'node:events'
import { mock, suite, test } from 'node:test'
import { setImmediate } from 'node:timers/promises'
import { TaskQueue } from './TaskQueue.js'

suite('TaskQueue', () => {
  test('should execute all tasks and emit "empty" event', async () => {
    const queue = new TaskQueue(2)
    const emptyPromise = once(queue, 'empty')

    const task1 = mock.fn(async () => {
      await setImmediate()
      return 1
    })
    const task2 = mock.fn(async () => {
      await setImmediate()
      return 2
    })
    const task3 = mock.fn(async () => {
      await setImmediate()
      return 3
    })

    const results = await Promise.all([
      queue.runTask(task1),
      queue.runTask(task2),
      queue.runTask(task3),
    ])

    await emptyPromise

    assert.deepEqual(results, [1, 2, 3])
    assert.equal(task1.mock.callCount(), 1)
    assert.equal(task2.mock.callCount(), 1)
    assert.equal(task3.mock.callCount(), 1)
  })

  test('should respect concurrency limits', async () => {
    const queue = new TaskQueue(2)
    let currentlyRunning = 0
    let maxRunning = 0

    const createTask = () =>
      mock.fn(async () => {
        currentlyRunning++
        maxRunning = Math.max(maxRunning, currentlyRunning)
        await setImmediate()
        currentlyRunning--
      })

    const tasks = Array.from({ length: 5 }, createTask)

    await Promise.all(tasks.map(task => queue.runTask(task)))

    assert.equal(maxRunning, 2)
  })

  test('should handle task errors and still process subsequent tasks', async () => {
    const queue = new TaskQueue(2)

    const failingTask = mock.fn(async () => {
      await setImmediate()
      throw new Error('Task failed')
    })

    const succeedingTask = mock.fn(async () => {
      await setImmediate()
      return 'success'
    })

    const results = await Promise.allSettled([
      queue.runTask(failingTask),
      queue.runTask(succeedingTask),
    ])

    assert.equal(results[0].status, 'rejected')
    assert.equal(results[0].reason.message, 'Task failed')
    assert.equal(results[1].status, 'fulfilled')
    assert.equal(results[1].value, 'success')
  })

  test('should emit "empty" immediately if no tasks are added', async () => {
    const queue = new TaskQueue(2)
    const emptyHandler = mock.fn()

    queue.on('empty', emptyHandler)
    await queue.next()

    assert.equal(emptyHandler.mock.callCount(), 1)
  })
})
```

---

#### Mocking

El objeto `mock` exportado por `node:test` proporciona utilidades integradas para crear espías, interceptar métodos y simular módulos.

##### Creación de espías con `mock.fn()`

La función `mock.fn()` crea una función espía que envuelve una función existente o actúa como una función ficticia independiente:

```javascript
const mySpy = mock.fn((a, b) => a + b)

mySpy(1, 2)
mySpy(3, 4)

console.log(mySpy.mock.callCount()) // 2
console.log(mySpy.mock.calls[0].arguments) // [1, 2]
console.log(mySpy.mock.calls[0].result) // 3
```

##### Mocking de solicitudes HTTP con el mock integrado

Consideremos una función `getInternalLinks()` que analiza una página HTML y extrae todos los enlaces internos:

```javascript
// getPageLinks.js
import { Parser } from 'htmlparser2' // v9.1.0

export async function getInternalLinks(pageUrl) {
  const url = new URL(pageUrl)
  const response = await fetch(url)
  const body = await response.text()
  const links = new Set()

  const parser = new Parser({
    onopentag(name, attribs) {
      if (name === 'a' && attribs.href) {
        try {
          const resolved = new URL(attribs.href, url)
          if (resolved.origin === url.origin) {
            links.add(resolved.href)
          }
        } catch {
          // ignorar URLs no válidas
        }
      }
    },
  })

  parser.write(body)
  parser.end()

  return Array.from(links)
}
```

Podemos mockear la función global `fetch` usando `mock.method(globalThis, 'fetch')`:

```javascript
test('should extract internal links from a page', async () => {
  const html = `
    <html>
      <body>
        <a href="/about">About</a>
        <a href="https://example.com/contact">Contact</a>
        <a href="https://other.com/page">External</a>
      </body>
    </html>
  `

  mock.method(globalThis, 'fetch', async () => {
    return {
      text: async () => html,
    }
  })

  const links = await getInternalLinks('https://example.com')
  assert.deepEqual(links, [
    'https://example.com/about',
    'https://example.com/contact',
  ])
})
```

##### Mocking de solicitudes HTTP con Undici

Si bien mockear `globalThis.fetch` funciona, puede ser frágil y no simular fielmente el comportamiento de la red. La biblioteca oficial de HTTP para Node.js, **Undici**, proporciona una utilidad mucho más robusta: `MockAgent`.

```javascript
// getPageLinks.test.js
import assert from 'node:assert/strict'
import { afterEach, beforeEach, suite, test } from 'node:test'
import { MockAgent, getGlobalDispatcher, setGlobalDispatcher } from 'undici'
import { getInternalLinks } from './getPageLinks.js'

suite('getInternalLinks', () => {
  let mockAgent
  let originalDispatcher

  beforeEach(() => {
    originalDispatcher = getGlobalDispatcher()
    mockAgent = new MockAgent()
    mockAgent.disableNetConnect()
    setGlobalDispatcher(mockAgent)
  })

  afterEach(() => {
    setGlobalDispatcher(originalDispatcher)
  })

  test('should return all internal links from a page', async () => {
    const mockClient = mockAgent.get('https://example.com')
    mockClient
      .intercept({
        path: '/',
        method: 'GET',
      })
      .reply(
        200,
        `
        <html>
          <body>
            <a href="/about">About</a>
            <a href="https://example.com/contact">Contact</a>
            <a href="https://other.com/page">External</a>
          </body>
        </html>
        `,
        {
          headers: { 'content-type': 'text/html' },
        }
      )

    const links = await getInternalLinks('https://example.com')
    assert.deepEqual(links, [
      'https://example.com/about',
      'https://example.com/contact',
    ])
  })

  test('should return an empty array if no links are present', async () => {
    const mockClient = mockAgent.get('https://example.com')
    mockClient
      .intercept({
        path: '/empty',
        method: 'GET',
      })
      .reply(200, '<html><body>No links here!</body></html>', {
        headers: { 'content-type': 'text/html' },
      })

    const links = await getInternalLinks('https://example.com/empty')
    assert.deepEqual(links, [])
  })
})
```

##### Mocking de módulos centrales de Node.js

Node.js admite el mocking a nivel de módulo mediante `mock.module()`. Veamos cómo mockear el módulo `node:fs/promises` al probar una función `saveConfig()`:

```javascript
// saveConfig.js
import { access, mkdir, writeFile } from 'node:fs/promises'
import { dirname } from 'node:path'

export async function saveConfig(path, config) {
  const dir = dirname(path)
  try {
    await access(dir)
  } catch (err) {
    if (err.code === 'ENOENT') {
      await mkdir(dir, { recursive: true })
    } else {
      throw err
    }
  }

  await writeFile(path, JSON.stringify(config, null, 2))
}
```

Escribimos la prueba mockeando las exportaciones con nombre de `node:fs/promises`:

```javascript
// saveConfig.test.js
import assert from 'node:assert/strict'
import { mock, suite, test } from 'node:test'
import { setImmediate } from 'node:timers/promises'

suite('saveConfig', () => {
  test('should create directory if it does not exist and write config file', async () => {
    const writtenFiles = []
    const createdDirs = []

    mock.module('node:fs/promises', {
      namedExports: {
        access: async () => {
          await setImmediate()
          const err = new Error('ENOENT')
          err.code = 'ENOENT'
          throw err
        },
        mkdir: async (path, options) => {
          await setImmediate()
          createdDirs.push({ path, options })
        },
        writeFile: async (path, content) => {
          await setImmediate()
          writtenFiles.push({ path, content })
        },
      },
    })

    const { saveConfig } = await import('./saveConfig.js')

    const config = { port: 3000, host: 'localhost' }
    await saveConfig('/some/dir/config.json', config)

    assert.deepEqual(createdDirs, [{ path: '/some/dir', options: { recursive: true } }])
    assert.deepEqual(writtenFiles, [
      {
        path: '/some/dir/config.json',
        content: JSON.stringify(config, null, 2),
      },
    ])
  })
})
```

Para ejecutar pruebas que utilizan `mock.module()`, debemos proporcionar el indicador experimental:

```bash
node --test --experimental-test-module-mocks
```

##### Mocking de otras dependencias

También podemos mockear módulos de archivos locales o paquetes de terceros. Consideremos un módulo `payments.js` que instancia internamente un cliente de base de datos `DbClient`:

```javascript
// dbClient.js
export class DbClient {
  // biome-ignore lint/suspicious/useAwait: just for demonstration
  async query(_sql, _params) {
    // En la vida real, esto ejecutaría una consulta en una base de datos real
    throw new Error('Not implemented')
  }
}
```

```javascript
// payments.js
import { DbClient } from './dbClient.js'

const db = new DbClient()

export async function canPayWithVouchers(userId, amount) {
  const vouchers = await db.query(
    `SELECT * FROM vouchers 
       WHERE user_id = ? AND 
             used = false AND 
             expires_at > NOW()`,
    [userId]
  )

  const totalVoucherAmount = vouchers.reduce(
    (acc, voucher) => acc + voucher.amount,
    0
  )

  return totalVoucherAmount >= amount
}
```

Podemos mockear la clase exportada `DbClient` en `payments.test.js`:

```javascript
// payments.test.js
import assert from 'node:assert/strict'
import { after, beforeEach, mock, suite, test } from 'node:test'
import { setImmediate } from 'node:timers/promises'

suite('payments', () => {
  let queryMock
  let canPayWithVouchers

  beforeEach(async () => {
    queryMock = mock.fn()

    mock.module('./dbClient.js', {
      namedExports: {
        DbClient: class {
          query = queryMock
        },
      },
    })

    const payments = await import('./payments.js')
    canPayWithVouchers = payments.canPayWithVouchers
  })

  after(() => {
    mock.reset()
  })

  test('should return true if vouchers cover the amount', async () => {
    queryMock.mock.mockImplementation(async () => {
      await setImmediate()
      return [{ amount: 50 }, { amount: 60 }]
    })

    const result = await canPayWithVouchers('user-1', 100)
    assert.equal(result, true)
    assert.equal(queryMock.mock.callCount(), 1)
  })

  test('should return false if vouchers do not cover the amount', async () => {
    queryMock.mock.mockImplementation(async () => {
      await setImmediate()
      return [{ amount: 30 }, { amount: 20 }]
    })

    const result = await canPayWithVouchers('user-1', 100)
    assert.equal(result, false)
    assert.equal(queryMock.mock.callCount(), 1)
  })
})
```

##### Problemas con el mocking de imports

Aunque el mocking de módulos es posible, introduce varios inconvenientes importantes:
- **Acoplamiento con la implementación interna:** Las pruebas deben saber exactamente qué archivos y qué identificadores importa el módulo.
- **Complejidad y estado global:** Modificar la tabla de resolución de módulos puede provocar efectos secundarios entre pruebas.
- **Importaciones dinámicas obligatorias:** Exige el uso de `await import(...)` dentro de los bloques de prueba para que los mocks surtan efecto antes de que se evalúe el módulo.

##### Mocking de imports frente a Inyección de Dependencias

Una alternativa mucho más limpia, desacoplada y robusta que el mocking de imports es utilizar el patrón de **Inyección de Dependencias** (*Dependency Injection* o DI), que estudiamos en el [Capítulo 7](https://subscription.packtpub.com/book/web-development/9781803238944/7), Patrones de diseño creacionales.

En lugar de crear la instancia de `DbClient` dentro del módulo, la pasamos explícitamente como argumento:

```javascript
// payments.js
export async function canPayWithVouchers(db, userId, amount) {
  const vouchers = await db.query(
    `SELECT * FROM vouchers 
       WHERE user_id = ? AND 
             used = false AND 
             expires_at > NOW()`,
    [userId]
  )

  const totalVoucherAmount = vouchers.reduce(
    (acc, voucher) => acc + voucher.amount,
    0
  )

  return totalVoucherAmount >= amount
}
```

Ahora, la prueba es notablemente más simple, elegante y completamente libre de herramientas mágicas de mocking:

```javascript
// payments.test.js
import assert from 'node:assert/strict'
import { suite, test } from 'node:test'
import { setImmediate } from 'node:timers/promises'
import { canPayWithVouchers } from './payments.js'

suite('payments with DI', () => {
  test('should return true if vouchers cover the amount', async () => {
    const mockDb = {
      async query(_sql, _params) {
        await setImmediate()
        return [{ amount: 50 }, { amount: 60 }]
      },
    }

    const result = await canPayWithVouchers(mockDb, 'user-1', 100)
    assert.equal(result, true)
  })

  test('should return false if vouchers do not cover the amount', async () => {
    const mockDb = {
      async query(_sql, _params) {
        await setImmediate()
        return [{ amount: 30 }, { amount: 20 }]
      },
    }

    const result = await canPayWithVouchers(mockDb, 'user-1', 100)
    assert.equal(result, false)
  })
})
```

---

### Sección 5: Escribir pruebas de integración

Las pruebas unitarias verifican la lógica aislada, pero ¿cómo sabemos si nuestras consultas SQL son sintácticamente válidas o si nuestros esquemas de base de datos se comportan como se espera? Aquí es donde entran en juego las **pruebas de integración**.

#### Probar con una base de datos local

Veamos cómo probar la interacción real con una base de datos SQLite utilizando `sqlite3` y el wrapper `sqlite`.

Primero, implementamos un `DbClient`:

```javascript
// dbClient.js
import { open } from 'sqlite'
import sqlite3 from 'sqlite3'

export class DbClient {
  #dbPath
  #db

  constructor(dbPath) {
    this.#dbPath = dbPath
  }

  async #getDb() {
    if (!this.#db) {
      this.#db = await open({
        filename: this.#dbPath,
        driver: sqlite3.Database,
      })
    }
    return this.#db
  }

  async query(sql, params = []) {
    const db = await this.#getDb()
    return db.all(sql, params)
  }

  async execute(sql, params = []) {
    const db = await this.#getDb()
    return db.run(sql, params)
  }

  async close() {
    if (this.#db) {
      await this.#db.close()
      this.#db = null
    }
  }
}
```

A continuación, creamos una función para inicializar el esquema de la base de datos:

```javascript
// dbSetup.js
export async function createTables(db) {
  await db.query(`
    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL
    )
  `)

  await db.query(`
    CREATE TABLE IF NOT EXISTS vouchers (
      id TEXT PRIMARY KEY,
      userId TEXT NOT NULL,
      amount REAL NOT NULL,
      used INTEGER DEFAULT 0,
      expiresAt TEXT NOT NULL,
      FOREIGN KEY (userId) REFERENCES users(id)
    )
  `)
}
```

Implementamos la lógica de negocio en `payments.js`:

```javascript
// payments.js
export async function getActiveVouchers(db, userId) {
  const vouchers = await db.query(
    `SELECT * FROM vouchers 
       WHERE userId = ? AND 
             used = 0 AND 
             expiresAt > datetime('now')`,
    [userId]
  )

  return vouchers
}

export async function canPayWithVouchers(db, userId, amount) {
  const vouchers = await getActiveVouchers(db, userId)

  const totalVoucherAmount = vouchers.reduce(
    (acc, voucher) => acc + voucher.amount,
    0
  )

  return totalVoucherAmount >= amount
}
```

Ahora escribimos una prueba de integración que utiliza una base de datos SQLite real en memoria (`:memory:`), poblando datos reales y comprobando el comportamiento exacto de las cláusulas `datetime('now')` de SQL:

```javascript
// payments.int.test.js
import assert from 'node:assert/strict'
import { suite, test } from 'node:test'
import { DbClient } from './dbClient.js'
import { createTables } from './dbSetup.js'
import { canPayWithVouchers } from './payments.js'

suite('payments integration test', () => {
  test('should return true if vouchers cover the amount', async () => {
    const db = new DbClient(':memory:')
    await createTables(db)

    // Sembrar base de datos
    await db.execute('INSERT INTO users (id, name) VALUES (?, ?)', [
      'user-1',
      'Alice',
    ])
    await db.execute(
      'INSERT INTO vouchers (id, userId, amount, used, expiresAt) VALUES (?, ?, ?, ?, datetime("now", "+1 day"))',
      ['v-1', 'user-1', 50, 0]
    )
    await db.execute(
      'INSERT INTO vouchers (id, userId, amount, used, expiresAt) VALUES (?, ?, ?, ?, datetime("now", "+1 day"))',
      ['v-2', 'user-1', 60, 0]
    )
    // cupón vencido
    await db.execute(
      'INSERT INTO vouchers (id, userId, amount, used, expiresAt) VALUES (?, ?, ?, ?, datetime("now", "-1 day"))',
      ['v-3', 'user-1', 100, 0]
    )
    // cupón ya usado
    await db.execute(
      'INSERT INTO vouchers (id, userId, amount, used, expiresAt) VALUES (?, ?, ?, ?, datetime("now", "+1 day"))',
      ['v-4', 'user-1', 100, 1]
    )

    const canPay = await canPayWithVouchers(db, 'user-1', 100)
    assert.equal(canPay, true)

    const cannotPay = await canPayWithVouchers(db, 'user-1', 120)
    assert.equal(cannotPay, false)

    await db.close()
  })
})
```

---

#### Probar una aplicación web

Ahora veamos cómo realizar pruebas de integración de una API HTTP completa construida con **Fastify**.

##### Configuración del proyecto

Definimos el esquema de tablas en `dbSetup.js`:

```javascript
// dbSetup.js
export async function createTables(db) {
  await db.query(`
    CREATE TABLE IF NOT EXISTS events (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      totalSeats INTEGER NOT NULL
    )
  `)

  await db.query(`
    CREATE TABLE IF NOT EXISTS reservations (
      id TEXT PRIMARY KEY,
      eventId TEXT NOT NULL,
      userId TEXT NOT NULL,
      createdAt TEXT NOT NULL,
      FOREIGN KEY (eventId) REFERENCES events(id)
    )
  `)
}
```

Creamos la fábrica de aplicaciones en `app.js`:

```javascript
// app.js
import Fastify from 'fastify'
import { bookEventRoute } from './routes/bookEvent.js'
import { createEventRoute } from './routes/createEvent.js'

export function createApp(db, options = {}) {
  const fastify = Fastify(options)

  fastify.decorate('db', db)

  fastify.register(createEventRoute)
  fastify.register(bookEventRoute)

  return fastify
}
```

Y el servidor en `server.js`:

```javascript
// server.js
import { createApp } from './app.js'
import { DbClient } from './dbClient.js'
import { createTables } from './dbSetup.js'

const db = new DbClient('data/db.sqlite')
await createTables(db)

const app = createApp(db)

try {
  await app.listen({ port: 3000, host: '0.0.0.0' })
  console.log('Server is running at http://localhost:3000')
} catch (err) {
  app.log.error(err)
  process.exit(1)
}
```

##### La lógica de negocio

En `booking.js`, implementamos las funciones para crear eventos y reservar asientos:

```javascript
// booking.js
import { randomUUID } from 'node:crypto'

export async function reserveSeat(db, eventId, userId) {
  const [event] = await db.query('SELECT * FROM events WHERE id = ?', [
    eventId,
  ])
  if (!event) {
    throw new Error('Event not found')
  }

  const reservations = await db.query(
    'SELECT * FROM reservations WHERE eventId = ?',
    [eventId]
  )
  if (reservations.length >= event.totalSeats) {
    throw new Error('Event is sold out')
  }

  const hasAlreadyReserved = reservations.some(r => r.userId === userId)
  if (hasAlreadyReserved) {
    throw new Error('User has already reserved a seat')
  }

  const reservationId = randomUUID()
  await db.execute(
    'INSERT INTO reservations (id, eventId, userId, createdAt) VALUES (?, ?, ?, datetime("now"))',
    [reservationId, eventId, userId]
  )

  return reservationId
}

export async function createEvent(db, name, totalSeats) {
  const eventId = randomUUID()
  await db.execute(
    'INSERT INTO events (id, name, totalSeats) VALUES (?, ?, ?)',
    [eventId, name, totalSeats]
  )
  return eventId
}
```

##### Escribir el código de las rutas

Ruta para crear eventos (`routes/createEvent.js`):

```javascript
// routes/createEvent.js
import { createEvent } from '../booking.js'

export function createEventRoute(fastify) {
  fastify.post('/events', {
    schema: {
      body: {
        type: 'object',
        required: ['name', 'totalSeats'],
        properties: {
          name: { type: 'string' },
          totalSeats: { type: 'integer', minimum: 1 },
        },
      },
    },
    handler: async (request, reply) => {
      const { name, totalSeats } = request.body
      const eventId = await createEvent(fastify.db, name, totalSeats)
      return reply.code(201).send({ success: true, eventId })
    },
  })
}
```

Ruta para reservar eventos (`routes/bookEvent.js`):

```javascript
// routes/bookEvent.js
import { reserveSeat } from '../booking.js'

export function bookEventRoute(fastify) {
  fastify.post('/events/:eventId/reservations', {
    schema: {
      params: {
        type: 'object',
        required: ['eventId'],
        properties: {
          eventId: { type: 'string' },
        },
      },
      body: {
        type: 'object',
        required: ['userId'],
        properties: {
          userId: { type: 'string' },
        },
      },
    },
    handler: async (request, reply) => {
      const { eventId } = request.params
      const { userId } = request.body

      try {
        const reservationId = await reserveSeat(fastify.db, eventId, userId)
        return reply.code(201).send({ success: true, reservationId })
      } catch (err) {
        if (err.message === 'Event not found') {
          return reply.code(404).send({ error: err.message })
        }
        if (
          err.message === 'Event is sold out' ||
          err.message === 'User has already reserved a seat'
        ) {
          return reply.code(409).send({ error: err.message })
        }
        throw err
      }
    },
  })
}
```

##### Pruebas de integración con `inject()`

Una de las características más potentes de Fastify es su método `.inject()`, que permite enviar solicitudes HTTP simuladas directamente a la pila de la aplicación sin tener que vincular un socket TCP real ni escuchar en un puerto de red:

```javascript
// booking.int.test.js
import assert from 'node:assert/strict'
import { suite, test } from 'node:test'
import { createApp } from './app.js'
import { DbClient } from './dbClient.js'
import { createTables } from './dbSetup.js'

suite('events API integration test', () => {
  test('should create an event and allow reservations', async () => {
    const db = new DbClient(':memory:')
    await createTables(db)

    const app = createApp(db)

    // Crear evento
    const createEventResponse = await app.inject({
      method: 'POST',
      url: '/events',
      payload: {
        name: 'Node.js Conf',
        totalSeats: 1,
      },
    })

    assert.equal(createEventResponse.statusCode, 201)
    const { eventId } = createEventResponse.json()
    assert.ok(eventId)

    // Reservar asiento para el usuario 1
    const bookSeat1Response = await app.inject({
      method: 'POST',
      url: `/events/${eventId}/reservations`,
      payload: {
        userId: 'user-1',
      },
    })

    assert.equal(bookSeat1Response.statusCode, 201)
    const { reservationId } = bookSeat1Response.json()
    assert.ok(reservationId)

    // Intentar reservar de nuevo para el usuario 1 (debe fallar con 409)
    const duplicateBookResponse = await app.inject({
      method: 'POST',
      url: `/events/${eventId}/reservations`,
      payload: {
        userId: 'user-1',
      },
    })

    assert.equal(duplicateBookResponse.statusCode, 409)

    // Intentar reservar para el usuario 2 (debe fallar con 409 porque está agotado)
    const soldOutResponse = await app.inject({
      method: 'POST',
      url: `/events/${eventId}/reservations`,
      payload: {
        userId: 'user-2',
      },
    })

    assert.equal(soldOutResponse.statusCode, 409)

    await db.close()
  })
})
```

---

### Sección 6: Escribir pruebas E2E

Las pruebas de extremo a extremo (*End-to-End* o E2E) son el nivel superior de la pirámide de pruebas. Validan la experiencia del usuario de extremo a extremo ejecutando la aplicación completa en un navegador web real o automatizado.

#### La estructura de la aplicación

Consideremos una aplicación web de muestra para reserva de eventos ([nodejsdp.link/events-app](https://nodejsdp.link/events-app)).

El sitio web incluye varias secciones clave:
- **Una página de inicio:** Donde los usuarios pueden explorar los próximos eventos.
- **Una página de inicio de sesión:** Donde los usuarios registrados pueden autenticarse.
- **Una página de registro:** Donde los usuarios que aún no tienen un perfil pueden registrarse.
- **Un panel de control (Dashboard):** Visible solo para los usuarios que han iniciado sesión, listando sus próximas reservas.

##### Página de inicio
*Figura 10.2 – La página de inicio de nuestro sitio web de eventos de muestra.*

##### Formularios de inicio de sesión y registro
*Figura 10.3 – Las páginas de inicio de sesión y registro de nuestro sitio web de eventos de muestra.*

##### Página del evento
*Figura 10.4 – La página de detalles del evento de nuestro sitio web de eventos de muestra.*

##### Mis reservas
*Figura 10.5 – La página de reservas de nuestro sitio web de eventos de muestra.*

#### El flujo de usuario

Queremos verificar el siguiente recorrido crítico del usuario:
1. Un nuevo usuario no registrado llega a la página de inicio y hace clic en el enlace "Sign In".
2. Hace clic en "Sign up" para crear una cuenta y rellena el formulario de registro.
3. Introduce su nombre, correo electrónico y contraseña.
4. Envía el formulario y es redirigido a la página principal.
5. Desde la página de inicio, hace clic en un evento para ver su página de detalles.
6. Hace clic en "Reserve your spot" para reservar un asiento.
7. Ve la confirmación visual de que la reserva fue exitosa.
8. Navega a su página "My Reservations".
9. Confirma que el evento recién reservado aparece allí.

*Figura 10.6 – Un ejemplo de flujo de usuario en nuestro sitio web de eventos de muestra.*

---

#### Automatización del navegador

Para automatizar este flujo, utilizaremos **Playwright**, una biblioteca moderna y potente para la automatización de navegadores desarrollada por Microsoft.

Playwright destaca por:
- Soporte para todos los motores de navegación modernos: Chromium, WebKit (Safari) y Firefox.
- Espera automática inteligente (*Auto-waiting*), lo que reduce drásticamente las pruebas inestables (*flaky tests*).
- Excelente soporte para TypeScript y ricas herramientas de depuración (modo UI, inspector, trazas).

---

#### Escribir una prueba E2E con Playwright

##### Configuración de un nuevo proyecto Playwright

Podemos inicializar Playwright en nuestro proyecto con:

```bash
npm init playwright@^1.51.1
```

O instalar navegadores con:

```bash
npx playwright install
```

Configuración en `playwright.config.ts`:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

/**
 * Read environment variables from file.
 * https://github.com/motdotla/dotenv
 */
// import dotenv from 'dotenv';
// import path from 'path';
// dotenv.config({ path: path.resolve(__dirname, '.env') });

/**
 * See https://playwright.dev/docs/test-configuration.
 */
export default defineConfig({
  testDir: './e2e',
  /* Run tests in files in parallel */
  fullyParallel: true,
  /* Fail the build on CI if you accidentally left test.only in the source code. */
  forbidOnly: !!process.env.CI,
  /* Retry on CI only */
  retries: process.env.CI ? 2 : 0,
  /* Opt out of parallel tests on CI. */
  workers: process.env.CI ? 1 : undefined,
  /* Reporter to use. See https://playwright.dev/docs/test-reporters */
  reporter: 'html',
  /* Shared settings for all the projects below. See https://playwright.dev/docs/api/class-testoptions. */
  use: {
    /* Base URL to use in actions like `await page.goto('/')`. */
    baseURL: 'http://localhost:3000',

    /* Collect trace when retrying the failed test. See https://playwright.dev/docs/trace-viewer */
    trace: 'on-first-retry',
  },

  /* Configure projects for major browsers */
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },

    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },

    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
})
```

Para ejecutar las pruebas:

```bash
npx playwright test
```

O en modo gráfico/interactivo:

```bash
npx playwright test --headed
npx playwright test --ui
```

##### Entender la API de Playwright

###### Navegación

```typescript
await page.goto('https://playwright.dev/')
```

###### La API Locator

Los localizadores (*locators*) encuentran elementos en la página y admiten acciones directas:

```typescript
await page.getByRole('link', { name: 'Get started' }).click()
```

Los localizadores integrados más recomendados se basan en atributos de accesibilidad:
- `page.getByRole()`: Localiza por atributos de accesibilidad explícitos e implícitos.
- `page.getByText()`: Localiza por contenido de texto visible.
- `page.getByLabel()`: Localiza un control de formulario por el texto de su etiqueta asociada.
- `page.getByPlaceholder()`: Localiza un campo de entrada por su texto de marcador de posición.
- `page.getByTitle()`: Localiza un elemento por su atributo `title`.
- `page.getByTestId()`: Localiza por el atributo `data-testid`.

Acciones disponibles en los localizadores:
- `locator.click()`: Hace clic en el elemento.
- `locator.fill()`: Rellena un campo de entrada de texto o formulario.
- `locator.check()`: Marca una casilla de verificación (*checkbox*).
- `locator.uncheck()`: Desmarca una casilla de verificación.
- `locator.hover()`: Pasa el cursor sobre el elemento.
- `locator.focus()`: Enfoca programáticamente el elemento.
- `locator.press()`: Simula la pulsación de una tecla del teclado.
- `locator.setInputFiles()`: Sube un archivo.
- `locator.selectOption()`: Elige una opción de un menú desplegable.

###### Aserciones

Playwright incluye aserciones web que esperan automáticamente hasta que se cumpla la condición especificada:

```typescript
await expect(page).toHaveTitle(/Playwright/)
await expect(locator).toBeVisible()
await expect(locator).toBeEnabled()
await expect(locator).toHaveText('Booked')
```

Principales aserciones web:
- `expect(locator).toBeVisible()`: Espera a que un elemento se vuelva visible.
- `expect(locator).toBeEnabled()`: Comprueba que un control esté habilitado.
- `expect(locator).toBeChecked()`: Verifica que una casilla de verificación esté seleccionada.
- `expect(locator).toHaveText()`: Confirma que un elemento contenga un texto específico.
- `expect(locator).toContainText()`: Coincide con parte del contenido de texto.
- `expect(locator).toHaveAttribute()`: Garantiza que un elemento tenga un atributo específico.
- `expect(locator).toHaveValue()`: Comprueba el valor de un campo de entrada.
- `expect(locator).toHaveCount()`: Verifica el número de elementos coincidentes.
- `expect(page).toHaveTitle()`: Espera a que una página tenga un título específico.
- `expect(page).toHaveURL()`: Confirma la URL de la página actual.

###### Entender los timeouts

En Playwright, diferentes tipos de operaciones manejan los tiempos de espera (*timeouts*) de formas distintas:
- **Aserciones síncronas:** Aserciones directas como `expect(value).toBe(4)` fallan de inmediato si no coinciden.
- **Aserciones web:** Aserciones asíncronas como `await expect(locator).toBeVisible()` reintentan la comprobación continuamente hasta un timeout por defecto de 5 segundos.
- **Acciones:** Acciones como `locator.click()` y `locator.fill()` esperan automáticamente a que el elemento sea visible, interactuable y esté listo antes de actuar.
- **Navegaciones:** Operaciones como `page.goto()` utilizan el timeout de navegación (por defecto 30 segundos).

---

#### Probar nuestro flujo de usuario

Veamos la implementación completa de la prueba E2E para nuestro flujo de reserva:

```typescript
// e2e/userFlow.spec.ts
import { expect, test } from '@playwright/test'
// import { GenericContainer } from 'testcontainers'

test('A user can sign up and book an event', async ({ page }) => {
  // Iniciar el flujo
  await page.goto('http://localhost:3000')

  // Navegar al registro
  await page.getByRole('link', { name: 'Sign In' }).click()
  await page.getByRole('link', { name: 'Sign up' }).click()

  // Rellenar el formulario de registro
  const seed = Date.now().toString()
  const name = `TestUser ${seed}`
  const email = `test${seed}@example.com`
  const password = `someRandomPassword${seed}`

  await page.getByRole('textbox', { name: 'name' }).fill(name)
  await page.getByRole('textbox', { name: 'email' }).fill(email)
  await page.getByRole('textbox', { name: 'password' }).fill(password)
  await page.getByRole('button', { name: 'Create account' }).click()

  // Reservar un evento
  await page.getByRole('link', { name: 'Marathon City Run' }).click()

  const availableCapacity = Number.parseInt(
    (await page.getByTestId('available-capacity').textContent()) as string
  )

  await page.getByRole('button', { name: 'Reserve your spot' }).click()

  // Verificar la reserva
  await expect(page.getByTestId('badge').first()).toHaveText('Booked')

  const bookButton = await page.getByRole('button', {
    name: 'You have booked this event!',
  })
  await expect(bookButton).toBeDisabled()
  await expect(bookButton).toBeVisible()

  const newAvailableCapacity = Number.parseInt(
    (await page.getByTestId('available-capacity').textContent()) as string
  )
  expect(newAvailableCapacity).toBeLessThan(availableCapacity)

  // Comprobar el panel de control
  await page.getByRole('link', { name: 'My Reservations' }).click()

  await expect(
    page.getByRole('heading', { name: 'My Reservations' })
  ).toBeVisible()
  expect(
    await page.getByRole('heading', { name: 'Marathon City Run' })
  ).toBeVisible()
})
```

##### Consideraciones finales

Al cerrar este ejemplo, es importante reconocer que, aunque nuestra prueba fue relativamente sencilla, todavía hace varias suposiciones que podrían no ser ciertas en todos los entornos.

Por ejemplo, nuestra prueba espera encontrar un evento llamado "Marathon City Run" ya listado en la página de inicio. Esto presupone que tal evento existe y está disponible de manera consistente. En situaciones del mundo real, especialmente en entornos dinámicos como producción, este podría no ser el caso: los usuarios pueden agregar o eliminar eventos nuevos y los datos pueden cambiar rápidamente.

Para evitar estos problemas, es común ejecutar pruebas E2E en un entorno dedicado de *staging* o QA. Estos entornos se suelen sembrar con datos de prueba predecibles antes de cada ejecución de pruebas. Esto permite que la suite de pruebas se ejecute contra un conjunto de datos conocido y estable, reduciendo las posibilidades de inestabilidad debido a datos inconsistentes o faltantes.

Otra simplificación que hicimos es en el flujo de registro. Nuestro usuario de prueba puede registrarse y comenzar a reservar de inmediato. La mayoría de las aplicaciones del mundo real requerirán confirmación por correo electrónico antes de activar una nueva cuenta. Esto introduce una nueva complejidad en las pruebas E2E: ¿deberíamos simular el flujo de correo electrónico utilizando una bandeja de entrada de correo electrónico real, interceptar correos electrónicos en un servicio de prueba como MailHog ([nodejsdp.link/mailhog](https://nodejsdp.link/mailhog)), o eludir el paso de verificación por completo en el entorno de prueba? Estas decisiones requieren coordinación entre desarrolladores, evaluadores y equipos de producto.

También habrás notado que los eventos en nuestra aplicación tienen un precio, pero no procesamos ningún pago durante la reserva. Esta fue una elección deliberada para simplificar las cosas, pero en un escenario del mundo real, probablemente necesitarías integrarte con un procesador de pagos como Stripe o PayPal. Probar pagos en flujos E2E puede ser un desafío. Un enfoque consiste en utilizar entornos *sandbox* proporcionados por estos servicios, que simulan transacciones reales sin mover dinero real. Otra opción es mockear o eludir el paso de pago por completo en tu entorno de prueba para aislar la lógica de negocio de la integración de terceros.

Algunas empresas van un paso más allá. Uno de los autores (Luciano) trabajó anteriormente en una empresa que ejecutaba pruebas E2E realizando pagos reales para garantizar una confianza absoluta de que el flujo de pagos funcionaba en condiciones idénticas a las de producción. Utilizaban una tarjeta de crédito corporativa para procesar pequeñas transacciones contra su propia cuenta comercial. Dado que el dinero permanecía dentro de la misma organización, solo se perdían tarifas de transacción mínimas. Sin embargo, esto requería un seguimiento cuidadoso y una coordinación estrecha con el equipo de finanzas para conciliar adecuadamente esas transacciones, especialmente a efectos contables y fiscales.

Lo mismo ocurre con los sistemas de inicio de sesión social, como "Sign in with Google" o "Login with GitHub". Estos pueden ser difíciles de probar de manera confiable en escenarios E2E, ya que involucran redireccionamientos, flujos de interfaz externos y, a veces, incluso desafíos CAPTCHA. En entornos de prueba, es común sustituir (*stub*) o simular (*mock*) estos flujos de inicio de sesión con un usuario de prueba predecible para evitar depender de servicios externos.

---

### Sección 7: Resumen

En este capítulo, nos sumergimos de lleno en los principios, patrones y herramientas fundamentales del testing de software en Node.js:

- Comenzamos estableciendo las **definiciones clave** del testing: el Sistema Bajo Prueba (**SUT**), la metodología **Arrange-Act-Assert (AAA)**, las métricas y límites de la **cobertura de código**, y el rol de los dobles de prueba (**stubs**, **spies** y **mocks**).
- Exploramos las metodologías de desarrollo como **TDD** (Rojo-Verde-Refactorización) y **BDD** (Dado-Cuando-Entonces), así como la automatización continua mediante **CI/CD**.
- Analizamos la **pirámide de pruebas**, aprendiendo a equilibrar una base sólida de pruebas unitarias rápidas y económicas con pruebas de integración y un conjunto enfocado de pruebas de extremo a extremo (**E2E**).
- Conocimos el **test runner integrado de Node.js** (`node:test` y `node:assert/strict`), aprovechando sus funciones de subtests, concurrencia, pruebas parametrizadas, suites de pruebas, modo watch, filtrado por nombres/patrones, reporteros personalizables y soporte nativo para **TypeScript**.
- Vimos cómo escribir **pruebas unitarias** para código asíncrono y exploramos técnicas de **mocking**, contrastando el mocking de módulos con el uso superior de la **Inyección de Dependencias (DI)**.
- Implementamos **pruebas de integración** conectándonos a una base de datos local SQLite y probando APIs web de **Fastify** sin sobrecarga de red usando `inject()`.
- Finalmente, construimos **pruebas E2E** completas con **Playwright**, automatizando la navegación, la interacción y las aserciones web en flujos de usuario del mundo real.

Con estas herramientas y patrones en tu haber, estás listo para escribir software en Node.js que no solo esté bien diseñado, sino que sea demostrablemente robusto, confiable y mantenible.

---

### Sección 8: Ejercicios

- **10.1 Pruebas unitarias para una cola de tareas con reintentos:** Amplía la clase `TaskQueue` que vimos en la Sección 4 para que admita una opción de reintentos automáticos (`retries`) en caso de que una tarea falle. Escribe una suite de pruebas unitarias exhaustiva con `node:test` que verifique: (1) que una tarea que falla se reintente el número especificado de veces, (2) que si la tarea tiene éxito en un reintento posterior, la promesa se resuelva correctamente, y (3) que si se agotan todos los reintentos, la promesa se rechace con el último error. Utiliza espías con `mock.fn()` para contar con precisión el número de ejecuciones de cada tarea.
- **10.2 Mocking de una API de tipos de cambio con Undici:** Imagina una función `convertCurrency(amount, from, to)` que realiza una solicitud HTTP a un servicio externo de tipos de cambio (por ejemplo, `https://api.exchangerate.host/convert?from=USD&to=EUR&amount=100`) utilizando `fetch`. Escribe una suite de pruebas unitarias utilizando `MockAgent` de `undici` para: (1) simular una respuesta exitosa con una tasa de conversión específica, (2) simular una respuesta de error del servidor (código de estado 500) y verificar que la función maneje el error adecuadamente, y (3) verificar que no se realicen conexiones de red reales durante la ejecución de las pruebas (`disableNetConnect()`).
- **10.3 Pruebas de integración para cancelación de reservas:** Utilizando el proyecto de la aplicación de eventos con Fastify y SQLite de la Sección 5, agrega una nueva ruta `DELETE /events/:eventId/reservations/:reservationId` que permita a un usuario cancelar su reserva previa. Escribe una suite de pruebas de integración con `app.inject()` que valide: (1) la cancelación exitosa de una reserva existente, (2) que un usuario que canceló su reserva pueda volver a reservar el evento, (3) que si el evento estaba agotado, la cancelación libere un asiento para otros usuarios, y (4) que intentar cancelar una reserva inexistente devuelva un código de estado 404.
- **10.4 Prueba E2E para el flujo de cancelación de eventos:** En el proyecto de pruebas E2E con Playwright de la Sección 6, añade una nueva especificación de prueba E2E que automatice el flujo completo de cancelación: un usuario inicia sesión, reserva un evento, navega a su panel "My Reservations", hace clic en el botón "Cancel reservation", confirma el diálogo de confirmación y verifica visualmente que el evento ya no figure en su lista de reservas activas y que la capacidad disponible del evento se incremente correspondientemente.
