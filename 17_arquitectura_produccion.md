# Guía de Ejercicios — Arquitectura de Producción: Sistemas Concurrentes que Duran

> Implementar en el lenguaje que más uses o el que mejor aplique al contexto.
> Los ejercicios de esta sección son más grandes — cada uno es un sistema mini completo.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## El problema que cierra el repositorio

Todo lo anterior enseñó las primitivas: goroutines, mutexes, canales, CAS,
lock-free data structures, hardware cache effects, async/await en cuatro lenguajes.

Pero en producción, los sistemas fallan de formas que las primitivas no predicen:

```
Incidente real (compuesto de varios):
  Sistema de pagos. 3AM. Black Friday.
  
  Síntoma: latencia p99 sube de 50ms a 8000ms. El sistema "parece" funcionar.
  
  Causa raíz (reconstruida después):
    1. Un spike de tráfico saturó el pool de conexiones a BD
    2. Las requests empezaron a esperar conexiones
    3. El timeout de las requests era 10s — lo suficiente para acumular
    4. Los circuit breakers tenían el threshold demasiado alto — no abrieron
    5. Las métricas de "tasa de error" eran 0% — todos los requests "completaban"
       (en 8 segundos, pero completaban)
    6. El equipo de on-call tardó 45 minutos en diagnosticar

  Lo que hubiera ayudado:
    - Connection pool con backpressure que rechazara antes (no en timeout)
    - Circuit breaker calibrado para latencia, no solo errores
    - Métricas de p99 en tiempo real (no solo tasa de error)
    - Runbook con los pasos de diagnóstico
```

Este capítulo construye los componentes que transforman "código concurrente correcto"
en "sistema concurrente operable en producción".

---

## Tabla de contenidos

- [Sección 17.1 — Observabilidad: ver lo que no puedes depurar](#sección-171--observabilidad-ver-lo-que-no-puedes-depurar)
- [Sección 17.2 — Resilience patterns: fallar bien](#sección-172--resilience-patterns-fallar-bien)
- [Sección 17.3 — Backpressure end-to-end](#sección-173--backpressure-end-to-end)
- [Sección 17.4 — Testing en producción: chaos y load testing](#sección-174--testing-en-producción-chaos-y-load-testing)
- [Sección 17.5 — Capacity planning: dimensionar antes de saturar](#sección-175--capacity-planning-dimensionar-antes-de-saturar)
- [Sección 17.6 — Runbooks: operar sistemas concurrentes](#sección-176--runbooks-operar-sistemas-concurrentes)
- [Sección 17.7 — Diseño integral: un sistema real de principio a fin](#sección-177--diseño-integral-un-sistema-real-de-principio-a-fin)

---

## Sección 17.1 — Observabilidad: Ver lo que No Puedes Depurar

La regla de la observabilidad: nunca adjuntar un debugger a un sistema en producción
bajo carga. Lo que necesitas son datos que te permitan reconstruir qué pasó.

**Los tres pilares:**

```
Métricas (qué está pasando ahora):
  Counters: requests totales, errores totales, bytes transferidos
  Gauges: goroutines activas, conexiones abiertas, tamaño de queue
  Histogramas: distribución de latencia (p50, p99, p999)
  
  Herramientas: Prometheus, StatsD, DataDog

Logs (qué pasó y por qué):
  Structured logging (JSON): searchable, filtrable
  Correlation ID: rastrear una request a través de múltiples servicios
  Sampling para logs de alta frecuencia (no loguear cada request)
  
  Herramientas: Loki, CloudWatch, Splunk

Traces (cómo fluyó una request específica):
  Distributed tracing: ver el tiempo en cada servicio
  Span: la unidad de tracing (inicio + duración + metadata)
  Context propagation: pasar el trace ID entre servicios
  
  Herramientas: Jaeger, Zipkin, DataDog APM, OpenTelemetry
```

---

### Ejercicio 17.1.1 — Instrumentar un Worker Pool con métricas en tiempo real

**Enunciado:** Instrumenta el Worker Pool del Cap.03 para que en producción puedas
responder estas preguntas en menos de 30 segundos:

1. ¿Cuántas tareas están en la queue ahora mismo?
2. ¿Cuántos workers están activos vs ociosos?
3. ¿Cuál es la latencia p99 de las tareas en los últimos 60 segundos?
4. ¿Cuántas tareas han fallado en los últimos 5 minutos?
5. ¿El pool está en backpressure (rechazando tareas)?

```go
type WorkerPoolInstrumentado struct {
    workers        int
    queueSize      atomic.Int64
    workersActivos atomic.Int64
    tareasTotal    atomic.Int64
    tareasFallidas atomic.Int64
    latencias      *histograma    // distribución de latencias

    // Expuesto via /metrics en formato Prometheus:
    // worker_pool_queue_size gauge
    // worker_pool_workers_active gauge
    // worker_pool_tasks_total counter
    // worker_pool_task_duration_seconds histogram
}
```

**Restricciones:** Las métricas deben ser actualizables sin degradar el throughput
del pool en más del 1%. Exponer en formato Prometheus (`/metrics`).
El histograma de latencias debe usar buckets exponenciales (1ms, 2ms, 5ms, 10ms, 25ms...).

**Pista:** Los histogramas de Prometheus usan buckets predefinidos — no son exactos
para percentiles arbitrarios. Para p99 exacto en memoria, usar una estructura
similar al `HistogramaExponencial` del Cap.10 §10.4.2.
El overhead de `atomic.Int64` para los contadores es ~5ns — irrelevante comparado
con el costo del trabajo del pool.

**Implementar en:** Go · Java · C# (con OpenTelemetry)

---

### Ejercicio 17.1.2 — Distributed tracing a través de múltiples goroutines

**Enunciado:** Propagar el trace context a través de goroutines concurrentes
es no trivial — las goroutines no tienen un "scope" como los threads de Java:

```go
// El problema:
func procesarRequest(ctx context.Context, req Request) {
    span, ctx := tracer.Start(ctx, "procesarRequest")
    defer span.End()

    // Lanzar goroutines que deben heredar el span:
    for _, item := range req.Items {
        go func(i Item) {
            // ¿Cómo acceder al span del padre aquí?
            // ctx se cierra (capturado por closure) pero podría cambiar
            childSpan, _ := tracer.Start(ctx, "procesarItem")  // ← ¿es esto seguro?
            defer childSpan.End()
            procesarItem(i, childSpan)
        }(item)
    }
}
```

**Restricciones:** Implementar el trace propagation para el Worker Pool del Ejercicio 17.1.1.
Cada tarea debe tener un span hijo del span del request original.
El trace completo debe ser visible en Jaeger con la jerarquía correcta.

**Pista:** En Go, el `context.Context` es inmutable e implícitamente propagado.
La goroutine puede capturar `ctx` (el contexto del padre que incluye el span).
El riesgo: si el span del padre termina antes que las goroutines hijas,
los spans hijos se "pierden" en el árbol de traces. Solución: usar `span.End()`
solo después de que todas las goroutines hijas terminen (con WaitGroup).

**Implementar en:** Go (con OpenTelemetry)

---

### Ejercicio 17.1.3 — Dashboards de producción: qué medir y cómo visualizarlo

**Enunciado:** Diseña un dashboard de Grafana para el sistema de procesamiento
de imágenes del Ejercicio 10.7.1 que responda en 10 segundos si el sistema
está saludable o no, y en 2 minutos cuál es el problema.

**El dashboard debe tener exactamente:**

```
Fila 1: Estado general (verde/amarillo/rojo en 3 paneles)
  - SLO de latencia: ¿está p99 por debajo del objetivo?
  - Tasa de error: ¿está debajo del umbral?
  - Throughput: ¿está por encima del mínimo?

Fila 2: Detalles de recursos (4 paneles)
  - Queue size por etapa del pipeline
  - Workers activos vs máximo
  - CPU usage
  - Memory usage

Fila 3: Distribución de latencias (2 paneles)
  - Heatmap de latencias (eje x: tiempo, eje y: latencia, color: frecuencia)
  - Percentiles p50/p90/p99 a lo largo del tiempo

Fila 4: Errores y alertas (3 paneles)
  - Tasa de error por tipo de error
  - Top 5 errores por frecuencia
  - Log stream de errores recientes
```

**Restricciones:** La primera fila debe responder "¿está bien?" sin leer números.
La segunda fila debe permitir identificar el cuello de botella en < 2 minutos.
Implementar el dashboard con archivos de configuración (Grafana JSON o código).

**Pista:** El "USE method" de Brendan Gregg: para cada recurso, medir
Utilization (qué % del tiempo está ocupado), Saturation (cuánto trabajo está esperando),
y Errors (cuántos errores por unidad de tiempo). Es el framework más eficiente
para diagnosticar problemas de rendimiento en sistemas concurrentes.

**Implementar en:** Go · Prometheus · Grafana (configuración)

---

### Ejercicio 17.1.4 — Alertas: los umbrales que importan

**Enunciado:** Las alertas mal calibradas son tan malas como no tener alertas.
Demasiadas alertas → alert fatigue → ignorar todas. Pocas alertas → incidentes sin detectar.

Implementa el sistema de alertas para el servidor del Ejercicio 17.1.1 siguiendo
las reglas de Google SRE:

```yaml
# Alertas sobre síntomas, no causas:
# MAL — alerta sobre causa:
- alert: PoolDeConexionesLleno
  expr: db_connection_pool_available == 0
  # → No sabes si esto afecta a los usuarios todavía

# BIEN — alerta sobre síntoma:
- alert: LatenciaAltaParaUsuarios
  expr: histogram_quantile(0.99, rate(request_duration_seconds_bucket[5m])) > 1.0
  for: 5m
  # → Los usuarios están experimentando latencia > 1s

# Alertas con severidad apropiada:
- alert: ErrorRateAlto
  expr: rate(requests_failed_total[5m]) / rate(requests_total[5m]) > 0.01
  severity: warning
  # > 1% de error rate → investigar pero no despertar a nadie a las 3AM

- alert: ErrorRateCritico
  expr: rate(requests_failed_total[5m]) / rate(requests_total[5m]) > 0.05
  severity: critical
  # > 5% de error rate → despertar al on-call
```

**Restricciones:** El sistema debe tener alertas para:
- Latencia p99 sobre SLO (síntoma de usuario)
- Error rate sobre umbral (síntoma de usuario)
- Saturación de recursos (causa — para investigación proactiva)
- Anomalías de tráfico (>3x el baseline)

**Pista:** El libro "Site Reliability Engineering" de Google describe los "four golden signals":
latency, traffic, errors, saturation. Las alertas sobre los primeros tres son
alertas sobre síntomas (afectan a usuarios). Las alertas sobre saturation son
alertas sobre causas (para investigación proactiva antes de que afecten usuarios).

**Implementar en:** Prometheus AlertManager (YAML config) + testing de alertas

---

### Ejercicio 17.1.5 — Debug de un deadlock en producción sin reiniciar el proceso

**Enunciado:** Un proceso Go en producción está "colgado" — no responde requests.
El CPU está al 0%. Sin poder reiniciar el proceso, diagnostica el problema:

```bash
# Herramientas disponibles (sin reiniciar):
kill -SIGQUIT <pid>    # Go imprime todas las goroutines al stdout
curl localhost:6060/debug/pprof/goroutine?debug=2  # dump de goroutines via HTTP
go tool pprof http://localhost:6060/debug/pprof/goroutine  # interactivo

# Leer el dump de goroutines para identificar el deadlock:
goroutine 18 [semacquire]:
sync.runtime_SemacquireMutex(...)
sync.(*Mutex).Lock(...)
main.(*Cache).Set(...)  ← esperando este mutex
main.procesarRequest(...)

goroutine 19 [semacquire]:
sync.runtime_SemacquireMutex(...)
sync.(*Mutex).Lock(...)
main.(*DB).Query(...)  ← esperando este mutex
main.(*Cache).Get(...)  ← mientras tiene el mutex de Cache
```

**Restricciones:** Implementar un sistema con un deadlock intencional y practicar
el diagnóstico con las herramientas disponibles. Luego fijar el deadlock
y documentar el proceso de diagnóstico como un runbook.

**Pista:** El dump de goroutines de Go muestra el stack trace de cada goroutine
y su estado (`running`, `sleeping`, `semacquire`, `chan receive`, etc.).
Un deadlock típico: goroutine A está en `semacquire` esperando un lock que
goroutine B tiene, y goroutine B está en `semacquire` esperando un lock que A tiene.
Para Java: `jstack <pid>` produce un dump similar.

**Implementar en:** Go (con intencional deadlock para practicar)

---

## Sección 17.2 — Resilience Patterns: Fallar Bien

Los sistemas distribuidos fallan. La pregunta no es si fallará, sino cuándo
y cómo. Los patterns de resilience diseñan el fallo para que sea controlado,
recuperable, y no cascadante.

**La taxonomía de fallos:**

```
Fallo transient (transitorio):
  Dura poco — reintentar usualmente funciona
  Ejemplos: timeout de red, GC pause temporal, spike de CPU

Fallo intermitente:
  Ocurre aleatoriamente — retry a veces funciona
  Ejemplos: instancia sobreargada, memory pressure

Fallo permanente:
  No se recupera sin intervención — no reintentes
  Ejemplos: nodo caído, disco lleno, bug de código

Fallo en cascada:
  Un componente falla → sobrecarga al siguiente → falla también
  La causa más común de outages completos
```

---

### Ejercicio 17.2.1 — Circuit Breaker calibrado para latencia y errores

**Enunciado:** El circuit breaker del Cap.03 §3.6 solo reaccionaba a errores.
En producción, la latencia alta es tan dañina como los errores:

```go
type CircuitBreakerConfig struct {
    // Umbrales de apertura:
    ErrorRate       float64        // % de errores → abrir (ej: 0.50 = 50%)
    SlowCallRate    float64        // % de llamadas lentas → abrir (ej: 0.80)
    SlowCallThreshold time.Duration // qué es "lenta" (ej: 2*time.Second)

    // Ventana deslizante:
    WindowSize      int            // últimas N llamadas para calcular rates

    // Umbrales de cierre:
    MinCallsToOpen  int            // mínimo de llamadas antes de abrir
    WaitDuration    time.Duration  // cuánto esperar en Open antes de probar

    // Half-open:
    PermittedCallsInHalfOpen int   // cuántas llamadas de prueba
}
```

**Restricciones:** Implementar el circuit breaker con estado "half-open" correcto:
en half-open, solo dejar pasar `PermittedCallsInHalfOpen` llamadas de prueba.
Si todas tienen éxito, cerrar. Si alguna falla, volver a open.
Verificar con un test que simula el patrón: éxito → fallo → recuperación.

**Pista:** La calibración más común para servicios web:
`ErrorRate = 0.50` (50% de errores en últimas 100 llamadas),
`SlowCallRate = 0.80, SlowCallThreshold = 2s` (80% tardando >2s),
`WaitDuration = 30s` (esperar 30s antes de probar en half-open).
Ajustar basándose en los SLOs del servicio específico.

**Implementar en:** Go · Java (con Resilience4j) · Python · C#

---

### Ejercicio 17.2.2 — Retry con backoff exponencial y jitter

**Enunciado:** El retry incorrecto puede hacer peor un sistema que ya está sobrecargado:

```go
// MAL — todos los clientes reintentan al mismo tiempo:
func reintentar(fn func() error, maxIntentos int) error {
    for i := 0; i < maxIntentos; i++ {
        if err := fn(); err == nil { return nil }
        time.Sleep(time.Second)  // TODOS esperan 1s y reintentan juntos → thundering herd
    }
    return errors.New("agotados los intentos")
}

// BIEN — backoff exponencial con jitter:
func reintentarConBackoff(ctx context.Context, fn func() error, config BackoffConfig) error {
    delay := config.InitialDelay
    for intento := 0; intento < config.MaxAttempts; intento++ {
        if err := fn(); err == nil { return nil }
        if !esReintentable(err) { return err }  // no reintentar errores permanentes

        jitter := time.Duration(rand.Int63n(int64(delay)))
        espera := delay + jitter
        select {
        case <-time.After(espera):
        case <-ctx.Done():
            return ctx.Err()
        }
        delay = min(delay*2, config.MaxDelay)
    }
    return errors.New("agotados los intentos")
}
```

**Restricciones:** El retry NO debe reintentar errores no-retryables (4xx HTTP,
errores de validación). Solo reintentar errores transient (5xx, timeout, connection refused).
El jitter debe reducir el "thundering herd" — verificar con un benchmark que
100 clientes con jitter generan menos picos que sin jitter.

**Pista:** El "thundering herd" ocurre cuando múltiples clientes fallan al mismo tiempo
(por ejemplo, cuando el servidor se reinicia) y todos reintentan en el mismo momento.
Sin jitter, el servidor recibe un spike de requests justo cuando está más vulnerable.
Con jitter, los reintentos se distribuyen en el tiempo.

**Implementar en:** Go · Python · C#

---

### Ejercicio 17.2.3 — Bulkhead: aislar grupos de recursos

**Enunciado:** El bulkhead (mamparo de barco) aísla grupos de requests para que
el fallo de uno no afecte a los otros:

```go
type BulkheadPool struct {
    pools map[string]*WorkerPool  // un pool por "tenant" o "prioridad"
}

// Sin bulkhead: una API lenta puede saturar el pool compartido
// Con bulkhead: cada API tiene su propio pool limitado
func (b *BulkheadPool) Submit(tenant string, tarea Task) error {
    pool, existe := b.pools[tenant]
    if !existe {
        return errors.New("tenant desconocido")
    }
    return pool.Submit(tarea)
    // Si el pool del tenant A está lleno, el tenant B no se ve afectado
}
```

**Restricciones:** Implementar bulkhead para un sistema multi-tenant donde el tenant "premium"
tiene un pool más grande y mayor prioridad que el tenant "free".
Si el tenant "free" satura su pool, el tenant "premium" no debe degradarse.
Verificar con un test de carga que el aislamiento funciona.

**Pista:** El bulkhead es especialmente importante en sistemas multi-tenant
y sistemas con dependencias externas: si el servicio A responde lento,
sin bulkhead puede agotar todos los threads disponibles esperando A,
dejando sin recursos para las requests que usan B (que está bien).
La solución: un pool dedicado por dependencia externa.

**Implementar en:** Go · Java · C#

---

### Ejercicio 17.2.4 — Fallback y degradación elegante

**Enunciado:** Cuando un servicio falla, la degradación elegante mantiene la funcionalidad
básica con datos menos frescos o funcionalidad reducida:

```go
type ServicioRecomendaciones struct {
    recomendador *Cliente  // servicio externo, puede fallar
    cache        *Cache    // caché local, siempre disponible
    fallback     []Item    // lista estática de "recomendaciones populares"
}

func (s *ServicioRecomendaciones) Recomendar(ctx context.Context, userID string) []Item {
    // Intentar el servicio externo:
    if items, err := s.recomendador.Get(ctx, userID); err == nil {
        s.cache.Set(userID, items, 5*time.Minute)
        return items
    }

    // Fallback 1: caché local (puede estar algo desactualizado):
    if items, ok := s.cache.Get(userID); ok {
        return items
    }

    // Fallback 2: lista estática (siempre disponible):
    return s.fallback
}
```

**Restricciones:** Implementar los tres niveles de fallback con métricas que
indican en qué nivel está operando el sistema. El dashboard debe mostrar
"90% servicio externo, 8% caché, 2% fallback estático".

**Pista:** El fallback en cascada es estándar en Netflix y otros sistemas de alta disponibilidad.
La clave: cada nivel de fallback debe tener su propia métrica para que
el equipo de ops sepa qué está pasando. Un sistema que opera en fallback
estático sin que nadie lo sepa es un sistema que parece estar bien cuando no lo está.

**Implementar en:** Go · Python · Java

---

### Ejercicio 17.2.5 — Timeout hierarchy: timeouts anidados correctamente

**Enunciado:** Los timeouts deben estar anidados correctamente — el timeout
del padre debe ser mayor que el del hijo más el overhead:

```
Request de usuario: timeout = 2s
  └── Servicio A: timeout = 1.5s
        └── DB query: timeout = 1s
              └── Query SQL: timeout = 800ms

Si la DB query tarda 900ms (sobre su timeout de 800ms):
  - La DB retorna timeout error a los 800ms
  - El servicio A recibe el error a los 800ms + overhead
  - El servicio A tiene 700ms para manejar el error y responder al usuario
  - El usuario ve un error claro, no un timeout silencioso
```

**Restricciones:** Implementar el servidor con la jerarquía de timeouts correcta.
Si los timeouts están al revés (DB > request del usuario), el usuario puede recibir
un timeout sin que el servidor haya recibido respuesta — la request "huérfana"
sigue ejecutando y consumiendo recursos.

**Pista:** El problema de las "orphaned requests": si el cliente hace timeout
antes que el servidor, el servidor sigue procesando la request (consumiendo CPU, DB, etc.)
aunque el resultado ya no se usará. Los timeouts anidados correctamente evitan
que los recursos se desperdicien en trabajo inútil.

**Implementar en:** Go · C# · Java

---

## Sección 17.3 — Backpressure End-to-End

El backpressure es el mecanismo por el que los consumidores señalizan a los productores
que necesitan ir más despacio. Sin backpressure, los sistemas bajo carga acumulan
trabajo hasta que la memoria se agota o los timeouts empiezan a dispararse.

**El backpressure en cascada:**

```
Cliente → API Gateway → Service A → Service B → Database
                                      ↑
                         Si la DB está lenta, el backpressure debe
                         propagarse de vuelta hasta el cliente.
                         Sin backpressure: cada capa acumula requests
                         hasta que algún timeout los mata (o la memoria se agota).
```

---

### Ejercicio 17.3.1 — Implementar backpressure con colas bounded y rechazo explícito

**Enunciado:** Diseña un sistema donde cada capa tiene una cola bounded
y rechaza explícitamente cuando está llena (en lugar de bloquear infinitamente):

```go
type ServicioConBackpressure struct {
    cola    chan Task  // cola bounded — sin backpressure implícito
    workers int
}

func (s *ServicioConBackpressure) Submit(ctx context.Context, t Task) error {
    select {
    case s.cola <- t:
        return nil  // aceptado
    case <-ctx.Done():
        return ctx.Err()
    default:
        // La cola está llena — rechazo explícito con error específico:
        metricas.IncrementarBackpressure()
        return ErrSaturado  // el cliente puede decidir qué hacer
    }
}
```

**Restricciones:** El rechazo debe usar un error específico (`ErrSaturado` o HTTP 429)
que permita al cliente distinguir "error del servidor" de "servidor saturado".
La distinción importa: para "error del servidor" → retry; para "servidor saturado" → backoff.
Verificar que bajo carga el sistema rechaza de forma estable en lugar de degradarse.

**Pista:** HTTP 429 (Too Many Requests) con `Retry-After` header es el mecanismo
estándar para backpressure en APIs HTTP. El cliente puede respetar el `Retry-After`
para no saturar más al servidor. Sin `Retry-After`, los clientes bien diseñados
igual harán backoff, pero el cabecero los ayuda a calibrarlo.

**Implementar en:** Go · Python (FastAPI) · C#

---

### Ejercicio 17.3.2 — Load shedding: cuándo rechazar trabajo

**Enunciado:** El load shedding descarta trabajo para sobrevivir a picos de carga:

```go
type LoadShedder struct {
    maxLatencia   time.Duration
    latenciaActual *ExpMovAvg  // promedio exponencial de latencia
}

func (ls *LoadShedder) DebeAtender(ctx context.Context, req Request) bool {
    latencia := ls.latenciaActual.Get()

    // Si la latencia supera el SLO, rechazar requests de baja prioridad:
    if latencia > ls.maxLatencia {
        if req.Priority < PriorityHigh {
            metricas.IncrementarShedding()
            return false  // rechazar
        }
    }

    // Si la latencia es extremadamente alta, rechazar todo excepto crítico:
    if latencia > ls.maxLatencia*3 {
        return req.Priority == PriorityCritical
    }

    return true
}
```

**Restricciones:** El load shedding debe ser proporcional: rechazar más trabajo
a medida que la latencia aumenta (no un switch binario).
Implementar con al menos 3 niveles de prioridad y 3 umbrales de rechazo.
Verificar con una prueba de carga que el sistema se estabiliza bajo sobrecarga
en lugar de degradarse completamente.

**Pista:** El load shedding es la última línea de defensa antes del fallo completo.
A diferencia del backpressure (que pausa al productor), el load shedding descarta
trabajo que ya entró al sistema. La decisión de qué trabajo descartar depende
de las prioridades del negocio: mantener los pagos funcionando aunque el
análisis de datos falle.

**Implementar en:** Go · Java · C#

---

### Ejercicio 17.3.3 — Admission control: decidir antes de empezar

**Enunciado:** El admission control rechaza requests antes de empezar a procesarlas,
basándose en la capacidad actual del sistema (Little's Law):

```
Little's Law: L = λW
  L = número de requests en el sistema
  λ = tasa de llegada (requests/segundo)
  W = tiempo de servicio (segundos)

Si λ aumenta y W aumenta (sistema más lento):
  L = λW crece rápidamente → el sistema se satura

Admission control: si L > L_max, rechazar nuevas requests
  L_max = capacidad máxima que puede manejar sin degradarse
```

```go
type AdmissionController struct {
    activas     atomic.Int64    // requests actualmente en el sistema
    maxActivas  int64           // L_max
}

func (ac *AdmissionController) Admitir() bool {
    activas := ac.activas.Add(1)
    if activas > ac.maxActivas {
        ac.activas.Add(-1)
        return false
    }
    return true
}

func (ac *AdmissionController) Completar() {
    ac.activas.Add(-1)
}
```

**Restricciones:** `maxActivas` debe ajustarse automáticamente basándose en
la latencia p99 observada: si p99 > SLO, reducir `maxActivas`; si p99 << SLO, aumentar.
Este es el "aimd" del Cap.10 §10.6.1 aplicado al admission control.

**Pista:** Netflix usa exactamente este patrón en su "Concurrency Limits" library.
El algoritmo ajusta el límite de concurrencia basándose en el gradiente de la latencia:
si la latencia aumenta cuando se añade más concurrencia, el sistema está saturado.
Si la latencia se mantiene estable, hay capacidad para más.

**Implementar en:** Go · Java (Netflix Concurrency Limits) · C#

---

### Ejercicio 17.3.4 — Backpressure de extremo a extremo con HTTP/2

**Enunciado:** HTTP/2 tiene flow control nativo — el servidor puede señalizar
al cliente que necesita ir más despacio sin necesidad de rechazar requests:

```go
// Con HTTP/2, el servidor puede enviar WINDOW_UPDATE frames para controlar
// cuántos bytes puede enviar el cliente antes de recibir un ACK.
// Go's net/http servidor de HTTP/2 lo maneja automáticamente cuando el handler
// tarda en leer el body.

// Para backpressure en streaming:
http.HandleFunc("/stream", func(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok { http.Error(w, "streaming not supported", 500); return }

    for evento := range eventos {
        fmt.Fprintf(w, "data: %s\n\n", evento)
        flusher.Flush()
        // Si el cliente es lento, Flush() bloqueará hasta que lea
        // → backpressure automático desde el cliente hasta el productor
    }
})
```

**Restricciones:** Implementar un servidor de Server-Sent Events (SSE) con
backpressure: si el cliente es lento, el servidor reduce la tasa de producción
de eventos. Verificar con un cliente lento que el servidor no acumula memoria
ilimitada esperando que el cliente lea.

**Pista:** SSE es más simple que WebSockets para streaming server-to-client.
El backpressure con SSE funciona porque TCP tiene su propio flow control:
si el buffer del socket del cliente está lleno, la escritura del servidor se bloquea.
Con goroutines, el bloqueo se propaga a través del pipeline de producción.

**Implementar en:** Go · C# (con Kestrel) · Python (con FastAPI)

---

### Ejercicio 17.3.5 — Medir y visualizar el backpressure

**Enunciado:** El backpressure debe ser visible en las métricas para distinguir
entre "el sistema rechaza porque está saturado" y "el sistema rechaza por un bug":

```
Métricas de backpressure a exponer:
  - requests_rejected_total{reason="queue_full|rate_limit|admission_control"}
  - queue_size{stage="ingress|processing|egress"}
  - queue_wait_time_seconds histogram
  - concurrent_requests_active gauge
  - concurrent_requests_limit gauge (el límite dinámico del admission control)
```

**Restricciones:** Implementar el dashboard de backpressure para el sistema
del Ejercicio 17.3.3. El dashboard debe mostrar si el sistema está:
1. Bien (p99 < SLO, sin rechazos)
2. Presionado (algún rechazo, p99 acercándose al SLO)
3. Saturado (muchos rechazos, p99 sobre el SLO)

**Pista:** La distinción entre "presionado" y "saturado" es crítica para el on-call:
presionado es normal bajo carga alta; saturado requiere intervención.
La métrica más útil es el ratio `rejected / (accepted + rejected)`: si sube
de 0% a 5% → presionado; si sube a 50% → saturado.

**Implementar en:** Go · Prometheus · Grafana

---

## Sección 17.4 — Testing en Producción: Chaos y Load Testing

La única forma de saber cómo se comporta un sistema bajo fallo real
es creando fallos reales en un entorno controlado.

---

### Ejercicio 17.4.1 — Chaos testing: inyectar fallos de forma controlada

**Enunciado:** Implementa un "chaos middleware" que inyecta fallos aleatoriamente
para verificar que los resilience patterns funcionan:

```go
type ChaosMiddleware struct {
    enabled     bool
    errorRate   float64          // % de requests que fallan con error
    slowRate    float64          // % de requests con latencia extra
    slowMin     time.Duration    // latencia mínima adicional
    slowMax     time.Duration    // latencia máxima adicional
}

func (c *ChaosMiddleware) Handler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !c.enabled { next.ServeHTTP(w, r); return }

        if rand.Float64() < c.errorRate {
            http.Error(w, "chaos error", 500)
            return
        }
        if rand.Float64() < c.slowRate {
            delay := c.slowMin + time.Duration(rand.Int63n(int64(c.slowMax-c.slowMin)))
            time.Sleep(delay)
        }
        next.ServeHTTP(w, r)
    })
}
```

**Restricciones:** El chaos debe poder activarse/desactivarse en runtime
(sin reiniciar el servidor). Con 10% de error rate, el circuit breaker debe
abrirse automáticamente. Con el circuit breaker abierto, el sistema debe
manejar el 90% de tráfico restante sin degradación.

**Pista:** Netflix Chaos Monkey, Gremlin, y Chaos Toolkit son herramientas
de chaos engineering de producción. El chaos middleware del ejercicio es
una versión simplificada para desarrollar intuición sobre los patrones de resiliencia.
La regla de oro del chaos: nunca en producción sin un "stop" bien probado.

**Implementar en:** Go · Python · C#

---

### Ejercicio 17.4.2 — Load testing con patrones de tráfico realistas

**Enunciado:** Un benchmark que llama constantemente a un endpoint no revela
cómo se comporta el sistema bajo tráfico real. El tráfico real tiene:
- Spikes (aumentos súbitos 5-10x)
- Ramp-ups (aumento gradual)
- Diurnal patterns (más tráfico de día que de noche)
- Long tail (distribución de Pareto: el 20% de los endpoints recibe el 80% del tráfico)

```python
# Con locust — framework de load testing en Python:
from locust import HttpUser, task, between, events
import random

class UsuarioRealista(HttpUser):
    wait_time = between(0.1, 2.0)  # tiempo entre requests (distribución uniforme)

    @task(80)  # 80% del tiempo
    def get_popular(self):
        self.client.get(f"/item/{random.choice(ITEMS_POPULARES)}")

    @task(20)  # 20% del tiempo
    def get_random(self):
        self.client.get(f"/item/{random.randint(1, 1_000_000)}")

    @task(5)
    def post_create(self):
        self.client.post("/item", json={"nombre": f"item-{random.random()}"})
```

**Restricciones:** Implementar un escenario de load test con:
- Ramp-up de 0 a 1000 usuarios en 5 minutos
- Spike: ir de 1000 a 5000 usuarios en 30 segundos
- Steady state: 5000 usuarios por 10 minutos
- Ramp-down: 5000 a 0 en 2 minutos
El sistema debe mantener p99 < SLO durante el steady state.

**Pista:** El spike es el test más importante: ¿cómo se comporta el sistema cuando
el tráfico sube 5x en 30 segundos? Sin auto-scaling: el circuit breaker y el
admission control deben proteger el servicio. Con auto-scaling: ¿cuánto tarda
en arrancar nuevas instancias y redistribuir el tráfico?

**Implementar en:** Python (Locust) · Go (k6 DSL en JS) · Java (Gatling)

---

### Ejercicio 17.4.3 — Property-based testing para propiedades de concurrencia

**Enunciado:** Los tests de propiedad del Cap.07 §7.2 se pueden aplicar
a sistemas completos, no solo a funciones individuales:

```python
from hypothesis import given, settings
from hypothesis import strategies as st

# Propiedad: el total de saldos bancarios siempre se conserva
@given(
    saldos_iniciales=st.lists(st.integers(min_value=0, max_value=10000), min_size=2),
    transferencias=st.lists(
        st.tuples(st.integers(), st.integers(), st.integers(min_value=1, max_value=100)),
        max_size=1000
    )
)
@settings(max_examples=100)
def test_conservacion_de_saldos(saldos_iniciales, transferencias):
    banco = Banco(saldos_iniciales)
    total_inicial = sum(banco.saldos())

    # Ejecutar transferencias en paralelo con 8 threads:
    with ThreadPoolExecutor(max_workers=8) as pool:
        futures = [
            pool.submit(banco.transferir, de % len(saldos_iniciales),
                       a % len(saldos_iniciales), monto)
            for de, a, monto in transferencias
        ]
        for f in futures: f.result()

    total_final = sum(banco.saldos())
    assert total_inicial == total_final, f"Saldo perdido: {total_inicial} → {total_final}"
```

**Restricciones:** Implementar property tests para las invariantes del Worker Pool:
- Todos los items enviados se procesan exactamente una vez
- El número de workers activos nunca excede el máximo configurado
- Después del shutdown, no quedan goroutines/threads activos (no leaks)

**Pista:** El property testing es especialmente valioso para código concurrente
porque genera automáticamente los edge cases que un humano no pensaría
(listas vacías, valores máximos, patrones de transferencia circular, etc.).
Cada fallo que Hypothesis encuentra es un test de regresión gratuito.

**Implementar en:** Go (con gopter o rapid) · Python (con Hypothesis) · Java (jqwik)

---

### Ejercicio 17.4.4 — Soak testing: detectar memory leaks y goroutine leaks

**Enunciado:** Un soak test (prueba de resistencia) corre durante horas o días
para detectar degradación gradual — memory leaks, goroutine leaks, connection leaks:

```go
// Verificar goroutine leaks con goleak:
func TestWorkerPoolNoLeaks(t *testing.T) {
    defer goleak.VerifyNone(t)  // falla si hay goroutines activas al salir

    pool := NuevoWorkerPool(8)
    for i := 0; i < 10_000; i++ {
        pool.Submit(func() { time.Sleep(time.Millisecond) })
    }
    pool.Shutdown()
    // Si alguna goroutine del pool no terminó, goleak lo detecta aquí
}

// Verificar memory leaks con métricas de heap:
func soak_test(duracion time.Duration) {
    inicio := time.Now()
    for time.Since(inicio) < duracion {
        procesarUnBatch()
        runtime.GC()
        var stats runtime.MemStats
        runtime.ReadMemStats(&stats)
        log.Printf("HeapAlloc: %d MB", stats.HeapAlloc/1024/1024)
        // Si HeapAlloc crece indefinidamente → memory leak
        time.Sleep(30 * time.Second)
    }
}
```

**Restricciones:** Correr el soak test durante al menos 1 hora con el servidor
del Ejercicio 17.1.1. El heap no debe crecer más del 10% durante el steady state.
Ninguna goroutine debe quedar activa después del shutdown.

**Pista:** Los leaks más comunes en Go:
1. Goroutines que esperan en un canal que nunca se cierra
2. Closures que capturan variables en maps (el map crece pero nunca se limpia)
3. Listeners de HTTP que no se cierran al salir de un test
4. Connections de BD que no se retornan al pool

**Implementar en:** Go (con goleak) · Java (con Leak Canary) · Python

---

### Ejercicio 17.4.5 — Failure mode analysis: documentar cómo falla el sistema

**Enunciado:** Un FMEA (Failure Mode and Effects Analysis) documenta
cómo puede fallar cada componente y cuál es el impacto:

```markdown
# FMEA: Sistema de Procesamiento de Imágenes

| Componente | Modo de fallo | Causa | Efecto en el sistema | Detección | Mitigación |
|---|---|---|---|---|---|
| Worker Pool | Cola llena | Tasa de arrival > capacidad | Backpressure, rechaza nuevas imágenes | Métrica queue_size | Escalar horizontalmente |
| Worker Pool | Goroutine leak | Context no cancelado | OOM gradual | Métrica goroutines_count | Timeout en cada tarea |
| Cache | Cache miss storm | Restart del servicio | Todos los workers van a BD directamente | Métrica cache_miss_rate | Circuit breaker BD, warm-up |
| BD | Conexiones agotadas | Goroutines en espera | Timeouts en cascada | Métrica db_pool_available | Connection pool con límite |
| BD | Query lenta | Tabla sin índice | Latencia p99 alta | Métrica query_duration_p99 | Slow query log, alertas |
```

**Restricciones:** Completar el FMEA para el sistema completo del Ejercicio 17.7.5.
Para cada modo de fallo, implementar la "mitigación" y verificar con un test de chaos
que la mitigación funciona.

**Pista:** El FMEA no necesita ser exhaustivo para ser útil. Con 10-20 filas
que cubren los fallos más probables y de mayor impacto, ya tienes una guía
de operaciones valiosa. La priorización: `probabilidad × impacto` determina
qué mitigaciones implementar primero.

**Implementar en:** Documento + implementaciones de mitigación

---

## Sección 17.5 — Capacity Planning: Dimensionar Antes de Saturar

El capacity planning es la habilidad de predecir cuándo el sistema se saturará
y dimensionarlo para que no lo haga.

---

### Ejercicio 17.5.1 — Modelo de capacidad: de métricas a predicciones

**Enunciado:** Un modelo de capacidad simple usa los datos actuales para predecir
cuándo el sistema se saturará:

```python
import pandas as pd
import numpy as np
from scipy.stats import linregress

# Métricas históricas (del sistema de monitoreo):
metricas = pd.DataFrame({
    'timestamp': pd.date_range('2024-01-01', periods=90, freq='D'),
    'requests_por_dia': np.random.poisson(100_000, 90) * (1 + 0.02 * np.arange(90) / 90),
    'cpu_promedio': np.random.normal(40, 5, 90),
    'memoria_gb': np.random.normal(8, 1, 90),
    'latencia_p99_ms': np.random.normal(50, 10, 90),
})

def predecir_saturacion(df, metrica, umbral):
    """¿Cuándo alcanzará 'metrica' el valor 'umbral'?"""
    slope, intercept, r, _, _ = linregress(
        range(len(df)), df[metrica]
    )
    if slope <= 0: return "nunca (tendencia descendente)"
    dias_hasta_umbral = (umbral - intercept) / slope
    return f"{int(dias_hasta_umbral - len(df))} días"

print(f"CPU al 80%: {predecir_saturacion(metricas, 'cpu_promedio', 80)}")
print(f"Memoria a 32GB: {predecir_saturacion(metricas, 'memoria_gb', 32)}")
```

**Restricciones:** Implementar el modelo para las métricas del servidor del Ejercicio 17.1.1.
El modelo debe predecir cuándo el sistema necesitará escalar horizontalmente
(más instancias) o verticalmente (más recursos por instancia).

**Pista:** La regresión lineal simple es suficiente para tendencias de 30-90 días.
Para predicciones a más largo plazo, usar modelos de series temporales (SARIMA, Prophet).
El capacity planning no necesita ser exacto — un margen de error del 20% es aceptable
si planeas con 6 semanas de anticipación.

**Implementar en:** Python (con pandas y scipy)

---

### Ejercicio 17.5.2 — Benchmarking para capacity planning

**Enunciado:** El capacity planning requiere saber el throughput máximo de cada
componente bajo condiciones controladas:

```
Para el Worker Pool, medir:
  - Throughput máximo (tasks/segundo) con 1, 2, 4, 8 workers y tasks de 1ms, 10ms, 100ms
  - Latencia p99 en el 50%, 80%, y 95% de carga
  - El punto de saturación (cuando añadir más carga no añade throughput)
  
Para el servidor HTTP:
  - Requests/segundo con payload de 1KB, 10KB, 100KB
  - Latencia p99 a distintas cargas
  - Número de conexiones simultáneas antes de degradar

Construir una "capacity matrix":
  Workers | Task 1ms | Task 10ms | Task 100ms
  1       | 850/s    | 95/s      | 9.5/s
  2       | 1680/s   | 188/s     | 19/s
  4       | 3350/s   | 375/s     | 38/s
  8       | 6700/s   | 748/s     | 75/s    ← ~linear scaling hasta aquí
  16      | 8200/s   | 900/s     | 89/s    ← saturación de I/O o CPU
```

**Restricciones:** La "capacity matrix" debe cubrir al menos 3 tamaños de task
y 4 configuraciones de workers. El punto de saturación debe ser identificable
claramente en los datos.

**Pista:** El throughput escala linealmente con workers hasta que otro recurso
se satura (CPU, red, BD, memoria). Identificar qué recurso satura primero
es el objetivo del capacity planning — porque es ese recurso el que hay que escalar.

**Implementar en:** Go · benchmark suite

---

### Ejercicio 17.5.3 — Dimensionar la cola: Little's Law en la práctica

**Enunciado:** Little's Law permite calcular el tamaño de cola necesario
para cumplir un SLO de latencia:

```
Little's Law: L = λ × W
  L = tamaño de cola necesario
  λ = tasa de arrival (requests/segundo)
  W = tiempo en el sistema (segundos)

Si: λ = 1000 req/s, tiempo de procesamiento = 10ms
  En estado estable: L = 1000 × 0.010 = 10 items en la cola

Para manejar un spike de 2x durante 5s:
  Spike λ = 2000 req/s, excess = 1000 req/s × 5s = 5000 items adicionales
  Cola necesaria = 10 (steady state) + 5000 (spike) = 5010

Tamaño de cola recomendado: ~5500 (con margen del 10%)
```

**Restricciones:** Calcula el tamaño de cola para el Worker Pool del Ejercicio 17.3.1
que puede absorber un spike de 3x durante 30 segundos sin rechazar requests.
Verifica con un load test que el cálculo es correcto.

**Pista:** Little's Law asume estado estable — durante transiciones, el sistema
puede tener más items de los que L predice. El "margen" (10-20% extra)
acomoda las transiciones. Para spikes más largos o más grandes,
el auto-scaling es más eficiente que hacer la cola más grande.

**Implementar en:** Python (cálculo) + Go (verificación con load test)

---

### Ejercicio 17.5.4 — Auto-scaling: responder a la demanda automáticamente

**Enunciado:** Implementa un auto-scaler simplificado que ajusta el número
de workers en el pool basándose en la carga observada:

```go
type AutoScaler struct {
    pool          *WorkerPool
    minWorkers    int
    maxWorkers    int
    targetUtilization float64  // ej: 0.70 = 70% de utilización objetivo

    scaleUpCooldown   time.Duration  // tiempo mínimo entre scale-ups
    scaleDownCooldown time.Duration  // tiempo mínimo entre scale-downs (más conservador)

    lastScaleTime time.Time
}

func (as *AutoScaler) Tick() {
    utilizacion := as.pool.Utilizacion()  // workers activos / workers totales

    if utilizacion > as.targetUtilization && time.Since(as.lastScaleTime) > as.scaleUpCooldown {
        nuevosTrabajadores := min(as.pool.Workers()*2, as.maxWorkers)
        as.pool.Resize(nuevosTrabajadores)
        as.lastScaleTime = time.Now()
    } else if utilizacion < as.targetUtilization*0.5 && time.Since(as.lastScaleTime) > as.scaleDownCooldown {
        nuevosTrabajadores := max(as.pool.Workers()/2, as.minWorkers)
        as.pool.Resize(nuevosTrabajadores)
        as.lastScaleTime = time.Now()
    }
}
```

**Restricciones:** El scale-up debe ser más agresivo que el scale-down
(subir rápido, bajar lento). El cooldown para scale-down debe ser
al menos 5x el cooldown para scale-up. Verificar con el load test del Ejercicio 17.4.2
que el auto-scaler responde al spike.

**Pista:** El scale-down conservador ("slowly shrink, quickly grow") es el patrón
estándar en cloud auto-scaling (AWS Auto Scaling Groups, Kubernetes HPA).
Escalar hacia abajo demasiado rápido puede causar oscilaciones: se escala hacia abajo,
llega más tráfico, se escala hacia arriba, menos tráfico, hacia abajo... en bucle.

**Implementar en:** Go · Python · C#

---

### Ejercicio 17.5.5 — Cost modeling: eficiencia de recursos en producción

**Enunciado:** El capacity planning incluye el costo. Un sistema que usa el 10%
de sus recursos el 90% del tiempo es costoso. Un sistema que usa el 80-90%
de sus recursos la mayor parte del tiempo es eficiente.

```python
# Modelo de costo simplificado:
def costo_mensual(instancias, tipo_instancia, horas_mes=720):
    precios = {
        "c5.xlarge": 0.170,   # $/hora — 4 vCPU, 8 GB RAM
        "c5.2xlarge": 0.340,  # $/hora — 8 vCPU, 16 GB RAM
        "c5.4xlarge": 0.680,  # $/hora — 16 vCPU, 32 GB RAM
    }
    return instancias * precios[tipo_instancia] * horas_mes

# Comparar configuraciones:
# Opción A: 4 instancias c5.2xlarge (32 vCPU total, $979/mes)
# Opción B: 8 instancias c5.xlarge  (32 vCPU total, $979/mes)  ← misma CPU, mismo costo
# Opción C: 2 instancias c5.4xlarge (32 vCPU total, $979/mes)  ← misma CPU, mismo costo
# La diferencia: resilience (A y B sobreviven un fallo de instancia, C no) y
#                eficiencia de red y memoria
```

**Restricciones:** Para el sistema del Ejercicio 17.7.5, calcular el costo mensual
de tres configuraciones distintas y determinar cuál ofrece el mejor balance entre
costo, performance, y resilience.

**Pista:** La regla del 80%: diseña para que la CPU promedio sea ~70-80% durante
las horas pico. Menos del 50% → sobredimensionado (estás pagando por capacidad que no usas).
Más del 90% → subdimensionado (sin margen para spikes).
Los costos de nube tienen componentes variables (tráfico de red) que no son predecibles
— siempre añadir un 20-30% de margen al estimado.

**Implementar en:** Python (análisis) + documento de decisión

---

## Sección 17.6 — Runbooks: Operar Sistemas Concurrentes

Un runbook es la documentación de cómo diagnosticar y resolver problemas
en producción. Sin runbook, el on-call reinventa la solución en cada incidente.

---

### Ejercicio 17.6.1 — Runbook para latencia alta

**Enunciado:** Escribe el runbook para el incidente del inicio del capítulo
(latencia p99 sube de 50ms a 8000ms):

```markdown
# Runbook: Latencia p99 Alta

## Síntoma
- Alerta: `request_duration_p99 > 1s` por más de 5 minutos

## Impacto para el usuario
- Los usuarios experimentan requests lentas o timeouts

## Diagnóstico (en este orden)

### Paso 1: Verificar si es un problema de carga (2 minutos)
```
kubectl top nodes              # ¿CPU o memoria al límite?
kubectl top pods              # ¿Algún pod con CPU alta?
curl /metrics | grep queue    # ¿Cola del worker pool creciendo?
```
Si la carga es alta: [ir a Paso 1a: Escalar]
Si la carga es normal: [ir a Paso 2]

### Paso 1a: Escalar
```
kubectl scale deployment/servicio --replicas=8  # o el número adecuado
```
Esperar 3 minutos. ¿Mejora la latencia? Si no: [ir a Paso 2]

### Paso 2: Verificar dependencias externas (3 minutos)
```
curl /metrics | grep db_query_duration  # ¿Queries lentas?
curl /metrics | grep cache_miss         # ¿Cache miss rate alto?
```
[... continuar el árbol de decisión ...]

## Resolución
[Lista de comandos para cada causa identificada]

## Post-mortem
[Enlace a la plantilla de post-mortem]
```

**Restricciones:** El runbook debe llevar al on-call a la causa raíz en menos
de 10 minutos para los casos más comunes. Cada paso debe tener comandos exactos
(no "verificar la DB" sino "curl /metrics | grep db_pool_available").

**Pista:** Los mejores runbooks se escriben justo después de resolver un incidente —
cuando aún recuerdas los pasos que realmente funcionaron. Un runbook escrito
"en frío" tende a ser incompleto porque olvidas los comandos exactos y los casos edge.

**Implementar en:** Documento Markdown

---

### Ejercicio 17.6.2 — Post-mortem: aprender de los incidentes

**Enunciado:** Un post-mortem blameless documenta qué pasó, por qué, y cómo prevenir
que vuelva a ocurrir:

```markdown
# Post-mortem: Latencia Alta el Black Friday

## Resumen
Durante 45 minutos (03:15 - 04:00 AM), el sistema experimentó latencia
p99 > 5 segundos, afectando al 30% de los usuarios activos.

## Cronología
03:15 - Primer aumento de latencia detectado en métricas
03:22 - Alerta disparada (demora de 7 minutos por el "for: 5m" del alert)
03:25 - On-call despertado
03:40 - Causa identificada: connection pool agotado
03:52 - Mitigación aplicada: aumentar maxConns de 10 a 50
04:00 - Sistema restaurado

## Causa raíz
El pool de conexiones a BD tenía maxConns=10, configurado para carga normal.
El Black Friday generó 8x el tráfico normal. Los timeouts de conexión eran
10s — largo suficiente para acumular 800 requests en cola antes de que
los circuit breakers abrieran.

## Acciones de seguimiento
- [ ] Aumentar maxConns permanentemente a 30 (1 semana)
- [ ] Implementar circuit breaker calibrado para latencia (2 semanas)
- [ ] Reducir timeout de alerta de 5min a 2min (1 día)
- [ ] Añadir runbook para este escenario (esta semana)
```

**Restricciones:** Escribe el post-mortem del incidente que implementaste en el
Ejercicio 17.1.5 (deadlock). Incluye las métricas que mostraron el problema,
el tiempo de diagnóstico, y al menos 3 acciones concretas para prevenir recurrencia.

**Pista:** El principio del post-mortem blameless: el objetivo es aprender del sistema,
no culpar a personas. Los sistemas concurrentes fallan de formas que ningún
individuo podría haber predicho completamente — la solución es mejorar el sistema
(observabilidad, resilience, runbooks) no castigar a quien tomó la decisión incorrecta
con la información disponible en ese momento.

**Implementar en:** Documento Markdown

---

### Ejercicio 17.6.3 — SLO, SLA, y error budgets

**Enunciado:** Define y calcula los SLOs para el sistema del Ejercicio 17.7.5:

```
SLO (Service Level Objective): el objetivo interno de calidad
SLA (Service Level Agreement): el compromiso externo (subset del SLO)

Ejemplo:
  SLO de latencia: p99 < 200ms durante el 99.9% del tiempo (ventana de 30 días)
  SLO de disponibilidad: 99.9% de requests exitosas (< 0.1% de errores)
  
  Error budget: 0.1% de requests = 100 * 0.1% = 0.1 request de cada 1000 puede fallar
  En una hora con 10,000 requests: máximo 10 errores
  En un mes con 30M requests: máximo 30,000 errores
```

```python
def calcular_error_budget(requests_por_mes, slo_percent):
    budget_total = requests_por_mes * (1 - slo_percent/100)
    burn_rate_diaria = budget_total / 30
    print(f"Budget total: {budget_total:.0f} errores/mes")
    print(f"Burn rate saludable: {burn_rate_diaria:.0f} errores/día")
    return budget_total

# Si el sistema está quemando el error budget a 2x la tasa normal:
# → El SLO estará violado antes del fin del mes
# → Activar "feature freeze" hasta reducir la tasa de quema
```

**Restricciones:** Define SLOs para latencia, disponibilidad, y throughput mínimo.
Implementa el cálculo de "burn rate" en tiempo real y una alerta cuando
el burn rate implica que el SLO se violará antes del fin del mes.

**Pista:** El "error budget burn rate" es la métrica más valiosa de los SLOs.
Una alerta que dice "latencia p99 = 210ms (SLO: 200ms)" es menos útil que
"estás quemando el error budget 3x más rápido de lo normal — el SLO se violará
en 8 días si continúa así". Google SRE usa esta métrica como alerta principal.

**Implementar en:** Python (cálculo) + Prometheus (alerta)

---

### Ejercicio 17.6.4 — On-call rotation y operational maturity

**Enunciado:** La madurez operacional de un sistema se puede medir:

```
Nivel 1 (caótico):
  - Sin monitoring
  - Los problemas los reportan los usuarios
  - Sin runbooks
  - Cada incidente es una emergencia única

Nivel 2 (reactivo):
  - Monitoring básico (uptime, error rate)
  - Alertas que despiertan al on-call
  - Runbooks incompletos
  - Post-mortems a veces

Nivel 3 (proactivo):
  - Observabilidad completa (métricas, logs, traces)
  - Alertas sobre síntomas de usuario
  - Runbooks para los 10 incidentes más comunes
  - Post-mortems para todos los incidentes > 15 min
  - Error budgets y SLOs

Nivel 4 (preventivo):
  - Chaos engineering regular
  - Capacity planning con 6 semanas de anticipación
  - Auto-scaling y auto-remediation
  - Reducción del toil (trabajo operacional repetitivo)
```

**Restricciones:** Evalúa el sistema del Ejercicio 17.7.5 en la escala de madurez
operacional. Para cada ítem del Nivel 3, implementarlo si no está presente.

**Pista:** El objetivo del Nivel 3 es que el on-call pueda resolver el 80% de
los incidentes en menos de 30 minutos con el runbook, sin necesitar escalate.
El Nivel 4 es aspiracional — pocos sistemas lo alcanzan completamente, y no todos
lo necesitan. La decisión de cuánta madurez operacional necesita un sistema
depende de su criticidad de negocio.

**Implementar en:** Documento de evaluación + implementación de gaps

---

### Ejercicio 17.6.5 — Operational review: revisión trimestral de salud del sistema

**Enunciado:** Una revisión trimestral sistemática previene que los sistemas
se deterioren gradualmente:

```
Revisión trimestral — checklist:

Observabilidad:
  □ ¿Todas las métricas críticas tienen alertas calibradas?
  □ ¿Los dashboards responden "¿está bien?" en 10 segundos?
  □ ¿Los traces son legibles en Jaeger?

Resilience:
  □ ¿Se hizo chaos testing en los últimos 3 meses?
  □ ¿Los circuit breakers están correctamente calibrados?
  □ ¿Los timeouts están en la jerarquía correcta?

Capacity:
  □ ¿Cuándo se saturará el sistema basándose en la tendencia actual?
  □ ¿El error budget se ha quemado a una tasa aceptable?
  □ ¿Hay deuda técnica de concurrencia que resolver?

Operaciones:
  □ ¿Los runbooks están actualizados después de los últimos incidentes?
  □ ¿El FMEA está al día con los cambios del sistema?
  □ ¿El equipo de on-call siente confianza operacional?
```

**Restricciones:** Completa la revisión trimestral para el sistema del Ejercicio 17.7.5.
Cada ítem marcado con □ debe tener una respuesta o una acción de seguimiento.
El resultado es un "health score" del sistema y una lista priorizada de mejoras.

**Pista:** La revisión trimestral es más valiosa cuando la hace el equipo completo,
no solo el tech lead. Los ingenieros que hacen on-call tienen información sobre
los problemas más comunes que el tech lead puede no tener. La revisión es también
una oportunidad de transferir conocimiento sobre el sistema.

**Implementar en:** Documento + sesión de equipo

---

## Sección 17.7 — Diseño Integral: Un Sistema Real de Principio a Fin

---

### Ejercicio 17.7.1 — Sistema de procesamiento de eventos en tiempo real

**Enunciado:** Diseña e implementa un sistema que procesa 100,000 eventos por segundo
con p99 < 50ms, persistiendo los resultados en una BD y actualizando dashboards en tiempo real.

**Arquitectura:**

```
Ingesta (HTTP/gRPC)
  └─ Validación y descompresión
      └─ Cola bounded (backpressure)
          └─ Worker Pool (N workers)
              ├─ Procesamiento CPU-bound
              ├─ Escritura en BD (batch)
              └─ Publicación en WebSocket (tiempo real)
```

**Restricciones:** El sistema completo debe tener:
- Observabilidad completa (métricas, logs con correlation ID, traces)
- Circuit breakers para BD y para el servicio de WebSocket
- Backpressure en cada etapa (no acumular ilimitadamente)
- Graceful shutdown (procesar el buffer antes de apagar)
- Tests: unit, integration, property-based, y load test

**Pista:** El diseño en capas con queues bounded entre ellas es el patrón
más probado para este tipo de sistema. Cada queue actúa como un buffer
y como punto de backpressure. El graceful shutdown drena las queues
en orden inverso (primero dejar de recibir, luego procesar lo acumulado,
luego cerrar la BD).

**Implementar en:** Go (lenguaje principal) + Prometheus + Grafana

---

### Ejercicio 17.7.2 — Sistema de caché distribuido con consistencia configurable

**Enunciado:** Implementa un caché distribuido de dos nodos con consistencia configurable:

```
Modo Strong Consistency (CP):
  - Writes van a ambos nodos antes de confirmar
  - Reads pueden ir a cualquier nodo (siempre frescos)
  - Disponibilidad: si un nodo falla, rechazar writes

Modo Eventual Consistency (AP):
  - Writes van al nodo primario y se propagan asíncronamente
  - Reads pueden ir a cualquier nodo (pueden ser stalenes)
  - Disponibilidad: si el primario falla, el secundario acepta writes
  
Implementar la transición entre modos:
  - Detectar fallo de nodo (heartbeat)
  - Actualizar el modo automáticamente
  - Propagar los cambios pendientes cuando el nodo vuelve
```

**Restricciones:** Demostrar las diferencias con tests que muestran:
- En modo Strong: reads siempre ven el último write
- En modo Eventual: reads pueden ver datos con retraso, pero eventualmente convergen
- La disponibilidad en cada modo cuando un nodo falla

**Pista:** Este ejercicio implementa el teorema CAP del Cap.09 §9.5 en código real.
La transición de CP a AP durante un fallo se llama "failing open" — priorizar
la disponibilidad sobre la consistencia. La decisión de qué modo usar depende
del tipo de datos: para el saldo de una cuenta → CP; para el número de likes → AP.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 17.7.3 — Pipeline de ML con inferencia de baja latencia

**Enunciado:** Implementa el pipeline completo de inferencia de ML del Ejercicio 10.7.1
con todos los patrones de producción aplicados:

```
Componentes:
  1. HTTP endpoint (FastAPI/Go net/http)
  2. Request validation y preprocessing (async)
  3. Batching dinámico (acumular requests para GPU efficiency)
  4. Model inference (CPU o GPU — bloqueante)
  5. Postprocessing (async)
  6. Response

Patrones aplicados:
  - Backpressure: bounded queue entre 1-2 y 3-4
  - Circuit breaker: si la inferencia tarda > 500ms en p99, abrir
  - Hedged requests: si no hay respuesta en 200ms, lanzar segunda request
  - Observabilidad: latencia por etapa, batch size, throughput
  - Graceful shutdown: completar inflight requests, no aceptar nuevas
```

**Restricciones:** Con 1000 requests/segundo, el sistema debe mantener p99 < 100ms.
Los patrones de resiliencia deben estar activos y verificados con chaos testing.

**Pista:** El batching dinámico es el patrón más importante para eficiencia de GPU:
acumular 32-64 requests antes de enviar al modelo es mucho más eficiente que
inferir de una en una. El "dynamic batching" espera hasta que el batch está lleno
O hasta que pasa un timeout (ej: 5ms) — lo que ocurra primero.

**Implementar en:** Python (FastAPI + PyTorch) + Prometheus

---

### Ejercicio 17.7.4 — Migration pattern: actualizar un sistema en producción sin downtime

**Enunciado:** Implementa un sistema de migración de la versión 1 a la versión 2
sin downtime (blue-green deployment):

```
Estado inicial: 100% del tráfico → V1

Paso 1: Desplegar V2 sin tráfico
  - V1 y V2 corriendo en paralelo
  - V2 procesando 0% del tráfico
  - Verificar que V2 está healthy

Paso 2: Canary deployment (5% → V2)
  - Monitorear métricas de V2 vs V1
  - Si p99 de V2 > p99 de V1 × 1.5: rollback automático
  - Si las métricas son comparables: continuar

Paso 3: Gradual rollout (5% → 25% → 50% → 100%)
  - En cada step, esperar N minutos y verificar métricas
  - Rollback automático si las métricas se degradan

Paso 4: Deprecar V1
  - V1 sigue corriendo por 24h (per si acaso)
  - Después de 24h sin tráfico: apagar V1
```

**Restricciones:** El rollback debe ser automático (no manual) y tomar < 30 segundos.
La decisión de continuar vs rollback debe basarse en métricas (p99, error rate),
no en juicio humano.

**Pista:** El canary deployment requiere que el sistema de routing pueda dividir
el tráfico entre V1 y V2. Nginx, Envoy, o Kubernetes (con Flagger) pueden hacer
esto automáticamente basándose en headers, cookies, o porcentaje aleatorio.
La clave: las métricas de V1 y V2 deben estar separadas en el monitoring
para que la comparación sea válida.

**Implementar en:** Go + docker-compose (para simular el routing)

---

### Ejercicio 17.7.5 — El sistema final: integrar todos los patrones

**Enunciado:** Este es el ejercicio final del repositorio — integrar todos los
conceptos de los 17 capítulos en un sistema de producción real.

**Sistema: Plataforma de procesamiento de pedidos**

```
Componentes:
  API Gateway (Go o C#)
    - Rate limiting por cliente (Cap.03 §3.7)
    - Circuit breaker hacia los servicios (Cap.03 §3.6)
    - Distributed tracing (Cap.17 §17.1)

  Servicio de Pedidos (Go o Java 21)
    - Worker Pool con auto-scaling (Cap.03, Cap.17 §17.5)
    - Saga pattern para transacciones distribuidas (Cap.09 §9.2)
    - Event sourcing para auditoría

  Servicio de Inventario (Go o Rust)
    - Skip list para búsqueda por rango (Cap.12 §12.4)
    - CRDTs para actualización concurrente (Cap.09 §9.5)

  Servicio de Pagos (Java 21 o C#)
    - Virtual Threads para alta concurrencia (Cap.14 §14.4)
    - Timeout hierarchy correcta (Cap.17 §17.2)

  Cola de Eventos (implementada con canales)
    - Múltiples consumers (Cap.05 §5.4)
    - Backpressure end-to-end (Cap.17 §17.3)

Observabilidad (requerida):
  - Dashboard Grafana con los 4 golden signals
  - Distributed tracing en Jaeger
  - Alertas con error budget
  - Runbook para los 5 incidentes más comunes

Tests (requeridos):
  - Unit tests: > 80% cobertura en lógica de concurrencia
  - Integration tests: cada par de servicios
  - Property tests: invariantes del sistema (saldos, inventario)
  - Load test: 1000 pedidos/segundo con p99 < 200ms
  - Chaos test: qué pasa si el servicio de pagos cae
```

**Restricciones:** El sistema es demasiado grande para implementarlo completamente —
la restricción es implementar al menos 3 de los 5 componentes con los patrones de
observabilidad y resiliencia. Documentar las decisiones de diseño con referencias
a los capítulos del repositorio donde se discutió cada patrón.

**Pista:** En un sistema real, cada componente tomaría semanas. El objetivo de este
ejercicio no es completar el sistema — es experimentar la integración de los patrones
y entender dónde emergen los trade-offs. Las decisiones de diseño más difíciles
no son las técnicas (mutex vs canal) sino las arquitectónicas (dónde poner el límite
entre servicios, cómo manejar la consistencia entre ellos).

**Implementar en:** Lenguaje de tu elección + docker-compose para integración

---

## Resumen del capítulo y del repositorio

**Los 10 principios que cierra el repositorio:**

```
1. Mide antes de optimizar.
   El profiler tiene la última palabra. Las intuiciones sobre rendimiento
   son frecuentemente incorrectas.

2. El código correcto primero, rápido después.
   Un sistema incorrecto rápido es peor que un sistema correcto lento.
   La concurrencia mal implementada introduce bugs que tardan meses en encontrarse.

3. El GC no es gratis, pero tampoco es el problema.
   El GC de Go, Java, y C# es suficientemente bueno para el 99% de los sistemas.
   Optimizar para el GC antes de que sea un problema medido es prematuro.

4. Los timeouts son contrato, no sugerencia.
   Un sistema sin timeouts correctos puede degradarse indefinidamente.
   Los timeouts deben estar anidados correctamente.

5. El backpressure debe propagarse hasta el cliente.
   Un servidor que acepta trabajo que no puede procesar acumula deuda.
   El rechazo explícito y rápido es mejor que el timeout silencioso y lento.

6. Fallar rápido, recuperar ordenado.
   Los circuit breakers, retry con backoff, y graceful shutdown
   son la diferencia entre un incidente de 5 minutos y uno de 5 horas.

7. La observabilidad no es opcional.
   Un sistema sin métricas, logs, y traces es un sistema que no puedes operar.
   Implementar observabilidad desde el principio, no como afterthought.

8. Los documentos más valiosos son los runbooks.
   Cada hora invertida en un runbook ahorra horas de diagnóstico en producción.
   Escribir el runbook justo después de resolver un incidente.

9. La concurrencia correcta es difícil; la concurrencia operable es más difícil.
   El código que pasa todos los tests puede fallar de formas inesperadas en producción.
   El chaos testing y los property tests son la red de seguridad.

10. La habilidad más escasa es el juicio.
    Saber qué primitiva usar es el Cap.01.
    Saber cuándo no usarla es la experiencia de los Cap.02-17.
```

---

## Epílogo: el camino desde aquí

Este repositorio cubre las primitivas (Cap.01-07), el paralelismo y hardware (Cap.08-12),
cuatro lenguajes (Cap.13-16), y la arquitectura de producción (Cap.17).

Los temas que quedan fuera (y las referencias para continuar):

```
Sistemas distribuidos avanzados:
  - Consensus algorithms (Raft, Paxos)
  - Distributed transactions (2PC, Saga)
  - Conflict-free Replicated Data Types (CRDTs)
  Libro: "Designing Data-Intensive Applications" — Martin Kleppmann

Hardware avanzado:
  - SIMD / AVX programming
  - GPU programming (CUDA, Metal, WebGPU)
  - FPGA acceleration
  Libro: "Computer Architecture: A Quantitative Approach" — Patterson & Hennessy

Formal verification:
  - TLA+ para algoritmos distribuidos
  - Coq / Lean para proofs de correctness
  - Model checking con Spin/SPIN
  Paper: "Use of Formal Methods at Amazon Web Services" — Newcombe et al.

Real-time systems:
  - RTOS (FreeRTOS, Zephyr)
  - Rate Monotonic Scheduling
  - Priority ceiling protocol
  Libro: "Real-Time Systems" — Jane Liu

La lectura más importante:
  "The Art of Multiprocessor Programming" — Herlihy & Shavit
  El libro de texto definitivo para algoritmos concurrentes.
  Este repositorio es el prefacio. El libro de Herlihy es el texto completo.
```

**Fin del repositorio.**
