# Guía de Ejercicios — Estructuras de Datos Lock-Free

> Implementar cada ejercicio en: **Go · Rust · C++ · Java**
> según corresponda. C++ aparece más aquí: la literatura clásica
> de lock-free usa C++11 atomics como referencia.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el problema que las motiva

Un `sync.Map` de Go bajo alta contención puede ser el cuello de botella
de un sistema entero. Si 64 goroutines hacen Get/Set simultáneamente,
el mutex interno serializa todo:

```
sync.Map con 64 goroutines:
  Throughput:   ~8M ops/s
  Latencia p99: ~12 µs  ← el mutex bajo contención

HashMap lock-free con 64 goroutines:
  Throughput:   ~180M ops/s  (22x más)
  Latencia p99: ~0.5 µs
```

El costo: complejidad de implementación. Las estructuras lock-free son
notoriamente difíciles de implementar correctamente. Un bug puede no
manifestarse hasta que el sistema lleva semanas en producción y recibe
la carga exacta que lo triggered.

Este capítulo construye las estructuras más importantes, con verificación
formal de correctness en cada paso.

---

## Los tres errores universales del lock-free

Antes de implementar nada, estos son los errores que aparecen siempre:

```
1. ABA Problem:
   Thread A: lee ptr = X
   Thread B: extrae X de la estructura, inserta Y, extrae Y, inserta X
   Thread A: CAS(ptr, X, nuevo) → éxito ← pero la estructura cambió!
   
   Solución: tagged pointers (versión counter en los bits altos del puntero)

2. Memory reclamation:
   Thread A: lee ptr = X (X puede ser liberado en cualquier momento)
   Thread B: libera X
   Thread A: desreferencia ptr → use-after-free
   
   Solución: hazard pointers, epoch-based reclamation, o GC

3. Starvation:
   Thread A: CAS falla una y otra vez (otro thread siempre gana)
   Thread A: espera indefinidamente
   
   Solución: backoff exponencial, o algoritmos wait-free
```

En Go, el GC resuelve el problema #2 automáticamente. El ABA problem (#1)
existe pero es menos frecuente por el GC. El starvation (#3) requiere
diseño cuidadoso.

---

## Tabla de contenidos

- [Sección 12.1 — Stack lock-free: la base](#sección-121--stack-lock-free-la-base)
- [Sección 12.2 — Queue lock-free: dos punteros independientes](#sección-122--queue-lock-free-dos-punteros-independientes)
- [Sección 12.3 — HashMap concurrente sin locks globales](#sección-123--hashmap-concurrente-sin-locks-globales)
- [Sección 12.4 — Skip list: orden sin bloqueos](#sección-124--skip-list-orden-sin-bloqueos)
- [Sección 12.5 — Ring buffer lock-free (SPSC y MPMC)](#sección-125--ring-buffer-lock-free-spsc-y-mpmc)
- [Sección 12.6 — Correctness: testing y verificación formal](#sección-126--correctness-testing-y-verificación-formal)
- [Sección 12.7 — Cuándo usar lock-free vs mutex](#sección-127--cuándo-usar-lock-free-vs-mutex)

---

## Sección 12.1 — Stack Lock-Free: La Base

El stack lock-free es el punto de entrada al mundo lock-free por su
simplicidad conceptual. Solo necesita modificar un puntero atómico
(la cima del stack).

**La invariante del stack:**

```
En todo momento:
  tope apunta al elemento más reciente
  cada elemento apunta al anterior (o nil)
  esta invariante se mantiene incluso durante operaciones concurrentes
```

**Por qué un puntero atómico es suficiente:**

```
Push:
  nuevo.siguiente = tope          ← leer el tope actual
  CAS(tope, tope_leído, nuevo)    ← instalar el nuevo tope atómicamente
  Si CAS falla: reintentar

Pop:
  actual = tope                   ← leer el tope actual
  CAS(tope, actual, actual.siguiente) ← instalar el siguiente como nuevo tope
  Si CAS falla: reintentar
  Retornar actual.valor
```

La correctness viene del CAS: si entre leer el tope y hacer el CAS,
otro goroutine modificó el tope, el CAS detecta el cambio y falla.
El loop de retry garantiza que eventualmente uno de los goroutines tiene éxito.

---

### Ejercicio 12.1.1 — Stack lock-free básico con `atomic.Pointer`

**Enunciado:** Implementa el stack lock-free usando `atomic.Pointer[Nodo]` de Go 1.19+:

```go
type Nodo[T any] struct {
    valor     T
    siguiente *Nodo[T]
}

type Stack[T any] struct {
    tope atomic.Pointer[Nodo[T]]
}

func (s *Stack[T]) Push(v T)
func (s *Stack[T]) Pop() (T, bool)
func (s *Stack[T]) Peek() (T, bool)  // leer sin extraer
func (s *Stack[T]) Len() int         // estimado, no exacto bajo contención
```

**Restricciones:** El `Len()` puede ser eventualmente consistente (no necesita ser
exacto en presencia de modificaciones concurrentes). Las otras operaciones deben
ser linearizables: cada operación parece ocurrir instantáneamente.
Verificar con `go test -race` y con 1M operaciones concurrentes.

**Pista:** `atomic.Pointer[T]` en Go 1.19+ usa `sync/atomic.Pointer` que bajo el capó
es un `unsafe.Pointer` con todas las garantías del modelo de memoria de Go.
El problema del `Len()` exacto: requeriría un contador atómico adicional,
pero el contador + el stack no se pueden actualizar atómicamente juntos → el contador
puede estar out-of-sync. Documenta este tradeoff.

**Implementar en:** Go · Rust (`Arc<Mutex<...>>` vs `crossbeam::stack`) · Java · C++

---

### Ejercicio 12.1.2 — El ABA problem demostrado

**Enunciado:** Demuestra el ABA problem en el stack lock-free. El escenario:

```
Estado inicial: [A] → [B] → nil

Goroutine 1:
  1. Lee tope = A
  2. Pausa (simular con sleep/Gosched)

Goroutine 2:
  3. Pop A  → stack: [B] → nil
  4. Pop B  → stack: nil
  5. Push A → stack: [A] → nil  ← A está de vuelta, pero A.siguiente ahora es nil

Goroutine 1 (reanuda):
  6. CAS(tope, A, A.siguiente)  ← éxito! (tope sigue siendo A)
     Pero A.siguiente era B (antes) y ahora es nil
     → Perdimos el elemento B del stack
```

**Restricciones:** Demostrar el bug con código que produce resultados incorrectos.
Luego implementar la solución con **tagged pointers**: el puntero almacena
no solo la dirección sino también un contador de versión en los bits altos.

```go
// Tagged pointer: 48 bits de dirección + 16 bits de versión
type TaggedPtr[T any] struct {
    ptr atomic.Uint64  // top 16 bits: versión; bottom 48 bits: puntero
}
```

**Pista:** En sistemas de 64 bits, los punteros reales solo usan 48 bits
(los 16 bits altos siempre son 0 para adreses en el espacio de usuario).
Podemos usar esos 16 bits para el contador de versión.
En Go, hacer esto requiere `unsafe.Pointer` — documentar el riesgo.
Alternativa más segura (y la usada en Go real): el GC previene el reuso
de dirección antes de que todas las referencias sean descartadas.

**Implementar en:** Go · C++ · Rust

---

### Ejercicio 12.1.3 — Stack wait-free con array

**Enunciado:** El stack CAS es lock-free (al menos un goroutine progresa) pero
no wait-free (un goroutine individual puede esperar indefinidamente si siempre pierde el CAS).

Implementa un stack wait-free usando un array preallocado:

```go
type StackWaitFree[T any] struct {
    data    []T
    top     atomic.Int64  // índice del próximo slot libre
    capacidad int
}

func (s *StackWaitFree[T]) Push(v T) bool {
    idx := s.top.Add(1) - 1  // reservar un slot atómicamente
    if idx >= int64(s.capacidad) {
        s.top.Add(-1)  // revertir
        return false   // stack lleno
    }
    s.data[idx] = v  // escribir en el slot reservado
    return true
}
```

**Restricciones:** El Push debe ser wait-free — nunca hay retry.
El Pop es más complicado (requiere CAS en el índice). Implementarlo correctamente.
La capacidad máxima es fija — documenta esta limitación vs el stack dinámico.

**Pista:** El Push wait-free es sencillo: `fetch-and-add` en el índice.
El Pop es más sutil: múltiples goroutines pueden intentar hacer Pop al mismo tiempo,
y solo uno puede obtener el último elemento. Un CAS sobre el índice es suficiente
para el Pop, pero si el CAS falla, el goroutine reintenta — eso hace el Pop lock-free,
no wait-free. Un Pop verdaderamente wait-free requiere un algoritmo más complejo.

**Implementar en:** Go · Java (`AtomicInteger`) · Rust · C++

---

### Ejercicio 12.1.4 — Backoff exponencial bajo alta contención

**Enunciado:** Con muchos goroutines haciendo Push/Pop simultáneamente,
el loop de retry del CAS puede generar "contention storms" — muchos goroutines
reintentando simultáneamente, cada uno invalidando el trabajo de los otros.

El backoff exponencial reduce la contención:

```go
type BackoffConfig struct {
    minDelay time.Duration  // ej: 1 µs
    maxDelay time.Duration  // ej: 1 ms
    factor   float64        // ej: 2.0
}

func (s *Stack[T]) PushConBackoff(v T, cfg BackoffConfig) {
    delay := cfg.minDelay
    for {
        nuevo := &Nodo[T]{valor: v}
        viejo := s.tope.Load()
        nuevo.siguiente = viejo
        if s.tope.CompareAndSwap(viejo, nuevo) {
            return
        }
        // CAS falló — esperar antes de reintentar
        time.Sleep(delay)
        delay = min(time.Duration(float64(delay)*cfg.factor), cfg.maxDelay)
    }
}
```

**Restricciones:** Compara el throughput con y sin backoff para 1, 4, 8, 16, 32 goroutines.
El backoff debe mejorar el throughput a alta contención y no degradar a baja contención.
El factor óptimo y el delay inicial deben determinarse experimentalmente.

**Pista:** El backoff exponencial con jitter (añadir aleatoriedad al delay)
es aún mejor que el determinista — evita que todos los goroutines se reinicien
al mismo tiempo después del backoff. `delay += rand.Int63n(int64(delay))` añade jitter.
Esta técnica es la base del exponential backoff with jitter en sistemas distribuidos.

**Implementar en:** Go · Java · Rust · C++

---

### Ejercicio 12.1.5 — Eliminación: el truco para mayor throughput

**Enunciado:** El **elimination-backoff stack** usa un "área de eliminación"
donde un pusher y un popper pueden intercambiar valores directamente,
evitando el cuello de botella del tope del stack:

```
Si hay muchos pushers y poppers simultáneamente:
  En lugar de todos contender por el tope,
  un pusher y un popper se "encuentran" en el área de eliminación
  y se cancelan mutuamente → ambos completan sin tocar el stack principal.
```

```go
type SlotEliminacion[T any] struct {
    estado atomic.Int32  // 0=vacío, 1=esperando push, 2=esperando pop
    valor  T
}

type StackEliminacion[T any] struct {
    stack      *Stack[T]
    ranura     []SlotEliminacion[T]  // N slots para eliminación
    timeout    time.Duration
}
```

**Restricciones:** Bajo alta contención (push y pop simultáneos), el throughput
del elimination stack debe ser > 2x el del stack CAS simple.
Bajo baja contención (solo push o solo pop), debe comportarse igual que el stack CAS.

**Pista:** El área de eliminación tiene K slots. Un pusher elige un slot aleatorio:
si está vacío, lo marca como "esperando push" y espera un tiempo corto (timeout).
Un popper también elige un slot aleatorio: si hay un pusher esperando, intercambian
directamente. Si no hay encuentro antes del timeout, ambos van al stack principal.
Este algoritmo es del paper de Hendler et al. (2004) — uno de los más citados en lock-free.

**Implementar en:** Go · C++ · Rust

---

## Sección 12.2 — Queue Lock-Free: Dos Punteros Independientes

La queue lock-free de Michael y Scott (1996) es el algoritmo más influyente
del campo. Permite push y pop completamente independientes gracias a que
head y tail son punteros atómicos separados.

**La invariante de la queue:**

```
Siempre hay al menos un nodo (el sentinel/dummy):
  head apunta al sentinel (el elemento que fue extraído más recientemente)
  tail apunta al último nodo encolado (o al sentinel si está vacía)
  Todos los nodos entre head y tail contienen valores válidos

  head → [sentinel] → [val1] → [val2] → [val3] = tail
```

**Por qué el sentinel simplifica el algoritmo:**

Sin sentinel, el caso de una queue de 1 elemento requiere modificar tanto
head como tail en el mismo Pop — no se puede hacer con un solo CAS.
Con sentinel, el Pop solo modifica head y el Push solo modifica tail —
en la mayoría de los casos.

---

### Ejercicio 12.2.1 — Queue de Michael-Scott

**Enunciado:** Implementa la queue de Michael-Scott exactamente como en el paper original:

```go
type NodoQueue[T any] struct {
    valor     T
    siguiente atomic.Pointer[NodoQueue[T]]
}

type Queue[T any] struct {
    head atomic.Pointer[NodoQueue[T]]  // siempre apunta al sentinel
    tail atomic.Pointer[NodoQueue[T]]  // siempre apunta al último o al sentinel
}

func NuevaQueue[T any]() *Queue[T] {
    sentinel := &NodoQueue[T]{}
    q := &Queue[T]{}
    q.head.Store(sentinel)
    q.tail.Store(sentinel)
    return q
}

func (q *Queue[T]) Enqueue(v T)
func (q *Queue[T]) Dequeue() (T, bool)
```

**Restricciones:** El algoritmo debe manejar correctamente el "tail lag":
cuando tail no apunta al último nodo (porque otro goroutine hizo enqueue a medias).
En ese caso, el goroutine que detecta el lag debe ayudar a avanzar tail.
Verificar con 1M operaciones concurrentes y `go test -race`.

**Pista:** El "tail lag" ocurre cuando el enqueue está a medias: el nuevo nodo
fue añadido a la lista, pero tail aún no fue actualizado.
Si otro goroutine detecta que `tail.next != nil`, debe intentar `CAS(tail, tail, tail.next)`
para avanzar tail. Luego reintenta su propia operación. Esta "cooperative advancement"
es la clave de la correctness del algoritmo.

**Implementar en:** Go · Java · Rust · C++

---

### Ejercicio 12.2.2 — Queue con batching para mayor throughput

**Enunciado:** La queue de Michael-Scott tiene alta latencia para throughput
de un solo thread porque cada enqueue/dequeue hace un CAS.
El batching reduce el número de CAS intercalando múltiples items:

```go
type QueueBatch[T any] struct {
    queue    *Queue[T]
    buffer   []T
    mu       sync.Mutex  // solo para el buffer local
    flushAt  int         // tamaño del batch
}

func (q *QueueBatch[T]) Enqueue(v T) {
    q.mu.Lock()
    q.buffer = append(q.buffer, v)
    if len(q.buffer) >= q.flushAt {
        q.flush()
    }
    q.mu.Unlock()
}

func (q *QueueBatch[T]) flush() {
    // Enqueue todo el buffer en la queue lock-free de una vez
    // Usando un nodo de lista enlazada que contiene todos los items del batch
}
```

**Restricciones:** El flush debe ser atómico — no puede ser interrumpido a mitad.
Implementa un nodo de batch que contiene un array de valores (no un nodo por valor).
El throughput con batchSize=32 debe ser > 3x el de la queue item-by-item.

**Pista:** El batching introduce una tradeoff: mayor throughput vs mayor latencia.
Con batchSize=32, el primer item del batch espera hasta que el batch esté lleno.
Esto introduce una latencia mínima de `flushAt × tiempo_por_enqueue`.
Para sistemas de streaming donde el throughput importa más que la latencia, el batching vale la pena.

**Implementar en:** Go · Java · Python (`asyncio.Queue` con batching) · Rust

---

### Ejercicio 12.2.3 — MPSC Queue: múltiples productores, un consumidor

**Enunciado:** Cuando hay exactamente un consumidor, la queue se simplifica
significativamente. El consumidor puede leer head sin CAS porque es el único
que modifica head:

```go
type QueueMPSC[T any] struct {
    head *NodoQueue[T]             // solo lo modifica el consumidor
    tail atomic.Pointer[NodoQueue[T]]  // lo modifican los productores
}

func (q *QueueMPSC[T]) Enqueue(v T) {
    // Varios productores pueden llamar esto simultáneamente
    // El tail se actualiza con CAS
}

func (q *QueueMPSC[T]) Dequeue() (T, bool) {
    // Solo el consumidor llama esto
    // head se mueve sin CAS (no hay contención)
}
```

**Restricciones:** Verificar que la implementación es incorrecta si múltiples
consumidores la usan (mostrar el bug). Con un solo consumidor, debe ser
correcta bajo cualquier número de productores.
El throughput del consumidor debe ser mayor que con la queue MPMC.

**Pista:** La MPSC queue es más eficiente porque el Dequeue no necesita CAS.
El consumidor simplemente mueve head.next al frente — ningún otro thread
toca head. La queue de Tokio (Rust async runtime) usa exactamente esta
estructura para su task scheduler: los workers son productores,
el scheduler es el consumidor.

**Implementar en:** Go · Rust · C++ · Java

---

### Ejercicio 12.2.4 — Queue con prioridad lock-free

**Enunciado:** Una priority queue lock-free combina la queue de M&S con un
heap o skip list para mantener el orden. Es notoriamente difícil:

```
Problema: extract-min requiere modificar el elemento raíz del heap,
que es accedido por todos los threads → alta contención.

Solución: "relaxed" priority queue — aproximadamente ordenada,
no garantiza el mínimo exacto pero garantiza que en K extracciones,
uno de los K elementos más pequeños es retornado.
```

Implementa una priority queue "relajada" con N colas separadas,
eligiendo aleatoriamente entre las K colas con menor prioridad:

```go
type PriorityQueueRelajada[T any] struct {
    colas    [NumColas]*Queue[T]      // una cola por nivel de prioridad
    conteos  [NumColas]atomic.Int64   // cuántos elementos por cola
}
```

**Restricciones:** Con K=4 (elegir aleatoriamente entre las 4 colas más llenas),
el throughput debe ser > 5x el de una priority queue con mutex.
La "relajación" garantiza que el elemento extraído es uno de los primeros 4K.

**Pista:** La priority queue relajada es el approach de la mayoría de sistemas
de alta frecuencia: prefieren throughput a ordering exacto.
Si el sistema puede tolerar que un elemento de prioridad 5 sea procesado
antes que uno de prioridad 4 en algunas situaciones, la relajación
da un throughput enormemente mayor.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 12.2.5 — Medir: ¿cuándo la queue lock-free vale la pena?

**Enunciado:** La queue lock-free es más compleja que un `chan T` de Go.
¿Cuándo vale la pena la complejidad?

Mide y compara para estos escenarios:
1. 1 productor, 1 consumidor (SPSC)
2. 4 productores, 4 consumidores (MPMC balanceado)
3. 64 productores, 1 consumidor (MPSC extremo)
4. Items grandes (1 KB por item)
5. Items pequeños (8 bytes por item)

Para cada escenario, compara:
- `chan T` buffered
- `chan T` unbuffered
- Queue de Michael-Scott
- `sync.Mutex` + `[]T` slice

**Restricciones:** El benchmarks debe reportar throughput (millones de ops/segundo)
y latencia p99 para cada combinación escenario × estructura.

**Pista:** El canal de Go tiene overhead por su implementación general (soporta select,
close, ranging). Para SPSC puro, el ring buffer lock-free (Sección 12.5) suele ganar.
Para MPSC, la queue M&S gana sobre el mutex slice a > 8 goroutines.
Para SPSC con items grandes, la diferencia entre todas las opciones es pequeña
(la copia del item domina el tiempo, no la sincronización).

**Implementar en:** Go · Java · Rust · C#

---

## Sección 12.3 — HashMap Concurrente sin Locks Globales

Un HashMap lock-free completo es muy complejo. La aproximación práctica:
lock striping — dividir el hashmap en N segmentos, cada uno con su propio lock.

**La jerarquía de concurrencia del hashmap:**

```
Nivel 1: Mutex global
  Un lock para todo el mapa → máxima contención, mínima complejidad

Nivel 2: Lock striping (N locks)
  N locks, cada uno cubre 1/N del espacio de claves
  Contención reducida N veces con costo de N locks en memoria

Nivel 3: Lock-free (sin locks)
  CAS por operación
  Máxima concurrencia, máxima complejidad
  Solo vale la pena para los hot paths más críticos

Go's sync.Map es algo diferente: optimizado para el caso read-heavy
(lecturas sin lock, escrituras con lock + copy-on-write)
```

---

### Ejercicio 12.3.1 — HashMap con lock striping

**Enunciado:** Implementa un HashMap concurrente con N segmentos:

```go
const NumSegmentos = 64

type HashMapStriped[K comparable, V any] struct {
    segmentos [NumSegmentos]struct {
        mu     sync.RWMutex
        datos  map[K]V
        _      [40]byte  // padding anti-false-sharing
    }
}

func (h *HashMapStriped[K, V]) Get(k K) (V, bool)
func (h *HashMapStriped[K, V]) Set(k K, v V)
func (h *HashMapStriped[K, V]) Delete(k K)
func (h *HashMapStriped[K, V]) Len() int  // O(N) — recorre todos los segmentos
```

**Restricciones:** `NumSegmentos` debe ser configurable. Mide el throughput para
NumSegmentos = 1, 4, 16, 64, 256 con 8 goroutines de lectura y 2 de escritura.
El óptimo para 10 goroutines es típicamente `NumSegmentos ≈ 4 × NumGoroutines`.

**Pista:** El segmento se determina por `hash(k) % NumSegmentos`. Para claves de tipo
comparable genérico, necesitas una función hash. En Go: convertir a string y usar FNV,
o usar `fmt.Sprintf("%v", k)` (más lento). Para tipos específicos, la función hash
puede ser más eficiente.

**Implementar en:** Go · Java (`ConcurrentHashMap` ya lo hace) · Rust · C#

---

### Ejercicio 12.3.2 — sync.Map de Go: cuándo usarlo

**Enunciado:** `sync.Map` está optimizado para dos casos específicos:
1. El mapa se escribe una vez y se lee muchas veces (read-heavy)
2. Múltiples goroutines leen/escriben claves disjuntas (sin solapamiento)

Para otros casos, puede ser más lento que `sync.Mutex` + `map[K]V`.

Diseña y ejecuta benchmarks que demuestran:
- Cuándo `sync.Map` supera al map protegido con mutex
- Cuándo el map con mutex es mejor
- Cuándo el HashMap striped del Ejercicio 12.3.1 es mejor que ambos

**Restricciones:** Los benchmarks deben cubrir los cuatro cuadrantes:
- Read-heavy (90% reads) × claves concentradas
- Read-heavy (90% reads) × claves dispersas
- Write-heavy (50% writes) × claves concentradas
- Write-heavy (50% writes) × claves dispersas

**Pista:** `sync.Map` tiene un "dirty map" que se promueve al "read map" cuando
el ratio de misses es suficientemente alto. Este mecanismo amortiza las escrituras
pero introduce latencia en las primeras lecturas después de una escritura.
Para writes frecuentes de la misma clave, sync.Map tiene mayor overhead que mutex+map.

**Implementar en:** Go · Java (comparar con `ConcurrentHashMap`) · C# (`ConcurrentDictionary`)

---

### Ejercicio 12.3.3 — HashMap con resize lock-free

**Enunciado:** El resize (rehashing) es el caso más difícil del HashMap concurrente.
Durante el resize, el mapa existe en dos versiones simultáneamente.
Implementa el resize sin bloquear todas las operaciones:

```go
type HashMap[K comparable, V any] struct {
    tabla   atomic.Pointer[Tabla[K, V]]
}

type Tabla[K comparable, V any] struct {
    buckets   []atomic.Pointer[Lista[K, V]]
    capacidad int
    siguiente *Tabla[K, V]  // nueva tabla si estamos en medio de resize
    migrados  atomic.Int64  // cuántos buckets migrados a la nueva tabla
}

func (h *HashMap[K, V]) Get(k K) (V, bool) {
    // Buscar en la tabla actual; si no está y hay resize en curso,
    // buscar también en la nueva tabla
}
```

**Restricciones:** Durante el resize, Get y Set deben funcionar correctamente.
El resize no bloquea operaciones — puede ralentizarlas, pero no indefinidamente.
El resize es incremental: migrar un bucket por vez, no todos a la vez.

**Pista:** El resize incremental ("split migration") es cómo Redis hace su rehashing.
Redis mantiene dos tablas y en cada operación, migra un bucket de la tabla vieja
a la nueva. Después de migraciones suficientes, la tabla vieja se descarta.
Las lecturas deben verificar ambas tablas durante la migración.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 12.3.4 — Consistent hashing map: los datos siguen al hash

**Enunciado:** Combina el consistent hashing del Cap.09 con el HashMap concurrente
para un mapa que puede expandirse añadiendo buckets sin rebalanceo completo:

```go
type ConsistentHashMap[K comparable, V any] struct {
    anillo  *AnilloHash         // del Cap.09
    shards  map[uint32]*ShardMap[K, V]  // shard por posición en el anillo
}

type ShardMap[K comparable, V any] struct {
    mu    sync.RWMutex
    datos map[K]V
}
```

**Restricciones:** Añadir un nuevo shard solo mueve ~1/N de las claves.
Las operaciones Get/Set durante la migración deben ser correctas (sin perder datos).

**Pista:** La migración al añadir un shard: las claves que ahora corresponden al nuevo shard
(según el consistent hashing) deben moverse. Durante la migración, las lecturas
pueden ir a cualquier shard — el shard viejo (si la clave aún no se migró)
o el shard nuevo (si ya se migró). Sin duplicados, sin pérdidas.

**Implementar en:** Go · Java · Python

---

### Ejercicio 12.3.5 — Cache con eviction concurrente (LRU lock-free)

**Enunciado:** Un LRU cache tiene dos operaciones: Get (que mueve el elemento
al frente) y Set (que puede evictar el elemento más antiguo).
Hacer ambas correctas y concurrentes es el problema que resuelve `sync.Map`
combinado con una estructura LRU.

Implementa un LRU cache que soporta:
- Gets concurrentes sin bloquear Sets
- Sets que evictan el elemento menos reciente
- Configuración del tamaño máximo

```go
type LRUConcurrente[K comparable, V any] struct {
    capacidad int
    mapa      *HashMapStriped[K, *EntradaLRU[K, V]]
    lista     *ListaDoble[K, V]   // para el orden LRU
    mu        sync.Mutex           // solo para la lista de evicción
}
```

**Restricciones:** Get debe ser O(1) y mayoritariamente sin locks (usando el hashmap striped).
El lock en mu solo se toma para actualizar el orden LRU y para evictar.
Bajo 90% reads y 10% writes, el throughput debe ser > 10M ops/s con 8 goroutines.

**Pista:** El LRU concurrente "perfecto" (sin locks para Get) requiere una lista
doblemente enlazada con nodos que se pueden mover al frente atómicamente —
muy difícil de implementar correctamente. La solución práctica: hacer el "promote to front"
en un goroutine de background que los Gets alimentan con un canal. Los Gets leen
sin lock, y el reordenamiento ocurre eventualmente (LRU aproximado).
Esto es lo que hace `groupcache` de Brad Fitzpatrick.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 12.4 — Skip List: Orden sin Bloqueos

La skip list es una alternativa lock-free al árbol balanceado (B-tree, AVX-tree).
Permite búsqueda, inserción, y eliminación en O(log N) con alta concurrencia.

**La estructura:**

```
Nivel 3: [−∞] ─────────────────────────────── [7] ─── [+∞]
Nivel 2: [−∞] ──────────── [4] ──────────────[7] ─── [+∞]
Nivel 1: [−∞] ───── [2] ── [4] ───── [6] ─── [7] ─── [+∞]
Nivel 0: [−∞] [1] ─ [2] ─ [3] [4] ─ [5][6] ─[7] [8] [+∞]
```

Cada nodo tiene múltiples "next" pointers (uno por nivel).
El nivel de cada nodo se determina aleatoriamente al insertar
(nivel k con probabilidad `p^k`, típicamente `p=0.5`).

**Por qué es más fácil de hacer lock-free que un árbol:**

Un árbol balanceado requiere rotaciones al insertar/eliminar — operaciones
que modifican múltiples nodos simultáneamente. La skip list solo modifica
los punteros del nodo nuevo y sus predecesores en cada nivel — operaciones
que se pueden hacer con CAS independientes.

---

### Ejercicio 12.4.1 — Skip list básica (con locks por nivel)

**Enunciado:** Antes de la versión lock-free, implementa una skip list
con locks por nivel (más simple, para entender la estructura):

```go
type NodoSkip[K comparable, V any] struct {
    clave    K
    valor    V
    next     []*NodoSkip[K, V]  // puntero al siguiente en cada nivel
    nivel    int
}

type SkipList[K comparable, V any] struct {
    cabeza   *NodoSkip[K, V]
    comparar func(K, K) int
    maxNivel int
    nivel    int          // nivel actual del skip list
    mu       []sync.RWMutex  // un lock por nivel
}

func (s *SkipList[K, V]) Buscar(k K) (V, bool)
func (s *SkipList[K, V]) Insertar(k K, v V)
func (s *SkipList[K, V]) Eliminar(k K) bool
func (s *SkipList[K, V]) Rango(min, max K) []V  // todos los valores entre min y max
```

**Restricciones:** La operación Rango debe ser eficiente — usar los niveles
superiores para saltar hasta `min` y luego iterar el nivel 0 hasta `max`.
El nivel de cada nodo nuevo se determina con `rand.Float32() < 0.5` en un loop.

**Pista:** La skip list con locks por nivel ya es más eficiente que un árbol
con un lock global: múltiples inserciones en distintas partes de la lista
pueden ocurrir en paralelo si no solapan los mismos niveles.
La versión con locks es el punto de partida — la lock-free viene después.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 12.4.2 — Skip list lock-free con marcado lógico

**Enunciado:** La skip list lock-free de Herlihy, Lev, Luchangco y Shavit (2006)
usa "logical deletion" — marcar un nodo como eliminado antes de desenlazarlo:

```go
type NodoSkipLF[K comparable, V any] struct {
    clave    K
    valor    V
    next     []atomic.Pointer[NodoSkipLF[K, V]]
    marcado  atomic.Bool  // true si está lógicamente eliminado
    nivel    int
}
```

**Algoritmo de eliminación:**
1. Marcar el nodo como eliminado (atómicamente, puede fallar → reintentar)
2. Desenlazarlo de cada nivel con CAS (de abajo hacia arriba)
3. Si otro goroutine encuentra un nodo marcado durante una búsqueda,
   lo desenlaza cooperativamente

**Restricciones:** Las búsquedas deben ignorar los nodos marcados.
Las inserciones deben evitar insertar después de un nodo marcado (race condition).
Verificar con 1M operaciones de buscar, insertar, y eliminar concurrentes.

**Pista:** El marcado lógico resuelve el problema de la eliminación en skip lists:
sin marcado, un goroutine puede estar "en medio" de un nodo que otro goroutine
está eliminando, corrompiendo la estructura. Con marcado, el nodo sigue existiendo
en memoria hasta que todos los goroutines que lo estaban usando terminan
(el GC lo liberará después).

**Implementar en:** Go · Java · C++ · Rust

---

### Ejercicio 12.4.3 — Rango concurrente en skip list lock-free

**Enunciado:** La operación de rango (todos los valores entre K1 y K2)
es el caso de uso principal de la skip list sobre el HashMap:

```go
func (s *SkipListLF[K, V]) Rango(min, max K) []V {
    // 1. Encontrar el primer elemento >= min usando los niveles superiores
    // 2. Iterar el nivel 0 hasta que la clave > max
    // Desafío: durante la iteración, otros goroutines pueden insertar o eliminar
}
```

**Restricciones:** El Rango debe retornar un snapshot consistente —
no incluir elementos insertados después del inicio del Rango,
no omitir elementos que existían al inicio (aunque estén siendo eliminados).

**Pista:** El snapshot consistente en una estructura lock-free se logra
con "snapshot isolation" — registrar el timestamp de inicio y solo incluir
elementos cuyo timestamp de inserción es anterior y cuyo timestamp de eliminación
(si existe) es posterior al inicio del Rango.
Una alternativa más simple pero menos precisa: tolerar cierta inconsistencia
(incluir elementos recién insertados, omitir elementos recién eliminados).

**Implementar en:** Go · Java · Rust

---

### Ejercicio 12.4.4 — Benchmark: Skip list vs B-tree vs HashMap para rangos

**Enunciado:** Para un sistema de time-series (datos indexados por timestamp),
compara tres estructuras para las operaciones típicas:
- Insert(timestamp, valor)
- Get(timestamp)
- RangeQuery(t1, t2) — todos los valores entre dos timestamps

Con N = 10M registros y 8 goroutines de lectura/escritura simultáneas.

**Restricciones:** Para RangeQuery, la skip list y el B-tree deben ser
significativamente más rápidos que el HashMap + sort posterior.
Para Get, el HashMap debe ser el más rápido.
Reporta throughput y latencia p99 para cada operación × estructura.

**Pista:** La skip list gana en RangeQuery porque los datos están ordenados
y el nivel 0 permite iteración secuencial eficiente.
El B-tree (btree.BTreeG de Go) gana en lecturas porque tiene mejor locality
de caché que la skip list (los nodos del B-tree son arrays contiguos).
El HashMap gana en Get porque no tiene overhead de comparación de claves para el orden.

**Implementar en:** Go (con `github.com/google/btree`) · Java (`TreeMap`) · Rust (`BTreeMap`)

---

### Ejercicio 12.4.5 — Skip list como índice secundario

**Enunciado:** Un sistema de base de datos simple necesita un índice secundario
que soporte búsquedas por rango sobre un campo no-primario:

```go
type BaseDatos struct {
    datosXID  map[int64]*Registro        // índice primario: O(1) por ID
    indicXNombre *SkipListLF[string, int64]  // índice secundario: O(log N) por nombre
}

func (db *BaseDatos) InsertarRegistro(r *Registro) error
func (db *BaseDatos) BuscarPorNombre(nombre string) (*Registro, bool)
func (db *BaseDatos) BuscarPorRangoDeNombre(min, max string) []*Registro
```

**Restricciones:** Las inserciones deben mantener ambos índices consistentes
bajo concurrencia. Si la inserción en el índice secundario falla (race condition),
el registro primario debe ser revertido (o el índice secundario retried).
El rango de nombres debe retornar todos los registros existentes al inicio del query,
en orden alfabético.

**Pista:** El mayor desafío: la consistencia entre el índice primario y el secundario.
Solución: insertar primero en el primario, luego en el secundario. La ventana de
inconsistencia (el registro está en el primario pero no en el secundario) es aceptable
si las búsquedas por nombre no requieren ver inmediatamente los últimos insertos.
Esto es "eventual consistency" a nivel de índice.

**Implementar en:** Go · Java · Rust

---

## Sección 12.5 — Ring Buffer Lock-Free (SPSC y MPMC)

El ring buffer (circular buffer) es la estructura más eficiente para
productor-consumidor cuando el tamaño máximo es conocido de antemano.
Sin necesidad de alocar memoria dinámicamente ni de seguir punteros.

**La estructura:**

```
[0] [1] [2] [3] [4] [5] [6] [7]   ← array circular de tamaño N
      ↑head                ↑tail
      consumidor lee aquí  productor escribe aquí

Lleno cuando: (tail + 1) % N == head
Vacío cuando: tail == head
```

**SPSC (Single Producer Single Consumer) — el caso más simple:**

```
El productor modifica tail.
El consumidor modifica head.
No hay contención entre ellos — nunca modifican el mismo puntero.
→ Solo necesitamos que el consumidor vea la escritura del productor:
  una barrera de memoria es suficiente.
```

---

### Ejercicio 12.5.1 — SPSC ring buffer

**Enunciado:** El SPSC ring buffer es la estructura de datos concurrente
más eficiente para un canal de comunicación:

```go
type RingBufferSPSC[T any] struct {
    data     []T
    capacidad int
    head     atomic.Uint64  // solo el consumidor escribe
    tail     atomic.Uint64  // solo el productor escribe
    // padding para evitar false sharing entre head y tail
    _headPad [56]byte
    _tailPad [56]byte
}

func (r *RingBufferSPSC[T]) Enqueue(v T) bool   // false si lleno
func (r *RingBufferSPSC[T]) Dequeue() (T, bool) // false si vacío
func (r *RingBufferSPSC[T]) Len() int
```

**Restricciones:** El throughput debe ser > 100M ops/s para items de 8 bytes.
Comparar con `chan T` buffered para el mismo tamaño de buffer.
El SPSC ring buffer debe ser al menos 3x más rápido que el canal de Go.

**Pista:** El secreto de la eficiencia del SPSC: el productor solo necesita
leer head una vez (para saber si está lleno), y puede cachear el valor.
La próxima vez que produzca, si el caché dice "no lleno", puede producir sin leer head.
Solo cuando el caché dice "lleno", relee head. Esto minimiza las cacheline
invalidations entre el productor y el consumidor.

**Implementar en:** Go · Rust · C++ · Java

---

### Ejercicio 12.5.2 — MPMC ring buffer con sequence numbers

**Enunciado:** El MPMC (Multi Producer Multi Consumer) ring buffer usa
números de secuencia para coordinar múltiples productores y consumidores:

```go
type SlotMPMC[T any] struct {
    secuencia atomic.Uint64  // número de secuencia del slot
    dato      T
}

type RingBufferMPMC[T any] struct {
    slots    []SlotMPMC[T]
    capacidad uint64
    // separados para evitar false sharing:
    enqPos  atomic.Uint64  // próxima posición de enqueue
    _       [56]byte
    deqPos  atomic.Uint64  // próxima posición de dequeue
}
```

**Algoritmo de enqueue:**
1. Leer enqPos (sea `pos`)
2. El slot en `pos % N` debe tener secuencia `pos` (indica que está libre)
3. Si secuencia < pos: la queue está llena
4. Si secuencia > pos: otro productor tomó esta posición — reintentar con pos nuevo
5. Si secuencia == pos: CAS(enqPos, pos, pos+1), escribir dato, setar secuencia=pos+1

**Restricciones:** El throughput debe ser > 20M ops/s con 4 productores y 4 consumidores.
La implementación es el algoritmo de Dmitry Vyukov (2010), uno de los más eficientes publicados.

**Pista:** El número de secuencia actúa como un "generational count" —
slot en posición `i` con secuencia `k*N + i` está disponible para el enqueue número `k*N + i`.
Después de N enqueues, el slot vuelve a estar disponible con secuencia `(k+1)*N + i`.
Este mecanismo previene el ABA problem sin tagged pointers.

**Implementar en:** Go · C++ · Rust · Java

---

### Ejercicio 12.5.3 — Ring buffer con backpressure

**Enunciado:** Cuando el ring buffer está lleno, el productor puede:
1. Retornar false (non-blocking) — el productor descarta el item
2. Bloquear (blocking) — esperar a que haya espacio
3. Bloquear con timeout — esperar hasta un tiempo máximo

Implementa las tres políticas:

```go
type PoliticaLleno int
const (
    Descartar PoliticaLleno = iota
    Bloquear
    BloquearConTimeout
)

type RingBufferConPolitica[T any] struct {
    ring    *RingBufferMPMC[T]
    politica PoliticaLleno
    timeout  time.Duration
    noLleno chan struct{}  // señal cuando hay espacio
}
```

**Restricciones:** La política Bloquear debe despertar automáticamente cuando
hay espacio (no polling). El timeout debe ser preciso: no despertar antes,
no esperar mucho más del timeout.

**Pista:** Para la política Bloquear, el consumidor señaliza al producir:
después de cada Dequeue, enviar a un canal `noLleno`. El productor espera en ese canal
cuando encuentra el buffer lleno. El canal debe tener buffer para que el Dequeue
no se bloquee al señalizar.

**Implementar en:** Go · Java (`BlockingQueue`) · Rust · C#

---

### Ejercicio 12.5.4 — Disruptor pattern: el ring buffer más rápido

**Enunciado:** El LMAX Disruptor (Martin Thompson, 2011) es el ring buffer
más rápido publicado — diseñado para trading de alta frecuencia.
Su innovación principal: separar los "sequence numbers" para producción y consumición,
y usar "barrier sequences" para coordinación:

```
Producer Sequencer:  productores solicitan slots usando fetch-and-add
Consumer Barrier:    consumidores esperan hasta que la secuencia esperada esté publicada
Ring Buffer:         el array circular compartido entre productores y consumidores

Throughput: > 25 millones de mensajes por segundo en 1 thread
```

Implementa el Disruptor para un caso de uso específico:
1 productor → ring buffer → N consumidores en pipeline:

```go
type Disruptor[T any] struct {
    buffer    []T
    capacidad int64
    cursorProd atomic.Int64  // productor
    cursoresCons []atomic.Int64  // uno por consumidor
}
```

**Restricciones:** Con 1 productor y 3 consumidores en pipeline (C3 espera a C2, C2 espera a C1),
el throughput debe ser > 50M eventos/segundo para eventos de 8 bytes.

**Pista:** La clave del Disruptor es evitar locks y canales completamente.
Los consumidores usan spinning (busy-waiting) sobre las secuencias del productor —
esto consume CPU pero tiene latencia mínima (<1µs).
Para sistemas de trading, intercambiar un core de CPU por 1µs de latencia es
frecuentemente el tradeoff correcto.

**Implementar en:** Go · Java (el Disruptor original es en Java) · C++

---

### Ejercicio 12.5.5 — Comparar ring buffer vs canal de Go

**Enunciado:** Go tiene canales integrados que son ring buffers internamente.
Compara los canales de Go con el SPSC y MPMC ring buffer propios
para distintos patrones de uso:

**Casos de benchmark:**
1. Latencia de round-trip (ping-pong entre dos goroutines)
2. Throughput de streaming (1M items)
3. Latencia bajo carga (p99 con 90% utilización)
4. Overhead de creación (crear y destruir 1M canales/buffers)
5. Uso de memoria para buffer de 1M elementos de 8 bytes

**Restricciones:** Para cada caso, reportar el valor absoluto y la razón
ring buffer / canal. Identificar en qué casos el canal de Go es suficientemente
bueno (razón < 2x) y en qué casos el ring buffer lock-free vale la pena.

**Pista:** Los canales de Go tienen overhead de ~400-600 ns por operación
por su implementación general (soporta select, close, ranging).
El SPSC ring buffer puede hacer ~3-5 ns por operación.
Para la mayoría de aplicaciones, 400 ns es perfectamente acceptable.
Para HFT (high-frequency trading) o sistemas de logging de ultra-baja latencia,
la diferencia importa.

**Implementar en:** Go · comparar con Java (ArrayBlockingQueue) · Rust (crossbeam-channel)

---

## Sección 12.6 — Correctness: Testing y Verificación Formal

Las estructuras lock-free son notoriamente difíciles de verificar.
Un bug puede no manifestarse durante meses y luego aparecer bajo carga específica.

**Las cuatro herramientas de verificación:**

```
1. Race detector (go test -race):
   Detecta accesos concurrentes a la misma memoria sin sincronización.
   No detecta bugs de linearizabilidad (uso incorrecto de atómicos).

2. Linearizabilidad checker (porcupine):
   Verifica que la historia de operaciones es linearizable.
   Detecta bugs de correctness que el race detector no ve.

3. Model checker (TLA+, Alloy):
   Exploración exhaustiva del espacio de estados.
   Puede probar ausencia de bugs, no solo encontrar bugs existentes.

4. Stress testing con determinismo:
   Replay exacto de schedulings que produjeron bugs.
   Requiere herramientas especializadas (CHESS, CDSChecker).
```

---

### Ejercicio 12.6.1 — Generar historias de operaciones para linearizabilidad

**Enunciado:** Instrumenta el Stack lock-free para registrar todas las
operaciones con sus tiempos de inicio y fin:

```go
type OperacionHistoria struct {
    tipo    string         // "push" o "pop"
    valor   int
    inicio  time.Time
    fin     time.Time
    retorno interface{}    // nil para push, (valor, bool) para pop
}

type StackInstrumentado[T any] struct {
    stack    *Stack[T]
    historia []OperacionHistoria
    mu       sync.Mutex
}
```

**Restricciones:** El registro de la historia no debe afectar la correctness
del stack (el historial es un side effect, no parte de la lógica).
Con la historia, usa `porcupine` para verificar linearizabilidad.

**Pista:** Porcupine necesita: (a) la especificación secuencial (qué retorna
un stack en cualquier secuencia de operaciones), (b) la historia de operaciones
con tiempos de inicio y fin. El checker verifica que existe un linearization point
para cada operación que es consistente con la especificación secuencial.

**Implementar en:** Go (porcupine es una librería de Go)

---

### Ejercicio 12.6.2 — Differential testing contra la implementación de referencia

**Enunciado:** El differential testing ejecuta la misma secuencia de operaciones
en la implementación lock-free y en una implementación correcta pero no concurrente,
verificando que producen los mismo resultados:

```go
type StackDifferential[T comparable] struct {
    lockFree  *Stack[T]
    referencia *[]T       // protegida por mutex, implementación trivialmente correcta
    mu         sync.Mutex
    rng        *rand.Rand
}

func (d *StackDifferential[T]) TestAleatorio(n int, goroutines int) error {
    // Ejecutar n operaciones aleatorias con `goroutines` goroutines
    // Para cada operación, ejecutar en ambas implementaciones
    // Verificar que los resultados son "equivalentes" (mismo estado final)
}
```

**Restricciones:** El "mismo estado final" en estructuras concurrentes no significa
exactamente el mismo output — el ordenamiento puede variar. Lo que sí debe ser
igual: el número de elementos, el conjunto de valores, y las propiedades LIFO.

**Pista:** El differential testing es más práctico que la verificación formal
para encontrar bugs en código existente. Cuando el test encuentra una discrepancia,
el shrinking (de la Sección 7.2.3) reduce la secuencia al caso mínimo que reproduce el bug.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 12.6.3 — Sanitizers: ThreadSanitizer y AddressSanitizer

**Enunciado:** ThreadSanitizer (TSan) y AddressSanitizer (ASan) son herramientas
de instrumentación en tiempo de compilación para detectar bugs de concurrencia
y memoria:

```bash
# Go usa TSan internamente en go test -race
# Para código C/C++/Rust:

# ThreadSanitizer:
clang -fsanitize=thread -O1 -o stack_tsan stack_lock_free.c
./stack_tsan  # reporta races al detectarlas

# AddressSanitizer (para use-after-free en código sin GC):
clang -fsanitize=address -O1 -o stack_asan stack_lock_free.c
./stack_asan  # reporta use-after-free, buffer overflows
```

**Restricciones:** Compila y ejecuta la implementación C del Stack lock-free
con TSan y ASan. Fuerza un bug (comentar la barrera de memoria) y verifica
que TSan lo detecta.

**Pista:** TSan y ASan tienen overhead de ejecución (2-5x más lento, 2-10x más memoria).
Son herramientas para desarrollo y testing, no para producción.
En Rust, AddressSanitizer se activa con `RUSTFLAGS="-Z sanitizer=address"` en nightly.
El race detector de Go (`-race`) usa TSan internamente.

**Implementar en:** C · Rust · C++ (en Go, -race es la interfaz)

---

### Ejercicio 12.6.4 — Especificación TLA+ de la queue de Michael-Scott

**Enunciado:** Escribe la especificación TLA+ de la queue de Michael-Scott
y verifica con TLC que es linearizable:

```tla
EXTENDS Sequences, Naturals, TLC

VARIABLES head, tail, nodos, valor, siguiente, marcado, pc

(* La especificación cubre los estados de las goroutines y la queue *)
(* TLC verifica que siempre se mantiene la propiedad de linearizabilidad *)
```

**Restricciones:** La especificación debe modelar al menos 2 productores
y 2 consumidores. TLC debe verificar en todos los estados que:
1. No hay race conditions
2. Un Dequeue en una queue no vacía siempre retorna un valor
3. El orden FIFO se mantiene (si A fue enqueued antes que B, A es dequeued antes)

**Pista:** TLA+ para algoritmos concurrentes es uno de los usos más valiosos.
Amazon usa TLA+ para verificar DynamoDB y S3 internamente.
El modelo de TLA+ de la queue de M&S está publicado — este ejercicio
te pide escribirlo tú mismo, no copiarlo.

**Implementar en:** TLA+ (lenguaje de especificación, no programación)

---

### Ejercicio 12.6.5 — Invariante-checking en runtime con assertions

**Enunciado:** Implementa checking de invariantes en runtime para el HashMapStriped
del Ejercicio 12.3.1, activable con una build tag:

```go
//go:build assertions

func (h *HashMapStriped[K, V]) verificarInvariantes() {
    // Tomar un snapshot consistente (lock todos los segmentos)
    // Verificar:
    // 1. El número total de elementos coincide con h.Len()
    // 2. Cada clave está en el segmento correcto (hash(k) % NumSegmentos)
    // 3. No hay claves duplicadas entre segmentos
    // 4. Todos los valores son no-nil (si el tipo no permite nil)
}
```

**Restricciones:** El checking de invariantes puede ser lento (O(N))
porque bloquea toda la estructura. Solo se activa con `-tags assertions`.
Sin la build tag, la función compila a no-op.

**Pista:** El pattern de invariant checking en runtime es muy valioso para
debugging de estructuras concurrentes. La checklist de invariantes para un
hashmap: distribución uniforme de claves entre segmentos, sin duplicados,
tamaño consistente. Cuando se encuentra una violación, el stack trace del
momento de detección apunta al lugar donde se rompió la invariante.

**Implementar en:** Go · Rust (`debug_assert!`) · Java (`assert`)

---

## Sección 12.7 — Cuándo Usar Lock-Free vs Mutex

La pregunta más importante del capítulo no es "¿cómo implementar estructuras
lock-free?" sino "¿cuándo vale la pena la complejidad?"

**La matriz de decisión:**

```
                         Contención baja    Contención alta
                        ┌────────────────┬──────────────────┐
Operación simple        │   mutex (2)     │  CAS / atomic    │
(1-2 variables)         │                │  lock-free       │
                        ├────────────────┼──────────────────┤
Operación compleja      │   mutex (1)     │  mutex striped   │
(múltiples variables)   │                │  o lock-free     │
                        └────────────────┴──────────────────┘

(1) Mutex simple — la complejidad no vale la pena con baja contención
(2) Mutex simple — lock-free no mejora mucho con baja contención
```

**Los números que guían la decisión:**

```
sync.Mutex sin contención:      ~20 ns  (lock + unlock)
sync.Mutex con contención (4):  ~100 ns
sync.Mutex con contención (16): ~500 ns
atomic.AddInt64:                ~5 ns
CAS con contención (4):         ~15 ns
CAS con contención (16):        ~30 ns
```

---

### Ejercicio 12.7.1 — La prueba de contención

**Enunciado:** Determina empíricamente cuándo la contención es "alta" para
diferentes operaciones en tu hardware específico:

```go
// Para cada estructura y número de goroutines:
// 1. Medir el throughput con 1 goroutine (sin contención)
// 2. Medir con N goroutines hasta encontrar donde el throughput se degrada
// El punto de degradación es la "contención alta" para esa estructura

func PruebaContención(estructura interface{}, maxGoroutines int) []PuntoDegradacion
```

**Restricciones:** Medir para: `sync.Mutex`, `sync.RWMutex`, `atomic.Int64`,
Stack CAS, y Queue M&S. Para cada uno, reportar el número de goroutines donde
el throughput se degrada > 20% respecto a la linealidad teórica.

**Pista:** El mutex suele degradar a partir de 4-8 goroutines en hardware moderno.
Los atomics degradan más tarde (8-16 goroutines) porque no bloquean.
El punto de degradación depende de la sección crítica:
para secciones críticas de 0ns, el lock overhead domina y la degradación es inmediata.
Para secciones de 1µs, el lock overhead es irrelevante y la degradación es tarde.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 12.7.2 — El costo total de ownership de una estructura lock-free

**Enunciado:** Cuantifica el costo total de ownership (TCO) de usar una
estructura lock-free vs un mutex:

```
Costos del mutex:
  - Contención: throughput reducido bajo carga
  - Líneas de código: ~5 líneas para proteger una operación
  - Testing: test de concurrencia estándar

Costos lock-free:
  - Implementación: 200+ líneas vs 5
  - Testing: verificación de linearizabilidad necesaria
  - Debugging: bugs sutiles que aparecen bajo carga específica
  - Mantenimiento: los ingenieros que no la escribieron no la entienden
  - Portabilidad: puede no funcionar en todos los modelos de memoria
```

**Restricciones:** Para un sistema de caché con 90% reads y 10% writes,
y 8 goroutines, determina si el lock-free o el mutex tienen menor TCO
basándote en los benchmarks del capítulo.

**Pista:** La respuesta correcta casi siempre es: empieza con mutex,
mide bajo carga real, y migra a lock-free solo si el profiler muestra
que el mutex es el cuello de botella. El 99% de los sistemas nunca llegan
a ese punto.

**Implementar en:** Análisis cuantitativo + benchmark de validación

---

### Ejercicio 12.7.3 — Bibliotecas de producción: cuándo usarlas

**Enunciado:** Antes de implementar tu propia estructura lock-free,
considera las bibliotecas existentes:

```go
// Go
import "github.com/petermattis/concurrent"  // concurrent hashmap
import "github.com/cockroachdb/pebble"      // LSM tree concurrente
import "go.uber.org/atomic"                  // atomics con API mejorada

// Rust
// crossbeam (queue, channel, deque — las mejores implementaciones publicadas)
// dashmap (hashmap concurrente)

// Java
// java.util.concurrent (ConcurrentHashMap, ConcurrentLinkedQueue — producción)
// Disruptor (LMAX — para ultra-baja latencia)
```

Para el sistema de recomendaciones del Ejercicio 10.7.4, reemplaza las
estructuras personalizadas con bibliotecas de producción y mide la diferencia.

**Restricciones:** Las bibliotecas deben ser drop-in replacements —
misma API, mismo comportamiento externo. Mide la diferencia de throughput
y latencia p99.

**Pista:** Las bibliotecas de producción (especialmente `java.util.concurrent`)
han sido optimizadas durante décadas y verificadas formalmente.
Tu implementación del Ejercicio 12.2.1 probablemente es correcta pero
probablemente no es más rápida que `ConcurrentLinkedQueue` de Java.
El valor del capítulo no es reemplazar las bibliotecas — es entender qué hay dentro.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 12.7.4 — La regla del 80/20 para lock-free

**Enunciado:** El 80% de los beneficios del lock-free se logra con el 20% de la complejidad.
Identifica cuáles son esas "ganancias fáciles":

```
Ganancias fáciles (bajo costo, alto beneficio):
  □ Usar atomic.Int64 en vez de mutex para un contador
  □ Usar sync.Map para mapas read-heavy (el propio Go lo implementó)
  □ Usar canales buffered en vez de mutex + slice para producer-consumer
  □ Usar sync.Once en vez de double-checked locking manual
  □ Usar atomic.Pointer[T] para un singleton inicializado una vez

Ganancias difíciles (alto costo, beneficio marginal para la mayoría):
  □ Stack lock-free para baja contención
  □ Queue M&S para < 8 goroutines
  □ HashMap lock-free si striped + RWMutex funciona bien
```

**Restricciones:** Para cada "ganancia fácil", implementa el antes y después
y verifica que la complejidad adicional es mínima.

**Pista:** La gran lección del capítulo: las primitivas atómicas de la biblioteca
estándar (`sync/atomic`) son el 80% del valor de lock-free para la mayoría
de casos. Las estructuras de datos lock-free completas (stacks, queues, skip lists)
son para el 1% de casos donde las primitivas no son suficientes y el profiler
lo ha demostrado.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 12.7.5 — Sistema de producción: auditoría de sincronización

**Enunciado:** Toma cualquier sistema implementado en los capítulos anteriores
(el Worker Pool del Cap.03, el Pipeline del Cap.03, o el Workflow Engine del Cap.10)
y realiza una auditoría completa de sincronización:

1. **Inventario**: listar todas las variables compartidas entre goroutines
2. **Clasificación**: para cada variable, ¿es apropiado el mecanismo actual?
3. **Hotspots**: ¿qué variables tienen mayor contención bajo carga?
4. **Propuestas**: ¿qué cambios mejorarían el throughput > 2x?
5. **Riesgo**: ¿qué cambios introducirían riesgos de correctness?

**Restricciones:** La auditoría debe ser cuantitativa: usar `go test -race` y
benchmarks con GOMAXPROCS=1, 4, y 8 para identificar los hotspots reales.
Las propuestas deben implementarse y medirse.

**Pista:** La auditoría de sincronización es una habilidad de alta demanda
en equipos de infraestructura. El resultado típico: el 80% de los mecanismos
de sincronización son apropiados; el 15% están sobre-sincronizados (pueden
simplificarse); el 5% están bajo-sincronizados (tienen bugs latentes).

**Implementar en:** Go (el sistema auditado) + análisis

---

## Resumen del capítulo y de la Parte 2

**Las estructuras en orden de complejidad / cuando usarlas:**

| Estructura | Complejidad | Usar cuando |
|---|---|---|
| `atomic.Int64` | Trivial | Contador o flag con 1 variable |
| `sync.RWMutex` + map | Baja | Read-heavy, operaciones complejas |
| `sync.Map` | Baja | Claves disjuntas o escrituras raras |
| HashMap striped | Media | Alta contención, necesitas rangos |
| Stack/Queue lock-free | Alta | > 16 goroutines, profiler lo indica |
| Skip list | Muy alta | Rangos + alta concurrencia + benchmarks lo justifican |
| Ring buffer SPSC/MPMC | Alta | Latencia ultra-baja, HFT, logging crítico |

**El principio que cierra la Parte 2:**

> El paralelismo eficiente requiere entender tres capas simultáneamente:
> el algoritmo (Cap.08-10), el hardware (Cap.11), y las estructuras de datos (Cap.12).
> Una optimización que ignora cualquiera de las tres puede hacer el sistema más lento.
> Un cambio de `sync.Mutex` a CAS que aumenta el número de cacheline invalidations
> puede ser net-negative. Medir siempre.

## La pregunta que guía la Parte 3

> La Parte 1 (Cap.01-07) cubrió concurrencia en Go.
> La Parte 2 (Cap.08-12) cubrió paralelismo y hardware.
> La Parte 3 (Cap.13-17) cubre los mismos conceptos en Rust, Java, Python, y C# —
> cada lenguaje con su modelo de concurrencia propio, sus primitivas, y sus tradeoffs.
> El Cap.13 empieza con Rust: el lenguaje donde el compilador hace cumplir
> las reglas de concurrencia en tiempo de compilación.
