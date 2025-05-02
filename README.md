# LI-Microbit
## Matriz led:

Incluye tres animaciones distintas:

- Efecto “ripple” (ondas concéntricas)

- Lluvia de píxeles (efecto Matrix)

- Espiral de brillo (sube y baja intensidad por filas)

Puedes lanzar cada animación con un botón o gesto:

- Botón A → Ripple

- Botón B → Lluvia

- Sacudida → Espiral

### Codigo
```
from microbit import *
import random
import math

# — Parámetros comunes —
FPS = 50          # velocidad de refresco (ms)
MAX_BRI = 9       # máximo brillo

# — 1) RIPPLE ANIMATION — ondas concéntricas
def ripple():
    center = (2, 2)
    max_r = 4
    for r in range(max_r + 1):
        display.clear()
        bri = MAX_BRI - r*2 if MAX_BRI - r*2 > 0 else 1
        for x in range(5):
            for y in range(5):
                # distancia Manhattan al centro
                if abs(x-center[0]) + abs(y-center[1]) == r:
                    display.set_pixel(x, y, bri)
        sleep(FPS)
    # fade out
    for b in range(MAX_BRI, -1, -1):
        display.clear()
        for angle in range(0, 360, 30):
            # coordenadas circulares aproximadas
            rad = r = max_r
            x = int(center[0] + r * (math.cos(angle * math.pi/180)))
            y = int(center[1] + r * (math.sin(angle * math.pi/180)))
            if 0 <= x < 5 and 0 <= y < 5:
                display.set_pixel(x, y, b)
        sleep(FPS // 2)
    display.clear()

# — 2) MATRIX RAIN — lluvia de píxeles verticales
def matrix_rain(duration=100):
    drops = [random.randint(0,4) for _ in range(10)]
    for _ in range(duration):
        display.clear()
        # cada drop avanza
        for i, col in enumerate(drops):
            y = i % 5
            display.set_pixel(col, y, MAX_BRI)
            # baja intensidad en pixeles anteriores
            if y>0:
                display.set_pixel(col, y-1, MAX_BRI//3)
        # rotamos posiciones y cambiamos algunas columnas
        drops = [(d + 1) % 5 for d in drops]
        if random.random() < 0.1:
            drops[random.randint(0,9)] = random.randint(0,4)
        sleep(FPS)

# — 3) SPIRAL FADE — espiral de brillo creciente/decreciente
def spiral_fade(cycles=2):
    # orden de coordenadas en espiral desde (0,0) hacia dentro
    path = [
        (0,0),(1,0),(2,0),(3,0),(4,0),
        (4,1),(4,2),(4,3),(4,4),
        (3,4),(2,4),(1,4),(0,4),
        (0,3),(0,2),(0,1),
        (1,1),(2,1),(3,1),
        (3,2),(3,3),
        (2,3),(1,3),
        (1,2),(2,2)
    ]
    for _ in range(cycles):
        # sube
        for b in range(MAX_BRI + 1):
            display.clear()
            for x,y in path:
                display.set_pixel(x, y, b)
            sleep(FPS // 2)
        # baja
        for b in range(MAX_BRI, -1, -1):
            display.clear()
            for x,y in path:
                display.set_pixel(x, y, b)
            sleep(FPS // 2)
    display.clear()

# — Bucle principal —  
while True:
    if button_a.was_pressed():
        ripple()
    if button_b.was_pressed():
        matrix_rain()
    if accelerometer.was_gesture('shake'):
        spiral_fade()
    sleep(100)
```

## Sensores basicos

- Lee temperatura (°C)

- Lee luz ambiental (0–255)

- Lee acelerómetro (X/Y/Z)

- Muestra cada dato de forma clara y gráfica en la matriz 5×5

### Codigo
```
from microbit import *

def bar_graph(value, max_value, row):
    """
    Dibuja una barra horizontal en 'row' que represente
    value/max_value en 0–5 LEDs.
    """
    # Calcula cuántos LEDs encender (0–5)
    bars = int(min(value, max_value) * 5 // (max_value + 1))
    for x in range(5):
        display.set_pixel(x, row, 9 if x < bars else 0)

while True:
    if button_a.was_pressed():
        # 1) Temperatura
        temp = temperature()            # en °C (por lo general 0–40)
        display.scroll('T={}C'.format(temp), delay=100)
        display.clear()
        bar_graph(temp, max_value=40, row=4)  # fila 4 (abajo)
        sleep(1000)
        display.clear()

        # 2) Luz ambiental
        light = display.read_light_level()     # 0–255
        display.scroll('L={}'.format(light), delay=75)
        display.clear()
        bar_graph(light, max_value=255, row=0) # fila 0 (arriba)
        sleep(1000)
        display.clear()

        # 3) Acelerómetro: orientación simple
        x = accelerometer.get_x()   # valores aprox. -1024 a +1024
        y = accelerometer.get_y()
        # Elegimos la dirección predominante:
        if abs(x) > abs(y):
            if x > 200:
                display.show(Image.ARROW_W)
            elif x < -200:
                display.show(Image.ARROW_E)
            else:
                display.show(Image.ARROW_N)
        else:
            if y > 200:
                display.show(Image.ARROW_N)
            elif y < -200:
                display.show(Image.ARROW_S)
            else:
                display.show(Image.ARROW_N)
        sleep(1000)
        display.clear()

    sleep(100)

```


## Radio

1. Botón A: envía el mensaje "DATOS" y espera un "ACK" hasta 3 veces (1 segundo cada intento).

2. En cada intento verás en la matriz el número de intento (1, 2 o 3).

3. Si recibe el ACK dentro del tiempo, muestra 😊, si no, muestra 😞.

4. Al recibir cualquier otro mensaje (p. ej. "DATOS"), el receptor:

- Lo despliega con display.scroll()

- Envía "ACK" de vuelta

- Muestra ✓ brevemente.

### Codigo
```
from microbit import *
import radio

# — Configuración de radio (simula 900 MHz) —
radio.on()
radio.config(power=7, channel=19, queue=5)

def send_with_ack(msg, retries=3, timeout=1000):
    """
    Envía `msg` y espera un 'ACK'.  
    Reintenta hasta `retries` veces, cada vez esperando `timeout` ms.
    Devuelve True si recibe ACK, False si agota reintentos.
    """
    for attempt in range(1, retries+1):
        radio.send(msg)
        display.show(str(attempt))          # muestra número de intento
        start = running_time()
        while running_time() - start < timeout:
            resp = radio.receive()
            if resp == 'ACK':
                return True
        # si no llegó ACK, continúa al siguiente intento
    return False

while True:
    # — Emisor: botón A dispara el envío con handshake —
    if button_a.was_pressed():
        display.show(Image.ARROW_N)  
        ok = send_with_ack('DATOS', retries=3, timeout=1000)
        display.show(Image.HAPPY if ok else Image.SAD)
        sleep(800)
        display.clear()

    # — Receptor: si recibe algo distinto de 'ACK', lo muestra y responde ACK —
    incoming = radio.receive()
    if incoming and incoming != 'ACK':
        display.scroll(incoming, delay=100)
        radio.send('ACK')                # confirma recepción
        display.show(Image.YES)
        sleep(500)
        display.clear()

    sleep(50)

```
## Otros

- Muestra en la matriz un aviso al entrar en cada función.

- Reduce las iteraciones de arcoíris para que veas el cambio más rápido.

- Confirma el “shake” con un icono antes de disparar el efecto.

### Codigo
```
from microbit import *
from neopixel import NeoPixel
import music

# Configuración NeoPixel (pin0, 16 LEDs)
np = NeoPixel(pin0, 16)

def wheel(pos):
    if pos < 85:
        return (pos * 3, 255 - pos * 3, 0)
    elif pos < 170:
        pos -= 85
        return (255 - pos * 3, 0, pos * 3)
    else:
        pos -= 170
        return (0, pos * 3, 255 - pos * 3)

def rainbow_cycle(wait):
    # Depuración: avisar en la matriz
    display.scroll("RC", delay=80, wait=False)
    # Hacemos solo 4 pasos grandes en lugar de 255
    for j in range(0, 256, 64):
        for i in range(16):
            idx = (i * 256 // 16 + j) & 255
            np[i] = wheel(idx)
        np.show()
        sleep(wait)
    # Limpieza
    np.clear()
    np.show()
    display.clear()

def play_tune():
    display.scroll("MU", delay=100, wait=False)  # MU = Music
    tune = ['C4:4', 'E4:4', 'G4:4', 'C5:4', 'G4:4', 'E4:4', 'C4:8']
    music.play(tune)

while True:
    # Botón A → NeoPixel
    if button_a.was_pressed():
        display.show(Image.HEART)
        sleep(300)
        display.clear()
        rainbow_cycle(50)

    # Botón B → Buzzer
    if button_b.was_pressed():
        display.show(Image.MUSIC_CROTCHET)
        sleep(300)
        display.clear()
        play_tune()

    # Sacudida → ambos con depuración
    if accelerometer.was_gesture('shake'):
        display.show(Image.SURPRISED)
        sleep(300)
        display.clear()
        rainbow_cycle(30)
        play_tune()

    sleep(100)

```
