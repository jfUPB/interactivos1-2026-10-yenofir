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

## Bitácora de aplicación 



## Bitácora de reflexión


