# Guía de Ejercicios — Goroutines, Scheduler y el Runtime de Go

> Implementar cada ejercicio en: **Go** (capítulo centrado en Go)
> Equivalencias en Java · Python · Rust · C# donde aplique.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: ¿cuántas goroutines puedes crear?

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    n := 1_000_000  // un millón de goroutines

    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            // no hace nada — solo existe
        }()
    }

    wg.Wait()
    fmt.Printf("Goroutines activas al pico: ~%d\n", runtime.NumGoroutine())
    fmt.Println("Terminado sin errores")
}
```

```
$ go run main.go
Goroutines activas al pico: ~1000000
Terminado sin errores

Tiempo: ~1.2 segundos
Memoria usada: ~4 GB (cada goroutine empieza con 2-8 KB de stack)
```

Con un millón de threads del sistema operativo esto no funcionaría —
Linux permite típicamente 10,000–50,000 threads antes de quedarse sin memoria.
Go puede manejar millones de goroutines porque no son threads del SO.

**¿Qué son entonces?**

---

## El modelo M:N — la base de todo

Go implementa un scheduler M:N: **M goroutines** se multiplexan sobre **N threads** del SO,
que a su vez se ejecutan sobre **P procesadores lógicos** (núcleos).

```
Goroutines (millones)       G G G G G G G G G G G G
                                     ↕ scheduler
Threads del SO (decenas)    T   T   T   T
                                     ↕ OS
Núcleos físicos             CPU0  CPU1  CPU2  CPU3
```

**Los tres actores del scheduler de Go:**

```
G — Goroutine: la unidad de concurrencia de Go
    Stack inicial: 2-8 KB (crece y se encoge dinámicamente hasta 1 GB)
    Estado: Running, Runnable, Waiting, Dead

M — Thread del SO (Machine): ejecuta goroutines
    Hay tantos M como goroutines en syscalls + GOMAXPROCS
    Es caro crear — el scheduler evita hacerlo

P — Procesador lógico: contexto de ejecución
    Cantidad: GOMAXPROCS (por defecto = número de CPUs)
    Cada P tiene su propia run queue de goroutines listas
    Solo un G puede ejecutar en un P a la vez
```

**Por qué las goroutines son baratas:**

```
Thread del SO:
  Stack inicial:    8 MB (fijo)
  Creación:         ~10,000 ns (syscall)
  Context switch:   ~1,000 ns (syscall al kernel)

Goroutine:
  Stack inicial:    2-8 KB (dinámico, crece si necesita)
  Creación:         ~400 ns (solo en user space)
  Context switch:   ~200 ns (el scheduler de Go, no el kernel)
```

---

## Tabla de contenidos

- [Sección 4.1 — El scheduler GMP](#sección-41--el-scheduler-gmp)
- [Sección 4.2 — Goroutine lifecycle](#sección-42--goroutine-lifecycle)
- [Sección 4.3 — select: multiplexar canales](#sección-43--select-multiplexar-canales)
- [Sección 4.4 — Goroutine leaks: causas y detección](#sección-44--goroutine-leaks-causas-y-detección)
- [Sección 4.5 — GOMAXPROCS y paralelismo real](#sección-45--gomaxprocs-y-paralelismo-real)
- [Sección 4.6 — Goroutines vs threads: cuándo importa la diferencia](#sección-46--goroutines-vs-threads-cuándo-importa-la-diferencia)
- [Sección 4.7 — Perfilar y optimizar goroutines](#sección-47--perfilar-y-optimizar-goroutines)

---

## Sección 4.1 — El Scheduler GMP

El scheduler de Go es cooperativo con preempción. Hasta Go 1.13 era puramente
cooperativo (las goroutines cedían el control en puntos de yield). Desde Go 1.14,
el scheduler puede preemptar goroutines que llevan más de 10ms ejecutando
sin un punto de yield — esto evita que una goroutine con un loop infinito
bloquee a todas las demás.

**Los puntos de yield — cuándo cede una goroutine el procesador:**

```go
// 1. Operaciones de canal (blocking)
ch <- valor    // si el canal está lleno o nadie lee
<-ch           // si el canal está vacío

// 2. Syscalls (I/O, sleep, etc.)
time.Sleep(d)
os.File.Read(buf)
net.Conn.Read(buf)

// 3. Llamadas a runtime
runtime.Gosched()   // yield explícito — cede voluntariamente

// 4. Creación de goroutines
go func() { ... }() // puede causar reschedule

// 5. Preempción forzada (Go 1.14+)
// loop largo sin yield → scheduler interrumpe en ~10ms
```

**Work stealing — cómo se balancea la carga:**

```
P0 tiene 5 goroutines en su queue
P1 no tiene ninguna → roba la mitad de la queue de P0

Esto ocurre automáticamente — sin intervención del programador.
La implicación: el orden de ejecución de goroutines es no determinista
incluso si GOMAXPROCS=1.
```

**El global run queue:**

```
Cada P tiene su local run queue (LRQ) de hasta 256 goroutines.
Cuando LRQ se llena, las nuevas van al global run queue (GRQ).
Los P revisan el GRQ periódicamente para evitar starvation.
```

---

### Ejercicio 4.1.1 — Observar el scheduler en acción

**Enunciado:** Implementa un experimento que muestra que el orden de ejecución
de goroutines es no determinista. Lanza 5 goroutines que imprimen su ID,
ejecuta 1000 veces y registra todos los órdenes distintos observados.

```go
func main() {
    ordenes := make(map[string]int)
    for iter := 0; iter < 1000; iter++ {
        var mu sync.Mutex
        var orden []int
        var wg sync.WaitGroup

        for id := 0; id < 5; id++ {
            wg.Add(1)
            go func(n int) {
                defer wg.Done()
                mu.Lock()
                orden = append(orden, n)
                mu.Unlock()
            }(id)
        }
        wg.Wait()
        clave := fmt.Sprint(orden)
        ordenes[clave]++
    }
    fmt.Printf("Órdenes distintos observados: %d\n", len(ordenes))
}
```

**Restricciones:** Reporta cuántos órdenes distintos de los 120 posibles (5!)
se observaron en 1000 ejecuciones. ¿Hay órdenes que nunca aparecen?

**Pista:** El scheduler tiende a ejecutar las goroutines en el orden en que
fueron creadas cuando no hay contención — pero esto no está garantizado.
Con `GOMAXPROCS=1` el no-determinismo es menor que con `GOMAXPROCS=4`.
Prueba ambos y compara.

**Implementar en:** Go

---

### Ejercicio 4.1.2 — runtime.Gosched() y el yield explícito

**Enunciado:** Demuestra la diferencia entre una goroutine que nunca cede
el procesador y una que usa `runtime.Gosched()`:

```go
// Sin yield — monopoliza P hasta que termina (o se preempta en 10ms)
func trabajoSinYield(n int) {
    for i := 0; i < n; i++ {
        _ = i * i  // cómputo puro, sin puntos de yield naturales
    }
}

// Con yield — cede cooperativamente
func trabajoConYield(n int) {
    for i := 0; i < n; i++ {
        _ = i * i
        if i%1000 == 0 {
            runtime.Gosched()
        }
    }
}
```

Mide el tiempo de respuesta de una goroutine de "alta prioridad" (que
imprime cada 10ms) cuando compite con 4 goroutines de cómputo intensivo,
con y sin `Gosched()`.

**Restricciones:** Usa `GOMAXPROCS=1` para que el efecto sea más pronunciado.
Mide el jitter (variación en el tiempo de respuesta) en los dos casos.

**Pista:** Con `GOMAXPROCS=1` y sin Gosched, las goroutines de cómputo
pueden monopolizar el P hasta ser preemptadas (~10ms). La goroutine de
respuesta puede sufrir latencias de hasta 10ms. Con Gosched, cede
cada 1000 iteraciones — la latencia es mucho más predecible.

**Implementar en:** Go

---

### Ejercicio 4.1.3 — Work stealing observable

**Enunciado:** Diseña un experimento que muestra el work stealing del scheduler.
Crea una situación donde P0 tiene mucho trabajo y P1 no tiene nada,
y verifica que P1 "roba" trabajo de P0.

```go
// Instrumentar goroutines para saber en qué thread se ejecutan
// usando runtime.LockOSThread() como proxy

func goroutineInstrumentada(id int, ch chan string) {
    // ¿en qué P estoy ejecutando?
    // runtime no expone directamente el P, pero podemos inferirlo
}
```

**Restricciones:** No hay API pública para saber en qué P ejecuta una goroutine.
Usa la afinidad de thread (`LockOSThread`) y GOMAXPROCS para inferir
si el work stealing ocurrió.

**Pista:** `runtime.LockOSThread()` fija la goroutine actual a su thread del SO.
Si creas una goroutine con LockOSThread en el thread 1, y luego creas trabajo
para el thread 2, el scheduler puede redistribuirlo. Instrumenta con contadores
por thread para ver el balance.

**Implementar en:** Go

---

### Ejercicio 4.1.4 — El global run queue y la starvation

**Enunciado:** Crea una situación donde goroutines se acumulan en el global
run queue porque las local run queues están llenas. Mide la latencia adicional
que introduce el GRQ vs el LRQ.

**Restricciones:** Las local run queues tienen capacidad 256 por P.
Para saturarlas con GOMAXPROCS=4 necesitas lanzar más de 1024 goroutines
que no ceden el procesador (o que se bloquean en algo).

**Pista:** Lanza goroutines que hagan `time.Sleep(1 * time.Second)`.
Las goroutines dormidas no están en el LRQ (están en una wait list interna).
Las goroutines listas para ejecutar pero sin P disponible están en LRQ o GRQ.
Usa `runtime.NumGoroutine()` y `runtime/metrics` para observar las colas.

**Implementar en:** Go

---

### Ejercicio 4.1.5 — Preempción en Go 1.14+

**Enunciado:** Demuestra que Go 1.14+ puede preemptar un loop infinito.
Antes de Go 1.14, este código bloqueaba el programa para siempre
con `GOMAXPROCS=1`. Ahora funciona correctamente.

```go
func loopInfinito() {
    for {
        // loop sin puntos de yield — antes bloqueaba todo
    }
}

func main() {
    runtime.GOMAXPROCS(1)
    go loopInfinito()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("El scheduler preemptó el loop infinito")
}
```

Verifica esto y luego modifica el experimento para medir el jitter
de preempción — ¿cuánto tiempo real pasa antes de que el loop sea preemptado?

**Restricciones:** El jitter de preempción debería ser ≤ 10ms en condiciones
normales. Mide el percentil p99 sobre 1000 observaciones.

**Pista:** La preempción de Go 1.14 usa señales del SO (SIGURG) para interrumpir
goroutines en puntos seguros. El overhead es ~10µs por preempción.
Para código que genuinamente necesita latencia garantizada < 10ms,
es necesario usar `runtime.Gosched()` explícito en los loops.

**Implementar en:** Go

---

## Sección 4.2 — Goroutine Lifecycle

Una goroutine tiene un ciclo de vida bien definido. Entender cada estado
ayuda a diagnosticar problemas de rendimiento y bugs de concurrencia.

```
go func() { ... }()
        ↓
    [Runnable]  ← está lista para ejecutar, espera un P disponible
        ↓  (P disponible)
    [Running]   ← ejecutando en un P
        ↓  (se bloquea en canal, syscall, mutex, etc.)
    [Waiting]   ← esperando que algo ocurra
        ↓  (la condición se cumple)
    [Runnable]  ← de vuelta a la cola
        ↓  (la función retorna)
    [Dead]      ← terminada, su stack se libera
```

**Cómo ver el estado de todas las goroutines:**

```go
// En cualquier momento del programa:
buf := make([]byte, 1<<20)
n := runtime.Stack(buf, true)  // true = todas las goroutines
fmt.Printf("%s", buf[:n])

// Output (ejemplo):
// goroutine 1 [running]:
// main.main()
//     /home/user/main.go:10
//
// goroutine 6 [chan receive]:
// main.worker(0xc000018060)
//     /home/user/worker.go:15
//
// goroutine 7 [sleep]:
// time.Sleep(0x3b9aca00)
//     /usr/local/go/src/runtime/time.go:193
```

**Los estados de Waiting más comunes en un stack trace:**

```
[chan receive]      → goroutine bloqueada esperando recibir de un canal
[chan send]        → goroutine bloqueada intentando enviar a un canal lleno
[select]           → goroutine en un select sin ningún case listo
[sleep]            → goroutine en time.Sleep
[IO wait]          → goroutine esperando I/O de red o disco
[semacquire]       → goroutine esperando un mutex o semáforo
[GC sweep wait]    → goroutine esperando al garbage collector
```

---

### Ejercicio 4.2.1 — Leer e interpretar stack traces

**Enunciado:** Escribe un programa que intencionalmente crea goroutines
en distintos estados (running, chan receive, sleep, mutex wait).
Usa `runtime.Stack` para capturar el stack trace completo y explica
el estado de cada goroutine.

**Restricciones:** El programa debe tener al menos 6 goroutines en estados distintos.
El stack trace debe capturarse en un momento donde todas están en los estados esperados.

**Pista:** Para forzar estados específicos:
- `[chan receive]`: goroutine bloqueada en `<-make(chan int)`
- `[sleep]`: goroutine en `time.Sleep(1 * time.Minute)`
- `[semacquire]`: goroutine esperando un mutex que otra tiene
- `[running]`: la goroutine que llama a `runtime.Stack`

**Implementar en:** Go

---

### Ejercicio 4.2.2 — Ciclo de vida completo con métricas

**Enunciado:** Implementa un wrapper de goroutine que registra métricas
del ciclo de vida: tiempo en estado Runnable (tiempo de espera en queue),
tiempo en estado Running (tiempo real de ejecución), y tiempo total de vida.

```go
func GoConMetricas(f func(), metricas *MetricasGoroutine) {
    inicio := time.Now()
    go func() {
        tiempoEnQueue := time.Since(inicio)  // approximación del tiempo Runnable
        inicioEjecucion := time.Now()
        f()
        duracion := time.Since(inicioEjecucion)
        metricas.Registrar(tiempoEnQueue, duracion)
    }()
}
```

**Restricciones:** Las métricas deben ser thread-safe. Reporta percentiles
p50, p95, p99 del tiempo en queue y tiempo de ejecución para 10,000 goroutines.

**Pista:** `tiempoEnQueue` es una aproximación — el tiempo entre `go func()`
y el inicio real de la goroutine incluye el scheduling overhead.
Para casos de alta contención (muchas goroutines, pocos P), el tiempo en
queue puede ser significativamente mayor que el tiempo de ejecución real.

**Implementar en:** Go · Java (ThreadPoolExecutor metrics) · Python

---

### Ejercicio 4.2.3 — Stack dinámico: pequeño empieza, crece si necesita

**Enunciado:** Las goroutines empiezan con un stack de 2KB y crecen dinámicamente.
Implementa una función recursiva que profundiza hasta un nivel configurable
y mide el uso de memoria de las goroutines según la profundidad del stack.

```go
func recursiva(profundidad int, resultado chan<- uintptr) {
    if profundidad == 0 {
        // capturar tamaño actual del stack
        var g runtime.MemStats
        runtime.ReadMemStats(&g)
        resultado <- g.StackInuse
        return
    }
    // crear variable local para forzar crecimiento del stack
    var buffer [1024]byte  // 1KB por nivel
    _ = buffer
    recursiva(profundidad-1, resultado)
}
```

**Restricciones:** Mide el uso de stack para profundidades de
{10, 100, 1000, 5000} niveles. Verifica que el stack crece y se encoge
después de que la goroutine retorna.

**Pista:** Go implementa "stack splitting" — cuando el stack se queda sin
espacio, se crea uno nuevo dos veces más grande y se copian los frames.
Esto hace que los punteros a variables locales sean relocatables — por eso
Go no permite aritmética de punteros sobre variables en el stack.

**Implementar en:** Go

---

### Ejercicio 4.2.4 — Goroutines y el garbage collector

**Enunciado:** Las goroutines pueden causar retención de memoria si tienen
referencias a objetos grandes en su stack o closure. Demuestra este efecto
y su solución:

```go
// Problema: la goroutine retiene datosGrandes hasta que termina
datosGrandes := make([]byte, 100*1024*1024)  // 100 MB
go func() {
    // usa datosGrandes al inicio
    _ = datosGrandes[0]

    // luego hace trabajo largo que no necesita datosGrandes
    time.Sleep(1 * time.Minute)  // datosGrandes sigue en memoria!
}()
```

**Restricciones:** Demuestra la retención con `runtime.ReadMemStats`.
Implementa la solución: nil-ificar la referencia cuando ya no se necesita.

**Pista:** El GC de Go no puede liberar un objeto mientras exista alguna
referencia a él. Una goroutine larga que tiene una referencia en su closure
mantiene vivo el objeto durante toda su vida. La solución es hacer la variable
local nula (`datosGrandes = nil`) justo después de usarla.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 4.2.5 — Detectar goroutines zombies

**Enunciado:** Una goroutine zombie es una que terminó su trabajo útil pero
sigue viva esperando algo que nunca llegará. Implementa un sistema de
detección que identifica goroutines que llevan más de N segundos en estado
[chan receive] o [select] sin hacer progreso.

**Restricciones:** El detector usa `runtime.Stack` periódicamente y
compara los stack traces entre iteraciones. Si el stack trace de una
goroutine es idéntico durante K iteraciones consecutivas, es sospechosa.

**Pista:** La comparación de stack traces es una heurística — una goroutine
legítimamente bloqueada en un select también tendría el mismo trace.
El detector es útil en tests y debugging, no en producción.
Herramientas como `goleak` hacen algo similar pero más sofisticado.

**Implementar en:** Go

---

## Sección 4.3 — select: Multiplexar Canales

`select` es la instrucción que hace a Go especialmente expresivo para
concurrencia. Permite esperar en múltiples operaciones de canal
simultáneamente, ejecutando el case que esté listo primero.

**La semántica completa de select:**

```go
select {
case v := <-ch1:
    // ch1 tenía un valor
case ch2 <- valor:
    // ch2 aceptó el envío
case v, ok := <-ch3:
    if !ok { /* ch3 cerrado */ }
    // ch3 tenía un valor
default:
    // ningún case estaba listo — no bloquea
}
```

**Reglas clave:**

```
1. Si múltiples cases están listos, Go elige uno al azar (uniforme)
2. Si ninguno está listo y no hay default, bloquea hasta que uno esté listo
3. Si hay default y ningún case está listo, ejecuta default inmediatamente
4. Un canal nil nunca está listo — se puede usar para deshabilitar un case
5. close(ch) hace que <-ch esté siempre listo (retorna zero value, ok=false)
```

**El canal nil como interruptor:**

```go
// Patrón: deshabilitar un case dinámicamente
var chActivo <-chan int = ch  // habilitado

// Para deshabilitar:
chActivo = nil  // este case nunca se selecciona

select {
case v := <-chActivo:   // nunca si chActivo == nil
    procesar(v)
case <-done:
    return
}
```

---

### Ejercicio 4.3.1 — select con timeout

**Enunciado:** Implementa tres variantes de operación con timeout:

```go
// 1. Timeout fijo — time.After
func LeerConTimeout(ch <-chan int, timeout time.Duration) (int, bool)

// 2. Timeout con contexto
func LeerConContext(ctx context.Context, ch <-chan int) (int, error)

// 3. Deadline absoluto
func LeerAntesDe(ch <-chan int, deadline time.Time) (int, bool)
```

**Restricciones:** La versión 1 tiene un memory leak sutil — identifícalo
y explica por qué la versión 2 es preferida en código de producción.

**Pista:** `time.After(d)` crea un timer que no puede cancelarse antes de
que expire. Si se usa en un loop, cada iteración crea un timer que vive
hasta que expira aunque el valor ya se leyó. Con muchos timeouts cortos,
acumula miles de timers en memoria. `time.NewTimer` + `defer t.Stop()`
resuelve el leak.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 4.3.2 — select para combinar streams

**Enunciado:** Implementa `Merge` que combina N canales de entrada en uno
de salida usando select, sin goroutines adicionales por canal:

```go
// Versión ingenua: una goroutine por canal (Ejercicio 3.3.x)
// Versión con select: sin goroutines adicionales

func MergeConSelect(done <-chan struct{}, channels ...<-chan int) <-chan int
```

**Restricciones:** Funciona con hasta N=10 canales. Sin goroutines adicionales
más allá de la goroutine del merger principal.

**Pista:** `select` no acepta un slice de canales dinámico — necesitas listarlos
estáticamente. Para N canales variable, usa `reflect.Select` (costoso) o
genera código. Compara el rendimiento de `reflect.Select` vs la versión
con una goroutine por canal del Ejercicio 3.3.

**Implementar en:** Go

---

### Ejercicio 4.3.3 — Canal nil para deshabilitar cases

**Enunciado:** Implementa un multiplexor que alterna entre dos fuentes de datos.
Cuando la fuente A tiene más de N elementos pendientes, deja de leerla
(deshabilita el case) hasta que se procesen.

```go
func Multiplexor(
    fuenteA, fuenteB <-chan int,
    maxPendientesA int,
) <-chan int
```

**Restricciones:** La inhabilitación usa el patrón de canal nil — no canales
adicionales ni flags booleanos. Los elementos de A y B se procesan en orden
de llegada cuando A no está throttleada.

**Pista:**
```go
var activaA <-chan int = fuenteA  // habilitada
pendientesA := 0

select {
case v := <-activaA:   // nil si pendientesA >= maxPendientesA
    pendientesA++
    // ...
}

if pendientesA >= maxPendientesA {
    activaA = nil  // deshabilitar
} else {
    activaA = fuenteA  // habilitar
}
```

**Implementar en:** Go

---

### Ejercicio 4.3.4 — select no bloqueante (try-receive, try-send)

**Enunciado:** Implementa operaciones no bloqueantes sobre canales usando
`select` con `default`:

```go
func TryReceive[T any](ch <-chan T) (T, bool)
func TrySend[T any](ch chan<- T, v T) bool
func TryReceiveWithFallback[T any](ch <-chan T, fallback T) T
```

Implementa además una cola de prioridad no bloqueante: intenta leer
del canal urgente; si está vacío, intenta el normal; si ambos están vacíos,
retorna inmediatamente en lugar de bloquear.

**Restricciones:** Ninguna operación bloquea — siempre retorna inmediatamente.
Verifica que no hay race condition entre el check y la operación.

**Pista:** `select { case v := <-ch: return v, true; default: var zero T; return zero, false }`.
Este patrón es atómico — Go garantiza que el check del canal y la recepción
ocurren juntos bajo el scheduler.

**Implementar en:** Go · Java (`poll()` en BlockingQueue) · Python (`get_nowait()`) · C#

---

### Ejercicio 4.3.5 — Heartbeat y detección de goroutines atascadas

**Enunciado:** Implementa un sistema de heartbeat donde una goroutine larga
envía señales periódicas. Si no envía un heartbeat en `timeout`, el monitor
la considera atascada y la cancela.

```go
func TareaConHeartbeat(
    ctx context.Context,
    heartbeat chan<- struct{},
    trabajo func(ctx context.Context) error,
) error

func Monitor(
    heartbeat <-chan struct{},
    timeout time.Duration,
    cancelar context.CancelFunc,
)
```

**Restricciones:** La tarea envía heartbeats en cada iteración de su loop
principal. El monitor cancela el contexto si pasan más de `timeout` entre heartbeats.
Verifica que una tarea realmente atascada es detectada y cancelada.

**Pista:** El heartbeat es un canal con buffer 1 — si el monitor está ocupado,
el heartbeat no bloquea a la tarea. La tarea usa:
```go
select {
case heartbeat <- struct{}{}:
default:  // monitor ocupado — no bloquear
}
```

**Implementar en:** Go · Java · Python · C#

---

## Sección 4.4 — Goroutine Leaks: Causas y Detección

Un goroutine leak es una goroutine que nunca termina. Es el equivalente
concurrente de un memory leak — acumula recursos silenciosamente hasta
que el sistema se degrada.

**Las cinco causas más comunes:**

```go
// Causa 1: canal sin buffer, nadie lee
func leak1() {
    ch := make(chan int)
    go func() {
        ch <- 42  // bloqueado para siempre si nadie lee
    }()
    // función retorna sin leer ch
}

// Causa 2: canal nunca se cierra
func leak2(jobs chan Job) {
    go func() {
        for job := range jobs {  // espera para siempre si jobs nunca se cierra
            procesar(job)
        }
    }()
}

// Causa 3: goroutine esperando WaitGroup que nunca llega a cero
func leak3() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        wg.Wait()  // espera para siempre si Done nunca se llama
    }()
    // se olvidó llamar wg.Done()
}

// Causa 4: select sin done channel
func leak4(ch <-chan int) {
    go func() {
        for {
            select {
            case v := <-ch:
                procesar(v)
            // sin case <-done: → goroutine no puede terminar
            }
        }
    }()
}

// Causa 5: goroutine en select esperando algo que nunca llega
func leak5() {
    ch := make(chan int)
    go func() {
        select {
        case v := <-ch:  // nadie enviará nunca
            _ = v
        }
    }()
}
```

**Detección con goleak:**

```bash
go get go.uber.org/goleak

# En tests:
func TestMiFuncion(t *testing.T) {
    defer goleak.VerifyNone(t)  // falla el test si hay goroutine leaks al final
    miFuncion()
}
```

---

### Ejercicio 4.4.1 — Identificar y corregir los 5 tipos de leak

**Enunciado:** Para cada una de las cinco causas del inicio de la sección,
escribe un test que:
1. Demuestra el leak con `goleak.VerifyNone`
2. Propone la corrección
3. Verifica que la corrección elimina el leak

**Restricciones:** El test debe fallar **antes** de la corrección y pasar
**después**. Usa `goleak` o equivalente.

**Pista:** Para los leaks de tipo canal, la solución estándar es agregar un
`done` channel o usar `context.Context` con cancelación. Para los de WaitGroup,
garantizar que `Done()` se llama incluso si hay error o panic (usar `defer`).

**Implementar en:** Go · Java · Python

---

### Ejercicio 4.4.2 — Leak en tests: el test que pasa pero deja goroutines

**Enunciado:** Este test pasa pero deja goroutines activas que interfieren
con otros tests. Identifica el leak y arréglalo:

```go
func TestServidor(t *testing.T) {
    servidor := NuevoServidor()
    servidor.Iniciar()

    resp := servidor.Procesar(Request{Datos: "test"})
    assert.Equal(t, "ok", resp.Estado)

    // ← falta algo aquí
}

func NuevoServidor() *Servidor {
    s := &Servidor{
        jobs: make(chan Request, 100),
    }
    return s
}

func (s *Servidor) Iniciar() {
    for i := 0; i < 5; i++ {
        go s.worker()  // workers que nunca terminan
    }
}
```

**Restricciones:** La corrección debe agregar un mecanismo de parada limpia
al servidor. El test debe llamar `servidor.Detener()` al final.
Verifica con `goleak.VerifyNone(t)`.

**Pista:** El patrón estándar: `Iniciar` guarda un `cancel context.CancelFunc`.
`Detener` llama `cancel()` y espera con `WaitGroup`. Los workers hacen
`select { case job := <-jobs: ... case <-ctx.Done(): return }`.

**Implementar en:** Go · Java · Python

---

### Ejercicio 4.4.3 — Leak por goroutine bloqueada en send

**Enunciado:** El patrón más sutil de leak: la goroutine que envía al canal
se queda bloqueada porque el receptor ya terminó.

```go
func BuscarConTimeout(query string, timeout time.Duration) (Result, error) {
    ch := make(chan Result)  // sin buffer
    go func() {
        ch <- buscar(query)  // ← si el timeout dispara antes, nadie leerá ch
    }()

    select {
    case result := <-ch:
        return result, nil
    case <-time.After(timeout):
        return Result{}, ErrTimeout  // la goroutine de búsqueda queda bloqueada
    }
}
```

**Restricciones:** Arréglalo usando un canal con buffer de 1.
Explica por qué buffer de 1 (no 0, no 2) es la solución correcta.

**Pista:** Con buffer 1, la goroutine de búsqueda puede enviar aunque nadie
esté leyendo — su resultado va al buffer y la goroutine termina limpiamente.
Si el llamador ya leyó (caso no-timeout), el buffer puede estar vacío.
Si el timeout disparó, el buffer tiene el resultado que nadie leerá —
eventualmente el GC libera el canal y el resultado.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 4.4.4 — Leak en goroutines de larga vida con referencias

**Enunciado:** Una goroutine de larga vida que tiene una referencia a
una request HTTP completa (headers, body, etc.) retiene esa memoria
aunque solo necesite un campo pequeño.

```go
func procesarAsync(req *http.Request) {
    userID := req.Header.Get("X-User-ID")
    go func() {
        // solo necesita userID, pero req sigue en la closure
        time.Sleep(1 * time.Minute)
        log.Printf("Procesando usuario %s", userID)
        // req (con todo su contenido) se libera solo cuando esta goroutine termine
    }()
}
```

**Restricciones:** Demuestra la retención de memoria con `runtime.ReadMemStats`.
Arréglalo extrayendo solo los campos necesarios antes de lanzar la goroutine.

**Pista:** La regla: si una goroutine larga captura un puntero en su closure,
ese puntero (y todo lo que apunta) vive hasta que la goroutine termina.
Antes de lanzar goroutines largas, copia los valores necesarios a variables locales.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 4.4.5 — Sistema de monitoreo de goroutines en producción

**Enunciado:** Implementa un endpoint de diagnóstico que expone el número
de goroutines y sus estados a lo largo del tiempo:

```go
// GET /debug/goroutines
// Retorna: conteo actual, historial de los últimos 5 minutos,
//          goroutines en estado de espera más de 30 segundos

type EstadoGoroutines struct {
    Total        int
    PorEstado    map[string]int  // running, chan receive, select, sleep, etc.
    Sospechosas  []InfoGoroutine // llevan mucho tiempo en el mismo estado
}
```

**Restricciones:** El endpoint no debe afectar el rendimiento del sistema —
`runtime.Stack(buf, true)` detiene todas las goroutines brevemente.
Llámalo máximo una vez cada 5 segundos.

**Pista:** Parsear el output de `runtime.Stack` para extraer el estado
de cada goroutine es el núcleo del ejercicio. El estado aparece entre
corchetes: `goroutine 7 [chan receive, 42 minutes]` — el número de minutos
indica cuánto lleva en ese estado.

**Implementar en:** Go

---

## Sección 4.5 — GOMAXPROCS y Paralelismo Real

`GOMAXPROCS` controla cuántos núcleos físicos puede usar Go simultáneamente.
Su valor por defecto es `runtime.NumCPU()` — el número de CPUs lógicas disponibles.

**Cuándo cambiar GOMAXPROCS:**

```go
// Ver el valor actual
fmt.Println(runtime.GOMAXPROCS(0))  // 0 = solo consultar, no cambiar

// Cambiar (retorna el valor anterior)
anterior := runtime.GOMAXPROCS(4)  // usar exactamente 4 CPUs

// En containers: el default puede ser incorrecto
// Linux containers ven todos los CPUs del host, no los asignados al container
// go.uber.org/automaxprocs lee los cgroups correctamente
```

**La ley de Amdahl aplicada a Go:**

```
Si un programa tiene:
  F = fracción secuencial (no paralelizable)
  1-F = fracción paralelizable

Speedup máximo con N procesadores = 1 / (F + (1-F)/N)

Para F=0.1 (10% secuencial):
  N=2:   speedup = 1 / (0.1 + 0.45) = 1.82x
  N=4:   speedup = 1 / (0.1 + 0.225) = 3.08x
  N=8:   speedup = 1 / (0.1 + 0.1125) = 4.71x
  N=∞:   speedup = 1 / 0.1 = 10x máximo

Implicación: añadir más CPUs tiene retornos decrecientes.
El cuello de botella es la parte secuencial, no el número de CPUs.
```

---

### Ejercicio 4.5.1 — Medir el speedup real con GOMAXPROCS

**Enunciado:** Implementa una función CPU-bound (calcular N números primos)
y mídela con `GOMAXPROCS = {1, 2, 4, 8, runtime.NumCPU()}`.

Compara el speedup real con el predicho por la ley de Amdahl.
Identifica cuál es la fracción secuencial de tu implementación.

**Restricciones:** Usa la Criba de Eratóstenes del Cap.14 §2 del repo
de algoritmos. Divide el trabajo en chunks iguales para los workers.
El resultado debe ser idéntico con cualquier GOMAXPROCS.

**Pista:** El speedup real suele ser menor que el de Amdahl por:
overhead de sincronización, false sharing en caché, y la fracción secuencial
de unir los resultados. Mide con `testing.Benchmark` para resultados reproducibles.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 4.5.2 — CPU-bound vs I/O-bound: GOMAXPROCS óptimo

**Enunciado:** Para tareas CPU-bound, el GOMAXPROCS óptimo es `NumCPU`.
Para tareas I/O-bound, puede ser mayor porque las goroutines bloqueadas
en I/O liberan el P para otras. Verifica esto empíricamente:

1. Tarea CPU-bound (calcular hashes): mide throughput con GOMAXPROCS = 1..16
2. Tarea I/O-bound (simular con Sleep): mide throughput con GOMAXPROCS = 1..16

**Restricciones:** La tarea I/O-bound simula 10ms de I/O con `time.Sleep`.
El número óptimo de workers para I/O-bound puede ser mucho mayor que NumCPU.

**Pista:** Para I/O-bound, el throughput óptimo se alcanza cuando
`N_workers ≈ N_cpu / (1 - utilización_cpu)`. Con 1ms de CPU y 10ms de I/O,
`utilización_cpu ≈ 0.09`, así que `N_workers_optimo ≈ N_cpu / 0.91 ≈ 1.1 * N_cpu`.
Para duraciones de I/O más largas, el multiplicador crece.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 4.5.3 — False sharing entre goroutines (conexión con Cap.19)

**Enunciado:** En el Cap.19 §1.3 del repo de algoritmos vimos false sharing
entre threads. En Go, la situación es similar pero con goroutines.
Demuestra false sharing entre dos goroutines que modifican campos contiguos
de un struct:

```go
type Contadores struct {
    A int64  // goroutine 1 modifica A
    B int64  // goroutine 2 modifica B
    // A y B están en la misma línea de caché (64 bytes)
}

type ContadoresSinFalseSharing struct {
    A   int64
    _   [56]byte  // padding para separar A y B en líneas distintas
    B   int64
}
```

**Restricciones:** Con `GOMAXPROCS=2`, la versión sin padding debe ser
~2-3x más rápida que la versión con false sharing.

**Pista:** False sharing en Go ocurre igual que en C/C++/Java — las líneas
de caché son compartidas entre los núcleos físicos donde ejecutan los
threads del SO subyacentes. El padding asegura que A y B están en
líneas de caché distintas y no se invalidan mutuamente.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 4.5.4 — GOMAXPROCS en contenedores: el bug más común

**Enunciado:** Un servicio Go en un contenedor Docker tiene asignados 2 CPUs
pero `runtime.NumCPU()` retorna 32 (los CPUs del host). Esto causa que
el scheduler intente usar 32 P sobre 2 CPUs reales — más context switches,
más overhead.

Implementa el problema y la solución usando `automaxprocs`:

```go
import _ "go.uber.org/automaxprocs"  // lee cgroups y ajusta GOMAXPROCS

func main() {
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    // Con automaxprocs: GOMAXPROCS = 2 (los asignados al container)
    // Sin automaxprocs: GOMAXPROCS = 32 (todos los del host)
}
```

**Restricciones:** Simula el ambiente de contenedor ajustando manualmente
`GOMAXPROCS` a 2 y midiendo el impacto en throughput vs el valor incorrecto de 32.

**Pista:** Con GOMAXPROCS=32 y 2 CPUs reales, el scheduler crea 32 threads
del SO que compiten por 2 CPUs. El overhead de context switching degrada
el rendimiento un 20-40% comparado con GOMAXPROCS=2.

**Implementar en:** Go

---

### Ejercicio 4.5.5 — Saturación de CPU: encontrar el límite

**Enunciado:** Implementa un benchmark que encuentra el punto de saturación
de tu sistema: el número de goroutines concurrentes a partir del cual
el throughput deja de crecer (o decrece).

Para una tarea CPU-bound de 1ms, el throughput teórico máximo es
`NumCPU * 1000 operaciones/segundo`. Verifica dónde está el punto real.

**Restricciones:** Varía el número de workers de 1 a `4 * NumCPU`.
El benchmark debe ejecutarse durante al menos 5 segundos por configuración
para resultados estables.

**Pista:** El throughput crece con el número de workers hasta `≈ NumCPU`,
luego se estabiliza o decrece ligeramente por overhead de scheduling.
El punto de decrecimiento depende de la carga del sistema — en un sistema
cargado (otras aplicaciones corriendo), el punto de saturación es más bajo.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 4.6 — Goroutines vs Threads: Cuándo Importa la Diferencia

En la práctica, la diferencia entre goroutines y threads del SO importa
en escenarios específicos. Fuera de esos escenarios, puedes programar
con goroutines como si fueran threads ligeros y el runtime hace lo correcto.

**Cuándo la diferencia importa:**

```
1. Llamadas a código C (cgo):
   Las goroutines que hacen cgo se anclan a threads del SO.
   Si tienes muchas goroutines haciendo cgo, puedes agotar los threads.
   runtime.LockOSThread() fija explícitamente una goroutine a un thread.

2. APIs que requieren thread affinity:
   OpenGL, algunas APIs de GUI, TLS (thread-local storage de C)
   requieren que las llamadas vengan del mismo thread.
   Solución: runtime.LockOSThread() en esa goroutine.

3. Syscalls bloqueantes (no-Go):
   time.Sleep, net.Conn.Read → el runtime gestiona esto (non-blocking internamente)
   Llamadas a C que bloquean → crean un nuevo thread del SO

4. Señales del SO:
   Las señales se entregan al thread del SO, no a la goroutine.
   El runtime de Go gestiona las señales automáticamente.
```

**Cuándo NO importa la diferencia (la mayoría de los casos):**

```
- Concurrencia de aplicación normal (web servers, workers, etc.)
- I/O de red (Go usa epoll/kqueue internamente — non-blocking)
- I/O de archivos (el runtime usa thread pool para esto)
- Comunicación entre goroutines con canales
```

---

### Ejercicio 4.6.1 — runtime.LockOSThread para thread affinity

**Enunciado:** Implementa un renderer de OpenGL simulado que requiere
thread affinity. La goroutine del renderer debe ejecutar siempre en el
mismo thread del SO.

```go
func IniciarRenderer() {
    runtime.LockOSThread()  // esta goroutine siempre en este thread
    defer runtime.UnlockOSThread()

    for cmd := range comandos {
        // todas las llamadas a "OpenGL" desde el mismo thread
        ejecutarComandoGL(cmd)
    }
}
```

**Restricciones:** Verifica con `runtime.Stack` que la goroutine del renderer
siempre ejecuta en el mismo thread (el ID del thread debe ser constante).

**Pista:** En Go no hay una API directa para obtener el ID del thread.
Se puede inferir usando `syscall.Gettid()` en Linux o equivalente.
Un cambio en el thread ID indica que `LockOSThread` no está funcionando.

**Implementar en:** Go

---

### Ejercicio 4.6.2 — cgo y la interacción con el scheduler

**Enunciado:** Implementa una función que llama a código C bloqueante y
observa el impacto en el scheduler de Go. Cuando una goroutine hace cgo
con una llamada bloqueante, el runtime crea un nuevo thread del SO para
las otras goroutines.

```go
// #include <unistd.h>
// void llamadaBloqueanteC() { sleep(1); }
import "C"

func usarCgo() {
    C.llamadaBloqueanteC()  // bloquea el thread del SO durante 1 segundo
}
```

**Restricciones:** Lanza 10 goroutines que hacen cgo bloqueante simultáneamente.
Observa cómo el número de threads del SO (`/proc/self/status` en Linux)
crece a 10+ aunque GOMAXPROCS=4.

**Pista:** El runtime de Go crea un nuevo M (thread del SO) cuando una goroutine
entra en cgo bloqueante, para no dejar a los otros P sin threads. Esto significa
que cgo intensivo puede crear muchos threads del SO — a diferencia de I/O de red
que es non-blocking internamente.

**Implementar en:** Go (Linux)

---

### Ejercicio 4.6.3 — Comparar goroutines con threads del SO en Java

**Enunciado:** Implementa el mismo servidor concurrente (acepta N conexiones
y responde a cada una en paralelo) en Go con goroutines y en Java con
threads del SO (pre-Java 21) y con Virtual Threads (Java 21+).

Compara:
- Memoria usada con 10,000 conexiones simultáneas
- Tiempo de creación de 10,000 unidades de concurrencia
- Throughput bajo carga

**Restricciones:** Usa el mismo protocolo y la misma lógica de negocio
en ambas implementaciones. El cliente de prueba es el mismo.

**Pista:** Java Virtual Threads (Project Loom, Java 21+) son la respuesta
de Java a las goroutines de Go — M:N scheduling sobre threads del SO.
Los Virtual Threads resuelven el mismo problema que Go resolvió hace 10 años,
pero con la retrocompatibilidad de la JVM.

**Implementar en:** Go · Java 21+

---

### Ejercicio 4.6.4 — Goroutines vs async/await en Python

**Enunciado:** Implementa el mismo pipeline de transformación de datos
en Go con goroutines y en Python con asyncio. Compara:
- Facilidad de razonamiento sobre el código
- Manejo de errores
- Rendimiento relativo para I/O bound vs CPU bound

**Restricciones:** La comparación debe ser justa — misma lógica, mismo
número de workers/coroutines, mismo tipo de carga.

**Pista:** Python asyncio es concurrencia cooperativa en un solo thread
(como Go con GOMAXPROCS=1 y sin preempción). El GIL de CPython impide
paralelismo real en CPU-bound. Para CPU-bound real, Python necesita
multiprocessing — no coroutines.

**Implementar en:** Go · Python

---

### Ejercicio 4.6.5 — El modelo de ownership de Rust como alternativa

**Enunciado:** Implementa en Rust el contador concurrente del inicio del Cap.01.
El compilador debe rechazar la versión sin sincronización y aceptar la versión
con `Arc<Mutex<i32>>` o `Arc<AtomicI32>`.

Compara el mensaje de error del compilador de Rust con el output de `go run -race`.
¿Cuál es más útil para el desarrollador?

**Restricciones:** La versión incorrecta debe no compilar — ese es el punto.
La versión correcta con Mutex y con Atomic deben compilar y funcionar.

**Pista:** Rust garantiza en tiempo de compilación:
`Send` significa que un tipo es seguro de enviar entre threads.
`Sync` significa que es seguro compartirlo entre threads.
`i32` no implementa estos traits automáticamente para acceso concurrente.
`Arc<Mutex<i32>>` sí los implementa — es la forma correcta de compartirlo.

**Implementar en:** Rust · Go (para comparar errores de compilación vs race detector)

---

## Sección 4.7 — Perfilar y Optimizar Goroutines

El profiler de Go tiene soporte nativo para concurrencia. Además del CPU
profiler estándar, `go tool trace` muestra la actividad de goroutines
con microsegundo de resolución.

**Las tres herramientas de profiling para concurrencia:**

```bash
# 1. go tool pprof — CPU y memoria
go test -cpuprofile cpu.prof -bench .
go tool pprof cpu.prof

# 2. go tool trace — actividad detallada del scheduler
go test -trace trace.out -bench .
go tool trace trace.out
# Abre browser con visualización de goroutines, GC, syscalls, etc.

# 3. runtime/metrics — métricas del scheduler en tiempo real
import "runtime/metrics"
// Lee: /sched/goroutines:goroutines, /sched/latencies:seconds, etc.
```

**Lo que muestra go tool trace:**

```
- Timeline de cada goroutine: cuándo ejecuta, cuándo espera, cuándo duerme
- Eventos del GC: stop-the-world, barrido, etc.
- Syscalls bloqueantes: qué goroutines bloquean y por cuánto tiempo
- Network poller: eventos de I/O
- Contención de mutexes: qué goroutines esperan qué locks
```

---

### Ejercicio 4.7.1 — Generar e interpretar un trace

**Enunciado:** Genera un trace del Worker Pool del Cap.03 §3.1 bajo carga
y responde estas preguntas mirando el trace:

1. ¿Cuántas goroutines están Running simultáneamente en el pico?
2. ¿Cuánto tiempo pasan los workers en estado Runnable antes de ejecutar?
3. ¿Hay periodos donde todos los P están ociosos? ¿Por qué?
4. ¿El GC interrumpe el trabajo? ¿Con qué frecuencia?

**Restricciones:** El trace debe capturarse durante al menos 5 segundos de carga.
Las respuestas deben estar respaldadas por capturas del trace.

**Pista:** `go test -trace trace.out -run TestWorkerPool -benchtime=5s`
El trace de Go muestra cada goroutine como una línea horizontal con colores:
azul = Running, verde = Runnable, rosa = Waiting.

**Implementar en:** Go

---

### Ejercicio 4.7.2 — Encontrar el cuello de botella de scheduling

**Enunciado:** Este sistema tiene un cuello de botella de scheduling
que no es obvio sin el trace. Identifícalo:

```go
func procesarLote(items []Item) []Result {
    results := make([]Result, len(items))
    var wg sync.WaitGroup

    for i, item := range items {
        wg.Add(1)
        i, item := i, item
        go func() {
            defer wg.Done()
            mu.Lock()           // ← ¿cuánto tiempo esperan aquí?
            results[i] = procesar(item)
            mu.Unlock()
        }()
    }
    wg.Wait()
    return results
}
```

**Restricciones:** Usa `go tool trace` para medir el tiempo que cada goroutine
pasa esperando el mutex. Propón una solución que elimine la contención.

**Pista:** Si `procesar(item)` es rápido pero hay N goroutines todas esperando
el mutex, el trace mostrará todas en estado `semacquire` — contención de mutex.
La solución es eliminar el mutex procesando el item fuera del lock o
usando canales para recolectar resultados.

**Implementar en:** Go

---

### Ejercicio 4.7.3 — Optimizar el GC con sync.Pool

**Enunciado:** Un worker pool que procesa mensajes aloca un buffer por mensaje.
Con alta frecuencia de mensajes, el GC trabaja intensamente reciclando esos buffers.
Usa `sync.Pool` para reutilizar buffers y mide la reducción de presión en el GC.

```go
// Sin pool — aloca y libera en cada mensaje
func procesarMensaje(datos []byte) Result {
    buffer := make([]byte, 64*1024)  // 64KB por mensaje
    copy(buffer, datos)
    return transformar(buffer)
}

// Con pool — reutiliza buffers entre mensajes
var bufPool = sync.Pool{
    New: func() any { return make([]byte, 64*1024) },
}

func procesarMensajeConPool(datos []byte) Result {
    buffer := bufPool.Get().([]byte)
    defer bufPool.Put(buffer)
    copy(buffer, datos)
    return transformar(buffer)
}
```

**Restricciones:** Mide con `runtime.ReadMemStats`: `NumGC`, `PauseNs`, `TotalAlloc`.
La versión con pool debe reducir `NumGC` en al menos 50% para 1000 mensajes/segundo.

**Pista:** `sync.Pool` es vaciado por el GC — no es una caché permanente.
Su valor es que durante una fase activa de procesamiento, los buffers se reutilizan
entre goroutines sin alocación. Es la optimización de GC más impactante en
sistemas Go de alta frecuencia de mensajes.

**Implementar en:** Go · Java (`ThreadLocal` pools) · C# (`ArrayPool<T>`)

---

### Ejercicio 4.7.4 — Contención de mutex visible en trace

**Enunciado:** Diseña un microbenchmark que muestra la diferencia de rendimiento
entre alta contención y baja contención de mutex, visible en el trace de Go:

1. Alta contención: 100 goroutines, un mutex compartido, sección crítica de 1µs
2. Baja contención: 100 goroutines, 10 mutexes con sharding, sección crítica de 1µs

Mide throughput y latencia de la sección crítica en ambos casos.

**Restricciones:** El sharding debe distribuir uniformemente la carga entre
los 10 mutexes — usa hash del ID del item para asignar mutex.

**Pista:** La diferencia de throughput puede ser 5-10x entre alta y baja contención.
El trace mostrará en el caso de alta contención que las goroutines pasan
más tiempo en `semacquire` que en estado `Running` — el 90% del tiempo esperando.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 4.7.5 — Dashboard de salud del runtime de Go

**Enunciado:** Implementa un endpoint HTTP `/debug/runtime` que expone
las métricas más relevantes del runtime de Go para diagnóstico en producción:

```go
type RuntimeStats struct {
    // Goroutines
    GoroutinesActivas    int
    GoroutinesEnEspera   int

    // Scheduler
    GOMAXPROCS           int
    NumThreads           int
    LatenciaScheduling   time.Duration  // p99

    // Memoria
    HeapEnUso           uint64
    HeapIdle            uint64
    StackEnUso          uint64
    NumGC               uint32
    UltimaGCDuracion    time.Duration

    // Canales (via runtime/metrics en Go 1.16+)
    // ...
}
```

**Restricciones:** El endpoint debe ser barato de llamar — no puede detener
el programa para obtener métricas. Usa `runtime/metrics` en lugar de
`runtime.ReadMemStats` donde sea posible (menos overhead).

**Pista:** `runtime/metrics` (Go 1.16+) expone métricas sin stop-the-world.
`runtime.ReadMemStats` requiere stop-the-world para obtener datos consistentes.
Para producción, preferir `runtime/metrics` para métricas frecuentes.

**Implementar en:** Go

---

## Resumen del capítulo

**El modelo GMP en una línea:**
M goroutines se multiplexan sobre N threads del SO que ejecutan en P procesadores.
El scheduler balancea automáticamente con work stealing. La preempción evita
que una goroutine monopolice un P.

**Los números que importan:**

| Operación | Costo aproximado |
|---|---|
| Crear goroutine | ~400 ns |
| Context switch de goroutine | ~200 ns |
| Crear thread del SO | ~10,000 ns |
| Context switch de thread del SO | ~1,000 ns |
| Goroutine bloqueada en canal | 0 CPU (waiting state) |
| Goroutine en Sleep | 0 CPU (waiting state) |
| Stack inicial de goroutine | 2-8 KB |
| Stack de thread del SO | 8 MB |

**Las tres reglas de oro para goroutines:**

```
1. Toda goroutine que lanzas necesita una forma de terminar
   (ctx.Done(), canal cerrado, señal explícita)

2. El llamador que crea la goroutine es responsable de esperarla
   (WaitGroup, canal de done) y de detectar sus errores

3. Una goroutine que no puede terminar es un leak —
   detecta con goleak en tests, con /debug/runtime en producción
```

## La pregunta que guía el Cap.05

> Los Cap.01–04 cubrieron los problemas, las herramientas, los patrones
> y el runtime de Go. Ahora llega la pregunta de entrevistas senior:
> ¿Cómo usas todo esto en un lenguaje específico?
>
> El Cap.05 entra en Go con profundidad: goroutines como primitiva de diseño,
> canales como forma de pensar el problema (no solo de implementarlo),
> y los patrones idiomáticos de Go que no tienen equivalente directo
> en Java o Python.
