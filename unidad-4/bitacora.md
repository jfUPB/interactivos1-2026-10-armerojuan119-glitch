# Unidad 4

## Bitácora de proceso de aprendizaje

### 1. Descripción del caso de estudio

El objetivo de este proyecto es **integrar un dispositivo físico (micro:bit)** con una pieza de **arte generativo en p5.js**.

El sistema debe permitir que **los datos del hardware controlen visualmente el arte generativo**.

El reto principal es que:

- No se puede modificar el firmware del hardware.
- No se puede modificar la arquitectura del sistema existente.
- Solo se puede adaptar el hardware mediante un **Adapter**.

Esto simula un escenario real de desarrollo donde debemos **integrar hardware nuevo en un sistema existente sin romperlo**.

---

### 2. Hardware del caso de estudio

El dispositivo basado en **micro:bit** envía información por **UART (serial)** con la siguiente configuración:

- **Baudios:** 115200  
- **Frecuencia:** 10 Hz (cada 100 ms)

El firmware del micro:bit envía los datos en formato:
x,y,a,b

Ejemplo:
-123,45,1,0


Donde:

| Variable | Descripción |
|--------|-------------|
| x | aceleración eje X |
| y | aceleración eje Y |
| a | botón A presionado |
| b | botón B presionado |

### Código del firmware

```python
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()

    data = "{},{},{},{}\n".format(xValue, yValue, aState, bState)
    uart.write(data)

    sleep(100)
```
Esto permite que el micro:bit actúe como sensor de movimiento y controlador físico.

### 3. Arquitectura del sistema

El sistema utiliza una arquitectura desacoplada en tres capas.

Flujo de datos
```
Hardware (micro:bit)
        ↓
Puerto Serial
        ↓
Adapter (Node.js)
        ↓
JSON estandarizado
        ↓
WebSocket
        ↓
bridgeClient.js
        ↓
FSM (PainterTask)
        ↓
Renderizado en p5.js
```
Cada capa tiene una responsabilidad específica, lo que permite modificar componentes sin afectar todo el sistema.

### 4. Capa Backend – Adapter

El servidor Node.js no conoce el hardware directamente.

Solo espera recibir datos en este formato:
```Js
{
  x: number,
  y: number,
  btnA: boolean,
  btnB: boolean
}
```
Por esta razón se utiliza el patrón Adapter.

El Adapter se encarga de:

- Leer datos crudos del puerto serial.

- Interpretar el protocolo del hardware.

- Convertir los datos a un formato estándar.

- Emitir el objeto JSON esperado por el sistema.

### Ejemplo de salida esperada
```Js
this.onData?.({
  x: -245,
  y: 12,
  btnA: true,
  btnB: false
});
```
Esto permite que el servidor funcione igual sin importar el hardware conectado.

## Bitácora de aplicación 
### Código del adapter
**MicrobitV2Adapter.js**
```Js
import BaseAdapter from "./BaseAdapter.js";

export default class MicrobitV2Adapter extends BaseAdapter {

  onLine(line) {

    // verificar inicio de trama
    if (!line.startsWith("$")) return;

    // quitar $
    line = line.replace("$", "").trim();

    // separar campos
    const parts = line.split("|");

    let data = {};

    for (let p of parts) {
      let [key, value] = p.split(":");
      data[key] = value;
    }

    let x = parseInt(data.X);
    let y = parseInt(data.Y);
    let a = parseInt(data.A);
    let b = parseInt(data.B);
    let chk = parseInt(data.CHK);

    // calcular checksum
    let calc = Math.abs(x) + Math.abs(y) + a + b;

    // validar trama
    if (calc !== chk) {
      console.warn("Trama corrupta recibida");
      return;
    }

    // enviar datos al servidor
    this.onData?.({
      x: x,
      y: y,
      btnA: a === 1,
      btnB: b === 1
    });

  }

}
```
### Cambios en el sketch
**UpdateLogic**
```Js
updateLogic(data) {

    this.rxData.ready = true;

    // mapear acelerómetro al canvas
    this.rxData.x = map(data.x, -2048, 2047, 0, width);
    this.rxData.y = map(data.y, -2048, 2047, 0, height);

    this.rxData.btnA = data.btnA;
    this.rxData.btnB = data.btnB;

}
```
**DrawRunning**
```Js
function drawRunning() {

    let mb = painter.rxData;

    if (!mb.ready) return;

    push();
    translate(width / 2, height / 2);

    // resolución del círculo (usa eje Y)
    let circleResolution = int(map(mb.y + 100, 0, height, 2, 10));

    // radio (usa eje X)
    let radius = mb.x - width / 2;

    let angle = TAU / circleResolution;

    // botón B activa relleno
    if (mb.btnB) {
        fill(34, 45, 122, 50);
    } else {
        noFill();
    }

    stroke(0, 25);

    beginShape();

    for (let i = 0; i <= circleResolution; i++) {

        let x = cos(angle * i) * radius;
        let y = sin(angle * i) * radius;

        vertex(x, y);

    }

    endShape();

    pop();
}
```


## Bitácora de reflexión
