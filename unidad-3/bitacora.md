# Unidad 3

## Bitácora de proceso de aprendizaje
### Actividad 01

De acuerdo a los estados anteriores generamos un estado nocturno que tiene una condición que encendía o apagaba el pixel, generando una repetición en el encendido y apagado del pixel.

Además, generamos el evento 'A' dentro del estado nocturno para realizar una transición al semáforo en el pixel amarillo:

````
def estado_nocturno(self,ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)

        if ev == "Timeout":
            if display.get_pixel(self.x,self.y+1) == 0:
                display.set_pixel(self.x,self.y+1,9)
                
            else:
                display.set_pixel(self.x,self.y+1,0)
                
            self.myTimer.start(self.timeInYellow)
        
        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

````
Se agregaron 2 Eventos más en el `estado_waitInGreen` , el evento del botón A y el botón B. 
**A** - ir a modo peaton **B** - modo nocturno:

````
if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

        if ev == "B":
            self.transicion_a(self.estado_nocturno)
````

Entonces el código completo en el **main.py** quedaría así:
````
from microbit import *
import utime

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration

        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout",self.timeInRed)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transicion_a(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)

        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_nocturno(self,ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)

        if ev == "Timeout":
            if display.get_pixel(self.x,self.y+1) == 0:
                display.set_pixel(self.x,self.y+1,9)
                
            else:
                display.set_pixel(self.x,self.y+1,0)
                
            self.myTimer.start(self.timeInYellow)
        
        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
            
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transicion_a(self.estado_waitInRed)
            
        if ev == "B":
            self.transicion_a(self.estado_nocturno)

semaforo1 = Semaforo(0,0,2000,1000,500)

while True:

    if button_a.was_pressed():
        semaforo1.post_event("A")

    if button_b.was_pressed():
        semaforo1.post_event("B")
    
    semaforo1.update()
    utime.sleep_ms(20)

````
### Actividad 02
#### Solución Punto 1
Modificamos el temporizador, necesitamos una orden que nos permita pausar el conteo, en la solicitud se espera que sea con el botón A, sin embargo, este será un problema cuando realicemos la solicitud 2.

En el estado de configuración añadimos un evento que nos hace la transición al estado de armado. Donde inicia el conteo regresivo.

````
if ev == "S":
            self.transition_to(self.estado_armed)
````

Para pausar el conteo hicimos con un evento en el estado de armado, el cual tiene una condición que llama el timer y le dice que si se usa la función 'S' el temporizador se pausara o se renaudara.

````
if ev == "S":
            if self.myTimer.active == False:
                self.myTimer.start()
            else:
                self.myTimer.stop()
````

#### Solución Punto 2

Primero creamos una secuencia que nos almacenará los datos de los botones presionados y definimos la contraseña de acceso para habilitar de nuevo el estado de configuración:
````
self.sequence = []
self.myPassword = ["A", "B", "A"]
````

También añadimos el evento de la secuencia de botones A-B-A, el temporizador debe volver a modo de configuración. 

El evento me dice que  hace la lectura de los botones presionados y guarda estos datos `self.sequence.append(ev)` , nos dice que si son igual a 3 y son los datos de la contraseña (A-B-A) nos lleva al estado de configuración, si no limpia la secuencia para que podamos volver a ingresar dicha contraseña, por ende solo nos permite pasar al estado de configuración presionando los botones en la secuencia correcta.

````
if ev == "A" or ev == "B":
            self.sequence.append(ev)
            if len(self.sequence) == 3:
                if self.sequence == self.myPassword:
                    self.transition_to(self.estado_config)
                else:
                    self.sequence.clear()
````
Entonces el main.py quedaría así:

```
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.sequence = []
        self.myPassword = ["A", "B", "A"]
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)
        self.estado_actual = None
        self.transition_to(self.estado_config)


    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.myTimer.start()
        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])
        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])
        if ev == "S":
            self.transition_to(self.estado_armed)

    def estado_armed(self, ev):
        if ev == ENTRY:
            self.sequence.clear()
            self.myTimer.start()
        if ev == "Timeout":
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])
                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()
                    
        if ev == "S":
            if self.myTimer.active == False:
                self.myTimer.start()
            else:
                self.myTimer.stop()
                
#Esto es para sumar las presiones de A Y B en el punto 2 de la actividad.
        
        if ev == "A" or ev == "B":
            self.sequence.append(ev)
            if len(self.sequence) == 3:
                if self.sequence == self.myPassword:
                    self.transition_to(self.estado_config)
                else:
                    self.sequence.clear()
            

    def estado_timeout(self, ev):
        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)
        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)

temporizador = Temporizador()

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")
    if button_b.was_pressed():
        temporizador.post_event("B")
    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")

    temporizador.update()
    utime.sleep_ms(20)

```

### Actividad 03

Trabajamos con varios archivos:

**index.html** → La estructura base que une todo

**style.css** → El aspecto visual de la página

**fsm.js** → La máquina de estados

**sketch.js**→ El temporizador en sí y todo lo visual

**jsconfig.json** → Configuración técnica para VS Code

#### index.html: El index nos une todo

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Sketch</title>

    <link rel="stylesheet" type="text/css" href="style.css">

    <!-- Carga la librería de P5, desde internet. -->

    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  </head>

  <body>

    <!-- En este caso importa el orden porque el código carga línea a línea, quiere decir que en los siguiente scripts, primero llama al fsm.js que es la máquina de estados, es decir nuestro Cerebro
    Y por último nuestro el sketch ya que este usa elementos de la mádquina de estados fsm.js -->

    <script src="fsm.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>

```

#### style.css : El estilo
Elimina las margenes para que el canvas de P5 ocupe toda la pantalla sin bordes.

```
html, body {
  margin: 0;
  padding: 0;
}

canvas {
  display: block;
}
```
#### fsm.py : La máquina de estados
Este archivo no define estados ni eventos concretos. Solo crea las herramientas para manejarlos. Es como construir el motor de un carro sin decidir aún a dónde va a ir.

```
const ENTRY = "ENTRY";
const EXIT = "EXIT";

// Este es un temporizador. Es el que hace la cuenta en 1000ms para que descuente un segundo y cuando termina le avisa a su dueño owner enviando un evento.
// this es una palabra clave que hace referencia al objeto actual que está ejecutando el código o a quien "posee" la función.

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;  // ¿A quién le avisa cuando termina?
    this.event = eventToPost; // ¿Qué evento envía cuando termina?
    this.duration = duration; // ¿Cuánto tiempo espera? (en milisegundos)
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis(); // millis() es una función de P5 que dice cuántos milisegundos han pasado desde que arrancó el programa.
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) { //La resta millis() - this.startTime mide cuánto tiempo ha transcurrido.
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = []; // Lista de eventos pendientes
    this.timers = []; // Lista de temporizadores
    this.state = null; // Estado actual
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT); // Le dice al estado actual "saliste"
    this.state = newState;
    this.state(ENTRY); // Le dice al nuevo estado "entraste"
    //Cada vez que cambias de estado, el estado anterior recibe un evento EXIT y el nuevo recibe un evento ENTRY. Esto permite ejecutar acciones específicas al entrar o salir de un estado.
  }

  update() {
    for (let t of this.timers) { 
      t.update(); // Revisa si algún timer terminó
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift(); // Toma el primer evento pendiente
      if (this.state) this.state(ev);  // Se lo manda al estado actual
      // Este método se llama 60 veces por segundo (por P5) y es el que mantiene todo funcionando.
    }
  }
}

```
`if (newDuration !== null) this.duration = newDuration;` 
`!==` significa "es diferente de". Y null en JavaScript significa "no existe" o "está vacío intencionalmente". Entonces esto se lee: "si el nuevo valor que me pasaron no es vacío, úsalo".



#### sketh.js : el temporizador + visual de este

**Eventos** (definidos en el objeto `EVENTS`):

- `DEC → "A"` → reducir el valor
- `INC → "B"` → aumentar el valor
- `START → "S"` → iniciar la cuenta
- `TICK → "Timeout"` → señal de cada segundo


```
// El temporizador puede configurarse entre 15 y 25 segundos, con 20 como valor por defecto.
const TIMER_LIMITS = { 
  min: 15,
  max: 25,
  defaultValue: 20,
};
// Estos son todos los eventos posibles. A reduce, B aumenta, S inicia, y Timeout es el que manda el Timer cada segundo.
const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};
//-> NO ENTIENDO 
const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};


// Esta clase hereda de FSMTask (con extends), lo que significa que tiene todo lo de FSMTask más sus propios estados:
// HERENCIA:  Temporizador hereda de FSMTask. Eso significa que Temporizador ya tiene todo lo de FSMTask (la cola de eventos, los timers, el método update, etc.) sin repetir código.
class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super(); // Llama al constructor de FSMTask -> NO ENTIENDO CÓMO
    // -> Cuando Temporizador nace (se crea), también necesita que FSMTask inicialice sus propias cosas (la cola queue, la lista timers, el state). El super() es literalmente decirle: "oye FSMTask, ejecuta tu constructor primero". Sin esto, las cosas heredadas de FSMTask no existirían y el programa fallaría.


    //this significa "este objeto en particular"
    this.minValue = minValue; 
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;
    
    this.myTimer = this.addTimer(EVENTS.TICK, 1000); // Timer de 1 segundo
    this.transitionTo(this.estado_config); // Empieza en config

  }

  get currentState() { //-> NO ENTIENDO 
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue; // Resetea a 20
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;  /* reduce el valor */ 
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++; /* aumenta el valor */ 
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start(); // Arranca el timer de 1 segundo
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--; // Descuenta un segundo
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout); // ¡Se acabó!
        } else {
          this.myTimer.start();  // Reinicia el timer para el próximo segundo
        }
      }
    } else if (ev === EXIT) {
      this.myTimer.stop(); // Para el timer al salir
    }

  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config); // Vuelve al inicio con tecla A
    }
  }
}

let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);
// El renderer es un mapa que asocia cada estado con su función de dibujo correspondiente. Si el estado es estado_config, dibuja la pantalla de configuración; si es estado_armed, dibuja el anillo animado, etc.
  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {
  temporizador.update(); // Actualiza la lógica
  renderer.get(temporizador.currentState)?.();  // Dibuja según el estado
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}
//Esta función escucha el teclado y traduce las teclas en eventos. Cuando conectes el Microbit, 
// los botones físicos A y B harán exactamente lo mismo que las teclas A y B del teclado.

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent("A");
  if (key === "b" || key === "B") temporizador.postEvent("B");
  if (key === "s" || key === "S") temporizador.postEvent("S");
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

```

### ACTIVIDAD 04

Al presionar los botones le estaba enviando a p5.js con print la presión de cada botón en el microbit. Por ende en usos más generales de acuerdo al sistema lo ideal es usar uart.write ya que nos permite finalizar el mensaje enviado de acuerdo con el sistema que lo reciba, en este caso sería '\n' ya que es JavaScrip.

**Código Microbit:**

```
from microbit import *
import utime
uart.init(baudrate=115200)

while True:
    if button_a.was_pressed():
        uart.write('A\n')                  
        
    if button_b.was_pressed():
        uart.write('B\n')                    
      
    if accelerometer.was_gesture("shake"):
       uart.write('S\n')                   

    utime.sleep_ms(20)
```


Código P5.js:


```

// ─────────────────────────────────────────────────────────────
//  CONSTANTES DE CONFIGURACIÓN
// ─────────────────────────────────────────────────────────────
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",      // Botón A  → decrementar / pausar / parte de la secuencia
  INC: "B",      // Botón B  → incrementar / parte de la secuencia
  START: "S",    // Tecla S o agitar micro:bit → iniciar
  TICK: "Timeout",
};
6
const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};

// ─────────────────────────────────────────────────────────────
//  SERIAL  (Web Serial API → micro:bit por USB)
// ─────────────────────────────────────────────────────────────
let serialPort = null;
let serialConnected = false;
let serialBuffer = "";
let connectButton;

// El usuario hace clic en el botón → abre el diálogo del navegador
// para seleccionar el puerto USB del micro:bit
async function connectSerial() {
  try {
    serialPort = await navigator.serial.requestPort();
    await serialPort.open({ baudRate: 115200 });
    serialConnected = true;
    connectButton.hide();        // Oculta el botón una vez conectado
    readSerialLoop();            // Inicia la lectura continua
  } catch (e) {
    console.warn("No se pudo conectar al micro:bit:", e);
  }
}

// Lee el puerto serie en un loop asíncrono.
// Cada línea recibida ("A\n", "B\n", "S\n") se traduce en un evento.
async function readSerialLoop() {
  const decoder = new TextDecoderStream();
  serialPort.readable.pipeTo(decoder.writable);
  const reader = decoder.readable.getReader();

  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      serialBuffer += value;
      const lines = serialBuffer.split("\n");
      serialBuffer = lines.pop();          // deja la línea incompleta para después

      for (const line of lines) {
        const cmd = line.trim().toUpperCase();
        if (cmd === "A") temporizador.postEvent(EVENTS.DEC);
        else if (cmd === "B") temporizador.postEvent(EVENTS.INC);
        else if (cmd === "S") temporizador.postEvent(EVENTS.START);
      }
    }
  } catch (e) {
    serialConnected = false;
    connectButton.show();
    console.warn("Serial desconectado:", e);
  }
}

// ─────────────────────────────────────────────────────────────
//  CLASE TEMPORIZADOR  (extiende FSMTask de fsm.js)
// ─────────────────────────────────────────────────────────────
class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    // PUNTO 1: bandera para saber si está pausado
    this.paused = false;

    // PUNTO 2: historial de últimas teclas para detectar A-B-A
    this.sequence = [];
    this.password = [EVENTS.DEC, EVENTS.INC, EVENTS.DEC]; // ["A","B","A"]

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  // ── Estado 1: Configuración ──────────────────────────────────
  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    } else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  // ── Estado 2: Corriendo ──────────────────────────────────────
  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
      this.sequence = [];    // limpia la secuencia al entrar
      this.paused = false;
    }

    // PUNTO 2: A y B se acumulan en la secuencia
    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {
      this.sequence.push(ev);

      // Cuando hay 3 elementos, verificamos si es la contraseña A-B-A
      if (this.sequence.length === 3) {
        if (this.sequence.join("") === this.password.join("")) {
          // ¡Secuencia correcta! → volver a configuración
          this.transitionTo(this.estado_config);
          return; // salimos antes de ejecutar el toggle de pausa
        } else {
          // No coincide: ventana deslizante (guarda el último elemento)
          // Así el usuario puede intentar de nuevo solapando
          this.sequence = [this.sequence[2]];
        }
      }

      // PUNTO 1: solo la tecla A pausa / reanuda
      if (ev === EVENTS.DEC) {
        if (!this.paused) {
          this.paused = true;
          this.myTimer.stop();        // detiene el contador
        } else {
          this.paused = false;
          this.myTimer.start();       // reanuda desde donde se quedó
        }
      }
    }

    // El tick llega cada 1 segundo cuando no está pausado
    else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();  // reinicia el timer para el siguiente segundo
        }
      }
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
      this.paused = false;
    }
  };

  // ── Estado 3: Tiempo agotado ─────────────────────────────────
  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}

// ─────────────────────────────────────────────────────────────
//  P5.JS — setup y draw
// ─────────────────────────────────────────────────────────────
let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  // Cada estado tiene su propia función de dibujo
  renderer.set(temporizador.estado_config, () =>
    drawConfig(temporizador.configValue)
  );
  renderer.set(temporizador.estado_armed, () =>
    drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, temporizador.paused)
  );
  renderer.set(temporizador.estado_timeout, () => drawTimeout());

  // Botón HTML para conectar el micro:bit (solo aparece si no está conectado)
  connectButton = createButton("🔌 Conectar micro:bit");
  connectButton.position(width/2.25, height/5);
  connectButton.style("font-size", "14px");
  connectButton.style("padding", "8px 16px");
  connectButton.style("cursor", "pointer");
  connectButton.style("border-radius", "6px");
  connectButton.style("border", "none");
  connectButton.style("background", "#2600ff");
  connectButton.style("color", "white");
  connectButton.mousePressed(connectSerial);
}

function draw() {
  temporizador.upd6666666ate();
  renderer.get(temporizador.currentState)?.();

  // Indicador de conexión serial (esquina superior derecha)
  drawSerialStatus();
}

// ─────────────────────────────────────────────────────────────
//  FUNCIONES DE DIBUJO
// ─────────────────────────────────────────────────────────────
function drawConfig(val) {
  background("#000000");
  fill(255);
  textSize(UI.configText);
  text(val, width / 2, height / 2);
  textSize(UI.helpText);
  fill(200);
  text("A(−)   B(+)   S / agitar micro:bit (iniciar)", width / 2, height / 2 + 100);
}

function drawArmed(val, total, paused) {
  background(20, 20, 20);

  // Anillo de fondo (apagado)
  noFill();
  strokeWeight(UI.ringWeight);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, UI.dialSize);

  // Arco de progreso — azul si pausado, naranja si corriendo
  stroke(paused ? color(80, 130, 255) : color(255, 150, 0));
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, UI.dialSize, UI.dialSize, -HALF_PI, angle - HALF_PI);

  // Número central — sin pulso si pausado
  let pulse = paused ? 0 : sin(frameCount * 0.1) * 10;
  fill(paused ? color(80, 130, 255) : 255);
  noStroke();
  textSize(UI.bigText + pulse);
  text(val, width / 2, height / 2);

  // Texto de ayuda según estado
  textSize(UI.helpText);
  if (paused) {
    fill(80, 130, 255);
    text("⏸ PAUSADO  —  A para reanudar", width / 2, height / 2 + 180);
  } else {
    fill(160);
    text("A pausar  ·  secuencia A-B-A → configurar", width / 2, height / 2 + 180);
  }
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(UI.bigText);
  text("¡TIEMPO!", width / 2, height / 2);
  textSize(UI.helpText);
  fill(255, 200, 200);
  text("A para volver a configurar", width / 2, height / 2 + 80);
}

function drawSerialStatus() {
  noStroke();
  fill(serialConnected ? color(0, 220, 100) : color(120));
  ellipse(width - 20, 20, 12);
  textSize(12);
  fill(serialConnected ? color(0, 220, 100) : color(120));
  textAlign(RIGHT, CENTER);
  text(serialConnected ? "micro:bit conectado" : "sin micro:bit", width - 30, 20);
  textAlign(CENTER, CENTER); // restaura la alineación global
}

// ─────────────────────────────────────────────────────────────
//  TECLADO  (funciona en paralelo con el micro:bit)
// ─────────────────────────────────────────────────────────────
function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent(EVENTS.DEC);
  if (key === "b" || key === "B") temporizador.postEvent(EVENTS.INC);
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```


### ACTIVIDAD 05

**Código para enviar mensajes:**

```
from microbit import *
import utime
import radio

radio.config(group=12)
radio.on()


while True:
    if button_a.was_pressed():
        radio.send("A")        

    if button_b.was_pressed():
        radio.send("B")         

    if accelerometer.was_gesture("shake"):
        radio.send("S")        

    utime.sleep_ms(20)

```


**Código de recibir los mensajes de la microbit remota:**

```
from microbit import *
import utime
import radio

radio.config(group=12)
radio.on()

uart.init(baudrate=115200)

while True:
    if button_a.was_pressed():
        uart.write('A\n')                  # ← nuevo: avisa a p5.js

    if button_b.was_pressed():
        uart.write('B\n')                 # ← nuevo: avisa a p5.js

    if accelerometer.was_gesture("shake"):
        uart.write('S\n')  
        
    message = radio.receive()
    if message:
        
        if message=='A':
            uart.write('A\n') 
            
        if message=='B':
            uart.write('B\n') 

        if message=='S':
            uart.write('S\n') 


```

**Código de p5.js:**

```

// ─────────────────────────────────────────────────────────────
//  CONSTANTES DE CONFIGURACIÓN
// ─────────────────────────────────────────────────────────────
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",      // Botón A  → decrementar / pausar / parte de la secuencia
  INC: "B",      // Botón B  → incrementar / parte de la secuencia
  START: "S",    // Tecla S o agitar micro:bit → iniciar
  TICK: "Timeout",
};
6
const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};

// ─────────────────────────────────────────────────────────────
//  SERIAL  (Web Serial API → micro:bit por USB)
// ─────────────────────────────────────────────────────────────
let serialPort = null;
let serialConnected = false;
let serialBuffer = "";
let connectButton;

// El usuario hace clic en el botón → abre el diálogo del navegador
// para seleccionar el puerto USB del micro:bit
async function connectSerial() {
  try {
    serialPort = await navigator.serial.requestPort();
    await serialPort.open({ baudRate: 115200 });
    serialConnected = true;
    connectButton.hide();        // Oculta el botón una vez conectado
    readSerialLoop();            // Inicia la lectura continua
  } catch (e) {
    console.warn("No se pudo conectar al micro:bit:", e);
  }
}

// Lee el puerto serie en un loop asíncrono.
// Cada línea recibida ("A\n", "B\n", "S\n") se traduce en un evento.
async function readSerialLoop() {
  const decoder = new TextDecoderStream();
  serialPort.readable.pipeTo(decoder.writable);
  const reader = decoder.readable.getReader();

  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      serialBuffer += value;
      const lines = serialBuffer.split("\n");
      serialBuffer = lines.pop();          // deja la línea incompleta para después

      for (const line of lines) {
        const cmd = line.trim().toUpperCase();
        if (cmd === "A") temporizador.postEvent(EVENTS.DEC);
        else if (cmd === "B") temporizador.postEvent(EVENTS.INC);
        else if (cmd === "S") temporizador.postEvent(EVENTS.START);
      }
    }
  } catch (e) {
    serialConnected = false;
    connectButton.show();
    console.warn("Serial desconectado:", e);
  }
}

// ─────────────────────────────────────────────────────────────
//  CLASE TEMPORIZADOR  (extiende FSMTask de fsm.js)
// ─────────────────────────────────────────────────────────────
class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    // PUNTO 1: bandera para saber si está pausado
    this.paused = false;

    // PUNTO 2: historial de últimas teclas para detectar A-B-A
    this.sequence = [];
    this.password = [EVENTS.DEC, EVENTS.INC, EVENTS.DEC]; // ["A","B","A"]

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  // ── Estado 1: Configuración ──────────────────────────────────
  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    } else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  // ── Estado 2: Corriendo ──────────────────────────────────────
  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
      this.sequence = [];    // limpia la secuencia al entrar
      this.paused = false;
    }

    // PUNTO 2: A y B se acumulan en la secuencia
    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {
      this.sequence.push(ev);

      // Cuando hay 3 elementos, verificamos si es la contraseña A-B-A
      if (this.sequence.length === 3) {
        if (this.sequence.join("") === this.password.join("")) {
          // ¡Secuencia correcta! → volver a configuración
          this.transitionTo(this.estado_config);
          return; // salimos antes de ejecutar el toggle de pausa
        } else {
          // No coincide: ventana deslizante (guarda el último elemento)
          // Así el usuario puede intentar de nuevo solapando
          this.sequence = [this.sequence[2]];
        }
      }

      // PUNTO 1: solo la tecla A pausa / reanuda
      if (ev === EVENTS.DEC) {
        if (!this.paused) {
          this.paused = true;
          this.myTimer.stop();        // detiene el contador
        } else {
          this.paused = false;
          this.myTimer.start();       // reanuda desde donde se quedó
        }
      }
    }

    // El tick llega cada 1 segundo cuando no está pausado
    else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();  // reinicia el timer para el siguiente segundo
        }
      }
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
      this.paused = false;
    }
  };

  // ── Estado 3: Tiempo agotado ─────────────────────────────────
  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}

// ─────────────────────────────────────────────────────────────
//  P5.JS — setup y draw
// ─────────────────────────────────────────────────────────────
let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  // Cada estado tiene su propia función de dibujo
  renderer.set(temporizador.estado_config, () =>
    drawConfig(temporizador.configValue)
  );
  renderer.set(temporizador.estado_armed, () =>
    drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, temporizador.paused)
  );
  renderer.set(temporizador.estado_timeout, () => drawTimeout());

  // Botón HTML para conectar el micro:bit (solo aparece si no está conectado)
  connectButton = createButton("🔌 Conectar micro:bit");
  connectButton.position(width/2.25, height/5);
  connectButton.style("font-size", "14px");
  connectButton.style("padding", "8px 16px");
  connectButton.style("cursor", "pointer");
  connectButton.style("border-radius", "6px");
  connectButton.style("border", "none");
  connectButton.style("background", "#2600ff");
  connectButton.style("color", "white");
  connectButton.mousePressed(connectSerial);
}

function draw() {
  temporizador.upd6666666ate();
  renderer.get(temporizador.currentState)?.();

  // Indicador de conexión serial (esquina superior derecha)
  drawSerialStatus();
}

// ─────────────────────────────────────────────────────────────
//  FUNCIONES DE DIBUJO
// ─────────────────────────────────────────────────────────────
function drawConfig(val) {
  background("#000000");
  fill(255);
  textSize(UI.configText);
  text(val, width / 2, height / 2);
  textSize(UI.helpText);
  fill(200);
  text("A(−)   B(+)   S / agitar micro:bit (iniciar)", width / 2, height / 2 + 100);
}

function drawArmed(val, total, paused) {
  background(20, 20, 20);

  // Anillo de fondo (apagado)
  noFill();
  strokeWeight(UI.ringWeight);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, UI.dialSize);

  // Arco de progreso — azul si pausado, naranja si corriendo
  stroke(paused ? color(80, 130, 255) : color(255, 150, 0));
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, UI.dialSize, UI.dialSize, -HALF_PI, angle - HALF_PI);

  // Número central — sin pulso si pausado
  let pulse = paused ? 0 : sin(frameCount * 0.1) * 10;
  fill(paused ? color(80, 130, 255) : 255);
  noStroke();
  textSize(UI.bigText + pulse);
  text(val, width / 2, height / 2);

  // Texto de ayuda según estado
  textSize(UI.helpText);
  if (paused) {
    fill(80, 130, 255);
    text("⏸ PAUSADO  —  A para reanudar", width / 2, height / 2 + 180);
  } else {
    fill(160);
    text("A pausar  ·  secuencia A-B-A → configurar", width / 2, height / 2 + 180);
  }
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(UI.bigText);
  text("¡TIEMPO!", width / 2, height / 2);
  textSize(UI.helpText);
  fill(255, 200, 200);
  text("A para volver a configurar", width / 2, height / 2 + 80);
}

function drawSerialStatus() {
  noStroke();
  fill(serialConnected ? color(0, 220, 100) : color(120));
  ellipse(width - 20, 20, 12);
  textSize(12);
  fill(serialConnected ? color(0, 220, 100) : color(120));
  textAlign(RIGHT, CENTER);
  text(serialConnected ? "micro:bit conectado" : "sin micro:bit", width - 30, 20);
  textAlign(CENTER, CENTER); // restaura la alineación global
}

// ─────────────────────────────────────────────────────────────
//  TECLADO  (funciona en paralelo con el micro:bit)
// ─────────────────────────────────────────────────────────────
function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent(EVENTS.DEC);
  if (key === "b" || key === "B") temporizador.postEvent(EVENTS.INC);
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```


## Bitácora de aplicación 



## Bitácora de reflexión







