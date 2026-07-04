---
title: "MARKER BASED VISUAL LOCALIZATION"
date: 2026-07-02
---
Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/Localization/marker_visual_loc/">Marker Based Visual Loc </a>
de Robotics Academy (JdeRobot).

El objetivo de la práctica es que un robot equipado con una cámara y odometría se autolocalice de forma fiable en un entorno doméstico donde hay repartidas balizas visuales tipo AprilTag. Para ello el robot debe deambular por el entorno visitando varias zonas y, cada vez que detecta una baliza, estimar su posición y orientación absolutas mediante el algoritmo Perspective-n-Point (PnP), combinando la pose relativa cámara-baliza con la posición conocida de esa baliza en el mapa. Cuando no ve ninguna baliza, debe mantener su estimación integrando el incremento de odometría desde la última vez que vio una. Se comanda por velocidad lineal y angular con `HAL.setV` y `HAL.setW`, y se dispone de láser, cámara y odometría.

La solución se organiza en dos subsistemas que conviven en el mismo bucle de control pero son lógicamente independientes: un autómata reactivo de exploración que hace deambular al robot, y un módulo de localización que decide cómo estimar la pose según cuántas balizas ve. Esta separación fue una decisión de diseño deliberada: la navegación no necesita saber nada de la localización, y la localización no depende de cómo se mueva el robot. La pieza central del ejercicio es la cadena de transformadas homogéneas que convierte lo que ve la cámara en una posición absoluta en el mapa, y a resolverla correctamente se dedicó la mayor parte del esfuerzo.

<pre class="mermaid">
stateDiagram-v2
    [*] --> FORWARD
    FORWARD --> TURN: obstáculo frontal (láser < umbral)
    TURN --> FORWARD: giro aleatorio completado

    state "Localización (cada iteración)" as LOC {
        [*] --> VE_BALIZA
        VE_BALIZA --> PnP: >= 1 baliza detectada
        PnP --> FIX: pose absoluta + guarda ancla odometría
        VE_BALIZA --> DEAD_RECKONING: 0 balizas
        DEAD_RECKONING --> INCREMENTO: compone incremento odom sobre ancla
    }
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

## Exploración

La navegación se resuelve con un autómata reactivo de dos estados tipo choca-gira que late en el bucle de control. En `FORWARD` el robot avanza en línea recta mientras vigila un cono frontal del láser; la función `front_empty` recorre una ventana de ±10 grados alrededor del frente y devuelve falso si alguna medida cae por debajo del umbral de seguridad. Al detectar un obstáculo transita a `TURN`, donde gira sobre sí mismo durante un número aleatorio de iteraciones (acotado por `MAX_SECONDS_TURN`) antes de volver a avanzar. El giro aleatorio es lo que garantiza que el robot deambule y visite distintas zonas en lugar de quedar atrapado en un patrón repetitivo, cumpliendo el requisito de la parte de exploración del enunciado.

Un detalle que hubo que verificar empíricamente fue qué índice del array de láser corresponde al frente del robot. La documentación sugiere que el índice central es el frontal, pero un `print` de las distancias reales confirmó que en este entorno el índice cero apuntaba al frente, de modo que el cono de vigilancia se centró ahí. No fiarse de lo asumido y comprobarlo sobre el simulador ahorró un fallo silencioso en el que el robot habría frenado mirando a un lateral.

## Localización por PnP

Toda la localización visual se apoya en la cadena de transformadas. Cada iteración se pasa la imagen de la cámara por el detector de AprilTags (`pyapriltags`, familia `tag36h11`), que devuelve una lista de detecciones con las esquinas en píxeles y el identificador de cada baliza. Cuando hay varias balizas visibles, se elige la más cercana quedándose con la de mayor área en imagen, calculada sobre las cuatro esquinas con `cv2.contourArea`; una baliza más próxima se proyecta más grande, de modo que el área es un buen sustituto de la distancia.

Sobre la baliza elegida se llama a `cv2.solvePnP`, que a partir de las cuatro esquinas 3D de la baliza en su propio marco (`obj_pts`, un cuadrado de 0.24 m centrado en el origen) y sus correspondientes píxeles devuelve la rotación y traslación de la baliza respecto a la cámara. Esta pose relativa se ensambla en una matriz homogénea 4×4 (`T_C_T`) convirtiendo antes el vector de rotación de formato Rodrigues a matriz con `cv2.Rodrigues`. La matriz de intrínsecos de la cámara se construye con la aproximación que ofrece el enunciado, tomando la focal igual al ancho de la imagen y el centro óptico en el medio.

La estimación absoluta resulta de encadenar tres transformadas: la pose de la baliza en el mundo (`T_W_T`, construida a partir del `[X, Y, Z, Yaw]` del mapa YAML cruzando el identificador de la baliza), la pose de la baliza respecto a la cámara invertida, y la pose de la cámara respecto al robot invertida. El producto `T_W_R = T_W_T · inv(T_C_T) · inv(T_R_C)` da la pose del robot en el mundo, de la que se extraen la posición de la columna de traslación y el yaw con `atan2` sobre el bloque de rotación. Cuando el robot ve una baliza publica directamente esta estimación y guarda un ancla (pose estimada y lectura de odometría de ese instante) para el dead-reckoning.

## Dead-reckoning

Cuando no se ve ninguna baliza, la pose se mantiene integrando el movimiento medido por odometría desde la última vez que se vio una. La odometría se usa exclusivamente para medir el incremento relativo, nunca como posición absoluta: se calcula el desplazamiento entre la lectura del ancla y la actual como una composición SE(2), tomando `inv(T_odom_fix) · T_odom_now` para obtener el movimiento en el marco local del ancla, y ese incremento se aplica sobre la pose absoluta que se estimó en el ancla mediante `T_world_fix · T_delta`. Trabajar siempre desde el ancla en lugar de acumular incrementos iteración a iteración expresa de forma limpia el requisito del enunciado y acumula menos error numérico.

<iframe width="560" height="315" src="https://www.youtube.com/embed/LL2klXZEQJc" frameborder="0" allowfullscreen></iframe>

## Problemas afrontados

Durante el desarrollo aparecieron varios problemas cuya resolución fue la parte más instructiva de la práctica. En casi todos, la clave fue no fiarse de lo asumido y verificarlo empíricamente sobre el simulador, avanzando paso a paso y comprobando cada transformada antes de encadenar la siguiente.

**Verificación de PnP antes de encadenar.** Antes de montar toda la cadena de transformadas se comprobó que PnP funcionaba de forma aislada, imprimiendo `norm(tvec)` (la distancia cámara-baliza) mientras el robot se acercaba y alejaba de las balizas. Al observar que las balizas lejanas daban distancias mayores y las cercanas menores se confirmó que la detección, el orden de esquinas y la matriz de intrínsecos eran consistentes, evitando arrastrar un error de base hasta el final de la cadena.

**Orientación de la cámara óptica.** El error más costoso fue el cambio de ejes entre el marco óptico de la cámara con el que razona PnP (Z hacia delante, X derecha, Y abajo) y el marco del robot (X adelante, Y izquierda, Z arriba). Los joints del SDF solo aportan la traslación de la cámara, con rotación cero, de modo que ese giro había que introducirlo a mano. Situarlo mal producía una pose girada o reflejada respecto a la baliza: la distancia salía correcta pero el robot aparecía al lado equivocado del marcador.

**Offset de posición con la distancia.** Con la cadena ya correcta persiste un pequeño error de posición que crece a medida que la baliza está más lejos. Seguramente se debe a la matriz de intrínsecos aproximada (focal igual al ancho de la imagen), que no coincide con la focal real. Es asumible para la práctica; afinarla con el campo de visión real del SDF sería la vía para reducirlo.

## Limitaciones y mejora futura

Como mejoras, calibrar la matriz de intrínsecos con el campo de visión real reduciría el offset de posición, y un filtrado temporal de la estimación (en lugar de publicar directamente cada medida de PnP) suavizaría el salto entre la corrección visual y el dead-reckoning.

---

Referencias:
  <a href="https://jderobot.github.io/RoboticsAcademy/exercises/Localization/marker_visual_loc/"> Marker Based Visual Loc - Robotics Academy </a>
