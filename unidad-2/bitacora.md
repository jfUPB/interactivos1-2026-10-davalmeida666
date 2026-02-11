# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 1
Código:
```.py

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


class Pixel:
    def __init__(self,_x,_y,_interval):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.pixelState = 0
        self.myTimer = self.createTimer("Timeout",_interval)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInON)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_waitInON(self, ev):
        if ev == "ENTRY":
            self.pixelState = 9
            display.set_pixel(self.x,self.y,self.pixelState)
            self.myTimer.start()
        elif ev == "Timeout":
            self.transicion_a(self.estado_waitInOFF)

    def estado_waitInOFF(self, ev):
        if ev == "ENTRY":
            self.pixelState = 0
            display.set_pixel(self.x,self.y,self.pixelState)
            self.myTimer.start()
        elif ev == "Timeout":
            self.transicion_a(self.estado_waitInON)

pixel1 = Pixel(0,0,1000)
pixel2 = Pixel(4,4,600)

while True:
    pixel1.update()
    pixel2.update()
    utime.sleep_ms(20)
```
#### ¿Cuáles son los estados en el programa?
Los estados están definidos como métodos dentro de la clase `Pixel`.

### Estados existentes

1. **`estado_waitInON`**
   - El píxel se encuentra encendido (brillo 9).
   - Permanece en este estado hasta que ocurre el evento `Timeout`.

2. **`estado_waitInOFF`**
   - El píxel se encuentra apagado (brillo 0).
   - Permanece en este estado hasta que ocurre el evento `Timeout`.

Cada objeto `Pixel` alterna continuamente entre estos dos estados.


#### ¿Cuáles son los eventos en el programa?

**`ENTRY`**
  - Se genera automáticamente al entrar a un estado.
  - Se usa para inicializar el estado.

- **`EXIT`**
  - Se genera automáticamente al salir de un estado.
  - No se utiliza en este programa, pero está implementado.

- **`Timeout`**
  - Generado por el temporizador (`Timer`) cuando se cumple el tiempo configurado.
  - Provoca el cambio de estado.

---
#### ¿Cuáles son las acciones en el programa?

##### Acciones en `estado_waitInON`

- **Evento `ENTRY`:**
  - Encender el píxel.
  - Iniciar el temporizador.

- **Evento `Timeout`:**
  - Transicionar al estado `estado_waitInOFF`.

---

##### Acciones en `estado_waitInOFF`

- **Evento `ENTRY`:**
  - Apagar el píxel.
  - Iniciar el temporizador.

- **Evento `Timeout`:**
  - Transicionar al estado `estado_waitInON`.

---
### Actividad 02
--
Código modificado :
```.py

from microbit import
import utime
from fsm import Timer, FSMTask, ENTRY
class Semaforo(FSMTask):
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        super().__init__()
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.add_timer("Timeout",self.timeInRed)
        self.transition_to(self.estado_waitInRed)
    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)
    def estado_waitInRed(self,ev):
         if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transition_to(self.estado_waitInGreen)      
    def estado_waitInGreen(self,ev):
         if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)
         if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)
    def estado_waitInYellow(self,ev):
         if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transition_to(self.estado_waitInRed)   
semaforo1 = Semaforo(0,0,2000,1000,500)
while True:
    #Input processing
    if button_a.was_pressed(): semaforo1.post_event("A")        
    semaforo1.update()
    utime.sleep_ms(20)
```
--
Diagrama UML:
<img width="815" height="494" alt="image" src="https://github.com/user-attachments/assets/c74a4002-f150-4fb4-a293-ba658ca8a4a8" />

Código:
```
@startuml


title Máquina de Estados - Clase Semaforo

state Semaforo {

    [*] --> WaitInRed

    WaitInRed : entry / clear()
    WaitInRed : entry / display.set_pixel(x, y, 9)
    WaitInRed : entry / myTimer.start(timeInRed)

    WaitInGreen : entry / clear()
    WaitInGreen : entry / display.set_pixel(x, y+2, 9)
    WaitInGreen : entry / myTimer.start(timeInGreen)

    WaitInYellow : entry / clear()
    WaitInYellow : entry / display.set_pixel(x, y+1, 9)
    WaitInYellow : entry / myTimer.start(timeInYellow)

    WaitInRed --> WaitInGreen : Timeout / display.set_pixel(x, y, 0)
    WaitInGreen --> WaitInYellow : Timeout / display.set_pixel(x, y+2, 0)
    WaitInGreen --> WaitInYellow : A / display.set_pixel(x, y+2, 0)
    WaitInYellow --> WaitInRed : Timeout / display.set_pixel(x, y+1, 0)
}

@enduml
```


## Bitácora de aplicación 

```.py
from microbit import *
import utime
import music


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


class Temporizador:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        self.myTimer = self.createTimer("Timeout", 1000)

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

        if ev == "B":
            if self.count < 25:
                self.count += 1
                display.show(FILL[self.count])

        if ev == "A":
            if self.count > 15:
                self.count -= 1
                display.show(FILL[self.count])

        if ev == "S":
            self.transicion_a(self.estado_armado)

   
    def estado_armado(self, ev):
        if ev == "ENTRY":
            self.myTimer.start(1000)

        if ev == "Timeout":
            self.count -= 1
            display.show(FILL[self.count])

            if self.count > 0:
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_explosion)

   
    def estado_explosion(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.DADADADUM)

        if ev == "A":
            self.count = 20
            self.transicion_a(self.estado_configuracion)



task = Temporizador()

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






