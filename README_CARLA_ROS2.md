# CARLA + ROS 2 Humble — Inicio rápido

Mini tutorial para iniciar CARLA y conectarlo con ROS 2. Mantén abiertas las terminales de los procesos activos.

## Requisitos

- Ubuntu 22.04
- CARLA en `~/CARLA`
- ROS 2 Humble
- Bridge compilado en `~/Workspace/ros-bridge`
- API de Python de CARLA en `~/carla-env`

## 1. Terminal 1 — Iniciar CARLA

Abre una terminal (`Ctrl + Alt + T`) y ejecuta:

```bash
~/CARLA/CarlaUE4.sh
```

Espera a que cargue el mapa. **Deja esta terminal abierta.**

## 2. Terminal 2 — Iniciar ROS 2 Bridge

Abre una segunda terminal y ejecuta:

```bash
source /opt/ros/humble/setup.bash
source ~/Workspace/ros-bridge/install/setup.bash
export PYTHONPATH="$HOME/carla-env/lib/python3.10/site-packages:$PYTHONPATH"

ros2 launch carla_ros_bridge carla_ros_bridge.launch.py
```

Este proceso conecta ROS 2 con CARLA. **Deja la terminal abierta.**

## 3. Terminal 3 — Crear el vehículo `hero`

Abre una tercera terminal y ejecuta:

```bash
source /opt/ros/humble/setup.bash
source ~/Workspace/ros-bridge/install/setup.bash
export PYTHONPATH="$HOME/carla-env/lib/python3.10/site-packages:$PYTHONPATH"

ros2 launch carla_spawn_objects carla_example_ego_vehicle.launch.py
```

Espera unos segundos. Este lanzador intenta crear/configurar el vehículo principal (`hero`). **Deja la terminal abierta** mientras necesites sus nodos.

## 4. Terminal opcional — Comprobar vehículos y centrar la cámara

Si el vehículo no se ve, abre otra terminal y ejecuta:

```bash
source ~/carla-env/bin/activate

python - <<'PY'
import carla

client = carla.Client('127.0.0.1', 2000)
client.set_timeout(10)
world = client.get_world()

vehicles = world.get_actors().filter('vehicle.*')
print(f"Vehículos encontrados: {len(vehicles)}")

for v in vehicles:
    print(f"ID={v.id} | tipo={v.type_id} | rol={v.attributes.get('role_name', 'sin rol')}")
    if v.attributes.get('role_name') == 'hero':
        t = v.get_transform()
        world.get_spectator().set_transform(
            carla.Transform(
                t.location + carla.Location(z=12),
                carla.Rotation(pitch=-90)
            )
        )
        print("Cámara colocada sobre hero")
PY
```

Este script solo consulta los vehículos y centra la cámara sobre `hero` si lo encuentra. Puedes cerrar esta terminal cuando termine.

## Verificación rápida

En una terminal nueva:

```bash
source /opt/ros/humble/setup.bash
source ~/Workspace/ros-bridge/install/setup.bash
ros2 topic list
```

Si el vehículo está registrado como `hero`, pueden aparecer tópicos como:

```text
/carla/hero/vehicle_control_cmd
/carla/hero/vehicle_control_cmd_manual
/carla/hero/vehicle_control_manual_override
```

## Cerrar la sesión

1. En la Terminal 3, pulsa `Ctrl + C`.
2. En la Terminal 2, pulsa `Ctrl + C`.
3. Cierra CARLA normalmente.

No hace falta reinstalar ni reconstruir el bridge para cada sesión. Si un comando falla, revisa el error antes de continuar.
