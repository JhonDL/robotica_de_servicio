---
title: "LASER MAPPING"
date: 2026-07-02
---

Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/ServiceRobots/laser_mapping/">Laser Mapping</a>
de Robotics Academy (JdeRobot).

El objetivo de la práctica es que un robot TurtleBot3 explore autónomamente un almacén desconocido y construya un mapa 2D del entorno usando un LIDAR de 360°. El robot dispone de odometría (`HAL.getPose3d`) y datos del laser (`HAL.getLaserData`), y se comanda con `HAL.setV` y `HAL.setW`. El mapa se visualiza en tiempo real mediante `WebGUI.setUserMap`.

La solución se organiza en dos bloques independientes: la construcción del mapa por ocupación probabilística y la navegación autónoma mediante una máquina de estados con patrón Lawn-Mower.

<pre class="mermaid">
stateDiagram-v2
    [*] --> GO_2_WALL
    GO_2_WALL --> ALIGN_TO_FOLLOW: obstáculo frontal, giro 90°
    ALIGN_TO_FOLLOW --> FOLLOW_WALL: rayos 45° simétricos
    FOLLOW_WALL --> GO_2_WALL: wall_break (pared termina)
    FOLLOW_WALL --> ALIGN: obstáculo frontal
    ALIGN --> FORWARD: alineado con pared
    FORWARD --> TURN_1: obstáculo frontal
    TURN_1 --> SIDE_STEP: giro completado
    SIDE_STEP --> TURN_2: distancia recorrida > FORWARD_TH
    TURN_2 --> FORWARD: giro completado (alterna dirección)
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

## Construcción del mapa

La función `polar2map` convierte cada medida del laser a coordenadas mundo. Para cada rayo se aplica la rotación por el yaw del robot y se traslada a la posición global, obteniendo el punto de impacto en coordenadas del mapa mediante `WebGUI.poseToMap`. Se distinguen dos casos: rayos con impacto real (`d < maxRange`) que generan puntos de obstáculo, y rayos sin impacto (`inf` o `d >= maxRange`) que se tratan como espacio libre hasta `maxRange`. Esto fue un descubrimiento clave: ignorar los rayos `inf` dejaba sin pintar la mayor parte del espacio libre.

El trazado de celdas libres entre el robot y cada punto de impacto se realiza con el algoritmo de Bresenham, que recorre todos los píxeles intermedios en coordenadas de mapa. La actualización del mapa usa una representación log-odds sobre un array numpy uint8 inicializado a 127 (desconocido): las celdas libres bajan su valor y las ocupadas lo suben, con guardas para evitar overflow y underflow. Los obstáculos se aplican después de los puntos libres en cada iteración para que prevalezcan frente a un rayo que pase por la misma celda.

Un problema encontrado fue el offset angular del laser: `minAngle = π/2`, lo que significa que el rayo de índice 0 apunta al frente del robot pero con ángulo `π/2` en el sistema del sensor. La corrección fue restar `laser.minAngle` al ángulo de cada rayo en `polar2map`.

## Navegación

La navegación sigue el patrón Lawn-Mower implementado como máquina de estados finitos. Antes de iniciar el barrido el robot busca una pared (`GO_2_WALL`) y se alinea en paralelo a ella (`ALIGN_TO_FOLLOW`) comparando los rayos laser a ±45° del lateral: cuando la diferencia es menor que un umbral, el robot está suficientemente paralelo. Durante el seguimiento (`FOLLOW_WALL`) un controlador PD mantiene la distancia lateral objetivo usando el rayo a 270° (derecha del robot).

El patrón de barrido alterna la dirección de giro en cada fila gracias a la variable `turn_right` que se invierte al completar cada ciclo `TURN_1 → SIDE_STEP → TURN_2`. Los giros usan un controlador PD sobre el yaw para alcanzar exactamente 90°, con normalización del ángulo para manejar el wraparound en ±π.

## Problemas afrontados

**Offset angular del laser.** El laser no empieza en 0° sino en π/2, lo que giraba todos los obstáculos 90° en el mapa. Se diagnosticó comparando la dirección de avance del robot con el movimiento del píxel correspondiente en `WebGUI.poseToMap`, y se corrigió restando `laser.minAngle` en la conversión polar-cartesiana.

**Rayos sin impacto ignorados.** Inicialmente se descartaban todos los rayos `inf`, lo que dejaba sin marcar como libre la mayor parte del espacio navegable. La solución fue tratarlos como rayos libres hasta `maxRange` y añadirlos como `free_endpoints`, generando un mapa mucho más completo.

**Wraparound del yaw.** Al comparar ángulos cerca de ±π, la diferencia podía dar valores incorrectos. Se resolvió normalizando `yaw_diff` al rango (-π, π] sumando o restando 2π hasta entrar en rango.

<iframe width="560" height="315" src="https://www.youtube.com/embed/lUzLy-D1W8E" frameborder="0" allowfullscreen></iframe>

## Limitaciones y mejoras futuras

Los umbrales de distancia (`FRONT_TH`, `RIGHT_TH`, `FORWARD_TH`) están ajustados para el mapa concreto del ejercicio. Un sistema de detección de pasillos estrechos evitaría que el robot intentara hacer Lawn-Mower en espacios entre estanterías donde no caben los giros. La condición de parada basada en porcentaje de celdas desconocidas (`np.sum(global_map == 127) / global_map.size`) está implementada pero el umbral óptimo depende del mapa. Sustituir el controlador P del seguimiento lateral por un PID completo reduciría el error en estado estacionario.

---

Referencias:
  <a href="https://jderobot.github.io/RoboticsAcademy/exercises/ServiceRobots/laser_mapping/">Laser Mapping - Robotics Academy</a>
