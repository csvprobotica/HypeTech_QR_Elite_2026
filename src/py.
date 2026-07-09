from pybricks.hubs import PrimeHub
from pybricks.pupdevices import Motor, ColorSensor, UltrasonicSensor, ForceSensor
from pybricks.parameters import Button, Color, Direction, Port, Side, Stop
from pybricks.robotics import DriveBase
from pybricks.tools import wait, StopWatch

hub = PrimeHub()

"""
=========================================================
  RETO "FUTUROS INGENIEROS" - Seguimiento de paredes con
  conteo de esquinas (traducido del programa de Mindstorms),
  ahora usando un SENSOR FRONTAL real en el puerto F.

    A = Sensor ultrasonico IZQUIERDO
    B = Sensor ultrasonico DERECHO
    C = Motor grande - TRACCION
    D = Motor mediano - DIRECCION
    E = Sensor de color (no se usa en este programa)
    F = Sensor ultrasonico FRONTAL (nuevo)

  Presiona el boton DERECHO del hub para el modo HORARIO
  (cuenta esquinas y gira a la derecha).
  Presiona el boton IZQUIERDO del hub para el modo
  ANTIHORARIO (cuenta esquinas y gira a la izquierda).

  Logica (fiel al programa original, ahora con sensor frontal
  real en vez de aproximarlo con el sensor de color):
    - Guarda la distancia frontal inicial (sensor F) al arrancar.
    - Avanza recto por defecto.
    - Si el lado correspondiente se abre mucho (pasillo/esquina)
      -> cuenta una esquina y gira fuerte hacia ese lado.
    - Si una pared lateral queda muy cerca -> giro suave para
      alejarse (no cuenta como esquina).
    - Si el sensor FRONTAL detecta algo muy cerca -> frena de
      emergencia (proteccion nueva, el programa original no
      tenia esto porque no tenia un sensor realmente frontal).
    - Al llegar a TOPE_ESQUINAS: se detiene, espera, avanza un
      poco mas y para definitivamente cuando el sensor frontal
      vuelve a marcar menos que la distancia inicial guardada
      (igual que el programa original).
=========================================================
"""

from pybricks.hubs import PrimeHub
from pybricks.pupdevices import Motor, UltrasonicSensor
from pybricks.parameters import Port, Stop, Button
from pybricks.tools import wait, StopWatch

hub = PrimeHub()

sensor_izq = UltrasonicSensor(Port.A)
sensor_der = UltrasonicSensor(Port.B)
motor_traccion = Motor(Port.C)
motor_direccion = Motor(Port.D)
sensor_frontal = UltrasonicSensor(Port.F)

# ---------------- PARAMETROS AJUSTABLES ----------------
VELOCIDAD_TRACCION = 600

VELOCIDAD_GIRO_FUERTE = 400
VELOCIDAD_GIRO_SUAVE = 280
ANGULO_GIRO_FUERTE = 45
ANGULO_GIRO_SUAVE = 20

# Umbrales laterales (identicos al programa original: 198cm, 16cm, 8cm)
DIST_ESQUINA_MM = 1980
DIST_CERCA_MM = 160
DIST_MUY_CERCA_MM = 80

# NUEVO: umbral de emergencia frontal (el original no tenia esto)
DIST_FRONTAL_CRITICA_MM = 100
TIEMPO_RETROCESO_MS = 400

TOPE_ESQUINAS = 12

# NUEVO: ensanche antes de girar en la esquina (wide turn), simetrico
# para horario y antihorario, en vez de depender de que ocurra "por
# casualidad" segun la geometria de la pista de ese lado.
ANGULO_ENSANCHE = 15         # grados que se aleja del lado contrario antes de girar
TIEMPO_ENSANCHE_MS = 150     # cuanto se mantiene el ensanche antes del giro fuerte

TAMANO_FILTRO = 3   # promedio de lecturas para suavizar ruido de los ultrasonicos
# ---------------------------------------------------------

print("Bateria:", hub.battery.voltage(), "mV /", hub.battery.current(), "mA")

historial_izq = []
historial_der = []
historial_frontal = []


def centrar_direccion(orden_invertido=False):
    """Encuentra los topes fisicos de la direccion y la deja centrada.
    Si orden_invertido=False: busca primero el tope izquierdo y despues
    el derecho, terminando el movimiento final desde la derecha hacia
    el centro.
    Si orden_invertido=True: busca primero el tope derecho y despues
    el izquierdo, terminando el movimiento final desde la izquierda
    hacia el centro (justo al reves)."""
    if not orden_invertido:
        motor_direccion.run_until_stalled(-200, duty_limit=50)
        tope_izq = motor_direccion.angle()
        motor_direccion.run_until_stalled(200, duty_limit=50)
        tope_der = motor_direccion.angle()
    else:
        motor_direccion.run_until_stalled(200, duty_limit=50)
        tope_der = motor_direccion.angle()
        motor_direccion.run_until_stalled(-200, duty_limit=50)
        tope_izq = motor_direccion.angle()

    centro = (tope_izq + tope_der) / 2
    motor_direccion.run_target(200, centro, then=Stop.HOLD)
    motor_direccion.reset_angle(0)
    print("Direccion centrada. Tope izq:", tope_izq, "Tope der:", tope_der)


def leer_distancias_filtradas():
    """Promedia las ultimas TAMANO_FILTRO lecturas de cada sensor
    (izq, der, frontal) para suavizar el ruido normal del ultrasonico."""
    global historial_izq, historial_der, historial_frontal

    historial_izq.append(sensor_izq.distance())
    historial_der.append(sensor_der.distance())
    historial_frontal.append(sensor_frontal.distance())

    if len(historial_izq) > TAMANO_FILTRO:
        historial_izq.pop(0)
    if len(historial_der) > TAMANO_FILTRO:
        historial_der.pop(0)
    if len(historial_frontal) > TAMANO_FILTRO:
        historial_frontal.pop(0)

    return (sum(historial_izq) / len(historial_izq),
            sum(historial_der) / len(historial_der),
            sum(historial_frontal) / len(historial_frontal))


def dirigir(angulo, velocidad=VELOCIDAD_GIRO_FUERTE):
    """Mueve la direccion sin bloquear el resto del programa
    (equivalente al 'fork' del proyecto original)."""
    motor_direccion.run_target(velocidad, angulo, then=Stop.HOLD, wait=False)


def frenar_emergencia_frontal(dist_frontal):
    print("Obstaculo frontal critico:", dist_frontal, "mm")
    motor_traccion.hold()
    wait(150)
    motor_traccion.run_time(-300, TIEMPO_RETROCESO_MS)
    wait(200)


def secuencia_final(distancia_frontal_inicial):
    """Al llegar al tope de esquinas: se detiene, espera, avanza un
    poco mas, y para definitivamente cuando el sensor frontal vuelve
    a marcar menos que la distancia inicial (igual que el original)."""
    motor_traccion.hold()
    wait(1000)
    motor_traccion.run(VELOCIDAD_TRACCION * 0.5)
    wait(500)

    while True:
        _, _, dist_frontal = leer_distancias_filtradas()
        if dist_frontal < distancia_frontal_inicial:
            break
        wait(10)

    motor_traccion.hold()
    dirigir(0, VELOCIDAD_GIRO_FUERTE)
    hub.speaker.beep()
    print("Programa finalizado: se completaron", TOPE_ESQUINAS, "esquinas.")


def correr_ruta(sensor_esquina_derecha):
    """sensor_esquina_derecha = True  -> modo HORARIO
       sensor_esquina_derecha = False -> modo ANTIHORARIO"""
    contador = 0
    esquina_abierta_anterior = False

    # Para HORARIO invertimos el orden de busqueda de topes, para que el
    # movimiento final de centrado venga del lado derecho hacia el medio
    # (en vez de izquierda hacia el medio, como pasaba antes en los dos modos).
    centrar_direccion(orden_invertido=sensor_esquina_derecha)
    wait(500)

    # Distancia frontal inicial, igual que en el programa original
    _, _, distancia_frontal_inicial = leer_distancias_filtradas()
    print("Distancia frontal inicial guardada:", distancia_frontal_inicial, "mm")

    motor_traccion.run(VELOCIDAD_TRACCION)
    wait(100)

    while True:
        dist_izq, dist_der, dist_frontal = leer_distancias_filtradas()

        # Proteccion nueva: frenar si hay algo justo enfrente muy cerca
        if dist_frontal < DIST_FRONTAL_CRITICA_MM:
            frenar_emergencia_frontal(dist_frontal)
            motor_traccion.run(VELOCIDAD_TRACCION)
            continue

        if sensor_esquina_derecha:
            # HORARIO: el sensor derecho hace esquina (>198cm) y muy-cerca (<8cm);
            # el izquierdo solo hace el chequeo de cerca (<16cm). Igual que el original.
            sensor_principal = dist_der
            sensor_secundario = dist_izq
            signo = 1
        else:
            # ANTIHORARIO: se intercambian los roles (igual que el original):
            # el sensor izquierdo hace esquina (>198cm) y muy-cerca (<8cm);
            # el derecho solo hace el chequeo de cerca (<16cm).
            sensor_principal = dist_izq
            sensor_secundario = dist_der
            signo = -1

        esquina_abierta = sensor_principal > DIST_ESQUINA_MM

        if esquina_abierta and not esquina_abierta_anterior:
            contador += 1
            print("Esquina", contador, "de", TOPE_ESQUINAS)

            # Ensanche: se aleja levemente hacia el lado contrario antes
            # de tomar el giro fuerte, para entrar con mejor angulo
            # (esto es lo mismo que pasaba "gratis" en antihorario).
            dirigir(-signo * ANGULO_ENSANCHE, VELOCIDAD_GIRO_SUAVE)
            wait(TIEMPO_ENSANCHE_MS)

            dirigir(signo * ANGULO_GIRO_FUERTE, VELOCIDAD_GIRO_FUERTE)

            if contador >= TOPE_ESQUINAS:
                secuencia_final(distancia_frontal_inicial)
                return
        elif sensor_secundario < DIST_CERCA_MM:
            dirigir(signo * ANGULO_GIRO_SUAVE, VELOCIDAD_GIRO_SUAVE)
        elif sensor_principal < DIST_MUY_CERCA_MM:
            dirigir(-signo * ANGULO_GIRO_SUAVE, VELOCIDAD_GIRO_SUAVE)
        elif not esquina_abierta:
            dirigir(0, VELOCIDAD_GIRO_FUERTE)

        esquina_abierta_anterior = esquina_abierta
        wait(10)


# =========================================================
# PROGRAMA PRINCIPAL: espera boton derecho o izquierdo
# =========================================================
print("Presiona el boton DERECHO (horario) o IZQUIERDO (antihorario) del hub...")

while True:
    presionados = hub.buttons.pressed()
    if Button.RIGHT in presionados:
        hub.speaker.beep()
        correr_ruta(sensor_esquina_derecha=True)
        break
    elif Button.LEFT in presionados:
        hub.speaker.beep()
        correr_ruta(sensor_esquina_derecha=False)
        break
    wait(20)
