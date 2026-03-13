# Unidad 4

## Bitácora de proceso de aprendizaje
### Actividad 01

Análisis del código de microbit:

```
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = "{},{},{},{}\n".format(xValue, yValue, aState,bState) # Esta llave reemplaza por los valores dados en x y, luego hace unas cadenas con este.
    # \n -> Quiere decir que el cierra con este caracter especial para decir que ya estan listos todos los datos.
    # Se puede invertir el dato, dado por el serial que lo podemos visualizar con un Serial Terminal.
    # Estos datos que muestra son datos decodificados, el fabricante me dice el protocolo.
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz 
                # En cada frame lee el valor x y del acelerometro. 
                # Igualmete esta leyendo el estado de lod botones. 
                # Estamos usando protocolo ASCII por ende con el Serial Terminal esta codificando a texto.
```
Queremos simular los botones A y B:

```
// P_2_3_1_02
//
// Generative Gestaltung – Creative Coding im Web
// ISBN: 978-3-87439-902-9, First Edition, Hermann Schmidt, Mainz, 2018
// Benedikt Groß, Hartmut Bohnacker, Julia Laub, Claudius Lazzeroni
// with contributions by Joey Lee and Niels Poldervaart
// Copyright 2018
//
// http://www.generative-gestaltung.de
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * draw tool. draw with a rotating element (svg file).
 *
 * MOUSE
 * drag                : draw
 *
 * KEYS
 * 1-4                 : switch default colors
 * 5-9                 : switch brush element
 * delete/backspace    : clear screen
 * d                   : reverse direction and mirrow angle
 * space               : new random color
 * arrow left          : rotaion speed -
 * arrow right         : rotaion speed +
 * arrow up            : module size +
 * arrow down          : module size -
 * shift               : limit drawing direction
 * s                   : save png
 */

let c;
let lineModuleSize = 0;
let angle = 0;
let angleSpeed = 1;
let lineModule = [];
let lineModuleIndex = 0;

let clickPosX = 0;
let clickPosY = 0;

function preload() {
  lineModule[1] = loadImage('data/02.svg');
  lineModule[2] = loadImage('data/03.svg');
  lineModule[3] = loadImage('data/04.svg');
  lineModule[4] = loadImage('data/05.svg');
}

function setup() {
  createCanvas(windowWidth, windowHeight);
  background(255);
  //cursor(CROSS);
  noCursor();
  strokeWeight(0.75);

  c = color(181, 157, 0);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

function draw() {
  if (mouseIsPressed && mouseButton == LEFT) {
    let x = mouseX;
    let y = mouseY;

    push();
    translate(x, y);
    rotate(radians(angle));
    stroke(c);
    line(0, 0, lineModuleSize, lineModuleSize);
    angle += angleSpeed;
    pop();
  }
}

function mousePressed() {
  // create a new random color and line length
  lineModuleSize = random(50, 160);

}

function keyReleased() {

  // change color
  if (key == ' ') c = color(random(255), random(255), random(255), random(80, 100));
  // default colors from 1 to 4
 
}
```

Con Bit Bash clonamos el repositorio e instalamos el servidor bridgeServer. 
Cuando no tenenos el promt [S]de la terminal activo no se puede ejecutar porque tenemos activa la terminal en otra cosa, es decir activo el servidor. **Para desactivar Ctrl+C → De esta manera podemos prender o apagar servidores.**

<img width="1418" height="415" alt="image" src="https://github.com/user-attachments/assets/c570503c-1a20-4d81-8679-429b63de8ca8" />


Normalmente los servidores que muestra el browser es una dirección IP local. El servidor que vamos a usar se llama Node.js nos permite ejecutar en un computador como un escritorio.
<img width="1613" height="937" alt="image" src="https://github.com/user-attachments/assets/2210912e-72b0-4515-b9ff-b96bad50e9bd" />

En nuestro caso lo hacemos a través de visual code.

**Adapter:**
El adapter es el que hace que de acuerdo a como llega la información del serial la interprete para el servidor: 

<img width="443" height="215" alt="image" src="https://github.com/user-attachments/assets/dfb6db8e-9c94-4669-b0d1-6d374b69afd4" />

El servidor es el que recoge toda la información de las diferentes entradas: simulador, microbit y las saca con datos normalizados, gracias al adapter.

<img width="2584" height="1425" alt="image" src="https://github.com/user-attachments/assets/2a14310d-6cdb-4ee5-a539-127c0eec0f0c" />


Transmitimos los datos a través del WEBSOCKET: es un protocolo de transmición, permite hacer comunicaciones entre aplicaciones y browser. → Es el que informa al usuario en el código de bridgeServer.js


**Función main**

Es la que ejecuta el sofware, lo llama y si hay un error nos dice.

```
main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
```

### Códigos anteriores con entendimiento más profundo:

**01 BaseAdapter**
```
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
```

**02 MicrobitAsciiAdapter**
```
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter"); // Hereda del Base Adapter.

class ParseError extends Error { } // Esta línea crea un tipo de error personalizado
// Crear ParseError es como tener etiquetas diferentes en tus carpetas — te permite reaccionar diferente según el tipo de problema.

function parseCsvLine(line) { //  Parsear: toma texto crudo del serial y lo convierte en datos que JavaScript puede entender.
  const values = line.trim().split(","); //Aquí es dónde se dice como se separan los valores.
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  //  ¿este valor es un número real y finito? El ! lo invierte — si no es finito, lanza el error. Esto atrapa casos como que llegue "abc" o "NaN" por el serial en lugar de un número.
 
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  // El acelerómetro del micro:bit tiene un rango físico limitado: de -2048 a 2047. Si llega un valor fuera de ese rango, algo salió mal — dato corrupto, ruido en el serial, etc. Esta línea lo rechaza.
 
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");
  // ["true", "false"].includes(btnA) pregunta: ¿btnA es exactamente la palabra "true" o "false"? Si llegó cualquier otra cosa — "1", "yes", "" — no es válido.
  
  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" }; //Una vez tenga los valores los empaqueto en esta línea para transmitir.
//Entonces para responder tu pregunta directamente: sí, la información se transforma. Entra como texto crudo "102,-45,true,false" y sale como un objeto JavaScript limpio y tipado { x: 102, y: -45, btnA: true, btnB: false }.
}


class MicrobitAsciiAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path; // Dirección del puerto serial
    this.baud = baud;// Velocidad del puerto
    this.port = null;
    this.buf = ""; // un buffer de texto para acumular datos del serial
    this.verbose = verbose;
  }

  async connect() { // Este conect es muy parecido al otro. 
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve())); //Se abre el puerto serial.
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk)); //Onchunk es la que tiene el protocolo serial ASCII, cada que lleguen datos se activa esta función. 
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed()); // De acuerdo a la función se ejecuntan las otras funciones
  } 

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
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) { // Cómo entrega los datos microbit? por partes, podría llegar en Chunks (partes)
    this.buf += chunk.toString("utf8");

    let idx; // variable idx que guardará la posición del \n dentro del buffer
    while ((idx = this.buf.indexOf("\n")) >= 0) { // Este while se repite mientras haya un \n en el buffer
      const line = this.buf.slice(0, idx).trim(); // Corta el buffer desde el inicio hasta justo antes del \n, obteniendo una línea completa. .trim() elimina espacios o caracteres invisibles sobrantes en los bordes.
      this.buf = this.buf.slice(idx + 1); // Elimina del buffer la línea que ya se extrajo (más el \n)

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);  // manda la línea a parseCsvLine() para validarla y transformarla en un objeto {x, y, btnA, btnB}
        this.onData?.(parsed); //  Si todo salió bien, llama a onData para entregarle los datos
      } catch (e) {
        if (e instanceof ParseError) { // ParseError — dato mal formateado — simplemente lo registra en consola 
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line); 
        } else {
          this._fail(e); // error inesperado y grave, llama a _fail() que desconecta todo limpiamente.
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
    // Se limpia con this.buf = "" cuando algo salió muy mal — si el buffer acumuló más de 4096 caracteres sin encontrar ningún \n, significa que los datos están corruptos o el micro:bit dejó de enviar correctamente. 
  }

  _fail(err) { // Si hay un error no identificado se desconecta el adapter.
    this.onError?.(String(err?.message || err)); //convierte el error en texto legible para humanos
    // El ?. significa "si existe" — evita que el programa explote si err no tiene una propiedad .message
    this.disconnect();
  }
/*
Closed: esta función la llama el sistema operativo automáticamente cuando el 
puerto se cierra de forma inesperada — por ejemplo si desconectas físicamente el cable USB. 
No la llamas tú, el sistema te avisa.
*/
  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  /*
 async writeLine(line):  envía datos hacia el micro:bit. Es la comunicación en sentido contrario
El => es solo una forma corta de escribir una función, como una flecha que dice "cuando pase esto, haz esto otro".
 */
  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  /*
Math.max y Math.min juntos limitan el valor a un rango válido. 
Math.trunc elimina los decimales — 3.9 se convierte en 3.
writeLine envía eso al micro:bit como texto: `LED,2,3,9\n`
Que el micro:bit interpreta como "enciende el LED en posición x=2, y=3 con brillo 9"
  */
  async handleCommand(cmd) { // Recibe un comando desde el servidor y lo ejecuta en el micro:bit
    if (cmd?.cmd === "setLed") { // setLed es el comando y es para enceder o apagar un led en el microbit
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAsciiAdapter;
// module.exports es la forma de decir "esto es lo que ofrezco al resto del sistema".
```

**03 bridgeServer**
```

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

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
// const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
}; // Son 3 funciones empaquedatas en una base de datos. Es un objeto con tres funciones para registrar mensajes en la consola. 
// son los compartimentos — cada uno contiene una función. Para usar uno escribes `log.info("mensaje")`.
// El objeto `log` **agrega automáticamente la hora y una etiqueta** antes del mensaje.

function getArg(name, def = null) { // 
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
} // 

function hasFlag(name) {
  return process.argv.includes(`--${name}`); // Los backticks ` permiten insertar variables dentro de un texto. ${name} se reemplaza por el valor de name. Si name es "device", entonces `--${name}` se convierte en "--device"
} //  verifica si una palabra aparece en la lista de argumentos, pero sin necesitar un valor después

function nowMs() { return Date.now(); } // nowMs() Retorna la hora actual en milisegundos
// ara marcar con timestamp cada mensaje enviado, para saber exactamente cuándo ocurrió.

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
} // JSON es el formato de texto que usan el servidor y el navegador para comunicarse

function broadcast(wss, obj) { // Esta función envía un mensaje a todos los navegadores conectados al servidor al mismo tiempo.
  const text = JSON.stringify(obj); // JSON.stringify convierte un objeto JavaScript en texto JSON para poder enviarlo
  for (const client of wss.clients) { //recorre cada cliente conectado
    if (client.readyState === 1) client.send(text); // verifica que ese cliente sigue conectado y listo antes de enviarle algo.
  }
}


function status(wss, state, detail = "") { // Envía mensajes de estado a todos los clientes con broadcast
  broadcast(wss, { type: "status", state, detail, t: nowMs() }); // Mensaje en {}
} 

// Las constantes de configuración:
const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();// ¿qué dispositivo usamos? (sim o microbit)
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);// ¿por qué puerta WebSocket escuchamos? (por defecto 8081)
const SERIAL_PATH = getArg("serialPort", null); // SERIAL_PATH → ¿en qué puerto físico está el micro:bit? (ej: COM5)
const BAUD = parseInt(getArg("baud", "115200"), 10); //  ¿a qué velocidad habla el serial?
const SIM_HZ = parseInt(getArg("hz", "30"), 10);// ¿cuántas veces por segundo genera datos el simulador?
const VERBOSE = hasFlag("verbose"); //  ¿imprimimos más detalles en consola?

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
} // Esta función busca automáticamente en qué puerto serial está conectado el micro:bit

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`); // Lee el microbit si no nos dice que hay un error
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // if (DEVICE === "microbit-bin") {
  //   const path = SERIAL_PATH ?? await findMicrobitPort();
  //   if (!path) {
  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
  //     process.exit(1);
  //   }
  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
  // }

  return new SimAdapter({ hz: SIM_HZ });
} // Esta función decide qué adaptador crear según el dispositivo elegido.

async function main() { // Se programan un monton de funciones 
  const wss = new WebSocketServer({ port: WS_PORT }); // Aquí se abre la "puerta" WebSocket del servidor
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);
// rea el servidor que estará escuchando conexiones entrantes desde el navegador, en el puerto que definiste antes (8081).

  const adapter = await createAdapter(); // Aquí se crea la conexión entre el simulador y el adaptador o con conecto el microbit.


  /* Los cuatro callbacks del adaptador:
  En resumen, estas cuatro líneas conectan el mundo del hardware con el mundo del navegador */
  adapter.onConnected = (detail) => { // Este onConnected es llamado del SimADAPTER Y EL SIME ADAPTER LLAMA EL Base adapter.
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail); // el servidor imprime en consola que conectó y avisa a todos los navegadores
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };// Cuando el dispositivo se desconecta avisa a todos los navegadores

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  }; // Si algo sale mal con el hardware, imprime el error y avisa a todos.

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };/*
 cada vez que llegan datos del micro:bit, los reempaqueta y los envía a todos los navegadores con broadcast
  El !!d.btnA convierte cualquier valor a booleano estricto true/false — los !! son un truco corto para garantizar ese tipo.
  */


  status(wss, "ready", `bridge up (${DEVICE})`); // Aquí empieza a transmitir 

  wss.on("connection", (ws, req) => {// wss Esta función que dice cuando alguien se conecte, quiere que ocurra todo lo que esta dento.
  // ws almacena la información de cada uno de los clientes.
  // Todo lo que está adentro se ejecuta cada vez que un navegador nuevo se conecta. ws es la conexión individual de ese cliente, req tiene info de la red como su dirección IP.
    
  
  log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => { // Escucha mensajes que llegan desde ese navegador.
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

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) { // cambia la velocidad del simulador en tiempo real.
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") { // envía el comando al micro:bit para encender un LED.
        try {
          await adapter.handleCommand?.(msg); // 
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => { // Se ejecuta cuando un navegador se desconecta
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") { // Si mi dispositivo, esta conectado es un simulador se conecta con el adapter.
    // En este caso es para el simulador ¿Pero qué pasa si es microbit? 
    // micro:bit no se conecta automáticamente — espera a que el navegador envíe el comando "connect" manualmente.
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
}); // Ejecuta toda la función main()

// El programa no termina hasta que se deje de ejecutar el servidor con la terminal. 
```
## Bitácora de aplicación 

### ACTIVIDAD 02

**Código Microbit ajustado:** contiene el nuevo formato solicitado, pasamos de `data = "{},{},{},{}\n".format(xValue, yValue, aState,bState)` a `data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, xValue, yValue, aState, bState, chk)`

```
from microbit import *
import utime

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0
    t = utime.ticks_ms()
    chk = abs(xValue) + abs(yValue) + abs(aState) + abs(bState)

    # ÚNICO CAMBIO: formato nuevo con $, |, T y CHK
    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, xValue, yValue, aState, bState, chk)
    uart.write(data)
    sleep(100)
```

**Sketch.js ajustado:** 
En updateLogic — antes mapeaba x/y a coordenadas de pantalla. Ahora mapea a las variables que necesita el arte: circleResolution para los lados del polígono y radius para el tamaño.
En drawRunning — la estructura del draw() original quedó intacta. Solo se reemplazaron los controles: btnA en lugar del mouse izquierdo, btnB en lugar del teclado.

```
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false,
            circleResolution: 2, // cuántos lados tiene el polígono — controlado por eje Y
            radius: 0            // tamaño del polígono — controlado por eje X
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            noFill();
            background(255);
            strokeWeight(2);
            stroke(0, 25);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false,
                circleResolution: 2,
                radius: 0
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        // El eje Y del acelerómetro controla cuántos lados tiene el polígono.
        // En el original era: int(map(mouseY + 100, 0, height, 2, 10))
        // Ahora el rango de entrada es el del acelerómetro (-2048 a 2047), misma salida (2 a 10)
        this.rxData.circleResolution = int(map(data.y, -2048, 2047, 2, 10));

        // El eje X del acelerómetro controla el radio del polígono.
        // En el original era: mouseX - width/2
        // Ahora el rango de entrada es el del acelerómetro (-2048 a 2047), misma salida (-width/2 a width/2)
        this.rxData.radius = map(data.x, -2048, 2047, -width/1, width/2);

        this.rxData.prevA = this.rxData.btnA;
        this.rxData.prevB = this.rxData.btnB;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(720, 720);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA, payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    let mb = painter.rxData;

    if (!mb.ready) return;

    // btnA reemplaza mouseIsPressed && mouseButton == LEFT del original
    if (mb.btnA) {
        push();
        translate(width / 2, height / 2);

        // btnB reemplaza keyIsPressed del original
        if (mb.btnB) {
            fill(34, 45, 122, 20);
        } else {
            noFill();
        }

        // Estructura de dibujo idéntica al original —
        // solo cambian las fuentes de circleResolution y radius
        let angle = TAU / mb.circleResolution;
        beginShape();
        for (let i = 0; i <= mb.circleResolution; i++) {
            let x = cos(angle * i) * mb.radius;
            let y = sin(angle * i) * mb.radius;
            vertex(x, y);
        }
        endShape();

        pop();
    }
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}

```
## Bitácora de reflexión




