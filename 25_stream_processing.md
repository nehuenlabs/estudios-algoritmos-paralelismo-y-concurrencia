# Guía de Ejercicios — Cap.25: Stream Processing con Beam, Kafka y Flink

> Lenguajes: **Python (Beam/PySpark Streaming)**, **Java (Beam/Kafka Streams)**,
> **con referencias a Rust (Polars/DataFusion)** donde aplica.
>
> Este capítulo extiende el Cap.23 §23.4 (stream processing conceptual)
> con implementaciones concretas en frameworks de producción.
>
> La diferencia central respecto al Cap.24 (Spark batch):
> en batch, procesas datos que ya existen.
> En streaming, procesas datos que están llegando ahora mismo —
> y no sabes cuándo terminarán de llegar.

---

## El problema único del streaming

```
Batch processing:
  Input: todos los datos existen antes de empezar
  Output: resultado cuando termina el job
  Tiempo: minutos a horas
  Ejemplo: "calcular el revenue del mes pasado"

Stream processing:
  Input: los datos llegan continuamente, sin fin
  Output: resultado actualizado continuamente
  Tiempo: milisegundos a segundos
  Ejemplo: "calcular el revenue de los últimos 5 minutos, actualizado cada segundo"

El problema de los datos tardíos (late arrivals):
  Un evento ocurrió a las 14:00 pero llegó al sistema a las 14:07.
  En batch: no es un problema — el job del día siguiente lo incluye.
  En streaming: ¿lo incluyes en la ventana de 14:00-14:05 o en la de 14:05-14:10?
  ¿O lo descarta porque la ventana de 14:00 ya "cerró"?
  → Este problema no existe en batch. En streaming es central.
```

---

## El ecosistema: qué herramienta para qué caso

```
Kafka:
  Rol: el log distribuido — el "sistema nervioso" del streaming
  No procesa datos: los almacena y transporta
  Garantías: at-least-once, ordenado dentro de una partición
  Cuándo: como backbone de cualquier arquitectura de streaming

Kafka Streams (Java/Scala):
  Rol: procesamiento stateful sobre Kafka, embebido en la aplicación
  Sin cluster separado: la aplicación ES el procesador
  Cuándo: microservicios que procesan eventos de Kafka, transformaciones simples

Apache Beam (Python/Java/Go):
  Rol: modelo unificado batch + streaming, portable entre runners
  Runners: Dataflow (Google), Flink, Spark, Direct
  Cuándo: cuando necesitas el mismo código para batch y streaming,
           o necesitas cambiar de runner sin reescribir

Apache Flink (Java/Python):
  Rol: streaming puro, stateful, con garantías exactly-once
  Más potente que Spark Streaming para casos de streaming genuino
  Cuándo: low-latency (< 100ms), stateful complejo, event-time exacto

Spark Structured Streaming (Python/Scala):
  Rol: micro-batching como si fuera streaming
  Integrado con el ecosistema Spark/Delta Lake
  Cuándo: tienes inversión en Spark, latencia de segundos es aceptable

Polars/DataFusion (Rust):
  Rol: procesamiento de datos eficiente en single-node
  No son frameworks de streaming distribuido
  Cuándo: datos que caben en una máquina, máxima eficiencia sin JVM
```

---

## Tabla de contenidos

- [Sección 25.1 — Kafka: el log distribuido](#sección-251--kafka-el-log-distribuido)
- [Sección 25.2 — Apache Beam: el modelo unificado](#sección-252--apache-beam-el-modelo-unificado)
- [Sección 25.3 — Windowing: computar sobre tiempo](#sección-253--windowing-computar-sobre-tiempo)
- [Sección 25.4 — Estado en streaming: más allá de las ventanas](#sección-254--estado-en-streaming-más-allá-de-las-ventanas)
- [Sección 25.5 — Spark Structured Streaming](#sección-255--spark-structured-streaming)
- [Sección 25.6 — Polars y DataFusion: Rust en el pipeline](#sección-256--polars-y-datafusion-rust-en-el-pipeline)
- [Sección 25.7 — Arquitectura: el pipeline de streaming completo](#sección-257--arquitectura-el-pipeline-de-streaming-completo)

---

## Sección 25.1 — Kafka: el Log Distribuido

Kafka no es una cola de mensajes — es un log distribuido.
La diferencia es fundamental para entender su modelo de paralelismo.

```
Cola de mensajes (RabbitMQ, SQS):
  Productor → [ msg1, msg2, msg3 ] → Consumidor
  Cuando el consumidor lee msg1, msg1 desaparece de la cola
  Solo un consumidor puede leer msg1
  El orden entre consumidores no está garantizado

Kafka (log distribuido):
  Productor → [ msg1, msg2, msg3, msg4... ] → N consumidores independientes
  Cuando el consumidor A lee msg1, msg1 sigue en el log
  Consumidor B puede leer msg1 independientemente, más tarde
  El orden dentro de una partición está garantizado
  Los mensajes se retienen N días (no se borran al leer)
```

### Ejercicio 25.1.1 — Producir y consumir mensajes básicos

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json

# Productor:
producer = Producer({
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',           # esperar confirmación de todas las réplicas
    'retries': 3,
    'enable.idempotence': True,  # exactly-once en el productor
})

def producir_evento(topic: str, key: str, valor: dict):
    producer.produce(
        topic=topic,
        key=key.encode('utf-8'),
        value=json.dumps(valor).encode('utf-8'),
        on_delivery=callback_confirmacion
    )
    producer.poll(0)  # no bloquear — confirmar en background

def callback_confirmacion(err, msg):
    if err:
        print(f"Error produciendo: {err}")
    else:
        print(f"Entregado: {msg.topic()}[{msg.partition()}]@{msg.offset()}")

# Consumidor:
consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'mi-consumer-group',
    'auto.offset.reset': 'earliest',  # leer desde el principio si es nuevo
    'enable.auto.commit': False,       # commit manual para at-least-once
})

consumer.subscribe(['eventos-ventas'])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None:
        continue
    if msg.error():
        if msg.error().code() == KafkaError._PARTITION_EOF:
            continue
        raise KafkaError(msg.error())

    evento = json.loads(msg.value())
    procesar_evento(evento)

    # Commit DESPUÉS de procesar (at-least-once):
    consumer.commit(asynchronous=False)
```

**Restricciones:**
1. Implementar productor y consumidor
2. Simular un fallo del consumidor después de procesar pero antes del commit
3. Verificar que el mensaje se reprocesa (at-least-once)
4. Implementar idempotencia en el consumidor para manejar el reprocesamiento

**Pista:** El commit manual es el mecanismo de at-least-once. Si el consumidor
falla después de procesar pero antes del commit, el próximo consumidor
que arranque reempieza desde el último offset commitado — reprocesando
el último mensaje. Para idempotencia: guardar en la base de datos el `offset`
junto con el resultado. Si el offset ya fue procesado, saltarlo.

---

### Ejercicio 25.1.2 — Particiones y consumer groups: el paralelismo de Kafka

```
Un topic con N particiones puede ser consumido en paralelo por N consumers
en el mismo consumer group.

Topic "ventas" con 6 particiones:
  [P0] [P1] [P2] [P3] [P4] [P5]

Consumer Group "procesador-ventas" con 3 consumers:
  Consumer A → P0, P1
  Consumer B → P2, P3
  Consumer C → P4, P5

Si añades un 4to consumer:
  Consumer A → P0
  Consumer B → P1, P2
  Consumer C → P3, P4
  Consumer D → P5

Si añades un 7mo consumer:
  6 consumers tienen una partición cada uno
  El 7mo consumer está OCIOSO (no hay más particiones que asignar)
  → El número máximo de paralelismo = número de particiones
```

**Restricciones:**
1. Crear un topic con 6 particiones
2. Verificar que un consumer group con 3 consumers distribuye las particiones
3. Escalar a 6 consumers: verificar que cada uno tiene exactamente 1 partición
4. Escalar a 7 consumers: verificar que uno está ocioso
5. Matar un consumer: verificar el rebalanceo automático

**Pista:** El rebalanceo (rebalance) ocurre cuando un consumer entra o sale del grupo.
Durante el rebalanceo, TODOS los consumers del grupo paran de procesar.
Para minimizar el impacto: usar `cooperative sticky rebalancing` (Kafka 2.4+)
que solo mueve las particiones necesarias en lugar de reasignar todas.

---

### Ejercicio 25.1.3 — Offsets: control exacto de la posición de lectura

```python
from confluent_kafka import TopicPartition

# Obtener la posición actual:
particiones = consumer.assignment()
for p in particiones:
    posicion = consumer.position([p])
    print(f"Partición {p.partition}: offset {posicion[0].offset}")

# Seek a una posición específica (útil para reprocessing):
consumer.seek(TopicPartition(topic='ventas', partition=0, offset=1000))

# Seek al inicio:
consumer.seek_to_beginning([TopicPartition('ventas', 0)])

# Seek al final (ignorar mensajes históricos):
consumer.seek_to_end([TopicPartition('ventas', 0)])

# Seek a un timestamp específico:
# "quiero procesar todos los mensajes desde las 14:00 de ayer"
timestamp_ms = int(datetime(2024, 1, 15, 14, 0, 0).timestamp() * 1000)
offsets = consumer.offsets_for_times([
    TopicPartition('ventas', 0, timestamp_ms)
])
consumer.seek(offsets[0])
```

**Restricciones:** Implementar un sistema de "replay":
- Los mensajes se procesan normalmente (at-least-once)
- Un operador puede solicitar "reprocesar desde las 14:00 de ayer"
- El sistema hace seek al timestamp correcto y reprocesa
- Los resultados del reprocesamiento reemplazan los anteriores (idempotente)

**Pista:** El seek por timestamp (`offsets_for_times`) es una de las funcionalidades
más poderosas de Kafka. Para el reprocesamiento idempotente: incluir el offset
de Kafka como parte de la primary key en la tabla destino. Si el mismo evento
se procesa dos veces, el segundo upsert sobreescribe el primero — mismo resultado.

---

### Ejercicio 25.1.4 — Kafka Streams: procesamiento embebido en Java

Kafka Streams es un cliente de Kafka que permite hacer stream processing
directamente en la aplicación, sin un cluster separado:

```java
// Kafka Streams en Java:
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "procesador-ventas");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> ventasStream = builder.stream("ventas");

// Filtrar y transformar:
KStream<String, VentaEnriquecida> ventas_procesadas = ventasStream
    .filter((key, value) -> {
        Venta venta = deserializar(value);
        return venta.monto > 0;
    })
    .mapValues(value -> {
        Venta venta = deserializar(value);
        return enriquecer(venta);
    });

// Agregación con ventana de tiempo:
KTable<Windowed<String>, Long> conteosPorRegion = ventas_procesadas
    .groupBy((key, venta) -> venta.region)
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();

conteosPorRegion.toStream().to("metricas-por-region");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();

// Graceful shutdown:
Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

**Restricciones:**
1. Implementar el pipeline de Kafka Streams
2. Verificar que el estado (conteo por región) sobrevive un restart del proceso
3. ¿Dónde guarda el estado Kafka Streams? ¿Qué pasa si se pierde ese almacenamiento?
4. Escalar a múltiples instancias: ¿cómo se distribuyen las particiones?

**Pista:** Kafka Streams guarda el estado en RocksDB local + en un Kafka topic
de changelog. Si el proceso muere, el estado se restaura desde el changelog topic
al reiniciar — exactamente-una-vez garantizado. Para escalar: múltiples instancias
con el mismo `application.id` forman un grupo y se distribuyen las particiones
(igual que consumer groups en el Ejercicio 25.1.2).

---

### Ejercicio 25.1.5 — Leer: diagnosticar consumer lag

**Tipo: Diagnosticar**

Este dashboard de monitoreo muestra el estado de un consumer group:

```
Consumer Group: procesador-pagos
Topic: pagos-eventos
Partitions: 12

Partition  Offset Actual  Offset del Consumer  Lag      Consumer
P0         1,245,891      1,245,891            0        consumer-1
P1         1,198,432      1,198,432            0        consumer-1
P2         1,267,103      1,267,103            0        consumer-2
P3         1,189,654      1,189,654            0        consumer-2
P4         1,234,876      1,234,876            0        consumer-3
P5         1,201,234      1,201,234            0        consumer-3
P6         1,198,765      1,045,234            153,531  consumer-4  ← !!
P7         1,267,890      1,114,321            153,569  consumer-4  ← !!
P8         1,223,456      1,070,043            153,413  consumer-4  ← !!
P9         1,256,789      1,103,345            153,444  consumer-4  ← !!
P10        2,456,789       891,234           1,565,555  consumer-5  ← !!
P11        2,478,123       889,012           1,589,111  consumer-5  ← !!
```

**Preguntas:**

1. ¿Qué está pasando con consumer-4 y consumer-5?
2. ¿El problema de consumer-5 es igual al de consumer-4?
3. ¿Cuánto tiempo tardará cada consumer en ponerse al día (asumir 10,000 msg/s)?
4. ¿Qué impacto tiene el lag en el negocio (sistema de pagos)?
5. ¿Qué acción tomarías para cada consumer?

**Pista:** El consumer-4 tiene un lag de ~150,000 mensajes distribuido uniformemente
entre sus 4 particiones — sugiere que el consumer está procesando lentamente
o tuvo un downtime breve. Consumer-5 tiene ~1.5M mensajes de lag en solo 2 particiones
(P10 y P11 tienen 2.4M offsets vs ~1.2M de las otras) — sugiere que P10 y P11
tienen un hot spot (muchos más mensajes), O que consumer-5 estuvo caído mucho más tiempo,
O ambos. Las acciones difieren: para consumer-4, escalar o optimizar; para consumer-5,
investigar si P10/P11 tienen skew de producción.

---

## Sección 25.2 — Apache Beam: el Modelo Unificado

Apache Beam resuelve un problema específico: escribir el mismo código
para batch Y para streaming, y poder cambiar el runner (Spark, Flink, Dataflow)
sin cambiar el código.

```
El modelo de Beam:
  Pipeline: el grafo de transformaciones
  PCollection: la colección de elementos (puede ser boundada o unboundada)
  PTransform: una transformación sobre PCollections
  Runner: el motor de ejecución (Spark, Flink, Dataflow, Direct)

La clave: PCollection puede ser:
  - Bounded (batch): todos los elementos existen, el pipeline termina
  - Unbounded (streaming): los elementos llegan continuamente, el pipeline corre forever
  El mismo código funciona para ambos casos.
```

### Ejercicio 25.2.1 — Pipeline básico en Python (Beam)

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

# El mismo pipeline funciona en batch y en streaming:
def pipeline_ventas(runner='DirectRunner'):
    options = PipelineOptions([
        f'--runner={runner}',
        '--streaming' if runner != 'DirectRunner' else '',
    ])

    with beam.Pipeline(options=options) as p:
        ventas = (p
            # Batch: leer de archivo
            | 'LeerVentas' >> beam.io.ReadFromText('ventas.csv')

            # O Streaming: leer de Kafka (mismo código, diferente fuente)
            # | 'LeerVentas' >> beam.io.ReadFromKafka(
            #     consumer_config={'bootstrap.servers': 'localhost:9092'},
            #     topics=['ventas-eventos']
            # )

            | 'ParsearCSV' >> beam.Map(parsear_linea_csv)
            | 'FiltrarValidas' >> beam.Filter(lambda v: v['monto'] > 0)
            | 'AgregarPorRegion' >> beam.CombinePerKey(sum)
            | 'EscribirResultado' >> beam.io.WriteToText('resultado')
        )

# Ejecutar localmente (batch, para testing):
pipeline_ventas(runner='DirectRunner')

# Ejecutar en Dataflow (Google Cloud, streaming):
# pipeline_ventas(runner='DataflowRunner')

# Ejecutar en Flink (streaming con estado):
# pipeline_ventas(runner='FlinkRunner')
```

**Restricciones:**
1. Implementar el pipeline completo con DirectRunner (local, para testing)
2. Añadir un stage de enriquecimiento con datos de referencia (side inputs)
3. Verificar que el mismo código funciona con datos batch y con un generador de streaming simulado

**Pista:** Los side inputs en Beam permiten incluir datos de referencia en
transformaciones: `beam.pvalue.AsSingleton(df_productos)` pasa el DataFrame
de productos como un valor único accesible en cada elemento. Esto es el equivalente
del broadcast de Spark pero en el modelo de Beam.

---

### Ejercicio 25.2.2 — DoFn: la transformación fundamental de Beam

```python
class EnriquecerVentaDoFn(beam.DoFn):
    """
    DoFn es la clase base para transformaciones en Beam.
    Permite lógica más compleja que beam.Map o beam.Filter.
    """

    def setup(self):
        """Llamado una vez por worker cuando se inicia el pipeline."""
        self.cliente_bd = conectar_bd()
        self.modelo_ml = cargar_modelo()  # cargado una vez por worker
        self.cache = {}

    def teardown(self):
        """Llamado una vez cuando el worker termina."""
        self.cliente_bd.close()

    def process(self, elemento, productos_tabla):
        """
        Llamado para cada elemento.
        Puede emitir 0, 1, o múltiples elementos.
        """
        venta = elemento
        producto_id = venta['producto_id']

        # Usar caché para evitar hits a BD por cada elemento:
        if producto_id not in self.cache:
            self.cache[producto_id] = productos_tabla.get(producto_id)

        producto = self.cache[producto_id]
        if producto is None:
            # Emitir a una side output (rama alternativa):
            yield beam.pvalue.TaggedOutput('sin_producto', venta)
            return

        venta_enriquecida = {**venta, 'categoria': producto['categoria']}
        yield venta_enriquecida  # emitir al output principal

    @beam.DoFn.yields_elements
    def process_batch(self, elementos):
        """Alternativa: procesar en batches (más eficiente para modelos ML)."""
        predicciones = self.modelo_ml.predict_batch([e['texto'] for e in elementos])
        for elemento, pred in zip(elementos, predicciones):
            yield {**elemento, 'prediccion': pred}
```

**Restricciones:**
1. Implementar `EnriquecerVentaDoFn` completo con side outputs
2. Verificar que `setup()` se llama una vez por worker (no por elemento)
3. Implementar el routing de los elementos `sin_producto` a una cola de DLQ
4. Comparar el rendimiento de `process` vs `process_batch` para un modelo ML

**Pista:** `setup()` vs `__init__`: en Beam, los DoFn se serializan y envían a los workers.
El constructor (`__init__`) se ejecuta en el driver. `setup()` se ejecuta en cada worker.
Por eso los recursos costosos (conexiones BD, modelos ML) deben iniciarse en `setup()`,
no en `__init__`. Si los inicializas en `__init__`, intentas serializar objetos
no-serializables (conexiones TCP) y el pipeline falla.

---

### Ejercicio 25.2.3 — Side inputs: datos de referencia en el pipeline

```python
# Los side inputs son PCollections que se pasan como argumentos a otras transforms
# Son inmutables durante la ejecución del pipeline

# Cargar datos de referencia:
productos = (p
    | 'LeerProductos' >> beam.io.ReadFromText('productos.csv')
    | 'ParsearProductos' >> beam.Map(parsear_producto)
    | 'IndexarProductos' >> beam.combiners.ToDict(
        key_fn=lambda p: p['id'],
        value_fn=lambda p: p
    )
)

# Usar como side input:
ventas_enriquecidas = (ventas
    | 'EnriquecerVentas' >> beam.Map(
        enriquecer_venta,
        productos=beam.pvalue.AsSingleton(productos)  # dict completo como side input
    )
)

def enriquecer_venta(venta, productos):
    producto = productos.get(venta['producto_id'])
    if producto:
        return {**venta, 'categoria': producto['categoria']}
    return venta
```

**Restricciones:** Implementar tres tipos de side inputs:
1. `AsSingleton`: un valor único (dict de productos)
2. `AsIter`: iterable (lista de regiones válidas para validación)
3. `AsDict`: un diccionario (mapa de descuentos por categoría)

**Pista:** `AsSingleton` falla si la PCollection tiene más de un elemento —
debe ser exactamente uno. Para datos de referencia que son un DataFrame completo,
usar `AsDict` o `AsIter`. Los side inputs se materialializan en memoria en cada worker
— si son grandes (>1GB), pueden causar OOM. Para side inputs grandes, considera
si un join es más apropiado.

---

### Ejercicio 25.2.4 — Beam vs Spark: cuándo elegir cada uno

**Tipo: Diseñar**

Para cada caso de uso, determina si es mejor Beam o Spark Structured Streaming
y justifica la elección:

```
Caso 1:
  Pipeline de ETL que corre en batch cada noche Y también necesita
  una versión de streaming para alertas en tiempo real.
  Infraestructura: Google Cloud (Dataflow disponible).

Caso 2:
  Pipeline de transformación complejo con muchos joins y aggregations.
  Los datos son 10 TB de Parquet en S3.
  El equipo tiene experiencia en PySpark.
  No hay requisito de streaming.

Caso 3:
  Detección de fraude en tiempo real.
  Latencia requerida: < 50ms por evento.
  Estado: historial de 30 días de transacciones por usuario.
  Infraestructura: on-premise, sin cloud.

Caso 4:
  Pipeline que hoy es batch pero el producto quiere añadir
  streaming en 6 meses.
  Equipo pequeño, no experto en ninguno de los dos.

Caso 5:
  Aggregations por ventana de tiempo sobre un stream de Kafka.
  El resultado va a otro topic de Kafka.
  No se necesita escritura a S3 ni a una BD.
```

**Pista:** Beam gana cuando: mismo código para batch y streaming, cambio de runner
es posible (cloud migration), o cuando el pipeline es conceptualmente simple
(filter, map, aggregate). Spark gana cuando: joins complejos, ecosistema Delta Lake,
equipo ya conoce PySpark, o datos principalmente en batch. Para el Caso 3
(< 50ms, estado complejo): ni Beam ni Spark Structured Streaming son ideales —
Flink o Kafka Streams son mejores candidatos. Para el Caso 5: Kafka Streams es
la respuesta más simple (Kafka → Kafka sin cluster separado).

---

### Ejercicio 25.2.5 — Leer: el pipeline de Beam que es más lento en streaming

**Tipo: Diagnosticar**

Un pipeline de Beam funciona correctamente en batch (DirectRunner) pero
es 5× más lento que lo esperado en streaming (FlinkRunner):

```python
# El pipeline:
resultados = (
    eventos  # PCollection unbounded desde Kafka
    | 'ParsearJSON' >> beam.Map(json.loads)
    | 'EnriquecerConBD' >> beam.Map(enriquecer_desde_bd)  # hit a PostgreSQL por elemento
    | 'AgruparPorUsuario' >> beam.GroupByKey()
    | 'CalcularMetricas' >> beam.Map(calcular_metricas_usuario)
    | 'EscribirABigQuery' >> beam.io.WriteToBigQuery(...)
)

def enriquecer_desde_bd(evento):
    conn = psycopg2.connect(...)  # nueva conexión por elemento ← !!
    cursor = conn.execute("SELECT * FROM usuarios WHERE id = %s", [evento['user_id']])
    usuario = cursor.fetchone()
    conn.close()
    return {**evento, 'nombre': usuario['nombre']}
```

**Preguntas:**

1. ¿Por qué funciona en batch pero es lento en streaming?
2. ¿Cuántas conexiones a PostgreSQL se abren si el stream tiene 10,000 eventos/s?
3. ¿Cómo `DoFn.setup()` resuelve el problema de las conexiones?
4. ¿Qué pasa con los datos de usuarios que cambian mientras el pipeline corre?
5. Reescribe la función `enriquecer_desde_bd` correctamente.

**Pista:** En batch, el pipeline procesa los datos de una vez — una conexión
por elemento es ineficiente pero finita. En streaming, el pipeline corre forever —
10,000 eventos/s × 3600 segundos = 36 millones de conexiones por hora.
PostgreSQL tiene un límite de ~100-1000 conexiones concurrentes.
La solución con `DoFn.setup()`: una conexión por worker, reutilizada para todos
los elementos procesados por ese worker. Con pool de conexiones: aún mejor.

---

## Sección 25.3 — Windowing: Computar sobre Tiempo

El windowing es lo que diferencia el stream processing del batch processing:
cómo agrupar eventos que llegan en el tiempo para calcular resultados.

```
Sin ventanas (global window):
  Agregar TODOS los eventos desde el inicio → resultado crece infinitamente
  Útil solo para: acumuladores (total histórico)

Tumbling windows (ventanas fijas sin overlap):
  [14:00-14:05] [14:05-14:10] [14:10-14:15]
  Cada evento pertenece a exactamente una ventana
  Útil para: "revenue cada 5 minutos"

Sliding windows (ventanas con overlap):
  [14:00-14:10] [14:01-14:11] [14:02-14:12] ...
  Cada evento puede pertenecer a múltiples ventanas
  Útil para: "promedio móvil de los últimos 10 minutos, actualizado cada minuto"

Session windows (ventanas por actividad):
  La ventana dura mientras haya actividad; se cierra después de N segundos de inactividad
  Útil para: "sesiones de usuario" (sin duración fija)
```

### Ejercicio 25.3.1 — Implementar tumbling windows en Beam

```python
import apache_beam as beam
from apache_beam.transforms.window import FixedWindows, TimestampedValue
from apache_beam.utils.timestamp import Duration

# Los elementos deben tener timestamps para windowing por event-time:
def añadir_timestamp(evento):
    # El timestamp del evento (cuándo ocurrió, no cuándo llegó al sistema)
    ts = evento['timestamp']
    return TimestampedValue(evento, ts)

pipeline = (p
    | 'Leer' >> beam.io.ReadFromKafka(...)
    | 'ParsearJSON' >> beam.Map(json.loads)
    | 'AñadirTimestamp' >> beam.Map(añadir_timestamp)

    # Ventana de 5 minutos (event-time):
    | 'VentanaFija' >> beam.WindowInto(FixedWindows(5 * 60))

    # Agrupar y calcular dentro de cada ventana:
    | 'AgruparPorRegion' >> beam.Map(lambda e: (e['region'], e['monto']))
    | 'SumarPorVentana' >> beam.CombinePerKey(sum)

    # El output incluye la ventana como contexto:
    | 'FormatearResultado' >> beam.Map(
        lambda kv, window=beam.DoFn.WindowParam: {
            'region': kv[0],
            'total': kv[1],
            'ventana_inicio': window.start.to_utc_datetime(),
            'ventana_fin': window.end.to_utc_datetime(),
        }
    )
)
```

**Restricciones:**
1. Implementar tumbling windows de 5 minutos
2. Implementar sliding windows de 10 minutos con slide de 1 minuto
3. Implementar session windows con gap de 30 minutos de inactividad
4. Verificar que el mismo código funciona con batch y streaming

**Pista:** En batch, las ventanas se calculan sobre los timestamps históricos
— Beam procesa todos los eventos de cada ventana de una vez.
En streaming, Beam espera a que los eventos lleguen (considerando late arrivals)
antes de emitir el resultado de cada ventana. El watermark determina cuándo
Beam considera que una ventana está "completa" y puede emitir el resultado.

---

### Ejercicio 25.3.2 — Watermarks: cuándo cerrar una ventana

```python
# El watermark dice "todos los eventos con timestamp < T han llegado"
# Cuando el watermark supera el fin de una ventana, la ventana "cierra"

# Beam calcula el watermark automáticamente basándose en los timestamps observados
# Pero puede configurarse:
from apache_beam.transforms.trigger import AfterWatermark, AfterProcessingTime, AfterCount

# Trigger por defecto: emitir cuando el watermark cierra la ventana
pipeline = (eventos
    | 'Ventana' >> beam.WindowInto(
        FixedWindows(5 * 60),
        trigger=AfterWatermark(
            late=AfterProcessingTime(60)  # re-emitir hasta 60s después para late arrivals
        ),
        allowed_lateness=Duration(seconds=300),  # aceptar late arrivals hasta 5 minutos
        accumulation_mode=AccumulationMode.ACCUMULATING
    )
)
```

**Restricciones:**
1. Simular late arrivals: enviar eventos con timestamps 3 minutos antes del actual
2. Verificar que los late arrivals se incluyen en la ventana correcta
3. Simular un late arrival de 10 minutos (después del `allowed_lateness`):
   ¿qué pasa con este evento?
4. Comparar `ACCUMULATING` vs `DISCARDING` accumulation mode

**Pista:** `ACCUMULATING`: cada vez que hay un nuevo late arrival, re-emite el resultado
acumulado (incluye todos los eventos anteriores + el nuevo).
`DISCARDING`: solo emite los nuevos eventos del trigger (no acumula los anteriores).
Para sistemas que usan upserts (como BigQuery), `ACCUMULATING` es más simple.
Para sistemas que usan incrementos (ej: "añadir este delta al total"), `DISCARDING`.

---

### Ejercicio 25.3.3 — Leer: el pipeline con ventanas incorrectas

**Tipo: Diagnosticar**

Un data engineer implementó un pipeline de "revenue de los últimos 5 minutos,
actualizado cada minuto". El resultado es incorrecto:

```python
# La implementación:
resultado = (eventos
    | 'Ventana' >> beam.WindowInto(FixedWindows(5 * 60))  # ventanas de 5 minutos
    | 'AgruparPorRegion' >> beam.Map(lambda e: (e['region'], e['monto']))
    | 'Sumar' >> beam.CombinePerKey(sum)
)
```

El resultado en la consola:
```
14:00-14:05: region=norte, total=45,230
14:05-14:10: region=norte, total=51,100
14:10-14:15: region=norte, total=49,870
```

El business user dice: "quiero ver el revenue de los ÚLTIMOS 5 minutos,
actualizado cada minuto — no el revenue de cada ventana de 5 minutos."

**Preguntas:**

1. ¿Cuál es la diferencia entre lo que el código hace y lo que quiere el usuario?
2. ¿Qué tipo de ventana debería usar?
3. ¿Con la ventana correcta, cuántas veces se emite el resultado por hora?
4. ¿Cuántas ventanas simultáneas están activas en cada momento con la corrección?
5. ¿El overhead de la sliding window es mayor que el de la tumbling window? ¿Por qué?

**Pista:** El usuario quiere sliding windows: `SlidingWindows(size=5*60, period=60)`.
A las 14:07, la ventana [14:02-14:07] emite su resultado, y también están activas
[14:03-14:08], [14:04-14:09], [14:05-14:10]. Con period=60s y size=5min:
hay 5 ventanas activas simultáneamente, y se emite 1 resultado por minuto.
El overhead es `size/period = 5` veces mayor que tumbling — cada evento
se procesa en 5 ventanas en lugar de 1.

---

### Ejercicio 25.3.4 — Session windows: agrupar por actividad

```python
from apache_beam.transforms.window import Sessions

# Session windows: la ventana dura mientras hay actividad
# Se cierra después de N segundos sin eventos del mismo key

resultados = (eventos
    | 'AñadirTimestamp' >> beam.Map(añadir_timestamp)

    # Session window con gap de 30 minutos:
    | 'SesionPorUsuario' >> beam.WindowInto(
        Sessions(gap_size=30 * 60),
        # La key para sesiones es (user_id) — implícito en el GroupByKey siguiente
    )

    | 'AgruparPorUsuario' >> beam.Map(lambda e: (e['user_id'], e))
    | 'ColectarSesion' >> beam.GroupByKey()

    | 'AnalizarSesion' >> beam.Map(lambda kv, window=beam.DoFn.WindowParam: {
        'user_id': kv[0],
        'eventos': list(kv[1]),
        'duracion_sesion': (window.end - window.start).seconds,
        'inicio': window.start.to_utc_datetime(),
        'fin': window.end.to_utc_datetime(),
    })
)
```

**Restricciones:**
1. Implementar el análisis de sesiones de usuario
2. Calcular: duración de sesión, eventos totales, primer y último evento
3. Detectar "sesiones anómalas" (más de 1000 eventos o más de 4 horas)
4. ¿Cómo se comporta con usuarios que tienen actividad continua durante horas?

**Pista:** Con Sessions y gap de 30 minutos, un usuario que hace un click cada
25 minutos durante 8 horas tendrá UNA sesión de 8 horas (nunca hay un gap de 30min).
Si eso no es el comportamiento deseado, añadir un límite de duración máxima de sesión:
`Sessions(gap_size=30*60, max_gap=4*3600)` — aunque Beam no tiene esta opción nativa.
La alternativa: usar un trigger que emita resultados parciales cada hora.

---

### Ejercicio 25.3.5 — Event-time vs processing-time: la diferencia crucial

```python
# Event-time: el timestamp del evento (cuándo ocurrió)
# Processing-time: el timestamp del sistema cuando el evento se procesa (cuándo llegó)

# El mismo evento puede tener timestamps muy diferentes:
evento = {
    'timestamp': '2024-01-15T14:00:00',  # event-time: ocurrió a las 14:00
    'procesado_at': '2024-01-15T14:07:00',  # processing-time: llegó 7 minutos después
    'user_id': 'u123',
    'monto': 150.0
}

# Ventana por event-time (lo correcto para análisis de negocio):
beam.WindowInto(FixedWindows(5 * 60))  # usa el timestamp del TimestampedValue

# Ventana por processing-time (más simple pero incorrecto para análisis):
beam.WindowInto(FixedWindows(5 * 60),
    timestamp_combiner=TimestampCombiner.OUTPUT_AT_CURRENT_PROCESSING_TIME
)
```

**Restricciones:** Crear un dataset con late arrivals y demostrar que:
1. Con event-time windowing, el evento tardío se incluye en la ventana correcta
2. Con processing-time windowing, el evento tardío va a la ventana de cuando llegó
3. ¿Cuál es "correcto"? Depende de la pregunta de negocio — argumentar ambos casos

**Pista:** Para preguntas de negocio retrospectivas ("¿cuánto vendimos entre 14:00 y 14:05?"),
event-time es la respuesta correcta — incluye todos los eventos que OCURRIERON en ese rango,
aunque llegaran tarde. Para preguntas operacionales ("¿cuántos eventos estamos procesando ahora?"),
processing-time puede ser más apropiado. La mayoría de analytics usa event-time.

---

## Sección 25.4 — Estado en Streaming: Más Allá de las Ventanas

Las ventanas son estado con lifetime fijo. Hay casos que requieren estado
sin un límite de tiempo claro.

### Ejercicio 25.4.1 — Stateful DoFn en Beam

```python
class DetectorFraudeDoFn(beam.DoFn):
    """
    Mantiene el historial de transacciones por usuario.
    Detecta patrones sospechosos: >5 transacciones en 10 minutos.
    """

    # El estado se declara como atributos de clase:
    CONTADOR = beam.transforms.userstate.ReadModifyWriteStateSpec(
        'contador', beam.coders.VarIntCoder()
    )
    ULTIMO_RESET = beam.transforms.userstate.ReadModifyWriteStateSpec(
        'ultimo_reset', beam.coders.FloatCoder()
    )
    TIMER_RESET = beam.transforms.userstate.TimerSpec(
        'timer_reset', beam.transforms.userstate.TimeDomain.WATERMARK
    )

    def process(
        self,
        elemento,
        contador=beam.DoFn.StateParam(CONTADOR),
        ultimo_reset=beam.DoFn.StateParam(ULTIMO_RESET),
        timer=beam.DoFn.TimerParam(TIMER_RESET),
        timestamp=beam.DoFn.TimestampParam,
    ):
        count = (contador.read() or 0) + 1
        contador.write(count)

        if count == 1:
            # Primera transacción: programar reset en 10 minutos
            timer.set(timestamp + Duration(seconds=600))

        if count > 5:
            yield beam.pvalue.TaggedOutput('fraude', elemento)
        else:
            yield elemento

    @beam.DoFn.on_timer(TIMER_RESET)
    def reset_contador(self, contador=beam.DoFn.StateParam(CONTADOR)):
        contador.clear()
```

**Restricciones:**
1. Implementar el detector de fraude completo
2. ¿Qué pasa si el timer se dispara pero llegan eventos tardíos después?
3. Implementar: "bloquear usuario si >10 transacciones en 1 hora"
4. ¿Cómo se persiste el estado entre reinicios del pipeline?

**Pista:** El estado en Beam se persiste automáticamente en el runner
(RocksDB en Flink, Bigtable en Dataflow). Al reiniciar el pipeline,
el estado se restaura — el detector de fraude "recuerda" el contador del usuario
aunque el worker que lo procesaba haya muerto. El timer también se restaura.

---

### Ejercicio 25.4.2 — Kafka Streams: joins entre streams

```java
// Join entre dos streams: ventas + clicks
// "Para cada venta, encontrar el click que la generó (en los últimos 5 minutos)"

KStream<String, Venta> ventas = builder.stream("ventas");
KStream<String, Click> clicks = builder.stream("clicks");

// Stream-Stream Join (ventana de tiempo):
KStream<String, VentaConClick> ventasConClick = ventas.join(
    clicks,
    (venta, click) -> new VentaConClick(venta, click),
    JoinWindows.of(Duration.ofMinutes(5)),  // ventana de 5 minutos para el join
    StreamJoined.with(Serdes.String(), ventaSerde, clickSerde)
);

// Stream-Table Join (sin ventana - el KTable representa el estado actual):
KTable<String, PerfilUsuario> perfiles = builder.table("perfiles-usuarios");

KStream<String, VentaEnriquecida> ventasEnriquecidas = ventas.join(
    perfiles,
    (venta, perfil) -> new VentaEnriquecida(venta, perfil)
    // El perfil que se usa es el más reciente al momento del join
);
```

**Restricciones:**
1. Implementar el stream-stream join en Java
2. ¿Qué pasa con ventas que no tienen un click correspondiente?
3. ¿Cómo se persiste el estado del join (los clicks esperando su venta)?
4. Implementar el stream-table join con actualización del KTable

**Pista:** El stream-stream join mantiene una ventana de ambos streams en estado local.
Kafka Streams guarda este estado en RocksDB + un changelog topic de Kafka.
Si el stream de clicks tiene mucho más volumen que ventas, el estado puede crecer
significativamente. Para controlar: la ventana de tiempo determina cuánto estado
se retiene — con ventana de 5 minutos, solo los clicks de los últimos 5 minutos.

---

### Ejercicio 25.4.3 — Gestionar el crecimiento del estado

El estado en streaming puede crecer indefinidamente si no se limpia:

```python
# Problema: el estado de "sesiones de usuario" crece sin límite
# Un usuario que nunca vuelve sigue ocupando estado para siempre

class GestorSesionesDoFn(beam.DoFn):
    SESION = ReadModifyWriteStateSpec('sesion', SesionCoder())
    TIMER_EXPIRACION = TimerSpec('expiracion', TimeDomain.WATERMARK)

    def process(self, elemento, sesion=StateParam(SESION), timer=TimerParam(TIMER_EXPIRACION), ts=TimestampParam):
        sesion_actual = sesion.read() or Sesion()
        sesion_actual.añadir_evento(elemento)
        sesion.write(sesion_actual)

        # Renovar el timer de expiración: si no hay actividad en 30 min, limpiar
        timer.set(ts + Duration(seconds=1800))
        yield elemento

    @beam.DoFn.on_timer(TIMER_EXPIRACION)
    def expirar_sesion(self, sesion=StateParam(SESION)):
        sesion_final = sesion.read()
        if sesion_final:
            # Emitir la sesión completa antes de limpiar:
            # (esto requiere un output en el método del timer)
            sesion.clear()
```

**Restricciones:**
1. Implementar la limpieza de estado con timers
2. Calcular cuánto estado acumula un sistema con 1M usuarios activos/día
3. Implementar TTL configurable por tipo de estado
4. ¿Qué pasa con el estado cuando un worker falla y se reinicia?

**Pista:** El estado sin limpieza puede ser catastrófico. Con 1M usuarios/día,
cada uno con ~1KB de estado de sesión: 1 GB de estado nuevo por día.
Si el sistema lleva 1 año corriendo y no limpia el estado: ~365 GB de estado.
RocksDB puede manejarlo, pero el startup time (restaurar el estado) crece.
Los timers de limpieza son obligatorios para cualquier estado de larga duración.

---

### Ejercicio 25.4.4 — Leer: el pipeline con memory leak de estado

**Tipo: Diagnosticar**

Un pipeline de Kafka Streams lleva 30 días corriendo y el operador nota:

```
Métricas del procesador (hace 30 días):
  Heap usado: 2 GB / 8 GB
  RocksDB size: 500 MB
  Processing rate: 10,000 msg/s
  Latencia P99: 15ms

Métricas actuales:
  Heap usado: 7.8 GB / 8 GB  ← casi OOM
  RocksDB size: 45 GB         ← !!
  Processing rate: 3,500 msg/s ← degradado
  Latencia P99: 850ms          ← !!
  GC pauses: cada 30 segundos, 2-3 segundos cada una ← !!

El código del state store:
  stateStore.put(userId, nuevoEstado);  // siempre añade, nunca limpia
```

**Preguntas:**

1. ¿Por qué RocksDB creció de 500 MB a 45 GB en 30 días?
2. ¿Cómo el crecimiento de RocksDB afecta el heap de Java?
3. ¿Por qué bajó el processing rate de 10,000 a 3,500 msg/s?
4. ¿Qué cambio en el código habría prevenido esto?
5. ¿Cómo recuperarse sin downtime del estado actual de 45 GB?

**Pista:** RocksDB es en disco pero usa el page cache del SO y el heap de Java
para el block cache. Con 45 GB de RocksDB, el block cache de Java consume
una fracción significativa del heap. Los GC pauses se disparan porque el heap
está casi lleno (el GC trabaja más para liberar espacio). La solución es añadir
retention policy: `stateStore.delete(userId)` después de cierto tiempo de inactividad,
o usar un compaction listener que limpie estados viejos. Sin reinicio, se puede
añadir la lógica de limpieza y dejar que el compaction de RocksDB reduzca el tamaño.

---

### Ejercicio 25.4.5 — Exactly-once en streaming stateful

```python
# El problema: si el worker falla después de actualizar el estado
# pero antes de commitear el offset de Kafka, ¿qué pasa?

# Sin exactly-once:
# 1. Procesar evento → actualizar estado local
# 2. [CRASH]
# 3. Reiniciar: releer el evento desde Kafka (at-least-once)
# 4. Actualizar estado de nuevo → el estado se duplicó

# Con exactly-once (Flink/Beam con checkpointing):
# 1. Checkpoint: guardar el estado + el offset de Kafka juntos
# 2. Procesar evento → actualizar estado
# 3. [CRASH antes del siguiente checkpoint]
# 4. Reiniciar desde el último checkpoint: estado + offset restaurados
# 5. Reprocesar desde el offset del checkpoint → exactly-once
```

**Restricciones:** Implementar con Spark Structured Streaming:
1. Un pipeline con estado (running total por usuario)
2. Simular un fallo (matar el proceso)
3. Reiniciar y verificar que el estado se restauró correctamente
4. Verificar que los eventos no se duplican

**Pista:** En Spark Structured Streaming, el checkpointing se configura con:
```python
query = df.writeStream \
    .outputMode("update") \
    .option("checkpointLocation", "s3://bucket/checkpoints/") \
    .start()
```
El checkpoint incluye el estado de las aggregations y el offset de Kafka.
Al reiniciar con el mismo checkpoint location, Spark restaura exactamente
desde donde quedó — el "exactly-once" en Spark Streaming es realmente
"at-least-once procesamiento + idempotent sinks" (el resultado final es correcto
aunque los eventos se procesen más de una vez internamente).

---

## Sección 25.5 — Spark Structured Streaming

Structured Streaming trata el stream como una tabla que crece continuamente.

### Ejercicio 25.5.1 — Structured Streaming básico desde Kafka

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StringType, FloatType, TimestampType

spark = SparkSession.builder \
    .appName("streaming-ventas") \
    .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.0") \
    .getOrCreate()

# Schema del mensaje de Kafka:
schema = StructType() \
    .add("user_id", StringType()) \
    .add("region", StringType()) \
    .add("monto", FloatType()) \
    .add("timestamp", TimestampType())

# Leer del stream de Kafka:
df_stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("subscribe", "ventas-eventos")
    .option("startingOffsets", "latest")
    .load()
)

# El valor de Kafka viene como bytes — parsear:
df_ventas = (df_stream
    .select(F.from_json(F.col("value").cast("string"), schema).alias("data"))
    .select("data.*")
)

# Aggregation con ventana de tiempo (event-time):
df_metricas = (df_ventas
    .withWatermark("timestamp", "10 minutes")  # tolerar 10 min de late arrivals
    .groupBy(
        F.window("timestamp", "5 minutes"),
        "region"
    )
    .agg(F.sum("monto").alias("total"))
)

# Escribir el resultado:
query = (df_metricas.writeStream
    .format("console")
    .outputMode("update")  # solo emitir filas que cambiaron
    .option("checkpointLocation", "/tmp/checkpoints/metricas")
    .start()
)

query.awaitTermination()
```

**Restricciones:**
1. Implementar el pipeline completo
2. Usar `outputMode("append")` vs `outputMode("update")` vs `outputMode("complete")`:
   ¿cuándo funciona cada uno con aggregations con watermark?
3. Escribir a Delta Lake en lugar de console
4. Simular late arrivals y verificar que el watermark los maneja

**Pista:** `outputMode("append")` para aggregations con watermark: solo emite
cuando la ventana está completamente cerrada (watermark pasó el fin de la ventana).
`outputMode("update")`: emite cuando cualquier fila cambia (más frecuente).
`outputMode("complete")`: emite toda la tabla en cada trigger (solo para aggregations
sin watermark — con muchas ventanas puede ser muy grande).

---

### Ejercicio 25.5.2 — foreachBatch: la puerta de salida al mundo real

```python
# Para writes complejos (upserts, joins con datos estáticos, APIs externas):
def procesar_batch(df_batch, batch_id):
    """
    Se llama con cada micro-batch.
    df_batch es un DataFrame batch normal — todas las operaciones de Spark aplican.
    """
    # Upsert a Delta Lake:
    from delta.tables import DeltaTable

    if DeltaTable.isDeltaTable(spark, "s3://bucket/ventas_acumuladas"):
        delta_table = DeltaTable.forPath(spark, "s3://bucket/ventas_acumuladas")
        delta_table.alias("target").merge(
            df_batch.alias("source"),
            "target.region = source.region AND target.ventana = source.ventana"
        ).whenMatchedUpdateAll() \
         .whenNotMatchedInsertAll() \
         .execute()
    else:
        df_batch.write.format("delta").save("s3://bucket/ventas_acumuladas")

    # También escribir a una API externa:
    alertas = df_batch.filter(F.col("total") > 1_000_000)
    if alertas.count() > 0:
        notificar_alertas(alertas.collect())

query = (df_metricas.writeStream
    .foreachBatch(procesar_batch)
    .option("checkpointLocation", "/tmp/checkpoints/")
    .start()
)
```

**Restricciones:**
1. Implementar `foreachBatch` con upsert a Delta Lake
2. Manejar errores en `foreachBatch` sin detener el stream
3. Hacer idempotente el `foreachBatch` usando `batch_id`
4. ¿Cuál es el guarantee de exactamente-una-vez con `foreachBatch`?

**Pista:** `foreachBatch` con `batch_id` permite idempotencia: si el mismo batch
se reprocesa (por un reinicio), tiene el mismo `batch_id`. Puedes usar `batch_id`
como condición: si el batch ya fue procesado (guardado en una tabla de control),
saltarlo. Esto convierte at-least-once en effectively-exactly-once.

---

### Ejercicio 25.5.3 — Structured Streaming vs Flink: cuándo Spark no es suficiente

**Tipo: Analizar**

Un sistema requiere:
- Latencia P99 < 100ms entre que el evento llega y el resultado está disponible
- Estado complejo: el historial de 90 días de cada usuario (>1TB de estado total)
- Joins entre un stream y una tabla que cambia cada segundo
- Procesamiento de 10 millones de eventos por segundo

**Preguntas:**

1. ¿Puede Spark Structured Streaming satisfacer la latencia de < 100ms? ¿Por qué?
2. ¿Cómo maneja Spark el estado de 1TB comparado con Flink?
3. ¿Qué pasa con la tabla que cambia cada segundo en un stream-static join de Spark?
4. ¿10M eventos/segundo es escalable en Spark? ¿En Flink?
5. ¿Cuál es el framework correcto para este caso de uso?

**Pista:** Spark Structured Streaming usa micro-batching — la latencia mínima real
es el intervalo del trigger (típicamente 100ms-1s). Para latencias < 100ms,
necesitas un sistema de streaming puro como Flink (que procesa evento a evento).
El estado de 1TB: Flink puede manejar estado en RocksDB on-disk con incremental checkpointing.
Spark Structured Streaming también puede, pero los checkpoints son más costosos.
El join con tabla que cambia cada segundo: en Spark, las tablas estáticas se
cargan una vez por micro-batch — si cambia cada segundo y el trigger es cada segundo,
funciona pero es costoso.

---

### Ejercicio 25.5.4 — Delta Lake y streaming: el lakehouse

```python
# Delta Lake permite stream desde y hacia tablas Delta
# con garantías ACID y schema evolution:

# Leer un stream desde una tabla Delta:
df_stream = (spark.readStream
    .format("delta")
    .option("ignoreChanges", "true")
    .load("s3://bucket/ventas_raw/")
)

# Escribir un stream a una tabla Delta:
query = (df_resultado
    .writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "s3://bucket/checkpoints/ventas_proc/")
    .option("mergeSchema", "true")  # permitir schema evolution
    .start("s3://bucket/ventas_procesadas/")
)

# Leer la tabla Delta con time-travel:
df_ayer = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-14") \
    .load("s3://bucket/ventas_procesadas/")
```

**Restricciones:**
1. Implementar un pipeline streaming → Delta Lake
2. Verificar el time-travel: leer el estado de la tabla hace 1 hora
3. Implementar schema evolution: añadir una columna al schema del stream
4. ¿Cómo funciona el ACID de Delta Lake con escrituras concurrentes de streams?

**Pista:** Delta Lake usa Optimistic Concurrency Control (OCC): cada escritura
intenta commitear y falla si otra escritura modificó los mismos datos entre
el inicio y el commit. Para streams que escriben particiones distintas, no hay conflicto.
Para streams que usan `MERGE` (upserts), puede haber conflictos si múltiples
streams hacen upsert en las mismas keys — solo uno gana, el otro reintenta.

---

### Ejercicio 25.5.5 — Leer: el stream que procesa el pasado

**Tipo: Diagnosticar**

Un data engineer configuró un stream de Kafka que debería procesar
eventos en tiempo real pero el dashboard muestra que está procesando
eventos de hace 3 horas:

```python
# Configuración del stream:
df_stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("subscribe", "ventas")
    .option("startingOffsets", "earliest")  # ← !!
    .load()
)
```

El consumer group lag:
```
Partición  Offset disponible  Offset del consumer  Lag
P0         4,567,890          1,234,567             3,333,323  ← !!
P1         4,598,123          1,267,890             3,330,233
...
```

**Preguntas:**

1. ¿Por qué el stream está procesando eventos de hace 3 horas?
2. ¿Qué significa `startingOffsets: earliest` en un consumer nuevo?
3. ¿Cuándo debería usarse `earliest` vs `latest`?
4. ¿Cuánto tiempo tardará el stream en "ponerse al día"?
5. ¿Cómo hacer una migración segura: procesar el backlog histórico
   Y al mismo tiempo no perderse los eventos nuevos?

**Pista:** `startingOffsets: earliest` hace que el stream empiece desde
el principio del topic — todos los mensajes históricos. Si el topic tiene
3 horas de retención y el stream es nuevo, leerá todo el historial antes
de llegar al presente. Para un nuevo stream que solo necesita datos recientes:
`startingOffsets: latest`. Para el backlog: un job batch separado procesa
el historial, y el stream empieza desde `latest` — evitando la espera.

---

## Sección 25.6 — Polars y DataFusion: Rust en el Pipeline

### Ejercicio 25.6.1 — Polars: paralelismo sin GIL

```python
import polars as pl

# Polars procesa en paralelo automáticamente (Rust, sin GIL)
# Sin necesidad de Spark ni JVM

# Lazy evaluation (similar a Spark):
df = (pl.scan_parquet("s3://bucket/ventas/*.parquet")
    .filter(pl.col("año") == 2024)
    .group_by(["region", "mes"])
    .agg([
        pl.sum("monto").alias("total"),
        pl.count().alias("transacciones"),
        pl.mean("monto").alias("promedio")
    ])
    .sort("total", descending=True)
)

# .collect() ejecuta el plan:
resultado = df.collect()

# Ver el plan de ejecución (como explain() en Spark):
print(df.explain())
```

**Restricciones:**
1. Comparar el rendimiento de Polars vs Pandas para:
   - GroupBy de 100M filas
   - Join de dos DataFrames de 10M filas cada uno
   - Filter + sort de 500M filas
2. ¿Cuándo Polars supera a Spark para un mismo dataset?
3. ¿Cuándo Spark sigue siendo necesario a pesar de Polars?

**Pista:** Polars supera a Spark para datasets que caben en una sola máquina
(hasta ~100GB en una máquina con 256GB RAM). El overhead de Spark (JVM, serialización,
coordinación de cluster) es significativo para datos pequeños. Para 1 GB de datos:
Polars puede ser 10-100× más rápido que Spark. Para 10 TB de datos: Spark
es la única opción (Polars no distribuye across machines).

---

### Ejercicio 25.6.2 — Polars streaming: datasets más grandes que la memoria

```python
# Polars tiene un modo streaming para datasets que no caben en memoria:
df_lazy = (pl.scan_parquet("s3://bucket/ventas_historicas/*.parquet")
    .filter(pl.col("año").is_between(2020, 2024))
    .group_by("region")
    .agg(pl.sum("monto"))
)

# Streaming: procesa por batches sin cargar todo en memoria
resultado = df_lazy.collect(streaming=True)

# También puede escribir directamente sin cargar en memoria:
df_lazy.sink_parquet("s3://bucket/resultado.parquet")
```

**Restricciones:**
1. Comparar `collect()` vs `collect(streaming=True)` para un dataset de 50GB
2. ¿Qué operaciones soporta el modo streaming y cuáles no?
3. ¿Cuándo `sink_parquet()` es mejor que `collect()` + `write_parquet()`?

**Pista:** El modo streaming de Polars NO es streaming como Kafka — es procesamiento
por batches de un dataset estático para evitar OOM. Las operaciones que NO soporta
en streaming: sorts globales (requieren ver todos los datos), joins complejos.
Para operaciones que sí soporta (filter, group_by, sum), procesa batch a batch
manteniendo solo el estado necesario en memoria.

---

### Ejercicio 25.6.3 — DataFusion: SQL distribuido en Rust

```python
# DataFusion (Python bindings):
import datafusion

ctx = datafusion.SessionContext()

# Registrar fuentes de datos:
ctx.register_parquet("ventas", "s3://bucket/ventas/*.parquet")
ctx.register_csv("regiones", "regiones.csv")

# Ejecutar SQL (optimizado internamente por Rust):
resultado = ctx.sql("""
    SELECT
        r.nombre as region,
        DATE_TRUNC('month', v.fecha) as mes,
        SUM(v.monto) as total,
        COUNT(*) as transacciones
    FROM ventas v
    JOIN regiones r ON v.region_id = r.id
    WHERE v.fecha >= '2024-01-01'
    GROUP BY r.nombre, DATE_TRUNC('month', v.fecha)
    ORDER BY total DESC
""").collect()
```

**Restricciones:**
1. Comparar DataFusion vs Polars vs Spark para una query SQL sobre 10GB de Parquet
2. ¿Cuándo DataFusion tiene ventaja sobre Polars?
3. ¿DataFusion puede ser un runner de Beam? ¿Hay planes para esto?

**Pista:** DataFusion usa Apache Arrow internamente (igual que Polars).
La diferencia principal: DataFusion está orientado a SQL y puede funcionar como
motor embebido en otras aplicaciones (es el motor de Ballista, DeltaFusion, etc.).
Polars está orientado a DataFrames (API más similar a Pandas/Spark).
Para queries SQL ad-hoc sobre Parquet, DataFusion puede ser más conveniente.
Para transformaciones programáticas complejas, Polars.

---

### Ejercicio 25.6.4 — Integrar Rust (Polars/DataFusion) con el ecosistema Python

```python
# Polars y Pandas comparten Apache Arrow como formato en memoria
# La conversión es zero-copy cuando es posible:

import polars as pl
import pandas as pd
import pyarrow as pa

# Polars → Pandas (zero-copy para tipos compatibles):
df_polars = pl.read_parquet("datos.parquet")
df_pandas = df_polars.to_pandas()  # puede ser zero-copy

# Pandas → Polars:
df_polars = pl.from_pandas(df_pandas)

# Polars → PyArrow:
tabla_arrow = df_polars.to_arrow()

# PyArrow → Polars:
df_polars = pl.from_arrow(tabla_arrow)

# Usar Polars para el ETL pesado, Pandas para la parte final:
resultado_polars = (pl.scan_parquet("100gb_dataset.parquet")
    .filter(pl.col("activo") == True)
    .group_by("segmento")
    .agg(pl.sum("valor"))
    .collect()  # ~30 segundos vs ~5 minutos con Pandas
)

# Análisis final con Pandas (librería tiene más herramientas estadísticas):
resultado_pandas = resultado_polars.to_pandas()
print(resultado_pandas.describe())
```

**Restricciones:**
1. Implementar un pipeline que usa Polars para el procesamiento pesado
   y Pandas para el análisis estadístico final
2. Medir si la conversión Polars → Pandas es zero-copy o requiere copia
3. ¿Cuándo la conversión requiere una copia? ¿Cuándo es zero-copy?

**Pista:** La conversión es zero-copy cuando los tipos de datos son directamente
compatibles entre Polars y Arrow (int64, float64, string). Requiere copia cuando:
tipos de Polars no tienen equivalente directo en Arrow (como categorías/enums),
o cuando el DataFrame de Pandas necesita tipos Python nativos vs tipos C.
En la práctica, las conversiones simples son zero-copy y muy rápidas.

---

### Ejercicio 25.6.5 — Leer: ¿Rust reemplazará a la JVM en data engineering?

**Tipo: Analizar**

El ecosistema de data engineering en Rust está creciendo rápidamente:
DataFusion, Polars, Delta-rs, Lance, Ballista.

**Preguntas:**

1. ¿Cuáles son las ventajas concretas de Rust para data engineering
   respecto a la JVM?
2. ¿Qué capacidades de Spark/Flink no tienen equivalente en Rust todavía?
3. ¿Qué barreras de adopción tiene Rust en organizaciones de data engineering?
4. ¿En qué timeframe crees que Rust/DataFusion podría ser viable como
   alternativa a Spark para casos de uso comunes?
5. ¿Es un reemplazo o una complementación?

**Pista:** Las ventajas de Rust: sin GC pauses (importante para latencia p99),
sin JVM overhead (startup time, memory overhead), mejor uso del hardware
(SIMD, cache-friendly), sin GIL (paralelismo real en Python bindings).
Las brechas actuales: ecosistema de ML/analytics (PyTorch, sklearn son Python-first),
operadores SQL complejos en distribución (Ballista es early stage),
streaming stateful complejo (nada al nivel de Flink), y la curva de aprendizaje
de Rust para data engineers que vienen de Python.

---

## Sección 25.7 — Arquitectura: el Pipeline de Streaming Completo

### Ejercicio 25.7.1 — Lambda architecture vs Kappa architecture

```
Lambda Architecture (clásica, ~2012):
  Batch layer: procesa todos los datos históricos, produce "batch views"
  Speed layer: procesa solo datos recientes, produce "real-time views"
  Serving layer: combina batch + speed views para las queries

  Problema: mantener dos pipelines (batch + streaming) del mismo cómputo.
  El mismo cálculo de "revenue por región" necesita ser implementado dos veces.

Kappa Architecture (Kreps, 2014):
  Un solo pipeline de streaming para todo.
  El stream log (Kafka) es la fuente de verdad.
  Si necesitas reprocesar: relee el stream desde el principio.
  
  Problema: el reprocesamiento puede ser costoso para datos históricos largos.
  Solución moderna: Delta Lake/Iceberg como "stream log persistente"
```

**Restricciones:** Diseñar la arquitectura para un sistema de analytics
de e-commerce con:
- Datos históricos: 5 años de transacciones (50 TB)
- Datos en tiempo real: 100,000 eventos/segundo
- Queries: revenue por región (batch diario), alertas de fraude (< 1 segundo)

¿Lambda o Kappa? Justificar con los tradeoffs específicos de este caso.

**Pista:** La tendencia en 2024 es hacia Kappa con streaming batch (Spark Structured
Streaming + Delta Lake o Flink + Iceberg). La clave es que el stream log sea
suficientemente largo (retención de Kafka de 7-30 días) para cubrir el reprocessing
normal. Para datos de 5 años: necesitas un "compacted log" (Delta/Iceberg) que
sirve como stream para el reprocesamiento histórico, y Kafka para los datos recientes.

---

### Ejercicio 25.7.2 — Diseño: pipeline de detección de fraude en tiempo real

```
Requisitos:
  - Input: stream de transacciones bancarias (Kafka, 50,000 msg/s)
  - Latencia: decisión de aprobación/rechazo en < 200ms
  - Estado: historial de 30 días por usuario (velocidad, patrones)
  - ML model: scoring de riesgo (inferencia < 50ms)
  - Exactamente-una-vez: ninguna transacción puede procesarse dos veces
  - Disponibilidad: 99.99% (menos de 1 hora de downtime al año)
```

**Restricciones:** Diseñar el sistema especificando:
- Framework de streaming (Flink, Kafka Streams, Spark, Beam)
- Gestión del estado (RocksDB, Redis, Cassandra)
- Integración del modelo ML (inline vs microservicio externo)
- Estrategia de checkpoint y recuperación
- Cómo garantizar 99.99% de disponibilidad

**Pista:** Para < 200ms con ML model: el scoring debe ser inline (en el mismo proceso)
o muy cercano (misma máquina/rack). Una llamada HTTP a un microservicio externo
añade 10-50ms de overhead de red — viable si el modelo responde en < 20ms.
Para 99.99% disponibilidad: múltiples réplicas del job, failover automático
en < 30 segundos (Flink's high-availability con ZooKeeper/etcd).

---

### Ejercicio 25.7.3 — Backpressure: cuando el procesador es más lento que el productor

```python
# El problema: Kafka produce a 100,000 msg/s
# El procesador procesa a 50,000 msg/s
# El consumer lag crece a 50,000 msg/s → en 1 hora: 180M mensajes de lag

# Opciones:
# 1. Escalar el procesador (añadir más workers/particiones)
# 2. Reducir la producción (rate limiting en el productor)
# 3. Aceptar el lag y procesar eventualmente (si la latencia no es crítica)
# 4. Degradar elegantemente: procesar solo eventos de alta prioridad

# En Spark Structured Streaming: backpressure automático
spark.conf.set("spark.streaming.backpressure.enabled", "true")
spark.conf.set("maxOffsetsPerTrigger", "10000")  # máximo 10,000 mensajes por trigger

# En Kafka Streams: no tiene backpressure automático
# Hay que monitorear el lag y escalar manualmente
```

**Restricciones:** Implementar un sistema con backpressure que:
1. Detecta cuando el consumer lag supera 1 millón de mensajes
2. Activa automáticamente un "modo degradado": procesar solo eventos de prioridad alta
3. Alerta al equipo de operaciones
4. Vuelve a modo normal cuando el lag baja de 100,000 mensajes

**Pista:** El "modo degradado" en Kafka: usar consumer groups con filtro en el consumer,
no en el productor. El consumer A procesa todos los eventos. En modo degradado,
el consumer A salta eventos de baja prioridad. Esto es más simple que modificar
el productor o añadir tópicos separados por prioridad.

---

### Ejercicio 25.7.4 — Testing de pipelines de streaming

El testing de streaming es más difícil que batch porque el tiempo es parte del input.

```python
# Testing con TestStream (Beam):
from apache_beam.testing.test_stream import TestStream

def test_pipeline_con_late_arrivals():
    with TestPipeline() as p:
        # Simular un stream controlado:
        test_stream = (TestStream()
            .add_elements([
                TimestampedValue({'user': 'alice', 'monto': 100}, timestamp=0),
                TimestampedValue({'user': 'bob', 'monto': 200}, timestamp=5),
            ])
            .advance_watermark_to(60)  # avanzar el watermark a T=60
            .add_elements([
                TimestampedValue({'user': 'alice', 'monto': 50}, timestamp=10),  # late!
            ])
            .advance_watermark_to_infinity()
        )

        resultado = (p
            | test_stream
            | 'Ventana' >> beam.WindowInto(FixedWindows(30))
            | 'Agrupar' >> beam.Map(lambda e: (e['user'], e['monto']))
            | 'Sumar' >> beam.CombinePerKey(sum)
        )

        # Verificar el resultado:
        assert_that(resultado, equal_to([
            ('alice', 150),  # 100 + 50 (el late arrival se incluye)
            ('bob', 200),
        ]))
```

**Restricciones:**
1. Implementar tests para: late arrivals, watermark advancement, stateful processing
2. ¿Cómo testear el comportamiento con fallos del worker en medio del procesamiento?
3. ¿Cómo hacer property-based testing para pipelines de streaming?

---

### Ejercicio 25.7.5 — El pipeline completo: de Kafka a Delta Lake

Diseña e implementa el pipeline de producción completo para
una plataforma de e-commerce con estos requisitos:

```
Fuentes:
  - Kafka: eventos de clicks (100,000/s), eventos de compra (1,000/s)
  - PostgreSQL: catálogo de productos (1M productos, actualizado cada hora)

Transformaciones:
  - Enriquecer clicks con datos del catálogo
  - Detectar sesiones de usuario (gap de 30 minutos)
  - Calcular métricas por ventana de 5 minutos: clicks, conversiones, revenue
  - Detectar anomalías: CTR > 50% en 5 minutos para un producto → alerta

Destinos:
  - Delta Lake: métricas para el BI (latencia aceptable: < 5 minutos)
  - Redis: métricas en tiempo real para el dashboard (latencia: < 5 segundos)
  - Slack/PagerDuty: alertas de anomalías (latencia: < 30 segundos)
```

**Restricciones:** Para cada componente, especificar:
- Framework elegido y justificación
- Garantía de consistencia
- Estrategia de recuperación ante fallos
- Monitoreo (qué métricas, qué alertas)

**Pista:** Una arquitectura razonable:
Kafka → Flink (para la latencia baja en Redis y alertas) + Spark Structured
Streaming → Delta Lake (para el BI). El catálogo de productos: cargarlo como
broadcast state en Flink (se actualiza cada hora sin reiniciar el job).
Alternativamente: Kafka → Spark Structured Streaming para todo, con trigger
de 1 minuto — la latencia de 5 minutos al BI está bien, pero para Redis y alertas
necesitarías foreachBatch con lógica de threshold, lo que puede funcionar
con trigger de 10 segundos.

---

## Resumen del capítulo

**El mapa de decisión de stream processing:**

```
¿Los datos caben en una máquina? (< ~100GB)
  Sí → Polars (batch) o procesamiento en memoria
  No → continuar

¿Necesitas latencia < 100ms?
  Sí → Flink o Kafka Streams
  No → continuar

¿Tienes código batch que quieres reutilizar en streaming?
  Sí → Apache Beam (mismo código, diferente runner)
  No → continuar

¿Tienes inversión existente en Spark o en Delta Lake?
  Sí → Spark Structured Streaming
  No → continuar

¿Es solo Kafka → Kafka sin estado complejo?
  Sí → Kafka Streams (embebido, sin cluster separado)
  No → Flink (la opción más potente y flexible)
```

**Lo que este capítulo conecta con el repositorio:**

```
Cap.22 §22.4 (consistencia eventual) →
  25.1 (Kafka at-least-once, exactly-once)

Cap.23 §23.4 (stream processing conceptual) →
  25.2–25.4 (implementaciones concretas en Beam/Flink)

Cap.23 §23.2 (event sourcing) →
  25.5 (Delta Lake como event store con Structured Streaming)

Cap.13 (Rust y ownership) →
  25.6 (Polars/DataFusion: el ownership previene data races en el procesamiento)

Cap.21 §21.6 (degradación elegante) →
  25.7.3 (backpressure y modo degradado en streaming)
```

**El principio que une todo:**

> En batch, el tiempo es un dato más — puedes volver atrás y reprocesar.
> En streaming, el tiempo es el contexto de todo — los late arrivals,
> los watermarks, y los timers existen porque el tiempo avanza
> mientras los datos todavía están en camino.
>
> Todos los problemas únicos del streaming (late arrivals, exactly-once,
> estado que crece) son consecuencia de ese único hecho:
> el tiempo no se puede pausar mientras el sistema procesa.
