# Unidad 6

## Bitácora de proceso de aprendizaje


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
