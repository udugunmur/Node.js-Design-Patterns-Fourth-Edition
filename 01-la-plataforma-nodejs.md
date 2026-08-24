# Parte 1: Fundamentos de Node.js

## Capítulo 1: La plataforma Node.js

Durante la última década, Node.js se ha convertido en una piedra angular del desarrollo web moderno, revolucionando la forma en que las empresas de todos los tamaños abordan la creación de aplicaciones escalables y de alto rendimiento. Su capacidad para gestionar operaciones asíncronas permite soluciones en tiempo real y con gran capacidad de respuesta que pueden administrar grandes cantidades de conexiones concurrentes de manera eficiente. Node.js ofrece un entorno unificado donde JavaScript puede ejecutarse sin problemas tanto en el cliente como en el servidor, optimizando el proceso de desarrollo.

Dominar Node.js va más allá de aprender su sintaxis y bibliotecas; de hecho, algunas ideas y patrones clave moldean la forma en que los desarrolladores utilizan Node.js y su ecosistema. Su naturaleza asíncrona significa que debemos adoptar primitivas asíncronas como callbacks, promesas y async/await, las cuales conllevan sus propios desafíos y patrones únicos. En este primer capítulo, analizaremos por qué Node.js funciona de esta manera. Esto no es meramente teórico: comprender cómo funciona Node.js en su núcleo te brindará una base sólida para entender el razonamiento detrás de los temas y patrones más complejos que cubriremos más adelante en el libro.

Otro aspecto determinante de Node.js es su filosofía con opiniones firmes (*opinionated*). Adoptar Node.js significa unirse a una cultura y comunidad que influyen profundamente en cómo diseñamos aplicaciones e interactuamos con el ecosistema en general.

En este capítulo descubrirás:
- La filosofía de Node.js o el **"estilo Node"** (*the Node way*).
- El patrón Reactor: el mecanismo central de la arquitectura asíncrona orientada a eventos de Node.js.
- Lo que significa ejecutar JavaScript en el servidor en comparación con el navegador.

Para destacar la importancia de esta filosofía, permíteme compartir un poco de mi trayectoria (la de Luciano) con Node.js. Como desarrollador web, tenía una sólida formación en JavaScript para frontend (¡sí, mucho jQuery!), por lo que cuando descubrí Node.js —en aquel momento en su versión 0.12— me entusiasmó usar JavaScript en el backend y reemplazar los otros lenguajes con los que trabajaba, como PHP, Java y .NET.

Comencé a aprender Node.js con la primera edición de este mismo libro, que Mario había escrito en solitario. Tras terminar el libro, estaba impaciente por construir algo, así que emprendí un proyecto personal: descargar una galería de fotos completa de Flickr (un popular sitio de alojamiento de fotos en ese momento). Flickr no ofrecía una forma de descargar todas las fotos de una sola vez, así que decidí utilizar su API y mis nuevas habilidades en Node.js para crear una herramienta CLI que pudiera hacer precisamente eso.

Este proyecto fue la oportunidad perfecta para aprovechar la naturaleza asíncrona de Node.js. Descargar cientos de archivos desde URLs es ideal para la concurrencia: no hay razón para obtenerlos uno por uno cuando puedes descargar varios al mismo tiempo. Sin embargo, hacer esto eficazmente requería limitar la concurrencia para evitar saturar los recursos del sistema, como la memoria y el ancho de banda de red. Tras construir la herramienta, compartí el proyecto en GitHub ([nodejsdp.link/flickr-set-get](https://nodejsdp.link/flickr-set-get)) y pedí retroalimentación a varias comunidades de Node.js. ¡Y recibí muchísimos comentarios! Muchas personas señalaron que, aunque mi código funcionaba, no adoptaba plenamente el "estilo Node". Mi enfoque aún conservaba vestigios de un estilo propio de PHP, lo que dificultaba su integración fluida con el ecosistema más amplio de Node.js.

Esa retroalimentación fue invaluable: me ayudó a mejorar mis habilidades en Node.js y a comprender la importancia de adoptar sus principios de diseño. Y aquí hay un giro curioso: ¡Mario fue una de las personas que me dio retroalimentación sobre ese proyecto! Esa interacción inició una conexión entre nosotros que, con el tiempo, me llevó a unirme a él como coautor para la segunda edición del libro unos años más tarde.

Por lo tanto, quizás la lección sea no tener miedo de compartir tu trabajo y pedir retroalimentación sincera. La comunidad de Node.js es muy comprensiva y servicial, y ofrece innumerables oportunidades para aprender y crecer. ¡Quién sabe a dónde pueden llevar esas oportunidades!

Basta de historias motivacionales: sumerjámonos en el aprendizaje.

---

### Sección 1: La filosofía de Node.js

Cada plataforma de programación posee su propia filosofía, un conjunto de principios y directrices generalmente aceptados por la comunidad, o una ideología de hacer las cosas que influye tanto en la evolución de la plataforma como en la forma en que se desarrollan y diseñan las aplicaciones. Algunos de estos principios surgen de la propia tecnología, otros son habilitados por su ecosistema, algunos son tendencias en la comunidad y otros son evoluciones de ideologías tomadas de otras plataformas. En Node.js, algunos de estos principios provienen directamente de su creador —Ryan Dahl—, mientras que otros proceden de quienes contribuyen al núcleo (*core*) o de figuras carismáticas de la comunidad y, finalmente, algunos son heredados del movimiento más amplio de JavaScript.

Estas pautas no son reglas estrictas y deben usarse con pragmatismo. No obstante, pueden ser muy útiles cuando buscas inspiración para diseñar tu software.

Si tienes curiosidad por ver otros ejemplos de filosofías de desarrollo de software, puedes encontrar una lista extensa en Wikipedia en [nodejsdp.link/dev-philosophies](https://nodejsdp.link/dev-philosophies).

#### Núcleo pequeño (*Small core*)

Históricamente, el núcleo de Node.js, que incluye el entorno de ejecución (*runtime*) y los módulos integrados, se ha mantenido mínimo, dejando la mayoría de las funcionalidades al "espacio de usuario" (*userland* o *userspace*): el ecosistema de módulos fuera del núcleo. Este enfoque ha permitido a la comunidad experimentar y desarrollar nuevas soluciones con rapidez, en lugar de depender de una única solución central de evolución lenta. Mantener el núcleo mínimo facilita su mantenimiento y genera un impacto positivo en la comunidad al fomentar la innovación en los módulos del espacio de usuario. En los últimos años, el principio de mantener el núcleo de Node.js minimalista se ha vuelto menos estricto. La comunidad ha mostrado interés en contar con más funciones integradas, por lo que se han añadido varias capacidades importantes directamente al núcleo. Estas incluyen análisis de argumentos de línea de comandos, WebSockets, un framework de pruebas unitarias, capacidad de observación de archivos (*file watch*), coincidencia de patrones de archivos (*file globbing*), la API web Fetch y más. Este cambio no altera los principios fundamentales de la comunidad de Node.js, sino que refleja su evolución. Muchas interfaces comunes han madurado y se han vuelto estables, por lo que tiene sentido incluirlas en el núcleo para un acceso sencillo sin necesidad de bibliotecas de terceros.

#### Módulos pequeños (*Small modules*)

Node.js utiliza el concepto de módulo como el medio fundamental para estructurar el código de un programa. Es el bloque de construcción para crear aplicaciones y bibliotecas reutilizables. En Node.js, uno de los principios más promovidos es el diseño de módulos (y paquetes) pequeños, no solo en términos de tamaño bruto de código, sino también, y más importante, en términos de alcance (*scope*).

Este principio tiene sus raíces en la filosofía Unix, y en particular en dos de sus preceptos, que son los siguientes:
- "Lo pequeño es hermoso" (*Small is beautiful*).
- "Haz que cada programa haga una sola cosa y la haga bien" (*Make each program do one thing well*).

Node.js ha llevado estos conceptos a un nivel completamente nuevo. Junto con la ayuda de sus gestores de módulos —siendo npm, pnpm y yarn los más populares—, Node.js ayuda a resolver el problema del "infierno de dependencias" (*dependency hell*) asegurándose de que dos (o más) paquetes que dependen de diferentes versiones del mismo paquete utilicen sus propias instalaciones de dicho paquete, evitando así conflictos. Este aspecto permite que los paquetes dependan de un gran número de dependencias pequeñas y bien enfocadas sin riesgo de crear conflictos.

Por ejemplo, supongamos que tenemos un proyecto que utiliza dos dependencias: `depA` y `depB`. Tanto `depA` como `depB` dependen de una tercera biblioteca, `depC`, pero necesitan versiones diferentes: `depA` requiere `depC@1.0.0`, mientras que `depB` requiere `depC@2.0.0`. Node.js maneja esto sin conflictos organizando las dependencias de la siguiente forma:

```text
.
└── node_modules
    ├── depA@1.0.0
    │   └── node_modules
    │       └── depC@1.0.0
    └── depB@1.0.0
        └── node_modules
            └── depC@2.0.0
```

> **Consejo rápido:** Mejora tu experiencia de programación con las funciones AI Code Explainer y Quick Copy. Abre este libro en el lector de última generación de Packt (*next-gen Packt Reader*). Haz clic en el botón Copy (1) para copiar código rápidamente en tu entorno de desarrollo, o haz clic en el botón Explain (2) para que el asistente de IA te explique un bloque de código.
>
> El lector Packt de última generación está incluido de forma gratuita con la compra de este libro. Visita [https://packtpub.com/unlock](https://packtpub.com/unlock), luego utiliza la barra de búsqueda para encontrar este libro por su nombre. Verifica bien la edición mostrada para asegurarte de obtener la correcta.

Aquí se instalan dos versiones de `depC`: una bajo `depA` para la versión 1.0.0 y otra bajo `depB` para la versión 2.0.0. De este modo, Node.js garantiza que cuando `depA` necesita `depC`, cargue la versión 1.0.0, y cuando `depB` necesita `depC`, cargue la versión 2.0.0, evitando por completo los conflictos de versiones.

Aunque esto podría considerarse poco práctico o incluso inviable en otras plataformas, en Node.js esta práctica es la norma. Esto permite niveles extremos de reutilización; son tan extremos, de hecho, que a veces podemos encontrar paquetes compuestos por un solo módulo que contiene apenas unas pocas líneas de código —por ejemplo, `is-sorted`, una biblioteca para verificar si un array está ordenado ([nodejsdp.link/is-sorted](https://nodejsdp.link/is-sorted))—.

Además de la clara ventaja en términos de reutilización, un módulo pequeño es también:
- Más fácil de entender y usar.
- Más simple de probar y mantener.
- Ligero en términos de kilobytes, ideal en el navegador (dado que el registro npm es utilizado tanto por Node.js como por aplicaciones frontend) y en entornos serverless que requieren tiempos de inicio rápidos, como AWS Lambda.

Tener módulos más pequeños y enfocados permite que cualquiera pueda compartir o reutilizar hasta la pieza más pequeña de código; es el principio *Don’t Repeat Yourself* (DRY) aplicado a un nivel totalmente nuevo.

Si bien se aplica el principio de módulos pequeños, el reciente aumento de vulnerabilidades en la cadena de suministro ([nodejsdp.link/supply-chain](https://nodejsdp.link/supply-chain)) ha hecho que la industria del software sea más cautelosa a la hora de añadir dependencias de terceros. Esto también se aplica a los proyectos de Node.js. Es fundamental evaluar cuidadosamente si un módulo de terceros tiene un buen mantenimiento y si es realmente necesario añadir una nueva dependencia. Cuantas más dependencias de terceros existan, mayor será el riesgo de que una de ellas se vea comprometida y afecte a la seguridad del proyecto.

#### Superficie reducida (*Small surface area*)

Además de ser pequeños en tamaño y alcance, una característica deseable de los módulos de Node.js es exponer un conjunto mínimo de funcionalidades al mundo exterior. Esto produce una API que resulta más clara de usar y menos propensa a usos erróneos. De hecho, la mayoría de las veces, el usuario de un componente solo está interesado en un conjunto muy limitado y específico de funciones, sin necesidad de extender su funcionalidad o recurrir a aspectos más avanzados.

En Node.js, un patrón muy común para definir módulos es exponer solo una funcionalidad, como una función o una clase, por el simple hecho de que proporciona un punto de entrada único e inequívocamente claro.

Otra característica de muchos módulos de Node.js es que están diseñados para ser utilizados, no extendidos. Bloquear el interior de un módulo impidiendo extensiones puede parecer inflexible, pero simplifica la implementación, facilita el mantenimiento y mejora la usabilidad. En la práctica, esto se traduce en preferir la exposición de funciones en lugar de clases y asegurarse de no exponer componentes internos al exterior.

#### Simplicidad y pragmatismo (*Simplicity and pragmatism*)

¿Has oído hablar alguna vez del principio *Keep It Simple, Stupid* (KISS)? Richard P. Gabriel, un destacado científico de la computación, acuñó el término "peor es mejor" (*worse is better*) para describir el modelo según el cual una funcionalidad menor y más simple constituye una buena elección de diseño de software. En su ensayo *The Rise of “Worse is Better”*, afirma:

> "El diseño debe ser simple, tanto en la implementación como en la interfaz. Es más importante que la implementación sea simple a que lo sea la interfaz. La simplicidad es la consideración más importante en un diseño."

Diseñar software simple, en contraposición a software perfecto y repleto de funciones, es una buena práctica por varias razones:
- Requiere menos esfuerzo de implementación.
- Permite lanzar productos con mayor rapidez y con menos recursos.
- Es más fácil de adaptar.
- Es más fácil de mantener y comprender.

Los efectos positivos de estos factores fomentan las contribuciones de la comunidad y permiten que el software crezca y mejore.

En Node.js, este principio está respaldado por la naturaleza pragmática de JavaScript. En lugar de utilizar complejas jerarquías de clases, es habitual ver clases simples, funciones y clausuras (*closures*). Los diseños puramente orientados a objetos suelen intentar modelar el mundo real utilizando conceptos matemáticos, lo que puede pasar por alto las imperfecciones y complejidades del mundo real. En realidad, el software es siempre una aproximación de la realidad. Es más probable que tengamos éxito si nos enfocamos en conseguir algo funcional rápidamente con una complejidad manejable, en lugar de esforzarnos por abstracciones de software casi perfectas que requieran un esfuerzo descomunal y toneladas de código para mantenerse.

A lo largo de este libro, verás este principio aplicado a menudo. Por ejemplo, los patrones de diseño tradicionales como Singleton o Decorator pueden implementarse de forma sencilla, lo que tal vez no sea perfecto pero suele ser sumamente práctico.

En Node.js preferimos soluciones directas y prácticas en lugar de diseños complejos e impecables. Esto no significa que rebajemos nuestros estándares; más bien, sopesamos cuidadosamente las ventajas y desventajas entre una cobertura exhaustiva y la complejidad frente a la simplicidad y los límites claros.

A continuación, echaremos un vistazo al interior del núcleo de Node.js para revelar sus patrones internos y su arquitectura orientada a eventos.

---

### Sección 2: Cómo funciona Node.js

En esta sección aprenderás cómo opera Node.js internamente y conocerás el patrón Reactor, fundamental para la naturaleza asíncrona de Node.js. Cubriremos conceptos clave como la arquitectura de un solo hilo (*single-threaded*) y la E/S no bloqueante (*non-blocking I/O*), y mostraremos cómo estos elementos forman la base de la plataforma Node.js. Comprender el funcionamiento de Node.js será crucial para dominar el entorno de ejecución y escribir código limpio y eficiente.

Aunque comúnmente describimos a Node.js como de "un solo hilo" debido a su capacidad para gestionar tareas asíncronas concurrentemente en un único hilo, esto no significa que no pueda aprovechar hilos en segundo plano para ciertas operaciones. Node.js utiliza un solo hilo para el bucle de eventos (*event loop*), pero cuando es necesario, puede ejecutar tareas intensivas en CPU en hilos separados, permitiendo una gestión más eficiente de operaciones complejas. En el [Capítulo 11](https://subscription.packtpub.com/book/web-development/9781803238944/11), Recetas Avanzadas (*Advanced Recipes*), exploraremos cómo usar *worker threads* para realizar tales tareas pesadas de CPU en paralelo, demostrando cómo Node.js puede ir más allá de su naturaleza monohilo cuando es necesario.

#### La E/S suele ser el cuello de botella (*I/O is often the bottleneck*)

Independientemente del lenguaje de programación que elijas, la E/S (abreviatura de entrada/salida) es definitivamente la más lenta entre las operaciones fundamentales de un ordenador. El acceso a la memoria RAM es del orden de nanosegundos ($10^{-9}$ segundos), mientras que el acceso a datos en el disco o la red es del orden de milisegundos ($10^{-3}$ segundos). Lo mismo ocurre con el ancho de banda. La memoria RAM tiene una tasa de transferencia sistemáticamente del orden de GB/s, mientras que el disco o la red varían desde MB/s hasta, con optimismo, GB/s. Por lo general, la E/S no es costosa en términos de CPU, pero añade un retraso entre el momento en que se envía la solicitud al dispositivo y el momento en que se completa la operación.

Además de esto, debemos considerar el factor humano. De hecho, en muchas circunstancias, la entrada de una aplicación proviene de una persona real —un clic de ratón, por ejemplo—, por lo que la velocidad y la frecuencia de la E/S no solo dependen de aspectos técnicos, y pueden ser muchos órdenes de magnitud más lentas que el disco o la red.

A pesar de la lentitud inherente a la E/S, Node.js está diseñado para gestionarla con notable eficiencia. Su arquitectura no bloqueante y orientada a eventos permite que el sistema permanezca receptivo incluso mientras espera a que se completen las operaciones de E/S, lo que lo convierte en una excelente opción para aplicaciones que necesitan realizar grandes volúmenes de E/S sin sacrificar el rendimiento. Pero para comprender cómo funciona el modelo no bloqueante de Node.js, debemos explorar brevemente el concepto de E/S bloqueante.

#### E/S bloqueante (*Blocking I/O*)

En la programación tradicional con E/S bloqueante, la llamada a la función correspondiente a una solicitud de E/S bloqueará la ejecución del hilo hasta que se complete la operación. Esto puede variar desde unos pocos milisegundos, en el caso del acceso al disco, hasta minutos o incluso más, en el caso de datos generados a partir de acciones del usuario, como presionar una tecla. El siguiente pseudocódigo muestra un hilo bloqueante típico ejecutado contra un socket:

```javascript
// blocks the thread until the data is available
data = socket.read()
// data is available
print(data)
```

Es sumamente importante comprender que un servidor web implementado mediante E/S bloqueante no podrá gestionar múltiples conexiones en el mismo hilo. Esto se debe a que cada operación de E/S en un socket bloqueará el procesamiento de cualquier otra conexión. El enfoque tradicional para resolver este problema consiste en utilizar un hilo (o proceso) independiente para gestionar cada conexión concurrente. De este modo, un hilo bloqueado en una operación de E/S no afectará a la disponibilidad de las demás conexiones, ya que se gestionan en hilos independientes.

El siguiente diagrama ilustra este escenario:

**Figura 1.1:** Uso de múltiples hilos para procesar múltiples conexiones.

La Figura 1.1 enfatiza la cantidad de tiempo que cada hilo pasa inactivo y esperando recibir nuevos datos de la conexión asociada. Ahora bien, si también consideramos que cualquier tipo de E/S puede llegar a bloquear una petición —por ejemplo, al interactuar con bases de datos o con el sistema de archivos—, pronto nos daremos cuenta de cuántas veces debe bloquearse un hilo para esperar el resultado de una operación de E/S. Desafortunadamente, un hilo no es económico en términos de recursos del sistema —consume memoria y provoca cambios de contexto—, por lo que mantener un hilo de larga duración para cada conexión y no usarlo durante la mayor parte del tiempo implica desperdiciar valiosos ciclos de memoria y CPU.

La conclusión clave aquí es que lograr concurrencia con un enfoque bloqueante tradicional suele requerir múltiples hilos, los cuales son costosos en recursos del sistema. Por el contrario, Node.js utiliza un enfoque no bloqueante y monohilo que puede ser mucho más eficiente al manejar operaciones de E/S.

Para profundizar en los conceptos tratados, recomendamos consultar esta página de Wikipedia sobre hilos de computación: [nodejsdp.link/thread](https://nodejsdp.link/thread).

#### E/S no bloqueante (*Non-blocking I/O*)

Además de la E/S bloqueante, la mayoría de los sistemas operativos modernos admiten otro mecanismo para acceder a los recursos, denominado E/S no bloqueante. En este modo de operación, la llamada al sistema siempre retorna de inmediato sin esperar a que los datos sean leídos o escritos. Si no hay resultados disponibles en el momento de la llamada, la función simplemente devolverá una constante predefinida, indicando que no hay datos disponibles para retornar en ese instante.

Por ejemplo, en los sistemas operativos Unix, la función `fcntl()` se utiliza para modificar un descriptor de archivo existente (que es una referencia a un archivo local o un socket de red) para cambiar su modo de operación a no bloqueante mediante el indicador `O_NONBLOCK`. Cuando un recurso está en modo no bloqueante, cualquier operación de lectura devolverá el código de error `EAGAIN` si no hay datos listos para ser leídos.

La forma más sencilla de gestionar la E/S no bloqueante es verificar activamente el recurso en un bucle hasta que haya datos disponibles, método conocido como espera activa (*busy-waiting* o *polling* activo). El siguiente pseudocódigo demuestra cómo leer de múltiples recursos utilizando E/S no bloqueante y un bucle de sondeo activo:

```javascript
resources = [socketA, socketB, fileA]
while (!resources.isEmpty()) {
  for (resource of resources) {
    // try to read
    data = resource.read()
    if (data === NO_DATA_AVAILABLE) {
      // there is no data to read at the moment
      continue
    }
    if (data === RESOURCE_CLOSED) {
      // the resource was closed, remove it from the list
      resources.remove(i)
    } else {
      //some data was received, process it
      consumeData(data)
    }
  }
}
```

Como puedes ver, con esta sencilla técnica es posible gestionar diferentes recursos en el mismo hilo, pero sigue sin ser eficiente. De hecho, en el ejemplo anterior, el bucle consumirá preciosos ciclos de CPU para iterar sobre recursos que no están disponibles la mayor parte del tiempo. Los algoritmos de sondeo (*polling*) suelen derivar en una enorme cantidad de tiempo de CPU desperdiciado. Veamos cómo podemos implementar la E/S no bloqueante de una forma más eficiente.

#### Demultiplexación de eventos (*Event demultiplexing*)

La espera activa (*busy-waiting*) definitivamente no es una técnica ideal para procesar recursos no bloqueantes, pero afortunadamente la mayoría de los sistemas operativos modernos proporcionan un mecanismo nativo para gestionar recursos no bloqueantes concurrentes de manera eficiente. Nos referimos al demultiplexor de eventos síncrono (*synchronous event demultiplexer*, también conocido como interfaz de notificación de eventos).

Si no estás familiarizado con el término, en telecomunicaciones, la multiplexación se refiere al método mediante el cual múltiples señales se combinan en una sola para que puedan transmitirse fácilmente a través de un medio con capacidad limitada.

La demultiplexación se refiere a la operación contraria, por la cual la señal se divide nuevamente en sus componentes originales. Ambos términos se utilizan en otras áreas (por ejemplo, el procesamiento de vídeo) para describir la operación general de combinar diferentes elementos en uno, y viceversa.

Este tipo de demultiplexor de eventos monitoriza múltiples recursos y genera un evento (o un conjunto de eventos) cuando se completa una operación de lectura o escritura en uno de esos recursos. La ventaja es que el demultiplexor de eventos síncrono se bloquea hasta que haya nuevos eventos que procesar. El siguiente pseudocódigo muestra un algoritmo que utiliza un demultiplexor de eventos síncrono genérico para leer de dos recursos diferentes:

```javascript
watchList.add(socketA, FOR_READ) // 1
watchList.add(fileB, FOR_READ)
while (events = demultiplexer.watch(watchList)) { // 2
  // event loop
  for (event of events) { // 3
    // This read will never block and will always return data
    data = event.resource.read()
    if (data === RESOURCE_CLOSED) {
      // the resource was closed, remove it from the watched list
      demultiplexer.unwatch(event.resource)
    } else {
      // some actual data was received, process it
      consumeData(data)
    }
  }
}
```

Veamos qué sucede en el pseudocódigo anterior:
1. Los recursos se añaden a una estructura de datos, asociando cada uno de ellos a una operación específica (en nuestro ejemplo, una lectura).
2. El demultiplexor se configura con el grupo de recursos que se van a observar. La llamada a `demultiplexer.watch()` es síncrona y se bloquea hasta que cualquiera de los recursos observados esté listo para lectura. Cuando esto ocurre, el demultiplexor de eventos retorna de la llamada y un nuevo conjunto de eventos queda disponible para ser procesado.
3. Se procesa cada evento devuelto por el demultiplexor de eventos. En este punto, se garantiza que el recurso asociado a cada evento está listo para leer y no se bloqueará durante la operación. Cuando se procesan todos los eventos, el flujo se bloqueará nuevamente en el demultiplexor de eventos hasta que haya nuevos eventos disponibles para procesar. Esto se denomina el **bucle de eventos** (*event loop*).

Es interesante observar que, con este patrón, ahora podemos gestionar varias operaciones de E/S dentro de un solo hilo, sin utilizar la técnica de espera activa. Ahora debería quedar claro por qué hablamos de demultiplexación: utilizando un solo hilo podemos manejar múltiples recursos. La Figura 1.2 te ayudará a visualizar lo que sucede en un servidor web que utiliza un demultiplexor de eventos síncrono y un solo hilo para gestionar múltiples conexiones concurrentes:

**Figura 1.2:** Uso de un solo hilo para procesar múltiples conexiones.

Como se muestra aquí, el uso de un solo hilo no perjudica nuestra capacidad de ejecutar múltiples tareas limitadas por E/S (*I/O-bound*) de forma concurrente. Las tareas se distribuyen en el tiempo, en lugar de distribuirse entre múltiples hilos. Esto tiene la clara ventaja de minimizar el tiempo de inactividad total del hilo, como se muestra claramente en la Figura 1.2.

Pero esta no es la única razón para elegir este modelo de E/S. De hecho, tener un solo hilo también influye de forma beneficiosa en la manera en que los programadores abordan la concurrencia en general. A lo largo del libro verás cómo la ausencia de condiciones de carrera intra-proceso (*in-process race conditions*) y de múltiples hilos que sincronizar nos permite utilizar estrategias de concurrencia mucho más simples.

#### El patrón Reactor (*The reactor pattern*)

Ahora podemos presentar el patrón Reactor, que es una variación de los algoritmos presentados en las secciones anteriores y lo que Node.js utiliza bajo el capó. La idea principal detrás del patrón Reactor es tener un manejador (*handler*) asociado a cada operación de E/S. Un manejador en Node.js está representado por una función callback (o cb para abreviar).

El manejador será invocado tan pronto como el bucle de eventos produzca y procese un evento. La estructura del patrón Reactor se muestra en la Figura 1.3:

**Figura 1.3:** El patrón Reactor.

> **Consejo rápido:** ¿Necesitas ver una versión en alta resolución de esta imagen? Abre este libro en el lector de última generación de Packt o míralo en la copia PDF/ePub.
>
> El lector de última generación de Packt y una copia gratuita en PDF/ePub de este libro están incluidos con tu compra. Visita [https://packtpub.com/unlock](https://packtpub.com/unlock), luego usa la barra de búsqueda para encontrar este libro por su nombre. Verifica la edición mostrada para asegurarte de obtener la correcta.

Esto es lo que sucede en una aplicación que utiliza el patrón Reactor:
1. La aplicación genera una nueva operación de E/S enviando una solicitud al **Demultiplexor de Eventos** (*Event Demultiplexer*). La aplicación también especifica un manejador, que será invocado cuando la operación se complete. Enviar una nueva solicitud al Demultiplexor de Eventos es una llamada no bloqueante y devuelve inmediatamente el control a la aplicación.
2. Cuando se completa un conjunto de operaciones de E/S, el Demultiplexor de Eventos inserta un conjunto de eventos correspondientes en la **Cola de Eventos** (*Event Queue*).
3. En este punto, el **Bucle de Eventos** (*Event Loop*) itera sobre los elementos de la Cola de Eventos.
4. Para cada evento, se invoca el manejador asociado.
5. El manejador, que es parte del código de la aplicación, devuelve el control al Bucle de Eventos cuando finaliza su ejecución (**5a**). Mientras el manejador se ejecuta, puede solicitar nuevas operaciones asíncronas (**5b**), lo que provoca que se añadan nuevos elementos al Demultiplexor de Eventos (**1**).
6. Cuando se procesan todos los elementos de la Cola de Eventos, el Bucle de Eventos se bloquea nuevamente en el Demultiplexor de Eventos, que luego activa otro ciclo cuando haya un nuevo evento disponible.

El comportamiento asíncrono ahora ha quedado claro. La aplicación expresa interés en acceder a un recurso en un momento determinado (sin bloquearse) y proporciona un manejador, que luego será invocado en otro momento cuando la operación se complete.

Una aplicación Node.js finaliza cuando el bucle de eventos determina que no quedan operaciones ni eventos pendientes por procesar. Específicamente, cuando ya no hay más manejadores (*handles*) ni solicitudes activas (como temporizadores, sockets abiertos u operaciones del sistema de archivos) dentro del demultiplexor de eventos, el bucle de eventos se detiene. Sin embargo, ciertos recursos como un servidor HTTP activo, sockets de red abiertos u operaciones de E/S pendientes mantienen vivo el bucle de eventos registrando eventos continuamente. Por ejemplo, en el caso de un servidor HTTP, la llamada `http.createServer()` crea un servidor que escucha en un puerto específico. El bucle de eventos considera este socket de escucha como un manejador activo. Mientras el servidor esté escuchando, Node.js mantiene este socket para aceptar conexiones entrantes, lo que mantiene el proceso en ejecución. El bucle de eventos solo finalizará cuando este socket se cierre explícitamente y no queden otros manejadores activos en el sistema.

Finalmente podemos definir el patrón Reactor, el patrón en el corazón de Node.js: maneja la E/S bloqueándose hasta que haya nuevos eventos disponibles de un conjunto de recursos observados y luego reacciona despachando cada evento a un manejador asociado.

El patrón Reactor no debe confundirse con el patrón Proactor, aunque ambos tienen como objetivo gestionar múltiples operaciones de E/S de forma concurrente. Si bien comparten un objetivo similar, difieren significativamente en cómo manejan el procesamiento de eventos. En el patrón Reactor, la aplicación tiene más control sobre cuándo y cómo se realizan las operaciones de E/S, ya que responde a señales que indican que la E/S está lista. Por el contrario, el patrón Proactor abstrae todo el proceso de E/S, notificando a la aplicación solo una vez que la operación se ha completado. Dado que Node.js utiliza el patrón Reactor, no profundizaremos aquí en el enfoque Proactor. No obstante, si te interesa, puedes obtener más información visitando [nodejsdp.link/proactor](https://nodejsdp.link/proactor).

#### libuv, el motor de E/S de Node.js (*libuv, the I/O engine of Node.js*)

Cada sistema operativo tiene su propia interfaz para el demultiplexor de eventos: `epoll` en Linux, `kqueue` en macOS y la API `I/O completion port` (IOCP) en Windows. Además de eso, cada operación de E/S puede comportarse de manera muy diferente según el tipo de recurso, incluso dentro del mismo sistema operativo. En los sistemas operativos Unix, por ejemplo, los archivos regulares del sistema de archivos no admiten operaciones no bloqueantes, por lo que para simular un comportamiento no bloqueante es necesario utilizar un hilo separado fuera del bucle de eventos.

Todas estas inconsistencias entre los diferentes sistemas operativos y dentro de ellos requerían la creación de una abstracción de nivel superior para el demultiplexor de eventos. Por esta razón exacta, el equipo central de Node.js creó una biblioteca nativa llamada **libuv**, con el objetivo de hacer compatible a Node.js con todos los sistemas operativos principales y normalizar el comportamiento no bloqueante de los diferentes tipos de recursos. libuv representa el motor de E/S de bajo nivel de Node.js y es probablemente el componente más importante sobre el que se construye Node.js.

Además de abstraer las llamadas al sistema subyacentes, libuv también implementa el patrón Reactor, proporcionando así una API para crear bucles de eventos, gestionar la cola de eventos, ejecutar operaciones de E/S asíncronas y poner en cola otros tipos de tareas.

Un gran recurso para aprender más sobre libuv es el libro en línea gratuito creado por Nikhil Marathe, disponible en [nodejsdp.link/uvbook](https://nodejsdp.link/uvbook).

#### La receta completa para Node.js (*The complete recipe for Node.js*)

Con nuestra comprensión del patrón Reactor y libuv, hemos descubierto algunos de los componentes clave que hacen funcionar a Node.js. Sin embargo, para completar la receta completa de construcción de la plataforma Node.js, necesitamos tres ingredientes cruciales más:
- Un conjunto de enlaces (*bindings*) responsables de envolver y exponer libuv y otras funcionalidades de bajo nivel a JavaScript.
- **V8**, el motor de JavaScript desarrollado originalmente por Google para el navegador Chrome. Esta es una de las razones por las que Node.js es tan rápido y eficiente. V8 es aclamado por su diseño revolucionario, su velocidad y su eficiente gestión de memoria.
- Una biblioteca central de JavaScript que implementa la API de alto nivel de Node.js.

Esta es la receta para crear Node.js, y la siguiente imagen representa su arquitectura final:

**Figura 1.4:** Componentes internos de Node.js.

Esto concluye nuestro recorrido por los mecanismos internos de Node.js. A continuación, veremos algunos aspectos importantes a tener en cuenta al trabajar con JavaScript en Node.js.

---

### Sección 3: JavaScript en Node.js

Una consecuencia importante de la arquitectura que acabamos de analizar es que el JavaScript que utilizamos en Node.js es algo diferente del JavaScript que usamos en el navegador.

La principal diferencia entre ejecutar JavaScript en el navegador (lado cliente) y en Node.js (lado servidor) radica en sus entornos de ejecución. En el navegador, JavaScript se ejecuta cuando un usuario visita una página web, lo que requiere un entorno seguro para evitar el acceso sin restricciones al sistema del usuario y posibles vulnerabilidades. En el lado del servidor, sin embargo, tenemos más control y normalmente necesitamos acceso a bases de datos, al sistema de archivos, a la red y a otros recursos del sistema para construir aplicaciones funcionales. Como resultado, Node.js tiene acceso a todos los servicios proporcionados por el sistema operativo.

Aunque tanto Node.js como el navegador pueden ejecutar JavaScript, sus diferentes casos de uso y requisitos de seguridad dan lugar a APIs significativamente diferentes. La diferencia más importante es que, en Node.js, no disponemos de un DOM, ni existen los objetos `window` o `document`.

En esta visión general, revisaremos algunos datos clave a tener en cuenta al utilizar JavaScript en Node.js.

#### Ejecutar el JavaScript más reciente con confianza (*Run the latest JavaScript with confidence*)

Uno de los principales retos de utilizar JavaScript en el navegador es que nuestro código debe ejecutarse en diversos dispositivos y navegadores. Diferentes navegadores pueden presentar pequeñas diferencias y es posible que no admitan las características más recientes del lenguaje o de la plataforma web. Esto solía ser un gran problema en el pasado. Si alguna vez tuviste que dar soporte a Internet Explorer 6, sabes a qué nos referimos. Afortunadamente, los navegadores actuales siguen mucho mejor los estándares y se actualizan con frecuencia, lo que reduce enormemente los problemas imprevistos. Si deseas utilizar las funciones más nuevas, a menudo puedes recurrir a transpiladores y polyfills para asegurarte de que funcionen. Sin embargo, estas herramientas añaden complejidad a tu proyecto y no todo se puede cubrir con un polyfill.

Un **transpilador** es una herramienta que convierte código de un lenguaje de programación o versión a otro, generalmente para hacerlo compatible con entornos más antiguos. Por ejemplo, un transpilador puede convertir la sintaxis moderna de JavaScript (ES2025) a la sintaxis equivalente de ES5 para que se ejecute en navegadores antiguos o versiones anteriores de Node.js.

Un **polyfill** es un fragmento de código (normalmente una biblioteca o función) que añade funcionalidad a entornos más antiguos que no admiten de forma nativa las características más nuevas. Imita el comportamiento de las nuevas características, permitiendo a los desarrolladores utilizarlas sin romper la compatibilidad.

Todos estos desafíos no se aplican al desarrollar aplicaciones con Node.js. Nuestras aplicaciones de Node.js suelen ejecutarse en un sistema y un entorno de ejecución de Node.js conocidos de antemano. Esto marca una gran diferencia, ya que nos permite escribir código para versiones específicas de JavaScript y Node.js, garantizando que no habrá sorpresas cuando lo ejecutemos en máquinas de producción.

Esto, combinado con el hecho de que las versiones más recientes de Node.js incluyen versiones recientes de V8, significa que podemos utilizar con total confianza la mayoría de las características más recientes de ECMAScript (ES) sin necesidad de pasos adicionales de transpilación (ES es el estándar en el que se basa el lenguaje JavaScript).

Ten en cuenta, no obstante, que si estamos desarrollando una biblioteca pensada para terceros, aún debemos considerar que nuestro código puede ejecutarse en diferentes versiones de Node.js. La práctica común en este caso es apuntar a la versión con soporte a largo plazo (LTS) activa más antigua y especificar la sección `engines` en nuestro `package.json`. De esta manera, el gestor de paquetes advertirá a los usuarios si intentan instalar un paquete que no sea compatible con su versión de Node.js.

Puedes obtener más información sobre los ciclos de lanzamiento de Node.js en [nodejsdp.link/node-releases](https://nodejsdp.link/node-releases). Además, puedes encontrar la referencia para la sección `engines` de `package.json` en [nodejsdp.link/package-engines](https://nodejsdp.link/package-engines). Finalmente, puedes ver qué característica de ES es compatible con cada versión de Node.js en [nodejsdp.link/node-green](https://nodejsdp.link/node-green).

#### El sistema de módulos (*The module system*)

Desde sus inicios, Node.js incluyó un sistema de módulos, incluso cuando JavaScript no contaba con soporte oficial para ello. El sistema de módulos original de Node.js se llama **CommonJS**, y utiliza la palabra clave `require` para importar funciones, variables y clases desde módulos integrados o desde otros módulos en el sistema de archivos del dispositivo.

CommonJS fue revolucionario para el mundo de JavaScript, ganando popularidad incluso en el lado del cliente, donde se utiliza con empaquetadores de módulos (*module bundlers*) como Webpack o Rollup para crear paquetes de código fácilmente ejecutables por el navegador. CommonJS fue esencial para Node.js, permitiendo a los desarrolladores crear aplicaciones grandes y bien organizadas comparables a las de otras plataformas del lado del servidor.

Un **empaquetador de módulos** (*module bundler*) es una herramienta que combina múltiples archivos JavaScript y sus dependencias en un solo archivo o en un número menor de archivos, a menudo llamados paquetes (*bundles*). Este proceso ayuda a optimizar los tiempos de carga y reduce la complejidad de gestionar dependencias. Los empaquetadores analizan las relaciones entre varios módulos, resuelven dependencias y las generan en un formato que puede ejecutarse eficientemente en un navegador u otros entornos de ejecución.

Hoy en día, JavaScript dispone de la sintaxis de módulos ES (utilizando la palabra clave `import`), que Node.js adopta únicamente en sintaxis, ya que su implementación subyacente difiere de la del navegador. Mientras que los navegadores gestionan principalmente módulos remotos, Node.js, al menos por ahora, se ocupa únicamente de módulos en el sistema de archivos local.

Trataremos los módulos en el contexto de Node.js con más detalle en el próximo capítulo.

#### Acceso completo a los servicios del sistema operativo (*Full access to operating system services*)

Como ya mencionamos, aunque Node.js utilice JavaScript, no se ejecuta dentro de los límites de un navegador. Esto permite a Node.js disponer de enlaces (*bindings*) para todos los servicios principales ofrecidos por el sistema operativo subyacente.

Por ejemplo, podemos acceder a cualquier archivo del sistema de archivos (sujeto a los permisos a nivel de sistema operativo) gracias al módulo `fs`, o podemos escribir aplicaciones que utilicen sockets TCP o UDP de bajo nivel gracias a los módulos `net` y `dgram`. Podemos crear servidores HTTP(S) (con los módulos `http` e `https`) o utilizar los algoritmos estándar de cifrado y hashing de OpenSSL (con el módulo `crypto`). También podemos acceder a algunos aspectos internos de V8 (con el módulo `v8`) o ejecutar código en un contexto V8 diferente (con el módulo `vm`).

Asimismo, podemos ejecutar otros procesos (con el módulo `child_process`) u obtener información sobre el proceso de nuestra propia aplicación mediante la variable global `process`. En particular, a partir de la variable global `process`, podemos obtener una lista de las variables de entorno asignadas al proceso (con `process.env`) o los argumentos de la línea de comandos pasados a la aplicación en el momento de su inicio (con `process.argv`).

A lo largo del libro tendrás la oportunidad de utilizar muchos de los módulos descritos aquí, pero para una referencia completa puedes consultar la documentación oficial de Node.js en [nodejsdp.link/node-docs](https://nodejsdp.link/node-docs).

#### Ejecución de código nativo (*Running native code*)

Una de las capacidades más potentes que ofrece Node.js es sin duda la posibilidad de crear módulos en el espacio de usuario que puedan enlazarse con código compilado nativo (por ejemplo, escrito en lenguajes compilados como C, C++ o Rust). Esto confiere a la plataforma una enorme ventaja, ya que nos permite reutilizar componentes existentes o utilizar nuevos componentes escritos en lenguajes compilados de alto rendimiento como C/C++. Node.js ofrece oficialmente un excelente soporte para implementar módulos nativos gracias a la interfaz **Node-API**.

Pero, ¿cuál es la ventaja? En primer lugar, nos permite reutilizar, con poco esfuerzo, una gran cantidad de bibliotecas de código abierto existentes y, lo que es más importante, permite a una empresa reutilizar su propio código heredado (*legacy*) en C/C++ sin necesidad de migrarlo.

Otra consideración importante es que el código nativo sigue siendo necesario para acceder a funciones de bajo nivel, como la comunicación con controladores de hardware o puertos de hardware (por ejemplo, USB o serie). De hecho, gracias a su capacidad para enlazarse con código nativo, Node.js se ha vuelto popular en el mundo del Internet de las Cosas (IoT) y la robótica casera.

Por último, aunque V8 es muy (muy) rápido ejecutando JavaScript, todavía tiene una penalización de rendimiento que pagar en comparación con la ejecución de código nativo. En la informática cotidiana esto rara vez supone un problema, pero para aplicaciones con un uso intensivo de CPU, como aquellas con gran procesamiento y manipulación de datos, delegar el trabajo al código nativo puede tener mucho sentido.

También debemos mencionar que hoy en día la mayoría de las máquinas virtuales (VM) de JavaScript (y también Node.js) admiten **WebAssembly** (Wasm), un formato de instrucciones de bajo nivel que nos permite compilar lenguajes distintos de JavaScript (como C++ o Rust) en un formato "ejecutable" por las máquinas virtuales de JavaScript. Esto aporta muchas de las ventajas mencionadas, sin necesidad de interactuar directamente con código nativo.

Puedes obtener más información sobre Wasm en el sitio web oficial del proyecto en [nodejsdp.link/webassembly](https://nodejsdp.link/webassembly).

#### Node.js y TypeScript (*Node.js and TypeScript*)

TypeScript es un lenguaje de código abierto desarrollado por Microsoft para añadir un sistema de tipos estático y fuerte a JavaScript. Piensa en TypeScript como una versión mejorada de JavaScript con funciones adicionales. Por ejemplo, con TypeScript, los desarrolladores pueden especificar los tipos de argumentos que acepta una función, el tipo que devuelve o la estructura de un objeto. Estos tipos se comprueban antes de que se ejecute el código, lo que ayuda a detectar problemas como el acceso a propiedades inexistentes o el paso de parámetros incorrectos. Esto hace que tu código sea más seguro y robusto, evitando muchos errores antes incluso de que el código entre en producción.

Es importante saber que el código TypeScript no puede ejecutarse directamente en plataformas JavaScript como Node.js. En su lugar, TypeScript se utiliza para el análisis estático y debe compilarse (o "transpilarse") a JavaScript simple para ejecutarse. Este proceso convierte TypeScript en archivos JavaScript que pueden funcionar en un entorno compatible con JavaScript. Aunque este paso adicional pueda parecer más laborioso, las ventajas de detectar errores a tiempo y mejorar la calidad del código lo hacen sumamente beneficioso, especialmente para proyectos grandes.

#### Uso de TypeScript con Node.js (*Using TypeScript with Node.js*)

Si deseas utilizar TypeScript con Node.js, tienes varias opciones disponibles.

La primera opción es utilizar el compilador oficial de TypeScript para convertir tus archivos TypeScript en archivos JavaScript equivalentes que puedan ser ejecutados por Node.js. Para instalar TypeScript en tu proyecto, ejecuta:

```bash
npm install --save-dev typescript
```

Si tienes un archivo TypeScript llamado `example.ts`, por ejemplo, puedes transpilarlo a JavaScript con el siguiente comando:

```bash
npx tsc example.ts
```

Este comando primero verificará `example.ts` en busca de errores de tipo y, si todo está correcto, lo convertirá a JavaScript plano, creando un archivo llamado `example.js`. Luego puedes ejecutar `example.js` con Node.js:

```bash
node example.js
```

Al desarrollar una aplicación, puede resultar tedioso transpilar manualmente el código cada vez que deseas ejecutarlo. Afortunadamente, existen herramientas que se encargan de esto automáticamente, como `ts-node` y `tsx`.

Para instalar `ts-node` o `tsx`, ejecuta:

```bash
npm install --save-dev ts-node # or npm install --save-dev tsx
```

Luego puedes ejecutar directamente `example.ts` con:

```bash
npx ts-node example.ts # or npx tsx example.ts
```

Además, `tsx` también se puede utilizar como un cargador (*loader*) de Node.js, lo cual puede ser conveniente en caso de que quieras invocar directamente la CLI de `node` (por ejemplo, si necesitas pasar argumentos y opciones adicionales de CLI mientras ejecutas tu código):

```bash
node --import=tsx example.ts
```

Los **cargadores** (*loaders*) son un mecanismo que permite a los desarrolladores personalizar cómo se cargan y procesan los módulos en Node.js. Por defecto, Node.js utiliza su sistema de carga de módulos integrado (CommonJS o módulos ES), pero con cargadores personalizados puedes interceptar y modificar el comportamiento original. Al usar un cargador personalizado de TypeScript como `tsx`, puedes interceptar los módulos TypeScript durante el proceso de carga, transpilar los mismos a JavaScript y luego pasar el código resultante a Node.js para su ejecución. Esto elimina la necesidad de un paso de compilación separado, haciendo que los flujos de trabajo de desarrollo sean más fluidos y flexibles.

Ten en cuenta que al usar herramientas como `ts-node` y `tsx` para ejecutar código TypeScript directamente, el código se transpila "al vuelo" (*on the fly*). Esto significa que incurres en el costo de tiempo de la transpilación cada vez que ejecutas tu código. Si bien este enfoque es conveniente para el desarrollo, es más eficiente pre-transpilar el código antes de desplegarlo en producción. Esto evita la sobrecarga de la transpilación durante el tiempo de ejecución y garantiza un mejor rendimiento.

El equipo central de Node.js está dedicando un gran esfuerzo para hacer de TypeScript un ciudadano de primera clase en la plataforma, por lo que podemos esperar que en el futuro sea aún más fácil ejecutar código TypeScript sin tener que instalar herramientas de terceros.

De hecho, a partir de Node.js 24 puedes ejecutar archivos TypeScript directamente con la CLI de `node` mediante la eliminación de tipos integrada (*type stripping*). Solo se admite la sintaxis de TypeScript borrable y Node ignora `tsconfig.json`, por lo que los resultados pueden variar y un ejecutor (*runner*) sigue siendo la mejor opción para contar con todas las funciones; para consultar la guía más reciente visita [nodejsdp.link/node-ts](https://nodejsdp.link/node-ts).

#### El paquete @types/node (*The @types/node package*)

Al desarrollar aplicaciones TypeScript en un entorno Node.js, debes utilizar el paquete `@types/node` para una experiencia de desarrollo óptima. Este paquete proporciona a TypeScript las definiciones de tipos necesarias para Node.js, habilitando un tipado fuerte y un autocompletado mejorado dentro de tu IDE. Node.js en sí está escrito en JavaScript, que intrínsecamente no incluye definiciones de tipos. Por lo tanto, sin `@types/node`, TypeScript carecería del conocimiento sobre los módulos, APIs y variables globales específicas de Node.js, lo que dificultaría escribir código seguro en cuanto a tipos (*type-safe*).

El paquete `@types/node` incluye definiciones de tipos para toda la API de Node.js, cubriendo todo, desde módulos centrales como `fs`, `http` y `path`, hasta objetos globales como `process` y `Buffer`. Al incorporar este paquete a tu proyecto, obtienes acceso a las potentes funciones de verificación estática de tipos de TypeScript, que pueden ayudarte a detectar posibles errores en las primeras etapas del proceso de desarrollo. Además, proporciona un autocompletado integral en tu editor de código, lo que agiliza el desarrollo y reduce la probabilidad de errores provocados por un uso incorrecto de métodos o parámetros mal configurados.

Para instalar el paquete `@types/node`, puedes usar npm:

```bash
npm install --save-dev @types/node
```

Estos comandos añaden el paquete `@types/node` como una dependencia de desarrollo (es decir, se instalará solo durante el desarrollo y no en entornos de producción).

TypeScript tiene mucho que ofrecer, pero no es el enfoque principal de este libro. Proporcionaremos ejemplos y consejos de TypeScript cuando ayuden a clarificar temas específicos, pero para una introducción completa a TypeScript recomendamos visitar el sitio web oficial de TypeScript en [nodejsdp.link/ts](https://nodejsdp.link/ts). Si ya estás familiarizado con TypeScript, este libro seguirá siendo muy valioso para ti. Los patrones y técnicas que analizamos se pueden aplicar fácilmente a cualquier proyecto TypeScript.

---

### Sección 4: Resumen

En este capítulo has visto cómo la plataforma Node.js se fundamenta en unos pocos principios importantes que moldean tanto su arquitectura interna como el código que escribimos. Has aprendido que Node.js tiene un núcleo mínimo y que adoptar el "estilo Node" significa escribir módulos que sean más pequeños, más simples y que expongan solo la funcionalidad mínima necesaria.

A continuación, descubriste el patrón Reactor, que es el corazón palpitante de Node.js, y analizaste en profundidad la arquitectura interna del entorno de ejecución de la plataforma para revelar sus otros pilares: V8, libuv, los enlaces (*bindings*) y la biblioteca central de JavaScript.

Finalmente, analizamos algunas de las principales características del uso de JavaScript en Node.js en comparación con el navegador y aprendimos cómo se puede aprovechar TypeScript al trabajar con Node.js.

Además de las evidentes ventajas técnicas facilitadas por su arquitectura interna, Node.js despierta un gran interés debido a los principios que encarna y a la vibrante comunidad que lo rodea. Su enfoque en la simplicidad y la eficiencia resuena entre los desarrolladores, ofreciendo un método de programación más centrado en el ser humano que equilibra la facilidad de uso con la escalabilidad. Esta es la razón por la que tantos desarrolladores terminan enamorándose de Node.js.

En el próximo capítulo, profundizaremos en uno de los temas más fundamentales e importantes de Node.js: su sistema de módulos.
