---
title: "RESCUE PEOPLE"
date: 2026-07-02
---
Implementación de una solución para el ejercicio
<a href="https://jderobot.github.io/RoboticsAcademy/exercises/Drones/rescue_people/">Rescue People </a>
de Unibotics.

El objetivo de la práctica es que un dron participe en una misión de búsqueda y rescate en alta mar. Se conocen las coordenadas GPS aproximadas del accidente y la posición de la lancha desde la que despega, ambas en UTM. El dron debe aproximarse a la zona, hacer una búsqueda de supervivientes, anotar la posición de cada persona localizada (sin repetir) y regresar a la lancha, con la restricción de volver a cargar cuando se le acabe la batería. La detección de personas se hace con los detectores en cascada Haar de OpenCV sobre la cámara ventral.

La solución se organiza como una máquina de estados, de modo que el bucle de control late a frecuencia fija (10 Hz) y cada estado da un paso y cede el control, comandando siempre velocidades no bloqueantes. Los estados son: takeoff (despegue), go_to_people (aproximación al punto de rescate), search_other_people (barrido en espiral), goToCharge (retorno por batería baja), goBack (retorno tras completar la misión), land (aterrizaje) y finished.

<pre class="mermaid">
stateDiagram-v2
    [*] --> takeoff
    takeoff --> go_to_people: altura alcanzada
    go_to_people --> search_other_people: en zona de rescate
    search_other_people --> goBack: 6 personas / rejilla completa
    go_to_people --> goToCharge: bateria baja
    search_other_people --> goToCharge: bateria baja
    goToCharge --> land: llegada a la lancha
    goBack --> land: llegada a la lancha
    land --> takeoff: recarga (mision incompleta)
    land --> finished: mision completa
    finished --> [*]
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

Como las coordenadas UTM ya están en metros, la conversión al marco local del dron (referido a su punto de despegue) es la simple diferencia respecto al origen, sin fórmulas de latitud/longitud. La navegación se comanda por velocidad con `HAL.set_cmd_vel`: la función `go_to` calcula distancia y rumbo al objetivo, gira el morro hacia el destino y avanza reduciendo la velocidad al acercarse.

En la zona de rescate se realiza un barrido en espiral cuadrada de paso constante. La detección emplea el clasificador `haarcascade_frontalface_default.xml` sobre la cámara ventral; como desde una vista cenital la cara puede aparecer con cualquier orientación en el plano, la imagen se rota en pasos de 30° y se busca en cada rotación. Cada cara detectada se proyecta de la imagen 2D a coordenadas del mundo aprovechando que la altura del dron es conocida.

![Pasted image](https://github.com/JhonDL/robotica_de_servicio/assets/60139647/09b28f96-4c80-4e3a-a544-dbda3360b1ba)

<img width="699" alt="image" src="https://github.com/JhonDL/robotica_de_servicio/assets/60139647/54374234-be15-4f0e-8ca6-cae1643ea815">

Conociendo la distancia en z (la altura) y la posición del píxel en la imagen, se obtiene la distancia en X e Y en el marco del dron, y con la pose y el yaw se pasa a coordenadas del mundo. Una deduplicación por distancia evita anotar dos veces a la misma persona. Finalmente, el aterrizaje sobre la lancha se resuelve por posición, y una batería simulada dispara el retorno a carga cuando baja de un umbral.

<iframe width="560" height="315" src="https://www.youtube.com/embed/JxpqGpD__74" frameborder="0" allowfullscreen></iframe>

## Problemas afrontados

Durante el desarrollo aparecieron varios problemas cuya resolución fue la parte más instructiva de la práctica. En casi todos, la clave fue no fiarse de lo documentado y verificarlo empíricamente sobre el simulador.

**El marco de referencia real no es el documentado.** La documentación indica que el marco de posición del dron es `North=+x, West=+y`. Al aplicar esa convención, el dron viajaba a un punto desplazado unos 90° respecto a la zona real. Para averiguar el marco correcto se voló a los cuatro candidatos rotados de la conversión y se comprobó con la cámara ventral cuál tenía supervivientes debajo. El marco real resultó ser (`x=Este, y=Norte, z=Arriba`).

**La cámara ventral está girada y espejada respecto al cuerpo.** La proyección imagen→mundo daba coordenadas rotadas y espejadas.

**Deduplicación de víctimas.** Las distintas detecciones de una misma persona tienen una dispersión de unos 2.5 m según la pose desde la que se observa, mientras que personas distintas están separadas varios metros. El umbral de "detección nueva" debe caer entre ambas magnitudes: un valor de unos 4 m separa correctamente a las víctimas sin fusionarlas ni contarlas por duplicado.

**Aterrizaje con velocidades de cuerpo.** `set_cmd_vel` comanda velocidades en el marco del cuerpo, pero el dron llega a la lancha con un yaw cualquiera porque venía girando hacia el objetivo. El error de posición, calculado en el mundo, se rota por `-yaw` para pasarlo al cuerpo antes de comandarlo, y el descenso solo se activa cuando el dron está centrado, para no chocar con el borde de la base.

## Limitaciones y mejora futura

El detector Haar `frontalface` es débil en vista cenital: la cara se ve pequeña y picada, por lo que acierta en pocos frames. Se mitiga volando más bajo y más despacio durante el barrido, para que la cara aparezca mayor y permanezca más tiempo en cuadro. Un detector de cuerpos por color permitiría buscar mucho más alto y rápido, pero la práctica exige el uso de Haar sobre caras.

---

Referencias:
  <a href="https://www.geeksforgeeks.org/mapping-coordinates-from-3d-to-2d-using-opencv-python/"> Mapping coordinates from 3d to 2d </a>
  <a href="https://docs.opencv.org/4.5.0/db/d28/tutorial_cascade_classifier.html"> Face Detection using Haar Cascades </a>
