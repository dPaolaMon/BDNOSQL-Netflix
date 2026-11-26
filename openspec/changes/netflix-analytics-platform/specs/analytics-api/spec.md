# Spec Delta: analytics-api

## Purpose

Expone las consultas analíticas del Data Mart y las acciones de demostración de mensajería como una API HTTP de solo lectura (salvo los comandos de demo), separando consultas de comandos. También provee un canal en tiempo real que notifica al dashboard los eventos de ingesta y las métricas de colas.

## ADDED Requirements

### Requirement: Separación de consultas y comandos
El backend SHALL separar las operaciones de lectura (consultas al Data Mart) de las operaciones que provocan efectos (comandos de demo). Los comandos nunca escriben directo en la base de datos: siempre producen mensajes.

#### Scenario: Ejecución de un comando
- **WHEN** se invoca el comando de inicio de ingesta
- **THEN** se publica un mensaje de comando y la respuesta no espera a que la ingesta termine

### Requirement: Endpoints de consulta del dashboard
La API SHALL exponer bajo `/api` un endpoint por cada consulta del dashboard:
- KPIs;
- títulos por año;
- clasificación por edad;
- resumen de géneros;
- heatmap género × década;
- títulos de un género;
- estadísticas por país;
- títulos de un país;
- ranking;
- búsqueda en catálogo;
- detalle de título;
- top personas;
- filmografía de una persona;
- estado de ingesta;
- métricas de colas.

#### Scenario: KPIs
- **WHEN** se hace `GET /api/kpis` con el catálogo ingerido
- **THEN** la respuesta es 200 con, entre otros, `total_titles = 6137`, `movies = 3831` y `shows = 2306`

#### Scenario: Detalle de título
- **WHEN** se hace `GET /api/titles/tm82169`
- **THEN** la respuesta es 200 con los atributos del título y su lista de créditos

### Requirement: Recurso inexistente
La API SHALL responder 404 con un cuerpo de problema estándar cuando el título o la persona solicitados no existen.

#### Scenario: Título inexistente
- **WHEN** se hace `GET /api/titles/tm000000`
- **THEN** la respuesta es 404 con un cuerpo de problema que identifica el recurso

### Requirement: Validación de parámetros
La API SHALL responder 400 con detalle de validación ante parámetros inválidos: tipo distinto de MOVIE, SHOW o ALL; década no múltiplo de 10; tamaño de página fuera de 1–100.

#### Scenario: Tipo inválido en ranking
- **WHEN** se hace `GET /api/rankings/imdb?type=EPISODE`
- **THEN** la respuesta es 400 indicando que `type` no es válido

### Requirement: Búsqueda paginada del catálogo
La búsqueda del catálogo SHALL aceptar texto libre sin distinguir mayúsculas ni acentos, filtros por tipo, género, década y rango de puntuación, ordenamiento y paginación. Devuelve el total de coincidencias.

#### Scenario: Búsqueda por texto
- **WHEN** se busca `rocky` con tipo `MOVIE`
- **THEN** los resultados incluyen "Rocky" y la respuesta informa el total de coincidencias y la página actual

#### Scenario: Búsqueda sin coincidencias
- **WHEN** se busca un texto que no coincide con ningún título
- **THEN** la respuesta es 200 con una lista vacía y total 0

### Requirement: Consistencia del catálogo tras una ingesta
Los resultados de búsqueda del catálogo SHALL reflejar los datos de la última corrida completada a más tardar cuando se emite su evento de finalización.

#### Scenario: Catálogo tras reingesta
- **WHEN** termina una corrida de ingesta
- **THEN** la siguiente búsqueda refleja los datos de esa corrida

### Requirement: Endpoints de comandos de demostración
La API SHALL exponer bajo `/api/demo` los comandos: iniciar ingesta, pausar y reanudar consumidor, inyectar mensaje corrupto, simular falla transitoria, reenviar duplicado y reprocesar cola de errores. Todos responden 202 al aceptarse.

#### Scenario: Comando aceptado
- **WHEN** se hace `POST /api/demo/inject-corrupt` con la demo habilitada
- **THEN** la respuesta es 202 y el mensaje corrupto se publica en el broker

#### Scenario: Demo deshabilitada
- **WHEN** se invoca un endpoint de `/api/demo` con la configuración de demostración deshabilitada
- **THEN** la respuesta es 404

### Requirement: Inicio de ingesta con corrida activa
La API SHALL rechazar con 409 una solicitud de inicio de ingesta mientras haya una corrida en curso.

#### Scenario: Corrida ya activa
- **WHEN** se solicita iniciar una ingesta con otra corrida en estado en curso
- **THEN** la respuesta es 409 indicando el id de la corrida activa

### Requirement: Documentación de la API
La API SHALL publicar su descripción OpenAPI y una interfaz interactiva para explorarla en entorno de desarrollo.

#### Scenario: Exploración de la API
- **WHEN** se abre la interfaz de documentación en desarrollo
- **THEN** se listan todos los endpoints de consulta y de demostración con sus parámetros

### Requirement: Canal de eventos en tiempo real
El backend SHALL ofrecer un canal en tiempo real que retransmite a los clientes conectados:
- los eventos de ingesta: título almacenado, duplicado, reintento, falla, progreso y cambio de estado;
- las métricas de colas, al menos cada 2 segundos.

#### Scenario: Cliente suscrito
- **WHEN** un cliente está conectado al canal y se almacena un título
- **THEN** el cliente recibe el evento correspondiente en menos de 1 segundo

### Requirement: Obtención de métricas del broker
El backend SHALL obtener del broker, para cada cola, los mensajes listos, los mensajes sin confirmar, los consumidores y las tasas de publicación y entrega. Si el broker no responde, marca las métricas como no disponibles sin interrumpir el resto del servicio.

#### Scenario: Broker inaccesible
- **WHEN** la API de administración del broker no responde
- **THEN** las métricas de colas se reportan como no disponibles y las consultas analíticas siguen respondiendo

### Requirement: Tolerancia al arranque de dependencias
El backend SHALL arrancar aunque Cassandra o el broker aún no estén disponibles, reintentar la conexión con espera creciente y reportar el estado de cada dependencia en un endpoint de salud.

#### Scenario: Cassandra todavía iniciando
- **WHEN** el backend arranca antes de que Cassandra acepte conexiones
- **THEN** el endpoint de salud reporta Cassandra como no disponible y, cuando Cassandra está lista, pasa a disponible sin reiniciar el backend
