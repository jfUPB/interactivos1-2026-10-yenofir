# Unidad 6

## Bitácora de proceso de aprendizaje
### Actividad 1


**¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?**

Recibir el mensaje, me da la información pero no me dice cómo usarlo, entonces es como cuándo uno recibe un manual de instrucción para ejecutar algún dispositivo, hay elementos como un botón que activan esa información y es la que permite dar sincronía. 


**¿Por qué un sistema audiovisual puede necesitar `timestamp` además de los datos del evento?**

El timestamp es el que nos permite sincronizar lo mejor posible el sonido con las visuales. **Anticipa el momento o dice cuando tiene que mandar el evento para usar la misma base de tiempo en P5.js**


**¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?**

Usamos el Bridge, Node.js. Lo que hace es escuchar toda la información que sale del browser y las envía a las visuales, que están en el browser. 


<img width="1600" height="632" alt="WhatsApp Image 2026-04-08 at 3 03 44 PM" src="https://github.com/user-attachments/assets/f9996044-e38a-4f10-a24d-3bc025c8c0e6" />




<img width="1600" height="634" alt="image" src="https://github.com/user-attachments/assets/7e88fa69-dc50-48fe-addd-ce7c848f88fd" />


**Si Strudel fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?**

Con Strudel usamos el protocolo OSC


**¿Qué variables mínimas necesitarías extraer para poder construir una visualización útil?**

Qué sonido ocurrió, cuándo debería ocurrir, cuánto dura su ciclo rítmico, otros parámetros asociados al evento.

  **cycle:** Repetición
  
  **delta:** Duración de la animación
 
  **s:** Que nota es
  
  **bank:** Que banco es
  
  **timestamp:** Sincronización.


**¿Qué problema resuelve la cola de eventos?**

La latencia de la red, al enviar los datos, algunos pueden llegar a la vez por la misma latencia, entonces los eventos sonarían exactamente al mismo tiempo. Con la cola y el timestamp, p5.js sabe que evento suena primero, los ejecuta en el orden correcto aunque hayan llegado juntos.


**¿Por qué esta capa no pertenece al bridge sino al lado que interpreta el evento?**

Porque la función principal del bridge es enviar los datos lo más rápido posible, por ende la tarea del interprete también es decidir en que momento la activa.


**¿Qué papel cumple el Adapter en U4 y U5?**

En las unidades anteriores cada adapter nos permitía cambiar el protocolo para recibir y enviar mensajes, donde usamos desde parseo para el formato de ASCII o framing para el protocolo binario de acuerdo al protocolo, 


**¿Qué Adapter necesitas ahora para que los eventos de Strudel no entren “crudos” al sistema visual?**

Necesitamos un adapter que nos permita recibir el protocolo OSC para enviar los datos a P5.js


## Bitácora de aplicación 


## Bitácora de reflexión
