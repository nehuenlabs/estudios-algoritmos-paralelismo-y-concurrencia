# Guía de Ejercicios — Java: El Ecosistema de Concurrencia más Rico

> Implementar todos los ejercicios en **Java 21+**.
> Comparaciones con Go y Rust donde iluminen el contraste.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: 25 años de evolución

Java es el lenguaje donde se inventaron muchas de las abstracciones de concurrencia
que usamos hoy. La evolución es visible en el código:

```java
// Java 1.0 (1996): synchronized y wait/notify
synchronized (this) {
    while (cola.isEmpty()) wait();
    return cola.remove();
}

// Java 5 (2004): java.util.concurrent — Doug Lea
// El paquete más influyente de la historia de Java
ExecutorService pool = Executors.newFixedThreadPool(8);
Future<Result> future = pool.submit(() -> calcular());

// Java 8 (2014): CompletableFuture — programación reactiva
CompletableFuture.supplyAsync(() -> buscarUsuario(id))
    .thenCompose(u -> buscarOrdenes(u.id()))
    .thenApply(ordenes -> calcularTotal(ordenes))
    .exceptionally(e -> 0.0);

// Java 21 (2023): Virtual Threads — Project Loom
// Millones de threads con el modelo mental simple de threads bloqueantes
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // Este thread es virtual — puede haber 1M simultáneos
        String resultado = httpClient.get("https://api.example.com/data");
        // La llamada bloquea el virtual thread pero NO el OS thread
    });
}
```

---

## El modelo mental de Java 21

```
Java 1-20:
  OS Thread (pesado, ~1MB)
    └─ Java Thread (1:1 con OS Thread)
    
  Problema: 10,000 threads simultáneos = 10 GB de memoria + overhead del OS

Java 21 (Virtual Threads / Project Loom):
  OS Thread (carrier thread, ~8 por núcleo)
    └─ Virtual Thread (ligero, ~1KB) ← puede haber millones
         └─ cuando hace I/O, el carrier thread lo "estaciona" y
              ejecuta otro Virtual Thread
    
  Resultado: modelo mental de Thread-per-request pero sin el costo
  1,000,000 virtual threads = ~1 GB de memoria (vs 1000 GB con OS threads)
```

---

## Tabla de contenidos

- [Sección 14.1 — Fundamentos: synchronized, volatile, y el JMM](#sección-141--fundamentos-synchronized-volatile-y-el-jmm)
- [Sección 14.2 — java.util.concurrent: el toolkit de Doug Lea](#sección-142--javautilconcurrent-el-toolkit-de-doug-lea)
- [Sección 14.3 — CompletableFuture: composición asíncrona](#sección-143--completablefuture-composición-asíncrona)
- [Sección 14.4 — Virtual Threads: Project Loom en la práctica](#sección-144--virtual-threads-project-loom-en-la-práctica)
- [Sección 14.5 — Structured Concurrency: Java 21+](#sección-145--structured-concurrency-java-21)
- [Sección 14.6 — Reactive Streams: Project Reactor y RxJava](#sección-146--reactive-streams-project-reactor-y-rxjava)
- [Sección 14.7 — Java vs Go vs Rust: la comparación completa](#sección-147--java-vs-go-vs-rust-la-comparación-completa)

---

## Sección 14.1 — Fundamentos: synchronized, volatile, y el JMM

El Java Memory Model (JMM) define las garantías de visibilidad y ordering
entre threads. Es más complejo que el memory model de Go pero similar al de C++11.

**Las tres primitivas fundamentales:**

```java
// 1. synchronized: exclusión mutua + visibilidad
synchronized (lock) {
    // Solo un thread a la vez
    // Todas las escrituras anteriores son visibles al adquirir el lock
    // Todas las escrituras aquí son visibles al liberar el lock
}

// 2. volatile: visibilidad sin exclusión mutua
private volatile boolean activo = true;
// Leer un volatile: garantiza que ves el último valor escrito
// Escribir un volatile: garantiza que tu escritura es visible a todos

// 3. AtomicXxx: operaciones atómicas con Ordering configurable
AtomicLong contador = new AtomicLong(0);
long nuevo = contador.incrementAndGet();  // read-modify-write atómico
```

---

### Ejercicio 14.1.1 — El JMM en la práctica: visibility bugs

**Enunciado:** Demuestra los tres tipos principales de visibility bugs en Java:

```java
// Bug 1: sin volatile — el thread puede leer el valor en caché
private boolean activo = true;  // sin volatile

void detener() { activo = false; }
void correr() {
    while (activo) { /* puede iterar para siempre */ }
}

// Bug 2: double-checked locking sin volatile (Java pre-5)
private Singleton instancia;
Singleton obtener() {
    if (instancia == null) {
        synchronized (this) {
            if (instancia == null)
                instancia = new Singleton();  // puede publicarse parcialmente
        }
    }
    return instancia;
}

// Bug 3: acceso no sincronizado a long/double
// En JVMs de 32 bits, los accesos a long/double pueden no ser atómicos
private long contador;
void incrementar() { contador++; }  // puede ser no-atómico
```

**Restricciones:** Para cada bug, demostrar el comportamiento incorrecto con
un test de stress (1M iteraciones, múltiples threads). Luego implementar la corrección.

**Pista:** El Bug 1 puede no manifestarse en x86 (TSO) pero sí en ARM.
En x86, añadir `volatile` hace que el JIT no guarde el valor en registro.
El Bug 2 se corrige con `volatile` en la declaración de `instancia`.
El Bug 3 en JVMs de 64 bits modernas casi nunca ocurre, pero `AtomicLong` es la solución correcta.

**Implementar en:** Java

---

### Ejercicio 14.1.2 — happens-before en el JMM

**Enunciado:** El JMM define "happens-before" como la relación que garantiza
visibilidad. Las relaciones happens-before en Java:

```
1. Dentro de un thread: cada acción pasa-antes que la siguiente
2. Monitor lock: unlock pasa-antes que el siguiente lock del mismo monitor
3. volatile write: write pasa-antes que el siguiente read del mismo campo
4. Thread start: start() pasa-antes que cualquier acción en el thread hijo
5. Thread join: cualquier acción en el thread pasa-antes que join() retorna
6. Inicialización de clase: completar el static initializer pasa-antes que
   cualquier acceso al campo static
```

Diseña tests que verifiquen cada relación happens-before:

```java
// Test para la relación #2 (monitor lock):
int[] dato = {0};
Thread t1 = new Thread(() -> {
    synchronized (dato) { dato[0] = 42; }
});
Thread t2 = new Thread(() -> {
    synchronized (dato) {
        assert dato[0] == 42;  // garantizado por happens-before del lock
    }
});
```

**Restricciones:** Para cada relación, el test debe fallar si se rompe
la relación (comentar el synchronized, el volatile, etc.).

**Pista:** Los tests de JMM son difíciles de hacer deterministas porque
el JMM solo garantiza el behavior en ciertos interleavings, no todos.
La herramienta `jcstress` de OpenJDK es el framework estándar para tests del JMM.

**Implementar en:** Java (con jcstress si está disponible)

---

### Ejercicio 14.1.3 — ForkJoinPool: el scheduler de tareas de Java

**Enunciado:** `ForkJoinPool` es el equivalente en Java del trabajo stealing
del Cap.08 §8.6. Implementa un merge sort paralelo con `RecursiveTask`:

```java
class MergeSortTask extends RecursiveTask<int[]> {
    private final int[] array;
    private static final int UMBRAL = 1000;

    @Override
    protected int[] compute() {
        if (array.length <= UMBRAL) {
            return mergeSort(array);  // secuencial para arrays pequeños
        }
        int mid = array.length / 2;
        MergeSortTask izq = new MergeSortTask(Arrays.copyOfRange(array, 0, mid));
        MergeSortTask der = new MergeSortTask(Arrays.copyOfRange(array, mid, array.length));
        izq.fork();                   // ejecutar izq en background
        int[] resultDer = der.compute();  // ejecutar der en el thread actual
        int[] resultIzq = izq.join();     // esperar izq
        return merge(resultIzq, resultDer);
    }
}
```

**Restricciones:** El umbral debe ser configurable y determinarse experimentalmente.
Mide el speedup para arrays de 1M, 10M, y 100M elementos.
El patrón `izq.fork(); der.compute(); izq.join()` es el correcto — ¿por qué?

**Pista:** `fork()` envía la tarea a la cola de trabajo del thread actual.
`compute()` ejecuta en el thread actual (sin crear nueva tarea).
Si hicieras `izq.fork(); der.fork()`, tendrías que `join()` ambos,
lo que es equivalente pero crea una tarea extra innecesaria.
El patrón `fork-compute-join` es más eficiente: el thread actual hace trabajo útil
en lugar de solo esperar.

**Implementar en:** Java

---

### Ejercicio 14.1.4 — StampedLock: optimistic locking

**Enunciado:** `StampedLock` (Java 8+) soporta lecturas optimistas — leer
sin adquirir el lock, y luego validar que nadie escribió:

```java
StampedLock lock = new StampedLock();
double x, y;  // coordenadas de un punto

double distancia() {
    // Intento optimista primero (más rápido)
    long sello = lock.tryOptimisticRead();
    double cx = x, cy = y;
    if (!lock.validate(sello)) {
        // Alguien escribió durante la lectura — adquirir lock real
        sello = lock.readLock();
        try {
            cx = x; cy = y;
        } finally {
            lock.unlockRead(sello);
        }
    }
    return Math.sqrt(cx*cx + cy*cy);
}
```

**Restricciones:** Mide el throughput del `StampedLock` (optimistic) vs
`ReentrantReadWriteLock` para 90% reads y 10% writes.
El `StampedLock` debe ganar por >2x bajo alta contención de lectura.

**Pista:** La lectura optimista de `StampedLock` no adquiere ningún lock —
solo lee un contador de versión y lo valida al final. Si no hubo escritura,
es gratis (solo 2 accesos a memoria). Si hubo escritura, paga el costo del lock real.
Con 90% reads y 10% writes, el 90% de las lecturas son gratis.
`ReentrantReadWriteLock` siempre paga el costo del lock de lectura.

**Implementar en:** Java

---

### Ejercicio 14.1.5 — ThreadLocal: estado por thread

**Enunciado:** `ThreadLocal` provee un valor separado por thread, sin sincronización:

```java
// Un ThreadLocal para el contexto de request (usado en sistemas web)
private static final ThreadLocal<RequestContext> contexto = new ThreadLocal<>();

void manejarRequest(Request req) {
    contexto.set(new RequestContext(req.userId(), req.traceId()));
    try {
        procesarNegocio();  // puede acceder a contexto.get() sin pasar parámetros
    } finally {
        contexto.remove();  // CRÍTICO: limpiar para evitar memory leaks en thread pools
    }
}

void procesarNegocio() {
    RequestContext ctx = contexto.get();  // sin pasar el contexto explícitamente
    log.info("procesando para usuario: " + ctx.userId());
}
```

**Restricciones:** Implementar un sistema de logging con `ThreadLocal` que
añade automáticamente el ID de usuario y trace ID a cada log.
El `remove()` en el finally es obligatorio — demostrar el memory leak sin él.

**Pista:** En thread pools (ExecutorService), los threads se reutilizan.
Si no haces `remove()`, el valor del `ThreadLocal` del request anterior
sigue presente para el próximo request ejecutado en ese thread.
`InheritableThreadLocal` propaga el valor a threads hijo — útil pero peligroso:
si el thread hijo vive más que el padre, puede ver datos del padre ya desaparecidos.

**Implementar en:** Java

---

## Sección 14.2 — java.util.concurrent: El Toolkit de Doug Lea

`java.util.concurrent` (JDK 5, 2004) es la biblioteca de concurrencia más
influyente de la historia de Java. Las estructuras:

```
Executors y pools:
  ExecutorService, ScheduledExecutorService
  ThreadPoolExecutor, ForkJoinPool

Estructuras de datos:
  ConcurrentHashMap, ConcurrentLinkedQueue, CopyOnWriteArrayList
  BlockingQueue (ArrayBlockingQueue, LinkedBlockingQueue, PriorityBlockingQueue)
  ConcurrentSkipListMap (skip list del Cap.12!)

Sincronización:
  Semaphore, CountDownLatch, CyclicBarrier, Phaser
  ReentrantLock, ReentrantReadWriteLock, StampedLock
  Condition (equivalente a Condvar de Rust)

Atómicos:
  AtomicInteger, AtomicLong, AtomicReference, AtomicStampedReference
  LongAdder, LongAccumulator (counters con menos contención que AtomicLong)
```

---

### Ejercicio 14.2.1 — ThreadPoolExecutor: configurar el pool correctamente

**Enunciado:** `ThreadPoolExecutor` tiene más parámetros que `Executors.newFixedThreadPool`:

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    corePoolSize,    // threads siempre activos (aunque no haya trabajo)
    maximumPoolSize, // máximo de threads simultáneos
    keepAliveTime,   // cuánto tiempo vive un thread extra sin trabajo
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(queueCapacity),  // cola de tareas pendientes
    new ThreadFactory() { /* nombre de threads personalizado */ },
    new RejectedExecutionHandler() { /* qué hacer si la cola está llena */ }
);
```

**Las políticas de rechazo:**
```java
ThreadPoolExecutor.AbortPolicy      // lanzar RejectedExecutionException (default)
ThreadPoolExecutor.CallerRunsPolicy // el thread que llama execute() ejecuta la tarea
ThreadPoolExecutor.DiscardPolicy    // silenciosamente descartar la tarea
ThreadPoolExecutor.DiscardOldestPolicy // descartar la tarea más antigua de la cola
```

**Restricciones:** Implementa un pool con `CallerRunsPolicy` y demuestra que
actúa como backpressure: cuando el pool está saturado, el producer se ralentiza
en lugar de acumular tareas o lanzar excepciones.
Mide el throughput vs `AbortPolicy` para distintas tasas de arrival.

**Pista:** `CallerRunsPolicy` es el mecanismo de backpressure más simple en Java.
Si el pool está lleno y la cola también, el thread que llamó `submit()` ejecuta
la tarea directamente — efectivamente bloqueando al producer hasta que el pool tenga capacidad.
Es análogo al envío a un canal bloqueante en Go.

**Implementar en:** Java

---

### Ejercicio 14.2.2 — ConcurrentHashMap: cuándo y cómo

**Enunciado:** `ConcurrentHashMap` es el HashMap concurrente de producción en Java.
Sus métodos atómicos son más potentes que un Map protegido con synchronized:

```java
ConcurrentHashMap<String, Long> conteo = new ConcurrentHashMap<>();

// compute: actualizar atómicamente (sin race condition)
conteo.compute("clave", (k, v) -> v == null ? 1L : v + 1L);

// computeIfAbsent: crear solo si no existe (para caché)
Cache cache = mapa.computeIfAbsent(key, k -> calcularCaro(k));

// merge: combinar valor existente con nuevo
conteo.merge("clave", 1L, Long::sum);

// forEach paralelo
conteo.forEach(4, (k, v) -> procesar(k, v));  // paralelismo threshold=4
```

**Restricciones:** Implementa un word counter concurrente que procesa
un archivo de texto con 100 threads simultáneos. Sin `synchronized` extra.
El resultado debe ser exactamente correcto (mismo que la versión secuencial).

**Pista:** `compute`, `computeIfAbsent`, y `merge` son atómicos por clave
(no por todo el mapa). Dos llamadas a `compute` con distintas claves pueden
ejecutarse simultáneamente. Dos con la misma clave se serializan.
Esto es mucho más eficiente que `synchronized(map) { ... }` que bloquea todo el mapa.

**Implementar en:** Java

---

### Ejercicio 14.2.3 — CountDownLatch y CyclicBarrier: coordinación de grupos

**Enunciado:** `CountDownLatch` y `CyclicBarrier` coordinan grupos de threads:

```java
// CountDownLatch: esperar que N threads completen (one-shot)
CountDownLatch latch = new CountDownLatch(N);
for (int i = 0; i < N; i++) {
    executor.submit(() -> {
        hacerTrabajo();
        latch.countDown();  // decrementar el contador
    });
}
latch.await();  // bloquear hasta que el contador llegue a 0

// CyclicBarrier: sincronizar N threads en un punto, y repetir
CyclicBarrier barrera = new CyclicBarrier(N, () -> {
    System.out.println("todos llegaron a la barrera");  // barrierAction
});
for (int i = 0; i < N; i++) {
    executor.submit(() -> {
        for (int ronda = 0; ronda < RONDAS; ronda++) {
            hacerParteDeLaRonda(ronda);
            barrera.await();  // esperar a todos en este punto
        }
    });
}
```

**Restricciones:** Implementa una simulación de N-body con `CyclicBarrier`:
cada thread calcula la fuerza sobre sus cuerpos, la barrera sincroniza,
luego cada thread actualiza las posiciones. 10 rondas, 1000 cuerpos, 8 threads.

**Pista:** `CyclicBarrier` puede reutilizarse (de ahí "Cyclic"). `CountDownLatch` no.
El `barrierAction` corre una vez cuando todos los threads llegan — útil para
reducir resultados o preparar la siguiente ronda.
Si un thread lanza una excepción antes de llegar a la barrera,
todos los threads que esperan reciben `BrokenBarrierException`.

**Implementar en:** Java

---

### Ejercicio 14.2.4 — Semaphore: limitar acceso a recursos

**Enunciado:** `Semaphore` limita el acceso concurrente a un recurso:

```java
// Pool de conexiones a BD: máximo 10 conexiones simultáneas
Semaphore permiso = new Semaphore(10, true);  // fair=true: FIFO

void ejecutarQuery(String sql) throws InterruptedException {
    permiso.acquire();  // bloquear hasta tener permiso
    try {
        try (Connection conn = pool.getConnection()) {
            conn.execute(sql);
        }
    } finally {
        permiso.release();  // siempre liberar
    }
}

// Con timeout:
if (permiso.tryAcquire(5, TimeUnit.SECONDS)) {
    try { /* ... */ } finally { permiso.release(); }
} else {
    throw new TimeoutException("no se pudo obtener conexión en 5s");
}
```

**Restricciones:** Implementa un rate limiter basado en `Semaphore` que permite
N requests por segundo. Usando el patrón "semaphore release timer":
un thread background hace `release()` N veces por segundo.

**Pista:** El rate limiter con Semaphore es uno de los más simples de implementar.
El `fair=true` garantiza FIFO — importante para evitar starvation.
Para rate limiting de alta precisión, el token bucket (Cap.03 §3.7) es más preciso,
pero el semaphore es suficientemente bueno para la mayoría de usos.

**Implementar en:** Java

---

### Ejercicio 14.2.5 — LongAdder: contador de alta contención

**Enunciado:** `LongAdder` (Java 8) es un contador optimizado para alta contención,
usando el mismo patrón de contador particionado del Cap.12 §11.6.3:

```java
LongAdder contador = new LongAdder();

// Múltiples threads incrementan:
contador.increment();  // o add(n)

// Un thread lee el total:
long total = contador.sum();  // más lento que AtomicLong.get() pero más exacto bajo contención
```

**Restricciones:** Mide el throughput de `LongAdder.increment()` vs
`AtomicLong.incrementAndGet()` para 1, 2, 4, 8, 16, 32 threads.
`LongAdder` debe superar a `AtomicLong` para > 4 threads.
Verifica que el valor final es exacto (no approximado).

**Pista:** `LongAdder` mantiene un array de `Cell` (contadores separados).
Cada thread prefiere su propia Cell — sin contención entre threads.
`sum()` suma todas las Cells. La implementación es análoga al contador
particionado del Ejercicio 11.6.3. `LongAccumulator` es la versión generalizable
para operaciones distintas de la suma (max, min, etc.).

**Implementar en:** Java

---

## Sección 14.3 — CompletableFuture: Composición Asíncrona

`CompletableFuture` (Java 8) permite componer operaciones asíncronas
sin callbacks anidados (callback hell):

```java
// Callback hell (pre-Java 8):
buscarUsuario(id, usuario -> {
    buscarOrdenes(usuario.id(), ordenes -> {
        calcularTotal(ordenes, total -> {
            enviarEmail(usuario.email(), total, () -> {
                // 4 niveles de anidación...
            });
        });
    });
});

// CompletableFuture (Java 8+):
CompletableFuture.supplyAsync(() -> buscarUsuario(id))
    .thenCompose(u -> CompletableFuture.supplyAsync(() -> buscarOrdenes(u.id())))
    .thenApply(ordenes -> calcularTotal(ordenes))
    .thenAccept(total -> enviarEmail(usuario.email(), total))
    .exceptionally(e -> { log.error("error", e); return null; });
```

---

### Ejercicio 14.3.1 — Encadenar operaciones con thenCompose y thenApply

**Enunciado:** Reimplementa el pipeline del Cap.03 §3.2 con `CompletableFuture`:

```java
CompletableFuture<List<Integer>> pipeline(List<Integer> entrada) {
    return CompletableFuture.supplyAsync(() -> entrada)
        .thenApply(lista -> lista.stream().filter(x -> x > 0).toList())
        .thenApply(lista -> lista.stream().map(x -> x * 2).toList())
        .thenApply(lista -> lista.stream().map(x -> x + 1).toList());
}
```

**La diferencia entre thenApply y thenCompose:**
```java
// thenApply: la función retorna un valor (no un CompletableFuture)
// Análogo a map()
.thenApply(x -> transformar(x))  // x → T

// thenCompose: la función retorna un CompletableFuture
// Análogo a flatMap() — para operaciones que son asíncronas en sí mismas
.thenCompose(x -> buscarEnDB(x))  // x → CompletableFuture<T>
```

**Restricciones:** Implementa el pipeline con cada etapa corriendo en un thread pool
diferente (una etapa en `commonPool`, otra en un pool de I/O dedicado).
La composición debe ser lazy — no ejecutar hasta que se consume el resultado.

**Pista:** `thenApply` ejecuta en el thread que completó el stage anterior.
`thenApplyAsync` ejecuta en el `ForkJoinPool.commonPool()` u otro executor especificado.
Para sistemas con etapas de distintos tipos (CPU-bound vs I/O-bound),
especificar el executor en cada `Async` variant es importante para no saturar
el commonPool con I/O bloqueante.

**Implementar en:** Java

---

### Ejercicio 14.3.2 — allOf y anyOf: paralelismo y first-response

**Enunciado:** Implementa los patrones scatter/gather y first-response:

```java
// Scatter/gather: lanzar N operaciones en paralelo, esperar todas
List<String> servicios = List.of("svc1", "svc2", "svc3", "svc4");
List<CompletableFuture<Integer>> futures = servicios.stream()
    .map(svc -> CompletableFuture.supplyAsync(() -> consultar(svc)))
    .toList();

CompletableFuture<Void> todas = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0])
);
todas.thenRun(() -> {
    List<Integer> resultados = futures.stream()
        .map(CompletableFuture::join)
        .toList();
    // todos los resultados disponibles
});

// First-response: retornar el primero que responda
CompletableFuture<Object> primera = CompletableFuture.anyOf(
    futures.toArray(new CompletableFuture[0])
);
```

**Restricciones:** Implementa el sistema de recomendaciones del Ejercicio 10.7.4
con `allOf` y timeout: esperar hasta que todas las fuentes respondan O hasta
el timeout, tomando los resultados disponibles hasta ese momento.

**Pista:** `allOf` con timeout requiere `orTimeout(duration, unit)` (Java 9+):
`allOf(...).orTimeout(500, MILLISECONDS)`. Si el timeout expira,
el future completa con `TimeoutException`. Para tomar los resultados disponibles,
verificar `future.isDone()` para cada future individual después del timeout.

**Implementar en:** Java

---

### Ejercicio 14.3.3 — CompletableFuture vs Go channels: filosofías diferentes

**Enunciado:** Implementa el mismo sistema (fetch paralelo de 5 APIs con timeout)
en Java con `CompletableFuture` y en Go con goroutines + canales. Compara:

```java
// Java:
List<CompletableFuture<ApiResponse>> futures = apis.stream()
    .map(api -> CompletableFuture.supplyAsync(() -> api.fetch())
         .orTimeout(2, SECONDS)
         .exceptionally(e -> ApiResponse.error(e)))
    .toList();

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList())
    .get();
```

```go
// Go:
resultados := make(chan ApiResponse, len(apis))
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

for _, api := range apis {
    go func(a Api) {
        resultados <- a.Fetch(ctx)
    }(api)
}

var todos []ApiResponse
for range apis {
    todos = append(todos, <-resultados)
}
```

**Restricciones:** Mide y compara latencia p99, throughput, y legibilidad del código.
La implementación de Go debe usar `context.Context` para la cancelación.

**Pista:** `CompletableFuture` es más verboso pero más componible (encadenar transformaciones).
Los canales de Go son más directos para patrones simples pero requieren más
código para errores y timeouts. Para sistemas complejos con muchas transformaciones,
`CompletableFuture` puede ser más claro. Para sistemas con muchos patrones de fan-out/fan-in,
los canales de Go son más naturales.

**Implementar en:** Java + Go (comparación)

---

### Ejercicio 14.3.4 — Error handling en CompletableFuture

**Enunciado:** El manejo de errores en cadenas de CompletableFuture es más
complejo que en código sincrónico:

```java
CompletableFuture<String> resultado = buscarDato()
    .thenCompose(dato -> procesarDato(dato))
    .handle((resultado, error) -> {
        if (error != null) {
            log.warn("fallo el procesamiento", error);
            return "valor_default";
        }
        return resultado;
    })
    .thenApply(String::toUpperCase);

// Diferencia entre exceptionally y handle:
// exceptionally: solo se ejecuta si hubo error, retorna T o lanza excepción
// handle: siempre se ejecuta, recibe (T resultado, Throwable error)

// whenComplete: como handle pero no puede cambiar el resultado
.whenComplete((resultado, error) -> {
    if (error != null) log.error("error no manejado", error);
    // no puede cambiar el resultado — solo side effects
})
```

**Restricciones:** Implementa el Circuit Breaker del Cap.03 §3.6 usando
`CompletableFuture` con `handle` para contar fallos. El estado del circuit breaker
debe ser thread-safe (múltiples futures pueden fallar simultáneamente).

**Pista:** Un error no manejado en una cadena de `CompletableFuture` se propaga
silenciosamente hasta `get()` o `join()`. Si no llamas a estos métodos,
el error se "pierde". Siempre añadir un `exceptionally` o `handle` al final
de una cadena que no se consume inmediatamente con `get()`.

**Implementar en:** Java

---

### Ejercicio 14.3.5 — CompletableFuture con caching y deduplicación

**Enunciado:** Implementa un sistema de fetch con deduplicación: si múltiples
callers piden el mismo recurso simultáneamente, solo una request va al servidor:

```java
class FetchDeduplicado<K, V> {
    private final ConcurrentHashMap<K, CompletableFuture<V>> pending = new ConcurrentHashMap<>();
    private final Function<K, V> fetcher;

    CompletableFuture<V> get(K key) {
        return pending.computeIfAbsent(key, k ->
            CompletableFuture.supplyAsync(() -> fetcher.apply(k))
                .whenComplete((v, e) -> pending.remove(k))
        );
    }
}
```

**Restricciones:** El `computeIfAbsent` garantiza que solo se crea un future por clave.
Múltiples callers del mismo key comparten el mismo future.
Verificar que si el fetch falla, el error se propaga a todos los callers.
Después de completar (éxito o error), el future se elimina del mapa.

**Pista:** Este patrón es el "request coalescing" o "deduplication" — común en
sistemas de caché. La clave: `computeIfAbsent` en `ConcurrentHashMap` es atómica
por clave, garantizando que solo un future se crea aunque múltiples threads
llamen `get(key)` simultáneamente.

**Implementar en:** Java

---

## Sección 14.4 — Virtual Threads: Project Loom en la Práctica

Los Virtual Threads (Java 21) son el cambio más grande en concurrencia
en Java desde JDK 5. Permiten el modelo mental de "thread por request"
con la eficiencia del I/O asíncrono.

**El problema que resuelven:**

```java
// Con OS threads (pre-Java 21):
// 10,000 threads simultáneos ≈ 10 GB de RAM + overhead del OS scheduler
// → Por eso los frameworks async (reactive) se volvieron populares
// → Pero reactive = código complejo y difícil de debuggear

// Con Virtual Threads (Java 21):
// 10,000,000 virtual threads ≈ 10 GB de RAM (solo el estado de los Futures/stacks)
// → Puedes usar threads bloqueantes con la escala de async
// → El código es simple y síncrono
```

---

### Ejercicio 14.4.1 — Virtual Threads básico: crear y comparar

**Enunciado:** Compara la creación y overhead de OS threads vs Virtual Threads:

```java
// OS Thread (platform thread):
Thread osThread = Thread.ofPlatform().start(() -> {
    Thread.sleep(Duration.ofSeconds(1));
});

// Virtual Thread:
Thread virtualThread = Thread.ofVirtual().start(() -> {
    Thread.sleep(Duration.ofSeconds(1));  // no bloquea el carrier thread
});

// Crear 1,000,000 virtual threads (factible):
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> Thread.sleep(Duration.ofSeconds(10)));
    }
}  // espera que todos terminen (AutoCloseable)
```

**Restricciones:** Mide el tiempo y memoria para crear 1,000, 10,000, 100,000,
y 1,000,000 threads (OS vs Virtual). Mide el tiempo de la primera request HTTP
con 10,000 threads simultáneos (OS vs Virtual).

**Pista:** Los Virtual Threads se "estacionan" (unmounted del carrier) cuando hacen I/O.
El carrier thread (OS thread) queda libre para ejecutar otro Virtual Thread.
Las operaciones que bloquean el carrier (código nativo, synchronized, File I/O)
"anclan" (pin) el Virtual Thread al carrier, reduciendo la eficiencia.
Usar `ReentrantLock` en lugar de `synchronized` evita el pinning.

**Implementar en:** Java 21+

---

### Ejercicio 14.4.2 — El impacto del pinning: synchronized vs ReentrantLock

**Enunciado:** `synchronized` ancla el Virtual Thread al carrier — bloquea
el carrier mientras el Virtual Thread está en un bloque synchronized.
`ReentrantLock` no ancla — el carrier queda libre.

```java
// Con synchronized (ancla el carrier):
synchronized (lock) {
    result = database.query(sql);  // bloquea el carrier durante la query
}

// Con ReentrantLock (no ancla):
reentrantLock.lock();
try {
    result = database.query(sql);  // el carrier queda libre durante la query
} finally {
    reentrantLock.unlock();
}
```

**Restricciones:** Implementa un servidor que maneja 10,000 requests simultáneas,
cada una haciendo una query a una BD simulada que tarda 100ms.
Con `synchronized`: mide el throughput.
Con `ReentrantLock`: mide el throughput.
La diferencia debe ser > 5x.

**Pista:** La JVM en Java 21 puede detectar pinning con:
`-Djdk.tracePinnedThreads=full`. El flag imprime un stack trace cada vez
que un Virtual Thread se ancla. Es la forma de encontrar `synchronized` ocultos
en librerías de terceros que causan pinning no esperado.

**Implementar en:** Java 21+

---

### Ejercicio 14.4.3 — Migrar de reactive a virtual threads

**Enunciado:** El código reactive de Spring WebFlux puede migrar a Spring MVC
con Virtual Threads manteniendo la misma escala:

```java
// Código reactive (WebFlux):
@GetMapping("/usuario/{id}")
Mono<Usuario> getUsuario(@PathVariable String id) {
    return usuarioRepo.findById(id)              // async
        .flatMap(u -> ordenRepo.findByUserId(u.id()))  // async
        .map(ordenes -> u.withOrdenes(ordenes));
}

// Código con Virtual Threads (Spring MVC + Java 21):
@GetMapping("/usuario/{id}")
Usuario getUsuario(@PathVariable String id) {
    Usuario u = usuarioRepo.findById(id);        // bloqueante, OK con VT
    List<Orden> ordenes = ordenRepo.findByUserId(u.id());  // bloqueante, OK con VT
    return u.withOrdenes(ordenes);
}
```

**Restricciones:** Implementa ambas versiones y mide para 1000 requests/segundo
con latencia de BD de 50ms. Mide: throughput, latencia p99, líneas de código,
y número de errores introducidos al escribir el código.

**Pista:** La migración de reactive a Virtual Threads no siempre tiene sentido:
- Si ya tienes código reactive que funciona bien: no migres
- Si estás escribiendo código nuevo: Virtual Threads son más simples
- Si tienes problemas de pinning (synchronized en librerías): evalúa si reactive es mejor
La principal ventaja de Virtual Threads sobre reactive: los stack traces son legibles
(código síncrono) vs ilegibles (lambdas encadenadas de reactive).

**Implementar en:** Java 21+

---

### Ejercicio 14.4.4 — Virtual Threads y CPU-bound work: el error común

**Enunciado:** Virtual Threads NO mejoran el throughput de código CPU-bound.
Para CPU-bound, la limitación es el número de cores, no el número de threads.

```java
// Esto NO mejora con Virtual Threads:
// El CPU ya está saturado — más VTs solo causan más context switching
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            double resultado = calcularPI(100_000);  // CPU-bound
        });
    }
}

// Esto SÍ mejora con Virtual Threads:
// La espera de I/O libera el carrier — más VTs = más requests in-flight
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            String datos = httpClient.get("https://api.example.com");  // I/O-bound
        });
    }
}
```

**Restricciones:** Mide el throughput de CPU-bound work con:
- OS threads: NumCPU, 2*NumCPU, 4*NumCPU threads
- Virtual Threads: 1K, 10K, 100K threads
Demostrar que el throughput de CPU-bound no mejora más allá de NumCPU threads.

**Pista:** Para CPU-bound, el número óptimo de threads es ≈ NumCPU (±1).
Con más threads, el overhead de context switching supera el beneficio.
Para I/O-bound, el número óptimo de threads con OS threads era ~(latencia/tiempo_de_servicio)
pero con Virtual Threads no importa — el runtime maneja el stacionamiento automáticamente.

**Implementar en:** Java 21+

---

### Ejercicio 14.4.5 — ThreadLocal y ScopedValue con Virtual Threads

**Enunciado:** `ThreadLocal` funciona con Virtual Threads pero tiene un problema:
si se crean millones de VTs, cada uno tiene su propia copia del `ThreadLocal`.
`ScopedValue` (Java 21, preview) es la alternativa diseñada para VTs:

```java
// ThreadLocal — una copia por thread (funciona pero escala peor)
private static final ThreadLocal<RequestContext> CTX = new ThreadLocal<>();

// ScopedValue — immutable, no necesita remove(), mejor escala
private static final ScopedValue<RequestContext> CTX = ScopedValue.newInstance();

// Usar con ScopedValue.where():
ScopedValue.where(CTX, new RequestContext(userId, traceId))
    .run(() -> {
        // dentro del scope, CTX.get() retorna el contexto
        procesarRequest();
    });
// fuera del scope, CTX.get() es el valor anterior (o sin valor)
// sin necesidad de remove() — se limpia automáticamente al salir del scope
```

**Restricciones:** Implementa el sistema de logging del Ejercicio 14.1.5
con `ScopedValue` y mide el uso de memoria con 1M VTs activos
vs `ThreadLocal` con 1M VTs activos.

**Pista:** `ScopedValue` es immutable dentro de su scope — no tiene `set()`.
Esto elimina toda una clase de bugs (modificar el ThreadLocal desde un método
llamado que no debería hacerlo). La inmutabilidad también permite optimizaciones:
si el scope no cambia, el JIT puede inlinear el valor.

**Implementar en:** Java 21+ (ScopedValue puede requerir `--enable-preview`)

---

## Sección 14.5 — Structured Concurrency: Java 21+

`StructuredTaskScope` (Java 21, incubator) implementa el principio de que
los threads hijo no deben vivir más que el thread padre — el mismo principio
de `thread::scope` de Rust y `errgroup` de Go.

```java
// Sin Structured Concurrency:
Future<String> f1 = executor.submit(() -> buscarUsuario(id));
Future<Integer> f2 = executor.submit(() -> buscarSaldo(id));
// ¿Qué pasa si f1 falla? ¿Se cancela f2 automáticamente?
// No — hay que hacerlo manualmente

// Con StructuredTaskScope:
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<String> usuario = scope.fork(() -> buscarUsuario(id));
    Subtask<Integer> saldo = scope.fork(() -> buscarSaldo(id));
    scope.join();           // esperar que ambos terminen
    scope.throwIfFailed();  // propagar el error si alguno falló
    // Si uno falla, el scope cancela automáticamente el otro
    
    return new DatosUsuario(usuario.get(), saldo.get());
}
```

---

### Ejercicio 14.5.1 — ShutdownOnFailure vs ShutdownOnSuccess

**Enunciado:** `StructuredTaskScope` tiene dos políticas integradas:

```java
// ShutdownOnFailure: si alguna subtask falla, cancelar las demás
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // Todas las subtasks deben tener éxito
}

// ShutdownOnSuccess: terminar cuando la primera subtask tenga éxito
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    // La primera en terminar gana — las demás se cancelan
    scope.fork(() -> buscarEnServidor1());
    scope.fork(() -> buscarEnServidor2());
    scope.fork(() -> buscarEnServidor3());
    scope.join();
    String resultado = scope.result();  // el primero en terminar
}
```

**Restricciones:** Implementa el sistema de hedged requests del Ejercicio 10.4.1
usando `ShutdownOnSuccess`. Compara con la implementación de `CompletableFuture.anyOf`.

**Pista:** `ShutdownOnSuccess` es exactamente el patrón de hedged requests:
lanzar múltiples requests y usar la primera que responda.
La ventaja sobre `anyOf`: la cancelación es automática cuando el scope se cierra.
Con `anyOf`, tienes que cancelar manualmente los futures no completados.

**Implementar en:** Java 21+ (con `--enable-preview` si es necesario)

---

### Ejercicio 14.5.2 — Custom StructuredTaskScope

**Enunciado:** Implementa un scope que sigue una política personalizada:
"terminar cuando al menos K de N subtasks hayan tenido éxito":

```java
class ShutdownWhenKSucceed<T> extends StructuredTaskScope<T> {
    private final int k;
    private final List<T> resultados = new CopyOnWriteArrayList<>();

    @Override
    protected void handleComplete(Subtask<? extends T> subtask) {
        if (subtask.state() == Subtask.State.SUCCESS) {
            resultados.add(subtask.get());
            if (resultados.size() >= k) {
                shutdown();  // cancelar las demás
            }
        }
    }

    List<T> resultados() {
        ensureOwnerAndJoined();
        return Collections.unmodifiableList(resultados);
    }
}
```

**Restricciones:** Verificar que exactamente K resultados son retornados
(ni más ni menos, incluso si hay race conditions en `handleComplete`).
Implementar para el sistema de votación: N nodos, retornar cuando K respondan.

**Pista:** `shutdown()` señaliza el scope para cerrar, pero los threads activos
no se cancelan inmediatamente — deben verificar `Thread.interrupted()` o usar
operaciones interrumpibles. `handleComplete` puede llamarse concurrentemente
desde múltiples threads — usar `synchronized` o atomics para el contador.

**Implementar en:** Java 21+

---

### Ejercicio 14.5.3 — Structured Concurrency vs errgroup de Go

**Enunciado:** `errgroup` de Go y `StructuredTaskScope` de Java implementan
el mismo principio (structured concurrency) con distintas APIs:

```go
// Go errgroup:
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error { return buscarUsuario(ctx, id) })
g.Go(func() error { return buscarSaldo(ctx, id) })
if err := g.Wait(); err != nil {
    return err  // el primero en fallar cancela los demás vía ctx
}
```

```java
// Java StructuredTaskScope:
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var usuario = scope.fork(() -> buscarUsuario(id));
    var saldo = scope.fork(() -> buscarSaldo(id));
    scope.join().throwIfFailed();
}
```

**Restricciones:** Implementa el mismo sistema (fetch paralelo + fallo en una rama)
en ambos lenguajes. Mide y compara latencia, throughput, y overhead de error propagation.

**Pista:** La diferencia principal: Go usa `context.Context` para la cancelación
(explícita, pasada como parámetro). Java usa `Thread.interrupt()` (implícita,
el scope la gestiona). La filosofía de Go (cancelación explícita) hace más
claro qué cancela qué. La filosofía de Java (cancelación implícita) es más
conveniente pero más difícil de razonar.

**Implementar en:** Java 21+ + Go

---

### Ejercicio 14.5.4 — El árbol de tareas estructurado

**Enunciado:** Las tareas estructuradas forman un árbol: el scope padre no termina
hasta que todos sus hijos terminan, y los hijos no pueden vivir más que el padre.
Implementa un árbol de tareas de 3 niveles:

```
TareaPrincipal
├── Fase1 (StructuredTaskScope)
│   ├── SubtareaA
│   └── SubtareaB
└── Fase2 (StructuredTaskScope) — empieza cuando Fase1 termina
    ├── SubtareaC
    └── SubtareaD

Si SubtareaA falla:
  - SubtareaB se cancela
  - Fase1 se cancela con el error de A
  - Fase2 nunca empieza
  - TareaPrincipal recibe el error
```

**Restricciones:** El árbol debe cancelar correctamente en todos los niveles.
Verifica que no hay goroutines/threads huérfanos después de un fallo.

**Pista:** Este es el problema fundamental que structured concurrency resuelve:
sin ella, un thread hijo puede seguir corriendo después de que el padre falla,
causando comportamiento inesperado. El árbol garantiza que la cancelación
se propaga de arriba hacia abajo, y el join garantiza que el padre espera
a todos los hijos.

**Implementar en:** Java 21+

---

### Ejercicio 14.5.5 — Migrar de CompletableFuture a StructuredTaskScope

**Enunciado:** Refactoriza el Ejercicio 14.3.2 (allOf/anyOf) para usar
`StructuredTaskScope` y compara la legibilidad y la corrección:

```java
// Antes (CompletableFuture):
var futures = servicios.stream()
    .map(s -> CompletableFuture.supplyAsync(() -> s.consultar())
              .orTimeout(2, SECONDS))
    .toList();
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .get();
// manejo de errores parciales: complicado

// Después (StructuredTaskScope):
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var subtasks = servicios.stream()
        .map(s -> scope.fork(() -> s.consultar()))
        .toList();
    scope.joinUntil(Instant.now().plus(2, SECONDS));
    scope.throwIfFailed();
    var resultados = subtasks.stream().map(Subtask::get).toList();
}
// manejo de errores: automático y más claro
```

**Restricciones:** La versión con `StructuredTaskScope` debe ser funcionalmente
equivalente. Mide si hay diferencia de rendimiento (no debería ser significativa).
Documenta los casos donde `CompletableFuture` sigue siendo mejor.

**Pista:** `CompletableFuture` sigue siendo mejor para:
- Composición compleja (thenCompose, handle, etc. anidados múltiples niveles)
- Cuando el resultado de una tarea se pasa como entrada de otra (composición funcional)
- Interoperabilidad con APIs que retornan `CompletableFuture`
`StructuredTaskScope` es mejor para:
- Patrones fan-out simples (lanzar N, esperar todas)
- Cuando el error handling y la cancelación son críticos

**Implementar en:** Java 21+

---

## Sección 14.6 — Reactive Streams: Project Reactor y RxJava

Los Reactive Streams (push-based, backpressure integrado) son la alternativa
a los streams lazy de Java y al async/await antes de Virtual Threads.

```java
// Project Reactor (Spring WebFlux):
Flux<Integer> numeros = Flux.range(1, 1000)
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .take(10);

// Con I/O asíncrono:
Mono<String> resultado = webClient.get()
    .uri("/api/datos")
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(5))
    .onErrorReturn("valor_default");
```

---

### Ejercicio 14.6.1 — Flux y Mono: los tipos fundamentales de Reactor

**Enunciado:** Implementa el pipeline del Cap.03 §3.2 con `Flux`:

```java
Flux<Integer> pipeline = Flux.fromIterable(entrada)
    .filter(x -> x > 0)
    .map(x -> x * 2)
    .map(x -> x + 1)
    .publishOn(Schedulers.boundedElastic())  // cambiar al scheduler de I/O
    .flatMap(x -> Mono.fromCallable(() -> guardarEnDB(x)));
    // flatMap lanza N operaciones en paralelo (sin orden garantizado)
    // concatMap mantiene el orden pero secuencial
```

**Restricciones:** Implementar con `flatMap` (paralelo) y `concatMap` (secuencial)
y medir la diferencia de throughput. El `publishOn` cambia el scheduler para
las operaciones downstream — documentar cuándo usar `publishOn` vs `subscribeOn`.

**Pista:** `publishOn` cambia el scheduler para las operaciones DESPUÉS del operador.
`subscribeOn` cambia el scheduler para la fuente (dónde se emiten los elementos).
Para I/O-bound, usar `Schedulers.boundedElastic()`.
Para CPU-bound, usar `Schedulers.parallel()`.
El `flatMap` con `maxConcurrency` controla el paralelismo:
`.flatMap(x -> guardar(x), 8)` ejecuta hasta 8 en paralelo.

**Implementar en:** Java (con Spring Reactor o RxJava)

---

### Ejercicio 14.6.2 — Backpressure: cuando el productor es más rápido que el consumidor

**Enunciado:** Los Reactive Streams tienen backpressure integrado:
el consumidor controla cuántos elementos recibe del productor.

```java
Flux.range(1, 1_000_000)
    .onBackpressureDrop(dropped -> log.warn("dropped: {}", dropped))
    // o
    .onBackpressureBuffer(100, dropped -> log.warn("buffer lleno"))
    // o
    .onBackpressureLatest()  // solo el último, descarta intermedios
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(10);  // pedir solo 10 elementos inicialmente
        }
        @Override
        protected void hookOnNext(Integer value) {
            procesarLento(value);
            request(1);  // pedir 1 más cuando procesamos uno
        }
    });
```

**Restricciones:** Implementa un consumidor que procesa 100 elementos por segundo
de un productor que emite 10,000 por segundo. Mide cuántos elementos se pierden
con cada estrategia de backpressure.

**Pista:** `onBackpressureDrop` descarta los más nuevos (el buffer de demanda está lleno).
`onBackpressureBuffer` tiene un buffer interno — si se llena, descarta.
`onBackpressureLatest` solo guarda el más reciente — como un canal de tamaño 1 de Go.
La estrategia correcta depende de si cada elemento es crítico o si es aceptable
perder algunos.

**Implementar en:** Java (con Spring Reactor)

---

### Ejercicio 14.6.3 — Error handling en streams reactivos

**Enunciado:** Los errores en streams reactivos terminan el stream por defecto.
Las estrategias para manejarlos sin terminar el stream:

```java
Flux<String> stream = Flux.just("a", "b", "c", "d")
    .flatMap(key -> buscarEnDB(key)
        .onErrorReturn("DEFAULT")          // reemplazar error con valor
        // o
        .onErrorResume(e -> buscarEnCache(key))  // fallback a otro stream
        // o
        .retry(3)                          // reintentar 3 veces
        // o
        .retryWhen(Retry.backoff(3, Duration.ofMillis(100)))  // retry con backoff
    );
```

**Restricciones:** Implementa el Circuit Breaker del Cap.03 §3.6 como un operador
de Reactor personalizado que envuelve un `Flux` y aplica la lógica de circuit breaker.

**Pista:** Un operador personalizado de Reactor implementa `Publisher<T>` o extiende
`FluxOperator`. El circuit breaker necesita estado compartido (el contador de errores
y el estado) — usar `AtomicInteger` para el estado y `AtomicLong` para el timestamp
del último error.

**Implementar en:** Java (con Spring Reactor)

---

### Ejercicio 14.6.4 — Reactor vs Virtual Threads: ¿cuándo usar cada uno?

**Enunciado:** Con Virtual Threads, el código bloqueante puede escalar como
el código reactive. ¿Cuándo elegir cada uno?

```java
// Reactive (Reactor/WebFlux):
Mono<String> resultado = webClient.get().retrieve().bodyToMono(String.class)
    .flatMap(body -> procesarAsync(body));
// Ventaja: composition, operators, backpressure integrado

// Virtual Threads (Spring MVC + Java 21):
String resultado = restTemplate.getForObject(url, String.class);
String procesado = procesar(resultado);
// Ventaja: legibilidad, stack traces, debugging, testing
```

**Restricciones:** Implementa el mismo servicio con ambas tecnologías.
Bajo 10,000 requests/segundo con 50ms de latencia de BD, compara:
throughput, latencia p99, memoria, y tiempo de desarrollo estimado.

**Pista:** La recomendación de Spring (2024): usar Virtual Threads para código nuevo
a menos que ya tengas un código base reactive maduro o necesites los operadores
específicos de Reactor (throttle, window, buffer, merge de streams, etc.).
La mayoría de sistemas web se benefician más de la simplicidad de VTs que de
la flexibilidad de Reactor.

**Implementar en:** Java 21+

---

### Ejercicio 14.6.5 — Streaming de grandes volúmenes con Reactor

**Enunciado:** Procesa un archivo CSV de 10 GB sin cargarlo en memoria,
usando Reactor para el streaming:

```java
Flux<String> lineas = Flux.using(
    () -> Files.lines(Path.of("datos.csv")),
    Flux::fromStream,
    Stream::close
);

lineas
    .skip(1)  // encabezado
    .map(CSVParser::parsear)
    .buffer(1000)  // procesar en lotes de 1000
    .flatMap(lote -> guardarLoteEnDB(lote), 4)  // 4 lotes en paralelo
    .doOnNext(n -> log.debug("guardados {} registros", n))
    .blockLast();
```

**Restricciones:** El proceso debe usar memoria constante (sin importar el tamaño del archivo).
Con backpressure, el ritmo de lectura del archivo debe adaptarse al ritmo de escritura en BD.

**Pista:** `Flux.fromStream` con `Stream::close` garantiza que el stream del archivo
se cierra cuando el Flux completa o hay un error. El `buffer(1000)` agrupa
elementos para procesamiento en lotes, que es más eficiente para BDs.
El `flatMap(f, 4)` permite hasta 4 operaciones de DB simultáneas, evitando
saturar el pool de conexiones.

**Implementar en:** Java (con Spring Reactor)

---

## Sección 14.7 — Java vs Go vs Rust: La Comparación Completa

---

### Ejercicio 14.7.1 — El mismo servidor en tres lenguajes

**Enunciado:** Implementa un servidor de caché HTTP simple (como memcached)
en Java, Go, y Rust. Mide para 10,000 requests/segundo con 99% reads y 1% writes:
- Latencia p50, p99, p999
- Throughput máximo
- Memoria bajo carga
- Líneas de código

**Restricciones:** La implementación debe ser idiomática en cada lenguaje.
Go: goroutines + canales. Java 21: Virtual Threads. Rust: Tokio async.

**Pista:** Los tres lenguajes pueden alcanzar throughputs similares para este workload.
Las diferencias más notables: memoria (Rust < Java < Go para este caso),
latencia p999 (Rust < Go < Java por el GC), complejidad del código
(Go más simple, Rust más verboso, Java en el medio con Java 21).

**Implementar en:** Go + Java 21+ + Rust

---

### Ejercicio 14.7.2 — Interoperabilidad: Java, Go, y Rust en el mismo sistema

**Enunciado:** Diseña un sistema donde cada lenguaje hace lo que mejor hace:

```
Rust: core de procesamiento de datos (alta performance, sin GC pauses)
  → expuesto como librería nativa (.so/.dll)

Go: orquestación y distribución (microservicios, comunicación)
  → llama al core de Rust via CGo o gRPC

Java: lógica de negocio y APIs (ecosistema rico, frameworks maduros)
  → llama al servicio de Go via gRPC o HTTP
```

**Restricciones:** Implementa un pipeline completo: Java recibe la request,
llama a Go para orquestar el procesamiento, Go llama a Rust para el cómputo,
el resultado sube la cadena. El overhead del cruce de lenguajes debe medirse.

**Pista:** Los cruces de lenguaje tienen costos: gRPC introduce ~0.5-1ms de overhead
por llamada (serialización + red local). CGo introduce ~100-200ns.
Para llamadas frecuentes (>10K/s), el overhead puede dominar si la operación es pequeña.

**Implementar en:** Go + Java + Rust (con gRPC entre ellos)

---

### Ejercicio 14.7.3 — Ecosistema de observabilidad: qué tiene cada lenguaje

**Enunciado:** Implementa distributed tracing en los tres lenguajes con OpenTelemetry:

```java
// Java (Spring Boot con OTEL):
@Autowired Tracer tracer;
Span span = tracer.spanBuilder("operacion").startSpan();
try (Scope scope = span.makeCurrent()) {
    // código instrumentado
} finally {
    span.end();
}
```

```go
// Go (con OTEL SDK):
tracer := otel.Tracer("mi-servicio")
ctx, span := tracer.Start(ctx, "operacion")
defer span.End()
```

```rust
// Rust (con tracing crate + OTEL):
#[tracing::instrument]
async fn operacion(param: &str) -> Result<String> {
    // instrumentado automáticamente por el atributo
}
```

**Restricciones:** Genera traces completos end-to-end a través de los tres
lenguajes y visualízalos en Jaeger. Los traces deben incluir: latencia por etapa,
errores, y metadata custom.

**Pista:** El contexto de tracing debe propagarse entre servicios via HTTP headers
(W3C TraceContext). OpenTelemetry tiene SDKs para los tres lenguajes con
APIs compatibles. La diferencia más notable: el SDK de Java es el más maduro
y completo; el de Go es el más simple; el de Rust tiene la mejor integración
con el async runtime (tracing crate + tokio-console).

**Implementar en:** Go + Java + Rust

---

### Ejercicio 14.7.4 — Testing de concurrencia: las diferencias entre lenguajes

**Enunciado:** Implementa los mismos tests de concurrencia del Cap.07 en los
tres lenguajes y compara las herramientas disponibles:

```
Herramientas de testing:
  Go:   go test -race (TSan integrado), goleak (goroutine leaks)
  Java: jcstress (litmus tests del JMM), ThreadSanitizer (para JNI)
  Rust: -Z sanitizer=thread (solo nightly), tokio::test (async testing)
  
Testing de linearizabilidad:
  Go:   porcupine
  Java: jcstress puede verificar algunas propiedades
  Rust: no hay equivalente directo — usar pruebas de stress
```

**Restricciones:** Para el Stack lock-free implementado en los tres lenguajes,
diseña tests que verifican linearizabilidad con las herramientas disponibles.
Reporta qué bugs encontró cada herramienta.

**Pista:** Los tests de concurrencia son más difíciles en Java que en Go/Rust:
Java no tiene un equivalente built-in del race detector de Go.
ThreadSanitizer para Java existe pero requiere JNI o código nativo.
La solución práctica para Java: stress tests con alta concurrencia + jcstress.

**Implementar en:** Go + Java + Rust

---

### Ejercicio 14.7.5 — Guía de selección de lenguaje para el próximo proyecto

**Enunciado:** Basándote en los ejercicios de los capítulos 13 y 14, sintetiza
una guía de selección de lenguaje para proyectos concurrentes:

**Para cada dimensión, escala 1-5 (1=malo, 5=excelente):**

```
Dimensión                    Go    Java   Rust
─────────────────────────────────────────────
Safety de concurrencia        4      3      5
Facilidad de escribir         5      4      2
Rendimiento bruto             4      4      5
Latencia consistente (p999)   4      3      5
Ecosistema web/cloud          5      5      3
Ecosistema sistemas           3      3      5
Velocidad de desarrollo       5      4      2
Testing de concurrencia       5      4      3
Observabilidad                4      5      4
Curva de aprendizaje (bajo=fácil)  2   3   5
```

**Restricciones:** Justifica cada puntuación con una referencia a ejercicios
específicos del repo. Añade dos casos de uso donde la recomendación es ambigua
y explica cómo tomar la decisión.

**Pista:** No hay una respuesta correcta universal. Las puntuaciones deben
reflejar tu experiencia real con los ejercicios. Los casos ambiguos más comunes:
(1) servicio web con alta concurrencia donde el rendimiento importa → Go vs Java 21;
(2) sistema embebido con lógica de negocio compleja → Rust vs C++.

**Implementar en:** Documento técnico

---

## Resumen del capítulo

**La evolución de Java en concurrencia:**

| Versión | Año | Innovación |
|---|---|---|
| Java 1.0 | 1996 | synchronized, wait/notify |
| Java 5 | 2004 | java.util.concurrent (Doug Lea) |
| Java 7 | 2011 | ForkJoinPool |
| Java 8 | 2014 | CompletableFuture, streams paralelos |
| Java 9 | 2017 | reactive streams integrados |
| Java 21 | 2023 | Virtual Threads, StructuredTaskScope, ScopedValues |

**El cambio de paradigma de Java 21:**

> Con Virtual Threads, Java recupera el modelo mental simple de los threads bloqueantes
> sin sacrificar la escala. El código vuelve a ser lineal y legible.
> El costo: el ecosistema de librerías debe adaptarse para evitar el pinning
> (synchronized → ReentrantLock, ThreadLocal → ScopedValue en casos críticos).

## La pregunta que guía el Cap.15

> Los Cap.13-14 cubrieron Rust y Java.
> El Cap.15 cubre Python: el lenguaje donde el GIL hace que la concurrencia
> sea radicalmente diferente — y donde async/await tiene el ecosistema más maduro
> de los cuatro lenguajes cubiertos.
