# Unidad 2

## Bitácora de proceso de aprendizaje
### actividad 1... Análisis de una máquina de estados simple (micro:bit)

## ¿Cuáles son los estados en el programa?

Depende de la implementación que analicemos.

### Primera versión del programa

Existe **un solo estado explícito**:

- `estado_waitTimeout`

Sin embargo, dentro de ese estado el sistema alterna entre dos condiciones internas:

- Pixel encendido (`pixelState = 9`)
- Pixel apagado (`pixelState = 0`)

En esta versión el cambio entre encendido y apagado ocurre dentro del mismo estado cuando sucede el evento `"Timeout"`.

### Segunda versión del programa (modelo más formal)

En esta implementación los estados están claramente definidos:

1. `estado_waitInON` → El pixel está encendido.
2. `estado_waitInOFF` → El pixel está apagado.

Este modelo representa mejor una máquina de estados porque cada estado describe una condición clara del sistema y las transiciones están bien separadas.

## ¿Cuáles son los eventos en el programa?

Los eventos son señales que provocan acciones o cambios de estado.

Los eventos del programa son:

- `"ENTRY"`  
  Se ejecuta automáticamente cuando se entra a un estado.  
  Se usa para realizar acciones iniciales.

- `"EXIT"`  
  Se ejecuta cuando se sale de un estado.  
  (En este programa no se usa mucho, pero está definido.)

- `"Timeout"`  
  Es generado por la clase `Timer` cuando el tiempo programado se cumple.  
  Este evento provoca el cambio de estado o el cambio de valor del pixel.

## ¿Cuáles son las acciones en el programa?

Las acciones son las operaciones que el sistema realiza cuando ocurre un evento.

Las acciones principales son:

## Encender el pixel

```python
display.set_pixel(self.x, self.y, 9)
```
## Apagar el pixel
```python
display.set_pixel(self.x, self.y, 0)
```
## Iniciar el temporizador
```python
self.myTimer.start()
```
## Bitácora de aplicación 
Diagrama de PlantUML
```
@startuml
[*] --> Configuracion

Configuracion : entry/ count = inicia 20
Configuracion : A / incrementar count (max 25)
Configuracion : B / decrementar count (min 15)
Configuracion : S / armar temporizador
Configuracion --> Contando : S

Contando : iniciar timer (1000ms)= 1s
Contando : Timeout --, mostrar
Contando --> Alarma : Timeout [count == 0]
Contando --> Contando : Timeout [count > 0]

Alarma : entry/ mostrar calavera, sonar speaker
Alarma : A / reiniciar
Alarma --> Configuracion : A is pressed

@enduml
```
Código funcional
```Jv
from microbit import *
import utime
import music

# Crear las imágenes de llenado del display
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()


class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# Máquina de estados
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.countdownTimer = self.createTimer("Timeout", 1000)
        
        self.count = 20  
        
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        
        for t in self.timers:
            t.update()

        
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_configuracion(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.count])
            
        elif ev == "A":
            
            if self.count < 25:
                self.count += 1
                display.show(FILL[self.count])
                
        elif ev == "B":
            
            if self.count > 15:
                self.count -= 1
                display.show(FILL[self.count])
                
        elif ev == "S":
            
            self.transicion_a(self.estado_contando)

    def estado_contando(self, ev):
        if ev == "ENTRY":
            self.countdownTimer.start(1000)
            
        elif ev == "Timeout":
            self.count -= 1
            display.show(FILL[self.count])
            
            if self.count == 0:
              
                self.transicion_a(self.estado_alarma)
            else:
                
                self.countdownTimer.start(1000)
                
        elif ev == "EXIT":
            self.countdownTimer.stop()

    def estado_alarma(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.POWER_UP)
            
        elif ev == "A":
            
            self.count = 20
            self.transicion_a(self.estado_configuracion)

# Ciclo principal
task = Task()

while True:
   
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```

## Bitácora de reflexión




