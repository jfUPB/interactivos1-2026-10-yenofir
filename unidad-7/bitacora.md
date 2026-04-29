# Unidad 7

## Bitácora de proceso de aprendizaje
## ACTIVIDAD 1

**¿Qué diferencia hay entre un evento musical y un mensaje de control?**

La diferencia sería que desde el evento musical no podemos controlar nosotros mismos la acción de cambiar, el mensaje de control permite modificar un parametro en especifico en tiempo real. Es decir con el evento músical programo acciones puntuales determinadas desde el Código porque este pasa en un momento especifico, con el mensaje de control puedo accionar un cambio en el estado que estemos en vivo, por ende nos permite tener más control sobre los objetos.


**¿Qué quiere decir que un parámetro del sistema sea persistente?**

Un estado persistente es el que se mantiene sin niguna modificación, por ejemplo, si me llega un rgb en rojo, este mismo seguira llegando porque no tiene ninguna modificación del estado, contrario a un evento que llega y se va.


**¿Qué partes del sistema de la unidad 6 permanecen intactas en este nuevo caso?**
Permanece el Bridge de Strudel, que corre de REPL Strudel a través del Web Socket en el localhost 8080, el cual envia los datos a p5.js a t


**Si Open Stage Control fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?**

El protocolo sería OSC sobre UDP ya que como servidor, recibe mensajes OSC (comandos de faders, botones, XY pads) desde clientes (tablets, navegadores web) y los reenvía a otras aplicaciones o dispositivos (DAWs, servidores de video, consolas de luces), de acuerdo con la necesidad. También el protocolo HTTP/WebSockets para servir la interfaz gráfica de usuario (GUI) personalizada al navegador del cliente.


**¿Qué parte de ese protocolo te interesa conservar y cuál te gustaría normalizar?**

Conservar es la dirección OSC (/rgb_1) y los argumentos numéricos (R, G, B), porque eso es la semántica del mensaje — te dice qué parámetro cambiar y a qué valor.

Normalizar es el formato de los argumentos, porque la librería osc puede enviártelos de dos formas distintas: como número crudo (0.3) o como objeto con metadatos ({ type: "f", value: 0.3 }). Eso es exactamente lo que hace la función normalizeArg en bridgeOSC.js — aplanarlo siempre a un número simple antes de pasarlo al cliente.


**¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?**

Por lo mencionado, Strudel alimenta una cola de eventos temporizados, es decir, suceden en momentos especificos, OSC, actualiza el estado del control del sistema, es decir esta cambiando con los estados en un parametro especifico, por ende no tiene una apertura o cierre, siempre esta leyendo. 
La diferencia es semántica: los mensajes de Strudel tienen timestamp y van a una cola temporizada. Los de Open Stage Control no tienen tiempo, son actualizaciones de estado inmediatas. Meterlos en la misma cola rompería la sincronización.


**¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?**

En este caso del panel RGB, las variables persistentes, serían las de los sonidos, declararlos como elementeos de color con un RGB, que luego se puedan cambiar en el OSC.


**¿Qué componentes de la arquitectura necesitas conservar obligatoriamente?**

El StrudelAdapter y su conexión al BaseAdapter, el bridgeServer.js como broadcaster, el bridgeClient en el navegador, y toda la cadena FSMTask → processQueue → drawRunning. 


**¿Qué nuevas estructuras de estado necesitas introducir para soportar control paramétrico?**

En el momento las variables persistentes son globales, podríamos hacer un objeto de estado de control. Así podríamos actualizarlo cada vez que llegue un mensaje de Open Stade Control. 
Y tener 2 entradas de mensajes que permitan leer Strudel y OSC a la vez y los pueda diferenciarlos para enrutarlos a las funciones diferentes en el sketch. 



## Bitácora de aplicación 


## Bitácora de reflexión
