# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01

1.¿Qué es un sistema físico interactivo?
Es una experiencia que se le ofrece a un ususario la cual fusiona métodos fisicos y digitales para que sea algo único.
2.¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Podría aplicar esto creando experiencias interactivas innovadoras que ayuden a resolver grandes problemáticas, para qqe asi el cambio y la mejora sean de una manera mas sencilla y dinámica para todos.

---

### Actividad 02
1.¿Qué es un sistema físico interactivo?
Es una forma de crear arte usando programas o algoritmos que hacen que las imágenes se generen casi solas. En vez de hacer todo a mano,solo definimos ciertas condiciones y el sistema produce diferentes resultados. Es una mezcla entre creatividad y tecnología, donde la lógica juega es importante.

2.¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

Puedo aplicar lo que aprendí sobre diseño generativo en mi perfil profesional usando estas herramientas para crear propuestas más originales, explorar varias ideas en poco tiempo y mejorar mi proceso creativo.

---

### Actividad 03
En esta actividad lo que sucede es: el micro bit nos enseña la imagen de un circulo que al presionar la letra **A**, se pinta de color **AMARILLO**, al presionar la letra **B**, se torna **ROJO** y si lo **sacudimos** se vuelve **VERDE**... Además tiene la opción de mandar señal e información al microbit físico mediante**(SEND LOVE)** la cual muestra una **CARITA FELÍZ** y seguido de esta un **CORAZÓN** pero el programa inicia con una **MARIPOSA.**
Código java script
``` js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    let sendBtn = createButton('Send Love');
    sendBtn.position(220, 300);
    sendBtn.mousePressed(sendBtnClick);
    fill('white');
    ellipse(width / 2, height / 2, 100, 100);
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('red');
        }
        else if(dataRx == 'B'){
            fill('yellow');
        }
        else{
            fill('green');
        }
        background(220);
        ellipse(width / 2, height / 2, 100, 100);
        fill('black');
        text(dataRx, width / 2, height / 2);
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

function sendBtnClick() {
    port.write('h');
}
```
Código python
```py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)
    if uart.any():
        data = uart.read(1)
        if data:
            if data[0] == ord('h'):
                display.show(Image.HEART)
                sleep(500)
                display.show(Image.HAPPY)
```

---

### Actividad 04
¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()?
porque la finalidad del ejercicio es que funcione mientras esté presionado y al utilizar was, eso indica que es una acción que se hizó y se dejó de hacer por lo tanto no tiene coherencia al momento de ejecutar el código y el resto del código está escrito para una acción que se está realizando en el momento.

---

## Bitácora de aplicación 
### Actividad 05
CODIGO JAVA SCRIPT
``` js
let port;
let connectBtn;
let x;

function setup() {
  createCanvas(400, 400);
  background(220);

  port = createSerial();
  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(80, 300);
  connectBtn.mousePressed(connectBtnClick);

  fill("white");
  ellipse(width / 2, height / 2, 100, 100);
}

function draw() {
  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);
    if (dataRx == "A") {
      x = 0;
    } else if (dataRx == "B") {
      x = width;
    }
    background(220);
    ellipse(x, height / 2, 100, 100);
  }

  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
  } else {
    port.close();
  }
}

```
CODIGO PHYTON
``` py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
```
¿Cómo funciona el sistema físico interactivo?

El sistema físico interactivo que creé conecta una **micro:bit** con un programa hecho en **p5.js**, para que algo que hago en el mundo real se refleje en la pantalla del computador.

Primero, la **micro:bit** funciona como un control. Cuando presiono el **botón A** o el **botón B**, la micro:bit envía una letra por el cable USB:
- El botón **A** envía la letra **“A”**
- El botón **B** envía la letra **“B”**

Esa información viaja por el **puerto serial**, que funciona como un puente de comunicación entre la micro:bit y el computador.

En el computador, el programa en **p5.js** está todo el tiempo escuchando si llega algún mensaje desde la micro:bit. Cuando recibe una letra, el programa la interpreta y decide qué hacer:

- Si llega la letra **“A”**, el círculo que aparece en la pantalla se mueve hacia la **izquierda**.
- Si llega la letra **“B”**, el círculo se mueve hacia la **derecha**.

El círculo se dibuja dentro de un **canvas**, que es como una hoja en blanco digital. Cada vez que presiono un botón, el programa borra la pantalla y vuelve a dibujar el círculo en una nueva posición, por eso parece que el círculo se mueve.

En resumen, este sistema es interactivo porque **una acción física** (presionar un botón) **genera una respuesta digital** (el movimiento del círculo). Esto demuestra cómo se pueden conectar dispositivos reales con programas para crear experiencias interactivas, como en videojuegos o controles digitales.


## Bitácora de reflexión
### Actividad 6
Explicación de actividad 4, lineas de código js.
En las primeras 3 líneas de código lo que sucede es que se crean las variables, en las lineas 5 y 6 lo que se hace es designar un espacio para el fondo del programa, la línea 8 es la que se conecta con el microbit mediante la biblioteca nueva que agregamos, las líneas 9,10 y 11 son las encargadas de creaar, dar utilidad y situar el botón, que nos permite conectar al microbit con nuestro código mediante un click, background(220) es la que se encarga de que nuestro lienzo esté limpio al ejecutar otra acción, las líneas 17,18 y 19 se encargan de analizar si la conección de codigo está abierta y limpia para empezar a enviar las señales, la línea 22 se encarga de detectar si el microbit tiene información esperando a ser enviada y procesada por la computadora, líneas 24 y 25 son las encargadas de asignar el color y leer la información enviada desde el microbit físico, en este caso ella inicia el cuadro con color verde como se estipula en la lineas 26 y 27, luego si recibe la señal de que se presionó el botón a ella inmediatamente cambia a color rojo como lo dice la línea 26 hasta que se suelte y vuelva a su color oriiginal, las líneas 31 y 32 se encargan de dibujar un circulo de 50x50 en toda la mitad del background que creamos anteriormente, de las líneas 34 a la 37 el código nos muestra la función del cuadro que al clickear conecta el programa con el microbit, como aparece  antes de estar conectado y después, y para finalizar de la línea 41 a la 46 si el micro:bit está desconectado, intenta abrir la comunicación usando el protocolo de MicroPython (a una velocidad de 115200 baudios) y reinicia la configuración; si ya está conectado, cierra la comunicación de forma segura.






