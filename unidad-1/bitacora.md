# Unidad 1

## Bitácora de proceso de aprendizaje
## ACTIVIDAD 01
### ¿Qué es un sistema físico interactivo?
Es un artefacto tangible que, a través de distintos estímulos, genera una acción o respuesta. Ese resultado puede ser digital o análogo. En esencia, se trata de una relación constante entre un activador (el usuario), el artefacto y el resultado que emerge de esa interacción.

### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Puedo aplicarlo desde experiencias museográficas hasta soluciones más ágiles y eficaces en distintos mercados. Es decir, puedo diseñar artefactos que apoyen una compra presencial como mupis interactivos o crear experiencias experimentales de simulación. El abanico de posibilidades es amplio y se adapta tanto a contextos comerciales como exploratorios.

## ACTIVIDAD 02



## ACTIVIDAD 04
####¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.

Porque is_pressed() consulta el estado actual del botón y devuelve un valor de forma continua mientras el botón está presionado, por ende al estar consultando constantemente, la acción se activa, en este caso cambiar de color. En cambio, was_pressed() no es un estado continuo sino un evento puntual: solo se activa una vez cuando ocurre la pulsación y luego se reinicia, por eso cambia el color cada tanto, porque tomaba eventos puntuales.

## ACTIVIDAD 05
#### Programa de p5.js.
```

let port;
let connectBtn;
let x;

function setup() {
    createCanvas(400, 400);
  
    x = width / 2;78
    background(80);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(width/3,300);
    connectBtn.mousePressed(connectBtnClick);
    fill('rgb(0,16,255)');
    ellipse(x, height / 2, 100, 100);
    
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){ 
          x += 10;
        }
  
        else if(dataRx == 'B'){
          x-=10;
        }
        background(80);
        ellipse(x, height / 2, 100, 100);
        fill('rgb(0,16,255)')

    }

    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}
```
#### Programa de micro:bit.
```
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.was_pressed():
        uart.write('B')
        sleep(500)
## Bitácora de aplicación 

```
####Explica detalladamente cómo funciona el sistema físico interactivo que has creado.

## Bitácora de reflexión




