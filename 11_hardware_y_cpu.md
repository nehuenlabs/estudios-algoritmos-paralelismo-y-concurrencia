# Guía de Ejercicios — Hardware y CPU: Diseñar para el Silicio

> Implementar cada ejercicio en: **Go · Rust · C · Java**
> según corresponda. C aparece más en este capítulo: es el lenguaje más cercano
> al hardware sin ensamblador.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el mismo código, 100x de diferencia

```c
// Versión A: acceso secuencial
for (int i = 0; i < N; i++)
    suma += a[i];

// Versión B: acceso con stride 16
for (int i = 0; i < N; i += 16)
    suma += a[i];

// Versión C: acceso aleatorio
for (int i = 0; i < N; i++)
    suma += a[indices[i]];  // indices es una permutación aleatoria
```

Para N = 100 millones de enteros (400 MB):

```
Versión A (secuencial):   85 ms   — hardware prefetch funciona perfectamente
Versión B (stride 16):    85 ms   — stride regular, prefetch aún funciona
Versión C (aleatorio):  8,500 ms  — 100x más lento. Cache miss en cada acceso.
```

El algoritmo tiene la misma complejidad O(N). La diferencia es completamente
explicada por el hardware: el acceso aleatorio genera un cache miss cada iteración,
y cada cache miss cuesta ~100 ciclos (ir a RAM).

Este capítulo enseña a entender y predecir este tipo de diferencias.

---

## Los números que importan (repetición enfática)

```
Registro:    0.3 ns     1 ciclo
L1 caché:    1 ns       3 ciclos      (32-64 KB, privada por core)
L2 caché:    4 ns       12 ciclos     (256 KB - 2 MB, privada por core)
L3 caché:    20 ns      60 ciclos     (8-64 MB, compartida entre cores)
RAM:         100 ns     300 ciclos
Disco NVMe: 100,000 ns  300,000 ciclos
```

Una línea de caché es 64 bytes — la unidad mínima de transferencia.
Cuando haces un cache miss, se cargan 64 bytes aunque solo necesitaras 8.

---

## Tabla de contenidos

- [Sección 11.1 — La jerarquía de caché: medir para creer](#sección-111--la-jerarquía-de-caché-medir-para-creer)
- [Sección 11.2 — False sharing: la penalización oculta del paralelismo](#sección-112--false-sharing-la-penalización-oculta-del-paralelismo)
- [Sección 11.3 — Memory ordering y barreras](#sección-113--memory-ordering-y-barreras)
- [Sección 11.4 — Branch prediction y su impacto](#sección-114--branch-prediction-y-su-impacto)
- [Sección 11.5 — NUMA: Non-Uniform Memory Access](#sección-115--numa-non-uniform-memory-access)
- [Sección 11.6 — Lock-free programming y CAS](#sección-116--lock-free-programming-y-cas)
- [Sección 11.7 — Profiling de hardware con perf y pprof](#sección-117--profiling-de-hardware-con-perf-y-pprof)

---

## Sección 11.1 — La Jerarquía de Caché: Medir para Creer

La mejor forma de internalizar la jerarquía de caché es medirla directamente.
El "latency ladder" es un benchmark clásico que accede a arrays de distintos tamaños
para forzar misses en distintos niveles de caché.

**El experimento:**

```
Array de 4 KB:    cabe en L1 → acceso en ~1 ns
Array de 64 KB:   cabe en L2 → acceso en ~4 ns
Array de 2 MB:    cabe en L3 → acceso en ~20 ns
Array de 512 MB:  no cabe en ninguna caché → acceso en ~100 ns
```

El acceso es aleatorio para forzar misses (acceso secuencial sería prefetcheado).

```go
func latencyLadder(tamaño int) time.Duration {
    data := make([]int64, tamaño/8)

    // Construir un "pointer chase" — cada elemento apunta al siguiente aleatoriamente
    indices := rand.Perm(len(data))
    for i := range indices {
        data[indices[i]] = int64(indices[(i+1)%len(indices)])
    }

    inicio := time.Now()
    const pasos = 1_000_000
    pos := int64(0)
    for i := 0; i < pasos; i++ {
        pos = data[pos]  // siguiente posición determinada por el dato
    }
    return time.Since(inicio) / pasos
    _ = pos  // evitar dead code elimination
}
```

El "pointer chase" es imposible de prefetchear porque la siguiente dirección
depende del dato actual — hay una dependencia de datos en el crítico path.

---

### Ejercicio 11.1.1 — Construir el latency ladder

**Enunciado:** Implementa el latency ladder para arrays de tamaño
1 KB, 2 KB, 4 KB, 8 KB, 16 KB, 32 KB, 64 KB, 128 KB, 256 KB, 512 KB,
1 MB, 2 MB, 4 MB, 8 MB, 16 MB, 64 MB, 256 MB.

Para cada tamaño, reporta la latencia media de un acceso en nanosegundos.

**Restricciones:** El benchmark debe correr con el proceso aislado (minimizar
interferencia del OS). Ejecutar cada tamaño al menos 3 veces y reportar la mediana.
Identificar los "saltos" que corresponden a los límites de L1, L2, y L3.

**Pista:** Los saltos típicos:
- L1 → L2 (32-64 KB): latencia sube de ~1ns a ~4ns
- L2 → L3 (256 KB - 2 MB): latencia sube de ~4ns a ~20ns  
- L3 → RAM (> tamaño L3): latencia sube de ~20ns a ~100ns

Los límites exactos dependen del CPU. Puedes verificarlos con:
`lscpu | grep -i cache` en Linux o `sysctl hw.l*cachesize` en macOS.

**Implementar en:** Go · C · Rust · Java

---

### Ejercicio 11.1.2 — Spatial locality: el costo del stride

**Enunciado:** Mide el throughput de suma de array para distintos strides:

```go
func sumaConStride(data []int64, stride int) int64 {
    suma := int64(0)
    for i := 0; i < len(data); i += stride {
        suma += data[i]
    }
    return suma
}
```

Mide para strides 1, 2, 4, 8, 16, 32, 64, 128 con un array de 256 MB.
Grafica el throughput (elementos/segundo) vs stride.

**Restricciones:** El array tiene 256 MB para que no quepa en ninguna caché.
Normalizar el throughput por elementos accedidos (no por tiempo total).
Identificar el stride donde el throughput cae bruscamente — eso es el tamaño de línea de caché.

**Pista:** El throughput cae cuando `stride * 8 bytes >= 64 bytes` (tamaño de línea de caché).
Con stride=1 a stride=7: cada acceso usa una cacheline cargada → throughput alto.
Con stride=8 y enteros de 8 bytes: cada acceso necesita una cacheline nueva → throughput cae.
El throughput se estabiliza después — ya no hay más cachelines por cacheline cargada.

**Implementar en:** Go · C · Rust

---

### Ejercicio 11.1.3 — Temporal locality: el costo de no reutilizar datos

**Enunciado:** La multiplicación de matrices tiene distinta localidad temporal
dependiendo del orden de los loops. Mide la diferencia entre ijk y ikj:

```go
// ijk — acceso column-major a B (baja localidad temporal de B)
for i := range C {
    for j := range C[i] {
        for k := range A[i] {
            C[i][j] += A[i][k] * B[k][j]  // B[k][j]: salta N elementos entre accesos
        }
    }
}

// ikj — acceso row-major a B (alta localidad temporal de B)
for i := range A {
    for k := range A[i] {
        for j := range B[k] {
            C[i][j] += A[i][k] * B[k][j]  // B[k][j]: secuencial
        }
    }
}
```

**Restricciones:** Mide para matrices de 64×64, 256×256, 512×512, 1024×1024.
La diferencia debe ser > 3x para matrices grandes que no caben en L3.

**Pista:** Para matrices de 64×64 (32 KB), ambas versiones caben en L2 — poca diferencia.
Para matrices de 1024×1024 (8 MB por matriz), B no cabe en L3 con ijk:
cada acceso B[k][j] salta N=1024 elementos = 8 KB = 128 cachelines distintas por fila de C.
Con ikj, B[k][j..j+N] es una fila completa de B — acceso secuencial.

**Implementar en:** Go · C · Rust · Java

---

### Ejercicio 11.1.4 — Prefetch distance: cuánto leer por adelantado

**Enunciado:** El hardware prefetch lanza lecturas especulativas al detectar
patrones de acceso. La "prefetch distance" es cuántos elementos por delante
empieza a cargar.

Implementa un benchmark que determina la prefetch distance de tu CPU:

```go
// Acceso con stride 1, pero la distancia de "lookahead" varía
// Si el prefetcher mira X elementos por delante, acceder con latency
// simulada de X+1 pasos debería ser más lento que X pasos
func testPrefetchDistance(data []int64, lookahead int) time.Duration {
    // Construir cadena de punteros con lookahead específico
    // ...
}
```

**Restricciones:** El experimento usa pointer chasing con distintos "lookahead" valores.
El objetivo es identificar el punto donde el hardware prefetcher ya no puede ayudar.

**Pista:** Los prefetchers modernos (Intel, AMD) pueden detectar strides de hasta
±2048 bytes y hacer prefetch hasta 8-12 cachelines por delante.
La "prefetch distance" típica es 4-8 cachelines (256-512 bytes).
Para strides que excedan la capacidad del prefetcher, el hardware no puede ayudar.

**Implementar en:** C · Rust (más control que Go sobre el layout de memoria)

---

### Ejercicio 11.1.5 — TLB misses: el otro tipo de miss de memoria

**Enunciado:** El TLB (Translation Lookaside Buffer) cachea traducciones
de dirección virtual a física. Con páginas de 4 KB y TLB de 64 entradas,
el TLB cubre 64 × 4 KB = 256 KB. Accesos a más de 256 KB de memoria
en patrones aleatorios pueden generar TLB misses.

Diseña un benchmark que distingue cache misses de TLB misses:

```
Cache miss puro:    acceder a 2 MB en orden secuencial con stride grande
                    → cache miss, pero el mismo TLB entry se reutiliza

TLB miss puro:      acceder a 256 MB en order que cambia de página en cada acceso
                    → TLB miss en cada acceso, pero la cacheline está en L3
```

**Restricciones:** Mide por separado la latencia de cache miss (sin TLB miss)
y de TLB miss (sin cache miss). La diferencia es la penalización del TLB miss: ~10 ciclos.

**Pista:** Para forzar TLB miss sin cache miss: usa un array muy grande (256+ MB)
de enteros de 64 bytes (una cacheline completa), con acceso secuencial (stride = 1 página = 512 elementos).
El acceso secuencial permite al hardware prefetcher cargar la cacheline antes de necesitarla,
pero cada página es nueva → TLB miss en cada acceso.

**Implementar en:** C · Rust

---

## Sección 11.2 — False Sharing: La Penalización Oculta del Paralelismo

El false sharing ocurre cuando dos threads modifican variables distintas
que comparten una línea de caché. El hardware de coherencia de caché
(MESI protocol) trata la línea completa como la unidad de sincronización —
si un thread modifica cualquier parte de la línea, todos los otros threads
con una copia de esa línea la invalidan y deben re-leerla.

**El protocolo MESI:**

```
Modified:   solo este core tiene la línea, y está modificada
Exclusive:  solo este core tiene la línea, no modificada
Shared:     múltiples cores tienen copias limpias de la línea
Invalid:    la línea está inválida, debe re-leerse
```

Cuando el Core 0 escribe en una dirección, la línea de caché en todos los
otros cores que la tienen en estado S (Shared) o M (Modified) transición a I (Invalid).
Esos cores deben hacer un "cache miss" la próxima vez que accedan a esa línea.

---

### Ejercicio 11.2.1 — Medir el false sharing con precisión

**Enunciado:** Demuestra el false sharing con un benchmark controlado:

```go
type SinPadding struct {
    a int64  // 8 bytes
    b int64  // 8 bytes
    // a y b en la misma línea de caché (16 bytes < 64 bytes)
}

type ConPadding struct {
    a   int64
    _   [56]byte  // padding para completar 64 bytes
    b   int64
    _   [56]byte
    // a y b en líneas de caché distintas
}

func benchmark(datos interface{}, cores int) time.Duration {
    // Core 0 modifica solo `a`
    // Core 1 modifica solo `b`
    // Medir tiempo total con 10M modificaciones cada uno
}
```

**Restricciones:** La diferencia entre `SinPadding` y `ConPadding` debe ser visible
con GOMAXPROCS=2. Con GOMAXPROCS=1, no debe haber diferencia significativa.

**Pista:** El false sharing solo ocurre con múltiples cores reales — no simulados.
La diferencia típica es 5-20x dependiendo del hardware.
Para aislar el efecto: la operación debe ser minimamente costosa (solo un write atómico)
para que el overhead del coherence protocol domine.

**Implementar en:** Go · C · Rust · Java

---

### Ejercicio 11.2.2 — Detectar false sharing con herramientas

**Enunciado:** Linux `perf` puede detectar false sharing con el evento
`cache-misses` y análisis de `perf c2c` (cache-to-cache transfers).

Instrumenta el benchmark del Ejercicio 11.2.1 con perf y verifica:
1. Las cache-to-cache transfers (líneas transferidas entre cores)
2. El hitm (hit in modified cacheline) — acceder a una línea que otro core modificó

```bash
# Correr con perf stat:
perf stat -e cache-misses,cache-references,L1-dcache-load-misses \
    ./benchmark_false_sharing

# Análisis detallado de false sharing:
perf c2c record ./benchmark_false_sharing
perf c2c report
```

**Restricciones:** El reporte de `perf c2c` debe mostrar las líneas de caché
con mayor contención (HITM%). Verificar que las líneas reportadas corresponden
a las variables sin padding.

**Pista:** `perf c2c` no está disponible en macOS (solo Linux). En macOS,
usa `Instruments.app` con el profiler de "Cache Misses".
En CI de GitHub Actions (Linux), `perf` sí está disponible.

**Implementar en:** C · Rust (con perf en Linux)

---

### Ejercicio 11.2.3 — False sharing en estructuras de Go runtime

**Enunciado:** El runtime de Go tiene estructuras que pueden tener false sharing.
Investiga si `sync.WaitGroup`, `sync.Mutex`, y `atomic.Int64` tienen padding
para evitar false sharing:

```go
// ¿Cuál es el tamaño de estas estructuras?
fmt.Println(unsafe.Sizeof(sync.WaitGroup{}))   // ¿64 bytes (alineada a cacheline)?
fmt.Println(unsafe.Sizeof(sync.Mutex{}))        // ¿8 bytes? ¿tiene padding implícito?
fmt.Println(unsafe.Sizeof(atomic.Int64{}))      // ¿8 bytes? ¿64 bytes?
```

**Restricciones:** Verifica empíricamente si un array de `[N]sync.Mutex`
tiene false sharing entre elementos adyacentes.
Diseña un fix usando un array de structs con padding si existe el problema.

**Pista:** `sync.Mutex` tiene tamaño 8 bytes en Go — sin padding. Si tienes
un array `[16]sync.Mutex`, los elementos están densamente empaquetados.
Los elementos 0-7 están en la misma línea de caché.
Si distintos threads toman `mu[0]` y `mu[1]`, hay false sharing.
La solución de Go estándar: nunca usar arrays de mutexes sin padding.

**Implementar en:** Go · Java (¿ConcurrentHashMap tiene padding en sus buckets?)

---

### Ejercicio 11.2.4 — True sharing vs false sharing

**Enunciado:** El true sharing (contención real en el mismo dato) y el false sharing
(contención accidental en la misma línea) tienen comportamientos distintos
bajo distintas condiciones de carga.

Implementa y mide los tres casos:

```
Caso 1: True sharing
  Thread 0 y Thread 1 incrementan el mismo contador `shared_counter`
  Todos quieren el mismo dato → contención inevitable

Caso 2: False sharing
  Thread 0 y Thread 1 incrementan contadores diferentes en la misma línea
  No quieren el mismo dato → contención evitable

Caso 3: No sharing
  Thread 0 y Thread 1 incrementan contadores en líneas distintas
  Sin contención → referencia de velocidad máxima
```

**Restricciones:** Mide para 1, 2, 4, 8 threads. El Caso 3 debe escalar
linealmente. El Caso 1 y 2 deben escalar peor — documenta cuánto peor.

**Pista:** El true sharing (Caso 1) escala sub-linealmente a cualquier escala —
la contención es fundamental. El false sharing (Caso 2) puede ser tan malo
como el true sharing para operaciones simples. La solución es diferente:
el true sharing requiere rediseño algorítmico; el false sharing solo requiere padding.

**Implementar en:** Go · C · Rust · Java

---

### Ejercicio 11.2.5 — Padding automático en structs

**Enunciado:** Implementa un analizador de structs de Go que detecta posibles
false sharing y sugiere padding:

```go
type AnalizadorFalseSharing struct{}

func (a *AnalizadorFalseSharing) Analizar(t reflect.Type) []Sugerencia {
    // Para cada campo del struct:
    // 1. Calcular su offset y tamaño
    // 2. Verificar si comparte línea de caché (64 bytes) con el campo anterior
    // 3. Si el struct se usa con múltiples goroutines, sugerir padding
}
```

**Restricciones:** El analizador trabaja en tiempo de ejecución con `reflect`.
Detectar campos adyacentes que comparten línea de caché y reportar con su offset.

**Pista:** El problema clásico: `type Contador struct { activas int32; total int64 }`.
`activas` (4 bytes) y `total` (8 bytes) están en la misma línea.
Si `activas` la modifican muchos threads y `total` lo lee un thread de métricas,
cada write a `activas` invalida la línea de `total` para el lector.
La solución: `type Contador struct { activas int32; _ [60]byte; total int64 }`.

**Implementar en:** Go · Java · Rust

---

## Sección 11.3 — Memory Ordering y Barreras

Los CPUs modernos ejecutan instrucciones fuera de orden (out-of-order execution)
y pueden reordenar accesos a memoria. Los compiladores también reordenan
para optimizar. El resultado: el orden en que el código *parece* ejecutarse
puede ser diferente al orden en que *realmente* ocurren los efectos en memoria.

**El modelo de memoria de x86 (TSO — Total Store Order):**

```
x86 garantiza:
  - Las escrituras de un mismo thread se ven en orden por otros threads
  - Pero las lecturas pueden reordenarse con escrituras
  - Y el store buffer puede hacer que una escritura no sea visible inmediatamente
```

**El modelo de memoria de ARM/POWER (más débil):**

```
ARM/POWER solo garantiza:
  - Las operaciones del mismo thread se ven en orden por ese mismo thread
  - Otros threads pueden ver las escrituras en cualquier orden
  - Requiere barreras explícitas (DMB, DSB, ISB) para garantizar orden
```

**Go's memory model:**

```
Go garantiza: si event A "happens-before" event B, entonces B ve los efectos de A.
La relación happens-before se establece con:
  - send/receive en un canal
  - sync.Mutex Lock/Unlock
  - sync/atomic operations (con el orden apropiado)
  - runtime.Gosched() y goroutine creation/join
```

---

### Ejercicio 11.3.1 — Demostrar reordenamiento de memoria

**Enunciado:** El reordenamiento es visible en hardware real (especialmente ARM).
Implementa el test de Dekker para demostrar que sin barreras, el orden
de las escrituras no está garantizado:

```go
// Test de reordenamiento (litmus test clásico)
var x, y int64

func goroutine1() {
    x = 1          // escritura a x
    r1 := y        // lectura de y
    _ = r1
}

func goroutine2() {
    y = 1          // escritura a y
    r2 := x        // lectura de x
    _ = r2
}

// ¿Puede observarse r1=0 y r2=0 simultáneamente?
// En x86: raro pero teóricamente posible (store buffer)
// En ARM: sí, con mayor frecuencia
```

**Restricciones:** Ejecutar el test 10M veces y registrar cuántas veces
se observa `r1=0 && r2=0`. Luego añadir barreras atómicas y verificar que
el comportamiento desaparece.

**Pista:** En macOS con Apple Silicon (ARM), el resultado `r1=0 && r2=0`
se puede observar regularmente sin barreras. En x86, es teóricamente posible
pero muy raro en la práctica (TSO es más fuerte).
La barrera en Go: usar `atomic.StoreInt64` y `atomic.LoadInt64` en lugar
de accesos directos — esto inserta las barreras necesarias.

**Implementar en:** Go · C (con `__sync_*` o `__atomic_*`) · Rust (`Ordering`)

---

### Ejercicio 11.3.2 — acquire-release semántica

**Enunciado:** Las operaciones atómicas tienen distintos niveles de ordering:

```
Relaxed: ningún orden garantizado, solo atomicidad
Acquire: ninguna lectura/escritura posterior puede moverse antes de esta operación
Release: ninguna lectura/escritura anterior puede moverse después de esta operación
Seq_cst: orden secuencial total — más costoso, máximas garantías
```

Implementa un flag de inicialización usando acquire-release:

```go
var (
    inicializado int32  // 0 = no inicializado, 1 = inicializado
    dato         string
)

func inicializar() {
    dato = "valor costoso de calcular"
    atomic.StoreInt32(&inicializado, 1)  // Release — las escrituras anteriores
                                         // son visibles antes de este store
}

func obtenerDato() string {
    if atomic.LoadInt32(&inicializado) == 1 {  // Acquire — las lecturas siguientes
                                                // ven las escrituras antes del store
        return dato  // seguro: el dato está completamente inicializado
    }
    return ""
}
```

**Restricciones:** Demuestra que sin acquire-release (usando accesos no-atómicos),
`obtenerDato` puede retornar `dato` parcialmente inicializado.
Con acquire-release, esto no es posible.

**Pista:** En Go, `sync/atomic` usa seq_cst por defecto (el más fuerte).
Go no expone acquire/release explícitamente — la documentación del Go memory model
(go.dev/ref/mem) explica qué operaciones son suficientes.
En Rust, `Ordering::Acquire` y `Ordering::Release` son explícitos.

**Implementar en:** Go · Rust · C++11 (`std::atomic` con `memory_order_acquire/release`)

---

### Ejercicio 11.3.3 — Double-checked locking: por qué necesita barreras

**Enunciado:** El patrón de double-checked locking (para singletons thread-safe)
es notoriamente difícil de implementar correctamente sin barreras de memoria:

```go
// INCORRECTO sin barreras — el compilador/CPU puede reordenar
var instancia *Singleton
var mu sync.Mutex

func ObtenerInstancia() *Singleton {
    if instancia == nil {              // check 1 (sin lock)
        mu.Lock()
        if instancia == nil {          // check 2 (con lock)
            instancia = new(Singleton) // asignación — puede publicarse antes de inicializar
        }
        mu.Unlock()
    }
    return instancia
}
```

El problema: `instancia = new(Singleton)` puede hacer visible el puntero
antes de que el objeto esté completamente inicializado, en hardware con weak
memory model.

La solución correcta en Go: `sync.Once`.

**Restricciones:** Implementa las tres versiones y verifica el comportamiento:
1. Double-checked locking incorrecto (sin barreras)
2. Double-checked locking con barreras (`atomic.Pointer[Singleton]`)
3. `sync.Once` (la forma idiomática)

Demuestra con un test que la versión 1 puede retornar un singleton parcialmente
inicializado (en ARM o simulando el reordenamiento con Gosched).

**Pista:** `sync.Once` internamente usa `atomic.Uint32` para el estado y `sync.Mutex`
para la exclusión. El patrón es correcto porque los atomics de Go tienen garantías
seq_cst. La versión 2 (atomic.Pointer) también es correcta pero más verbosa.
La versión 1 es la que aparece en código legado en C++98 y Java pre-1.5.

**Implementar en:** Go · Java (`volatile` es necesario) · C++ · Rust

---

### Ejercicio 11.3.4 — Implementar un spinlock con barreras correctas

**Enunciado:** Un spinlock es un mutex que espera activamente (spinning)
en lugar de bloquearse. Es más rápido que un mutex del OS para secciones
críticas muy cortas (<200 ns), pero consume CPU.

```go
type Spinlock struct {
    estado int32  // 0 = libre, 1 = ocupado
}

func (s *Spinlock) Lock() {
    for !atomic.CompareAndSwapInt32(&s.estado, 0, 1) {
        runtime.Gosched()  // ceder al scheduler durante el spin
    }
}

func (s *Spinlock) Unlock() {
    atomic.StoreInt32(&s.estado, 0)
}
```

**Restricciones:** Mide el tiempo de lock/unlock del spinlock vs sync.Mutex
para secciones críticas de 0 ns, 10 ns, 100 ns, 1 µs, 10 µs.
El spinlock debe ganar para secciones < 100 ns y perder para secciones > 1 µs.

**Pista:** El spinlock que usa `atomic.CompareAndSwapInt32` en un loop apretado
sin `Gosched` es problemático: en un sistema con N cores y N threads spinando,
ningún thread puede avanzar (el thread que tiene el lock no tiene CPU para terminar).
El `Gosched()` lo mitiga, pero la heurística correcta es: `runtime.Gosched()` después
de K intentos fallidos, no en cada intento.

**Implementar en:** Go · C (pthreads spinlock) · Rust · Java

---

### Ejercicio 11.3.5 — Hazard pointers: memory reclamation sin GC

**Enunciado:** En estructuras lock-free, liberar memoria es difícil: otro thread
puede estar usando el objeto que queremos liberar. Los hazard pointers son
una solución estándar en sistemas sin GC.

```
Idea: cada thread anuncia qué objetos está usando actualmente.
Al liberar un objeto, verificar que ningún thread lo tiene anunciado.
Si alguien lo tiene anunciado, diferir la liberación.
```

```c
// Pseudocódigo de hazard pointers
thread_local HazardPointer *hazard = NULL;

void *safe_load(void **ptr) {
    while (1) {
        void *p = atomic_load(ptr);
        hazard->ptr = p;  // anunciar que estamos usando p
        if (atomic_load(ptr) == p) return p;  // verificar que no cambió
        // si cambió, reintentar
    }
}

void retire(void *ptr) {
    retired_list.add(ptr);
    if (retired_list.size() > THRESHOLD) scan();
}

void scan() {
    // recolectar todos los hazard pointers activos
    // liberar los objetos en retired_list que no están en hazard pointers
}
```

**Restricciones:** Implementa hazard pointers para una stack lock-free en C.
Verifica con AddressSanitizer que no hay use-after-free.

**Pista:** Los hazard pointers son la alternativa a epoch-based reclamation (EBR).
EBR es más eficiente pero menos flexible. Hazard pointers son más seguros para
patrones donde los threads pueden estar "dormidos" durante una época (lo que invalida EBR).
Go no necesita esto por tener GC — el GC rastrea las referencias.

**Implementar en:** C · Rust (con `Arc` como alternativa idiomática) · C++

---

## Sección 11.4 — Branch Prediction y Su Impacto

Los CPUs modernos tienen branch predictors que adivinan el resultado de un
salto condicional antes de evaluarlo — y especulan ejecutando instrucciones
basándose en esa predicción. Si la predicción es incorrecta, se descarta
todo el trabajo especulativo (pipeline flush) — costando ~15-20 ciclos.

**Branches predecibles vs impredecibles:**

```
Predecible (siempre la misma dirección):
  for i := range a { if a[i] > 0 { suma += a[i] } }
  Si el 99% de a[i] > 0, el predictor aprende y rara vez falla.

Impredecible (dirección aleatoria):
  for i := range a { if a[i] > rng.Int() { suma += a[i] } }
  El predictor no puede aprender el patrón → fallo en cada iteración → lento.
```

**Branch-free programming:**

Para branches impredecibles en código crítico, a veces es mejor eliminar
el branch usando aritmética:

```go
// Con branch (lento si impredecible):
if a > b { max = a } else { max = b }

// Branch-free (siempre el mismo tiempo):
mask := (a - b) >> 63      // -1 (all 1s) si a < b, 0 si a >= b
max = b ^ ((a ^ b) & ^mask) // max = a si mask=0, max = b si mask=-1
```

---

### Ejercicio 11.4.1 — Medir el costo de branch misprediction

**Enunciado:** Diseña un benchmark que aísla el costo de branch misprediction:

```go
// Caso 1: array ordenado — branch predecible
data := make([]int64, N)
for i := range data { data[i] = int64(i) }
// En el loop, `if data[i] < N/2` es predecible: 50% true luego 50% false, en orden

// Caso 2: array aleatorio — branch impredecible
rand.Shuffle(len(data), func(i, j int) { data[i], data[j] = data[j], data[i] })
// Ahora `if data[i] < N/2` es completamente aleatorio → misprediction ~50% del tiempo

func sumaCondicional(data []int64, umbral int64) int64 {
    suma := int64(0)
    for _, v := range data {
        if v < umbral { suma += v }
    }
    return suma
}
```

**Restricciones:** El array debe estar en RAM (> L3 caché) para aislar el efecto
del branch predictor de los cache misses. La diferencia debe ser > 5x entre
el caso predecible y el impredecible.

**Pista:** Para aislar el branch predictor de los cache misses: accede al array
secuencialmente (el prefetcher carga los datos), pero la condición es impredecible.
El costo de cada misprediction es ~15 ciclos × (frecuencia de misprediction) × N.
Para N=10M y 50% misprediction: 10M × 0.5 × 15 ciclos / (3 GHz) = 25ms extra.

**Implementar en:** Go · C · Rust

---

### Ejercicio 11.4.2 — Branch-free max, min, y abs

**Enunciado:** Implementa versiones branch-free de max, min, y abs para enteros,
y verifica que son más rápidas para datos aleatorios:

```go
// Con branch:
func MaxBranch(a, b int64) int64 {
    if a > b { return a }
    return b
}

// Branch-free:
func MaxBranchFree(a, b int64) int64 {
    diff := a - b
    mask := diff >> 63  // arithmetic right shift: -1 si a < b, 0 si a >= b
    return a - (diff & mask)
}
```

**Restricciones:** Las versiones branch-free deben producir resultados idénticos.
Mide la diferencia de rendimiento con data aleatoria vs data ordenada.
La versión branch-free debe ser más rápida para data aleatoria y similar para data ordenada.

**Pista:** El arithmetic right shift (`>>`) firma-extiende en la mayoría de CPUs.
Para `int64`, `x >> 63` produce `0` si x ≥ 0 y `-1` (todos unos) si x < 0.
Esta técnica se usa en SIMD y en código de criptografía donde el tiempo constante
(constant-time execution) es crítico por seguridad.

**Implementar en:** Go · C · Rust · Java

---

### Ejercicio 11.4.3 — Branchless sort de redes de ordenamiento

**Enunciado:** Una "sorting network" ordena un número fijo de elementos
con una secuencia predeterminada de comparaciones e intercambios.
Es más rápida que un sort general para N pequeño porque:
1. El número de comparaciones es fijo (no depende del input)
2. Puede implementarse sin branches
3. Se vectoriza perfectamente con SIMD

Implementa una sorting network para N=4 elementos (el óptimo tiene 5 comparaciones):

```go
// Comparador branchless: intercambiar a[i] y a[j] si a[i] > a[j]
func comparar(a []int64, i, j int) {
    diff := a[i] - a[j]
    mask := diff >> 63
    intercambio := diff & mask  // 0 si a[i] <= a[j], diff si a[i] > a[j]
    a[i] -= intercambio
    a[j] += intercambio
}

// Sorting network de 4 elementos (secuencia óptima de 5 comparaciones):
func Sort4(a []int64) {
    comparar(a, 0, 1)
    comparar(a, 2, 3)
    comparar(a, 0, 2)
    comparar(a, 1, 3)
    comparar(a, 1, 2)
}
```

**Restricciones:** Verifica que la sorting network produce resultados correctos
para todos los 24 permutaciones de 4 elementos. Mide vs `sort.Slice` para N=4.

**Pista:** Las sorting networks se usan en GPUs (donde los branches son muy costosos)
y en el BLAS (Basic Linear Algebra Subprograms). Para N pequeño (≤ 16), una sorting
network bien diseñada es más rápida que cualquier sort general.

**Implementar en:** Go · C · Rust

---

### Ejercicio 11.4.4 — Profile-guided optimization (PGO)

**Enunciado:** Go 1.21 introdujo PGO (Profile-Guided Optimization).
El compilador usa datos de profiling de ejecuciones anteriores para hacer
mejores decisiones de branch prediction y inline:

```bash
# 1. Compilar sin PGO
go build -o servidor_normal .

# 2. Generar perfil de producción
go tool pprof -proto cpu.pprof  # necesitas un cpu.pprof de producción

# 3. Compilar con PGO
go build -pgo=cpu.pprof -o servidor_pgo .
```

**Restricciones:** Implementa un servidor HTTP de prueba, genera un perfil CPU
bajo carga sintética, y mide la mejora de rendimiento con PGO.
La mejora típica es 2-14% según Google.

**Pista:** Las optimizaciones que hace PGO: mejora las decisiones de inlining
(inline funciones frecuentemente llamadas aunque excedan el límite normal),
mejora el layout de código (pone el hot path al principio de las funciones),
y ajusta las heurísticas del branch predictor.

**Implementar en:** Go · C (con `-fprofile-generate` y `-fprofile-use` en GCC/Clang)

---

### Ejercicio 11.4.5 — Spectre y Meltdown: cuando el branch predictor es un problema de seguridad

**Enunciado:** Las vulnerabilidades Spectre (2018) explotan el branch predictor
para leer memoria arbitraria. La mitigación (retpoline) introduce overhead.

Implementa un benchmark que mide el overhead de las mitigaciones de Spectre:

```bash
# Con mitigaciones (default en kernels modernos):
sudo dmesg | grep "Spectre\|Meltdown\|mitigation"

# Ejecutar con y sin mitigaciones (requiere reboot con mitigaciones desactivadas):
# mitigations=off en kernel cmdline — SOLO para benchmarking, no para producción
```

**Restricciones:** Mide el overhead de syscalls (que tienen overhead de Spectre
por el kernel page table isolation) vs operaciones puramente en userspace.
El overhead típico de syscalls con KPTI es 5-30%.

**Pista:** El overhead de Spectre es más visible en workloads con muchas syscalls
(servidores con muchas conexiones, sistemas de archivos). Para código que rara vez
llama al kernel (simulaciones numéricas, machine learning), el overhead es mínimo.

**Implementar en:** C · Go (para medir overhead de syscalls)

---

## Sección 11.5 — NUMA: Non-Uniform Memory Access

En servidores multi-socket (múltiples CPUs físicas en la misma máquina),
cada CPU tiene acceso directo a su propia RAM (local) y acceso más lento
a la RAM de las otras CPUs (remoto).

**La topología NUMA:**

```
Socket 0:
  CPU 0-31 (32 cores)
  RAM local: 256 GB
  Acceso local:  ~100 ns

Socket 1:
  CPU 32-63 (32 cores)
  RAM local: 256 GB
  Acceso local:  ~100 ns
  Acceso remoto (a RAM del Socket 0): ~200 ns  ← NUMA penalty

Total: 64 cores, 512 GB RAM
```

**Por qué importa en código paralelo:**

Si lanzas 64 goroutines y todas acceden al mismo array de datos:
- Las goroutines del Socket 0 acceden a los datos en la RAM del Socket 0: ~100 ns
- Las goroutines del Socket 1 acceden a los datos en la RAM del Socket 0: ~200 ns

La mitad de los cores tiene 2x más latencia de memoria que la otra mitad.

---

### Ejercicio 11.5.1 — Detectar la topología NUMA

**Enunciado:** Implementa un programa que detecta la topología NUMA del sistema:

```go
func TopologiaNUMA() []NodoNUMA {
    // Lee /sys/devices/system/node/ en Linux
    // Retorna los nodos NUMA con sus CPUs y la cantidad de memoria
}

type NodoNUMA struct {
    ID    int
    CPUs  []int
    RAM   int64  // bytes
}
```

**Restricciones:** En máquinas con un solo socket (mayoría de desktops y laptops),
habrá un solo nodo NUMA. En servidores cloud (AWS c5.metal, por ejemplo), 2-4 nodos.
Si no hay soporte NUMA, reportar esto claramente.

**Pista:** En Linux: `/sys/devices/system/node/node0/cpulist` lista las CPUs del nodo 0.
`/sys/devices/system/node/node0/meminfo` reporta la memoria del nodo 0.
La biblioteca `numactl` proporciona esto con una API de C: `numa_available()`, `numa_num_configured_nodes()`.

**Implementar en:** Go (Linux) · C (`libnuma`) · Python (`py-cpuinfo`)

---

### Ejercicio 11.5.2 — Medir la penalización NUMA

**Enunciado:** Mide la diferencia de latencia entre acceso local y remoto NUMA:

```c
// Allocar en el nodo 0 con numactl
void *local = numa_alloc_onnode(tamaño, 0);

// Correr el benchmark desde CPUs del nodo 0 y del nodo 1:
// ¿Cuánto más lento es el acceso remoto?

// En Linux, usando taskset para fijar el proceso a CPUs específicas:
// taskset -c 0 ./benchmark_numa      ← CPU 0 (nodo 0)
// taskset -c 32 ./benchmark_numa     ← CPU 32 (nodo 1)
```

**Restricciones:** La diferencia entre acceso local y remoto debe ser medible
y reportada en nanosegundos y como factor multiplicativo.

**Pista:** En hardware real, el NUMA factor (razón entre acceso remoto y local)
es típicamente 1.5-3x. En VMs de cloud, puede ser mayor.
Si no tienes acceso a un servidor multi-socket, usa `numactl --hardware`
para ver si la máquina tiene múltiples nodos.

**Implementar en:** C (`libnuma`) · Go (con `runtime.LockOSThread()` + syscalls)

---

### Ejercicio 11.5.3 — NUMA-aware memory allocation

**Enunciado:** Para un sistema de procesamiento que corre en múltiples sockets,
implementa un allocator NUMA-aware que asigna memoria en el nodo local al worker:

```go
type AllocatorNUMA struct {
    nodos []int  // nodos NUMA disponibles
}

func (a *AllocatorNUMA) Alocar(tamaño int, cpu int) []byte {
    nodo := a.NodoDeCPU(cpu)
    // alocar en el nodo NUMA local al cpu
    return alocarEnNodo(tamaño, nodo)
}
```

**Restricciones:** Los workers fijados a CPUs del nodo 0 deben obtener memoria
del nodo 0. Mide la mejora de rendimiento vs allocación aleatoria.

**Pista:** En Go, no hay control directo sobre dónde se aloca el GC heap.
La forma práctica: usar `mmap` con `MPOL_BIND` para alocar memoria en un nodo específico.
En Go, esto requiere `syscall.Mmap` con la política NUMA establecida con `syscall.Syscall(SYS_MBIND, ...)`.

**Implementar en:** C · Go (con syscalls) · Rust

---

### Ejercicio 11.5.4 — Thread pinning para NUMA locality

**Enunciado:** Fijar goroutines a CPUs específicas (thread pinning) puede
mejorar la locality NUMA para workloads intensivos:

```go
func PinearACPU(cpu int) error {
    runtime.LockOSThread()
    var mask unix.CPUSet
    mask.Set(cpu)
    return unix.SchedSetaffinity(0, &mask)
}
```

**Restricciones:** Implementa un Worker Pool donde cada worker está fijado
a una CPU específica y aloca memoria en el nodo NUMA de esa CPU.
Mide la mejora vs el Worker Pool sin pinning para una carga memory-intensive.

**Pista:** El thread pinning tiene costos: el OS ya no puede redistribuir el trabajo.
Si un worker termina su tarea antes que los otros, no puede "robar" CPUs vacías.
El beneficio (mejor NUMA locality) vs el costo (peor balanceo) depende
de si el cuello de botella es la memoria o el compute.

**Implementar en:** Go (Linux) · C · Rust

---

### Ejercicio 11.5.5 — El costo oculto del NUMA en sistemas cloud

**Enunciado:** Las instancias de cloud (AWS, GCP) pueden tener topología NUMA
que el código de la aplicación ignora. Investiga y mide:

1. ¿Tu instancia de cloud tiene múltiples nodos NUMA?
2. ¿El runtime de Go asigna goroutines a distintos nodos NUMA?
3. ¿El GC de Go tiene en cuenta la topología NUMA?

**Restricciones:** Si no tienes acceso a cloud, usa Docker con límites de CPU:
`docker run --cpuset-cpus="0-7" --memory-policy=membind:1 ...`
para simular un escenario NUMA.

**Pista:** AWS c5.metal tiene 2 sockets NUMA. c5.xlarge (VM) puede estar en 1 socket
pero si se migra a otro host, la topología NUMA del VM puede cambiar.
El Go runtime no tiene conocimiento de NUMA — las goroutines se schedulan
en cualquier OS thread disponible, sin considerar localidad.

**Implementar en:** Go · análisis con herramientas (perf, numastat)

---

## Sección 11.6 — Lock-Free Programming y CAS

Las operaciones lock-free usan instrucciones atómicas del hardware para
sincronizar sin mutexes. La instrucción fundamental es CAS (Compare-And-Swap):

```
CAS(dirección, esperado, nuevo):
  Si *dirección == esperado:
    *dirección = nuevo
    retornar true  ← éxito
  Sino:
    retornar false ← fallo (otro thread cambió el valor)
```

**Las tres garantías de lock-free:**

```
Wait-free: cada thread completa en un número finito de pasos
  → Garantía más fuerte. Difícil de implementar.
  → Ejemplo: operaciones sobre arreglos de tamaño fijo con índice atómico

Lock-free: al menos un thread progresa en un número finito de pasos
  → Garantía intermedia. Más común.
  → Ejemplo: CAS loops donde eventualmente alguien tiene éxito

Obstruction-free: un thread aislado completa en un número finito de pasos
  → Garantía más débil. No impide starvation con contención.
```

---

### Ejercicio 11.6.1 — Stack lock-free con CAS

**Enunciado:** Implementa un stack concurrente sin mutex usando CAS:

```go
type NodoStack[T any] struct {
    valor T
    siguiente *NodoStack[T]
}

type StackLockFree[T any] struct {
    tope atomic.Pointer[NodoStack[T]]
}

func (s *StackLockFree[T]) Push(v T) {
    nuevo := &NodoStack[T]{valor: v}
    for {
        viejo := s.tope.Load()
        nuevo.siguiente = viejo
        if s.tope.CompareAndSwap(viejo, nuevo) {
            return  // éxito
        }
        // fallo: otro goroutine hizo push — reintentar
    }
}

func (s *StackLockFree[T]) Pop() (T, bool) {
    // ...
}
```

**Restricciones:** La implementación debe ser correcta bajo `go test -race`.
Mide el throughput vs sync.Mutex stack para 1, 4, 8, 16 goroutines.
El lock-free debe ganar a alta contención (8+ goroutines).

**Pista:** El ABA problem: entre `Load` y `CAS`, otro goroutine puede hacer:
Pop(A), Push(B), Push(A) — el CAS tiene éxito porque ve A en la cima,
pero la lista cambió. En Go con GC, el ABA problem en stacks es mitigado
porque los nodos liberados no se reutilizan inmediatamente.
En C sin GC, necesitas hazard pointers o tagged pointers para resolver ABA.

**Implementar en:** Go · Rust · C++

---

### Ejercicio 11.6.2 — Queue lock-free con dos punteros

**Enunciado:** La queue lock-free de Michael y Scott (1996) usa dos punteros
atómicos (head y tail) para permitir push y pop concurrentes sin que interfieran:

```go
type NodoQueue[T any] struct {
    valor T
    siguiente atomic.Pointer[NodoQueue[T]]
}

type QueueLockFree[T any] struct {
    head atomic.Pointer[NodoQueue[T]]  // para Pop
    tail atomic.Pointer[NodoQueue[T]]  // para Push
}
```

**Restricciones:** Push y Pop deben poder ejecutarse en paralelo sin bloquearse.
El algoritmo original tiene una "invariante de sentinel": siempre hay un nodo dummy
al frente de la queue.
Verifica la correctness con `go test -race` y con stress test de 1M operaciones.

**Pista:** La queue de Michael-Scott tiene un paso sutil: al hacer Push, si `tail.next != nil`,
significa que alguien hizo Push a medias — ayuda avanzar el puntero `tail` antes de insertar.
Esta "cooperative advancement" es lo que hace que el algoritmo sea lock-free
(no wait-free): si un thread falla, otro lo "ayuda" y progresa.

**Implementar en:** Go · Rust · C++ · Java

---

### Ejercicio 11.6.3 — Contador lock-free con particionado

**Enunciado:** Un contador atómico simple tiene alta contención cuando
muchos threads lo incrementan simultáneamente. El particionado reduce la contención:

```go
type ContadorParticionado struct {
    buckets [NumBuckets]struct {
        valor int64
        _     [56]byte  // padding — sin false sharing
    }
}

func (c *ContadorParticionado) Incrementar() {
    // Elegir un bucket basándose en el ID del goroutine/core
    bucket := goroutineID() % NumBuckets
    atomic.AddInt64(&c.buckets[bucket].valor, 1)
}

func (c *ContadorParticionado) Valor() int64 {
    // Sumar todos los buckets
    total := int64(0)
    for i := range c.buckets {
        total += atomic.LoadInt64(&c.buckets[i].valor)
    }
    return total
}
```

**Restricciones:** Mide la contención (tiempo por incremento) con NumBuckets = 1, 4, 8, 16, 64.
Con más buckets, los incrementos son más rápidos pero Valor() es más lento.
Encuentra el NumBuckets óptimo para 8 cores.

**Pista:** El contador particionado es el patrón "stripeded counter" — usado en
Java's `LongAdder`, Go's `expvar`, y en la implementación de `net/http` para
métricas internas. El valor ideal de NumBuckets ≈ 4 × NumCPU para minimizar
la contención en los incrementos sin hacer Valor() demasiado lento.

**Implementar en:** Go · Java (`LongAdder` vs `AtomicLong`) · Rust · C#

---

### Ejercicio 11.6.4 — Read-Copy-Update (RCU): reads sin sincronización

**Enunciado:** RCU es una técnica de sincronización donde los lectores nunca
se bloquean y los escritores copian-modifican-publican:

```
RCU para una lista enlazada:
  Lectura: simplemente recorrer la lista (sin lock)
  Escritura:
    1. Copiar el nodo a modificar
    2. Modificar la copia
    3. Atomically swap el puntero (publicar la nueva versión)
    4. Esperar que todos los lectores activos terminen (grace period)
    5. Liberar el nodo viejo
```

**Implementa una caché de configuración con RCU:**

```go
type ConfigRCU struct {
    config atomic.Pointer[Configuracion]  // los lectores solo leen este puntero
}

func (c *ConfigRCU) Leer() *Configuracion {
    return c.config.Load()  // sin lock, siempre consistente
}

func (c *ConfigRCU) Actualizar(nueva *Configuracion) {
    c.config.Store(nueva)  // atómico — los lectores ven la nueva o la vieja, nunca corrupta
}
```

**Restricciones:** Los lectores nunca se bloquean. Las actualizaciones son infrecuentes.
Mide el throughput de lectura con 1M lectores concurrentes y 1 escritor ocasional.

**Pista:** En Go, `atomic.Pointer[T]` implementa exactamente RCU para el caso simple
(un solo puntero). El GC maneja la liberación del objeto viejo automáticamente.
En el kernel de Linux, RCU es más sofisticado: el grace period requiere verificar
que todos los cores pasaron por un "quiescent state" (un context switch).

**Implementar en:** Go · C (kernel Linux usa RCU) · Rust

---

### Ejercicio 11.6.5 — Benchmark: lock-free vs mutex vs channel

**Enunciado:** Para distintos patrones de acceso, compara:
1. `sync.Mutex` + operación
2. `sync/atomic` / CAS
3. Canal de Go (comunicación por canales)
4. Operación sin sincronización (solo correcta con GOMAXPROCS=1)

**Patrones de acceso:**
- Solo escritores (alta contención de escritura)
- Solo lectores (sin contención — referencia)
- 90% lectores, 10% escritores
- 50% lectores, 50% escritores

**Restricciones:** Para cada patrón, mide el throughput en ops/ns con 1, 4, 8 cores.
La "operación sin sincronización" es la referencia del throughput máximo teórico.

**Pista:** Para 90% lectores, `sync.RWMutex` supera a `sync.Mutex`. Para operaciones
simples, `atomic` supera a `sync.Mutex`. Para estados complejos que no caben en 64 bits,
`sync.Mutex` es necesario. El canal es más lento que el mutex para datos simples
pero más idiomático en Go para comunicación de estados complejos.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 11.7 — Profiling de Hardware con perf y pprof

El profiling de hardware mide lo que el CPU realmente hace — no solo
cuánto tiempo tarda el código, sino por qué tarda ese tiempo.

**Los contadores de hardware (PMU — Performance Monitoring Unit):**

```
cycles:           ciclos de reloj del CPU
instructions:     instrucciones ejecutadas
IPC:              instructions per cycle (eficiencia del CPU)
cache-misses:     misses en el último nivel de caché (LLC)
branch-misses:    predicciones de branch incorrectas
tlb-misses:       misses en el Translation Lookaside Buffer
mem-loads:        operaciones de lectura de memoria
mem-stores:       operaciones de escritura de memoria
```

**IPC como indicador de salud del código:**

```
IPC ≈ 4.0:  excelente — el CPU ejecuta 4 instrucciones por ciclo (máximo teórico moderno)
IPC ≈ 2.0:  bueno — utilización decente del pipeline
IPC ≈ 1.0:  regular — probablemente memory-bound o muchos branches
IPC < 0.5:  malo — muchos cache misses o el código está muy serializado
```

---

### Ejercicio 11.7.1 — Profiling básico con perf stat

**Enunciado:** Usa `perf stat` para perfilar los programas de las secciones anteriores:

```bash
# IPC y cache misses:
perf stat -e cycles,instructions,cache-misses,branch-misses ./programa

# Más detalle:
perf stat -e L1-dcache-load-misses,L1-dcache-loads,\
             LLC-load-misses,LLC-loads,\
             branch-misses,branches \
    ./programa
```

Perfilar:
1. La suma secuencial del inicio del capítulo
2. La suma con acceso aleatorio
3. La multiplicación de matrices ijk vs ikj

**Restricciones:** Reporta IPC y cache miss rate para cada programa.
Verificar que las predicciones teóricas (suma aleatoria tiene más cache misses
que la suma secuencial) se confirman en los datos.

**Pista:** `perf stat` en macOS usa `Instruments.app` o `sudo powermetrics`.
En Linux, `sudo perf stat` (requiere `perf_event_paranoid <= 1`).
En Docker/CI, los contadores de hardware pueden no estar disponibles —
verifica con `perf stat echo "test"` primero.

**Implementar en:** C · Go · Rust (en Linux)

---

### Ejercicio 11.7.2 — Flame graph de CPU con pprof

**Enunciado:** Un flame graph muestra dónde pasa el tiempo el programa
en una representación visual intuitiva. Genera un flame graph para el
Worker Pool bajo carga:

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)  // endpoint de pprof
    // ... código principal
}

// Para capturar el perfil:
// go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// (go tool pprof) web  ← genera flame graph en el browser
```

**Restricciones:** El flame graph debe mostrar claramente la diferencia entre
tiempo en el código de usuario vs tiempo en el scheduler vs tiempo bloqueado en I/O.
Identifica las 3 funciones donde más tiempo se pasa.

**Pista:** En el flame graph, los "pisos" amplios son las funciones donde más tiempo
se gasta. Las funciones en el tope (sin hijos) son donde el CPU realmente está corriendo
instrucciones. Las funciones en los "valles" son las que llaman a otras pero pasan
poco tiempo en sí mismas (overhead de calling convention).

**Implementar en:** Go (pprof es nativo) · Java (async-profiler) · Python (py-spy)

---

### Ejercicio 11.7.3 — Profiling de memory allocation con pprof

**Enunciado:** El profiling de allocaciones muestra qué código genera más
presión sobre el GC:

```bash
# Capturar heap profile:
go tool pprof http://localhost:6060/debug/pprof/heap

# En el REPL de pprof:
# (pprof) top10       ← top 10 funciones por allocación
# (pprof) list NombreFuncion  ← ver las líneas específicas
# (pprof) web         ← flame graph de allocaciones
```

Identifica y elimina las allocaciones innecesarias en el Worker Pool:
- Allocaciones en el hot path (cada request)
- Allocaciones que podrían usar `sync.Pool`
- Interfaces que causan boxing (escapan al heap)

**Restricciones:** Después del fix, las allocaciones por request deben caer > 50%.
Verifica con `-benchmem` que el número de allocs/op se reduce.

**Pista:** Las causas más comunes de allocaciones en el hot path:
1. Closures que capturan variables (always escape to heap)
2. Interfaces (si el concrete type > pointer size, escapa)
3. Slices que crecen más allá de su capacidad inicial
4. `fmt.Sprintf` (siempre aloca)
`go build -gcflags="-m"` muestra las decisiones de escape analysis del compilador.

**Implementar en:** Go · Java (JVM allocation profiling) · Rust (no GC, pero sí malloc)

---

### Ejercicio 11.7.4 — Tracing de goroutines con go tool trace

**Enunciado:** `go tool trace` genera una visualización timeline de todas las
goroutines del programa: cuándo estaban corriendo, bloqueadas, o esperando scheduler.

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
    defer trace.Stop()
    // ... código a trazar
}

// Para visualizar:
// go tool trace trace.out
```

**Restricciones:** Genera un trace del pipeline del Ejercicio 3.2 (Worker Pool + Pipeline).
Identifica en el trace:
1. ¿Cuánto tiempo pasan las goroutines en cada estado (running, waiting, blocked)?
2. ¿Hay goroutines que esperan el GC frecuentemente?
3. ¿El GOMAXPROCS está siendo utilizado al máximo?

**Pista:** El trace viewer muestra los "GC assist" — cuando las goroutines ayudan al GC
en lugar de ejecutar código de usuario. Un alto % de GC assist indica que el programa
aloca demasiado y el GC no da abasto.

**Implementar en:** Go (trace es una característica única de Go)

---

### Ejercicio 11.7.5 — Continous profiling: detectar regresiones de rendimiento en producción

**Enunciado:** El profiling continuo toma muestras de CPU periódicamente
en producción y las agrega en un perfil histórico. Cuando hay una regresión,
puedes comparar el perfil antes y después del cambio.

Implementa un sistema de profiling continuo simplificado:

```go
type ProfilingContinuo struct {
    intervalo time.Duration
    duracion  time.Duration  // cuánto tiempo capturar en cada muestra
    storage   ProfileStorage
}

func (p *ProfilingContinuo) Iniciar(ctx context.Context) {
    ticker := time.NewTicker(p.intervalo)
    for {
        select {
        case <-ticker.C:
            perfil := p.capturar()
            p.storage.Guardar(perfil, time.Now())
        case <-ctx.Done():
            return
        }
    }
}
```

**Restricciones:** El overhead del profiling continuo debe ser < 1% del CPU total.
Con `pprof` a 1% de sampling rate y muestras de 5s cada 60s, el overhead es ≈ 0.08%.

**Pista:** Datadog, Pyroscope, Parca, y Google Cloud Profiler implementan continuous profiling.
El formato estándar es pprof, soportado por todos los lenguajes mayores.
La comparación de perfiles entre versiones (pprof diff) es lo que permite detectar
cuándo un PR introdujo una regresión de rendimiento.

**Implementar en:** Go · Python (py-spy) · Java (async-profiler)

---

## Resumen del capítulo

**Las cinco penalizaciones de hardware que impactan el paralelismo:**

| Penalización | Costo típico | Causa | Mitigación |
|---|---|---|---|
| L3 cache miss | ~20 ns | Acceso no secuencial | Mejorar locality, blocking |
| RAM miss | ~100 ns | Datos > tamaño de caché | Partición de datos, NUMA awareness |
| False sharing | 5-20x overhead | Variables distintas en misma cacheline | Padding de cacheline |
| Branch misprediction | ~15 ciclos | Condiciones impredecibles | Branchless code, datos ordenados |
| NUMA remote access | ~200 ns | Cross-socket memory access | Thread pinning, NUMA-aware alloc |

**Los tres niveles de optimización de hardware:**

```
Nivel 1 (sin código):   medir con perf/pprof antes de optimizar
Nivel 2 (diseño):       memory layout, access patterns, struct padding
Nivel 3 (código):       branchless, SIMD, lock-free, prefetch explícito
```

**La regla que nunca falla:**

> Medir primero. Las intuiciones sobre el rendimiento son frecuentemente incorrectas.
> El CAS "más rápido que un mutex" puede ser más lento con alta contención.
> El acceso "en orden" puede ser más lento que el aleatorio si el prefetcher
> carga datos que no usarás.
> Los números del profiler tienen la última palabra.

## La pregunta que guía el Cap.12

> El Cap.11 cubrió el hardware. El Cap.12 cierra la Parte 2 con las estructuras
> de datos lock-free: implementaciones concretas de stacks, queues, hashmaps,
> y skip lists que escalan a cientos de cores sin locks.
