---
title: "Vacuum Cleaner (BSA)"
date: 2026-07-01
---
Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/Other/vacuum_cleaner_loc/">Vacuum Cleaner </a>
de Unibotics.

El objetivo de la práctica es que un robot aspirador cubra toda la superficie navegable de una vivienda de la que se conoce el mapa a priori y en la que el robot se localiza mediante SLAM. El pilotaje es por posición: se usa la localización para decidir cada movimiento, no un plan calculado de antemano.

El algoritmo elegido es BSA (Backtracking Spiral Algorithm): el robot barre en espiral hasta quedar rodeado de celdas ya visitadas o de obstáculos, y en ese momento navega hasta la celda sin visitar más cercana para reiniciar otra espiral. El proceso se repite hasta que no quedan celdas pendientes alcanzables.

La solución se organiza como una máquina de estados, de modo que el bucle de control late a frecuencia fija y cada estado da un paso y cede el control, sin funciones bloqueantes. Los estados son: followBSA (decisión), rotate (giro), forward (avance), goToBacktrackingPoint (retorno) y finished.

Sobre el mapa se construye una rejilla de navegación a partir de la imagen conocida: binarización, inversión y dilatación de los obstáculos por el radio del robot (espacio de configuraciones). Cada celda tiene tres estados posibles: libre, visitada u obstáculo.

Las reglas de decisión de BSA en cada celda son:
1. Si los cuatro vecinos están bloqueados, se activa el backtracking.
2. Si el lado derecho de la espiral está libre, se gira hacia él.
3. Si el frente está bloqueado, se gira al otro lado.
4. En cualquier otro caso, se avanza de frente.

Los giros se hacen a orientaciones absolutas (cardinales) en lugar de relativas, para no acumular error angular a lo largo del recorrido.

Para el retorno al punto de backtracking se usa una búsqueda en anchura (BFS) sobre la rejilla, que garantiza el camino más corto en número de celdas y es robusto frente a obstáculos cóncavos. El robot sigue el camino resultante celda a celda hasta reanudar la espiral.

{% include youtube.html id="TU_VIDEO_AQUI" %}

## Problemas afrontados

Durante el desarrollo aparecieron varios problemas cuya resolución fue la parte más instructiva de la práctica.

**Coherencia de umbrales.** La pregunta "¿está libre esta celda?" tiene dos respuestas según el contexto: la exploración quiere celdas *no visitadas* (umbral estricto), mientras que el backtracking necesita poder *cruzar* celdas ya visitadas (umbral laxo). Descoordinar ambos umbrales hacía que el robot planificara rutas que después se negaba a recorrer, quedándose bloqueado. La solución fue usar umbrales explícitos y coherentes entre el planificador y el ejecutor.

**Sincronización entre plan y ejecución.** El plan es discreto (celdas ideales) pero la ejecución es continua y con deriva. Avanzar en el camino sin confirmar la llegada real desincronizaba plan y robot, que acababa persiguiendo puntos que ya había dejado atrás. Se corrigió descartando cada punto del camino solo cuando se confirma la llegada, manteniendo el invariante de que el primer punto pendiente es siempre el objetivo actual.

**Deriva de localización.** Con una deriva pequeña, el láser detecta pared donde el mapa dice que hay espacio libre. Marcar eso como obstáculo llegaba a "tapiar" al robot. Como el mapa a priori es correcto y no hay obstáculos nuevos, la interpretación adecuada es que esas detecciones reflejan un error de posición, no una pared real. La solución final corrige la pose estimada comparando la distancia a la pared que da el láser con la que dice el mapa, en lugar de introducir obstáculos falsos.

## Limitaciones y mejora futura

La corrección de pose actúa sobre la componente normal a la pared que el robot encara, por lo que la deriva lateral se compensa al enfrentar paredes perpendiculares en otros carriles. Una corrección más general por scan-matching (ICP) eliminaría la deriva residual, pero se optó por la solución más simple por ser proporcional al problema, dado que el SLAM ya proporciona una localización suficiente.
