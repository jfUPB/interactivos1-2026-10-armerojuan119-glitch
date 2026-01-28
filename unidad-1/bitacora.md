# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01

1.¿Qué es un sistema físico interactivo?
Es una experiencia que se le ofrece a un ususario la cual fusiona métodos fisicos y digitales para que sea algo único.

2.¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Podría aplicar esto creando experiencias interactivas innovadoras que ayuden a resolver grandes problemáticas, para qqe asi el cambio y la mejora sean de una manera mas sencilla y dinámica para todos.

### Actividad 02
1.¿Qué es un sistema físico interactivo?

Es una forma de crear arte usando programas o algoritmos que hacen que las imágenes se generen casi solas. En vez de hacer todo a mano,solo definimos ciertas condiciones y el sistema produce diferentes resultados. Es una mezcla entre creatividad y tecnología, donde la lógica juega es importante.

2.¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

Puedo aplicar lo que aprendí sobre diseño generativo en mi perfil profesional usando estas herramientas para crear propuestas más originales, explorar varias ideas en poco tiempo y mejorar mi proceso creativo.

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






