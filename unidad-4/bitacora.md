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
### Contexto del problema

Me pidieron integrar un micro:bit con un firmware fijo (protocolo nuevo) a una pieza de arte generativo en p5.js, usando una arquitectura de software ya existente que no podía romper. El reto principal fue entender el flujo de datos completo antes de escribir una sola línea de código.

---

### Análisis de la arquitectura existente

Antes de tocar cualquier archivo, leí todo el código del repositorio para entender cómo fluyen los datos. Identifiqué tres capas:

1. **Backend (Node.js)** — `bridgeServer.js` orquesta todo. Crea un adaptador según el argumento `--device` y expone los datos por WebSocket.
2. **Adaptadores (`/adapters`)** — cada uno hereda de `BaseAdapter` y debe emitir `{ x, y, btnA, btnB }` mediante `this.onData?.()`. Este es el **contrato**.
3. **Frontend (`sketch.js`)** — recibe los datos vía `BridgeClient`, los inyecta en una máquina de estados (`PainterTask`) y los renderiza con p5.js.

La clave que entendí: **el servidor no sabe qué hardware hay**. Solo espera el objeto normalizado del adaptador. Si mi adaptador cumple el contrato, el resto funciona sin modificaciones.

---

### Protocolo del nuevo hardware

El dispositivo emite tramas ASCII a 115200 baudios a 10 Hz con este formato:

`$T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n`



**Ejemplo válido:**
`$T:45020|X:-245|Y:12|A:1|B:0|CHK:258`



**Cálculo del checksum:**
CHK = |X| + |Y| + |A| + |B|
= |-245| + |12| + |1| + |0|
= 245 + 12 + 1 + 0 = 258 ✓



Si el checksum no coincide, la trama está corrupta y debo descartarla silenciosamente, pero registrar un `console.warn`.

---

### Firmware del micro:bit

Este es el código Python que cargué en el micro:bit para generar el protocolo nuevo:

```python
from microbit import *

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    t = running_time()
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    a = 1 if button_a.is_pressed() else 0
    b = 1 if button_b.is_pressed() else 0
    chk = abs(x) + abs(y) + abs(a) + abs(b)
    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, x, y, a, b, chk)
    uart.write(data)
    sleep(100)
```

Decisiones que tomé:

- Uso 1 if button_a.is_pressed() else 0 en lugar de True/False porque el parser espera enteros para calcular el checksum correctamente.
- Calculo chk en el micro:bit con la misma fórmula que valida el adaptador, para que coincidan siempre.
- running_time() da los milisegundos desde el arranque, que corresponde al campo T del protocolo.
- Adaptador creado: Microbit2ASCIIAdapter.js
- Creé el archivo /adapters/Microbit2ASCIIAdapter.js. No modifiqué ningún archivo existente.

Responsabilidades del adaptador:

- Abrir el puerto serial y acumular bytes en un buffer.
- Extraer líneas completas (separadas por \n).
- Parsear cada línea con el formato $T|X|Y|A|B|CHK.
- Validar el checksum — si falla, descartar la trama y emitir console.warn.
- Emitir this.onData?.({ x, y, btnA, btnB }) — el contrato con el servidor.

Lógica del parser:

```js
function parseLine(line) {
  if (!line.startsWith("$")) throw new ParseError(...);
  
  const parts = line.slice(1).split("|"); // quita el $ y separa por |
  // construye un diccionario { T, X, Y, A, B, CHK }
  
  const x = Number(fields.X), y = Number(fields.Y);
  const a = Number(fields.A), b = Number(fields.B);
  const chk = Number(fields.CHK);
  
  // Validación de rango del acelerómetro
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError(...);
  
  // Validación del checksum
  const computed = Math.abs(x) + Math.abs(y) + Math.abs(a) + Math.abs(b);
  if (computed !== chk) throw new ParseError("Checksum mismatch");
  
  return { x: x | 0, y: y | 0, btnA: a === 1, btnB: b === 1 };
}
```

**Decisión importante:**  diferencié los errores de checksum de los demás errores de parseo. Las tramas corruptas siempre generan console.warn (como exige el enunciado), mientras que otros errores de formato solo se muestran si verbose está activo.

**Integración en bridgeServer.js**
El servidor ya tenía soporte para --device microbit2 y hacía require('./adapters/Microbit2ASCIIAdapter'). Solo necesitaba que el archivo existiera con la clase correcta exportada. No modifiqué bridgeServer.js.

**Integración en sketch.js**
La pieza de arte generativo ya estaba adaptada a la arquitectura FSM. Identifiqué la equivalencia entre el prototipo original y la solución final:

| Prototipo original (p5.js)              | Solución final (hardware)                          |
|----------------------------------------|--------------------------------------------------|
| `mouseIsPressed && mouseButton == LEFT`| `mb.btnA === true`                               |
| `mouseY` (eje Y canvas)                | `mb.y` (acelerómetro Y mapeado a `[0, height]`)  |
| `mouseX` (eje X canvas)                | `mb.x` (acelerómetro X mapeado a `[0, width]`)   |
| `keyIsPressed`                         | `mb.btnB === true`                               |

El escalado matemático ocurre en updateLogic():

```js
updateLogic(data) {
  const gain = 4; // amplifica movimiento cuando el micro:bit está casi plano
  const gx = constrain(data.x * gain, -2048, 2047);
  const gy = constrain(data.y * gain, -2048, 2047);
  this.rxData.x = map(gx, -2048, 2047, 0, width);
  this.rxData.y = map(gy, -2048, 2047, 0, height);
}
```
El renderizado en draw() solo lee painter.rxData — no procesa datos, solo dibuja.

**Cómo ejecutar el sistema**

- Con simulador (sin hardware):
`node bridgeServer.js --device sim`

- Con micro:bit (protocolo viejo x,y,A,B):
`node bridgeServer.js --device microbit --serialPort COM10`

- Con micro:bit (protocolo nuevo $T|X|Y|A|B|CHK):
`node bridgeServer.js --device microbit2 --serialPort COM10`

Luego abro index.html con Go Live en VS Code.

### Código del adapter
**MicrobitV2Adapter.js**

```Js
from microbit import *

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    t = running_time()
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    a = 1 if button_a.is_pressed() else 0
    b = 1 if button_b.is_pressed() else 0
    chk = abs(x) + abs(y) + abs(a) + abs(b)
    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, x, y, a, b, chk)
    uart.write(data)
    sleep(100)


```
### Cambios en el sketch

```Js

```


## Bitácora de reflexión
