# Unidad 3

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
Programa p5
```.js
// ===============================
// VARIABLES DE ESTADO (FSM)
// ===============================

let estado;
const CONFIG = "CONFIG";
const RUNNING = "RUNNING";
const TIMEOUT = "TIMEOUT";

let contador = 20;
let maxValor = 25;
let minValor = 15;

let ultimoTiempo = 0;
let intervalo = 1000;

let pausado = false;
let secuencia = [];

// ===============================
// SERIAL (p5.serialport style)
// ===============================

let port;
let botonConectar;
let conectado = false;

// ===============================
// SETUP
// ===============================

function setup() {
  createCanvas(400, 400);
  textAlign(CENTER, CENTER);
  textSize(64);

  estado = CONFIG;

  // BOTÓN DE CONEXIÓN
  botonConectar = createButton("Conectar micro:bit");
  botonConectar.position(10, 10);
  botonConectar.mousePressed(connectBtnClick);

  // Inicializar puerto serial (p5.serialcontrol)
  port = createSerial(); // requiere p5.serialport.js
}

// ===============================
// DRAW
// ===============================

function draw() {
  background(30);

  // Indicador de conexión
  textSize(14);
  fill(conectado ? "lime" : "red");
  text(conectado ? "Serial: Conectado" : "Serial: No conectado", width - 100, 20);

  textSize(64);

  if (estado == CONFIG) {
    fill(0, 200, 255);
    text(contador, width / 2, height / 2);
  }

  if (estado == RUNNING) {
    fill(0, 255, 100);
    text(contador, width / 2, height / 2);

    if (!pausado && millis() - ultimoTiempo > intervalo) {
      ultimoTiempo = millis();
      contador--;

      if (contador <= 0) {
        estado = TIMEOUT;
      }
    }

    if (pausado) {
      textSize(20);
      text("PAUSADO", width / 2, height - 40);
      textSize(64);
    }
  }

  if (estado == TIMEOUT) {
    fill(255, 0, 0);
    text("💀", width / 2, height / 2);
  }

  // ==========================
  // LECTURA DE BOTONES MICRO:BIT
  // ==========================
  if (port.opened() && port.availableBytes() > 0) {
    let dataRx = port.read(1); // lee un byte
    if (dataRx) {
      let v = dataRx.trim();
      if (v == "A" || v == "B" || v == "S") {
        manejarEvento(v);
      }
    }
  }
}

// ===============================
// BOTÓN DE CONEXIÓN
// ===============================

function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
    conectado = true;
  } else {
    port.close();
    conectado = false;
  }
}

// ===============================
// MANEJO DE EVENTOS
// ===============================

function manejarEvento(ev) {

  if (estado == CONFIG) {

    if (ev == "A" && contador > minValor) {
      contador--;
    }

    if (ev == "B" && contador < maxValor) {
      contador++;
    }

    if (ev == "S") {
      estado = RUNNING;
      pausado = false;
      secuencia = [];
      ultimoTiempo = millis();
    }
  }

  else if (estado === RUNNING) {

    if (ev == "A" || ev == "B") {

      secuencia.push(ev);
      if (secuencia.length > 3) {
        secuencia.shift();
      }

      if (secuencia.join("") == "ABA") {
        estado = CONFIG;
        contador = 20;
        pausado = false;
        secuencia = [];
        return;
      }
    }

    if (ev == "A") {
      pausado = !pausado;
    }
  }

  else if (estado == TIMEOUT) {

    if (ev == "A") {
      estado = CONFIG;
      contador = 20;
    }
  }
}

// ===============================
// TECLADO
// ===============================

function keyPressed() {
  if (key == 'a' || key == 'A') manejarEvento("A");
  if (key == 'b' || key == 'B') manejarEvento("B");
  if (key == 's' || key == 'S') manejarEvento("S");
}
```
Programa micro:bit
main
```.py
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

uart.init(baudrate=115200)


class Temporizador(FSMTask):

    def __init__(self):
        super().__init__()
        self.counter = 20
        self.myTimer = self.add_timer("Timeout", 1000)

        # NUEVAS VARIABLES
        self.pausado = False
        self.secuencia = []

        self.transition_to(self.estado_config)

    # ==================================================
    # ESTADO CONFIGURACIÓN
    # ==================================================
    def estado_config(self, ev):

        if ev == ENTRY:
            self.counter = 20
            self.pausado = False
            self.secuencia = []
            display.show(FILL[self.counter])

        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])

        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])

        if ev == "S":
            self.transition_to(self.estado_armed)

    # ==================================================
    # ESTADO CORRIENDO
    # ==================================================
    def estado_armed(self, ev):

        if ev == ENTRY:
            self.pausado = False
            self.secuencia = []
            self.myTimer.start()

        # ----------------------------------------------
        # CAPTURA DE BOTONES (para pausa y secuencia)
        # ----------------------------------------------
        if ev in ["A", "B"]:
            self.secuencia.append(ev)

            # Mantener solo los últimos 3 eventos
            if len(self.secuencia) > 3:
                self.secuencia.pop(0)

            # Detectar secuencia A-B-A
            if self.secuencia == ["A", "B", "A"]:
                self.myTimer.stop()
                self.transition_to(self.estado_config)
                return

        # ----------------------------------------------
        # PAUSA / REANUDAR con botón A
        # ----------------------------------------------
        if ev == "A":
            if not self.pausado:
                self.myTimer.stop()
                self.pausado = True
            else:
                self.myTimer.start()
                self.pausado = False

        # ----------------------------------------------
        # CONTEO
        # ----------------------------------------------
        if ev == "Timeout" and not self.pausado:

            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])

                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()

    # ==================================================
    # ESTADO TIMEOUT
    # ==================================================
    def estado_timeout(self, ev):

        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)

        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)


# ======================================================
# LOOP PRINCIPAL
# ======================================================

temporizador = Temporizador()

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")
        uart.write('A')
        

    if button_b.was_pressed():
        temporizador.post_event("B")
        uart.write('B')

    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")
        uart.write('S')

    temporizador.update()
    utime.sleep_ms(20)
```
fms:
```.py
import utime

ENTRY = "ENTRY"
EXIT  = "EXIT"

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
        if self.active and utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
            self.active = False
            self.owner.post_event(self.event)


class FSMTask:
    def __init__(self):
        self._q = []
        self._timers = []
        self._state = None

    def post_event(self, ev):
        self._q.append(ev)

    def add_timer(self, event, duration):
        t = Timer(self, event, duration)
        self._timers.append(t)
        return t

    def transition_to(self, new_state):
        if self._state:
            self._state(EXIT)
        self._state = new_state
        self._state(ENTRY)

    def update(self):
        for t in self._timers:
            t.update()
        while self._q:
            ev = self._q.pop(0)
            if self._state:
                self._state(ev)
```
utils
```.py
from microbit import Image

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
```
## Bitácora de reflexión
