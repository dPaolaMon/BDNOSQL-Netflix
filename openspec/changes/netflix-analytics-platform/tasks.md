# Tasks

Responsables: **DM** = Diana Paola Montes · **JC** = Jonathan Cervantes · **FB** = Fernanda Baez · **XJ** = Ximena Jasso. El calendario está en `cronograma.md`.

## 1. Infraestructura local (DM)

- [ ] 1.1 Crear `docker-compose.yml` con `cassandra`, `cassandra-init`, `rabbitmq`, `etl`, `ingestion` y `web`, con healthchecks y `depends_on` por condición. Verificar: `docker compose up -d` deja `cassandra` y `rabbitmq` en estado healthy.
- [ ] 1.2 Añadir `rabbitmq/rabbitmq.conf` con `collect_statistics_interval = 1000`. Verificar que el panel `:15672` muestra tasas que se actualizan cada segundo.
- [ ] 1.3 Escribir `cassandra/schema.cql` (keyspace `netflix_mart`, UDT `credit`, tablas de consulta, agregados y control, según design D6). Verificar: `cassandra-init` termina con código 0 y reejecutarlo no da error.
- [ ] 1.4 Documentar arranque y puertos en el README del repositorio. Verificar: los comandos documentados funcionan tal cual en una máquina limpia.

## 2. ETL en Python (DM)

- [ ] 2.1 Crear el paquete `etl/netflix_etl` con `pyproject.toml`/`requirements.txt` (`pandas`, `pika`, `pytest`) y su `Dockerfile`. Verificar: `docker compose build etl` funciona.
- [ ] 2.2 Implementar `extract.py` con lectura robusta de multilínea y comillas. Verificar con pytest: 6,137 títulos y 81,355 créditos leídos, y un error claro si falta un archivo.
- [ ] 2.3 Implementar `transform.py` con las reglas de D15 (colecciones, nulos, rangos, NR, seasons, runtime 0, década, créditos) y la validación con rechazados. Verificar con pytest un caso por regla, más "0 rechazados en el dataset original".
- [ ] 2.4 Embeber los créditos en cada título. Verificar con pytest: `tm82169` incluye a Sylvester Stallone como ACTOR y `decade = 1970`, y un título sin créditos queda con lista vacía.
- [ ] 2.5 Implementar `aggregate.py` (kpis, by_year, genre_summary, genre_decade, country, age_certification, top_people, ranking con umbral de votos). Verificar con pytest los KPIs (6,137 / 3,831 / 2,306 / 56,754) y la exclusión del ranking por votos.
- [ ] 2.6 Generar `quality_report.json` y `rejected.csv`. Verificar: existen después de una corrida y los conteos cuadran con los registros leídos.
- [ ] 2.7 Documentar las reglas de limpieza y el uso de la CLI en `etl/README.md`. Verificar que los ejemplos de comandos corren.

## 3. Publicación y mensajería (DM + JC)

- [ ] 3.1 (DM) Implementar `publish.py`: confirms, `mandatory`, `delivery_mode=2`, `message_id` uuid5 por corrida+título, header `x-run-id`, 3 reintentos y `rejected_publish.jsonl`. Verificar con pytest + RabbitMQ de compose: todos los mensajes quedan confirmados y la corrida devuelve exit 0.
- [ ] 3.2 (DM) Implementar la secuencia de corrida (`run.started` → títulos → agregados → `run.completed`) y la opción `--rate`. Verificar que a 60 msg/s la corrida dura unos 100 s y el orden de routing keys es el esperado.
- [ ] 3.3 (DM) Implementar el modo `listen` sobre `etl-commands`, que rechaza un comando si ya hay una corrida activa. Verificar: un comando dispara una corrida y un segundo comando concurrente se descarta con log.
- [ ] 3.4 (JC) Declarar la topología (exchange `netflix.catalog` topic y colas quorum con bindings, D4) y documentarla en `docs/topologia-rabbitmq.md` con captura del panel. Verificar en `:15672` que existen exchange, colas y bindings.

## 4. Capa de datos .NET (FB)

- [ ] 4.1 Crear la solución con `Netflix.Contracts`, `Netflix.Core`, `Netflix.Ingestion`, `Netflix.Web` y sus proyectos de test (.NET 10). Verificar: `dotnet build` en verde.
- [ ] 4.2 Definir los records de mensajes en `Netflix.Contracts` con `JsonPropertyName` snake_case. Verificar con xUnit: deserializa un JSON de ejemplo generado por el ETL.
- [ ] 4.3 Implementar la sesión Cassandra compartida (conexión con reintento y backoff, sentencias preparadas, mapeo de UDT `credit`). Verificar: un test de integración lee y escribe `titles_by_id` con créditos.
- [ ] 4.4 Implementar el CQRS propio (`IQuery`, `IQueryHandler`, `IQueryDispatcher`, `ICommand`, `ICommandHandler` y registro por escaneo). Verificar con xUnit: el dispatcher resuelve el handler correcto y falla con un error claro si no existe.
- [ ] 4.5 Implementar los repositorios de lectura para cada tabla de consulta y de agregados. Verificar con tests de integración contra Cassandra de compose: cada consulta lee una sola partición y devuelve el orden esperado.

## 5. Ingesta transaccional .NET (JC)

- [ ] 5.1 Configurar MassTransit 8.x en `Netflix.Ingestion`: endpoints raw JSON, `ConfigureConsumeTopology=false`, bindings por routing key, `PrefetchCount=16`, `ConcurrentMessageLimit=8`. Verificar que los consumidores aparecen conectados en `:15672`.
- [ ] 5.2 Implementar `TitleIngestedConsumer` (validación → LWT en `processed_messages` → LOGGED BATCH de vistas → créditos en paralelo → progreso → eventos, D8). Verificar con el test harness: un título nuevo queda en todas sus tablas.
- [ ] 5.3 Implementar la idempotencia y la reversión del registro ante fallas. Verificar con tests: un duplicado publica `DuplicateSkipped` sin escribir, y una falla en el batch borra el registro para que el reintento procese.
- [ ] 5.4 Configurar los reintentos exponenciales (3) y `Ignore<ValidationException>`. Verificar con tests: un corrupto va directo a `_error` y una falla transitoria ×2 termina almacenada con 2 eventos de reintento.
- [ ] 5.5 Implementar `AggregateSnapshotConsumer` y `RunControlConsumer` (estado y completitud de la corrida). Verificar con tests: los snapshots hacen upsert y la corrida pasa a COMPLETED al cumplir las condiciones.
- [ ] 5.6 Implementar el consumidor de `Fault<TitleIngested>`, que publica `MessageFailed` con el motivo. Verificar con test: un mensaje en `_error` produce el evento.
- [ ] 5.7 Implementar pausa/reanudación por `ingestion-control` con `IReceiveEndpointConnector`. Verificar en manual: con el consumidor pausado `messages_ready` crece y al reanudar llega a 0.
- [ ] 5.8 Implementar la falla transitoria de demo (header `x-demo-transient-failures`). Verificar con test: un mensaje con valor 2 se almacena al tercer intento.

## 6. API y tiempo real (FB)

- [ ] 6.1 Implementar los query handlers del dashboard (KPIs, años, clasificación, géneros, heatmap, género→títulos, países, país→títulos, ranking, detalle, top personas, filmografía, estado de ingesta). Verificar con xUnit sobre datos de prueba.
- [ ] 6.2 Implementar `SearchCatalog` con caché en memoria, búsqueda sin distinguir mayúsculas ni acentos, filtros, orden, paginación e invalidación al recibir `RunStateChanged(COMPLETED)`. Verificar con xUnit: "rocky" encuentra Rocky y la caché se invalida al recibir el evento.
- [ ] 6.3 Exponer la Minimal API `/api/...` con validación (400), 404 con ProblemDetails y OpenAPI/Swagger en desarrollo. Verificar con tests de integración (`WebApplicationFactory`) de cada endpoint y sus códigos.
- [ ] 6.4 Implementar los comandos `/api/demo/*` (202, 404 con la demo deshabilitada, 409 con corrida activa) publicando a `etl-commands`, `ingestion-control` y al exchange, más `RetryDeadLetters` con `basic.get`. Verificar con tests de integración y el panel de RabbitMQ.
- [ ] 6.5 Implementar el hub SignalR `/hubs/live` y el consumidor de eventos de ingesta en Web (cola temporal), con agrupación cada 250 ms. Verificar con un cliente de prueba que recibe `TitleStored` en menos de 1 s.
- [ ] 6.6 Implementar `QueueStatsPoller` (Management HTTP API cada 1 s, estado "no disponible" ante falla). Verificar: si se detiene RabbitMQ, las métricas pasan a no disponibles y `/api/kpis` sigue respondiendo.
- [ ] 6.7 Añadir health checks (`/health`) de Cassandra y RabbitMQ con arranque tolerante. Verificar: si se arranca `web` antes que Cassandra, la salud reporta "unhealthy" y luego "healthy" sin reiniciar.

## 7. Dashboard Blazor (XJ)

- [ ] 7.1 Configurar MudBlazor, el tema oscuro (tokens de D12), la fuente Inter y el layout con barra lateral, barra superior e indicador en vivo. Verificar: la navegación funciona entre las 7 rutas y el contraste cumple AA.
- [ ] 7.2 Crear componentes compartidos: tarjeta KPI, contenedor de gráfica con estados de carga, vacío y error, y drawer de detalle de título. Verificar visualmente cada estado forzando datos vacíos y errores.
- [ ] 7.3 Vista Resumen (KPIs, área por año, donas tipo y clasificación, barras de géneros) con actualización en vivo. Verificar que muestra 6,137 títulos y que los KPIs se animan al terminar una corrida.
- [ ] 7.4 Vista Catálogo con `MudDataGrid` (búsqueda, filtros, orden, paginación) y detalle. Verificar: "rocky" + Películas muestra Rocky y el detalle muestra IMDb 8.1 y el reparto.
- [ ] 7.5 Vista Géneros (heatmap ApexCharts, tarjetas, panel de títulos por género). Verificar: clic en drama lista títulos ordenados por IMDb.
- [ ] 7.6 Vista Países con el wrapper de interop jsVectorMap, tooltip, top 15 y lista por país. Verificar: clic en US muestra 2,325 títulos y el tooltip aparece.
- [ ] 7.7 Vista Ranking (pestañas IMDb/Popularidad, chips de tipo, selector de década). Verificar: Series + 2010 muestra hasta 50 elementos numerados.
- [ ] 7.8 Vista Personas (pestañas, top 100, ficha con scatter año×score y filmografía) y navegación desde el reparto del detalle. Verificar: clic en Sylvester Stallone desde Rocky abre su ficha.
- [ ] 7.9 Diseño adaptable (sidebar colapsable, apilado en una columna). Verificar a 375 px que no hay scroll horizontal.

## 8. Centro de Mensajería (XJ + JC)

- [ ] 8.1 (XJ) Componente SVG de topología con partículas (límite 30/s) y contadores por arista, incluida la `_error`. Verificar: durante una corrida las partículas fluyen y los mensajes corruptos van a la DLQ en rojo.
- [ ] 8.2 (XJ) Gráficas en tiempo real de publish/deliver y ready/unacked (ventana de 60 s) y tarjetas por cola. Verificar: al pausar el consumidor, `ready` sube en pantalla.
- [ ] 8.3 (XJ) Barra de progreso de la corrida, feed de eventos (200) y panel DLQ con motivos. Verificar: los estados almacenado, reintento n/3, error y duplicado aparecen con el estilo correcto.
- [ ] 8.4 (JC + XJ) Controles de demo (Ingestar, Pausar/Reanudar, Corrupto, Falla transitoria, Duplicado, Reintentar DLQ) con estados deshabilitados (corrida activa, ETL ausente, demo apagada). Verificar cada escenario de `specs/messaging-center-ui`.
- [ ] 8.5 (XJ) Indicador de desconexión, reconexión automática y resincronización del estado. Verificar: reiniciar `web` o cortar la red unos segundos hace que la vista vuelva a "en vivo" con los valores actuales.

## 9. Integración, demo y entrega (todo el equipo)

- [ ] 9.1 Prueba end-to-end desde cero: `docker compose down -v && docker compose up -d --build`, Ingestar desde la UI y comprobar 6,137 títulos en el Resumen y en `cqlsh` (`SELECT COUNT(*) FROM titles_by_id`).
- [ ] 9.2 Prueba de idempotencia de sistema: dos corridas seguidas mantienen 6,137 títulos y KPIs idénticos.
- [ ] 9.3 Prueba de resiliencia: reiniciar el contenedor `ingestion` a mitad de corrida y comprobar que la corrida completa sin pérdidas ni duplicados.
- [ ] 9.4 Ejecutar todas las suites (`pytest`, `dotnet test`). Verificar que todas pasan en verde.
- [ ] 9.5 Escribir el guion de demo (`docs/guion-demo.md`) con orden de vistas y escenarios de mensajería, y ensayarlo cronometrado (≤ 10 min).
- [ ] 9.6 Actualizar el README final con capturas de cada vista y el diagrama de arquitectura. Verificar que los mermaid renderizan en GitHub.

## Workflow follow-up

- Sincronizar las specs (`/opsx:sync`) y archivar el change (`/opsx:archive`) cuando la entrega esté aprobada.
