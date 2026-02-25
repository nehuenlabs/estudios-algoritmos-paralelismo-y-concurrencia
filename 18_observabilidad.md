# Guía de Ejercicios — Cap.18: Observabilidad en Sistemas Concurrentes

> Lenguaje principal: **Go**. Las herramientas y patrones aplican a cualquier lenguaje.
>
> Este capítulo tiene tres tipos de ejercicios:
> - **Instrumentar**: añadir observabilidad a código existente
> - **Leer**: interpretar métricas, logs, y traces ya recolectados
> - **Diagnosticar**: dado un síntoma, encontrar la causa usando las herramientas
>
> La habilidad de *leer* es la más escasa y la más valiosa. Un ingeniero
> que puede mirar un flame graph y decir "el cuello de botella está aquí"
> en 30 segundos vale más que uno que tarda una hora instalando un debugger.

---

## El problema central de observar sistemas concurrentes

Observar un sistema secuencial es fácil: el estado es una secuencia de pasos.
Observar un sistema concurrente es difícil porque:

```
1. El orden de eventos no es determinista
   Log de thread A: "adquirí el lock"
   Log de thread B: "adquirí el lock"
   ¿Cuál llegó primero? Los logs tienen timestamps del sistema, no del orden lógico.

2. El observador afecta al observado
   Añadir logging dentro de una sección crítica puede cambiar el timing
   y hacer que el bug sea irreproducible.

3. Las métricas agregadas ocultan el problema
   "Latencia promedio = 45ms" puede esconder que el 0.1% de requests
   tardan 10 segundos — justo los requests más importantes.

4. La causalidad no es obvia
   Un spike de latencia en el servicio A puede ser causado por
   el servicio B que no tiene el spike (porque el problema está
   en la *espera* de B, no en B mismo).
```

La solución a cada uno de estos problemas tiene un nombre:

```
Problema 1: orden no determinista  → Vector clocks, Lamport timestamps
Problema 2: observador invasivo    → Sampling, métricas atómicas, pprof
Problema 3: agregados que mienten  → Histogramas de percentiles (p99, p999)
Problema 4: causalidad no obvia    → Distributed tracing (trace IDs, spans)
```

---

## Tabla de contenidos

- [Sección 18.1 — Métricas: lo que mides es lo que operas](#sección-181--métricas-lo-que-mides-es-lo-que-operas)
- [Sección 18.2 — Logs estructurados en sistemas concurrentes](#sección-182--logs-estructurados-en-sistemas-concurrentes)
- [Sección 18.3 — Profiling: flame graphs y pprof](#sección-183--profiling-flame-graphs-y-pprof)
- [Sección 18.4 — Distributed tracing: seguir una request](#sección-184--distributed-tracing-seguir-una-request)
- [Sección 18.5 — Leer dashboards: diagnóstico en 60 segundos](#sección-185--leer-dashboards-diagnóstico-en-60-segundos)
- [Sección 18.6 — Observabilidad de la concurrencia interna](#sección-186--observabilidad-de-la-concurrencia-interna)
- [Sección 18.7 — Construir el sistema de observabilidad mínimo viable](#sección-187--construir-el-sistema-de-observabilidad-mínimo-viable)

---

## Sección 18.1 — Métricas: lo que mides es lo que operas

### Ejercicio 18.1.1 — Leer: ¿qué dice esta métrica?

**Tipo: Leer/diagnosticar**

Dado el siguiente output de Prometheus para un Worker Pool en producción,
responde las preguntas sin ejecutar ningún código:

```
# HELP worker_pool_queue_size Número de tareas esperando en la cola
# TYPE worker_pool_queue_size gauge
worker_pool_queue_size 847

# HELP worker_pool_workers_active Workers procesando activamente
# TYPE worker_pool_workers_active gauge
worker_pool_workers_active 8

# HELP worker_pool_workers_total Workers totales configurados
# TYPE worker_pool_workers_total gauge
worker_pool_workers_total 8

# HELP worker_pool_tasks_total Total de tareas procesadas
# TYPE worker_pool_tasks_total counter
worker_pool_tasks_total{status="ok"} 142831
worker_pool_tasks_total{status="error"} 287
worker_pool_tasks_total{status="timeout"} 1204

# HELP worker_pool_task_duration_seconds Distribución de latencias
# TYPE worker_pool_task_duration_seconds histogram
worker_pool_task_duration_seconds_bucket{le="0.01"} 89234
worker_pool_task_duration_seconds_bucket{le="0.025"} 112445
worker_pool_task_duration_seconds_bucket{le="0.05"} 131200
worker_pool_task_duration_seconds_bucket{le="0.1"} 138900
worker_pool_task_duration_seconds_bucket{le="0.25"} 141500
worker_pool_task_duration_seconds_bucket{le="0.5"} 142100
worker_pool_task_duration_seconds_bucket{le="1.0"} 142600
worker_pool_task_duration_seconds_bucket{le="2.5"} 142780
worker_pool_task_duration_seconds_bucket{le="+Inf"} 144322
worker_pool_task_duration_seconds_sum 8934.2
worker_pool_task_duration_seconds_count 144322
```

**Preguntas:**

1. ¿Cuál es la tasa de error actual (porcentaje de tareas que fallan o hacen timeout)?
2. ¿Está el pool en backpressure? ¿Cómo lo sabes?
3. ¿Cuál es la latencia promedio? ¿Es un número útil en este caso?
4. Aproxima el p50 y el p99 de latencia usando los buckets del histograma.
   (Pista: el p50 está donde el bucket acumulado cruza el 50% de `+Inf`)
5. ¿Hay algo en la distribución de latencias que debería preocuparte?
   Los números `144322` en `+Inf` y `142831` en `status="ok"` no cuadran.
   ¿Qué explica la diferencia?

**Pista:** El bucket `+Inf` siempre contiene el total de observaciones.
La diferencia entre `worker_pool_tasks_total` y `worker_pool_task_duration_seconds_count`
puede indicar tareas que están en vuelo (started pero no finished) — o un bug en la instrumentación.
Para el p99: busca el bucket donde el valor acumulado cruza el 99% de `+Inf`.
`99% × 144322 ≈ 142878`. El primer bucket que supera 142878 es `le="0.25"` (141500 < 142878 < 142780... revisa los números).

---

### Ejercicio 18.1.2 — Instrumentar: añadir métricas a un Worker Pool

**Tipo: Instrumentar**

Dado este Worker Pool sin métricas, añade instrumentación completa
sin degradar el throughput más del 2%:

```go
type WorkerPool struct {
    tasks   chan Task
    wg      sync.WaitGroup
    workers int
}

func NewWorkerPool(n int) *WorkerPool {
    p := &WorkerPool{
        tasks:   make(chan Task, n*10),
        workers: n,
    }
    for i := 0; i < n; i++ {
        p.wg.Add(1)
        go func() {
            defer p.wg.Done()
            for task := range p.tasks {
                task.Run()
            }
        }()
    }
    return p
}

func (p *WorkerPool) Submit(t Task) {
    p.tasks <- t
}

func (p *WorkerPool) Shutdown() {
    close(p.tasks)
    p.wg.Wait()
}
```

**Restricciones:**

- Exponer en formato Prometheus en `/metrics`
- Las métricas de latencia deben usar histograma (no gauge de promedio)
- El tamaño de la cola debe reflejar el valor instantáneo (gauge, no counter)
- No usar locks en el hot path — solo atomics o las operaciones atómicas de Prometheus

**Pista:** `prometheus.MustRegister` en `init()`. Para la latencia, usar
`prometheus.NewHistogramVec` con buckets en milisegundos: `{1, 5, 10, 25, 50, 100, 250, 500, 1000}`.
El tamaño de la cola se puede leer con `len(p.tasks)` — es una operación atómica en Go
para canales buffered. El overhead de `time.Now()` + `time.Since()` es ~20ns;
el overhead de `Observe()` en un histograma de Prometheus es ~100ns. Ambos son
aceptables para tareas que tardan ms.

---

### Ejercicio 18.1.3 — Leer: histogramas que mienten

**Tipo: Leer/diagnosticar**

Un compañero instrumentó el servicio con esta métrica:

```go
var latenciaPromedio = prometheus.NewGauge(prometheus.GaugeOpts{
    Name: "request_latency_ms",
    Help: "Latencia promedio de requests",
})

func handler(w http.ResponseWriter, r *http.Request) {
    inicio := time.Now()
    procesarRequest(r)
    latenciaPromedio.Set(float64(time.Since(inicio).Milliseconds()))
}
```

**Preguntas:**

1. ¿Qué valor muestra `request_latency_ms` cuando hay 100 requests concurrentes?
2. Si el 99% de requests tarda 10ms y el 1% tarda 5000ms, ¿qué valor
   verás en el dashboard la mayoría del tiempo?
3. ¿Por qué esta métrica es peligrosa para alertas?
4. Reescribe la instrumentación usando un histograma correcto.
5. ¿Qué query de PromQL usarías para calcular el p99 de latencia
   de los últimos 5 minutos con el histograma correcto?

**Pista:** Un gauge que se actualiza con cada request refleja el valor de
*la última request*, no un promedio ni un percentil. Con alta concurrencia,
refleja el valor de un request aleatorio en el momento del scrape.
La query de PromQL para p99: `histogram_quantile(0.99, rate(request_duration_seconds_bucket[5m]))`.

---

### Ejercicio 18.1.4 — Diseñar: qué métricas para un sistema específico

**Tipo: Diseñar/instrumentar**

Para un sistema de websockets que gestiona 50,000 conexiones activas simultáneas,
diseña el conjunto mínimo de métricas que permite responder estas preguntas
en menos de 30 segundos durante un incidente:

- ¿Cuántas conexiones activas hay ahora mismo?
- ¿A qué velocidad se crean y destruyen conexiones?
- ¿Cuántos mensajes por segundo se envían y reciben?
- ¿Hay conexiones que llevan más de 1 hora sin actividad (posibles zombies)?
- ¿Está el sistema cerca de su límite de file descriptors del OS?

**Restricciones:** Máximo 8 métricas totales. Cada métrica debe tener un nombre,
tipo (counter/gauge/histogram), y la query de PromQL que la usa para responder
una de las preguntas anteriores.

**Pista:** El límite de file descriptors del OS se puede leer con
`/proc/self/fd` en Linux o con `syscall.Getrlimit(syscall.RLIMIT_NOFILE, &limit)`.
Para conexiones zombies, un gauge de "tiempo de última actividad" por conexión
no escala a 50K conexiones — mejor un histograma de "tiempo desde última actividad"
que se actualiza periódicamente (cada 30s).

---

### Ejercicio 18.1.5 — El USE method aplicado a un Worker Pool

**Tipo: Diagnosticar**

El "USE method" de Brendan Gregg mide para cada recurso:
- **U**tilization: % del tiempo que el recurso está ocupado
- **S**aturation: cuánto trabajo está esperando (cola)
- **E**rrors: tasa de errores

Dado este snapshot de métricas de un Worker Pool bajo carga:

```
workers_active:   16 de 16        (utilización = 100%)
queue_size:       450             (saturación = alta)
tasks_errored:    0.3% en 5min   (errores = bajo)
task_p99:         2.3 segundos   (latencia = alta)
task_timeout:     8% en 5min    (timeouts = preocupante)
```

**Preguntas:**

1. Usando el USE method, ¿cuál es el diagnóstico?
2. ¿Cuál es la causa raíz más probable?
3. ¿Qué acción inmediata (en < 5 minutos) tomarías?
4. ¿Qué acción a largo plazo (en < 1 semana)?
5. Si añadieras 8 workers más (total 24), ¿qué esperas que pase
   con cada una de las 5 métricas?

**Pista:** Utilización al 100% + cola creciente = el recurso (workers) es el cuello de botella.
El 8% de timeouts indica que la cola acumula tan rápido que las tareas expiran
antes de ser procesadas. La acción inmediata es escalar el pool (más workers).
La acción a largo plazo: investigar por qué las tareas tardan 2.3s en p99
(puede ser una dependencia externa lenta que debería tener su propio circuit breaker).

---

## Sección 18.2 — Logs Estructurados en Sistemas Concurrentes

### Ejercicio 18.2.1 — Leer: ordenar eventos concurrentes con logs

**Tipo: Leer/diagnosticar**

Estos logs llegaron en este orden al sistema de log aggregation
(Loki, Splunk, CloudWatch — no importa cuál):

```json
{"time":"10:00:01.523","level":"INFO","msg":"request recibido","request_id":"abc-123","user":"alice"}
{"time":"10:00:01.891","level":"INFO","msg":"request recibido","request_id":"def-456","user":"bob"}
{"time":"10:00:02.104","level":"INFO","msg":"lock adquirido","request_id":"def-456","recurso":"inventario"}
{"time":"10:00:02.231","level":"INFO","msg":"lock adquirido","request_id":"abc-123","recurso":"pagos"}
{"time":"10:00:02.891","level":"INFO","msg":"esperando lock","request_id":"abc-123","recurso":"inventario"}
{"time":"10:00:02.902","level":"INFO","msg":"esperando lock","request_id":"def-456","recurso":"pagos"}
{"time":"10:00:32.904","level":"ERROR","msg":"timeout esperando lock","request_id":"abc-123","recurso":"inventario","waited_ms":30013}
{"time":"10:00:32.906","level":"ERROR","msg":"timeout esperando lock","request_id":"def-456","recurso":"pagos","waited_ms":30004}
```

**Preguntas:**

1. ¿Qué bug de concurrencia muestran estos logs?
2. Dibuja el diagrama de los dos threads y los dos locks con timestamps.
3. ¿Qué información en los logs te permite reconstruir esto sin tener el código?
4. ¿Qué campo falta en estos logs que haría el diagnóstico más rápido?
5. Propón la query de Loki (o SQL equivalente) para detectar este patrón
   automáticamente y alertar antes de que ocurra el timeout.

**Pista:** Los logs muestran un deadlock clásico: `abc-123` tiene `pagos` y espera `inventario`;
`def-456` tiene `inventario` y espera `pagos`. El campo que falta es el `thread_id` o
`goroutine_id` — con él, podrías agrupar todos los logs de la misma goroutine.
Para la query de Loki: buscar pares de "lock adquirido" donde los recursos
aparezcan en orden inverso en requests distintos.

---

### Ejercicio 18.2.2 — Instrumentar: correlation IDs a través de goroutines

**Tipo: Instrumentar**

El problema del logging concurrente: cuando una request lanza múltiples goroutines,
¿cómo asocias todos sus logs con el request original?

```go
// El problema:
func handleRequest(w http.ResponseWriter, r *http.Request) {
    log.Info("request recibido")  // ← ¿qué request?

    go fetchDB()    // los logs de estas goroutines no tienen
    go fetchCache() // contexto del request original
    go fetchAPI()

    log.Info("request completado")
}

// La solución:
func handleRequest(w http.ResponseWriter, r *http.Request) {
    requestID := uuid.New().String()
    ctx := context.WithValue(r.Context(), "request_id", requestID)
    logger := log.With("request_id", requestID)

    logger.Info("request recibido")

    go fetchDB(ctx)    // ← pasar el contexto
    go fetchCache(ctx) //    que contiene el request_id
    go fetchAPI(ctx)   //

    logger.Info("request completado")
}
```

**Restricciones:** Implementar un middleware HTTP en Go que:
1. Genera un `request_id` único si no viene en el header `X-Request-ID`
2. Lo propaga en el contexto y en el header de respuesta
3. Lo incluye automáticamente en todos los logs del handler y sus goroutines
4. Permite filtrar todos los logs de un request específico con una sola query

**Pista:** `context.WithValue` es el mecanismo para propagar el `request_id`,
pero acceder a él en cada goroutine es verboso. Una alternativa es usar
`slog` (Go 1.21+) con un `Handler` que extrae el `request_id` del contexto
automáticamente. Para que `go fetchDB(ctx)` incluya el `request_id`, la función
debe aceptar `ctx context.Context` y extraer el ID del contexto al loguear.

---

### Ejercicio 18.2.3 — El problema del log buffering en sistemas concurrentes

**Tipo: Leer/diagnosticar y corregir**

Este código tiene un bug sutil relacionado con el buffering de logs:

```go
var logBuffer = make([]string, 0, 1000)
var logMu sync.Mutex

func logAsync(msg string) {
    logMu.Lock()
    logBuffer = append(logBuffer, msg)
    logMu.Unlock()
}

func flushLogs() {
    logMu.Lock()
    defer logMu.Unlock()
    for _, msg := range logBuffer {
        fmt.Println(msg) // escribir al stdout
    }
    logBuffer = logBuffer[:0]
}

// Goroutine que hace flush cada segundo:
go func() {
    ticker := time.NewTicker(time.Second)
    for range ticker.C {
        flushLogs()
    }
}()
```

**Preguntas:**

1. Si el proceso muere (SIGKILL, OOM, panic), ¿qué pasa con los logs en el buffer?
2. Si `flushLogs()` tarda 500ms (stdout lento), ¿qué pasa con los logs que llegan
   durante ese tiempo?
3. ¿Cuándo el buffer puede crecer indefinidamente?
4. ¿Qué pasa si hay un panic en una goroutine: el log del panic llega al buffer
   antes o después de que el proceso empiece a apagarse?
5. Propón dos mejoras concretas que mitigan los problemas 1 y 3.

**Pista:** El buffer asíncrono es una optimización de throughput (no bloquear las goroutines
por I/O de logging), pero introduce el riesgo de perder logs en un crash.
Para sistemas donde los logs son críticos (auditoría, seguridad), el log síncrono
es más seguro aunque más lento. El compromiso estándar: buffer pequeño (1-10ms de datos)
con flush en panic usando `defer`. Para el problema 3: bounded buffer con política
de descarte (drop oldest) cuando está lleno.

---

### Ejercicio 18.2.4 — Sampling de logs en sistemas de alta concurrencia

**Tipo: Diseñar/instrumentar**

Un sistema que procesa 100,000 requests/segundo no puede loguear cada request
en nivel INFO — sería ~1GB/minuto de logs. El sampling permite reducir el volumen
manteniendo la visibilidad:

```go
type SampledLogger struct {
    logger  *slog.Logger
    rate    float64 // 0.01 = loguear el 1% de los eventos
    counter atomic.Uint64
}

func (sl *SampledLogger) InfoSampled(ctx context.Context, msg string, args ...any) {
    n := sl.counter.Add(1)
    // Loguear cada 1/rate-ésimo evento:
    if n % uint64(1.0/sl.rate) == 0 {
        sl.logger.InfoContext(ctx, msg, args...)
    }
}
```

**Restricciones:** Implementar un logger con sampling que:
- Siempre loguea errores (sampling = 1.0)
- Loguea el 1% de los requests exitosos
- Loguea el 100% de requests que superan el p99 de latencia
- Incluye en cada log el "sample rate" para que los dashboards puedan
  extrapolat el volumen real (`count * sample_rate`)

**Pista:** El sampling determinista (basado en un contador) es reproducible
pero puede perderse patrones periódicos. El sampling aleatorio (`rand.Float64() < rate`)
es más representativo pero con variance alta para tasas bajas.
Un tercer enfoque: "head-based sampling" donde el sampling se decide al principio
del request y se propaga a todas las goroutines vía contexto — toda la request
se loguea o no se loguea, no solo algunos eventos.

---

### Ejercicio 18.2.5 — Leer: encontrar el bug con logs estructurados

**Tipo: Diagnosticar**

El siguiente sistema de caché tiene un bug de concurrencia. Usando solo estos logs
(sin ver el código), determina cuál es el bug:

```json
{"time":"12:00:00.001","op":"get","key":"session:alice","result":"miss","goroutine":42}
{"time":"12:00:00.002","op":"fetch_db","key":"session:alice","goroutine":42}
{"time":"12:00:00.003","op":"get","key":"session:alice","result":"miss","goroutine":87}
{"time":"12:00:00.004","op":"fetch_db","key":"session:alice","goroutine":87}
{"time":"12:00:00.103","op":"set","key":"session:alice","ttl":3600,"goroutine":42}
{"time":"12:00:00.104","op":"set","key":"session:alice","ttl":3600,"goroutine":87}
{"time":"12:00:00.200","op":"get","key":"session:alice","result":"hit","goroutine":99}
{"time":"12:00:00.201","op":"get","key":"session:alice","result":"hit","goroutine":100}
```

**Preguntas:**

1. ¿Qué patrón de concurrencia muestra esta secuencia de logs?
2. ¿Cuál es el nombre técnico de este problema?
3. ¿Cuántos queries innecesarios a la DB se hicieron?
4. ¿Cómo se resuelve sin usar un lock global que serialice todos los cache get?
5. ¿Qué log adicional agregarías para detectar este patrón automáticamente
   y enviar una alerta?

**Pista:** Goroutines 42 y 87 ambas detectaron un cache miss para la misma key
y ambas fueron a la DB. Es el "thundering herd" o "cache stampede" — múltiples
requests concurrentes para la misma key ausente generan múltiples queries a la DB.
La solución estándar es "single-flight" o "request coalescing": la primera goroutine
que detecta el miss "registra" que está fetching, y las siguientes esperan el resultado
de la primera en lugar de ir a la DB también. En Go: `golang.org/x/sync/singleflight`.

---

## Sección 18.3 — Profiling: Flame Graphs y pprof

### Ejercicio 18.3.1 — Leer: interpretar un flame graph de CPU

**Tipo: Leer/diagnosticar**

Un flame graph de CPU muestra el tiempo que el programa pasa en cada función.
El eje X es tiempo (más ancho = más tiempo), el eje Y es el call stack (más alto = más profundo).

Dado este flame graph (descrito en texto — en práctica lo verías en el browser):

```
main.main (100%)
├── main.ServeHTTP (92%)
│   ├── main.handleRequest (90%)
│   │   ├── main.fetchFromCache (45%)
│   │   │   ├── sync.(*RWMutex).RLock (38%)  ← !!
│   │   │   │   └── runtime.lock (38%)
│   │   │   └── main.deserialize (7%)
│   │   ├── main.fetchFromDB (40%)
│   │   │   ├── database/sql.(*DB).QueryContext (35%)
│   │   │   └── main.scanRows (5%)
│   │   └── main.buildResponse (5%)
│   └── encoding/json.Marshal (2%)
└── runtime.gcBgMarkWorker (8%)
```

**Preguntas:**

1. ¿Dónde está el cuello de botella principal?
2. ¿Por qué `fetchFromCache` que debería ser "rápido" consume el 45% del tiempo?
3. El GC consume el 8% del tiempo. ¿Es preocupante en este contexto?
4. Si quisieras reducir la latencia en un 30%, ¿en qué función pondrías el esfuerzo?
5. ¿Qué cambio de una línea en el código podría reducir el 38% de `RWMutex.RLock`?

**Pista:** `RWMutex.RLock` en el 38% del tiempo total indica contención severa
en el lock de lectura del caché. Esto ocurre cuando hay muchos readers concurrentes
o cuando el lock tiene secciones críticas largas. Una solución: sharding del caché
(dividir en N mapas con N locks — como el `ConcurrentHashMap` de Java). Otra solución:
cambiar a un caché concurrente sin locks para operaciones de lectura (usando atomics
y snapshots inmutables).

---

### Ejercicio 18.3.2 — Instrumentar: activar pprof en un servidor Go

**Tipo: Instrumentar**

`net/http/pprof` expone endpoints de profiling que pueden conectarse en producción
sin reiniciar el proceso:

```go
import _ "net/http/pprof"  // registra los handlers automáticamente

// Los siguientes endpoints están disponibles:
// GET /debug/pprof/            — índice
// GET /debug/pprof/goroutine   — dump de todas las goroutines
// GET /debug/pprof/heap        — heap profile
// GET /debug/pprof/profile     — CPU profile (30s por defecto)
// GET /debug/pprof/trace       — execution trace
```

**Restricciones:** Configura pprof en el servidor del Ejercicio 18.1.2 con estas
restricciones de seguridad:

1. El endpoint `/debug/pprof/` solo debe ser accesible desde la red interna (127.0.0.1
   o la subnet del cluster), no desde internet
2. El CPU profile no debe poder tomarse por más de 60 segundos (para evitar overhead prolongado)
3. Añadir autenticación básica con un token secreto leído de una variable de entorno

**Pista:** La forma más simple de restringir pprof es exponerlo en un servidor HTTP
separado en un puerto diferente (ej: 6060) que no está expuesto al load balancer.
En Kubernetes, esto se puede controlar con NetworkPolicy. Para el límite de 60s,
envolver el handler de `/profile` para leer el parámetro `seconds` y limitar.

---

### Ejercicio 18.3.3 — Leer: goroutine dump en producción

**Tipo: Leer/diagnosticar**

Este es un extracto de un goroutine dump (`/debug/pprof/goroutine?debug=2`)
de un servidor en producción que tiene latencia alta y CPU al 2%:

```
goroutine 1 [running]:
main.main()
    /app/main.go:45

goroutine 18 [select]:
main.(*Server).run(0xc000120000)
    /app/server.go:87

goroutine 19 [chan receive]:
main.(*WorkerPool).worker(0xc000132000)
    /app/pool.go:134
goroutine 20 [chan receive]:
main.(*WorkerPool).worker(0xc000132001)
    /app/pool.go:134
[... 14 goroutines más en "chan receive" ...]

goroutine 34 [semacquire, 312 minutes]:
sync.runtime_SemacquireMutex(0xc000145020, 0x0, 0x1)
    runtime/sema.go:77
sync.(*Mutex).Lock(...)
    sync/mutex.go:102
main.(*Cache).Set(0xc000145000, ...)
    /app/cache.go:67
main.handleRequest(...)
    /app/handler.go:43

goroutine 35 [semacquire, 312 minutes]:
sync.runtime_SemacquireMutex(0xc000147040, 0x0, 0x1)
    runtime/sema.go:77
sync.(*Mutex).Lock(...)
    sync/mutex.go:102
main.(*Cache).Get(0xc000145000, ...)
    /app/cache.go:45
main.handleRequest(...)
    /app/handler.go:38

[... 847 goroutines más en "semacquire", todas esperando Cache.Get o Cache.Set ...]
```

**Preguntas:**

1. ¿Cuántas goroutines están esperando el mismo lock?
2. ¿Cuánto tiempo llevan esperando? ¿Es posible eso?
3. El CPU está al 2% pero 847 goroutines están bloqueadas. ¿Qué está procesando ese 2%?
4. ¿Cuál es la causa raíz del problema?
5. Propón tres soluciones de complejidad creciente para este problema.

**Pista:** "312 minutes" en semacquire significa que estas goroutines llevan 5+ horas
esperando el lock. Esto es un deadlock o livelock donde ninguna goroutine puede
progresar porque la que tiene el lock también está bloqueada en otro lock.
El CPU al 2% es el scheduler de Go verificando si alguna goroutine puede progresar
y descubriendo que no. Para encontrar la goroutine que *tiene* el lock, busca
en el dump la goroutine que está en `cache.go:67` o `cache.go:45` pero en estado
`running` o `syscall` en lugar de `semacquire`.

---

### Ejercicio 18.3.4 — Instrumentar: heap profiling para detectar memory leaks

**Tipo: Instrumentar**

```go
// Obtener el heap profile actual:
f, _ := os.Create("heap.prof")
pprof.WriteHeapProfile(f)
f.Close()

// Analizar:
// go tool pprof heap.prof
// (pprof) top10      — top 10 allocadores por memoria retenida
// (pprof) list main  — ver líneas exactas del código
// (pprof) web        — abrir flame graph en el browser
```

**Restricciones:** Dado el siguiente código con un memory leak, usa heap profiling
para encontrar la línea exacta del leak:

```go
var cache = make(map[string][]byte)
var mu sync.Mutex

func handleRequest(key string, data []byte) {
    mu.Lock()
    cache[key] = data  // ← ¿hay leak aquí?
    mu.Unlock()

    // procesar data...
    result := process(data)
    sendResponse(result)

    // limpiar caché antigua...
    go func() {
        time.Sleep(5 * time.Minute)
        mu.Lock()
        // bug sutil: el key podría haber cambiado
        if _, existe := cache[key]; existe {
            delete(cache, key)
        }
        mu.Unlock()
    }()
}
```

**Preguntas:**

1. ¿Hay un memory leak en este código? Descríbelo.
2. ¿Qué mostraría el heap profile después de 1 hora con 1000 requests/minuto?
3. ¿Cómo confirmarías con el heap profile que encontraste el leak correcto?
4. Corrige el bug.
5. ¿Qué otra estructura de datos sería más adecuada que `map[string][]byte`
   para un caché con TTL?

**Pista:** El leak no es inmediato — la goroutine de limpieza funciona correctamente.
El leak ocurre si el key cambia entre cuando la goroutine de limpieza se crea y cuando
se ejecuta. Pero en este código, el key *no* cambia (los strings son inmutables en Go).
El leak real es diferente: las goroutines de limpieza se acumulan. Si se crean
1000 goroutines/minuto y cada una duerme 5 minutos, en cualquier momento hay
5000 goroutines activas solo para limpieza. El heap profile mostraría que la pila
de esas goroutines consume memoria (no el mapa).

---

### Ejercicio 18.3.5 — Leer: execution trace de Go

**Tipo: Leer/diagnosticar**

El execution trace de Go (`go tool trace`) muestra exactamente qué hace cada goroutine
en cada microsegundo: cuándo corre, cuándo espera, cuándo hace syscall, cuándo el GC pausa.

Dado este análisis de un execution trace (extraído del output de `go tool trace`):

```
Goroutine analysis:
  goroutine 18: "main.worker"
    Execution time:    234ms (23.4% of total)
    Network wait:      756ms (75.6%)
    Sync block:        8ms   (0.8%)
    Scheduler wait:    2ms   (0.2%)

  goroutine 19: "main.worker"
    Execution time:    198ms (19.8%)
    Network wait:      795ms (79.5%)
    Sync block:        5ms   (0.5%)
    Scheduler wait:    2ms   (0.2%)

GC events:
  GC #1: start=100ms, duration=2.3ms, heap before=45MB, heap after=23MB
  GC #2: start=340ms, duration=2.8ms, heap before=48MB, heap after=24MB
  GC #3: start=580ms, duration=2.1ms, heap before=44MB, heap after=22MB

Processor utilization:
  P0: 45% (running), 55% (idle)
  P1: 42% (running), 58% (idle)
  P2: 8%  (running), 92% (idle)
  P3: 7%  (running), 93% (idle)
```

**Preguntas:**

1. ¿Cuál es el cuello de botella de estos workers?
2. ¿Están bien configurados los 4 procesadores (`GOMAXPROCS=4`)?
3. ¿Qué GC stop-the-world ocurre aquí? (El GC de Go es concurrent — ¿hay pausa?)
4. El heap antes del GC es ~45MB y después ~22MB — ¿es saludable esta ratio?
5. Dado el 75% de espera de red por worker, ¿cuántos workers necesitarías
   para saturar los 4 procesadores?

**Pista:** Si los workers esperan red el 75% del tiempo, solo están "corriendo" el 25%.
Para saturar 4 procesadores al 100%, necesitarías que en cualquier momento haya
siempre 4 workers en la fase de ejecución. Si cada worker ejecuta 25% del tiempo,
necesitas `4 / 0.25 = 16 workers` para mantener los 4 procesadores ocupados.
El GC de Go es concurrent (no stop-the-world para la mayoría del trabajo), pero
sí tiene una pausa corta para STW mark termination — generalmente < 1ms.

---

## Sección 18.4 — Distributed Tracing: Seguir una Request

### Ejercicio 18.4.1 — Leer: interpretar un trace de Jaeger

**Tipo: Leer/diagnosticar**

Este trace (representado en texto) muestra una request que tardó 850ms:

```
Request: POST /checkout [850ms total]
│
├─ [0ms] validate_cart [12ms]
│   └─ [2ms] db.query (SELECT items WHERE cart_id=...) [10ms]
│
├─ [12ms] check_inventory [320ms]  ← !!
│   ├─ [12ms] http.GET inventory-service/item/1 [95ms]
│   ├─ [12ms] http.GET inventory-service/item/2 [97ms]  ← ¿por qué igual que /item/1?
│   ├─ [12ms] http.GET inventory-service/item/3 [98ms]
│   └─ [12ms] http.GET inventory-service/item/4 [100ms]
│
├─ [332ms] process_payment [280ms]
│   ├─ [332ms] http.POST payment-gateway/charge [275ms]
│   └─ [607ms] db.insert (INSERT INTO orders...) [5ms]
│
└─ [612ms] send_confirmation_email [238ms]  ← !!
    └─ [612ms] http.POST email-service/send [238ms]
```

**Preguntas:**

1. Identifica dos problemas de rendimiento distintos en este trace.
2. ¿Por qué las 4 llamadas al inventory-service empiezan todas en `[12ms]`
   pero terminan escalonadas (95ms, 97ms, 98ms, 100ms)?
3. ¿Qué optimización haría mayor impacto en la latencia total?
4. ¿Es crítico el email de confirmación en el path de la request? ¿Qué harías?
5. Si el p99 del payment-gateway es 275ms y el SLO del checkout es 500ms,
   ¿cuánto tiempo queda para el resto del procesamiento?

**Pista:** Las 4 llamadas al inventory-service que empiezan al mismo tiempo son
llamadas paralelas (correctamente diseñadas) — el tiempo total es el máximo, no la suma.
El email de confirmación (`238ms`) probablemente no necesita completarse antes de
responder al usuario — se puede hacer async (fire-and-forget con una cola).
Moviendo el email a async, el tiempo total sería `332 + 280 = 612ms` → `332 + 280 - 238 = ~374ms`.

---

### Ejercicio 18.4.2 — Instrumentar: añadir tracing a un sistema existente

**Tipo: Instrumentar**

Instrumenta el Worker Pool del Ejercicio 18.1.2 con OpenTelemetry para que
cada tarea tenga un span hijo del span del request original:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func handleRequest(ctx context.Context, req Request) Response {
    tracer := otel.Tracer("mi-servicio")
    ctx, span := tracer.Start(ctx, "handleRequest")
    defer span.End()

    // El Worker Pool debe propagar el contexto a cada tarea:
    pool.Submit(ctx, func() {
        _, taskSpan := tracer.Start(ctx, "procesarTarea")
        defer taskSpan.End()
        // ...
    })
}
```

**Restricciones:**

- El contexto (con el trace ID) debe propagarse del request a cada tarea del pool
- Cada tarea debe tener su propio span como hijo del span del request
- El span debe incluir atributos: `task.id`, `task.type`, `task.queue_wait_ms`
- Si la tarea falla, el span debe marcarse con `StatusCode = Error` y el mensaje de error

**Pista:** El `task.queue_wait_ms` requiere dos timestamps: cuando la tarea se encoló
y cuando empezó a ejecutarse. El contexto se puede pasar como parte de la tarea
(`type Task struct { ctx context.Context; fn func() }`). El span del request no debe
terminar (`defer span.End()`) hasta que todas las goroutines hijas terminen —
usar un WaitGroup o `errgroup`.

---

### Ejercicio 18.4.3 — Leer: encontrar el bottleneck con tracing

**Tipo: Diagnosticar**

Un servicio de e-commerce tiene este trace promedio (p50):

```
GET /product-page [85ms]
├─ get_product [15ms]
├─ get_reviews [12ms]      ← en paralelo
├─ get_recommendations [18ms] ← en paralelo
└─ get_inventory [14ms]    ← en paralelo

Y este trace en p99:

GET /product-page [1850ms]  ← !!
├─ get_product [15ms]
├─ get_reviews [1820ms]      ← !!
├─ get_recommendations [18ms]
└─ get_inventory [14ms]
```

**Preguntas:**

1. ¿Por qué la latencia total en p99 es determinada casi completamente por `get_reviews`?
2. El `get_reviews` tiene p50=12ms pero p99=1820ms. ¿Qué tipo de distribución tiene?
3. Si `get_reviews` tarda más de 200ms, ¿qué harías para no bloquear la página completa?
4. Propón la modificación mínima al código que implementa esta solución.
5. ¿Qué métrica nueva añadirías para monitorear que esta solución funciona?

**Pista:** Una distribución bimodal (12ms la mayoría, 1820ms el 1%) sugiere que
el servicio de reviews tiene dos comportamientos: cache hit (rápido) y cache miss
con operación pesada (lento). La solución estándar para esto es "timeout con fallback":
si `get_reviews` no responde en 200ms, retornar la página sin reviews (degradación elegante).
En código: `ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond); defer cancel()`.

---

### Ejercicio 18.4.4 — Context propagation en sistemas multi-lenguaje

**Tipo: Instrumentar**

Un sistema tiene tres servicios en lenguajes distintos:
- Servicio A (Go) recibe el request HTTP
- Servicio B (Python/FastAPI) procesa la imagen
- Servicio C (Java/Spring) persiste el resultado

OpenTelemetry usa el header `traceparent` (W3C Trace Context) para propagar
el trace ID entre servicios:

```
traceparent: 00-{trace-id}-{span-id}-{flags}
Ejemplo: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

**Restricciones:** Implementa la propagación del trace context en los tres servicios.
Cuando el trace llega a Jaeger, debe mostrar la jerarquía completa:
```
Request A → Span A
              └─ HTTP call to B → Span B
                                    └─ HTTP call to C → Span C
```

**Pista:** En Go, `otel.GetTextMapPropagator().Inject(ctx, header)` añade el header
`traceparent` automáticamente. En Python con OpenTelemetry:
`TraceContextTextMapPropagator().inject(headers)`. En Java con Spring Boot:
`@Observability` o el auto-instrumentation agent de OpenTelemetry lo hace sin código.
La trampa: si uno de los servicios no propaga el header, el trace se "rompe" en ese punto.

---

### Ejercicio 18.4.5 — Sampling estratégico de traces

**Tipo: Diseñar/instrumentar**

Tracear el 100% de las requests de un sistema de 100K req/s generaría
terabytes de datos por día. El sampling reduce esto a un volumen manejable.

```go
// Head-based sampling (decide al inicio):
sampler := sdktrace.ParentBased(
    sdktrace.TraceIDRatioBased(0.01),  // 1% de todos los traces
)

// Pero esto pierde los traces más interesantes (los lentos, los errores).
// Tail-based sampling (decide al final, cuando ya sabes el resultado):
// - Siempre samplear si hubo error
// - Siempre samplear si la latencia > p95
// - 1% de los traces normales
```

**Restricciones:** Implementa tail-based sampling con estas reglas:
1. Error rate: 100% de traces con errores
2. Latencia alta: 100% de traces con latencia > 500ms
3. Muestreo base: 1% del resto

El problema del tail-based sampling: la decisión se toma al final, pero el span
ya se envió al collector al principio. Diseña cómo manejar esto.

**Pista:** El tail-based sampling requiere un "buffer" donde se guardan los spans
hasta que el trace completa (o hasta un timeout). Si el trace "califica" para
ser sampleado (error o latencia alta), todos los spans del buffer se envían.
Si no califica, se descartan. Jaeger y Grafana Tempo tienen collectors con
tail-based sampling integrado. En código propio: usar un buffer por trace_id
con TTL de 30s.

---

## Sección 18.5 — Leer Dashboards: Diagnóstico en 60 Segundos

### Ejercicio 18.5.1 — Diagnóstico rápido: ¿qué está mal?

**Tipo: Diagnosticar**

Dado este snapshot de un dashboard de producción a las 3AM durante un incidente:

```
Panel 1: Requests por segundo
  [normal: 5000 rps] → [ahora: 4800 rps]  ← bajó levemente

Panel 2: Latencia p99
  [normal: 45ms] → [ahora: 8200ms]  ← !!!

Panel 3: Tasa de error
  [normal: 0.1%] → [ahora: 0.1%]  ← igual

Panel 4: CPU
  [normal: 40%] → [ahora: 38%]  ← igual

Panel 5: Memoria
  [normal: 4.2 GB] → [ahora: 4.2 GB]  ← igual

Panel 6: Goroutines activas
  [normal: 850] → [ahora: 24,300]  ← !!!

Panel 7: DB query p99
  [normal: 8ms] → [ahora: 7,900ms]  ← !!!

Panel 8: DB connections active
  [normal: 45/100] → [ahora: 100/100]  ← pool lleno
```

**Preguntas (responder en orden como si fueras el on-call):**

1. ¿Cuál es el síntoma que afecta a los usuarios?
2. ¿Cuál es la causa raíz identificable con estos datos?
3. ¿Por qué las goroutines subieron de 850 a 24,300?
4. ¿Por qué la tasa de error se mantiene en 0.1% aunque p99 subió a 8200ms?
5. ¿Cuál es la acción inmediata? ¿Y la causa subyacente a investigar después?

**Pista:** El pool de conexiones a DB está al 100% (panel 8) → las goroutines esperan
una conexión (panel 6 con 24K goroutines) → las requests tardan lo que tarda conseguir
la conexión + la query (panel 2 con 8200ms p99). La tasa de error sigue baja porque
las requests *completan* eventualmente (en 8 segundos, pero completan).
La causa subyacente: algún cambio causó que las queries tarden más (panel 7: DB p99 subió de 8ms a 7900ms).
Probablemente: un índice que falta, una query sin optimizar, o una tabla que creció.

---

### Ejercicio 18.5.2 — Correlacionar métricas para encontrar causas

**Tipo: Diagnosticar**

Estos tres eventos ocurrieron con 15 minutos de diferencia:

```
14:00: Deployment de versión 2.3.1
14:07: Latencia p99 empieza a subir (45ms → 120ms)
14:15: Alerta disparada: p99 > 200ms

Métricas adicionales durante la ventana 14:00-14:15:
  - Tasa de error: estable (0.1%)
  - CPU: subió de 40% a 68%
  - Memoria: subió de 4.2GB a 4.8GB
  - GC pause p99: subió de 2ms a 45ms
  - Goroutines: estable (~850)
  - DB p99: estable (8ms)
```

**Preguntas:**

1. ¿El deployment causó el problema? ¿Cómo lo confirmarías o refutarías?
2. El CPU subió pero las goroutines no — ¿qué tipo de cambio explicaría esto?
3. El GC pause subió de 2ms a 45ms. ¿Qué relación tiene con el aumento de p99?
4. ¿Qué herramienta usarías para confirmar que el aumento de CPU es el GC?
5. ¿Cuál es el primer paso del rollback si decides que el deployment causó el problema?

**Pista:** CPU sube + GC pause sube + memoria sube tras un deployment sugiere que
la nueva versión aloca significativamente más memoria por request (más GC pressure).
El flame graph de CPU de la nueva versión mostraría `runtime.gcBgMarkWorker` con un
porcentaje más alto. Para confirmarlo: `go tool pprof http://host/debug/pprof/heap`
y comparar con el heap profile de la versión anterior (si lo guardaste).

---

### Ejercicio 18.5.3 — El dashboard que no ayudó

**Tipo: Leer/analizar**

El siguiente dashboard existía durante el incidente del Ejercicio 18.5.1
pero no ayudó a diagnosticarlo en menos de 30 minutos:

```
Dashboard: "API Health"
  Panel 1: "Uptime" — siempre 100% (el servicio no cayó)
  Panel 2: "Total requests" — counter, siempre creciendo
  Panel 3: "Error rate last 24h" — 0.08% (promedio diario, no tiempo real)
  Panel 4: "Average latency" — 180ms (promedio, no percentiles)
  Panel 5: "Server CPU" — 38% (servidor app, no la DB)
  Panel 6: "Deploys today" — 2
```

**Preguntas:**

1. ¿Cuál es el problema con cada panel?
2. ¿Qué debería mostrar cada panel para ser útil durante un incidente?
3. Propón un dashboard de 6 paneles que habría detectado el incidente en < 2 minutos.
4. ¿Cuál es la diferencia entre un dashboard de "health" y uno de "diagnosis"?
5. ¿Qué alerta habría disparado automáticamente este incidente si el dashboard estuviera bien diseñado?

**Pista:** El "average latency" es el peor percentil para alertas: un sistema con
99% de requests en 10ms y 1% en 10 segundos tiene un promedio de ~108ms — parece bien.
La regla: siempre p99 (o mejor, p999) para latencia, nunca promedio.
El panel de "Uptime" es inútil en la práctica — un sistema que responde en 8 segundos
está "up" pero no sirve. La métrica correcta: "SLO compliance" (% del tiempo que p99 < SLO).

---

### Ejercicio 18.5.4 — Diseñar el dashboard para tu sistema

**Tipo: Diseñar**

Diseña el dashboard de producción para el Worker Pool del Ejercicio 18.1.2,
siguiendo estas reglas:

```
Regla 1: La primera fila responde "¿está bien?" en < 10 segundos.
  Tres semáforos (verde/amarillo/rojo) basados en SLOs.

Regla 2: La segunda fila identifica el cuello de botella en < 2 minutos.
  Métricas de recursos con contexto (no solo valores absolutos).

Regla 3: La tercera fila provee el detalle para el diagnóstico.
  Histogramas, rates, distribuciones.

Regla 4: Sin paneles decorativos.
  Cada panel responde una pregunta específica de diagnóstico.
```

**Restricciones:** Máximo 12 paneles totales. Cada panel debe tener:
- Nombre de la pregunta que responde ("¿está el pool saturado?")
- Métrica y query de PromQL
- Umbrales de color (verde/amarillo/rojo)

**Pista:** Los 4 golden signals de Google SRE como estructura para las 3 filas:
- Fila 1: Latency + Error rate + Traffic (síntomas de usuario)
- Fila 2: Saturation (cuello de botella)
- Fila 3: Detalle de cada componente para diagnóstico

---

### Ejercicio 18.5.5 — Leer: detectar un memory leak en el tiempo

**Tipo: Diagnosticar**

Este gráfico de métricas muestra el comportamiento de un servicio a lo largo de 7 días:

```
Día 1: Heap = 2.1 GB, GC pause p99 = 2ms,  latencia p99 = 45ms
Día 2: Heap = 2.4 GB, GC pause p99 = 3ms,  latencia p99 = 47ms
Día 3: Heap = 2.8 GB, GC pause p99 = 5ms,  latencia p99 = 52ms
Día 4: Heap = 3.3 GB, GC pause p99 = 9ms,  latencia p99 = 61ms
Día 5: Heap = 3.9 GB, GC pause p99 = 18ms, latencia p99 = 78ms
Día 6: Heap = 4.6 GB, GC pause p99 = 35ms, latencia p99 = 112ms
Día 7: Heap = 5.4 GB, GC pause p99 = 68ms, latencia p99 = 180ms
```

El servicio se reinicia el día 7 y el heap vuelve a 2.1 GB.

**Preguntas:**

1. ¿Este servicio tiene un memory leak? ¿Cómo lo sabes?
2. ¿A qué velocidad crece el leak (GB/día)?
3. ¿Cuándo habría llegado a OOM si no se reinicia? (asumiendo límite de 8 GB)
4. ¿Por qué el GC pause aumenta junto con el heap?
5. Describe los pasos exactos para diagnosticar el origen del leak usando pprof.

**Pista:** El heap crece ~0.3-0.8 GB/día de forma consistente — patrón clásico de leak.
El GC pause aumenta porque hay más objetos para escanear en cada ciclo.
Para diagnosticar: tomar dos heap profiles con 1 hora de diferencia
(`go tool pprof -diff_base heap1.prof heap2.prof`) y buscar qué tipos de objetos
crecen más. El `-diff_base` muestra el delta, no el total — mucho más útil para leaks.

---

## Sección 18.6 — Observabilidad de la Concurrencia Interna

### Ejercicio 18.6.1 — Exponer el estado interno del scheduler

**Tipo: Instrumentar**

Go expone métricas del runtime que revelan el estado del scheduler:

```go
import "runtime/metrics"

func leerMetricasRuntime() map[string]float64 {
    samples := []metrics.Sample{
        {Name: "/sched/goroutines:goroutines"},      // goroutines activas
        {Name: "/sched/latencies:seconds"},           // histograma de latencia del scheduler
        {Name: "/memory/classes/heap/objects:bytes"}, // heap en uso
        {Name: "/gc/cycles/total:gc-cycles"},         // ciclos de GC totales
        {Name: "/gc/pauses/total/other:seconds"},     // tiempo en GC pauses
    }
    metrics.Read(samples)

    resultado := map[string]float64{}
    for _, s := range samples {
        switch s.Value.Kind() {
        case metrics.KindFloat64:
            resultado[s.Name] = s.Value.Float64()
        case metrics.KindUint64:
            resultado[s.Name] = float64(s.Value.Uint64())
        }
    }
    return resultado
}
```

**Restricciones:** Exponer estas métricas del runtime en el endpoint `/metrics`
de Prometheus. Crear un panel de Grafana con:
- Goroutines activas a lo largo del tiempo
- Latencia del scheduler (tiempo que una goroutine espera antes de correr)
- Frecuencia de GC y duración de pauses

**Pista:** `/sched/latencies:seconds` es un histograma — requiere un tratamiento
especial para convertirlo a formato Prometheus. Las métricas del runtime de Go
se exponen como float64 o uint64 pero el histograma es una `metrics.Float64Histogram`.
Para exportar el histograma: iterar sus buckets y crear un `prometheus.Histogram`
con los mismos valores.

---

### Ejercicio 18.6.2 — Detectar goroutine leaks en producción

**Tipo: Instrumentar + Leer**

Un goroutine leak es cuando goroutines se crean pero nunca terminan.
El síntoma: el contador de goroutines crece indefinidamente sin bajar.

```go
// Usar goleak en tests es fácil (Cap.07).
// En producción, necesitas detectarlo con métricas:

func monitorearGoroutines(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    anterior := runtime.NumGoroutine()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            actual := runtime.NumGoroutine()
            delta := actual - anterior
            goroutinesTotal.Set(float64(actual))

            if delta > 100 {  // creció más de 100 en 30s
                log.Warn("posible goroutine leak",
                    "goroutines_now", actual,
                    "goroutines_30s_ago", anterior,
                    "delta", delta,
                )
            }
            anterior = actual
        }
    }
}
```

**Restricciones:** Para el siguiente código con un goroutine leak, detectarlo
con métricas e identificar la goroutine que no termina con el goroutine dump:

```go
func procesarConLeak(items []Item) {
    for _, item := range items {
        resultCh := make(chan Result)  // ← unbuffered channel

        go func(i Item) {
            result := procesarItem(i)
            resultCh <- result  // ← si nadie lee resultCh, esta goroutine queda bloqueada
        }(item)

        // Bug: si procesarConLeak retorna antes de leer resultCh
        // (por ejemplo, por timeout), la goroutine queda colgada
        select {
        case result := <-resultCh:
            log.Info("procesado", "result", result)
        case <-time.After(100 * time.Millisecond):
            log.Warn("timeout, continuando")
            // ← la goroutine del item queda bloqueada aquí para siempre
        }
    }
}
```

**Pista:** El fix es hacer el channel buffered (`make(chan Result, 1)`) o
usar `context.WithCancel` y pasarlo a la goroutine para que pueda terminar.
Con channel buffered de tamaño 1, la goroutine puede escribir el resultado
aunque nadie esté leyendo — y luego terminar. El resultado se descarta,
pero la goroutine ya no queda colgada.

---

### Ejercicio 18.6.3 — Medir la contención de locks

**Tipo: Instrumentar**

La contención en un lock es cuando múltiples goroutines compiten por él simultáneamente.
El overhead no es el lock en sí (~25ns) sino la espera (~microsegundos a milisegundos):

```go
type MutexInstrumentado struct {
    mu            sync.Mutex
    contenciones  atomic.Int64  // veces que hubo que esperar
    esperaTotal   atomic.Int64  // nanosegundos totales esperando
}

func (m *MutexInstrumentado) Lock() {
    inicio := time.Now()
    // Intentar adquirir sin bloquear:
    if !m.mu.TryLock() {
        // No se pudo — hay contención:
        m.contenciones.Add(1)
        m.mu.Lock()  // ahora sí bloquear
        espera := time.Since(inicio).Nanoseconds()
        m.esperaTotal.Add(espera)
    }
}

func (m *MutexInstrumentado) Unlock() {
    m.mu.Unlock()
}
```

**Restricciones:** Implementa el `MutexInstrumentado` y añade sus métricas
al endpoint Prometheus. Exponer:
- `mutex_contentions_total` counter
- `mutex_wait_duration_seconds` histograma

Verificar que con alta contención (16 goroutines compitiendo por el mismo lock),
las métricas muestran el problema claramente.

**Pista:** `sync.Mutex.TryLock()` existe desde Go 1.18. Retorna `true` si el lock
se adquirió sin esperar, `false` si ya estaba tomado. El overhead de `TryLock`
seguido de `Lock` cuando hay contención es mínimo comparado con el tiempo de espera.
Para identificar *qué* locks tienen más contención: usar el mutex name como label
en las métricas (`mutex_wait_duration_seconds{name="cache.lock"}`).

---

### Ejercicio 18.6.4 — Tracing de operaciones internas del pool

**Tipo: Instrumentar**

Añade tracing interno al Worker Pool para que sea visible en qué estado
pasa el tiempo cada tarea:

```
Estados de una tarea:
  1. Encolada     (desde submit hasta que un worker la toma)
  2. En ejecución (desde que el worker la toma hasta que termina)
  3. Completada   (con o sin error)

El trace debería mostrar:
  Span "task.queued"     [0ms → 45ms]    ← esperó 45ms en la cola
  Span "task.executing"  [45ms → 123ms]  ← tardó 78ms en ejecutar
```

**Restricciones:** Instrumentar el pool con OpenTelemetry para que cada tarea
genere estos dos spans. El span `task.queued` debe incluir el atributo `queue_depth`
(cuántos items había en la cola cuando se encoló la tarea).

**Pista:** El timestamp de "encolado" se captura en `Submit()`. El timestamp de "inicio de ejecución"
se captura al inicio de la goroutine worker, justo antes de ejecutar la tarea.
Para el `queue_depth`: `len(pool.tasks)` en el momento de `Submit()`.
El contexto del trace (para que el span sea hijo del span correcto) debe pasarse
como parte de la tarea: `type Task struct { ctx context.Context; fn func() }`.

---

### Ejercicio 18.6.5 — Dashboard de concurrencia interna

**Tipo: Diseñar**

Diseña un dashboard de "concurrencia interna" — no las métricas de usuario
(latencia de requests) sino el estado interno del sistema concurrente:

```
Preguntas que debe responder este dashboard:

1. ¿Cuántas goroutines hay ahora y cómo ha evolucionado?
2. ¿Hay contención en los locks principales?
3. ¿El GC está impactando la latencia?
4. ¿Las goroutines del pool están activas o esperando?
5. ¿Hay algún goroutine leak en progreso?
```

**Restricciones:** El dashboard debe tener exactamente 5 paneles, uno por pregunta.
Para cada panel: query de PromQL, tipo de visualización (time series, gauge, heatmap),
y umbral de alerta.

**Pista:** Para el panel de goroutine leak: graficar `rate(goroutines_total[5m])`
en lugar del valor absoluto — si el rate es consistentemente positivo durante
más de 15 minutos, hay un leak. El valor absoluto puede ser alto pero estable
(que es normal) o alto y creciendo (que es un leak).

---

## Sección 18.7 — Construir el Sistema de Observabilidad Mínimo Viable

### Ejercicio 18.7.1 — Stack de observabilidad local con docker-compose

**Tipo: Instrumentar/construir**

Levanta un stack de observabilidad completo en local para desarrollar y testear:

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
    volumes: ["./grafana/dashboards:/var/lib/grafana/dashboards"]

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: ["16686:16686", "4318:4318"]

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]
```

**Restricciones:** El stack debe arrancar con `docker-compose up` y estar listo
en < 60 segundos. El Worker Pool instrumentado del Ejercicio 18.1.2 debe enviar
métricas, logs, y traces a este stack automáticamente.

**Pista:** Prometheus necesita un `scrape_config` que apunte a `host.docker.internal:<puerto>`
para scrappear el servicio Go corriendo fuera de Docker. Para Jaeger,
usar el protocolo OTLP/HTTP en el puerto 4318. Para Loki, usar el driver
de Docker (`docker logs` → Loki) o el SDK de logs de OpenTelemetry.

---

### Ejercicio 18.7.2 — Alertas automáticas con Alertmanager

**Tipo: Construir**

Añade Alertmanager al stack del ejercicio anterior y configura alertas
que envían notificaciones a Slack (o email simulado):

```yaml
# alerting_rules.yml
groups:
  - name: worker_pool
    rules:
      - alert: WorkerPoolSaturado
        expr: worker_pool_queue_size > 500
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Worker pool saturado"
          description: "{{ $value }} tareas en la cola durante más de 2 minutos"

      - alert: LatenciaAltaP99
        expr: |
          histogram_quantile(0.99,
            rate(worker_pool_task_duration_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: critical
        annotations:
          description: "p99 = {{ $value }}s"
```

**Restricciones:** Las alertas deben incluir el runbook URL en las anotaciones.
El `for: 2m` evita alertas por spikes transitorios — verificar que la alerta
no dispara durante spikes de menos de 2 minutos.

**Pista:** El campo `for:` en Prometheus espera que la condición sea verdadera
continuamente durante ese tiempo antes de disparar la alerta. Sin `for`,
cualquier spike (incluso de un scrape) dispara la alerta. Con `for: 2m`,
solo una degradación sostenida dispara la alerta. Para testear alertas:
`amtool alert add alertname=WorkerPoolSaturado` simula la alerta.

---

### Ejercicio 18.7.3 — Observabilidad en tests de integración

**Tipo: Instrumentar**

Los tests de integración deberían verificar las métricas, no solo el comportamiento:

```go
func TestWorkerPool_SaturacionMetricas(t *testing.T) {
    pool := NewWorkerPoolInstrumentado(4)

    // Saturar el pool:
    for i := 0; i < 1000; i++ {
        pool.Submit(Task{Duration: 100 * time.Millisecond})
    }

    time.Sleep(200 * time.Millisecond)

    // Verificar que las métricas reflejan la saturación:
    queueSize := testutil.ToFloat64(workerPoolQueueSize)
    assert.Greater(t, queueSize, float64(100),
        "la cola debe tener más de 100 items bajo saturación")

    workerActive := testutil.ToFloat64(workerPoolWorkersActive)
    assert.Equal(t, float64(4), workerActive,
        "todos los workers deben estar activos bajo saturación")
}
```

**Restricciones:** Implementa 3 tests que verifican métricas bajo condiciones específicas:
1. Bajo carga normal: p99 < 100ms, queue < 10
2. Bajo saturación: todos los workers activos, queue creciendo
3. Después del shutdown: 0 goroutines activas, 0 tareas en queue

**Pista:** `github.com/prometheus/client_golang/prometheus/testutil` tiene
`ToFloat64(collector)` que lee el valor actual de un gauge o counter.
Para histogramas, usar `CollectAndCompare` que compara el output completo.
Los tests de métricas son especialmente valiosos para asegurarse de que
la instrumentación no se rompe en un refactor.

---

### Ejercicio 18.7.4 — SLO tracking automatizado

**Tipo: Construir**

Implementa el tracking de error budget del Cap.17 §17.6.3 con Prometheus:

```
SLO: p99 de latencia < 200ms durante el 99.9% del tiempo en una ventana de 30 días

Error budget: 0.1% del tiempo = 0.1% × 30 días × 24h × 60min = 43.2 minutos
Burn rate actual = tiempo_violando_slo / tiempo_total

Si burn rate > 1.0: el SLO se violará antes de que acabe el mes
Si burn rate > 14.4: el SLO se violará en < 2 días (alerta crítica)
```

```yaml
# Regla de recording para el burn rate:
- record: job:slo_error_budget_burn_rate:ratio_rate5m
  expr: |
    sum(rate(request_duration_seconds_bucket{le="0.2"}[5m]))
    /
    sum(rate(request_duration_seconds_count[5m]))
    < bool 0.999  # 1 cuando viola el SLO, 0 cuando no
```

**Restricciones:** Implementar las reglas de recording y alerting para:
- Alerta "warning" cuando burn rate > 1.0 durante 1 hora
- Alerta "critical" cuando burn rate > 14.4 durante 5 minutos
- Panel de Grafana que muestra el error budget consumido vs disponible

**Pista:** Las "multiwindow, multi-burn-rate" alerts de Google SRE usan ventanas
de 1h y 6h para detectar burns rápidos y lentos. Una alerta solo en ventana de 1h
puede tener muchos falsos positivos. La combinación de `rate[1h]` y `rate[6h]`
siendo altas simultáneamente reduce los falsos positivos.

---

### Ejercicio 18.7.5 — Checklist de observabilidad para code review

**Tipo: Proceso/checklist**

Crea el checklist de observabilidad que usarías en un code review
cuando un PR introduce un nuevo componente concurrente:

```markdown
# Checklist de observabilidad — Code Review

## Métricas
- [ ] ¿Tiene métricas de latencia con histograma (no promedio)?
- [ ] ¿Tiene counter de errores con label por tipo de error?
- [ ] ¿Los recursos bounded (queues, pools) tienen gauge de tamaño actual?
- [ ] ¿Las métricas tienen nombres consistentes con el resto del sistema?

## Logs
- [ ] ¿Los errores se loguean con contexto suficiente para reproducir?
- [ ] ¿Los logs incluyen el request_id o correlation_id?
- [ ] ¿El nivel de log es apropiado? (no loguear DEBUG en producción)
- [ ] ¿Hay logging en el hot path que podría impactar el rendimiento?

## Tracing
- [ ] ¿Las operaciones que crean goroutines propagan el contexto?
- [ ] ¿Los spans tienen nombres descriptivos y atributos relevantes?
- [ ] ¿Los errores se registran en el span con SetStatus(Error)?

## Alertas
- [ ] ¿Existe alerta para la métrica de error del nuevo componente?
- [ ] ¿El runbook está actualizado con el nuevo componente?
```

**Restricciones:** Para cada ítem del checklist, añadir un ejemplo de
código correcto y uno incorrecto. El checklist debe poder completarse
en < 10 minutos revisando un PR de tamaño mediano (< 500 líneas).

**Pista:** El checklist más valioso es el que detecta los problemas que
realmente ocurren en producción, no todos los problemas posibles.
Empieza con los 5-7 problemas más comunes que has visto (o leído sobre)
y construye el checklist a partir de esos. Los checklists demasiado largos
no se usan.

---

## Resumen del capítulo

**Los cuatro errores más comunes de observabilidad en sistemas concurrentes:**

```
1. Medir promedios en lugar de percentiles
   → El promedio esconde el tail de latencia donde viven los problemas

2. No propagar el correlation ID a las goroutines hijas
   → Los logs de la goroutine hija son inútiles sin contexto del request original

3. No samplear los logs
   → 100K req/s × 1KB/log = 6 GB/minuto de logs, inmanejable y caro

4. Instrumentar dentro de secciones críticas
   → El overhead del logging puede cambiar el timing y hacer el bug irreproducible
```

**Las tres preguntas que debe responder tu sistema de observabilidad:**

```
En 10 segundos: "¿está el sistema sano para los usuarios?"
En 2 minutos:   "¿qué componente es el cuello de botella?"
En 10 minutos:  "¿cuál es la causa raíz y cómo la soluciono?"
```

Si tu sistema no puede responder las tres en esos tiempos, hay trabajo de observabilidad pendiente.
