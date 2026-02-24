# Guía de Ejercicios — Go Idiomático: Concurrencia como Diseño

> Implementar cada ejercicio en: **Go** (capítulo centrado en Go)
> Equivalencias en Java · Python · Rust donde enriquecen la comparación.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: la diferencia entre usar canales y pensar con canales

Hay dos formas de llegar al mismo programa concurrente en Go.

**La primera forma — traducir desde otro lenguaje:**

```go
// "Necesito ejecutar esto en background"
// → lanzar goroutine, proteger con mutex, usar WaitGroup

type Procesador struct {
    mu      sync.Mutex
    cola    []Item
    activo  bool
}

func (p *Procesador) Agregar(item Item) {
    p.mu.Lock()
    p.cola = append(p.cola, item)
    p.mu.Unlock()
}

func (p *Procesador) Iniciar() {
    p.activo = true
    go func() {
        for p.activo {
            p.mu.Lock()
            if len(p.cola) > 0 {
                item := p.cola[0]
                p.cola = p.cola[1:]
                p.mu.Unlock()
                procesar(item)
            } else {
                p.mu.Unlock()
                time.Sleep(10 * time.Millisecond)
            }
        }
    }()
}
```

**La segunda forma — pensar en términos de comunicación:**

```go
// "Un productor envía items, un consumidor los procesa"
// → el canal ES la cola, la goroutine ES el consumidor

type Procesador struct {
    items chan Item
    done  chan struct{}
}

func NuevoProcesador() *Procesador {
    p := &Procesador{
        items: make(chan Item, 100),
        done:  make(chan struct{}),
    }
    go p.ejecutar()
    return p
}

func (p *Procesador) Agregar(item Item) {
    p.items <- item
}

func (p *Procesador) ejecutar() {
    defer close(p.done)
    for item := range p.items {
        procesar(item)
    }
}

func (p *Procesador) Detener() {
    close(p.items)  // señal de terminación
    <-p.done        // esperar que termine
}
```

La segunda versión no tiene mutex. No tiene polling. No tiene flag `activo`.
Es más corta, más legible, y más correcta por construcción — no puede haber
race condition porque el canal es el único punto de acceso al estado.

**Ese cambio de perspectiva es lo que este capítulo enseña.**

---

## La filosofía de Go para concurrencia

Rob Pike, co-creador de Go, lo resumió en 2012:

> *"Don't communicate by sharing memory; share memory by communicating."*

No es una regla absoluta — los mutex tienen su lugar. Pero es el punto de partida
correcto para diseñar sistemas concurrentes en Go.

```
Compartir memoria y proteger con mutex:
  Múltiples goroutines acceden al mismo objeto.
  El mutex garantiza que solo una lo toca a la vez.
  El programador razona sobre qué está protegido y qué no.

Compartir memoria comunicando:
  Solo una goroutine posee el dato en cada momento.
  La propiedad se transfiere pasando el dato por un canal.
  No hay acceso concurrente — no hay race condition posible.
```

---

## Tabla de contenidos

- [Sección 5.1 — Ownership a través de canales](#sección-51--ownership-a-través-de-canales)
- [Sección 5.2 — context.Context: cancelación y timeouts](#sección-52--contextcontext-cancelación-y-timeouts)
- [Sección 5.3 — Patrones idiomáticos de Go](#sección-53--patrones-idiomáticos-de-go)
- [Sección 5.4 — errgroup y sincronización de errores](#sección-54--errgroup-y-sincronización-de-errores)
- [Sección 5.5 — Testing de código concurrente en Go](#sección-55--testing-de-código-concurrente-en-go)
- [Sección 5.6 — Go vs Java Virtual Threads vs Python asyncio](#sección-56--go-vs-java-virtual-threads-vs-python-asyncio)
- [Sección 5.7 — Diseño de APIs concurrentes](#sección-57--diseño-de-apis-concurrentes)

---

## Sección 5.1 — Ownership a Través de Canales

El principio de ownership dice que en cada momento, exactamente una goroutine
tiene el derecho de leer y modificar un dato. Cuando otra goroutine necesita
el dato, la primera se lo pasa — transfiriendo el ownership.

**El contrato de ownership con canales:**

```go
// El canal transfiere tanto el valor como el derecho a usarlo.

// Productor: crea el dato, lo envía, ya no lo usa más
go func() {
    buffer := make([]byte, 4096)
    n, _ := conn.Read(buffer)
    jobs <- buffer[:n]  // transfiere ownership → ya no accedo a buffer
    // si accedo a buffer después de este punto, violo el contrato
}()

// Consumidor: recibe el dato, ahora es el único dueño
go func() {
    for data := range jobs {
        procesar(data)  // acceso exclusivo garantizado — no hay race condition
    }
}()
```

**Cuándo el ownership a través de canales es la solución correcta:**

```
✅ El dato fluye de un productor a un consumidor
✅ El dato se transforma en etapas (pipeline)
✅ La goroutine dueña cambia con el tiempo
✅ No hay acceso de lectura concurrente — solo un dueño a la vez

❌ Múltiples lectores concurrentes (usar RWMutex)
❌ Estado que necesitan consultar muchas goroutines (caché compartida)
❌ Configuración inmutable accedida desde muchas goroutines (no necesita lock)
```

**El error más común — mezclar ownership:**

```go
buffer := make([]byte, 4096)
jobs <- buffer  // "transfiero" el buffer

// VIOLACIÓN: sigo usando buffer después de enviarlo
copy(buffer, "datos nuevos")  // race condition — el consumidor puede estar leyendo
```

---

### Ejercicio 5.1.1 — Refactorizar de mutex a ownership

**Enunciado:** Refactoriza este servidor de caché que usa mutex a uno
que usa ownership a través de canales. El servidor mantiene un mapa de
strings que puede consultarse y actualizarse concurrentemente.

```go
// Versión con mutex
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

**Restricciones:** La versión con canales debe tener la misma interfaz externa.
Implementa el patrón "goroutine propietaria del mapa": una sola goroutine
accede al mapa, las demás le envían requests a través de un canal.

**Pista:** El truco es que las requests deben incluir un canal de respuesta:
```go
type getRequest struct {
    key  string
    resp chan<- string
}
```
La goroutine propietaria procesa la request y envía la respuesta.
El llamador espera en su canal de respuesta.

**Implementar en:** Go · Java (Actor pattern con Akka) · Python (asyncio.Queue)

---

### Ejercicio 5.1.2 — Pipeline con ownership explícito

**Enunciado:** Implementa un pipeline de tres etapas donde el ownership
de cada item es explícito. Cada etapa debe documentar qué goroutine
posee el item en cada punto:

```
Etapa 1 (parser):    posee el []byte raw
    → envía al canal: transfiere ownership a Etapa 2
Etapa 2 (validator): posee el Record parseado
    → envía al canal: transfiere ownership a Etapa 3
Etapa 3 (writer):    posee el Record validado
    → escribe a disco: libera ownership
```

**Restricciones:** Verifica con `-race` que no hay acceso concurrente a los items.
Usa un pool de buffers (`sync.Pool`) para los `[]byte` de Etapa 1 — la Etapa 3
debe devolver los buffers al pool cuando termina.

**Pista:** El pool de buffers es el único punto donde el ownership se comparte
temporalmente — el pool garantiza que un buffer del pool no se usa en dos
lugares simultáneamente. La etapa 3 devuelve al pool solo después de procesar
completamente el item.

**Implementar en:** Go · Rust (ownership nativo del compilador)

---

### Ejercicio 5.1.3 — Goroutina propietaria (owner goroutine pattern)

**Enunciado:** Implementa un gestor de conexiones de base de datos donde
una única goroutine "propietaria" mantiene el pool de conexiones.
Las otras goroutines piden y devuelven conexiones a través de canales.

```go
type PoolManager struct {
    pedir    chan chan *Conexion  // goroutine envía su canal de respuesta
    devolver chan *Conexion
    done     chan struct{}
}
```

**Restricciones:** El pool tiene capacidad fija de N conexiones.
Si todas las conexiones están en uso, las nuevas peticiones esperan.
Verifica que en ningún momento dos goroutines usan la misma conexión.

**Pista:** La goroutina propietaria hace `select` entre `pedir` y `devolver`.
El estado interno (lista de conexiones disponibles) solo vive en esa goroutina —
no necesita ningún lock. Esto es la esencia del patrón: el canal es el
mecanismo de acceso, no un lock sobre estado compartido.

**Implementar en:** Go · Java (Actor con estado encapsulado)

---

### Ejercicio 5.1.4 — Detectar violaciones de ownership

**Enunciado:** Escribe un tipo `Owned[T]` que detecta en tiempo de ejecución
si el ownership se viola — si dos goroutines acceden al valor simultáneamente:

```go
type Owned[T any] struct {
    valor    T
    ownerID  int64  // ID de la goroutina actual propietaria
    mu       sync.Mutex
}

func (o *Owned[T]) Tomar() *T      // adquiere ownership
func (o *Owned[T]) Liberar()       // libera ownership
func (o *Owned[T]) Transferir(ch chan<- *Owned[T])  // transfiere por canal
```

**Restricciones:** `Tomar` debe fallar (panic o error) si otra goroutina
ya tiene el ownership. El tipo es solo para debugging — en producción
el overhead sería inaceptable.

**Pista:** Usa el ID de la goroutina actual como identificador.
Go no expone el ID de goroutina directamente — pero se puede extraer
del stack trace con `runtime.Stack`. La biblioteca `github.com/petermattis/goid`
lo hace eficientemente.

**Implementar en:** Go

---

### Ejercicio 5.1.5 — ¿Mutex o canal? El árbol de decisión aplicado

**Enunciado:** Para cada uno de estos sistemas, diseña la sincronización
eligiendo entre mutex y ownership a través de canales. Justifica cada elección.

1. **Contador de requests** — incrementado por cada handler HTTP
2. **Configuración dinámica** — leída por todos los handlers, actualizada por un goroutine de administración
3. **Buffer circular de logs** — escrito por múltiples goroutines, leído por una goroutine que vuelca a disco
4. **Árbol de rutas HTTP** — construido al inicio, solo lectura durante el runtime
5. **Mapa de sesiones activas** — insertado al login, eliminado al logout, leído en cada request

**Restricciones:** Para cada caso implementa la solución elegida y explica
por qué no elegiste la alternativa.

**Pista:** La guía de Go dice: "usa canales cuando pasas ownership de datos,
usa mutex cuando proteges estado interno". El contador (1) es un atómico.
La configuración (2) es `atomic.Pointer` o RWMutex. El buffer (3) es canal.
El árbol (4) no necesita nada — es inmutable después de construirse.
El mapa (5) es `sync.Map` o RWMutex según el ratio lectura/escritura.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 5.2 — context.Context: Cancelación y Timeouts

`context.Context` es el mecanismo estándar de Go para propagar cancelación,
timeouts y valores a través de una cadena de llamadas. Apareció en el Cap.03
como herramienta. Este sección lo analiza como principio de diseño.

**El contrato de context.Context:**

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}     // cerrado cuando el contexto se cancela
    Err() error               // nil, context.Canceled, o context.DeadlineExceeded
    Value(key any) any        // propagar valores a través de la cadena
}
```

**La cadena de contexts — herencia de cancelación:**

```
context.Background()          ← raíz, nunca se cancela
    └─ WithCancel             ← cancelable manualmente
        └─ WithTimeout(5s)    ← se cancela en 5s o cuando el padre se cancela
            └─ WithValue(k,v) ← agrega valores, hereda cancelación del padre
```

**La regla de oro: si puedes cancelar, debes cancelar.**

```go
// MAL: context creado, nunca cancelado
ctx, _ := context.WithCancel(context.Background())
llamar(ctx)

// BIEN: siempre defer cancel()
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // cancela cuando la función retorna, liberando recursos
llamar(ctx)
```

**Qué propagar con context.Value y qué no:**

```go
// SÍ propagar con Value:
ctx = context.WithValue(ctx, traceIDKey, "abc-123")    // trazabilidad
ctx = context.WithValue(ctx, userIDKey, userID)        // identidad del request

// NO propagar con Value:
ctx = context.WithValue(ctx, dbKey, db)                // dependencias opcionales
ctx = context.WithValue(ctx, timeoutKey, 5*time.Second) // usar WithTimeout
// Value es para datos del request, no para configuración de infraestructura
```

---

### Ejercicio 5.2.1 — Propagación de cancelación en cadena

**Enunciado:** Implementa un sistema de tres capas donde la cancelación
del contexto raíz se propaga a todas las goroutines activas:

```
Handler HTTP
    └─ ServicioA (ctx)
        └─ ServicioB (ctx)
            └─ AccesoBD (ctx)
```

Cuando el cliente HTTP cierra la conexión, `r.Context()` se cancela.
Verifica que todos los niveles limpian sus recursos y terminan.

**Restricciones:** Cada capa debe verificar `ctx.Done()` y retornar
`ctx.Err()` cuando se cancela. Verifica con un test que simula la
cancelación en distintos puntos de la cadena.

**Pista:** En un servidor HTTP, `r.Context()` se cancela automáticamente
cuando el cliente desconecta o cuando el handler retorna. Pasar este
contexto a todas las llamadas downstream garantiza que ningún trabajo
continúa para una request abandonada.

**Implementar en:** Go · Java (con CompletableFuture y CancellationToken) · C#

---

### Ejercicio 5.2.2 — Timeout por capa vs timeout total

**Enunciado:** Una request tiene un timeout total de 2 segundos.
El ServicioA tiene 3 llamadas secuenciales, cada una con su propio timeout.
Si el timeout total expira antes, todas las llamadas deben cancelarse.

```go
// Timeout total
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()

// Subtimeouts por llamada
ctxA, cancelA := context.WithTimeout(ctx, 500*time.Millisecond)
defer cancelA()
r1 := llamar(ctxA)

ctxB, cancelB := context.WithTimeout(ctx, 500*time.Millisecond)
defer cancelB()
r2 := llamar(ctxB)
```

**Restricciones:** Si el timeout total expira durante la segunda llamada,
la tercera nunca debe iniciarse. Los subtimeouts son más restrictivos que
el total — una llamada lenta consume tiempo del total para las siguientes.

**Pista:** El contexto hijo hereda el deadline del padre. Si el padre tiene
deadline de 2s y le quedan 300ms cuando se crea el hijo con timeout de 500ms,
el hijo usa 300ms — el menor de los dos. Esto es automático en Go.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 5.2.3 — context.Value: trace ID por request

**Enunciado:** Implementa un sistema de trazabilidad distribuida donde
cada request tiene un trace ID que se propaga automáticamente a todos
los logs y llamadas downstream.

```go
type traceKey struct{}

func WithTraceID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, traceKey{}, id)
}

func TraceIDFromContext(ctx context.Context) string {
    id, _ := ctx.Value(traceKey{}).(string)
    return id
}

// En cada función que loguea:
func procesar(ctx context.Context, item Item) {
    log.Printf("[%s] procesando item %d", TraceIDFromContext(ctx), item.ID)
}
```

**Restricciones:** El trace ID se genera una vez al inicio del request
y se propaga sin cambios. Las funciones internas no deben recibir el
trace ID como parámetro explícito — solo el contexto.

**Pista:** Usa un tipo privado como clave (`type traceKey struct{}`) para
evitar colisiones con otras bibliotecas que usen el mismo contexto.
Las claves de string o int son peligrosas — cualquier código puede sobreescribir
el valor accidentalmente.

**Implementar en:** Go · Java (MDC de SLF4J como equivalente) · Python

---

### Ejercicio 5.2.4 — Context para recursos: el leak de context sin cancel

**Enunciado:** `context.WithCancel`, `WithTimeout` y `WithDeadline` crean
una entrada en el contexto padre que solo se elimina cuando el contexto
hijo se cancela o expira. Sin `cancel()`, hay un leak.

Demuestra el leak y su impacto:

```go
// Leak: nunca se llama cancel()
for i := 0; i < 1_000_000; i++ {
    ctx, _ := context.WithTimeout(parentCtx, 5*time.Second)
    go usarContext(ctx)
    // el parentCtx acumula 1,000,000 entradas hasta que expiran
}
```

**Restricciones:** Mide el uso de memoria con y sin el leak.
Implementa el linter que detecta contexts sin cancel —
`go vet` ya detecta `context.WithCancel` sin asignar la función cancel.

**Pista:** El garbage collector no puede liberar el contexto hijo mientras
el padre esté vivo y tenga referencia al hijo. Con 1M de contextos activos,
la memoria crece significativamente. `go vet ./...` reporta:
`the cancel function returned by context.WithCancel should be called`.

**Implementar en:** Go

---

### Ejercicio 5.2.5 — Implementar context.Context desde cero

**Enunciado:** Implementa una versión simplificada de `context.Context`
que soporte `WithCancel` y `WithTimeout`. Esto consolida el entendimiento
de cómo funciona internamente.

```go
type miContext struct {
    parent   *miContext
    done     chan struct{}
    err      error
    deadline time.Time
    mu       sync.Mutex
    children map[*miContext]struct{}  // para propagar cancelación
}

func (c *miContext) Done() <-chan struct{}
func (c *miContext) Err() error
func (c *miContext) Deadline() (time.Time, bool)
func MiWithCancel(parent *miContext) (*miContext, func())
func MiWithTimeout(parent *miContext, d time.Duration) (*miContext, func())
```

**Restricciones:** La cancelación del padre debe propagarse a todos los hijos.
Los hijos cancelados deben eliminarse de la lista del padre para no acumular memoria.

**Pista:** Cuando se cancela un contexto, itera sus hijos y cancélalos recursivamente.
Cuando un hijo se cancela, se elimina a sí mismo del padre. Este mecanismo
es O(número de hijos directos) por cancelación — eficiente en práctica.

**Implementar en:** Go

---

## Sección 5.3 — Patrones Idiomáticos de Go

Estos son los patrones que los programadores de Go reconocen inmediatamente
y que no tienen equivalente directo en otros lenguajes.

**El patrón done channel — señalizar terminación:**

```go
// Versión con canal de done
done := make(chan struct{})
go func() {
    defer close(done)
    trabajar()
}()
// esperar:
<-done

// Versión moderna: context.Context
// (preferida para nuevos proyectos)
```

**El patrón or-done — cancelar al primer resultado:**

```go
func OrDone(done <-chan struct{}, c <-chan int) <-chan int {
    valStream := make(chan int)
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if !ok { return }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}
```

**El patrón tee — duplicar un stream:**

```go
func Tee(done <-chan struct{}, in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    go func() {
        defer close(out1)
        defer close(out2)
        for val := range OrDone(done, in) {
            var o1, o2 = out1, out2
            for i := 0; i < 2; i++ {
                select {
                case <-done:
                case o1 <- val:
                    o1 = nil  // ya envié a out1 — deshabilitar este case
                case o2 <- val:
                    o2 = nil  // ya envié a out2 — deshabilitar este case
                }
            }
        }
    }()
    return out1, out2
}
```

---

### Ejercicio 5.3.1 — Or-channel: cancelar cuando cualquiera termina

**Enunciado:** Implementa `Or` que retorna un canal que se cierra cuando
cualquiera de los canales de entrada se cierra:

```go
func Or(channels ...<-chan struct{}) <-chan struct{}
```

Esto permite combinar múltiples señales de done en una sola.

**Restricciones:** Funciona para cualquier número de canales (1 a N).
Para N=0, retorna un canal que nunca se cierra. Para N=1, retorna ese mismo canal.
La implementación recursiva es elegante para N grande.

**Pista:** La implementación recursiva: si hay 1 canal, retornarlo.
Si hay más, hacer `select` entre el primero y `Or(el_resto...)`.
Esto crea un árbol de goroutines que se cancelan en cascada cuando
cualquier hoja se cierra.

**Implementar en:** Go · (este patrón es específico de Go — compara con Java y Python)

---

### Ejercicio 5.3.2 — Bridge channel: aplanar canal de canales

**Enunciado:** Implementa `Bridge` que convierte un canal de canales en
un canal plano:

```go
// Entrada: <-chan (<-chan int)  — un stream de streams
// Salida:  <-chan int           — un stream plano

func Bridge(done <-chan struct{}, chanStream <-chan (<-chan int)) <-chan int
```

**Restricciones:** Los items del canal interno se emiten en orden.
Cuando el canal interno se cierra, Bridge empieza a leer el siguiente.
Cuando el canal externo se cierra, Bridge termina limpiamente.

**Pista:** Bridge es útil cuando tienes generadores que producen canales —
por ejemplo, páginas de resultados paginados. Cada página es un canal,
y Bridge los aplana en un stream continuo sin que el consumidor
necesite saber sobre la paginación.

**Implementar en:** Go

---

### Ejercicio 5.3.3 — Tee: procesar un stream en dos pipelines

**Enunciado:** Implementa el patrón Tee del inicio de la sección.
Úsalo para enviar el mismo stream de logs simultáneamente a:
1. Un procesador que analiza errores en tiempo real
2. Un escritor que persiste los logs a disco

**Restricciones:** Ambas ramas reciben todos los logs — no se pierden
items aunque una rama procese más lento que la otra.
Verifica que si una rama cancela, la otra continúa.

**Pista:** El Tee usa el patrón de canal nil para garantizar que ambas
ramas reciben el mismo item. El `select` con `o1 = nil` después de enviar
a out1 previene que se envíe el mismo item dos veces a out1.

**Implementar en:** Go · Java · Python

---

### Ejercicio 5.3.4 — Repeat y Take: generadores infinitos

**Enunciado:** Implementa generadores estilo functional programming con canales:

```go
// Repeat emite los valores indefinidamente en loop
func Repeat(done <-chan struct{}, values ...int) <-chan int

// RepeatFn llama la función indefinidamente y emite su resultado
func RepeatFn(done <-chan struct{}, fn func() int) <-chan int

// Take toma solo los primeros N items de un canal
func Take(done <-chan struct{}, in <-chan int, num int) <-chan int
```

Úsalos para generar datos de prueba y limitar los resultados.

**Restricciones:** `Repeat` y `RepeatFn` son infinitos — nunca cierran
su canal de salida sin una señal done. `Take` cierra su canal de salida
después de N items y señaliza done para que el generador upstream termine.

**Pista:** `Repeat(done, 1, 2, 3)` emite `1, 2, 3, 1, 2, 3, 1, 2, 3, ...`
hasta que done se cierra. `Take(done, Repeat(done, 1,2,3), 5)` da `1,2,3,1,2`.
Estos son los building blocks de la librería `iter` de Go 1.23+.

**Implementar en:** Go · Python (generators) · Rust (iterators)

---

### Ejercicio 5.3.5 — El patrón de goroutine supervisor

**Enunciado:** Implementa un supervisor que reinicia goroutines que terminan
inesperadamente (por pánico o error):

```go
type Supervisor struct {
    trabajos []func(ctx context.Context) error
}

func (s *Supervisor) Supervisar(ctx context.Context) error {
    // lanza cada trabajo como goroutine
    // si alguno termina con error, lo reinicia con backoff
    // si el contexto se cancela, para todo limpiamente
}
```

**Restricciones:** El reintento usa backoff exponencial con jitter.
Un trabajo que falla más de N veces consecutivas se abandona con error.
Si el contexto se cancela mientras un trabajo está en el backoff, termina
inmediatamente sin esperar el próximo reintento.

**Pista:** El supervisor es el patrón de supervisor de Erlang/OTP adaptado a Go.
Cada trabajo corre en su propia goroutine. El supervisor hace `select` entre
el done del trabajo y el done del contexto. Si termina el trabajo, decide
si reiniciar según la política.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 5.4 — errgroup y Sincronización de Errores

`golang.org/x/sync/errgroup` resuelve el problema más común con goroutines:
lanzar N goroutines, esperar todas, y recolectar el primer error.

**El problema que errgroup resuelve:**

```go
// Sin errgroup — verboso y propenso a errores:
var wg sync.WaitGroup
errs := make(chan error, N)

for _, item := range items {
    wg.Add(1)
    go func(i Item) {
        defer wg.Done()
        if err := procesar(i); err != nil {
            errs <- err
        }
    }(item)
}

wg.Wait()
close(errs)

var firstErr error
for err := range errs {
    if firstErr == nil {
        firstErr = err
    }
}

// Con errgroup:
g, ctx := errgroup.WithContext(context.Background())

for _, item := range items {
    item := item
    g.Go(func() error {
        return procesar(ctx, item)
    })
}

if err := g.Wait(); err != nil {
    // primer error de cualquier goroutine
}
```

**Lo que errgroup hace internamente:**
- Espera a que todas las goroutines terminen
- Retorna el primer error no-nil
- Si usa `WithContext`: cancela el contexto cuando cualquier goroutine falla

---

### Ejercicio 5.4.1 — errgroup básico

**Enunciado:** Refactoriza el Worker Pool del Cap.03 para usar `errgroup`.
El primer error debe cancelar el contexto, haciendo que las otras goroutines
terminen limpiamente.

**Restricciones:** Verifica que cuando una goroutine falla:
1. Las goroutines restantes detectan `ctx.Done()` y terminan
2. `g.Wait()` retorna el primer error
3. No hay goroutine leaks

**Pista:** `errgroup.WithContext` retorna un grupo y un contexto derivado.
Cuando cualquier goroutine lanzada con `g.Go` retorna un error no-nil,
el contexto se cancela automáticamente. Las otras goroutines deben verificar
`ctx.Done()` para detectar la cancelación.

**Implementar en:** Go · Java (`CompletableFuture.allOf` + primer error) · Python (`asyncio.gather`) · C#

---

### Ejercicio 5.4.2 — errgroup con límite de concurrencia

**Enunciado:** `errgroup` básico no limita la concurrencia. Con 10,000 items,
lanzaría 10,000 goroutines. Usa `errgroup.SetLimit` (Go 1.20+) o implementa
el límite manualmente:

```go
g.SetLimit(10)  // máximo 10 goroutines en paralelo

for _, item := range items {
    item := item
    g.Go(func() error {
        return procesar(ctx, item)
    })
}
```

**Restricciones:** Verifica que en ningún momento hay más de N goroutines
activas (cuenta con un atómico). Compara `SetLimit` con la implementación
manual usando un semáforo (canal con buffer).

**Pista:** `g.Go` con `SetLimit` bloquea si ya hay N goroutines activas.
Sin `SetLimit`, lanza inmediatamente. Para Go < 1.20, el semáforo manual
es: `sem := make(chan struct{}, N)` con `sem <- struct{}{}` antes y
`<-sem` después de cada goroutine.

**Implementar en:** Go · Java · Python · C#

---

### Ejercicio 5.4.3 — Errores parciales: continuar después de fallos

**Enunciado:** A veces quieres continuar aunque algunas goroutines fallen
y recolectar todos los errores. `errgroup` retorna solo el primero.
Implementa `MultiErrorGroup` que recolecta todos:

```go
type MultiErrorGroup struct {
    wg   sync.WaitGroup
    mu   sync.Mutex
    errs []error
}

func (g *MultiErrorGroup) Go(f func() error)
func (g *MultiErrorGroup) Wait() []error
```

**Restricciones:** Las goroutines que tienen éxito no se ven afectadas
por las que fallan. `Wait` retorna todos los errores, no solo el primero.

**Pista:** Un ejemplo de uso real: validar 1000 registros donde quieres
saber cuáles fallan (todos los errores), no abortar al primer fallo.
Los errores pueden ser de tipos distintos — considera usar `errors.Join`
(Go 1.20+) para combinarlos.

**Implementar en:** Go · Java · Python · C# · Rust

---

### Ejercicio 5.4.4 — semaphore del paquete x/sync

**Enunciado:** `golang.org/x/sync/semaphore` provee un semáforo ponderado:
cada goroutine puede adquirir N "unidades" de peso, no solo 1.

```go
// Útil cuando las goroutines tienen costos distintos:
// una goroutine que procesa 100MB debería contar más que
// una que procesa 1KB

sem := semaphore.NewWeighted(int64(runtime.NumCPU()))

for _, item := range items {
    peso := int64(len(item.Datos) / 1024 / 1024)  // MB como peso
    if err := sem.Acquire(ctx, peso); err != nil {
        return err
    }
    go func(i Item, p int64) {
        defer sem.Release(p)
        procesar(i)
    }(item, peso)
}
```

**Restricciones:** Implementa un sistema de procesamiento de archivos donde
items grandes "pesan" más. El sistema no debe exceder N GB de uso de memoria
simultáneo (donde N = peso total adquirido).

**Pista:** El semáforo ponderado es útil cuando los trabajos tienen costos
heterogéneos. Un semáforo simple (peso=1 siempre) subestimaría el costo
de items grandes y podría saturar la memoria.

**Implementar en:** Go · Java (`Semaphore.acquire(permits)`) · Python · C#

---

### Ejercicio 5.4.5 — singleflight revisitado con errgroup

**Enunciado:** Implementa una caché que:
1. Usa `singleflight` para evitar trabajo duplicado (Ejercicio 3.5.3)
2. Usa `errgroup` para refrescar múltiples entradas expiradas en paralelo
3. Limita la concurrencia de refrescos con `SetLimit`

```go
type CacheInteligente[K comparable, V any] struct {
    datos     map[K]entrada[V]
    sf        singleflight.Group
    mu        sync.RWMutex
    cargar    func(ctx context.Context, key K) (V, error)
}
```

**Restricciones:** Si múltiples goroutines piden la misma clave que está
refrescando, todas esperan el mismo resultado (singleflight).
El refresco de múltiples claves expiradas ocurre en paralelo (errgroup).
Máximo 5 refrescos simultáneos (SetLimit).

**Pista:** La composición singleflight + errgroup es el núcleo de muchas
cachés de producción. `singleflight` evita el thundering herd para una clave.
`errgroup` con límite evita saturar el backend cuando muchas claves expiran
simultáneamente (cache stampede).

**Implementar en:** Go · Java · Python

---

## Sección 5.5 — Testing de Código Concurrente en Go

El testing de código concurrente tiene requisitos que el testing normal no tiene.

**Los cinco principios:**

```
1. Determinismo:   el test debe fallar consistentemente cuando hay un bug,
                   no aleatoriamente

2. Velocidad:      los tests de concurrencia tienden a ser lentos por sleeps
                   y timeouts — minimizarlos con inyección de tiempo

3. Aislamiento:    cada test empieza con estado limpio — sin goroutines
                   residuales del test anterior

4. Race detector:  siempre correr con -race en CI

5. Timeouts:       todo test de concurrencia necesita timeout —
                   un deadlock haría que el test corra para siempre
```

**La herramienta más importante: goleak**

```go
func TestWorkerPool(t *testing.T) {
    defer goleak.VerifyNone(t)  // falla si hay goroutines al terminar

    pool := NuevoPool(5)
    pool.Submit(trabajo)
    pool.Detener()
    // si Detener() tiene un leak, goleak lo detecta
}
```

**Inyección de tiempo — eliminar sleeps de tests:**

```go
// En vez de: time.Sleep(100 * time.Millisecond)
// Inyectar el reloj:

type Clock interface {
    Now() time.Time
    Sleep(d time.Duration)
    After(d time.Duration) <-chan time.Time
}

type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }
func (RealClock) Sleep(d time.Duration) { time.Sleep(d) }

type FakeClock struct { /* tiempo controlable */ }
// En tests, FakeClock avanza el tiempo sin espera real
```

---

### Ejercicio 5.5.1 — Test con goleak

**Enunciado:** Toma el sistema del Cap.03 §3.7.3 (Pub/Sub + Worker Pool)
y escribe tests completos con `goleak`. Cada test debe verificar que
no hay goroutine leaks al terminar.

**Restricciones:** Los tests cubren el ciclo completo: iniciar, procesar
eventos, detener. Incluye el caso de error: el sistema se detiene
correctamente incluso si un worker entra en pánico.

**Pista:** El orden de operaciones en el teardown del test importa.
Si usas `defer goleak.VerifyNone(t)`, goleak verifica al final del test.
Si el sistema todavía tiene goroutines activas en ese punto, el test falla.
Asegúrate de llamar `sistema.Detener()` antes de que goleak verifique.

**Implementar en:** Go

---

### Ejercicio 5.5.2 — Inyección de tiempo en tests de timeout

**Enunciado:** El Circuit Breaker del Ejercicio 3.6.1 usa `time.Now()`
para decidir cuándo pasar de ABIERTO a SEMI. El test que verifica esta
transición necesitaría `time.Sleep(tiempoEspera)` — lento y frágil.

Refactoriza el Circuit Breaker para inyectar el reloj:

```go
type CircuitBreaker struct {
    clock        Clock  // inyectable
    tiempoEspera time.Duration
    // ...
}

// En el test:
fakeClock := &FakeClock{ahora: time.Now()}
cb := NuevoCircuitBreaker(fakeClock, 5*time.Second)

// Simular que pasan 6 segundos sin time.Sleep:
fakeClock.Avanzar(6 * time.Second)
// ahora el CB debería pasar a SEMI
```

**Restricciones:** El Clock real usa `time.Now()` y `time.Sleep()`.
El Clock falso avanza el tiempo con un método `Avanzar(d)`.
Los tests no deben tener ningún `time.Sleep`.

**Pista:** La inyección de tiempo es el patrón más importante para
tests de concurrencia rápidos. Cualquier componente que use tiempo
(timeouts, TTLs, rate limiters, circuit breakers) debe poder inyectar el reloj.

**Implementar en:** Go · Java (Clock de java.time) · Python · C#

---

### Ejercicio 5.5.3 — Table-driven tests para patrones concurrentes

**Enunciado:** Implementa tests table-driven para el Worker Pool que
cubren todas las combinaciones relevantes:

```go
func TestWorkerPool(t *testing.T) {
    tests := []struct {
        nombre      string
        numWorkers  int
        numJobs     int
        jobDuracion time.Duration
        errores     int  // cuántos jobs fallan
        esperado    int  // resultados exitosos esperados
    }{
        {"básico", 4, 100, 0, 0, 100},
        {"más jobs que workers", 2, 50, 0, 0, 50},
        {"con errores", 4, 100, 0, 10, 90},
        {"cancelación a mitad", 4, 100, 10*time.Millisecond, 0, -1},  // resultado variable
        // ...
    }
}
```

**Restricciones:** Cada test debe ejecutar con un timeout propio.
El test de cancelación debe verificar que no hay goroutine leaks.
Los tests deben ser reproducibles — sin dependencia de timing del sistema.

**Pista:** Para tests con cancelación donde el número de resultados es
variable, verifica invariantes en vez de valores exactos:
`len(resultados) <= numJobs` y `len(resultados) >= numWorkers`.

**Implementar en:** Go · Java · Python · Rust · C#

---

### Ejercicio 5.5.4 — Fuzz testing de código concurrente

**Enunciado:** Go 1.18+ tiene soporte nativo para fuzzing. Implementa un
fuzz test para la pila concurrente del Ejercicio 1.1.4 que genera
secuencias aleatorias de Push/Pop desde múltiples goroutines y verifica
las invariantes:

```go
func FuzzPilaConcurrente(f *testing.F) {
    f.Add([]byte{0, 1, 0, 1})  // seed: push, pop, push, pop
    f.Fuzz(func(t *testing.T, ops []byte) {
        // ops[i] % 2 == 0: Push, ops[i] % 2 == 1: Pop
        // verifica: Len() >= 0 siempre
    })
}
```

**Restricciones:** El fuzz test debe correr con `-race`.
Las invariantes a verificar: `Len() >= 0` siempre, y si solo hay Pushes,
`Len()` equals el número de Pushes.

**Pista:** El fuzzing de código concurrente es especialmente valioso porque
el fuzzer puede encontrar secuencias de operaciones que exponen race conditions
que los tests normales no alcanzan. Corre con `go test -fuzz=FuzzPilaConcurrente -fuzztime=30s`.

**Implementar en:** Go

---

### Ejercicio 5.5.5 — Benchmark de concurrencia: medir throughput real

**Enunciado:** Implementa benchmarks que miden el throughput real del
Worker Pool bajo distintas configuraciones:

```go
func BenchmarkWorkerPool(b *testing.B) {
    for _, tc := range []struct {
        workers int
        jobMs   time.Duration
    }{
        {1, 1 * time.Millisecond},
        {4, 1 * time.Millisecond},
        {runtime.NumCPU(), 1 * time.Millisecond},
        {runtime.NumCPU() * 4, 1 * time.Millisecond},  // sobre-provisionar
    } {
        b.Run(fmt.Sprintf("w%d_j%s", tc.workers, tc.jobMs), func(b *testing.B) {
            // b.N iteraciones del pool
        })
    }
}
```

**Restricciones:** El benchmark mide operaciones/segundo, no ns/operación.
Usa `b.SetParallelism` para controlar la concurrencia del benchmark mismo.
Reporta el throughput óptimo y el número de workers que lo logra.

**Pista:** `b.RunParallel` lanza goroutines que compiten por el pool —
esto es el escenario real de uso. La contención en el canal de jobs
puede convertirse en el cuello de botella para N workers muy alto.

**Implementar en:** Go · Java (JMH) · Rust (criterion)

---

## Sección 5.6 — Go vs Java Virtual Threads vs Python asyncio

Cada lenguaje tiene su modelo de concurrencia. Esta sección compara los
tres principales — no para declarar un ganador sino para entender
cuándo cada uno es la herramienta correcta.

**La comparación en una tabla:**

```
                    Go goroutines    Java VThreads    Python asyncio
────────────────    ─────────────    ─────────────    ──────────────
Modelo              M:N              M:N              1:1 (cooperative)
Paralelismo CPU     Sí (GOMAXPROCS)  Sí (JVM threads) No (GIL)
Stack inicial       2-8 KB           ~1 KB            N/A (frames)
Creación            ~400 ns          ~100 ns          ~1 µs
Preempción          Sí (Go 1.14)     Sí               No (yield explícito)
I/O async nativo    Sí               Sí (Java 21+)    Sí (event loop)
Cancelación         context.Context  CancellationToken asyncio.Task.cancel
Detección races     go run -race     ThreadSanitizer  Ninguna (GIL enmascara)
Ecosistema async    Completo         En transición    Completo
```

---

### Ejercicio 5.6.1 — El mismo servidor en Go, Java y Python

**Enunciado:** Implementa un servidor HTTP concurrente que maneja cada
request en paralelo. La lógica: recibe un número, duerme ese número
de milisegundos (simula I/O), retorna el número multiplicado por 2.

Mide con el mismo cliente de carga:
- Latencia p50, p95, p99
- Throughput (requests/segundo)
- Uso de memoria con 10,000 conexiones simultáneas

**Restricciones:** Exactamente la misma lógica de negocio en los tres.
El cliente de carga es el mismo (k6 o hey).

**Pista:** Go usa goroutinas por request (el modelo más simple).
Java puede usar Virtual Threads (Java 21+) para el mismo patrón.
Python asyncio usa `async def` handlers.
La diferencia más visible será en el uso de memoria con muchas conexiones.

**Implementar en:** Go · Java 21+ · Python (FastAPI + asyncio)

---

### Ejercicio 5.6.2 — CPU-bound: donde Python asyncio no ayuda

**Enunciado:** Implementa el mismo trabajo CPU-bound (calcular N primos)
en los tres lenguajes, con y sin "concurrencia":

- Go: con goroutines y `GOMAXPROCS=NumCPU`
- Java: con Virtual Threads y con ForkJoinPool
- Python: con asyncio, con threading, y con multiprocessing

**Restricciones:** Mide el speedup real vs la versión secuencial para cada.
Explica por qué Python asyncio no ayuda para CPU-bound.

**Pista:** Python asyncio es concurrencia cooperativa en **un solo thread**.
El GIL impide que múltiples threads Python ejecuten Python bytecode en paralelo.
Para CPU-bound real en Python, necesitas `multiprocessing` — procesos separados,
no threads. Esto es un compromiso fundamental, no un bug.

**Implementar en:** Go · Java · Python

---

### Ejercicio 5.6.3 — Manejo de errores: distintas filosofías

**Enunciado:** Para el mismo patrón (ejecutar N tareas concurrentes,
recolectar el primer error), compara la ergonomía en los tres lenguajes:

- Go: `errgroup.WithContext`
- Java: `CompletableFuture.allOf` + excepciones
- Python: `asyncio.gather(return_exceptions=True)`

**Restricciones:** Implementa el mismo escenario: 10 tareas, la 7ª falla.
Verifica que en los tres casos: la falla se propaga al llamador,
las tareas restantes se cancelan (si el lenguaje lo soporta),
y no hay resource leaks.

**Pista:** Go requiere que las goroutines verifiquen `ctx.Done()` — la cancelación
es cooperativa pero no automática. Java con Virtual Threads puede interrumpir
threads. Python con asyncio puede cancelar tasks. La diferencia es cuánto
trabajo manual requiere cada modelo.

**Implementar en:** Go · Java · Python

---

### Ejercicio 5.6.4 — Migrar de asyncio a goroutines

**Enunciado:** Tienes código Python con asyncio que necesitas migrar a Go.
Identifica los patrones equivalentes:

```python
# Python asyncio
async def procesar(item):
    resultado = await obtener_datos(item)
    return transformar(resultado)

async def main():
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(procesar(item)) for item in items]
    resultados = [t.result() for t in tasks]
```

**Restricciones:** La versión Go debe tener el mismo comportamiento:
todas las tareas en paralelo, cancelación si una falla, recolección de resultados.

**Pista:** `asyncio.TaskGroup` es el equivalente de `errgroup.WithContext`.
`await` es el equivalente de recibir de un canal o de `g.Wait()`.
La diferencia: asyncio es cooperativa (un thread), goroutines son paralelas.

**Implementar en:** Go · Python

---

### Ejercicio 5.6.5 — Cuándo elegir cada modelo

**Enunciado:** Para cada escenario, elige el lenguaje y el modelo de
concurrencia más apropiado y justifica:

1. Servidor web con 100,000 conexiones simultáneas, mayoritariamente I/O
2. Procesamiento de imágenes CPU-intensivo en un servidor con 32 cores
3. Script de automatización que hace 50 llamadas HTTP y agrega resultados
4. Sistema de trading de alta frecuencia con latencia < 1ms
5. Servicio de ML que corre inferencias en GPU

**Restricciones:** No hay una respuesta única correcta — la justificación
importa más que la elección. Considera: ecosistema, equipo, rendimiento,
mantenibilidad.

**Pista:** El escenario 4 (latencia < 1ms) descarta el GC de Go y Java —
los pauses del GC son impredecibles. El escenario 5 (GPU) el lenguaje importa
menos que el framework (CUDA, TensorFlow). El escenario 1 es el territorio
donde Go brilla más claramente.

**Implementar en:** Ensayo justificado + código de prueba de concepto

---

## Sección 5.7 — Diseño de APIs Concurrentes

Una API concurrente bien diseñada es difícil de usar incorrectamente.
Una mal diseñada invita a los bugs aunque el llamador tenga buenas intenciones.

**Los principios de una buena API concurrente:**

```
1. El llamador no necesita conocer los detalles de sincronización interna
   — la concurrencia es un detalle de implementación

2. La API expresa claramente qué es thread-safe y qué no
   — documentado en godoc, no solo en los comentarios internos

3. La cancelación y los errores se propagan de forma consistente
   — context.Context para cancelación, error return para errores

4. Las goroutines lanzadas por la API tienen un ciclo de vida claro
   — el llamador sabe cuándo terminan y cómo pararlas

5. La API no asume nada sobre el número de goroutines del llamador
   — funciona igual con 1 goroutine que con 1000
```

**Anti-patrones comunes:**

```go
// Anti-patrón 1: exponer el canal interno
type Cola struct {
    Items chan Item  // exportado — el llamador puede cerrarlo por error
}

// Mejor: canal interno, métodos públicos
type Cola struct {
    items chan Item  // no exportado
}
func (c *Cola) Enviar(item Item) error { ... }
func (c *Cola) Recibir() (Item, bool)  { ... }

// Anti-patrón 2: goroutine sin forma de terminar
func IniciarWorker() {
    go func() {
        for { trabajar() }  // nunca termina
    }()
    // el llamador no puede parar el worker
}

// Mejor: retornar una función de parada
func IniciarWorker() (detener func()) {
    ctx, cancel := context.WithCancel(context.Background())
    go func() {
        for {
            select {
            case <-ctx.Done(): return
            default: trabajar()
            }
        }
    }()
    return cancel
}
```

---

### Ejercicio 5.7.1 — Diseñar una API de Worker Pool pública

**Enunciado:** Diseña la API pública de un Worker Pool que sea difícil
de usar incorrectamente. La API debe:
- Hacer imposible enviar a un pool cerrado (sin panic)
- Proporcionar backpressure configurable
- Soportar cancelación limpia
- Exponer métricas sin race condition

```go
type Pool[J any, R any] struct{ /* ... */ }

func Nuevo[J, R any](opts Opciones) *Pool[J, R]
func (p *Pool[J, R]) Submit(ctx context.Context, job J) error
func (p *Pool[J, R]) Resultados() <-chan Resultado[R]
func (p *Pool[J, R]) Detener()
func (p *Pool[J, R]) Metricas() Metricas
```

**Restricciones:** `Submit` en un pool cerrado retorna `ErrPoolCerrado`,
no entra en pánico. `Detener` es idempotente — se puede llamar múltiples veces.
`Metricas` es thread-safe y sin mutex visible para el llamador.

**Pista:** El tipo `Pool` usa generics (Go 1.18+) para ser type-safe.
`Submit` retorna error en lugar de bloquear indefinidamente cuando el pool
está lleno o cerrado — esto da al llamador control sobre cómo manejar
el back-pressure.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 5.7.2 — API con opciones funcionales

**Enunciado:** Las opciones funcionales son el patrón idiomático de Go
para APIs configurables. Aplícalo al Pool del ejercicio anterior:

```go
type Option func(*Pool)

func ConWorkers(n int) Option {
    return func(p *Pool) { p.numWorkers = n }
}

func ConBuffer(n int) Option {
    return func(p *Pool) { p.bufferSize = n }
}

func ConTimeout(d time.Duration) Option {
    return func(p *Pool) { p.timeout = d }
}

// Uso:
pool := Nuevo(ConWorkers(10), ConBuffer(100), ConTimeout(5*time.Second))
```

**Restricciones:** Cada opción tiene un valor por defecto razonable.
Las opciones inválidas retornan error en `Nuevo`, no en tiempo de uso.
La API es extensible — se pueden añadir opciones sin cambiar la firma de `Nuevo`.

**Pista:** Las opciones funcionales son preferibles a structs de configuración
porque son autodocumentadas (`ConWorkers(10)` es más legible que `Config{Workers: 10}`)
y permiten añadir opciones sin breaking changes en la API.

**Implementar en:** Go · Java (Builder pattern) · Python (kwargs)

---

### Ejercicio 5.7.3 — Documentar la seguridad de concurrencia

**Enunciado:** Documenta el Pool del Ejercicio 5.7.1 siguiendo las
convenciones de Go para concurrencia en godoc:

```go
// Pool es un worker pool concurrente thread-safe.
// Todos los métodos son seguros para llamar desde múltiples goroutines.
//
// El ciclo de vida típico:
//
//    pool := Nuevo(ConWorkers(10))
//    defer pool.Detener()
//
//    pool.Submit(ctx, trabajo)
//    resultado := <-pool.Resultados()
//
// Submit retorna ErrPoolCerrado si se llama después de Detener.
// Detener bloquea hasta que todos los trabajos en curso terminan.
type Pool struct { ... }
```

**Restricciones:** La documentación debe responder sin ambigüedad:
¿Es thread-safe? ¿Qué pasa si se llama después de cerrar? ¿Bloquea o no bloquea?
¿Cuándo se cierra el canal de resultados?

**Pista:** La documentación de concurrencia en Go sigue el estilo de
la librería estándar. Ejemplos: `sync.Map` documenta "safe for concurrent use",
`sync.WaitGroup` documenta que "must not be copied after first use".

**Implementar en:** Go · Java (Javadoc de java.util.concurrent) · Python

---

### Ejercicio 5.7.4 — Versionar una API concurrente sin romper backward compatibility

**Enunciado:** El Pool v1 tiene esta API:

```go
// v1 — no soporta cancelación
func (p *Pool) Submit(job Job) error
```

El Pool v2 necesita soportar cancelación:

```go
// v2 — con cancelación
func (p *Pool) Submit(ctx context.Context, job Job) error
```

El cambio rompe todos los llamadores de v1. Diseña una estrategia de
migración que mantenga backward compatibility durante un periodo de transición.

**Restricciones:** Los llamadores de v1 siguen funcionando sin cambios.
Los nuevos llamadores usan v2. El código interno de Pool se implementa
una sola vez.

**Pista:** Opciones de migración:
1. Nuevo método `SubmitWithContext` — preserva `Submit` como wrapper con `context.Background()`
2. Módulo nuevo `pool/v2` — breaking change explícito
3. `Submit` acepta variadic `...SubmitOption` — backward compatible

**Implementar en:** Go · Java · Python

---

### Ejercicio 5.7.5 — Code review completo: el Pool en producción

**Enunciado:** Analiza esta implementación de Pool como si fuera un
code review real. Identifica todos los problemas de diseño de API,
de corrección concurrente, y de ergonomía:

```go
type Pool struct {
    Jobs     chan Job       // exportado — ¿problema?
    Results  chan Result    // exportado — ¿problema?
    Workers  int
    started  bool
}

func (p *Pool) Start() {
    p.started = true  // sin lock — ¿problema?
    for i := 0; i < p.Workers; i++ {
        go func() {
            for job := range p.Jobs {
                p.Results <- ejecutar(job)
            }
        }()
    }
}

func (p *Pool) Stop() {
    close(p.Jobs)   // ¿qué pasa si se llama dos veces?
    p.started = false
}

func (p *Pool) IsStarted() bool {
    return p.started  // sin lock — ¿problema?
}
```

**Restricciones:** Lista los problemas por categoría: API design, race conditions,
error handling, y goroutine lifecycle. Para cada problema, propón la corrección mínima.

**Pista:** Hay al menos 6 problemas. Algunos son sutiles de diseño de API (campos
exportados), otros son race conditions claras (acceso a `started` sin lock),
y otros son goroutine lifecycle (¿cómo sabe el llamador cuándo los workers terminaron?).

**Implementar en:** Go · Java · Python

---

## Resumen del capítulo

**El principio central:**

> No compartas memoria para comunicarte.
> Comunícate para compartir memoria.

No es una regla absoluta — el mutex tiene su lugar cuando el acceso concurrente
es mayoritariamente lectura o cuando el ownership no cambia. Pero es el punto
de partida correcto: antes de añadir un mutex, pregunta si el problema puede
modelarse como transferencia de ownership a través de un canal.

**Lo que distingue el Go idiomático:**

```
1. context.Context en cada función que puede bloquearse o que tarda tiempo
2. errgroup para lanzar N goroutines y esperar todas con manejo de errores
3. canal nil para deshabilitar cases de select dinámicamente
4. goleak en tests para detectar goroutine leaks
5. opciones funcionales para APIs configurables y extensibles
6. goroutine propietaria para encapsular acceso concurrente a estado
```

**Comparación final:**

```
Go goroutines:     paralelismo real + sintaxis simple + tooling excelente
Java VThreads:     compatible con código existente + JVM madura + ecosystem
Python asyncio:    simple para I/O + no para CPU + GIL es el límite
```

## La pregunta que guía el Cap.06

> Los Cap.01–05 cubrieron la concurrencia desde los problemas fundamentales
> hasta el código idiomático de Go. Los patrones son concretos, los bugs son conocidos,
> las herramientas son claras.
>
> Pero hay una clase de problemas donde el estado compartido mutable es el enemigo —
> no solo un riesgo a gestionar. El Cap.06 cubre los modelos de concurrencia que
> evitan el estado compartido por diseño: actores (Erlang/Akka), CSP (Go ya lo usa),
> y STM (Software Transactional Memory). Son las respuestas de la academia
> que los sistemas de producción adoptaron.
