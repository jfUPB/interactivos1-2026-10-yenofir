# Unidad 2

## Bitácora de proceso de aprendizaje
## Actividad 01
### ¿Cuáles son los estados en el programa?
```estado_waitTimeout``` El estado donde el píxel está esperando que termine el temporizador para cambiar de brillo.

### ¿Cuáles son los eventos en el programa?

```ENTRY``` Su propósito es activar el estado.

```EXIT``` Abandona el estado.

```Timeout``` Es el que indica que ya termino el tiempo del temporizador.

### ¿Cuáles son las acciones en el programa?

**ENTRY**
```
if ev == "ENTRY":
            self.pixelState = 9
            display.set_pixel(self.x,self.y,self.pixelState)
            self.myTimer.start() 
```

**1.** Enciende el Pixel ```pixelState = 9```

**2.** Actualiza el display ```display.set_pixel(self.x,self.y,self.pixelState)```

**3.** Arranca el temporizador ```myTimer.start()```

**Timeout**
```
elif ev == "Timeout":
            if self.pixelState == 0:
                self.pixelState = 9
                self.myTimer.start()
            else:
                self.pixelState = 0
                self.myTimer.start()

            display.set_pixel(self.x,self.y,self.pixelState)
```

**1.** El brillo del pixel baja a 0 o sube a 9. 

**2.** Reinicia el mismo temporizador. 

**3.** Actualiza el display ```display.set_pixel(self.x,self.y,self.pixelState) ```

## Actividad 02
### Modificación Código Evento "A"
```
from microbit import*
import utime
from fsm import Timer, FSMTask, ENTRY

class Semaforo(FSMTask):
	def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
		super().__init__()
		self.x = _x
		self.y = _y
		self.timeInRed = _timeInRed
		self.timeInGreen = _timeInGreen
		self.timeInYellow = _timeInYellow
		self.myTimer = self.add_timer("Timeout",self.timeInRed)
		    self.transition_to(self.estado_waitInRed)
	
	def clear(self): #No es un estado, es una función utilitaria para apagar los leds.
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
	        self.transition_to(self.estado_waitInGreen)
	
	
	def estado_waitInGreen(self, ev):
	    if ev == "ENTRY":
	        self.clear()
	        display.set_pixel(self.x,self.y+2,9)
	        self.myTimer.start(self.timeInGreen)
	        
	    if ev == "Timeout":
	        display.set_pixel(self.x,self.y+2,0)
	        self.transition_to(self.estado_waitInYellow)
	
	    if ev == "A":
	        display.set_pixel(self.x,self.y+2,0)
	        self.transition_to(self.estado_waitInYellow)
	
	def estado_waitInYellow(self, ev):
	    if ev == "ENTRY":
	        self.clear()
	        display.set_pixel(self.x,self.y+1,9)
	        self.myTimer.start(self.timeInYellow)
	
	    if ev == "Timeout":
	        display.set_pixel(self.x,self.y+1,0)
	        self.transition_to(self.estado_waitInRed)

            semaforo1 = Semaforo(0,0,2000,1000,500)
            
            while True:
            #IMPUT PROCESIN
            
            if button_a.was_pressed(): semaforo1.post_event("A")
            semaforo1.update()
            utime.sleep_ms(20)

```
### Máquina de estados
<img width="532" height="605" alt="image" src="https://github.com/user-attachments/assets/a6c533c7-c517-42bd-b141-40b9df06c9b4" />


Implementación plantUML 
```
@startuml
title Semaforo - UML State Machine

[*] --> WaitInRed : Semaforo() (constructor)

WaitInRed : entry /\n  clear()\n  display.set_pixel(x,y,9)\n  myTimer.start(timeInRed = 2000ms)

WaitInRed --> WaitInGreen : Timeout /\n  display.set_pixel(x,y,0)

WaitInGreen : entry /\n  clear()\n  display.set_pixel(x,y+2,9)\n  myTimer.start(timeInGreen = 1000ms)

WaitInGreen --> WaitInYellow : Timeout /\n  display.set_pixel(x,y+2,0)

WaitInYellow : entry /\n  clear()\n  display.set_pixel(x,y+1,9)\n  myTimer.start(timeInYellow = 500ms)

WaitInYellow --> WaitInRed : Timeout /\n  display.set_pixel(x,y+1,0)

@enduml

```

## ACTIVIDAD 3

- ¿Cómo es posible estructurar una aplicación usando una máquina de estados para poder atender varios eventos de manera concurrente?

Tenemos unos objetos de clase Timer que se actualizan constantemente, sin pausar el programa. En lugar de usar sleep() directamente en los estados, se usan temporizadores que se chequean en cada ciclo. Esto permite que el programa siga corriendo y detecte eventos externos (como botones) mientras "espera".
```
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
````

Tenemos la `class Game:` que es nuetra máquina de estados.
La máquina de estados, cuenta con 3 estados `waitInHeart` `waitInPacman` `waitInGhost` que al presionar el botón A, cambia. Inicializamos con una la lista de temporizadores y un temporizador reutilizable, tanto la duración de cada uno de los estados

````
class Game:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.timeInHeart = 2500
        self.timeInPacman = 1000
        self.timeInGhost = 2000
        self.myTimer = self.createTimer("Timeout",self.timeInHeart)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInHeart)

````
En este mismo objeto encontramos `createTimer`para abstraer temporizadores que son reutilizables, `post_event`  luego tenemos una llamado a la lista de eventos pendientes, `update(self)` que es la actualización de todos los timers,  `transicion_a` son las acciones de salida y entrada de diferenctes eventos de acuerdo con el estado actual.
````
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
````
Deinimos los estados que tienen unos eventeos: sáldia ENTRY, un Timeup que llama el temporizador para hacer la transición al otro esado. y la presión del botón A que activa la transición a otro estado.
````
def estado_waitInHeart(self, ev):
        if ev == "ENTRY":
            display.show(Image.HEART)
            self.myTimer.start(self.timeInHeart)
        if ev == "Timeout":
            self.transicion_a(self.estado_waitInPacman)
        if ev == "A":
            self.transicion_a(self.estado_waitInGhost)


    def estado_waitInPacman(self, ev):
        if ev == "ENTRY":
            display.show(Image.PACMAN)
            self.myTimer.start(self.timeInPacman)
        if ev == "Timeout":
            self.transicion_a(self.estado_waitInGhost)
        if ev == "A":
            self.transicion_a(self.estado_waitInHeart)

    def estado_waitInGhost(self, ev):
        if ev == "ENTRY":
            display.show(Image.GHOST)
            self.myTimer.start(self.timeInGhost)
        if ev == "Timeout":
            self.transicion_a(self.estado_waitInHeart)
        if ev == "A":
            self.transicion_a(self.estado_waitInPacman)
  ````
Para el cierre, tenemos el bucle `while True` que verifica si se presionó el botón A y postea el evento "A" si es así, llama a game.update() para procesar todo y duerme 20 ms (utime.sleep_ms(20)) para evitar consumir CPU innecesariamente, pero lo suficientemente corto para responder rápidamente a eventos.
  ````
while True:
    if button_a.was_pressed():
        game.post_event("A")
    game.update()
    utime.sleep_ms(20)
  ````

- ¿Cómo haces para probar que el programa está correcto?

Observando si los estados y los eventos funcionan bien el microbit, compararando a su vez con nuestra gráfica UML que muestra los eventos y estados esperados.

## Bitácora de aplicación 



## Bitácora de reflexión



