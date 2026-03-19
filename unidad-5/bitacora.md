# Unidad 5
## Bitácora de proceso de aprendizaje
## Actividad 1
**¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?**

El protocolo binario permite tener más ancho de banda. Quiere decir que si queremos envíar muchos datos la codificación en binario reduce la cantidad de bytes usados, por ende los procesos de transmisión de datos se vuelven más optimos.

La parte mala del protocolo: es necesario tener un enter para cerrar líneas de código. Ya que podría sin esto puede ser confuso donde termina o empiezan los datos. Tenemos que estar seguros de que byte es el que marca el inicio de la trama.

Esto se hace con checksum. Que es el calcula un valor total de la seguridad.

**¿Cómo se vería el paquete en hexadecimal?**

xValue=500, yValue=524, aState=True, bState=False

01 F4 = 500
02 0C = 524
01 = True 
00 = False 

HEX: 01 F4 02 0C 01 00

Es necesario tener un enter para cerrar líneas de código. Ya que podría sin esto puede ser confuso donde termina o empiezan los datos.

**¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización?** 

El código se cerraba con una caracter de enter(\n), delimita el conjunto de datos que llega, cual es el incio y cual es el cierre.

**¿Por qué en binario no podemos usar \n como delimitador?**

Por que este es usado en el protocolo ASCII, y este caracter se puede confundir con el cierre o el final, por ende debemos utilizar framing, que marca el incio y cierre de un conjunto de datos. Además creamos un que es el calcula un valor total que damos para asegurar seguridad, es decir que si el valor total se cumple el conjunto de datos llego de manera correcta.

## Bitácora de aplicación 


## Bitácora de reflexión
