# Parte 3: Patrones avanzados y arquitectura

## Capítulo 12: Patrones de escalabilidad y arquitectura

Node.js comenzó como un servidor web pequeño y no bloqueante construido con JavaScript y C++, pero su creación fue impulsada por un objetivo mayor: la escalabilidad. Ryan Dahl quería una plataforma que pudiera manejar muchos usuarios concurrentes sin agotar los recursos del servidor. La idea surgió al intentar implementar una barra de progreso de subida de archivos a principios de 2009 ([nodejsdp.link/first-commit](https://nodejsdp.link/first-commit)), lo cual era difícil con las herramientas disponibles en ese momento. Las plataformas existentes como Ruby on Rails tenían dificultades bajo alta concurrencia, y los experimentos con C, Haskell y Python conllevaban sus propias compensaciones. Luego, JavaScript, impulsado por el nuevo motor V8 de Google, ofreció un nuevo comienzo para el desarrollo del lado del servidor. Ryan construyó una plataforma donde el comportamiento asíncrono y no bloqueante era el valor predeterminado, utilizando un solo hilo y un bucle de eventos para administrar miles de conexiones de manera eficiente. A partir de ahí, Node.js se convirtió en una plataforma completa para crear aplicaciones rápidas y orientadas a eventos, con la escalabilidad como principio central de diseño. Pero el simple hecho de que Node.js pueda escalar no significa que todas las aplicaciones lo hagan. Lograr rendimiento y confiabilidad requiere comprender la plataforma, tomar decisiones de diseño bien pensadas y seguir las mejores prácticas.

Este capítulo adopta una visión de mayor nivel sobre la escalabilidad: no solo manejar más usuarios, sino construir sistemas que sean confiables, mantenibles y resilientes bajo carga. Cubriremos:
- Por qué la escalabilidad debe ser parte de tu proceso de diseño desde el principio
- Qué es el cubo de escala y cómo ayuda a enmarcar diferentes estrategias de escalado
- Cómo escalar una aplicación Node.js ejecutando múltiples instancias
- Cómo utilizar balanceadores de carga para distribuir el tráfico de manera eficiente
- Qué es un registro de servicios y por qué es importante en entornos dinámicos
- Cómo escalar utilizando herramientas de orquestación de contenedores como Kubernetes
- Cómo descomponer una aplicación monolítica en microservicios
- Cómo conectar muchos servicios utilizando patrones arquitectónicos simples y probados

Comprender estos conceptos no se trata solo de crear mejores aplicaciones. Se trata de subir de nivel como desarrollador. ¡Vamos a sumergirnos!

---

### Sección 1: Introducción al escalado de aplicaciones

La escalabilidad es la capacidad de un sistema para crecer y adaptarse a condiciones cambiantes. No se trata solo de la capacidad técnica, sino también de respaldar el crecimiento del negocio y de la organización que está detrás de él. Si estás construyendo un producto que se espera que llegue a millones de usuarios, te enfrentarás a preguntas como: ¿Cómo manejará el sistema la creciente demanda sin ralentizarse ni colapsar? ¿Cómo almacenará grandes volúmenes de datos y administrará la I/O de manera eficiente? A medida que tu equipo crezca, ¿cómo organizarás el trabajo para que las diferentes partes del código base puedan evolucionar de forma independiente? Incluso los proyectos más pequeños enfrentan desafíos de escalabilidad, solo que en formas diferentes, e ignorarlos puede perjudicar tanto al proyecto como a la empresa.

Por supuesto, no necesitas optimizar todo por adelantado. El camino pragmático es comprender las opciones técnicas disponibles y sus compensaciones para que puedas decidir cuándo vale la pena aplicarlas. El pragmatismo solo es posible con un amplio conocimiento de esas opciones técnicas y una visión clara de tus objetivos comerciales.

En este capítulo, exploraremos patrones y arquitecturas para escalar aplicaciones Node.js. Con estas herramientas y una comprensión de tu contexto empresarial, puedes diseñar sistemas que se adapten, funcionen de manera confiable y mantengan satisfechos a los usuarios.

#### Escalado de aplicaciones Node.js

La mayor parte del trabajo en una aplicación Node.js se ejecuta en un solo hilo. Como se discutió en el [Capítulo 1](https://subscription.packtpub.com/book/web-development/9781803238944/1), La plataforma Node.js, esto no es una limitación sino una fortaleza. Con su modelo de I/O no bloqueante, Node.js puede manejar cientos o miles de solicitudes concurrentes con un uso intensivo de I/O por segundo, incluso en hardware modesto.

Aun así, un solo hilo tiene límites. Para manejar un mayor tráfico o cargas de trabajo exigentes, necesitas escalar ejecutando múltiples procesos y, eventualmente, distribuyéndolos en varias máquinas. Escalar no solo aumenta la capacidad, sino que también mejora la confiabilidad y la tolerancia a fallos: si una instancia falla, otras pueden tomar el control.

La escalabilidad también se aplica a la estructura de la aplicación. A medida que los sistemas crecen en funciones, equipos y responsabilidades, diseñarlos como componentes modulares y distribuidos los mantiene mantenibles y adaptables. La flexibilidad de JavaScript fomenta módulos pequeños y enfocados, y con disciplina y herramientas como TypeScript, sus peculiaridades pueden convertirse en fortalezas.

Dominar cuándo y cómo escalar aplicaciones Node.js es clave para construir sistemas confiables y de alto rendimiento. En las siguientes secciones, exploraremos los patrones, herramientas y principios que lo hacen posible.

#### Las tres dimensiones de la escalabilidad

Cuando hablamos de escalabilidad, el primer principio fundamental que debemos entender es la distribución de carga. Esto se refiere a la práctica de dividir la carga de trabajo de una aplicación entre múltiples procesos y máquinas. Hay muchas formas de lograrlo, y el libro *The Art of Scalability*, de Martin L. Abbott y Michael T. Fisher, presenta un modelo útil para categorizarlas.

Este modelo se llama el **cubo de escala** (*scale cube*) y describe la escalabilidad en tres dimensiones distintas:
- **Eje X:** Clonación
- **Eje Y:** Descomposición por servicio/funcionalidad
- **Eje Z:** División por partición de datos

Estas tres dimensiones se pueden representar como un cubo:

```
        Y (Servicios / Funcionalidad)
        ^
        |     / Z (Partición de datos)
        |    /
        |   /
        |  /
        | /
        +-----------------> X (Clonación)
     Monolito
```

*Figura 12.1: El cubo de escala.*

La esquina inferior izquierda del cubo representa una aplicación monolítica típica. Esta es una aplicación que contiene toda su funcionalidad en un único código base y se ejecuta como una sola instancia. Esta configuración es común para aplicaciones en etapas tempranas de desarrollo o aquellas con cargas de trabajo pequeñas.

Desde este punto de partida, existen tres estrategias principales para escalar, visualizadas como movimiento a lo largo de los tres ejes del cubo: X, Y y Z.

##### Eje X: clonación

Escalar a lo largo del eje X es la evolución más directa de una aplicación monolítica. Implica duplicar la misma aplicación y ejecutarla en múltiples instancias. Cada instancia maneja una porción de la carga de trabajo total. Este enfoque suele ser simple de implementar, rentable en términos de esfuerzo de desarrollo y muy eficaz en la práctica. Por ejemplo, clonar la aplicación en cuatro instancias permite que cada una administre aproximadamente una cuarta parte del tráfico total.

##### Eje Y: descomposición por servicio o funcionalidad

Escalar a lo largo del eje Y significa descomponer la aplicación en servicios separados basados en la funcionalidad o el caso de uso. Cada servicio se convierte en una aplicación independiente con su propio código base, y también puede tener su propia base de datos e interfaz de usuario. Por ejemplo, podrías separar el panel de administración de la parte pública de tu producto, o aislar la autenticación de usuarios en un servicio dedicado. La forma en que divides la funcionalidad depende de factores como los requisitos comerciales, los límites de datos y las necesidades de los usuarios.

Esta forma de escalado tiene un impacto significativo, no solo en la arquitectura, sino también en cómo se desarrolla, despliega y mantiene el sistema. Aquí es donde normalmente entra en juego el concepto de microservicios, ya que está estrechamente vinculado a una descomposición refinada en el eje Y.

##### Eje Z: división por partición de datos

El eje Z introduce una estrategia de escalado más avanzada. Aquí, la aplicación está diseñada de modo que cada instancia sea responsable solo de una parte de los datos totales. Esta técnica, a menudo llamada partición de datos (*data partitioning* o *sharding*), se usa comúnmente a nivel de base de datos, pero también se puede aplicar a nivel de aplicación en escenarios específicos.

Por ejemplo, podrías dividir a los usuarios según el país (partición por lista), asignar particiones por la letra inicial de su nombre (partición por rango) o usar una función hash para determinar a qué partición pertenece un usuario (partición por hash). Cada instancia de la aplicación operaría únicamente en su partición asignada.

Esta estrategia requiere un mecanismo de búsqueda para determinar qué instancia es responsable de cualquier dato determinado. Aunque es potente, el escalado en el eje Z normalmente se reserva para sistemas altamente distribuidos o casos especiales, como cuando se crean capas personalizadas de persistencia de datos o se trabaja con bases de datos que carecen de soporte nativo para particionado. También se utiliza en sistemas que operan a escala masiva, como los que se encuentran en las grandes empresas de tecnología.

Debido a su complejidad, el escalado en el eje Z generalmente debe considerarse solo después de haber explorado por completo las oportunidades de los ejes X e Y.

##### Combinación de estrategias en el cubo de escala

Las tres dimensiones del cubo de escala no son mutuamente excluyentes. Muchos sistemas del mundo real escalan a lo largo de todas ellas con el tiempo. El objetivo no es elegir el "mejor" eje, sino encontrar el equilibrio adecuado para tu etapa de crecimiento, necesidades actuales y restricciones. Escalar es un proceso continuo. Un viaje común comienza con una arquitectura monolítica en las primeras etapas, cuando la velocidad y la simplicidad ayudan a lograr el ajuste producto-mercado (*product-market fit*). A medida que crece el uso, puedes escalar horizontalmente a lo largo del eje X ejecutando múltiples instancias de la misma aplicación, aumentando la capacidad con una complejidad mínima. Cuando el tamaño del equipo y el alcance del producto crean cuellos de botella de coordinación, escalar a lo largo del eje Y mediante la descomposición en servicios independientes permite a los equipos trabajar en paralelo y reduce la complejidad de cada servicio. Si los grandes volúmenes de datos se convierten en un desafío, el eje Z (particionar los datos para que diferentes instancias manejen diferentes porciones) puede abordar los límites de rendimiento o almacenamiento.

Escalar en cualquier dimensión es un espectro. Podrías descomponer ligeramente la aplicación en unos pocos servicios o adoptar docenas de microservicios, fragmentar solo algunos datos o particionarlos por completo. El enfoque correcto depende de tu contexto, las capacidades de tu equipo y las compensaciones involucradas.

En las siguientes secciones, nos centraremos en las dos técnicas más comunes para escalar aplicaciones Node.js: la clonación y la descomposición por funcionalidad o servicio.

---

### Sección 2: Clonación y balanceo de carga

Antes de sumergirnos en técnicas específicas, es importante introducir los conceptos de escalabilidad vertical y escalabilidad horizontal.

La **escalabilidad vertical** significa actualizar el hardware que ejecuta tu aplicación. Esto podría implicar agregar más memoria, aumentar la capacidad de la CPU o mejorar el rendimiento del disco. En los sistemas tradicionales, este suele ser el primer enfoque, porque permite que la aplicación permanezca exactamente como está mientras confía en máquinas más potentes para manejar la mayor carga. Es como intentar mover más mercancías con un solo vehículo. Puedes comenzar con una pequeña camioneta familiar, luego pasar a una furgoneta y, finalmente, usar un camión de gran tamaño. Pero solo puedes llegar hasta cierto punto. Tarde o temprano, tu vehículo se vuelve demasiado costoso, pesado o grande para manejarlo de manera eficiente. Ahí es cuando agregar más vehículos se convierte en la mejor opción.

Aquí es donde entra la **escalabilidad horizontal**. En lugar de llevar una sola máquina a sus límites, ejecutas múltiples instancias de tu aplicación, ya sea en la misma máquina utilizando procesos separados o en múltiples máquinas. Este es el enfoque al que nos referimos anteriormente cuando discutimos la clonación y el escalado a lo largo del eje X del cubo de escala. Es como cambiar de un solo camión gigante a una flota de vehículos, cada uno de los cuales maneja una parte de la carga.

En los servidores web tradicionales y multihilo, la escalabilidad horizontal normalmente se introduce solo después de que la escalabilidad vertical se ha maximizado o se ha vuelto demasiado cara. Estos servidores están diseñados para aprovechar al máximo todos los núcleos y la memoria disponibles mediante la ejecución de múltiples hilos.

Node.js adopta un enfoque diferente para la concurrencia. Debido a que el código se ejecuta en un solo hilo por defecto, un solo proceso normalmente utilizará solo un núcleo de CPU. Esto significa que, de fábrica, es posible que no aproveches por completo toda la potencia disponible en un servidor multinúcleo moderno.

Aunque el código JavaScript que escribes en Node.js se ejecuta en un solo hilo (el hilo principal), el propio proceso de `node` utiliza múltiples hilos entre bastidores. Por ejemplo, Node.js crea un grupo de hilos (*thread pool*, administrado por libuv) para tareas como operaciones del sistema de archivos, compresión y criptografía, junto con hilos para el recolector de basura y el bucle de eventos. Cuando interviene la I/O de red, se utilizan hilos adicionales para las búsquedas DNS. Este multihilo interno es transparente para la mayoría de los desarrolladores, pero es clave para la capacidad de Node.js de manejar muchos tipos de operaciones de manera eficiente.

Podríamos ver esto como una limitación, o podríamos tomarlo como un suave empujón para comenzar a pensar en la escalabilidad desde el principio. Dado que Node.js fomenta el escalado a través de múltiples procesos, los desarrolladores adoptan de forma natural patrones que promueven la resiliencia, la redundancia y la arquitectura modular en las primeras etapas del desarrollo. Por ejemplo, puedes clonar tu aplicación en varios procesos o contenedores, no necesariamente porque hayas alcanzado un techo de rendimiento, sino para aumentar la disponibilidad y la tolerancia a fallos. Si un proceso falla, otros pueden continuar manejando solicitudes sin interrupciones.

Esta característica intrínseca de Node.js también fomenta mejores hábitos. Si diseñas tu aplicación para que se ejecute en múltiples instancias desde el principio, evitarás de forma natural depender de recursos locales como la memoria o el almacenamiento en disco que no se pueden compartir entre instancias. Por ejemplo, almacenar datos de sesión en la memoria puede funcionar bien cuando tu aplicación se ejecuta como una sola instancia, pero una vez que la aplicación se clona o se despliega en múltiples procesos o máquinas, este enfoque falla. Dado que la memoria no se comparte entre procesos o máquinas, una solicitud puede llegar a una instancia donde el usuario está autenticado, pero la siguiente solicitud podría llegar a una instancia diferente que no tiene conocimiento de la sesión del usuario. Como resultado, el usuario puede aparecer como desconectado inesperadamente.

Con esto en mente, veamos ahora el mecanismo más básico para escalar aplicaciones Node.js en múltiples procesos: el módulo `node:cluster`.

#### El módulo cluster

En Node.js, el patrón más simple para distribuir la carga de una aplicación entre múltiples procesos que se ejecutan en una sola máquina es mediante el uso del módulo `node:cluster`, que forma parte de las bibliotecas centrales. Estos procesos a menudo se denominan instancias de la aplicación y todos se generan a partir del mismo código base. El módulo `node:cluster` facilita la bifurcación (*forking*) de estos procesos y equilibra automáticamente las conexiones entrantes entre ellos.

```
                  Peticiones de Clientes
                            |
                            v
                 [ Proceso Primario ]
                (Distribuye conexiones)
                 /        |                         /         |                         v          v           v
          [Worker 1]  [Worker 2]  [Worker 3]
```

*Figura 12.2: Esquema del módulo cluster.*

El proceso primario es responsable de generar una cantidad de procesos (*workers*), cada uno representando una instancia de la aplicación que queremos escalar. Cada conexión entrante se distribuye entre los workers clonados, repartiendo la carga entre ellos.

Dado que cada worker es un proceso independiente, puedes utilizar este enfoque para generar tantos workers como el número de CPUs disponibles en el sistema. Con este enfoque, puedes permitir fácilmente que una aplicación Node.js aproveche toda la potencia de cálculo disponible en el sistema.

##### Notas sobre el comportamiento del módulo cluster

En la mayoría de los sistemas, el módulo `node:cluster` utiliza un algoritmo explícito de balanceo de carga **round-robin**. Este algoritmo se utiliza dentro del proceso primario, que se asegura de que las solicitudes se distribuyan uniformemente entre todos los workers. La programación round-robin está habilitada de forma predeterminada en todas las plataformas excepto Windows, y se puede modificar globalmente configurando la variable `cluster.schedulingPolicy` utilizando las constantes `cluster.SCHED_RR` (round-robin) o `cluster.SCHED_NONE` (manejado por el sistema operativo).

El algoritmo round-robin distribuye la carga uniformemente entre las instancias disponibles de forma rotativa. La primera solicitud se reenvía a la primera instancia, la segunda a la siguiente instancia de la lista, y así sucesivamente. Cuando se llega al final de la lista, la iteración comienza de nuevo desde el principio. En el módulo `node:cluster`, la lógica round-robin está enriquecida con algunos comportamientos adicionales destinados a evitar la sobrecarga de un proceso worker determinado.

Cuando usamos el módulo `node:cluster`, cada invocación a `server.listen()` en un proceso worker se delega al proceso primario. Esto permite que el proceso primario reciba todos los mensajes entrantes y los distribuya al grupo de workers. El módulo cluster hace que este proceso de delegación sea muy simple para la mayoría de los casos de uso, pero hay varios casos extremos en los que llamar a `server.listen()` en un módulo worker podría no hacer lo que esperas:
- `server.listen({fd})`: Si un worker escucha usando un descriptor de archivo específico, por ejemplo, invocando `server.listen({fd: 17})`, esta operación puede producir resultados inesperados. Los descriptores de archivo se asignan a nivel de proceso, por lo que si un proceso worker asigna un descriptor de archivo, este no coincidirá con el mismo archivo en el proceso primario. Una forma de superar esta limitación es crear el descriptor de archivo en el proceso primario y luego pasarlo al proceso worker. De esta manera, el proceso worker puede invocar `server.listen()` usando un descriptor que es conocido por el primario.
- `server.listen(handle)`: Escuchar usando objetos manejadores (`FileHandle`) explícitamente en un proceso worker hará que el worker use el manejador suministrado directamente, en lugar de delegar la operación al proceso primario.
- `server.listen(0)`: Llamar a `server.listen(0)` generalmente hará que los servidores escuchen en un puerto aleatorio. Sin embargo, en un cluster, cada worker recibirá el mismo puerto "aleatorio" cada vez que llame a `server.listen(0)`. En otras palabras, el puerto es aleatorio solo la primera vez; se fijará a partir de la segunda llamada. Si deseas que cada worker escuche en un puerto aleatorio diferente, debes generar los números de puerto tú mismo.

##### Creación de un servidor HTTP simple

Comencemos a trabajar en un ejemplo. Construyamos un pequeño servidor HTTP, clonado y balanceado utilizando el módulo `node:cluster`. Primero, necesitamos una aplicación para escalar y, para este ejemplo, no necesitamos demasiado, solo un servidor HTTP muy básico.

Creemos un archivo llamado `app.js` que contenga el siguiente código:

```javascript
// app.js
import { createServer } from 'node:http'

const server = createServer((_req, res) => {
  // simulates CPU intensive work
  let i = 1e7
  while (i > 0) {
    i--
  }

  console.log(`Handling request from ${process.pid}`)
  res.end(`Hello from ${process.pid}
`)
})

server.listen(8080, () => console.log(`Started at ${process.pid}`))
```

El servidor HTTP que acabamos de construir responde a cualquier solicitud enviando de vuelta un mensaje que contiene su identificador de proceso (PID); esto es útil para identificar qué instancia de la aplicación está manejando la solicitud. En esta versión de la aplicación, solo tenemos un proceso, por lo que el PID que ves en las respuestas y en los registros siempre será el mismo.

Además, para simular algún trabajo real de CPU, realizamos un bucle vacío 10 millones de veces: sin esto, la carga del servidor sería casi insignificante y sería bastante difícil sacar conclusiones de los benchmarks que vamos a ejecutar.

El módulo de aplicación que creamos aquí es solo una abstracción simple para un servidor web genérico. No estamos utilizando un marco web como Express o Fastify por simplicidad, pero siéntete libre de reescribir estos ejemplos utilizando el marco web de tu elección.

Ahora puedes comprobar si todo funciona como se esperaba ejecutando la aplicación como de costumbre y enviando una solicitud a `http://localhost:8080` usando un navegador o curl.

También puedes intentar medir las solicitudes por segundo que el servidor puede manejar en un proceso. Para este propósito, puedes usar Autocannon para cargar el servidor con 10 conexiones concurrentes durante 10 segundos. Como referencia, el resultado obtenido en nuestra máquina es del orden de 300 solicitudes por segundo.

Las pruebas de carga en este capítulo están simplificadas con fines de aprendizaje y no son medidas de rendimiento exactas. Para aplicaciones de producción, ejecuta tus propios benchmarks después de cada cambio para ver qué técnicas funcionan mejor para tu caso específico.

Ahora que tenemos una aplicación web de prueba simple y algunos puntos de referencia, estamos listos para probar algunas técnicas para mejorar el rendimiento de la aplicación.

##### Escalado con el módulo cluster

Actualicemos ahora `app.js` para escalar nuestra aplicación usando el módulo `node:cluster`:

```javascript
// app.js
import { createServer } from 'node:http'
import { cpus } from 'node:os'
import cluster from 'node:cluster'

if (cluster.isPrimary) { // 1
  const availableCpus = cpus()
  console.log(`Clustering to ${availableCpus.length} processes`)
  for (const _ of availableCpus) {
    cluster.fork()
  }
} else { // 2
  const server = createServer((_req, res) => {
    // simulates CPU intensive work
    let i = 1e7
    while (i > 0) {
      i--
    }

    console.log(`Handling request from ${process.pid}`)
    res.end(`Hello from ${process.pid}
`)
  })

  server.listen(8080, () => console.log(`Started at ${process.pid}`))
}
```

Como podemos ver, usar el módulo cluster requiere muy poco esfuerzo. Analicemos lo que está sucediendo:
1. Cuando lanzamos `app.js` desde la línea de comandos, estamos ejecutando el proceso primario. En este caso, la variable `cluster.isPrimary` se establece en `true`, y el único trabajo que debemos hacer es bifurcar el proceso actual usando `cluster.fork()`. En el ejemplo anterior, estamos iniciando tantos workers como núcleos de CPU lógicos hay en el sistema para aprovechar toda la potencia de procesamiento disponible.
2. Cuando se ejecuta `cluster.fork()` desde el proceso primario, el módulo actual (`app.js`) se ejecuta nuevamente, pero esta vez en modo worker (`cluster.isWorker` se establece en `true`, mientras que `cluster.isPrimary` es `false`). Cuando la aplicación se ejecuta como worker, puede comenzar a hacer un trabajo real. En este caso, inicia un nuevo servidor HTTP.

Es importante recordar que cada worker es un proceso Node.js diferente con su propio bucle de eventos, espacio de memoria y módulos cargados.

Es interesante notar que el uso del módulo `node:cluster` se basa en un patrón recurrente, que hace que sea muy fácil ejecutar múltiples instancias de una aplicación:

```javascript
if (cluster.isPrimary) {
  // fork()
} else {
  // do work
}
```

Bajo el capó, la función `cluster.fork()` utiliza la API `child_process.fork()`; por lo tanto, también tenemos un canal de comunicación disponible entre el primario y los workers. Se puede acceder a los procesos worker desde la variable `cluster.workers`, por lo que transmitir un mensaje a todos ellos sería tan fácil como ejecutar la siguiente línea de código:

```javascript
Object.values(cluster.workers).forEach(worker => worker.send('Hello from the primary'))
```

Ahora, intentemos ejecutar nuestro servidor HTTP en modo cluster. Si nuestra máquina tiene más de un núcleo, deberíamos ver varios workers iniciados por el proceso primario, uno tras otro. Por ejemplo, en un sistema con cuatro núcleos lógicos, la terminal debería verse así:

```
Started 14107
Started 14099
Started 14102
Started 14101
```

Si ahora intentamos acceder nuevamente a nuestro servidor usando la URL `http://localhost:8080`, deberíamos notar que cada solicitud devolverá un mensaje con un PID diferente, lo que significa que estas solicitudes han sido manejadas por diferentes workers, confirmando que la carga se está distribuyendo entre ellos.

Ahora, podemos intentar realizar una prueba de carga de nuestro servidor nuevamente con Autocannon para descubrir el aumento de rendimiento obtenido al escalar nuestra aplicación en múltiples procesos. Como referencia, en nuestra máquina, que tiene 8 núcleos, vimos un aumento de rendimiento de aproximadamente **5,3 veces** (1.600 req/seg frente a 300 req/seg), ¡una ganancia impresionante para un cambio tan simple!

##### Resiliencia y disponibilidad con el módulo cluster

Debido a que todos los workers son procesos separados, se pueden terminar o volver a generar según las necesidades de un programa, sin afectar a otros workers. Mientras haya algunos workers con vida, el servidor continuará aceptando conexiones. Si no hay workers activos, las conexiones existentes se interrumpirán y se rechazarán las nuevas conexiones. Node.js no administra automáticamente la cantidad de workers; es responsabilidad de la aplicación administrar el grupo de workers según sus propias necesidades.

Como ya mencionamos, esta forma de escalar una aplicación también aporta otras ventajas, en particular, la capacidad de mantener un cierto nivel de servicio, incluso en presencia de fallos o bloqueos. Esta propiedad también se conoce como **resiliencia** y contribuye a la **disponibilidad** de un sistema.

Al iniciar múltiples instancias de la misma aplicación, estamos creando un sistema redundante, lo que significa que si una instancia se cae por cualquier motivo, todavía tenemos otras instancias listas para atender solicitudes. Este patrón es sencillo de implementar utilizando el módulo `node:cluster`. ¡Veamos cómo funciona!

Tomemos el código de la sección anterior como punto de partida. En particular, modifiquemos el módulo `app.js` para que falle después de un intervalo de tiempo aleatorio:

```javascript
// ...
} else {
  // Inside our worker block
  setInterval(
    () => {
      if (Math.random() < 0.5) {
        throw new Error(`Ooops... ${process.pid} crashed!`)
      }
    },
    Math.ceil(Math.random() * 3) * 1000
  )
  // ...
```

Con este cambio implementado, cada pocos segundos, cada worker comprueba si debe bloquearse. A veces lo hace, a veces no. El resultado es impredecible, un poco caótico y sorprendentemente cercano a lo que podría suceder en un sistema de producción real. Los fallos a menudo ocurren sin previo aviso, provocados por casos extremos, límites de recursos o errores que solo se manifiestan bajo ciertas condiciones.

¿Puedes ver cómo esto podría generar problemas? Si nadie vigila estos procesos y nada los reinicia, toda la aplicación eventualmente dejará de funcionar. Puedes ver esto en acción simplemente iniciando esta versión del servidor y esperando unos segundos.

Si estuviéramos ejecutando solo una instancia de la aplicación, la situación sería significativamente peor. Un bloqueo significaría una parada completa, sin ningún proceso restante para manejar las solicitudes entrantes. A menos que exista una herramienta de monitoreo externa para reiniciar la aplicación y lo haga rápidamente, los usuarios experimentarían tiempo de inactividad. Incluso en el mejor de los casos, habría una brecha entre el fallo y la recuperación, durante la cual las solicitudes podrían retrasarse o perderse por completo. Ese tipo de interrupción está lejos de ser ideal, especialmente para usuarios que esperan una experiencia rápida y confiable.

Sin embargo, con múltiples instancias implementadas, introducimos resiliencia. Cuando un worker se cae, los demás continúan atendiendo solicitudes sin interrupciones. Esta configuración simple nos ayuda a construir un sistema que esté mejor preparado para manejar la incertidumbre y el desorden de los fallos del mundo real.

Al usar el módulo `node:cluster` para generar múltiples procesos, ya estamos dando un paso sólido hacia la construcción de un servidor web más resiliente. La siguiente mejora es garantizar que cada vez que un proceso worker salga con un error, se reemplace de inmediato. De esta manera, el sistema puede recuperarse de bloqueos accidentales de procesos sin intervención manual. Actualicemos `app.js` para vigilar las salidas de los workers y generar automáticamente un reemplazo cuando sea necesario:

```javascript
// ...
if (cluster.isPrimary) {
  // ...
  cluster.on('exit', (worker, code) => {
    if (code !== 0 && !worker.exitedAfterDisconnect) {
      console.log(
        `Worker ${worker.process.pid} crashed. Starting a new worker`
      )
      cluster.fork()
    }
  })
} else {
  // ...
}
```

En el código actualizado, cada vez que el proceso primario recibe un evento `'exit'` de un worker, comprueba si la terminación fue intencional o el resultado de un bloqueo. Esto se determina inspeccionando el código de salida y la bandera `worker.exitedAfterDisconnect`, que indica si el worker fue desconectado explícitamente por el proceso primario. Si la terminación se debió a un error, se inicia inmediatamente un nuevo worker para reemplazarlo.

Lo importante aquí es que mientras se reemplaza el worker bloqueado, los workers restantes continúan manejando las solicitudes entrantes. Esto ayuda a mantener la disponibilidad de la aplicación, incluso cuando los procesos individuales fallan y se recuperan.

Para verificar este comportamiento, podemos ejecutar una prueba de estrés utilizando una herramienta como Autocannon. Cuando se completa la prueba, una de las métricas clave para examinar es la cantidad de solicitudes fallidas:

```
[...]
11k requests in 10.02s, 1.47 MB read
29 errors (0 timeouts)
```

Este resultado muestra que la aplicación respondió con éxito a la mayoría de las solicitudes, a pesar de los bloqueos continuos. Aproximadamente el **99,7% de las solicitudes se completaron con éxito**, lo cual es una fuerte indicación de que nuestra solución resiste bajo presión.

Por supuesto, los resultados variarán según la cantidad de procesos worker que se estén ejecutando y la frecuencia con la que se bloqueen durante la prueba. Aun así, esto nos da una idea realista de cómo se comporta el sistema en un entorno propenso a fallos.

En este ejemplo, la mayoría de las solicitudes fallidas son causadas por bloqueos que interrumpen las conexiones existentes (solicitudes en curso o *in-flight requests*). Desafortunadamente, hay poco que podamos hacer para evitarlas una vez que un proceso termina abruptamente. Pero la conclusión importante es que la aplicación se recupera automáticamente y sigue respondiendo en general. Para un sistema que falla con frecuencia por diseño, este nivel de disponibilidad es más que aceptable y valida la efectividad de nuestro enfoque.

> [!NOTE]
> En muchos sistemas del mundo real, los reintentos del lado del cliente son esenciales para la confiabilidad, permitiendo que las solicitudes fallidas se reintenten automáticamente. Pero los reintentos tienen riesgos. Si una solicitud modifica datos y falla a mitad de camino, reintentar sin cuidado puede generar resultados inconsistentes o duplicados. Por eso la **idempotencia** es tan importante. Por ejemplo, un pago puede tener éxito en el servidor pero fallar al devolver una respuesta, lo que lleva al cliente a reintentar y cobrar dos veces al cliente. Las operaciones críticas como pagos, depósitos o procesamiento de pedidos deben diseñarse para que los reintentos sean seguros, utilizando técnicas como manejo idempotente, transacciones, reversiones (*rollbacks*), identificadores de solicitud o patrones de transacciones distribuidas como Saga ([nodejsdp.link/cloud-design-patterns](https://nodejsdp.link/cloud-design-patterns)).

##### Reinicio con cero tiempo de inactividad

Hasta ahora, nos hemos centrado en manejar bloqueos inesperados mediante la ejecución de múltiples procesos worker. Esto mejora la resiliencia y ayuda a garantizar que las solicitudes sigan siendo atendidas incluso cuando los procesos individuales fallan. Pero, ¿qué sucede si necesitamos reiniciar toda la aplicación del servidor, por ejemplo, cuando queremos desplegar una nueva versión en producción?

Durante un despliegue, una aplicación Node.js normalmente necesita reiniciarse. Se aprovisiona la nueva versión del código, se detiene el servidor y luego se inicia nuevamente para que pueda cargar y ejecutar la aplicación actualizada. Mientras el servidor se detiene y se reinicia, hay un breve período en el que no se pueden manejar solicitudes. Esto da como resultado una pérdida de disponibilidad. Eso puede ser aceptable para un blog personal o un proyecto paralelo, pero no es adecuado para sistemas de producción con acuerdos de nivel de servicio (SLAs) y para aplicaciones que se actualizan varias veces al día a través de canalizaciones de entrega continua.

Para evitar este problema, debemos implementar **reinicios con cero tiempo de inactividad** (*zero-downtime restarts*). Esta técnica permite que la aplicación se actualice y reinicie sin interrumpir su capacidad para atender solicitudes. Conserva la disponibilidad durante todo el proceso de despliegue.

Con el módulo `node:cluster`, implementar esta técnica es relativamente sencillo. El patrón consiste en reiniciar los workers uno a la vez, para que los workers restantes puedan continuar manejando solicitudes y mantener la aplicación disponible durante todo el proceso.

Agreguemos esta función a nuestro servidor en cluster. Todo lo que debemos hacer es incluir un poco de lógica adicional en el proceso primario para administrar los reinicios de manera controlada y secuencial:

```javascript
import { once } from 'node:events'
// ...
if (cluster.isPrimary) {
  // ...
  process.on('SIGUSR2', async () => { // 1
    const workers = Object.values(cluster.workers)
    for (const worker of workers) { // 2
      console.log(`Stopping worker: ${worker.process.pid}`)
      worker.disconnect() // 3
      await once(worker, 'exit')
      if (!worker.exitedAfterDisconnect) {
        continue
      }
      const newWorker = cluster.fork() // 4
      await once(newWorker, 'listening') // 5
    }
  })
} else {
  // ...
}
```

Así es como funciona el bloque de código anterior:
1. La acción de reinicio de los workers se activa cuando la aplicación recibe la señal `SIGUSR2`. Ten en cuenta que estamos usando una función asíncrona para implementar el controlador de eventos, ya que necesitaremos realizar algunas tareas asíncronas aquí.
2. Cuando se recibe una señal `SIGUSR2`, iteramos sobre todos los valores del objeto `cluster.workers`. Cada elemento es un objeto worker que podemos usar para interactuar con un worker determinado activo actualmente en el grupo.
3. Lo primero que hacemos para el worker actual es invocar `worker.disconnect()`, que detiene el worker de forma elegante (*gracefully*). Esto significa que si el worker está manejando solicitudes actualmente, no se interrumpirá abruptamente; en su lugar, esperará a que complete el manejo de la solicitud. El worker sale solo después de completar todas las solicitudes en curso.
4. Cuando el proceso finalizado sale, podemos generar un nuevo worker.
5. Esperamos a que el nuevo worker esté listo y escuchando nuevas conexiones antes de proceder a reiniciar el siguiente worker.

Dado que nuestro programa hace uso de señales Unix, no funcionará correctamente en sistemas Windows (a menos que utilices el Subsistema de Windows para Linux). Las señales son el mecanismo más simple para implementar nuestra solución, aunque otros enfoques incluyen escuchar un comando proveniente de un socket, una tubería (*pipe*) o la entrada estándar.

Ahora, podemos probar nuestro reinicio sin tiempo de inactividad ejecutando la aplicación y luego enviando una señal `SIGUSR2`. Primero necesitamos obtener el PID del proceso primario con:

```bash
ps -af
```

El proceso primario debe ser el padre de un conjunto de procesos node. Una vez que tengamos el PID, podemos enviarle la señal:

```bash
kill -SIGUSR2 <PID>
```

Ahora, la salida de la aplicación debería mostrar algo como esto:

```
Restarting workers
Stopping worker: 19389
Started 19407
Stopping worker: 19390
Started 19409
```

Podemos intentar usar Autocannon nuevamente para verificar que no tenemos ningún impacto considerable en la disponibilidad de nuestra aplicación durante el reinicio de los workers.

> [!TIP]
> **pm2** ([nodejsdp.link/pm2](https://nodejsdp.link/pm2)) es una utilidad basada en `node:cluster` que ofrece balanceo de carga, monitoreo de procesos, reinicios sin tiempo de inactividad y otras ventajas listas para usar.

##### Manejo de comunicaciones con estado

El módulo `node:cluster` no funciona bien con comunicaciones con estado (*stateful*) donde el estado de la aplicación no se comparte entre las diversas instancias. Esto se debe a que diferentes solicitudes que pertenecen a la misma sesión pueden ser manejadas potencialmente por una instancia diferente de la aplicación. Este no es un problema limitado solo al módulo `node:cluster`, sino que, en general, se aplica a cualquier tipo de algoritmo de balanceo de carga sin estado.

```
Usuario ---> [ Balanceador de carga ] ---> [ Instancia A ] (Guarda sesión en memoria)
                                      \--> [ Instancia B ] (No conoce la sesión)
```

*Figura 12.3: Un ejemplo de problema con una aplicación con estado detrás de un balanceador de carga.*

En este ejemplo, el usuario envía una solicitud para iniciar sesión. La solicitud es manejada por una de las instancias de la aplicación (Instancia A), que almacena el resultado de la autenticación localmente en la memoria. Esto significa que solo la Instancia A sabe que el usuario está autenticado.

Más tarde, cuando el usuario envía otra solicitud, el balanceador de carga podría reenviarla a una instancia diferente, como la Instancia B. Dado que la Instancia B no tiene ningún registro de la autenticación del usuario, lo trata como no autenticado y rechaza la solicitud.

Afortunadamente, existen soluciones simples que pueden ayudar a resolver este problema.

###### Compartir el estado entre múltiples instancias

La primera opción para escalar una aplicación utilizando comunicaciones con estado es encontrar una forma de compartir de manera confiable el estado entre todas las instancias.

Esto se puede lograr con un almacén de datos compartido, como una base de datos relacional como PostgreSQL ([nodejsdp.link/postgresql](https://nodejsdp.link/postgresql)), una base de datos de documentos como MongoDB ([nodejsdp.link/mongodb](https://nodejsdp.link/mongodb)) o CouchDB ([nodejsdp.link/couchdb](https://nodejsdp.link/couchdb)), o un almacén en memoria como Redis ([nodejsdp.link/redis](https://nodejsdp.link/redis)) o Memcached ([nodejsdp.link/memcached](https://nodejsdp.link/memcached)).

```
Usuario ---> [ Balanceador de carga ] ---> [ Instancia A ] ---                                      \--> [ Instancia B ] ----+--> [ Almacén de datos compartido (Redis / DB) ]
                                       \-> [ Instancia C ] ---/
```

*Figura 12.4: Aplicación detrás de un balanceador de carga utilizando un almacén de datos compartido.*

El único inconveniente de utilizar un almacén compartido para el estado de comunicación es que la aplicación de este patrón puede requerir una cantidad significativa de refactorización del código base (por ejemplo, reemplazar el almacenamiento en memoria por llamadas asíncronas al almacén compartido).

###### Balanceo de carga persistente (Sticky load balancing)

En los casos en que la refactorización no sea factible debido a demasiados cambios requeridos o limitaciones de tiempo estrictas, podemos confiar en una solución menos invasiva: **balanceo de carga persistente** o sesiones persistentes (*sticky sessions*). En este enfoque, el balanceador de carga asegura que todas las solicitudes asociadas con una sesión determinada siempre se enruten a la misma instancia de la aplicación.

```
Usuario 1 (Sesión A) ---> [ Balanceador de carga ] ===(Ruta fija)===> [ Instancia A ]
Usuario 2 (Sesión B) ---> [ Balanceador de carga ] ===(Ruta fija)===> [ Instancia B ]
```

*Figura 12.5: Un ejemplo que ilustra cómo funciona el balanceo de carga persistente.*

Cuando el balanceador de carga recibe una solicitud inicial que inicia una nueva sesión, selecciona una instancia de la aplicación y almacena una asignación entre la sesión y esa instancia (a menudo mediante una cookie de sesión). En solicitudes posteriores de la misma sesión, el balanceador de carga utiliza esta asignación para enrutar el tráfico directamente a la misma instancia.

Un enfoque alternativo es utilizar la dirección IP del cliente haciéndola pasar por una función hash. Sin embargo, esto puede no ser confiable para clientes con direcciones IP que cambian con frecuencia (como usuarios móviles).

Una gran limitación del balanceo de carga persistente es que elimina muchos de los beneficios de tener un sistema redundante: si una instancia se cae, los usuarios vinculados a ella pierden su sesión y experimentan fallos. Por esta razón, generalmente se recomienda almacenar los datos de sesión en un almacén de datos compartido o utilizar métodos de autenticación sin estado como los **JSON Web Tokens (JWTs)**.

Un ejemplo real de una biblioteca que normalmente necesita balanceo de carga persistente es Socket.IO ([nodejsdp.link/socket-io](https://nodejsdp.link/socket-io)), que mantiene una conexión persistente y almacena datos de sesión en la memoria durante la fase de negociación de transporte HTTP de larga duración.

#### Escalado con un proxy inverso

El módulo `node:cluster`, aunque muy conveniente y simple de usar, no es la única opción que tenemos para escalar una aplicación web Node.js. Las técnicas tradicionales a menudo se prefieren porque ofrecen más control y potencia en entornos de producción de alta disponibilidad.

La alternativa al uso de `node:cluster` es iniciar múltiples instancias independientes de la misma aplicación que se ejecutan en diferentes puertos o máquinas y luego usar un **proxy inverso** (*reverse proxy* o pasarela) para proporcionar acceso a esas instancias, distribuyendo el tráfico entre ellas.

```
                              /---> [ Instancia 1 (Puerto 8081) ] (Máquina 1)
Clientes ---> [ Proxy Inverso /     [ Instancia 2 (Puerto 8082) ] (Máquina 1)
              Balanceador de Carga ] ---> [ Instancia 3 (Puerto 8081) ] (Máquina 2)
                              \---> [ Instancia 4 (Puerto 8082) ] (Máquina 2)
```

*Figura 12.6: Una configuración típica multiproceso y multimáquina con un proxy inverso actuando como balanceador de carga.*

Razones para elegir un proxy inverso:
- Puede distribuir la carga entre varias máquinas, no solo varios procesos.
- Admite sesiones persistentes (*sticky sessions*) de fábrica.
- Puede enrutar solicitudes a cualquier servidor disponible, independientemente de su lenguaje de programación o plataforma.
- Ofrece algoritmos de balanceo de carga más avanzados.
- Proporciona funciones adicionales como reescritura de URLs, almacenamiento en caché, terminación SSL, protección DoS y servicio de archivos estáticos.

> [!NOTE]
> **Patrón:** Utiliza un proxy inverso para balancear la carga de una aplicación entre múltiples instancias que se ejecutan en diferentes puertos o máquinas.

Soluciones populares:
- **Nginx** ([nodejsdp.link/nginx](https://nodejsdp.link/nginx))
- **HAProxy** ([nodejsdp.link/haproxy](https://nodejsdp.link/haproxy))
- **Caddy** ([nodejsdp.link/caddy](https://nodejsdp.link/caddy))
- **Proxies basados en Node.js**
- **Proxies basados en la nube** (AWS ALB, Cloud Load Balancing, etc.)
- **Watt** ([nodejsdp.link/wattpm](https://nodejsdp.link/wattpm))

##### Balanceo de carga con Nginx

Hagamos que el puerto de nuestro servidor `app.js` sea configurable a través de variables de entorno o argumentos de línea de comandos:

```javascript
// app.js
import { createServer } from 'node:http'

const server = createServer((_req, res) => {
  let i = 1e7
  while (i > 0) {
    i--
  }

  console.log(`Handling request from ${process.pid}`)
  res.end(`Hello from ${process.pid}
`)
})

const port = Number.parseInt(process.env.PORT || process.argv[2]) || 8080
server.listen(port, () => console.log(`Started at ${process.pid}`))
```

Para supervisar los procesos y reiniciarlos en caso de fallo, usaremos **pm2**:

```bash
npm install pm2@^6.0.6 -g
```

Iniciamos cuatro instancias en diferentes puertos supervisadas por pm2:

```bash
pm2 start --namespace 'app' --name 'app1' app.js -- 8081
pm2 start --namespace 'app' --name 'app2' app.js -- 8082
pm2 start --namespace 'app' --name 'app3' app.js -- 8083
pm2 start --namespace 'app' --name 'app4' app.js -- 8084
```

Podemos verificar el estado con `pm2 ls`.

Ahora creamos el archivo `nginx.conf`:

```nginx
daemon off; ## 1
error_log /dev/stderr info; ## 2

events { ## 3
  worker_connections 2048;
}

http { ## 4
  access_log /dev/stdout;

  upstream my-load-balanced-app {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
    server 127.0.0.1:8084;
  }

  server {
    listen 8080;

    location / {
      proxy_pass http://my-load-balanced-app;
    }
  }
}
```

Detalles de la configuración:
1. `daemon off;` ejecuta Nginx en primer plano en la terminal actual.
2. `error_log` y `access_log` transmiten los registros a la salida y error estándar.
3. `events` configura las conexiones máximas simultáneas por worker (2048).
4. `http` define el grupo upstream `my-load-balanced-app` con los 4 servidores backend y la directiva `proxy_pass`.

Iniciamos Nginx con:

```bash
nginx -c ${PWD}/nginx.conf
```

#### Escalado horizontal dinámico

En la nube, la capacidad de la aplicación a menudo se ajusta en tiempo real según el tráfico (escalado elástico). El desafío es que el balanceador de carga debe conocer siempre qué servidores están disponibles a medida que se agregan o eliminan instancias.

##### Uso de un registro de servicios

El patrón **Service Registry** utiliza un repositorio central para rastrear los servicios disponibles, cuántas instancias se están ejecutando y cuáles son sus direcciones de red.

```
                    [ Registro de Servicios (Consul / etcd) ]
                                 ^             ^
                   (Registran)   |             | (Consulta topología)
                                 |             |
[ Instancia API 1 ] -------------+     [ Balanceador de Carga ] <--- Peticiones
[ Instancia API 2 ] -------------/             |
[ Instancia Web 1 ] ---------------------------+
```

*Figura 12.7: Una arquitectura multiservicio con un balanceador de carga configurado dinámicamente mediante un registro de servicios.*

> [!NOTE]
> **Patrón (Service Registry):** Utiliza un repositorio central para almacenar una vista siempre actualizada de los servidores y servicios disponibles en un sistema.

##### Implementación de un balanceador de carga dinámico

Para este ejercicio, utilizaremos **Consul** ([nodejsdp.link/consul](https://nodejsdp.link/consul)) junto con los paquetes `httpxy` y `portfinder`.

Veamos la implementación del servicio (`app.js`):

```javascript
// app.js
import { createServer } from 'node:http'
import { randomUUID } from 'node:crypto'
import portfinder from 'portfinder' // v1.0.37
import { ConsulClient } from './consul.js'

const serviceType = process.argv[2] // 1
if (!serviceType) {
  console.error('Usage: node app.js <service-type>')
  process.exit(1)
}

const consulClient = new ConsulClient() // 2
const port = await portfinder.getPort() // 3
const address = process.env.ADDRESS || 'localhost'
const serviceId = randomUUID()

async function registerService() { // 4
  await consulClient.registerService({
    id: serviceId,
    name: serviceType,
    address,
    port,
    tags: [serviceType],
  })
  console.log(
    `${serviceType} registered successfully as ${serviceId} ` +
      `on ${address}:${port}`
  )
}

async function unregisterService(err) { // 5
  err && console.error(err)
  console.log(`deregistering ${serviceId}`)
  try {
    await consulClient.deregisterService(serviceId)
  } catch (deregisterError) {
    console.error(
      `Failed to deregister service: ` +
        `${deregisterError.message}`
    )
  }
  process.exit(err ? 1 : 0)
}

process.on('uncaughtException', unregisterService) // 6
process.on('SIGINT', unregisterService)

const server = createServer((_req, res) => { // 7
  // Simulate some processing time
  let i = 1e7
  while (i > 0) {
    i--
  }

  console.log(`Handling request from ${process.pid}`)
  res.end(`${serviceType} response from ${process.pid}
`)
})

server.listen(port, address, async () => {
  console.log(
    `Started ${serviceType} on port ${port} with PID ${process.pid}`
  )
  await registerService()
})
```

A continuación, implementamos el balanceador de carga (`loadBalancer.js`):

```javascript
// loadBalancer.js
import { createServer } from 'node:http'
import { createProxyServer } from 'httpxy' // v0.1.7
import { ConsulClient } from './consul.js'

const routing = [ // 1
  {
    path: '/api',
    service: 'api-service',
    index: 0,
  },
  {
    path: '/',
    service: 'webapp-service',
    index: 0,
  },
]

const consulClient = new ConsulClient() // 2
const proxy = createProxyServer()

const server = createServer(async (req, res) => { // 3
  const route = routing.find(route => req.url.startsWith(route.path))
  try {
    const services = await consulClient.getAllServices() // 4
    const servers = Object.values(services).filter(service =>
      service.Tags.includes(route.service)
    )

    if (servers.length > 0) {
      route.index = (route.index + 1) % servers.length // 5
      const server = servers[route.index]
      const target = `http://${server.Address}:${server.Port}`
      proxy.web(req, res, { target })
      return
    }
  } catch (err) {
    console.error(err)
  }

  // if servers not found or error occurs
  res.writeHead(502)
  return res.end('Bad gateway')
})

server.listen(8080, () => {
  console.log('Load balancer started on port 8080')
})
```

Iniciamos Consul en modo desarrollo:

```bash
consul agent -dev
```

Iniciamos el balanceador de carga:

```bash
node loadBalancer.js
```

Y luego iniciamos varias instancias de servicios en terminales separadas:

```bash
node app.js api-service
node app.js api-service
node app.js webapp-service
```

El balanceador de carga descubrirá automáticamente las instancias registradas en Consul y balanceará el tráfico entre ellas.

#### Balanceo de carga peer-to-peer

En lugar de enrutar las solicitudes a través de un balanceador de carga central, el cliente o servicio origen puede manejar la distribución directamente si conoce las direcciones de las instancias de destino. Esto se conoce como **balanceo de carga peer-to-peer** (o del lado del cliente).

```
Balanceo Centralizado:
Cliente ---> [ Balanceador de Carga ] ---> [ Instancia 1 ] / [ Instancia 2 ]

Balanceo Peer-to-Peer (Lado del Cliente):
Cliente ===(Petición 1)===> [ Instancia 1 ]
Cliente ===(Petición 2)===> [ Instancia 2 ]
```

*Figura 12.8: Balanceo de carga centralizado versus balanceo de carga peer-to-peer.*

Propiedades del balanceo peer-to-peer:
- Reduce la complejidad de la infraestructura al eliminar un intermediario.
- Permite una comunicación más rápida al enviar mensajes directamente al destino.
- Elimina el cuello de botella del balanceador central.
- Requiere que el cliente conozca la topología y gestione la lógica de balanceo.

##### Implementación de un cliente HTTP con balanceo de carga

Veamos el módulo `balancedRequest.js`:

```javascript
// balancedRequest.js
const servers = [
  { host: 'localhost', port: 8081 },
  { host: 'localhost', port: 8082 },
]

let i = 0

export function balancedRequest(url, fetchOptions = {}) {
  i = (i + 1) % servers.length
  const server = servers[i]
  const rewrittenUrl = new URL(url, `http://${server.host}:${server.port}`)
  rewrittenUrl.host = `${server.host}:${server.port}`
  return fetch(rewrittenUrl.toString(), fetchOptions)
}
```

Uso en `client.js`:

```javascript
// client.js
import { balancedRequest } from './balancedRequest.js'

for (let i = 0; i < 10; i++) {
  const response = await balancedRequest(`/?request=${i}`)
  const body = await response.text()
  console.log(
    `Request ${i} completed
Status: ${response.status}
Body: ${body}`
  )
}
```

Y el servidor de prueba (`app.js`):

```javascript
// app.js
import { createServer } from 'node:http'

const { pid } = process
const server = createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`)
  const searchParams = url.searchParams
  console.log(
    `Handling request ${searchParams.get('request')} from ${pid}`
  )
  res.end(`Hello from ${pid}
`)
})

const port = Number.parseInt(process.env.PORT || process.argv[2]) || 8080
server.listen(port, () => console.log(`Started at ${pid}`))
```

---

### Sección 3: Escalado de aplicaciones usando contenedores

Los contenedores y las plataformas de orquestación como **Kubernetes** permiten descargar preocupaciones como el balanceo de carga, el escalado elástico y la alta disponibilidad directamente en la plataforma.

#### ¿Qué es un contenedor?

Un contenedor Linux, según la Open Container Initiative (OCI), es *"una unidad estándar de software que empaqueta el código y todas sus dependencias para que la aplicación se ejecute de manera rápida y confiable de un entorno informático a otro"*. Herramientas como **Docker** ([nodejsdp.link/docker](https://nodejsdp.link/docker)) facilitan la creación y ejecución de contenedores compatibles con OCI.

##### Creación y ejecución de un contenedor con Docker

Creamos un servidor web simple (`app.js`):

```javascript
// app.js
import { createServer } from 'node:http'
import { hostname } from 'node:os'

const version = 1

const server = createServer((_req, res) => {
  res.end(`Hello from ${hostname()} (v${version})`)
})

server.listen(8080)
```

El archivo `package.json`:

```json
{
  "name": "my-simple-app",
  "version": "1.0.0",
  "main": "app.js",
  "type": "module",
  "scripts": {
    "start": "node app.js"
  }
}
```

Y el `Dockerfile`:

```dockerfile
FROM node:24-slim
EXPOSE 8080
COPY app.js package.json /app/
WORKDIR /app
CMD ["npm", "start"]
```

Construimos y ejecutamos la imagen etiquetada:

```bash
docker build -t hello-web:v1 .
docker run -it -p 8080:8080 hello-web:v1
```

#### ¿Qué es Kubernetes?

Kubernetes es un orquestador de contenedores de código abierto que proporciona:
- Agrupación de servidores en clusters.
- Alta disponibilidad y reinicios automáticos de contenedores que fallan.
- Descubrimiento de servicios y balanceo de carga interno.
- Almacenamiento persistente.
- Despliegues y actualizaciones progresivas sin tiempo de inactividad (*rollouts* y *rollbacks*).
- Gestión de configuración y secretos.

Se basa en un modelo declarativo donde defines el estado deseado mediante objetos de Kubernetes y el plano de control se encarga de mantenerlo.

##### Despliegue y escalado de una aplicación en Kubernetes

Utilizando `minikube` y `kubectl`:

Construimos la imagen en el entorno de minikube:

```bash
docker build -t hello-web:v1 .
```

###### Creación de un despliegue de Kubernetes

```bash
kubectl create deployment hello-web --image=hello-web:v1
```

Exponemos el despliegue con un balanceador de carga:

```bash
kubectl expose deployment hello-web --type=LoadBalancer --port=8080
minikube service hello-web
```

###### Escalado de un despliegue de Kubernetes

Para escalar a 5 réplicas:

```bash
kubectl scale --replicas=5 deployment hello-web
```

Verificamos el estado:

```bash
kubectl get deployments
kubectl get pods
```

###### Despliegues progresivos (Rollouts) en Kubernetes

Cambiamos a `const version = 2` en `app.js`, construimos la imagen `hello-web:v2` y actualizamos el despliegue:

```bash
docker build -t hello-web:v2 .
kubectl set image deployment/hello-web hello-web=hello-web:v2
```

Kubernetes reemplazará los contenedores uno a uno de forma progresiva sin tiempo de inactividad.

---

### Sección 4: Descomposición de aplicaciones complejas

Ahora nos centraremos en el **eje Y** del cubo de escala: descomponer una aplicación según la funcionalidad empresarial.

#### Arquitectura monolítica

Una aplicación monolítica contiene todos los servicios en el mismo código base y se ejecuta en un solo proceso (cuando no está clonada).

```
+-------------------------------------------------------------------+
|                        APLICACIÓN MONOLÍTICA                      |
|                                                                   |
| [ Frontend Tienda ]                      [ Frontend Admin ]       |
|          \                                      /                 |
|           v                                    v                  |
|     +-----------+  +--------+  +----------+  +--------+  +------+ |
|     | Productos |  | Carrito|  | Checkout |  | Buscar |  | Auth | |
|     +-----------+  +--------+  +----------+  +--------+  +------+ |
|                           \          /                            |
|                            v        v                             |
|                   [ Base de Datos Compartida ]                    |
+-------------------------------------------------------------------+
```

*Figura 12.9: Ejemplo de arquitectura monolítica.*

Desafíos del monolito:
- Un fallo en cualquier módulo puede hacer caer toda la aplicación.
- El acoplamiento fuerte suele filtrarse entre módulos a medida que el proyecto crece.
- Escalar equipos y bases de código grandes se vuelve cada vez más difícil.

#### La arquitectura de microservicios

El principio central es: **¡no construyas sistemas Node.js a gran escala!** En su lugar, divide el sistema en servicios pequeños, independientes y enfocados, cada uno con su propia responsabilidad y su propia base de datos.

##### Un ejemplo de arquitectura de microservicios

```
[ Frontend Tienda ]               [ Frontend Admin ]
        |                                 |
        v                                 v
+---------------+  +---------------+  +---------------+  +---------------+
| Servicio      |  | Servicio      |  | Servicio      |  | Servicio      |
| Productos     |  | Carrito       |  | Checkout      |  | Auth          |
| [DB Productos]|  | [DB Carrito]  |  | [DB Checkout] |  | [DB Auth]     |
+---------------+  +---------------+  +---------------+  +---------------+
        ^                  ^                  ^                  ^
        :..................:..................:..................:
                   (Comunicación entre servicios)
```

*Figura 12.10: Ejemplo de implementación de un sistema de comercio electrónico usando arquitectura de microservicios.*

> [!NOTE]
> **Patrón (Arquitectura de microservicios):** Divide una aplicación compleja creando varios servicios pequeños e independientes.

##### Microservicios: ventajas y desventajas

- **Cada servicio es prescindible:** Si un servicio falla, los demás continúan funcionando. Los servicios se pueden reescribir o actualizar individualmente.
- **Reutilización entre plataformas y lenguajes:** Cada servicio puede usar la tecnología más adecuada y exponer APIs estándar.
- **Escalado independiente:** Cada servicio puede clonarse en el eje X según su demanda específica.
- **Desafíos:** Mayor complejidad en la integración, despliegue, monitoreo y consistencia de datos entre servicios.

#### Patrones de integración en una arquitectura de microservicios

##### El proxy de API

Un **API Proxy** (o **API Gateway**) actúa como un punto de entrada único para los clientes, enrutando las solicitudes a los microservicios correspondientes y gestionando tareas transversales como autenticación, limitación de tasa y almacenamiento en caché.

```
Clientes ---> [ API Gateway / Proxy ] ---> [ Servicio Productos ]
                                      ---> [ Servicio Carrito ]
                                      ---> [ Servicio Checkout ]
```

*Figura 12.11: Uso del patrón API Proxy en una aplicación de comercio electrónico.*

##### Orquestación de APIs

La orquestación de APIs compone y coordina múltiples servicios para implementar flujos de trabajo de negocio específicos o agregar datos en una sola respuesta.

```
Frontend Tienda ---> [ Orquestador de API: completeCheckout() ]
                               |
            +------------------+------------------+
            | 1. Pagar         | 2. Vaciar        | 3. Actualizar
            v                  v                  v
    [Servicio Checkout] [Servicio Carrito] [Servicio Productos]
```

*Figura 12.12: Uso de una capa de orquestación para interactuar con múltiples microservicios.*

> [!TIP]
> El patrón **Backend for Frontend (BFF)** crea un backend dedicado y personalizado para cada interfaz de cliente específica (web, móvil, etc.), optimizando el formato y la cantidad de datos enviados.

##### Integración con un broker de mensajes

Para evitar el acoplamiento directo y los servicios "dios", un **broker de mensajes** permite una integración basada en eventos (Publish/Subscribe):

```
Store Frontend ---> [ Servicio Checkout ] ---> (Publica evento 'purchased')
                                                     |
                                            [ Message Broker ]
                                            /                                             (Evento 'purchased')       (Evento 'purchased')
                                          v                          v
                                [ Servicio Carrito ]       [ Servicio Productos ]
                                 (Vacía carrito)            (Reduce stock)
```

*Figura 12.14: Uso de un broker de mensajes para distribuir eventos en una aplicación de comercio electrónico.*

El Checkout emite un evento `purchased` al broker, y tanto el servicio de Carrito como el de Productos reaccionan de forma asíncrona y desacoplada.

---

### Sección 5: Resumen

En este capítulo, exploramos cómo diseñar arquitecturas Node.js que escalen tanto en capacidad como en complejidad:
- El **cubo de escala** nos mostró tres dimensiones: clonación (eje X), descomposición funcional (eje Y) y particionado de datos (eje Z).
- Aprendimos a clonar aplicaciones con `node:cluster`, gestionar fallos para lograr resiliencia y ejecutar **reinicios sin tiempo de inactividad**.
- Vimos cómo usar **proxies inversos** como Nginx, balanceadores dinámicos con registros de servicio como **Consul**, y balanceo **peer-to-peer**.
- Exploramos la orquestación con **Docker** y **Kubernetes**.
- Analizamos la descomposición en **microservicios** y patrones de integración como API Gateways, orquestadores, BFFs y brokers de mensajes.

En el próximo y último capítulo, exploraremos los patrones de mensajería e integración avanzada para sistemas distribuidos.

---

### Sección 6: Ejercicios

- **12.1 Biblioteca de libros escalable:** Revisa la aplicación de biblioteca construida en el libro y hazla más escalable utilizando el módulo `node:cluster` con reinicio automático de workers ante fallos, o despliégala en un cluster de Kubernetes.
- **12.2 Explorando el eje Z:** Construye una API REST que particione los datos de personas por la letra inicial de su nombre en tres grupos (A-D, E-P, Q-Z), ejecutando instancias separadas para cada grupo y exponiendo un único punto de entrada balanceado o mediante un orquestador.
- **12.3 Adicción a la música:** Diseña la arquitectura de un servicio de streaming de música (como Spotify) como una colección de microservicios desacoplados aplicando los principios vistos en este capítulo.

