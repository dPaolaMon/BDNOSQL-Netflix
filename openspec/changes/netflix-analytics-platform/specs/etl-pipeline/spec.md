# Spec Delta: etl-pipeline

## Purpose

Convierte los CSV crudos del catálogo de Netflix (títulos y créditos) en registros limpios, validados y enriquecidos, y en agregados analíticos. Los publica como mensajes confirmados hacia la capa de ingesta.

## ADDED Requirements

### Requirement: Extracción de los datasets fuente
El pipeline SHALL leer `titles.csv` y `credits.csv` desde un directorio configurable, respetando campos multilínea y comillas escapadas, sin perder registros.

#### Scenario: Lectura completa del dataset original
- **WHEN** el pipeline se ejecuta sobre los archivos de `archive/`
- **THEN** obtiene 6,137 registros de títulos y 81,355 de créditos antes de limpiar

#### Scenario: Archivo fuente ausente
- **WHEN** alguno de los dos archivos no existe en el directorio configurado
- **THEN** el pipeline termina con código de salida distinto de cero y un mensaje que indica qué archivo falta, sin publicar ningún mensaje

### Requirement: Deserialización de atributos de colección
El pipeline SHALL convertir `genres` y `production_countries`, que vienen como texto con forma de lista, en conjuntos sin duplicados. Los géneros se normalizan a minúsculas y los países a códigos ISO-2 en mayúsculas.

#### Scenario: Lista con varios elementos
- **WHEN** un título trae `genres = "['drama', 'sport']"`
- **THEN** el registro limpio contiene el conjunto `{drama, sport}`

#### Scenario: Lista vacía
- **WHEN** un título trae `production_countries = "[]"`
- **THEN** el registro limpio contiene un conjunto vacío y el título se contabiliza como "sin país" en el reporte de calidad

### Requirement: Normalización de valores nulos y fuera de rango
El pipeline SHALL aplicar reglas deterministas a los valores faltantes o inválidos:
- puntuaciones vacías o fuera de 0–10 → nulo;
- `runtime = 0` → nulo;
- `age_certification` vacía → `NR`;
- `seasons` → nulo en películas y entero en series.

#### Scenario: Película sin temporadas
- **WHEN** se procesa un título de tipo `MOVIE`
- **THEN** su campo `seasons` es nulo

#### Scenario: Clasificación faltante
- **WHEN** un título no tiene `age_certification`
- **THEN** el registro limpio tiene `age_certification = "NR"`

#### Scenario: Puntuación fuera de rango
- **WHEN** un título trae `imdb_score = 11.5`
- **THEN** el registro limpio tiene `imdb_score` nulo y el reporte de calidad incrementa el contador de esa regla

### Requirement: Validación y rechazo de registros
El pipeline SHALL validar cada título:
- id con formato `tm<n>` o `ts<n>`;
- tipo `MOVIE` o `SHOW`;
- año entre 1900 y 2030.

Los registros inválidos se excluyen de la publicación y se escriben en un archivo de rechazados que indica el motivo.

#### Scenario: Registro con tipo inválido
- **WHEN** un título tiene `type = "EPISODE"`
- **THEN** no se publica y aparece en `rejected.csv` con el motivo "tipo inválido"

#### Scenario: Dataset original válido
- **WHEN** el pipeline procesa el dataset original de `archive/`
- **THEN** ningún título es rechazado

### Requirement: Limpieza de créditos
El pipeline SHALL descartar créditos duplicados exactos y créditos cuyo título no exista. Un `character` vacío se convierte en nulo.

#### Scenario: Crédito huérfano
- **WHEN** un crédito referencia un id de título inexistente
- **THEN** el crédito se descarta y se contabiliza en el reporte de calidad

### Requirement: Enriquecimiento del título con su reparto
El pipeline SHALL agregar a cada título la lista de sus créditos (actores y directores) y su década (`año // 10 * 10`), de modo que cada título se publique como un documento autocontenido.

#### Scenario: Título con reparto
- **WHEN** se procesa el título `tm82169`
- **THEN** su documento incluye `decade = 1970` y la lista de créditos con Sylvester Stallone como `ACTOR`

#### Scenario: Título sin créditos
- **WHEN** un título no tiene créditos en `credits.csv`
- **THEN** su documento se publica con una lista de créditos vacía

### Requirement: Cálculo de agregados analíticos
El pipeline SHALL calcular sobre los datos limpios los agregados del Data Mart:
- KPIs globales;
- títulos por año y tipo;
- resumen por género;
- género × década;
- por país;
- por clasificación;
- top 100 personas por rol;
- rankings top 50 por IMDb y por popularidad, por tipo y década.

#### Scenario: KPIs del dataset original
- **WHEN** se calculan los agregados del dataset original
- **THEN** los KPIs reportan 6,137 títulos, 3,831 películas, 2,306 series y 56,754 personas distintas

#### Scenario: Elegibilidad para ranking IMDb
- **WHEN** un título tiene menos votos IMDb que el umbral configurado (1,000 por defecto)
- **THEN** no aparece en el ranking por IMDb aunque su puntuación sea alta

### Requirement: Publicación confirmada de mensajes
El pipeline SHALL publicar cada título, cada agregado y los eventos de inicio y fin de corrida como mensajes persistentes. Usa un identificador de corrida y un identificador de mensaje determinista por corrida y título. Cada mensaje queda confirmado por el broker.

#### Scenario: Corrida exitosa
- **WHEN** se ejecuta una corrida completa con el broker disponible
- **THEN** se publican un evento de inicio con el total esperado, un mensaje por título válido, un mensaje por tipo de agregado y un evento de fin, y todos quedan confirmados

#### Scenario: Mensaje no enrutable o rechazado
- **WHEN** el broker no confirma un mensaje después de 3 intentos
- **THEN** el mensaje se registra en `rejected_publish.jsonl` y la corrida termina con código de salida distinto de cero

#### Scenario: Identificador estable
- **WHEN** el mismo título se publica dos veces dentro de la misma corrida
- **THEN** ambos mensajes llevan el mismo identificador de mensaje

### Requirement: Control de velocidad de publicación
El pipeline SHALL aceptar una tasa máxima de mensajes por segundo para que la ingesta sea observable en la demo. Sin tasa configurada, publica a máxima velocidad.

#### Scenario: Tasa limitada
- **WHEN** se ejecuta con una tasa de 60 mensajes por segundo
- **THEN** el ritmo medio de publicación de títulos no supera 60 mensajes por segundo

### Requirement: Ejecución bajo demanda por comando
El pipeline SHALL ofrecer un modo de escucha en el que espera comandos de inicio de ingesta desde la mensajería y ejecuta una corrida por cada comando. Mientras una corrida está en curso, rechaza un segundo comando.

#### Scenario: Comando de inicio recibido
- **WHEN** el pipeline está en modo escucha y recibe un comando de inicio
- **THEN** ejecuta una corrida completa con los parámetros del comando

#### Scenario: Comando durante una corrida activa
- **WHEN** llega un comando de inicio mientras otra corrida está en curso
- **THEN** el comando se descarta y se registra que ya hay una corrida activa

### Requirement: Reporte de calidad de datos
El pipeline SHALL generar en cada corrida un reporte con los conteos de registros leídos, limpios, rechazados y afectados por cada regla de limpieza. El resumen se incluye en el evento de fin de corrida.

#### Scenario: Reporte generado
- **WHEN** termina una corrida
- **THEN** existe `quality_report.json` con los conteos por regla y el evento de fin incluye el número de filas rechazadas
