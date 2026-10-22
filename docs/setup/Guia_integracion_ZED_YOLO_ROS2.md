# Guía conjunta: cámara ZED y detector YOLO en ROS 2 Humble

Pixloop · Ubuntu 22.04.5 · 14 de octubre de 2026

## 1. Objetivo y funcionamiento

Ejecutar la cámara ZED y el detector YOLO en dos terminales independientes de la computadora del vehículo. La cámara publica imágenes; el detector recibe las imágenes, reconoce objetos y publica una imagen con cajas y etiquetas.

Flujo: ZED → `/zed/zed_node/rgb/color/rect/image` → `/detector_yolo` → `/pixloop/perception/yolo/image`.

El modelo es YOLOv8n preentrenado. Esta versión identifica objetos en 2D; todavía no calcula distancias ni publica una lista estructurada de obstáculos.

## 2. Ubicaciones y requisitos

Los comandos se ejecutan en el equipo remoto `dc-Nuvo-6108GC`, con usuario `dc`.

| Elemento | Ubicación |
|---|---|
| Driver de cámara | Paquete `pixloop_zed`, lanzamiento `zed_driver.launch.py` |
| Workspace principal | `/data/workspaces/pixloop-sensor-ws` |
| Script del detector | `/home/dc/plataforma_challenge/deteccion_obstaculos/detector_yolo.py` |
| Entorno Python | `/home/dc/yolo_env` |
| Pesos del modelo | `/home/dc/plataforma_challenge/yolov8n.pt` |
| Imagen utilizada para la prueba inicial | `/home/dc/plataforma_challenge/bus.jpg` |

El script se creó en el equipo remoto mediante `cat > detector_yolo.py <<'PY'`, pegando el código y cerrando con una línea `PY`. El archivo permanece guardado. Al ejecutarlo, la instrucción `super().__init__('detector_yolo')` crea el nodo ROS `/detector_yolo`. Al detenerlo, el nodo deja de estar activo.

El detector se ejecuta directamente con Python y no requiere compilación con `colcon`. Las dependencias ya probadas incluyen NumPy 1.26.4, OpenCV 4.11.0.86 y pyserial 3.5. La GPU utilizada en la prueba fue una NVIDIA GeForce RTX 3060.

## 3. Terminal 1: cámara ZED

Abre una terminal y conecta al vehículo:

```bash
ssh dc@10.48.204.199
```

La IP corresponde a la prueba registrada. Si cambia, usa la dirección actual. Introduce la contraseña cuando se solicite; no escribas el símbolo `$` del prompt.

Carga el entorno del proyecto:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Inicia la cámara:

```bash
ros2 launch pixloop_zed zed_driver.launch.py
```

Mantén esta terminal abierta. Si el driver ya está ejecutándose, utiliza ese proceso existente.

Los tópicos son salidas del driver: no se añaden al comando de lanzamiento. No es necesario activar el entorno de YOLO en esta terminal.

## 4. Terminal 2: detector YOLO

Abre otra terminal y conecta al mismo equipo:

```bash
ssh dc@10.48.204.199
```

Carga el mismo entorno ROS:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Activa el entorno Python:

```bash
source ~/yolo_env/bin/activate
```

El prefijo `(yolo_env)` debe aparecer en el prompt. Ejecuta el detector:

```bash
python3 ~/plataforma_challenge/deteccion_obstaculos/detector_yolo.py \
  --ros-args \
  -p image_topic:=/zed/zed_node/rgb/color/rect/image
```

`image_topic` selecciona la entrada del detector. Puedes ejecutar este comando desde cualquier carpeta porque la ruta del script es completa.

Resultado esperado:

```text
[detector_yolo]: Esperando imágenes en /zed/zed_node/rgb/color/rect/image
[detector_yolo]: 6 detecciones: person, person, person, person, chair, tv
```

Cada mensaje muestra las detecciones de una imagen; no es un conteo acumulado ni seguimiento de identidades. Coloca una persona u objeto visible frente a la cámara y comprueba que los resultados cambian. Las etiquetas son predicciones y pueden contener errores.

## 5. Qué hace el código del detector

1. Carga el modelo desde `yolov8n.pt`.
2. Se suscribe a imágenes `sensor_msgs/msg/Image` usando `qos_profile_sensor_data`.
3. Conserva la imagen más reciente recibida.
4. Un temporizador de 0.2 segundos intenta procesar una imagen nueva. La frecuencia objetivo es aproximadamente 5 Hz; la real depende de la entrada y del procesamiento.
5. `cv_bridge` convierte el mensaje ROS a una imagen OpenCV BGR.
6. YOLO ejecuta inferencia con umbral `conf=0.25` y `device=0`, la primera GPU CUDA.
7. Escribe las clases en la terminal y publica la imagen anotada, conservando la cabecera original.

El programa no abre ventanas; funciona por SSH.

## 6. Interfaces de cámara y detector

| Tópico | Contenido | Papel |
|---|---|---|
| `/zed/zed_node/rgb/color/rect/image` | Imagen RGB rectificada | Entrada de YOLO, `sensor_msgs/msg/Image`. |
| `/zed/zed_node/point_cloud/cloud_registered` | Nube de puntos 3D | Salida ZED disponible; no utilizada por este detector. |
| `/zed/zed_node/odom` | Odometría ZED | Salida disponible; no utilizada por este detector. |
| `/pixloop/perception/yolo/image` | Imagen con cajas y etiquetas | Salida del detector, `sensor_msgs/msg/Image`. |

## 7. Terminal 3 opcional: comprobaciones

Abre otra conexión SSH y carga el entorno ROS del apartado 3. Ejecuta:

```bash
ros2 node list
ros2 node info /detector_yolo
ros2 topic info /zed/zed_node/rgb/color/rect/image --verbose
ros2 topic hz /zed/zed_node/rgb/color/rect/image
```

Detén `topic hz` con `Ctrl+C` y comprueba después la salida:

```bash
ros2 topic hz /pixloop/perception/yolo/image
```

La comprobación de entrada confirma recepción de imágenes; la de salida permite comprobar que se publican imágenes anotadas. Los registros adjuntos confirmaron recepción e inferencia, pero no incluyen una verificación independiente de la visualización.

Para visualizar desde una sesión Ubuntu con escritorio gráfico y entorno ROS cargado:

```bash
ros2 run rqt_image_view rqt_image_view
```

Selecciona `/pixloop/perception/yolo/image`. Si aparece `cannot open display` por SSH, la sesión no tiene acceso gráfico; utiliza las comprobaciones de terminal o abre el visor desde el escritorio del equipo.

## 8. Validación inicial antes de usar la cámara

Esta validación ya se completó; no es necesario repetirla en cada arranque. Para volver a comprobar el entorno, carga ROS y activa `yolo_env`:

```bash
python3 - <<'PY'
import numpy as np
import rclpy
from cv_bridge import CvBridge
from ultralytics import YOLO

bridge = CvBridge()
imagen = np.zeros((480, 640, 3), dtype=np.uint8)
mensaje = bridge.cv2_to_imgmsg(imagen, encoding='bgr8')
recuperada = bridge.imgmsg_to_cv2(mensaje, desired_encoding='bgr8')
assert np.array_equal(imagen, recuperada)
print('ROS + cv_bridge + YOLO: OK')
PY
```

Prueba independiente con la imagen guardada:

```bash
cd ~/plataforma_challenge
yolo predict model=yolov8n.pt source=bus.jpg save=True
```

La prueba registrada detectó cuatro personas, un autobús y una señal de alto, con aproximadamente 5.4 ms de inferencia. Ese tiempo corresponde a una imagen fija y no representa la latencia completa del flujo ZED–ROS–YOLO.

Para usar un publicador de prueba que publique imágenes en `/camera/image_raw`, inicia el detector con su entrada por defecto:

```bash
python3 ~/plataforma_challenge/deteccion_obstaculos/detector_yolo.py
```

Esto requiere un publicador de prueba activo. La prueba con una imagen fija por CLI no publica imágenes en ROS por sí sola.

## 9. Atajo px

El alias es opcional. Para definirlo en cada terminal donde se necesite:

```bash
alias px='source /opt/ros/humble/setup.bash && source /data/workspaces/pixloop_ros2_ws/install/setup.bash && source /data/workspaces/pixloop_mapping_ws/install/setup.bash && source /data/workspaces/pixloop-sensor-ws/install/setup.bash && export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp'
```

Después `px` sustituye el bloque de carga del entorno ROS. En la terminal de YOLO todavía debes activar `source ~/yolo_env/bin/activate`.

El alias manual dura en esa terminal, salvo que se haya guardado en la configuración del shell. Si no existe, utiliza los comandos completos; no instales un programa llamado `px`.

## 10. Detener y editar

1. En la terminal de YOLO, pulsa `Ctrl+C` para detener el detector.
2. En la terminal ZED, pulsa `Ctrl+C` para detener la cámara cuando termines.

Para editar el detector:

```bash
nano ~/plataforma_challenge/deteccion_obstaculos/detector_yolo.py
```

Guarda con `Ctrl+O`, confirma con Enter y sal con `Ctrl+X`. Reinicia el detector para aplicar los cambios.

El registro mostró al detener YOLO el error `rcl_shutdown already called on the given context`. La detección funcionó antes del cierre. Corrección propuesta, pendiente de aplicar y verificar en el equipo: sustituir la llamada final `rclpy.shutdown()` por `rclpy.try_shutdown()`.

## 11. Resolución de problemas

| Síntoma | Acción |
|---|---|
| `px: command not found` | Usa el bloque completo de configuración ROS. |
| `Package 'pixloop_zed' not found` | Carga los tres workspaces; revisa cualquier error de ruta. |
| `ModuleNotFoundError: ultralytics` | Activa `/home/dc/yolo_env/bin/activate`. |
| No se encuentra `detector_yolo.py` | Comprueba la ruta del script indicada en el apartado 2. |
| YOLO solo espera imágenes | Comprueba el publicador, nombre de tópico, QoS y el mismo `ROS_DOMAIN_ID` en ambas terminales. |
| Error de cámara | Revisa conexión física y mensajes concretos del driver. |
| NumPy `_ARRAY_API not found` | La incompatibilidad observada se resolvió con NumPy 1.26.4 y OpenCV 4.11.0.86 en `yolo_env`. |
| `cannot open display` | Usa una sesión gráfica para el visor o comprueba tópicos desde la terminal. |

No es necesario reinstalar dependencias ni descargar el modelo en cada ejecución.

Referencia del arranque de Pixloop, proporcionada durante la sesión:
https://github.com/DCaicrag/pixloop-cyberphysical-labs/blob/main/docs/setup/PIXLOOP_BRINGUP.md
