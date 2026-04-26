# Unidad 6

## Bitácora de proceso de aprendizaje
## Actividad 1


**¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?**

Recibir el mensaje, me da la información pero no me dice cómo usarlo, entonces es como cuándo uno recibe un manual de instrucción para ejecutar algún dispositivo, hay elementos como un botón que activan esa información y es la que permite dar sincronía. 


**¿Por qué un sistema audiovisual puede necesitar `timestamp` además de los datos del evento?**

El timestamp es el que nos permite sincronizar lo mejor posible el sonido con las visuales. **Anticipa el momento o dice cuando tiene que mandar el evento para usar la misma base de tiempo en P5.js**


**¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?**

Usamos el Bridge, Node.js. Lo que hace es escuchar toda la información que sale del browser y las envía a las visuales, que están en el browser. 


<img width="1600" height="632" alt="WhatsApp Image 2026-04-08 at 3 03 44 PM" src="https://github.com/user-attachments/assets/f9996044-e38a-4f10-a24d-3bc025c8c0e6" />




<img width="1600" height="634" alt="image" src="https://github.com/user-attachments/assets/7e88fa69-dc50-48fe-addd-ce7c848f88fd" />


**Si Strudel fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?**

Con Strudel usamos el protocolo OSC


**¿Qué variables mínimas necesitarías extraer para poder construir una visualización útil?**

Qué sonido ocurrió, cuándo debería ocurrir, cuánto dura su ciclo rítmico, otros parámetros asociados al evento.

  **cycle:** Repetición
  
  **delta:** Duración de la animación
 
  **s:** Que nota es
  
  **bank:** Que banco es
  
  **timestamp:** Sincronización.


**¿Qué problema resuelve la cola de eventos?**

La latencia de la red, al enviar los datos, algunos pueden llegar a la vez por la misma latencia, entonces los eventos sonarían exactamente al mismo tiempo. Con la cola y el timestamp, p5.js sabe que evento suena primero, los ejecuta en el orden correcto aunque hayan llegado juntos.


**¿Por qué esta capa no pertenece al bridge sino al lado que interpreta el evento?**

Porque la función principal del bridge es enviar los datos lo más rápido posible, por ende la tarea del interprete también es decidir en que momento la activa.


**¿Qué papel cumple el Adapter en U4 y U5?**

En las unidades anteriores cada adapter nos permitía cambiar el protocolo para recibir y enviar mensajes, donde usamos desde parseo para el formato de ASCII o framing para el protocolo binario de acuerdo al protocolo, 


**¿Qué Adapter necesitas ahora para que los eventos de Strudel no entren “crudos” al sistema visual?**

Necesitamos un adapter que nos permita recibir el protocolo OSC para enviar los datos a P5.js


## Bitácora de aplicación 

## Actividad 2

### CONFIGURACIÓN STRUDEL:

En Strudel para emitir eventos usamos el protocolo OSC, sin este Strudel no envia datos al WwbSockect.Con esta línea de códgo `$: stack(pat.gain(0.7), pat.osc()) `

**Código Strudel:**

 ```
 setcps(0.5)
 
const pat = s("[bd*2 [~ sd] hh*4]").bank("tr909")
 
$: stack(
  pat.gain(0.7),
  pat.osc()           // .osc() envía los eventos al WebSocket configurado
)
 ```

### ESTRUCTURA DEL MENSAJE:

Strudel envia los datos de esta forma:

 ```
{
  address: '/dirt/play',
  args: ['cps', 0.5, 's', 'tr909bd', 'delta', 0.5],
  timestamp: 1774966984435
}
```
Los args son un array plano donde los valores van mezclados con sus nombres. Primero hacemos el parseo para diferenciar el nombre del valor.

```
function parseArgs(argsArray) {
  const result = {};
  for (let i = 0; i + 1 < argsArray.length; i += 2) {
    result[String(argsArray[i])] = argsArray[i + 1];
  }
  return result;
}
```

Luego lo clasificanos por tipo de instrumento para que el Adaptador lo pueda normalizar para el bridge:

```
function classifySound(s = "") {
  const name = String(s).toLowerCase();
  if (name.includes("bd")) return "bd";
  if (name.includes("sd") || name.includes("cp") || name.includes("sn")) return "sd";
  if (name.includes("hh") || name.includes("oh") || name.includes("ch")) return "hh";
  return "other";
```


Ya en el StudelAdapter lo normalizamos:

```
_onRawMessage(raw) {
    let msg;
    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch (e) {
      if (this.verbose) console.warn("[StrudelAdapter] Non-JSON message, ignored:", raw.toString("utf8").slice(0, 80));
      return;
    }

    // Solo procesamos mensajes /dirt/play — el canal de eventos rítmicos de SuperDirt
    if (msg.address !== "/dirt/play" || !Array.isArray(msg.args)) return;

    const params = parseArgs(msg.args);
    const soundName = String(params.s ?? "");
    const soundType = classifySound(soundName);

    const normalized = {
      timestamp: typeof msg.timestamp === "number" ? msg.timestamp : Date.now(),
      payload: {
        eventType: "noteEvent",
        soundType,          // "bd" | "sd" | "hh" | "other"  ← clasificación limpia
        s: soundName,       // nombre original: "tr909bd", "hh", etc.
        bank: String(params.bank ?? ""),
        delta: Number(params.delta ?? 0),
        cps: Number(params.cps ?? 0.5),
        cycle: Number(params.cycle ?? 0),
        gain: Number(params.gain ?? 1),
        note: params.note ?? null,
      },
    };
```

### CONEXIÓN ENTRE bridgeClient.js, FSMTask, updateLogic y drawRunning

La información se mueve a través de la siguiente ruta:
**1. bridgeClient:** recibe el mensaje del bridge

```
if (msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }
    };
```

**2. FSMTask:** decide a qué función enviarlo

**3. enqueue():**  guarda el evento en la cola

```
class PainterTask extends FSMTask {
  constructor() {
    super();

    this.microbit = {
      x: 0, y: 0, btnA: false, btnB: false,
      ready: false, circleResolution: 2, radius: 0,
    };

    this.eventQueue       = [];
    this.activeAnimations = [];

    this.transitionTo(this.estado_esperando);
  }
```

**4. processQueue():** activa los eventos cuando llega su momento

```
_enqueue(payload) {
    this.eventQueue.push({
      timestamp: payload.timestamp,
      sound:     payload.s,
      soundType: payload.soundType,
      delta:     payload.delta || 0.25,
    });
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
  }
processQueue() {
    const now = Date.now();
 if (frameCount % 60 === 0) {
    console.log(`Cola: ${this.eventQueue.length} eventos pendientes`);
  }

    while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
      const ev = this.eventQueue.shift();
      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000,
        type:      ev.sound,
        soundType: ev.soundType,
        x: random(width  * 0.2, width  * 0.8),
        y: random(height * 0.2, height * 0.8),
        color: getColorForSound(ev.sound),
      });
    }
```

**5. drawRunning():** dibuja lo que ya está activado

```
function drawRunning() {
  
  background(0, 30);

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
```

### RECEPCIÓN, COLA TEMPORAL Y RENDERIZADO

**RECEPCIÓN:** El mensaje llega en el estado corriendo del sketch. El evento entra a la cola ordenado por su timestamp.

```
if (msg.type === "strudel") {
      painter.postEvent({
        type: EVENTS.STRUDEL_EVENT,
        payload: { ...msg.payload, timestamp: msg.timestamp },
      });
    }
```
**COLA TEMPORAL:**

Se llama cada frame. El while saca todos los eventos cuyo momento ya llegó — puede ser 0, 1 o varios a la vez. Cuando saca uno, crea la animación con una posición aleatoria y el color correspondiente. La posición se fija aquí — no cambia mientras la animación vive.
¿el primer evento de la cola tiene un timestamp menor o igual a Date.now()? Si sí, lo saca de la cola y crea la animación. Si no, espera al próximo frame.

```
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
        color: getColorForSound(ev.sound),
      });
    }
  }
```

**RENDERIZADO:**

Para la animación tenemos un loop con `progress`es el que hace la animación de los elementos después de leer el estado, cuando progress esta en 0.0 es cuando inicia la animación y cuando esta en 1.0 es por que ya la termino, siendo el tamaño y el alpha proporcional:

progress = 0.0  →  inicio  (círculo pequeño, barra ancha, cuadro grande)
progress = 0.5  →  mitad   (tamaño medio, alpha medio)
progress = 1.0  →  final   (círculo grande, barra nada, cuadro nada, alpha 0)

```
for (let i = anims.length - 1; i >= 0; i--) {
    const a        = anims[i];
    const progress = (now - a.startTime) / a.duration;

    if (progress <= 1.0) {
      dibujarElemento(a, progress);
    } else {
      anims.splice(i, 1);
    }
  }
```

**PRUEBAS REALIZADAS PARA COMPROBAR ANIMACIÓN**

De las primeras pruebas es que al cambiar el tiempo de los ciclos en Strudel `setcps(0.25)` que es un poco más lento, permitiera ver la visual mucho más lento y analizar que tan sincronizado esta todo.

La otra prueba fue poner un `console.log` en el processqueue para que nos imprima el estado de la cola, con los eventos pendiendes:

```
processQueue() {
  const now = Date.now();
  // PRUEBA: imprime el estado de la cola cada 60 frames aprox.
  if (frameCount % 60 === 0) {
    console.log(`Cola: ${this.eventQueue.length} eventos pendientes`);
  }
```

La última prueba para validar en todo el tiempo el estado de la cola fue añadir un **HUB**

**active** — cuántas animaciones están dibujándose ahora mismo
**queue** — cuántos eventos están esperando en la cola

```
noStroke();
fill(55);
textAlign(RIGHT, BOTTOM);
textSize(11);
text(`active:${painter.**activeAnimations**.length}  **queue**:${painter.eventQueue.length}`, width - 12, height - 8);
```

### Códigos:

Bridge:
````
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbitv2 --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbitbinary --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device strudel --wsPort 8081  

//   WS contract (bridge → client):
//     {type:"status",  state:"ready|connected|disconnected|error", detail:"..."}
//     {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//     {type:"strudel",  timestamp:ms, payload:{eventType, soundType, s, delta, ...}}  ← NUEVO
//
//   WS contract (client → bridge):
//     {cmd:"connect"} | {cmd:"disconnect"}
//     {cmd:"setSimHz", hz:30}
//     {cmd:"setLed", x:2, y:3, value:9}

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
// NUEVO: adaptador para recibir eventos de Strudel por WebSocket
const StrudelAdapter = require("./adapters/StrudelAdapter");

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

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

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

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE    = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT   = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD      = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ    = parseInt(getArg("hz", "30"), 10);
const VERBOSE   = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p => p.vendorId && parseInt(p.vendorId, 16) === 0x0D28);
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found. Use --serialPort to specify manually."); process.exit(1); }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbitv2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found. Use --serialPort to specify manually."); process.exit(1); }
    log.info(`micro:bit V2 found at ${path}`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found. Use --serialPort to specify manually."); process.exit(1); }
    log.info(`micro:bit Binary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // NUEVO: adaptador Strudel — abre un WS server en 8080 para recibir eventos OSC de Strudel
  if (DEVICE === "strudel") {
    log.info("Creating StrudelAdapter (listening for Strudel on ws://127.0.0.1:8080)");
    return new StrudelAdapter({ verbose: VERBOSE });
  }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };

  // MODIFICACIÓN MÍNIMA: si el adapter sabe formatear su propio broadcast lo hace;
  // si no, usamos el formato microbit clásico. Esto evita poner lógica de dominio aquí.
  adapter.onData = (d) => {
    if (typeof adapter.formatBroadcast === "function") {
      // Adaptadores como StrudelAdapter definen su propio formato de broadcast
      const msg = adapter.formatBroadcast(d);
      console.log(`[${msg.type.toUpperCase()} DATA]`, JSON.stringify(msg.payload ?? d).slice(0, 120));
      broadcast(wss, msg);
    } else {
      // Formato clásico para adaptadores micro:bit y sim
      console.log(`[MICROBIT DATA] x:${d.x} y:${d.y} btnA:${d.btnA} btnB:${d.btnB}`);
      broadcast(wss, {
        type: "microbit",
        x: d.x,
        y: d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t: nowMs(),
      });
    }
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state  = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info("[NETWORK] Client requested adapter connect");
        if (adapter.connected) {
          log.info("[HW-POLICY] Adapter already open. Sending current status to incoming client.");
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        try { await adapter.connect(); }
        catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error("[ADAPTER] " + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info("[NETWORK] Client requested adapter disconnect");
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapter kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        try { await adapter.disconnect(); }
        catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error("[ADAPTER] " + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try { await adapter.handleCommand?.(msg); }
        catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error("[ADAPTER] " + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  // Los adaptadores seriales/sim se conectan al arrancar.
  // StrudelAdapter se conecta al recibir el primer cmd:"connect" del cliente.
  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});

````

**StrudelAdapter**

````
// StrudelAdapter.js
// Adaptador que recibe eventos OSC de Strudel por WebSocket (puerto 8080),
// los normaliza y los entrega a bridgeServer.js sin tomar decisiones visuales.
//
// Uso (desde bridgeServer.js):
//   node bridgeServer.js --device strudel --wsPort 8081
//
// Protocolo de entrada (Strudel → este adaptador):
//   { address: '/dirt/play', args: ['cps', 0.5, 's', 'bd', 'delta', 0.5, ...], timestamp: 1774966984435 }
//
// Protocolo de salida (este adaptador → bridgeServer → bridgeClient):
//   { type: "strudel", timestamp: <ms>, payload: { eventType, s, bank, delta, cps, gain } }

const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

// Puerto donde Strudel enviará sus eventos OSC-over-WebSocket.
// Distinto al 8081 del bridgeServer para que no haya conflicto.
const STRUDEL_WS_PORT = 8080;

// ── Parseo de los args de Strudel ──────────────────────────────────────────
// Strudel envía los parámetros como un array plano de pares [clave, valor, clave, valor, ...]
// Esta función los convierte en un objeto { cps: 0.5, s: 'bd', delta: 0.5, ... }
function parseArgs(argsArray) {
  const result = {};
  for (let i = 0; i + 1 < argsArray.length; i += 2) {
    result[String(argsArray[i])] = argsArray[i + 1];
  }
  return result;
}

// ── Clasificación del tipo de sonido ──────────────────────────────────────
// Normaliza el campo 's' de Strudel al tipo de instrumento de percusión.
// El campo 's' puede venir como "tr909bd", "bd", "tr808sd", "hh", etc.
function classifySound(s = "") {
  const name = String(s).toLowerCase();
  if (name.includes("bd")) return "bd";
  if (name.includes("sd") || name.includes("cp") || name.includes("sn")) return "sd";
  if (name.includes("hh") || name.includes("oh") || name.includes("ch")) return "hh";
  return "other";
}

// ── Adaptador ─────────────────────────────────────────────────────────────
class StrudelAdapter extends BaseAdapter {
  constructor({ verbose = false } = {}) {
    super();
    this.verbose = verbose;
    this._wss = null;
  }

  // Arranca un servidor WebSocket en STRUDEL_WS_PORT.
  // Strudel se conectará aquí y enviará sus eventos musicales.
  async connect() {
    if (this.connected) return;

    return new Promise((resolve, reject) => {
      this._wss = new WebSocketServer({ port: STRUDEL_WS_PORT });

      this._wss.on("listening", () => {
        this.connected = true;
        const detail = `Strudel WS listening on ws://127.0.0.1:${STRUDEL_WS_PORT}`;
        this.onConnected?.(detail);
        resolve();
      });

      this._wss.on("error", (err) => {
        this.onError?.(String(err.message || err));
        reject(err);
      });

      // Cada cliente Strudel que se conecte
      this._wss.on("connection", (ws, req) => {
        if (this.verbose) {
          console.log(`[StrudelAdapter] Strudel client connected from ${req.socket.remoteAddress}`);
        }

        ws.on("message", (raw) => {
          this._onRawMessage(raw);
        });

        ws.on("close", () => {
          if (this.verbose) console.log("[StrudelAdapter] Strudel client disconnected");
        });

        ws.on("error", (err) => {
          console.warn("[StrudelAdapter] Client WS error:", err.message);
        });
      });
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    await new Promise((resolve) => {
      if (!this._wss) return resolve();
      this._wss.close(() => resolve());
    });

    this._wss = null;
    this.onDisconnected?.("Strudel WS closed");
  }

  getConnectionDetail() {
    return `Strudel WS ws://127.0.0.1:${STRUDEL_WS_PORT}`;
  }

  // ── Normalización ────────────────────────────────────────────────────────
  // Transforma el mensaje crudo de Strudel en un objeto estable y tipado.
  // El resto del sistema (bridgeServer, bridgeClient, sketch.js) no conoce
  // el formato OSC de Strudel — solo ve este objeto normalizado.
  _onRawMessage(raw) {
    let msg;
    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch (e) {
      if (this.verbose) console.warn("[StrudelAdapter] Non-JSON message, ignored:", raw.toString("utf8").slice(0, 80));
      return;
    }

    // Solo procesamos mensajes /dirt/play — el canal de eventos rítmicos de SuperDirt
    if (msg.address !== "/dirt/play" || !Array.isArray(msg.args)) return;

    const params = parseArgs(msg.args);
    const soundName = String(params.s ?? "");
    const soundType = classifySound(soundName);

    const normalized = {
      timestamp: typeof msg.timestamp === "number" ? msg.timestamp : Date.now(),
      payload: {
        eventType: "noteEvent",
        soundType,          // "bd" | "sd" | "hh" | "other"  ← clasificación limpia
        s: soundName,       // nombre original: "tr909bd", "hh", etc.
        bank: String(params.bank ?? ""),
        delta: Number(params.delta ?? 0),
        cps: Number(params.cps ?? 0.5),
        cycle: Number(params.cycle ?? 0),
        gain: Number(params.gain ?? 1),
        note: params.note ?? null,
      },
    };

    if (this.verbose) {
      console.log(`[StrudelAdapter] Event: soundType=${soundType} s=${soundName} delta=${normalized.payload.delta}`);
    }

    // Entregamos el dato normalizado a bridgeServer, sin tocar visuales.
    this.onData?.(normalized);
  }

  // ── Formato de broadcast ─────────────────────────────────────────────────
  // bridgeServer.js llamará este método (si existe) para formatear el mensaje
  // que se envía a los clientes WebSocket (bridgeClient.js).
  // Esto evita que bridgeServer tenga que conocer el formato interno de Strudel.
  formatBroadcast(d) {
    return {
      type: "strudel",
      timestamp: d.timestamp,
      payload: d.payload,
      t: Date.now(),
    };
  }
}

module.exports = StrudelAdapter;
````

**Sketch:**

````
// bd    → círculo expansivo desde el centro (rojo/rosa)
// sd/cp → barra horizontal que se encoge (cyan)
// hh/oh → cuadro pequeño en posición aleatoria (amarillo)
// other → rombo giratorio con líneas de cruz (color por nombre)


// COLOR POR SONIDO — idéntico al sketch de referencia
function getColorForSound(s) {
  const palette = {
    'tr909bd': [255, 0,   80],
    'tr909sd': [0,   200, 255],
    'tr909hh': [255, 255, 0],
    'tr909oh': [255, 150, 0],
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

    this.transitionTo(this.estado_esperando);
  }

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") { cursor(); return; }
    if (ev.type === EVENTS.CONNECT) this.transitionTo(this.estado_corriendo);
  };

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
      this._enqueue(ev.payload);
      return;
    }
    if (ev.type === EVENTS.KEY_PRESSED && ev.key === " ") {
      this.activeAnimations = [];
      background(0);
    }
  };

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
    
      // PRUEBA: imprime el estado de la cola cada 60 frames aprox.
    if (frameCount % 60 === 0) {
    console.log(`Cola: ${this.eventQueue.length} eventos pendientes`);
  }

    while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
      const ev = this.eventQueue.shift();
      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000,
        type:      ev.sound,
        soundType: ev.soundType,
        x: random(width  * 0.2, width  * 0.8),
        y: random(height * 0.2, height * 0.8),
        color: getColorForSound(ev.sound),
      });
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// INSTANCIAS GLOBALES
// ═══════════════════════════════════════════════════════════════════════════
let painter;
let bridge;
let connectBtn;

// ═══════════════════════════════════════════════════════════════════════════
// SETUP / DRAW
// ═══════════════════════════════════════════════════════════════════════════
function setup() {
  createCanvas(windowWidth, windowHeight);
  rectMode(CENTER);
  noStroke();

  painter = new PainterTask();
  bridge  = new BridgeClient("ws://127.0.0.1:8081");

  bridge.onConnect(() => {
    connectBtn.html("Desconectar");
    painter.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    connectBtn.html("Conectar");
    painter.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => console.log("[BRIDGE]", s.state, s.detail ?? ""));

  bridge.onData((msg) => {
    if (msg.type === "strudel") {
    const ahora = Date.now();
    const diff  = msg.timestamp - ahora;
    console.log(`Timestamp diff: ${diff} ms`);   // + = futuro, - = pasado
    // ...resto del código
  }


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
// ═══════════════════════════════════════════════════════════════════════════
function drawRunning() {
  // Trail suave — igual que el sketch de referencia
  background(0, 30);

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
  text(`active:${painter.activeAnimations.length}  queue:${painter.eventQueue.length}`, width - 12, height - 8);
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
  textAlign(LEFT, BASELINE);
}

// ═══════════════════════════════════════════════════════════════════════════
// ELEMENTOS VISUALES — idénticos al sketch de referencia
// ═══════════════════════════════════════════════════════════════════════════
function dibujarElemento(anim, p) {
  push();
  const c = anim.color;
  switch (anim.soundType) {
    case 'bd': dibujarBombo(p, c);         break;
    case 'sd': dibujarCaja(p, c);          break;
    case 'hh': dibujarHat(anim, p, c);     break;
    default:   dibujarDefault(anim, p, c); break;
  }
  pop();
}

// bd — círculo expansivo desde el centro
function dibujarBombo(p, c) {
  const d     = lerp(100, 600, p);
  const alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2, height / 2, d);
}

// sd — barra horizontal que se encoge
function dibujarCaja(p, c) {
  const w     = lerp(width, 0, p);
  const alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  rect(width / 2, height / 2, w, 50);
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

**BridgeClient:**

````
// bridgeClient.js
// Cliente WebSocket que conecta el frontend con bridgeServer.js.
// Recibe mensajes normalizados del bridge y los enruta a los callbacks registrados.
//
// Tipos de mensaje soportados:
//   {type:"status",  state:"...", detail:"..."}   → onStatus / onConnect / onDisconnect
//   {type:"microbit", x, y, btnA, btnB, t}        → onData
//   {type:"strudel",  timestamp, payload, t}       → onData   ← NUEVO

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

      // ── Mensajes de estado del bridge ──────────────────────────────────
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

      // ── Datos del micro:bit (formato clásico) ─────────────────────────
      if (msg.type === "microbit") {
        this._onData?.(msg);
        return;
      }

      // ── Datos de Strudel (nuevo tipo) ─────────────────────────────────
      // El mensaje ya viene normalizado desde StrudelAdapter.formatBroadcast():
      //   { type:"strudel", timestamp:<ms>, payload:{ eventType, soundType, s, delta, ... } }
      // bridgeClient no interpreta ni transforma — solo enruta al consumidor (FSM).
      if (msg.type === "strudel") {
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

## Bitácora de reflexión

Diagrama de flujo de la aplicación:
<img width="1440" height="1756" alt="image" src="https://github.com/user-attachments/assets/01f13a54-c47d-430d-9386-27bba1ebf35a" />

