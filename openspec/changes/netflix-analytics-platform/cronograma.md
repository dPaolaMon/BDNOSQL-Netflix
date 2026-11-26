# Cronograma: Plataforma Netflix Analytics

- **Inicio:** martes 3 de noviembre de 2026
- **Entrega:** viernes 27 de noviembre de 2026
- **Jornadas:** días hábiles (lunes a viernes), 4 semanas

Las tareas detalladas de cada barra están en `tasks.md`; los números de grupo se indican entre corchetes.

## Diagrama de Gantt

```mermaid
gantt
    title Netflix Analytics · 3 al 27 de noviembre de 2026
    dateFormat YYYY-MM-DD
    axisFormat %d %b
    excludes weekends
    todayMarker off

    section Equipo completo
    Kickoff y definición de arquitectura        :done, t1, 2026-11-03, 2d
    Integración y pruebas end-to-end [9]        :t2, 2026-11-23, 2d
    Guion de demo, README final y ensayo [9]    :t3, 2026-11-25, 2d
    Entrega                                     :milestone, crit, m1, 2026-11-27, 0d

    section Diana Paola Montes · Datos y ETL
    Análisis y perfilado de datasets            :d1, 2026-11-05, 2d
    Docker Compose y schema CQL [1]             :d2, 2026-11-09, 2d
    ETL extracción y limpieza [2]               :d3, 2026-11-11, 3d
    Agregados y reporte de calidad [2]          :d4, 2026-11-16, 2d
    Publisher con confirms y modo listen [3]    :crit, d5, 2026-11-18, 3d

    section Jonathan Cervantes · Mensajería
    Topología RabbitMQ y contratos [3]          :j1, 2026-11-05, 2d
    Consumidores MassTransit [5]                :j2, 2026-11-09, 5d
    Idempotencia, reintentos y DLQ [5]          :crit, j3, 2026-11-16, 3d
    Pausa, faults y escenarios de demo [5]      :j4, 2026-11-19, 2d

    section Fernanda Baez · Backend .NET
    Solución .NET y CQRS propio [4]             :f1, 2026-11-05, 2d
    Repositorios Cassandra [4]                  :f2, 2026-11-09, 4d
    Queries y Minimal API [6]                   :f3, 2026-11-13, 4d
    SignalR, poller de colas y health [6]       :f4, 2026-11-19, 2d

    section Ximena Jasso · Frontend Blazor
    Mockups y tema visual [7]                   :x1, 2026-11-05, 2d
    Layout MudBlazor y Resumen [7]              :x2, 2026-11-09, 4d
    Catálogo, Géneros y Países [7]              :x3, 2026-11-13, 4d
    Ranking, Personas y responsive [7]          :x4, 2026-11-19, 2d
    Centro de Mensajería [8]                    :crit, x5, 2026-11-23, 2d
```

## Hitos

| Fecha | Hito | Criterio de cumplimiento |
|---|---|---|
| 04-nov | Arquitectura definida | Change OpenSpec con propuesta, diseño, specs y tareas |
| 10-nov | Infraestructura lista | `docker compose up` con Cassandra y RabbitMQ healthy y esquema aplicado |
| 13-nov | Primer flujo de punta a punta | Un título viaja ETL → RabbitMQ → consumidor → Cassandra |
| 18-nov | Ingesta transaccional completa | Idempotencia, reintentos y DLQ probados |
| 20-nov | Datos y API completos | Catálogo completo ingerido y endpoints `/api` respondiendo |
| 24-nov | Dashboard completo | 7 vistas funcionando, incluido el Centro de Mensajería |
| 26-nov | Demo ensayada | Guion ≤ 10 min ejecutado sin fallas |
| 27-nov | **Entrega** | Repositorio, documentación y demo en vivo |

## Ruta crítica

**Publisher con confirms → Idempotencia/DLQ → Centro de Mensajería → Integración.** El Centro de Mensajería depende de que existan los eventos de ingesta y los escenarios de falla, y es la pieza principal de la demo. Por eso las barras marcadas como `crit` no tienen holgura.
