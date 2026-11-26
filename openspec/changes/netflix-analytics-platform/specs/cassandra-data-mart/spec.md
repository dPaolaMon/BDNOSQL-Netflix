# Spec Delta: cassandra-data-mart

## Purpose

Define el almacenamiento NoSQL en Apache Cassandra como un Data Mart orientado a consultas. Cada vista del dashboard se resuelve con lecturas por clave de partición, sobre tablas desnormalizadas y agregados precalculados, sin JOINs ni agregaciones en tiempo de lectura.

## ADDED Requirements

### Requirement: Esquema reproducible e idempotente
El sistema SHALL crear el keyspace `netflix_mart`, sus tipos y todas sus tablas con un script CQL que se puede ejecutar varias veces sin error y sin alterar los datos existentes.

#### Scenario: Primera inicialización
- **WHEN** el script se ejecuta contra un Cassandra vacío
- **THEN** existen el keyspace, el tipo `credit` y todas las tablas de consulta, agregados y control

#### Scenario: Reejecución del script
- **WHEN** el script se ejecuta sobre un keyspace que ya tiene datos
- **THEN** termina sin error y los datos permanecen intactos

### Requirement: Una tabla por patrón de consulta
El Data Mart SHALL tener una tabla dedicada para cada consulta de detalle o listado: título por id, títulos por género, títulos por país, títulos por tipo y década, y filmografía por persona. Cada una debe poder consultarse indicando solo su clave de partición.

#### Scenario: Mejores títulos de un género
- **WHEN** se consultan los títulos del género `drama`
- **THEN** se obtienen de una sola partición, ordenados por puntuación IMDb descendente, sin filtrado adicional

#### Scenario: Filmografía de una persona
- **WHEN** se consulta la persona con `person_id = 3180`
- **THEN** se obtienen sus créditos de una sola partición, ordenados por año de estreno descendente

### Requirement: Detalle de título con reparto embebido
El registro de detalle de un título SHALL contener todos sus atributos y la lista completa de créditos, de modo que el detalle se obtenga con una sola lectura.

#### Scenario: Consulta de detalle
- **WHEN** se lee el título `tm82169`
- **THEN** una única lectura devuelve sus métricas, géneros, países y créditos

### Requirement: Títulos sin puntuación en tablas ordenadas
Las tablas ordenadas por puntuación SHALL admitir títulos sin puntuación IMDb y ubicarlos después de todos los títulos con puntuación.

#### Scenario: Título sin IMDb en su género
- **WHEN** un título de `comedy` no tiene `imdb_score`
- **THEN** aparece al final del listado de `comedy`, después de todos los títulos con puntuación

### Requirement: Agregados precalculados
El Data Mart SHALL guardar como tablas propias los agregados calculados por el ETL:
- KPIs;
- títulos por año y tipo;
- resumen por género;
- género × década;
- por país;
- por clasificación;
- top personas por rol;
- rankings.

Cada gráfica del dashboard debe resolverse leyendo una sola partición.

#### Scenario: Heatmap género × década
- **WHEN** se consulta el agregado de un género
- **THEN** se obtiene, desde una sola partición, una fila por década con totales de películas y series y puntuación media

#### Scenario: Ranking filtrado
- **WHEN** se consulta el ranking IMDb de series de la década de 2010
- **THEN** se obtienen hasta 50 títulos ya ordenados por posición desde una sola partición

### Requirement: Escritura atómica de las vistas de un título
Las filas de un título en las tablas de detalle, género, país y tipo-década SHALL escribirse de forma atómica: el título aparece en todas sus vistas o en ninguna.

#### Scenario: Falla durante la escritura
- **WHEN** la escritura de un título falla antes de completarse
- **THEN** el título no aparece en ninguna de las tablas de consulta hasta que un reintento lo escriba completo

### Requirement: Escrituras idempotentes
Toda escritura de datos analíticos SHALL ser un upsert por clave natural, de modo que repetir la misma escritura deje el mismo estado.

#### Scenario: Doble escritura del mismo título
- **WHEN** el mismo título se escribe dos veces con los mismos datos
- **THEN** cada tabla contiene una sola fila por clave para ese título

### Requirement: Registro de mensajes procesados
El Data Mart SHALL registrar el identificador de cada mensaje procesado mediante una inserción condicional, para detectar duplicados de forma confiable. Los registros caducan a los 30 días.

#### Scenario: Primer procesamiento
- **WHEN** se registra un identificador de mensaje nuevo
- **THEN** la inserción condicional se aplica

#### Scenario: Mensaje repetido
- **WHEN** se intenta registrar un identificador que ya existe
- **THEN** la inserción condicional no se aplica y el sistema lo trata como duplicado

### Requirement: Control de corridas de ingesta
El Data Mart SHALL guardar, por corrida de ingesta:
- estado;
- total esperado de títulos;
- marcas de inicio y fin;
- filas rechazadas;
- contadores de títulos almacenados, duplicados y fallidos.

#### Scenario: Progreso de una corrida
- **WHEN** se han almacenado 3,000 de 6,137 títulos de una corrida
- **THEN** el registro de la corrida permite calcular un progreso cercano al 49 %

### Requirement: Reinicio limpio opcional
El sistema SHALL permitir vaciar las tablas de consulta y de agregados antes de una corrida, conservando el esquema, para eliminar filas obsoletas.

#### Scenario: Corrida con reinicio
- **WHEN** se lanza una corrida con la opción de reinicio
- **THEN** al terminar, las tablas solo contienen los datos de esa corrida
