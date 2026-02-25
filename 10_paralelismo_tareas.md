# Guía de Ejercicios — Paralelismo de Tareas Heterogéneas

> Implementar cada ejercicio en: **Go · Java · Rust · Python**
> según corresponda.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el problema del trabajo desigual

El Cap.08 asumió que los items de trabajo son intercambiables: la suma de 1M números
divide el array en chunks iguales y cada worker hace la misma cantidad de trabajo.

En producción, el trabajo raramente es homogéneo:

```
Sistema de procesamiento de imágenes:
  Imagen pequeña (50 KB):   50ms para comprimir
  Imagen grande (50 MB):    50,000ms para comprimir   ← 1000x más

Compilador paralelo:
  Archivo simple (100 líneas):   5ms
  Archivo complejo (10,000 líneas, macros recursivas): 30,000ms ← 6000x más

Motor de búsqueda:
  Query "golang":        0.1ms (palabra común, índice pequeño)
  Query "the":           500ms (palabra muy común, índice enorme)
```

Con chunks iguales, el worker que recibió la imagen grande termina 1000 segundos
después que los otros. La paralelización fue peor que inútil — la latencia total
es idéntica a la del worker más lento.

```
Sin paralelismo:       t=100s (secuencial)
Con chunks iguales:    t=100s (el worker lento domina)
Con balanceo dinámico: t=12s  (8 workers, 8x speedup)
```

El balanceo dinámico requiere saber el costo de cada tarea antes de asignarla,
o adaptarse en tiempo real cuando resulta que una tarea es más cara de lo esperado.

---

## Tabla de contenidos

- [Sección 10.1 — Clasificación de tareas por costo](#sección-101--clasificación-de-tareas-por-costo)
- [Sección 10.2 — Priority queues y scheduling por prioridad](#sección-102--priority-queues-y-scheduling-por-prioridad)
- [Sección 10.3 — Speculative execution y cancelación anticipada](#sección-103--speculative-execution-y-cancelación-anticipada)
- [Sección 10.4 — Hedged requests y tail latency](#sección-104--hedged-requests-y-tail-latency)
- [Sección 10.5 — Dependency graphs y scheduling topológico](#sección-105--dependency-graphs-y-scheduling-topológico)
- [Sección 10.6 — Adaptive concurrency: ajustar dinámicamente](#sección-106--adaptive-concurrency-ajustar-dinámicamente)
- [Sección 10.7 — Casos reales: pipelines de ML y compilación](#sección-107--casos-reales-pipelines-de-ml-y-compilación)

---

## Sección 10.1 — Clasificación de Tareas por Costo

Antes de asignar una tarea, conviene saber cuánto va a costar.
Hay tres estrategias para estimar el costo:

```
1. Estimación estática (basada en atributos observables):
   - Tamaño del input: imagen de 50 MB probablemente es 1000x más costosa que 50 KB
   - Tipo de tarea: "compilar con optimizaciones" > "compilar sin optimizaciones"
   - Historial: este archivo tardó 30s la última vez

2. Estimación dinámica (medir durante la ejecución):
   - Medir el tiempo de las primeras N tareas, estimar las siguientes
   - Ajustar en tiempo real: si una tarea está tardando 10x el promedio, es "pesada"

3. Sin estimación (trabajo stealing puro):
   - No intentar predecir — solo reaccionar
   - Los workers ociosos roban trabajo de los ocupados
   - Simple pero efectivo cuando el costo varía mucho y es impredecible
```

**El tradeoff:**

```
Estimación precisa → mejor scheduling → menor latencia total
pero
Estimación costosa → overhead de medir → puede no valer la pena
```

Para muchos sistemas, la estimación estática por tamaño del input es suficiente
y tiene costo cero (el tamaño ya está disponible).

---

### Ejercicio 10.1.1 — Clasificador de tareas por histograma de costos

**Enunciado:** Implementa un clasificador que aprende la distribución de costos
de las tareas históricamente y las clasifica en "ligera", "media", y "pesada":

```go
type ClasificadorTareas struct {
    historial   []time.Duration   // últimas N duraciones
    ventana     int               // cuántas guardar
}

type Clase int
const (
    Ligera  Clase = iota  // percentil < 50
    Media                 // percentil 50-90
    Pesada                // percentil > 90
)

func (c *ClasificadorTareas) Registrar(duracion time.Duration)
func (c *ClasificadorTareas) Clasificar(estimacionTamaño int64) Clase
```

**Restricciones:** El historial usa una ventana deslizante de las últimas 1000 duraciones.
La clasificación por `estimacionTamaño` usa una regresión lineal simple sobre el historial.
El clasificador debe ser thread-safe para uso concurrente.

**Pista:** Una regresión lineal para esto es: `duracion_estimada = k * tamaño + b`,
donde `k` y `b` se calculan con los mínimos cuadrados sobre el historial.
Pero para la clasificación (no la predicción exacta), basta con los percentiles p50 y p90
del historial, sin importar el tamaño del input. La correlación tamaño-duración
se puede verificar después.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.1.2 — Separate queues por clase de tarea

**Enunciado:** Implementa un sistema con tres colas separadas (ligera, media, pesada)
y workers especializados. Los workers de tareas ligeras pueden "ayudar" con tareas
pesadas cuando están ociosos:

```go
type SchedulerMulticola struct {
    colaLigera chan Tarea
    colaMedia  chan Tarea
    colaPesada chan Tarea

    workersLigeros []*Worker  // especializados en ligeras
    workersPesados []*Worker  // especializados en pesadas
}

func (s *SchedulerMulticola) Enviar(t Tarea, clase Clase)
func (s *SchedulerMulticola) IniciarWorker(id int, clase Clase)
```

**Restricciones:** Un worker ligero ocioso revisa la cola pesada antes de bloquearse.
Un worker pesado no revisa la cola ligera — las tareas ligeras necesitan latencia baja.
Verifica que la latencia de tareas ligeras es consistente aunque haya muchas tareas pesadas.

**Pista:** El objetivo es SLO (Service Level Objective) diferenciado:
"las tareas ligeras terminan en < 100ms, las pesadas en < 60s".
Sin las colas separadas, una tarea ligera puede esperar detrás de 100 tareas pesadas.
Con las colas separadas, las tareas ligeras siempre tienen un camino libre.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 10.1.3 — Estimación de costo con muestreo adaptativo

**Enunciado:** Para tareas cuyo costo no se puede estimar por tamaño
(ej: queries a una BD donde la misma query puede tardar 1ms o 10s
dependiendo del estado de la BD), usa muestreo adaptativo:

```go
type EstimadorAdaptativo struct {
    ewma   float64  // Exponential Weighted Moving Average
    alpha  float64  // factor de suavizado: más alto = más reactivo
}

func (e *EstimadorAdaptativo) Actualizar(duracion time.Duration)
func (e *EstimadorAdaptativo) Estimar() time.Duration
func (e *EstimadorAdaptativo) EsAnomalia(duracion time.Duration) bool
```

**Restricciones:** El EWMA con α=0.1 da mucho peso al historial (90% historia, 10% nuevo dato).
Con α=0.9, reacciona rápido (10% historia, 90% nuevo dato). Implementa ambos
y mide cuál detecta mejor una regresión de rendimiento (cuando la BD empieza a ir lenta).

**Pista:** La anomalía se detecta con: `duracion > k * ewma` donde `k=2-3`.
Si una tarea tarda el triple del EWMA, es una anomalía — puede ser la señal de un
problema (BD lenta, timeout inminente, etc.) y puede merecer cancelación o retry.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.1.4 — Profiling online: medir el costo mientras se ejecuta

**Enunciado:** Un sistema de procesamiento de documentos no sabe el costo
de un documento hasta que lo está procesando. Implementa un "profiler online"
que mide el costo mientras ejecuta y ajusta el scheduling:

```go
type PerfiladorOnline struct {
    mu       sync.Mutex
    activos  map[TareaID]*PerfilTarea
}

type PerfilTarea struct {
    ID       TareaID
    Inicio   time.Time
    CPU      float64  // % uso de CPU hasta ahora
    Memoria  int64    // bytes en uso
}

func (p *PerfiladorOnline) Iniciar(id TareaID) *PerfilTarea
func (p *PerfiladorOnline) Terminar(id TareaID) PerfilTarea
func (p *PerfiladorOnline) TareasMasLentas(n int) []PerfilTarea
```

**Restricciones:** El overhead del profiler debe ser < 1% del tiempo de la tarea.
Las tareas más lentas en cualquier momento deben ser consultables en O(log N).

**Pista:** Para medir CPU de una goroutine específica en Go, no hay una API directa.
La aproximación es medir el tiempo de reloj y el tiempo de CPU del proceso completo
antes y después. Una mejor opción: usar `runtime.ReadMemStats()` para memoria
y `runtime/pprof` para CPU. El overhead del pprof sampling (~100 µs/sample) es aceptable.

**Implementar en:** Go · Java (`JMX ThreadMXBean`) · Python (`cProfile`) · Rust

---

### Ejercicio 10.1.5 — Bin packing: asignar tareas para maximizar utilización

**Enunciado:** El bin packing asigna tareas a workers para maximizar la utilización
sin sobrecargar ninguno. Es NP-duro en el caso general, pero heurísticas simples
funcionan bien en la práctica.

```go
type BinPacker struct {
    workers     []*Worker
    capacidad   time.Duration  // tiempo máximo por worker
}

// Heurística: First-Fit Decreasing (FFD)
// 1. Ordenar tareas por costo estimado (mayor primero)
// 2. Para cada tarea, asignar al primer worker que tenga espacio
func (b *BinPacker) Asignar(tareas []TareaConCosto) map[*Worker][]TareaConCosto
```

**Restricciones:** Implementa y compara tres heurísticas:
- First-Fit: asignar al primer worker disponible
- First-Fit Decreasing: primero las tareas pesadas
- Best-Fit: asignar al worker con menos espacio que aún puede alojar la tarea

Mide el makespan (tiempo total de completitud) para cada heurística.

**Pista:** FFD es la heurística clásica — garantiza una utilización > 11/9 del óptimo.
Para inputs de producción (distribución Pareto de costos), FFD suele estar
dentro del 10-15% del óptimo. La complejidad de los algoritmos óptimos
(ILP, branch-and-bound) raramente justifica su uso sobre FFD.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 10.2 — Priority Queues y Scheduling por Prioridad

No todas las tareas son igualmente urgentes. Una request de un usuario de pago
merece más recursos que una tarea de análisis en batch.

**Los tres modelos de prioridad:**

```
1. Prioridad fija (static priority):
   Las tareas tienen una prioridad asignada que no cambia.
   Problema: las tareas de baja prioridad pueden esperar para siempre (starvation).

2. Prioridad dinámica (aging):
   La prioridad de una tarea aumenta con el tiempo que lleva esperando.
   Resuelve la starvation: eventualmente, cualquier tarea tiene prioridad máxima.

3. Deadline scheduling:
   Las tareas tienen un deadline y la prioridad se calcula para que todas
   terminen antes de su deadline (si es posible).
   EDF (Earliest Deadline First) es óptimo para sistemas monoprocesador.
```

---

### Ejercicio 10.2.1 — Priority queue thread-safe

**Enunciado:** Implementa una priority queue thread-safe que soporte
prioridades de 0-9 (0 más alta) con aging:

```go
type PriorityQueue struct {
    colas    [10]chan Tarea  // una por prioridad
    aging    time.Duration  // cada cuánto subir la prioridad
}

func (pq *PriorityQueue) Push(t Tarea, prioridad int)
func (pq *PriorityQueue) Pop(ctx context.Context) (Tarea, int, error)
// Pop retorna la tarea de mayor prioridad disponible
// Aging: las tareas esperando más de `aging` suben una prioridad
```

**Restricciones:** El aging debe ocurrir en background — no en el crítico path de Pop.
Una goroutine de aging revisa las colas periódicamente y mueve tareas.
Las tareas que suben de prioridad mantienen su posición relativa dentro del nuevo nivel.

**Pista:** El aging con canales es: periódicamente, leer hasta N tareas del canal
de baja prioridad y reencolarlas en el canal de alta prioridad.
El número N es importante: si es demasiado grande, el aging goroutine monopoliza
el acceso a los canales. Un buen N: min(len(canal), 10).

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 10.2.2 — EDF scheduling: Earliest Deadline First

**Enunciado:** EDF asigna prioridad a la tarea cuyo deadline es el más próximo.
Es óptimo para un procesador — si hay una forma de cumplir todos los deadlines, EDF la encuentra.

```go
type TareaConDeadline struct {
    ID       TareaID
    Costo    time.Duration  // tiempo estimado de ejecución
    Deadline time.Time
    Ejecutar func()
}

type SchedulerEDF struct {
    heap  *MinHeap[TareaConDeadline]  // ordenado por deadline
    mu    sync.Mutex
}

func (s *SchedulerEDF) Agregar(t TareaConDeadline)
func (s *SchedulerEDF) Siguiente() (TareaConDeadline, bool)
func (s *SchedulerEDF) HayViolacion() bool  // ¿alguna tarea no puede cumplir su deadline?
```

**Restricciones:** `HayViolacion` detecta si el schedule actual puede cumplir todos los
deadlines asumiendo que las tareas se ejecutan secuencialmente.
Si hay violación, reporta cuál tarea es la que falla primero.

**Pista:** La prueba de cumplimiento de deadlines con EDF es: para cada tarea en orden
de deadline, verificar que el tiempo acumulado de todas las tareas anteriores
más el costo de esta tarea ≤ su deadline. Si alguna falla, hay violación.
En sistemas de tiempo real críticos (aviónica, marcapasos), esta verificación
se hace en el diseño, no en runtime.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 10.2.3 — Starvation prevention: Fair Queuing

**Enunciado:** El Fair Queuing (FQ) divide el bandwidth entre múltiples consumers
de forma justa — ninguno recibe más que su parte proporcional y ninguno
recibe menos.

```go
// Weighted Fair Queue — distintos clientes tienen distinto peso
type WeightedFairQueue struct {
    clientes map[ClienteID]*ColaCliente
}

type ColaCliente struct {
    tareas  []Tarea
    peso    float64  // qué fracción del bandwidth recibe
    credito float64  // crédito acumulado
}

func (wfq *WeightedFairQueue) Agregar(clienteID ClienteID, t Tarea)
func (wfq *WeightedFairQueue) Siguiente() (Tarea, ClienteID)
```

**Restricciones:** Con dos clientes de peso 2 y 1, el cliente de peso 2 debe recibir
el doble de throughput que el de peso 1, medido en ventanas de 100 tareas.
Si un cliente no tiene tareas, su parte va a los demás (no se desperdicia).

**Pista:** El algoritmo Deficit Round Robin (DRR) es simple y eficiente para FQ.
Cada cliente tiene un "deficit" que se incrementa en `quantum * peso` por ronda.
El cliente usa su deficit para procesar tareas. Si no alcanza para una tarea,
el deficit se acumula para la siguiente ronda.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.2.4 — Token bucket + priority: rate limiting con prioridades

**Enunciado:** Combina rate limiting (Token Bucket del Cap.03 §3.7) con
prioridades: las requests de alta prioridad pueden "tomar prestados" tokens
de las requests de baja prioridad:

```go
type RateLimiterPrioritario struct {
    buckets [3]*TokenBucket  // uno por nivel de prioridad
    global  *TokenBucket     // bucket global compartido
}

// Orden de consumo:
// 1. Intentar el bucket de la prioridad de la request
// 2. Si está vacío y la prioridad es alta, intentar el bucket global
// 3. Si el global también está vacío, rechazar (prioridad baja/media) o esperar (alta)
func (r *RateLimiterPrioritario) Permitir(prioridad int) bool
```

**Restricciones:** Las requests de prioridad alta nunca deben esperar si hay tokens
disponibles en cualquier bucket. Las de baja prioridad nunca consumen tokens
del bucket de alta prioridad.

**Pista:** El token bucket con prioridades es exactamente lo que usan sistemas como
AWS API Gateway y Kubernetes. En Kubernetes, los pods tienen garantías (requests)
y límites: los recursos de pods que no usan sus garantías van al pool compartido
para pods que quieren más que sus garantías.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 10.2.5 — Priority inversion: el bug que mató el Mars Pathfinder

**Enunciado:** En 1997, el rover Mars Pathfinder experimentaba reinicios inexplicables.
La causa: priority inversion — una tarea de alta prioridad bloqueada esperando
un mutex que tenía una tarea de baja prioridad, mientras una tarea de prioridad
media impedía que la tarea baja terminara.

```
Alta prioridad (meteorología): espera mutex M
Media prioridad (comunicaciones): corre, no necesita mutex M
Baja prioridad (telemetría): tiene mutex M, quiere terminar pero...
                             ...el scheduler prefiere la media sobre la baja
→ Alta espera a media, que no termina porque no tiene relación con alta → deadlock efectivo
```

**Solución: Priority Inheritance** — cuando una tarea baja tiene un mutex que
necesita una tarea alta, la tarea baja temporalmente hereda la prioridad alta.

Implementa el escenario del Pathfinder y luego la corrección con priority inheritance.

**Restricciones:** El escenario debe demostrar el problema con mediciones de tiempo:
la tarea alta debe tardar > 10x más de lo esperado debido a la inversión.
Con priority inheritance, debe tardar dentro de 2x del tiempo esperado.

**Pista:** En Go, los mutexes no tienen priority inheritance (ni la mayoría de las implementaciones).
Para simular priority inheritance, necesitas un mutex personalizado que, al ser adquirido
por una goroutine de baja prioridad, registra que una goroutine de alta prioridad está esperando,
y ajusta el scheduling (usando runtime.Gosched de forma controlada).

**Implementar en:** Go · Java (`PriorityQueue` + `ReentrantLock`) · C (pthreads con `PTHREAD_PRIO_INHERIT`)

---

## Sección 10.3 — Speculative Execution y Cancelación Anticipada

La ejecución especulativa inicia más trabajo del estrictamente necesario,
apostando a que parte de ese trabajo será útil.

**Los tres patrones de speculative execution:**

```
1. Prefetch especulativo:
   Antes de que el usuario pida los resultados de la página 2,
   ya estás calculando la página 2 especulativamente.
   Si el usuario va a la página 2: el prefetch fue útil.
   Si el usuario sale: cancelas el cálculo especulativo.

2. Alternativas en paralelo:
   Para una query ambigua, ejecutar múltiples interpretaciones en paralelo.
   Retornar el resultado de la primera que termina.
   Cancelar las demás.

3. Lookahead en compiladores:
   Compilar funciones que aún no han sido llamadas, especulando que
   eventualmente lo serán. Si la especulación era incorrecta, descarta el trabajo.
```

**El costo de la especulación:**

```
Recursos gastados en trabajo que se descarta:
  Si el 50% de las especulaciones son inútiles → 50% de recursos desperdiciados
  Si el 10% son inútiles → 10% de recursos desperdiciados
  Si el 1% son inútiles → overhead aceptable para muchos sistemas
```

---

### Ejercicio 10.3.1 — Prefetch especulativo para paginación

**Enunciado:** Un sistema de búsqueda retorna resultados paginados.
Cuando el usuario solicita la página N, empieza especulativamente a calcular
la página N+1 y N+2:

```go
type BuscadorEspeculativo struct {
    ejecutor  *WorkerPool
    prefetch  int          // cuántas páginas prefetchear
    cache     *sync.Map    // pageKey → resultado especulativo
}

func (b *BuscadorEspeculativo) Buscar(query string, pagina int) Resultados {
    // 1. Calcular la página solicitada (o servir del caché especulativo)
    // 2. Lanzar especulativamente las páginas N+1, N+2, ...
    // 3. Los especulativos van al caché si terminan antes de ser pedidos
}
```

**Restricciones:** Los cálculos especulativos se cancelan si el usuario
no pide la página en 30 segundos. El caché especulativo tiene un límite de
tamaño (evitar acumular especulaciones inútiles).

**Pista:** El patrón es: al servir la página N, lanzar goroutines para N+1 y N+2
con context.WithTimeout(30s). Si el usuario pide N+1 antes de que expire el contexto,
el resultado ya está en caché. Si no pide N+1, el contexto expira y la goroutine termina.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 10.3.2 — Or-done con múltiples candidatos

**Enunciado:** Para una operación de alta importancia, ejecutar la misma request
en múltiples backends en paralelo y retornar el resultado del más rápido
(cancelando los demás):

```go
func BuscarEnParalelo(backends []Backend, query string, timeout time.Duration) (Resultado, error) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    resultados := make(chan Resultado, len(backends))
    errores := make(chan error, len(backends))

    for _, backend := range backends {
        go func(b Backend) {
            r, err := b.Buscar(ctx, query)
            if err != nil {
                errores <- err
                return
            }
            resultados <- r
        }(backend)
    }

    // Retornar el primer resultado exitoso
    // Si todos fallan, retornar el último error
}
```

**Restricciones:** Si el primer resultado exitoso llega, cancelar todas las demás
goroutines inmediatamente. Si N backends fallan y queda 1, esperar ese 1.
Si todos fallan, retornar error con contexto de todos los fallos.

**Pista:** El `select` sobre múltiples canales es exactamente el mecanismo de Go para esto.
La cancelación del contexto propaga automáticamente a todos los backends que la respetan.
El resultado es similar al `Promise.race()` de JavaScript o `CompletableFuture.anyOf()` de Java.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 10.3.3 — Eager vs lazy evaluation bajo concurrencia

**Enunciado:** La evaluación eager (calcular inmediatamente) vs lazy (calcular cuando se pide)
tiene consecuencias diferentes en contextos concurrentes:

```go
// Eager: calcula todos los thumbnails al subir la imagen
func SubirImagenEager(img Imagen) {
    guardar(img)
    generarThumbnail(img, 128)   // bloquea el upload
    generarThumbnail(img, 256)   // bloquea el upload
    generarThumbnail(img, 512)   // bloquea el upload
}

// Lazy: calcula el thumbnail cuando se pide por primera vez
func ObtenerThumbnail(imgID string, tamaño int) []byte {
    clave := fmt.Sprintf("%s-%d", imgID, tamaño)
    if cached, ok := cache.Get(clave); ok {
        return cached
    }
    img := cargar(imgID)
    thumb := generarThumbnail(img, tamaño)
    cache.Set(clave, thumb)
    return thumb
}

// Eager especulativo: calcular en background después del upload
func SubirImagenEspeculativo(img Imagen) {
    guardar(img)
    go func() {
        // En background, sin bloquear
        for _, tamaño := range []int{128, 256, 512} {
            generarThumbnail(img, tamaño)
        }
    }()
}
```

**Restricciones:** Mide latencia del upload, latencia del primer thumbnail, y
uso de recursos para las tres estrategias con 100 uploads simultáneos.
La estrategia óptima depende del patrón de acceso a thumbnails — documenta cuándo
cada estrategia es mejor.

**Pista:** Si el 90% de los thumbnails subidos nunca se ven, lazy es mejor (no desperdicias CPU).
Si el 90% se ven en los primeros 10 minutos, eager especulativo es mejor (amortizas el costo).
Si no sabes el patrón, lazy + caché es la opción conservadora.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 10.3.4 — Timeout progresivo: el retry que aprende

**Enunciado:** Un cliente que llama a un servicio externo tiene que decidir
cuánto esperar. Demasiado poco: falla innecesariamente. Demasiado: bloquea recursos.

Implementa un timeout progresivo que se adapta basándose en el historial de latencias:

```go
type TimeoutAdaptativo struct {
    historial  []time.Duration
    percentil  float64  // qué percentil usar (ej: 0.99 para p99)
    multiplicador float64  // margen de seguridad (ej: 1.5 para 1.5x p99)
}

func (t *TimeoutAdaptativo) Registrar(duracion time.Duration)
func (t *TimeoutAdaptativo) Timeout() time.Duration
// Timeout = percentil(historial) * multiplicador
// Con un mínimo razonable y un máximo configurado
```

**Restricciones:** El timeout nunca baja de `minTimeout` ni sube de `maxTimeout`.
El historial tiene ventana deslizante de 1000 mediciones.
Si el servicio no ha tenido llamadas recientes, usar el timeout conservador (máximo).

**Pista:** El timeout basado en p99 es más robusto que el basado en la media.
Si la media es 100ms pero hay spikes de 500ms al p99, un timeout de 150ms (1.5 * media)
fallaría el 1% de las requests. Con 1.5 * p99 = 750ms, el fallo es insignificante.
Este es el approach de Google Dapper y muchos sistemas de observabilidad de latencia.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.3.5 — Speculative retry: no esperar el timeout completo

**Enunciado:** En lugar de esperar el timeout para hacer retry, lanza una segunda
request cuando la primera tarda más de cierto umbral, y usa la primera respuesta
que llegue (cancelando la otra):

```go
type RetryEspeculativo struct {
    cliente      Cliente
    umbralRetry  time.Duration  // cuándo lanzar la segunda request
    maxIntento   int
}

func (r *RetryEspeculativo) Ejecutar(ctx context.Context, req Request) (Response, error) {
    // Lanzar primera request
    // Si tarda > umbralRetry, lanzar segunda request en paralelo
    // Retornar la primera respuesta exitosa
    // Cancelar la otra
}
```

**Restricciones:** El `umbralRetry` debería ser el percentil 95 de la latencia normal.
Si la primera request tarda más de p95, es probable que tenga un problema —
el retry especulativo reduce la latencia p99 significativamente.
El overhead es: ~5% de las requests hacen dos llamadas al backend.

**Pista:** Google encontró que el retry especulativo (hedged requests) reduce la latencia
de operaciones largas en un 99% del tiempo. La idea fue presentada en el paper
"The Tail at Scale" (Jeffrey Dean, 2013) — uno de los papers más influyentes sobre
latency optimization en sistemas distribuidos.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 10.4 — Hedged Requests y Tail Latency

La "tail latency" es la latencia del percentil alto (p99, p999). Aunque la latencia
media sea baja, el p99 puede ser 10-100x mayor.

**Por qué la tail latency importa:**

```
Sistema con 10 microservicios en cadena, cada uno con p99 = 100ms:
  La probabilidad de que ALGUNO esté en el p99:
  P(alguno lento) = 1 - P(todos rápidos) = 1 - 0.99^10 ≈ 10%

  Así que el sistema completo tiene ~10% de requests en el p99 de 100ms.
  Con 100 microservicios: 1 - 0.99^100 ≈ 63% de requests experimentan latencia alta.
```

Los hedged requests son la solución de Google para el tail latency problem.

**La matemática de los hedged requests:**

```
Latencia de 1 request:   CDF(t) = P(respuesta llega antes de t)
Latencia de 2 requests:  P(primera llega antes de t) = 1 - (1 - CDF(t))^2

Con P99 = 100ms:
  1 request:  P(< 100ms) = 0.99, P(> 100ms) = 0.01
  2 requests: P(alguna < 100ms) = 1 - (0.01)^2 = 0.9999
  → El p99 de la latencia con hedged requests es ≈ el p99.9999 de la latencia individual
```

---

### Ejercicio 10.4.1 — Implementar hedged requests

**Enunciado:** Implementa el patrón de hedged requests descrito en el paper
"The Tail at Scale" de Google:

```go
type HedgedRequester struct {
    cliente      Cliente
    hedgeDelay   time.Duration  // cuando lanzar la segunda request
    maxHedges    int            // máximo de requests paralelas
}

func (h *HedgedRequester) Ejecutar(ctx context.Context, req Request) (Response, error) {
    // Primera request lanzada inmediatamente
    // Si no responde en hedgeDelay, lanzar segunda request
    // Si tampoco responde, lanzar tercera (si maxHedges >= 3)
    // Retornar la primera respuesta exitosa
}
```

**Restricciones:** `hedgeDelay` debe calibrarse al p95 de la latencia normal.
Las requests adicionales (hedges) se cancelan cuando llega la primera respuesta.
El sistema backend debe poder manejar el overhead (hasta `maxHedges` veces más requests).

**Pista:** Con `hedgeDelay = p95`, el ~5% de las requests lanza un hedge.
El overhead en el backend es: 5% de requests tiene 2x llamadas = 5% overhead total.
La reducción de tail latency: el p99 baja al p95 (las requests que antes eran lentas
ahora tienen una segunda oportunidad).

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.4.2 — Medir y visualizar la distribución de latencia

**Enunciado:** Para entender la tail latency, necesitas visualizar la distribución
completa, no solo la media:

```go
type HistogramaLatencia struct {
    buckets   []int64     // conteo por bucket exponencial
    minBucket time.Duration
    factor    float64     // base del crecimiento exponencial (ej: 1.5)
    // Bucket 0: [0, minBucket)
    // Bucket 1: [minBucket, minBucket*factor)
    // Bucket k: [minBucket*factor^(k-1), minBucket*factor^k)
}

func (h *HistogramaLatencia) Registrar(d time.Duration)
func (h *HistogramaLatencia) Percentil(p float64) time.Duration
func (h *HistogramaLatencia) Imprimir()  // ASCII art del histograma
```

**Restricciones:** El histograma usa 50 buckets exponenciales de 1µs a 10s.
Los percentiles p50, p75, p90, p95, p99, p999 deben estar disponibles.
Implementar el ASCII art para visualización en la terminal.

**Pista:** Los histogramas de latencia con buckets exponenciales (HdrHistogram, Prometheus)
son el estándar de la industria porque tienen precisión relativa constante (1-5%)
en todos los rangos. Los buckets lineales darían baja precisión para latencias pequeñas
o un número enorme de buckets.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 10.4.3 — Latency SLO: cuándo rechazar trabajo para proteger el p99

**Enunciado:** Un servicio con SLO de p99 < 100ms puede necesitar rechazar requests
cuando está sobrecargado para mantener el p99, en lugar de aceptar todo y degradarse:

```go
type AdmisionControlSLO struct {
    histograma  *HistogramaLatencia
    sloP99      time.Duration
    umbralCarga float64  // rechazar si la carga supera este umbral
}

func (a *AdmisionControlSLO) PuedeAdmitir() bool {
    // Si el p99 actual está cerca del SLO y la carga es alta,
    // rechazar para proteger el SLO de las requests en vuelo
}
```

**Restricciones:** El control de admisión debe ser proactivo — rechazar antes de
que el p99 supere el SLO, no después. Usa la ecuación de Little's Law:
`latencia_media = requests_en_vuelo / throughput`.

**Pista:** La Ley de Little: L = λ × W. Con λ = requests/s, W = latencia media,
L = requests en vuelo. Si L supera la capacidad (workers × tiempo_servicio),
el sistema entra en sobrecarga y la latencia crece. El control de admisión
rechaza requests cuando L se acerca al límite.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.4.4 — Comparing backends: el A/B test concurrente

**Enunciado:** Durante una migración de sistema, quieres enviar el mismo request
a dos backends (viejo y nuevo) en paralelo para comparar resultados:

```go
type ComparadorBackends struct {
    viejo  Backend
    nuevo  Backend
    log    ComparisonLogger
}

func (c *ComparadorBackends) Ejecutar(ctx context.Context, req Request) (Response, error) {
    // 1. Enviar request a ambos backends en paralelo
    // 2. Usar el resultado del backend VIEJO (fuente de verdad)
    // 3. Comparar el resultado del nuevo con el viejo (en background)
    // 4. Registrar diferencias
    // Sin impactar la latencia del usuario (el nuevo corre en paralelo)
}
```

**Restricciones:** La latencia del usuario debe ser idéntica a usar solo el backend viejo.
El nuevo backend corre en un goroutine separado — si tarda más, no bloquea al usuario.
Las diferencias se registran con suficiente contexto para debuggear.

**Pista:** Esta técnica se llama "shadow traffic" o "dark launching".
GitHub la usa para migrar entre sistemas. La clave: el backend nuevo puede fallar
o divergir sin que el usuario lo vea. Solo cuando el nuevo backend es correcto
al 99.9%+ de los casos se migra el tráfico real.

**Implementar en:** Go · Java · Python

---

### Ejercicio 10.4.5 — Percentile tracking con decaimiento temporal

**Enunciado:** Los percentiles calculados sobre todo el historial histórico
son menos útiles que los calculados sobre la ventana reciente. Si el sistema
mejoró hace 1 hora, el p99 histórico aún refleja el rendimiento antiguo.

Implementa un tracker de percentiles con decaimiento exponencial:

```go
type PercentilesDecaimiento struct {
    muestras []MuestraDecaimiento
    halfLife time.Duration  // cada halfLife, el peso de las muestras viejas se reduce a la mitad
}

type MuestraDecaimiento struct {
    valor  time.Duration
    tiempo time.Time
}

func (p *PercentilesDecaimiento) Registrar(d time.Duration)
func (p *PercentilesDecaimiento) Percentil(pct float64) time.Duration
// El peso de cada muestra es: exp(-ln(2) * edad / halfLife)
```

**Restricciones:** Con halfLife = 5 minutos, las muestras de hace 5 minutos
tienen la mitad del peso de las recientes. Las de hace 10 minutos, un cuarto. etc.
Implementa limpieza periódica de muestras con peso < 0.01 (prácticamente irrelevantes).

**Pista:** Los percentiles con decaimiento son más complejos de calcular que
los percentiles exactos porque los pesos cambian con el tiempo. Una aproximación
práctica: muestrear datos recientes con más frecuencia (reservoir sampling con decaimiento).
`t-digest` es una estructura de datos eficiente para esto, disponible en Java y Go.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 10.5 — Dependency Graphs y Scheduling Topológico

Muchas tareas tienen dependencias: la tarea B no puede empezar hasta que
termine la tarea A. El scheduling topológico determina el orden óptimo de ejecución.

**El DAG de tareas:**

```
Compilar proyecto Go:
  A: compilar pkg/util    ──┐
  B: compilar pkg/core    ──┼─→ D: compilar cmd/server ──→ F: link
  C: compilar pkg/models  ──┘       ↑
                                    E: compilar cmd/worker ──→ F: link

El critical path (camino más largo) determina el tiempo mínimo:
  A + D + F = 5s + 3s + 2s = 10s (si A es el más lento)
  B + D + F = 3s + 3s + 2s = 8s
  C + D + F = 4s + 3s + 2s = 9s
  → Critical path: A → D → F = 10s mínimo, aunque todo lo demás sea paralelo
```

---

### Ejercicio 10.5.1 — DAG scheduler con detección de ciclos

**Enunciado:** Implementa un scheduler de tareas con dependencias usando un DAG:

```go
type DAGScheduler struct {
    tareas  map[TareaID]*Tarea
    deps    map[TareaID][]TareaID  // tarea → sus dependencias
    pool    *WorkerPool
}

func (d *DAGScheduler) Agregar(t *Tarea, dependencias []TareaID) error
func (d *DAGScheduler) Ejecutar(ctx context.Context) error
// Ejecutar respeta el orden topológico y paraleliza donde sea posible
// Detectar ciclos en las dependencias → retornar error
```

**Restricciones:** La detección de ciclos ocurre al añadir tareas (eager),
no al ejecutar (lazy). Las tareas sin dependencias (raíces del DAG) se ejecutan
en paralelo inmediatamente. Las tareas cuyos deps completaron se ejecutan
en cuanto haya un worker disponible.

**Pista:** La detección de ciclos usa DFS con coloración (blanco/gris/negro).
La ejecución paralela usa un contador de dependencias pendientes por tarea:
cuando llega a 0, la tarea se encola para ejecución.
Al completar una tarea, decrementar el contador de sus dependientes — los que lleguen a 0 se encolan.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.5.2 — Critical path analysis

**Enunciado:** El critical path es el camino más largo en el DAG — determina
el tiempo mínimo de ejecución independientemente de cuántos workers haya.

```go
func CriticalPath(dag *DAGScheduler) []TareaID {
    // Retorna las tareas del critical path en orden de ejecución
}

func TiempoMinimoEjecucion(dag *DAGScheduler) time.Duration {
    // Suma de los costos de las tareas en el critical path
}
```

**Restricciones:** El análisis del critical path debe funcionar aunque los costos
de las tareas sean desconocidos (estimados) — acepta un costo estimado por tarea.
Implementa la visualización del critical path en ASCII art mostrando el Gantt chart.

**Pista:** El critical path se calcula con programación dinámica en el DAG:
1. Ordenar topológicamente
2. Para cada nodo, `tiempo_más_tarde = max(tiempo_más_tarde de dependencias) + costo_propio`
3. Las tareas con `tiempo_más_tarde == tiempo_total` están en el critical path

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.5.3 — Make paralelo: el caso de uso real

**Enunciado:** `make` analiza un Makefile y ejecuta los targets en orden
topológico, paralelizando los que no tienen dependencias entre sí.
Implementa un `make` simplificado:

```go
type MakeParalelo struct {
    targets map[string]*Target
}

type Target struct {
    Nombre  string
    Deps    []string
    Comando string
    Hash    string  // hash de los archivos de input — para saber si recompilar
}

func (m *MakeParalelo) Leer(makefile string) error
func (m *MakeParalelo) Build(target string, jobs int) error
// `jobs` es el número de targets en paralelo (equivalente a make -j)
```

**Restricciones:** Solo recompilar si el hash de los inputs cambió o si
alguna dependencia fue recompilada. La salida de cada comando se imprime
en orden (no intercalada) incluso con ejecución paralela.

**Pista:** La salida ordenada requiere bufferizar los outputs de cada comando
y imprimirlos cuando el comando termina, en el orden del DAG (no el orden de
completitud). Esto es exactamente lo que hace `make -j -O` en GNU make.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.5.4 — Pipeline con fanout y fanin dinámicos

**Enunciado:** Un pipeline donde el número de tareas downstream cambia
según el resultado de upstream:

```go
// Leer un documento → extraer N imágenes → procesar cada imagen
// El número de imágenes no se conoce hasta leer el documento

type Pipeline struct {
    etapas []Etapa
}

type Etapa interface {
    Procesar(ctx context.Context, input interface{}) ([]interface{}, error)
    // Retorna 0 o más outputs por input — fanout dinámico
}
```

**Restricciones:** Las etapas se ejecutan en pipeline (streaming), no batch.
Un documento se procesa en todas las etapas antes de empezar el siguiente.
Las etapas con fanout > 1 generan trabajo adicional que se procesa en paralelo.

**Pista:** El fanout dinámico requiere que cada etapa pueda retornar múltiples
valores. El pipeline se implementa con un DAG donde los nodos de "fanout"
generan múltiples nodos hijos en tiempo de ejecución (un DAG dinámico).
Esto es la base de sistemas como Apache Spark y Apache Flink.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.5.5 — Scheduling de tareas con recursos compartidos

**Enunciado:** Las tareas tienen requisitos de recursos que no son solo tiempo:
algunas necesitan 4 GB de RAM, otras 1 GPU, otras 8 cores.
El scheduler debe respetar los límites de recursos disponibles:

```go
type RecursosTarea struct {
    RAM   int64    // bytes
    Cores int
    GPU   bool
}

type SchedulerRecursos struct {
    disponibles RecursosTarea
    activos     []*Tarea
    mu          sync.Mutex
}

func (s *SchedulerRecursos) Agregar(t *Tarea, recursos RecursosTarea)
func (s *SchedulerRecursos) Ejecutar(ctx context.Context) error
// Solo ejecutar una tarea si hay recursos suficientes disponibles
```

**Restricciones:** El scheduler maximiza la utilización de recursos sin violar
los límites. Una tarea esperando no debe bloquear a otras que sí tienen recursos.

**Pista:** Este es exactamente el modelo de Kubernetes. Los pods tienen `requests`
(recursos mínimos garantizados) y `limits` (máximo que pueden usar).
El scheduler de Kubernetes asigna pods a nodos verificando que los requests
caben en los recursos disponibles del nodo. La estrategia bin-packing del §10.1.5
aplica aquí directamente.

**Implementar en:** Go · Java · Python

---

## Sección 10.6 — Adaptive Concurrency: Ajustar Dinámicamente

El número óptimo de goroutines concurrentes no es constante — depende
de la carga del sistema, la latencia de las dependencias, y la disponibilidad de recursos.

**El problema del número fijo de workers:**

```
Workers fijos = 10:
  Con carga baja (2 req/s):   8 workers ociosos — waste
  Con carga media (10 req/s): 10 workers ocupados — óptimo
  Con carga alta (20 req/s):  10 workers saturados, cola creciendo — problema
  Con backend lento:          10 workers bloqueados esperando — throughput cae
```

**AIMD (Additive Increase, Multiplicative Decrease) — el algoritmo de TCP:**

```
Cuando todo va bien:    workers += 1  (additive increase)
Cuando hay errores:     workers /= 2  (multiplicative decrease)

Propiedades:
  - Reactivo a problemas (reduce rápido)
  - Conservador en la recuperación (sube lento)
  - Estable en estado estacionario (oscila alrededor del óptimo)
```

---

### Ejercicio 10.6.1 — Adaptive concurrency con AIMD

**Enunciado:** Implementa un controlador de concurrencia que usa AIMD:

```go
type ControladorConcurrencia struct {
    min, max    int
    actual      atomic.Int64
    errorRate   *VentanaDeslizante
    latencia    *PercentilesDecaimiento
}

func (c *ControladorConcurrencia) Ajustar() {
    errorRate := c.errorRate.Tasa()
    p99 := c.latencia.Percentil(0.99)
    p99SLO := 100 * time.Millisecond

    if errorRate > 0.01 || p99 > p99SLO {
        // Problemas: reducir concurrencia (multiplicative decrease)
        nuevo := max(c.min, int(float64(c.actual.Load()) * 0.5))
        c.actual.Store(int64(nuevo))
    } else {
        // Todo bien: incrementar concurrencia (additive increase)
        nuevo := min(c.max, int(c.actual.Load()) + 1)
        c.actual.Store(int64(nuevo))
    }
}
```

**Restricciones:** El ajuste ocurre periódicamente (cada 1s) en background.
Los cambios se aplican gradualmente — no matar todas las goroutines extra de golpe.

**Pista:** Netflix Concurrency Limits y Google's Square use exactamente este patrón.
La biblioteca `resilience4j` para Java lo implementa.
La clave es que el ajuste sea suave — cambiar el límite de 100 a 50 de golpe
produce un spike de timeouts. Reducir de 100 a 90, luego 80, etc. es más estable.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.6.2 — Gradient descent para encontrar el óptimo de concurrencia

**Enunciado:** El gradient descent puede encontrar el número óptimo de workers
experimentando sistemáticamente:

```go
type OptimizadorConcurrencia struct {
    actual    int
    paso      int         // cuánto cambiar en cada step
    metrica   func() float64  // throughput u otro indicador a maximizar
    historial [][2]float64    // (workers, metrica)
}

func (o *OptimizadorConcurrencia) Optimizar(ctx context.Context) {
    // 1. Medir throughput con `actual` workers
    // 2. Probar con `actual+paso` workers
    // 3. Si mejoró, continuar en esa dirección
    // 4. Si empeoró, reducir el paso y cambiar de dirección
    // 5. Cuando el paso es pequeño, hemos encontrado el óptimo (local)
}
```

**Restricciones:** El optimizador no debe degradar el sistema durante la búsqueda —
solo hace cambios pequeños y revierte si la métrica empeora.
El óptimo encontrado puede ser local, no global.

**Pista:** Hill climbing (gradient ascent) para concurrencia:
empieza con los workers actuales, experimenta con +1 y -1,
mueve hacia donde la métrica es mejor. Converge al óptimo local en O(distancia_al_óptimo).
El riesgo: si hay dos óptimos locales (raro en la práctica), puede quedarse en el peor.

**Implementar en:** Go · Java · Python

---

### Ejercicio 10.6.3 — Bulkhead pattern: aislar fallas entre pools

**Enunciado:** El bulkhead pattern (de los compartimentos estancos de un barco)
asigna pools separados de workers a distintos clientes o funcionalidades.
Si un cliente genera carga excesiva, solo afecta a su pool — no degrada a los demás:

```go
type BulkheadManager struct {
    pools map[string]*WorkerPool  // un pool por cliente/funcionalidad
    mu    sync.RWMutex
}

func (b *BulkheadManager) Pool(nombre string) *WorkerPool
func (b *BulkheadManager) AgregarPool(nombre string, workers int, capacidad int)
func (b *BulkheadManager) EliminarPool(nombre string)
```

**Restricciones:** Cada pool tiene su propio límite de workers y de cola.
Si un pool está lleno, retorna error — no afecta a otros pools.
Los pools pueden tener distinto tamaño (clientes premium tienen pools más grandes).

**Pista:** El bulkhead es fundamental en sistemas multi-tenant.
Sin él, un cliente que hace 10,000 requests/s puede saturar todos los workers
y causar timeouts para todos los demás clientes.
Con bulkhead, cada cliente tiene su "cuota" de workers garantizada.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 10.6.4 — Feedback loop: ajustar concurrencia según el backend

**Enunciado:** Implementa un feedback loop que ajusta la concurrencia
basándose en la salud del backend:

```go
type MonitorBackend struct {
    healthCheck  func() HealthStatus
    concurrencia *ControladorConcurrencia
    ticker       *time.Ticker
}

type HealthStatus struct {
    Latencia  time.Duration
    ErrorRate float64
    QueueSize int
}

func (m *MonitorBackend) Monitorear(ctx context.Context) {
    for {
        select {
        case <-m.ticker.C:
            status := m.healthCheck()
            m.ajustar(status)
        case <-ctx.Done():
            return
        }
    }
}
```

**Restricciones:** Si el backend reporta cola grande o latencia alta,
reducir la concurrencia inmediatamente. Si el backend está sano, aumentar gradualmente.
El ajuste es asimétrico: reducir rápido (para no empeorar), aumentar lento (conservador).

**Pista:** Este es el modelo del TCP congestion control llevado al nivel de aplicación.
TCP reduce a la mitad cuando detecta congestión (packet loss) y aumenta linealmente
cuando todo va bien. La misma asimetría (reducir rápido, aumentar lento)
es exactamente la correcta para control de flujo a nivel de aplicación.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.6.5 — Comparar: fijo vs adaptive vs gradient

**Enunciado:** Para un sistema con carga variable (baja de día, alta de noche,
spikes en fin de semana), compara las tres estrategias de control de concurrencia:

1. **Fijo**: 10 workers siempre
2. **Adaptive AIMD**: ajusta basándose en errores y latencia
3. **Gradient**: optimiza continuamente el throughput

Métricas a comparar:
- Throughput promedio (requests completadas por segundo)
- Latencia p99
- Tasa de errores
- Utilización de workers (% del tiempo activos)

**Restricciones:** La carga variable se simula con un generador de carga
que sigue un patrón sinusoidal + spikes aleatorios.
El backend simulado degrada linealmente cuando tiene > 8 requests concurrentes.

**Pista:** La estrategia óptima depende de la variabilidad de la carga y del costo
de los errores. Para carga estable, fijo es lo más simple. Para carga muy variable,
adaptive reduce el waste. Para maximizar throughput con backend variable, gradient.
En la práctica, adaptive AIMD es el mejor compromiso entre simplicidad y adaptabilidad.

**Implementar en:** Go · Java · Python

---

## Sección 10.7 — Casos Reales: Pipelines de ML y Compilación

Las secciones anteriores presentaron los patrones. Esta los conecta con
sistemas reales que los usan extensivamente.

---

### Ejercicio 10.7.1 — Pipeline de inferencia de ML

**Enunciado:** Un pipeline de inferencia de ML tiene etapas con distintos costos
y distintos requerimientos de paralelismo:

```
Etapa 1: Preprocessing (CPU-bound, paralelizable)
  - Resize, normalize, convert
  - Costo: 5ms por imagen
  - Paralelismo: ilimitado

Etapa 2: Model inference (GPU si disponible, CPU si no)
  - Costo: 50ms por imagen (CPU), 2ms (GPU con batch de 32)
  - Paralelismo: batch processing es más eficiente

Etapa 3: Postprocessing (CPU-bound, paralelizable)
  - Decode outputs, threshold, NMS
  - Costo: 3ms por imagen
  - Paralelismo: ilimitado
```

Implementa el pipeline con batching en la etapa 2:

```go
type PipelineInferencia struct {
    batchSize    int
    batchTimeout time.Duration  // cuánto esperar a completar un batch
    // ...
}
```

**Restricciones:** El batching en la etapa 2 acumula hasta `batchSize` imágenes
o espera `batchTimeout` (lo que ocurra primero). Si hay 1 imagen después de
`batchTimeout`, procesarla de todas formas (no esperar indefinidamente).

**Pista:** El batching introduce un tradeoff latencia vs throughput.
Con batchSize=32 y batchTimeout=100ms, la latencia mínima es 100ms (esperando el batch).
Con batchSize=1, la latencia es mínima pero el throughput del GPU es terrible.
El óptimo depende del SLO del sistema: si la latencia importa más que el throughput, batch pequeño.

**Implementar en:** Go · Python (`asyncio` + simulación de GPU) · Java

---

### Ejercicio 10.7.2 — Compilador incremental paralelo

**Enunciado:** Un compilador incremental solo recompila los archivos que
cambiaron (o cuyas dependencias cambiaron). Implementa el análisis de dependencias
y la compilación paralela incremental:

```go
type CompiladorIncremental struct {
    grafo    map[string][]string    // archivo → sus importaciones
    hashes   map[string]string      // archivo → hash de su contenido
    compiled map[string]time.Time   // archivo → última compilación
}

func (c *CompiladorIncremental) Analizar(archivos []string) error
func (c *CompiladorIncremental) Compilar(changed []string, jobs int) error
// Solo recompilar changed y todos sus dependientes transitivos
```

**Restricciones:** El análisis de dependencias detecta ciclos de importación
(error de compilación). La compilación paralela respeta el orden topológico.
Un cambio en un archivo de bajo nivel (util.go) puede desencadenar la recompilación
de muchos archivos dependientes.

**Pista:** El conjunto de archivos a recompilar se calcula con BFS/DFS desde
los archivos cambiados en el grafo de dependencias (hacia adelante — los que dependen de ellos).
Este es exactamente el algoritmo de `make -j` y de `go build`: solo reconstruir
lo que es alcanzable desde los cambios.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.7.3 — Workflow engine: orquestar tareas complejas

**Enunciado:** Implementa un workflow engine que ejecuta DAGs de tareas
con soporte para retry, timeout, y compensación (rollback):

```go
type WorkflowEngine struct {
    scheduler *DAGScheduler
    registro  *RegistroEjecucion
}

type WorkflowDefinicion struct {
    Nombre  string
    Pasos   []Paso
    Timeout time.Duration
}

type Paso struct {
    Nombre       string
    Ejecutar     func(ctx context.Context, input interface{}) (interface{}, error)
    Compensar    func(ctx context.Context, input interface{}) error  // rollback
    Reintentos   int
    Timeout      time.Duration
    Dependencias []string
}
```

**Restricciones:** Si un paso falla después de los reintentos, ejecutar los `Compensar`
de todos los pasos completados (en orden inverso) — similar a Saga.
El registro de ejecución permite reanudar el workflow desde donde se detuvo
después de un fallo del proceso.

**Pista:** Este es el modelo de Temporal.io, Apache Airflow, y AWS Step Functions.
La característica clave es la durabilidad: el estado del workflow se persiste
periódicamente, permitiendo recovery tras fallos. La compensación (Saga pattern)
permite "deshacer" operaciones que no son transaccionales (como enviar un email).

**Implementar en:** Go · Java · Python

---

### Ejercicio 10.7.4 — Sistema de recomendaciones con múltiples fuentes

**Enunciado:** Un sistema de recomendaciones combina resultados de múltiples
fuentes con distintas latencias y confiabilidades:

```go
type SistemaRecomendaciones struct {
    fuentes []FuenteRecomendacion
    timeout time.Duration
    mezclador Mezclador
}

type FuenteRecomendacion interface {
    Obtener(ctx context.Context, userID string) ([]Recomendacion, error)
    Latencia() time.Duration  // latencia histórica
}
```

**Estrategia:** Ejecutar todas las fuentes en paralelo. Retornar cuando:
- Todas responden, o
- El timeout expira, tomando las fuentes que respondieron hasta ese momento

**Restricciones:** Las fuentes con mayor latencia histórica se lanzan primero
(para maximizar la probabilidad de que respondan antes del timeout).
Si una fuente falla repetidamente, aplicar circuit breaker.

**Pista:** La estrategia de "lanzar las lentas primero" parece contraintuitiva
pero tiene sentido: si el timeout es 200ms y la fuente A tarda 150ms en promedio,
lanzarla al t=0 le da 200ms. Si esperas a lanzarla, le das menos tiempo.
Las fuentes rápidas (<20ms) pueden lanzarse en cualquier momento.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 10.7.5 — Sistema completo: orquestador de tareas heterogéneas

**Enunciado:** Integra los patrones de todo el capítulo en un sistema completo
de procesamiento de documentos:

**Requisitos:**
- 3 tipos de documentos: PDF (lento, 10s), Word (medio, 2s), texto (rápido, 200ms)
- 4 etapas: extracción de texto, análisis de sentimiento, indexación, notificación
- SLO: documentos de texto en < 500ms, Word en < 5s, PDF en < 30s
- Throughput objetivo: 100 documentos/segundo en mix típico (70% texto, 20% Word, 10% PDF)

**Usar:**
- Multicola por tipo (§10.1.2) para los SLOs diferenciados
- Priority queue (§10.2.1) para documentos urgentes
- DAG scheduler (§10.5.1) para las 4 etapas con dependencias
- Hedged requests (§10.4.1) para la notificación (servicio externo poco confiable)
- Adaptive concurrency (§10.6.1) para ajustar workers según la carga

**Restricciones:** El sistema debe funcionar correctamente bajo estas condiciones:
- Ráfaga de 1000 PDFs simultáneos
- Fallo del servicio de notificación durante 30 segundos
- Degradación del servicio de análisis (latencia 10x normal)

**Pista:** El sistema integrado prueba que los patrones componen bien —
o revela dónde no componen. La multicola + DAG scheduler + adaptive concurrency
es una combinación que aparece en sistemas de producción como Airflow y Temporal.
El ejercicio más largo del capítulo, y el más valioso.

**Implementar en:** Go · Java · Python

---

## Resumen del capítulo

**El toolkit del paralelismo heterogéneo:**

| Problema | Solución |
|---|---|
| Tareas de distinto costo | Clasificación + multicola + bin packing |
| Prioridades + starvation | Priority queue con aging, EDF para deadlines |
| Tail latency alta | Hedged requests, percentile tracking, speculative retry |
| Dependencias entre tareas | DAG scheduler, critical path analysis |
| Número óptimo de workers | Adaptive concurrency (AIMD, gradient) |
| Aislamiento de fallas | Bulkhead pattern, circuit breaker por pool |

**La regla de oro:**

> El paralelismo de tareas homogéneas es fácil: divide en chunks iguales.
> El paralelismo de tareas heterogéneas requiere medir, clasificar, y adaptar.
> La mayoría del trabajo real es heterogéneo.
> Diseña para la varianza, no para el promedio.

## La pregunta que guía el Cap.11

> El Cap.10 cubrió cómo paralelizar trabajo con costos variables.
> El Cap.11 entra en el hardware: cómo el CPU, la caché, y el sistema de memoria
> afectan el rendimiento del código paralelo — y cómo diseñar para el hardware,
> no solo para el algoritmo.
