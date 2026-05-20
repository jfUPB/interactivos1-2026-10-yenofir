# Unidad 8

## Bitácora de proceso de aprendizaje

# ACTIVIDAD 01

### Arquitectura Inicial Propuesta 

<img width="1002" height="766" alt="image" src="https://github.com/user-attachments/assets/d8f03b12-6e8e-4459-b124-410770e9fb0c" />

### Adaptadores

**01 | Microbit Binary**

````
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

````

**02 | Strudel Adapter**

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

**03 | OSC Adapter**

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

### Contrato

BaseAdapter:

````
class BaseAdapter {
  constructor() {
    this.connected = false; // abrir la conexión con el hardware
    this.onData = null;
    this.onError = null;
    this.onConnected = null;
    this.onDisconnected = null; // cerrarla limpiamente
  }
/* async:son funciones asíncronas. 
Eso significa que pueden "pausarse" mientras esperan algo (como que un puerto serial abra), sin bloquear el resto del programa. 
*/
  async connect() {
    throw new Error("connect() not implemented");// Throw significa lanzar, en este caso  una señal de que algo salió mal 
  }

  async disconnect() {
    throw new Error("disconnect() not implemented");
  }

  getConnectionDetail() {
    throw new Error("getConnectionDetail() must be implemented by subclass");
  }  

  async handleCommand(_cmd) {
    console.warn("handleCommand() not implemented for command", _cmd);
    // Las subclases pueden o no sobreescribir, hacer lo que quieran ahí adentro, incluyendo cambiar lo que sale en consola.
  }
}

module.exports = BaseAdapter;
````

### Implementación

Desde las anteriones unidades con ayuda de la IA le he pedido mantener la funcionalidad y lectura de Microbit, por ende mantuvo en el código la posible implentación de esta, por eso los ajustes que tuvimos que realizar fueron:

**01 | BridgeServer**

Aquí agregamos un nuevo array, que envía a un solo puerto OSC + Strudel + Microbit, esto suponiendo que ya recibe los datos de los adaptadores, tanto Strudel como OSC formatean los mensajes con broadcast y el Server ya solo los entrega al Cliente a través de OnData.

**OnData:**

````
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
````

**Device:**
````
  if (DEVICE === "strudel+osc+microbit") {
  const path = SERIAL_PATH ?? await encontrarPuertoMicrobit();
  if (!path) {
    log.error("micro:bit no encontrado. Usa --serialPort COM3 (o el puerto que corresponda).");
    process.exit(1);
  }
  log.info("Creando adaptador Strudel   (ws://127.0.0.1:8080)");
  log.info(`Creando adaptador OSC       (UDP :${OSC_PORT})`);
  log.info(`Creando adaptador micro:bit (${path})`);
  return [
    new StrudelAdapter({ verbose: VERBOSE }),
    new OSCAdapter({ port: OSC_PORT, verbose: VERBOSE }),
    new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE }),
  ];
}
````

**02 | Sketch**

Este ya esta ajustado para recibir el mensaje tipo Microbit y activarlo:

````
 _updateMicrobit(data) {
    this.microbit.ready  = true;
    this.microbit.btnA   = data.btnA;
    this.microbit.btnB   = data.btnB;
    this.microbit.circleResolution = int(map(data.y, -2048, 2047, 2, 10));
    this.microbit.radius = map(data.x, -2048, 2047, -width / 2, width / 2);
````

**03 | Código Microbit para adaptador Binario**

````
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0, 0, 9)

HEADER = 0xAA

while True:
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    a = int(button_a.is_pressed())
    b = int(button_b.is_pressed())

    # 6 bytes de datos: X(2) + Y(2) + btnA(1) + btnB(1)
    payload = struct.pack('>2h2B', x, y, a, b)

    # checksum = suma de los 6 bytes de datos mod 256
    checksum = sum(payload) % 256

    # paquete final: header + datos + checksum = 8 bytes
    packet = bytes([HEADER]) + payload + bytes([checksum])

    uart.write(packet)
    sleep(100)
````


Entonces siendo así en nuestra terminal lanzamos la aplicación de esta forma:
````
node bridgeServer.js --device strudel+osc+microbit --wsPort 8081 --oscPort 9000
````


Pruebas de llegada de datos de todos los adaptadores:

<img width="1084" height="284" alt="Captura de pantalla 2026-05-06 153821" src="https://github.com/user-attachments/assets/46074b4b-2da3-4df1-ba01-e0172eac16df" />

## Bitácora de aplicación 
# ATIVIDAD 02
##  Concepto de la obra
La Iglesia es una performance de live coding audiovisual que usa la arquitectura visual y simbólica de una iglesia católica como materia prima para explorar tres estados de intensidad emocional: el orden representado (la nave, el vitral, la geometría sagrada), el cuerpo que sufre dentro de ese orden (los rostros, el llanto, el afán), y la violencia que lo funda (la crucifixión, la sangre, los clavos).

El referente sonoro es el dark ambient industrial: instrumental oscuro, textura de ruido blanco que se densifica hacia el clímax. La performance no ilustra ni explica la iglesia —la procesa, la desfigura y la reconstruye en tiempo real.

El sistema tiene dos estados visuales que el performer navega durante la ejecución:

**- Momento 1:** VITRIAL IGLESIA

Vitral gótico generativo. La arquitectura como símbolo: mandala, arcos, cruz de luz. El sistema está vivo pero en orden. La música lo pulsa, OSC lo tinta y lo escala, el cuerpo (micro:bit) puede romperlo.

**- Momento 2:** ¿LA IGLESIA?

Fotografías reales pixeladas como LEDs. La imagen documental degradada. El performer navega las fotos con el cuerpo, ajusta la escala con OSC. La cruz puede aparecer sobre las fotos como overlay audio-reactivo.


La obra tiene tres capas de significado que corresponden a los tres niveles anotados en el boceto conceptual:

**a. Plano detalle, Acciones. La iglesia como institución simbólica:** la geometría del vitral, el orden de los ritos, la arquitectura del poder.

**b. Dolor, llanto, Rostros enojo. Los cuerpos dentro de esa institución:** las fotografías de personas, de planos cerrados, de gestos. La experiencia humana atrapada en el símbolo.

**c. Sangre, clavos, Cruzada**. La violencia fundante.


<img width="1600" height="1011" alt="image" src="https://github.com/user-attachments/assets/47082fbd-ab01-4101-a690-a95b57a8305a" />


##  Rol de micro:bit, Strudel y Open Stage Control

**Micro:bit**

Es la única fuente que puede cambiar el estado visual del sistema. Botón B alterna entre los dos momentos. Botón A cicla fotografías en M2. El acelerómetro controla zoom y scatter de píxeles. Sin micro:bit, la obra no tiene dirección ni navegación.


**Strudel**

Cada instrumento del patrón activa una zona visual distinta. Define cuáles partes del vitral o de la fotografía respiran, pulsan o se iluminan en cada momento.

bd -> Cruz latina central + halos de luz

cp -> Burbujas celulares en el campo interior del vitral

saw (orbit 1) -> Anillos del mandala

saw (orbit 2) -> Partículas exteriores del mandala

metal -> Arcos de red del abanico superior

sd -> Nervios del abanico



**Open Stage Control**

OSC no activa eventos: sostiene el estado. El color de la luz, la escala del vitral, el blur, la aparición de la cruz sobre las fotos. Son parámetros que el performer ajusta y que permanecen hasta el siguiente cambio.

/rgb_1 -> glassColor [r,g,b] -> Tinte de la luz del vitral — define la temperatura emocional del Momento 1

/size_1 -> waveSize (0.5–2.0×) -> Escala global del vitral

/trail_1 -> blurEnabled (toggle) -> Feedback luminoso en M1

/cruz_1 -> overlayActive (toggle) -> Superpone la cruz audio-reactiva sobre las fotografías en M2

/zoom_1 -> zoom de foto -> Acercamiento a la imagen en Momento 2

/knob_1 -> pixelSize LED -> Densidad del efecto pixelado en las fotografía

/photo_N -> foto activa -> Selección directa de fotografía desde la interfaz OSC



**Combinación de las 3 Fuentes**

La cruz audio-reactiva sobre las fotografías requiere simultáneamente: que micro:bit (botón B) haya puesto el sistema en Momento 2, que OSC (/cruz_1) haya activado el overlay, y que Strudel esté enviando eventos bd que pulsen la zona nucleus. Ninguna de las tres fuentes por sí sola produce ese estado. Si el micro:bit está en M1, la cruz no aparece sobre las fotos aunque OSC la active. Si OSC no la activó, no aparece aunque el bd golpee. Si Strudel no envía bd, la cruz existe pero no pulsa. La imagen resultante —una fotografía de iglesia con una cruz luminosa que late al ritmo del bombo— es el producto de las tres fuentes en simultáneo.


## Decisiones visuales, musicales y performáticas

**Desiciones visuales**

El vitral gótico no es una referencia decorativa: es la arquitectura misma del sistema visual. Cada función de dibujo (drawNucleus, drawConcentricMandala, drawFanWeb, drawCellularField) corresponde a un elemento arquitectónico real de una vidriera gótica. La decisión de usar la paleta sepia-cálida (glassColor: [195, 162, 118] por defecto) sitúa el visual en el tiempo de los materiales: vidrio envejecido, luz filtrada, opacidad acumulada.

La pixelación LED de las fotografías en M2 es una decisión deliberada de degradación: la imagen documental (la iglesia real, fotografiada) pasa por un filtro que la convierte en datos, en puntos de luz. El performer puede controlar cuánto se ve la imagen (zoom, pixelSize) y si la cruz aparece sobre ella o no.

El blur de persistencia de frames (/trail_1) en M1 crea rastro luminoso: el vitral no se borra entre frames sino que se acumula. A mayor waveSize, mayor probabilidad de desborde visual. Esa inestabilidad es intencional —el sistema puede volverse ilegible si el performer lo decide.



**Decisiones musicales**

La decisión de usar 180 bpm como base de referencia para la escritura en Strudel define una tensión constante. El dark ambient no es lento ni contemplativo: tiene urgencia. Los patrones están diseñados para activar zonas visuales de forma diferenciada —el bd no activa lo mismo que el metal, aunque ambos sean golpes percusivos. Esa diferenciación es la que hace que el visual sea polifónico y no una masa que parpadea uniforme.

Tenemos tres momentos que coinciden con las capas conceptuales:

- Momento 1: (INTRO) Patrones dispersos, vitral en reposo, la geometría sagrada aparece lentamente.

- Momento 2: (GROOVE) Los patrones se acumulan, el performer transita a M2, las fotos aparecen.

- Momento 3: (CLIMAX) Ruido → Sonido. Todos los canales activos, la cruz sobre las fotos, el blur al máximo.



**Decisiones performáticas**

El micro:bit es visible durante la performance: el gesto de cambiar de momento es deliberadamente físico. El performer no oculta que está manipulando el sistema —lo contrario: el movimiento del cuerpo y el cambio visual son simultáneos y causales.

Strudel se toca en vivo: el live coding es parte de la performance. El performer puede alterar los patrones durante la ejecución, agregar o quitar instrumentos, cambiar densidades. El código es partitura abierta, no preset fijo.

Open Stage Control se opera en paralelo, ajustando el estado del espacio: el color de la luz, la escala del vitral, la aparición de la cruz. Son decisiones de dirección artística tomadas en tiempo real.


## Cambios realizados entre la iteración ingenieril y la iteración estética

**Orbit como distinguidor de bass1/bass2:**

en la iteración ingenieril, ambos canales "saw" llegaban a la misma zona. Se agregó el campo orbit al payload normalizado del StrudelAdapter para que el frontend pueda bifurcarlos hacia inner (mandala) y cells (partículas), dando a cada línea de bajo una expresión visual distinta.



**Cruz como overlay en M2:**

la cruz no existía sobre las fotografías en la iteración técnica. Se agregó como elemento audio-reactivo que aparece solo cuando OSC lo habilita (/cruz_1), creando la convergencia de las tres fuentes descrita arriba.



**Paleta cromática definida como intención:**

El color por defecto del vitral ([195, 162, 118]) fue elegido como punto de partida cálido-sepia. En la iteración ingenieril era un placeholder; ahora es una decisión estética fundamentada en el referente material de los vitrales.



**Arco performático de 3 momentos:**

La iteración ingenieril verificaba que las fuentes llegaban al sistema. La iteración estética define qué hace el performer con ellas y en qué orden, transformando una demo técnica en una partitura de acciones.



**ProcessQueue siempre activo independiente del momento visual:**

se modificó drawRunning para que Strudel y tickZones se ejecuten incluso en M2, de modo que la cruz sea audio-reactiva con bd aunque las fotografías sean el visual principal.



**Selección directa de foto por OSC (/photo_N):**

se agregó control desde la interfaz OSC para seleccionar fotografías específicas sin depender solo del botón A del micro:bit, dando al performer más opciones de composición visual durante la ejecución.


## Evidencias de ensayo
<img width="3840" height="1080" alt="Captura de pantalla 2026-05-18 173350" src="https://github.com/user-attachments/assets/bf61a468-1260-4064-a2c6-ee590f7816d4" />
<img width="1231" height="348" alt="Captura de pantalla 2026-05-18 154544" src="https://github.com/user-attachments/assets/1096013e-0bd9-4a13-afa6-1a6a9580c7f7" />


# ACTIVIDAD 03
## 01 | Diagrama de flujo de datos del sistema:
<img width="912" height="1095" alt="Código - Diagrama" src="https://github.com/user-attachments/assets/1dd1fe70-4782-4fbf-89da-3b92c3139c75" />

## 02 | Tabla de roles

| | micro:bit | Strudel | Open Stage Control |
| --- | --- | --- | --- |
| **Qué controla** | Cambio de momento (btnB), ciclar fotos (btnA), scatter/navegación de imagen (acelerómetro) | Pulsos visuales por zona: bombo → cruz, bajo → anillos, metal → abanico, etc. | Color de la luz, escala global, feedback/blur, pixel size, zoom, selección de foto |
| **Cómo entra** | Binario por USB Serial → `MicrobitBinaryAdapter` parsea el paquete de 8 bytes (header + X + Y + btnA + btnB + checksum) → normaliza a `{x, y, btnA, btnB}` → `bridgeServer` → WS → `bridgeClient` → evento `MICROBIT_DATA` | Strudel envía OSC-over-WebSocket a `ws://localhost:8080` → `StrudelAdapter` parsea `/dirt/play` con sus args planos `[clave, valor, ...]` → normaliza a `{soundType, s, orbit, delta, ...}` → `bridgeServer` → WS → evento `STRUDEL_EVENT` | Open Stage Control envía paquetes OSC UDP al puerto 9000 → `OSCAdapter` normaliza el mensaje a `{address, args}` → `bridgeServer` → WS → evento `OSC_CONTROL` |
| **Por qué así** | El binario de 8 bytes es mucho más eficiente que el ASCII por serial: menos parseo, sin ambigüedad de fin de línea, checksum propio para detectar corrupción. La separación en adaptador significa que `bridgeServer` no sabe nada sobre serial. | Strudel ya habla OSC internamente (SuperDirt). El adaptador crea un servidor WS propio en el 8080 para que Strudel se conecte con `.osc()`, separado del WS principal en el 8081. Así no hay conflicto de puertos. | OSC UDP es el protocolo nativo de Open Stage Control. El adaptador lo normaliza antes de que llegue al sketch, que solo ve `address` y `args` — sin saber nada de UDP. |

## 03 | Recorrido: Adapter -> bridgeServer -> bridgeClient -> FSMTask -> updateLogic -> drawRunning

**Adapter** — Cada adaptador tiene una sola responsabilidad: hablar el idioma de su fuente y entregar un objeto normalizado. MicrobitBinaryAdapter habla serial binario. StrudelAdapter habla WS + JSON. OSCAdapter habla UDP + OSC. Todos entregan datos a bridgeServer llamando this.onData(normalizado).

**bridgeServer** — Recibe de cualquier adaptador y hace broadcast a todos los clientes WebSocket conectados en el puerto 8081. Formatea el mensaje con formatBroadcast() si el adaptador lo define, o usa el formato microbit por defecto. No toma decisiones visuales — solo reenvía.

**bridgeClient** — Corre en el navegador. Recibe el mensaje JSON del WS y lo enruta según msg.type: "microbit" → callback onData, "strudel" → onData, "osc" → onData. El sketch registra ese callback con bridge.onData(msg => ...).

**FSMTask** — El sketch convierte cada mensaje en un evento tipado y lo postea a la FSM (MICROBIT_DATA, STRUDEL_EVENT, OSC_CONTROL). La FSM tiene dos estados: estado_esperando (ignora datos, muestra animación idle) y estado_corriendo (procesa todo). En estado_corriendo, cada tipo de evento llama al método correcto.

**updateLogic** — Tres caminos paralelos, cada uno actualiza una parte del estado:

`_updateMicrobit` → modifica activeVisual (btnB) o llama a ledVisual.handleMicrobit (btnA, acelerómetro)

`processQueue` → consume la cola de eventos Strudel ordenada por timestamp y llama a pulseZone("nucleus"), pulseZone("tracery"), etc.

`_updateControl` → actualiza controlState.glassColor, waveSize, blurEnabled con los valores OSC

**drawRunning** — Cada frame, p5.js llama draw(). Si la FSM está en estado_corriendo, se llama drawRunning(), que lee activeVisual, controlState y Z[zona].beat (que ya fueron actualizados por los tres caminos anteriores) y produce el visual. No recibe datos directamente — solo consume estado.

## 04 | Justificación de la propuesta estética y performática.

El recorrido busca habitar un espacio atmosférico construido en torno a la iglesia, contrastando su definición y mirada desde lo liminal, lo profundo, la perfección buscada en sus espacios y el sacrificio que conlleva alcanzarla. El vitral abre el camino —inicio, perfección geométrica, anhelo de lo divino—; las fotografías, en cambio, invitan a leer el desgaste, el sacrificio, el dolor y la sangre. Una tensión sostenida entre la forma idealizada y la materia que la sostiene.

El sonido articula y guía este tránsito en tres momentos: intro, groove y clímax.

## 05 | Evidencias de pruebas y ensayos.
<img width="2666" height="1422" alt="image" src="https://github.com/user-attachments/assets/f2e75703-288d-4401-be19-182a7b7ce080" />
<img width="1917" height="1031" alt="Captura de pantalla 2026-05-18 221904" src="https://github.com/user-attachments/assets/c8e1f1a2-0449-4833-a0d7-0dd9040f4a0a" />
<img width="2638" height="1424" alt="image" src="https://github.com/user-attachments/assets/6cd840c3-5ec6-4685-84aa-8a24d1479709" />


## Bitácora de reflexión
