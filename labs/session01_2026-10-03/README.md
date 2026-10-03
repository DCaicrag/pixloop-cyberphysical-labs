# Sesión 01 — Introducción a Pixloop
## 3 de octubre de 2026
### SSH · Git/GitHub · ROS 2 · CARLA · Exploración del sistema

**Plataforma:** Pixloop  
**Duración sugerida:** 3–4 horas  
**Modalidad:** Trabajo colaborativo  
**Nivel:** Introducción al entorno de desarrollo de Pixloop  

---

# 1. Objetivo

Esta primera sesión corresponde al onboarding técnico al entorno de desarrollo de **Pixloop**.

El objetivo principal no es comprender todavía todos los algoritmos del vehículo autónomo, sino aprender a entrar al sistema, explorarlo de forma segura y trabajar colaborativamente sobre él.

Al finalizar la sesión, el equipo deberá ser capaz de:

- conectarse remotamente a Pixloop mediante SSH;
- identificar el entorno Linux y ROS 2;
- localizar el workspace de Pixloop;
- utilizar Git y GitHub mediante un flujo basado en branches;
- ejecutar CARLA como entorno de simulación;
- explorar un sistema ROS 2 desconocido;
- identificar nodos, tópicos y tipos de mensajes;
- localizar sensores y canales de control;
- enviar un comando básico de movimiento al vehículo en CARLA;
- detener el vehículo explícitamente;
- documentar resultados;
- realizar `commit`, `push` y abrir un Pull Request.

---

# 2. Filosofía de trabajo

Durante estas sesiones no se busca memorizar comandos específicos de Pixloop.

Se busca aprender a **descubrir cómo funciona un sistema robótico desconocido utilizando sus propias interfaces**.

> ## Regla de ingeniería
>
> Antes de publicar sobre un tópico desconocido:
>
> 1. identificar el tópico;
> 2. consultar su tipo;
> 3. inspeccionar la interfaz del mensaje;
> 4. identificar quién publica y quién se suscribe;
> 5. solamente después construir un comando.

Las principales herramientas de descubrimiento serán:

```bash
ros2 node list
ros2 node info <NODE>

ros2 topic list
ros2 topic info <TOPIC>
ros2 topic info <TOPIC> --verbose
ros2 topic echo <TOPIC>
ros2 topic hz <TOPIC>

ros2 interface show <MESSAGE_TYPE>
```

---

# 3. Tipos de instrucciones

Durante la práctica se utilizarán tres etiquetas.

## 🟢 STUDENT

Comando que puede ejecutar directamente el estudiante.

## 🟡 INSTRUCTOR

Comando que inicialmente será ejecutado o supervisado por el instructor.

## 🔴 SIMULATION ONLY

Comando que puede generar movimiento y deberá ejecutarse únicamente en CARLA durante esta sesión.

---

# 4. Seguridad

## 4.1 Pixloop físico

> **No ejecutar comandos de movimiento sobre Pixloop físico durante esta sesión salvo autorización explícita del instructor.**

El vehículo físico será utilizado principalmente para:

- conexión SSH;
- inspección del sistema;
- identificación del entorno ROS 2;
- exploración de nodos y tópicos;
- observación del stack.

Los experimentos de movimiento se realizarán inicialmente en:

```text
CARLA
```

---

## 4.2 Git

No realizar:

```bash
git push origin main
```

Todo desarrollo deberá realizarse desde una branch independiente.

Formato:

```text
feature/<team>/session01
```

Ejemplos:

```text
feature/team-a/session01
feature/team-b/session01
```

---

# 5. Comandos pendientes de validación

Algunos valores aparecen como:

```text
<TO_CONFIRM>
```

Esto significa que deben verificarse contra la instalación real de Pixloop.

> No reemplazar un `TO_CONFIRM` mediante suposiciones.

---

# 6. Arquitectura general

Durante las siguientes sesiones se explorarán progresivamente las distintas capas del sistema.

```text
┌─────────────────────────────┐
│           Sensors           │
│ LiDAR · Camera · IMU · ...  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       LiDAR Odometry        │
│          KISS-ICP           │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      State Estimation       │
│             EKF             │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        Localization         │
│ Persistent Map / NDT / ...  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        Path Planning        │
│        Nav2 / Smac          │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Tracking / Control      │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│           Vehicle           │
└─────────────────────────────┘
```

En esta sesión únicamente se observará esta arquitectura a alto nivel.

---

# 7. Agenda sugerida

| Tiempo | Actividad |
|---|---|
| 00:00–00:15 | Introducción a Pixloop y seguridad |
| 00:15–00:35 | SSH + Linux + ROS 2 |
| 00:35–01:10 | Git y GitHub |
| 01:10–01:20 | Troubleshooting / pausa |
| 01:20–01:40 | CARLA |
| 01:40–02:15 | ROS 2 discovery |
| 02:15–02:40 | Identificación de sensores y control |
| 02:40–03:00 | Movimiento en CARLA |
| 03:00–03:20 | Demostración del stack Pixloop |
| 03:20–03:40 | `results.md` + commit + push |
| 03:40–04:00 | Pull Request y cierre |

La agenda es orientativa. Si alguna instalación consume más tiempo, las actividades marcadas como opcionales pueden omitirse.

---

# 8. Parte A — Conexión SSH a Pixloop

## 8.1 Verificar conectividad

### 🟢 STUDENT

```bash
ping <PIXLOOP_IP>
```

Configuración real:

```bash
ping <TO_CONFIRM_PIXLOOP_IP>
```

Detener:

```text
Ctrl+C
```

---

## 8.2 Conectarse mediante SSH

Formato:

```bash
ssh <USER>@<PIXLOOP_IP>
```

Configuración real:

```bash
ssh <TO_CONFIRM_USER>@<TO_CONFIRM_PIXLOOP_IP>
```

La primera conexión puede mostrar:

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Responder:

```text
yes
```

---

## 8.3 Identificar la computadora

### 🟢 STUDENT

Ejecutar:

```bash
hostname
```

```bash
whoami
```

```bash
pwd
```

```bash
uname -a
```

```bash
lsb_release -a
```

Registrar:

```text
Hostname:
Usuario:
Ubuntu:
Kernel:
```

---

# 9. Verificar ROS 2

### 🟢 STUDENT

```bash
printenv ROS_DISTRO
```

```bash
ros2 --help
```

```bash
env | grep ROS
```

Si se utiliza `ROS_DOMAIN_ID`:

```bash
echo $ROS_DOMAIN_ID
```

Registrar:

```text
ROS_DISTRO:
ROS_DOMAIN_ID:
```

---

# 10. Localizar el workspace de Pixloop

Ruta:

```bash
cd <TO_CONFIRM_PIXLOOP_WORKSPACE>
```

Verificar:

```bash
pwd
```

```bash
ls
```

Si corresponde a un workspace de `colcon`, podrían aparecer:

```text
build/
install/
log/
src/
```

Explorar:

```bash
ls src
```

---

## 10.1 Cargar el workspace

Si es necesario:

```bash
source /opt/ros/<TO_CONFIRM_ROS_DISTRO>/setup.bash
```

Después:

```bash
source <TO_CONFIRM_PIXLOOP_WORKSPACE>/install/setup.bash
```

---

## 10.2 Ver paquetes disponibles

```bash
ros2 pkg list
```

Filtrar posibles componentes relevantes:

```bash
ros2 pkg list | grep -Ei "pix|local|nav|lidar|kiss|ekf|ndt|planning|control"
```

---

# 11. Parte B — Git y GitHub

## 11.1 Verificar Git

### 🟢 STUDENT

```bash
git --version
```

Verificar identidad:

```bash
git config --global user.name
git config --global user.email
```

Si todavía no está configurada:

```bash
git config --global user.name "Nombre Apellido"
git config --global user.email "correo@ejemplo.com"
```

---

# 12. Clonar el repositorio

Repositorio del proyecto:

```text
<TO_CONFIRM_GITHUB_REPOSITORY>
```

Mediante SSH:

```bash
git clone git@github.com:<ORGANIZATION>/<REPOSITORY>.git
```

Comando real:

```bash
git clone <TO_CONFIRM_GIT_URL>
```

Entrar:

```bash
cd <TO_CONFIRM_REPOSITORY_NAME>
```

---

# 13. Inspeccionar el repositorio

Antes de modificar cualquier archivo:

```bash
git status
```

```bash
git branch -a
```

```bash
git log --oneline --decorate -10
```

```bash
git remote -v
```

---

# 14. Actualizar `main`

```bash
git switch main
```

```bash
git pull origin main
```

```bash
git status
```

El repositorio debería estar sincronizado antes de crear una nueva branch.

---

# 15. Crear la branch del equipo

Formato:

```text
feature/<team>/session01
```

Ejemplo:

```bash
git switch -c feature/team-a/session01
```

Verificar:

```bash
git branch
```

Ejemplo:

```text
* feature/team-a/session01
  main
```

---

# 16. Crear la carpeta del equipo

Ejemplo para Team A:

```bash
mkdir -p students/team-a/session01/evidence
```

Crear archivo de resultados:

```bash
touch students/team-a/session01/results.md
```

Abrir con:

```bash
nano students/team-a/session01/results.md
```

o, si está disponible:

```bash
code students/team-a/session01/results.md
```

---

# 17. Plantilla de `results.md`

Utilizar:

```markdown
# Session 01 Results

## Team

- Student 1:
- Student 2:
- Student 3:
- Student 4:
- Student 5:

## Pixloop environment

Hostname:

Ubuntu:

ROS_DISTRO:

ROS_DOMAIN_ID:

Pixloop workspace:

## Git

Branch:

Commit:

Pull Request:

## CARLA

Version:

Map:

Vehicle role:

## ROS 2

Number of nodes:

Number of topics:

## Sensors

| Sensor | Topic | Message type | Frequency |
|---|---|---|---|
| Camera | | | |
| LiDAR | | | |
| IMU | | | |
| Odometry | | | |
| Vehicle State | | | |

## Vehicle Control

Control topic:

Message type:

Publisher:

Subscriber:

## Observations

...

## Problems found

...

## Analysis

### Q1

¿Cuál es la diferencia entre un nodo, un tópico y un tipo de mensaje en ROS 2?

### Q2

Describe el procedimiento que utilizaste para descubrir cómo controlar el vehículo sin conocer inicialmente el tópico ni el tipo de mensaje.

### Q3

¿Por qué trabajamos sobre una feature branch en lugar de modificar `main` directamente?

### Q4

Identifica cuatro componentes observados durante la sesión y clasifícalos como sensor, estimación, planificación o control.
```

---

# 18. Primer commit

Antes de continuar:

```bash
git status
```

Agregar:

```bash
git add students/team-a/session01/results.md
```

Commit:

```bash
git commit -m "docs: add team-a session01 environment validation"
```

---

# 19. Publicar la branch

```bash
git push -u origin feature/team-a/session01
```

Después del primer push bastará con:

```bash
git push
```

---

# 20. Parte C — Ejecutar CARLA

> Los comandos exactos de esta sección deben validarse con la instalación del laboratorio.

## 20.1 Acceder a CARLA

### 🟡 INSTRUCTOR / 🟢 STUDENT según configuración

```bash
cd <TO_CONFIRM_CARLA_PATH>
```

Verificar:

```bash
pwd
ls
```

---

## 20.2 Iniciar CARLA

```bash
<TO_CONFIRM_CARLA_START_COMMAND>
```

Ejemplo común de referencia:

```bash
./CarlaUE4.sh
```

No utilizar opciones adicionales hasta verificar la instalación utilizada.

Registrar:

```text
CARLA version:
Map:
Server:
Port:
```

---

# 21. Parte D — ROS 2 Bridge

En una terminal independiente:

```bash
source /opt/ros/<TO_CONFIRM_ROS_DISTRO>/setup.bash
```

Si existe un workspace para el bridge:

```bash
source <TO_CONFIRM_CARLA_ROS_WS>/install/setup.bash
```

---

## 21.1 Levantar el bridge

### 🟡 INSTRUCTOR / 🟢 STUDENT según configuración

```bash
<TO_CONFIRM_CARLA_BRIDGE_COMMAND>
```

Mantener esta terminal abierta.

---

# 22. Parte E — Descubrir el ROS graph

A partir de este punto, el objetivo es utilizar ROS 2 para descubrir el sistema.

## 22.1 Nodos

### 🟢 STUDENT

```bash
ros2 node list
```

Contar:

```bash
ros2 node list | wc -l
```

Inspeccionar uno:

```bash
ros2 node info <NODE_NAME>
```

---

## 22.2 Tópicos

```bash
ros2 topic list
```

Contar:

```bash
ros2 topic list | wc -l
```

Buscar tópicos relacionados con CARLA:

```bash
ros2 topic list | grep -i carla
```

---

# 23. Buscar sensores

## Cámara

```bash
ros2 topic list | grep -Ei "camera|image"
```

## LiDAR

```bash
ros2 topic list | grep -Ei "lidar|point|scan"
```

## IMU

```bash
ros2 topic list | grep -Ei "imu"
```

## GNSS / GPS

Opcional si existe:

```bash
ros2 topic list | grep -Ei "gnss|gps"
```

## Odometría

```bash
ros2 topic list | grep -Ei "odom|odometry"
```

## Control

```bash
ros2 topic list | grep -Ei "cmd|control|vehicle|throttle|steer"
```

---

# 24. Inspeccionar un tópico

Una vez identificado un tópico:

```bash
ros2 topic info <TOPIC>
```

Para obtener más información:

```bash
ros2 topic info <TOPIC> --verbose
```

Identificar:

```text
Topic type
Publisher count
Subscription count
Publisher node
Subscriber node
QoS
```

---

# 25. Inspeccionar el mensaje

Tomar el tipo reportado por:

```bash
ros2 topic info <TOPIC>
```

Después:

```bash
ros2 interface show <MESSAGE_TYPE>
```

Ejemplo conceptual:

```text
TOPIC
   │
   ▼
ros2 topic info
   │
   ▼
MESSAGE TYPE
   │
   ▼
ros2 interface show
   │
   ▼
FIELDS
```

---

# 26. Leer un sensor

Formato:

```bash
ros2 topic echo <TOPIC>
```

Para intentar leer solamente un mensaje:

```bash
ros2 topic echo <TOPIC> --once
```

---

# 27. Medir frecuencia

```bash
ros2 topic hz <TOPIC>
```

Detener:

```text
Ctrl+C
```

Registrar la frecuencia aproximada de:

- LiDAR;
- IMU;
- odometría.

---

# 28. Actividad principal de descubrimiento

Sin consultar directamente el código fuente, identificar:

1. tópico de cámara;
2. tópico de LiDAR;
3. tópico de IMU;
4. tópico de odometría;
5. tópico de control;
6. tipo de mensaje utilizado para controlar el vehículo;
7. nodo que publica la odometría;
8. frecuencia aproximada del LiDAR;
9. frecuencia aproximada de la IMU.

Completar los resultados en:

```text
students/<team>/session01/results.md
```

---

# 29. Parte F — Descubrir el control del vehículo

Buscar:

```bash
ros2 topic list | grep -Ei "control|cmd|vehicle|ego"
```

Registrar:

```text
Vehicle role:
Control topic:
Control message:
```

---

## 29.1 Inspeccionar el control

```bash
ros2 topic info <TO_CONFIRM_CONTROL_TOPIC> --verbose
```

Después:

```bash
ros2 interface show <TO_CONFIRM_CONTROL_MESSAGE_TYPE>
```

Identificar campos equivalentes a:

```text
throttle
steering
brake
reverse
hand_brake
```

Los nombres exactos dependerán del mensaje instalado.

---

# 30. Parte G — Movimiento básico

> # 🔴 SIMULATION ONLY
>
> Los siguientes comandos deberán ejecutarse únicamente en CARLA durante esta sesión.

---

## 30.1 Preparar primero el STOP

Antes de cualquier movimiento:

```bash
<TO_CONFIRM_CARLA_STOP_COMMAND>
```

Mantener este comando disponible.

---

## 30.2 Avanzar

```bash
<TO_CONFIRM_CARLA_FORWARD_COMMAND>
```

Condiciones:

```text
Throttle bajo
Steering cercano a cero
Brake liberado
Duración limitada
```

Observar el movimiento.

---

## 30.3 Detener

Ejecutar:

```bash
<TO_CONFIRM_CARLA_STOP_COMMAND>
```

Confirmar visualmente que el vehículo se detiene.

---

# 31. Guardar evidencias automáticamente

Crear la carpeta:

```bash
mkdir -p students/team-a/session01/evidence
```

Guardar nodos:

```bash
ros2 node list > students/team-a/session01/evidence/nodes.txt
```

Guardar tópicos:

```bash
ros2 topic list > students/team-a/session01/evidence/topics.txt
```

Guardar entorno ROS:

```bash
env | grep ROS > students/team-a/session01/evidence/ros_environment.txt
```

Guardar información del sistema:

```bash
uname -a > students/team-a/session01/evidence/system.txt
lsb_release -a >> students/team-a/session01/evidence/system.txt 2>&1
```

---

# 32. Parte H — Demostración del stack completo de Pixloop

> ## 🟡 INSTRUCTOR
>
> Esta sección será inicialmente ejecutada por el instructor.
>
> Los estudiantes no necesitan comprender todavía cada algoritmo.

El objetivo es observar cómo cambia el sistema ROS 2 cuando se levanta el stack completo.

---

## 32.1 Antes de iniciar el stack

Guardar el estado actual:

```bash
ros2 node list | sort > /tmp/nodes_before.txt
```

```bash
ros2 topic list | sort > /tmp/topics_before.txt
```

---

## 32.2 Levantar Pixloop

### 🟡 INSTRUCTOR

Si existe un comando único:

```bash
<TO_CONFIRM_FULL_PIXLOOP_STACK_COMMAND>
```

Si requiere varias terminales:

```text
Terminal 1:
<TO_CONFIRM>

Terminal 2:
<TO_CONFIRM>

Terminal 3:
<TO_CONFIRM>
```

Durante esta primera sesión no es necesario comprender cada comando.

---

## 32.3 Después de iniciar el stack

```bash
ros2 node list | sort > /tmp/nodes_after.txt
```

```bash
ros2 topic list | sort > /tmp/topics_after.txt
```

Comparar:

```bash
diff /tmp/nodes_before.txt /tmp/nodes_after.txt
```

```bash
diff /tmp/topics_before.txt /tmp/topics_after.txt
```

Responder en `results.md`:

```text
¿Qué nodos nuevos aparecieron?

¿Qué tópicos nuevos aparecieron?

¿Qué nombres parecen corresponder a:
- localization?
- planning?
- control?
```

---

# 33. Evidencias requeridas

## E1 — SSH

Mostrar:

```bash
hostname
whoami
printenv ROS_DISTRO
```

---

## E2 — Git

Mostrar:

```bash
git branch
git status
git log --oneline -5
```

La branch activa deberá ser:

```text
feature/<team>/session01
```

---

## E3 — CARLA

Captura mostrando:

- CARLA ejecutándose;
- vehículo;
- mapa.

---

## E4 — ROS 2 Discovery

Evidencia de:

```bash
ros2 node list
```

y:

```bash
ros2 topic list
```

---

## E5 — Sensores

Tabla completada con al menos:

```text
Camera
LiDAR
IMU
Odometry
Vehicle Control
```

---

## E6 — Movimiento

Evidencia de:

```text
Vehicle moving
Vehicle stopped
```

únicamente en CARLA.

---

## E7 — GitHub

Pull Request desde:

```text
feature/<team>/session01
```

hacia:

```text
main
```

---

# 34. Entregables

Cada equipo deberá dejar una estructura equivalente a:

```text
students/
└── team-a/
    └── session01/
        ├── results.md
        └── evidence/
            ├── nodes.txt
            ├── topics.txt
            ├── ros_environment.txt
            └── system.txt
```

No subir:

- rosbag de gran tamaño;
- datasets;
- grabaciones;
- archivos de CARLA;
- mapas pesados;
- binarios;
- modelos;
- archivos generados automáticamente;

sin autorización del instructor.

---

# 35. Preguntas de análisis

Responder dentro de:

```text
results.md
```

## Q1

¿Cuál es la diferencia entre un **nodo**, un **tópico** y un **tipo de mensaje** en ROS 2?

---

## Q2

Describe el procedimiento utilizado para descubrir cómo controlar el vehículo sin conocer previamente el tópico ni el mensaje utilizado.

---

## Q3

¿Por qué el desarrollo se realiza sobre:

```text
feature/<team>/session01
```

en lugar de modificar directamente:

```text
main
```

?

---

## Q4

Identifica cuatro componentes observados durante la sesión y clasifícalos como:

```text
Sensor
Estimation
Localization
Planning
Control
```

---

# 36. Definition of Done

La parte obligatoria de la sesión está terminada cuando:

- [ ] El equipo logró conectarse por SSH a Pixloop.
- [ ] Se identificó Ubuntu.
- [ ] Se identificó la distribución ROS 2.
- [ ] Se localizó el workspace.
- [ ] Se clonó correctamente el repositorio.
- [ ] Se creó `feature/<team>/session01`.
- [ ] Se realizó al menos un commit.
- [ ] La branch fue publicada en GitHub.
- [ ] CARLA se ejecutó correctamente.
- [ ] El ROS 2 bridge fue levantado.
- [ ] Se ejecutó `ros2 node list`.
- [ ] Se ejecutó `ros2 topic list`.
- [ ] Se identificó la cámara.
- [ ] Se identificó el LiDAR.
- [ ] Se identificó la IMU.
- [ ] Se identificó la odometría.
- [ ] Se identificó el tópico de control.
- [ ] Se inspeccionó al menos un mensaje con `ros2 interface show`.
- [ ] El vehículo se movió en CARLA.
- [ ] Se ejecutó un STOP explícito.
- [ ] Se completó `results.md`.
- [ ] Se guardaron las evidencias.
- [ ] Se realizó commit y push final.
- [ ] Se abrió un Pull Request.

---

# 37. Commit final

Verificar:

```bash
git status
```

Agregar únicamente los archivos correspondientes:

```bash
git add students/team-a/session01
```

Verificar:

```bash
git status
```

Commit:

```bash
git commit -m "docs: complete team-a session01"
```

Push:

```bash
git push
```

---

# 38. Pull Request

En GitHub crear:

```text
feature/team-a/session01
            │
            ▼
           main
```

Título sugerido:

```text
Session 01 - Team A - Pixloop onboarding
```

> No realizar merge directamente.

El Pull Request será revisado antes de integrarse a `main`.

Si el instructor solicita cambios:

```text
review
  ↓
modify files
  ↓
commit
  ↓
push
  ↓
same Pull Request updated automatically
```

---

# 39. Troubleshooting

## SSH no conecta

```bash
ping <PIXLOOP_IP>
```

Después:

```bash
ssh -v <USER>@<PIXLOOP_IP>
```

---

## `ros2: command not found`

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
```

Verificar:

```bash
ros2 --help
```

---

## No aparecen paquetes del workspace

```bash
source <PIXLOOP_WORKSPACE>/install/setup.bash
```

Después:

```bash
ros2 pkg list
```

---

## No aparecen nodos

```bash
ros2 node list
```

Verificar:

```bash
echo $ROS_DOMAIN_ID
```

Confirmar que los procesos correspondientes están activos.

---

## CARLA no responde

```bash
ps aux | grep -i carla
```

Confirmar que el servidor CARLA se ejecutó antes del ROS bridge.

---

## El bridge está activo pero no aparecen tópicos

```bash
ros2 node list
```

```bash
ros2 topic list
```

Revisar la terminal del bridge y buscar errores de conexión.

---

## El vehículo no responde

Primero:

```bash
ros2 topic info <CONTROL_TOPIC> --verbose
```

Confirmar que existe un subscriber.

Después:

```bash
ros2 interface show <MESSAGE_TYPE>
```

Verificar que la estructura enviada corresponda exactamente con el mensaje instalado.

Finalmente ejecutar:

```bash
<TO_CONFIRM_CARLA_STOP_COMMAND>
```

---

## Git rechaza el push

```bash
git branch
```

```bash
git remote -v
```

Después:

```bash
git push -u origin $(git branch --show-current)
```

---

## Se modificó accidentalmente un archivo

Primero:

```bash
git status
```

Después:

```bash
git diff
```

No ejecutar sin autorización:

```bash
git reset --hard
```

ni:

```bash
git clean -fd
```

---

# 40. Actividades opcionales

Las siguientes actividades **no forman parte del Definition of Done** de la Sesión 01.

Pueden realizarse si queda tiempo.

---

## 40.1 Explorar TF

```bash
ros2 topic list | grep tf
```

Si está disponible:

```bash
ros2 run tf2_tools view_frames
```

Pregunta:

> ¿Por qué un vehículo con múltiples sensores necesita conocer las transformaciones espaciales entre sus diferentes sistemas de coordenadas?

TF será estudiado con mayor profundidad en una sesión posterior.

---

## 40.2 Explorar servicios

```bash
ros2 service list
```

---

## 40.3 Explorar actions

```bash
ros2 action list -t
```

---

## 40.4 Explorar parámetros

```bash
ros2 param list
```

---

## 40.5 Visualizar el ROS graph

Si está disponible:

```bash
rqt_graph
```

Identificar visualmente:

```text
Sensors
Localization
Planning
Control
CARLA bridge
```

---

# 41. Reto adicional — Primer nodo ROS 2

> Esta sección solamente deberá realizarse si el equipo terminó todas las actividades obligatorias.

Crear:

```bash
mkdir -p students/team-a/session01/src
```

```bash
touch students/team-a/session01/src/basic_vehicle_control.py
```

El nodo deberá realizar:

```text
START
  │
  ▼
ROS 2 Node
  │
  ▼
Publisher
  │
  ▼
Forward Command
  │
  ▼
Wait
  │
  ▼
STOP
  │
  ▼
Shutdown
```

La implementación dependerá del tipo de mensaje descubierto durante la práctica.

Plantilla:

```python
#!/usr/bin/env python3

import rclpy
from rclpy.node import Node

# TODO:
# Import the message type discovered with:
#
# ros2 topic info <CONTROL_TOPIC>
# ros2 interface show <MESSAGE_TYPE>


class BasicVehicleControl(Node):

    def __init__(self):
        super().__init__('basic_vehicle_control')

        # TODO:
        # self.publisher = self.create_publisher(
        #     MessageType,
        #     '<CONTROL_TOPIC>',
        #     10
        # )

        self.get_logger().info(
            'Basic vehicle control started'
        )

    def publish_forward(self):
        # TODO
        pass

    def publish_stop(self):
        # TODO
        pass


def main(args=None):

    rclpy.init(args=args)

    node = BasicVehicleControl()

    try:
        rclpy.spin(node)

    except KeyboardInterrupt:
        pass

    finally:
        node.publish_stop()
        node.destroy_node()

        if rclpy.ok():
            rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Verificar sintaxis:

```bash
python3 -m py_compile \
students/team-a/session01/src/basic_vehicle_control.py \
&& echo COMPILA_OK
```

---

# 42. Referencia rápida

## Linux

```bash
pwd
ls
ls -la
cd
mkdir
touch
cat
less
grep
find
history
ps aux
```

## Git

```bash
git status
git diff

git branch
git branch -a

git switch main
git pull origin main

git switch -c feature/<team>/session01

git add <FILE>
git commit -m "message"

git push -u origin feature/<team>/session01
git push

git log --oneline --decorate -10
git remote -v
```

## ROS 2

```bash
ros2 node list
ros2 node info <NODE>

ros2 topic list
ros2 topic info <TOPIC>
ros2 topic info <TOPIC> --verbose
ros2 topic echo <TOPIC>
ros2 topic hz <TOPIC>

ros2 interface show <MESSAGE_TYPE>
```

---

# 43. Flujo esperado de la sesión

```text
Connect to Pixloop
        │
        ▼
Inspect Linux + ROS 2
        │
        ▼
Locate workspace
        │
        ▼
Clone repository
        │
        ▼
Create feature branch
        │
        ▼
First commit
        │
        ▼
Start CARLA
        │
        ▼
Start ROS 2 bridge
        │
        ▼
ros2 node list
ros2 topic list
        │
        ▼
Discover sensors
        │
        ▼
Discover control topic
        │
        ▼
Inspect message
        │
        ▼
Move vehicle in CARLA
        │
        ▼
STOP
        │
        ▼
Observe Pixloop stack
        │
        ▼
Complete results.md
        │
        ▼
Commit + Push
        │
        ▼
Pull Request
```

---

# 44. Próxima sesión

## Sesión 02 — 10 de octubre de 2026

En la siguiente sesión se profundizará en:

```text
ROS 2 graph
Publishers
Subscribers
Sensors
TF
Frames
RViz / Foxglove
Sensor visualization
```

El objetivo será pasar de **descubrir que los componentes existen** a comprender **cómo se relacionan espacialmente y cómo fluye la información entre ellos**.

---

# Instructor Checklist

Antes de iniciar la práctica verificar:

- [ ] `PIXLOOP_IP`
- [ ] usuario SSH
- [ ] autenticación SSH
- [ ] versión Ubuntu
- [ ] ROS 2 distro
- [ ] `ROS_DOMAIN_ID`
- [ ] ruta del workspace Pixloop
- [ ] URL del repositorio GitHub
- [ ] protección de `main`
- [ ] acceso de los estudiantes al repositorio
- [ ] versión CARLA
- [ ] ruta de CARLA
- [ ] comando de inicio de CARLA
- [ ] workspace del ROS bridge
- [ ] comando del ROS bridge
- [ ] role name del vehículo
- [ ] tópico de cámara
- [ ] tópico de LiDAR
- [ ] tópico de IMU
- [ ] tópico de odometría
- [ ] tópico de control
- [ ] tipo de mensaje de control
- [ ] comando de avance probado
- [ ] comando de STOP probado
- [ ] comando(s) para levantar el stack completo
- [ ] CARLA probado antes de la sesión
- [ ] movimiento y STOP probados antes de la sesión
- [ ] workflow de branch → push → Pull Request probado
