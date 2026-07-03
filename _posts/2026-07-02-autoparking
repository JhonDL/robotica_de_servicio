---
title: "AUTOPARKING"
date: 2026-07-02
---
Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/autoparking/">Autoparking </a>
de Robotics Academy (JdeRobot).

El objetivo de la práctica es que un vehículo Prius recorra una calle, localice una plaza libre entre los coches aparcados a su derecha y ejecute la maniobra de aparcamiento en linea. Se dispone de tres láseres (frontal, derecho y trasero) y de la odometría del vehículo, y se comanda por velocidad lineal y angular con `HAL.setV` y `HAL.setW`.

La solución se organiza como una máquina de estados que late en el bucle de control: cada iteración lee los sensores, evalúa el estado actual, comanda velocidades no bloqueantes y decide si transita. Una decisión de diseño transversal es que la orientación de la fila de coches (`parking_angle`) se calcula una sola vez y se guarda en el marco del mapa, de modo que todos los estados posteriores miden su error de rumbo contra esa referencia absoluta y sus condiciones de parada son fiables.

<pre class="mermaid">
stateDiagram-v2
    [*] --> SEARCHING
    SEARCHING --> APPROACHING: coche detectado a la derecha
    APPROACHING --> ALLIGNING: distancia lateral/frontal < 2.4 m
    ALLIGNING --> SCANNING: rumbo alineado con parking_angle
    SCANNING --> PREPARKING_BW: hueco entre 5 y 8 m
    SCANNING --> PREPARKING_FW: hueco > 8 m
    PREPARKING_BW --> PARKING_BW_TURN_1: adelantado 3.4 m
    PARKING_BW_TURN_1 --> PARKING_BW_TURN_2: desviacion ~40 / freno trasero
    PARKING_BW_TURN_2 --> CENTER: rumbo recuperado / freno trasero
    PREPARKING_FW --> PARKING_FW_TURN_1: reaparece coche a la derecha
    PARKING_FW_TURN_1 --> PARKING_FW_TURN_2: desviacion ~40 / freno frontal
    PARKING_FW_TURN_2 --> CENTER: rumbo recuperado / freno frontal
    CENTER --> finished: centrado y alineado
    finished --> [*]
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

Toda la percepción se apoya en los láseres y en la odometría. La función `laser_to_cartesian` convierte las 180 medidas polares del láser a coordenadas cartesianas en el marco del sensor, interpolando el paso angular entre `minAngle` y `maxAngle` y descartando las medidas fuera de rango. Sobre esos puntos, `agrupar_puntos` aprovecha que vienen ordenados por ángulo para separar objetos: dos puntos consecutivos alejados más de un umbral marcan el inicio de otro cluster. Cada cluster se reduce a una caja rectangular con `caja`, que en lugar del mínimo/máximo absoluto (sensible a un único punto de ruido) promedia los puntos más extremos en cada dirección y sitúa el centro en la media de todos ellos.

Para las decisiones de navegación se usan dos medidas complementarias. `min_center` devuelve la distancia mínima en una ventana central del escaneo, es decir, la distancia perpendicular real hacia delante, atrás o al lado; `dist_cajas` devuelve la distancia al punto más cercano de cualquier cluster y sirve para saber si hay coches a un lado, no para medir el vacío. La orientación de la fila se obtiene con `orientacion_fila`, que ajusta una recta a los puntos laterales y devuelve su ángulo.

La maniobra se descompone en fases. El vehículo avanza recto (`SEARCHING`) hasta detectar coches a la derecha, se aproxima a la fila con un ligero giro (`APPROACHING`), y fija la orientación de la plaza y se alinea con ella mediante un control proporcional con saturación (`ALLIGNING`). Después recorre la fila midiendo el hueco lateral por odometría (`SCANNING`): marca la posición donde empieza el hueco e integra el desplazamiento. Si el hueco alcanza los 8 m se compromete de inmediato como plaza grande y entra en la rama de marcha adelante; si al cerrarse medía entre 5 y 8 m entra en la de marcha atrás; si medía menos de 5 m lo descarta y sigue escaneando.

La rama de marcha atrás se adelanta hasta quedar a la altura del coche delantero (`PREPARKING_BW`), gira el volante a tope hacia el hueco reverseando hasta desviarse unos 40 grados del rumbo de parking (`PARKING_BW_TURN_1`) y contragira para enderezarse dentro del hueco (`PARKING_BW_TURN_2`). La rama de marcha adelante es simétrica pero avanzando de morro, con los frenos de seguridad mirando la distancia frontal. Ambas confluyen en `CENTER`, que corrige orientación y centra el vehículo a la vez: elige el sentido de avance hacia el hueco mayor y solo invierte la marcha al acercarse a un extremo, con una histéresis que evita el temblor en el punto de equilibrio. Cuando el error de rumbo y el balance entre distancia frontal y trasera entran en tolerancia, el vehículo se detiene (`FINISHED`).

<iframe width="560" height="315" src="https://www.youtube.com/embed/T_vbDtGF0JU" frameborder="0" allowfullscreen></iframe>

## Problemas afrontados

Durante el desarrollo aparecieron varios problemas cuya resolución fue la parte más instructiva de la práctica. En casi todos, la clave fue no fiarse de lo asumido y verificarlo empíricamente sobre el simulador.

**Elección del sensor para detectar el hueco.** En un primer momento se usó `dist_cajas` para saber si había coche al lado, pero devuelve el punto más cercano de cualquier cluster: en mitad de un hueco entrega la esquina del coche de delante o de detrás, no el vacío lateral. Se cambió a `min_center` sobre el láser derecho, que mide la distancia perpendicular real: en el hueco lee la calle de enfrente y junto a un coche lee la carrocería, separando limpiamente ambos casos.

**Medida del hueco por odometría.** Medir longitudes con el láser directamente resultaba frágil. Se optó por marcar la posición donde empieza el hueco e integrar el desplazamiento recorrido. Esto permite además la lógica de seguir avanzando para comprobar si el hueco supera los 8 m antes de decidir la rama, sin comprometerse prematuramente.

**Entrada torcida en el centrado.** Se puede llegar a `CENTER` desde un freno de seguridad por proximidad trasera, es decir, mal orientado. La solución fue un único controlador que corrige rumbo y centra simultáneamente: cada pasada, adelante o atrás con el signo de volante adecuado, reduce el error de rumbo mientras el balance entre distancia frontal y trasera converge a cero en paralelo. Para evitar el vaivén nervioso que producía invertir el sentido en cuanto el balance cruzaba cero, se añadió una histéresis con memoria del último sentido: el coche solo cambia de marcha al acercarse mucho a un extremo, haciendo pasadas completas que además enderezan mejor el rumbo. Esto ayuda a afrontar las limitaciones de movimiento de un vehículo Ackermann

## Limitaciones y mejora futura

el robot se basa en entornos ideales y aparcamiento en linea, otro tipo de entornos mas complejos o otro tipo de ordenación de coches daría errores.
---

Referencias:
  <a href="https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/autoparking/"> Autoparking - Robotics Academy </a>
