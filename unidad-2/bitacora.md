# Unidad 2

## Bitácora de proceso de aprendizaje


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

# Función para crear las imágenes de llenado del display
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

# Clase Timer
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

# Clase Task con máquina de estados
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


