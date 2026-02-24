# Guía de Ejercicios — Los 5 Problemas Fundamentales de la Concurrencia

> Implementar cada ejercicio en: **Go · Java · Python · Rust · C#**
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.
>
> Nota sobre los lenguajes: Go es el lenguaje de referencia principal de este
> repositorio. Los ejemplos más ilustrativos están en Go. Java y Python siguen
> para entrevistas en esos ecosistemas. Rust aparece donde la seguridad en
> compilación es el punto.

---

## Antes de la teoría: un bug que aparece una vez cada mil ejecuciones

```go
// contador.go — ¿qué está mal?

package main

import (
    "fmt"
    "sync"
)

var contador int

func incrementar(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        contador++   // ← esta línea parece inocente
    }
}

func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    go incrementar(&wg)
    go incrementar(&wg)
    wg.Wait()
    fmt.Println(contador)  // ¿qué imprime?
}
```

La respuesta esperada es `2000`. La respuesta real varía:

```
$ go run contador.go → 1847
$ go run contador.go → 2000
$ go run contador.go → 1923
$ go run contador.go → 2000
$ go run contador.go → 1756
```

El programa es **no determinista**. Tiene un bug que no se reproduce siempre,
que no lanza ninguna excepción, y que en producción podría pasar desapercibido
durante meses hasta que las condiciones sean exactas.

Este es el tipo de bug más peligroso que existe en software. Se llama **race condition**.

El resto del capítulo explica por qué ocurre y los cuatro problemas hermanos
que comparten la misma raíz.

---

## Por qué ocurre — la ilusión de la línea única

`contador++` parece una operación atómica. No lo es. El procesador la descompone en tres pasos:

```
1. LEER   el valor actual de contador desde memoria → registro
2. SUMAR  1 al valor en el registro
3. ESCRIBIR el nuevo valor desde el registro a memoria
```

Con dos goroutines ejecutándose simultáneamente, los pasos pueden intercalarse:

```
Tiempo  Goroutine A              Goroutine B        contador en memoria
──────  ──────────────────────   ──────────────────  ─────────────────
  1     LEER contador = 100                          100
  2                              LEER contador = 100  100
  3     SUMAR: 100 + 1 = 101                         100
  4                              SUMAR: 100 + 1 = 101 100
  5     ESCRIBIR 101                                  101
  6                              ESCRIBIR 101          101  ← perdimos un incremento
```

Ambas goroutines leyeron `100`, ambas calcularon `101`, ambas escribieron `101`.
El resultado es `101` en lugar de `102`. Se perdió un incremento.

Esto ocurre porque `contador++` **no es atómico** — puede ser interrumpido entre
sus tres pasos por el scheduler del sistema operativo o por la ejecución paralela
en múltiples núcleos.

---

## Los 5 problemas fundamentales

Todo problema de concurrencia es una variante de alguno de estos cinco.
Conocerlos es reconocer el tipo de problema antes de buscar la solución.

```
1. Race condition    → dos agentes acceden al mismo dato, al menos uno escribe
2. Deadlock          → dos agentes se esperan mutuamente para siempre
3. Livelock          → dos agentes se mueven pero no progresan
4. Starvation        → un agente nunca obtiene acceso al recurso
5. Visibilidad       → un agente no ve los cambios que hizo otro
```

---

## Tabla de contenidos

- [Sección 1.1 — Race Condition](#sección-11--race-condition)
- [Sección 1.2 — Deadlock](#sección-12--deadlock)
- [Sección 1.3 — Livelock](#sección-13--livelock)
- [Sección 1.4 — Starvation](#sección-14--starvation)
- [Sección 1.5 — Visibilidad de memoria](#sección-15--visibilidad-de-memoria)
- [Sección 1.6 — Detectar el tipo de problema](#sección-16--detectar-el-tipo-de-problema)
- [Sección 1.7 — El detector: -race de Go y herramientas equivalentes](#sección-17--el-detector--race-de-go-y-herramientas-equivalentes)

---

## Sección 1.1 — Race Condition

**Definición formal:** dos o más goroutines/hilos acceden a la misma
variable compartida de forma concurrente, y al menos una de ellas escribe.
El resultado final depende del orden de ejecución — que es no determinista.

**Condiciones necesarias (las tres deben cumplirse):**

```
1. Acceso compartido    → dos agentes acceden al mismo dato
2. Al menos una escritura → no es race condition si todos solo leen
3. Sin sincronización   → no hay mecanismo que ordene los accesos
```

Si se elimina cualquiera de las tres condiciones, no hay race condition.

**Las tres soluciones correspondientes:**

```
Eliminar 1: no compartir el dato       → cada goroutine tiene su copia
Eliminar 2: hacer el dato inmutable    → solo lectura después de inicializar
Eliminar 3: sincronizar los accesos    → mutex, canal, atómico
```

**El patrón más común en entrevistas:**

```go
// PROBLEMA: banco con dos goroutines haciendo transferencias
type Cuenta struct {
    saldo int
}

func (c *Cuenta) Depositar(monto int) {
    c.saldo += monto  // race condition si se llama desde múltiples goroutines
}

func (c *Cuenta) Retirar(monto int) bool {
    if c.saldo < monto {
        return false
    }
    c.saldo -= monto  // race condition: entre el if y el -= alguien pudo retirar
    return true
}
```

La race condition en `Retirar` es especialmente peligrosa: el saldo puede
volverse negativo si dos goroutines pasan el `if` simultáneamente antes de
que cualquiera ejecute el `-=`.

---

### Ejercicio 1.1.1 — Encontrar y confirmar la race condition

**Enunciado:** El siguiente código tiene una race condition. Sin ejecutarlo,
identifica exactamente en qué líneas ocurre y describe el escenario de
intercalación que produce un resultado incorrecto.

```go
package main

import "sync"

type Cache struct {
    datos map[string]string
}

func (c *Cache) Set(clave, valor string) {
    c.datos[clave] = valor
}

func (c *Cache) Get(clave string) (string, bool) {
    v, ok := c.datos[clave]
    return v, ok
}

func main() {
    cache := &Cache{datos: make(map[string]string)}
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("clave%d", n), "valor")
        }(i)
    }
    wg.Wait()
}
```

**Restricciones:** Primero analiza el código sin ejecutarlo. Luego ejecútalo con `go run -race` para confirmar tu análisis.

**Pista:** En Go, los maps no son seguros para acceso concurrente. El runtime de Go puede lanzar un panic fatal si detecta acceso concurrente a un map — pero no siempre lo detecta.

**Implementar en:** Go · Java (HashMap equivalente) · Python (dict equivalente)

---

### Ejercicio 1.1.2 — El saldo negativo

**Enunciado:** Implementa el sistema bancario con `Cuenta` del ejemplo anterior. Escribe un test que demuestre que el saldo puede volverse negativo con acceso concurrente. Luego arréglalo con un mutex.

**Restricciones:** El test debe reproducir el bug al menos el 90% de las ejecuciones para ser útil. Usar `testing` de Go o JUnit. No es válido usar `time.Sleep` para "esperar a que ocurra el bug".

**Pista:** Para aumentar la probabilidad de intercalación, usa `runtime.GOMAXPROCS(runtime.NumCPU())` y lanza muchas goroutines pequeñas. El `go test -race` debe reportar la race condition antes de arreglarla.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.1.3 — Singleton con race condition

**Enunciado:** El patrón Singleton en entornos concurrentes tiene una race condition clásica. Identifica el problema en este código y arréglalo sin usar un mutex global:

```go
var instancia *Servicio
var inicializado bool

func ObtenerServicio() *Servicio {
    if !inicializado {          // ← línea A
        instancia = &Servicio{} // ← línea B
        inicializado = true     // ← línea C
    }
    return instancia
}
```

**Restricciones:** La solución debe ser correcta para cualquier número de goroutines. La instancia debe crearse exactamente una vez. No uses `init()` de Go.

**Pista:** Go tiene `sync.Once` exactamente para esto. Antes de usarlo, describe el escenario de race condition: ¿qué pasa si dos goroutines pasan la línea A simultáneamente antes de que ninguna ejecute la línea B?

**Implementar en:** Go (`sync.Once`) · Java (`synchronized` + double-checked locking) · Python (`threading.Lock`) · C# (`Lazy<T>`) · Rust (`std::sync::OnceLock`)

---

### Ejercicio 1.1.4 — Race condition en estructuras de datos

**Enunciado:** Implementa una pila (stack) concurrente que sea segura para múltiples goroutines/hilos. Escribe primero la versión con race condition, luego la versión correcta. Verifica con `go test -race`.

**Restricciones:** La pila debe soportar `Push(valor)`, `Pop() (valor, ok)` y `Len() int`. La race condition de `Pop` en una pila vacía es especialmente interesante — descríbela.

**Pista:** La race condition de `Len` + `Pop` es sutil: verificar que `Len() > 0` y luego llamar `Pop()` no es atómico aunque cada operación individualmente sea segura.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.1.5 — Check-then-act: el patrón más frecuente en entrevistas

**Enunciado:** El patrón "check-then-act" es la fuente más común de race conditions en entrevistas. Identifica y corrige el problema en estos tres casos:

1. `if archivo.Existe() { archivo.Leer() }` — el archivo puede borrarse entre el check y el read
2. `if saldo >= monto { saldo -= monto }` — el saldo puede cambiar entre el if y el -=
3. `if !mapa.Contiene(clave) { mapa.Insertar(clave, valor) }` — otro hilo puede insertar entre el check y el insert

**Restricciones:** Cada caso tiene una solución idiomática distinta en cada lenguaje. No es válido poner un mutex global alrededor de todo el programa.

**Pista:** El patrón correcto convierte "check-then-act" en una **operación atómica**. En bases de datos esto se llama "compare-and-swap". En Go hay `sync/atomic.CompareAndSwap`. En Java hay `ConcurrentHashMap.putIfAbsent`.

**Implementar en:** Go · Java · Python · C# · Rust

---

## Sección 1.2 — Deadlock

**Definición:** dos o más goroutines/hilos esperan cada una un recurso
que está bloqueado por la otra. Ninguna puede progresar.

**Las cuatro condiciones de Coffman (todas deben cumplirse para que haya deadlock):**

```
1. Exclusión mutua    → el recurso solo puede usarlo un agente a la vez
2. Hold and wait      → un agente tiene un recurso y espera otro
3. No preemption      → el recurso no puede quitársele al agente a la fuerza
4. Espera circular    → A espera a B, B espera a C, C espera a A
```

Eliminar cualquiera de las cuatro condiciones previene el deadlock.

**El ejemplo canónico — dos mutexes en orden inverso:**

```go
var mu1, mu2 sync.Mutex

// Goroutine A
go func() {
    mu1.Lock()
    // ... hace algo ...
    mu2.Lock()   // espera a que B libere mu2
    mu2.Unlock()
    mu1.Unlock()
}()

// Goroutine B
go func() {
    mu2.Lock()
    // ... hace algo ...
    mu1.Lock()   // espera a que A libere mu1
    mu1.Unlock()
    mu2.Unlock()
}()

// A tiene mu1, espera mu2.
// B tiene mu2, espera mu1.
// Ambas esperan para siempre.
```

**Diferencia clave con race condition:**

```
Race condition:  el programa produce resultados incorrectos (o crashes)
Deadlock:        el programa se cuelga — no produce nada
```

En producción, un deadlock se manifiesta como un servicio que "deja de responder"
sin lanzar ningún error. Es silencioso y difícil de diagnosticar sin herramientas.

---

### Ejercicio 1.2.1 — Construir y diagnosticar un deadlock

**Enunciado:** Implementa el deadlock clásico de dos mutexes en orden inverso. Luego usa el detector de deadlocks de Go (`go run` detecta deadlocks automáticamente cuando **todas** las goroutines están bloqueadas) para confirmar. Describe exactamente en qué estado está cada goroutine cuando el deadlock ocurre.

**Restricciones:** El deadlock debe ser reproducible al 100%. Documenta el output del runtime de Go cuando detecta el deadlock.

**Pista:** Go lanza `fatal error: all goroutines are asleep - deadlock!` cuando detecta que ninguna goroutine puede progresar. Este detector solo funciona cuando **todas** las goroutines están bloqueadas — si hay una goroutine activa (aunque sea inútil), Go no lo detecta.

**Implementar en:** Go · Java (ThreadMXBean para detección) · Python

---

### Ejercicio 1.2.2 — El problema de los filósofos

**Enunciado:** Cinco filósofos están sentados en una mesa circular. Entre cada par de filósofos hay un tenedor. Para comer, un filósofo necesita ambos tenedores (izquierdo y derecho). Implementa la simulación y demuestra el deadlock cuando todos intentan tomar el tenedor izquierdo simultáneamente.

Luego implementa tres soluciones:
1. **Ordenamiento de recursos**: el filósofo siempre toma primero el tenedor de menor número
2. **Árbitro**: un mutex global que permite a lo sumo 4 filósofos intentar comer a la vez
3. **Timeout**: si no consigue el segundo tenedor en 100ms, suelta el primero y reintenta

**Restricciones:** La simulación debe correr durante 10 segundos sin deadlock con cualquiera de las tres soluciones.

**Pista:** El problema de los filósofos es el ejemplo canónico de la condición de espera circular. La solución de ordenamiento de recursos elimina esa condición: si todos piden los recursos en el mismo orden, no puede haber ciclo.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.2.3 — Deadlock con canales en Go

**Enunciado:** Los canales de Go también pueden producir deadlocks. Identifica el problema en cada caso y arréglalo:

```go
// Caso 1: canal sin buffer, nadie lee
ch := make(chan int)
ch <- 42  // bloquea para siempre

// Caso 2: goroutine esperando a sí misma
ch := make(chan int)
go func() {
    ch <- 1
    v := <-ch  // espera algo que nunca llega
}()

// Caso 3: dos goroutines enviando a canales del otro
ch1 := make(chan int)
ch2 := make(chan int)
go func() { ch1 <- 1; <-ch2 }()
go func() { ch2 <- 1; <-ch1 }()
```

**Restricciones:** Para cada caso, explica por qué ocurre el deadlock antes de proponer la solución.

**Pista:** Los deadlocks con canales son especialmente comunes en entrevistas de Go. La regla general: un canal sin buffer requiere que el receptor esté listo **antes** de que el emisor envíe. Si el emisor envía primero, se bloquea esperando al receptor.

**Implementar en:** Go

---

### Ejercicio 1.2.4 — Prevención por ordenamiento de recursos

**Enunciado:** Implementa un sistema de transferencias bancarias entre cuentas donde cada transferencia necesita bloquear ambas cuentas (origen y destino). Demuestra el deadlock cuando los bloqueos se toman en orden arbitrario, luego arréglalo ordenando siempre por ID de cuenta.

**Restricciones:** El sistema debe soportar transferencias concurrentes entre cualquier par de cuentas. La solución debe ser correcta sin un mutex global (que sería un cuello de botella).

**Pista:** Si `transferir(A→B)` bloquea A luego B, y `transferir(B→A)` bloquea B luego A, hay deadlock potencial. Si ambas bloquean `min(A,B)` luego `max(A,B)`, el ciclo es imposible.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.2.5 — Detección de deadlock en grafos (conexión con Cap.04 del repo)

**Enunciado:** Un deadlock puede representarse como un ciclo en un grafo de espera: los nodos son goroutines/hilos y las aristas son "A espera a B". Implementa un detector de deadlocks que construya este grafo en tiempo de ejecución y use DFS (Cap.04 §2 del repo de algoritmos) para detectar ciclos.

**Restricciones:** El detector debe funcionar en tiempo real — no puede pausar el programa para analizarlo. Actualiza el grafo de espera con cada `Lock()` y `Unlock()`.

**Pista:** Esta es exactamente la técnica que usa Java's `ThreadMXBean.findDeadlockedThreads()` internamente. El grafo de espera se llama "Wait-for graph" en la literatura. Un ciclo en este grafo es condición necesaria y suficiente para deadlock.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 1.3 — Livelock

**Definición:** dos o más agentes se responden mutuamente a los cambios
del otro, pero ninguno progresa. A diferencia del deadlock, los agentes
**no están bloqueados** — están activos, consumiendo CPU, pero atrapados
en un ciclo de reacción mutua.

**La metáfora más útil:**

```
Dos personas en un pasillo estrecho que se quieren ceder el paso:
  A se mueve a la derecha → B se mueve a la derecha también para dejar pasar
  A se mueve a la izquierda → B se mueve a la izquierda también
  Ambas están en movimiento. Ninguna pasa.
```

**En código — el ejemplo clásico:**

```go
type Agente struct {
    nombre    string
    tieneRecurso bool
}

func (a *Agente) intentarCeder(otro *Agente) {
    for a.tieneRecurso {
        if otro.tieneRecurso {
            // "El otro también tiene el recurso, yo cedo"
            a.tieneRecurso = false
            time.Sleep(time.Millisecond)
            a.tieneRecurso = true  // lo recupera para "intentarlo de nuevo"
        }
    }
}

// Si A y B ejecutan intentarCeder simultáneamente:
// A ve que B tiene el recurso → A cede
// B ve que A tiene el recurso → B cede
// A recupera → B recupera → ciclo infinito
```

**Por qué es más difícil de detectar que un deadlock:**

```
Deadlock:  CPU al 0%, el proceso no responde → obvio
Livelock:  CPU al 100%, el proceso no responde → parece "trabajando"
```

Un livelock puede parecer un problema de rendimiento antes de identificarse
como un problema de concurrencia.

---

### Ejercicio 1.3.1 — Construir un livelock observable

**Enunciado:** Implementa el ejemplo del pasillo con dos goroutines. Verifica que:
1. Ambas goroutines están activas (imprime un contador de intentos)
2. Ninguna progresa (el contador de éxitos permanece en 0)
3. La CPU está alta (no es un deadlock)

Luego arréglalo introduciendo aleatoriedad: cada agente espera un tiempo aleatorio antes de reintentar.

**Restricciones:** El livelock debe ser observable durante al menos 2 segundos antes del fix. La solución con aleatoriedad debe resolver el problema en menos de 100ms en el 99% de los casos.

**Pista:** La solución con aleatoriedad es exactamente cómo Ethernet resuelve las colisiones (CSMA/CD con backoff exponencial). La aleatoriedad rompe la simetría que causa el livelock.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.3.2 — Livelock en sistema de reintentos

**Enunciado:** Un sistema de colas tiene dos consumidores que procesan mensajes. Cuando un mensaje falla, se reencola. Si el mensaje falla para el Consumidor A, lo reencola. El Consumidor B lo intenta y también falla. Lo reencola. El Consumidor A lo intenta de nuevo... Implementa este escenario y demuestra el livelock.

Arréglalo con:
1. Dead letter queue: después de `n` intentos, el mensaje va a una cola de errores
2. Backoff exponencial: cada reintento espera el doble

**Restricciones:** El livelock debe ser reproducible. La solución debe garantizar que ningún mensaje se procesa infinitamente.

**Pista:** Este patrón — mensaje que siempre falla y siempre se reencola — es un livelock de sistema distribuido. RabbitMQ y Kafka tienen "dead letter queues" exactamente para esto.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.3.3 — Distinguir livelock de deadlock en código ajeno

**Enunciado:** Dado el siguiente código, determina sin ejecutarlo si tiene deadlock, livelock, race condition, o ningún problema:

```go
// Sistema A
func (s *Sistema) Procesar(recurso *Recurso) {
    for {
        if recurso.Disponible() {
            recurso.Tomar()
            s.Ejecutar(recurso)
            recurso.Liberar()
            return
        }
        time.Sleep(10 * time.Millisecond)
    }
}
```

Si dos goroutines ejecutan `Procesar` con el mismo `recurso` simultáneamente, ¿qué problema surge?

**Restricciones:** Justifica tu respuesta describiendo el escenario exacto de intercalación problemático. Luego verifica ejecutándolo.

**Pista:** `Disponible()` y `Tomar()` son dos operaciones separadas — no atómicas. Este es el patrón check-then-act de la Sección 1.1 combinado con un retry loop.

**Implementar en:** Go · Java · Python

---

### Ejercicio 1.3.4 — Protocolo de handshake con livelock

**Enunciado:** Dos servicios deben establecer una conexión: cada uno espera a que el otro esté "listo" antes de proceder. Implementa el protocolo y demuestra el livelock cuando ambos esperan la señal del otro simultáneamente.

Arréglalo designando un "líder" (el de menor ID actúa primero).

**Pista:** Este patrón aparece en protocolos de red. TCP resuelve el handshake con una secuencia asimétrica (SYN, SYN-ACK, ACK) — uno siempre inicia y el otro siempre responde. La asimetría previene el livelock.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.3.5 — Backoff exponencial con jitter

**Enunciado:** Implementa una función `ConReintentos(operacion, maxIntentos, backoffBase)` que:
1. Ejecuta `operacion`
2. Si falla, espera `backoffBase * 2^intento + jitter_aleatorio` antes de reintentar
3. Si todos los intentos fallan, retorna el último error

Verifica que con dos clientes compitiendo por un recurso, el backoff exponencial con jitter previene el livelock y ambos eventualmente tienen éxito.

**Restricciones:** El jitter debe ser aleatorio — no determinista. El backoff máximo debe estar acotado (`cap`). Retorna `Result[T, Error]` (conexión con Cap.17 del repo de algoritmos).

**Pista:** AWS recomienda "Full Jitter" sobre "Equal Jitter" para sistemas distribuidos. El jitter rompe la sincronización que causaría que todos los clientes reintenten al mismo tiempo — que es exactamente el livelock.

**Implementar en:** Go · Java · Python · C# · Rust

---

## Sección 1.4 — Starvation

**Definición:** un agente nunca obtiene acceso al recurso que necesita
porque otros agentes con mayor prioridad o más suerte siempre se lo ganan.
El agente hambriento puede estar listo para ejecutar pero nunca le toca.

**La diferencia con los anteriores:**

```
Race condition: resultado incorrecto (todos ejecutan, mal)
Deadlock:       ninguno ejecuta (todos bloqueados)
Livelock:       todos ejecutan pero nadie progresa (todos activos, sin avance)
Starvation:     algunos ejecutan, uno nunca (injusticia)
```

**El escenario más común:**

```go
// Lectores y escritores con prioridad a lectores
// Si siempre hay lectores activos, el escritor nunca puede escribir

var mu sync.RWMutex

// Muchos lectores
for i := 0; i < 100; i++ {
    go func() {
        for {
            mu.RLock()
            leer()
            mu.RUnlock()
            // inmediatamente vuelve a leer
        }
    }()
}

// Un escritor que nunca entra
go func() {
    for {
        mu.Lock()    // espera a que TODOS los lectores terminen
        escribir()   // pero siempre hay algún lector activo
        mu.Unlock()
    }
}()
```

---

### Ejercicio 1.4.1 — Demostrar starvation en lectores-escritores

**Enunciado:** Implementa el escenario de lectores-escritores donde el escritor muere de hambre. Mide cuántos milisegundos tarda el escritor en obtener acceso cuando hay 10 lectores continuos. Luego implementa una solución que garantice que el escritor obtiene acceso en a lo sumo `T` milisegundos.

**Restricciones:** El tiempo máximo de espera para el escritor debe ser configurable. Los lectores no deben bloquearse innecesariamente cuando no hay escritores esperando.

**Pista:** Una solución es "escritores tienen prioridad": cuando un escritor espera, los nuevos lectores deben esperar también. Esto garantiza que el escritor entra cuando los lectores actuales terminan. Java's `ReentrantReadWriteLock` tiene un parámetro `fair` que hace exactamente esto.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.4.2 — Scheduler injusto

**Enunciado:** Implementa una cola de tareas con prioridades donde las tareas de alta prioridad siempre se procesan antes. Demuestra que una tarea de baja prioridad puede esperar indefinidamente si siempre llegan tareas de alta prioridad.

Arréglalo con **aging**: la prioridad de una tarea aumenta con el tiempo que lleva esperando, hasta llegar a la prioridad máxima.

**Restricciones:** El aging debe garantizar que cualquier tarea eventualmente se procesa. El tiempo máximo de espera debe ser `O(max_prioridad * intervalo_aging)`.

**Pista:** El aging es la técnica que usan los schedulers de sistemas operativos (Linux CFS) para garantizar fairness. Una tarea que lleva mucho tiempo esperando eventualmente se ejecuta aunque su prioridad original sea baja.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 1.4.3 — Mutex no justo vs mutex justo

**Enunciado:** Go's `sync.Mutex` no garantiza orden FIFO — una goroutine nueva puede "robarle" el mutex a una que lleva tiempo esperando. Implementa un mutex justo (fair mutex) que garantiza orden FIFO usando una cola de waiters.

Verifica la diferencia: con 10 goroutines compitiendo, el mutex estándar puede hacer que alguna espere mucho más que otras. El mutex justo debe repartir el tiempo de forma equitativa.

**Restricciones:** El fair mutex debe tener la misma interfaz que `sync.Mutex`. El overhead sobre el mutex estándar debe ser menor de 2x para el caso sin contención.

**Pista:** Java's `ReentrantLock(fair=true)` implementa esto usando una cola de threads esperando. Go no tiene mutex justo nativo — este ejercicio lo construye desde cero con un canal y una lista de waiters.

**Implementar en:** Go · Java · Python

---

### Ejercicio 1.4.4 — Detección de starvation en tiempo de ejecución

**Enunciado:** Implementa un wrapper sobre cualquier mutex que detecta starvation: si una goroutine espera más de `umbral` milisegundos para obtener el lock, registra una advertencia con el stack trace de quién tiene el lock.

**Restricciones:** El wrapper no debe afectar el rendimiento cuando no hay contención. El overhead cuando hay contención es aceptable.

**Pista:** Esta es exactamente la funcionalidad que ofrecen herramientas como `go-deadlock` (biblioteca de Go) y Java's lock profiling. En producción, las advertencias de starvation son señales de que el diseño de locking necesita revisión.

**Implementar en:** Go · Java

---

### Ejercicio 1.4.5 — Starvation en sistemas de mensajería

**Enunciado:** Un sistema de mensajería tiene tres tipos de mensajes: Urgente, Normal y Batch. Los mensajes Urgentes siempre se procesan primero. Si el volumen de Urgentes es alto, los Batch pueden nunca procesarse.

Implementa el sistema con starvation intencional, luego arréglalo garantizando que los mensajes Batch se procesan al menos una vez cada `N` ciclos aunque haya Urgentes pendientes.

**Restricciones:** El sistema debe procesar Urgentes primero en condiciones normales. La garantía para Batch no debe afectar la latencia de Urgentes en más de un 5%.

**Pista:** Esta técnica se llama "work-stealing with quotas". Apache Kafka tiene "consumer group rebalancing" que garantiza que todos los partitions se procesan eventualmente.

**Implementar en:** Go · Java · Python · C# · Rust

---

## Sección 1.5 — Visibilidad de Memoria

**Definición:** una goroutine/hilo escribe un valor en memoria.
Otra goroutine/hilo lee esa misma posición pero ve el **valor viejo**.
No hay race condition — solo una escritura — pero el lector no ve la actualización.

**Por qué ocurre — los tres culpables:**

```
1. Caché del procesador
   Cada núcleo tiene su propio caché L1.
   Una escritura en el núcleo A puede no propagarse al núcleo B inmediatamente.

2. Reordenamiento del compilador
   El compilador puede reordenar instrucciones si cree que es equivalente.
   Para un solo hilo lo es. Para múltiples hilos puede no serlo.

3. Reordenamiento del procesador
   Los procesadores modernos ejecutan instrucciones fuera de orden
   para maximizar el rendimiento, respetando la semántica de un solo hilo.
```

**El ejemplo clásico:**

```go
var listo bool
var mensaje string

// Goroutine A (escritor)
go func() {
    mensaje = "hola mundo"
    listo = true
}()

// Goroutine B (lector)
go func() {
    for !listo {}          // espera activa
    fmt.Println(mensaje)   // ¿qué imprime?
}()

// Puede imprimir: "hola mundo"   (correcto)
// Puede imprimir: ""             (mensaje vacío)
// Puede loop infinito            (listo nunca se ve como true)
```

**Esto no es una race condition:**

```
No hay dos escrituras al mismo dato simultáneamente.
Solo A escribe, solo B lee.
El problema es que B puede ver los valores en el orden equivocado:
ver listo=true ANTES de ver mensaje="hola mundo".
```

**La solución: barreras de memoria (memory barriers)**

```go
// Con sync.Mutex o canal:
// El unlock/send actúa como barrera de escritura
// El lock/receive actúa como barrera de lectura
// Garantizan que B ve TODO lo que A escribió antes del unlock/send
```

---

### Ejercicio 1.5.1 — Demostrar el problema de visibilidad

**Enunciado:** Implementa el ejemplo del flag `listo` y demuestra que puede fallar. En Go con `-race`, este patrón es detectado como una data race. Arréglalo usando un canal en lugar de un bool compartido.

**Restricciones:** El canal debe comunicar también el valor del mensaje — no solo la señal de "listo".

**Pista:** El patrón correcto en Go para "pasar un valor de una goroutine a otra" es un canal, no una variable compartida. `ch <- mensaje` garantiza que el receptor ve todos los efectos del emisor antes del envío.

**Implementar en:** Go · Java (volatile) · C# (volatile) · Rust (Atomic)

---

### Ejercicio 1.5.2 — volatile en Java — cuándo es suficiente y cuándo no

**Enunciado:** En Java, `volatile` garantiza visibilidad pero no atomicidad. Implementa estos tres casos y determina si `volatile` es suficiente:

1. `volatile boolean activo` — un hilo escribe, otro lee
2. `volatile int contador` — múltiples hilos incrementan
3. `volatile long timestamp` — un hilo escribe, múltiples leen

**Restricciones:** Para cada caso, escribe un test que demuestre si hay bug con `volatile` y si lo hay, cuál es la solución correcta.

**Pista:** `volatile` garantiza que la lectura de `contador` da el último valor escrito, pero `contador++` (leer-incrementar-escribir) no es atómica aunque `contador` sea `volatile`. Para el caso 2, necesitas `AtomicInteger`.

**Implementar en:** Java · C# (volatile + Interlocked)

---

### Ejercicio 1.5.3 — El modelo de memoria de Go: happens-before

**Enunciado:** El modelo de memoria de Go define cuándo una escritura es garantizadamente visible para una lectura mediante la relación "happens-before". Implementa ejemplos que demuestren las garantías de happens-before de:

1. Inicialización de goroutine (`go func()` happens-before la ejecución de la goroutine)
2. Canal sin buffer (send happens-before receive)
3. Canal con buffer (el k-ésimo receive happens-before el k+1-ésimo send)
4. `sync.Mutex` (unlock happens-before el siguiente lock)
5. `sync.Once` (el return de `Do` happens-before el retorno de cualquier otra llamada a `Do`)

**Restricciones:** Para cada garantía, escribe un test que pase siempre (no flaky) gracias a happens-before.

**Pista:** La especificación del modelo de memoria de Go está en `go.dev/ref/mem`. Es corta y vale leerla. La idea central: si A happens-before B, B es garantizadamente visible para A.

**Implementar en:** Go

---

### Ejercicio 1.5.4 — Publicación segura de objetos

**Enunciado:** "Publicar" un objeto significa hacerlo visible para otras goroutines. Publicación segura garantiza que el receptor ve el objeto completamente inicializado. Implementa publicación segura e insegura y demuestra el problema:

```go
type Config struct {
    Host    string
    Puerto  int
    Timeout time.Duration
}

var config *Config  // publicación insegura

// ¿Puede otra goroutine ver config != nil pero config.Puerto == 0?
```

**Restricciones:** La publicación segura debe garantizar que nunca se ve un objeto parcialmente inicializado.

**Pista:** En Go, la forma segura es usar `sync/atomic.Pointer` o inicializar antes de lanzar goroutines. En Java, `final` fields tienen garantía especial de visibilidad — los objetos inmutables son siempre seguros de publicar sin sincronización.

**Implementar en:** Go · Java · C# · Rust

---

### Ejercicio 1.5.5 — Modelo de memoria en la práctica: Double-Checked Locking

**Enunciado:** Double-Checked Locking es un patrón para inicialización lazy de un singleton que intenta evitar el overhead del mutex en accesos subsecuentes. Sin barreras de memoria explícitas, está roto. Implementa la versión rota y la versión correcta:

```java
// Versión rota (sin volatile)
private static Singleton instancia;

public static Singleton getInstance() {
    if (instancia == null) {           // primera verificación sin lock
        synchronized (Singleton.class) {
            if (instancia == null) {   // segunda verificación con lock
                instancia = new Singleton();  // puede verse parcialmente construido
            }
        }
    }
    return instancia;
}
```

**Restricciones:** La versión correcta debe funcionar en Java 5+. Explica exactamente por qué la versión sin `volatile` está rota en términos del modelo de memoria de Java.

**Pista:** `instancia = new Singleton()` no es atómica — el compilador puede reordenar la asignación del puntero antes de completar la inicialización del objeto. Otro hilo puede ver `instancia != null` pero con campos no inicializados. `volatile` en Java 5+ previene este reordenamiento.

**Implementar en:** Java · Go (usar `sync.Once`) · C# · Rust

---

## Sección 1.6 — Detectar el tipo de problema

Con los cinco problemas claros, el desafío real en una entrevista es
**identificar cuál es el problema** antes de buscar la solución.

**El árbol de decisión:**

```
¿El programa produce resultados incorrectos?
    Sí → ¿Los resultados varían entre ejecuciones?
             Sí → Race condition (o visibilidad)
             No → Bug lógico, no de concurrencia
    No → ¿El programa se cuelga sin consumir CPU?
             Sí → Deadlock
             No → ¿El programa consume CPU pero no progresa?
                      Sí → Livelock
                      No → ¿Un agente específico nunca ejecuta?
                               Sí → Starvation
                               No → Problema de rendimiento, no de corrección
```

---

### Ejercicio 1.6.1 — Diagnóstico de código en entrevista

**Enunciado:** Para cada fragmento de código, determina sin ejecutarlo qué tipo de problema tiene (si tiene alguno), justifica el diagnóstico, y propón la solución mínima:

```go
// Caso 1
var cache = make(map[string]int)
func guardar(k string, v int) { cache[k] = v }
func leer(k string) int { return cache[k] }
// Usado desde múltiples goroutines

// Caso 2
func transferir(origen, destino *Cuenta, monto int) {
    origen.mu.Lock()
    destino.mu.Lock()
    origen.saldo -= monto
    destino.saldo += monto
    destino.mu.Unlock()
    origen.mu.Unlock()
}

// Caso 3
func worker(jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        result := procesar(job)
        results <- result
    }
}
// main lanza 5 workers y luego lee results, pero nunca cierra jobs
```

**Restricciones:** Clasifica cada caso antes de ver la solución. Luego verifica con `go run -race` o el equivalente en tu lenguaje.

**Pista:** El caso 3 es sutil — no es ninguno de los cinco problemas sino un goroutine leak: los workers nunca terminan porque `jobs` nunca se cierra. El goroutine leak no es un problema de corrección inmediato pero sí de memoria y recursos a largo plazo.

**Implementar en:** Go · Java · Python

---

### Ejercicio 1.6.2 — El bug de producción

**Enunciado:** Un servicio web en producción exhibe este síntoma: "el lunes por la mañana, después del fin de semana sin tráfico, el primer request tarda 30 segundos en lugar de 100ms". El resto del día funciona bien. Analiza el sistema simplificado y diagnostica el problema:

```go
type Pool struct {
    conexiones []*Conexion
    mu         sync.Mutex
    cond       *sync.Cond
}

func (p *Pool) Obtener() *Conexion {
    p.mu.Lock()
    defer p.mu.Unlock()
    for len(p.conexiones) == 0 {
        p.cond.Wait()
    }
    conn := p.conexiones[0]
    p.conexiones = p.conexiones[1:]
    return conn
}

func (p *Pool) Devolver(conn *Conexion) {
    p.mu.Lock()
    p.conexiones = append(p.conexiones, conn)
    p.mu.Unlock()
    // ← falta algo aquí
}
```

**Restricciones:** El diagnóstico debe explicar por qué el síntoma ocurre específicamente "el lunes por la mañana después del fin de semana".

**Pista:** `cond.Wait()` libera el mutex y espera una señal. Si `Devolver` nunca llama a `cond.Signal()`, los waiters esperan para siempre. Pero ¿por qué solo el lunes? Porque durante el fin de semana, algún proceso externo devuelve las conexiones al pool sin pasar por `Devolver`.

**Implementar en:** Go · Java · Python

---

### Ejercicio 1.6.3 — Los cinco problemas en el mismo sistema

**Enunciado:** El siguiente sistema tiene los cinco problemas. Identifica dónde está cada uno:

```go
type Sistema struct {
    cache    map[string]string  // 1. ¿?
    workers  []*Worker
    mu       sync.Mutex
    muLog    sync.Mutex
}

func (s *Sistema) Procesar(req Request) {
    s.mu.Lock()
    s.muLog.Lock()    // 2. ¿?
    log(req)
    s.muLog.Unlock()
    s.mu.Unlock()

    resultado := s.cache[req.Clave]  // acceso sin lock
    if resultado == "" {             // 3. ¿?
        resultado = calcular(req)
        s.cache[req.Clave] = resultado
    }
}

func (s *Sistema) Log(msg string) {
    s.muLog.Lock()
    s.mu.Lock()       // orden inverso al de Procesar
    escribir(msg)
    s.mu.Unlock()
    s.muLog.Unlock()
}

// Worker de alta prioridad que siempre tiene trabajo
func (s *Sistema) WorkerUrgente() {
    for {
        s.mu.Lock()   // 4. ¿?
        s.procesarUrgente()
        s.mu.Unlock()
    }
}
```

**Restricciones:** Para cada problema identificado, señala la línea exacta y describe el escenario que lo activa.

**Implementar en:** Go · Java · Python

---

### Ejercicio 1.6.4 — Construir el test que encuentra el bug

**Enunciado:** Dado un módulo de código que "a veces falla", implementa un test que encuentre el bug de forma reproducible. El módulo es una caché LRU (Cap.07 del repo de algoritmos) que se usa desde múltiples goroutines.

**Restricciones:** El test debe fallar consistentemente (no flaky) con `go test -race`. El test debe pasar después de agregar sincronización correcta.

**Pista:** Para hacer un test de concurrencia reproducible: lanza muchas goroutines (10-100), usa `sync.WaitGroup` para que todas empiecen simultáneamente (`sync.Barrier` en otros lenguajes), y verifica invariantes después de que todas terminen.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 1.6.5 — Code review: cinco problemas en veinte líneas

**Enunciado:** Este es el ejercicio de code review de entrevista. En 20 minutos, encuentra todos los problemas de concurrencia en este fragmento y propón correcciones:

```go
type Servicio struct {
    activo   bool
    contador int
    datos    []string
    mu       sync.Mutex
}

func (s *Servicio) Iniciar() {
    s.activo = true  // sin lock
    go s.procesarLoop()
}

func (s *Servicio) procesarLoop() {
    for s.activo {   // sin lock
        s.mu.Lock()
        if len(s.datos) > 0 {
            dato := s.datos[0]
            s.datos = s.datos[1:]
            s.contador++
            s.mu.Unlock()
            s.procesar(dato)
        } else {
            s.mu.Unlock()
            time.Sleep(10 * time.Millisecond)
        }
    }
}

func (s *Servicio) Agregar(dato string) {
    s.mu.Lock()
    s.datos = append(s.datos, dato)
    s.mu.Unlock()
}

func (s *Servicio) Detener() {
    s.activo = false  // sin lock
}

func (s *Servicio) Contador() int {
    return s.contador  // sin lock
}
```

**Restricciones:** Cronométrate. En una entrevista real tendrías 15-20 minutos para este análisis. Luego escribe la versión corregida y verifica con `-race`.

**Pista:** Hay al menos 4 problemas distintos. Algunos son race conditions obvias, uno es más sutil (relacionado con visibilidad).

**Implementar en:** Go · Java · Python · Rust

---

## Sección 1.7 — El Detector: -race de Go y Herramientas Equivalentes

El mejor diagnóstico de bugs de concurrencia no es el análisis estático sino
**ejecutar el código con un detector activo**.

**Go — el detector de races:**

```bash
# Compilar y ejecutar con el race detector
go run -race main.go
go test -race ./...
go build -race -o servicio_debug ./...

# Output cuando encuentra una race condition:
==================
WARNING: DATA RACE
Write at 0x00c0001b4010 by goroutine 7:
  main.incrementar()
      /home/user/contador.go:12 +0x44

Previous read at 0x00c0001b4010 by goroutine 6:
  main.incrementar()
      /home/user/contador.go:12 +0x34

Goroutine 7 (running) created at:
  main.main()
      /home/user/contador.go:20 +0x84
==================

# El detector tiene overhead: ~5-10x más lento, ~5-10x más memoria
# Úsalo en tests y staging, no en producción (normalmente)
```

**Herramientas equivalentes:**

```
Java:     ThreadSanitizer (TSan) via -fsanitize=thread en JVM nativa
          IntelliJ IDEA: análisis estático de concurrencia
          FindBugs/SpotBugs: detecta patrones comunes

Python:   No hay race detector nativo (el GIL enmascara muchos bugs)
          threading.settrace() para debugging manual
          pytest-timeout para detectar deadlocks en tests

Rust:     No necesita race detector: el compilador rechaza data races
          El borrow checker garantiza que las race conditions no compilan
          `cargo test` es suficiente para la mayoría de problemas

C#:       Concurrency Visualizer en Visual Studio
          CHESS: herramienta de Microsoft para testing sistemático
```

---

### Ejercicio 1.7.1 — Usar el race detector de Go

**Enunciado:** Toma los ejercicios 1.1.1, 1.2.1, 1.4.1 y 1.5.1. Para cada uno, ejecuta primero sin `-race` y observa si el bug es visible. Luego ejecuta con `-race` y compara el output. Interpreta el reporte del race detector: identifica las goroutines involucradas, las líneas de código, y el tipo de operación (read/write).

**Restricciones:** Documenta el output completo del race detector para cada caso. Explica por qué algunos bugs son invisibles sin el detector.

**Pista:** El race detector de Go usa Shadow Memory — mantiene metadatos para cada byte de memoria rastreando qué goroutine lo escribió por última vez y cuándo. El overhead de memoria es proporcional al tamaño del heap del programa.

**Implementar en:** Go

---

### Ejercicio 1.7.2 — ThreadSanitizer en Java/C++

**Enunciado:** Java no tiene un race detector tan integrado como Go. Usa una de estas alternativas para detectar la race condition del Ejercicio 1.1.2:
- `synchronized` faltante detectado por IntelliJ
- Prueba de estrés: ejecutar el test 10,000 veces con `-Xss` pequeño
- Instrumentación manual con `java.util.concurrent.atomic.AtomicInteger`

**Restricciones:** El método elegido debe detectar el bug de forma consistente (no depender de timing).

**Pista:** La ausencia de un race detector nativo en Java es una razón por la que los bugs de concurrencia en Java son más difíciles de encontrar. Los tests de estrés ayudan pero no garantizan encontrar el bug — solo aumentan la probabilidad.

**Implementar en:** Java · C#

---

### Ejercicio 1.7.3 — Por qué Rust no necesita race detector

**Enunciado:** Intenta implementar la race condition del contador del inicio del capítulo en Rust. El compilador debe rechazarlo. Describe exactamente qué error da el compilador y por qué el sistema de ownership previene la data race.

Luego implementa la versión correcta usando `Arc<Mutex<i32>>` o `Arc<AtomicI32>` y explica la diferencia de garantías entre los dos.

**Restricciones:** El código incorrecto debe no compilar — ese es el punto. El código correcto debe compilar y funcionar sin race detector.

**Pista:** Rust garantiza en compilación: si tienes una referencia `&mut T` (mutable), no puede haber ninguna otra referencia al mismo `T`. Esto hace imposibles las data races por construcción — no puedes tener dos threads con `&mut contador` simultáneamente.

**Implementar en:** Rust

---

### Ejercicio 1.7.4 — Construir un test de concurrencia robusto

**Enunciado:** Un test de concurrencia bien diseñado debe:
1. Ser reproducible (no flaky — no falla aleatoriamente)
2. Maximizar la probabilidad de encontrar el bug
3. Verificar invariantes, no solo "no crash"

Implementa un framework mínimo de testing de concurrencia que: lanza N goroutines simultáneamente usando una barrera, ejecuta operaciones mixtas (lectura/escritura), y verifica invariantes del estado final.

**Restricciones:** El framework debe funcionar con cualquier estructura de datos. El test debe fallar antes de agregar sincronización y pasar después.

**Pista:** La barrera de inicio es crucial: `wg.Add(N); for i in range(N): go func() { barrera.Wait(); operacion() }`. Sin barrera, las goroutines se lanzan secuencialmente y la probabilidad de intercalación es baja.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 1.7.5 — Integrar el race detector en CI/CD

**Enunciado:** Integra `go test -race` en un pipeline de CI/CD (GitHub Actions, GitLab CI, o similar). El pipeline debe:
1. Ejecutar todos los tests con el race detector
2. Fallar si se detecta cualquier race condition
3. Reportar el output del detector como artefacto del build

**Restricciones:** El pipeline debe correr en menos de 5 minutos para un repositorio pequeño. El overhead del race detector (~5-10x) es aceptable en CI.

**Pista:**
```yaml
# .github/workflows/race.yml
- name: Test with race detector
  run: go test -race -timeout 5m ./...
```
El `-timeout 5m` es importante: un deadlock en los tests haría que el pipeline nunca terminara sin el timeout.

**Implementar en:** Go (GitHub Actions) · Java (Maven Surefire + ThreadSanitizer)

---

## Resumen del capítulo

| Problema | Síntoma | Condición necesaria | Solución general |
|---|---|---|---|
| **Race condition** | Resultados incorrectos e intermitentes | Acceso compartido + escritura + sin sincronización | Mutex, canal, atómico, o eliminar lo compartido |
| **Deadlock** | El programa se cuelga, CPU al 0% | Espera circular entre agentes | Orden de bloqueos, timeout, evitar múltiples locks |
| **Livelock** | El programa no progresa, CPU alta | Reacción simétrica entre agentes | Aleatoriedad, asimetría, backoff exponencial |
| **Starvation** | Un agente nunca ejecuta | Prioridad injusta o contención alta | Fair mutex, aging, quotas |
| **Visibilidad** | Un agente no ve cambios de otro | Sin barrera de memoria | volatile, canal, mutex |

## La pregunta que guía todo lo siguiente

> ¿Cómo eliminamos lo compartido en lugar de protegerlo?

El Cap.02 responde con el modelo de memoria formal.
Los Cap.03–05 responden con las herramientas de sincronización.
El Cap.06 responde con modelos que evitan el estado compartido por diseño.

La concurrencia no es difícil porque los locks son complicados.
Es difícil porque **el estado compartido mutable es difícil de razonar**.
Las soluciones más elegantes eliminan el estado compartido — no lo protegen.
