# Unidad 3

## Bitácora de proceso de aprendizaje
- Primero analicé la máquina de estados original del semáforo.

- Identifiqué que debía agregar nuevos estados en lugar de meter lógica dentro de los existentes.

- Implementé el modo peatonal creando un estado intermedio amarillo antes del rojo.

- Implementé el modo nocturno usando dos estados que alternan el amarillo para generar parpadeo.

- Tuve errores con eventos repetidos del botón, pero los solucioné usando `was_pressed()`.

- Aprendí que las máquinas de estados permiten agregar comportamiento sin romper el código anterior.
  
``` C#
from microbit import *
import utime

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


class Semaforo:
    def __init__(self,x,y,tRed,tGreen,tYellow):
        self.event_queue = []
        self.timers = []
        self.x=x
        self.y=y

        self.tRed=tRed
        self.tGreen=tGreen
        self.tYellow=tYellow

        self.timer=self.createTimer("Timeout",tRed)

        self.estado_actual=None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t=Timer(self,event,duration)
        self.timers.append(t)
        return t

    def post_event(self,ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()

        while self.event_queue:
            ev=self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self,nuevo):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual=nuevo
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    # ===== NORMAL =====
    def estado_waitInRed(self,ev):
        if ev=="ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.timer.start(self.tRed)

        if ev=="Timeout":
            self.transicion_a(self.estado_waitInGreen)

        if ev=="B":
            self.transicion_a(self.estado_nightBlinkOn)

    def estado_waitInGreen(self,ev):
        if ev=="ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.timer.start(self.tGreen)

        if ev=="Timeout":
            self.transicion_a(self.estado_waitInYellow)

        if ev=="A":  # peatón
            self.transicion_a(self.estado_pedestrianYellow)

        if ev=="B":
            self.transicion_a(self.estado_nightBlinkOn)

    def estado_waitInYellow(self,ev):
        if ev=="ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.timer.start(self.tYellow)

        if ev=="Timeout":
            self.transicion_a(self.estado_waitInRed)

        if ev=="B":
            self.transicion_a(self.estado_nightBlinkOn)

    # ===== PEATONAL =====
    def estado_pedestrianYellow(self,ev):
        if ev=="ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.timer.start(self.tYellow)

        if ev=="Timeout":
            self.transicion_a(self.estado_waitInRed)

    # ===== NOCTURNO =====
    def estado_nightBlinkOn(self,ev):
        if ev=="ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.timer.start(400)

        if ev=="Timeout":
            self.transicion_a(self.estado_nightBlinkOff)

        if ev=="A":
            self.transicion_a(self.estado_waitInRed)

    def estado_nightBlinkOff(self,ev):
        if ev=="ENTRY":
            self.clear()
            self.timer.start(400)

        if ev=="Timeout":
            self.transicion_a(self.estado_nightBlinkOn)

        if ev=="A":
            self.transicion_a(self.estado_waitInRed)


semaforo1=Semaforo(0,0,2000,1000,500)

while True:
    if button_a.was_pressed():
        semaforo1.post_event("A")

    if button_b.was_pressed():
        semaforo1.post_event("B")

    semaforo1.update()
    utime.sleep_ms(20)
```
**Errores y soluciones durante la implementación**

---
**Error 1: El botón A generaba muchos cambios de estado**

**Problema:**  
Al presionar el botón A el semáforo cambiaba varias veces seguidas y saltaba estados.

**Causa:**  
Se estaba usando `button_a.is_pressed()` dentro del loop, lo que generaba múltiples eventos.

**Solución:**  
Se reemplazó por `button_a.was_pressed()` para registrar solo una pulsación.

**Aprendizaje:**  
En sistemas basados en eventos es importante evitar eventos duplicados.

---
**Error 2: El modo nocturno no parpadeaba**

**Problema:**  
El semáforo se quedaba en amarillo fijo en lugar de parpadear.

**Causa:**  
Solo había un estado nocturno y no alternaba entre encendido y apagado.

**Solución:**  
Se crearon dos estados: uno con el amarillo encendido y otro apagado, que se alternan mediante un temporizador.

**Aprendizaje:**  
El parpadeo se modela mejor como una alternancia de estados.

---
**Error 3: El modo peatonal saltaba directamente a rojo**

**Problema:**  
Al presionar el botón A el semáforo no pasaba por amarillo antes de ponerse en rojo.

**Causa:**  
Se hacía una transición directa al estado rojo.

**Solución:**  
Se agregó un estado intermedio de amarillo peatonal antes del rojo.

**Aprendizaje:**  
Las transiciones intermedias son necesarias para representar comportamientos reales.

---
**Error 4: El temporizador seguía activo al cambiar de modo**

**Problema:**  
Al cambiar a modo nocturno el semáforo a veces cambiaba inesperadamente.

**Causa:**  
El temporizador del estado anterior seguía generando eventos.

**Solución:**  
Se reinició el temporizador en cada `ENTRY` del nuevo estado.

**Aprendizaje:**  
Cada estado debe controlar sus propios temporizadores.

---
**Error 5: Los eventos de botones no se procesaban**

**Problema:**  
En algunos momentos el semáforo no respondía a los botones.

**Causa:**  
Los eventos no se estaban enviando a la cola del semáforo.

**Solución:**  
Se utilizó `semaforo.post_event("A")` y `semaforo.post_event("B")` para enviar los eventos correctamente.

**Aprendizaje:**  
En arquitecturas orientadas a eventos, los botones no cambian estados directamente, sino que generan eventos que el sistema procesa.

## Bitácora de aplicación 



## Bitácora de reflexión
