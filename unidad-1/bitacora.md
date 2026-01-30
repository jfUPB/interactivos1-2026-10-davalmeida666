# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 1
#### ¿Qué es un sistema físico interactivo?

Un sistema físico interactivo es una herramienta que permite extraer información del mundo real, analizar su comportamiento, y trasladarlo nuevamente, de manera virtual. Este proceso se logra mediante sistemas tanto visuales, de sensación ( por ejemplo máquinas de humo), o programas que se acoplen junto a objetos tangibles, como es el caso de los simuladores de videojuegos.

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

Como Ingeniero en Diseño de Entretenimiento Digital, el conocimiento en sistemas interactivos me permite conectar con las audiencias aún más que los 2 sentidos principales: vista y oído. Con el uso de estos mecanismos y programas, el espectador conecta con el momento, con el tacto, el olor, y sobretodo, conecta con una realidad mixta entre lo físico y lo virtual. De tal manera que, en mi perfil profesional, puedo brindarle a las personas una experiencia única que despierte todos sus sentidos y los haga vivir momentos inolvidables.

### Actividad 2
#### ¿Qué es el diseño/arte generativo?

El diseño generativo es la creación, no de elementos gráficos estáticos, sino de programas, que, insertadas ciertas condiciones, pueden llegar a producir una innumerable cantidad de diseños. Es decir. En vez de que el artista cree una sola obra, crea una serie de instrucciones, que reaccionan a comandos específicos, que generan variaciones en los resultados.

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

El diseño generativo me permite, como artista, diseñador e ingeniero en Diseño de Entretenimiento Digital generar elementos gráficos únicos. Estos pueden usarse, para darle un aspecto exclusivo a los diseños, permitiendo entonces una especie de interactividad entre el usuario y el diseño. Por ejemplo, en una aplicaión, dependiendo del usuario, cada menú puede tener una paleta única, o en un videojuego sus partículas.

### Actividad 3
Se realizó un sistema interactivo simple, que involucraba conexión entre los botones del microbit y una aplicación interactiva en la pantalla. El sistema consistía en:

#### Presiona el botón A del microbit: 
cambiaba el color del círculo de la aplicación a rojo, y mostabra A
#### Presiona el botón B del microbit: 
cambiaba el color del círculo de la aplicación a amarillo y mostraba B
#### Sacude el micro:bit. ¿Qué pasa?: 
cambiaba el color del círculo de la aplicación a verde y mostraba C
#### Presiona el botón Send Love. ¿Qué pasa? 
Si se presionaba el botón de la aplicación, en la pantalla del microbit aparecía la imagen de un corazón y una carita feliz.

### Actividad 4
#### Programa micro: bit:
```
from microbit import *

uart.init(baudrate=115200)

while True:

    if button_a.is_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)
```
#### Programa p5.js
``` js.
  let port;
  let connectBtn;
  let connectionInitialized = false;

  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
  }

  function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }

```
#### ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.

El programa no funciona correctamente con was_pressed() porque esta función llama a cada frame, pero solamente considera el botón como un click, por lo que al cambiar de frame, vuelve a su color original. Con is_pressed(), cada frame el programa va a considerar si el botón sigue presionado, permitiendo que el color siga en su variación hasta que el botón se deje de presionar.

## Bitácora de aplicación 
### Actividad 5
#### index.
```
<script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
```
#### Programa p5.js.
``` js.
let port;
  let connectBtn;
  let connectionInitialized = false;
  let x;


  function setup() {
    createCanvas(400, 400);
    
    x = width/2;
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    

   
  }

  function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
       x -= 20;
     
      } else if (dataRx == "N") {
        x += 20;
      }
    }

    ellipse(CENTER);
    ellipse(x, height / 2, 100, 100);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }
```
#### Programa micro:bit
```
from microbit import *

uart.init(baudrate=115200)


while True:

    if button_a.is_pressed():
        uart.write('A')
    if button_b.is_pressed():
        uart.write('N')

    sleep(100)
```
#### Explicación:
En p5.js , se declaran variables, luego se prepara el setup con tamaño de canvas, se define x, y las funcionalidades de cada botón., y su posición. Seguido a eso, se llama a la función draw en cada frame. Se lee la entrada del microbit, si es A, la posición x se desplaza 20 unidades a la izquierda, y si se lee N, lo hace hacia la derecha. Luego,  dibuja la elipse en cada frame, ajustando la posición en x dada.

## Bitácora de reflexión

En p5.js , se declaran las variables necesarias para poner el código en funcionamiento, luego se prepara el setup con tamaño de canvas, para definir el espacio donde apareceran las cosas en la pantalla, y las funcionalidades de cada botón,a l igual que su posición, y . Seguido a eso, se llama a la función draw en cada frame, la cual dibujara en cada frame un cuadrado en el centro de pantalla con dimensiones de 50 por 50. Posteriormente, se lee la entrada del microbit, si es A, el color del cuadrado cambia a rojo, y si la señal es N cambia a verde. CAda rame, el programa revisa si la condición de botón apretado se cumple. En este caso, el programa de microbit únicamente envía señal del botón A, por lo tanto, el programa solo cambiará el color si esta tecla está oprimida. Principalmente El programa no funciona correctamente con was_pressed() porque esta función llama a cada frame, pero solamente considera el botón como un click, por lo que al cambiar de frame, vuelve a su color original. Con is_pressed(), cada frame el programa va a considerar si el botón sigue presionado, permitiendo que el color siga en su variación hasta que el botón se deje de presionar.










