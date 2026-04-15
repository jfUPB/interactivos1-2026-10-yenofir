# Unidad 6

## Bitácora de proceso de aprendizaje
### Actividad 1

• ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?

Recibir el mensaje, me da la información pero no me dice cómo usarlo, entonces es como cuándo uno recibe un manual de instrucción para ejecutar algún dispositivo, hay elementos como un botón que activan esa información y es la que permite dar sincronía. 


• ¿Por qué un sistema audiovisual puede necesitar `timestamp` además de los datos del evento?

El timestamp es el que nos permite sincronizar lo mejor posible el sonido con las visuales. **Anticipa el momento o dice cuando tiene que mandar el evento para usar la misma base de tiempo en P5.js**


• ¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?

Usamos el Bridge, Node.js. Lo que hace es escuchar toda la información que sale del browser y las envía a las visuales, que están en el browser. 

![WhatsApp Image 2026-04-08 at 3.03.44 PM.jpeg](attachment:e9bbec78-1b01-48d2-9735-7d24c5fb9987:WhatsApp_Image_2026-04-08_at_3.03.44_PM.jpeg)


**• Si Strudel fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?**

Con Strudel usamos el protocolo OSC

**• ¿Qué variables mínimas necesitarías extraer para poder construir una visualización útil?**

Qué sonido ocurrió, cuándo debería ocurrir, cuánto dura su ciclo rítmico, otros parámetros asociados al evento.
  **cycle:** Repetición
  
  **delta:** Duración de la animación
 
  **s:** Que nota es
  
  **bank:** Que banco es
  
  **timestamp:** Sincronización.


**• ¿Qué problema resuelve la cola de eventos?**
La latencia de la red, al enviar los datos, algunos pueden llegar a la vez por la misma latencia, entonces los eventos sonarían exactamente al mismo tiempo. Con la cola y el timestamp, p5.js sabe que evento suena primero, los ejecuta en el orden correcto aunque hayan llegado juntos.


**• ¿Por qué esta capa no pertenece al bridge sino al lado que interpreta el evento?**

Porque la función principal del bridge es enviar los datos lo más rápido posible, por ende la tarea del interprete también es decidir en que momento la activa.

**• ¿Qué papel cumple el Adapter en U4 y U5?**
**• ¿Qué Adapter necesitas ahora para que los eventos de Strudel no entren “crudos” al sistema visual?**

## Bitácora de aplicación 


## Bitácora de reflexión
