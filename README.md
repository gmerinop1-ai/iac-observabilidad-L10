# Laboratorio de Monitoreo - Iac10

En este laboratorio exploraremos monitoreo con herramientas disponibles

### Pre-requisitos

- Docker Engine + Docker Compose v2
- Puertos libres en el host: `3000`, `3001`, `8080`, `8081`, `9090`, `9100`, `3100`, `12345`

### 1. Levantar el stack

```bash
cd iac-observabilidad
docker compose up -d --build
```

Espera ~30 segundos y verifica el estado:

```bash
docker compose ps
```
![Contenedores corriendo](/home/zalethpg/iac-observabilidad/capturas/contenedores-levantados.png)

`backend`, `frontend`, `prometheus`, `loki` y `cadvisor` deben mostrar `Up (healthy)`. `alloy`, `grafana` y `node-exporter` muestran solo `Up`

### 2. Verificar las apps y Prometheus

## Servicios y URLs
| Servicio       | URL                         | Notas                                  |
|----------------|-----------------------------|----------------------------------------|
| Frontend       | http://localhost:8080       | Hello World + botones de tráfico/carga |
| Backend (API)  | http://localhost:3001       | `/api/hello`, `/metrics`, `/load`      |
| Grafana        | http://localhost:3000       | admin / admin                          |
| Prometheus     | http://localhost:9090       | datasource ya provisionado             |
| Loki           | http://localhost:3100       | datasource ya provisionado             |
| Alloy (UI)     | http://localhost:12345      | estado del recolector de logs          |
| cAdvisor       | http://localhost:8081       | métricas por contenedor                |
| node-exporter  | http://localhost:9100/metrics | métricas del host                    |

### 3. Verificar las datasources de Grafana

1. Entra a `http://localhost:3000` con `admin` / `admin` (Skip si pide cambiar contraseña).
2. Ve a **Connections → Data sources**.
3. Deben aparecer **Prometheus** y **Loki**, ambas provisionadas automáticamente (sin haberlas creado a mano).

### 4. Detener los contenedores (conservando dashboards)
```bash
docker compose down
```
![Contenedores Detenidos con dashboards](/home/zalethpg/iac-observabilidad/capturas/contenedores-stop-con-dashoard.png)

### 5. Detener los contenedores (borrando dashboards y alarmas)
```bash
docker compose down -v   # borra también dashboards/alarmas creados
```
![Contenedores detenidos totalmente](/home/zalethpg/iac-observabilidad/capturas/contenedores-stop.png)
