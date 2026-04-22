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

## Bitácora de reflexión
