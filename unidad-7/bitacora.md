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

También realizamos ajustes en nuestro sketch, ya que antes había dos variables globales (bridge y bridgeOSC) apuntando a dos puertos distintos. Ahora hay una sola (bridge) apuntando al único puerto donde corre el servidor.

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

## Bitácora de reflexión
