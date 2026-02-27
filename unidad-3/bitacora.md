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
## Bitácora de aplicación 



## Bitácora de reflexión




