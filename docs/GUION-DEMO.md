# Guion de demo (10 minutos como máximo)

## Antes de presentar (no cuenta en el tiempo)

1. Cierra programas pesados.
2. Empieza desde cero y espera a que todo esté arriba (1 a 3 minutos):
   ```bash
   docker compose down -v
   docker compose up -d --build
   docker compose --profile tools build etl
   ```
3. Abre dos pestañas: el dashboard (<http://localhost:8080>) y RabbitMQ (<http://localhost:15672>, `guest`/`guest`).
4. Deja dos terminales listas en la carpeta del proyecto.

Si el 8080 está ocupado, antepón `APP_PORT=8081` a los comandos `docker compose up`.

## Guion

| Minuto | Qué hacer | Qué decir |
|---|---|---|
| 0:00 | Mostrar **Resumen** vacío | Todavía no hay datos; la propia vista dice cómo cargarlos. |
| 0:45 | Explicar el diagrama del README | CSV → ETL Python → RabbitMQ → consumidor .NET → Cassandra. Solo los consumidores escriben. |
| 1:45 | Ir a **Mensajería** y ejecutar `docker compose run --rm etl --rate 60` | Cada título viaja como un mensaje persistente y confirmado. Señalar la barra, el feed y la gráfica. |
| 2:30 | En otra terminal: `docker compose stop app` | La app muere a mitad de la carga. En RabbitMQ, `catalog-titles` queda con 0 consumidores y "Ready" crece: no se pierde nada. |
| 3:30 | `docker compose start app` y volver a **Mensajería** | Tras unos segundos de arranque, la cola baja hasta 0. La app retoma donde quedó. |
| 4:30 | **Inyectar corrupto** | Título sin `id`: 3 reintentos y termina en `catalog-titles_error`. Los demás siguen. |
| 5:15 | **Reenviar duplicado** | Sale "ya existía". En Cassandra todo `INSERT` es un upsert: no se duplica. |
| 6:00 | **Resumen** | 6,137 títulos. Clic en la barra de `drama`. Cada gráfica lee una tabla ya agregada, sin JOINs. |
| 7:15 | **Catálogo**: buscar "rocky", filtrar Películas, abrir Rocky | El detalle y el reparto salen de una sola lectura. |
| 8:15 | **Ranking**: IMDb, Series | Top 50 precalculado; exige al menos 1,000 votos. |
| 8:45 | **Personas**: clic en un actor | La filmografía es una partición ordenada por año. |
| 9:15 | Cierre | Modelo orientado a consultas y mensajería con confirmación, reintentos, DLQ e idempotencia. |

## Si algo falla

| Síntoma | Qué hacer |
|---|---|
| La página no responde | `docker compose ps` y `docker compose logs app --tail 50` |
| Cassandra tarda en arrancar | Es normal: espera a que aparezca `healthy` |
| "Reenviar duplicado" está deshabilitado | La app acaba de arrancar y aún no recibió ningún título |
| La cola de errores ya tiene mensajes de un ensayo | Empieza de nuevo con `docker compose down -v` |

## Preguntas probables

- **¿Por qué no un esquema estrella?** Cassandra no hace JOINs ni `GROUP BY` entre particiones; se diseña una tabla por consulta.
- **¿Por qué RabbitMQ y no Kafka?** Confirmación por mensaje, DLQ nativa y una interfaz lista para enseñar.
- **¿Qué garantiza que no haya duplicados?** La clave natural: escribir dos veces el mismo título deja una sola fila.
- **¿Es "exactamente una vez"?** No. La entrega es "al menos una vez" y la idempotencia hace que repetir no tenga efecto.
- **¿Qué pasa si la app se cae?** Los mensajes sin confirmar vuelven a la cola, que es durable.
