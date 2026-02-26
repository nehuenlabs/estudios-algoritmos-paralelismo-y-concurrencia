# Guía de Ejercicios — Cap.24: Paralelismo en Apache Spark

> Lenguaje principal: **Python (PySpark)**. Los conceptos aplican a Scala y Java.
>
> Este capítulo invierte el modelo de todo el repositorio anterior.
> En Cap.01–23, tú controlabas el paralelismo: goroutines, canales, locks.
> En Spark, lo describes y el framework decide cómo ejecutarlo.
>
> El error más costoso que comete alguien que llega de programación concurrente
> a Spark es intentar controlar el paralelismo. El segundo error más costoso
> es no entender cómo funciona para poder diagnosticar cuando falla.
>
> Este capítulo cubre ambos errores.

---

## El modelo mental de Spark en una página

```
Programación concurrente (Cap.01–21):
  Tú decides: cuántos goroutines/threads, cuándo se comunican, cómo se sincronizan
  El paralelismo es explícito
  Los bugs son: races, deadlocks, starvation

Spark:
  Tú describes: qué transformaciones hacer sobre qué datos
  Spark decide: cuántas tasks, en qué nodos, en qué orden
  Los bugs son: skew, shuffles innecesarios, UDFs lentas, OOM en executors

La unidad de paralelismo en Spark no es el thread — es la PARTICIÓN.
  Cada partición se procesa por una task.
  Cada task corre en un executor (un proceso JVM en un worker node).
  El grado de paralelismo = número de particiones procesadas simultáneamente.
```

```
Driver (tu código Python/Scala):
  - Construye el DAG de transformaciones
  - Negocia recursos con el cluster manager
  - Coordina la ejecución
  - NO procesa datos (solo coordina)

Executor (JVM en cada worker node):
  - Procesa las particiones
  - Guarda datos en memoria/disco para stages siguientes
  - Reporta progreso al driver
```

---

## Por qué la GIL importa en PySpark (pero menos de lo que crees)

```
PySpark sin UDFs:
  Driver Python → instrucciones → Spark JVM → ejecuta en executors JVM
  La GIL solo afecta al driver (que no procesa datos)
  El procesamiento ocurre en Java → sin GIL

PySpark con UDFs de Python:
  Para cada fila: JVM → deserializar → Python → ejecutar función → serializar → JVM
  La GIL bloquea el procesamiento paralelo dentro de cada executor
  El overhead de serialización puede ser 10-100x el costo de la función

PySpark con Pandas UDFs (vectorizadas con Arrow):
  Para cada batch: JVM → Arrow → Pandas DataFrame → función → Arrow → JVM
  Sin serialización fila a fila
  Sin GIL por fila (Arrow es C++, no Python puro)
  10-100x más rápido que UDFs escalares
```

---

## Tabla de contenidos

- [Sección 24.1 — El DAG: cómo Spark planifica la ejecución](#sección-241--el-dag-cómo-spark-planifica-la-ejecución)
- [Sección 24.2 — Particiones: la unidad real de paralelismo](#sección-242--particiones-la-unidad-real-de-paralelismo)
- [Sección 24.3 — Shuffle: el cuello de botella distribuido](#sección-243--shuffle-el-cuello-de-botella-distribuido)
- [Sección 24.4 — Data skew: cuando el paralelismo falla](#sección-244--data-skew-cuando-el-paralelismo-falla)
- [Sección 24.5 — UDFs: cuándo Python entra al executor](#sección-245--udfs-cuándo-python-entra-al-executor)
- [Sección 24.6 — Memoria y spill: cuando los datos no caben](#sección-246--memoria-y-spill-cuando-los-datos-no-caben)
- [Sección 24.7 — Spark UI: diagnosticar en producción](#sección-247--spark-ui-diagnosticar-en-producción)

---

## Sección 24.1 — El DAG: Cómo Spark Planifica la Ejecución

### Ejercicio 24.1.1 — Leer: transformaciones lazy vs acciones

Spark no ejecuta nada hasta que encuentras una **acción**. Todo antes es una
descripción del trabajo — el DAG de transformaciones.

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("cap24").getOrCreate()

# Ninguna de estas líneas ejecuta nada:
df = spark.read.parquet("s3://bucket/ventas/")        # (1)
df_filtrado = df.filter(F.col("año") == 2024)         # (2)
df_agrupado = df_filtrado.groupBy("region").agg(      # (3)
    F.sum("monto").alias("total"),
    F.count("*").alias("num_transacciones")
)
df_enriquecido = df_agrupado.join(                    # (4)
    spark.read.parquet("s3://bucket/regiones/"),
    on="region"
)

# Esta línea ejecuta TODO lo anterior:
df_enriquecido.write.parquet("s3://bucket/resultado/") # (5) ← acción
```

**Preguntas:**

1. ¿Cuántos jobs genera esta secuencia? ¿Cuántos stages?
2. En qué punto Spark lee físicamente los archivos de `s3://bucket/ventas/`
3. Si añades `.show()` después de la línea (2) y luego ejecutas la línea (5),
   ¿cuántas veces se lee el archivo de ventas?
4. ¿Cómo evitarías la doble lectura del punto anterior?
5. ¿Qué es "predicate pushdown" y cómo se aplica al filtro de la línea (2)?

**Pista:** El predicate pushdown es cuando Spark empuja el filtro `año == 2024`
al lector de Parquet — en lugar de leer todo el archivo y luego filtrar,
Parquet solo lee los row groups donde `año == 2024`. Si el archivo está
particionado por año en S3 (`/ventas/año=2024/`), ni siquiera lee las carpetas
de otros años. Verificar con `df.explain(True)` — el plan físico mostrará
`PushedFilters` si el pushdown está activo.

---

### Ejercicio 24.1.2 — explain(): leer el plan de ejecución

`explain()` es el equivalente de `goroutine dump` del Cap.19 para Spark.
Cada problema de rendimiento en Spark empieza aquí.

```python
df = spark.read.parquet("s3://bucket/ventas/")
df_resultado = (df
    .filter(F.col("año") == 2024)
    .groupBy("region", "producto")
    .agg(F.sum("monto").alias("total"))
    .filter(F.col("total") > 1000000)
    .orderBy(F.col("total").desc())
)

df_resultado.explain(True)
```

Dado este plan (simplificado):
```
== Physical Plan ==
AdaptiveSparkPlan (isFinalPlan=false)
+- Sort [total DESC NULLS LAST]
   +- Exchange rangepartitioning(total DESC, 200)    ← shuffle
      +- Filter (total > 1000000)
         +- HashAggregate [region, producto] (partial)
            +- Exchange hashpartitioning(region, producto, 200)  ← shuffle
               +- Filter (año = 2024)
                  +- FileScan parquet [año,region,producto,monto]
                     PartitionFilters: [isnotnull(año), (año = 2024)]
                     PushedFilters: [IsNotNull(año), EqualTo(año,2024)]
```

**Preguntas:**

1. ¿Cuántos shuffles tiene este plan? ¿Para qué sirve cada uno?
2. ¿Dónde está el predicate pushdown del filtro `año == 2024`?
3. ¿Por qué el `Filter (total > 1000000)` está DESPUÉS del HashAggregate
   y no antes? ¿Podría Spark haberlo puesto antes?
4. ¿Qué significa `partial` en `HashAggregate (partial)`?
5. ¿Qué es `AdaptiveSparkPlan` y qué cambia en el plan cuando `isFinalPlan=true`?

**Pista:** El `HashAggregate (partial)` es la parte Map de un Map/Reduce:
cada executor hace una agregación parcial local antes del shuffle, reduciendo
los datos que viajan por la red. `AdaptiveSparkPlan` (AQE) puede cambiar
el número de particiones del shuffle en runtime basándose en el tamaño real
de los datos — `isFinalPlan=true` muestra el plan después de esos ajustes.

---

### Ejercicio 24.1.3 — Caché: cuándo materializar el DAG

```python
# Sin caché: el DAG se re-ejecuta completo para cada acción
df_ventas = (spark.read.parquet("s3://bucket/ventas/")
    .filter(F.col("año") == 2024)
    .join(spark.read.parquet("s3://bucket/productos/"), on="producto_id")
)

# Estas dos acciones ejecutan el DAG dos veces (doble lectura de S3):
conteo = df_ventas.count()           # acción 1
df_ventas.write.parquet("s3://...")  # acción 2

# Con caché: el DAG se ejecuta una vez y se guarda en memoria de executors
df_ventas.cache()  # o .persist(StorageLevel.MEMORY_AND_DISK)
df_ventas.count()  # primera acción: ejecuta el DAG Y guarda en caché
df_ventas.write.parquet("s3://...")  # segunda acción: lee de caché

# Limpiar la caché cuando ya no se necesita:
df_ventas.unpersist()
```

**Restricciones:** Implementar y medir:
1. Sin caché: dos acciones sobre el mismo DataFrame costoso
2. Con `.cache()`: las mismas dos acciones
3. Con `.persist(StorageLevel.DISK_ONLY)`: ¿cuándo es mejor que MEMORY?
4. Con `.persist(StorageLevel.MEMORY_AND_DISK_SER)`: ¿qué significa `SER`?

Usar Spark UI para verificar que la segunda acción lee de caché
(el stage marcado como "skipped" en la UI).

**Pista:** El nivel de storage `MEMORY_AND_DISK` guarda sin serializar —
más rápido de leer pero usa más memoria. `MEMORY_AND_DISK_SER` serializa
los datos (con Kryo) — ocupa menos memoria pero requiere deserializar al leer.
Para DataFrames grandes donde la memoria es limitada, `SER` puede ser la
diferencia entre funcionar y hacer spill to disk.

---

### Ejercicio 24.1.4 — Broadcast variables: compartir datos pequeños eficientemente

```python
# Sin broadcast: el DataFrame pequeño se envía como parte del shuffle
# Spark hace un sort-merge join → shuffle de ambos DataFrames
resultado = df_ventas_grande.join(df_productos_pequeño, on="producto_id")

# Con broadcast: df_productos_pequeño se envía completo a cada executor
# Sin shuffle → mucho más rápido
from pyspark.sql.functions import broadcast

resultado = df_ventas_grande.join(
    broadcast(df_productos_pequeño),
    on="producto_id"
)

# También puede configurarse automáticamente:
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100mb")
# Spark hará broadcast automático de cualquier tabla < 100MB
```

**Restricciones:**
1. Comparar el plan de ejecución (`.explain()`) con y sin broadcast
2. Medir el tiempo de ejecución con ambos approaches
3. ¿Cuál es el límite práctico para hacer broadcast? ¿Qué pasa si haces
   broadcast de un DataFrame demasiado grande?
4. Implementar el caso donde el broadcast falla con OOM y cómo detectarlo

**Pista:** Si el DataFrame que haces broadcast es > la memoria de los executors,
el job falla con OOM. El síntoma: el stage que hace el broadcast muere con
`java.lang.OutOfMemoryError`. El límite seguro depende del cluster pero
generalmente se mantiene bajo 1-2GB. Para datos entre 1-10GB, considerar
bucket join (pre-particionamiento) en lugar de broadcast.

---

### Ejercicio 24.1.5 — Leer: el job que hace más trabajo del necesario

**Tipo: Diagnosticar**

El siguiente código funciona pero es innecesariamente lento. Identifica los problemas:

```python
# Pipeline de análisis de ventas diarias
df = spark.read.parquet("s3://bucket/ventas/")  # 5 TB, particionado por fecha

# Análisis 1: ventas de hoy
hoy = df.filter(F.col("fecha") == "2024-01-15")
total_hoy = hoy.count()
promedio_hoy = hoy.agg(F.avg("monto")).collect()[0][0]
max_hoy = hoy.agg(F.max("monto")).collect()[0][0]

# Análisis 2: top 10 productos de hoy
top_productos = (hoy
    .groupBy("producto")
    .agg(F.sum("monto").alias("total"))
    .orderBy(F.col("total").desc())
    .limit(10)
    .collect()
)

# Análisis 3: ventas de la última semana para comparación
ultima_semana = df.filter(
    (F.col("fecha") >= "2024-01-08") & (F.col("fecha") <= "2024-01-15")
)
total_semana = ultima_semana.agg(F.sum("monto")).collect()[0][0]
```

**Preguntas:**

1. ¿Cuántas veces se lee el archivo de 5 TB en total?
2. ¿Cuántos jobs se ejecutan? (cada `.collect()`, `.count()` es un job)
3. ¿Cómo reducirías el número de lecturas de S3?
4. ¿Cómo combinarías los análisis 1 y 2 en un solo job?
5. Reescribe el pipeline optimizado.

**Pista:** `count()`, `collect()`, `show()` son acciones — cada una dispara
un job. El DataFrame `hoy` se re-computa para cada acción porque no está cacheado.
Con 5 TB y tres acciones sobre `hoy`, se leen 15 TB de S3.
La optimización: (1) cachear `hoy` si se usa múltiples veces,
(2) combinar múltiples aggregations en un solo `agg()`,
(3) usar `.summary()` o `describe()` para estadísticas básicas en un solo pass.

---

## Sección 24.2 — Particiones: la Unidad Real de Paralelismo

### Ejercicio 24.2.1 — Entender el particionamiento actual

```python
# Ver cuántas particiones tiene un DataFrame:
df = spark.read.parquet("s3://bucket/ventas/")
print(f"Particiones: {df.rdd.getNumPartitions()}")

# Distribución de filas por partición:
from pyspark.sql import functions as F

df_con_particion = df.withColumn(
    "partition_id",
    F.spark_partition_id()
)
df_con_particion.groupBy("partition_id").count().orderBy("partition_id").show()
```

Dado este output:
```
+------------+--------+
|partition_id|   count|
+------------+--------+
|           0| 125,432|
|           1| 124,891|
|           2| 126,103|
|           3|      47|   ← !!
|           4| 125,220|
|           5| 124,789|
+------------+--------+
```

**Preguntas:**

1. ¿Qué indica la partición 3 con solo 47 filas?
2. ¿Cómo afecta esto al tiempo de ejecución del job?
3. ¿Cuál es la causa más probable de este patrón?
4. ¿Cómo lo corregirías?
5. ¿Cuál es el número óptimo de particiones para este dataset
   en un cluster de 20 cores?

**Pista:** La partición 3 con 47 filas es un síntoma de que los archivos de origen
en S3 tienen tamaños muy desiguales — un archivo muy pequeño (quizás un archivo
de "late data" o de configuración) se convirtió en una partición diminuta.
El número óptimo de particiones = 2-4× el número de cores disponibles.
Con 20 cores: 40-80 particiones. Con 6 particiones actuales, 14 cores están
ociosos mientras las 6 tasks corren.

---

### Ejercicio 24.2.2 — repartition vs coalesce: cuándo usar cada uno

```python
# repartition(n): shuffle completo, redistribuye uniformemente
# Costoso: hace un shuffle
# Útil: cuando el dato está desbalanceado o necesitas más particiones
df_balanceado = df.repartition(100)

# coalesce(n): combina particiones sin shuffle (cuando reduces)
# Barato: no hace shuffle (lee múltiples particiones en una task)
# Útil: cuando reduces particiones ANTES de escribir
df_reducido = df.coalesce(10)

# repartition por columna: puts rows with same key in same partition
# Útil: para joins eficientes posteriores
df_por_region = df.repartition(100, F.col("region"))
```

**Restricciones:** Para cada escenario, determina cuál usar y justifica:

1. Leer 1000 archivos pequeños de S3 (1MB cada uno), procesar, escribir como 10 archivos
2. Un DataFrame con data skew severo (una key tiene el 80% de las filas)
3. Preparar un DataFrame para un join con otro DataFrame particionado por `user_id`
4. Reducir un DataFrame de 200 particiones a 50 para escribir a Parquet
5. Después de un `filter()` que eliminó el 90% de las filas

**Pista:** Regla general: `coalesce` para reducir, `repartition` para aumentar
o para redistribuir por una key. `coalesce` no puede aumentar particiones.
Para el caso 5 (después de filtrar 90% de filas), si tienes 200 particiones
con casi nada, `coalesce(20)` reduce el overhead de tasks vacías sin hacer
un shuffle costoso.

---

### Ejercicio 24.2.3 — Particionamiento en escritura: organizar para leer

```python
# Sin particionamiento en escritura:
df.write.parquet("s3://bucket/ventas/")
# Resultado: archivos sin estructura → cada query lee TODO

# Con particionamiento en escritura:
df.write.partitionBy("año", "mes").parquet("s3://bucket/ventas/")
# Resultado: s3://bucket/ventas/año=2024/mes=01/part-00001.parquet
#            s3://bucket/ventas/año=2024/mes=02/part-00001.parquet
#            ...

# Query con partition pruning:
spark.read.parquet("s3://bucket/ventas/").filter(
    (F.col("año") == 2024) & (F.col("mes") == 1)
)
# Spark solo lee s3://bucket/ventas/año=2024/mes=01/
# No toca el resto → 11/12 de los datos no se leen
```

**Restricciones:**

1. Diseñar el esquema de particionamiento para una tabla de eventos con
   estas queries frecuentes:
   - "eventos de las últimas 24 horas"
   - "eventos del tipo 'compra' de los últimos 30 días"
   - "todos los eventos de un usuario específico"

2. ¿Por qué no usar `user_id` como partition key si hay 10 millones de usuarios?

3. Calcular cuántos archivos genera:
   ```python
   df.repartition(200).write.partitionBy("año", "mes", "dia").parquet(...)
   # 3 años × 12 meses × 31 días × 200 particiones = ???
   ```

4. ¿Qué es el "small files problem" y cómo se relaciona con el punto 3?

**Pista:** El "small files problem": demasiados archivos pequeños en S3 es
casi tan malo como pocos archivos grandes. El overhead de listar archivos,
abrir conexiones S3, y leer headers de Parquet se acumula. Una tabla con
millones de archivos de 1KB es más lenta de leer que una con miles de archivos
de 100MB. La regla: apuntar a archivos de 128MB-1GB. Para el punto 3:
3 × 12 × 31 × 200 = 223,200 archivos — claramente demasiados.

---

### Ejercicio 24.2.4 — Bucket join: joins sin shuffle

Para joins repetidos entre las mismas tablas, pre-particionar puede eliminar
el shuffle completamente:

```python
# Sin bucket: cada join hace un shuffle completo
df_ventas.join(df_usuarios, on="user_id")  # shuffle de ambos

# Con bucket: si ambas tablas están pre-particionadas por la misma key
# el join no necesita shuffle
df_ventas.write \
    .bucketBy(256, "user_id") \
    .sortBy("user_id") \
    .saveAsTable("ventas_bucketed")

df_usuarios.write \
    .bucketBy(256, "user_id") \
    .sortBy("user_id") \
    .saveAsTable("usuarios_bucketed")

# Este join NO hace shuffle (ambas tablas tienen el mismo bucketing):
spark.table("ventas_bucketed").join(
    spark.table("usuarios_bucketed"),
    on="user_id"
)
```

**Restricciones:** Implementar y verificar con `.explain()` que el join
no tiene `Exchange` (shuffle) en el plan físico. Medir la diferencia de
tiempo con y sin bucket join para un dataset de 100M filas.

**Pista:** El bucket join solo funciona si ambas tablas tienen exactamente
el mismo número de buckets y la misma key. Si una tabla tiene 256 buckets
y la otra tiene 128, Spark hace shuffle de todas formas. El bucket join
es una optimización válida para joins que se repiten muchas veces
(ej: un dashboard que hace el mismo join cada hora).

---

### Ejercicio 24.2.5 — Leer: el pipeline con demasiadas particiones

**Tipo: Diagnosticar**

Un pipeline lento muestra esto en Spark UI:

```
Stage 3: groupBy + agg
  Tasks: 200
  Duration: 45 minutos
  Input: 50 GB
  
  Distribución de duración por task:
    Min:    0.1s
    Median: 0.3s
    P75:    0.5s
    P99:    43 min  ← !!
    Max:    43 min
```

Y la distribución de datos por task:
```
    Min:     1 KB
    Median:  250 MB
    P99:     48 GB  ← !!
    Max:     48 GB
```

**Preguntas:**

1. ¿Qué tipo de problema es este? ¿Cuántos tasks tienen el problema?
2. ¿Por qué el P99 de duración es casi igual al Max?
3. ¿Cuánto tiempo tardaría el stage si este problema no existiera?
4. ¿Cómo afecta este problema al resto del pipeline?
5. Propón dos estrategias para resolverlo.

**Pista:** Con P99 de datos = 48 GB y Median = 250 MB, hay una o dos tasks
que tienen 48 GB / 250 MB ≈ 192× más datos que la mediana. Esto es data skew.
El stage tarda 43 minutos porque hay que esperar a que terminen TODAS las tasks —
incluyendo las que tienen 192× más trabajo. El tiempo "correcto" sería ~0.5s
si los datos estuvieran distribuidos uniformemente.

---

## Sección 24.3 — Shuffle: el Cuello de Botella Distribuido

El shuffle es cuando Spark necesita redistribuir datos entre particiones —
es la operación más costosa y el mayor source de problemas de rendimiento.

```
Operaciones que NO causan shuffle (narrow transformations):
  filter(), select(), map(), withColumn(), limit()
  Cada partición de output depende de exactamente una partición de input.

Operaciones que SÍ causan shuffle (wide transformations):
  groupBy(), join(), distinct(), repartition(), orderBy()
  Una partición de output puede depender de múltiples particiones de input.
  Requiere que los datos "viajen" por la red.
```

### Ejercicio 24.3.1 — Medir el costo de un shuffle

```python
import time

# Sin shuffle:
inicio = time.time()
df.filter(F.col("año") == 2024).count()
print(f"Sin shuffle: {time.time() - inicio:.1f}s")

# Con shuffle (groupBy):
inicio = time.time()
df.groupBy("region").count().collect()
print(f"Con shuffle: {time.time() - inicio:.1f}s")

# Con shuffle doble (groupBy + orderBy):
inicio = time.time()
df.groupBy("region").count().orderBy("count").collect()
print(f"Con doble shuffle: {time.time() - inicio:.1f}s")
```

**Restricciones:**
1. Medir en un dataset de 10GB con 100 particiones
2. Ver en Spark UI cuántos bytes se "shufflearon" en cada caso
3. ¿Cuánto tiempo de CPU se gastó en serialización/deserialización vs cómputo?
4. Implementar la misma aggregation con `reduceByKey` (RDD API) y comparar

**Pista:** El shuffle en Spark tiene varias fases: (1) Map: cada task escribe
sus datos en archivos locales, ordenados por la partition key del shuffle.
(2) Reduce: cada task de la siguiente etapa lee de múltiples nodes. El costo
incluye serialización, escritura a disco, lectura de red, y deserialización.
En redes de 10 Gbps, leer 1 TB a través de la red tarda ~14 minutos — solo en I/O.

---

### Ejercicio 24.3.2 — Eliminar shuffles con reordenamiento de operaciones

```python
# Versión con shuffle innecesario:
resultado = (df
    .join(df_regiones, on="region_id")    # shuffle 1
    .filter(F.col("activo") == True)       # podría estar antes del join
    .groupBy("pais")                       # shuffle 2
    .agg(F.sum("ventas"))
)

# Versión optimizada: filtrar ANTES del join reduce los datos que se shufflean
resultado = (df
    .filter(F.col("activo") == True)       # reducir primero
    .join(df_regiones, on="region_id")    # shuffle 1 (pero con menos datos)
    .groupBy("pais")                       # shuffle 2
    .agg(F.sum("ventas"))
)
```

**Restricciones:** Para el siguiente pipeline, identificar todos los shuffles
y proponer la versión con el mínimo de shuffles necesarios:

```python
df_resultado = (
    spark.read.parquet("s3://ventas/")
    .join(spark.read.parquet("s3://usuarios/"), on="user_id")
    .join(spark.read.parquet("s3://productos/"), on="producto_id")
    .filter(F.col("activo") == True)
    .filter(F.col("categoria") == "electronico")
    .groupBy("region", "mes")
    .agg(F.sum("monto"), F.count("*"))
    .filter(F.col("sum(monto)") > 100000)
    .orderBy(F.col("sum(monto)").desc())
)
```

**Pista:** Las reglas de optimización manual: (1) filtrar lo más temprano posible,
(2) broadcast el DataFrame más pequeño, (3) el `orderBy` final agrega un shuffle —
si solo necesitas el top N, `df.orderBy(...).limit(N)` permite a Spark optimizar
(no necesita ordenar todos los datos, solo encontrar los top N).

---

### Ejercicio 24.3.3 — spark.sql.shuffle.partitions: el parámetro más importante

```python
# Default: 200 particiones después de un shuffle
spark.conf.get("spark.sql.shuffle.partitions")  # "200"

# Con un dataset pequeño (1 GB), 200 particiones es demasiado:
# → 200 tasks de ~5 MB cada una
# → overhead de scheduling supera el tiempo de procesamiento

# Con un dataset grande (10 TB), 200 particiones es muy poco:
# → cada task procesa 50 GB
# → likely OOM en executors

# Ajustar según el tamaño del dato:
spark.conf.set("spark.sql.shuffle.partitions", "2000")

# Mejor: Adaptive Query Execution (Spark 3.0+) ajusta automáticamente:
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

**Restricciones:** Experimentar con un dataset de 100 GB:
1. `shuffle.partitions = 200`: medir tiempo y tamaño por partición
2. `shuffle.partitions = 2000`: medir tiempo y tamaño por partición
3. `adaptive.enabled = true`: medir tiempo y ver cuántas particiones elige AQE
4. ¿Cómo elige AQE el número de particiones?

**Pista:** AQE mide el tamaño real de los datos después de cada stage y ajusta
el número de particiones del siguiente shuffle. Si el shuffle produce 100 GB
y la target de 64 MB por partición está configurada, AQE elige
100 GB / 64 MB ≈ 1600 particiones automáticamente. Sin AQE, usarías el default
de 200 y cada partición tendría 500 MB — probablemente lento pero manejable.

---

### Ejercicio 24.3.4 — El join que genera el shuffle más grande

```python
# Sort-Merge Join (el default para tablas grandes):
# Ambas tablas se shufflean y ordenan por la join key
# Necesario cuando ninguna tabla es pequeña para broadcast
df_ventas.join(df_clicks, on="session_id")

# vs Broadcast Hash Join (una tabla pequeña, sin shuffle):
df_ventas.join(broadcast(df_productos), on="producto_id")

# vs Bucket Join (pre-particionado, sin shuffle en join):
# (ver Ejercicio 24.2.4)

# El peor caso: cross join (producto cartesiano)
# N × M filas → solo si es intencional
df_a.crossJoin(df_b)  # requiere .config("spark.sql.crossJoin.enabled", "true")
```

**Restricciones:** Para cada combinación de tamaños, determina el tipo de join
óptimo y el shuffle generado:

| Tabla izquierda | Tabla derecha | Join óptimo |
|---|---|---|
| 1 TB | 1 MB | ??? |
| 1 TB | 10 GB | ??? |
| 1 TB | 1 TB (misma partition key) | ??? |
| 1 TB | 1 TB (distinta partition key) | ??? |

**Pista:** Para 1 TB + 10 GB: si la memoria de los executors lo permite,
broadcast del lado de 10 GB. Si no, sort-merge join con shuffle de ambos.
La regla de los 10 GB es orientativa — depende del cluster. Ver el plan
con `.explain()` para confirmar qué tipo de join eligió Spark.

---

### Ejercicio 24.3.5 — Leer: el shuffle que no debería existir

**Tipo: Diagnosticar**

Un pipeline de ETL tiene este plan (simplificado):

```
== Physical Plan ==
Sort [fecha ASC]
+- Exchange rangepartitioning(fecha, 200)          ← shuffle 3
   +- HashAggregate [region, fecha]
      +- Exchange hashpartitioning(region, fecha, 200)  ← shuffle 2
         +- Project [region, fecha, monto]
            +- BroadcastHashJoin [producto_id]
               :- Filter [activo = true]
               :  +- FileScan parquet ventas
               +- BroadcastExchange                  ← broadcast (no shuffle)
                  +- FileScan parquet productos
```

El data engineer que escribió este pipeline añadió al final:

```python
.orderBy("fecha")
```

porque quería los resultados ordenados para escribirlos a un archivo CSV legible.

**Preguntas:**

1. ¿El `orderBy("fecha")` al final era necesario para el resultado correcto?
2. ¿Cuánto cuesta el shuffle 3 (rangepartitioning) en comparación con los otros?
3. Si el resultado se escribe a Parquet (no CSV), ¿sigue siendo necesario el ordenamiento?
4. ¿Hay algún escenario donde el `orderBy` final sí agrega valor en un pipeline Spark?
5. ¿Cómo el `orderBy` local dentro de una partición (sin shuffle) podría conseguir
   un resultado parcialmente ordenado sin el costo del shuffle?

**Pista:** `orderBy` en Spark garantiza orden global — requiere reunir todos los datos
en un solo sort distribuido (shuffle de rangepartitioning). Para un archivo Parquet
que luego se lee con Spark, el orden global no importa porque Spark leerá en paralelo
de todas formas. Si necesitas orden dentro de cada partición (para row groups de Parquet
más eficientes), usa `sortWithinPartitions("fecha")` — no hace shuffle global.

---

## Sección 24.4 — Data Skew: Cuando el Paralelismo Falla

El data skew es el problema más frecuente de rendimiento en Spark.
Una partición con significativamente más datos que las demás convierte
el paralelismo en secuencialidad.

```
Grado de paralelismo teórico: 100 tasks en 100 cores = 100× speedup
Grado de paralelismo real con skew:
  99 tasks: 1 segundo cada una
  1 task:   100 segundos (tiene 100× más datos)
  
  Tiempo total: 100 segundos (igual que secuencial)
  Los 99 cores que terminaron antes están OCIOSOS esperando a la task lenta
```

### Ejercicio 24.4.1 — Detectar data skew

```python
# Método 1: ver la distribución del join key
df.groupBy("user_id").count().orderBy(F.col("count").desc()).show(20)
# Si el top user tiene 1M filas y la mediana es 10 filas → skew de 100,000×

# Método 2: ver la distribución de datos por partición DESPUÉS del shuffle
df_agrupado = df.groupBy("user_id").agg(F.sum("monto"))
df_agrupado.withColumn(
    "partition_id", F.spark_partition_id()
).groupBy("partition_id").agg(
    F.count("*").alias("filas"),
    F.sum("monto").alias("monto_total")
).orderBy(F.col("monto_total").desc()).show()

# Método 3: Spark UI → Stage → Tasks → ver la columna "Duration"
# Si la duración de las tasks tiene una distribución muy amplia → skew
```

**Restricciones:** Dado un dataset de transacciones donde hay:
- Un bot que generó 5 millones de transacciones con el mismo `session_id`
- El session_id mediano tiene 15 transacciones

1. Detectar el skew con los tres métodos
2. Cuantificar: ¿cuánto más lento hace esto el job?
3. ¿Aparece el skew en todas las operaciones o solo en algunas?

**Pista:** El skew aparece en operaciones que agrupan por `session_id` (groupBy, join).
Un `filter()` o `select()` no se ve afectado por skew — no agrupan datos.
El job es tan lento como la task más lenta. Si la task del bot tarda 30 minutos
y las demás tardan 30 segundos, el stage completo tarda 30 minutos independientemente
de cuántos cores tengas.

---

### Ejercicio 24.4.2 — Salting: distribuir el skew artificialmente

```python
# El problema: todos los registros del bot tienen session_id = "bot-123"
# Todos van a la misma partición → una task tiene 5M filas, las demás tienen 15

# Salting: añadir un sufijo aleatorio para distribuir las filas del bot
import random

SALT_FACTOR = 50  # distribuir en 50 particiones

# En el DataFrame con skew, añadir un salt:
df_con_salt = df.withColumn(
    "session_id_salted",
    F.concat(
        F.col("session_id"),
        F.lit("_"),
        (F.rand() * SALT_FACTOR).cast("int").cast("string")
    )
)

# Ahora agrupar por session_id_salted en lugar de session_id:
df_parcial = (df_con_salt
    .groupBy("session_id_salted")
    .agg(F.sum("monto").alias("monto_parcial"),
         F.count("*").alias("count_parcial"))
)

# Luego extraer el session_id original y reagrupar:
df_final = (df_parcial
    .withColumn("session_id", F.expr("split(session_id_salted, '_')[0]"))
    .groupBy("session_id")
    .agg(F.sum("monto_parcial").alias("monto_total"),
         F.sum("count_parcial").alias("count_total"))
)
```

**Restricciones:**
1. Implementar el salting para el dataset del Ejercicio 24.4.1
2. Verificar que el resultado es correcto (mismo total que sin salting)
3. Medir la mejora de tiempo con SALT_FACTOR = 10, 50, 100
4. ¿Cuál es el costo del salting? ¿Cuándo es demasiado?

**Pista:** El costo del salting: un segundo groupBy después del primero.
Si el skew es leve (2-3× peor que la mediana), el overhead del segundo groupBy
puede superar el beneficio. El salting vale la pena cuando el skew es severo (10×+).
El SALT_FACTOR óptimo ≈ el factor de skew: si una key tiene 100× más filas,
un SALT_FACTOR de 100 la distribuye uniformemente.

---

### Ejercicio 24.4.3 — Skew join hint: la solución automática de Spark 3.x

```python
# Spark 3.0+ puede detectar y manejar skew en joins automáticamente
# con Adaptive Query Execution:

spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256mb")

# Cuando AQE detecta una partición que es >5× la mediana Y >256MB,
# la divide automáticamente y la procesa en múltiples tasks

resultado = df_ventas.join(df_usuarios, on="user_id")
# Spark UI mostrará: "Splitting skewed partition 42 into 8 sub-partitions"
```

**Restricciones:**
1. Comparar: salting manual vs AQE skew join para el mismo dataset
2. ¿En qué casos AQE maneja el skew mejor que el salting manual?
3. ¿En qué casos el salting manual sigue siendo necesario?
4. Ver en Spark UI la evidencia de que AQE dividió particiones skewed

**Pista:** AQE skew join funciona bien cuando el skew es en el join. Para skew
en groupBy (aggregations), AQE también ayuda pero puede requerir configuración
adicional. El salting manual sigue siendo necesario para workloads complejos
donde AQE no tiene suficiente visibilidad (ej: RDD API, o pipelines con
múltiples stages donde el skew se acumula progresivamente).

---

### Ejercicio 24.4.4 — Skew en datos temporales: el hot spot de "ahora"

Un tipo especial de skew frecuente en data engineering: los datos recientes
tienen más tráfico que los datos históricos.

```python
# Tabla de eventos particionada por hora:
# hora=2024-01-15-14: 50 GB (hora actual, mucho tráfico)
# hora=2024-01-15-13: 5 GB (hora anterior)
# hora=2024-01-15-12: 5 GB
# ...

# Un join con esta tabla siempre tiene skew en la hora actual
df_eventos_recientes = (
    df_eventos
    .filter(F.col("hora") >= "2024-01-15-12")
    .join(df_usuarios, on="user_id")
)
# La partición de la hora 14 tiene 10× más datos → skew
```

**Restricciones:** Diseñar una estrategia de particionamiento que reduce
el skew temporal sin perder la capacidad de hacer partition pruning por hora.

**Pista:** Una estrategia: particionar por (hora, hash(user_id) % 10).
Los datos de la hora actual se distribuyen en 10 sub-particiones.
Las queries por hora siguen funcionando (leen las 10 sub-particiones de esa hora).
La query "todos los eventos de la hora 14" sigue con partition pruning pero
ya no hay una sola partición gigante — son 10 particiones de tamaño más uniforme.

---

### Ejercicio 24.4.5 — Leer: el skew que se esconde

**Tipo: Diagnosticar**

Un job procesa logs de acceso. El data engineer dice "no hay skew — verifiqué
la distribución de user_id y es uniforme". Sin embargo, el job es 10× más lento
de lo esperado.

```python
# El pipeline:
df_logs = spark.read.parquet("s3://logs/")
df_resultado = (df_logs
    .withColumn("dominio", F.regexp_extract("url", r"https?://([^/]+)", 1))
    .groupBy("dominio")
    .agg(F.count("*").alias("hits"))
    .orderBy(F.col("hits").desc())
)
```

Distribución de la columna `dominio`:
```
google.com:        450,000,000 hits  (45% del total)
youtube.com:       200,000,000 hits  (20%)
facebook.com:       80,000,000 hits   (8%)
otros (10M dominios):  270,000,000 hits (27%)
```

**Preguntas:**

1. ¿Por qué el data engineer no detectó el skew al verificar user_id?
2. ¿Dónde ocurre el skew real?
3. ¿Cuántas tasks sufren el skew en un groupBy con 200 particiones shuffle?
4. Propón la corrección.
5. ¿Cómo habrías encontrado este skew sin saber de antemano que el dominio era el problema?

**Pista:** El skew está en la columna `dominio`, no en `user_id`. El data engineer
verificó la columna equivocada. Para encontrar el skew sin saber la causa:
ver en Spark UI qué task es la más lenta, luego añadir instrumentación para saber
qué valores de `dominio` va procesando esa task. El skew en una columna que no
es la join/group key del paso anterior puede "aparecer" en pasos downstream.

---

## Sección 24.5 — UDFs: Cuándo Python Entra al Executor

### Ejercicio 24.5.1 — El costo de una UDF escalar

```python
import pandas as pd
from pyspark.sql.types import StringType
from pyspark.sql.functions import udf, pandas_udf

# UDF escalar (Python puro, row-by-row):
@udf(returnType=StringType())
def limpiar_texto_udf(texto):
    if texto is None:
        return None
    return texto.strip().lower().replace("  ", " ")

# Función nativa de Spark (C++/JVM, vectorizada):
def limpiar_texto_nativo(col):
    return F.trim(F.lower(F.regexp_replace(col, r"\s+", " ")))

# Pandas UDF (vectorizada con Arrow):
@pandas_udf(StringType())
def limpiar_texto_pandas_udf(serie: pd.Series) -> pd.Series:
    return serie.str.strip().str.lower().str.replace(r"\s+", " ", regex=True)
```

**Restricciones:** Medir el tiempo de procesamiento de 100M filas con cada approach:
1. UDF escalar Python
2. Función nativa Spark
3. Pandas UDF

Verificar en Spark UI el tiempo de "Python evaluation time" para la UDF escalar.

**Pista:** La UDF escalar en Python requiere:
para cada fila → serializar a Python → cruzar la frontera JVM/Python →
ejecutar la función → serializar de vuelta a JVM → continuar.
Con 100M filas, esto es 100M cruces de frontera JVM/Python.
La función nativa de Spark: cero cruces de frontera — todo ocurre en JVM/C++.
La Pandas UDF: un cruce por batch (ej: 10,000 filas por batch) → 10,000× mejor.

---

### Ejercicio 24.5.2 — Pandas UDFs: el tipo correcto para cada caso

```python
# Tipo 1: Scalar Pandas UDF (una Serie → una Serie)
@pandas_udf(StringType())
def normalizar_email(emails: pd.Series) -> pd.Series:
    return emails.str.lower().str.strip()

# Tipo 2: Grouped Map Pandas UDF (un grupo → un DataFrame)
# Útil para aplicar lógica compleja por grupo
from pyspark.sql.functions import PandasUDFType

schema = "user_id string, fecha date, zscore double"

@pandas_udf(schema, PandasUDFType.GROUPED_MAP)
def calcular_zscore(df: pd.DataFrame) -> pd.DataFrame:
    df["zscore"] = (df["monto"] - df["monto"].mean()) / df["monto"].std()
    return df[["user_id", "fecha", "zscore"]]

resultado = df.groupBy("user_id").apply(calcular_zscore)

# Tipo 3: Scalar Iterator Pandas UDF (genera datos)
# Útil para UDFs que mantienen estado entre batches
from typing import Iterator

@pandas_udf(StringType())
def procesar_con_modelo(batch_iter: Iterator[pd.Series]) -> Iterator[pd.Series]:
    modelo = cargar_modelo()  # cargado una vez por task, no por fila
    for batch in batch_iter:
        yield modelo.predict(batch)
```

**Restricciones:** Para cada caso de uso, determinar cuál tipo de Pandas UDF usar:
1. Normalizar nombres de columnas (trim, lowercase)
2. Calcular rolling average de 7 días por usuario
3. Aplicar un modelo de ML a cada fila
4. Generar recomendaciones personalizadas que dependen del historial completo del usuario

**Pista:** El Tipo 3 (Scalar Iterator) es ideal para modelos de ML — el modelo
se carga una vez por executor (en el `__init__` del iterador) en lugar de una vez
por fila o una vez por batch. Para un modelo de 1GB, la diferencia es significativa:
cargar 1M filas con modelo de 1GB → cargar el modelo ~1M veces vs ~1 vez.

---

### Ejercicio 24.5.3 — Reemplazar UDFs con funciones nativas

El mejor approach: no usar UDFs. Casi siempre hay una función nativa equivalente.

```python
# UDF innecesaria 1: lógica condicional
@udf(StringType())
def categoria_edad(edad):
    if edad < 18: return "menor"
    elif edad < 65: return "adulto"
    else: return "senior"

# Reemplazo nativo:
F.when(F.col("edad") < 18, "menor") \
 .when(F.col("edad") < 65, "adulto") \
 .otherwise("senior")

# UDF innecesaria 2: parsing de strings
@udf(StringType())
def extraer_dominio(url):
    from urllib.parse import urlparse
    return urlparse(url).netloc

# Reemplazo nativo (regex):
F.regexp_extract(F.col("url"), r"https?://([^/]+)", 1)

# UDF innecesaria 3: cálculo matemático
@udf("double")
def distancia_haversine(lat1, lon1, lat2, lon2):
    # ... implementación en Python
    pass

# Reemplazo nativo (usando funciones de Spark):
# No hay haversine directo, pero puede implementarse con funciones nativas
# sin cruzar la frontera Python/JVM:
F.acos(
    F.sin(F.radians(F.col("lat1"))) * F.sin(F.radians(F.col("lat2"))) +
    F.cos(F.radians(F.col("lat1"))) * F.cos(F.radians(F.col("lat2"))) *
    F.cos(F.radians(F.col("lon2") - F.col("lon1")))
) * F.lit(6371)
```

**Restricciones:** Para cada una de estas UDFs, encontrar el reemplazo nativo:
1. `udf` que hace `json.loads()` y extrae un campo
2. `udf` que aplica una transformación de fecha (formato custom)
3. `udf` que calcula el percentil de un valor dentro de su grupo
4. `udf` que hace window functions custom

**Pista:** `F.from_json()` para JSON. `F.to_date()` con format string para fechas.
`F.percent_rank()` o `F.ntile()` para percentiles dentro de grupos.
Para window functions custom que genuinamente no se pueden expresar con las
funciones nativas — ahí sí es razonable usar una Pandas UDF de tipo GROUPED_MAP.

---

### Ejercicio 24.5.4 — Serialización: Kryo vs Java vs Arrow

```python
# Spark usa Java serialization por defecto (lenta, voluminosa)
# Kryo es más rápido y produce objetos más pequeños
# Arrow es la más eficiente para operaciones columnar (Pandas UDFs)

# Configurar Kryo:
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
spark.conf.set("spark.kryo.registrationRequired", "false")

# Para tipos custom, registrarlos explícitamente (mejor performance):
# spark.conf.set("spark.kryo.classesToRegister", "com.empresa.MiClase")
```

**Restricciones:** Comparar el tamaño de datos serializado y el tiempo de
deserialización para un DataFrame de 1M filas con cada método:
1. Java serialization (default)
2. Kryo serialization
3. Arrow (Pandas UDF)

¿Cuándo importa la elección de serializer?

**Pista:** La serialización importa principalmente durante shuffles — cuando los datos
viajan por la red entre executors. Para operaciones sin shuffle (filter, select),
la serialización es irrelevante. Con Kryo, los shuffles pueden ser 2-3× más rápidos
para tipos Java complejos. Para DataFrames (formato columnar interno de Spark),
el impacto es menor porque ya están en formato eficiente.

---

### Ejercicio 24.5.5 — Leer: el pipeline que parece Python pero no lo es

**Tipo: Diagnosticar**

Un pipeline tiene estas métricas en Spark UI:

```
Stage 2: apply UDFs
  Duration: 47 minutos
  Tasks: 200
  CPU Time: 8 minutos  ← !! muy poco CPU relativo al tiempo total
  GC Time: 2 minutos
  Python Evaluation Time: 42 minutos  ← !! el 89% del tiempo es Python
  Shuffle Read: 0 bytes (no es un shuffle)
  Input: 50 GB
```

El código del stage:
```python
df_resultado = (df
    .withColumn("categoria", clasificar_udf(F.col("descripcion")))  # NLP model
    .withColumn("sentimiento", sentimiento_udf(F.col("texto")))      # sentiment
    .withColumn("entidades", extraer_entidades_udf(F.col("texto")))  # NER model
)
```

Cada UDF carga un modelo de ML de 500MB.

**Preguntas:**

1. ¿Por qué el CPU Time es 8 minutos si el total es 47?
2. ¿Cuántas veces se carga cada modelo de 500MB?
3. ¿Cómo el Python Evaluation Time de 42 minutos se relaciona con la GIL?
4. Propón tres optimizaciones en orden de impacto.
5. ¿Cuánto tiempo esperarías después de la optimización?

**Pista:** Con 200 tasks y 3 UDFs, cada modelo se carga 200× si se carga
dentro de la UDF. Con Scalar Iterator UDF, cada modelo se carga 1× por task
(200× total pero en paralelo). Los 42 minutos de Python Evaluation incluyen:
carga del modelo + inferencia. Si la carga es 2 minutos por modelo × 3 modelos × 200 tasks
= 1,200 minutos de CPU desperdiciado en cargas redundantes.

---

## Sección 24.6 — Memoria y Spill: Cuando los Datos No Caben

### Ejercicio 24.6.1 — La arquitectura de memoria de Spark

```
Memoria total del executor:
├── Reserved Memory (300 MB fijo): overhead interno de Spark
├── User Memory (40% del resto): objetos creados por el usuario, RDDs cacheados
└── Spark Memory (60% del resto):
    ├── Storage Memory: caché de DataFrames (.cache(), .persist())
    └── Execution Memory: shuffles, sorts, joins, aggregations
        (Storage y Execution comparten este 60% — uno toma del otro si necesita)

Configuración típica (executor de 4 GB):
  spark.executor.memory = 4g
  Reserved = 300 MB
  Usable = 3.7 GB
  User Memory = 1.48 GB (40%)
  Spark Memory = 2.22 GB (60%)
```

**Preguntas:**

1. Un executor de 4 GB con un join de dos DataFrames de 3 GB cada uno:
   ¿cabe en memoria? ¿Qué pasa si no cabe?
2. ¿Qué es el "spill to disk" y cuándo ocurre?
3. ¿Cómo distinguir un job lento por spill vs un job lento por skew en Spark UI?
4. ¿Cómo ajustar la configuración para reducir el spill sin cambiar el hardware?

**Pista:** El spill to disk es cuando Spark escribe datos temporales al disco local
del executor porque no caben en la ejecución memory. Es mucho más lento que memoria
(10-100× para SSDs, 1000× para HDDs). En Spark UI: un stage con "Spill (memory)"
y "Spill (disk)" en columnas separadas. Si el spill es grande, el remedio típico
es aumentar el número de particiones (menos datos por partition → menos memoria por task).

---

### Ejercicio 24.6.2 — Detectar y diagnosticar spill

```python
# Habilitar métricas de spill:
spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")

# Trigger artificial de spill para observarlo:
df_grande = spark.range(0, 100_000_000).toDF("id")
df_resultado = (df_grande
    .withColumn("datos", F.repeat(F.lit("x"), 100))  # inflar el tamaño
    .groupBy((F.col("id") % 1000).alias("bucket"))
    .agg(F.collect_list("datos").alias("lista"))  # ← fuerza materialización en memoria
)
df_resultado.count()
```

**Restricciones:**
1. Ejecutar con executor memory = 1g y verificar el spill en Spark UI
2. Ejecutar con executor memory = 4g y verificar que el spill desaparece
3. Ejecutar con executor memory = 1g pero con 10× más particiones:
   ¿el spill desaparece?
4. ¿Hay operaciones en Spark que NO pueden hacer spill? ¿Cuáles?

**Pista:** `collect_list()` y `collect_set()` no pueden hacer spill — necesitan
materializar toda la lista en memoria. Si la lista de un grupo es más grande
que la execution memory, el job falla con OOM, no hace spill. Por eso son
las operaciones más peligrosas en Spark para datasets con skew: el grupo grande
puede generar una lista que no cabe en ningún executor.

---

### Ejercicio 24.6.3 — OOM: cuando el spill no es suficiente

```python
# Operaciones que pueden causar OOM (no hacen spill):
# 1. collect() en el driver:
df_enorme.collect()  # trae TODO al driver Python → OOM en driver

# 2. collect_list() o collect_set() con grupos muy grandes:
df.groupBy("user_id").agg(F.collect_list("evento"))
# Si un user_id tiene 10M eventos, la lista es 10M × tamaño_evento en memoria

# 3. Broadcast de un DataFrame demasiado grande:
df_ventas.join(broadcast(df_enorme), on="id")
# Si df_enorme > executor memory → OOM al recibir el broadcast

# 4. Caché de más datos de los que cabe:
df.persist(StorageLevel.MEMORY_ONLY)  # si no cabe, Spark descarta particiones
# vs
df.persist(StorageLevel.MEMORY_AND_DISK)  # spill a disco si no cabe
```

**Restricciones:** Para cada OOM, implementar el fix correcto:
1. `collect()` en el driver de un DataFrame de 10GB
2. `collect_list()` con grupos de hasta 1M elementos
3. Broadcast de un DataFrame de 5GB en executors de 4GB
4. Caché de un DataFrame de 100GB en un cluster con 50GB de memoria total

**Pista:** Para `collect_list()` con grupos enormes: si necesitas agrupar
y hacer cálculos sobre los elementos del grupo, usa window functions en lugar
de `collect_list()`. Window functions procesan los datos en su lugar sin
materializar la lista completa. Si genuinamente necesitas la lista: asegúrate
de que ningún grupo supera los ~500MB (configurable) o el job fallará.

---

### Ejercicio 24.6.4 — Configurar la memoria del cluster correctamente

Para un cluster con estas características:
```
Nodos: 10 máquinas
  RAM por nodo: 64 GB
  Cores por nodo: 16
  
Workload: ETL batch con shuffles grandes, sin ML models
```

**Restricciones:** Calcular la configuración óptima:
- `spark.executor.memory`: ¿cuánta RAM por executor?
- `spark.executor.cores`: ¿cuántos cores por executor?
- `spark.executor.memoryOverhead`: ¿cuánto overhead off-heap?
- Número total de executors

Justificar por qué 1 executor por nodo (con todos los cores) es diferente
a 4 executors por nodo (con 4 cores cada uno).

**Pista:** La configuración estándar para 64GB RAM, 16 cores:
- 1 core y ~5GB para el OS y YARN/Kubernetes overhead
- Disponible: 15 cores, ~59 GB RAM
- Executors de 5 cores y ~18 GB RAM → 3 executors por nodo
- `spark.executor.cores = 5` (4-5 es el sweet spot para paralelismo dentro del executor)
- `spark.executor.memory = 15g` (deja 3GB para memoryOverhead)
- `spark.executor.memoryOverhead = 3g`
- Total: 10 nodos × 3 executors = 30 executors

Con 1 executor por nodo (16 cores, 59 GB): los 16 cores compiten por la misma
execution memory → más spill. Con demasiados executors pequeños: overhead de
scheduling supera el beneficio del paralelismo.

---

### Ejercicio 24.6.5 — Leer: el job que se quedó sin memoria

**Tipo: Diagnosticar**

Un job falló con este error:

```
ERROR SparkContext: Error initializing SparkContext.
java.lang.OutOfMemoryError: GC overhead limit exceeded
  at org.apache.spark.sql.execution.aggregate.HashAggregateExec...

Stage 5, Task 73 failed:
  ExecutorLostFailure (executor 12 exited caused by one of the running tasks)
  Reason: Remote RPC client disassociated. Likely due to containers exceeding
  thresholds, or network issues. Check driver logs for WARN RemoteActorRefProvider
```

Las métricas antes del crash:
```
Executor 12:
  Memory Used: 18.2 GB / 18 GB  ← !! (sobre el límite)
  GC Time: 45% del tiempo total
  Task siendo ejecutada: groupBy("user_id").agg(collect_set("productos"))
  user_id en proceso: "power_user_42" (150,000 productos únicos)
```

**Preguntas:**

1. ¿Qué causó el OOM?
2. ¿Por qué el GC time al 45% es una señal de advertencia temprana?
3. ¿Cómo podría haberse predicho este fallo antes de correr el job?
4. Propón el fix para que el job no falle.
5. ¿Cómo verificar que el fix funcionó sin esperar a que el job falle?

**Pista:** `collect_set("productos")` para un usuario con 150,000 productos
significa materializar un set de 150,000 strings en memoria de un solo task.
Si cada string tiene 50 bytes promedio, son 7.5 MB — manejable. Pero si son
UUIDs (36 bytes) con metadata adicional, puede ser mucho más. El GC al 45%
indica que la JVM está constantemente intentando liberar memoria — casi OOM.
Para predecir: antes de hacer `collect_set`, verificar la distribución con
`.groupBy("user_id").agg(F.countDistinct("productos")).orderBy(F.desc("count(DISTINCT productos)")).show(5)`.

---

## Sección 24.7 — Spark UI: Diagnosticar en Producción

La Spark UI es la herramienta principal de diagnóstico. Saber leerla es
equivalente a saber leer un goroutine dump o un heap profile.

### Ejercicio 24.7.1 — Leer la pestaña Jobs y Stages

**Tipo: Leer/diagnosticar**

```
Jobs tab:
  Job 0: Duration 2h 15m  Status: Succeeded
    ├── Stage 0: [==============================] 1 min (input: 5 TB)
    ├── Stage 1: [==============================] 2 min (shuffle: 200 GB)
    ├── Stage 2: [======] 8 min  (shuffle: 50 GB)
    └── Stage 3: [========================================] 2h 4m  ← !!
          Tasks: 200
          Failed Tasks: 0
          Skipped Tasks: 0
```

Stage 3 details:
```
  Summary Metrics for 200 Completed Tasks:
  
  Metric          Min     25th    Median  75th    Max
  Duration        0.1s    0.2s    0.4s    0.5s    2h 3m   ← !!
  GC Time         0s      0s      0.1s    0.1s    45min
  Input Size      0 B     0 B     0 B     0 B     0 B
  Shuffle Read    1 KB    1.2 MB  1.4 MB  1.6 MB  98 GB   ← !!
  Output          0 B     0 B     0 B     0 B     0 B
```

**Preguntas:**

1. ¿Qué tipo de problema tiene el Stage 3?
2. ¿Cuántos tasks tienen el problema?
3. ¿El Stage 3 es el cuello de botella del job? ¿O hay otros?
4. ¿Qué operación probablemente generó el Stage 3?
5. ¿Cuánto tiempo tardaría el job si el Stage 3 estuviera optimizado?

---

### Ejercicio 24.7.2 — Leer la pestaña SQL y el DAG

**Tipo: Leer/diagnosticar**

El DAG de Spark UI para un pipeline muestra:

```
SQL Query:
  [Scan parquet] → [Filter] → [Exchange] → [HashAggregate]
                                    ↓
  [Scan parquet] → [BroadcastExchange] →  [BroadcastHashJoin] → [Exchange] → [Sort] → [collect()]
```

Los tiempos de cada nodo:
```
Scan parquet (ventas): 3 min, 1.2 TB scanned, 400 GB output (despues del filter)
Filter: 0.1 min
BroadcastExchange (productos): 0.2 min, 500 MB
BroadcastHashJoin: 2 min
Exchange (shuffle para sort): 8 min ← !!
Sort: 15 min ← !!
collect(): 0.1 min
```

**Preguntas:**

1. ¿Por qué el `Exchange (shuffle para sort)` tardó 8 minutos?
2. ¿Por qué el `Sort` tardó 15 minutos?
3. ¿El `collect()` de 0.1 min es consistente con el sort anterior?
4. Si el objetivo era solo obtener los top 100 registros, ¿qué podría optimizarse?
5. ¿El BroadcastHashJoin fue la decisión correcta para 500 MB?

---

### Ejercicio 24.7.3 — Leer la pestaña Storage: entender el caché

**Tipo: Leer/diagnosticar**

```
Storage tab:
  RDD Name                          Storage Level    Cached Partitions    Fraction Cached    Size in Memory    Size on Disk
  [Stage 3, RDD 42]                 MEMORY_AND_DISK  180/200              90%                45 GB             12 GB
```

**Preguntas:**

1. ¿Por qué el 10% de las particiones (20/200) no está en memoria?
2. ¿Qué significa que 12 GB están en disco?
3. ¿Cómo afecta esto a las queries que leen de este caché?
4. ¿Cuándo es preferible `DISK_ONLY` sobre `MEMORY_AND_DISK`?
5. Si el job vuelve a leer las 20 particiones en disco, ¿Spark las mueve a memoria?

**Pista:** El 10% en disco indica que la memoria de los executors no fue suficiente
para guardar todas las particiones. Spark hace evicción LRU: cuando llega una
partición nueva que no cabe, saca la menos recientemente usada al disco.
Las lecturas del disco son lentas pero corrrectas — el caché no falla, solo degrada.
Si el mismo caché se lee repetidamente, las particiones en disco pueden terminar en memoria
(LRU hace que las "frías" sean desalojadas por las "calientes").

---

### Ejercicio 24.7.4 — Leer la pestaña Executors: diagnóstico de recursos

**Tipo: Leer/diagnosticar**

```
Executors tab:
  ID   Address         State   RDD Blocks  Storage Memory   Disk Used   Cores   Active Tasks   Failed Tasks   Completed Tasks   GC Time   Input    Shuffle Read   Shuffle Write
  0    worker-1:PORT   Active  45          12 GB / 18 GB    2 GB        5       5              0              1,247             8%        120 GB   45 GB          32 GB
  1    worker-2:PORT   Active  45          11 GB / 18 GB    0 GB        5       5              0              1,251             7%        118 GB   44 GB          31 GB
  2    worker-3:PORT   Active  45          17 GB / 18 GB    8 GB        5       2              12             847               52%       95 GB    38 GB          28 GB  ← !!
  3    worker-4:PORT   Active  45          16 GB / 18 GB    5 GB        5       3              8              912               41%       101 GB   40 GB          30 GB  ← !!
```

**Preguntas:**

1. ¿Qué está pasando con los executors 2 y 3?
2. ¿Por qué su GC Time es 5-7× mayor que los otros?
3. ¿Qué tienen en común los executors 2 y 3 (hint: mira Disk Used)?
4. ¿Por qué tienen más Failed Tasks?
5. ¿Qué acción inmediata tomarías?

**Pista:** GC Time al 41-52% es señal de que la JVM está constantemente luchando
por memoria — near-OOM. Los 8 GB y 5 GB en disco indican spill significativo.
Los failed tasks probablemente fallaron por OOM antes de spill. Los executors 2 y 3
pueden estar en nodos con menos RAM disponible (otros procesos corriendo),
o pueden ser los que recibieron las particiones con skew (más datos → más spill → más GC).

---

### Ejercicio 24.7.5 — Construir un checklist de diagnóstico

**Tipo: Construir/proceso**

Construye el checklist de diagnóstico de performance para Spark,
basado en los problemas que encontraste en este capítulo:

```markdown
# Checklist de Diagnóstico de Performance — Spark

## Cuando un job es más lento de lo esperado

### Paso 1: Spark UI → Jobs
- [ ] ¿Qué stage es el más lento?
- [ ] ¿El stage más lento tiene tasks con duración muy variable? → Data skew
- [ ] ¿Hay stages que dicen "Skipped"? → Caché funcionando correctamente

### Paso 2: Spark UI → Stage lento → Tasks
- [ ] Distribución de Shuffle Read: ¿hay outliers? → Skew en join/groupBy
- [ ] GC Time > 10% del tiempo total → Presión de memoria
- [ ] Python Evaluation Time alto → UDFs escalares lentas

### Paso 3: ...
[completar el checklist]
```

**Restricciones:** El checklist debe cubrir:
- Al menos 5 síntomas con su causa más probable
- El orden de diagnóstico (de lo más frecuente a lo menos)
- La acción correctiva para cada síntoma
- Cuándo escalar (el problema requiere cambio de arquitectura, no de configuración)

---

## Resumen del capítulo

**Los cinco problemas más comunes de performance en Spark y cómo detectarlos:**

```
1. Data skew
   Síntoma: una task tarda 100× más que las demás (Spark UI → Stage → Tasks)
   Causa: distribución desigual de datos por la join/group key
   Fix: salting, AQE skew join, repartition

2. Shuffle excesivo
   Síntoma: stages con Exchange en el plan (explain()), tiempo en Exchange >> cómputo
   Causa: joins sin broadcast, groupBy con cardinalidad alta, orderBy innecesario
   Fix: broadcast, bucket join, eliminar orderBy si no es necesario

3. UDFs Python lentas
   Síntoma: Python Evaluation Time alto en Spark UI (Stage → Tasks metrics)
   Causa: UDFs escalares row-by-row vs funciones nativas o Pandas UDFs
   Fix: reemplazar con funciones nativas, o Scalar Iterator Pandas UDF para modelos ML

4. Spill to disk
   Síntoma: "Spill (disk)" en Stage metrics, GC Time > 15%
   Causa: execution memory insuficiente para el tamaño de los datos
   Fix: aumentar particiones, aumentar executor memory, evitar collect_list/collect_set

5. Reads innecesarios (DAG re-ejecutado)
   Síntoma: mismo scan parquet aparece múltiples veces en el DAG (SQL tab)
   Causa: DataFrame sin .cache() accedido por múltiples acciones
   Fix: .cache() o .persist() antes de la primera acción
```

**La regla que engloba todo:**

> En Spark, el trabajo del data engineer es describir QUÉ datos procesar
> (filtros, particionamiento, formatos) y QUÉ transformaciones aplicar
> (funciones nativas, Pandas UDFs bien configuradas).
>
> El trabajo de Spark es decidir CÓMO hacerlo en paralelo.
> Cuando interfieren — con UDFs que cruzan la frontera JVM/Python fila a fila,
> o con `collect()` que rompe el paralelismo — el rendimiento colapsa.
>
> La Spark UI muestra exactamente dónde ocurre ese colapso.
> Saber leerla es la habilidad más valiosa de este capítulo.
