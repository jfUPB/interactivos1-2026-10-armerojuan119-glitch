# Unidad 7

## Bitácora de proceso de aprendizaje

###  Actividad 1
- ¿Qué diferencia hay entre un evento musical y un mensaje de control?

Un evento musical es algo que pasa una vez y ya. Strudel me manda un golpe de bombo con un timestamp, ese evento entra a la cola, se dibuja cuando le toca, y desaparece. Un mensaje de control no "pasa" — más bien le dice al sistema "a partir de ahora el color es este". No se consume, no se borra, simplemente queda ahí hasta que yo lo cambie de nuevo.

- ¿Qué quiere decir que un parámetro del sistema sea persistente?

Que sigue vivo aunque no llegue nada nuevo. En el sketch están tr909bdRed, tr909bdGreen y tr909bdBlue declaradas arriba del todo. Cada frame que se dibuja el sketch las usa para colorear el bombo. Si no llega ningún mensaje OSC nuevo, esas variables no cambian — el color que elegí sigue siendo el color. Eso es persistencia: el valor se queda hasta que decido reemplazarlo.

- ¿Qué partes del sistema de la unidad 6 permanecen intactas?

Todo lo que tenía que ver con Strudel quedó igual: la conexión al puerto 8081, la cola de eventos, el ordenamiento por timestamp, las animaciones y las funciones de dibujo. Lo único que se agregó fue una segunda conexión WebSocket al puerto 8082 y tres variables nuevas. No toqué nada de lo que ya funcionaba.

#### Paso 1
- Si Open Stage Control fuera "el dispositivo" de esta unidad, ¿cuál sería su protocolo?

OSC sobre UDP. Cuando muevo el widget RGB en Open Stage Control, eso sale como un paquete OSC por UDP hacia el puerto 9000. Es parecido a como en unidades anteriores había un hardware mandando datos por Serial — acá el "hardware" es la interfaz de control y el cable es la red local.

- ¿Qué parte de ese protocolo me interesa conservar y cuál normalizar?

La dirección /rgb_1 me interesa conservarla porque es el nombre del parámetro, es lo que le dice al sistema qué está cambiando. Lo que sí normalizo son los argumentos: OSC puede mandar los valores de dos formas distintas y en bridgeOSC.js hay una función normalizeArg que aplana eso antes de enviarlo por WebSocket. Así el sketch siempre recibe números simples y no tiene que preocuparse por el formato original.

#### Paso 2
- ¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?

Porque si meto el mensaje OSC en la cola de eventos, se va a consumir una sola vez y el color va a volver a lo que era. La gracia del control paramétrico es justamente que no se consume — que se queda. Y al revés: si tratara un evento de Strudel como control persistente, perdería el timing y el visual dejaría de estar sincronizado con el audio. Son dos cosas distintas y hay que tratarlas distinto.

- ¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?

Todo lo que describe cómo se ve el sistema en este momento, no lo que acaba de pasar. El color del bombo es el ejemplo obvio de esta unidad. Pero podría ser también la opacidad del fondo, el tamaño base de las figuras, o qué forma le toca a cada sonido. Cualquier cosa que quiero que "siga siendo así" hasta que yo decida cambiarlo.

#### Paso 3
- ¿Qué componentes de la arquitectura necesito conservar obligatoriamente?

El adapter que normaliza los datos antes de retransmitirlos, el WebSocket como canal de transporte hacia el browser, la cola de eventos con su ordenamiento por timestamp, y el draw loop de p5.js. Sin cualquiera de esos, algo se rompe. Lo interesante es que bridgeOSC.js hace exactamente el mismo rol que el bridge de Strudel — recibe un protocolo nativo y lo convierte a JSON por WebSocket — pero para una fuente completamente distinta.

- ¿Qué nuevas estructuras de estado necesito introducir para soportar control paramétrico?

En este caso con tres variables globales alcanzó. Pero si el sistema creciera, tendría que separar el estado de control en un objeto propio, algo como controlState, para no tener variables sueltas por todo el sketch. La clave es que ese objeto se lee en cada frame pero solo se escribe cuando llega un mensaje OSC no se mezcla con los eventos musicales.

## Bitácora de aplicación 


## Bitácora de reflexión
