# Pipeline Byma

Este proyecto es un pipeline de ingesta y transformación de data de byma hacia el data lakehouse de databricks.
Para el diseño se uso una arquitectura medallion qué permitió separar la ingesta de distintas fuentes de datos en un modelo de datos unificado, y limpio.
El proyecto consiste en 3 capas, bronce silver y gold, las cuáles permiten desacoplar el mantenimiento de la ingesta con el consumo del negocio.

- Bronce:
Se ingesta desde la fuente de datos qué corresponda (csv, s3, api externa) hacia una delta table particionada por dia, mes y año (Este particionamiento permite leer datos de manera eficiente en el caso qué el volumen de información escale ya qué evita tener qué cargar particiones no correspondan). En este caso ingestamos desde un csv qué esta cargado en un volumen de databricks. Esta etapa tiene dos roles, el de mantener un almancenamiento historico de toda la información ingestada más allá de los esquemas de cada modelo y el de mantener columnas qué monitorean la calidad de los datos ingestados. Todas las capas posteriores a esta pueden ser reconstruidas con la información de esta etapa. Además cada dato tiene cargado un timestamp con la fecha en la qué se ingestó. Todo lo procesado por esta etapa es mediante el formato append.

- Silver:
Según cuál sea el modelo a ingestar (instruments o transactions), se genera el esquema correspondiente qué se carga en una nueva delta table qué ya representa la información sin duplicados. Esta etapa es respondable de agregar una clave única a cada dato insertado y se encarga de asegurar la última información disponible para cada clave. Además se priorizan para cargar aquellos datos qué pasaron el monitoreo de calidad. Si llega información nueva, esta etapa es responsable de actualizarla. La idempotencia de esta solución es garantizada gracias a este proceso. 

- Gold:
Esta etapa consiste en una seríe de modelos factuales y dimensionales con información agregada y enriquecida qué estan listos para ser consumidos por el negocio. Estos modelos tienen cada uno sus claves surrogadas.

Modelos Gold incluidos:
- Clientes (Dim)
- Cotizaciones (Dim)
- Instrumentos (Dim)

- Transacciones: Granularidad por dia por cada cliente (Fact)


<img width="932" height="456" alt="image" src="https://github.com/user-attachments/assets/ba99ea92-1a4d-4af8-97c0-0e9dc025ba25" />
