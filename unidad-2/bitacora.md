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
## Bitácora de aplicación 



## Bitácora de reflexión


