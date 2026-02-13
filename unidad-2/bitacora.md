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
## ACTIVIDAD 4

Código en Micropython
```
from microbit import *
import utime
import audio

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25


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


class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        
        # Variables de estado del temporizador
        self.seconds = 20 # valor inicial
        self.MIN_SECONDS = 15
        self.MAX_SECONDS = 25
        
        # Personalizas el nombre del evento y la duración
        self.myTimer = self.createTimer("Timeout",1000) # 1 segundo = 1000 ms
        
        # Estado inicial
        self.estado_actual = None
        self.transicion_a(self.estado_estadoconfig)

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


    # ESTADO: CONFIGURACIÓN 
    def estado_estadoconfig(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.seconds])
            # No iniciamos timer aquí → está desarmado
    
        if ev == "A":
            if self.seconds < self.MAX_SECONDS:
                self.seconds += 1
                display.show(FILL[self.seconds])
    
        if ev == "B":
            if self.seconds > self.MIN_SECONDS:
                self.seconds -= 1
                display.show(FILL[self.seconds])
    
        if ev == "S":
            self.transicion_a(self.estado_conteo)
    

    
    # ESTADO: CUENTA REGRESIVA 
    def estado_conteo(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.seconds])
            self.myTimer.start(1000)          # comienza la cuenta cada 1 segundo
    
        if ev == "Timeout":
            self.seconds -= 1
            display.show(FILL[self.seconds])
    
            if self.seconds > 0:
                self.myTimer.start(1000)      # reiniciamos para el siguiente segundo
            else:
                self.transicion_a(self.estado_alarma)
    
        if ev == "S":
            self.transicion_a(self.estado_conteo)
    
 
    
    # ESTADO: ALARMA (tiempo terminado) 
    def estado_alarma(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            
            # Sonido de alarma (ejemplo simple - puedes ajustar frecuencia/duración)
            effect_up = audio.SoundEffect(
                    freq_start=300,
                    freq_end=900,
                    duration=300,
                    vol_start=200,
                    vol_end=200,
                    waveform=audio.SoundEffect.WAVEFORM_SQUARE,
                    fx=audio.SoundEffect.FX_NONE,
                    shape=audio.SoundEffect.SHAPE_LINEAR
                )
            
            audio.play(effect_up, wait=True)   # wait=True espera a que termine
                
                # Pequeña pausa silenciosa
            utime.sleep_ms(150)
                
                # Sonido bajando (opcional, para más dramatismo)
            effect_down = audio.SoundEffect(
                    freq_start=900,
                    freq_end=300,
                    duration=300,
                    vol_start=200,
                    vol_end=200,
                    waveform=audio.SoundEffect.WAVEFORM_SQUARE
                )
            audio.play(effect_down, wait=True)
                
            utime.sleep_ms(250)
    
        if ev == "A":
            # Reiniciamos a configuración con valor por defecto
            self.seconds = 20
            self.transicion_a(self.estado_estadoconfig)
    

task = Task()

while True:
    # Aquí generas los eventos de los botones y el gesto
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```

<img width="760" height="519" alt="image" src="https://github.com/user-attachments/assets/59c24c13-891e-47a6-8f8b-67e840c752e6" />


Fuente del diagrama anterior:

```
@startuml
title Temporizador - UML State Machine

[*] --> Configuracion : Task() (constructor)

Configuracion : entry /\n display.show(FILL[seconds])

Configuracion --> Configuracion : A / +1 segundo (≤25)

Configuracion --> Configuracion : B / -1 segundo (≥15)

Configuracion --> Conteo : S /

Conteo : entry /\n display.show(FILL[seconds])\n myTimer.start(1000)
Conteo --> Conteo : S / Timeout /\n seconds--\n if seconds > 0: myTimer.start(1000)
Conteo --> Alarma : Timeout / [seconds ≤ 0]


Alarma : entry /\n display.show(Image.SKULL)\n // sonido alarma
Alarma --> Configuracion : A /\n seconds = 20

@enduml

```

## Bitácora de reflexión





