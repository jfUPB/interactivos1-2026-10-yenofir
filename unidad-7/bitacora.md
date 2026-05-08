# Unidad 7

## Bitácora de proceso de aprendizaje
## ACTIVIDAD 1

**¿Qué diferencia hay entre un evento musical y un mensaje de control?**

La diferencia sería que desde el evento musical no podemos controlar nosotros mismos la acción de cambiar, el mensaje de control permite modificar un parametro en especifico en tiempo real. Es decir con el evento músical programo acciones puntuales determinadas desde el Código porque este pasa en un momento especifico, con el mensaje de control puedo accionar un cambio en el estado que estemos en vivo, por ende nos permite tener más control sobre los objetos.


**¿Qué quiere decir que un parámetro del sistema sea persistente?**

Un estado persistente es el que se mantiene sin niguna modificación, por ejemplo, si me llega un rgb en rojo, este mismo seguira llegando porque no tiene ninguna modificación del estado, contrario a un evento que llega y se va.


**¿Qué partes del sistema de la unidad 6 permanecen intactas en este nuevo caso?**
Permanece nuestro adaptador Strudel que sigue recibiendo los mensajes y normalizando para el bridge; el bridgeServer, que es el que retransmite los mensajes recibidos de los adaptadores , el baseAdapter, que sigue manteniendo un contrato con todos los adaptadores de conexión, data y errores; en nuestro sketch se mantiene intactas las funciones que reciben la cola de Eventos de Strudel.


**Si Open Stage Control fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?**

El protocolo sería OSC sobre UDP ya que como servidor, recibe mensajes OSC (comandos de faders, botones, XY pads) desde clientes (tablets, navegadores web) y los reenvía a otras aplicaciones o dispositivos (DAWs, servidores de video, consolas de luces), de acuerdo con la necesidad. También el protocolo HTTP/WebSockets para servir la interfaz gráfica de usuario (GUI) personalizada al navegador del cliente.


**¿Qué parte de ese protocolo te interesa conservar y cuál te gustaría normalizar?**

Conservar es la dirección OSC (/rgb_1) y los argumentos numéricos (R, G, B), porque eso es la semántica del mensaje — te dice qué parámetro cambiar y a qué valor.

Normalizar es el formato de los argumentos, porque la librería osc puede enviártelos de dos formas distintas: como número crudo (0.3) o como objeto con metadatos ({ type: "f", value: 0.3 }). Eso es exactamente lo que hace la función normalizeArg en bridgeOSC.js — aplanarlo siempre a un número simple antes de pasarlo al cliente.


**¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?**

Por lo mencionado, Strudel alimenta una cola de eventos temporizados, es decir, suceden en momentos especificos, OSC, actualiza el estado del control del sistema, es decir esta cambiando con los estados en un parametro especifico, por ende no tiene una apertura o cierre, siempre esta leyendo. 
La diferencia es semántica: los mensajes de Strudel tienen timestamp y van a una cola temporizada. Los de Open Stage Control no tienen tiempo, son actualizaciones de estado inmediatas. Meterlos en la misma cola rompería la sincronización.


**¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?**

En este caso del panel RGB, las variables persistentes, serían las de los sonidos, declararlos como elementeos de color con un RGB, que luego se puedan cambiar en el OSC.


**¿Qué componentes de la arquitectura necesitas conservar obligatoriamente?**

El StrudelAdapter y su conexión al BaseAdapter, el bridgeServer.js como broadcaster, el bridgeClient en el navegador, y toda la cadena FSMTask → processQueue → drawRunning. 


**¿Qué nuevas estructuras de estado necesitas introducir para soportar control paramétrico?**

En el momento las variables persistentes son globales, podríamos hacer un objeto de estado de control. Así podríamos actualizarlo cada vez que llegue un mensaje de Open Stade Control. 
Y tener 2 entradas de mensajes que permitan leer Strudel y OSC a la vez y los pueda diferenciarlos para enrutarlos a las funciones diferentes en el sketch. 



## Bitácora de aplicación 

## ACTIVIDAD 2


### Configuración Open Stage Control

**port: 8086:** http://127.0.0.1:8086 y ves los controles RGB, fader y switch. Este es el puerto de la interfaz web.

**send: ["127.0.0.1:9000"]** dónde envía los mensajes OSC cuando movemos un control.



### Qué widgets decidiste usar y por qué

**RGB**Que controla el color, a través de 3 datos como la referencia que veníamos manejando.

**FADER:** Controla el tamaño de 0.5 a 2.0, que tan grande o pequeño es nuestro bombo.

**SWITCH**encendido o apagado
La elección de estos es para usar diferentes widgets.



### Qué estructura final de mensaje decidiste usar para los controles

`{ type: "osc", payload: { address: "/rgb_1", args: [255, 120, 30] } }`

type:"osc" permite que bridgeClient distinga este flujo del de Strudel sin mirar el contenido. address identifica qué control fue movido. args tiene los valores. 

type le dice al sistema qué clase de mensaje es. payload contiene lo que realmente importa. Cuando _updateControl recibe el mensaje, solo le interesa el payload



### Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning

El bridge Client, es el que enruta los mensajes que vienen del Server, lo hace a través del mgs.type, que le da un tipo de mensaje a los datos de Strudel y a los datos de OSC:

```
// ── Eventos musicales de Strudel ──────────────────────────────────────
      // { type:"strudel", timestamp:ms, payload:{ eventType, soundType, s, delta, ... } }
      if (msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }

      // ── Mensajes de control de Open Stage Control ─────────────────────────
      // { type:"osc", payload:{ address:"/rgb_1", args:[255,120,30] }, t:ms }
      // Son actualizaciones de estado persistente, no eventos temporizados.
      if (msg.type === "osc") {
        this._onData?.(msg);
        return;
      }
    };
```

Luego el FSMTask recibe estos datos y allí definimos cuales son los eventos y los estados persistentes



### Cómo integraste ambas fuentes de datos en el mismo frontend

La clave es que dentro de PainterTask los dos flujos nunca se mezclan:

Strudel → eventQueue → activación por timestamp → animación efímera
OSC → controlState → valor que persiste hasta el próximo mensaje → drawRunning lo lee en cada frame

El canvas ve los dos al mismo tiempo porque drawRunning consume ambos: procesa la cola para activar animaciones, y lee controlState para saber con qué color y tamaño dibujarlas.

**STRUDEL:**

````
 processQueue() {
    const now = Date.now();

    while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
      const ev = this.eventQueue.shift();
      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000,
        type:      ev.sound,
        soundType: ev.soundType,
        x: random(width  * 0.2, width  * 0.8),
        y: random(height * 0.2, height * 0.8),
        // Color base se guarda aquí, pero bd lo sobreescribe en draw con controlState.
        // Así el color del bombo es siempre el último recibido por OSC,
        // no el que había cuando nació la animación.
        color: getColorForSound(ev.sound),
      });
    }
  }
````

**OSC:**

````
_updateControl({ address, args }) {
    if (address === "/rgb_1") {
      // RGB picker → color del bombo [r, g, b] en rango 0-255
      this.controlState.bdColor = [
        Number(args[0] ?? 255),
        Number(args[1] ?? 0),
        Number(args[2] ?? 80),
      ];
    }

    if (address === "/size_1") {
      // Slider → multiplicador de tamaño, acotado para evitar valores extremos
      this.controlState.sizeMultiplier = constrain(Number(args[0] ?? 1.0), 0.5, 2.0);
    }

    if (address === "/trail_1") {
      // Toggle → 1 = estela activa (background con alpha), 0 = flash (background limpio)
      this.controlState.trailEnabled = Number(args[0]) === 1;
    }
  }
````


### Qué pruebas hiciste para verificar que el control paramétrico funciona sin romper la sincronización de Strudel

Primero probamos con la velocidad del ciclo de Strudel, para enteder si la sincronización se mantenía.
Validar que datos estan llegando a la terminal, incluso si son los mismos del HUB.



### Qué problemas encontraste y cómo los solucionaste

La aplicación estaba corriendo con 2 servidores, en 2 terminales, lo ideal era hacer correr la aplicación en una sola terminal. Por ende hicimos una modificación en el bridge Server, este realizo unos arreglos que envíaban un solo device corriendo Strudel + OSC. 

````
  // ── Modo principal: Strudel + OSC en una sola terminal ──────────────────
  // Ambos adaptadores se amarran al mismo servidor WebSocket.
  // El cliente los distingue por msg.type ("strudel" vs "osc").
  // Solo necesitas un puerto WebSocket y una terminal.
  if (DEVICE === "strudel+osc") {
    log.info("Creando adaptador Strudel (ws://127.0.0.1:8080)");
    log.info(`Creando adaptador OSC     (UDP :${OSC_PORT})`);
    const adaptadorStrudel = new StrudelAdapter({ verbose: VERBOSE });
    const adaptadorOSC     = new OSCAdapter({ port: OSC_PORT, verbose: VERBOSE });
    return [ adaptadorStrudel, adaptadorOSC ];
  }
````
Entonces la función crearAdaptadores() devuelve un array
Antes devolvía un solo adaptador. Ahora devuelve una lista. Si el modo es "strudel", la lista tiene uno. Si es "strudel+osc", la lista tiene dos.
Ambos adaptadores se amarran al mismo servidor WebSocket en el mismo puerto. El cliente ya sabe distinguirlos por msg.type.


<img width="1737" height="1032" alt="Captura de pantalla 2026-04-28 230923" src="https://github.com/user-attachments/assets/0ddc46ba-6c77-4164-ae91-add843ea22a4" />


También realizamos ajustes en nuestro sketch, ya que antes había dos variables globales (bridge y bridgeOSC) apuntando a dos puertos distintos. Ahora hay una sola (bridge) apuntando al único puerto donde corre el servidor.


<img width="1916" height="1000" alt="Captura de pantalla 2026-04-29 005545" src="https://github.com/user-attachments/assets/23daf899-81cf-4349-af97-7b1cbe723afa" />

````
bridge.onData((msg) => {

  if (msg.type === "strudel") {
    painter.postEvent({
      type: EVENTS.STRUDEL_EVENT,
      payload: { ...msg.payload, timestamp: msg.timestamp },
    });
    return;
  }

  if (msg.type === "osc") {
    painter.postEvent({
      type: EVENTS.OSC_CONTROL,
      payload: msg.payload,
    });
    return;
  }

  if (msg.type === "microbit") {
    painter.postEvent({
      type: EVENTS.MICROBIT_DATA,
      payload: { x: msg.x, y: msg.y, btnA: msg.btnA, btnB: msg.btnB },
    });
    return;
  }

});
````

<img width="1084" height="284" alt="Captura de pantalla 2026-05-06 153821" src="https://github.com/user-attachments/assets/208bebf7-a14f-4c32-bc19-c6d68ac99273" />


## CÓDIGOS

### BridgeClient

````
// bridgeClient.js
// Cliente WebSocket que conecta el frontend con bridgeServer.js.
// Recibe mensajes normalizados del bridge y los enruta a los callbacks registrados.
//
// Tipos de mensaje soportados:
//   {type:"status",  state:"...", detail:"..."}   → onStatus / onConnect / onDisconnect
//   {type:"microbit", x, y, btnA, btnB, t}        → onData
//   {type:"strudel",  timestamp, payload, t}       → onData
//   {type:"osc",      payload:{address, args}, t}  → onData   ← NUEVO

class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData       = null;
    this._onConnect    = null;
    this._onDisconnect = null;
    this._onStatus     = null;
  }

  get isOpen() { return this._isOpen; }

  onData(callback)       { this._onData       = callback; }
  onConnect(callback)    { this._onConnect    = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback)     { this._onStatus     = callback; }

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) this.close();

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      // ── Mensajes de estado del bridge ──────────────────────────────────────
      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          this._onConnect?.();
        }

        if (msg.state === "disconnected" || msg.state === "error" || msg.state === "ready") {
          this._isOpen = false;
          this._onDisconnect?.();
          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }
        }
        return;
      }

      // ── Datos del micro:bit ───────────────────────────────────────────────
      if (msg.type === "microbit") {
        this._onData?.(msg);
        return;
      }

      // ── Eventos musicales de Strudel ──────────────────────────────────────
      // { type:"strudel", timestamp:ms, payload:{ eventType, soundType, s, delta, ... } }
      // No se interpreta aquí — solo se enruta al consumidor (FSM).
      if (msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }

      // ── Mensajes de control de Open Stage Control ─────────────────────────
      // { type:"osc", payload:{ address:"/rgb_1", args:[255,120,30] }, t:ms }
      // Son actualizaciones de estado persistente, no eventos temporizados.
      // No se interpretan aquí — se enrutan a la FSM para que actualice controlState.
      if (msg.type === "osc") {
        this._onData?.(msg);
        return;
      }
    };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}
````

### BridgeServer

````
//   Uso:
//     node bridgeServer.js --device sim         --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit    --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device strudel     --wsPort 8081
//     node bridgeServer.js --device osc         --wsPort 8081 --oscPort 9000
//     node bridgeServer.js --device strudel+osc --wsPort 8081 --oscPort 9000  ← UNA SOLA TERMINAL
//
//   WS contract (bridge → client):
//     {type:"status",  state:"ready|connected|disconnected|error", detail:"..."}
//     {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//     {type:"strudel",  timestamp:ms, payload:{eventType, soundType, s, delta, ...}}
//     {type:"osc",      payload:{address:"/rgb_1", args:[255,120,30]}, t:ms}
//
//   WS contract (client → bridge):
//     {cmd:"connect"} | {cmd:"disconnect"}
//     {cmd:"setSimHz", hz:30}
//     {cmd:"setLed", x:2, y:3, value:9}

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter            = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter  = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter     = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter        = require("./adapters/StrudelAdapter");
const OSCAdapter            = require("./adapters/OSCAdapter");

const log = {
  info:  (...args) => console.log(`[${new Date().toISOString()}] [INFO]`,  ...args),
  warn:  (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`,  ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args),
};

function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) { return process.argv.includes(`--${name}`); }
function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try { return JSON.parse(s); }
  catch (e) { log.warn("Failed to parse JSON:", s, e); return null; }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function statusBroadcast(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE      = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT     = parseInt(getArg("wsPort",   "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD        = parseInt(getArg("baud",     "115200"), 10);
const SIM_HZ      = parseInt(getArg("hz",       "30"), 10);
const OSC_PORT    = parseInt(getArg("oscPort",  "9000"), 10);
const VERBOSE     = hasFlag("verbose");

// ── Función central: amarra un adaptador al servidor WebSocket ─────────────
//
// Antes este bloque se repetía para cada adaptador:
//   adapter.onConnected = ...
//   adapter.onData = ...
//   adapter.onError = ...
//
// Ahora existe una sola función que recibe cualquier adaptador y lo conecta
// al servidor. El "label" es solo para identificarlo en los logs.
//
function amarrarAdaptador(adaptador, label, wss) {
  adaptador.onConnected = (detail) => {
    log.info(`[${label}] Conectado: ${detail}`);
    statusBroadcast(wss, "connected", detail);
  };

  adaptador.onDisconnected = (detail) => {
    log.warn(`[${label}] Desconectado: ${detail}`);
    statusBroadcast(wss, "disconnected", detail);
  };

  adaptador.onError = (detail) => {
    log.error(`[${label}] Error: ${detail}`);
    statusBroadcast(wss, "error", detail);
  };

  adaptador.onData = (datos) => {
    if (typeof adaptador.formatBroadcast === "function") {
      const mensaje = adaptador.formatBroadcast(datos);
      log.info(`[${label}] ${JSON.stringify(mensaje.payload ?? datos).slice(0, 120)}`);
      broadcast(wss, mensaje);
    } else {
      log.info(`[${label}] x:${datos.x} y:${datos.y} btnA:${datos.btnA} btnB:${datos.btnB}`);
      broadcast(wss, {
        type: "microbit",
        x: datos.x, y: datos.y,
        btnA: !!datos.btnA, btnB: !!datos.btnB,
        t: nowMs(),
      });
    }
  };
}

async function encontrarPuertoMicrobit() {
  const puertos = await SerialPort.list();
  const microbit = puertos.find(p => p.vendorId && parseInt(p.vendorId, 16) === 0x0D28);
  return microbit?.path ?? null;
}

// Devuelve un array de adaptadores según el modo elegido.
// Un solo adaptador → array de 1. Dos fuentes → array de 2.
async function crearAdaptadores() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await encontrarPuertoMicrobit();
    if (!path) { log.error("micro:bit no encontrado. Usa --serialPort."); process.exit(1); }
    return [ new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE }) ];
  }

  if (DEVICE === "microbitv2") {
    const path = SERIAL_PATH ?? await encontrarPuertoMicrobit();
    if (!path) { log.error("micro:bit no encontrado."); process.exit(1); }
    return [ new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE }) ];
  }

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await encontrarPuertoMicrobit();
    if (!path) { log.error("micro:bit no encontrado."); process.exit(1); }
    return [ new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE }) ];
  }

  if (DEVICE === "strudel") {
    log.info("Creando adaptador Strudel (ws://127.0.0.1:8080)");
    return [ new StrudelAdapter({ verbose: VERBOSE }) ];
  }

  if (DEVICE === "osc") {
    log.info(`Creando adaptador OSC (UDP :${OSC_PORT})`);
    return [ new OSCAdapter({ port: OSC_PORT, verbose: VERBOSE }) ];
  }

  // ── Modo principal: Strudel + OSC en una sola terminal ──────────────────
  // Ambos adaptadores se amarran al mismo servidor WebSocket.
  // El cliente los distingue por msg.type ("strudel" vs "osc").
  // Solo necesitas un puerto WebSocket y una terminal.
  if (DEVICE === "strudel+osc") {
    log.info("Creando adaptador Strudel (ws://127.0.0.1:8080)");
    log.info(`Creando adaptador OSC     (UDP :${OSC_PORT})`);
    const adaptadorStrudel = new StrudelAdapter({ verbose: VERBOSE });
    const adaptadorOSC     = new OSCAdapter({ port: OSC_PORT, verbose: VERBOSE });
    return [ adaptadorStrudel, adaptadorOSC ];
  }

  return [ new SimAdapter({ hz: SIM_HZ }) ];
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`Servidor WS escuchando en ws://127.0.0.1:${WS_PORT}  device=${DEVICE}`);

  const adaptadores = await crearAdaptadores();

  // Amarramos cada adaptador al mismo servidor con la misma función
  for (const adaptador of adaptadores) {
    amarrarAdaptador(adaptador, adaptador.constructor.name, wss);
  }

  statusBroadcast(wss, "ready", `bridge listo (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[RED] Cliente conectado desde ${req.socket.remoteAddress}. Total: ${wss.clients.size}`);

    const hayAlgunoConectado = adaptadores.some(a => a.connected);
    const estado  = hayAlgunoConectado ? "connected" : "ready";
    const detalle = hayAlgunoConectado
      ? adaptadores.filter(a => a.connected).map(a => a.getConnectionDetail()).join(" | ")
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state: estado, detail: detalle, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info("[RED] Cliente solicitó conexión");
        for (const adaptador of adaptadores) {
          if (adaptador.connected) {
            ws.send(JSON.stringify({
              type: "status", state: "connected",
              detail: adaptador.getConnectionDetail(), t: nowMs()
            }));
            continue;
          }
          try { await adaptador.connect(); }
          catch (e) {
            const detalle = `connect failed: ${e.message || e}`;
            log.error("[ADAPTADOR] " + detalle);
            statusBroadcast(wss, "error", detalle);
          }
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info("[RED] Cliente solicitó desconexión");
        if (wss.clients.size > 1) {
          log.info(`[POLÍTICA] Adaptadores activos compartidos con ${wss.clients.size - 1} cliente(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        for (const adaptador of adaptadores) {
          try { await adaptador.disconnect(); }
          catch (e) { log.error("[ADAPTADOR] disconnect failed:", e.message || e); }
        }
        return;
      }

      const simAdapter = adaptadores.find(a => a instanceof SimAdapter);
      if (msg.cmd === "setSimHz" && simAdapter) {
        await simAdapter.handleCommand(msg);
        statusBroadcast(wss, "connected", `sim hz=${simAdapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        for (const adaptador of adaptadores) {
          try { await adaptador.handleCommand?.(msg); }
          catch (e) { log.error("[ADAPTADOR] setLed failed:", e.message || e); }
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[RED] Cliente desconectado. Quedan: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[POLÍTICA] Sin clientes. Desconectando adaptadores...");
        adaptadores.forEach(a => a.disconnect());
      }
    });
  });

  // OSC se conecta solo al arrancar (fuente pasiva que siempre escucha).
  // Strudel y micro:bit esperan el cmd:"connect" del cliente.
  for (const adaptador of adaptadores) {
    if (adaptador instanceof SimAdapter || adaptador instanceof OSCAdapter) {
      await adaptador.connect();
    }
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
````

### Sketch

````
// sketch.js
// bd    → círculo expansivo desde el centro — color y tamaño controlados por OSC
// sd/cp → barra horizontal que se encoge   — ancho controlado por OSC
// hh/oh → cuadro pequeño en pos. aleatoria
// other → rombo giratorio con líneas de cruz
//
// Dos flujos de datos conviven en este frontend:
//   strudel → cola temporal → animaciones en tiempos específicos
//   osc     → controlState  → parámetros persistentes que el draw lee cada frame

// ═══════════════════════════════════════════════════════════════════════════
// COLOR POR SONIDO — colores base, bd se sobreescribe con controlState
// ═══════════════════════════════════════════════════════════════════════════
function getColorForSound(s) {
  const palette = {
    'tr909bd': [255, 0,   80],  // solo se usa como fallback; draw lee controlState
    'tr909sd': [0,   200, 255],
    'tr909hh': [255, 255,   0],
    'tr909oh': [255, 150,   0],
  };
  if (palette[s]) return palette[s];
  const code = (s || "").charCodeAt(0) || 0;
  return [(code * 123) % 255, (code * 456) % 255, (code * 789) % 255];
}

// ═══════════════════════════════════════════════════════════════════════════
// EVENTOS DE LA FSM
// ═══════════════════════════════════════════════════════════════════════════
const EVENTS = {
  CONNECT:       "CONNECT",
  DISCONNECT:    "DISCONNECT",
  MICROBIT_DATA: "MICROBIT_DATA",
  STRUDEL_EVENT: "STRUDEL_EVENT",
  KEY_PRESSED:   "KEY_PRESSED",
  OSC_CONTROL:   "OSC_CONTROL",   // ← NUEVO: actualiza controlState
};

// ═══════════════════════════════════════════════════════════════════════════
// PAINTER TASK — FSM
// ═══════════════════════════════════════════════════════════════════════════
class PainterTask extends FSMTask {
  constructor() {
    super();

    this.microbit = {
      x: 0, y: 0, btnA: false, btnB: false,
      ready: false, circleResolution: 2, radius: 0,
    };

    this.eventQueue       = [];
    this.activeAnimations = [];

    // ── Estado persistente de controles OSC ──────────────────────────────
    // Estos valores viven entre eventos. Se actualizan cuando llega un
    // mensaje OSC y permanecen activos hasta que llegue otro.
    //
    //   /rgb_1    → color del bombo (r, g, b en 0-255)
    //   /size_1   → multiplicador de tamaño máximo (0.5 – 2.0)
    //   /trail_1  → estela activada (1) o flash instantáneo (0)
    this.controlState = {
      bdColor:        [255, 0, 80],   // color inicial del bombo
      sizeMultiplier: 1.0,            // tamaño neutro
      trailEnabled:   true,           // estela activada por defecto
    };

    this.transitionTo(this.estado_esperando);
  }

  // ── Estado: esperando conexión ──────────────────────────────────────────
  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") { cursor(); return; }
    if (ev.type === EVENTS.CONNECT) this.transitionTo(this.estado_corriendo);

    // Los controles OSC actualizan estado incluso antes de conectar Strudel.
    // Así el operador puede preparar los valores antes de que empiece la música.
    if (ev.type === EVENTS.OSC_CONTROL) {
      this._updateControl(ev.payload);
    }
  };

  // ── Estado: corriendo ───────────────────────────────────────────────────
  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") { noCursor(); background(0); return; }
    if (ev.type === "EXIT")  { cursor(); return; }

    if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
      return;
    }
    if (ev.type === EVENTS.MICROBIT_DATA) {
      this._updateMicrobit(ev.payload);
      return;
    }
    if (ev.type === EVENTS.STRUDEL_EVENT) {
      // Evento musical → va a la cola temporal.
      // No toca controlState.
      this._enqueue(ev.payload);
      return;
    }
    if (ev.type === EVENTS.OSC_CONTROL) {
      // Mensaje de control → actualiza estado persistente inmediatamente.
      // No va a la cola temporal — no tiene timestamp ni delta.
      this._updateControl(ev.payload);
      return;
    }
    if (ev.type === EVENTS.KEY_PRESSED && ev.key === " ") {
      this.activeAnimations = [];
      background(0);
    }
  };

  // ── Actualización del estado de controles OSC ───────────────────────────
  // Aquí vive la semántica de cada dirección OSC.
  // bridgeClient y la FSM no saben qué hace /rgb_1 — solo saben que es OSC.
  // Este método traduce dirección+args a variables de estado concretas.
  _updateControl({ address, args }) {
    if (address === "/rgb_1") {
      // RGB picker → color del bombo [r, g, b] en rango 0-255
      this.controlState.bdColor = [
        Number(args[0] ?? 255),
        Number(args[1] ?? 0),
        Number(args[2] ?? 80),
      ];
    }

    if (address === "/size_1") {
      // Slider → multiplicador de tamaño, acotado para evitar valores extremos
      this.controlState.sizeMultiplier = constrain(Number(args[0] ?? 1.0), 0.5, 2.0);
    }

    if (address === "/trail_1") {
      // Toggle → 1 = estela activa (background con alpha), 0 = flash (background limpio)
      this.controlState.trailEnabled = Number(args[0]) === 1;
    }
  }

  _updateMicrobit(data) {
    this.microbit.ready  = true;
    this.microbit.btnA   = data.btnA;
    this.microbit.btnB   = data.btnB;
    this.microbit.circleResolution = int(map(data.y, -2048, 2047, 2, 10));
    this.microbit.radius = map(data.x, -2048, 2047, -width / 2, width / 2);
  }

  _enqueue(payload) {
    this.eventQueue.push({
      timestamp: payload.timestamp,
      sound:     payload.s,
      soundType: payload.soundType,
      delta:     payload.delta || 0.25,
    });
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
  }

  // Activa eventos cuyo timestamp llegó → crea animaciones
  processQueue() {
    const now = Date.now();

    while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
      const ev = this.eventQueue.shift();
      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000,
        type:      ev.sound,
        soundType: ev.soundType,
        x: random(width  * 0.2, width  * 0.8),
        y: random(height * 0.2, height * 0.8),
        // Color base se guarda aquí, pero bd lo sobreescribe en draw con controlState.
        // Así el color del bombo es siempre el último recibido por OSC,
        // no el que había cuando nació la animación.
        color: getColorForSound(ev.sound),
      });
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// INSTANCIAS GLOBALES
// ═══════════════════════════════════════════════════════════════════════════
let painter;
let bridge;       // BridgeClient → Strudel   (ws://127.0.0.1:8081)
let bridgeOSC;    // BridgeClient → OSC bridge (ws://127.0.0.1:8082)  ← NUEVO
let connectBtn;

// ═══════════════════════════════════════════════════════════════════════════
// SETUP / DRAW
// ═══════════════════════════════════════════════════════════════════════════
function setup() {
  createCanvas(windowWidth, windowHeight);
  rectMode(CENTER);
  noStroke();

  painter = new PainterTask();

  // ── Bridge de Strudel (manual) ──────────────────────────────────────────
  bridge = new BridgeClient("ws://127.0.0.1:8081");

  bridge.onConnect(() => {
    connectBtn.html("Desconectar");
    painter.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    connectBtn.html("Conectar");
    painter.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => console.log("[BRIDGE Strudel]", s.state, s.detail ?? ""));

  bridge.onData((msg) => {
    if (msg.type === "microbit") {
      painter.postEvent({
        type: EVENTS.MICROBIT_DATA,
        payload: { x: msg.x, y: msg.y, btnA: msg.btnA, btnB: msg.btnB },
      });
      return;
    }
    if (msg.type === "strudel") {
      painter.postEvent({
        type: EVENTS.STRUDEL_EVENT,
        payload: { ...msg.payload, timestamp: msg.timestamp },
      });
    }
  });

  // ── Bridge de Open Stage Control (automático) ───────────────────────────
  // Este bridge se conecta solo al arrancar — no necesita botón.
  // OSC es siempre pasivo: solo actualiza valores cuando el operador mueve un control.
  bridgeOSC = new BridgeClient("ws://127.0.0.1:8081");

  bridgeOSC.onConnect(() => {
    console.log("[BRIDGE OSC] Conectado — controles activos");
  });

  bridgeOSC.onStatus((s) => console.log("[BRIDGE OSC]", s.state, s.detail ?? ""));

  bridgeOSC.onData((msg) => {
    if (msg.type === "osc") {
      // Mensaje de control → va directo a actualizar controlState vía FSM.
      // NO entra a la cola temporal de eventos musicales.
      painter.postEvent({
        type: EVENTS.OSC_CONTROL,
        payload: msg.payload,  // { address: "/rgb_1", args: [255, 120, 30] }
      });
    }
  });

  // Se abre automáticamente al iniciar la aplicación
  bridgeOSC.open();

  // ── Botón de conexión (solo para Strudel) ───────────────────────────────
  connectBtn = createButton("Conectar");
  connectBtn.position(10, 10);
  connectBtn.mousePressed(() => {
    if (bridge.isOpen) bridge.close();
    else bridge.open();
  });
}

function draw() {
  painter.update();

  if (painter.state === painter.estado_corriendo) {
    drawRunning();
  } else {
    drawWaiting();
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// DRAW RUNNING
// Solo lee estado — no toca red ni lógica de dominio.
// Lee tanto los eventos activos (Strudel) como los parámetros persistentes (OSC).
// ═══════════════════════════════════════════════════════════════════════════
function drawRunning() {
  // ── Estela controlada por /trail_1 ─────────────────────────────────────
  // trailEnabled = true  → background semitransparente → las formas dejan rastro
  // trailEnabled = false → background opaco           → efecto flash, limpia cada frame
  if (painter.controlState.trailEnabled) {
    background(0, 30);
  } else {
    background(0);
  }

  // ── 1. Animaciones de Strudel ──────────────────────────────────────────
  painter.processQueue();

  const anims = painter.activeAnimations;
  const now   = Date.now();

  for (let i = anims.length - 1; i >= 0; i--) {
    const a        = anims[i];
    const progress = (now - a.startTime) / a.duration;

    if (progress <= 1.0) {
      dibujarElemento(a, progress);
    } else {
      anims.splice(i, 1);
    }
  }

  // ── 2. Polígono del Microbit ────────────────────────────────────────────
  const mb = painter.microbit;
  if (mb.ready && mb.btnA) {
    push();
    translate(width / 2, height / 2);
    strokeWeight(2);
    if (mb.btnB) { fill(34, 45, 122, 20); stroke(34, 45, 122, 120); }
    else         { noFill(); stroke(255, 255, 255, 60); }

    const angle = TAU / mb.circleResolution;
    beginShape();
    for (let i = 0; i <= mb.circleResolution; i++) {
      vertex(cos(angle * i) * mb.radius, sin(angle * i) * mb.radius);
    }
    endShape();
    pop();
  }

  // ── 3. HUD ─────────────────────────────────────────────────────────────
  noStroke();
  fill(55);
  textAlign(RIGHT, BOTTOM);
  textSize(11);
  const { bdColor, sizeMultiplier, trailEnabled } = painter.controlState;
  text(
    `active:${painter.activeAnimations.length}  queue:${painter.eventQueue.length}` +
    `  sz:${nf(sizeMultiplier, 1, 1)}  trail:${trailEnabled ? "on" : "off"}`,
    width - 12, height - 8
  );
  textAlign(LEFT, BASELINE);
}

// ═══════════════════════════════════════════════════════════════════════════
// DRAW WAITING
// ═══════════════════════════════════════════════════════════════════════════
function drawWaiting() {
  background(0);
  noStroke();
  fill(100);
  textAlign(CENTER, CENTER);
  textSize(18);
  text("Esperando conexión…", width / 2, height / 2 - 16);
  textSize(12);
  fill(55);
  text("ws://127.0.0.1:8081  |  node bridgeServer.js --device strudel --wsPort 8081", width / 2, height / 2 + 14);
  text("ws://127.0.0.1:8082  |  node bridgeServer.js --device osc --wsPort 8082", width / 2, height / 2 + 32);
  textAlign(LEFT, BASELINE);
}

// ═══════════════════════════════════════════════════════════════════════════
// ELEMENTOS VISUALES
// ═══════════════════════════════════════════════════════════════════════════
function dibujarElemento(anim, p) {
  push();

  // Para el bombo, el color viene de controlState (vivo, actualizado por /rgb_1).
  // Para los demás sonidos, viene del color base calculado al nacer la animación.
  const c = (anim.soundType === "bd")
    ? painter.controlState.bdColor
    : anim.color;

  // El multiplicador de tamaño (/size_1) afecta al bombo y a la caja.
  const sz = painter.controlState.sizeMultiplier;

  switch (anim.soundType) {
    case "bd": dibujarBombo(p, c, sz);     break;
    case "sd": dibujarCaja(p, c, sz);      break;
    case "hh": dibujarHat(anim, p, c);     break;
    default:   dibujarDefault(anim, p, c); break;
  }
  pop();
}

// bd — círculo expansivo; tamaño máximo escalado por sizeMultiplier
function dibujarBombo(p, c, sz = 1.0) {
  const d     = lerp(100, 600 * sz, p);
  const alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2, height / 2, d);
}

// sd — barra horizontal; altura escalada por sizeMultiplier
function dibujarCaja(p, c, sz = 1.0) {
  const w     = lerp(width, 0, p);
  const alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  rect(width / 2, height / 2, w, 50 * sz);
}

// hh — cuadro pequeño en posición aleatoria fijada al nacer
function dibujarHat(anim, p, c) {
  const sz = lerp(40, 0, p);
  noStroke();
  fill(c[0], c[1], c[2]);
  rect(anim.x, anim.y, sz, sz);
}

// other — rombo giratorio con líneas de cruz
function dibujarDefault(anim, p, c) {
  const size  = lerp(100, 0, p);
  const angle = p * TWO_PI;
  translate(anim.x, anim.y);
  rotate(angle);
  stroke(c[0], c[1], c[2]);
  strokeWeight(2);
  noFill();
  rect(0, 0, size, size);
  line(-size, 0, size, 0);
  line(0, -size, 0, size);
  noStroke();
  fill(255, 150);
  textSize(14);
  text(anim.type, 10, 10);
}

// ═══════════════════════════════════════════════════════════════════════════
// UTILIDADES p5
// ═══════════════════════════════════════════════════════════════════════════
function keyPressed() {
  painter.postEvent({ type: EVENTS.KEY_PRESSED, keyCode, key });
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
````
### OSC Adapter

````
// OpenStageControlAdapter.js
// Recibe mensajes OSC de Open Stage Control por UDP,
// los normaliza y los entrega a bridgeServer.js sin tomar decisiones visuales.
//
// Uso (desde bridgeServer.js):
//   node bridgeServer.js --device osc --wsPort 8082 --oscPort 9000
//
// Protocolo de entrada (Open Stage Control → UDP):
//   OSC message: address="/rgb_1", args=[255, 120, 30]
//   OSC message: address="/size_1", args=[1.4]
//   OSC message: address="/trail_1", args=[1]
//
// Protocolo de salida (este adaptador → bridgeServer → bridgeClient):
//   { type:"osc", payload:{ address:"/rgb_1", args:[255, 120, 30] } }

const osc = require("osc");
const BaseAdapter = require("./BaseAdapter");

const DEFAULT_OSC_PORT = 9000;

// ── Normalización de argumentos OSC ──────────────────────────────────────
// La librería `osc` puede entregar los args en dos formatos distintos:
//   - Crudo:      0.3
//   - Con metadatos: { type:"f", value:0.3 }
// Esta función los aplana siempre a un número o valor simple.
function normalizeArg(a) {
  if (a != null && typeof a === "object" && "value" in a) return a.value;
  return a;
}

// ── Adaptador ─────────────────────────────────────────────────────────────
class OpenStageControlAdapter extends BaseAdapter {
  constructor({ port = DEFAULT_OSC_PORT, verbose = false } = {}) {
    super();
    this.port = port;
    this.verbose = verbose;
    this._udpPort = null;
  }

  // Abre el socket UDP y escucha mensajes OSC entrantes.
  // Open Stage Control los enviará cada vez que el usuario mueva un control.
  async connect() {
    if (this.connected) return;

    return new Promise((resolve, reject) => {
      this._udpPort = new osc.UDPPort({
        localAddress: "0.0.0.0",
        localPort: this.port,
      });

      this._udpPort.on("ready", () => {
        this.connected = true;
        this.onConnected?.(`OSC UDP escuchando en 0.0.0.0:${this.port}`);
        resolve();
      });

      this._udpPort.on("error", (err) => {
        this.onError?.(String(err?.message ?? err));
        reject(err);
      });

      // Cada mensaje OSC que llega pasa por aquí
      this._udpPort.on("message", (msg) => this._onOscMessage(msg));

      this._udpPort.open();
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    this._udpPort?.close();
    this._udpPort = null;
    this.onDisconnected?.("OSC UDP cerrado");
  }

  getConnectionDetail() {
    return `OSC UDP 0.0.0.0:${this.port}`;
  }

  // ── Normalización ────────────────────────────────────────────────────────
  // Transforma el mensaje OSC crudo en un objeto estable.
  // bridgeServer y el frontend solo ven este formato — nunca el OSC binario.
  _onOscMessage(msg) {
    const normalized = {
      address: msg.address,
      args: Array.isArray(msg.args) ? msg.args.map(normalizeArg) : [],
    };

    if (this.verbose) {
      console.log(`[OSCAdapter] ${normalized.address}`, normalized.args);
    }

    // Entregamos el dato normalizado a bridgeServer, sin tocar visuales.
    this.onData?.(normalized);
  }

  // ── Formato de broadcast ─────────────────────────────────────────────────
  // bridgeServer.js llama este método para formatear el mensaje
  // que se envía a todos los clientes WebSocket (bridgeClient.js).
  // El type:"osc" permite que el cliente distinga este flujo del de Strudel.
  formatBroadcast(d) {
    return {
      type: "osc",
      payload: {
        address: d.address,
        args: d.args,
      },
      t: Date.now(),
    };
  }
}

module.exports = OpenStageControlAdapter;
````

### JSON OSC

Aquí tenemos un panel RGB que modifica el color del Bongo, un slicer que modifica el tamaño de este y un swith que nos permite activar o desactivar un desvanecido más degradado.

<img width="595" height="362" alt="Captura de pantalla 2026-04-29 005545" src="https://github.com/user-attachments/assets/e1cf523d-79a4-405e-b251-c474b3656490" />


````
{
  "createdWith": "Open Stage Control",
  "version": "1.30.1",
  "type": "session",
  "content": {
    "type": "root",
    "lock": false,
    "id": "root",
    "visible": true,
    "interaction": true,
    "comments": "",
    "width": "auto",
    "height": "auto",
    "colorText": "auto",
    "colorWidget": "auto",
    "alphaFillOn": "auto",
    "borderRadius": "auto",
    "padding": "auto",
    "html": "",
    "css": "",
    "colorBg": "auto",
    "layout": "default",
    "justify": "start",
    "gridTemplate": "",
    "contain": true,
    "scroll": true,
    "innerPadding": true,
    "tabsPosition": "top",
    "hideMenu": false,
    "variables": "@{parent.variables}",
    "traversing": false,
    "value": "",
    "default": "",
    "linkId": "",
    "address": "auto",
    "preArgs": "",
    "typeTags": "",
    "decimals": 2,
    "target": "",
    "ignoreDefaults": false,
    "bypass": false,
    "onCreate": "",
    "onValue": "",
    "onTouch": "",
    "onPreload": "",
    "widgets": [
      {
        "type": "rgb",
        "top": 30, "left": 40,
        "lock": false, "id": "rgb_1",
        "visible": true, "interaction": true,
        "comments": "Color del bombo — /rgb_1 — args: [r,g,b] 0-255",
        "width": 220, "height": 220,
        "expand": false,
        "colorText": "auto", "colorWidget": "auto", "colorStroke": "auto",
        "colorFill": "auto", "alphaStroke": "auto", "alphaFillOff": "auto",
        "alphaFillOn": "auto", "lineWidth": "auto", "borderRadius": "auto",
        "padding": "auto", "html": "", "css": "",
        "snap": false, "spring": false,
        "range": { "min": 0, "max": 255 },
        "alpha": false,
        "rangeAlpha": { "min": 0, "max": 1 },
        "value": "", "default": "", "linkId": "",
        "address": "auto", "preArgs": "", "typeTags": "",
        "decimals": 0, "target": "", "ignoreDefaults": false, "bypass": false,
        "onCreate": "", "onValue": "", "onTouch": ""
      },
      {
        "type": "fader",
        "top": 30, "left": 300,
        "lock": false, "id": "size_1",
        "visible": true, "interaction": true,
        "comments": "Tamaño máximo de animaciones — /size_1 — args: [0.5..2.0]",
        "width": 80, "height": 220,
        "colorText": "auto", "colorWidget": "auto", "colorStroke": "auto",
        "colorFill": "auto", "alphaStroke": "auto", "alphaFillOff": "auto",
        "alphaFillOn": "auto", "lineWidth": "auto", "borderRadius": "auto",
        "padding": "auto", "html": "", "css": "",
        "snap": false, "spring": false,
        "range": { "min": 0.5, "max": 2 },
        "steps": 0, "sensitivity": 1,
        "value": 1, "default": 1, "linkId": "",
        "address": "auto", "preArgs": "", "typeTags": "",
        "decimals": 2, "target": "", "ignoreDefaults": false, "bypass": false,
        "onCreate": "", "onValue": "", "onTouch": ""
      },
      {
        "type": "switch",
        "top": 30, "left": 420,
        "lock": false, "id": "trail_1",
        "visible": true, "interaction": true,
        "comments": "Estela on/off — /trail_1 — args: [1] on / [0] off",
        "width": 120, "height": 60,
        "colorText": "auto", "colorWidget": "auto", "colorStroke": "auto",
        "colorFill": "auto", "alphaStroke": "auto", "alphaFillOff": "auto",
        "alphaFillOn": "auto", "lineWidth": "auto", "borderRadius": "auto",
        "padding": "auto", "html": "", "css": "",
        "label": "TRAIL", "led": true, "doubleTap": false,
        "off": 0, "on": 1,
        "value": 1, "default": 1, "linkId": "",
        "address": "auto", "preArgs": "", "typeTags": "",
        "decimals": 0, "target": "", "ignoreDefaults": false, "bypass": false,
        "onCreate": "", "onValue": "", "onTouch": ""
      }
    ],
    "tabs": []
  }
}
````

### Strudel

````
setcps(0.3)
//const pat = s("[bd*2 sd hh oh]").bank("tr909")
// const pat = s("[bd*2]").bank("tr909")
const pat = s("bd*4 sd hh*8 oh").bank("tr909")
// const pat = s("bd*4 sd sd mt").bank("tr909")
// const pat = s("cr").bank("tr909")

$: stack(
  pat.gain('0.5'),
  pat.osc()
)
````
## Bitácora de reflexión
