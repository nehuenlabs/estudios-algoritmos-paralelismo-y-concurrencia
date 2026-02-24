# Guía de Ejercicios — Herramientas de Sincronización

> Implementar cada ejercicio en: **Go · Java · Python · Rust · C#**
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el contador arreglado

El Cap.01 terminó con el contador roto. Esta es la versión correcta:

```go
// Solución 1: mutex
var contador int
var mu sync.Mutex

func incrementar(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        mu.Lock()
        contador++
        mu.Unlock()
    }
}

// Solución 2: atómico
var contador int64

func incrementar(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        atomic.AddInt64(&contador, 1)
    }
}

// Solución 3: canal
func incrementar(ch chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        ch <- 1
    }
}

func acumulador(ch <-chan int, done chan<- int) {
    total := 0
    for v := range ch {
        total += v
    }
    done <- total
}
```

Las tres soluciones producen `2000` siempre. Pero no son equivalentes.
La diferencia no es solo de sintaxis — es de modelo mental y de cuándo usar cada una.

**Este capítulo responde:** ¿cuándo usar mutex, cuándo atómico, cuándo canal,
y por qué cada uno funciona a nivel del modelo de memoria?

---

## El modelo de memoria en una frase

> Si A **happens-before** B, entonces B ve todos los efectos de A.

Eso es todo. La complejidad está en saber qué establece la relación happens-before
y qué no la establece.

```
SÍ establece happens-before:
  mu.Unlock()  → siguiente mu.Lock()
  ch <- valor  → recepción de ese valor en <-ch
  sync.Once.Do termina → cualquier llamada posterior a Do retorna
  goroutine lanzada → primera instrucción de la goroutine
  wg.Done() (n veces) → wg.Wait() retorna

NO establece happens-before:
  escribir variable sin lock → leer esa variable en otra goroutine
  lanzar goroutine → instrucciones ANTES del go func()
  time.Sleep(100ms) → "seguramente ya terminó"
```

El mutex, el atómico y el canal son tres formas distintas de establecer
happens-before. Elegir entre ellos es elegir qué relación de orden necesitas.

---

## Tabla de contenidos

- [Sección 2.1 — Mutex: exclusión mutua](#sección-21--mutex-exclusión-mutua)
- [Sección 2.2 — RWMutex: lectores y escritores](#sección-22--rwmutex-lectores-y-escritores)
- [Sección 2.3 — Operaciones atómicas](#sección-23--operaciones-atómicas)
- [Sección 2.4 — Canales como sincronización](#sección-24--canales-como-sincronización)
- [Sección 2.5 — sync.WaitGroup y sync.Once](#sección-25--syncwaitgroup-y-synconce)
- [Sección 2.6 — Condition variables](#sección-26--condition-variables)
- [Sección 2.7 — Elegir la herramienta correcta](#sección-27--elegir-la-herramienta-correcta)

---

## Sección 2.1 — Mutex: Exclusión Mutua

El mutex es la herramienta más simple: garantiza que solo una goroutine
ejecuta la **sección crítica** a la vez.

**Por qué funciona — el modelo de memoria:**

```
mu.Unlock() en goroutine A  →  happens-before  →  mu.Lock() en goroutine B

Consecuencia:
  Todo lo que A escribió ANTES de mu.Unlock()
  es visible para B DESPUÉS de su mu.Lock().

Esto es lo que "proteger con mutex" significa formalmente:
no solo "solo uno a la vez", sino "el que sigue ve todo lo que hizo el anterior".
```

**La anatomía de una sección crítica:**

```go
mu.Lock()
// ↑ punto de entrada — solo una goroutine pasa a la vez
// ↓ sección crítica — debe ser lo más pequeña posible

estado = nuevoValor   // acceso exclusivo al estado compartido

// ↑ fin de la sección crítica
mu.Unlock()
// ↓ punto de salida — la siguiente goroutine puede entrar
```

**El error más común — sección crítica demasiado grande:**

```go
// MAL: operación costosa dentro del lock
mu.Lock()
resultado = llamadaABaseDeDatos()  // bloquea a todos durante 200ms
estado = procesar(resultado)
mu.Unlock()

// BIEN: solo el acceso al estado compartido dentro del lock
resultado = llamadaABaseDeDatos()  // fuera del lock — paralelo
mu.Lock()
estado = procesar(resultado)       // solo esto necesita exclusión
mu.Unlock()
```

**Defer para garantizar el unlock:**

```go
func (s *Servicio) Procesar(req Request) error {
    s.mu.Lock()
    defer s.mu.Unlock()  // se ejecuta cuando la función retorna, pase lo que pase
    // si hay un panic o un return temprano, el unlock ocurre de todas formas
    return s.procesarInterno(req)
}
```

**El costo real del mutex:**

```
Sin contención (solo una goroutine):  ~20ns por Lock/Unlock
Con contención moderada (2-4 cores):  ~100-300ns
Con contención alta (16+ cores):      ~1000ns+

Comparar con:
  atomic.AddInt64:  ~10ns sin contención
  operación en RAM: ~100ns
  canal sin buffer: ~50ns
```

---

### Ejercicio 2.1.1 — Implementar un contador seguro con tres enfoques

**Enunciado:** Implementa un contador concurrente con las tres soluciones del inicio del capítulo (mutex, atómico, canal). Para cada una:
1. Verifica que `go test -race` no reporta nada
2. Mide el tiempo para `10^6` incrementos con `4` goroutines
3. Identifica cuál es más rápida y explica por qué

**Restricciones:** Las tres deben dar exactamente el mismo resultado final. El benchmark debe usar `testing.B` de Go.

**Pista:** El atómico debería ser más rápido que el mutex para este caso — `atomic.AddInt64` es una sola instrucción de CPU (`LOCK XADD` en x86), mientras que el mutex requiere una syscall cuando hay contención. El canal puede ser el más lento por el overhead de scheduling.

**Implementar en:** Go · Java (`synchronized` vs `AtomicInteger` vs `BlockingQueue`) · Python · Rust · C#

---

### Ejercicio 2.1.2 — Mutex y defer: garantizar el unlock

**Enunciado:** Refactoriza este código para que el mutex siempre se libere, incluso si `procesarInterno` lanza un panic:

```go
func (s *Servicio) Guardar(clave, valor string) error {
    s.mu.Lock()
    if clave == "" {
        s.mu.Unlock()
        return errors.New("clave vacía")
    }
    err := s.bd.Guardar(clave, valor)
    if err != nil {
        s.mu.Unlock()
        return err
    }
    s.cache[clave] = valor
    s.mu.Unlock()
    return nil
}
```

**Restricciones:** La versión refactorizada debe tener exactamente un `Unlock`. Verifica que funciona correctamente cuando `bd.Guardar` lanza un panic.

**Pista:** Con `defer mu.Unlock()` inmediatamente después del `Lock()`, el unlock ocurre cuando la función retorna por cualquier razón — return normal, return con error, o panic. Esto elimina todos los paths de código que podrían olvidar el unlock.

**Implementar en:** Go · Java (`try/finally`) · Python (`with` statement) · Rust (RAII — automático) · C# (`using`)

---

### Ejercicio 2.1.3 — Detectar sección crítica innecesariamente grande

**Enunciado:** Este código es correcto pero tiene una sección crítica más grande de lo necesario. Identifica qué parte no necesita el lock y refactoriza:

```go
func (s *Servidor) ManejarRequest(req Request) Response {
    s.mu.Lock()
    defer s.mu.Unlock()

    // Validar el request (no accede a estado compartido)
    if err := validar(req); err != nil {
        return Response{Error: err}
    }

    // Llamar a servicio externo (tarda 50-200ms)
    datos, err := s.cliente.Obtener(req.ID)
    if err != nil {
        return Response{Error: err}
    }

    // Actualizar caché (SÍ accede a estado compartido)
    s.cache[req.ID] = datos

    // Formatear respuesta (no accede a estado compartido)
    return formatear(datos)
}
```

**Restricciones:** La versión refactorizada debe mantener la corrección — sin race conditions. Mide la diferencia de throughput con 10 goroutines concurrentes.

**Pista:** La llamada a `s.cliente.Obtener` dura 50-200ms dentro del lock — bloquea a todas las demás goroutines durante ese tiempo. Moverla fuera del lock permite que múltiples requests se procesen en paralelo para la parte lenta.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 2.1.4 — Mutex recursivo — el problema de Go

**Enunciado:** Go no tiene mutex recursivo (un mutex que la misma goroutine puede lockear múltiples veces). Este código causa un deadlock:

```go
func (s *Servicio) A() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.B()  // B también intenta lockear mu → deadlock
}

func (s *Servicio) B() {
    s.mu.Lock()
    defer s.mu.Unlock()
    // hacer algo
}
```

Arréglalo de **dos formas distintas**:
1. Extrayendo la lógica a métodos privados que no lockean
2. Usando un lock explícito con un campo que registra si ya está lockeado

Explica por qué Go no tiene mutex recursivo por diseño.

**Restricciones:** La solución 1 es la preferida en Go. La solución 2 es un anti-patrón — impleméntala pero documenta por qué es mala.

**Pista:** Go deliberadamente no tiene mutex recursivo porque enmascara problemas de diseño. Si una función necesita lockear algo que ya está lockeado, generalmente indica que el diseño del locking es incorrecto. La solución correcta es refactorizar, no añadir reentrancy.

**Implementar en:** Go · Java (`ReentrantLock` — sí tiene recursivo) · Python (`threading.RLock`)

---

### Ejercicio 2.1.5 — Composición de locks: el problema de invariantes

**Enunciado:** Una `CuentaBancaria` tiene su propio mutex. Cuando se hace una transferencia entre dos cuentas, necesitas lockear ambas. El Ejercicio 1.2.4 mostró que esto puede causar deadlock si no se ordena bien.

Ahora el reto es diferente: ¿cómo garantizar que el **invariante del sistema** (`suma_total_de_todas_las_cuentas == constante`) se mantiene durante las transferencias concurrentes?

```go
type Banco struct {
    cuentas map[int]*Cuenta
}

// Invariante: sum(cuenta.saldo for cuenta in cuentas) == TOTAL_INICIAL
// ¿Cómo verificar este invariante de forma concurrente sin freezar todo el banco?
```

**Restricciones:** La verificación del invariante no puede bloquear todas las transferencias. Considera si es posible verificar el invariante sin una "fotografía" consistente del estado.

**Pista:** Este es el problema de consistencia en sistemas concurrentes. La respuesta honesta es que **no se puede** verificar el invariante global sin alguna forma de snapshot consistente — a menos que uses una estructura de datos que lo permita (como MVCC en bases de datos).

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 2.2 — RWMutex: Lectores y Escritores

Cuando el acceso compartido es mayoritariamente lectura, el mutex estándar
es innecesariamente restrictivo — bloquea lectores que podrían ejecutar en paralelo.

**El principio:**

```
Lecturas concurrentes son seguras entre sí — dos goroutines pueden leer
el mismo valor simultáneamente sin problema.

Una escritura necesita exclusión total — ningún lector ni escritor puede
estar activo al mismo tiempo que el escritor.
```

**Cuándo usar RWMutex:**

```go
// Patrón típico: muchas lecturas, pocas escrituras
type Configuracion struct {
    datos map[string]string
    mu    sync.RWMutex
}

func (c *Configuracion) Leer(clave string) string {
    c.mu.RLock()         // múltiples goroutines pueden tener RLock simultáneamente
    defer c.mu.RUnlock()
    return c.datos[clave]
}

func (c *Configuracion) Escribir(clave, valor string) {
    c.mu.Lock()          // exclusión total — bloquea todos los RLock y Lock
    defer c.mu.Unlock()
    c.datos[clave] = valor
}
```

**Cuándo NO usar RWMutex:**

```
- Si las escrituras son frecuentes: el overhead de RWMutex > Mutex
- Si las secciones críticas son muy cortas: el overhead domina
- Si solo hay 1-2 goroutines: no hay beneficio del paralelismo de lecturas
- Regla práctica: benchmarkea antes de reemplazar Mutex por RWMutex
```

**El modelo de memoria:**

```
RUnlock() → happens-before → Lock() siguiente
Lock() (escritor) → happens-before → RLock() o Lock() siguiente

Importante: múltiples RLock() concurrentes NO tienen relación happens-before
entre sí — porque son independientes y eso es exactamente el punto.
```

---

### Ejercicio 2.2.1 — Caché con RWMutex

**Enunciado:** Implementa una caché en memoria segura para concurrencia usando RWMutex. Las operaciones son: `Get(clave)`, `Set(clave, valor)`, `Delete(clave)`, `Keys()`. Verifica con `-race` y mide el throughput con 90% lecturas / 10% escrituras vs 50% / 50%.

**Restricciones:** `Keys()` debe retornar una copia de las claves — no una referencia al slice interno. ¿Por qué?

**Pista:** Retornar una referencia al slice interno bajo RLock es peligroso: el llamador puede retener la referencia después de que el RLock se libera, y luego un escritor modifica el mapa — race condition fuera del control de la caché.

**Implementar en:** Go · Java (`ReadWriteLock`) · Python (`threading.RLock`) · Rust · C#

---

### Ejercicio 2.2.2 — Benchmark: Mutex vs RWMutex

**Enunciado:** Para una caché con diferentes ratios de lectura/escritura, mide cuándo RWMutex supera a Mutex:

| Ratio lecturas | Ratio escrituras | ¿Cuál es más rápido? |
|---|---|---|
| 50% | 50% | ? |
| 80% | 20% | ? |
| 95% | 5% | ? |
| 99% | 1% | ? |

**Restricciones:** Usa `testing.B` con `b.RunParallel`. El benchmark debe correr con `GOMAXPROCS=8` para que el paralelismo de lecturas sea real.

**Pista:** El breakeven típico está en ~70-80% lecturas. Por debajo de eso, el overhead de RWMutex (más complejo que Mutex) puede hacerlo más lento. La respuesta depende del hardware — mídelo en la máquina que importa.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 2.2.3 — El problema del escritor hambriento (revisitado)

**Enunciado:** Go's `sync.RWMutex` tiene una política de fairness: cuando un escritor espera, los nuevos lectores deben esperar también (no pueden "colarse" delante del escritor). Verifica esta propiedad empíricamente:

1. Lanza 8 goroutines lectoras que leen continuamente
2. Lanza 1 goroutine escritora
3. Mide cuánto tiempo espera el escritor

Luego compara con una implementación de RWMutex sin esta política de fairness.

**Restricciones:** La comparación debe ser en tiempo real medido, no solo teórica.

**Pista:** La política de Go es "escritores tienen prioridad sobre nuevos lectores". Esto previene starvation del escritor pero puede reducir el throughput de lecturas cuando hay escrituras frecuentes. No hay una política universalmente mejor — depende del caso de uso.

**Implementar en:** Go · Java (comparar `fair=true` vs `fair=false` en `ReentrantReadWriteLock`)

---

### Ejercicio 2.2.4 — Read-copy-update (RCU) simplificado

**Enunciado:** RCU es una técnica donde los lectores nunca bloquean — los escritores crean una nueva versión del dato y atómicamente reemplazan el puntero. Los lectores que ya empezaron con la versión vieja la terminan; los nuevos lectores ven la nueva versión.

Implementa un RCU simplificado para una configuración que se actualiza raramente pero se lee millones de veces por segundo.

**Restricciones:** Las lecturas no deben usar ningún lock. Las escrituras pueden ser lentas — ocurren una vez por minuto. Verifica con `-race`.

**Pista:** En Go, `atomic.Pointer[T]` permite intercambiar un puntero atómicamente. El escritor crea una nueva struct, la inicializa completamente, y luego hace `ptr.Store(nuevaConfig)`. Los lectores hacen `config := ptr.Load()` y trabajan con su copia local — aunque el escritor actualice después, trabajan con la versión que cargaron.

**Implementar en:** Go · Java (`AtomicReference`) · Rust · C#

---

### Ejercicio 2.2.5 — Snapshot consistente de múltiples variables

**Enunciado:** Un sistema tiene tres contadores que deben leerse de forma consistente: `requests`, `errores`, `latencia_total`. Con RWMutex sobre cada uno individualmente, no se puede garantizar que una lectura de los tres sea atómica.

Implementa dos soluciones:
1. Un único RWMutex para los tres (simple pero menos concurrente)
2. Snapshot con versión: cada escritura incrementa un contador de versión, el lector reintenta si la versión cambia entre la primera y última lectura

**Restricciones:** La solución 2 debe ser lock-free para los lectores. Verifica que el snapshot es siempre consistente — nunca mezcla valores de versiones distintas.

**Pista:** La solución 2 es una forma de "optimistic concurrency control" — el lector asume que no habrá escritura concurrente, y verifica al final. Si hubo escritura, reintenta. Funciona bien cuando las escrituras son infrecuentes.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 2.3 — Operaciones Atómicas

Las operaciones atómicas son la sincronización más barata: sin syscalls,
sin cambio de contexto, sin scheduling. Una sola instrucción de CPU.

**Qué garantiza una operación atómica:**

```
atomic.AddInt64(&x, 1) garantiza:
  1. La operación leer-incrementar-escribir es indivisible
     (ninguna goroutine puede interrumpir en medio)
  2. El valor nuevo es visible para todas las goroutines inmediatamente
     (barrera de memoria implícita)
  3. Las operaciones atómicas sobre x están totalmente ordenadas
     (no hay reordenamiento del compilador para x)

atomic.AddInt64(&x, 1) NO garantiza:
  Ninguna relación happens-before con otras variables
  "x se incrementó" no implica nada sobre y, z, o cualquier otra variable
```

**Las operaciones disponibles en Go:**

```go
import "sync/atomic"

// Operaciones sobre enteros
atomic.AddInt64(&x, delta)           // x += delta, retorna nuevo valor
atomic.LoadInt64(&x)                 // lectura atómica
atomic.StoreInt64(&x, valor)         // escritura atómica
atomic.SwapInt64(&x, nuevo)          // x = nuevo, retorna viejo
atomic.CompareAndSwapInt64(&x, esperado, nuevo)  // si x==esperado: x=nuevo, retorna true

// Punteros (Go 1.19+)
var ptr atomic.Pointer[MiStruct]
ptr.Store(nuevaInstancia)
actual := ptr.Load()
ptr.CompareAndSwap(viejo, nuevo)
```

**Compare-And-Swap (CAS) — la operación fundamental:**

```go
// CAS es la base de todas las estructuras lock-free
// "Si el valor actual es el esperado, cámbialo al nuevo — de forma atómica"

func incrementarConCAS(x *int64) {
    for {
        viejo := atomic.LoadInt64(x)
        nuevo := viejo + 1
        if atomic.CompareAndSwapInt64(x, viejo, nuevo) {
            return  // éxito — nadie cambió x entre Load y CAS
        }
        // alguien cambió x — reintentar con el valor nuevo
    }
}
// Este patrón se llama "optimistic locking" o "CAS loop"
```

**Por qué el CAS loop no es un livelock:**

```
A diferencia del livelock de Sección 1.3, el CAS loop progresa globalmente:
si A falla en CAS, es porque B tuvo éxito.
En cada iteración del sistema, al menos una goroutine avanza.
Esto se llama "lock-free" — no libre de bloqueo individual,
sino que el sistema como un todo siempre progresa.
```

---

### Ejercicio 2.3.1 — Implementar estructuras con CAS

**Enunciado:** Implementa estas tres operaciones usando solo CAS (sin mutex):

1. `Incrementar(x *int64)` — incremento atómico
2. `Max(x *int64, candidato int64)` — actualiza x solo si candidato > x
3. `SetSiNoCambia(x *int64, esperado, nuevo int64) bool` — CAS básico con reintento

Para cada una, explica cuántas veces puede iterar el loop en el peor caso y si eso es un problema.

**Restricciones:** Sin `sync.Mutex`. Verifica con `-race`.

**Pista:** `Max` atómico es interesante: no hay una instrucción de CPU para "atomic max". Se implementa con un CAS loop: carga el valor actual, calcula el máximo, intenta hacer CAS. Si falla (otro goroutine cambió el valor), recalcula con el nuevo valor.

**Implementar en:** Go · Java (`AtomicLong`) · Rust (`AtomicI64`) · C# (`Interlocked`)

---

### Ejercicio 2.3.2 — Contador de referencia (reference counting)

**Enunciado:** Implementa un sistema de reference counting para recursos costosos (como conexiones a BD). El recurso se crea cuando el primer cliente lo pide y se destruye cuando el último lo libera.

```go
type Recurso struct {
    refs int64  // contador de referencias
    // ... datos del recurso
}

func (r *Recurso) Adquirir() { /* incrementar refs atómicamente */ }
func (r *Recurso) Liberar()  { /* decrementar; si llega a 0, destruir */ }
```

**Restricciones:** `Liberar` debe destruir el recurso exactamente una vez — ni más ni menos. Verifica con 100 goroutines adquiriendo y liberando simultáneamente.

**Pista:** La condición de carrera sutil está en "si llega a 0, destruir": dos goroutines pueden decrementar de 1 a 0 simultáneamente si no se usa atómico. Con `atomic.AddInt64`, solo una puede ver el valor 0 — la que hizo la operación que llevó el contador a 0.

**Implementar en:** Go · Java · Rust (el ownership de Rust hace esto automático con `Arc`) · C#

---

### Ejercicio 2.3.3 — Spinlock implementado con CAS

**Enunciado:** Un spinlock es un mutex que "espera activamente" (spinning) en lugar de bloquear el hilo. Implementa un spinlock usando CAS:

```go
type Spinlock struct {
    estado int32  // 0 = libre, 1 = bloqueado
}

func (s *Spinlock) Lock() {
    for !atomic.CompareAndSwapInt32(&s.estado, 0, 1) {
        // spinning — CPU al 100%
    }
}

func (s *Spinlock) Unlock() {
    atomic.StoreInt32(&s.estado, 0)
}
```

Mide el rendimiento del spinlock vs `sync.Mutex` para secciones críticas de:
- 10 nanosegundos (operación muy corta)
- 100 microsegundos (operación media)
- 1 milisegundo (operación larga)

**Restricciones:** El spinlock solo es apropiado en el primer caso. Explica por qué usando los datos del benchmark.

**Pista:** Para secciones críticas largas, el spinlock desperdicia CPU esperando — es peor que el mutex que cede el procesador. Para secciones muy cortas, el overhead del mutex (syscall para dormir/despertar) supera el costo del spinning.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 2.3.4 — ABA problem: la trampa del CAS

**Enunciado:** El problema ABA es una trampa clásica del CAS: el valor va de A a B y vuelve a A entre el Load y el CAS. El CAS tiene éxito aunque el estado cambió.

Construye un ejemplo concreto donde el ABA problem causa un bug en una pila lock-free:

```go
// Pila lock-free con CAS
type Nodo struct {
    valor int
    sig   *Nodo
}

var cabeza atomic.Pointer[Nodo]

func Push(valor int) {
    nuevo := &Nodo{valor: valor}
    for {
        viejo := cabeza.Load()
        nuevo.sig = viejo
        if cabeza.CompareAndSwap(viejo, nuevo) {
            return
        }
    }
}

func Pop() (int, bool) {
    for {
        viejo := cabeza.Load()
        if viejo == nil {
            return 0, false
        }
        nuevo := viejo.sig
        if cabeza.CompareAndSwap(viejo, nuevo) {
            return viejo.valor, true
        }
    }
}
```

**Restricciones:** Describe el escenario ABA concreto para esta pila. Luego implementa la solución: tagged pointer (añadir un contador de versión al puntero).

**Pista:** Escenario ABA en la pila: A = [1, 2, 3]. Goroutine 1 lee cabeza = nodo(1). Goroutine 2 hace Pop(), Pop(), Push(1) — la cabeza vuelve a ser nodo(1) pero el nodo(2) fue liberado. Goroutine 1 hace CAS exitoso (la cabeza sigue siendo nodo(1)) pero nodo(1).sig ahora apunta a memoria liberada.

**Implementar en:** Go · Java (`AtomicStampedReference`) · C# · Rust

---

### Ejercicio 2.3.5 — Cuándo NO usar atómicos

**Enunciado:** Los atómicos tienen limitaciones que los hacen inapropiados en ciertos casos. Para cada escenario, determina si un atómico es suficiente o si se necesita un mutex:

1. Incrementar un contador — solo una variable
2. Incrementar dos contadores que deben mantenerse consistentes entre sí
3. Leer una struct de 32 bytes de forma atómica
4. Implementar una cola concurrente con múltiples productores y consumidores
5. Actualizar un mapa (map[string]int) concurrentemente

**Restricciones:** Para cada caso donde el atómico no es suficiente, explica el escenario de fallo.

**Pista:** Los atómicos operan sobre **una variable** de forma indivisible. Cuando la invariante involucra **múltiples variables** o **estructuras de datos complejas**, el mutex es necesario porque solo él puede hacer atómica una secuencia de operaciones.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 2.4 — Canales como Sincronización

En Go, los canales son la herramienta preferida para **comunicar valores
entre goroutines**. Pero también son un mecanismo de sincronización.

**Las dos semánticas de un canal:**

```go
// 1. COMUNICAR un valor (el uso más común)
resultado := <-ch  // goroutine B recibe lo que A envió

// 2. SINCRONIZAR sin valor (señalizar)
done := make(chan struct{})
go func() {
    trabajar()
    close(done)  // señal: terminé
}()
<-done  // esperar la señal
```

**Canal sin buffer vs canal con buffer:**

```go
// SIN BUFFER — el emisor bloquea hasta que alguien recibe
ch := make(chan int)
// send y receive deben sincronizarse — uno espera al otro

// CON BUFFER — el emisor puede enviar hasta llenar el buffer
ch := make(chan int, 10)
// el emisor solo bloquea cuando el buffer está lleno

// Implicaciones para happens-before:
// Sin buffer: send → happens-before → receive (garantizado)
// Con buffer: el k-ésimo receive → happens-before → el (k+cap)-ésimo send
```

**El patrón más importante: done channel**

```go
func trabajadorConCancelacion(done <-chan struct{}, tareas <-chan Tarea) {
    for {
        select {
        case tarea := <-tareas:
            procesar(tarea)
        case <-done:
            return  // cancelación solicitada
        }
    }
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// En Go moderno, context.Context reemplaza el done channel manual
```

**El modelo de memoria de los canales:**

```
Canal sin buffer:
  ch <- valor  →  happens-before  →  v := <-ch

Canal con buffer capacidad C:
  el k-ésimo receive  →  happens-before  →  el (k+C)-ésimo send

close(ch)  →  happens-before  →  cualquier receive que retorna el zero value
```

---

### Ejercicio 2.4.1 — Pipeline de canales

**Enunciado:** Implementa un pipeline de tres etapas usando canales:

```
generador → multiplicador → filtro → colector
```

- `generador`: produce números 1..N en un canal
- `multiplicador`: multiplica cada número por 2
- `filtro`: pasa solo los divisibles por 3
- `colector`: suma todos los valores recibidos

**Restricciones:** Cada etapa es una goroutine separada. El pipeline debe cancelarse limpiamente cuando el colector ya tiene suficientes valores (primeros K resultados).

**Pista:** La cancelación limpia es el reto real. Si el colector para de leer, el filtro se bloquea intentando enviar, el multiplicador se bloquea, y el generador también. Usa un `done` channel o `context.Context` para señalizar la cancelación upstream.

**Implementar en:** Go · Java (CompletableFuture pipeline) · Rust (iteradores + rayon)

---

### Ejercicio 2.4.2 — Fan-out y fan-in con canales

**Enunciado:** Dado un canal de tareas, distribúyelas a `N` workers (fan-out) y recoge sus resultados en un solo canal (fan-in):

```
                    ┌─ worker 1 ─┐
tareas ──── fan-out ┼─ worker 2 ─┼── fan-in ──── resultados
                    └─ worker 3 ─┘
```

**Restricciones:** El fan-in debe cerrar el canal de resultados cuando todos los workers terminan. Usa `sync.WaitGroup` para la coordinación. Maneja cancelación: si el contexto se cancela, los workers deben terminar limpiamente.

**Pista:** Este es el patrón del Cap.17 del repo de algoritmos (fan-out/fan-in) implementado con los primitivos de concurrencia de Go en lugar de `asyncio.gather`.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 2.4.3 — Semáforo con canal con buffer

**Enunciado:** Un canal con buffer de capacidad `N` puede usarse como semáforo: limita a `N` goroutines ejecutando simultáneamente.

```go
sem := make(chan struct{}, N)

func trabajadorLimitado() {
    sem <- struct{}{}   // adquirir
    defer func() { <-sem }()  // liberar
    trabajar()
}
```

Implementa un pool de conexiones a BD usando este patrón: máximo `10` conexiones simultáneas, las goroutines adicionales esperan.

**Restricciones:** El semáforo debe soportar `TryAcquire(timeout)` — intenta adquirir, pero si no hay slot disponible en `timeout`, retorna `false`. Implementa esto con `select` y `time.After`.

**Pista:** `select` con múltiples cases permite "probar" un canal sin bloquearse: `select { case sem <- struct{}{}: // adquirido case <-time.After(timeout): // timeout }`.

**Implementar en:** Go · Java (`Semaphore`) · Python (`asyncio.Semaphore`) · Rust · C#

---

### Ejercicio 2.4.4 — Timeout y context.Context

**Enunciado:** Implementa una función `BuscarConTimeout(ctx context.Context, query string) (Result, error)` que:
1. Lanza la búsqueda en una goroutine
2. Retorna el resultado si llega antes del timeout del contexto
3. Cancela la búsqueda y retorna `ctx.Err()` si el contexto se cancela

```go
func BuscarConTimeout(ctx context.Context, query string) (Result, error) {
    ch := make(chan Result, 1)
    go func() {
        // búsqueda costosa
        ch <- buscar(query)
    }()
    select {
    case resultado := <-ch:
        return resultado, nil
    case <-ctx.Done():
        return Result{}, ctx.Err()
    }
}
```

**Restricciones:** La goroutine de búsqueda debe poder detectar la cancelación y terminar limpiamente — no quedarse ejecutando aunque nadie lea su resultado.

**Pista:** El canal `ch` tiene buffer 1 para que la goroutine pueda enviar aunque el llamador ya canceló — evita el goroutine leak. La goroutine de búsqueda debe recibir el `ctx` y hacer `select` sobre `ctx.Done()` en los puntos de cancelación.

**Implementar en:** Go · Java (`CompletableFuture.orTimeout`) · Python (`asyncio.wait_for`) · C# (`CancellationToken`)

---

### Ejercicio 2.4.5 — Canal vs mutex: cuándo usar cada uno

**Enunciado:** Para cada escenario, elige canal o mutex y justifica:

1. Un contador de visitas a una página web (solo se incrementa y lee)
2. Una caché de resultados de cálculos costosos (lectura y escritura)
3. Coordinar el inicio de N goroutines simultáneamente
4. Pasar un resultado de una goroutine a otra
5. Proteger el acceso a un archivo de log
6. Implementar un rate limiter (máximo N requests por segundo)
7. Notificar a múltiples goroutines que ocurrió un evento

**Restricciones:** Para los casos donde la respuesta no es obvia, implementa ambas versiones y compara la legibilidad y el rendimiento.

**Pista:** La guía de Go dice: "Use channels when passing ownership of data, use mutexes when protecting internal state." El canal es para comunicación — transferir un valor con su ownership. El mutex es para exclusión — proteger acceso a estado compartido.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 2.5 — sync.WaitGroup y sync.Once

Dos primitivas de sincronización que resuelven patrones específicos
de forma más expresiva que un mutex manual.

**WaitGroup — esperar a que un grupo de goroutines termine:**

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        trabajar(n)
    }(i)
}

wg.Wait()  // bloquea hasta que todas las goroutines llamen Done()

// El modelo de memoria:
// wg.Done() (n veces) → happens-before → wg.Wait() retorna
// Garantiza que main ve todos los efectos de las 10 goroutines
```

**Los tres errores comunes con WaitGroup:**

```go
// Error 1: Add dentro de la goroutine (race condition)
go func() {
    wg.Add(1)  // MAL: puede ejecutarse después de Wait()
    defer wg.Done()
}()

// Error 2: Add después de Wait
wg.Wait()
wg.Add(1)  // MAL: Wait ya retornó

// Error 3: reutilizar WaitGroup antes de que llegue a cero
wg.Add(5)
// ... algunas goroutines llaman Done() ...
wg.Add(3)  // PELIGROSO si el contador aún no llegó a cero del Add anterior
```

**Once — ejecutar algo exactamente una vez:**

```go
var once sync.Once
var instancia *Servicio

func ObtenerServicio() *Servicio {
    once.Do(func() {
        instancia = &Servicio{} // se ejecuta exactamente una vez
        instancia.Inicializar() // aunque 100 goroutines llamen ObtenerServicio()
    })
    return instancia
}

// El modelo de memoria:
// once.Do(f) retorna → happens-before → cualquier llamada posterior a once.Do retorna
// Garantiza que todas las goroutines ven el resultado completamente inicializado
```

---

### Ejercicio 2.5.1 — WaitGroup: patrones correctos e incorrectos

**Enunciado:** Implementa un sistema que procesa una lista de URLs en paralelo y recolecta los resultados. Comienza con la versión incorrecta (Add dentro de la goroutine) y demuestra la race condition. Luego arréglala.

**Restricciones:** El sistema debe manejar correctamente URLs que fallan — la goroutine llama `Done()` incluso cuando hay error.

**Pista:** `defer wg.Done()` garantiza que Done se llama incluso si la goroutine termina con panic. Sin defer, un panic en la goroutine dejaría el WaitGroup desbalanceado y Wait() esperaría para siempre.

**Implementar en:** Go · Java (`CountDownLatch`) · Python (`threading.Barrier`) · C# (`CountdownEvent`)

---

### Ejercicio 2.5.2 — Once para inicialización lazy thread-safe

**Enunciado:** Implementa una conexión a base de datos con inicialización lazy: la primera vez que se necesita, se establece la conexión (operación costosa). Las llamadas siguientes reutilizan la conexión existente. Todo esto debe ser thread-safe.

**Restricciones:** La conexión debe establecerse exactamente una vez aunque 1000 goroutines la soliciten simultáneamente. Si la inicialización falla, las llamadas siguientes deben intentarlo de nuevo (Once no reinicia si el primer Do falla — necesitas otro patrón).

**Pista:** `sync.Once` no tiene mecanismo de retry si la función pasa a `Do` falla. Para initialization con posible fallo y retry, usa un patrón con mutex y un flag de "inicializado correctamente".

**Implementar en:** Go · Java (`volatile` + DCL) · Python (`threading.Lock`) · Rust (`OnceLock`) · C# (`Lazy<T>`)

---

### Ejercicio 2.5.3 — Barrera de sincronización

**Enunciado:** Una barrera hace que N goroutines esperen hasta que todas lleguen a un punto antes de continuar. Implementa una barrera reutilizable usando `sync.WaitGroup` o canales.

```go
barrier := NewBarrier(5)

for i := 0; i < 5; i++ {
    go func(id int) {
        fase1(id)
        barrier.Wait()  // todas esperan aquí
        fase2(id)       // todas comienzan fase2 simultáneamente
    }(i)
}
```

**Restricciones:** La barrera debe ser reutilizable — después de que todas las goroutines pasan, puede usarse para la siguiente ronda. Verifica que funciona correctamente para 3 rondas consecutivas.

**Pista:** Go no tiene una barrera reutilizable en `sync`. El `WaitGroup` es de un solo uso — cuando llega a cero, está "gastado". Para una barrera reutilizable, necesitas alternar entre dos WaitGroups o usar un mecanismo de generación (generation counter).

**Implementar en:** Go · Java (`CyclicBarrier`) · Python (`threading.Barrier`) · Rust · C#

---

### Ejercicio 2.5.4 — Implementar sync.WaitGroup desde cero

**Enunciado:** Implementa `WaitGroup` usando solo `sync/atomic` y canales. Tu implementación debe pasar los mismos tests que la estándar y comportarse igual ante los tres errores comunes del inicio de la sección.

**Restricciones:** Sin usar `sync.WaitGroup` internamente. Sin usar `sync.Mutex`. Solo atómicos y canales.

**Pista:** El estado interno es un contador atómico. `Add(n)` incrementa, `Done()` decrementa (equivalente a `Add(-1)`). `Wait()` bloquea hasta que el contador llegue a cero. El desafío es hacer `Wait()` eficiente — no espera activa.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 2.5.5 — Timeout sobre WaitGroup

**Enunciado:** `wg.Wait()` bloquea para siempre si alguna goroutine no llama `Done()`. Implementa `WaitWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool` que retorna `true` si todas terminaron, o `false` si se agotó el timeout.

**Restricciones:** La función no debe modificar el WaitGroup ni tener goroutine leaks.

**Pista:**
```go
func WaitWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()
    select {
    case <-done:
        return true
    case <-time.After(timeout):
        return false
    }
}
```
La goroutine interna puede quedar esperando si el timeout dispara antes. Para evitar el leak, la goroutine necesita una forma de cancelarse — pero wg.Wait() no es cancelable.

**Implementar en:** Go · Java · C# · Python

---

## Sección 2.6 — Condition Variables

Las condition variables resuelven el problema de "esperar hasta que
una condición sea verdadera" sin espera activa (polling).

**El patrón clásico:**

```go
var mu sync.Mutex
var cond = sync.NewCond(&mu)
var datos []int

// Productor
go func() {
    mu.Lock()
    datos = append(datos, 42)
    cond.Signal()   // despierta a un waiter
    mu.Unlock()
}()

// Consumidor
mu.Lock()
for len(datos) == 0 {   // loop — no if — porque pueden haber spurious wakeups
    cond.Wait()          // libera mu, duerme, re-adquiere mu al despertar
}
dato := datos[0]
datos = datos[1:]
mu.Unlock()
```

**Por qué `for` y no `if`:**

```
cond.Wait() puede retornar aunque nadie haya llamado Signal() — spurious wakeup.
El sistema operativo puede despertar un thread por razones internas.
El loop garantiza que la condición se verifica de nuevo antes de continuar.
Regla: SIEMPRE verificar la condición en un loop, nunca en un if.
```

**Signal vs Broadcast:**

```go
cond.Signal()    // despierta exactamente un waiter (cualquiera)
cond.Broadcast() // despierta TODOS los waiters

// Usar Signal cuando: solo un waiter puede progresar (productor/consumidor)
// Usar Broadcast cuando: todos los waiters pueden progresar (cambio de estado global)
```

---

### Ejercicio 2.6.1 — Cola bloqueante (blocking queue)

**Enunciado:** Implementa una cola bloqueante con capacidad máxima:
- `Put(valor)`: bloquea si la cola está llena hasta que haya espacio
- `Get()`: bloquea si la cola está vacía hasta que haya un elemento
- `PutWithTimeout(valor, timeout)`: intenta poner, retorna false si se agota el timeout

**Restricciones:** Implementa con condition variables (no con un canal con buffer — eso sería directo en Go). El propósito es entender las condition variables, no la solución más idiomática de Go.

**Pista:** Necesitas dos condition variables: `notFull` (para los productores que esperan) y `notEmpty` (para los consumidores que esperan). O una sola `cond` con `Broadcast()` cuando cambia el estado.

**Implementar en:** Go · Java (`ArrayBlockingQueue` como referencia) · Python · C#

---

### Ejercicio 2.6.2 — Monitor pattern

**Enunciado:** El Monitor Pattern combina un mutex y una condition variable en un solo objeto. Implementa un monitor para un "estacionamiento" con N plazas:

```go
type Estacionamiento struct {
    capacidad int
    ocupado   int
    mu        sync.Mutex
    cond      *sync.Cond
}

func (e *Estacionamiento) Entrar()  { /* espera si lleno */ }
func (e *Estacionamiento) Salir()   { /* libera plaza y notifica */ }
func (e *Estacionamiento) Ocupacion() int { /* lectura segura */ }
```

**Restricciones:** El invariante `0 <= ocupado <= capacidad` debe mantenerse siempre. Verifica con 100 goroutines entrando y saliendo simultáneamente.

**Pista:** Usa `Broadcast()` en `Salir()` o `Signal()`. ¿Cuál es más correcto aquí? Con `Signal()`, solo un coche que espera se despierta por cada salida — correcto si la plaza liberada es para un solo coche. Con `Broadcast()`, todos los que esperan se despiertan y compiten — más overhead pero más seguro ante spurious wakeups.

**Implementar en:** Go · Java (`synchronized` + `wait`/`notify`) · Python · C# (`Monitor.Wait`/`Pulse`)

---

### Ejercicio 2.6.3 — Condition variable vs canal: cuándo usar cada uno

**Enunciado:** Implementa el mismo problema (cola bloqueante de capacidad N) con:
1. `sync.Cond` + `sync.Mutex`
2. Canal con buffer de tamaño N

Compara: legibilidad, rendimiento, y facilidad de añadir timeout.

**Restricciones:** Ambas versiones deben tener exactamente la misma interfaz externa y el mismo comportamiento observable.

**Pista:** En Go, la versión con canal es generalmente preferida para este caso — es más idiomática y más simple. `sync.Cond` es útil cuando la condición de espera es más compleja que "el canal está vacío/lleno" — por ejemplo, "el valor en la cola es mayor que X".

**Implementar en:** Go · Java · Python

---

### Ejercicio 2.6.4 — Barrera reutilizable con condition variable

**Enunciado:** Implementa la barrera reutilizable del Ejercicio 2.5.3 usando `sync.Cond` en lugar de canales. Compara la implementación.

**Restricciones:** Usa un contador de generación para manejar la reutilización. El patrón es: cuando el N-ésimo goroutine llega a la barrera, incrementa la generación y hace `Broadcast()`. Los que esperaban verifican que la generación cambió.

**Pista:** El contador de generación resuelve el spurious wakeup: si un goroutine se despierta antes de tiempo, verifica que la generación no cambió y vuelve a esperar. Cuando sí cambia, todos saben que la barrera fue liberada.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 2.6.5 — Problema productor-consumidor completo

**Enunciado:** Implementa el problema clásico productor-consumidor con:
- N productores generando ítems
- M consumidores procesando ítems
- Buffer de capacidad K
- Terminación limpia: cuando todos los productores terminan, los consumidores procesan lo restante y terminan

**Restricciones:** No hay goroutine leaks — todas las goroutines terminan. El buffer se drena completamente antes de que los consumidores terminen.

**Pista:** La terminación limpia es el reto: los consumidores necesitan saber cuándo los productores terminaron Y el buffer está vacío. Un patrón: los productores decrementan un contador atómico; cuando llega a cero, hacen `Broadcast()`. Los consumidores verifican tanto el buffer como el contador.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 2.7 — Elegir la Herramienta Correcta

Con cinco herramientas disponibles (mutex, RWMutex, atómico, canal, cond),
la pregunta correcta no es "¿cuál es mejor?" sino "¿cuál resuelve este problema?"

**El árbol de decisión:**

```
¿Necesito transferir un valor entre goroutines?
    Sí → Canal

¿Necesito que múltiples goroutines esperen un evento?
    Sí → sync.Cond con Broadcast, o close(done)

¿Necesito ejecutar algo exactamente una vez?
    Sí → sync.Once

¿Necesito esperar a que N goroutines terminen?
    Sí → sync.WaitGroup

¿Necesito proteger acceso a estado compartido?
    ¿Es mayoritariamente lectura (>80%)?
        Sí → sync.RWMutex
        No → sync.Mutex
    ¿Es una sola variable numérica?
        Sí → sync/atomic
        No → sync.Mutex
```

**La tabla de características:**

```
Herramienta    Overhead    Uso principal                   Cuándo NO usar
───────────    ─────────   ─────────────────────────────   ────────────────────────
Mutex          ~20-300ns   Proteger estado compartido      Secciones críticas largas
RWMutex        ~40-400ns   Estado con muchas lecturas      Escrituras frecuentes
Atómico        ~10ns       Un contador o puntero           Múltiples variables juntas
Canal          ~50-200ns   Comunicar valores               Estado compartido complejo
Cond           ~50-100ns   Esperar condición compleja      Cuando un canal es suficiente
Once           ~5ns post   Inicialización lazy             —
WaitGroup      ~50ns       Esperar N goroutines            —
```

---

### Ejercicio 2.7.1 — Refactorizar código con la herramienta incorrecta

**Enunciado:** Este código usa mutex para todo. Identifica qué partes deberían usar una herramienta diferente y refactoriza:

```go
type Sistema struct {
    mu        sync.Mutex
    contador  int
    resultados []int
    terminado bool
}

func (s *Sistema) Incrementar() {
    s.mu.Lock()
    s.contador++
    s.mu.Unlock()
}

func (s *Sistema) AgregarResultado(r int) {
    s.mu.Lock()
    s.resultados = append(s.resultados, r)
    s.mu.Unlock()
}

func (s *Sistema) EsperarTerminado() {
    for {
        s.mu.Lock()
        t := s.terminado
        s.mu.Unlock()
        if t { return }
        time.Sleep(10 * time.Millisecond)
    }
}

func (s *Sistema) Terminar() {
    s.mu.Lock()
    s.terminado = true
    s.mu.Unlock()
}
```

**Restricciones:** Cada campo del struct debe usar la herramienta más apropiada. `EsperarTerminado` no debe usar polling.

**Pista:** `contador` → atómico, `terminado` → canal `done` cerrado con `close()`, `resultados` → puede quedarse con mutex o usar un canal. `EsperarTerminado` con polling es un anti-patrón — el `time.Sleep` añade latencia innecesaria.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 2.7.2 — Diseñar la sincronización de un sistema nuevo

**Enunciado:** Diseña la sincronización para un servidor de caché en memoria (estilo Redis simplificado) con estas operaciones:
- `GET(clave)` → lectura frecuente
- `SET(clave, valor, ttl)` → escritura menos frecuente
- `SUBSCRIBE(patron)` → registrar callback para cambios
- `PUBLISH(clave, valor)` → notificar a suscriptores
- `EXPIRE()` → goroutine de fondo que elimina claves expiradas

Para cada operación, elige la herramienta de sincronización y justifica.

**Restricciones:** El diseño debe ser eficiente para 10,000 operaciones/segundo con 100 goroutines concurrentes.

**Pista:** `GET`/`SET` sugieren RWMutex. `SUBSCRIBE`/`PUBLISH` sugieren un patrón de broadcast — canal o cond. `EXPIRE` es una goroutine de fondo que necesita acceso al mismo estado que GET/SET — hay que pensar cómo coordinarla sin crear contención.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 2.7.3 — Code review de sincronización

**Enunciado:** Analiza este código de producción y lista todos los problemas de sincronización, clasificándolos por herramienta incorrecta, herramienta correcta mal usada, y herramienta correcta bien usada:

```go
type ServicioMetricas struct {
    mu           sync.RWMutex
    contadores   map[string]int64
    histogramas  map[string][]float64
    ultimaLimpieza time.Time
}

func (s *ServicioMetricas) Incrementar(nombre string) {
    s.mu.Lock()
    s.contadores[nombre]++
    s.mu.Unlock()
}

func (s *ServicioMetricas) Registrar(nombre string, valor float64) {
    s.mu.Lock()
    s.histogramas[nombre] = append(s.histogramas[nombre], valor)
    if time.Since(s.ultimaLimpieza) > time.Hour {
        s.limpiar()   // operación costosa dentro del lock
        s.ultimaLimpieza = time.Now()
    }
    s.mu.Unlock()
}

func (s *ServicioMetricas) ObtenerContador(nombre string) int64 {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.contadores[nombre]
}

func (s *ServicioMetricas) ObtenerHistograma(nombre string) []float64 {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.histogramas[nombre]  // ← retorna referencia interna
}
```

**Restricciones:** Para cada problema, propón la corrección mínima que lo resuelve.

**Pista:** Hay al menos 4 problemas. Uno es de rendimiento (Lock donde RLock es suficiente o viceversa), uno es de seguridad (retornar referencia interna bajo RLock), uno es de diseño (operación costosa dentro del lock), y uno es más sutil.

**Implementar en:** Go · Java · Python

---

### Ejercicio 2.7.4 — Medir el costo de cada herramienta

**Enunciado:** Construye un benchmark que mida el costo de cada herramienta de sincronización en nanosegundos para la operación más simple posible:
- `sync.Mutex`: Lock + Unlock sin contención
- `sync.RWMutex`: RLock + RUnlock sin contención
- `atomic.AddInt64`: una operación
- Canal sin buffer: send + receive
- Canal con buffer (no lleno): send + receive

Mide con `GOMAXPROCS=1` y con `GOMAXPROCS=runtime.NumCPU()`.

**Restricciones:** Usa `testing.B` para los benchmarks. Reporta ns/op y ops/s.

**Pista:** Los resultados con `GOMAXPROCS=1` muestran el overhead sin contención real. Con `GOMAXPROCS=N`, el mutex puede ser más caro porque el sistema operativo tiene que coordinar entre núcleos físicos (cache coherency protocol).

**Implementar en:** Go · Java (JMH) · Rust (criterion)

---

### Ejercicio 2.7.5 — El sistema completo del Cap.17 con sincronización correcta

**Enunciado:** Toma el sistema de análisis de grafos del Cap.17 §7 del repo de algoritmos y agrega sincronización correcta para hacerlo thread-safe. El sistema debe soportar:
- Múltiples requests concurrentes de análisis
- Caché de grafos compartida entre requests
- Métricas concurrentes (contador de requests, tiempo promedio)
- Cancelación de requests largos

Para cada componente, documenta qué herramienta usas y por qué.

**Restricciones:** Verifica con `go test -race`. El throughput debe escalar con el número de goroutines hasta el límite de la CPU.

**Pista:** La caché compartida es el componente más complejo: lectura frecuente (RWMutex o RCU), escritura infrecuente, e invalidación cuando el grafo cambia. Las métricas concurrentes son simples: atómicos. La cancelación es context.Context.

**Implementar en:** Go · Java · Python · Rust

---

## Resumen del capítulo

| Herramienta | Garantía happens-before | Uso principal | Overhead |
|---|---|---|---|
| `sync.Mutex` | Unlock → Lock siguiente | Proteger estado compartido complejo | ~20-300ns |
| `sync.RWMutex` | RUnlock/Unlock → Lock siguiente | Estado con muchas lecturas | ~40-400ns |
| `sync/atomic` | Barrera de memoria implícita | Una variable numérica o puntero | ~10ns |
| Canal sin buffer | Send → Receive | Comunicar valor + sincronizar | ~50ns |
| Canal con buffer C | k-ésimo Receive → (k+C)-ésimo Send | Comunicar con desacoplamiento | ~50-200ns |
| `sync.Cond` | Signal/Broadcast → Wait retorna | Esperar condición compleja | ~50-100ns |
| `sync.Once` | Do retorna → Do siguiente retorna | Inicialización lazy exactamente una vez | ~5ns post-init |
| `sync.WaitGroup` | Done×N → Wait retorna | Esperar N goroutines | ~50ns |

## La pregunta que guía el Cap.03

> Ahora que sabemos **cómo** sincronizar, ¿cómo organizamos código concurrente
> para que sea correcto, legible, y mantenible?

Los locks protegen variables. Los patrones organizan el código.
El Cap.03 cubre los patrones clásicos: productor/consumidor,
lectores/escritores, worker pool, y pipeline — los mismos problemas
que aparecen una y otra vez en entrevistas y en producción.
