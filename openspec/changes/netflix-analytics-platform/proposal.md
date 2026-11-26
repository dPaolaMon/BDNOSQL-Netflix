# Propuesta: Plataforma Netflix Analytics

## Why

El catálogo público de Netflix (Kaggle) llega como dos CSV planos: `titles.csv` con 6,137 títulos y `credits.csv` con 81,355 créditos. Contestar preguntas analíticas sobre ellos exige cruzarlos y agregarlos. En un esquema relacional eso se resuelve con JOINs y GROUP BY costosos, y la carga de datos queda acoplada a las lecturas. El curso de Bases de Datos NoSQL pide demostrar un modelo **orientado a consultas** en Apache Cassandra, alimentado por un ETL en Python, con un backend .NET y una ingesta **transaccional** por mensajería. La propuesta original (README) era vaga: mencionaba un "Data Mart" en sentido relacional y Power BI como capa de presentación. Hay que redefinir el alcance antes de construir.

## What Changes

- **Pipeline ETL en Python:** extrae ambos CSV y los limpia:
  - colecciones serializadas como texto (`"['drama', 'sport']"`);
  - nulos en `imdb_score` (468), `age_certification` (2,743) y `seasons` (solo aplica a series);
  - `runtime = 0`;
  - textos vacíos.

  Después embebe el reparto en cada título, precalcula agregados y publica el resultado como mensajes.
- **Ingesta transaccional con RabbitMQ + MassTransit:**
  - publisher confirms, colas durables, reintentos con backoff y cola de errores (DLQ);
  - consumidores .NET **idempotentes**: reprocesar un mensaje no duplica datos.
- **Data Mart NoSQL orientado a consultas en Cassandra.** Reemplaza al "Data Mart" relacional del README:
  - una tabla desnormalizada por cada consulta del dashboard;
  - tablas de agregados precalculados (KPIs, por año, por género×década, por país, top personas);
  - sin JOINs en tiempo de lectura.
- **Backend .NET 10:** consultas CQRS con clases propias (sin MediatR), Minimal API REST y hub SignalR para eventos en tiempo real.
- **Dashboard Blazor (Interactive Server + MudBlazor + ApexCharts):** tema oscuro estilo Netflix con siete vistas: Resumen, Géneros, Países, Ranking, Catálogo, Personas y **Centro de Mensajería**. Esta última hace visible el flujo de mensajes (topología animada, tasas de colas, feed, DLQ y controles de demostración).
- **Despliegue local con Docker Compose:** Cassandra, RabbitMQ con Management, inicialización del esquema, ETL, Worker de ingesta y aplicación web.
- **Se eliminan del alcance:** Power BI y el modelado dimensional en estrella.

## Capabilities

### New Capabilities

- `etl-pipeline`: extracción, limpieza, validación, transformación y cálculo de agregados del catálogo en Python, y publicación de los resultados como mensajes.
- `messaging-ingestion`: topología RabbitMQ, contratos de mensajes y consumidores .NET idempotentes que persisten en Cassandra, con reintentos, DLQ y eventos de progreso.
- `cassandra-data-mart`: keyspace, tipos (UDT) y tablas orientadas a consultas, más tablas de agregados y de control de ingesta en Cassandra.
- `analytics-api`: consultas CQRS sobre el Data Mart expuestas por Minimal API, y hub SignalR para notificaciones en vivo.
- `dashboard-ui`: vistas analíticas en Blazor (Resumen, Géneros, Países, Ranking, Catálogo, Personas), con el tema visual y la navegación.
- `messaging-center-ui`: vista que hace observable la mensajería en tiempo real y permite ejecutar escenarios de demostración (pausa del consumidor, mensaje corrupto, duplicado, reintento de la DLQ).

### Modified Capabilities

_Ninguna._ No existen especificaciones previas en `openspec/specs/`.

## Impact

- **Repositorio:** se crean `docker-compose.yml`, `cassandra/schema.cql`, `etl/` (Python), `src/Netflix.Contracts`, `src/Netflix.Core`, `src/Netflix.Ingestion`, `src/Netflix.Web` y `tests/`. El README se corrige para reflejar este alcance.
- **Dependencias:**
  - Python: `pandas`, `pika`, `pytest`.
  - .NET: `MassTransit.RabbitMQ` 8.x, `CassandraCSharpDriver`, `MudBlazor`, `Blazor-ApexCharts`, SignalR (incluido en ASP.NET Core) y `xUnit`.
- **Infraestructura:** imágenes Docker de `cassandra:5` y `rabbitmq:management`. Puertos 9042 (CQL), 5672/15672 (AMQP y Management) y 8080 (web).
- **Fuera de alcance:** autenticación de usuarios, despliegue en la nube, clúster multinodo de Cassandra, edición de datos desde la UI (es solo lectura, salvo los controles de demo de mensajería) y Power BI.
