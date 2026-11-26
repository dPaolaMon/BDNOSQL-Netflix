# Spec Delta: dashboard-ui

## Purpose

Presenta las estadísticas del catálogo de Netflix en un dashboard web interactivo con tema oscuro estilo Netflix. Tiene vistas de resumen, géneros, países, ranking, catálogo y personas, alimentadas por el Data Mart y actualizadas en vivo durante la ingesta.

## ADDED Requirements

### Requirement: Layout y navegación
El dashboard SHALL tener una barra lateral con la marca "Netflix Analytics" y enlaces a: Resumen, Géneros, Países, Ranking, Catálogo, Personas y Mensajería. Una barra superior muestra el título de la vista y el indicador de conexión en vivo. El enlace activo se resalta.

#### Scenario: Navegación entre vistas
- **WHEN** el usuario pulsa "Géneros" en la barra lateral
- **THEN** se muestra la vista de géneros sin recargar la página y el enlace queda resaltado

### Requirement: Tema visual oscuro
El dashboard SHALL usar un tema oscuro: fondo casi negro, acento rojo, rojo para películas, amarillo para series, y colores de estado verde (éxito) y rojo (error). Los textos deben tener un contraste mínimo AA sobre el fondo.

#### Scenario: Codificación de color por tipo
- **WHEN** una gráfica distingue películas de series
- **THEN** las películas usan el color de película y las series el color de serie en todas las vistas

### Requirement: Vista Resumen
La vista Resumen SHALL mostrar:
- tarjetas KPI: títulos, películas, series, IMDb promedio y personas;
- un área apilada de estrenos por año y tipo;
- una dona de películas vs. series;
- barras horizontales de los géneros principales;
- una dona de clasificación por edad.

#### Scenario: Resumen con datos
- **WHEN** el usuario abre Resumen con el catálogo ingerido
- **THEN** la tarjeta de títulos muestra 6,137 y las cuatro gráficas se dibujan con datos

### Requirement: Actualización en vivo del Resumen
Durante una corrida de ingesta, la vista Resumen SHALL actualizar sus KPIs y gráficas cuando se almacenan nuevos agregados, sin recargar la página.

#### Scenario: KPIs durante ingesta
- **WHEN** se almacenan los agregados de una nueva corrida mientras Resumen está abierta
- **THEN** los KPIs se actualizan con animación sin intervención del usuario

### Requirement: Vista Géneros
La vista Géneros SHALL mostrar un heatmap género × década con el número de títulos y tarjetas por género con puntuación IMDb y duración promedio. Al seleccionar un género se abre la lista de sus mejores títulos.

#### Scenario: Seleccionar un género
- **WHEN** el usuario pulsa la fila `drama` del heatmap
- **THEN** se abre un panel con los títulos de drama ordenados por puntuación IMDb descendente

### Requirement: Vista Países
La vista Países SHALL mostrar un mapa mundial coroplético coloreado por número de producciones y un listado de los 15 países principales con total, proporción película/serie y género dominante. Al seleccionar un país se listan sus títulos.

#### Scenario: Seleccionar un país en el mapa
- **WHEN** el usuario pulsa Estados Unidos en el mapa
- **THEN** se muestran el total de producciones de US y la lista de sus títulos

#### Scenario: Tooltip del mapa
- **WHEN** el usuario pasa el cursor sobre un país con producciones
- **THEN** se muestra un tooltip con el nombre del país y su total de títulos

### Requirement: Vista Ranking
La vista Ranking SHALL mostrar los 50 mejores títulos según el criterio elegido (IMDb o popularidad), filtrables por tipo (todos, películas, series) y por década (todas o una específica).

#### Scenario: Cambio de filtro
- **WHEN** el usuario elige el ranking IMDb, tipo "Series" y década 2010
- **THEN** la lista muestra hasta 50 series de la década de 2010 numeradas por posición

### Requirement: Vista Catálogo
La vista Catálogo SHALL mostrar una tabla paginada de títulos con búsqueda por texto, filtros por tipo, género, década y rango de puntuación, y columnas ordenables.

#### Scenario: Buscar y filtrar
- **WHEN** el usuario escribe "rocky" y filtra por "Películas"
- **THEN** la tabla muestra solo películas cuyo título contiene "rocky" y el total de resultados

### Requirement: Detalle de título
Al seleccionar un título en cualquier vista, el dashboard SHALL abrir un panel de detalle con:
- sinopsis, año, tipo, duración o temporadas, y clasificación;
- puntuaciones IMDb y TMDB, votos y popularidad;
- géneros, países, directores y reparto con sus personajes.

#### Scenario: Abrir detalle
- **WHEN** el usuario selecciona "Rocky" en el catálogo
- **THEN** se abre el panel con su sinopsis, IMDb 8.1, el director y el reparto con sus personajes

#### Scenario: Navegar a una persona desde el detalle
- **WHEN** el usuario pulsa un nombre del reparto en el detalle
- **THEN** se abre la ficha de esa persona en la vista Personas

### Requirement: Vista Personas
La vista Personas SHALL mostrar pestañas de actores y directores con las 100 personas con más títulos. Al seleccionar una persona se muestra su ficha con una línea de tiempo (año × puntuación) y la tabla de su filmografía.

#### Scenario: Ficha de una persona
- **WHEN** el usuario selecciona un actor del top
- **THEN** se muestran su número de títulos, su puntuación media y su filmografía ordenada por año descendente

### Requirement: Estados de carga, vacío y error
Cada vista SHALL mostrar un estado de carga mientras obtiene datos. Si el Data Mart está vacío, muestra un estado vacío con un acceso al Centro de Mensajería para ingerir. Si una consulta falla, muestra un mensaje de error con opción de reintentar.

#### Scenario: Data Mart vacío
- **WHEN** el usuario abre Resumen antes de cualquier ingesta
- **THEN** se muestra un estado vacío con un botón que lleva al Centro de Mensajería

#### Scenario: Error de consulta
- **WHEN** una consulta falla porque la base de datos no responde
- **THEN** la sección afectada muestra un mensaje de error y un botón "Reintentar", y el resto de la vista sigue funcionando

### Requirement: Diseño adaptable
El dashboard SHALL adaptarse a anchos desde 360 px hasta pantallas de escritorio. En pantallas angostas, la barra lateral se colapsa en un menú y las tarjetas y gráficas se apilan en una columna.

#### Scenario: Pantalla de teléfono
- **WHEN** el dashboard se abre con un ancho de 375 px
- **THEN** la barra lateral está colapsada y no hay desplazamiento horizontal de la página
