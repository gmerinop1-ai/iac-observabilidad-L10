# Respuestas a las preguntas del laboratorio

**Curso:** Infraestructura como Código  
**Alumna:** Grezia Zaleth Merino Pères 
**Docente:** Ing. Walter Leturia Rodriguez

---

## Pregunta 1 — ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

Durante el laboratorio tuve dos fuentes de información sobre el backend: el
endpoint `/metrics` (recogido por Prometheus) y los logs JSON que escribe a
stdout (recogidos por Alloy y almacenados en Loki). Al ejecutar el lab quedó
claro que Prometheus mide, Loki narra. Son complementarios porque las
métricas cuantifican el comportamiento y los logs explican las causas.

Prometheus almacena **series de tiempo numéricas**. Su modelo de datos es
`{etiqueta=valor} → [(timestamp, número)]`. Puede decirme cuántas peticiones
recibió el backend en el último minuto, cuál fue la latencia promedio, o qué
porcentaje de CPU está consumiendo. Todo son números que cambian con el tiempo.

Lo que Prometheus **no puede** almacenar es texto narrativo. Cuando el backend
ejecuta el escenario `fallo_conexion_inventario`, escribe esto a stdout:

Loki sí: almacena streams de texto etiquetados y permite consultarlos con LogQL para filtrar por nivel, por
servicio, o por cualquier campo del JSON.

La diferencia se hizo visible al cerrar el ciclo en el paso 11: cuando la
alarma de CPU se disparó, Grafana envió el webhook al backend y este escribió:

```json
{"level":"ERROR","msg":"grafana_alert_received","alert_status":"firing"}
```

Ese evento apareció en el panel "Logs de aplicación" minutos después de que
el panel de CPU mostrara el pico. Prometheus me dijo *cuándo* subió la CPU;
Loki me dijo *qué hizo el sistema* como consecuencia de ese pico.

---

## Pregunta 2 — ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

El archivo grafana/provisioning/datasources/datasources.yml define las
conexiones a Prometheus y Loki. Grafana lo lee al arrancar y las configura
automáticamente.

La ventaja concreta del provisioning como código es la **Reproducibilidad sin pasos manuales.** 
El archivo `grafana/provisioning/datasources/datasources.yml` define que Grafana debe
conectarse a `http://prometheus:9090` y a `http://loki:3100`. Cualquier persona
que clone el repositorio y ejecute `docker compose up` obtiene exactamente el
mismo entorno, sin necesidad de recordar que hay que ir a
`Connections → Data sources → Add data source` y rellenar los campos
manualmente. Si ese paso se olvida, los paneles del dashboard aparecen sin
datos pero sin ningún mensaje de error claro — simplemente parece que las
queries fallan.

**Consistencia entre entornos.** En un proyecto real con entornos de
desarrollo, QA y producción, el provisioning como código garantiza que los
tres entornos tienen exactamente los mismos datasources configurados.
Crearlos a mano en cada entorno introduce posibilidad de errores humanos
y divergencia entre configuraciones.

---

## Pregunta 3 — Los paneles "CPU contenedor" y "CPU host" muestran valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

Esta diferencia quedó muy clara en el laboratorio al generar carga con:

```bash
curl "http://localhost:3001/load?seconds=60"
```

El panel **CPU contenedor** mostró un pico significativo. El panel **CPU host**
mostró un incremento mucho menor en el mismo período. Ambos paneles estaban
mirando la misma máquina al mismo tiempo — ¿por qué valores tan distintos?

**Por qué difieren:**

La query del panel de CPU del host es:

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

Esta expresión calcula el promedio de CPU ocupada sobre **todos los cores** de
la máquina. Si mi equipo tiene 8 cores y solo uno está al 100% por el test de
carga del backend, el promedio del host muestra `100 / 8 = 12.5%`. El host
"apenas se movió" aunque un proceso estuviera quemando un core completo.

La query del panel de CPU del contenedor es:

```promql
sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
```

Esta expresión mide exactamente el tiempo de CPU consumido **por ese contenedor
específico**, expresado en porcentaje de un core (100 = un core completo
utilizado). El test de carga del backend usa un worker thread que quema cálculos
matemáticos en un bucle, saturando un core. Por eso este panel subió cerca
de 100 mientras el host apenas cambió.

Durante el diagnóstico del problema de cAdvisor (desafío 1) también pude
observar cómo cAdvisor usa los cgroups del kernel para aislar exactamente qué
CPU consume cada contenedor, independientemente de cuántos cores tenga el host.

**¿Cuál usar para alertar sobre una aplicación concreta?**

Para alertar sobre una aplicación concreta: siempre la métrica del
contenedor. La CPU del host puede subir por procesos ajenos a la aplicación
(cAdvisor, Prometheus, node-exporter). Con la métrica del contenedor, la
alarma apunta exactamente al servicio que se quiere monitorear.
---

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

Durante el laboratorio pude observar la transición completa al generar carga:

```
Normal → [CPU supera umbral] → Pending → [30s sostenido] → Firing
```

El dashboard de `Alerting → Alert rules` mostraba primero el estado `Pending`
y aproximadamente 30-40 segundos después pasaba a `Firing`. Capturé ambos
estados porque la transición `Pending` demuestra que el pending period está
funcionando.
