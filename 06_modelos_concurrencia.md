# Guía de Ejercicios — Modelos de Concurrencia: Actores, CSP y STM

> Implementar cada ejercicio en: **Go · Erlang/Elixir · Java (Akka) · Python**
> según el modelo que se estudia.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el mismo problema, tres soluciones distintas

El problema: un contador compartido que 1000 goroutines/threads incrementan.

Ya lo resolvimos con mutex (Cap.02) y con canales (Cap.05).
Ahora lo veremos desde tres perspectivas radicalmente distintas.

**Modelo 1 — Actores (Erlang/Akka):**

```erlang
%% El contador es un proceso. Solo él modifica su estado.
%% Los demás le envían mensajes.

counter(N) ->
    receive
        increment -> counter(N + 1);
        {get, From} -> From ! N, counter(N)
    end.

% 1000 procesos envían increment
% El contador procesa los mensajes secuencialmente — sin race condition posible
```

**Modelo 2 — CSP / Go (Communicating Sequential Processes):**

```go
// Un canal transfiere valores. Las goroutines son anónimas — no tienen identidad.
// La comunicación ocurre en los canales, no entre procesos específicos.

incrementos := make(chan struct{}, 1000)
resultado := make(chan int)

go func() {  // la goroutine propietaria del estado
    n := 0
    for range incrementos {
        n++
    }
    resultado <- n
}()
```

**Modelo 3 — STM (Software Transactional Memory):**

```haskell
-- Haskell con STM. El runtime garantiza que las transacciones
-- son atómicas, consistentes y aisladas — como una base de datos.

contador :: TVar Int
counter <- newTVar 0

-- Cada transacción es atómica:
atomically $ modifyTVar counter (+1)
-- Si dos transacciones entran en conflicto, una reintenta automáticamente
```

**La diferencia fundamental:**

```
Mutex:    "Solo yo puedo tocar esto ahora" — exclusión
Actores:  "Solo yo tengo este estado — envíame mensajes" — ownership permanente
CSP:      "El dato viaja entre procesos" — transferencia de ownership
STM:      "Esta secuencia es una transacción" — composición atómica
```

---

## Por qué importa conocer estos modelos

En entrevistas senior aparecen preguntas como:
- "¿Cómo diseñarías el sistema de chat de Discord?" → Actores
- "¿Por qué Go eligió CSP sobre el modelo de actores?" → diferencias fundamentales
- "¿Cómo harías este update de dos cuentas bancarias sin deadlock?" → STM
- "¿Cuándo usarías Akka en lugar de threads de Java?" → tradeoffs

No es necesario ser experto en Erlang para responderlas bien. Lo que se
necesita es entender el modelo mental de cada uno.

---

## Tabla de contenidos

- [Sección 6.1 — El modelo de actores](#sección-61--el-modelo-de-actores)
- [Sección 6.2 — CSP vs Actores: la diferencia que importa](#sección-62--csp-vs-actores-la-diferencia-que-importa)
- [Sección 6.3 — Actores en Go: simulación](#sección-63--actores-en-go-simulación)
- [Sección 6.4 — Software Transactional Memory](#sección-64--software-transactional-memory)
- [Sección 6.5 — Reactive Streams y back-pressure formal](#sección-65--reactive-streams-y-back-pressure-formal)
- [Sección 6.6 — Elegir el modelo correcto](#sección-66--elegir-el-modelo-correcto)
- [Sección 6.7 — Conexión con sistemas distribuidos](#sección-67--conexión-con-sistemas-distribuidos)

---

## Sección 6.1 — El Modelo de Actores

El modelo de actores fue formalizado por Carl Hewitt en 1973. La idea central:

```
Un actor es la unidad fundamental de computación. Tiene:
  — Comportamiento: qué hace cuando recibe un mensaje
  — Estado:         datos que solo él puede leer y modificar
  — Mailbox:        cola de mensajes entrantes (procesada secuencialmente)

Cuando recibe un mensaje, un actor puede:
  1. Enviar mensajes a otros actores (incluyendo a sí mismo)
  2. Crear nuevos actores
  3. Cambiar su comportamiento para el próximo mensaje
```

**Las tres garantías del modelo de actores:**

```
1. Un actor procesa un mensaje a la vez — no hay concurrencia interna
2. El estado de un actor es completamente privado — nadie puede leerlo directamente
3. La única forma de comunicarse es por mensajes — no hay memoria compartida
```

**Por qué el modelo de actores elimina las race conditions:**

```
Race condition requiere:
  ✓ Acceso compartido a un dato
  ✓ Al menos una escritura
  ✓ Sin sincronización

Con actores:
  ✗ No hay acceso compartido — el estado es privado
  → Race condition es imposible por diseño
```

**Sistemas de producción basados en actores:**

```
Erlang/OTP:   el modelo original, usado en telecomunicaciones
Elixir:       Erlang con sintaxis moderna — Phoenix, Ecto
Akka:         actores en JVM (Java/Scala) — usado en Lightbend, LinkedIn
Microsoft Orleans: actores en .NET — usado en Halo, Azure
```

---

### Ejercicio 6.1.1 — Implementar el modelo de actores desde cero en Go

**Enunciado:** Implementa una biblioteca mínima de actores en Go:

```go
type Actor struct {
    mailbox chan Mensaje
    estado  any
    handler func(estado any, msg Mensaje) any
}

type Mensaje struct {
    Tipo    string
    Cuerpo  any
    Remite  chan<- any  // canal de respuesta opcional
}

func NuevoActor(estadoInicial any, handler func(any, Mensaje) any) *Actor
func (a *Actor) Enviar(msg Mensaje)
func (a *Actor) Pedir(msg Mensaje, timeout time.Duration) (any, error)
```

**Restricciones:** El handler procesa un mensaje a la vez — internamente secuencial.
`Pedir` es el patrón ask: envía mensaje y espera respuesta con timeout.
El actor arranca su goroutine interna al crearse.

**Pista:** La goroutina interna del actor es simplemente:
```go
for msg := range a.mailbox {
    a.estado = a.handler(a.estado, msg)
}
```
El estado nunca sale del actor — el handler recibe el estado actual y retorna
el estado nuevo. Es puro: sin efectos secundarios sobre el estado.

**Implementar en:** Go · Java (sin Akka, implementar desde cero) · Python

---

### Ejercicio 6.1.2 — Sistema de chat con actores

**Enunciado:** Implementa un sistema de chat donde cada usuario es un actor:

```
Actores:
  ChatRoom  — gestiona la lista de usuarios, redistribuye mensajes
  Usuario   — tiene su nombre y su historial de mensajes recibidos

Mensajes:
  {Tipo: "unirse",   Cuerpo: nombre}
  {Tipo: "mensaje",  Cuerpo: texto, Remite: usuarioID}
  {Tipo: "salir",    Cuerpo: usuarioID}
  {Tipo: "historial", Remite: canal_respuesta}
```

**Restricciones:** El ChatRoom redistribuye cada mensaje a todos los
usuarios excepto al remitente. Sin locks — el estado de cada actor
es completamente privado. Verifica con `-race`.

**Pista:** El ChatRoom mantiene `map[string]*Actor` de usuarios activos.
Cuando recibe un mensaje de tipo "mensaje", itera el mapa y envía
a cada usuario. Esta iteración ocurre en la goroutine del ChatRoom —
por tanto es secuencial y sin race condition.

**Implementar en:** Go · Elixir/Erlang · Java (Akka) · Python

---

### Ejercicio 6.1.3 — Supervisión y tolerancia a fallos

**Enunciado:** Una de las fortalezas de Erlang es "let it crash" + supervisores.
Un supervisor monitorea actores y los reinicia cuando fallan.

Implementa un supervisor básico:

```go
type Supervisor struct {
    actores   map[string]*ActorInfo
    estrategia Estrategia  // OneForOne, AllForOne, RestForOne
}

// OneForOne: si A falla, solo reiniciar A
// AllForOne: si A falla, reiniciar todos
// RestForOne: si A falla, reiniciar A y los creados después de A
```

**Restricciones:** El supervisor detecta cuándo un actor termina (por fallo o normal).
Reinicia con backoff exponencial hasta `maxReintentos`.
Si un actor falla más de N veces en M segundos, el supervisor también falla.

**Pista:** Para detectar el fin de un actor, usa un canal `done` que el actor
cierra al terminar. El supervisor hace `select` sobre los canales `done` de
todos sus actores. Cuando uno se cierra, decide según la estrategia qué reiniciar.

**Implementar en:** Go · Elixir (OTP Supervisor) · Java (Akka SupervisorStrategy)

---

### Ejercicio 6.1.4 — Actores con estado tipado

**Enunciado:** La implementación del Ejercicio 6.1.1 usa `any` para el estado
y los mensajes — no es type-safe. Implementa una versión con generics que
preserva el tipo del estado:

```go
type Actor[S any, M any] struct {
    mailbox chan M
    estado  S
    handler func(S, M) S
}

func NuevoActor[S, M any](inicial S, h func(S, M) S) *Actor[S, M]
func (a *Actor[S, M]) Enviar(msg M)
```

**Restricciones:** El compilador debe rechazar mensajes del tipo incorrecto.
El estado solo es accesible dentro del handler.

**Pista:** Go generics (1.18+) permiten parametrizar tanto el tipo del estado
como el tipo del mensaje. El handler es una función pura: recibe estado
y mensaje, retorna nuevo estado. Esto es exactamente el modelo de un
reducer de Redux — el mismo patrón, aplicado a concurrencia.

**Implementar en:** Go · Rust · Java

---

### Ejercicio 6.1.5 — Benchmarking: actores vs mutex para el problema del contador

**Enunciado:** Compara el rendimiento del modelo de actores vs mutex para
el problema clásico del contador con 1, 4, 16, y 64 goroutines concurrentes.

**Restricciones:** El actor procesa mensajes secuencialmente — esto puede ser
más lento que un atómico o mutex para el contador simple. Mide y reporta
cuándo el modelo de actores escala mejor.

**Pista:** Para un contador simple, el mutex/atómico gana siempre — el overhead
del canal del actor es innecesario. Los actores brillan cuando el estado es
complejo y el número de mensajes por actor es alto. El breakeven depende de
la ratio entre overhead del canal y el tiempo de procesamiento por mensaje.

**Implementar en:** Go · Java · Erlang/Elixir

---

## Sección 6.2 — CSP vs Actores: La Diferencia que Importa

Go está basado en CSP (Communicating Sequential Processes), formalizado por
Tony Hoare en 1978 — cinco años después de los actores de Hewitt. Son similares
pero con una diferencia filosófica importante.

**La diferencia central:**

```
Actores:
  — Los procesos tienen identidad (PID en Erlang, ActorRef en Akka)
  — Se envía a un proceso específico: send(pid, mensaje)
  — El proceso tiene su propio mailbox
  — La comunicación es asimétrica: emisor → receptor específico

CSP:
  — Los canales tienen identidad, no los procesos
  — Se envía a un canal: ch <- mensaje
  — Las goroutines son anónimas — no tienen PID
  — La comunicación es simétrica: cualquiera puede enviar/recibir del canal
```

**Implicaciones prácticas:**

```
Con actores:
  Si quiero hablar con el gestor de caché, necesito su referencia (PID/ActorRef)
  Puedo enviarle mensajes aunque esté en otra máquina — transparencia de ubicación

Con CSP (Go):
  Si quiero hablar con el gestor de caché, necesito su canal
  El canal es local — para redes necesitas otra abstracción (gRPC, etc.)
  Múltiples goroutines pueden compartir el mismo canal (fan-out/fan-in natural)
```

**Por qué Go eligió CSP:**

Go's design decision: los canales son más composables que los actores para
el caso local. Fan-out, fan-in, pipelines — todos son naturales con canales.
Con actores, distribuir trabajo a múltiples workers requiere más infraestructura.

Para sistemas distribuidos, los actores ganan en transparencia de ubicación
— el código no cambia si el actor está en el mismo proceso o en otra máquina.

---

### Ejercicio 6.2.1 — El mismo sistema en CSP y actores

**Enunciado:** Implementa un sistema de procesamiento de pedidos en dos versiones:

**Versión CSP (Go):**
- Canal `pedidos` donde el parser pone pedidos parseados
- Canal `validados` donde el validador pone pedidos válidos
- Canal `procesados` donde el procesador pone resultados

**Versión Actores:**
- Actor `Parser` que recibe strings y envía a `Validador`
- Actor `Validador` que valida y envía a `Procesador`
- Actor `Procesador` que procesa y envía al `Colector`

**Restricciones:** Ambas versiones tienen la misma lógica de negocio.
Compara: legibilidad, manejo de errores, y facilidad de añadir una
cuarta etapa al medio.

**Pista:** Añadir una etapa al pipeline CSP es: crear un nuevo canal,
conectar la goroutine nueva entre los dos canales existentes.
Con actores, el Actor B necesita conocer al nuevo Actor C (referencia directa).
CSP es más composable para pipelines; actores son más naturales para
topologías con estado complejo.

**Implementar en:** Go (CSP) · Go (actores con la lib del Ejercicio 6.1.1)

---

### Ejercicio 6.2.2 — Transparencia de ubicación: el argumento de los actores

**Enunciado:** Implementa un sistema que funcione igual con actores locales
y con actores remotos (en otro proceso). La transparencia de ubicación
significa que el código que usa el actor no sabe si es local o remoto.

```go
type ActorRef interface {
    Enviar(msg Mensaje)
    Pedir(msg Mensaje, timeout time.Duration) (any, error)
}

// Implementaciones:
type ActorLocal struct { /* actor en el mismo proceso */ }
type ActorRemoto struct { /* actor en otro proceso via gRPC */ }
```

**Restricciones:** El código que usa `ActorRef` no cambia entre local y remoto.
El Actor remoto serializa el mensaje (JSON), lo envía por red, deserializa la respuesta.

**Pista:** La interfaz `ActorRef` abstrae el mecanismo de comunicación.
Esto es exactamente lo que hace Akka — `ActorRef` en Akka funciona igual
si el actor está en el mismo JVM o en otro nodo del cluster.

**Implementar en:** Go · Java (Akka remote actors)

---

### Ejercicio 6.2.3 — Broadcast: actores vs canales

**Enunciado:** El problema: notificar a 100 suscriptores cuando ocurre un evento.

**Con actores:** El broker tiene referencias a todos los suscriptores.
Al recibir un evento, itera la lista y envía a cada uno.

**Con canales (Go):** El broker tiene una lista de canales.
Al recibir un evento, itera y envía a cada canal (el patrón Pub/Sub del Cap.03).

Implementa ambos y compara: ¿Qué pasa si un suscriptor es lento?
¿Qué pasa si un suscriptor falla?

**Restricciones:** En ambos casos, un suscriptor lento no debe bloquear
al broker ni a los otros suscriptores. Implementa la política de drop
y la política de backpressure.

**Pista:** Con actores, el mailbox del actor lento se acumula — el broker
no se bloquea. Con canales, si el canal del suscriptor lento está lleno,
el broker se bloquea a menos que use `select` con `default` (drop).
Los actores tienen backpressure natural por la capacidad del mailbox.

**Implementar en:** Go · Elixir · Java (Akka)

---

### Ejercicio 6.2.4 — Dead letter mailbox: mensajes sin destinatario

**Enunciado:** En Erlang/Akka, si envías a un actor que ya terminó,
el mensaje va al "dead letter mailbox" — no se pierde silenciosamente.
Implementa este mecanismo:

```go
type Sistema struct {
    actores    map[string]*Actor
    deadLetter chan Mensaje
    mu         sync.RWMutex
}

func (s *Sistema) Enviar(destinatario string, msg Mensaje) {
    s.mu.RLock()
    actor, ok := s.actores[destinatario]
    s.mu.RUnlock()

    if !ok {
        s.deadLetter <- msg  // el actor no existe o ya terminó
        return
    }
    actor.Enviar(msg)
}
```

**Restricciones:** El dead letter mailbox tiene un consumidor que loguea
o reencamina los mensajes. Verifica que ningún mensaje se pierde silenciosamente.

**Pista:** El dead letter mailbox es una de las herramientas de debugging
más valiosas de Akka. En producción, monitorear el rate de dead letters
ayuda a detectar bugs donde actores intentan comunicarse con actores
que ya terminaron o nunca existieron.

**Implementar en:** Go · Java (Akka EventStream para dead letters)

---

### Ejercicio 6.2.5 — ¿Cuándo elegir actores y cuándo canales?

**Enunciado:** Para cada sistema, determina si actores o canales es la
abstracción más natural y por qué:

1. **Sistema de caché distribuida** — múltiples nodos, cada nodo gestiona su fragmento
2. **Pipeline de transformación de datos** — lectura, parse, validación, escritura
3. **Sistema de reservas de asientos** — estado de cada asiento, múltiples agentes reservando
4. **Agregación de métricas** — 1000 fuentes enviando métricas a un solo agregador
5. **Sistema de cola de emails** — reintentos, prioridades, dead letter queue

**Restricciones:** Para cada caso, implementa la solución con la abstracción
elegida. Si elegiste actores, implementa con la lib del Ejercicio 6.1.1.
Si elegiste canales, implementa con los patrones del Cap.03.

**Pista:** Los actores son más naturales cuando el estado tiene identidad
(asiento 42A, nodo 3 del cluster, usuario X). Los canales son más naturales
cuando el dato fluye sin identidad permanente (el stream de métricas,
el pipeline de transformación).

**Implementar en:** Go · Elixir (para los casos con actores)

---

## Sección 6.3 — Actores en Go: Simulación

Go no es un lenguaje de actores — pero el patrón "goroutine propietaria del estado"
del Cap.05 §5.1.3 es funcionalmente equivalente a un actor.

**Las diferencias con actores "reales" (Erlang/Akka):**

```
Actores reales (Erlang):          Go con goroutine propietaria:
  Supervisión automática       vs  Supervisión manual (Cap.03 §3.5.5)
  Transparencia de ubicación   vs  Local solamente (necesita gRPC para remoto)
  Hot code reload              vs  No disponible
  Let it crash + reinicio      vs  recover() + lógica manual
  Mailbox con prioridades      vs  Canal con buffer
  Monitoreo de procesos (link) vs  context.WithCancel + WaitGroup
```

**Cuándo simular actores en Go tiene sentido:**

```
✅ El sistema es local (no distribuido)
✅ El equipo conoce Go pero no Erlang/Akka
✅ La tolerancia a fallos necesaria es básica
✅ El número de "actores" es manejable (decenas, no millones)

❌ Necesitas transparencia de ubicación real
❌ Necesitas supervisión sofisticada (OTP)
❌ El sistema tiene millones de actores con ciclos de vida cortos
   (Erlang crea/destruye actores en nanosegundos — Go es más caro)
```

---

### Ejercicio 6.3.1 — Sistema bancario con actores en Go

**Enunciado:** Implementa un sistema bancario donde cada cuenta es un actor:

```go
type CuentaActor struct {
    id       string
    saldo    int
    mailbox  chan MensajeCuenta
}

type MensajeCuenta struct {
    Tipo     string  // "depositar", "retirar", "consultar", "transferir"
    Monto    int
    Destino  string  // para transferir
    Responde chan<- RespuestaCuenta
}
```

**Restricciones:** Las transferencias entre cuentas no deben tener deadlock.
¿Cómo se hace una transferencia atómica sin que dos actores se esperen mutuamente?

**Pista:** Con actores, la transferencia se convierte en una secuencia de mensajes:
(1) El actor Cuenta A recibe "transferir 100 a Cuenta B",
(2) Cuenta A se debita a sí misma,
(3) Cuenta A envía "depositar 100" a Cuenta B.
Si el depósito falla (Cuenta B no existe), Cuenta A se auto-credita.
No hay deadlock porque los actores no se esperan mutuamente — usan mensajes.

**Implementar en:** Go · Elixir · Java (Akka)

---

### Ejercicio 6.3.2 — Actor con comportamiento dinámico (become)

**Enunciado:** En Erlang, un actor puede cambiar su comportamiento con `become` —
cómo procesa mensajes futuros cambia según su estado actual.

Implementa un semáforo como actor con comportamiento dinámico:

```
Estado LIBRE:
  Recibe "adquirir" → cambia a OCUPADO, responde "ok"
  Recibe "liberar"  → ignora (ya está libre)

Estado OCUPADO:
  Recibe "adquirir" → encola el request, permanece OCUPADO
  Recibe "liberar"  → procesa el primer request encolado, responde "ok"
                      si cola vacía → cambia a LIBRE
```

**Restricciones:** El semáforo es justo — responde en orden FIFO.
La cola de requests pendientes es parte del estado del actor.

**Pista:** "Become" en Go se implementa cambiando la función handler en el estado:
```go
type EstadoSemaforo struct {
    ocupado   bool
    pendientes []chan<- bool
    handler    func(EstadoSemaforo, Msg) EstadoSemaforo  // cambiable
}
```
O más simplemente, con un campo `ocupado` y lógica condicional en el handler.

**Implementar en:** Go · Elixir (`become` nativo)

---

### Ejercicio 6.3.3 — Cluster de actores: ring topology

**Enunciado:** Crea un anillo de N actores donde cada mensaje circula
por todos antes de volver al origen. Esto simula el problema clásico
de Dijkstra del anillo de procesos comunicantes.

```
A → B → C → D → A
```

Cada actor recibe el mensaje, lo modifica (suma su ID), y lo pasa al siguiente.
Mide cuántas veces puede circular el mensaje en 1 segundo.

**Restricciones:** El anillo debe funcionar con N = {4, 16, 64, 256} actores.
El throughput debe escalar sublinealmente con N (más actores = más overhead de mensajes).

**Pista:** Este benchmark es clásico para medir el overhead de message passing.
En Erlang, 256 actores en anillo pueden hacer ~1M de round-trips por segundo.
En Go con canales, el número depende del overhead del canal y del scheduler.
Mide y compara.

**Implementar en:** Go · Elixir · Java (Akka)

---

### Ejercicio 6.3.4 — Sharding de actores para escalabilidad

**Enunciado:** Con 1 millón de "cuentas" (actores), tener una goroutina
por cuenta es posible pero costoso. Implementa sharding: N workers
que manejan las cuentas por hash del ID.

```go
type RegistroCuentas struct {
    shards []*ShardCuentas
    N      int
}

func (r *RegistroCuentas) shard(cuentaID string) *ShardCuentas {
    h := fnv.New32a()
    h.Write([]byte(cuentaID))
    return r.shards[h.Sum32()%uint32(r.N)]
}
```

**Restricciones:** Cada shard gestiona su propio mapa de cuentas.
Las operaciones de una cuenta siempre van al mismo shard.
Operaciones entre cuentas en shards distintos necesitan coordinación.

**Pista:** El sharding convierte el problema de N millones de actores en
N workers, donde cada worker es un actor que gestiona miles de cuentas.
Las transacciones cross-shard son el reto: secuencia de mensajes en lugar
de lock de dos fases, igual que el Ejercicio 6.3.1.

**Implementar en:** Go · Java · Python

---

### Ejercicio 6.3.5 — Comparar overhead: actor vs mutex vs atómico

**Enunciado:** Benchmark comparativo para el problema del contador con 64 goroutines:

```
Implementación 1: Atómico         (~10 ns por operación)
Implementación 2: Mutex           (~50 ns sin contención, ~500 ns con)
Implementación 3: Canal (CSP)     (~200 ns por mensaje)
Implementación 4: Actor (goroutina propietaria) (~300 ns por mensaje)
```

**Restricciones:** Mide con distintos ratios read/write (100% write, 50/50, 90% read).
Para 90% lectura, RWMutex y el actor tienen comportamientos distintos — ¿por qué?

**Pista:** El actor (goroutina propietaria) serializa todas las operaciones —
incluso las lecturas. Para 90% lectura, RWMutex permite parallelismo de lecturas.
El actor no. Para ese caso, RWMutex gana claramente. El actor gana cuando
la lógica de actualización es compleja y las invariantes son difíciles de
mantener con locks.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 6.4 — Software Transactional Memory

STM toma prestada la idea de las transacciones de las bases de datos
y la aplica a la memoria compartida.

**La idea central:**

```
En lugar de: "Solo yo puedo modificar este dato" (mutex)
STM dice:    "Esta secuencia de operaciones es una transacción"

atomically $ do
    saldoA <- readTVar cuentaA
    saldoB <- readTVar cuentaB
    when (saldoA >= monto) $ do
        writeTVar cuentaA (saldoA - monto)
        writeTVar cuentaB (saldoB + monto)
```

**Las garantías de STM:**

```
Atomicidad:   todas las operaciones de la transacción ocurren, o ninguna
Consistencia: la transacción ve un snapshot consistente del estado
Aislamiento:  transacciones concurrentes no interfieren entre sí
              Si hay conflicto, una reintenta automáticamente
```

**El problema que resuelve STM elegantemente:**

```go
// Con mutex — el problema de transferencia entre cuentas (Cap.01 §1.2.4)
// Necesita ordenar los locks para evitar deadlock:
func transferir(a, b *Cuenta, monto int) {
    if a.id < b.id { a.mu.Lock(); b.mu.Lock() }
    else            { b.mu.Lock(); a.mu.Lock() }
    defer a.mu.Unlock()
    defer b.mu.Unlock()
    // la lógica de negocio tiene que cargar con la lógica de locking
}

// Con STM — composición directa:
atomically $ transferir cuentaA cuentaB monto
// La transacción falla y reintenta si hay conflicto — automáticamente
```

**STM en Go: no existe nativamente, pero se puede simular**

Go no tiene STM. Haskell (STM library) y Clojure (refs) son los lenguajes
mainstream con STM de primera clase. En Go, el patrón más cercano es:
- Atómicos para una variable
- Mutex para varias variables relacionadas
- Canales para transferencia de ownership

La biblioteca `github.com/nicholasgasior/gostm` existe pero no es ampliamente usada.

---

### Ejercicio 6.4.1 — Simular STM con versiones y retry

**Enunciado:** Implementa una STM simplificada para Go usando
optimistic concurrency control:

```go
type TVar[T any] struct {
    versión int64
    valor   T
    mu      sync.RWMutex
}

type Transaccion struct {
    lecturas  map[*tvarBase]int64  // TVar → versión leída
    escrituras map[*tvarBase]any   // TVar → valor nuevo
}

func Atomically(f func(*Transaccion) error) error
```

**Restricciones:** `Atomically` ejecuta `f`. Si al commitear algún TVar
tiene una versión distinta a la leída (conflicto), descarta y reintenta.
Si no hay conflicto, aplica todas las escrituras atómicamente.

**Pista:** El commit es una sección crítica: verificar versiones y escribir
nuevos valores debe ser atómico. Un mutex global para el commit es correcto
aunque no óptimo. La optimización es usar un lock por TVar y lock-ordering.

**Implementar en:** Go · Clojure (STM nativo) · Haskell (STM nativo)

---

### Ejercicio 6.4.2 — Transferencia bancaria con STM

**Enunciado:** Implementa el problema de transferencia entre cuentas usando
la STM del Ejercicio 6.4.1. La solución no debe tener ninguna lógica
de locking explícita — solo operaciones `readTVar`, `writeTVar` y `atomically`.

```go
func Transferir(cuentaA, cuentaB *TVar[int], monto int) error {
    return Atomically(func(tx *Transaccion) error {
        saldoA := tx.Read(cuentaA)
        saldoB := tx.Read(cuentaB)
        if saldoA < monto {
            return ErrSaldoInsuficiente
        }
        tx.Write(cuentaA, saldoA-monto)
        tx.Write(cuentaB, saldoB+monto)
        return nil
    })
}
```

**Restricciones:** Lanza 100 goroutines haciendo transferencias concurrentes.
La suma total de todas las cuentas debe ser constante al final.
Sin deadlock, sin starvation.

**Pista:** La belleza de STM para este problema: no hay lógica de ordenamiento
de locks, no hay posibilidad de deadlock. Si hay conflicto, la transacción
reintenta automáticamente. La invariante (suma constante) se verifica con un
scan de todas las cuentas dentro de una transacción — snapshot consistente.

**Implementar en:** Go · Haskell · Clojure

---

### Ejercicio 6.4.3 — retry y orElse en STM

**Enunciado:** Haskell STM tiene dos operaciones especiales: `retry` y `orElse`.

- `retry`: aborta la transacción y la reintenta cuando algún TVar leído cambia
- `orElse a b`: intenta `a`; si `a` hace `retry`, intenta `b`

Implementa estas operaciones en la STM simplificada:

```go
// retry: espera hasta que algún TVar leído cambie
// (bloquea de forma eficiente — no polling)

func (tx *Transaccion) Retry()

// orElse: intenta tx1; si hace retry, intenta tx2
func OrElse(tx1, tx2 func(*Transaccion) error) error
```

**Restricciones:** `Retry` no debe hacer busy-waiting — debe suspender
la goroutine hasta que un TVar cambie. Usa condition variables o canales.

**Pista:** Cuando la transacción hace `retry`, registra los TVars que leyó.
Luego se suscribe a notificaciones de cualquiera de esos TVars.
Cuando alguno cambia, despierta y reintenta la transacción.
`orElse` ejecuta tx1 en una "transacción especulativa" — si hace retry,
descarta y ejecuta tx2.

**Implementar en:** Go · Haskell (nativo)

---

### Ejercicio 6.4.4 — STM vs mutex: cuándo cada uno gana

**Enunciado:** Implementa el mismo escenario (100 goroutines modificando
un grafo de 1000 nodos con invariantes globales) con STM y con mutexes.
Compara:

1. Claridad del código
2. Corrección: ¿es más fácil verificar las invariantes?
3. Rendimiento bajo alta contención (80% de transacciones en conflicto)
4. Rendimiento bajo baja contención (5% de transacciones en conflicto)

**Restricciones:** Las invariantes del grafo: suma de pesos = constante,
no hay ciclos negativos. Verificar al final con un scan completo.

**Pista:** STM tiene overhead significativo bajo alta contención — las
transacciones reintentan muchas veces. Bajo baja contención, STM es comparable
a mutex (o más rápido porque no hay lock contention). El punto de quiebre
depende del ratio de conflictos y del tamaño de la transacción.

**Implementar en:** Go · Haskell · Clojure

---

### Ejercicio 6.4.5 — STM composable: el argumento de fondo

**Enunciado:** El argumento más fuerte para STM es la composabilidad.
Con mutex, dos funciones thread-safe no se pueden componer para formar
una función thread-safe:

```go
// Cada función es thread-safe individualmente
func (q *Cola) EstaVacia() bool { q.mu.Lock(); defer q.mu.Unlock(); return len(q.items) == 0 }
func (q *Cola) Sacar() Item    { q.mu.Lock(); defer q.mu.Unlock(); return q.sacarInterno() }

// Pero esto NO es thread-safe — check-then-act:
if !q.EstaVacia() {  // libera el lock
    item := q.Sacar()  // adquiere el lock de nuevo — puede fallar
}
```

Con STM:

```go
// Con STM, componer transacciones es seguro:
func sacarSiNoVacia(q *ColaSTM, tx *Transaccion) (Item, bool) {
    items := tx.Read(q.items)
    if len(items) == 0 { return Item{}, false }
    tx.Write(q.items, items[1:])
    return items[0], true
}
// Esta función puede ser parte de una transacción mayor
```

Implementa tres ejemplos de composición que son imposibles (o difíciles) con mutex
y naturales con STM.

**Restricciones:** Los ejemplos deben ser casos reales, no académicos.
Documenta por qué la versión con mutex requiere coordinación especial
(lock ordering, lock externo, etc.).

**Implementar en:** Go (STM simulada) · Haskell · Clojure

---

## Sección 6.5 — Reactive Streams y Back-pressure Formal

Reactive Streams es una especificación (no un modelo de concurrencia)
que formaliza el back-pressure entre componentes asíncronos.

**El problema que resuelve:**

```
Sin back-pressure formal:
  Producer → [canal] → Consumer (lento)

  Si Producer es 10x más rápido que Consumer:
  Opción A: Drop — se pierden mensajes
  Opción B: Block — el Producer se bloquea (back-pressure implícita)
  Opción C: Buffer infinito — OOM eventual

Con Reactive Streams:
  El Consumer declara cuántos items puede procesar: consumer.request(N)
  El Producer envía exactamente N items
  El Consumer pide más cuando está listo: consumer.request(M)
  Back-pressure es explícita y bidireccional
```

**Los cuatro interfaces de Reactive Streams:**

```java
// Java — la especificación original
Publisher<T>    // produce items
Subscriber<T>   // consume items — implementa onNext, onError, onComplete
Subscription    // conexión entre Publisher y Subscriber — tiene request(n) y cancel()
Processor<T,R>  // es Publisher y Subscriber — etapa del pipeline
```

**Implementaciones en producción:**

```
Java:    Project Reactor (Spring WebFlux), RxJava, Akka Streams
Kotlin:  Flows (Kotlin Coroutines)
Go:      No hay equivalente directo — los canales con buffer son la aproximación
Python:  RxPY, aunque el GIL limita el paralelismo
```

---

### Ejercicio 6.5.1 — Implementar Reactive Streams en Go

**Enunciado:** Implementa los cuatro interfaces de Reactive Streams en Go:

```go
type Publisher[T any] interface {
    Subscribe(s Subscriber[T])
}

type Subscriber[T any] interface {
    OnSubscribe(sub Subscription)
    OnNext(item T)
    OnError(err error)
    OnComplete()
}

type Subscription interface {
    Request(n int)
    Cancel()
}
```

**Restricciones:** El Publisher no envía más items de los que el Subscriber
ha pedido con `Request`. Si el Subscriber cancela, el Publisher para.

**Pista:** El Publisher mantiene un contador de "créditos" (items que puede enviar).
`Request(n)` incrementa los créditos. Cada vez que envía un item, decrementa.
Si los créditos llegan a 0, el Publisher espera (bloquea o en select).

**Implementar en:** Go · Java (Project Reactor) · Python (RxPY)

---

### Ejercicio 6.5.2 — Pipeline Reactive con back-pressure

**Enunciado:** Implementa un pipeline de tres etapas donde la back-pressure
se propaga automáticamente:

```
Publisher rápido → Map (lento, 50ms/item) → Filter → Subscriber (pide 5 a la vez)
```

Verifica que el Publisher nunca produce más de lo que el Subscriber puede consumir.

**Restricciones:** El Publisher no usa buffer ilimitado. En ningún momento
hay más de 5 items "en vuelo" entre el Publisher y el Subscriber.

**Pista:** El Subscriber empieza pidiendo 5 items (`Request(5)`).
Cuando recibe el ítem número 3, pide 5 más — manteniendo el pipeline lleno
sin overflow. El Map hace `Request(1)` al Publisher por cada item que procesa.
La back-pressure fluye de derecha a izquierda: Subscriber ← Map ← Publisher.

**Implementar en:** Go · Java (Project Reactor) · Python

---

### Ejercicio 6.5.3 — Reactive vs canales: cuándo usar cada uno

**Enunciado:** Para cada escenario, determina si canales de Go o Reactive Streams
es la abstracción más apropiada:

1. Pipeline de procesamiento de logs (rate fijo, consumer siempre disponible)
2. Stream de eventos de red (rate variable, consumer puede no estar disponible)
3. WebSocket con múltiples clientes a distintas velocidades
4. Pipeline ETL batch (rate controlado, sin usuarios)
5. API que agrega streams de múltiples microservicios

**Restricciones:** Para los casos con canales, implementa con los patrones del Cap.03.
Para los casos con Reactive, implementa con la librería del Ejercicio 6.5.1.

**Pista:** Los canales de Go son suficientes cuando el producer y el consumer
tienen rates similares o cuando el buffer absorbe las diferencias.
Reactive Streams brilla cuando el consumer necesita control fino sobre la
demanda — especialmente en sistemas con usuarios humanos o redes lentas.

**Implementar en:** Go · Java · Python

---

### Ejercicio 6.5.4 — Hot vs Cold Publishers

**Enunciado:** Un Publisher "frío" empieza a producir cuando alguien se suscribe.
Un Publisher "caliente" produce independientemente de los suscriptores.

```
Frío:  un archivo en disco — empieza a leer cuando alguien escucha
Caliente: precios de acciones en tiempo real — existen aunque nadie escuche
```

Implementa ambos y las implicaciones para los suscriptores que se suscriben tarde:

**Restricciones:** Un suscriptor tardío de un Publisher frío recibe todos los items
desde el principio. Un suscriptor tardío de un Publisher caliente solo recibe
los items producidos a partir de su suscripción.

**Pista:** Para hacer un Publisher caliente "compartible" (todos los suscriptores
reciben los mismos items), se usa un operador `publish()` o `share()` que
multicastea a todos los suscriptores activos. Es exactamente el Pub/Sub del Cap.03.

**Implementar en:** Go · Java (Project Reactor: Flux.hot()) · Python

---

### Ejercicio 6.5.5 — Integrar back-pressure con el Worker Pool

**Enunciado:** Extiende el Worker Pool del Cap.03 para implementar back-pressure
explícita al estilo Reactive: el llamador declara cuántos resultados puede procesar,
y el pool no produce más hasta recibir nueva demanda.

```go
type PoolReactivo[J, R any] struct {
    // ...
}

func (p *PoolReactivo[J, R]) Suscribir(s Subscriber[R])
// El subscriber llama s.OnSubscribe con una Subscription
// El subscriber llama sub.Request(n) para pedir n resultados
// El pool procesa n trabajos y los entrega
// Luego espera hasta que el subscriber pida más
```

**Restricciones:** El pool no procesa más trabajos de los que el subscriber
ha pedido. Si el subscriber cancela, el pool para limpiamente.

**Pista:** El pool interno sigue siendo el del Cap.03 — workers, canal de jobs,
canal de results. La capa Reactive añade el protocolo de demanda encima:
un contador de "créditos" (items pedidos y no entregados aún).

**Implementar en:** Go · Java

---

## Sección 6.6 — Elegir el Modelo Correcto

Con cuatro modelos en la mesa (mutex, actores, CSP, STM), la pregunta
es cuándo cada uno es la herramienta natural.

**El árbol de decisión:**

```
¿El estado es accedido principalmente de forma concurrente?
    No  → mutex o atómico (Cap.02)
    Sí  → ¿el dato fluye de productor a consumidor?
              Sí → CSP / canales (Cap.03-05)
              No → ¿el estado tiene identidad permanente?
                       Sí → Actores (sección 6.1-6.3)
                       No → ¿necesitas composición de operaciones atómicas?
                                Sí → STM (sección 6.4)
                                No → RWMutex (Cap.02)
```

**La tabla de tradeoffs:**

```
Modelo     Overhead    Composable    Distribuible   Debugging
────────   ─────────   ──────────    ────────────   ─────────
Mutex      Mínimo      No            No             Difícil
Atómico    Mínimo      No (1 var)    No             Moderado
Canales    Bajo        Sí            No (local)     Moderado
Actores    Moderado    Parcial       Sí (Erlang)    Trazable
STM        Variable    Sí            No*            Fácil
```

---

### Ejercicio 6.6.1 — Rediseñar un sistema con el modelo correcto

**Enunciado:** Este sistema usa mutex para todo pero el diseño tiene problemas.
Analiza y rediseña usando el modelo más apropiado para cada componente:

```go
type Sistema struct {
    mu         sync.Mutex
    usuarios   map[string]*Usuario    // muchas lecturas, pocas escrituras
    mensajes   []Mensaje              // productor rápido, consumidor lento
    sesiones   map[string]time.Time   // expiran — necesita cleanup periódico
    contadores map[string]int64       // solo incrementan
}
```

**Restricciones:** Cada componente puede usar un modelo distinto.
Documenta por qué elegiste cada uno.

**Pista:** `usuarios` → RWMutex (muchas lecturas).
`mensajes` → canal con buffer (productor/consumidor).
`sesiones` → actor propietario o sync.Map.
`contadores` → sync/atomic por clave (si las claves son fijas) o sync.Map + atómico.

**Implementar en:** Go · Java · Python

---

### Ejercicio 6.6.2 — Migrar de mutex a actores: cuándo merece la pena

**Enunciado:** Tienes un subsistema con 15 mutexes y bugs de deadlock ocasionales.
Evalúa si migrar a actores resuelve el problema y a qué costo:

1. Identifica los deadlocks en el código (usa la técnica del grafo de espera del Ejercicio 1.2.5)
2. Diseña la versión con actores
3. Implementa ambas versiones
4. Compara: corrección, rendimiento, legibilidad del código

**Restricciones:** La migración no debe cambiar la API externa del subsistema.

**Pista:** Los deadlocks con mutex generalmente vienen de múltiples locks
adquiridos en orden inconsistente. Los actores eliminan esto porque el estado
es privado — no hay locks entre actores. El costo: más overhead por mensaje,
y la lógica de negocio se distribuye entre handlers.

**Implementar en:** Go · Java (Akka)

---

### Ejercicio 6.6.3 — Diseño de un sistema de juego multijugador

**Enunciado:** Diseña el sistema de concurrencia para un juego multijugador simple:
- 100 jugadores conectados simultáneamente
- Cada jugador tiene posición, salud, inventario
- El estado del mundo (mapa, objetos) es compartido
- Los eventos (movimientos, ataques) se procesan en orden

Para cada componente, elige el modelo y justifica:

```go
type Juego struct {
    mundo    ???  // estado compartido del mapa
    jugadores map[string]??? // estado de cada jugador
    eventos  ???  // cola de eventos a procesar
    tick     ???  // loop de actualización del mundo
}
```

**Restricciones:** El juego debe procesar 60 ticks por segundo.
Latencia de input a output < 50ms. Sin cheating (estado del jugador
solo modificable por el servidor).

**Pista:** El mundo (mapa) podría ser RWMutex (muchas lecturas, pocas escrituras).
Cada jugador podría ser un actor (estado privado). Los eventos podrían ser
un canal. El tick loop es una goroutine que procesa todos los eventos del
tick actual antes de enviar el siguiente estado del mundo.

**Implementar en:** Go · Java · Elixir

---

### Ejercicio 6.6.4 — El caso Erlang: por qué "let it crash"

**Enunciado:** La filosofía de Erlang es no defensividad: si algo falla,
déjalo caer y que el supervisor lo reinicie. Contrasta esta filosofía con
la de Go (recover(), errores explícitos).

Implementa el mismo servicio con las dos filosofías:

**Erlang style en Go:** el servidor no hace `recover()`. Si entra en pánico,
el supervisor lo reinicia. El estado se reconstruye desde cero.

**Go style:** el servidor hace `recover()` de todos los pánicos,
loguea el error, y continúa.

**Restricciones:** Compara: ¿cuál es más fácil de razonar sobre la corrección?
¿Cuál tiene mejor observabilidad? ¿Cuál se recupera más rápido de un bug?

**Pista:** "Let it crash" funciona bien cuando el estado puede reconstruirse
rápidamente (desde una BD, desde mensajes en cola). Cuando el estado es
costoso de reconstruir o cuando el proceso tiene conexiones abiertas que
tardan en recuperarse, la estrategia de recover() puede ser preferible.

**Implementar en:** Go · Elixir

---

### Ejercicio 6.6.5 — Síntesis: diseñar un sistema desde cero

**Enunciado:** Diseña el sistema de concurrencia para un servicio de streaming
de música con estas características:

- 50,000 usuarios simultáneos
- Cada usuario tiene una sesión (estado: canción actual, posición, historial)
- Las canciones se prefetchean (predictive loading) en background
- Las recomendaciones se calculan asíncronamente
- Los eventos de analytics se acumulan y se envían en batch

Para cada componente, documenta:
- El modelo de concurrencia elegido y por qué
- El número estimado de goroutines/actores/threads
- Cómo se maneja el ciclo de vida (inicio, parada limpia)
- Cómo se manejan los fallos

**Restricciones:** El diseño debe poder procesar 50,000 sesiones
con un servidor de 16 cores y 32 GB de RAM.

**Pista:** 50,000 goroutines (una por usuario) es completamente factible en Go —
~2 KB por goroutine = 100 MB total para las goroutines solas.
El prefetch y las recomendaciones son buenos candidatos para Worker Pool
con límite de concurrencia. Los analytics son buenos para batch con canal con buffer.

**Implementar en:** Go · Java · Elixir

---

## Sección 6.7 — Conexión con Sistemas Distribuidos

Los modelos de concurrencia de este capítulo tienen análogos directos
en los sistemas distribuidos del Cap.22-23.

**El puente:**

```
Concurrencia local                  Sistemas distribuidos
──────────────────                  ──────────────────────
Mutex                    →          Distributed Lock (Redis SETNX, ZooKeeper)
Canal (message passing)  →          Message Queue (RabbitMQ, Kafka)
Actor                    →          Microservicio
Pub/Sub local            →          Event Bus distribuido (Kafka, NATS)
STM                      →          Distributed Transactions (2PC, Saga)
back-pressure (Reactive) →          Flow Control (TCP, gRPC flow control)
```

**La diferencia fundamental:**

```
Concurrencia local: fallos son raros y recuperables (panic → recover)
Sistemas distribuidos: fallos son la norma (red particionada, nodo caído)

Concurrencia local: mensajes nunca se pierden (canal garantiza entrega)
Sistemas distribuidos: mensajes pueden perderse, duplicarse, llegar fuera de orden

Concurrencia local: consistencia fuerte (transacciones atómicas)
Sistemas distribuidos: consistencia eventual o consistencia fuerte con costo alto
```

---

### Ejercicio 6.7.1 — Actor local → microservicio

**Enunciado:** Toma el sistema de chat del Ejercicio 6.1.2 (actores locales)
y conviértelo en un sistema distribuido donde cada instancia del servicio
puede manejar sus propios usuarios y comunicarse con otras instancias.

**Restricciones:** Los usuarios en la instancia A pueden enviar mensajes a
usuarios en la instancia B. La solución puede usar HTTP o gRPC entre instancias.
No hace falta consistencia fuerte — los mensajes pueden llegar ligeramente fuera de orden.

**Pista:** El actor `ChatRoom` local se convierte en un servicio.
El `ActorRef` del Ejercicio 6.2.2 (transparencia de ubicación) es la
clave: el código que envía un mensaje no sabe si el destinatario es local o remoto.

**Implementar en:** Go · Elixir (nativo con Phoenix PubSub)

---

### Ejercicio 6.7.2 — Canal local → Kafka

**Enunciado:** Refactoriza el Worker Pool del Cap.03 para que funcione
tanto con canales locales (para tests y single-instance) como con Kafka
(para multi-instance y persistencia).

```go
type Queue[T any] interface {
    Publish(ctx context.Context, item T) error
    Consume(ctx context.Context) (<-chan T, error)
    Close() error
}

type LocalQueue[T any] struct { ch chan T }
type KafkaQueue[T any] struct { /* productor y consumidor Kafka */ }
```

**Restricciones:** El Worker Pool funciona con cualquier implementación de `Queue`.
Los tests usan `LocalQueue`. Producción usa `KafkaQueue`.
La interfaz debe ser idéntica para ambos.

**Pista:** Esta abstracción es el patrón "Port and Adapter" (Hexagonal Architecture).
El Worker Pool es el núcleo. La `Queue` es el port. `LocalQueue` y `KafkaQueue`
son los adapters. Los tests no tocan Kafka — usan el adapter local.

**Implementar en:** Go · Java · Python

---

### Ejercicio 6.7.3 — STM local → Saga distribuida

**Enunciado:** La transferencia bancaria del Ejercicio 6.4.2 (STM local)
no puede trasladarse directamente a un sistema distribuido — no hay STM
distribuida práctica. El patrón Saga es el equivalente distribuido.

Implementa la misma transferencia usando el patrón Saga:

```
Paso 1: Debitar cuenta A → evento "debitado_A"
Paso 2: Creditar cuenta B → evento "creditado_B"

Compensaciones (si algo falla):
  Falla paso 2: "revertir_debitado_A" → acreditar cuenta A
```

**Restricciones:** La Saga no usa locks entre los pasos.
Si el proceso falla entre paso 1 y paso 2, el sistema puede recuperarse
y completar o compensar la Saga.

**Pista:** La Saga es el Cap.17 §4 del repo de algoritmos (Saga pattern).
La diferencia con la STM: la Saga es eventualmente consistente — hay un momento
donde A fue debitada pero B no fue creditada aún. La STM local es instantánea.
Los sistemas distribuidos deben aceptar esta ventana de inconsistencia.

**Implementar en:** Go · Java · Python

---

### Ejercicio 6.7.4 — Back-pressure local → flow control de TCP/gRPC

**Enunciado:** La back-pressure del Ejercicio 6.5.2 (Reactive local) tiene
su análogo en TCP (ventana de recepción) y en gRPC (HTTP/2 flow control).

Implementa un cliente gRPC que respeta el flow control del servidor:
el servidor puede ralentizar al cliente si no puede procesar más items.

**Restricciones:** Usa gRPC server streaming. El servidor envía N items
y luego espera a que el cliente pida más (similar a Reactive `request(N)`).

**Pista:** gRPC sobre HTTP/2 tiene flow control nativo: la capa de transporte
lleva cuenta de los bytes en vuelo. Cuando el buffer del receptor está lleno,
el emisor se bloquea automáticamente. No es necesario implementarlo — ya existe.
El ejercicio es observar este comportamiento y entender su relación con Reactive.

**Implementar en:** Go (gRPC) · Java (gRPC)

---

### Ejercicio 6.7.5 — El CAP theorem y los modelos de concurrencia

**Enunciado:** El teorema CAP dice que un sistema distribuido no puede tener
simultáneamente Consistency, Availability y Partition tolerance.

Relaciona los modelos de concurrencia de este capítulo con el CAP:

- STM local → consistencia fuerte, no distribuible (no hay partición)
- Actores con Erlang distributed → disponibilidad alta, consistencia eventual
- Mutex distribuido (Redis SETNX) → consistencia fuerte, latencia alta

Implementa tres sistemas que prioricen C, A, o P respectivamente,
y demuestra el tradeoff en cada caso.

**Restricciones:** Los sistemas son simplificados (no son producción).
El objetivo es hacer el tradeoff observable y medible.

**Pista:** El sistema CP (Consistency + Partition tolerance) bloquea cuando
hay partición de red. El sistema AP (Availability + Partition tolerance)
responde siempre aunque los datos no estén sincronizados. El sistema CA
(Consistency + Availability) no existe en presencia de particiones reales.

**Implementar en:** Go · Java

---

## Resumen del capítulo

**Los cuatro modelos y cuándo cada uno es natural:**

| Modelo | Cuándo usar | Ejemplo real |
|---|---|---|
| **Mutex** | Estado compartido, acceso rápido | Caché compartida, contadores |
| **Canales / CSP** | Datos que fluyen entre etapas | Pipelines, Worker Pools |
| **Actores** | Estado con identidad, sistemas distribuidos | Sesiones de usuario, microservicios |
| **STM** | Transacciones sobre múltiples variables | Transferencias, invariantes complejas |

**El continuum:**

```
Concurrencia local ────────────────────────── Sistemas distribuidos
    Mutex → Canales → Actores → STM                → Saga, Eventual Consistency
    (más control)                              (menos garantías, más escala)
```

## La pregunta que guía el Cap.07

> Los Cap.01–06 cubren los problemas, las herramientas, los patrones y los modelos.
> Todo eso es inútil si el código concurrente no puede verificarse.
>
> El Cap.07 cubre el testing sistemático de código concurrente:
> cómo encontrar bugs que el -race no detecta, cómo hacer benchmarks
> que no mienten, y cómo diseñar código concurrente que sea testeable
> por construcción — no por casualidad.
