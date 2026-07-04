---
title: "AMAZON WAREHOUSE"
date: 2026-07-02
---
Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/MobileRobots/amazon_warehouse/">Amazon Warehouse</a>
de Unibotics.

El objetivo de la práctica es que un robot logístico entregue estanterías en un almacén desde su ubicación conocida hasta un punto de destino. El robot dispone de un mapa del almacén y conoce su pose dentro de él, así que no hay que estimar la localización: el reto es planificar el camino más corto hasta la estantería, recogerla, y volver a llevarla al punto de entrega. La práctica no consiste en implementar un planificador propio, sino en aprender a usar la librería OMPL (Open Motion Planning Library) para resolver el problema.

La solución se organiza como una máquina de estados, de modo que el bucle de control late a frecuencia fija (10 Hz) y cada estado da un paso y cede el control, comandando siempre velocidades no bloqueantes. Los estados son: plan_go (planificación de la ida), go_shelf (navegación hasta la estantería), plan_back (planificación de la vuelta), go_back (navegación al punto de entrega) y finished. La recogida y la entrega se resuelven con `HAL.lift()` y `HAL.putdown()` al alcanzar cada objetivo.

<pre class="mermaid">
stateDiagram-v2
    [*] --> plan_go
    plan_go --> go_shelf: camino planificado
    go_shelf --> plan_back: estanteria recogida (lift)
    plan_back --> go_back: camino planificado
    go_back --> finished: estanteria entregada (putdown)
    go_shelf --> finished: sin camino
    go_back --> finished: sin camino
    finished --> [*]
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

El mapa se carga con `WebGUI.getMap` y se convierte en una rejilla de ocupación: los píxeles negros son obstáculo y los blancos, espacio libre. Se engordan los obstáculos con el radio del robot mediante una dilatación. La navegación se comanda por velocidad: la función `go_to` calcula distancia y rumbo al siguiente punto del camino, gira sobre el sitio hasta alinearse y avanza reduciendo la velocidad al acercarse, siguiendo el camino waypoint a waypoint.

La planificación se hace con OMPL sobre un `RealVectorStateSpace(2)`, es decir, un espacio euclídeo en (x, y). El validador de estados convierte cada coordenada del mundo a píxel y comprueba que caiga en zona libre del mapa engordado. Como objetivo se fija `PathLengthOptimizationObjective` y se usa el planificador `RRTstar`, que es el que optimiza la longitud del camino; de ahí que la trayectoria resultante tienda al camino más corto. La solución de OMPL se convierte en una lista de waypoints en coordenadas del mundo que el controlador va siguiendo.

La conversión entre el mundo (metros, Gazebo) y el mapa (píxeles) es una transformación afín con escala uniforme en cada eje y el origen calibrado sobre el spawn del robot. Todo el ejercicio depende de acertar esta transformación: un error en el origen o en la asignación de ejes desplaza tanto el objetivo como el camino dibujado.

<iframe width="560" height="315" src="https://www.youtube.com/embed/_v0GFPuxHG0" frameborder="0" allowfullscreen></iframe>

## Problemas afrontados

Durante el desarrollo aparecieron varios problemas cuya resolución fue la parte más instructiva de la práctica. En casi todos, la clave fue no fiarse de lo documentado y verificarlo empíricamente sobre el simulador.

**Calibración fina del origen.** Con la orientación de ejes ya correcta, quedaba un pequeño desplazamiento constante: unos 0.5 m en X y 0.25 m en Y. Se tradujo a píxeles con las escalas (~20 px/m) y se ajustó el píxel del origen moviéndolo hasta que el robot cayera clavado sobre sus posicion real.

## Limitaciones y mejora futura

El planificador no tiene en cuenta que el robot cambia de tamaño al coger una estantería lo que puede llegar a colisionar.

---

Referencias:
  <a href="https://ompl.kavrakilab.org/">The Open Motion Planning Library (OMPL)</a>
  <a href="https://ompl.kavrakilab.org/OMPL_Primer.pdf">OMPL Primer</a>
