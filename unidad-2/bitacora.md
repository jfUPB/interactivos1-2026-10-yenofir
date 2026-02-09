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

## Bitácora de aplicación 



## Bitácora de reflexión
