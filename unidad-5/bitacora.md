# Unidad 5
## Bitácora de proceso de aprendizaje
## Actividad 1
**¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?**

El protocolo binario permite tener más ancho de banda. Quiere decir que si queremos envíar muchos datos la codificación en binario reduce la cantidad de bytes usados, por ende los procesos de transmisión de datos se vuelven más optimos.

La parte mala del protocolo: es necesario tener un enter para cerrar líneas de código. Ya que podría sin esto puede ser confuso donde termina o empiezan los datos. Tenemos que estar seguros de que byte es el que marca el inicio de la trama.

Esto se hace con checksum. Que es el calcula un valor total de la seguridad.

**¿Cómo se vería el paquete en hexadecimal?**

xValue=500, yValue=524, aState=True, bState=False

01 F4 = 500
02 0C = 524
01 = True 
00 = False 

HEX: 01 F4 02 0C 01 00

Es necesario tener un enter para cerrar líneas de código. Ya que podría sin esto puede ser confuso donde termina o empiezan los datos.

**¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización?** 

El código se cerraba con una caracter de enter(\n), delimita el conjunto de datos que llega, cual es el incio y cual es el cierre.

**¿Por qué en binario no podemos usar \n como delimitador?**

Por que este es usado en el protocolo ASCII, y este caracter se puede confundir con el cierre o el final, por ende debemos utilizar framing, que marca el incio y cierre de un conjunto de datos. Además creamos un que es el calcula un valor total que damos para asegurar seguridad, es decir que si el valor total se cumple el conjunto de datos llego de manera correcta.


## Actividad 2

### Código de Adaptador Binario:

```
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

// Byte de sincronización — marca el inicio de cada paquete binario.
// 0xAA en hex = 170 en decimal.
const HEADER = 0xAA;

// Tamaño fijo de cada paquete en bytes:
// 1 (header) + 2 (X) + 2 (Y) + 1 (btnA) + 1 (btnB) + 1 (checksum) = 8 bytes
const PACKET_SIZE = 8;

class MicrobitBinaryAdapter extends BaseAdapter {

  // Constructor idéntico a los adaptadores anteriores.
  // DIFERENCIA: this.buf es Buffer.alloc(0) — buffer de bytes vacío
  // en lugar de "" — buffer de texto vacío.
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path; // Dirección del puerto serial
    this.baud = baud; // Velocidad del puerto
    this.port = null;
    this.buf = Buffer.alloc(0); // Acumula bytes en lugar de texto
    this.verbose = verbose;
  }

  // connect() idéntico a los adaptadores anteriores.
  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbitBinary device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  // disconnect() idéntico a los adaptadores anteriores.
  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  // _onChunk es donde vive TODA la lógica — framing, validación y parseo juntos.
  // A diferencia de los adaptadores anteriores, no hay función externa de parseo
  // porque en binario el framing y el parseo están completamente entrelazados:
  // no puedes parsear sin antes haber encontrado el 0xAA y verificado el checksum.
  _onChunk(chunk) {

    console.log(`[CHUNK RECIBIDO] ${chunk.length} bytes: ${chunk.toString('hex').toUpperCase()}`);

    // Acumulamos bytes con Buffer.concat.
    // DIFERENCIA con ASCII: en texto era this.buf += chunk.toString("utf8")
    // Aquí concatenamos bytes directamente sin convertir a texto.
    this.buf = Buffer.concat([this.buf, chunk]);

    // Procesamos todos los paquetes completos disponibles en el buffer.
    while (this.buf.length >= PACKET_SIZE) {

      // FRAMING PASO 1: buscamos el byte 0xAA — inicio del paquete.
      // DIFERENCIA con ASCII: en texto buscábamos "\n" con indexOf("\n")
      // Aquí buscamos el byte 0xAA con indexOf(0xAA).
      const headerIdx = this.buf.indexOf(HEADER);

      // Si no hay 0xAA en todo el buffer, descartamos todo — no hay nada útil.
      if (headerIdx === -1) {
        this.buf = Buffer.alloc(0);
        break;
      }

      // Si el 0xAA no está al inicio, descartamos los bytes previos — son basura.
      // Ejemplo: buffer = [01 FF AA 01 F4 ...] → descartamos [01 FF]
      if (headerIdx > 0) {
        this.buf = this.buf.slice(headerIdx);
        continue;
      }

      // FRAMING PASO 2: tenemos 0xAA al inicio pero no suficientes bytes todavía.
      // Esperamos más chunks antes de continuar.
      if (this.buf.length < PACKET_SIZE) break;

      // Tenemos 0xAA al inicio y exactamente 8 bytes — extraemos el paquete.
      const packet = this.buf.slice(0, PACKET_SIZE);

      // VALIDACIÓN DEL CHECKSUM: suma de bytes 1 a 6, módulo 256.
      // buf.slice(1, 7) extrae los bytes de datos (sin header ni checksum).
      // reduce() los suma acumulativamente desde 0.
      // % 256 asegura que el resultado cabe en un byte (0 a 255).
      const expectedChk = packet.slice(1, 7).reduce((sum, byte) => sum + byte, 0) % 256;
      const receivedChk = packet.readUInt8(7);

      if (expectedChk !== receivedChk) {
        // Checksum inválido — paquete corrupto, lo descartamos silenciosamente.
        // Avanzamos solo 1 byte para no perdernos el siguiente 0xAA válido.
        if (this.verbose) console.log(`[MicrobitBinaryAdapter] Corrupt packet discarded: checksum mismatch got ${receivedChk}, expected ${expectedChk}`);
        else console.warn(`[MicrobitBinaryAdapter] Corrupt packet discarded`);
        this.buf = this.buf.slice(1);
        continue;
      }

      // PARSEO: paquete válido — leemos los bytes en sus posiciones.
      // readInt16BE lee 2 bytes con signo en Big Endian — para X e Y del acelerómetro.
      // readUInt8 lee 1 byte sin signo — para los botones.
      const x    = packet.readInt16BE(1); // bytes 1-2 → acelerómetro X
      const y    = packet.readInt16BE(3); // bytes 3-4 → acelerómetro Y
      const btnA = packet.readUInt8(5) === 1; // byte 5 → botón A como booleano
      const btnB = packet.readUInt8(6) === 1; // byte 6 → botón B como booleano

      // Emitimos el objeto — contrato idéntico al de todos los adaptadores anteriores.
      // El servidor y el sketch no saben ni les importa que esto vino en binario.
      this.onData?.({ x, y, btnA, btnB });

      // Avanzamos el buffer exactamente PACKET_SIZE bytes — eliminamos el paquete procesado.
      // DIFERENCIA con ASCII: en texto avanzábamos hasta el \n con slice(idx + 1).
      // Aquí siempre avanzamos exactamente 8 bytes.
      this.buf = this.buf.slice(PACKET_SIZE);
    }

    // Red de seguridad: buffer demasiado grande sin paquetes válidos — limpiamos.
    if (this.buf.length > 4096) this.buf = Buffer.alloc(0);
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }
}

module.exports = MicrobitBinaryAdapter;

```

Como esta en los comentarios del código, los nuevos cambios se dan principalmente en que hacemos framing, el parseo esta dentro del _OnChunk, en primera instancia acumulamos en el buffer el conjunto de datos y en esta misma función validamos que los datos incien con el header y que sean 8, para el cierre, también hacemos el checksum que valida que los datos sean correctos para enviarlos. En esta misma función hacemos leemos los datos en Big Endian y readUInt8 para los botones.
Así enviamos los datos a través del onData y cerramos con la validación de seguridad del buffer.


**Validación de qué los datos enviados en binario:**


<img width="1865" height="980" alt="image" src="https://github.com/user-attachments/assets/93d73a3c-4a98-4e6d-bf6c-9162cb62a892" />


**Código del BridgeServer**

```
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbitv2 --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
// NUEVO: importamos el adaptador para el nuevo protocolo
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
// const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
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
  try {
    return JSON.parse(s);
  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
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

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // NUEVO: caso para el nuevo protocolo V2
  // Se activa con --device microbitv2 en la terminal
  if (DEVICE === "microbitv2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit V2 found at ${path}`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // NUEVO: caso para el protocolo binario.
  // Se activa con --device microbitBinary en la terminal.
  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit Binary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
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

  adapter.onData = (d) => {
    console.log(`[MICROBIT DATA] x:${d.x} y:${d.y} btnA:${d.btnA} btnB:${d.btnB}`);
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);
        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        try {
          await adapter.connect();
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
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
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
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

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});

```
El cambio principal se da eb qye se agrega una constante que llama al adaptador y creamos una condición de activación del protocolo binario. Si el usuario arracon con microbitbinary, crea la constante path para buscar el puerto, si no lo encuentra imprime error. Si lo encuentra imprime donde lo encontro.

```
 if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit Binary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

```


## Bitácora de aplicación 


## Bitácora de reflexión
