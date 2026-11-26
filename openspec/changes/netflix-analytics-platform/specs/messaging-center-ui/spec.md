# Spec Delta: messaging-center-ui

## Purpose

Hace visible en tiempo real el uso de la mensajería transaccional dentro del dashboard. Muestra el flujo de mensajes, el estado de las colas y los eventos de procesamiento, y permite ejecutar escenarios de demostración que evidencian durabilidad, reintentos, cola de errores e idempotencia.

## ADDED Requirements

### Requirement: Vista Centro de Mensajería
El dashboard SHALL ofrecer en el menú de navegación una vista "Centro de Mensajería" que reúne topología, métricas de colas, progreso, feed de eventos, panel de errores y controles de demostración.

```
┌ Centro de Mensajería ───────────────────────────────── ● En vivo ┐
│  [Python ETL] ══●══●══► (exchange) ══●══► [cola ingest] ══●══►    │
│                    netflix.catalog      ready 1,204 unack 8       │
│                                    └─► [cola _error] 3 ⚠          │
│         ══●══► [Consumer .NET] ══●══► (Cassandra)                 │
├──────────────────────┬────────────────────────────────────────────┤
│ Publish/s ▁▃▆█▇▅  412 │ Feed: 12:01:03 tm82169 Rocky  ✔ 4ms        │
│ Deliver/s ▁▂▅▇█▆  398 │       12:01:03 ts3003… ↻ reintento 2/3     │
│ Progreso ████████░ 82%│       12:01:04 tm1234  ⊘ duplicado (skip)  │
├──────────────────────┴────────────────────────────────────────────┤
│ [▶ Ingestar] [⏸ Pausar consumidor] [☠ Inyectar corrupto]           │
│ [⚡ Falla transitoria] [⧉ Reenviar duplicado] [↺ Reintentar DLQ]    │
└───────────────────────────────────────────────────────────────────┘
```

#### Scenario: Acceso a la vista
- **WHEN** el usuario selecciona "Mensajería" en el menú
- **THEN** se muestra la vista con todas sus secciones y un indicador de conexión en vivo

### Requirement: Topología animada del flujo
La vista SHALL dibujar la topología ETL → exchange → colas → consumidor → Cassandra, con la cola de errores visible. Representa cada lote de eventos con partículas animadas sobre la arista correspondiente, sin superar 30 partículas por segundo.

#### Scenario: Mensajes fluyendo
- **WHEN** se están almacenando títulos
- **THEN** se ven partículas recorriendo las aristas desde el ETL hasta Cassandra

#### Scenario: Mensaje enviado a errores
- **WHEN** un mensaje se mueve a la cola de errores
- **THEN** una partícula de color de error viaja hacia la cola de errores y su contador aumenta

### Requirement: Métricas de colas en tiempo real
La vista SHALL mostrar, por cada cola, los mensajes listos, los mensajes sin confirmar y los consumidores conectados, más gráficas de las tasas de publicación y entrega de los últimos 60 segundos. Los datos se actualizan al menos cada 2 segundos.

#### Scenario: Cola acumulando mensajes
- **WHEN** el consumidor está pausado y el ETL publica
- **THEN** el contador de mensajes listos de la cola de títulos crece en pantalla sin recargar la página

### Requirement: Progreso de la corrida
La vista SHALL mostrar el estado de la corrida actual y una barra de progreso de títulos almacenados sobre el total esperado, con contadores de duplicados y fallas.

#### Scenario: Corrida en curso
- **WHEN** hay 4,000 títulos almacenados de 6,137 esperados
- **THEN** la barra muestra alrededor de 65 % y el estado "en curso"

### Requirement: Feed de eventos
La vista SHALL listar los eventos de procesamiento más recientes (hasta 200), cada uno con hora, id y nombre del título, estado (almacenado, reintento n/3, error, duplicado omitido) y latencia cuando aplique.

#### Scenario: Evento de duplicado
- **WHEN** la ingesta omite un mensaje duplicado
- **THEN** aparece en el feed una entrada con estado "duplicado omitido" para ese título

### Requirement: Panel de cola de errores
La vista SHALL mostrar el número de mensajes en la cola de errores y, para los más recientes, el id del título (si existe) y el motivo de la falla.

#### Scenario: Falla de validación
- **WHEN** un mensaje corrupto llega a la cola de errores
- **THEN** el panel muestra el contador incrementado y el motivo de la falla

### Requirement: Control de inicio de ingesta
La vista SHALL tener un control para iniciar una corrida de ingesta. El control está deshabilitado mientras hay una corrida en curso o cuando el ETL no está escuchando comandos.

#### Scenario: Inicio de ingesta
- **WHEN** el usuario pulsa "Ingestar" con el ETL disponible
- **THEN** comienza una corrida y su progreso se actualiza en la vista

#### Scenario: ETL no disponible
- **WHEN** no hay ningún proceso del ETL escuchando comandos
- **THEN** el control "Ingestar" está deshabilitado y muestra el motivo

### Requirement: Control de pausa y reanudación
La vista SHALL ofrecer un control alternante para pausar y reanudar el consumidor de títulos, y reflejar el estado actual del consumidor.

#### Scenario: Pausar consumidor
- **WHEN** el usuario pulsa "Pausar consumidor"
- **THEN** el consumidor deja de procesar, el control cambia a "Reanudar" y la cola de títulos empieza a acumular mensajes

### Requirement: Escenarios de demostración de fallas
La vista SHALL ofrecer tres controles de demostración:
- inyectar un mensaje corrupto, que termina en la cola de errores;
- simular una falla transitoria, que se recupera tras reintentos;
- reenviar el último mensaje procesado como duplicado, que se omite.

#### Scenario: Inyectar corrupto
- **WHEN** el usuario pulsa "Inyectar corrupto"
- **THEN** en pocos segundos el feed muestra un error y el contador de la cola de errores aumenta en uno

#### Scenario: Falla transitoria
- **WHEN** el usuario pulsa "Falla transitoria"
- **THEN** el feed muestra "reintento 1/3" y "reintento 2/3" seguidos de "almacenado" para el mismo título

#### Scenario: Reenviar duplicado
- **WHEN** el usuario pulsa "Reenviar duplicado"
- **THEN** el feed muestra "duplicado omitido" y el total de títulos del Data Mart no cambia

### Requirement: Control de reprocesamiento de errores
La vista SHALL ofrecer un control para reprocesar la cola de errores y mostrar el resultado: cuántos mensajes se recuperaron y cuántos volvieron a la cola de errores.

#### Scenario: Reprocesar cola de errores
- **WHEN** el usuario pulsa "Reintentar DLQ" con mensajes en la cola de errores
- **THEN** se muestra cuántos mensajes se recuperaron y cuántos regresaron a la cola de errores

### Requirement: Controles de demo protegidos por configuración
Los controles que alteran el flujo (pausa, inyección, falla simulada, duplicado y reprocesamiento) SHALL estar disponibles solo cuando la configuración de demostración está habilitada.

#### Scenario: Demo deshabilitada
- **WHEN** la configuración de demostración está deshabilitada
- **THEN** la vista muestra topología, métricas y feed, pero no los controles de demostración

### Requirement: Resiliencia de la conexión en vivo
La vista SHALL indicar cuando se pierde la conexión en tiempo real y reconectarse automáticamente, recuperando el estado actual de colas y corrida al reconectar.

#### Scenario: Reconexión
- **WHEN** la conexión en tiempo real se interrumpe y luego se restablece
- **THEN** el indicador pasa de "desconectado" a "en vivo" y los contadores muestran los valores actuales
