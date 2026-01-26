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

## Bitácora de aplicación 
### Actividad 05

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



## Bitácora de reflexión


