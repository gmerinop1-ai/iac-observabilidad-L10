# Respuestas a las preguntas del laboratorio

## Pregunta 1 — ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

Al ejecutar el lab quedó claro que Prometheus mide, Loki narra. Son complementarios porque las
métricas cuantifican el comportamiento y los logs explican las causas.

Prometheus almacena **series de tiempo numéricas**. Su modelo de datos es
`{etiqueta=valor} → [(timestamp, número)]`. Puede decirme cuántas peticiones
recibió el backend en el último minuto, cuál fue la latencia promedio, o qué
porcentaje de CPU está consumiendo. Todo son números que cambian con el tiempo.

Lo que Prometheus **no puede** almacenar es texto narrativo. Cuando el backend
ejecuta el escenario `fallo_conexion_inventario`, escribe esto a stdout:

Loki sí: almacena streams de texto etiquetados y permite consultarlos con LogQL para filtrar por nivel, por
servicio, o por cualquier campo del JSON.


## Pregunta 2 — ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

La ventaja concreta del provisioning como código es la **Reproducibilidad sin pasos manuales.** 
El archivo `grafana/provisioning/datasources/datasources.yml` define que Grafana debe
conectarse a `http://prometheus:9090` y a `http://loki:3100`. Cualquier persona
que clone el repositorio y ejecute `docker compose up` obtiene exactamente el
mismo entorno, sin necesidad de recordar que hay que ir a
`Connections → Data sources → Add data source` y rellenar los campos
manualmente.

## Pregunta 3 — Los paneles "CPU contenedor" y "CPU host" muestran valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

Para alertar sobre una aplicación concreta: siempre la métrica del
contenedor. La CPU del host puede subir por procesos ajenos a la aplicación
(cAdvisor, Prometheus, node-exporter). Con la métrica del contenedor, la
alarma apunta exactamente al servicio que se quiere monitorear.


## Pregunta 4 — ¿Qué diferencia hay entre el *evaluation interval* y el *pending period* de una alarma?

Son dos controles distintos que actúan en momentos diferentes.

**Evaluation interval (10s):** cada cuánto Grafana **evalúa** la condición de
la regla. Cada 10 segundos, Grafana ejecuta la query PromQL configurada,
obtiene el valor actual de la CPU del backend y comprueba si supera el umbral.
Es la frecuencia del chequeo — como revisar el termómetro cada 10 segundos.

**Pending period (30s):** cuánto tiempo la condición debe mantenerse
**verdadera de forma consecutiva** antes de que la alarma pase a estado
`Firing`. Cuando la CPU supera el umbral por primera vez, la alarma entra en
`Pending`. Si en las siguientes evaluaciones (a los 10s, 20s, 30s) la CPU
sigue sobre el umbral, recién entonces pasa a `Firing`. Si en alguna de esas
evaluaciones la CPU baja, la alarma vuelve a `Normal` y el contador se reinicia.
