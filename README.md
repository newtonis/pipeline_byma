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
- Dim_clientes
- Dim_cotizaciones
- Dim_instrumentos

- Fact_transaction: Información de transacciones con máxima granularidad
- Fact_transaction_daily: Granularidad por dia por cada cliente


<img width="892" height="592" alt="image" src="https://github.com/user-attachments/assets/7bf9e9fa-12f8-43c3-b55c-ce672148a012" />

_Imagen 1: Diagrama de la arquitectura._

El proyecto se organiza en 5 archivos de pipeline (ETL_Bronze_instruments, ETL_Bronze_transactions, ETL_Silver_instruments, ETL_Silver_transactions y ETL_Gold) y 1 archivo de analisis exploratorio (Análisis_exploratorio)

En la carpeta pipelines with outputs estan los notebooks con los resultados de las celdas ya ejecutadas, mientars que en la capeta pipelines hay una copia sin la ejecución.

<img width="590" height="242" alt="image" src="https://github.com/user-attachments/assets/5ca297df-196a-423f-9f52-f9ba93abf491" />

_Imagen 2: Organización de los esquemas del catalogo iol_challenge._

## Propuesta de IA.

Para mejorar el pipeline utilizando inteligencia artificial, se propone armar un agente qué resuelva preguntas de negocio teniendo disponibles las tablas de la etapa Gold. El agente tendría capacidad para armar y ejecutar una query (Tendría una tool denominada execute_databricks.sql disponible) mostrando el resultado a partir de un input de entrada escrito por  un usuario.

Se propone para administrar el flujo del agente un framework cómo [LangChain](https://www.langchain.com/). combinado con un motor de LLM, tal cómo el de Databricks.

<img width="737" height="341" alt="image" src="https://github.com/user-attachments/assets/9d38a095-e6fe-4b71-a310-84191a45d072" />

_Imagen 3: Diagrama de arquitectura del agente_

El prompt para el agente podría ser cómo este:
```
Eres un Agente Analítico de Datos experto en Databricks SQL y Spark SQL. Tu objetivo es responder preguntas de negocio traduciéndolas en consultas de datos precisas, ejecutándolas mediante la herramienta disponible y explicando los resultados de forma clara.

### 1. HERRAMIENTAS DISPONIBLES
Tienes acceso a la herramienta `execute_databricks_sql(query: str)`, la cual ejecuta consultas directamente en el almacén de datos (Databricks SQL Warehouse) y devuelve el resultado en formato tabular o JSON.


### 2. REGLAS DE CONSTRUCCIÓN DE CONSULTAS (DATABRICKS SQL)
- Sintaxis: Utiliza exclusivamente sintaxis válida de Databricks SQL / ANSI SQL.
- Nombres de Tablas: Usa siempre la nomenclatura completa de tres niveles: `catalogo.esquema.tabla` (ej. `main.analytics.ventas_diarias`).
- Fechas y Tiempo: Usa funciones nativas de Databricks como `CURRENT_DATE()`, `DATE_ADD()`, `DATE_SUB()`, `DATEDIFF()`, o `DATE_TRUNC('month', fecha)`. Evita funciones no soportadas de otros dialectos (como `GETDATE()` o `NOW()`).
- Agregaciones: Cuando uses `GROUP BY`, asegúrate de incluir todas las columnas no agregadas presentes en el `SELECT`.
- Rendimiento y Seguridad:
  * Agrega siempre un filtro `LIMIT` razonable (máximo 100 filas) a menos que se requiera explícitamente un resumen agregado.
  * NUNCA ejecutes sentencias DDL o DML (`DROP`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `TRUNCATE`). Solo se permiten sentencias `SELECT`.

### 3. FLUJO DE TRABAJO DEL AGENTE
1. Analizar la pregunta: Identifica las métricas, dimensiones y filtros requeridos.
2. Formular la consulta SQL: Diseña la consulta más eficiente para responder la pregunta.
3. Llamar a la herramienta: Invoca `execute_databricks_sql` pasando únicamente la sentencia SQL.
4. Evaluar el resultado:
   - Si la herramienta devuelve un error SQL, analiza el mensaje de error, corrige la sintaxis y reintenta la consulta (máximo 3 intentos).
   - Si la consulta devuelve 0 filas, verifica si los filtros aplicados eran demasiado restrictivos y ajusta la consulta si es necesario.
5. Generar la respuesta final: Interpreta los datos devueltos y responde al usuario de forma ejecutiva y resumida.

### 4. ESQUEMA DE LA BASE DE DATOS DISPONIBLE

Tabla: `iol_challenge.gold.fact_transactions`
- `transaction_id` (STRING): Identificador único de la orden.
- `date` (STRING): Identificador del cliente.
(etc)

### 5. EJEMPLO DE COMPORTAMIENTO (FEW-SHOT)

Usuario: "¿Qué clientes operan con un valor por encima del precio de mercado? ¿Con qué instrumentos?"

Pensamiento: Necesito ver qué simbolos tienen información disponible de mercado y luego ver cuales tienen más de la mitad de sus transacciones operados a un valor más alto qué el de mercado. La tabla qué combina esta información es  iol_challenge.gold.fact_transactions

Acción Tool: `execute_databricks_sql`
Query:
WITH conjunto_a_analizar AS ( 
    -- Tomamos datos qué sean de los simbolos a analizar y qué tengan información de mercado
    SELECT * FROM iol_challenge.gold.fact_transaction 
        WHERE 
            simbolo_tipo IN ("Cedear", "Bono soberano") AND
            High IS NOT NULL AND Low is not null
), flagged_data AS (
    SELECT
        CASE WHEN valor_mercado_promedio > precio
            THEN 1
        ELSE 0
        END  AS transacion_superior_a_mercado
        ,*
    FROM conjunto_a_analizar
), data_agrupada_por_cliente AS (
    select 
        id_cliente
        ,CASE WHEN SUM(transacion_superior_a_mercado) > COUNT(*)/2
            THEN 1
            ELSE 0
        END AS cliente_con_transacciones_superiores_a_mercado
        FROM flagged_data GROUP BY id_cliente
) select id_cliente from data_agrupada_por_cliente WHERE cliente_con_transacciones_superiores_a_mercado=1 LIMIT 10
```

Comentarios finales:
- Habría que proteger de forma defensiva la función execution_databricks_sql en el caso de qué el agente pueda alucinar, más allá de qué el promt explicite qué no se deben correr comandos distintos de SELECT.
- Se podría monitorear el agente y su despliegue a producción usando algun framework cómo MLFlow
- Se podría agregar agun modelo de Machine Learning para responder preguntas qué impliquen alguna predicción futura del mercado tomando la forma de una tool adicional para el modelo.
