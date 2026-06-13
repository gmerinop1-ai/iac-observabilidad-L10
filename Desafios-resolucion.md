# Desafíos y resolución — Laboratorio de Observabilidad

Este documento registra los problemas reales que encontré al ejecutar el
laboratorio, el proceso de diagnóstico que seguí en cada caso y la solución
que apliqué. No todos los errores eran fallos del stack — algunos eran ruido
esperado que era necesario saber interpretar.

---

## Desafío 1 — cAdvisor no generaba métricas de contenedores (el más crítico)

### Síntoma
Al crear el panel 8.1 con la query de la guía, Grafana mostraba "No data".
El stack estaba levantado y `docker compose ps` reportaba todos los servicios
como `Up`, pero ninguna query con `{name="lab-backend"}` devolvía resultados.

### Diagnóstico
Antes de modificar la query asumí que el problema podía ser el label `name`.
Fui directamente a Prometheus para ver qué métricas existían realmente:

```bash
curl -s "http://localhost:9090/api/v1/query?query=container_cpu_usage_seconds_total"
```

La métrica existía, pero todos los resultados tenían
`id="/system.slice/..."` — eran cgroups del sistema, ninguno correspondía a
un contenedor del lab. Luego revisé los logs de cAdvisor:

```bash
docker compose logs cadvisor | head -30
```

El error se repetía para cada contenedor:

```
Failed to create existing container: /system.slice/docker-<id>.scope:
failed to identify the read-write layer ID for container "<id>".
open /rootfs/var/lib/docker/image/overlayfs/layerdb/mounts/<id>/mount-id:
no such file or directory
```

Inspeccioné si esa ruta existía en mi sistema:

```bash
ls /var/lib/docker/image/overlayfs/layerdb/mounts/
```

El directorio no existía en absoluto. Busqué dónde estaban las capas realmente:

```bash
mount | grep overlay
```

Todos los mounts overlay apuntaban a
`/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/`.

### Causa raíz
Mi instalación de Docker Engine en Manjaro tenía activado el
**containerd image store** (`containerd-snapshotter: true`). Este modo gestiona
las capas de imágenes a través de snapshots de containerd en lugar del driver
`overlay2` legacy. cAdvisor v0.49.1 tiene hardcodeada la ruta
`/var/lib/docker/image/overlayfs/layerdb/` para mapear cada contenedor a su
capa de escritura. Bajo containerd image store esa ruta nunca se crea, por lo
que cAdvisor detecta los contenedores vía cgroups pero no puede construir sus
handlers completos. El resultado es que no emite métricas con `name="lab-backend"`
ni de ningún otro contenedor del lab.

### Solución

Deshabilitar el containerd image store para forzar a Docker a usar el driver
`overlay2` legacy:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "features": {
    "containerd-snapshotter": false
  }
}
EOF

cd /home/zalethpg/iac-observabilidad
docker compose down
sudo systemctl restart docker
docker compose up -d --build
```

### Verificación

```bash
# Cero errores de "failed to create" en cAdvisor
docker compose logs cadvisor 2>&1 | grep -c "failed to create"
# → 0

# Los contenedores del lab ya aparecen con su nombre real
curl -s "http://localhost:9090/api/v1/query?query=container_last_seen" \
  | grep -o '"name":"[^"]*"' | sort -u
# → "name":"lab-backend", "name":"lab-frontend", etc.
```

### Nota sobre la query
Verifiqué en Prometheus que en mi sistema el label `name` reporta
`"lab-backend"` sin slash inicial. La query original de la guía es correcta
para mi entorno. Documenté este proceso de verificación porque en Linux el
valor puede variar entre versiones de cAdvisor.

---

## Desafío 2 — Alloy no enviaba logs a Loki (configuración eliminada intencionalmente)

### Síntoma
Los paneles 8.3 y 8.4 aparecían vacíos aunque el contenedor `lab-alloy`
figuraba como `Up`. Prometheus funcionaba, las métricas llegaban, pero Loki
no recibía ningún stream de logs.

### Diagnóstico

```bash
docker compose logs alloy | head -20
```

El log mostraba errores de acceso al socket:

```
level=error msg="error connecting to Docker"
err="permission denied while accessing /var/run/docker.sock"
```

Revisé los permisos del socket en mi sistema:

```bash
ls -la /var/run/docker.sock
# srw-rw---- 1 root docker  /var/run/docker.sock
```

El socket solo permite acceso a `root` o al grupo `docker` del host. El
contenedor `grafana/alloy:v1.10.0` corre por defecto como el usuario `alloy`
(no root). Ese usuario dentro del contenedor no pertenece al grupo `docker` del
host — los GIDs del host y del contenedor son independientes.

### Causa raíz
El bloque del servicio `alloy` en el `docker-compose.yml` original no tenía
`user: root`. Esta es la **configuración que fue eliminada intencionalmente**
del archivo entregado por el docente. El efecto es silencioso: Alloy arranca,
aparece como `Up`, pero no puede acceder al socket para descubrir contenedores.
El único síntoma visible es que los paneles de logs permanecen vacíos.

### Solución

Agregar `user: root` al servicio `alloy` en `docker-compose.yml`:

```yaml
alloy:
  image: grafana/alloy:v1.10.0
  user: root          # esto fue lo que se eliminó intencionalmente
  container_name: lab-alloy
  volumes:
    - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
    - /var/run/docker.sock:/var/run/docker.sock:ro
```

---

## Desafío 3 — Grafana arrancaba sin datasources (problema de estructura de carpetas)

### Síntoma
Al entrar a `Connections → Data sources` en Grafana aparecía
"No data sources defined", aunque el archivo `datasources.yml` existía en el
proyecto.

### Diagnóstico

```bash
docker compose exec grafana ls /etc/grafana/provisioning/datasources/
# ls: /etc/grafana/provisioning/datasources/: No such file or directory
```

El directorio `datasources/` no existía dentro del contenedor. Inspeccioné la
estructura real del archivo en mi máquina:

```bash
find /home/zalethpg/iac-observabilidad/grafana -type f
# /home/zalethpg/iac-observabilidad/grafana/provisioning/datasources.yml
```

El archivo estaba en `provisioning/datasources.yml` directamente, sin la
subcarpeta `datasources/` que Grafana requiere.

### Causa raíz
Grafana no lee archivos de provisioning sueltos en `/etc/grafana/provisioning/`.
Los busca en subdirectorios tipados: `provisioning/datasources/`,
`provisioning/dashboards/`, `provisioning/alerting/`, etc. El archivo había
quedado un nivel arriba de donde debía estar.

### Solución

```bash
mkdir -p /home/zalethpg/iac-observabilidad/grafana/provisioning/datasources
mv /home/zalethpg/iac-observabilidad/grafana/provisioning/datasources.yml \
   /home/zalethpg/iac-observabilidad/grafana/provisioning/datasources/datasources.yml

docker compose down -v
docker compose up -d --build
```

El `-v` fue necesario para eliminar el volumen `grafana-data` que ya tenía el
estado sin datasources persistido. Sin `-v`, Grafana reutilizaba el estado
guardado aunque el archivo estuviera ahora en la ruta correcta.

---

## Desafío 4 — Healthchecks faltantes y `/healthz` sin usar

### Observación al leer el código

En `apps/backend/server.js` encontré esta línea:

```javascript
app.get('/healthz', (req, res) => res.json({ status: 'ok' }));
```

El mismo endpoint existe en `apps/frontend/server.js`. Sin embargo, el
`docker-compose.yml` original no tenía ningún `healthcheck` definido, ni
usaba `condition: service_healthy` en los `depends_on`. El endpoint existía
pero nadie lo usaba.

### El problema concreto
Sin healthcheck, Docker considera un contenedor "listo" en cuanto su proceso
arranca — aunque Node.js todavía esté cargando módulos. El frontend puede
intentar hacer llamadas al backend antes de que este esté escuchando en el
puerto 3001. Esto genera errores transitorios de conexión al inicio que
aparecen en los logs.

### Solución aplicada

Agregué healthchecks a todos los servicios que exponen endpoints de salud:

```yaml
backend:
  healthcheck:
    test: ["CMD-SHELL", "wget -qO- http://localhost:3001/healthz || exit 1"]
    interval: 10s
    timeout: 5s
    retries: 3
    start_period: 15s   # Node.js necesita ~5-10s para cargar módulos
```

Usé `wget` en lugar de `curl` porque la imagen base `node:20-alpine` incluye
`wget` por defecto pero no `curl`. Poner `curl` habría roto el healthcheck con
`command not found`.

Para Loki usé `start_period: 30s` porque inicializa un índice TSDB al arrancar
que toma más tiempo que un servidor HTTP simple.

Actualicé el `depends_on` del frontend para usar `condition: service_healthy`,
garantizando que el proxy al backend no intenta conectarse antes de que el
healthcheck pase.

El resultado visible: `docker compose ps` ahora muestra `Up (healthy)` en lugar
de solo `Up`.

---

## Desafío 5 — Revisión de volúmenes de cAdvisor en Manjaro

La guía indicaba revisar si todos los volúmenes del servicio `cadvisor` eran
utilizables. Los verifiqué en mi sistema:

```bash
ls -la /var/run/docker.sock  # → existe, root:docker
ls -la /dev/disk             # → existe, con subdirectorios by-id, by-uuid, by-path
ls -la /dev/kmsg             # → existe, character device
```

Resultado en mi Manjaro con Docker Engine estándar:

| Volumen/Device | Existe | Función |
|---|---|---|
| `/:/rootfs:ro` | ✅ | Sistema de archivos raíz para métricas de disco |
| `/var/run:/var/run:ro` | ✅ | Contiene el socket de Docker |
| `/sys:/sys:ro` | ✅ | Interfaces de cgroups del kernel (CPU/memoria por proceso) |
| `/var/lib/docker/:/var/lib/docker:ro` | ✅ | Metadatos de contenedores |
| `/dev/disk/:/dev/disk:ro` | ✅ | Métricas de disco por dispositivo (by-id, by-uuid) |
| `/dev/kmsg` (device) | ✅ | Buffer de mensajes del kernel |

No fue necesario eliminar ningún volumen. En sistemas sin Docker Engine estándar
(por ejemplo sin `/var/lib/docker`) o sin batería física (`/dev/disk` vacío)
habría que eliminar las líneas problemáticas para que cAdvisor arranque.

---

## Desafío 6 — Errores `powersupplyclass` en el panel de logs de infraestructura

### Lo que aparecía

El panel "Logs de infraestructura" mostraba errores repetidos de node-exporter:

```
level=error name=powersupplyclass
err="could not get power_supply class info:
failed to read file \"/sys/class/power_supply/BAT0/power_now\": no such device"
```

### Diagnóstico
Node-exporter intenta recolectar **todos** los tipos de métricas disponibles,
incluyendo el estado de la batería (`powersupplyclass`). Mi equipo no tiene
batería — `BAT0` no existe en `/sys/class/power_supply/`. Node-exporter detecta
la ausencia del archivo y registra el fallo, pero continúa funcionando para el
resto de colectores (CPU, memoria, disco, red).

### Conclusión
Estos errores son comportamiento esperado en un equipo de escritorio o sin
batería. No afectan las métricas de CPU ni memoria que usa el lab. El panel
de logs de infraestructura funciona correctamente — precisamente está mostrando
actividad real de los componentes del stack.

---

## Desafío 7 — El log del webhook no aparece donde dice la guía

La guía indica que al configurar el webhook, el log `grafana_alert_received`
aparecerá en el panel **"Logs de infraestructura"**. Eso es incorrecto.

El backend tiene `tier=application` según las reglas de relabeling en
`alloy/config.alloy`. Cualquier log que escriba el backend — incluyendo los que
genera al recibir el webhook — es etiquetado como `tier=application` por Alloy
y va a Loki con esa etiqueta. Por tanto, `grafana_alert_received` aparece en el
panel **"Logs de aplicación"** (query `{tier="application"}`), no en
infraestructura.

Verificación:

```bash
# Ver el log llegando en tiempo real
docker compose logs -f backend | grep -i "alert"
```

En Grafana, filtrar en el panel 8.3:

```logql
{tier="application"} | json | msg="grafana_alert_received"
```
