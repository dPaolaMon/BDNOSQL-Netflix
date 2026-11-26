# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Estado del proyecto

Proyecto escolar de Bases de Datos NoSQL: **Netflix Analytics**, un dashboard sobre el catálogo de Netflix de Kaggle (`archive/titles.csv`, `archive/credits.csv`).

**Por ahora solo existe la documentación; todavía no hay código.** La fuente de verdad es el change de OpenSpec `openspec/changes/netflix-analytics-platform/`:
- `proposal.md`: el porqué y el alcance;
- `design.md`: el cómo. Tiene las decisiones D1–D13, el esquema de tablas, los contratos JSON y los mockups;
- `specs/*/spec.md`: los requisitos;
- `tasks.md`: las tareas, en 6 grupos;
- `cronograma.md`: el Gantt.

Para implementar se usa `/opsx:apply`, que avanza `tasks.md` marcando las casillas. Si cambian el diseño o el alcance, actualiza el change con `/opsx:update` antes de tocar código, y valida con `openspec validate netflix-analytics-platform --strict`.

## Reglas de trabajo

- **KISS ante todo.** Usa lo que el framework ya da: la DLQ y los reintentos de MassTransit, el upsert de Cassandra, el tiempo real de Blazor Server. No construyas mecanismos propios. Ya se descartaron, a propósito, la API REST, MediatR o dispatchers CQRS, la tabla de mensajes procesados con LWT, el hub SignalR propio, el mapa geográfico y Power BI.
- Todos los documentos y artefactos de OpenSpec van **en español**.
- **Sesiones separadas:** en una sesión de documentación solo se editan docs.
- **PRs:**
  - Documentación: todo en un solo PR.
  - Código: un PR por grupo completo de `tasks.md`, con una rama por grupo.
  - Nunca hagas merge.
  - `gh` no está instalado: haz push de la rama y pasa el enlace `https://github.com/dPaolaMon/BDNOSQL-Netflix/compare/main...<rama>`.
- Las fechas (3–27 nov 2026) y los 4 integrantes del README se mantienen tal cual en los docs.

## Arquitectura (big picture)

```
CSV → ETL Python (pandas, pika) ──publisher confirms──► RabbitMQ: exchange topic `netflix.catalog`
        ├─ routing key `title`     → cola `catalog-titles`     (+ `catalog-titles_error`)
        └─ routing key `aggregate` → cola `catalog-aggregates`
                 ▼
   src/Netflix.App (una sola app .NET 10: Blazor Web App Interactive Server)
        Messaging/  consumidores MassTransit 8.x → escriben en Cassandra (único camino de escritura)
        Data/       DashboardQueries → solo leen (una partición por consulta)
        Live/       LiveEvents (singleton con eventos C#) → los componentes Blazor se redibujan
                    QueueStatsPoller → Management API de RabbitMQ cada 2 s
                 ▼
   Cassandra 5, keyspace `netflix_mart` (9 tablas)
```

Invariantes que no son obvias leyendo un solo archivo:

- **Camino único de escritura.** Solo los consumidores escriben en Cassandra. El ETL publica y la UI lee. La separación lectura/escritura se hace con clases distintas (consumidores frente a `DashboardQueries`), sin interfaces genéricas.
- **El contrato Python → .NET es JSON plano en snake_case, sin el sobre de MassTransit.** Por eso los endpoints usan `UseRawJsonDeserializer()`, `ConfigureConsumeTopology = false` y un binding explícito por routing key. Cada cola transporta un solo tipo de mensaje. Los contratos de ejemplo están en `design.md` D3.
- **MassTransit se queda en 8.x**, porque la v9 tiene licencia comercial.
- **La idempotencia se basa en el upsert por clave natural.** El consumidor solo hace un `SELECT id` previo para etiquetar el evento como "nuevo" o "ya existía". Un mensaje sin `id` agota los 3 reintentos (`Interval(3, 1s)`) y acaba en `_error`; `FaultConsumer` lo notifica a la UI.
- **Data Mart NoSQL orientado a consultas, no esquema estrella.** Cada pantalla lee una partición. Todos los agregados (KPIs, por año, por género, por país, rankings top 50 y top 50 personas) los calcula el ETL con pandas y viajan como mensajes `aggregate`. El agregado `kpis` se publica **al final** de cada corrida, y su llegada refresca la caché en memoria del Catálogo (que carga todo `titles_by_id`, ~6k filas).
- `titles_by_id` embebe el reparto como `list<frozen<credit>>` para que el detalle se lea con una sola lectura. En `titles_by_genre`, los títulos sin `imdb_score` se guardan con `-1` para que queden al final.
- **Las 5 vistas de la UI:** Resumen, Catálogo, Ranking, Personas y Mensajería. Mensajería es la pieza central de la demo. Tiene dos botones (`DemoPublisher`): "Inyectar corrupto" (termina en la DLQ) y "Reenviar duplicado" (aparece como "ya existía"). La durabilidad se demuestra con `docker compose stop app` y después `start`.

## Comandos (previstos en el diseño; verifica que existan antes de usarlos)

```bash
docker compose up -d --build          # cassandra, cassandra-init (aplica cassandra/schema.cql), rabbitmq, app
docker compose run --rm etl           # carga el catálogo (servicio con perfil `tools`)
docker compose run --rm etl --rate 60 # carga lenta para la demo
docker compose down -v                # reinicio total, borra los volúmenes

pytest etl/tests                      # tests del ETL
pytest etl/tests/test_transform.py::test_nombre   # un solo test
dotnet test                           # tests .NET (tests/Netflix.App.Tests)
dotnet test --filter "FullyQualifiedName~TitleConsumer"   # filtrar tests

openspec validate netflix-analytics-platform --strict
```

Puertos: app en 8080, RabbitMQ Management en 15672, AMQP en 5672 y CQL en 9042.

## Datos de referencia (útiles para tests)

- 6,137 títulos: 3,831 `MOVIE` y 2,306 `SHOW`.
- 81,355 créditos, de 56,754 personas distintas. No hay créditos huérfanos.
- 19 géneros y 107 países.
- `genres` y `production_countries` vienen como texto con forma de lista (`"['drama', 'sport']"`), así que se parsean con `ast.literal_eval`.
- `seasons` es nulo en todas las películas.
- Caso canónico: `tm82169` es *Rocky* (1976), IMDb 8.1, con Sylvester Stallone (`person_id` 3180) como `ACTOR`.
