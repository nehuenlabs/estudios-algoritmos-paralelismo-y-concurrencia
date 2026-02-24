# Guía de Ejercicios — Patrones Clásicos de Concurrencia

> Implementar cada ejercicio en: **Go · Java · Python · Rust · C#**
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el patrón que aparece en toda entrevista

```go
// La pregunta más frecuente de concurrencia en entrevistas:
// "Tienes N tareas lentas. ¿Cómo las ejecutas concurrentemente?"

// Sin patrón — problemas evidentes:
for _, url := range urls {
    go func(u string) {
        resultado := descargar(u)
        // ¿dónde va el resultado?
        // ¿cómo sé cuándo terminaron todas?
        // ¿cómo limito a máximo 10 descargas?
    }(url)
}

// Con Worker Pool — estructura clara:
func descargarTodo(urls []string, concurrencia int) []Resultado {
    jobs := make(chan string, len(urls))
    results := make(chan Resultado, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < concurrencia; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range jobs {
                results <- Resultado{URL: url, Datos: descargar(url)}
            }
        }()
    }

    for _, url := range urls {
        jobs <- url
    }
    close(jobs)

    go func() {
        wg.Wait()
        close(results)
    }()

    var todos []Resultado
    for r := range results {
        todos = append(todos, r)
    }
    return todos
}
```

La diferencia no es solo de corrección — es de **estructura reconocible**.
La segunda versión tiene nombre, partes predecibles y comportamiento documentado.
Eso es lo que comunican los patrones.

---

## Los patrones de este capítulo

```
Patrón               Problema que resuelve
───────────────────  ──────────────────────────────────────────
Worker Pool          N tareas, M workers — control de concurrencia
Pipeline             Transformaciones en cadena sobre un stream
Fan-out / Fan-in     Distribuir trabajo y consolidar resultados
Pub/Sub              Notificar a múltiples suscriptores de eventos
Future / Promise     Obtener un resultado que aún no está disponible
Circuit Breaker      Proteger un sistema que puede fallar
Rate Limiter         Controlar la velocidad de procesamiento
Composición          Combinar patrones para sistemas reales
```

---

## Tabla de contenidos

- [Sección 3.1 — Worker Pool](#sección-31--worker-pool)
- [Sección 3.2 — Pipeline](#sección-32--pipeline)
- [Sección 3.3 — Fan-out y Fan-in](#sección-33--fan-out-y-fan-in)
- [Sección 3.4 — Pub/Sub](#sección-34--pubsub)
- [Sección 3.5 — Future y Promise](#sección-35--future-y-promise)
- [Sección 3.6 — Circuit Breaker y Rate Limiter](#sección-36--circuit-breaker-y-rate-limiter)
- [Sección 3.7 — Componer patrones](#sección-37--componer-patrones)

---

## Sección 3.1 — Worker Pool

**El problema:** lanzar una goroutine por tarea no escala.
Con 100,000 tareas tendrías 100,000 goroutines activas — el scheduler
de Go puede manejarlo, pero el consumo de recursos crece sin control.

**La solución:** un número fijo de workers que procesan tareas de una cola.

```
                    ┌─────────────┐
                    │  Worker 1   │ ──┐
jobs ──→ [queue] ──→│  Worker 2   │   ├──→ [results]
                    │  Worker 3   │ ──┘
                    └─────────────┘
```

**La implementación canónica:**

```go
func WorkerPool(numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result, numWorkers)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- procesar(job)
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

**Las cuatro propiedades de un Worker Pool correcto:**

```
1. Control de concurrencia: exactamente numWorkers goroutines activas
2. Terminación limpia:      workers terminan cuando jobs se cierra
3. Sin goroutine leaks:     ninguna goroutine queda esperando
4. Propagación de errores:  errores de workers llegan al llamador
```

---

### Ejercicio 3.1.1 — Worker Pool básico

**Enunciado:** Implementa un Worker Pool que descarga URLs concurrentemente.
Recibe `[]string` de URLs y número de workers, retorna `[]Resultado`
con `URL`, `Contenido` y `Error`.

**Restricciones:** El número de goroutines activas no debe superar `numWorkers`.
El orden de los resultados no necesita coincidir con el input.
Verifica con `go test -race`.

**Pista:** Simula la descarga con `time.Sleep(rand.Intn(100) * time.Millisecond)`.
El ejercicio es sobre la estructura, no sobre networking.

**Implementar en:** Go · Java (`ExecutorService`) · Python (`ThreadPoolExecutor`) · C# (`Task.WhenAll`) · Rust (`rayon`)

---

### Ejercicio 3.1.2 — Worker Pool con cancelación

**Enunciado:** Extiende el Worker Pool para soportar cancelación via
`context.Context`. Si el contexto se cancela, los workers terminan
limpiamente sin empezar trabajo nuevo.

```go
func DescargarTodoConContext(
    ctx context.Context,
    urls []string,
    numWorkers int,
) ([]Resultado, error)
```

**Restricciones:** Cuando el contexto se cancela, retorna lo completado hasta
ese momento más `ctx.Err()`. Sin goroutines activas después de retornar.

**Pista:** El worker verifica cancelación en cada iteración:
```go
select {
case job, ok := <-jobs:
    if !ok { return }
    procesar(job)
case <-ctx.Done():
    return
}
```

**Implementar en:** Go · Java (`Future.cancel`) · Python (`asyncio.CancelledError`) · C# (`CancellationToken`)

---

### Ejercicio 3.1.3 — Recuperación de pánicos en workers

**Enunciado:** Un worker que entra en pánico mata su goroutine y puede
dejar el WaitGroup desbalanceado. Implementa un worker que recupera pánicos
y los convierte en errores que llegan al llamador.

**Restricciones:** El pánico de un worker no debe afectar a los demás workers
ni al llamador. El pool debe continuar procesando el trabajo restante.

**Pista:** `defer recover()` dentro del worker captura el pánico y permite
enviar un `Resultado{Error: ...}` antes de que la goroutine termine.
Sin recover, el pánico de una goroutine mata todo el proceso.

**Implementar en:** Go · Java (`try/catch` en Runnable) · Python (`try/except`) · Rust

---

### Ejercicio 3.1.4 — Worker Pool con tamaño dinámico

**Enunciado:** Implementa un pool que ajusta el número de workers según la carga.
Si la cola supera el 80% de capacidad, agrega workers hasta un máximo.
Si los workers están mayoritariamente ociosos, reduce.

**Restricciones:** El escalado no interrumpe el trabajo en curso.
Nunca se crean más de `maxWorkers` goroutines.

**Pista:** Una goroutine de monitoreo revisa el tamaño de la cola cada segundo.
Para reducir workers, cierra un canal que ellos monitorean con `select` —
cuando se cierra, un worker termina limpiamente.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.1.5 — Worker Pool con prioridades

**Enunciado:** Extiende el Worker Pool con dos colas: `urgente` y `normal`.
Los workers siempre procesan los urgentes primero.

**Restricciones:** Un trabajo urgente nunca espera si hay un worker disponible,
aunque haya normales en cola. Verifica que la latencia promedio de urgentes
es menor que la de normales.

**Pista:** `select` con múltiples canales tiene semántica aleatoria cuando ambos
tienen elementos. Para dar prioridad, usa `select` anidado: primero intenta
el canal urgente sin bloquear (`default`), solo si está vacío lee del normal.

**Implementar en:** Go · Java (`PriorityBlockingQueue`) · Python · C#

---

## Sección 3.2 — Pipeline

**El problema:** una transformación compleja tiene etapas independientes
de duración distinta. Procesarlas secuencialmente desperdicia capacidad.

**La solución:** conectar etapas con canales. Cada etapa es una goroutine.
El trabajo fluye como en una línea de ensamblaje.

```
entrada → [etapa 1] → [etapa 2] → [etapa 3] → salida
           goroutine   goroutine   goroutine
```

**La propiedad clave — throughput en estado estacionario:**

```
Etapas: 50ms + 30ms + 20ms = 100ms secuencial por item

Sin pipeline: item 2 empieza cuando item 1 termina → 200ms para 2 items
Con pipeline: las etapas se solapan → throughput ≈ 1/max(etapa) = 20 items/s

La etapa más lenta es el cuello de botella del pipeline.
```

**La implementación canónica:**

```go
func etapa(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            out <- transformar(v)
        }
    }()
    return out
}

// Composición natural:
resultado := etapa3(etapa2(etapa1(generador(datos))))
```

---

### Ejercicio 3.2.1 — Pipeline de procesamiento de logs

**Enunciado:** Implementa un pipeline de tres etapas para un stream de logs:

1. `leer(filename) <-chan string` — líneas del archivo
2. `filtrar(in) <-chan string` — elimina líneas vacías y comentarios
3. `parsear(in) <-chan Registro` — convierte a struct

**Restricciones:** Con un archivo de 1 millón de líneas, el pipeline no carga
todo en memoria — procesa línea por línea. Verifica con `-race`.

**Pista:** El generador usa `bufio.Scanner` para leer línea por línea.
Cuando el canal upstream se cierra, el `range` del siguiente termina y
ese también cierra su canal de salida — la terminación se propaga downstream.

**Implementar en:** Go · Java (`Stream` API) · Python (generadores) · Rust (iteradores)

---

### Ejercicio 3.2.2 — Etapa splitter

**Enunciado:** Implementa una etapa que divide el stream en dos canales
según una condición:

```go
func Partir(in <-chan Registro, condicion func(Registro) bool) (
    coinciden <-chan Registro,
    noCoinciden <-chan Registro,
)
```

**Restricciones:** El splitter no bloquea si un consumidor deja de leer.
Sin goroutine leaks cuando uno de los consumidores cancela.

**Pista:** Si el consumidor de `coinciden` deja de leer, el splitter se bloquea
enviando. Solución: canales con buffer, o `select` con `ctx.Done()`.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.2.3 — Back-pressure observable

**Enunciado:** Implementa un pipeline donde se puede observar y controlar
el back-pressure:

- Generador: 1000 items/segundo
- Procesador: 100 items/segundo
- Escritor: consume lo que produce el procesador

Implementa dos estrategias: **drop** (descarta si la cola está llena)
y **block** (el generador espera).

**Restricciones:** La longitud de la cola entre generador y procesador
debe ser observable en tiempo real con `len(canal)`.

**Pista:** Un canal con buffer lleno bloquea al emisor automáticamente — eso es
back-pressure por defecto. Para "drop", usa `select { case ch <- v: default: }`.

**Implementar en:** Go · Java (Flow API) · Python (`asyncio.Queue`) · Rust

---

### Ejercicio 3.2.4 — Etapa con reintentos

**Enunciado:** Implementa una etapa del pipeline con reintentos automáticos
y backoff exponencial para operaciones que pueden fallar transitoriamente:

```go
func EtapaConReintentos(
    in <-chan Job,
    procesar func(Job) (Result, error),
    maxIntentos int,
    backoffBase time.Duration,
) <-chan Result
```

**Restricciones:** Un fallo permanente envía `Result{Error: err}` sin bloquear
el pipeline. Los reintentos son por item, no por toda la etapa.

**Pista:** Este ejercicio conecta el Ejercicio 1.3.5 de este repo con los pipelines.
El patrón de backoff exponencial con jitter es el mismo — solo cambia el contexto.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.2.5 — Identificar y eliminar el cuello de botella

**Enunciado:** Pipeline con cuatro etapas:

```
etapa1: 10ms/item
etapa2: 50ms/item  ← cuello de botella
etapa3: 20ms/item
etapa4: 15ms/item
```

Verifica que el throughput real se acerca a `1/50ms = 20 items/s`.
Luego paraleliza la etapa 2 con N workers — el throughput debe acercarse
a `N/50ms`.

**Restricciones:** Mide el throughput real con un benchmark. El speedup
con N workers será sublineal — ese resultado también es correcto e interesante.

**Pista:** Paralelizar una etapa es combinar Worker Pool y Pipeline.
La etapa 2 se convierte en fan-out a N workers → fan-in de resultados.
El orden de los items puede cambiar — decide si eso es aceptable.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 3.3 — Fan-out y Fan-in

**Fan-out:** distribuir trabajo a múltiples canales o workers.
**Fan-in:** consolidar múltiples canales en uno solo.

Juntos forman el patrón scatter-gather: dispersar, ejecutar en paralelo, reunir.

```
              ┌─→ worker A ─→┐
canal ──→ fan-out ─→ worker B ─→┤──→ fan-in ──→ resultados
              └─→ worker C ─→┘
```

**Fan-in en Go — la implementación estándar:**

```go
func FanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    wg.Add(len(channels))
    for _, c := range channels {
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                out <- v
            }
        }(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

**La diferencia con Worker Pool:**

```
Worker Pool:    workers intercambiables, trabajo distribuido dinámicamente
Fan-out/Fan-in: canales con semántica propia, distribución estática
```

---

### Ejercicio 3.3.1 — Búsqueda en múltiples fuentes, primer resultado

**Enunciado:** Consulta tres fuentes simultáneamente (BD, caché, índice)
y retorna el primer resultado que llega, cancelando las demás búsquedas.

```go
func BuscarPrimero(ctx context.Context, query string) (Resultado, error)
```

**Restricciones:** Las búsquedas que no ganaron deben cancelarse y terminar.
Cero goroutine leaks al terminar.

**Pista:** Canal con buffer 1 + `context.WithCancel`. La primera goroutine que
envía gana. Las otras detectan `ctx.Done()` y terminan. El buffer 1 evita que
la goroutine ganadora se bloquee si el llamador ya no lee.

**Implementar en:** Go · Java (`CompletableFuture.anyOf`) · Python (`asyncio.wait(FIRST_COMPLETED)`) · C# (`Task.WhenAny`)

---

### Ejercicio 3.3.2 — Scatter-Gather con quórum

**Enunciado:** Implementa scatter-gather que retorna cuando K de N nodos
respondieron (quórum):

```go
func ScatterGather(
    ctx context.Context,
    nodos []Nodo,
    quorum int,
    request Request,
) ([]Respuesta, error)
```

**Restricciones:** Una vez alcanzado el quórum, cancela las peticiones
pendientes. El resultado tiene exactamente `quorum` respuestas.

**Pista:** Canal con buffer `quorum`. El llamador lee `quorum` veces y
luego cancela el contexto. Las goroutines restantes detectan la cancelación.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.3.3 — Merge ordenado de streams (Cap.05 sobre canales)

**Enunciado:** Dados N canales con elementos en orden ascendente, implementa
un fan-in que produce un canal con todos los elementos ordenados.
Este es el k-way merge del Cap.05 §2.4, ahora sobre canales.

```go
func MergeOrdenado(canales ...<-chan int) <-chan int
```

**Restricciones:** Usa un heap para el merge — `O(k log k)` por elemento.
El canal de salida se cierra cuando todos los de entrada se cierran.

**Pista:** El heap contiene `(valor, índice_canal)`. Al sacar el mínimo,
lee el siguiente del mismo canal e inserta en el heap.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.3.4 — Fan-out con balanceo de carga

**Enunciado:** Implementa un fan-out que envía al worker con menos carga
en lugar de round-robin:

```go
type WorkerBalanceado struct {
    trabajo chan Job
    carga   int64  // atómico — trabajos pendientes
}

func FanOutBalanceado(workers []*WorkerBalanceado, job Job)
```

**Restricciones:** La selección del worker menos cargado es atómica.
Verifica que la distribución es más equitativa que round-robin
con trabajos de duración variable.

**Pista:** Comparar `carga` de todos los workers requiere leer N atómicos.
Puede haber cambios entre lecturas — pero es una aproximación suficientemente
buena, no necesita ser perfecta.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.3.5 — Reducción paralela (Map-Reduce simplificado)

**Enunciado:** Implementa map-reduce para contar frecuencia de palabras
en un corpus:

```
docs → [fan-out: N chunks] → [map: frecuencia por chunk] → [fan-in] → [reduce: combinar]
```

**Restricciones:** El número de workers es configurable. El resultado es
determinista — mismo input, mismo output.

**Pista:** La fase map es paralela. La fase reduce es secuencial — un goroutine
combina los mapas parciales. El fan-in recolecta los parciales para el reduce.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 3.4 — Pub/Sub

**El problema:** un evento ocurre y múltiples partes necesitan reaccionar.
El emisor no sabe ni le importa quién escucha.

**La solución:** un broker que desacopla emisores de receptores.

```
                   ┌─→ suscriptor A
emisor ──→ broker ─┼─→ suscriptor B
                   └─→ suscriptor C
```

**Implementación básica:**

```go
type Broker[T any] struct {
    mu           sync.RWMutex
    suscriptores map[string][]chan T
}

func (b *Broker[T]) Suscribir(tema string) <-chan T {
    b.mu.Lock()
    defer b.mu.Unlock()
    ch := make(chan T, 10)
    b.suscriptores[tema] = append(b.suscriptores[tema], ch)
    return ch
}

func (b *Broker[T]) Publicar(tema string, evento T) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, ch := range b.suscriptores[tema] {
        select {
        case ch <- evento:
        default:  // suscriptor lento — drop o block según política
        }
    }
}
```

---

### Ejercicio 3.4.1 — Broker con suscripción y cancelación

**Enunciado:** Implementa el broker completo con:
`Suscribir`, `Cancelar`, `Publicar`, `CerrarTema`.

Cuando se cierra un tema, todos los suscriptores reciben la señal de cierre.

**Restricciones:** `Cancelar` es seguro desde cualquier goroutine.
Un suscriptor cancelado no recibe eventos posteriores a la cancelación.
Verifica con `-race`.

**Pista:** `Cancelar` modifica la lista de suscriptores — necesita `Lock()`.
`Publicar` itera la lista — usa `RLock()`. Cuidado con el deadlock si
`Cancelar` se llama desde dentro de un handler de evento.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.4.2 — Broker con wildcards

**Enunciado:** Soporta suscripciones con wildcards:
- `usuario.creado` — tema exacto
- `usuario.*` — todos los eventos de usuario
- `*.creado` — todos los eventos de creación

**Restricciones:** Matching `O(suscripciones)` — no hace falta trie.

**Pista:** Separa el tema por `.` y verifica segmento a segmento.
`*` coincide con cualquier segmento único.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.4.3 — Políticas para suscriptores lentos

**Enunciado:** Implementa tres políticas para slow consumers:

1. **Drop**: si el canal está lleno, descarta el evento
2. **Block**: `Publicar` espera al suscriptor lento
3. **Drop-oldest**: descarta el evento más antiguo, pone el nuevo

**Restricciones:** La política se configura por suscriptor.
Mide cuántos eventos se pierden con Drop bajo carga alta.

**Pista:** "Drop-oldest" requiere `<-ch` (descartar) seguido de `ch <- evento`.
Esta secuencia no es atómica — necesita sincronización adicional.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.4.4 — Event sourcing simplificado

**Enunciado:** Implementa un store de eventos con pub/sub:

```go
func (e *EventStore) Append(evento Evento)
func (e *EventStore) Replay(desde int) <-chan Evento  // históricos
func (e *EventStore) Suscribir(tema string) <-chan Evento  // nuevos
```

**Restricciones:** `Replay` emite eventos históricos seguidos de nuevos
en tiempo real — sin gap ni duplicados en el "join point".

**Pista:** Suscríbete al broker ANTES de iniciar el replay. Guarda los nuevos
eventos durante el replay. Emítelos al final del replay.
El orden correcto previene el gap.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.4.5 — Broker distribuido entre procesos

**Enunciado:** Extiende el broker para que funcione entre múltiples instancias
via TCP. Un evento publicado en instancia A llega a suscriptores en instancia B.

**Restricciones:** Protocolo JSON sobre TCP. Si un peer se cae, se desconecta
sin afectar al broker local. No reenviar a peers lo que vino de un peer.

**Pista:** Este es el primer ejercicio que toca el Cap.22.
El broker local ya funciona. El distribuido agrega reenvío a peers.
El flag "originado aquí" evita los loops de reenvío.

**Implementar en:** Go · Java

---

## Sección 3.5 — Future y Promise

**El problema:** una operación lenta devuelve un valor que no está disponible
aún. El llamador no quiere bloquearse mientras espera.

**La solución:** un Future es un contenedor para un valor futuro.
Se puede pasar, combinar, y encadenar sin bloquear.

```go
type Future[T any] struct {
    ch chan T
}

func Async[T any](f func() T) *Future[T] {
    ch := make(chan T, 1)
    go func() { ch <- f() }()
    return &Future[T]{ch: ch}
}

func (f *Future[T]) Get() T { return <-f.ch }

func (f *Future[T]) GetWithTimeout(d time.Duration) (T, bool) {
    select {
    case v := <-f.ch:
        return v, true
    case <-time.After(d):
        var zero T
        return zero, false
    }
}
```

---

### Ejercicio 3.5.1 — Future con error y encadenamiento

**Enunciado:** Extiende el Future para manejar errores y encadenamiento:

```go
func (f *Future[T]) Then(g func(T) T) *Future[T]
func (f *Future[T]) Catch(g func(error) T) *Future[T]
func (f *Future[T]) Get() (T, error)
```

**Restricciones:** `Then` retorna un nuevo Future inmediatamente — no bloquea.
Si el Future original tiene error, `Then` lo propaga sin ejecutar `g`.

**Pista:** `Then` lanza una goroutine que espera el original y aplica `g`
si no hay error. El nuevo Future recibe el resultado de `g`.

**Implementar en:** Go · Java (`CompletableFuture`) · Python (`asyncio.Task`) · C# (`Task`) · Rust (`tokio::spawn`)

---

### Ejercicio 3.5.2 — WhenAll: esperar N futures

**Enunciado:** Implementa `Todos` que se completa cuando todos los futures
de entrada se completan:

```go
func Todos[T any](futures ...*Future[T]) *Future[[]T]
```

Si alguno falla, el resultado falla con ese error.

**Restricciones:** Los futures se esperan concurrentemente.
El tiempo total es el del future más lento — no la suma.

**Pista:** Una goroutine por future, WaitGroup para saber cuándo
terminaron todos, atómico o canal para propagar el primer error.

**Implementar en:** Go · Java (`CompletableFuture.allOf`) · Python (`asyncio.gather`) · C# (`Task.WhenAll`)

---

### Ejercicio 3.5.3 — Singleflight: evitar trabajo duplicado

**Enunciado:** Si dos goroutines piden `calcular("x")` simultáneamente,
solo se ejecuta una vez — la segunda espera el resultado de la primera:

```go
func (c *CacheAsync[K, V]) ObtenerOCalcular(
    clave K,
    calcular func() (V, error),
) (V, error)
```

**Restricciones:** `calcular` se ejecuta exactamente una vez por clave,
aunque N goroutines lo soliciten simultáneamente.

**Pista:** Go tiene `golang.org/x/sync/singleflight`. Implementa tu propia
versión antes de mirarla — es el mejor ejercicio de este capítulo.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.5.4 — Future con timeout y retry

**Enunciado:** Combina Future, timeout y retry:

```go
func ConTimeoutYRetry[T any](
    ctx context.Context,
    op func(context.Context) (T, error),
    timeout time.Duration,
    maxReintentos int,
) (T, error)
```

**Restricciones:** El timeout es por intento, no total.
El backoff exponencial tiene límite máximo.
El contexto externo puede cancelar todo.

**Pista:** Cada intento usa `context.WithTimeout` derivado del contexto externo.
Si el externo se cancela, el derivado también se cancela.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.5.5 — Lazy Future

**Enunciado:** Un Lazy Future no lanza la goroutine hasta que alguien llama
`Get()`. Si nadie lo llama, no se gasta ningún recurso.

Compara el consumo entre 1000 Futures inmediatos vs 1000 Lazy de los cuales
solo se consultan 10.

**Restricciones:** El cómputo inicia exactamente una vez aunque múltiples
goroutines llamen `Get()` simultáneamente.

**Pista:** `sync.Once` garantiza que la función se ejecuta exactamente una vez.
`Get()` llama `once.Do(func() { go func() { ch <- f() }() })`.

**Implementar en:** Go · Java (`Supplier` lazy) · Python · Rust · C#

---

## Sección 3.6 — Circuit Breaker y Rate Limiter

**Circuit Breaker — el problema:** un servicio externo lento puede hacer que
todas las goroutines del sistema se bloqueen esperando — cascading failure.

**Los tres estados:**

```
CERRADO  → llamadas pasan, se cuentan los fallos
ABIERTO  → llamadas fallan inmediatamente (fail fast)
SEMI     → deja pasar una llamada de prueba

Transiciones:
  CERRADO → ABIERTO:  fallos >= umbral en ventana de tiempo
  ABIERTO → SEMI:     después del timeout de espera
  SEMI → CERRADO:     si la llamada de prueba tiene éxito
  SEMI → ABIERTO:     si la llamada de prueba falla
```

**Rate Limiter — token bucket:**

```
- El bucket tiene N tokens
- Se añade 1 token cada 1/rate segundos (hasta el máximo)
- Cada llamada consume 1 token
- Sin tokens: esperar o fallar
```

---

### Ejercicio 3.6.1 — Circuit Breaker básico

**Enunciado:** Implementa el Circuit Breaker con los tres estados:

```go
type CircuitBreaker struct {
    estado        Estado
    fallos         int
    umbralFallos   int
    tiempoEspera   time.Duration
    ultimoFallo    time.Time
    mu             sync.Mutex
}

func (cb *CircuitBreaker) Ejecutar(f func() error) error
```

**Restricciones:** Thread-safe. Las transiciones de estado son atómicas.
En estado Abierto, `Ejecutar` retorna `ErrCircuitoAbierto` inmediatamente.

**Pista:** El check "¿pasar a SemiAbierto?" ocurre bajo el mismo lock que
la lectura del estado — evita que múltiples goroutines entren en SemiAbierto.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.6.2 — Circuit Breaker con ventana deslizante

**Enunciado:** En lugar de contar fallos totales, cuenta fallos en los
últimos N segundos usando la ventana deslizante del Cap.03 §3 del repo de algoritmos.

**Restricciones:** La ventana es `O(1)` para actualizar y consultar.
Usa un buffer circular de timestamps de fallos.

**Pista:** Al consultar el conteo, descarta timestamps más viejos que
`ahora - ventana` y cuenta los restantes.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.6.3 — Rate Limiter con token bucket

**Enunciado:** Implementa el algoritmo token bucket:

```go
func (r *RateLimiter) Allow() bool
func (r *RateLimiter) Wait(ctx context.Context) error
```

**Restricciones:** `Allow` es O(1). `Wait` respeta cancelación del contexto.
Con `rate=10` y 100 goroutines, el throughput real se acerca a 10 llamadas/segundo.

**Pista:** Calcula tokens acumulados al momento de la llamada:
`tokens += rate * tiempo_transcurrido`. Más preciso que un ticker y sin
goroutine de fondo.

**Implementar en:** Go · Java (Guava `RateLimiter`) · Python · Rust · C#

---

### Ejercicio 3.6.4 — Circuit Breaker en el Worker Pool

**Enunciado:** Integra el Circuit Breaker en el Worker Pool.
Si el CB está abierto, el worker usa un endpoint de backup.

**Restricciones:** El CB es compartido entre todos los workers.
Si el backup también falla, retorna error.

**Pista:** El CB protege el endpoint, no el worker.
Múltiples workers usando el mismo CB es el caso de uso exacto —
el CB debe ser thread-safe (ya lo es del Ejercicio 3.6.1).

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.6.5 — ClienteRobusto: combinando todo

**Enunciado:** Combina Rate Limiter + Circuit Breaker + Timeout + Retry:

```go
type ClienteRobusto struct {
    cb      *CircuitBreaker
    rl      *RateLimiter
    timeout time.Duration
    retry   int
}

func (c *ClienteRobusto) Llamar(ctx context.Context, req Request) (Response, error)
```

Orden de evaluación: Rate Limiter → Circuit Breaker → Timeout → Llamada → Retry.

**Restricciones:** Errores identifican el componente que los generó.
Cada componente es configurable independientemente.

**Pista:** Este es la versión simplificada de Hystrix (Java), go-resilience,
o Polly (C#). La combinación de los cuatro protege de todos los modos
de fallo del servicio externo.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 3.7 — Componer Patrones

Los patrones no son exclusivos. En sistemas reales se combinan.

```
Ejemplo real — ingesta de datos:
  Fuente → [Rate Limiter] → [Worker Pool] → [Pipeline] → [Pub/Sub] → suscriptores
```

---

### Ejercicio 3.7.1 — Worker Pool + Pipeline

**Enunciado:** Sistema que procesa imágenes:
1. Worker Pool descarga imágenes (I/O bound, 10 workers)
2. Pipeline procesa: redimensionar → comprimir → guardar (CPU bound)

**Restricciones:** Descarga y procesamiento ocurren en paralelo —
mientras se procesa la imagen N, se descarga la N+1.
Verifica que el throughput supera el secuencial.

**Pista:** El canal de salida del Worker Pool es el canal de entrada del Pipeline.
La composición es natural con la interfaz `<-chan T`.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 3.7.2 — Fan-out + Circuit Breaker

**Enunciado:** Búsqueda en tres backends, cada uno con su Circuit Breaker.
Si un backend está caído (CB abierto), usa solo los disponibles.

**Restricciones:** Si todos están caídos, retorna error inmediatamente.
La degradación es elegante — funciona con 3, 2, o 1 backend.

**Pista:** Antes del fan-out, verifica cuáles backends tienen CB cerrado.
Si ninguno está disponible, retorna `ErrTodosLosCBsAbiertos` sin esperar timeouts.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.7.3 — Pub/Sub + Worker Pool

**Enunciado:** Recibe eventos vía Pub/Sub y los procesa con un Worker Pool.

```go
type ProcesadorEventos struct {
    broker *Broker
    pool   *WorkerPool
}

func (p *ProcesadorEventos) Iniciar(temas []string)
func (p *ProcesadorEventos) Detener()
```

**Restricciones:** La parada es limpia: cancelar suscripciones, esperar
workers en curso, cerrar el pool. El orden importa — documentarlo.

**Pista:** Orden correcto: (1) cancelar suscripciones, (2) cerrar canal de jobs,
(3) WaitGroup.Wait(). Si se cierra el pool antes de cancelar suscripciones,
el broker envía a un canal cerrado → panic.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 3.7.4 — Sistema completo de ingesta

**Enunciado:** Sistema que combina todos los patrones:

1. Lee de API externa a 100 eventos/segundo (Rate Limiter)
2. Filtra y enriquece cada evento (Pipeline, 3 etapas)
3. Procesa en paralelo con 5 workers (Worker Pool)
4. Publica a 3 sistemas downstream (Pub/Sub, Fan-out)
5. Tolera downstream caído (Circuit Breaker por downstream)

**Restricciones:** Arranque y parada limpios. Cero goroutine leaks
verificados con `goleak`. Latencia end-to-end < 500ms bajo carga.

**Pista:** Construye el sistema conectando componentes de las secciones
anteriores. Dibuja el diagrama de flujo antes de escribir código.
La parada limpia es la parte más difícil — el orden importa.

**Implementar en:** Go · Java · Python

---

### Ejercicio 3.7.5 — Code review de composición

**Enunciado:** Este sistema tiene bugs en la composición. Identifica todos:

```go
func (s *Sistema) Iniciar() {
    eventos := s.broker.Suscribir("eventos")
    go func() {
        for evento := range eventos {
            s.pool.Submit(Job{
                Procesar: func() error {
                    return s.cb.Ejecutar(func() error {
                        return enviar(evento)  // ← captura variable del loop
                    })
                },
            })
        }
    }()
}

func (s *Sistema) Detener() {
    s.pool.Cerrar()    // ← cierra el pool
    s.broker.Cerrar()  // ← cierra el broker después
    // ¿falta algo?
}
```

**Restricciones:** Identifica al menos 3 problemas por tipo
(race condition, goroutine leak, orden de parada, etc.).

**Pista:** La captura de variable de loop en Go pre-1.22 es un bug clásico.
Los otros bugs son de diseño de concurrencia y orden de parada.

**Implementar en:** Go · Java · Python

---

## Resumen del capítulo

| Patrón | Problema | Estructura clave | Cuándo usarlo |
|---|---|---|---|
| **Worker Pool** | N tareas, control de concurrencia | `jobs chan`, N goroutines, `results chan` | I/O bound con muchas tareas |
| **Pipeline** | Transformaciones en cadena | Canales conectando etapas | Procesamiento stream con etapas independientes |
| **Fan-out/Fan-in** | Paralelismo y consolidación | Múltiples canales, merge | Búsqueda paralela, scatter-gather |
| **Pub/Sub** | Desacoplar emisores y receptores | Broker con mapa de suscriptores | Eventos con múltiples consumidores |
| **Future** | Resultado asíncrono | Canal buffer 1 | Paralelismo de latencia |
| **Circuit Breaker** | Fallos en cascada | Máquina de estados + contador | Llamadas a servicios externos |
| **Rate Limiter** | Control de velocidad | Token bucket + mutex | APIs externas con límite |

## La pregunta que guía el Cap.04

> Los patrones de este capítulo son estáticos — N workers fijos, K etapas fijas.
> ¿Qué pasa cuando el número de goroutines necesita ajustarse dinámicamente
> según la carga, en tiempo de ejecución?

El Cap.04 cubre las primitivas nativas de Go: goroutines, canales y select
desde los fundamentos del scheduler — y cómo el runtime de Go gestiona el
paralelismo real sobre múltiples núcleos físicos.
