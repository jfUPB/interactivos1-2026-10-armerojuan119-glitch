# Unidad 6

## Bitácora de proceso de aprendizaje

## actividad1

### Diferencia entre recibir un mensaje y ejecutarlo

Recibir un mensaje significa que el dato llegó a la aplicación y está
disponible en memoria. Ejecutarlo significa activar la respuesta — en
este caso, dibujar algo en pantalla.

Son dos momentos separados. Un mensaje puede llegar varios milisegundos
antes de que deba ejecutarse. Si los confundo y ejecuto apenas recibo,
la respuesta visual depende de la latencia de red y la carga del CPU —
factores impredecibles que desincronizarían el audio del visual.


### Por qué un sistema audiovisual necesita timestamp

El audio en Strudel se programa con precisión usando Web Audio API, que
tiene su propio reloj de alta resolución. Cuando Strudel emite un evento,
el sonido ya está programado para ocurrir en un instante exacto.

Si el sistema visual no tiene ese timestamp, reacciona cuando el mensaje
llega — que puede ser antes o después del golpe real. El timestamp permite
guardar el evento en una cola y activarlo exactamente cuando `Date.now()`
alcanza ese valor, logrando que visual y audio ocurran en el mismo instante
sin importar cuándo llegó el mensaje por red.


### Aspectos de la arquitectura U4/U5 que permanecen intactos

- **Patrón Adapter:** sigue existiendo una fuente externa con su propio
  protocolo y un adapter que lo traduce a un contrato interno limpio.
- **bridgeServer:** intermediario entre adapter y frontend, sin saber nada
  del protocolo ni de la visualización.
- **bridgeClient:** recibe el mensaje normalizado y lo entrega al sistema
  de estados.
- **FSM (PainterTask):** recibe eventos ya traducidos, sin parsear
  protocolos crudos.
- **Separación updateLogic / drawRunning:** lógica y renderizado en capas
  distintas.

Lo único nuevo es el scheduling temporal — pero se inserta entre
`updateLogic` y `draw()` sin romper nada existente.

## paso 1
Si Strudel fuera "el dispositivo", su protocolo sería:

- **Transporte:** WebSocket sobre `ws://localhost:8080`
- **Formato:** JSON con campos `address`, `args[]` y `timestamp`
- **Cadencia:** un mensaje por cada evento musical programado, con anticipación
- **Dirección:** unidireccional — Strudel solo envía, no escucha

El campo `args` es un array plano de pares clave-valor intercalados:

`['cps', 0.5, 'cycle', 15.25, 'delta', 0.5, 's', 'tr909sd', 'bank', 'tr909']`

lo que lo hace difícil de consumir directamente — justamente por eso necesita un adapter que lo convierta a un objeto limpio.

**Variables mínimas para una visualización útil:**

| Variable | Campo en args | Para qué sirve |
|---|---|---|
| `s` | `s` | Identificar la familia de sonido (bd, sd, hh...) y mapearla a forma/color |
| `timestamp` | raíz del mensaje | Programar cuándo activar la respuesta visual |
| `delta` | `delta` | Duración del evento — define cuánto tiempo dura la respuesta visual |
| `cps` | `cps` | Tempo — permite escalar duraciones visuales al ritmo actual |

## Paso 2 

**¿Qué problema resuelve la cola?**

Sin cola, el sistema reacciona en cuanto el mensaje llega. Pero la red no es determinista: un mensaje puede llegar 20ms antes de su timestamp o 5ms después. Eso produce desfase entre audio y visual que el oído detecta inmediatamente.

La cola guarda los mensajes ordenados por `timestamp` y los ejecuta únicamente cuando `Date.now()` alcanza su valor. Así el visual siempre ocurre en el instante correcto, independientemente de cuándo llegó por la red.

También protege contra ráfagas: si llegan varios mensajes juntos (por ejemplo al conectarse), no se ejecutan todos al mismo tiempo sino cada uno en su momento programado.

**¿Por qué la cola pertenece al frontend y no al bridge?**

El bridge no sabe qué significa el timestamp — no sabe si el receptor es un visualizador, un controlador de luces o un robot. El scheduling es una decisión de quien interpreta el evento, no de quien lo transporta.

Si la cola viviera en el bridge, este empezaría a tomar decisiones de dominio que no le corresponden, rompiendo el principio de responsabilidad única. El bridge solo mueve datos — no decide cuándo ni cómo se usan.

## paso 3

**¿Qué papel cumple el Adapter en U4 y U5?**

En ambas unidades el adapter tiene exactamente el mismo rol: absorber la complejidad del protocolo externo y emitir un objeto limpio y estable hacia el resto del sistema.

- En U4: parsea texto CSV o formato `$T|X|Y|A|B|CHK`, valida checksum y emite `{ x, y, btnA, btnB }`.
- En U5: busca el header `0xAA`, extrae bytes con `readInt16BE`, verifica checksum binario y emite el mismo `{ x, y, btnA, btnB }`.

En los dos casos el adapter es el único componente que conoce el protocolo del dispositivo. El resto del sistema no sabe — ni necesita saber — si los datos vienen en ASCII, binario o cualquier otro formato.

**¿Qué adapter necesito para Strudel?**

Necesito un adapter que:
1. Actúe como servidor WebSocket en el puerto 8080 para recibir las conexiones de Strudel
2. Filtre solo los mensajes con `address === "/dirt/play"`
3. Convierta el array plano `args[]` en un objeto clave-valor
4. Emita un objeto normalizado con `type`, `timestamp` y `payload` — sin exponer la estructura cruda de Strudel al resto del sistema

Sin este adapter, los mensajes crudos de Strudel entrarían directamente al frontend con su estructura de array intercalado, mezclando la lógica de parseo con la lógica visual — exactamente lo que la arquitectura del curso existe para evitar.


## Bitácora de aplicación 

### Cómo configuré Strudel para emitir eventos

Abrí `[strudel.cc](https://strudel.cc)` en el navegador y escribí el siguiente patrón:

```javascript
setcps(0.5)

$: stack(
  s("[bd*2 sd hh oh]").bank("tr909").gain(0.8),
  s("[bd*2 sd hh oh]").bank("tr909").osc()
)
```
La clave está en el método .osc(): sin él el patrón solo produce audio en el navegador mediante Web Audio API, pero no envía ningún dato al sistema visual. Con .osc(), Strudel abre automáticamente una conexión `WebSocket hacia ws://localhost:8080` y envía un mensaje por cada evento musical que ocurre en el patrón.

Arranco el servidor con:


`node bridgeServer.js --device strudel`
Y abro `index.html` con `Go Live`. Al presionar Play en Strudel, el ritmo suena en el navegador y los anillos aparecen sincronizados en el canvas.

Estructura final del mensaje
Strudel envía mensajes crudos con este formato:

```
{
  "address": "/dirt/play",
  "args": ["cps", 0.5, "cycle", 15.25, "delta", 0.5, "s", "tr909sd", "bank", "tr909"],
  "timestamp": 1774966984435.2805
}
```
El array args es una lista plana de pares clave-valor. Decidí parsearla en StrudelAdapter._normalize() convirtiéndola a un objeto, y emitir este contrato normalizado:

```
{
  "type": "strudel",
  "timestamp": 1774966984435.2805,
  "payload": {
    "s": "tr909sd",
    "delta": 0.5,
    "cps": 0.5,
    "cycle": 15.25
  }
}
```
Decidí incluir cps y cycle aunque la visualización no los use directamente, porque son útiles para calcular duraciones relativas si en el futuro quiero escalas visuales proporcionales al tempo.

Solo proceso mensajes con address `=== "/dirt/play"`  los demás (metadatos de Strudel) se descartan silenciosamente.

### Cómo conecté bridgeClient.js, FSMTask, updateLogic y drawRunning
El flujo de datos completo es:


Strudel (browser)


└─ .osc() → WS ws://localhost:8080

    └─ StrudelAdapter._normalize()   
    
        └─ this.onData?.({ type: "strudel", timestamp, payload })
            └─ bridgeServer.js (broadcast) → WS ws://localhost:8081
                └─ bridgeClient.js (onmessage)
                    └─ if (msg.type === "strudel")
                        └─ this._onData?.(msg)
                            └─ bridge.onData(data)
                                └─ painter.postEvent({ type: EVENTS.DATA, payload: data })
                                    └─ estado_corriendo → updateLogic(data)
                                        └─ if (data.type === "strudel")
                                            └─ eventQueue.push(...)
                                            
bridgeClient.js: añadí msg.type === "strudel" a la condición existente:


```
if (msg.type === "microbit" || msg.type === "strudel") {
  this._onData?.(msg);
  return;
}

```
FSMTask: no se modificó. El evento llega como  `EVENTS.DATA` igual que con el micro:bit — la FSM no sabe ni le importa si el dato viene de hardware o de Strudel.

updateLogic: detecta el tipo antes de procesar:

```
updateLogic(data) {
  if (data.type === "strudel") {
    this.eventQueue.push({ timestamp: data.timestamp, payload: data.payload });
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    return;
  }
  // lógica microbit existente sin cambios
}
drawRunning (en draw()): lee solo activeEvents — no interpreta mensajes de red:


for (const ev of painter.activeEvents) {
  const age   = (millis() - ev.startAt) / ev.duration;
  const alpha = 255 * (1 - age);
  const sz    = ev.maxSize * age;
  push();
  noFill();
  stroke(ev.c[0], ev.c[1], ev.c[2], alpha);
  strokeWeight(max(1, 3 * (1 - age)));
  ellipse(ev.x, ev.y, sz, sz);
  pop();
}
```

### Cómo separé recepción, cola temporal y renderizado
Identifiqué tres momentos distintos que nunca mezclo:

| Momento     | Dónde ocurre                         | Qué hace                                              |
|------------|--------------------------------------|-------------------------------------------------------|
| Recepción  | `StrudelAdapter` + `bridgeClient`    | Recibe el mensaje crudo y lo normaliza                |
| Cola temporal | `updateLogic` + `tickQueue`       | Guarda el evento y espera a que `Date.now()` alcance su timestamp |
| Renderizado | `draw()` leyendo `activeEvents`     | Solo dibuja lo que el estado ya calculó              |


tickQueue() es el método que conecta la cola con el renderizado. Se llama cada frame en draw() antes de dibujar:

```
tickQueue() {
  const now = Date.now();
  while (this.eventQueue.length > 0 && this.eventQueue[0].timestamp <= now) {
    const ev = this.eventQueue.shift();
    this._fireVisualEvent(ev.payload); // traduce payload → estado visual
  }
  this.activeEvents = this.activeEvents.filter(
    (e) => millis() - e.startAt < e.duration
  );
}
```

`_fireVisualEvent` es la única función que mapea sonido a visual: asigna color, tamaño y posición según la familia de sonido. No dibuja nada — solo actualiza activeEvents.

### Pruebas de sincronización
**Prueba 1** — llegada de mensajes: abrí la consola del servidor y verifiqué que los mensajes llegaban correctamente desde Strudel:


`[INFO] [ADAPTER] Device Connected: Strudel WS listening on ws://localhost:8080`
`[INFO] [NETWORK] Remote Client connected from ::1. Total clients: 1`

**Prueba 2** — tipo correcto: verifiqué en la consola del navegador que bridge.onData recibía mensajes con type: "strudel" y que payload.s tenía los valores correctos `(tr909bd, tr909sd, tr909hh, tr909oh).`

**Prueba 3** — sincronización visual: comparé el onset del sonido con la aparición del anillo en pantalla. Los anillos aparecen sincrónicos al golpe del ritmo porque el timestamp de Strudel indica exactamente cuándo debe ocurrir el evento, y `tickQueue` lo ejecuta cuando `Date.now()` lo alcanza.

**Prueba 4** — coexistencia con microbit: verifiqué que --device sim seguía funcionando sin cambios en el canvas del micro:bit. La lógica Strudel solo se activa si `data.type === "strudel"`, por lo que ambos sistemas coexisten sin interferencia.

### Problemas encontrados y soluciones

**Problema 1** — onData hardcodeado para microbit:
bridgeServer.js construía siempre `{ type:"microbit", x, y, btnA, btnB }.` Los eventos de Strudel llegaban al servidor pero se deformaban al salir.

**Solución:** modifiqué adapter`.onData` para detectar si el adapter ya emite un mensaje tipado:

```
adapter.onData = (d) => {
  if (d.type) {
    broadcast(wss, { ...d, t: nowMs() }); // Strudel y futuros adapters
  } else {
    broadcast(wss, { type:"microbit", x:d.x, y:d.y, ... }); // legacy microbit
  }
};
```

**Problema 2** — StrudelAdapter no se auto-conectaba:
Los adapters de micro:bit esperan el comando connect del cliente. Pero StrudelAdapter es un servidor — debe arrancar inmediatamente al iniciar `Node.js.`

**Solución:** añadí strudel a la condición de auto-conexión en bridgeServer.js:

```
if (DEVICE === "sim" || DEVICE === "strudel") {
  await adapter.connect();
}
```

**Problema 3** — bridgeClient ignoraba mensajes Strudel:
Solo manejaba `msg.type === "microbit"`. Los mensajes Strudel llegaban al cliente pero se descartaban silenciosamente.

**Solución:** una sola línea en bridgeClient.js:


`if (msg.type === "microbit" || msg.type === "strudel") {`

## Bitácora de reflexión



### 1. Diagrama detallado del flujo de datos

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        STRUDEL (Navegador Tab 1)                         │
│                                                                          │
│  Patrón rítmico:                                                         │
│  $: stack(                                                               │
│       s("[bd2 sd hh oh]").bank("tr909").gain(0.8),  ← audio             │
│       s("[bd2 sd hh oh]").bank("tr909").osc()       ← eventos WS        │
│     )                                                                    │
│                                                                          │
│  Por cada evento emite:                                                  │
│  { address:"/dirt/play", args:[...], timestamp:1774966984435 }           │
└─────────────────────────────┬────────────────────────────────────────────┘
                              │ WebSocket CLIENT → ws://localhost:8080
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               StrudelAdapter.js  (Node.js)                               │
│                                                                          │
│  WebSocketServer en puerto 8080 ← adapter actúa como servidor           │
│                                                                          │
│  _normalize(msg):                                                        │
│  ├── Filtra: solo address === "/dirt/play"                               │
│  ├── Parsea args[] (array plano → objeto clave-valor)                    │
│  └── Emite: { type:"strudel", timestamp, payload:{s,delta,cps,cycle} }  │
│                                                                          │
│  this.onData?.( objeto normalizado )  ← CONTRATO con bridgeServer       │
└─────────────────────────────┬────────────────────────────────────────────┘
                              │ callback onData
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               bridgeServer.js  (Node.js)                                 │
│                                                                          │
│  adapter.onData = (d) => {                                               │
│    if (d.type) broadcast(wss, { ...d, t: nowMs() });  ← Strudel          │
│    else broadcast(wss, { type:"microbit", x,y,... }); ← legacy           │
│  }                                                                       │
│                                                                          │
│  WebSocketServer en puerto 8081                                          │
│  Broadcast → todos los clientes conectados                               │
└─────────────────────────────┬────────────────────────────────────────────┘
                              │ WebSocket SERVER → ws://localhost:8081
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               bridgeClient.js  (Navegador Tab 2 — index.html)            │
│                                                                          │
│  ws.onmessage → JSON.parse → inspecciona msg.type                       │
│  ├── "status"   → actualiza UI                                           │
│  ├── "microbit" → this._onData?.(msg)                                    │
│  └── "strudel"  → this._onData?.(msg)  ← añadido en esta unidad         │
│                                                                          │
│  bridge.onData(data) →                                                   │
│    painter.postEvent({ type: EVENTS.DATA, payload: data })               │
└─────────────────────────────┬────────────────────────────────────────────┘
                              │ EVENTS.DATA → FSM
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               PainterTask FSM  (sketch.js)                               │
│                                                                          │
│  estado_corriendo recibe EVENTS.DATA → updateLogic(data)                 │
│                                                                          │
│  updateLogic(data):                                                      │
│  ├── if data.type === "strudel"                                          │
│  │     eventQueue.push({ timestamp, payload })                           │
│  │     eventQueue.sort()  ← ordenado por timestamp                       │
│  └── else (microbit)                                                     │
│        rxData.x/y/btnA/btnB ← escalado + map()                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────┐            │
│  │              COLA DE EVENTOS TEMPORALES                  │            │
│  │  [ {ts:1774966984400, payload:{s:"bd",...}},             │            │
│  │    {ts:1774966984600, payload:{s:"sd",...}},  ... ]      │            │
│  └─────────────────────────┬───────────────────────────────┘            │
│                            │ tickQueue() — llamado cada frame            │
│                            ▼                                             │
│  while queue[0].timestamp <= Date.now():                                 │
│    _fireVisualEvent(payload) → activeEvents.push({x,y,c,size,dur})      │
└─────────────────────────────┬────────────────────────────────────────────┘
                              │ activeEvents (estado visual ya calculado)
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               Render Visual — draw()  (p5.js)                            │
│                                                                          │
│  for (ev of painter.activeEvents):                                       │
│    age   = (millis() - ev.startAt) / ev.duration  // 0 → 1              │
│    alpha = 255 * (1 - age)                         // fade out           │
│    sz    = ev.maxSize * age                        // expansión          │
│    ellipse(ev.x, ev.y, sz, sz)  con color de familia de sonido          │
│                                                                          │
│  bd  → anillo púrpura grande desde el centro                            │
│  sd  → anillo blanco mediano en posición aleatoria                      │
│  hh  → anillo dorado pequeño en posición aleatoria                      │
│  oh  → anillo naranja mediano en posición aleatoria                     │
└──────────────────────────────────────────────────────────────────────────┘
```
## 2. Tabla comparativa Unidades 4, 5 y 6

| Característica | Unidad 4 — ASCII | Unidad 5 — Binario | Unidad 6 — Strudel |
|---|---|---|---|
| **Fuente de datos** | micro:bit físico por puerto serial | micro:bit físico por puerto serial | Strudel corriendo en el navegador |
| **Formato del mensaje** | Texto ASCII: `x,y,True,False\n` o `$T:\|X:\|CHK:\n` | Bytes binarios: `[0xAA][X_H][X_L][Y_H][Y_L][A][B][CHK]` | JSON por WebSocket: `{address, args[], timestamp}` |
| **Problema técnico principal** | Parsear texto, validar checksum ASCII | Sincronizar buffer de bytes, detectar header `0xAA`, evitar falsos positivos | Programar cuándo ejecutar el evento, no solo recibirlo |
| **Mecanismo de validación** | Checksum numérico en campo `CHK:` | `(sum bytes 1-6) % 256 === byte[7]` | Filtro por `address === "/dirt/play"` + parseo de args |
| **Lugar de traducción** | `MicrobitASCIIAdapter.parseCsvLine()` | `MicrobitBinaryAdapter.parsePacket()` | `StrudelAdapter._normalize()` |
| **Papel del tiempo / sincronización** | Sin relevancia — dato llega y se renderiza inmediatamente | Sin relevancia — flush al abrir puerto para evitar desalineación | Central — el `timestamp` del mensaje determina cuándo ocurre la respuesta visual |

---

### 3. Por qué esta unidad sigue siendo la misma arquitectura

Aunque la fuente de datos ya no es hardware físico, la arquitectura es idéntica porque el problema de diseño es el mismo: **aislar capas para que ninguna sepa cómo funciona la anterior**.

En las unidades 4 y 5 el adapter leía bytes del puerto serial y emitía `{ x, y, btnA, btnB }`. En la unidad 6 el adapter lee mensajes de un WebSocket y emite `{ type:"strudel", timestamp, payload }`. El mecanismo es diferente pero el rol es el mismo: absorber la complejidad del protocolo externo y convertirlo en un objeto limpio.

`bridgeServer.js` nunca supo qué había al otro lado del adapter — ni en la unidad 4 ni ahora. `sketch.js` nunca supo de dónde vienen los datos — ni antes ni ahora. `fsm.js` no cambió ni una línea.

La única diferencia conceptual real es que en esta unidad apareció el **scheduling temporal**: los datos no se renderizan al llegar sino cuando su `timestamp` lo indica. Eso vive en `tickQueue()`, una capa nueva que no rompió nada de lo existente — simplemente se insertó entre `updateLogic` y `draw()`.

La arquitectura desacoplada absorbió ese cambio sin resistencia porque cada capa tiene una responsabilidad única y definida.

---

### 4. Decisiones de traducción de eventos musicales a visualidad

Decidí mapear cada familia de sonido a un **anillo expansivo** con color y tamaño propios:

| Sonido | Color | Tamaño máx. | Posición | Justificación |
|---|---|---|---|---|
| `bd` (kick) | Púrpura `[80,40,220]` | 300px | Centro del canvas | El bombo es el pulso central del ritmo — su posición fija en el centro ancla visualmente la composición |
| `sd` / `cp` (snare/clap) | Blanco `[220,220,220]` | 160px | Aleatoria | El snare rompe el patrón — su posición aleatoria refleja esa irregularidad |
| `hh` (hi-hat cerrado) | Dorado `[255,210,0]` | 60px | Aleatoria | El hi-hat es pequeño y rápido — pequeño y brillante |
| `oh` (hi-hat abierto) | Naranja `[255,140,0]` | 90px | Aleatoria | Más largo que el hh cerrado, lo refleja en tamaño y tono más cálido |

La forma de anillo expansivo tiene sentido porque replica la física del sonido: una fuente puntual que irradia energía hacia afuera y se disipa. La opacidad y el trazo disminuyen con la edad del evento (`age = elapsed / duration`), de modo que los sonidos cortos desaparecen rápido y los largos dejan huella más tiempo — igual que en el audio.

El uso de `delta` como duración visual hace que el tempo del patrón dicte directamente la velocidad de los anillos: a tempo rápido los anillos son cortos y ágiles, a tempo lento son más largos y contemplativos.

---

### 5. Si tuviera que integrar una tercera aplicación

**Conservaría sin cambios:**

- `BaseAdapter.js` — el contrato es sólido y agnóstico
- `bridgeServer.js` — el `onData` genérico ya acepta cualquier adapter que emita `d.type`
- `bridgeClient.js` — solo necesitaría añadir otro `msg.type` a la condición
- `fsm.js` — no necesita saber nada de la fuente
- El patrón `updateLogic` → cola → `tickQueue` → render — es reutilizable

**Cambiaría o extendería:**

- Crearía un nuevo adapter específico para la tercera app (igual que hice con `StrudelAdapter`)
- Si esa app usa un protocolo de tiempo diferente (ej. MIDI clock, ticks de BPM), añadiría un mecanismo de conversión de tiempo dentro del adapter, no en el servidor
- Si necesitara múltiples adapters activos simultáneamente (ej. micro:bit + Strudel al mismo tiempo), modificaría `bridgeServer.js` para soportar un array de adapters en lugar de uno solo — ese sería el cambio estructural más significativo
