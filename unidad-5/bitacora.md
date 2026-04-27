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

### Fase 2: Construcción de `Microbit2ASCIIAdapter.js`

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

### Resumen de archivos modificados

| Archivo                               | Acción                                                      |
|--------------------------------------|-------------------------------------------------------------|
| `adapters/Microbit2ASCIIAdapter.js`  | Creado completo (antes era stub vacío)                      |
| `adapters/MicrobitBinaryAdapter.js`  | Creado completo con framing + flush                         |
| `bridgeServer.js`                    | Importación y caso `"microbitbinary"` añadidos; bloque `"microbitauto"` eliminado |
| Todos los demás                      | Sin modificaciones                                          |


### Firmwares por adapter

| Comando                  | Firmware del micro:bit                          |
|--------------------------|------------------------------------------------|
| `--device sim`           | No necesita micro:bit                          |
| `--device microbit`      | CSV: `"{},{},{},{}\n".format(x,y,a,b)`         |
| `--device microbit2`     | `$T{}`                                         |
| `--device microbitBinary`| Binario con `0xAA` + checksum                  |

## Bitácora de reflexión

### Tabla comparativa: Microbit2ASCIIAdapter vs MicrobitBinaryAdapter

| Característica | `Microbit2ASCIIAdapter` (ASCII `$`) | `MicrobitBinaryAdapter` (Binario) |
|---|---|---|
| **Tamaño del paquete** | Variable (~35–50 bytes típico) | Fijo: exactamente 8 bytes |
| **Mecanismo de framing** | Carácter `$` al inicio + `\n` al final | Byte `0xAA` al inicio, tamaño fijo conocido |
| **Checksum** | Suma de `|X|+|Y|+|A|+|B|` en decimal, campo `CHK:` al final | `(sum bytes 1..6) % 256`, 1 byte al final |
| **Parser** | Separar por `|`, buscar `:`, convertir strings a números | `readInt16BE()` y acceso directo por índice de byte |
| **Depuración con terminal serial** | Muy fácil — legible directamente: `$T:45020|X:-245|Y:12|A:1|B:0|CHK:258` | Imposible a simple vista — se ven caracteres extraños o basura |
| **Overhead por paquete** | Alto — los nombres de campo (`T:`, `X:`, `CHK:`, etc.) ocupan la mayoría del paquete | Mínimo — 1 byte header + 1 byte checksum sobre 6 bytes de datos |
| **Resistencia a desalineación** | Alta — `\n` es un delimitador único que no aparece en los datos | Media — `0xAA` puede aparecer como dato válido (falso positivo) |
| **Velocidad de parseo** | Más lenta — requiere split, trim, conversión de string a número | Más rápida — operaciones bit a bit directas sobre buffer |

---

### ¿Por qué la arquitectura desacoplada permite añadir protocolos sin tocar el frontend?

La arquitectura tiene tres capas que se comunican mediante **contratos fijos**:

`[Hardware] → [Adapter] → {x, y, btnA, btnB} → [bridgeServer] → WebSocket JSON → [bridgeClient] → [FSM / sketch.js]`


El contrato entre capas nunca cambia:

- El adapter siempre emite `this.onData?.({ x, y, btnA, btnB })` — no importa si parsea CSV, `$` o binario.
- El bridgeServer siempre transmite `{ type:"microbit", x, y, btnA, btnB, t }` — no sabe nada del protocolo físico.
- El `sketch.js` siempre lee `ev.payload.x`, `ev.payload.btnA`, etc. — no sabe que existe el serial.

Cuando añado `MicrobitBinaryAdapter`, el cambio está **completamente contenido** dentro de ese archivo. El resto del sistema no necesita saber que ahora los datos vienen en 8 bytes binarios en lugar de 40 caracteres ASCII. Esto es el **principio de sustitución**: cualquier adapter que cumpla el contrato puede reemplazar a cualquier otro sin romper nada.

Si no existiera esta arquitectura y el parsing estuviera mezclado con el renderizado en `sketch.js`, cada nuevo protocolo requeriría modificar el frontend, el transporte y el servidor al mismo tiempo.

---

## ¿Cuándo usar protocolo binario vs ASCII en el mundo real?

### Prefiero binario cuando:

- **Ancho de banda es limitado:** sensores inalámbricos (BLE, Zigbee, LoRa) tienen payloads máximos pequeños. Un paquete binario de 8 bytes vs 40 bytes ASCII es una diferencia crítica en una red LoRa con 250 bps.
- **Alta frecuencia de muestreo:** un IMU enviando datos a 1000 Hz genera 40 KB/s en ASCII vs 8 KB/s en binario. A 115200 baud, el ASCII casi satura el canal.
- **Sistemas embebidos con CPU limitada:** generar strings con `format()` en un microcontrolador de 16 MHz es costoso. `struct.pack` es una sola instrucción.
- **Ejemplo concreto:** sensores médicos (oxímetros, ECG portátiles) que transmiten por Bluetooth a 100 Hz — cada byte extra agota más rápido la batería.

### Prefiero ASCII cuando:

- **Depuración activa:** durante desarrollo puedo abrir un terminal serial y leer los datos directamente sin ninguna herramienta adicional. Con binario necesito un decodificador hex.
- **Interoperabilidad:** si el sistema debe integrarse con herramientas distintas (una app móvil, un servidor Python, un dashboard web), el ASCII es parseable en cualquier lenguaje sin acordar un schema binario.
- **Protocolos de configuración:** comandos como `SET_FREQ:50\n` son más seguros de implementar en ASCII porque los errores son legibles.
- **Ejemplo concreto:** estaciones meteorológicas que reportan a servidores de terceros — usar CSV o JSON permite que cualquier sistema los consuma sin documentación adicional del protocolo binario.

---

### Diagrama de flujo actualizado con protocolo binario
```
┌─────────────────────────────────────────────────────────────────────┐
│                     MICRO:BIT (Hardware)                            │
│                                                                     │
│  Acelerómetro → xValue, yValue                                      │
│  Botón A → aState (0/1)       Firmware Python (struct.pack)         │
│  Botón B → bState (0/1)                                             │
│                                                                     │
│  data     = struct.pack('>2h2B', x, y, a, b)  ← 6 bytes            │
│  checksum = sum(data) % 256                   ← 1 byte              │
│  packet   = b'\xAA' + data + bytes([chk])     ← 8 bytes total       │
│                                                                     │
│  UART 115200 baud @ 10 Hz — paquetes binarios de 8 bytes           │
│  AA 01 F4 02 0C 01 00 FE                                            │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │ Puerto Serial (COM10) ← CAMBIÓ (binario)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   BACKEND NODE.JS  ← CAMBIÓ SOLO EL ADAPTER         │
│                                                                     │
│  bridgeServer.js  (sin cambios)                                     │
│  └── createAdapter("microbitbinary")                                │
│       └── MicrobitBinaryAdapter  ← NUEVO                            │
│            ├── connect() → flush() para limpiar buffer del OS       │
│            ├── _onChunk(): acumula bytes en Buffer binario          │
│            ├── Busca header 0xAA al inicio del buffer               │
│            ├── Espera 8 bytes completos                             │
│            ├── computed = (sum bytes[1..6]) % 256                   │
│            ├── Si computed ≠ pkt[7] → console.warn + descarta       │
│            └── onData?.({ x, y, btnA, btnB })  ← MISMO CONTRATO     │
│                                                                     │
│  bridgeServer transmite (sin cambios):                              │
│  { type:"microbit", x, y, btnA, btnB, t }                           │
│                                                                     │
│  WebSocketServer ws://127.0.0.1:8081  (sin cambios)                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │ WebSocket JSON  ← SIN CAMBIOS
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│              FRONTEND (Navegador)  ← SIN NINGÚN CAMBIO              │
│                                                                     │
│  bridgeClient.js  (sin cambios)                                     │
│  sketch.js / PainterTask FSM  (sin cambios)                         │
│  updateLogic() / draw()  (sin cambios)                              │
└─────────────────────────────────────────────────────────────────────┘
```


### Componentes que cambiaron vs los que no:

| Componente | ¿Cambió? | Razón |
|---|---|---|
| Firmware micro:bit | Sí | Nuevo protocolo binario con framing |
| `MicrobitBinaryAdapter.js` | Sí (creado nuevo) | Parser binario específico |
| `bridgeServer.js` | Mínimo | Solo añadí el `require` y el caso `"microbitbinary"` |
| `bridgeClient.js` | No | El transporte WS no depende del protocolo físico |
| `sketch.js` | No | Solo lee `{x, y, btnA, btnB}`, no le importa el origen |
| `fsm.js` | No | Infraestructura pura de estados |
| `BaseAdapter.js` | No | El contrato ya cubría este caso |

El hecho de que **5 de 7 componentes no requirieran ningún cambio** para soportar un protocolo completamente diferente es la demostración práctica de que la arquitectura desacoplada funciona correctamente.
