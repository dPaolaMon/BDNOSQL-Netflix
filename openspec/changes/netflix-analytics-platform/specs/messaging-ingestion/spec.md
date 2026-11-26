# Spec Delta: messaging-ingestion

## Purpose

Garantiza que todo dato del catálogo llegue a Cassandra a través de una mensajería confiable. La entrega es durable, los reintentos están acotados, las fallas permanentes se aíslan en una cola de errores y el procesamiento es idempotente. Además, emite eventos que hacen observable cada paso.

## ADDED Requirements

### Requirement: Camino único de escritura
El sistema SHALL escribir datos en el Data Mart solo desde los consumidores de la capa de ingesta, en respuesta a mensajes. Ningún otro componente (ETL ni aplicación web) escribe directamente en Cassandra.

#### Scenario: Acción iniciada desde la interfaz
- **WHEN** un usuario lanza una ingesta desde el dashboard
- **THEN** la acción se traduce en un mensaje y los datos llegan a Cassandra únicamente a través del consumidor de ingesta

### Requirement: Topología durable
El broker SHALL tener:
- un exchange de catálogo enrutado por tema;
- colas durables separadas para títulos, agregados y control de corrida;
- una cola de errores por cada cola de consumo;
- colas de comandos hacia el ETL y hacia la ingesta.

Los mensajes persisten ante reinicios del broker.

#### Scenario: Reinicio del broker con mensajes pendientes
- **WHEN** el broker se reinicia mientras hay mensajes sin consumir en la cola de títulos
- **THEN** después del reinicio esos mensajes siguen en la cola y se consumen

#### Scenario: Enrutamiento por tipo de mensaje
- **WHEN** el ETL publica un mensaje de título y uno de agregado
- **THEN** cada uno llega solo a su cola correspondiente

### Requirement: Aceptación de mensajes publicados por Python
Los consumidores SHALL aceptar mensajes JSON planos publicados por el ETL, con nombres de campo en snake_case, sin requerir un sobre propio del framework de mensajería.

#### Scenario: Mensaje de título desde Python
- **WHEN** el ETL publica un documento de título en JSON plano
- **THEN** el consumidor de títulos lo interpreta correctamente y lo almacena

### Requirement: Persistencia de títulos
Al consumir un mensaje de título válido, el sistema SHALL escribir el título en todas sus tablas de consulta y sus créditos en la filmografía de cada persona. Después incrementa el progreso de la corrida y confirma el mensaje.

#### Scenario: Título nuevo
- **WHEN** se consume un mensaje válido del título `tm82169`
- **THEN** el título se puede consultar por id, por cada uno de sus géneros y países, y por tipo y década, y sus créditos aparecen en la filmografía de cada persona

### Requirement: Persistencia de agregados
Al consumir un snapshot de agregado, el sistema SHALL reemplazar con upsert las filas de ese agregado en su tabla.

#### Scenario: Snapshot de KPIs
- **WHEN** se consume el snapshot de KPIs de una corrida
- **THEN** las tarjetas de KPI reflejan los valores de ese snapshot

### Requirement: Procesamiento idempotente
El sistema SHALL procesar cada identificador de mensaje como máximo una vez con efecto. Un mensaje repetido se confirma sin escribir datos, se contabiliza como duplicado y se publica un evento de duplicado omitido.

#### Scenario: Mensaje duplicado
- **WHEN** llega por segunda vez un mensaje con un identificador ya procesado
- **THEN** no se modifica ningún dato analítico, el contador de duplicados de la corrida aumenta en uno y se emite el evento de duplicado

#### Scenario: Reingesta completa
- **WHEN** el catálogo completo se ingiere dos veces
- **THEN** el número de títulos en el Data Mart sigue siendo 6,137

### Requirement: Reintentos ante fallas transitorias
Ante una falla transitoria (indisponibilidad o timeout de la base de datos, o falla simulada de demo), el sistema SHALL reintentar el mensaje hasta 3 veces con espera exponencial. Antes de cada reintento revierte el registro de mensaje procesado, para que el reintento no se tome por duplicado.

#### Scenario: Falla transitoria que se recupera
- **WHEN** el procesamiento de un mensaje falla en los dos primeros intentos y tiene éxito en el tercero
- **THEN** el título queda almacenado una sola vez y se emiten eventos de reintento por cada intento fallido

#### Scenario: Falla transitoria persistente
- **WHEN** el procesamiento falla en todos los intentos permitidos
- **THEN** el mensaje se mueve a la cola de errores con la descripción de la excepción

### Requirement: Aislamiento de fallas permanentes
Un mensaje que no cumple el contrato (JSON inválido, campos obligatorios faltantes, valores fuera de dominio) SHALL moverse a la cola de errores sin reintentos. La cola principal no se bloquea.

#### Scenario: Mensaje corrupto
- **WHEN** se consume un mensaje de título sin `id` y con `release_year = "abc"`
- **THEN** el mensaje aparece en la cola de errores, se emite un evento de falla con el motivo, y los mensajes siguientes se procesan con normalidad

### Requirement: Reprocesamiento de la cola de errores
El sistema SHALL permitir reenviar los mensajes de la cola de errores a la cola de origen. Los recuperables se procesan y los que vuelven a fallar regresan a la cola de errores, sin que se pierda ningún mensaje.

#### Scenario: Reenvío de la cola de errores
- **WHEN** se solicita reprocesar una cola de errores con un mensaje recuperable y uno corrupto
- **THEN** el recuperable se almacena y el corrupto vuelve a la cola de errores

### Requirement: Pausa y reanudación del consumo
La capa de ingesta SHALL poder pausar y reanudar el consumo de la cola de títulos por comando. Durante la pausa, los mensajes se acumulan en la cola sin perderse.

#### Scenario: Pausa durante una corrida
- **WHEN** se pausa el consumidor de títulos con una corrida en curso
- **THEN** los mensajes pendientes en la cola aumentan y no se almacenan títulos nuevos

#### Scenario: Reanudación
- **WHEN** se reanuda el consumidor
- **THEN** la cola se vacía y la corrida llega al total esperado

### Requirement: Seguimiento del ciclo de vida de una corrida
El sistema SHALL registrar el inicio de una corrida con su total esperado. La corrida se marca como completada cuando los títulos almacenados igualan al total esperado y ya se recibió el evento de fin.

#### Scenario: Corrida completada
- **WHEN** se han almacenado todos los títulos esperados y se recibió el evento de fin
- **THEN** la corrida queda con estado completada y se emite un evento de cambio de estado

### Requirement: Eventos de observabilidad
La capa de ingesta SHALL publicar un evento por cada uno de estos hechos:
- título almacenado (con su latencia de procesamiento);
- duplicado omitido;
- reintento;
- falla enviada a la cola de errores;
- progreso de la corrida;
- cambio de estado de la corrida.

#### Scenario: Título almacenado
- **WHEN** un título se almacena correctamente
- **THEN** se publica un evento con el id del título, su nombre, el id de la corrida y la latencia en milisegundos
