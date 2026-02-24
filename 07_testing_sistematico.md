# Guía de Ejercicios — Testing Sistemático de Código Concurrente

> Implementar cada ejercicio en: **Go · Java · Python · Rust · C#**
> según corresponda.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el test que pasa y el bug que existe

```go
// Este test pasa. El bug existe.

func TestSaldo(t *testing.T) {
    cuenta := &Cuenta{saldo: 1000}

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            cuenta.Retirar(10)
        }()
    }
    wg.Wait()

    // Esperamos 0, a veces obtenemos 0... y a veces algo positivo.
    // El test pasa el 70% de las veces. Falla el 30%.
    // En CI, donde la máquina está cargada, falla menos.
    // En producción, donde hay más carga, falla más.
    if cuenta.Saldo() != 0 {
        t.Errorf("saldo esperado 0, obtenido %d", cuenta.Saldo())
    }
}
```

Un test de concurrencia que falla aleatoriamente es peor que no tener test.
Da falsa confianza cuando pasa, y falsa alarma cuando falla por timing.

**El problema no es el test — es la estructura del test.**
Un test correcto de concurrencia debe:

```
1. Reproducir el bug de forma determinista (no depender de timing)
2. Verificar invariantes, no valores específicos
3. Cubrir todos los estados concurrentes posibles, no solo los más probables
4. Terminar en tiempo acotado (timeout explícito)
5. Limpiar todos los recursos al terminar (sin goroutine leaks)
```

Este capítulo enseña cómo construir tests que cumplan estas cinco propiedades.

---

## Tabla de contenidos

- [Sección 7.1 — Tests que encuentran bugs determinísticamente](#sección-71--tests-que-encuentran-bugs-deterministicamente)
- [Sección 7.2 — Property-based testing para concurrencia](#sección-72--property-based-testing-para-concurrencia)
- [Sección 7.3 — Model checking: explorar todos los estados](#sección-73--model-checking-explorar-todos-los-estados)
- [Sección 7.4 — Benchmarks que no mienten](#sección-74--benchmarks-que-no-mienten)
- [Sección 7.5 — Testing de integración: el sistema completo](#sección-75--testing-de-integración-el-sistema-completo)
- [Sección 7.6 — Diseñar para testabilidad](#sección-76--diseñar-para-testabilidad)
- [Sección 7.7 — Regresiones de concurrencia en CI/CD](#sección-77--regresiones-de-concurrencia-en-cicd)

---

## Sección 7.1 — Tests que Encuentran Bugs Determinísticamente

El objetivo: convertir un test flaky (pasa a veces, falla a veces)
en un test que falla siempre cuando el bug existe.

**Las cuatro técnicas:**

```
1. Barrera de inicio:   todas las goroutines empiezan simultáneamente
2. Saturación:          más goroutines que CPUs disponibles
3. GOMAXPROCS=1:        forzar intercalaciones no naturales
4. -race:               el detector encuentra races que los asserts no ven
```

**La barrera de inicio — la técnica más importante:**

```go
// Sin barrera — las goroutines se lanzan secuencialmente:
for i := 0; i < 100; i++ {
    go func() { cuenta.Retirar(10) }()
}
// Las primeras goroutines pueden terminar antes de que las últimas empiecen.
// Poca oportunidad de intercalación.

// Con barrera — todas empiezan al mismo tiempo:
var barrera sync.WaitGroup
barrera.Add(1)  // todas esperan aquí

var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        barrera.Wait()     // esperar la señal de inicio
        cuenta.Retirar(10)
    }()
}

barrera.Done()  // ¡ahora! — todas empiezan simultáneamente
wg.Wait()
```

**El problema: `sync.WaitGroup` no es una barrera bidireccional.**
Para una barrera real (todas esperan hasta que todas lleguen), necesitas
`sync.Cond` o el patrón con canal cerrado:

```go
inicio := make(chan struct{})

for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        <-inicio           // bloqueado hasta que se cierre el canal
        cuenta.Retirar(10)
    }()
}

close(inicio)  // despierta todas las goroutines simultáneamente
```

---

### Ejercicio 7.1.1 — Convertir un test flaky en determinista

**Enunciado:** El test del inicio del capítulo falla aleatoriamente.
Conviértelo en uno que falla siempre cuando hay una race condition,
usando la barrera de canal cerrado.

Verifica que:
1. Con la implementación buggy de `Retirar` (sin sincronización), el test falla siempre
2. Con la implementación correcta, el test pasa siempre
3. `go test -race` reporta la race condition incluso cuando el assertion no falla

**Restricciones:** El test no debe usar `time.Sleep` en ningún punto.
Debe funcionar con `GOMAXPROCS=1` y con `GOMAXPROCS=runtime.NumCPU()`.

**Pista:** Para maximizar la probabilidad de intercalación, usa
`runtime.GOMAXPROCS(runtime.NumCPU())` y lanza al menos
`4 * runtime.NumCPU()` goroutines. Más goroutines que CPUs garantiza
que el scheduler debe intercalarlas.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 7.1.2 — GOMAXPROCS=1 para forzar intercalaciones específicas

**Enunciado:** Con `GOMAXPROCS=1`, Go ejecuta una sola goroutine a la vez
y hace context switches solo en puntos de yield (canales, syscalls, `runtime.Gosched()`).
Esto permite forzar intercalaciones específicas que normalmente son raras.

Implementa una función que inserta `runtime.Gosched()` en puntos específicos
de una sección crítica para maximizar la probabilidad de race condition:

```go
func RetirarBuggy(c *Cuenta, monto int) {
    saldo := c.saldo        // leer
    runtime.Gosched()       // ceder aquí — forzar intercalación
    if saldo >= monto {
        c.saldo = saldo - monto  // escribir
    }
}
```

**Restricciones:** El `Gosched` es solo para testing — no debe estar
en el código de producción. Usa build tags para que compile solo en tests.

**Pista:** Los build tags en Go: `//go:build testing` en el archivo con Gosched,
y correr con `go test -tags testing`. El código de producción no incluye
los Gosched. Los tests sí.

**Implementar en:** Go

---

### Ejercicio 7.1.3 — Stress test: ejecutar N veces en paralelo

**Enunciado:** Un stress test ejecuta el mismo test muchas veces en paralelo
para aumentar la probabilidad de encontrar bugs de timing.
Go tiene `-count=N` para repetir, pero para paralelismo necesitas más:

```go
func TestStress(t *testing.T) {
    if testing.Short() {
        t.Skip("stress test omitido en modo short")
    }

    const (
        iteraciones = 10_000
        goroutines  = 100
    )

    var exitos, fallos int64

    var wg sync.WaitGroup
    for i := 0; i < goroutines; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < iteraciones/goroutines; j++ {
                if err := operacionConcurrente(); err != nil {
                    atomic.AddInt64(&fallos, 1)
                } else {
                    atomic.AddInt64(&exitos, 1)
                }
            }
        }()
    }
    wg.Wait()

    t.Logf("éxitos: %d, fallos: %d", exitos, fallos)
    if fallos > 0 {
        t.Errorf("hubo %d fallos en el stress test", fallos)
    }
}
```

Aplica este patrón al Stack concurrente del Ejercicio 1.1.4.

**Restricciones:** El stress test debe completar en menos de 30 segundos.
Usa `-short` para omitirlo en runs rápidos. El resultado debe ser 0 fallos
con la implementación correcta y > 0 fallos con la buggy.

**Pista:** La combinación `go test -race -count=10 -run TestStress` ejecuta
el stress test 10 veces con el race detector. Si hay un bug, es muy probable
que se manifieste en alguna de las 10 ejecuciones.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 7.1.4 — Testing de invariantes, no de valores

**Enunciado:** Los tests de concurrencia no deben verificar valores específicos
(que dependen del ordering) sino invariantes (que deben ser verdad
independientemente del ordering).

Para el sistema bancario del Cap.06, las invariantes son:
- La suma total de saldos siempre es constante
- Ninguna cuenta tiene saldo negativo (si está prohibido)
- El número de operaciones completadas = operaciones iniciadas

Implementa tests que verifiquen estas invariantes bajo carga concurrente.

**Restricciones:** Las invariantes se verifican después de que todas las
goroutines terminan — no durante la ejecución concurrente (eso requeriría
sincronización adicional que cambiaría el comportamiento).

**Pista:** "La suma total es constante" es una invariante global que requiere
un snapshot consistente para verificarse. Una forma: pausar todas las
transacciones, tomar el snapshot, reanudar. Otra: calcular la suma antes y
después de las transacciones concurrentes — sin medir durante.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 7.1.5 — Deadlock detection en tests

**Enunciado:** Un deadlock en un test cuelga el proceso para siempre.
Go detecta deadlocks globales (todas las goroutines bloqueadas), pero no
detecta deadlocks parciales (algunas goroutines bloqueadas, otras activas).

Implementa un detector de deadlock para tests que falla si alguna goroutine
está esperando más de un timeout configurable:

```go
func TestConDeadlockDetection(t *testing.T, timeout time.Duration, f func(t *testing.T)) {
    done := make(chan struct{})
    go func() {
        f(t)
        close(done)
    }()

    select {
    case <-done:
        // el test terminó normalmente
    case <-time.After(timeout):
        // capturar stack traces de todas las goroutines
        buf := make([]byte, 1<<20)
        n := runtime.Stack(buf, true)
        t.Errorf("timeout después de %s — posible deadlock:\n%s", timeout, buf[:n])
    }
}
```

**Restricciones:** El timeout debe ser razonable — no tan corto que falle
en máquinas lentas, no tan largo que haga el CI lento.
Captura los stack traces para diagnosticar el deadlock.

**Pista:** Go tiene `-timeout` para el test runner completo. Para deadlocks
específicos dentro de un test, el patrón con `time.After` funciona bien.
Una alternativa: `go test -timeout 30s` mata todo el proceso si supera 30s.

**Implementar en:** Go · Java · Python · C#

---

## Sección 7.2 — Property-Based Testing para Concurrencia

El testing basado en propiedades (property-based testing, PBT) genera
inputs aleatorios y verifica que las propiedades del sistema se mantienen
para todos ellos. Para concurrencia, genera secuencias aleatorias de
operaciones concurrentes.

**La idea:**

```go
// En vez de:
func TestCola(t *testing.T) {
    cola := NuevaCola()
    cola.Enqueue(1)
    cola.Enqueue(2)
    assert(cola.Dequeue() == 1)  // solo prueba este caso específico
}

// Con PBT:
// Propiedad: para cualquier secuencia de enqueue/dequeue,
// los elementos salen en orden FIFO
func PropCola(ops []Operacion) bool {
    cola := NuevaCola()
    var esperados []int

    for _, op := range ops {
        switch op.Tipo {
        case "enqueue":
            cola.Enqueue(op.Valor)
            esperados = append(esperados, op.Valor)
        case "dequeue":
            v, ok := cola.Dequeue()
            if ok {
                if v != esperados[0] { return false }  // FIFO violado
                esperados = esperados[1:]
            }
        }
    }
    return true
}
// El framework genera miles de secuencias aleatorias y verifica la propiedad
```

**Herramientas en Go:**

```
gopter:   go get github.com/leanovate/gopter
rapid:    go get pgregory.net/rapid  (más idiomático, recomendado)
```

---

### Ejercicio 7.2.1 — Propiedades de una cola concurrente

**Enunciado:** Define y verifica estas propiedades para una cola FIFO concurrente:

1. **FIFO**: los elementos salen en el orden en que entraron (desde cada productor)
2. **Sin pérdida**: cada elemento encolado eventualmente sale
3. **Sin duplicados**: cada elemento sale exactamente una vez
4. **Bounded**: la longitud nunca supera la capacidad configurada

```go
func TestPropiedadesCola(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        capacidad := rapid.IntRange(1, 100).Draw(t, "capacidad").(int)
        ops := rapid.SliceOf(rapid.Custom(generarOp)).Draw(t, "ops").([]Op)
        verificarPropiedades(t, capacidad, ops)
    })
}
```

**Restricciones:** Las propiedades deben verificarse con operaciones
concurrentes — no secuenciales. Usa una barrera para que todos los
productores/consumidores empiecen simultáneamente.

**Pista:** La propiedad FIFO en un sistema concurrente con múltiples productores
es más débil: el orden global no está garantizado, pero el orden por productor sí.
Si el productor A envía [1, 2, 3], el consumidor debe recibirlos en ese orden
(aunque intercalados con los de otros productores).

**Implementar en:** Go · Java (jqwik) · Python (hypothesis) · Rust (proptest)

---

### Ejercicio 7.2.2 — Propiedades de linearizabilidad

**Enunciado:** La linearizabilidad es la propiedad de corrección más importante
para estructuras de datos concurrentes: cada operación parece ejecutarse
instantáneamente en algún punto entre su inicio y su fin.

Para un registro concurrente (read/write), verifica que:
- Las lecturas siempre retornan el valor del último write que terminó antes
- Nunca se lee un valor que no fue escrito

```go
// Historia de operaciones:
// t=0: W(5) inicia
// t=1: R() inicia
// t=2: W(5) termina → valor escrito: 5
// t=3: R() termina → valor leído: 5  ← linearizable
//               ó → valor leído: 0  ← ¿linearizable? (depende del valor inicial)
```

**Restricciones:** Registra el inicio y fin de cada operación (timestamps).
Verifica que existe un orden lineal de las operaciones que es consistente
con los timestamps y con la semántica secuencial.

**Pista:** Verificar linearizabilidad es NP-completo en general,
pero para estructuras simples (register, queue) existen algoritmos eficientes.
La biblioteca `porcupine` para Go implementa el checker de linearizabilidad.

**Implementar en:** Go (con porcupine) · Java

---

### Ejercicio 7.2.3 — Shrinking: encontrar el caso mínimo que falla

**Enunciado:** Cuando PBT encuentra un fallo, el input generado puede ser
grande y difícil de entender. El "shrinking" reduce automáticamente el input
al caso mínimo que sigue fallando.

```
Input que falla: [op1, op2, op3, ..., op47]  ← difícil de entender
Después de shrinking: [op1, op3]              ← el mínimo que reproduce el fallo
```

Verifica que `rapid` hace shrinking automáticamente para la Cola concurrente
del Ejercicio 7.2.1. Introduce un bug intencional y observa el output del shrinking.

**Restricciones:** El bug intencional debe ser sutil — no un error obvio.
El output del shrinking debe ser lo suficientemente pequeño para entender
el fallo manualmente.

**Pista:** `rapid` hace shrinking automáticamente — no necesitas código adicional.
Cuando el test falla, `rapid` itera reduciendo el input mientras el test sigue
fallando. El resultado final es el caso mínimo que reproduce el fallo.
Esto es una de las diferencias más prácticas entre PBT y testing manual.

**Implementar en:** Go · Python (hypothesis tiene shrinking excelente)

---

### Ejercicio 7.2.4 — Modelo de referencia para verificación

**Enunciado:** Un modelo de referencia es una implementación secuencial
y obvia del mismo sistema. El test verifica que la implementación concurrente
produce los mismos resultados que el modelo de referencia.

```go
type ModeloReferencia struct {
    datos map[string]string
    mu    sync.Mutex  // solo para hacer el modelo thread-safe en el test
}

// Para cada secuencia aleatoria de operaciones:
// 1. Ejecutar en el modelo de referencia (secuencial)
// 2. Ejecutar en la implementación real (concurrente)
// 3. Verificar que el estado final es el mismo
```

Aplica este patrón a la caché del Cap.02 §2.2.1.

**Restricciones:** El modelo de referencia es tan simple como sea posible
— su corrección debe ser obvia. La implementación real puede ser compleja.
El test verifica que ambas producen el mismo estado final.

**Pista:** El estado final incluye: todos los valores almacenados, el tamaño,
y el resultado de cada operación de lectura. Para operaciones concurrentes,
el orden exacto puede variar — verifica el estado final, no el orden de operaciones.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 7.2.5 — Fuzzing con estado en Go 1.18+

**Enunciado:** Go 1.18 introdujo fuzzing nativo. Implementa un fuzz test
para el Worker Pool que genera secuencias arbitrarias de Submit, Cancel,
y Shutdown, y verifica que el pool no entra en estados inválidos.

```go
func FuzzWorkerPool(f *testing.F) {
    f.Add([]byte{0, 1, 0, 2, 3})  // seed: submit, submit, submit, cancel, shutdown
    f.Fuzz(func(t *testing.T, ops []byte) {
        pool := NuevoPool(4)
        defer pool.Detener()

        for _, op := range ops {
            switch op % 4 {
            case 0: pool.Submit(Job{})
            case 1: // cancelar un job pendiente
            case 2: pool.Metricas()
            case 3: return  // shutdown anticipado
            }
            // invariante: pool.Metricas().Activos <= 4
        }
    })
}
```

**Restricciones:** El fuzz test debe correr con `-race`. Las invariantes
verificadas en cada paso no deben requerir locks adicionales — solo lecturas
de métricas atómicas.

**Pista:** El fuzzer genera secuencias de bytes que se mapean a operaciones.
Con el tiempo, el fuzzer descubre secuencias que maximizan la cobertura de código
— incluyendo paths de error poco frecuentes. Para código concurrente, es
especialmente valioso porque puede descubrir interacciones inesperadas.

**Implementar en:** Go

---

## Sección 7.3 — Model Checking: Explorar Todos los Estados

El model checking verifica que un sistema satisface una propiedad en
**todos los estados posibles**, no solo en los que el testing aleatorio alcanza.

**TLA+ y el model checking formal:**

TLA+ es un lenguaje de especificación formal creado por Leslie Lamport.
Amazon, Microsoft y otros lo usan para verificar sistemas distribuidos
y algoritmos concurrentes antes de implementarlos.

```tla
(* Especificación TLA+ del mutex *)
VARIABLES estado, propietario

Invariante == estado = "libre" \/ estado = "ocupado"

Adquirir(proc) ==
    /\ estado = "libre"
    /\ estado' = "ocupado"
    /\ propietario' = proc

Liberar(proc) ==
    /\ estado = "ocupado"
    /\ propietario = proc
    /\ estado' = "libre"
    /\ propietario' = "ninguno"

(* El model checker verifica que estas propiedades se mantienen en TODOS los estados *)
```

**En la práctica para Go — herramientas más accesibles:**

```
go-deadlock:   detecta deadlocks potenciales en tiempo de ejecución
CHESS:         Microsoft — explora todos los schedulings posibles (solo Windows)
ThreadSanitizer: similar al race detector pero más potente
Spin:          model checker para protocolos — versión simplificada
```

---

### Ejercicio 7.3.1 — Especificación informal como documentación ejecutable

**Enunciado:** Antes del model checking formal, escribe la especificación
informal del sistema como comentarios estructurados que pueden verificarse:

```go
// Especificación: Semáforo con capacidad N
//
// Estado:
//   ocupados ∈ [0, N]
//
// Invariante:
//   0 ≤ ocupados ≤ N
//
// Operaciones:
//   Adquirir: ocupados < N → ocupados++ (bloqueante si ocupados == N)
//   Liberar:  ocupados > 0 → ocupados--
//
// Propiedades de safety:
//   Exclusión: en cualquier momento, como mucho N goroutines entre Adquirir y Liberar
//
// Propiedades de liveness:
//   Progress: si hay capacidad, una goroutine bloqueada en Adquirir eventualmente pasa

func verificarInvariante(sem *Semaforo) {
    n := sem.Ocupados()
    if n < 0 || n > sem.Capacidad() {
        panic(fmt.Sprintf("invariante violada: ocupados=%d, capacidad=%d", n, sem.Capacidad()))
    }
}
```

Instrumenta el Semáforo del Cap.02 §2.4.3 con verificación de invariantes
en cada operación, y actívala con una build tag `debug`.

**Restricciones:** La verificación de invariantes no debe cambiar el
comportamiento observable del sistema — solo detectar violaciones.
Con `-tags debug`, el overhead puede ser alto. Sin la tag, cero overhead.

**Pista:** Este patrón de "especificación ejecutable" viene del diseño por
contrato (Bertrand Meyer). Go no tiene soporte nativo, pero se puede simular
con funciones de verificación envueltas en build tags.

**Implementar en:** Go · Java (`assert` con `-ea`) · Python (`assert`) · Rust (`debug_assert!`)

---

### Ejercicio 7.3.2 — go-deadlock: detección de deadlocks en tiempo de ejecución

**Enunciado:** `go-deadlock` reemplaza `sync.Mutex` con una versión
instrumentada que detecta potenciales deadlocks analizando el orden
de adquisición de locks.

```go
import "github.com/sasha-s/go-deadlock"

// En vez de sync.Mutex, usa deadlock.Mutex
// La interfaz es idéntica — drop-in replacement

type Banco struct {
    mu deadlock.Mutex
    // ...
}
```

Instrumenta el sistema bancario del Ejercicio 6.3.1 con go-deadlock
y verifica que detecta el deadlock potencial cuando los locks se adquieren
en orden inconsistente.

**Restricciones:** go-deadlock debe detectar el problema antes de que
ocurra el deadlock real — es un análisis preventivo basado en el grafo
de lock ordering.

**Pista:** go-deadlock mantiene un grafo de "A adquiere B mientras tiene A".
Si detecta un ciclo potencial (A→B y B→A en distintas goroutines), alerta
aunque el deadlock no haya ocurrido aún. Es detección estática en tiempo
de ejecución — más conservadora que el race detector.

**Implementar en:** Go · Java (FindBugs/SpotBugs para análisis estático)

---

### Ejercicio 7.3.3 — Simulación exhaustiva de schedulings con GOMAXPROCS=1

**Enunciado:** Con `GOMAXPROCS=1` y `runtime.Gosched()` estratégicamente
ubicados, se pueden explorar manualmente distintos schedulings de forma
casi exhaustiva para sistemas pequeños.

Implementa un tester que ejecuta el mismo código con todas las combinaciones
de `Gosched()` en N puntos específicos:

```go
// Para N=3 puntos de yield, hay 2^3 = 8 combinaciones
// Para cada combinación, ejecutar el test y verificar la invariante

func TestExhaustivoTransferencia(t *testing.T) {
    for mask := 0; mask < (1 << 3); mask++ {
        cuenta := &Cuenta{saldo: 100}
        transferirConYields(cuenta, 50, mask)  // mask controla qué yields están activos
        if cuenta.Saldo() < 0 {
            t.Errorf("saldo negativo con scheduling mask=%b", mask)
        }
    }
}
```

**Restricciones:** Solo funciona con `GOMAXPROCS=1`. El número de puntos
de yield debe ser pequeño (≤ 10) para que el número de combinaciones sea manejable.

**Pista:** Este es un model checking manual y limitado. Para sistemas más
grandes, herramientas como CHESS (Microsoft) o CDSChecker (para C++) hacen
esto automáticamente explorando todos los schedulings posibles de forma
sistemática — no aleatoria.

**Implementar en:** Go

---

### Ejercicio 7.3.4 — Verificar el modelo de memoria con litmus tests

**Enunciado:** Los "litmus tests" son programas mínimos que verifican
el modelo de memoria de un lenguaje o arquitectura. El litmus test más
famoso para x86:

```
Inicialmente: x = 0, y = 0

Thread 1: x = 1; r1 = y
Thread 2: y = 1; r2 = x

¿Puede ocurrir r1 = 0 y r2 = 0 simultáneamente?
En x86: NO (strong memory model)
En ARM/POWER: SÍ (weak memory model)
En Go con sync/atomic: NO (barrera de memoria implícita)
En Go sin sync/atomic: SÍ (no hay garantía de ordering)
```

Implementa y ejecuta este litmus test en Go para verificar
cuándo las barreras de memoria son necesarias.

**Restricciones:** Ejecutar el test millones de veces y registrar si alguna
vez se observa `r1 = 0 && r2 = 0`. Con variables normales (sin atómico),
el resultado puede ocurrir. Con `sync/atomic`, no debería.

**Pista:** En x86 (la mayoría de máquinas de desarrollo), el modelo de memoria
fuerte hace que el resultado sea imposible incluso sin atómicos. En ARM
(Mac M1/M2, algunos CI), puede observarse. Este test es más revelador
en arquitecturas ARM — considera correrlo en un Mac M1 si tienes acceso.

**Implementar en:** Go · Rust · C#

---

### Ejercicio 7.3.5 — TLA+ para el algoritmo del panadero (Lamport's Bakery)

**Enunciado:** El algoritmo del panadero de Lamport es un mutex sin instrucciones
atómicas. La especificación TLA+ verifica sus propiedades de forma exhaustiva.

Primero, implementa el algoritmo en Go:

```go
// Algoritmo del panadero — exclusión mutua sin hardware atómico
var (
    eligiendo [N]bool
    numero    [N]int
)

func adquirir(id int) {
    eligiendo[id] = true
    maxNumero := 0
    for i := range numero {
        if numero[i] > maxNumero {
            maxNumero = numero[i]
        }
    }
    numero[id] = maxNumero + 1
    eligiendo[id] = false
    // esperar turno...
}
```

Luego, escribe la especificación TLA+ que verifica que el algoritmo
satisface exclusión mutua y es libre de starvation.

**Restricciones:** La implementación Go debe ser verificable con `go run -race`.
La especificación TLA+ debe ser verificable con TLC (el model checker de TLA+).

**Pista:** El algoritmo del panadero es interesante porque usa solo acceso
a variables ordinarias (sin mutex ni atómicos), pero es correcto si el
modelo de memoria del hardware garantiza que las escrituras son visibles
en el orden en que ocurren. En hardware moderno con weak memory models,
necesitaría barreras adicionales.

**Implementar en:** Go + TLA+

---

## Sección 7.4 — Benchmarks que No Mienten

Los benchmarks de concurrencia son fáciles de hacer mal. Un benchmark
incorrecto puede mostrar speedup donde hay slowdown, o viceversa.

**Los cuatro problemas más comunes:**

```
1. Dead code elimination:
   El compilador optimiza el código que no tiene efectos observables.
   Un benchmark que calcula algo sin usarlo puede ser optimizado a nada.
   Solución: usar el valor calculado (sink variable).

2. Cache warming:
   Las primeras iteraciones son más lentas (cache fría).
   El benchmark debe calentar antes de medir.
   Go's testing.B hace esto automáticamente con b.ResetTimer().

3. False sharing en el benchmark mismo:
   Las goroutines del benchmark comparten datos por accidente.
   Resultado: el benchmark mide contención, no el código que se quiere medir.

4. Scheduling overhead vs trabajo real:
   Para secciones críticas muy cortas (<100ns), el overhead del scheduler
   domina el resultado. El benchmark mide el scheduler, no el código.
```

**El benchmark correcto:**

```go
func BenchmarkCola(b *testing.B) {
    cola := NuevaCola(1000)

    b.ResetTimer()  // no incluir la inicialización en la medición
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            cola.Enqueue(runtime.GOMAXPROCS(0))  // usar el valor — no dead code
            v, _ := cola.Dequeue()
            _ = v  // sink — evita que el compilador elimine el Dequeue
        }
    })
}
```

---

### Ejercicio 7.4.1 — Detectar y corregir dead code elimination

**Enunciado:** Estos tres benchmarks tienen el problema de dead code elimination.
Identifica cada uno y corrígelo:

```go
// Benchmark 1: ¿el compilador elimina el cálculo?
func BenchmarkHash(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = fnv.New32a().Sum32()  // ¿se elimina?
    }
}

// Benchmark 2: ¿la lectura se elimina?
func BenchmarkLeer(b *testing.B) {
    cache := NuevaCache()
    cache.Set("k", "v")
    for i := 0; i < b.N; i++ {
        cache.Get("k")  // resultado ignorado — ¿se elimina?
    }
}

// Benchmark 3: ¿el cálculo de la goroutine se elimina?
func BenchmarkGoroutine(b *testing.B) {
    ch := make(chan int, 1)
    for i := 0; i < b.N; i++ {
        go func() { ch <- computar() }()
        <-ch  // ¿el compilador puede eliminar todo esto?
    }
}
```

**Restricciones:** Usa el ensamblador de Go (`go tool compile -S`) para
verificar si el código fue eliminado. Añade una variable sink global
para prevenir la eliminación.

**Pista:** El patrón estándar de Go para prevenir dead code elimination:
```go
var Sink int64  // variable global — no puede eliminarse
func BenchmarkHash(b *testing.B) {
    var r uint32
    for i := 0; i < b.N; i++ {
        r = fnv.New32a().Sum32()
    }
    Sink = int64(r)  // usar el resultado
}
```

**Implementar en:** Go · Java (JMH BlackHole) · Rust (criterion::black_box)

---

### Ejercicio 7.4.2 — False sharing en benchmarks

**Enunciado:** Este benchmark de Worker Pool mide contención accidental,
no el rendimiento del pool:

```go
var resultados [16]int64  // los workers escriben en índices adyacentes

func BenchmarkWorkerPool(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        id := /* ID del worker */
        for pb.Next() {
            resultados[id]++  // false sharing si id*8 < 64
        }
    })
}
```

Identifica el false sharing, mide su impacto, y corrígelo con padding.

**Restricciones:** Mide la diferencia de rendimiento antes y después del fix.
El speedup debe ser visible (>2x) en una máquina con múltiples cores.

**Pista:** `int64` son 8 bytes. Un array de 16 `int64` ocupa 128 bytes.
Los primeros 8 elementos caben en 2 líneas de caché (64 bytes cada una).
Si el worker 0 modifica `resultados[0]` y el worker 1 modifica `resultados[1]`,
ambas modificaciones invalidan la misma línea de caché — false sharing.
El fix: padding de 56 bytes entre elementos para que cada uno ocupe una línea.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 7.4.3 — Medir throughput vs latencia

**Enunciado:** Throughput y latencia son métricas distintas que a veces
se optimizan en conflicto. Un sistema puede tener alto throughput pero
alta latencia (procesa muchos items por segundo, pero cada item tarda mucho).

Implementa benchmarks que miden ambas para el Worker Pool:

```go
// Throughput: ¿cuántos items procesa por segundo?
func BenchmarkThroughput(b *testing.B) {
    pool := NuevoPool(runtime.NumCPU())
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        pool.Submit(Job{})
    }
    // ops/s = b.N / tiempo_total
}

// Latencia: ¿cuánto tiempo tarda un item específico?
func BenchmarkLatencia(b *testing.B) {
    pool := NuevoPool(runtime.NumCPU())
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        inicio := time.Now()
        pool.SubmitYEsperar(Job{})
        b.ReportMetric(float64(time.Since(inicio).Nanoseconds()), "ns/op")
    }
}
```

**Restricciones:** Reporta p50, p95, p99 de latencia — no solo el promedio.
El promedio de latencia puede ser engañoso cuando hay outliers.

**Pista:** `b.ReportMetric` en Go permite reportar métricas personalizadas.
Para percentiles, acumula las latencias en un slice y calcula al final.
`go test -bench . -benchmem` reporta alocaciones además de tiempo.

**Implementar en:** Go · Java (JMH) · Python (pytest-benchmark)

---

### Ejercicio 7.4.4 — Benchmark con calentamiento y estado estacionario

**Enunciado:** Los primeros resultados de un benchmark son siempre peores
(cache fría, JIT no calentado, OS scheduler ajustándose).
Go hace el calentamiento automáticamente con `testing.B`, pero para
benchmarks complejos necesitas control manual.

Implementa un benchmark del pipeline del Cap.03 §3.2 que:
1. Tiene una fase de calentamiento de 1 segundo (no medida)
2. Mide el estado estacionario durante 10 segundos
3. Reporta el throughput en estado estacionario

**Restricciones:** La fase de medición debe excluir los primeros resultados
transitorios. El reporte incluye el tiempo hasta alcanzar el estado estacionario.

**Pista:** Para detectar el estado estacionario, mide el throughput en ventanas
de 100ms. Cuando el coeficiente de variación (desviación estándar / media)
cae por debajo de 5%, se alcanzó el estado estacionario.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 7.4.5 — Comparar implementaciones: el benchmark justo

**Enunciado:** Quieres comparar Mutex vs RWMutex vs sync.Map para una caché.
Un benchmark injusto puede favorecer uno sobre otro por razones que no
tienen que ver con la implementación.

Diseña un benchmark justo que:
- Controla el ratio lectura/escritura (parámetro configurable)
- Controla el número de goroutines concurrentes
- Controla el tamaño del mapa (impacta cache del CPU)
- Mide la misma carga útil para todas las implementaciones

**Restricciones:** Las tres implementaciones tienen exactamente la misma
carga útil (mismo número de operaciones, mismos datos, mismo ratio).
La única variable es la implementación de sincronización.

**Pista:** La comparación justa requiere que el setup sea idéntico.
Un truco: generar la secuencia de operaciones (tipo + clave + valor)
antes del benchmark, y luego ejecutarla para cada implementación.
Así el overhead de generación de datos no está en la medición.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 7.5 — Testing de Integración: El Sistema Completo

Los tests unitarios verifican componentes individuales. Los tests de integración
verifican que los componentes funcionan juntos correctamente.

**El desafío específico de la integración concurrente:**

```
Tests unitarios: componente A está correctamente sincronizado ✓
Tests unitarios: componente B está correctamente sincronizado ✓
Tests de integración: A y B juntos tienen una race condition ✗
```

Esto ocurre porque las garantías de cada componente pueden no componer
de la forma esperada cuando se conectan.

**Los tres niveles de testing de integración para sistemas concurrentes:**

```
Nivel 1: Integración local
  Múltiples componentes en el mismo proceso.
  Verificar que las interfaces entre componentes son thread-safe.

Nivel 2: Integración con estado externo
  Componentes que interactúan con BD, cache, colas de mensajes.
  Verificar que el comportamiento es correcto bajo carga concurrente.

Nivel 3: Integración distribuida
  Múltiples instancias del servicio.
  Verificar que el sistema en conjunto mantiene sus invariantes.
```

---

### Ejercicio 7.5.1 — Test de integración: Worker Pool + Pub/Sub

**Enunciado:** El sistema del Ejercicio 3.7.3 (Pub/Sub + Worker Pool) tiene
una interfaz entre los dos componentes. Implementa un test de integración
que verifica que esta interfaz funciona correctamente bajo carga.

**Invariantes a verificar:**
- Cada evento publicado es procesado exactamente una vez
- El orden de procesamiento respeta el orden de publicación (dentro de cada topic)
- No hay eventos perdidos cuando el sistema se detiene limpiamente

**Restricciones:** El test debe ejecutar bajo carga real (1000 eventos en
1 segundo). Verifica las invariantes con contadores atómicos, no con
sleeps ni con esperas de tiempo fijo.

**Pista:** Para verificar "exactamente una vez", usa un `sync.Map` con IDs únicos
de eventos. Cada evento tiene un ID. El procesador marca el ID como procesado.
Al final, verifica que todos los IDs están marcados exactamente una vez.

**Implementar en:** Go · Java · Python

---

### Ejercicio 7.5.2 — Test de caos: matar goroutines aleatoriamente

**Enunciado:** El chaos engineering aplica fallas aleatorias para verificar
que el sistema se recupera correctamente. Para código concurrente local,
esto puede significar cancelar contextos en momentos aleatorios.

Implementa un "chaos test" para el sistema de ingesta del Ejercicio 3.7.4:

```go
func TestCaos(t *testing.T) {
    sistema := NuevoSistema()
    sistema.Iniciar()

    // Inyectar fallos aleatorios cada 100ms
    go func() {
        for {
            time.Sleep(100 * time.Millisecond)
            sistema.InyectarFallo(falloAleatorio())
        }
    }()

    // Verificar que el sistema sigue funcionando a pesar de los fallos
    time.Sleep(5 * time.Second)
    metricas := sistema.Metricas()
    if metricas.EventosPerdidos > 0 {
        t.Errorf("se perdieron %d eventos", metricas.EventosPerdidos)
    }
}
```

**Restricciones:** Los fallos inyectados deben ser realistas: context cancellation,
goroutine panics, canales llenos, y timeouts.
El sistema debe recuperarse de cada fallo sin pérdida de datos.

**Pista:** El chaos test no verifica la ausencia de errores — verifica que
los errores se manejan correctamente. Un circuit breaker que abre es correcto.
Un evento perdido porque el canal estaba lleno puede ser aceptable si la
política es drop — lo que no es aceptable es corrupción silenciosa.

**Implementar en:** Go · Java · Python

---

### Ejercicio 7.5.3 — Test de carga progresiva

**Enunciado:** Un test de carga progresiva incrementa gradualmente la carga
y registra cuándo el sistema empieza a degradarse. El objetivo no es
encontrar el límite máximo sino entender el comportamiento de degradación.

```go
func TestCargaProlongresiva(t *testing.T) {
    sistema := NuevoSistema()
    sistema.Iniciar()
    defer sistema.Detener()

    for rpm := 100; rpm <= 10_000; rpm += 100 {
        t.Run(fmt.Sprintf("rpm=%d", rpm), func(t *testing.T) {
            metricas := ejecutarCarga(sistema, rpm, 5*time.Second)
            t.Logf("rpm=%d latencia_p99=%s errores=%d",
                rpm, metricas.LatenciaP99, metricas.Errores)

            // El sistema no debe fallar antes de los rpm esperados
            if rpm < 1000 && metricas.Errores > 0 {
                t.Errorf("errores inesperados a %d rpm", rpm)
            }
        })
    }
}
```

**Restricciones:** El test reporta el punto de saturación — el rpm donde
la latencia empieza a crecer no linealmente. Este punto debe ser documentado
como el límite operacional del sistema.

**Pista:** La degradación graceful es el objetivo: el sistema debe rechazar
requests con un error explícito (503, ErrSobrecargado) antes de colgarse.
El test verifica que la degradación es ordenada — no que el sistema soporta
carga infinita.

**Implementar en:** Go · Java · Python

---

### Ejercicio 7.5.4 — Reproducir un bug de producción

**Enunciado:** Se reportó un bug en producción: "el servidor se cuelga
después de 24 horas de operación continua". El equipo sospecha de un
goroutine leak acumulativo.

Implementa un test de larga duración que:
1. Monitorea el número de goroutines cada 10 segundos
2. Detecta si el número crece indefinidamente (leak)
3. Reporta el stack trace de las goroutines nuevas que aparecen

```go
func TestGoroutineLeak(t *testing.T) {
    if testing.Short() {
        t.Skip("test de larga duración — solo en CI")
    }

    inicial := runtime.NumGoroutine()
    sistema := NuevoSistema()
    sistema.Iniciar()

    ticker := time.NewTicker(10 * time.Second)
    limite := time.After(5 * time.Minute)  // versión corta del test de 24h

    for {
        select {
        case <-ticker.C:
            actual := runtime.NumGoroutine()
            if actual > inicial+10 {  // tolerancia de 10
                t.Errorf("posible leak: %d goroutines (inicial: %d)", actual, inicial)
            }
        case <-limite:
            return  // test completado sin leak detectado
        }
    }
}
```

**Restricciones:** El test debe detectar leaks lentos (1 goroutine por request).
Con 100 requests por segundo durante 5 minutos, un leak de 1 goroutine/request
produciría 30,000 goroutines.

**Pista:** La señal más clara de un leak acumulativo es el crecimiento monótono
del número de goroutines. Un crecimiento y reducción cíclico es normal
(goroutines que se crean y terminan). Un crecimiento sin reducción es el leak.

**Implementar en:** Go · Java · Python

---

### Ejercicio 7.5.5 — Test de recuperación tras reinicio

**Enunciado:** Implementa un test que verifica que el sistema se reinicia
correctamente bajo carga: detener el sistema mientras procesa requests,
reiniciarlo, y verificar que el estado es consistente.

**Escenario:**
1. Iniciar el sistema
2. Lanzar 100 requests simultáneas
3. Detener el sistema a los 500ms (con algunas requests en vuelo)
4. Reiniciar el sistema
5. Verificar que el estado final es correcto: ningún request procesado
   dos veces, todos los requests completados o fallidos (no en estado limbo)

**Restricciones:** El test debe ser determinista — producir el mismo
resultado cada vez. Define claramente qué es "estado consistente"
después del reinicio.

**Pista:** Para detención bajo carga, `context.WithTimeout` con un timeout
corto fuerza que algunas requests estén en vuelo cuando el sistema se detiene.
El sistema debe garantizar: procesados-antes-del-stop + procesados-después-del-reinicio
= total de requests sin duplicados.

**Implementar en:** Go · Java · Python

---

## Sección 7.6 — Diseñar para Testabilidad

El código concurrente testeable se diseña diferente del código que
"funciona pero no se puede verificar".

**Los cinco principios de diseñabilidad para testing concurrente:**

```
1. Inyectar el tiempo:
   Cualquier código que use time.Now(), time.Sleep(), time.After()
   debe recibir un Clock inyectable para que los tests no necesiten sleeps.

2. Inyectar el scheduler:
   Para tests deterministas, power el runtime.Gosched() o el propio scheduler.
   En la práctica: puntos de yield configurables con build tags.

3. Exponer métricas internas:
   Goroutines activas, tamaño de colas, contadores de operaciones.
   No para el usuario final — para que los tests puedan verificar el estado.

4. Parada limpia determinista:
   El método Detener() debe bloquear hasta que todo esté limpio.
   No "enviar señal y esperar un segundo".

5. Estado exportable para snapshot:
   El sistema puede exportar un snapshot completo de su estado interno.
   Útil para verificar invariantes en tests de integración.
```

---

### Ejercicio 7.6.1 — Refactorizar el CircuitBreaker para testabilidad

**Enunciado:** El CircuitBreaker del Ejercicio 3.6.1 usa `time.Now()` directamente.
Refactorízalo para que sea completamente testeable sin `time.Sleep`:

```go
type CircuitBreaker struct {
    clock        Clock           // inyectable
    // ... resto del estado
}

// En tests:
fakeClock := NewFakeClock()
cb := NewCircuitBreaker(WithClock(fakeClock), WithThreshold(5), WithTimeout(10*time.Second))

// Simular 5 fallos:
for i := 0; i < 5; i++ {
    cb.Execute(func() error { return errors.New("fallo") })
}
// El CB debe estar abierto ahora

// Simular que pasan 11 segundos:
fakeClock.Advance(11 * time.Second)
// El CB debe pasar a semi-abierto
```

**Restricciones:** Ningún test del CircuitBreaker usa `time.Sleep`.
El FakeClock es thread-safe para uso en tests concurrentes.
`fakeClock.Advance(d)` notifica a todas las goroutines que esperan
un tiempo menor o igual a `d`.

**Pista:** El FakeClock necesita implementar `After(d)` que retorna un canal.
Cuando se llama `Advance(d)`, cierra todos los canales cuyo tiempo de expiración
es menor o igual al nuevo tiempo. Los tests pueden controlar el tiempo
con precisión de nanosegundos sin esperar nada real.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 7.6.2 — Inyectar el Worker Pool para tests

**Enunciado:** Un servicio que usa Worker Pool internamente es difícil de
testear si el Pool está hardcodeado. Con inyección de dependencias:

```go
// Difícil de testear — Pool hardcodeado
type Servicio struct {
    pool *WorkerPool
}

func NuevoServicio() *Servicio {
    return &Servicio{pool: NuevoPool(runtime.NumCPU())}
}

// Fácil de testear — Pool inyectado
type Ejecutor interface {
    Submit(ctx context.Context, job Job) error
    Detener()
}

type Servicio struct {
    ejecutor Ejecutor
}

func NuevoServicio(e Ejecutor) *Servicio {
    return &Servicio{ejecutor: e}
}

// En tests:
type EjecutorSincrono struct{}  // ejecuta síncronamente — sin concurrencia
func (e *EjecutorSincrono) Submit(ctx context.Context, job Job) error {
    return job.Ejecutar(ctx)  // inmediato, sin goroutine
}
```

**Restricciones:** El `EjecutorSincrono` para tests no lanza goroutines —
ejecuta síncronamente. Esto hace que los tests sean deterministas.
El servicio en producción usa el Pool real.

**Pista:** El `EjecutorSincrono` es el patrón "Test Double" (fake vs mock).
Al ejecutar síncronamente, los tests no tienen race conditions ni necesitan
sincronización especial. La concurrencia real se testa por separado,
en los tests del Worker Pool.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 7.6.3 — Señales de parada verificables

**Enunciado:** Un método `Detener()` que no garantiza que todo está limpio
al retornar hace los tests frágiles. Verifica que estas implementaciones
tienen parada determinista:

```go
// Versión 1 — no determinista:
func (s *Servicio) Detener() {
    s.activo = false
    time.Sleep(100 * time.Millisecond)  // esperar "suficiente"
}

// Versión 2 — determinista:
func (s *Servicio) Detener() {
    close(s.done)    // señalizar
    s.wg.Wait()      // esperar que todas las goroutines terminen
    close(s.resultados)  // cerrar el canal de resultados solo cuando todo terminó
}
```

**Restricciones:** Para cada sistema del repositorio que hayas implementado,
verifica que su `Detener()` es determinista: al retornar, no hay goroutines
activas del sistema ni recursos sin liberar.

**Pista:** El patrón correcto: `Detener` cierra un canal done, luego llama
`wg.Wait()`. Las goroutines detectan `<-done` y llaman `wg.Done()`.
Solo cuando el `wg.Wait()` retorna, `Detener` retorna. Esto es determinista:
la parada completa exactamente cuando todas las goroutines terminaron.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 7.6.4 — Snapshot del estado para verificación

**Enunciado:** Implementa un método `Snapshot()` para el sistema de ingesta
del Ejercicio 3.7.4 que retorna el estado completo en un momento dado:

```go
type SnapshotSistema struct {
    EventosProcesados  int64
    EventosEnCola      int64
    WorkersActivos     int
    CircuitBreakerState string
    Timestamp          time.Time
}

func (s *Sistema) Snapshot() SnapshotSistema
```

**Restricciones:** El Snapshot debe ser consistente — no puede tener valores
de momentos distintos. Implementa con un lock breve que pausa brevemente
todas las operaciones para tomar el snapshot.

**Pista:** Un snapshot completamente consistente requiere pausar el sistema
momentáneamente — igual que `runtime.ReadMemStats` pausa el GC.
Para métricas de monitoreo, la consistencia perfecta no es siempre necesaria.
Para tests de corrección, sí lo es.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 7.6.5 — Checklist de testabilidad

**Enunciado:** Crea un checklist de testabilidad para código concurrente
y aplícalo a todos los sistemas implementados en el repositorio.

**El checklist:**

```
□ ¿Usa Clock inyectable? (no time.Now() hardcodeado)
□ ¿Tiene método Detener() determinista? (wg.Wait(), no time.Sleep)
□ ¿Las goroutinas internas tienen nombres en el stack trace? (pprof labels)
□ ¿Expone métricas internas para tests? (goroutines activas, tamaño de colas)
□ ¿Acepta inyección de dependencias para el Ejecutor/Pool/Broker?
□ ¿El test usa goleak.VerifyNone(t)?
□ ¿El test tiene timeout explícito?
□ ¿El test verifica invariantes, no valores específicos?
□ ¿El test corre con go test -race?
□ ¿El test tiene una barrera de inicio para maximizar intercalación?
```

**Restricciones:** Para cada sistema que no pase el checklist, identificar
qué refactoring mínimo lo hace pasar. Priorizar por impacto.

**Pista:** Un sistema que pasa el 80% del checklist es significativamente
más testeable que uno que pasa el 20%, aunque ambos tengan bugs similares.
La testabilidad no es binaria — es un espectro.

**Implementar en:** Revisión de código + refactoring donde sea necesario

---

## Sección 7.7 — Regresiones de Concurrencia en CI/CD

Una regresión de concurrencia es un bug que se introduce con un cambio
y que no se detecta hasta producción. Son los más costosos porque
el cambio ya está en producción.

**La pirámide de testing para concurrencia:**

```
                        [Tests de carga]        ← lentos, costosos
                      [Tests de integración]     ← moderados
                 [Tests unitarios con -race]      ← rápidos, siempre en CI
             [Análisis estático (go vet, staticcheck)] ← instantáneo
```

**Lo mínimo para CI:**

```bash
# En todo PR antes de merge:
go vet ./...
staticcheck ./...        # análisis estático con más checks que go vet
go test -race ./...      # todos los tests con race detector
go test -race -count=3 ./... # los tests concurrentes, 3 veces

# En merge a main:
go test -race -count=10 ./...  # más repeticiones
```

---

### Ejercicio 7.7.1 — Pipeline de CI para código concurrente

**Enunciado:** Diseña e implementa el pipeline de CI para el repositorio
de concurrencia. El pipeline debe:

1. Análisis estático instantáneo (`go vet`, `staticcheck`)
2. Tests rápidos con race detector (`go test -race -short ./...`)
3. Tests completos con race detector (sin `-short`)
4. Stress tests (con `-run TestStress`)
5. Benchmark baseline (para detectar regresiones de rendimiento)

**Restricciones:** El pipeline en PR debe completar en menos de 5 minutos.
Los benchmarks corren en merge a main, no en PR.

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - name: Vet
        run: go vet ./...
      - name: Race detector
        run: go test -race -timeout 5m ./...
```

**Pista:** Separar los jobs permite ver qué falla: si solo falla el race detector,
el bug es de concurrencia. Si falla el vet, es un problema de código.
La separación facilita el diagnóstico y el fix.

**Implementar en:** GitHub Actions · GitLab CI · Jenkins (el que uses)

---

### Ejercicio 7.7.2 — Benchmark baseline y detección de regresiones

**Enunciado:** Las regresiones de rendimiento son silenciosas — el programa
sigue funcionando pero 20% más lento. Implementa un sistema de baseline
de benchmarks que detecta regresiones en CI:

```bash
# Guardar baseline en main:
go test -bench . -benchmem -count=5 ./... > baseline.txt

# En PR, comparar con baseline:
go test -bench . -benchmem -count=5 ./... > actual.txt
benchstat baseline.txt actual.txt

# Output de benchstat:
# name          old time/op  new time/op  delta
# WorkerPool-8  1.23ms ±2%   1.87ms ±3%  +52.0% (p=0.008 n=5+5)  ← REGRESIÓN
```

**Restricciones:** Usar `golang.org/x/perf/cmd/benchstat` para la comparación.
Una regresión > 10% en cualquier benchmark debe fallar el CI.

**Pista:** Los benchmarks son ruidosos — el mismo código puede variar ±5-10%
entre runs. `benchstat` usa estadística para distinguir variaciones normales
de regresiones reales. Con `-count=5`, tienes suficientes muestras para
que la comparación sea estadísticamente significativa.

**Implementar en:** Go + GitHub Actions

---

### Ejercicio 7.7.3 — Mutant testing para código concurrente

**Enunciado:** El mutant testing genera versiones "mutadas" del código
(cambios pequeños) y verifica que algún test falla para cada mutante.
Si ningún test falla para un mutante, esa parte del código no está cubierta.

Para código concurrente, los mutantes más interesantes son:
- Eliminar un `mu.Lock()` (simular la omisión del lock)
- Cambiar `<-ch` por `ch <- valor` (invertir dirección del canal)
- Cambiar `wg.Done()` por no-op (simular el olvido del Done)
- Cambiar `close(done)` por no-op (simular el olvido del cierre)

```bash
# Herramienta: go-mutesting
go install github.com/zimmski/go-mutesting/cmd/go-mutesting@latest
go-mutesting ./...
```

**Restricciones:** Para cada mutante que no es detectado por ningún test,
escribe el test mínimo que lo detectaría.

**Pista:** El mutant testing para concurrencia es especialmente valioso porque
los bugs de concurrencia (olvidar un lock, olvidar cerrar un canal) son exactamente
los que el mutant testing simula. Si eliminar un lock no hace fallar ningún test,
ese lock no está cubierto por los tests actuales.

**Implementar en:** Go · Java (PIT mutation testing)

---

### Ejercicio 7.7.4 — Alertas de goroutine leak en producción

**Enunciado:** Los goroutine leaks en producción son silenciosos hasta que
el sistema se queda sin memoria. Implementa un monitor que alerta cuando
el número de goroutines crece de forma inesperada:

```go
type MonitorGoroutines struct {
    umbralCrecimiento float64  // alertar si crece más del X% en T segundos
    ventana           time.Duration
    historico         []medicion
    alertar           func(msg string)  // inyectable para tests
}

type medicion struct {
    timestamp  time.Time
    goroutines int
}

func (m *MonitorGoroutines) Iniciar(ctx context.Context)
func (m *MonitorGoroutines) Estado() EstadoMonitor
```

**Restricciones:** El monitor no debe afectar el rendimiento — muestrea
`runtime.NumGoroutine()` cada 30 segundos, no más frecuente.
La alerta es configurable: log, metrics, PagerDuty, etc.

**Pista:** Una regla simple: si el número de goroutines creció más del 20%
en los últimos 5 minutos sin que hubiera un aumento correspondiente en el
número de requests (la carga), es una señal de leak.
Una alerta más sofisticada correlaciona el número de goroutines con la carga.

**Implementar en:** Go · Java · Python

---

### Ejercicio 7.7.5 — Post-mortem: analizar un incidente de concurrencia

**Enunciado:** Escribe el post-mortem de un incidente hipotético:

**El incidente:** El servicio de procesamiento de pedidos empezó a tener
latencias de 30+ segundos a las 14:23 del lunes. A las 14:45 se reinició
el servicio y las latencias volvieron a normal. El incidente duró 22 minutos.

**Los síntomas:**
- Latencia p99 pasó de 200ms a 32 segundos a las 14:23
- CPU al 100% en todos los cores
- Número de goroutines: 847 a las 14:20, 45,231 a las 14:25
- Memory usage: estable en 2 GB
- Rate de requests: sin cambios (500 req/s)

**El post-mortem debe responder:**
1. ¿Qué causó el incidente? (diagnosing from symptoms)
2. ¿Por qué no se detectó antes?
3. ¿Qué tests habrían detectado el problema?
4. ¿Qué cambios en el código o en el sistema de monitoreo previenen la recurrencia?

**Restricciones:** El análisis debe ser técnicamente específico — no solo
"añadir más monitoring". Debe identificar la causa raíz concreta
(race condition, deadlock, goroutine leak, etc.) basándose en los síntomas.

**Pista:** 847 → 45,231 goroutines en 5 minutos con CPU al 100% y sin cambio
en los requests sugiere un goroutine leak con cómputo activo — posiblemente
un livelock o un loop infinito creado por algún path de error. El hecho de
que se resuelva con reinicio (no con reducción de carga) sugiere un estado
acumulativo, no una respuesta a carga puntual.

**Implementar en:** Documento escrito + implementación del fix

---

## Resumen del capítulo

**Las cinco propiedades de un test de concurrencia correcto:**

```
1. Determinista:  falla siempre cuando el bug existe (barrera de inicio, -race)
2. Acotado:       tiene timeout explícito (no puede colgar para siempre)
3. Aislado:       limpia goroutines al terminar (goleak.VerifyNone)
4. Específico:    verifica invariantes, no valores dependientes de timing
5. Completo:      cubre el comportamiento bajo carga, no solo el camino feliz
```

**El checklist mínimo para CI:**

```bash
go vet ./...                          # análisis estático
go test -race ./...                   # todos los tests con race detector
go test -race -count=3 ./...          # tests concurrentes, repetidos
goleak.VerifyNone(t)                  # en cada test que lanza goroutines
```

**La tabla de herramientas:**

| Problema | Herramienta |
|---|---|
| Race conditions | `go test -race` |
| Goroutine leaks | `goleak` |
| Deadlocks | `go-deadlock`, timeout en tests |
| Rendimiento | `go test -bench`, `benchstat` |
| Correctness formal | `gopter`/`rapid` (PBT), `porcupine` (linearizabilidad) |
| Cobertura de tests | `go-mutesting` |
| Producción | `runtime/metrics`, monitor de goroutines |

## Cierre de la Parte 1

> Los siete capítulos de la Parte 1 cubren la concurrencia local completa:
> los problemas (Cap.01), las herramientas (Cap.02), los patrones (Cap.03),
> el runtime de Go (Cap.04), el código idiomático (Cap.05),
> los modelos alternativos (Cap.06), y el testing (Cap.07).
>
> La Parte 2 cubre el paralelismo: cómo usar múltiples núcleos de forma
> eficiente para trabajo CPU-bound — distinto de la concurrencia, que es
> sobre la estructura del programa.
>
> La Parte 3 entra en las implementaciones específicas por lenguaje:
> Go, Rust, Java, y Python — cada uno con su modelo propio y sus tradeoffs.
