# Unidad 5
## Bitácora de proceso de aprendizaje


### Ventajas y desventajas del formato binario
**Ventajas**

**1. Menor tamaño de datos**

El protocolo ASCII puede ocupar muchos bytes dependiendo del número.
Por ejemplo:
```
"500,524,True,False\n" ≈ 19 bytes
```
Mientras que el protocolo binario siempre ocupa 6 bytes.

Esto reduce:

- uso de ancho de banda

- tiempo de transmisión

- carga de procesamiento

**2. Tamaño fijo del paquete**

Cada paquete siempre tiene el mismo tamaño, lo que facilita:

- leer datos del puerto serial

- interpretar correctamente los paquetes

**3. Mayor eficiencia**

Los sistemas embebidos como micro:bit tienen recursos limitados.
Enviar datos binarios evita conversiones innecesarias de texto.

**Desventajas**

**1. No es legible para humanos**

Mientras que ASCII se puede leer fácilmente:
```
500,524,True,False
```
El formato binario se ve como bytes incomprensibles.

**2. Más difícil de depurar**

Para entender los datos se necesitan herramientas como:

- analizadores hexadecimales

- scripts de decodificación

**3. Dependencia del formato**

Si el receptor interpreta mal el formato `(>2h2B)`, los datos se leerán incorrectamente.

---

### Representación del paquete en hexadecimal

Valores dados:
```
xValue = 500
yValue = 524
aState = True
bState = False
```
Formato:
```
>2h2B
```

---

**Convertir cada valor**

xValue = 500

En hexadecimal:
```
500 = 01F4
```
Como `h` usa 2 bytes:
```
01 F4
```

---

yValue = 524

En hexadecimal:
```
524 = 020C
```
En 2 bytes:
```
02 0C
```
---

aState = True

Se convierte a entero:
```
True → 1
```

Como `B` usa 1 byte:

```
01
```
---

bState = False
```
False → 0
```
En 1 byte:
```
00
```

---

**Hexadecimal Final**

Orden según el formato:

`xValue | yValue | aState | bState`

Resultado:
```
01 F4 02 0C 01 00
```
## Bitácora de aplicación 

### Fase 1: Análisis del repositorio existente

Lo primero que hice fue leer todos los archivos del repositorio antes de escribir una sola línea de código. Identifiqué que el sistema ya tenía:

- `BaseAdapter.js` — contrato base que todos los adapters deben cumplir  
- `MicrobitASCIIAdapter.js` — adapter funcional para protocolo CSV  
- `SimAdapter.js` — simulador sin hardware  
- `bridgeServer.js` — servidor Node.js con WebSocket  
- `sketch.js` — pieza de arte generativo con arquitectura FSM ya implementada  

Descubrí que `Microbit2ASCIIAdapter.js` existía pero solo tenía una función `parseCsvLine` suelta, sin clase ni `module.exports`. Era un archivo incompleto.

También encontré que `bridgeServer.js` tenía un bloque `"microbitauto"` que referenciaba `MicrobitAutoAdapter`, una clase que no existía ni estaba importada — código muerto que crashearía el servidor si alguien lo usaba. Lo eliminé.

---

#### Fase 2: Construcción de `Microbit2ASCIIAdapter.js`

Construí el adapter completo heredando de `BaseAdapter`, con la lógica de parseo del protocolo `$T|X|Y|A|B|CHK`.

**Error encontrado:** al implementar `parseLine()`, inicialmente solo manejé el protocolo `$`. Cuando probé con el firmware CSV (`"{},{},{},{}\n"`), `--device microbit2` no recibía ningún dato porque el parser rechazaba todas las líneas que no empezaban con `$`.

**Solución:** agregué el fallback al protocolo CSV viejo dentro de `parseLine()`, igual a como lo tenía `MicrobitASCIIAdapter`. Así ambos adapters ASCII funcionan con el mismo firmware.

---

### Fase 3: Construcción de `MicrobitBinaryAdapter.js`

Implementé el parser de protocolo binario con framing: `[0xAA] [X high] [X low] [Y high] [Y low] [btnA] [btnB] [CHK]`

La lógica de sincronización:
1. Buscar `0xAA` al inicio del buffer  
2. Si hay 8 bytes disponibles, extraer el paquete  
3. Calcular `(sum bytes 1..6) % 256` y comparar con byte 7  
4. Si coincide → emitir datos. Si no → descartar solo el `0xAA` y reintentar  

**Error encontrado — firmware incorrecto:** al probar por primera vez, el adapter no recibía ningún dato válido. El micro:bit tenía cargado este firmware:

```python
data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
uart.write(data)
```
Ese firmware envía 6 bytes sin header ni checksum. El adapter nunca encontraba 0xAA porque no existía en la trama, entonces descartaba todo silenciosamente.

Solución: flashear el firmware correcto con framing:

```python
data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
checksum = sum(data) % 256
packet = b'\xAA' + data + bytes([checksum])
uart.write(packet)
```

### Fase 4: Warnings de checksum al arrancar

`Error encontrado: al conectar --device microbitBinary, aparecían warnings en consola durante los primeros segundos:
[MicrobitBinaryAdapter] WARN: corrupted packet discarded (checksum mismatch) | hex: AA 00 80 00 60 00 E0 AA
[MicrobitBinaryAdapter] WARN: corrupted packet discarded (checksum mismatch) | hex: AA 00 74 5C 00 00 D0 AA
[MicrobitBinaryAdapter] WARN: corrupted packet discarded (checksum mismatch) | hex: AA 7C 00 58 00 00 D4 AA`

El patrón era claro: el último byte de cada paquete corrupto era AA — el header del siguiente paquete. El adapter estaba leyendo con un desfase de exactamente un byte porque el OS tenía datos bufferados desde antes de que se abriera el puerto serial.

Solución: llamar flush() sobre el puerto justo después de abrirlo, para descartar los bytes residuales:
```Js
await new Promise((resolve, reject) => {
  this.port.open((err) => (err ? reject(err) : resolve()));
});

// Descartar datos bufferados antes de la conexión para evitar desalineación
await new Promise((resolve) => this.port.flush(resolve));
```
Con este cambio los warnings desaparecieron completamente.

### Fase 5: Registro en bridgeServer.js

Descomenté la línea de importación del `MicrobitBinaryAdapter` y añadí el caso `"microbitbinary"` en `createAdapter()`. No modifiqué ninguna otra parte del servidor.

```Js
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

// dentro de createAdapter():
if (DEVICE === "microbitbinary") {
  const path = SERIAL_PATH ?? await findMicrobitPort();
  ...
  return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
}
```
### Evidencia de funcionamiento
**Simulador**
`node bridgeServer.js --device sim`
`[INFO] WS listening on ws://127.0.0.1:8081 device=sim
[INFO] [ADAPTER] Device Connected: sim connected
[INFO] [NETWORK] Remote Client connected from ::1. Total clients: 1
[INFO] [NETWORK] Client requested adapter connect
[INFO] [HW-POLICY] Adapter already open. Sending current status to incoming client.`

El sketch dibuja polígonos continuamente con los datos simulados.

**Protocolo CSV**
`node bridgeServer.js --device microbit --serialPort`
`[INFO] WS listening on ws://127.0.0.1:8081 device=microbit
[INFO] micro:bit found at COM10
[INFO] [NETWORK] Remote Client connected from ::1. Total clients: 1
[INFO] [NETWORK] Client requested adapter connect
[INFO] [ADAPTER] Device Connected: serial open COM10 @115200`

Datos recibidos en el sketch, polígonos respondiendo al movimiento del micro:bit.

**Protocolo $**
`node bridgeServer.js --device microbit2 --serialPort`
`[INFO] WS listening on ws://127.0.0.1:8081 device=microbit2
[INFO] micro:bit v2 found at COM10
[INFO] [NETWORK] Remote Client connected from ::1. Total clients: 1
[INFO] [NETWORK] Client requested adapter connect
[INFO] [ADAPTER] Device Connected: serial open COM10 @115200`

Detección de trama corrupta (checksum inválido):

`[Microbit2ASCIIAdapter] WARN: corrupted frame discarded — Checksum mismatch: expected 312, got 999 | raw: "$T:45021|X:-245|Y:12|A:1|B:0|CHK:999"`

La trama fue descartada silenciosamente y el sketch no se actualizó con datos basura.

**Protocolo binario**
`node bridgeServer.js --device microbitBinary --serialPort`
`[INFO] WS listening on ws://127.0.0.1:8081 device=microbitbinary
[INFO] micro:bit (binary) found at COM10
[INFO] [NETWORK] Remote Client connected from ::1. Total clients: 1
[INFO] [NETWORK] Client requested adapter connect
[INFO] [ADAPTER] Device Connected: serial open COM10 @115200`

Sin warnings de checksum gracias al flush(). Datos válidos desde el primer paquete.

## Resumen de archivos modificados

| Archivo                               | Acción                                                      |
|--------------------------------------|-------------------------------------------------------------|
| `adapters/Microbit2ASCIIAdapter.js`  | Creado completo (antes era stub vacío)                      |
| `adapters/MicrobitBinaryAdapter.js`  | Creado completo con framing + flush                         |
| `bridgeServer.js`                    | Importación y caso `"microbitbinary"` añadidos; bloque `"microbitauto"` eliminado |
| Todos los demás                      | Sin modificaciones                                          |


## Firmwares por adapter

| Comando                  | Firmware del micro:bit                          |
|--------------------------|------------------------------------------------|
| `--device sim`           | No necesita micro:bit                          |
| `--device microbit`      | CSV: `"{},{},{},{}\n".format(x,y,a,b)`         |
| `--device microbit2`     | `$T{}`                                         |
| `--device microbitBinary`| Binario con `0xAA` + checksum                  |

## Bitácora de reflexión
