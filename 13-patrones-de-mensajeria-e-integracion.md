# Parte 3: Patrones avanzados y arquitectura

## Capítulo 13: Patrones de mensajería e integración

Primero, ¡felicitaciones! Si has llegado hasta aquí, has explorado algunos de los patrones de diseño e ideas arquitectónicas más poderosos del ecosistema Node.js. No solo estás aprendiendo Node.js, sino que has estado progresando hacia la maestría, un patrón, un principio y un capítulo a la vez.

Este capítulo final añade la pieza que falta para que los sistemas distribuidos funcionen de verdad: la **integración**. Si la escalabilidad consiste en distribuir sistemas, la integración es lo que los mantiene unidos. En el capítulo anterior, aprendimos a dividir aplicaciones en múltiples procesos y máquinas. Para que esas partes sean útiles, a menudo necesitan comunicarse. Hay dos formas principales de hacerlo: usar almacenamiento compartido como una fuente central de verdad o intercambiar mensajes que contengan datos, eventos o comandos. La mensajería es lo que realmente desbloquea la escalabilidad y la flexibilidad en arquitecturas distribuidas.

Los mensajes están en todas partes: a través de Internet, entre procesos, dentro de las aplicaciones e incluso en los sistemas operativos. En arquitecturas distribuidas, la mensajería se refiere a patrones, herramientas y convenciones para intercambiar información de forma confiable a través de una red. Estos sistemas pueden ser centralizados con un broker o peer-to-peer, unidireccionales o de solicitud/respuesta (*request/reply*), y pueden usar colas para mayor confiabilidad.

Una de las obras más influyentes sobre este tema es *Enterprise Integration Patterns* de Gregor Hohpe y Bobby Woolf, que describe más de 60 patrones. Aquí nos centraremos en los más esenciales y en algunas alternativas modernas, siempre con la perspectiva de Node.js. Esto es lo que aprenderás:
- Los fundamentos de los sistemas de mensajería
- El patrón Publish/Subscribe
- Patrones de distribución de tareas y canalizaciones (*pipelines*)
- Modelos de comunicación Request/Reply

¡Sumerjámonos y concluyamos nuestro viaje por los patrones de diseño de Node.js con una de las áreas más poderosas y prácticas de la arquitectura moderna!

---

### Sección 1: Fundamentos de un sistema de mensajería

Al hablar de mensajes y sistemas de mensajería, hay cuatro elementos fundamentales a tener en cuenta:
1. **La dirección de la comunicación:** que puede ser solo unidireccional (*one-way*) o un intercambio de solicitud/respuesta (*request/reply*).
2. **El propósito del mensaje:** que también determina su contenido.
3. **El tiempo del mensaje:** que se puede enviar y recibir en contexto (sincrónicamente) o fuera de contexto (asíncronamente).
4. **La entrega del mensaje:** que puede ocurrir directamente o a través de un intermediario (*broker*).

A continuación formalizaremos estos aspectos.

#### Patrones unidireccionales frente a request/reply

El aspecto más fundamental de un sistema de mensajería es la dirección de la comunicación.

El patrón de comunicación más simple es cuando el mensaje se envía en una sola dirección desde un origen hasta un destino:

```
[ Origen ] ------------(Mensaje)------------> [ Destino ]
```

*Figura 13.1 – Comunicación unidireccional (One-way).*

Un ejemplo típico de comunicación unidireccional es un correo electrónico, un servidor web que envía un mensaje a un navegador conectado mediante Server-Sent Events (SSE), o un sistema que distribuye tareas a un grupo de workers.

En el otro extremo, tenemos el patrón de intercambio **Request/Reply**, donde el mensaje en una dirección siempre se corresponde (salvo condiciones de error) con un mensaje en la dirección opuesta:

```
[ Solicitante ] ------------(Solicitud)------------> [ Respondedor ]
[ Solicitante ] <-----------(Respuesta)------------ [ Respondedor ]
```

*Figura 13.2 – Patrón de intercambio de mensajes Request/Reply.*

El patrón Request/Reply se vuelve más complejo cuando el canal de comunicación es asíncrono o involucra múltiples nodos:

```
[ Iniciador ] ---(Solicitud)---> [ Nodo A ] ---> [ Nodo B ]
      ^                                                |
      |___________________(Respuesta)__________________|
```

*Figura 13.3 – Comunicación request/reply multinodo.*

En situaciones multinodo, lo que diferencia un patrón Request/Reply de un simple bucle unidireccional es la relación entre la solicitud y la respuesta, que se conserva en el iniciador y se procesa en el mismo contexto que la solicitud.

#### Tipos de mensajes

Podemos identificar tres tipos de mensajes según su propósito:

##### Mensajes de comando

El propósito de un **mensaje de comando** es desencadenar la ejecución de una acción o tarea en el receptor (esencialmente un objeto `Command` serializado). Contiene la información necesaria para ejecutar la tarea (el nombre de la operación y los argumentos). Se utiliza en llamadas RPC o peticiones HTTP RESTful donde cada verbo (GET, POST, PUT, DELETE) representa un comando específico.

##### Mensajes de evento

Un **mensaje de evento** se utiliza para notificar a otro componente que algo ha sucedido en el sistema. Contiene el tipo de evento y detalles sobre el contexto o actor involucrado. Permite notificaciones en tiempo real (por ejemplo, vía WebSockets) y mantiene desacoplados a los componentes distribuidos.

##### Mensajes de documento

El **mensaje de documento** está destinado principalmente a transferir datos entre componentes y máquinas sin instrucciones asociadas ni relación directa con un evento específico (por ejemplo, el resultado de una consulta a base de datos).

#### Semántica de entrega push versus pull

##### Entrega pull (iniciada por el consumidor)

El consumidor solicita activamente información al proveedor y controla cuándo obtener los datos:
- Navegadores que envían peticiones HTTP para páginas web.
- Aplicaciones que consultan bases de datos con SQL.
- Sistemas de monitoreo que sondean métricas a intervalos.
- Sistemas de colas en la nube como AWS SQS mediante sondeo largo (*long polling*).

##### Entrega push (iniciada por el productor)

El origen de datos envía actualizaciones proactivamente a los consumidores tan pronto como están disponibles:
- Webhooks activados por eventos externos.
- Brokers como RabbitMQ enviando mensajes a suscriptores.
- Feeds en tiempo real a través de WebSockets o SSE.
- Flujos de cambios (*change streams*) y disparadores de bases de datos.

##### Elección entre entrega push y pull

- **Push:** Ideal para necesidades inmediatas de baja latencia (chats, precios en vivo, alertas), pero añade complejidad en la gestión de conexiones y contrapresión (*backpressure*).
- **Pull:** Da el control al consumidor, facilitando la depuración y haciendo el uso de recursos más predecible.
- **Modelos híbridos:** Uso de notificaciones push para avisar de cambios, seguido de pull para obtener los datos completos.

#### Mensajería asíncrona, colas y streams

En la comunicación asíncrona, el emisor y el receptor están desacoplados en el tiempo.

Una **cola de mensajes (Message Queue - MQ)** almacena temporalmente los mensajes hasta que el consumidor esté listo para procesarlos:

```
[ Productor ] ---> [ Cola de Mensajes (Buffer) ] ---> [ Consumidor ]
```

*Figura 13.4 – Una cola de mensajes.*

Un **stream** (o registro de adición única / *log*) es una secuencia durable y ordenada de mensajes donde los registros no se eliminan al ser consumidos:

```
                  Posición Consumidor A
                            v
[ Reg 1 ] -> [ Reg 2 ] -> [ Reg 3 ] -> [ Reg 4 ] -> [ Reg 5 ] (Append-only)
               ^
     Posición Consumidor B
```

*Figura 13.5 – Un stream.*

Diferencias clave:
- **Colas:** Típicamente punto a punto; cada mensaje se entrega a un único consumidor y se elimina al procesarse.
- **Streams:** Aptos para múltiples suscriptores independientes; los mensajes se retienen y permiten reproducción histórica (*replay*).

#### Mensajería peer-to-peer o basada en brokers

```
Peer-to-Peer:
[ Nodo A ] <=========================> [ Nodo B ]
    ^                                      ^
    \================ [ Nodo C ] ==========/

Basado en Broker:
[ Nodo A ] ---\                      /---> [ Nodo B ]
               +-> [ Message Broker ]
[ Nodo C ] ---/                      \---> [ Nodo D ]
```

*Figura 13.6 – Comunicación peer-to-peer frente a mensajería con broker.*

- **Broker:** Desacopla emisores de receptores, unifica protocolos (AMQP, MQTT, STOMP), y ofrece enrutamiento, colas persistentes y monitorización.
- **Peer-to-peer:** Elimina el punto único de fallo, reduce la latencia y ofrece mayor flexibilidad.

---

### Sección 2: Patrón Publish/Subscribe

El patrón **Publish/Subscribe (Pub/Sub)** es el patrón de mensajería unidireccional más conocido (un patrón Observer distribuido). Los suscriptores registran su interés en categorías o temas específicos y el publicador emite mensajes sin saber de antemano quiénes son los destinatarios.

```
Peer-to-Peer Pub/Sub:
[ Publicador ] ------------(Topic: chat)------------> [ Suscriptor 1 ]
               \-----------(Topic: chat)------------> [ Suscriptor 2 ]

Broker-based Pub/Sub:
[ Publicador ] ---> [ Broker (Topic: chat) ] ---> [ Suscriptor 1 ]
                                             ---> [ Suscriptor 2 ]
```

*Figura 13.7 – Patrón de mensajería Publish/Subscribe.*

#### Construcción de una aplicación de chat minimalista en tiempo real

##### Implementación del lado del servidor

Creamos el servidor WebSocket básico (`index.js`):

```javascript
// index.js
import { createServer } from 'node:http'
import { WebSocketServer } from 'ws' // v8.18.2
import staticHandler from 'serve-handler' // v6.1.6

// serve static files
const server = createServer((req, res) => { // 1
  return staticHandler(req, res, { public: 'web' })
})

const wss = new WebSocketServer({ server }) // 2
wss.on('connection', client => {
  console.log('Client connected')
  client.on('message', msg => { // 3
    console.log(`Message: ${msg}`)
    broadcast(msg)
  })
})

function broadcast(msg) { // 4
  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg)
    }
  }
}

server.listen(process.argv[2] || 8080)
```

##### Implementación del lado del cliente

Creamos `web/index.html`:

```html
<body>
  <div>
    <div id="messages">
      <!-- Messages will be added here dynamically -->
    </div>
    <div>
      <form id="msgForm">
        <textarea
          id="msgBox"
          placeholder="Type a message..."
          rows="1"
        ></textarea>
        <button type="submit" id="sendButton">Send</button>
      </form>
    </div>
  </div>
  <script>
    const messagesArea = document.getElementById("messages")
    const messageInput = document.getElementById("msgBox")
    const sendButton = document.getElementById("sendButton")
    const form = document.getElementById("msgForm")

    // WebSocket connection
    const ws = new WebSocket(`ws://${window.document.location.host}`)

    // Receive messages from WebSocket
    ws.onmessage = async function (message) {
      const content = await message.data.text()

      // Create received message element
      const messageDiv = document.createElement("div")
      const messageContent = document.createElement("div")
      messageContent.textContent = content
      messageDiv.appendChild(messageContent)

      const messageTime = document.createElement("div")
      messageTime.textContent = new Date().toLocaleTimeString([], {
        hour: "2-digit",
        minute: "2-digit",
      })
      messageDiv.appendChild(messageTime)

      // Add to messages area
      messagesArea.appendChild(messageDiv)
    }

    // Send message function
    function sendMessage() {
      const content = messageInput.value.trim()
      if (!content || ws.readyState !== WebSocket.OPEN) return

      // Send via WebSocket
      ws.send(content)

      // Clear input
      messageInput.value = ""
      messageInput.style.height = "auto"
    }

    // Form submission handler
    form.addEventListener("submit", (event) => {
      event.preventDefault()
      sendMessage()
    })

    // Enter key to send (Shift+Enter for new line)
    messageInput.addEventListener("keydown", function (e) {
      if (e.key === "Enter" && !e.shiftKey) {
        e.preventDefault()
        sendMessage()
      }
    })

    // Initial scroll to bottom
    messagesArea.scrollTop = messagesArea.scrollHeight
  </script>
</body>
</html>
```

> [!IMPORTANT]
> El uso de `textContent` en lugar de `innerHTML` previene ataques de Cross-Site Scripting (XSS).

##### Ejecución y escalado de la aplicación de chat

Si ejecutamos múltiples instancias (`node index.js 8080` y `node index.js 8081`), los clientes conectados a diferentes instancias no pueden comunicarse entre sí porque cada servidor solo retransmite localmente. Necesitamos un mecanismo de mensajería para integrarlos.

#### Uso de Redis como un broker de mensajes simple

Utilizando las capacidades Pub/Sub de Redis con `ioredis`:

```
Cliente 1 ---> [ Servidor 8080 ] ---> (Publica en 'chat_messages') ---> [ Redis ]
                                                                            |
Cliente 2 <--- [ Servidor 8081 ] <--- (Recibe de 'chat_messages') <--------+
```

*Figura 13.9: Uso de Redis como broker de mensajes para la aplicación de chat.*

Actualizamos `index.js`:

```javascript
// index.js (Redis Pub/Sub)
import { createServer } from 'node:http'
import { WebSocketServer } from 'ws' // v8.18.2
import staticHandler from 'serve-handler' // v6.1.6
import Redis from 'ioredis' // v5.6.1

const redisPub = new Redis() // 1
const redisSub = new Redis()

const server = createServer((req, res) => {
  return staticHandler(req, res, { public: 'web' })
})

const wss = new WebSocketServer({ server })

wss.on('connection', client => {
  console.log('Client connected')
  client.on('message', msg => {
    console.log(`Sending message to Redis: ${msg}`)
    redisPub.publish('chat_messages', msg) // 2
  })
})

redisSub.subscribe('chat_messages') // 3
redisSub.on('message', (channel, msg) => {
  if (channel === 'chat_messages') {
    console.log(`Received message from Redis: ${msg}`)
    for (const client of wss.clients) {
      if (client.readyState === WebSocket.OPEN) {
        client.send(Buffer.from(msg))
      }
    }
  }
})

server.listen(process.argv[2] || 8080)
```

#### Publish/Subscribe peer-to-peer con ZeroMQ

##### Introducción a ZeroMQ

ZeroMQ proporciona primitivas de red de bajo nivel y sockets especiales como `Publisher` (PUB) y `Subscriber` (SUB) sin necesidad de un broker central.

```
[ Servidor A (PUB: 5000) ] <=======(SUB)=======> [ Servidor B (PUB: 5001) ]
```

*Figura 13.10: Arquitectura de mensajería del servidor de chat usando sockets PUB/SUB de ZeroMQ.*

##### Uso de sockets PUB/SUB de ZeroMQ

```javascript
// index.js (ZeroMQ Pub/Sub)
import { createServer } from 'node:http'
import { parseArgs } from 'node:util'
import { WebSocketServer } from 'ws' // v8.18.2
import staticHandler from 'serve-handler' // v6.1.6
import zmq from 'zeromq' // v6.3.0

const { values: args } = parseArgs({ // 1
  options: {
    http: { type: 'string' },
    pub: { type: 'string' },
    sub: { type: 'string', multiple: true },
  },
  args: process.argv.slice(2),
})

if (!(args.http && args.pub && args.sub)) {
  console.error(
    'Usage: node index.js --http <port> --pub <port> ' +
      '--sub <port1> [--sub <port2> ...]'
  )
  process.exit(1)
}

// serve static files
const server = createServer((req, res) => {
  return staticHandler(req, res, { public: 'web' })
})

// Initialize ZeroMQ sockets
const pubSocket = new zmq.Publisher() // 2
await pubSocket.bind(`tcp://127.0.0.1:${args.pub}`)

const subSocket = new zmq.Subscriber() // 3
for (const port of args.sub) {
  console.log(`Subscribing to port ${port}`)
  await subSocket.connect(`tcp://127.0.0.1:${port}`)
}
subSocket.subscribe('chat_messages')

// Receive messages from other servers
async function receiveMessages() { // 4
  for await (const [_topic, msg] of subSocket) {
    console.log(`Received message from another server: ${msg.toString()}`)
    broadcast(Buffer.from(msg))
  }
}
receiveMessages()

const wss = new WebSocketServer({ server })
wss.on('connection', client => {
  console.log('Client connected')
  client.on('message', msg => {
    console.log(`Message: ${msg}`)
    broadcast(msg)
    pubSocket.send(['chat_messages', msg]) // 5
  })
})

function broadcast(msg) {
  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg)
    }
  }
}

server.listen(args.http)
```

Iniciamos tres instancias interconectadas:

```bash
node index.js --http 8080 --pub 5000 --sub 5001 --sub 5002
node index.js --http 8081 --pub 5001 --sub 5000 --sub 5002
node index.js --http 8082 --pub 5002 --sub 5000 --sub 5001
```

#### Entrega confiable de mensajes con colas

Semánticas de entrega:
- **At most once (A lo sumo una vez):** Fire-and-forget; los mensajes pueden perderse ante fallos.
- **At least once (Al menos una vez):** Garantiza la entrega mediante persistencia y reintentos, pero pueden darse duplicados.
- **Exactly once (Exactamente una vez):** Entrega única garantizada mediante confirmaciones estrictas.

##### Introducción a AMQP

AMQP define:
- **Queue (Cola):** Almacena mensajes consumidos por clientes. Puede ser durable, exclusiva o auto-eliminable.
- **Exchange (Intercambiador):** Recibe mensajes y los enruta (Direct, Topic, Fan-out).
- **Binding (Enlace):** Vincula exchanges con colas según claves de enrutamiento.

##### Suscriptores duraderos con AMQP y RabbitMQ

Diseñamos un microservicio de historial (`historySvc.js`) conectado a RabbitMQ junto con las instancias del servidor de chat:

```
[ Servidor Chat 1 ] ---                        +-> [ Exchange 'chat' (fanout) ] ---> [ Cola 'chat_history' ] ---> [ Servicio Historial (LevelDB) ]
[ Servidor Chat 2 ] ---/                              ---> [ Cola 'chat_srv_8080' ]  ---> [ Servidor Chat 1 ]
                                                      ---> [ Cola 'chat_srv_8081' ]  ---> [ Servidor Chat 2 ]
```

*Figura 13.13: Arquitectura de la aplicación de chat con AMQP y servicio de historial.*

###### Implementación del servicio de historial (`historySvc.js`):

```javascript
// historySvc.js
import { createServer } from 'node:http'
import { Level } from 'level' // v10.0.0
import { monotonicFactory } from 'ulid' // v3.0.1
import amqp from 'amqplib' // v0.10.8

const ulid = monotonicFactory() // 1
const db = new Level('msgHistory', { valueEncoding: 'json' })

const connection = await amqp.connect('amqp://localhost') // 2
const channel = await connection.createChannel()
await channel.assertExchange('chat', 'fanout') // 3
const { queue } = await channel.assertQueue('chat_history') // 4
await channel.bindQueue(queue, 'chat') // 5

channel.consume(queue, async msg => { // 6
  const data = JSON.parse(msg.content.toString())
  console.log(`Saving message: ${msg.content.toString()}`)
  await db.put(ulid(), data)
  channel.ack(msg)
})

createServer(async (req, res) => { // 7
  const url = new URL(req.url, 'http://localhost')
  const lt = url.searchParams.get('lt')
  res.writeHead(200, { 'Content-Type': 'application/json' })
  const messages = []
  for await (const [key, value] of db.iterator({
    reverse: true,
    limit: 10,
    lt,
  })) {
    messages.unshift({ id: key, ...value })
  }
  res.end(JSON.stringify(messages, null, 2))
}).listen(8090)
```

###### Integración del servidor de chat con AMQP (`index.js`):

```javascript
// index.js (AMQP)
import { createServer } from 'node:http'
import staticHandler from 'serve-handler' // v6.1.6
import { WebSocketServer } from 'ws' // v8.18.2
import amqp from 'amqplib' // v0.10.8

const httpPort = process.argv[2] || 8080

// register the server with RabbitMQ and create a queue
const connection = await amqp.connect('amqp://localhost')
const channel = await connection.createChannel()
await channel.assertExchange('chat', 'fanout')
const { queue } = await channel.assertQueue( // 1
  `chat_srv_${httpPort}`,
  { exclusive: true }
)
await channel.bindQueue(queue, 'chat')

channel.consume( // 2
  queue,
  msg => {
    msg = msg.content.toString()
    console.log(`From queue: ${msg}`)
    broadcast(Buffer.from(msg))
  },
  { noAck: true }
)

// serve static files
const server = createServer((req, res) => {
  return staticHandler(req, res, { public: 'web' })
})

const wss = new WebSocketServer({ server })

wss.on('connection', async client => {
  console.log('Client connected')
  client.on('message', msg => {
    console.log(`Sending message: ${msg}`)
    channel.publish( // 3
      'chat',
      '',
      Buffer.from(
        JSON.stringify({
          text: msg.toString(),
          timestamp: Date.now(),
        })
      )
    )
  })

  // load previous messages from the history service
  const res = await fetch('http://localhost:8090') // 4
  const messages = await res.json()
  for (const message of messages) {
    client.send(Buffer.from(JSON.stringify(message)))
  }
})

function broadcast(msg) {
  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg)
    }
  }
}

server.listen(httpPort)
```

#### Mensajería confiable con streams

##### Características de una plataforma de streaming y streams de Redis

Con Redis streams podemos utilizar una estructura de registro duradera integrada directamente en Redis:

```javascript
// index.js (Redis Streams)
import { createServer } from 'node:http'
import staticHandler from 'serve-handler' // v6.1.6
import { WebSocketServer } from 'ws' // v8.18.2
import Redis from 'ioredis' // v5.6.1

const redisClient = new Redis()
const redisClientXread = new Redis()

// serve static files
const server = createServer((req, res) => {
  return staticHandler(req, res, { public: 'web' })
})

const wss = new WebSocketServer({ server })

wss.on('connection', async client => {
  console.log('Client connected')
  client.on('message', msg => {
    console.log(`Sending message: ${msg}`)
    redisClient.xadd( // 1
      'chat_stream',
      '*',
      'message',
      JSON.stringify({
        text: msg.toString(),
        timestamp: Date.now(),
      })
    )
  })

  // load previous messages from the history service
  const logs = await redisClient.xrange('chat_stream', '-', '+') // 2
  for (const [, [, message]] of logs) {
    client.send(Buffer.from(message))
  }
})

function broadcast(msg) {
  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg)
    }
  }
}

let lastRecordId = '$'
async function processStreamMessages() { // 3
  while (true) {
    const [[, records]] = await redisClientXread.xread(
      'BLOCK',
      '0',
      'STREAMS',
      'chat_stream',
      lastRecordId
    )
    for (const [recordId, [, message]] of records) {
      console.log(`Message from stream: ${message}`)
      broadcast(Buffer.from(message))
      lastRecordId = recordId
    }
  }
}
processStreamMessages()

server.listen(process.argv[2] || 8080)
```

---

### Sección 3: Patrones de distribución de tareas

Los patrones de distribución de tareas permiten distribuir el trabajo entre múltiples máquinas y procesos (patrón **Competing Consumers**, ventilador o distribución en abanico).

```
                      /---> [ Worker 1 ]
[ Productor de Tareas ] ---> [ Worker 2 ]
                      \---> [ Worker 3 ]
```

*Figura 13.15: Distribución de tareas a un conjunto de consumidores.*

Combinando la distribución de tareas con la agregación de resultados, obtenemos un **pipeline paralelo**:

```
[ Ventilador / Productor ] ===(Fan-out)===> [ Worker 1 ]                                            [ Worker 2 ] --+===(Fan-in)===> [ Colector / Sink ]
                                           [ Worker 3 ] /
```

*Figura 13.16: Un pipeline de mensajería.*

#### El patrón fan-out/fan-in de ZeroMQ

Con sockets **PUSH** y **PULL** de ZeroMQ construimos un descifrador distribuido de checksums SHA1 por fuerza bruta.

```
[ Productor (PUSH :5016) ] ===(Tareas)===> [ Worker 1 (PULL/PUSH) ]                                           [ Worker 2 (PULL/PUSH) ] --+===> [ Sink / Colector (PULL :5017) ]
```

*Figura 13.17: Arquitectura de un pipeline típico con ZeroMQ.*

##### Implementación del generador de tareas (`generateTasks.js`):

```javascript
// generateTasks.js
export function* generateTasks(searchHash, alphabet, maxWordLength, batchSize) {
  const alphabetLength = BigInt(alphabet.length)
  const maxWordLengthBigInt = BigInt(maxWordLength)

  let nVariations = 0n
  for (let n = 1n; n <= maxWordLengthBigInt; n++) {
    nVariations += alphabetLength ** n
  }

  console.log(
    `Finding the hashsum source string over ${nVariations} ` +
      `possible variations`
  )

  let batchStart = 1n
  while (batchStart <= nVariations) {
    const expectedBatchSize = batchStart + BigInt(batchSize) - 1n
    const batchEnd =
      expectedBatchSize > nVariations ? nVariations : expectedBatchSize
    yield JSON.stringify({
      searchHash,
      alphabet: alphabet,
      // convert BigInt to string for JSON serialization
      batchStart: batchStart.toString(),
      batchEnd: batchEnd.toString(),
    })
    batchStart = batchEnd + 1n
  }
}
```

##### Implementación del productor (`producer.js`):

```javascript
// producer.js
import zmq from 'zeromq' // v6.3.0
import { generateTasks } from './generateTasks.js'

const ALPHABET = 'abcdefghijklmnopqrstuvwxyz'
const BATCH_SIZE = 10000

const [, , maxLength, searchHash] = process.argv

const ventilator = new zmq.Push() // 1
await ventilator.bind('tcp://*:5016')

const generatorObj = generateTasks(searchHash, ALPHABET, maxLength, BATCH_SIZE)
for (const task of generatorObj) {
  await ventilator.send(task) // 2
}
```

##### Implementación del procesador de tareas (`processTask.js`):

```javascript
// processTask.js
import isv from 'indexed-string-variation' // v2.0.1
import { createHash } from 'node:crypto'

export function processTask(task) {
  const strings = isv({
    alphabet: task.alphabet,
    from: BigInt(task.batchStart),
    to: BigInt(task.batchEnd),
  })

  let first
  let last
  for (const string of strings) {
    if (!first) {
      first = string
    }
    const digest = createHash('sha1').update(string).digest('hex')
    if (digest === task.searchHash) {
      console.log(`>> Found: ${string} => ${digest}`)
      return string
    }
    last = string
  }
  console.log(
    `Processed ${first}..${last} (${task.batchStart}..${task.batchEnd})`
  )
}
```

##### Implementación del worker (`worker.js`):

```javascript
// worker.js
import zmq from 'zeromq' // v6.3.0
import { processTask } from './processTask.js'

const fromVentilator = new zmq.Pull()
const toSink = new zmq.Push()

fromVentilator.connect('tcp://localhost:5016')
toSink.connect('tcp://localhost:5017')

for await (const rawMessage of fromVentilator) {
  const found = processTask(JSON.parse(rawMessage.toString()))
  if (found) {
    console.log(`Found! => ${found}`)
    await toSink.send(`Found: ${found}`)
    break
  }
}
```

##### Implementación del colector de resultados (`collector.js`):

```javascript
// collector.js
import zmq from 'zeromq' // v6.3.0

const sink = new zmq.Pull()
await sink.bind('tcp://*:5017')

for await (const rawMessage of sink) {
  console.log('Message from worker: ', rawMessage.toString())
}
```

#### Pipelines y consumidores competidores en AMQP

En AMQP, enviamos directamente a una cola `tasks_queue` punto a punto sin intermediación de exchange, logrando que múltiples workers conectados a la misma cola actúen como consumidores competidores.

```
[ Productor ] ---> [ Cola 'tasks_queue' ] ---> [ Worker 1 ]                                           ---> [ Worker 2 ] --+-> [ Cola 'results_queue' ] ---> [ Colector ]
```

*Figura 13.19: Arquitectura de distribución de tareas usando un broker MQ.*

##### Productor AMQP (`producer.js`):

```javascript
// producer.js (AMQP)
import amqp from 'amqplib' // v0.10.8
import { generateTasks } from './generateTasks.js'

const ALPHABET = 'abcdefghijklmnopqrstuvwxyz'
const BATCH_SIZE = 10000

const [, , maxLength, searchHash] = process.argv

const connection = await amqp.connect('amqp://localhost')
const channel = await connection.createConfirmChannel() // 1
await channel.assertQueue('tasks_queue')

const generatorObj = generateTasks(searchHash, ALPHABET, maxLength, BATCH_SIZE)
for (const task of generatorObj) {
  console.log(`Sending task: ${task}`)
  await channel.sendToQueue('tasks_queue', Buffer.from(task)) // 2
}

await channel.waitForConfirms()
channel.close()
connection.close()
```

##### Worker AMQP (`worker.js`):

```javascript
// worker.js (AMQP)
import amqp from 'amqplib' // v0.10.8
import { processTask } from './processTask.js'

const connection = await amqp.connect('amqp://localhost')
const channel = await connection.createChannel()
const { queue } = await channel.assertQueue('tasks_queue')

channel.prefetch(1) // Ensure only one message is processed at a time

channel.consume(queue, async rawMessage => {
  const found = processTask(JSON.parse(rawMessage.content.toString()))
  await channel.ack(rawMessage)
  if (found) {
    console.log(`Found! => ${found}`)
    await channel.sendToQueue('results_queue', Buffer.from(`Found: ${found}`))
    // shuts down the worker
    await channel.close()
    await connection.close()
  }
})
```

##### Colector AMQP (`collector.js`):

```javascript
// collector.js (AMQP)
import amqp from 'amqplib' // v0.10.8

const connection = await amqp.connect('amqp://localhost')
const channel = await connection.createChannel()
const { queue } = await channel.assertQueue('results_queue')

channel.consume(queue, async msg => {
  console.log(`Message from worker: ${msg.content.toString()}`)
  await channel.ack(msg)
})
```

#### Distribución de tareas con Redis streams

Utilizando grupos de consumidores de Redis (`consumer groups`):

```
                                      /---> [ Consumidor 1 (Lee registro B) ]
[ Stream 'tasks_stream' ] ===(xreadgroup)===> [ Consumidor 2 (Lee registro C) ]
```

*Figura 13.20: Un grupo de consumidores de Redis stream.*

##### Productor con Redis streams (`producer.js`):

```javascript
// producer.js (Redis Streams)
import Redis from 'ioredis' // v5.6.1
import { generateTasks } from './generateTasks.js'

const ALPHABET = 'abcdefghijklmnopqrstuvwxyz'
const BATCH_SIZE = 10000

const [, , maxLength, searchHash] = process.argv

const redisClient = new Redis()

const generatorObj = generateTasks(searchHash, ALPHABET, maxLength, BATCH_SIZE)
for (const task of generatorObj) {
  console.log(`Sending task: ${task}`)
  await redisClient.xadd('tasks_stream', '*', 'task', task)
}

redisClient.disconnect()
```

##### Worker con Redis streams (`worker.js`):

```javascript
// worker.js (Redis Streams)
import Redis from 'ioredis' // v5.6.1
import { processTask } from './processTask.js'

const redisClient = new Redis()
const [, , consumerName] = process.argv

await redisClient // 1
  .xgroup('CREATE', 'tasks_stream', 'workers_group', '$', 'MKSTREAM')
  .catch(() => console.log('Consumer group already exists'))

const [[, records]] = await redisClient.xreadgroup( // 2
  'GROUP',
  'workers_group',
  consumerName,
  'STREAMS',
  'tasks_stream',
  '0'
)
for (const [recordId, [, rawTask]] of records) {
  await processAndAck(recordId, rawTask)
}

while (true) {
  const [[, records]] = await redisClient.xreadgroup( // 3
    'GROUP',
    'workers_group',
    consumerName,
    'BLOCK',
    '0',
    'COUNT',
    '1',
    'STREAMS',
    'tasks_stream',
    '>'
  )
  for (const [recordId, [, rawTask]] of records) {
    await processAndAck(recordId, rawTask)
  }
}

async function processAndAck(recordId, rawTask) { // 4
  const found = processTask(JSON.parse(rawTask))
  if (found) {
    console.log(`Found! => ${found}`)
    await redisClient.xadd('results_stream', '*', 'result', `Found: ${found}`)
  }
  await redisClient.xack('tasks_stream', 'workers_group', recordId)
}
```

##### Colector con Redis streams (`collector.js`):

```javascript
// collector.js (Redis Streams)
import Redis from 'ioredis' // v5.6.1

const redisClient = new Redis()

let lastRecordId = '$'
while (true) {
  const data = await redisClient.xread(
    'BLOCK',
    '0',
    'STREAMS',
    'results_stream',
    lastRecordId
  )
  for (const [, logs] of data) {
    for (const [recordId, [, message]] of logs) {
      console.log(`Message from worker: ${message}`)
      lastRecordId = recordId
    }
  }
}
```

---

### Sección 4: Patrones Request/Reply

#### Identificador de correlación (Correlation Identifier)

El patrón **Correlation Identifier** permite asociar una respuesta con su solicitud original a través de un identificador único cuando los mensajes viajan de forma asíncrona y desordenada por canales unidireccionales.

```
Solicitante ----------------(Req id: 1)---------------> Respondedor
Solicitante ----------------(Req id: 2)---------------> Respondedor
Solicitante <-------------(Resp inReplyTo: 2)---------- Respondedor
Solicitante <-------------(Resp inReplyTo: 1)---------- Respondedor
```

*Figura 13.21: Intercambio de mensajes Request/Reply usando identificadores de correlación.*

##### Abstracción del canal de solicitud (`createRequestChannel.js`):

```javascript
// createRequestChannel.js
import { nanoid } from 'nanoid' // v5.1.5

export function createRequestChannel(channel) { // 1
  const correlationMap = new Map()

  function sendRequest(data) { // 2
    console.log('Sending request', data)
    return new Promise((resolve, reject) => {
      const correlationId = nanoid()
      const replyTimeout = setTimeout(() => {
        correlationMap.delete(correlationId)
        reject(new Error('Request timeout'))
      }, 10000)
      correlationMap.set(correlationId, replyData => {
        correlationMap.delete(correlationId)
        clearTimeout(replyTimeout)
        resolve(replyData)
      })
      channel.send({
        type: 'request',
        data,
        id: correlationId,
      })
    })
  }

  channel.on('message', message => { // 3
    const replyCb = correlationMap.get(message.inReplyTo)
    if (replyCb) {
      replyCb(message.data)
    }
  })

  return sendRequest
}
```

##### Abstracción del canal de respuesta (`createReplyChannel.js`):

```javascript
// createReplyChannel.js
export function createReplyChannel(channel) {
  return function registerHandler(handler) {
    channel.on('message', async message => {
      if (message.type !== 'request') {
        return
      }

      const replyData = await handler(message.data)
      channel.send({
        type: 'response',
        data: replyData,
        inReplyTo: message.id,
      })
    })
  }
}
```

##### Respondedor (`replier.js`):

```javascript
// replier.js
import { createReplyChannel } from './createReplyChannel.js'

const registerReplyHandler = createReplyChannel(process)

registerReplyHandler(req => {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve({ sum: req.a + req.b })
    }, req.delay)
  })
})

process.send('ready')
```

##### Solicitante (`requestor.js`):

```javascript
// requestor.js
import { fork } from 'node:child_process'
import { join } from 'node:path'
import { once } from 'node:events'
import { createRequestChannel } from './createRequestChannel.js'

const channel = fork(join(import.meta.dirname, 'replier.js')) // 1
const request = createRequestChannel(channel)

try {
  const [message] = await once(channel, 'message') // 2
  console.log(`Child process initialized: ${message}`)

  const p1 = request({ a: 1, b: 2, delay: 500 }).then(res => { // 3
    console.log(`Reply: 1 + 2 = ${res.sum}`)
  })
  const p2 = request({ a: 6, b: 1, delay: 100 }).then(res => { // 4
    console.log(`Reply: 6 + 1 = ${res.sum}`)
  })

  await Promise.all([p1, p2]) // 5
} finally {
  channel.disconnect() // 6
}
```

#### Dirección de retorno (Return address)

Cuando intervienen múltiples canales o colas, el solicitante incluye además una **dirección de retorno** (`replyTo`), que indica al respondedor en qué cola privada debe publicar la respuesta.

```
[ Solicitante 1 ] ---(Req replyTo: q1)---> [ Cola de Solicitudes ] ---> [ Respondedor ]
        ^                                                                     |
        \----------------(Resp)---------- [ Cola Privada q1 ] <---------------+
```

*Figura 13.22: Arquitectura de mensajería Request/Reply usando AMQP.*

##### Abstracción de solicitud AMQP (`amqpRequest.js`):

```javascript
// amqpRequest.js
import { nanoid } from 'nanoid' // v5.1.5
import amqp from 'amqplib' // v0.10.8

export class AmqpRequest {
  constructor () {
    this.correlationMap = new Map()
  }

  async initialize() {
    this.connection = await amqp.connect('amqp://localhost')
    this.channel = await this.connection.createChannel()
    const { queue } = await this.channel.assertQueue( // 1
      '',
      { exclusive: true }
    )
    this.replyQueue = queue

    this.channel.consume( // 2
      this.replyQueue,
      msg => {
        const correlationId = msg.properties.correlationId
        const handler = this.correlationMap.get(correlationId)
        if (handler) {
          handler(JSON.parse(msg.content.toString()))
        }
      },
      { noAck: true }
    )
  }

  send(queue, message) {
    return new Promise((resolve, reject) => {
      const id = nanoid() // 1
      const replyTimeout = setTimeout(() => {
        this.correlationMap.delete(id)
        reject(new Error('Request timeout'))
      }, 10000)

      this.correlationMap.set(id, replyData => { // 2
        this.correlationMap.delete(id)
        clearTimeout(replyTimeout)
        resolve(replyData)
      })

      this.channel.sendToQueue( // 3
        queue,
        Buffer.from(JSON.stringify(message)),
        {
          correlationId: id,
          replyTo: this.replyQueue,
        }
      )
    })
  }

  destroy() {
    this.channel.close()
    this.connection.close()
  }
}
```

##### Abstracción de respuesta AMQP (`amqpReply.js`):

```javascript
// amqpReply.js
import amqp from 'amqplib' // v0.10.8

export class AmqpReply {
  constructor(requestsQueueName) {
    this.requestsQueueName = requestsQueueName
  }

  async initialize() {
    const connection = await amqp.connect('amqp://localhost')
    this.channel = await connection.createChannel()
    const { queue } = await this.channel.assertQueue( // 1
      this.requestsQueueName
    )
    this.queue = queue
  }

  handleRequests(handler) { // 2
    this.channel.consume(this.queue, async msg => {
      const content = JSON.parse(msg.content.toString())
      const replyData = await handler(content)
      this.channel.sendToQueue( // 3
        msg.properties.replyTo,
        Buffer.from(JSON.stringify(replyData)),
        { correlationId: msg.properties.correlationId }
      )
      this.channel.ack(msg)
    })
  }
}
```

##### Respondedor AMQP (`replier.js`):

```javascript
// replier.js (AMQP)
import { AmqpReply } from './amqpReply.js'

const reply = new AmqpReply('requests_queue')
await reply.initialize()

reply.handleRequests(req => {
  console.log('Request received', req)
  return { sum: req.a + req.b }
})
```

##### Solicitante AMQP (`requestor.js`):

```javascript
// requestor.js (AMQP)
import { AmqpRequest } from './amqpRequest.js'
import { setTimeout } from 'node:timers/promises'

const request = new AmqpRequest()
await request.initialize()

async function sendRandomRequest() {
  const a = Math.round(Math.random() * 100)
  const b = Math.round(Math.random() * 100)
  const reply = await request.send('requests_queue', { a, b })
  console.log(`${a} + ${b} = ${reply.sum}`)
}

for (let i = 0; i < 20; i++) {
  await sendRandomRequest()
  await setTimeout(1000)
}

request.destroy()
```

---

### Sección 5: Resumen

En este capítulo final, exploramos los patrones de mensajería e integración fundamentales para arquitecturas distribuidas:
- Analizamos las diferencias entre canales unidireccionales y **Request/Reply**, y entre modelos **push** y **pull**.
- Implementamos el patrón **Publish/Subscribe** con brokers (Redis), peer-to-peer (ZeroMQ), colas confiables duraderas (AMQP/RabbitMQ) y streams persistentes (Redis streams).
- Construimos canalizaciones de **distribución de tareas** y pipelines paralelos con consumidores competidores en ZeroMQ, RabbitMQ y Redis streams.
- Dominamos los patrones **Correlation Identifier** y **Return Address** para crear abstracciones asíncronas de Request/Reply sobre canales unidireccionales.

Este capítulo marca el final del libro. Ahora cuentas con un repertorio completo de patrones y técnicas de diseño para afrontar proyectos del mundo real con Node.js, comprendiendo sus fortalezas y construyendo arquitecturas escalables, resilientes y elegantes.

*Sinceramente, Luciano Mammino y Mario Casciaro.*

---

### Sección 6: Ejercicios

- **13.1 Servicio de historial con streams:** En nuestro ejemplo de publish/subscribe con Redis streams, implementa un servicio de historial independiente que almacene los mensajes en una base de datos externa y sirva el historial al conectarse un cliente nuevo.
- **13.2 Chat multicanal:** Actualiza la aplicación de chat para admitir múltiples salas de conversación con persistencia del historial.
- **13.3 Tareas que se detienen:** Añade a los ejemplos del descifrador de hashsums un mecanismo Pub/Sub para detener la computación en todos los workers y el ventilador una vez que se encuentre la coincidencia.
- **13.4 Procesamiento confiable de tareas con ZeroMQ:** Implementa un sistema de colas peer-to-peer y confirmaciones (ACKs) sobre ZeroMQ para garantizar que las tareas no se pierdan si un worker falla.
- **13.5 Agregador de datos:** Crea una abstracción que envíe una solicitud a todos los nodos conectados y devuelva una agregación de todas las respuestas recibidas.
- **13.6 CLI de estado de workers:** Utiliza el agregador de datos para crear una herramienta de línea de comandos que muestre el estado en tiempo real de todos los workers del descifrador de hashsums.
- **13.7 UI de estado de workers:** Implementa una aplicación web para visualizar el estado y progreso de los workers en tiempo real.
- **13.8 Retorno de las colas de pre-inicialización:** Refactoriza el ejemplo de Request/Reply en AMQP utilizando colas de pre-inicialización (como en el Capítulo 11) para evitar la necesidad de esperar manualmente a `initialize()`.
- **13.9 Request/Reply con Redis streams:** Construye una abstracción de Request/Reply sobre Redis streams utilizando grupos de consumidores y direcciones de retorno.
- **13.10 Kafka:** Reimplementa los ejemplos clave de mensajería de este capítulo utilizando Apache Kafka ([nodejsdp.link/kafka](https://nodejsdp.link/kafka)).

