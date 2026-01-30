# Unidad 1

## Bitácora de proceso de aprendizaje

## ACTIVIDAD 01
### ¿Qué es un sistema físico interactivo?
Es un artefacto tangible que, a través de distintos estímulos, genera una acción o respuesta. Ese resultado puede ser digital o análogo. En esencia, se trata de una relación constante entre un activador (el usuario), el artefacto y el resultado que emerge de esa interacción.

### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Puedo aplicarlo desde experiencias museográficas hasta soluciones más ágiles y eficaces en distintos mercados. Es decir, puedo diseñar artefactos que apoyen una compra presencial como mupis interactivos o crear experiencias experimentales de simulación. El abanico de posibilidades es amplio y se adapta tanto a contextos comerciales como exploratorios.


## ACTIVIDAD 02
### ¿Qué es el diseño / arte generativo?

El diseño o arte generativo consiste en crear visuales a través de código. Estos pueden ser formas, tipografía, imágenes, entre otros. Se generan a partir de patrones o ciclos que combinan el azar con un orden previo, lo que permite producir múltiples composiciones a partir de un mismo sistema.

### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?**

Soy diseñadora gráfica y el enfoque que deseo es dar soluciones visuales a las compañías. Además, me interesa generar diseño sostenible. El diseño generativo permite justamente eso: mantenerse en el tiempo, dando mayor durabilidad a los sistemas visuales y fortaleciendo su recordación.

También me interesa el diseño biomimético, por lo que el arte generativo me permite tomar patrones de la naturaleza y hacerlos visibles a través del diseño.


## ACTIVIDAD 04
#### ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.

Porque is_pressed() consulta el estado actual del botón y devuelve un valor de forma continua mientras el botón está presionado, por ende al estar consultando constantemente, la acción se activa, en este caso cambiar de color. En cambio, was_pressed() no es un estado continuo sino un evento puntual: solo se activa una vez cuando ocurre la pulsación y luego se reinicia, por eso cambia el color cada tanto, porque tomaba eventos puntuales.

## ACTIVIDAD 05
#### Programa de p5.js.
```

let port;
let connectBtn;
let x;

function setup() {
    createCanvas(400, 400);
  
    x = width / 2;
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
```
#### Explica detalladamente cómo funciona el sistema físico interactivo que has creado.
En el programa del microbit mantenemos la consulta de los botones A y B `button_a.is_pressed`, estos reciben información continua y cuando el estado es verdadero, se envía un caracter por serial.
En el programa de p5.Js agregamos una variable global de posición `x` de nuestro sistema, que luegoo definimos en septup `x = width / 2;`, y así dibujar el circulo en el centro `ellipse(x, height / 2, 100, 100);`. En el draw el ciclo se ejecutara, si el puerto recibe un evento serial, que en este caso es el boton A o B.
```
        if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){ 
          x += 10;
        }
  
        else if(dataRx == 'B'){
          x-=10;
        }
```
El circulo se movera hacía la derecha`x += 10;` si se presiona el botón A y a la izquierda `x-=10;` si se presiona el botón B. 
Así pinta de nuevo el Canvas y el circulo. 

## ACTIVIDAD 06
Vas a repasar lo aprendido en esta unidad. Regresa a la actividad 4 y trata de explicar en tus propias palabras de la manera más detallada que puedas cómo funciona el sistema físico interactivo. Analiza cada parte del código y su función dentro del sistema. Si aún tienes dudas sobre alguna parte, aprovecha para aclararlas.
Explicación detallada del sistema de la actividad 4.

#### Programa de micro:bit.

Hacemos la lectura del microbit y le cargamos el siguieten código, que se esta comunicando a 115200 el puerto serial. Entonces le indicamos con ek while True `if button_a.is_pressed():` Que nos estará dando datos continuos del estado del botón, de esta forma nos dice si esta presionado o no. 
```
from microbit import *

uart.init(baudrate=115200)

while True:

    if button_a.is_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)

```
#### Programa de p5.js.

Ponemos nuestras variables globales que nos dan el puerto, la conexión al botón y su estaso, conectado o no.
```
  let port;
  let connectBtn;
  let connectionInitialized = false;
```
En el septup haremos el canvas y el fondo. Luego llamaremos el puerto y lo defimimos como comunicación serial. 
Llamaremos la conexión al botón para crear uno en el canvas que es el que da apertuda al estado del botón del microbit. Definimos su posición y su activación a través del la clic del mousee.
```
  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
  }
```

En nuestro bucle draw volvemos a crear el canvas con las mismas carateristicas. Entonces si el puerto esta abierto(true) y la conexión no ha inicializado el puerto se limpia y la coneción se inicia. 
Se habilita el puerto y si este me da un dato mayor a 0, leeremos el puerto en 1. Lo que nos dice que es igual al botón A y que activa el relleno en rojo. Si no entonces es igual a N que me deja el relleno verde. 
Luego damos forma al sistema al cual le cargamos las características anteriores, en el centro se dibuja un cuadro verde en la mitad del canvas.
Si el puerto no esta habilitado se coneta con el boton de la microbit, si no el boton se desconecta.
```
function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

```
Esta ultima función es para desconectar el microbit, si el puerto no esta abierto, ser abre y manda datos a la velocidad de 115200, y la conexión finaliza. Por ende se cierrra el puerto.
```
  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }

```
## Bitácora de aplicación 
## Bitácora de reflexión









