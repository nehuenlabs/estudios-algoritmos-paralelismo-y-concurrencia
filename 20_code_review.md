# Guía de Ejercicios — Cap.20: Code Review de Código Concurrente

> Lenguaje principal: **Go**. Los patrones aplican a cualquier lenguaje.
>
> Este capítulo tiene un formato distinto a los anteriores.
> Cada ejercicio presenta código real (o basado en código real) con bugs,
> antipatrones, o decisiones de diseño discutibles.
> Tu trabajo es leerlo como lo harías en un PR — sin ejecutarlo,
> encontrar los problemas, y proponer mejoras concretas.
>
> Los tres tipos de problemas que buscar en un code review de concurrencia:
> - **Correctitud**: ¿puede este código tener un data race, deadlock, o resultado incorrecto?
> - **Liveness**: ¿puede bloquearse, morirse de hambre, o acumular recursos indefinidamente?
> - **Diseño**: ¿el ownership de goroutines y canales está claro? ¿es más complejo de lo necesario?

---

## Por qué el code review de concurrencia es diferente

En un code review normal, lees el código y piensas en secuencia:
"primero pasa esto, luego esto, luego esto."

En un code review de concurrencia, tienes que pensar en paralelo:
"¿qué pasa si esta goroutine llega aquí mientras aquella está allá?"

Eso requiere un modelo mental diferente — y una lista de patrones a buscar:

```
Checklist básico de code review de concurrencia:

Goroutines:
  □ ¿Toda goroutine tiene un mecanismo de terminación?
  □ ¿La goroutine puede quedar bloqueada esperando un canal que nadie cerrará?
  □ ¿La closure captura variables por referencia que pueden cambiar?
  □ ¿Quién es responsable de hacer join (WaitGroup, errgroup, canal)?

Canales:
  □ ¿Quién cierra el canal? ¿Solo hay un lugar que lo cierra?
  □ Si es un canal unbuffered, ¿puede el sender bloquearse indefinidamente?
  □ Si es buffered, ¿está el tamaño justificado?
  □ ¿Se usa select con default cuando debería bloquear (o al revés)?

Locks:
  □ ¿El lock protege todos los campos que necesita proteger?
  □ ¿Se puede adquirir el mismo lock dos veces (deadlock)?
  □ ¿El orden de adquisición es consistente en todo el codebase?
  □ ¿Hay código costoso dentro de la sección crítica?

Context y cancelación:
  □ ¿El contexto se pasa a todas las operaciones que pueden bloquearse?
  □ ¿Hay goroutines que ignoran ctx.Done()?
  □ ¿El timeout es apropiado para la operación?

Compartir memoria:
  □ ¿Hay variables globales modificadas por múltiples goroutines?
  □ ¿Hay slices o maps compartidos sin sincronización?
  □ ¿Los campos de structs que se comparten entre goroutines están protegidos?
```

---

## Tabla de contenidos

- [Sección 20.1 — Bugs de correctitud: encontrar el race](#sección-201--bugs-de-correctitud-encontrar-el-race)
- [Sección 20.2 — Bugs de liveness: goroutines que no terminan](#sección-202--bugs-de-liveness-goroutines-que-no-terminan)
- [Sección 20.3 — Diseño de canales: ownership y backpressure](#sección-203--diseño-de-canales-ownership-y-backpressure)
- [Sección 20.4 — Antipatrones de sincronización](#sección-204--antipatrones-de-sincronización)
- [Sección 20.5 — Context y cancelación](#sección-205--context-y-cancelación)
- [Sección 20.6 — Revisión de diseño: ¿es más complejo de lo necesario?](#sección-206--revisión-de-diseño-es-más-complejo-de-lo-necesario)
- [Sección 20.7 — PR completo: revisión de extremo a extremo](#sección-207--pr-completo-revisión-de-extremo-a-extremo)

---

## Sección 20.1 — Bugs de Correctitud: Encontrar el Race

### Ejercicio 20.1.1 — La caché con lazy initialization

Lee este PR. Encuentra todos los problemas de correctitud:

```go
// PR: añadir caché en memoria para reducir llamadas a la BD
// Autor: dev-junior
// Descripción: cachear los resultados de getUserByID para mejorar latencia

type UserService struct {
    db    *sql.DB
    cache map[int64]*User  // añadido en este PR
}

func NewUserService(db *sql.DB) *UserService {
    return &UserService{db: db}
}

func (s *UserService) GetUser(id int64) (*User, error) {
    // Verificar caché primero:
    if user, ok := s.cache[id]; ok {
        return user, nil
    }

    // Cache miss: ir a la BD
    user := &User{}
    err := s.db.QueryRow("SELECT * FROM users WHERE id = ?", id).Scan(user)
    if err != nil {
        return nil, err
    }

    // Guardar en caché:
    s.cache[id] = user
    return user, nil
}
```

**Tu review:** Escribe el comentario de code review completo que enviarías.
Debe incluir:
1. Lista de todos los problemas (hay al menos tres)
2. Para cada problema: explicación de por qué es un problema
3. Código corregido con comentarios
4. Pregunta para el autor si hay algo que necesitas entender

**Pista para el reviewer:** Busca: (1) inicialización del map, (2) acceso sin lock,
(3) el valor nulo que puede cachearse. El map `s.cache` se declara pero nunca
se inicializa — acceder a un map nil es un panic en Go. Aun si se inicializara,
el acceso concurrente sin lock es un data race. Y si la BD devuelve un error
pero parcialmente, ¿qué se cachea?

---

### Ejercicio 20.1.2 — El contador de estadísticas

Lee este PR. ¿Aprobarías? ¿Qué comentarios harías?

```go
// PR: añadir estadísticas de uso al servidor HTTP
// Descripción: contar requests por endpoint para el dashboard

type Estadisticas struct {
    Requests map[string]int64
    Errores  map[string]int64
    mu       sync.Mutex
}

func (e *Estadisticas) RegistrarRequest(endpoint string) {
    e.mu.Lock()
    e.Requests[endpoint]++
    e.mu.Unlock()
}

func (e *Estadisticas) RegistrarError(endpoint string, err error) {
    e.mu.Lock()
    e.Errores[endpoint]++
    e.mu.Unlock()
}

func (e *Estadisticas) Snapshot() map[string]int64 {
    e.mu.Lock()
    defer e.mu.Unlock()
    return e.Requests  // retornar el mapa directamente
}

var stats = &Estadisticas{
    Requests: make(map[string]int64),
    Errores:  make(map[string]int64),
}
```

**Tu review:** Identifica el bug en `Snapshot()` y propón la corrección.
También evalúa: ¿es `sync.Mutex` la mejor herramienta aquí o hay alternativas
más eficientes para este patrón?

**Pista para el reviewer:** `Snapshot()` retorna el mapa interno bajo lock, pero
el caller puede leerlo *después* de que el lock se libera. Si otra goroutine
modifica `Requests` mientras el caller itera el snapshot, hay un race.
La corrección: copiar el mapa dentro del lock.
Para la segunda pregunta: con muchas lecturas y pocas escrituras, considera
`sync.RWMutex`. Con muy alta concurrencia en escritura, considera `sync/atomic`
o un mapa por goroutine con reducción periódica.

---

### Ejercicio 20.1.3 — El rate limiter con sliding window

Lee este PR. Hay un bug de correctitud sutil:

```go
// PR: implementar rate limiter con ventana deslizante
// Límite: max 100 requests por segundo por cliente

type RateLimiter struct {
    mu       sync.Mutex
    ventanas map[string][]time.Time
}

func (rl *RateLimiter) Permitir(clientID string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    ahora := time.Now()
    hace1s := ahora.Add(-time.Second)

    // Obtener o crear la ventana del cliente:
    ventana := rl.ventanas[clientID]

    // Eliminar timestamps fuera de la ventana:
    inicio := 0
    for inicio < len(ventana) && ventana[inicio].Before(hace1s) {
        inicio++
    }
    ventana = ventana[inicio:]

    // Verificar el límite:
    if len(ventana) >= 100 {
        rl.ventanas[clientID] = ventana
        return false
    }

    // Añadir el timestamp actual:
    rl.ventanas[clientID] = append(ventana, ahora)
    return true
}
```

**Tu review:** El código es thread-safe (tiene el mutex). Pero hay un problema
de correctitud funcional y un problema de memoria. Identifica ambos.

**Pista para el reviewer:** El problema de correctitud: `ventana = ventana[inicio:]`
crea un nuevo slice que apunta al mismo array subyacente. El `append` posterior
puede sobrescribir elementos del array original si hay capacidad. Esto no es
un race (el mutex lo protege) sino un bug de semántica de slices. El problema
de memoria: el mapa crece indefinidamente — los clientes que dejan de hacer
requests siguen acumulando entradas en el mapa. Necesitas una limpieza periódica
o TTL por cliente.

---

### Ejercicio 20.1.4 — La goroutine con closure mal capturada

Lee este PR. El autor dice que los tests pasan. ¿Hay un bug?

```go
// PR: paralelizar el procesamiento de notificaciones
// Los tests pasan localmente y en CI

func EnviarNotificaciones(usuarios []Usuario, mensaje string) error {
    var wg sync.WaitGroup
    errores := make([]error, len(usuarios))

    for i, usuario := range usuarios {
        wg.Add(1)
        go func() {
            defer wg.Done()
            err := enviarEmail(usuario.Email, mensaje)
            if err != nil {
                errores[i] = err
            }
        }()
    }

    wg.Wait()

    for _, err := range errores {
        if err != nil {
            return err  // retornar el primero
        }
    }
    return nil
}
```

**Tu review:** Hay dos problemas. El primero es el bug clásico de Go con
closures en loops (pre-Go 1.22). El segundo es más sutil: incluso si la
closure fuera correcta, hay un race condition. Identifica ambos.

**Pista para el reviewer:** Bug 1: `usuario` e `i` son variables del loop.
Antes de Go 1.22, todas las goroutines capturan la *misma* variable,
que tiene el valor de la última iteración cuando la goroutine corre.
Bug 2: `errores[i] = err` — múltiples goroutines escriben en índices
distintos del slice `errores`. Esto es seguro (no hay race en la escritura
de índices distintos), PERO `return err` en el loop final itera el slice
sin garantía de que todas las goroutines hayan terminado... espera, sí hay
`wg.Wait()`. Entonces el bug 2 es: si los tests se ejecutan con Go 1.21,
el bug de closure es el problema. Con Go 1.22, el bug de closure desaparece
y el código es correcto. Necesitas documentar qué versión de Go se requiere.

---

### Ejercicio 20.1.5 — La sesión compartida

Lee este PR. El autor afirma que el singleton es thread-safe porque usa sync.Once:

```go
// PR: singleton para el cliente HTTP con configuración compartida

var (
    clienteHTTP *http.Client
    once        sync.Once
)

func ObtenerCliente() *http.Client {
    once.Do(func() {
        clienteHTTP = &http.Client{
            Timeout: 30 * time.Second,
            Transport: &http.Transport{
                MaxIdleConns:    100,
                IdleConnTimeout: 90 * time.Second,
            },
        }
    })
    return clienteHTTP
}

func HacerRequest(ctx context.Context, url string) (*http.Response, error) {
    cliente := ObtenerCliente()
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    // Añadir headers de autenticación (varían por request):
    req.Header.Set("Authorization", obtenerToken())
    cliente.Transport.(*http.Transport).TLSClientConfig = obtenerTLSConfig()
    // ↑ PROBLEMA: modificar el Transport compartido

    return cliente.Do(req)
}
```

**Tu review:** `sync.Once` garantiza la inicialización correcta del cliente.
Pero hay un data race en `HacerRequest`. Identifícalo y propón la corrección
que no requiere crear un nuevo cliente por request.

**Pista para el reviewer:** `cliente.Transport.(*http.Transport).TLSClientConfig = ...`
modifica el `Transport` compartido entre todos los requests. Si dos goroutines
hacen requests simultáneamente con diferentes configuraciones TLS, hay un race.
La corrección: no modificar el Transport compartido. En su lugar, crear un
`Transport` por request para las configuraciones que varían (o usar un middleware
de auth que no modifica el transport).

---

## Sección 20.2 — Bugs de Liveness: Goroutines que No Terminan

### Ejercicio 20.2.1 — El worker pool sin shutdown

Lee este PR. ¿Cuándo este código deja goroutines huérfanas?

```go
// PR: worker pool para procesar jobs en background

func IniciarWorkers(n int, jobs <-chan Job) {
    for i := 0; i < n; i++ {
        go func() {
            for job := range jobs {
                procesarJob(job)
            }
        }()
    }
    // No hay mecanismo de shutdown
}

// Uso:
func main() {
    jobs := make(chan Job, 100)
    IniciarWorkers(8, jobs)

    // Enviar jobs...
    for _, j := range obtenerJobs() {
        jobs <- j
    }
    // Al salir de main, los workers quedan corriendo
    // hasta que el proceso termina
}
```

**Tu review:** El código funciona para el caso de uso de `main` (donde el proceso
termina). Pero este `IniciarWorkers` también se usa en tests y en handlers HTTP
de larga vida. Escribe tu review completo incluyendo:
1. Cuándo el comportamiento actual es un problema
2. La API correcta con shutdown
3. Cómo hacer el shutdown graceful (procesar todos los jobs pendientes)

**Pista para el reviewer:** En tests, los workers siguen corriendo entre tests
si `jobs` no se cierra. `goleak.VerifyNone(t)` fallaría. En un servidor HTTP,
cada request que llame `IniciarWorkers` crea 8 goroutines nuevas que nunca terminan.
La corrección: retornar una función `shutdown()` o aceptar un `context.Context`.

---

### Ejercicio 20.2.2 — El fan-out con error handling

Lee este PR. Encuentra el goroutine leak:

```go
// PR: fan-out para hacer N requests paralelas con timeout global

func FanOut(ctx context.Context, urls []string) ([]string, error) {
    resultados := make(chan string, len(urls))
    errores := make(chan error, len(urls))

    for _, url := range urls {
        go func(u string) {
            resp, err := http.Get(u)
            if err != nil {
                errores <- err
                return
            }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                errores <- err
                return
            }
            resultados <- string(body)
        }(url)
    }

    var todos []string
    var primerError error

    for range urls {
        select {
        case r := <-resultados:
            todos = append(todos, r)
        case err := <-errores:
            if primerError == nil {
                primerError = err
            }
        case <-ctx.Done():
            return todos, ctx.Err()
            // ↑ PROBLEMA: goroutines en vuelo no se cancelan
        }
    }

    return todos, primerError
}
```

**Tu review:** Los canales son buffered (tamaño `len(urls)`), lo que parece
prevenir el goroutine leak. Pero hay un escenario donde las goroutines
quedan colgadas. Identifícalo y propón la corrección.

**Pista para el reviewer:** Si `ctx` se cancela, `FanOut` retorna inmediatamente.
Las goroutines en vuelo siguen ejecutando `http.Get(u)` (que puede tardar 30s)
y eventualmente intentan enviar a `resultados` o `errores`. Como los canales
son buffered de tamaño `len(urls)`, pueden enviar sin bloquearse — y luego
terminan. ¿Entonces no hay leak? El leak ocurre cuando `http.Get` no respeta
la cancelación del contexto. La corrección: pasar el contexto a las goroutines
usando `http.NewRequestWithContext(ctx, ...)`.

---

### Ejercicio 20.2.3 — El ticker que no para

Lee este PR. Hay un resource leak:

```go
// PR: health checker que verifica servicios externos periódicamente

type HealthChecker struct {
    servicios []string
    resultados map[string]bool
    mu         sync.RWMutex
}

func (h *HealthChecker) IniciarVerificacion(intervalo time.Duration) {
    go func() {
        ticker := time.NewTicker(intervalo)
        for range ticker.C {
            for _, srv := range h.servicios {
                ok := verificar(srv)
                h.mu.Lock()
                h.resultados[srv] = ok
                h.mu.Unlock()
            }
        }
        // ticker.Stop() nunca se llama
    }()
}

func (h *HealthChecker) EstaActivo(servicio string) bool {
    h.mu.RLock()
    defer h.mu.RUnlock()
    return h.resultados[servicio]
}
```

**Tu review:** El `ticker` no se detiene nunca. Escribe el review incluyendo:
1. El resource leak específico (`time.Ticker` y la goroutine)
2. La corrección con `context.Context`
3. Un test que verifica que el checker se detiene limpiamente

**Pista para el reviewer:** `time.NewTicker` crea un goroutine interna y un canal.
Sin `ticker.Stop()`, el canal nunca se cierra y la goroutine interna sigue
enviando ticks. Más importante: la goroutine del HealthChecker corre para siempre.
Cada llamada a `IniciarVerificacion` crea una goroutine nueva que nunca termina.
Si se llama múltiples veces (por un test o por un reinicio de configuración),
se acumulan goroutines.

---

### Ejercicio 20.2.4 — El producer que bloquea al shutdown

Lee este PR. ¿Qué pasa durante el graceful shutdown?

```go
// PR: producer que lee de Kafka y envía a un worker pool

func IniciarProducer(ctx context.Context, consumer *kafka.Consumer, pool *WorkerPool) {
    go func() {
        for {
            msg, err := consumer.ReadMessage(ctx)
            if err != nil {
                if ctx.Err() != nil {
                    return  // shutdown limpio
                }
                log.Error("error leyendo kafka", err)
                continue
            }

            pool.Submit(func() {  // ← puede bloquearse si el pool está lleno
                procesarMensaje(msg)
            })
        }
    }()
}

// WorkerPool.Submit bloquea si la cola está llena:
func (p *WorkerPool) Submit(fn func()) {
    p.tasks <- fn  // canal unbuffered o bounded
}
```

**Tu review:** Durante el shutdown (cuando `ctx` se cancela), `consumer.ReadMessage`
retorna con error. Pero puede haber un problema antes de llegar a ese punto.
Identifícalo y propón la corrección.

**Pista para el reviewer:** Si `pool.Submit()` bloquea (pool saturado) y `ctx` se cancela,
la goroutine del producer queda bloqueada en `pool.Submit` ignorando la cancelación.
El shutdown no puede completarse hasta que el pool tenga espacio.
La corrección: `Submit` debe aceptar un contexto:
```go
select {
case p.tasks <- fn:
case <-ctx.Done():
    return ctx.Err()
}
```

---

### Ejercicio 20.2.5 — El retry con goroutine acumulada

Lee este PR. El autor dice que el timeout previene goroutine leaks:

```go
// PR: función de retry con timeout por intento

func RetryConTimeout(operacion func() error, maxIntentos int, timeout time.Duration) error {
    for intento := 0; intento < maxIntentos; intento++ {
        errCh := make(chan error, 1)

        go func() {
            errCh <- operacion()
        }()

        select {
        case err := <-errCh:
            if err == nil {
                return nil
            }
            log.Printf("intento %d falló: %v", intento+1, err)
        case <-time.After(timeout):
            log.Printf("intento %d: timeout", intento+1)
            // la goroutine sigue corriendo operacion() en background
        }

        time.Sleep(time.Duration(intento+1) * 100 * time.Millisecond)
    }
    return fmt.Errorf("agotados %d intentos", maxIntentos)
}
```

**Tu review:** El autor dice que el timeout previene el leak. Demuestra que
está equivocado con un escenario específico y propón la corrección correcta.
¿Cuántas goroutines pueden estar activas simultáneamente en el peor caso?

**Pista para el reviewer:** En el peor caso: `maxIntentos` goroutines activas simultáneamente.
Si cada intento hace timeout y lanza una nueva goroutine, y la operación tarda
mucho (más que `maxIntentos × timeout`), todas las goroutines de todos los intentos
están vivas al mismo tiempo. La corrección: pasar un contexto cancelable a `operacion`.
```go
ctx, cancel := context.WithTimeout(context.Background(), timeout)
defer cancel()  // cancela la goroutine aunque el select tome el caso de timeout
errCh <- operacion(ctx)
```

---

## Sección 20.3 — Diseño de Canales: Ownership y Backpressure

### Ejercicio 20.3.1 — ¿Quién cierra el canal?

Lee este PR. El código tiene una convención de ownership inconsistente:

```go
// PR: sistema de pipeline para procesar órdenes

func etapa1(entrada <-chan Orden) <-chan OrdenValidada {
    salida := make(chan OrdenValidada, 100)
    go func() {
        for orden := range entrada {
            if validada, ok := validar(orden); ok {
                salida <- validada
            }
        }
        close(salida)  // esta etapa cierra su salida
    }()
    return salida
}

func etapa2(entrada <-chan OrdenValidada) (<-chan OrdenProcesada, <-chan error) {
    salida := make(chan OrdenProcesada, 100)
    errores := make(chan error)  // unbuffered — puede bloquear

    go func() {
        for orden := range entrada {
            resultado, err := procesar(orden)
            if err != nil {
                errores <- err  // puede bloquear si nadie lee errores
            } else {
                salida <- resultado
            }
        }
        close(salida)
        close(errores)
    }()

    return salida, errores
}

func main() {
    entrada := make(chan Orden)
    go alimentar(entrada)
    // Bug: alimentar no cierra entrada

    etapa1out := etapa1(entrada)
    etapa2out, errores := etapa2(etapa1out)

    // El caller ignora errores:
    for orden := range etapa2out {
        guardar(orden)
    }
}
```

**Tu review:** Hay tres problemas de diseño de canales. Identifícalos todos
y propón una versión rediseñada con ownership claro.

**Pista para el reviewer:** (1) `alimentar` no cierra `entrada` — el pipeline
nunca termina. (2) `errores` en `etapa2` es unbuffered — bloquea si nadie lee.
(3) El caller ignora el canal de errores — goroutine leak si hay errores.
Diseño alternativo: usar `errgroup` de `golang.org/x/sync` que maneja la
propagación de errores y la cancelación del pipeline completo.

---

### Ejercicio 20.3.2 — El canal que crece sin límite

Lee este PR. ¿Qué pasa bajo carga sostenida?

```go
// PR: sistema de notificaciones asíncronas

type Notificador struct {
    cola chan Notificacion
}

func NuevoNotificador() *Notificador {
    n := &Notificador{
        cola: make(chan Notificacion, 10000),  // buffer grande "para seguridad"
    }
    go n.procesar()
    return n
}

func (n *Notificador) Enviar(notif Notificacion) {
    select {
    case n.cola <- notif:
    default:
        log.Warn("cola de notificaciones llena, descartando")
        metricas.NotificacionDescartada.Inc()
    }
}

func (n *Notificador) procesar() {
    for notif := range n.cola {
        enviarEmail(notif)  // puede tardar 500ms por email
    }
}
```

**Tu review:** El diseño es mejor que uno sin límite (tiene `default` para no bloquear).
Pero hay un problema de capacidad. Calcula:
- ¿Cuántas notificaciones se acumulan en 1 minuto si llegan a 100/s pero solo se procesan a 2/s?
- ¿Cuánto tiempo tarda en vaciarse la cola de 10,000 si deja de llegar nueva carga?
- ¿Qué cambio de arquitectura (no solo de tamaño del buffer) resolvería el problema raíz?

**Pista para el reviewer:** Con 100/s de entrada y 2/s de procesamiento, el backlog
crece a 98/s → en 1 minuto: 5,880 notificaciones en cola (llenando el buffer de 10,000
en ~102 segundos). El buffer grande esconde el problema real: el procesador es demasiado lento.
Las soluciones arquitectónicas: (1) más workers de procesamiento paralelo, (2) batching de emails,
(3) delegar a un servicio externo de envío con mayor throughput.

---

### Ejercicio 20.3.3 — select mal usado

Lee este PR. El `select` tiene un comportamiento inesperado:

```go
// PR: leer mensajes de dos fuentes con prioridad

func LeerConPrioridad(alta, baja <-chan Mensaje) Mensaje {
    for {
        // Intentar la fuente de alta prioridad primero:
        select {
        case msg := <-alta:
            return msg
        default:
        }

        // Si no hay mensaje de alta, leer de cualquiera:
        select {
        case msg := <-alta:
            return msg
        case msg := <-baja:
            return msg
        }
    }
}
```

**Tu review:** La intención es correcta (priorizar `alta`). Pero el código
tiene un problema de fairness y un potencial de starvation. Identifícalos.

**Pista para el reviewer:** El primer `select` con `default` verifica si hay
un mensaje en `alta` sin bloquear. Si no hay, el segundo `select` elige
aleatoriamente entre `alta` y `baja` (cuando ambas tienen mensajes).
Problema 1: si `alta` recibe mensajes frecuentes pero `baja` también,
`baja` puede ser seleccionada en el segundo `select` incluso cuando `alta`
tiene mensajes (porque el segundo `select` es aleatorio). Esto viola la semántica de prioridad.
Problema 2: si `alta` siempre tiene mensajes, `baja` nunca se procesa (starvation).
No hay corrección perfecta en Go — la prioridad estricta vs starvation es un tradeoff.

---

### Ejercicio 20.3.4 — El canal como semáforo

Lee este PR. ¿Es idiomático? ¿Tiene problemas?

```go
// PR: limitar la concurrencia de requests a una API externa

const maxConcurrentes = 10

var semaforo = make(chan struct{}, maxConcurrentes)

func LlamarAPI(ctx context.Context, req Request) (Response, error) {
    semaforo <- struct{}{}  // adquirir
    defer func() { <-semaforo }()  // liberar

    return apiClient.Do(ctx, req)
}
```

**Tu review:** El patrón de usar un canal como semáforo es válido en Go.
Pero hay un problema con el contexto y tres mejoras opcionales. Identifícalos.

**Pista para el reviewer:** Problema principal: `semaforo <- struct{}{}` bloquea
sin respetar `ctx`. Si `ctx` se cancela mientras la goroutine espera el semáforo,
la goroutine queda bloqueada hasta que haya espacio (ignorando la cancelación).
La corrección:
```go
select {
case semaforo <- struct{}{}:
case <-ctx.Done():
    return Response{}, ctx.Err()
}
```
Mejoras opcionales: (1) usar `golang.org/x/sync/semaphore` que tiene mejor API,
(2) añadir métricas de cuántas goroutines están esperando el semáforo,
(3) considerar si la variable global `semaforo` es la abstracción correcta.

---

### Ejercicio 20.3.5 — Rediseñar: de canales a errgroup

Lee este PR. El autor implementó manualmente lo que `errgroup` hace mejor:

```go
// PR: procesar N archivos en paralelo y reportar errores

func ProcesarArchivos(archivos []string) error {
    errCh := make(chan error, len(archivos))
    var wg sync.WaitGroup

    for _, archivo := range archivos {
        wg.Add(1)
        go func(a string) {
            defer wg.Done()
            if err := procesar(a); err != nil {
                errCh <- fmt.Errorf("error en %s: %w", a, err)
            }
        }(archivo)
    }

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }

    if len(errs) > 0 {
        return errors.Join(errs...)
    }
    return nil
}
```

**Tu review:** El código funciona. Escribe el review que sugiere usar `errgroup`
en su lugar, mostrando el código equivalente y explicando qué se gana y qué se pierde.

**Pista para el reviewer:** Con `errgroup`:
```go
func ProcesarArchivos(archivos []string) error {
    g, ctx := errgroup.WithContext(context.Background())
    for _, archivo := range archivos {
        a := archivo  // captura (pre-Go 1.22)
        g.Go(func() error {
            return procesar(ctx, a)
        })
    }
    return g.Wait()
}
```
Diferencias: `errgroup` retorna solo el primer error (no todos). Si necesitas
todos los errores, el código manual es necesario. `errgroup` también permite
limitar la concurrencia con `SetLimit(n)`.

---

## Sección 20.4 — Antipatrones de Sincronización

### Ejercicio 20.4.1 — El mutex que protege muy poco

Lee este PR. ¿El mutex está usado correctamente?

```go
// PR: caché thread-safe con invalidación

type Cache struct {
    mu   sync.Mutex
    data map[string]Entry
}

type Entry struct {
    valor    interface{}
    versión  int
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    entrada, ok := c.data[key]
    c.mu.Unlock()
    // ← lock liberado aquí

    if !ok {
        return nil, false
    }

    // Usar entrada.versión sin lock:
    if entrada.versión < obtenerVersionActual() {
        return nil, false  // entrada obsoleta
    }

    return entrada.valor, true
}

func (c *Cache) Set(key string, valor interface{}, versión int) {
    c.mu.Lock()
    c.data[key] = Entry{valor: valor, versión: versión}
    c.mu.Unlock()
}
```

**Tu review:** El mutex protege el acceso al mapa, pero hay un window
entre liberar el lock y usar los datos. ¿Es esto un problema real o teórico?
¿Cuándo importa y cuándo no?

**Pista para el reviewer:** La secuencia: Lock → leer entrada del mapa → Unlock →
verificar entrada.versión. En el momento de verificar, `Set` puede haber
actualizado la entrada con una versión nueva. La goroutine que llama `Get`
verifica la versión de la entrada *antigua* (ya leída y fuera del lock).
Esto puede causar un falso "obsoleto" (retornar `nil, false` cuando hay
un valor válido más nuevo). No causa datos corruptos, pero sí inconsistencia.
Si la semántica deseada es "siempre el valor más reciente", necesita retener el lock
durante la verificación de versión. Si "puede devolver null transitoriamente cuando hay
actualización" es aceptable, el código está bien.

---

### Ejercicio 20.4.2 — El mutex dentro del loop

Lee este PR. Hay un problema de rendimiento y uno de correctitud:

```go
// PR: agregar elementos a una lista thread-safe

type Lista struct {
    mu       sync.Mutex
    elementos []int
}

func (l *Lista) AgregarLote(items []int) {
    for _, item := range items {
        l.mu.Lock()
        l.elementos = append(l.elementos, item)
        l.mu.Unlock()
    }
}

func (l *Lista) Longitud() int {
    l.mu.Lock()
    defer l.mu.Unlock()
    return len(l.elementos)
}
```

**Tu review:** Identifica el problema de rendimiento (claro) y el problema
de correctitud (sutil). Propón la corrección para ambos.

**Pista para el reviewer:** Rendimiento: adquirir y liberar el mutex por cada elemento
es muy costoso. Mejor adquirirlo una vez para todo el lote. Correctitud:
si otro goroutine lee `Longitud()` mientras `AgregarLote` está ejecutando,
verá la lista parcialmente construida — lo cual puede ser correcto o incorrecto
dependiendo de si la atomicidad del lote es un requisito de negocio.
Si el lote debe aparecer completo o no aparecer, necesitas adquirir el lock
una sola vez para todo el lote.

---

### Ejercicio 20.4.3 — sync.Map usado incorrectamente

Lee este PR. El uso de `sync.Map` no es idiomático:

```go
// PR: caché de configuración con sync.Map

var config sync.Map

func ObtenerConfig(clave string) (string, bool) {
    v, ok := config.Load(clave)
    if !ok {
        return "", false
    }
    return v.(string), true
}

func ActualizarConfig(clave, valor string) {
    // Patrón check-then-act sin atomicidad:
    if _, existe := config.Load(clave); existe {
        config.Store(clave, valor)
    }
}

func IncrementarContador(clave string) {
    v, _ := config.Load(clave)
    contador, _ := strconv.Atoi(v.(string))
    contador++
    config.Store(clave, strconv.Itoa(contador))
}
```

**Tu review:** Hay dos usos incorrectos de `sync.Map`. Identifícalos y
explica por qué `sync.Map` no es la herramienta correcta para `IncrementarContador`.

**Pista para el reviewer:** Bug 1: `ActualizarConfig` hace check-then-act sin atomicidad.
Entre `Load` y `Store`, otro goroutine puede borrar la clave o actualizar el valor.
Para esto: `config.CompareAndSwap` o `config.LoadOrStore`.
Bug 2: `IncrementarContador` es read-modify-write no-atómico: dos goroutines pueden
leer el mismo contador, ambas incrementarlo, y una sobreescribir el resultado de la otra.
Para un contador concurrente, usar `atomic.Int64`, no `sync.Map` con strings.
`sync.Map` es apropiado cuando: hay muchas lecturas, pocas escrituras, y las keys
no cambian mucho. Para contadores de alta frecuencia: `atomic`.

---

### Ejercicio 20.4.4 — El lock que protege demasiado

Lee este PR. El autor es cauteloso — quizás demasiado:

```go
// PR: servidor que procesa requests con logging

type Servidor struct {
    mu      sync.Mutex
    handler http.Handler
    logger  *log.Logger
    métricas map[string]int64
}

func (s *Servidor) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.logger.Printf("request: %s %s", r.Method, r.URL.Path)
    s.métricas[r.URL.Path]++
    s.handler.ServeHTTP(w, r)  // ← procesar el request con el lock adquirido
}
```

**Tu review:** La sección crítica es demasiado grande. Identifica:
1. Qué operaciones necesitan el lock y cuáles no
2. El problema de rendimiento específico
3. El problema de correctitud potencial
4. La versión rediseñada

**Pista para el reviewer:** `s.handler.ServeHTTP(w, r)` dentro del lock serializa
todos los requests — el servidor procesa uno a la vez aunque HTTP sea inherentemente
concurrente. El lock también bloquea el logging, que puede hacer una syscall (lenta).
Rediseño: el lock solo debe proteger `s.métricas` (y brevemente). El handler y el
logger son thread-safe por diseño y no necesitan protección adicional.
`*log.Logger` de la stdlib es thread-safe. El `http.Handler` también debe serlo.

---

### Ejercicio 20.4.5 — Condición de carrera en el shutdown

Lee este PR. El shutdown tiene un race:

```go
// PR: servidor con shutdown graceful

type Servidor struct {
    cerrado bool
    mu      sync.Mutex
    wg      sync.WaitGroup
}

func (s *Servidor) Manejar(req Request) {
    s.mu.Lock()
    if s.cerrado {
        s.mu.Unlock()
        return
    }
    s.wg.Add(1)
    s.mu.Unlock()

    defer s.wg.Done()
    procesarRequest(req)
}

func (s *Servidor) Cerrar() {
    s.mu.Lock()
    s.cerrado = true
    s.mu.Unlock()

    s.wg.Wait()  // esperar requests en vuelo
}
```

**Tu review:** El código parece correcto — usa el mutex para proteger `cerrado`
y el WaitGroup para esperar los requests en vuelo. ¿Hay un race?

**Pista para el reviewer:** El código es **correcto** — este ejercicio es un
"falso positivo" en el code review. El patrón lock-check-Add-unlock es la
forma correcta de coordinar entre nuevas llegadas y el shutdown. La ventana
entre `s.mu.Lock()` y `s.wg.Add(1)` está protegida por el mutex, así que no hay race.
El objetivo del ejercicio: entrenar al reviewer a distinguir código que parece
sospechoso pero es correcto de código que realmente tiene un bug.
Comentario apropiado: "Puedo confirmar que este patrón es correcto, pero
considera añadir un comentario explicando por qué `wg.Add(1)` debe estar
dentro del lock para que futuros mantenedores no lo muevan."

---

## Sección 20.5 — Context y Cancelación

### Ejercicio 20.5.1 — Context ignorado

Lee este PR. ¿El context se propaga correctamente?

```go
// PR: cliente HTTP con retry y context

func (c *Cliente) Get(ctx context.Context, url string) ([]byte, error) {
    for intento := 0; intento < 3; intento++ {
        resp, err := http.Get(url)  // ← context ignorado
        if err == nil {
            defer resp.Body.Close()
            return io.ReadAll(resp.Body)
        }

        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(time.Duration(intento+1) * time.Second):
        }
    }
    return nil, fmt.Errorf("agotados los intentos")
}
```

**Tu review:** El context se verifica en el backoff pero no en el request HTTP.
Escribe el review completo incluyendo el impacto de este bug.

**Pista para el reviewer:** `http.Get(url)` no recibe el context — si el context
se cancela *durante* el request HTTP (que puede tardar segundos), el request
continúa hasta completar o hasta que el TCP timeout lo corte. La corrección:
```go
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
if err != nil { return nil, err }
resp, err := c.httpClient.Do(req)
```
Nota adicional: `http.Get` usa el cliente HTTP global del paquete — el PR debería
usar un `*http.Client` configurado explícitamente, no la función global.

---

### Ejercicio 20.5.2 — Context creado en el lugar equivocado

Lee este PR. El context tiene un lifetime incorrecto:

```go
// PR: sistema de jobs con timeout por job

type JobRunner struct {
    timeout time.Duration
}

func (jr *JobRunner) EjecutarTodos(jobs []Job) []error {
    ctx, cancel := context.WithTimeout(context.Background(), jr.timeout)
    defer cancel()  // ← un timeout para todos los jobs juntos

    var errores []error
    for _, job := range jobs {
        if err := jr.ejecutar(ctx, job); err != nil {
            errores = append(errores, err)
        }
    }
    return errores
}

func (jr *JobRunner) ejecutar(ctx context.Context, job Job) error {
    return job.Run(ctx)
}
```

**Tu review:** El timeout aplica a todos los jobs juntos, no a cada job
individualmente. ¿Es esto lo correcto? ¿Cuándo cada interpretación tiene sentido?
Propón la versión con timeout por job.

**Pista para el reviewer:** Timeout global: "todos los jobs juntos deben completar en N segundos".
Si el primer job tarda mucho, los siguientes tienen menos tiempo.
Timeout por job: "cada job individualmente tiene N segundos".
Para trabajo en batch donde cada job es independiente, el timeout por job es más justo.
Corrección:
```go
for _, job := range jobs {
    jobCtx, jobCancel := context.WithTimeout(context.Background(), jr.timeout)
    err := jr.ejecutar(jobCtx, job)
    jobCancel()
    if err != nil { errores = append(errores, err) }
}
```

---

### Ejercicio 20.5.3 — Context value usado como API

Lee este PR. El uso de `context.Value` es un antipatrón aquí:

```go
// PR: sistema de autenticación usando context

type ctxKey string
const userKey ctxKey = "user"

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user, err := autenticar(r)
        if err != nil {
            http.Error(w, "no autorizado", 401)
            return
        }
        ctx := context.WithValue(r.Context(), userKey, user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func ObtenerUsuario(ctx context.Context) *Usuario {
    user, _ := ctx.Value(userKey).(*Usuario)
    return user  // puede ser nil si no hay usuario en el context
}

// Uso en el handler:
func handlePerfil(w http.ResponseWriter, r *http.Request) {
    user := ObtenerUsuario(r.Context())
    if user == nil {
        http.Error(w, "no autenticado", 401)
        return
    }
    renderPerfil(w, user)
}
```

**Tu review:** Tecnicamente funciona. Pero `context.Value` se documenta como
"para datos que cruzan límites de API". ¿Es el usuario autenticado un dato
de ese tipo? ¿Cuáles son los tradeoffs de este diseño vs pasar el usuario
explícitamente?

**Pista para el reviewer:** Este es un uso controversial pero aceptado de `context.Value`
en Go — el middleware de autenticación es exactamente el caso de uso donde
`context.Value` está diseñado (datos que cruzan capas de un request).
El problema real: si `ObtenerUsuario` retorna nil y el handler no verifica,
hay un panic en runtime. El code review debería preguntar si los handlers
*siempre* verifican nil y si hay un mecanismo que lo garantice (interface typed,
panic en ObtenerUsuario si no hay usuario, etc.).

---

### Ejercicio 20.5.4 — El context que se cancela demasiado pronto

Lee este PR. El context tiene un lifetime más corto de lo esperado:

```go
// PR: guardar resultado en DB de forma asíncrona

func (h *Handler) ProcesarPedido(w http.ResponseWriter, r *http.Request) {
    pedido, err := parsearPedido(r)
    if err != nil {
        http.Error(w, err.Error(), 400)
        return
    }

    // Responder inmediatamente:
    w.WriteHeader(202)
    json.NewEncoder(w).Encode(map[string]string{"status": "aceptado"})

    // Guardar en DB de forma asíncrona:
    go func() {
        if err := h.db.GuardarPedido(r.Context(), pedido); err != nil {
            log.Error("error guardando pedido", err)
        }
    }()
}
```

**Tu review:** El context de la request HTTP se cancela cuando el handler retorna.
Identifica cuándo esto causa el bug y propón la corrección.

**Pista para el reviewer:** Cuando `ProcesarPedido` retorna (después de escribir el 202),
el framework HTTP cancela `r.Context()`. La goroutine asíncrona intenta
`h.db.GuardarPedido(r.Context(), ...)` con un contexto ya cancelado — el driver de DB
puede rechazar la operación inmediatamente. La corrección: no usar `r.Context()` en
la goroutine asíncrona. Usar `context.Background()` o un contexto con timeout propio:
```go
go func() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    h.db.GuardarPedido(ctx, pedido)
}()
```

---

### Ejercicio 20.5.5 — Propagación de cancelación en tests

Lee este PR. Los tests no validan el comportamiento de cancelación:

```go
// Tests de la función FanOut del Ejercicio 20.2.2

func TestFanOut_Timeout(t *testing.T) {
    urls := []string{"http://slow.example.com"} // siempre tarda 10s

    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    inicio := time.Now()
    _, err := FanOut(ctx, urls)
    duracion := time.Since(inicio)

    assert.ErrorIs(t, err, context.DeadlineExceeded)
    assert.Less(t, duracion, 500*time.Millisecond)
    // Pero no verifica que no hay goroutine leaks
}
```

**Tu review:** El test verifica el timeout pero no verifica si hay goroutine leaks
después de la cancelación. Añade la verificación necesaria y explica por qué
es importante incluirla.

**Pista para el reviewer:** Añadir `defer goleak.VerifyNone(t)` al principio del test.
Después del timeout, el test debe verificar que todas las goroutines lanzadas por
`FanOut` terminaron. Si las goroutines usan `http.Get` sin contexto (el bug del
Ejercicio 20.2.2), `goleak.VerifyNone` fallará porque las goroutines siguen vivas
esperando que el request HTTP complete. Este test fallaría con la implementación buggy
y pasaría con la corrección.

---

## Sección 20.6 — Revisión de Diseño: ¿Es Más Complejo de lo Necesario?

### Ejercicio 20.6.1 — Canales donde un mutex es más simple

Lee este PR. El autor usó canales para proteger estado compartido:

```go
// PR: contador thread-safe usando canales

type ContadorCanal struct {
    operaciones chan func(int) int
    valor       int
}

func NuevoContador() *ContadorCanal {
    c := &ContadorCanal{
        operaciones: make(chan func(int) int),
    }
    go func() {
        for op := range c.operaciones {
            c.valor = op(c.valor)
        }
    }()
    return c
}

func (c *ContadorCanal) Incrementar() {
    c.operaciones <- func(v int) int { return v + 1 }
}

func (c *ContadorCanal) Valor() int {
    resultado := make(chan int, 1)
    c.operaciones <- func(v int) int {
        resultado <- v
        return v
    }
    return <-resultado
}
```

**Tu review:** El diseño funciona pero es más complejo de lo necesario para este caso.
Escribe el review que explica cuándo el actor pattern (canal + goroutine dedicada)
es apropiado y cuándo un mutex es más simple y eficiente.

**Pista para el reviewer:** El actor pattern con canales es apropiado cuando:
(1) el estado tiene operaciones complejas que no pueden expresarse con atomics,
(2) hay operaciones que necesitan observar el estado y luego actuar basándose en él,
(3) el componente necesita manejar timeouts o cancelación en sus operaciones internas.
Para un contador simple, `atomic.Int64` es la opción más simple y más eficiente.
El canal añade goroutinas, allocations, y overhead de scheduling sin beneficio real.

---

### Ejercicio 20.6.2 — El doble de goroutines sin beneficio

Lee este PR. ¿Cuántas goroutines son realmente necesarias?

```go
// PR: procesador de imágenes con pipeline de 4 etapas

func ProcesarImagenes(rutas []string) []ImagenProcesada {
    // Etapa 1: leer archivos
    lecturas := make(chan []byte, 10)
    go func() {
        for _, ruta := range rutas {
            data, _ := os.ReadFile(ruta)
            lecturas <- data
        }
        close(lecturas)
    }()

    // Etapa 2: decodificar
    imagenes := make(chan image.Image, 10)
    go func() {
        for data := range lecturas {
            img, _ := png.Decode(bytes.NewReader(data))
            imagenes <- img
        }
        close(imagenes)
    }()

    // Etapa 3: procesar (wrapping de otra goroutine)
    procesadas := make(chan image.Image, 10)
    go func() {
        for img := range imagenes {
            go func(i image.Image) {  // ← goroutine por imagen (sin límite)
                procesadas <- aplicarFiltros(i)
            }(img)
        }
        close(procesadas)  // BUG: cierra antes de que las goroutines terminen
    }()

    // Etapa 4: recolectar
    var resultados []ImagenProcesada
    for img := range procesadas {
        resultados = append(resultados, ImagenProcesada{img})
    }
    return resultados
}
```

**Tu review:** Hay un bug crítico y dos problemas de diseño. Identifícalos todos.

**Pista para el reviewer:** Bug: `close(procesadas)` en la etapa 3 cierra el canal
*después* de lanzar todas las goroutines pero *antes* de que terminen.
Las goroutines que intentan enviar a `procesadas` después del cierre causan un panic.
Diseño 1: la etapa 3 lanza goroutines sin límite — con 10,000 imágenes, 10,000 goroutines.
Diseño 2: la etapa de decodificación es secuencial — cuello de botella innecesario.
Rediseño con `errgroup` y límite de concurrencia: `g.SetLimit(runtime.NumCPU())`.

---

### Ejercicio 20.6.3 — Simplificar con errgroup

Lee este PR y el PR alternativo. ¿Cuál preferirías?

```go
// Versión A (PR original): implementación manual

func DescargarTodos(urls []string) ([][]byte, error) {
    resultados := make([][]byte, len(urls))
    errores := make(chan error, len(urls))
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(idx int, u string) {
            defer wg.Done()
            data, err := descargar(u)
            if err != nil {
                errores <- err
                return
            }
            resultados[idx] = data
        }(i, url)
    }

    wg.Wait()
    close(errores)

    if err := <-errores; err != nil {
        return nil, err
    }
    return resultados, nil
}

// Versión B (propuesta del reviewer): con errgroup

func DescargarTodos(urls []string) ([][]byte, error) {
    resultados := make([][]byte, len(urls))
    g, ctx := errgroup.WithContext(context.Background())

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            data, err := descargarConContext(ctx, url)
            if err != nil { return err }
            resultados[i] = data
            return nil
        })
    }

    return resultados, g.Wait()
}
```

**Tu review:** Compara las dos versiones en: correctitud, manejo de errores,
cancelación, legibilidad, y facilidad de mantenimiento. ¿Hay casos donde
la Versión A es preferible?

**Pista para el reviewer:** La Versión A tiene un bug: si múltiples goroutines
envían a `errores` (canal buffered de len(urls)), solo el primer error se retorna
(`<-errores` lee uno). Esto está bien si "el primer error" es suficiente,
pero el canal buffered sugiere que el autor quería capturar todos — pero el código
no lo hace. La Versión B es más correcta y más simple. El único caso donde A es
preferible: necesitas *todos* los errores, no solo el primero.

---

### Ejercicio 20.6.4 — ¿Necesitas una goroutine aquí?

Lee este PR. El autor lanzó goroutines innecesariamente:

```go
// PR: procesar configuración al inicio del servidor

func CargarConfiguracion() (*Config, error) {
    // Cargar en paralelo porque "puede ser más rápido":
    var (
        dbURL  string
        apiKey string
        mu     sync.Mutex
        errs   []error
        wg     sync.WaitGroup
    )

    wg.Add(1)
    go func() {
        defer wg.Done()
        url, err := leerEnvVar("DATABASE_URL")
        mu.Lock()
        if err != nil { errs = append(errs, err) } else { dbURL = url }
        mu.Unlock()
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        key, err := leerEnvVar("API_KEY")
        mu.Lock()
        if err != nil { errs = append(errs, err) } else { apiKey = key }
        mu.Unlock()
    }()

    wg.Wait()
    if len(errs) > 0 { return nil, errs[0] }
    return &Config{DBURL: dbURL, APIKey: apiKey}, nil
}
```

**Tu review:** `leerEnvVar` es una función que lee variables de entorno —
una operación de nanosegundos. El overhead del paralelismo es mayor que el beneficio.
Escribe el review explicando cuándo el paralelismo tiene sentido y cuándo no.

**Pista para el reviewer:** `os.Getenv()` es una llamada a una variable del proceso
en memoria — tarda ~50 nanosegundos. Lanzar dos goroutines con WaitGroup tarda
~2000-5000 nanosegundos. El "paralelismo" hace el código 40-100x más lento
para esta tarea específica. Regla: paralelizar cuando cada tarea tarda > 1ms
y son independientes. Para operaciones de nanosegundos, el overhead de goroutines
y sincronización supera siempre el beneficio.

---

### Ejercicio 20.6.5 — El diseño que escala vs el que no

Lee este PR. ¿Qué pasa cuando hay 10,000 clientes?

```go
// PR: sistema de notificaciones push con una goroutine por cliente

type NotificadorPush struct {
    clientes map[string]chan Notificacion
    mu       sync.RWMutex
}

func (n *NotificadorPush) Conectar(clienteID string) <-chan Notificacion {
    ch := make(chan Notificacion, 10)
    n.mu.Lock()
    n.clientes[clienteID] = ch
    n.mu.Unlock()

    go func() {  // ← una goroutine por cliente para el heartbeat
        ticker := time.NewTicker(30 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            select {
            case ch <- Notificacion{Tipo: "heartbeat"}:
            default:
                // cliente lento, descartar heartbeat
            }
        }
    }()

    return ch
}
```

**Tu review:** Con 10,000 clientes hay 10,000 goroutines de heartbeat, cada una
con su ticker. Calcula el overhead y propón un diseño que escala a 100,000 clientes.

**Pista para el reviewer:** Cada `time.Ticker` tiene una goroutine interna del runtime
y un canal. Con 10,000 clientes: 10,000 goroutines de heartbeat + 10,000 tickers internos.
Overhead de memoria: ~10,000 × (2KB goroutine stack + timer overhead) ≈ ~20MB solo en stacks.
Diseño alternativo: una sola goroutine de heartbeat que itera todos los clientes cada 30s.
Con un ticker global, el overhead es constante independientemente del número de clientes.

---

## Sección 20.7 — PR Completo: Revisión de Extremo a Extremo

### Ejercicio 20.7.1 — PR: sistema de caché distribuido simplificado

Haz el code review completo de este PR (~80 líneas). Escribe cada comentario
como lo harías en GitHub: indica la línea, el tipo de comentario (bug/style/question),
y la sugerencia concreta.

```go
// PR: caché en memoria con TTL, evicción LRU, y métricas
// Jira: PERF-1234 — reducir latencia de BD en un 40%

package cache

import (
    "sync"
    "time"
)

type entrada struct {
    valor    interface{}
    expira   time.Time
    accesos  int
    ultimo   time.Time
}

type Cache struct {
    mu       sync.Mutex
    datos    map[string]*entrada
    max      int
    metricas struct {
        hits   int64
        misses int64
    }
}

func Nuevo(max int) *Cache {
    c := &Cache{
        datos: make(map[string]*entrada),
        max:   max,
    }
    go c.limpiarLoop()
    return c
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    e, ok := c.datos[key]
    if !ok {
        c.metricas.misses++
        return nil, false
    }

    if time.Now().After(e.expira) {
        delete(c.datos, key)
        c.metricas.misses++
        return nil, false
    }

    e.accesos++
    e.ultimo = time.Now()
    c.metricas.hits++
    return e.valor, true
}

func (c *Cache) Set(key string, valor interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if len(c.datos) >= c.max {
        c.evictar()
    }

    c.datos[key] = &entrada{
        valor:  valor,
        expira: time.Now().Add(ttl),
        ultimo: time.Now(),
    }
}

func (c *Cache) evictar() {
    // Evictar el elemento con acceso más antiguo (LRU simplificado):
    var keyMasAntigua string
    var ultimoMasAntiguo time.Time

    for key, e := range c.datos {
        if ultimoMasAntiguo.IsZero() || e.ultimo.Before(ultimoMasAntiguo) {
            keyMasAntigua = key
            ultimoMasAntiguo = e.ultimo
        }
    }

    if keyMasAntigua != "" {
        delete(c.datos, keyMasAntigua)
    }
}

func (c *Cache) limpiarLoop() {
    for {
        time.Sleep(time.Minute)
        c.mu.Lock()
        ahora := time.Now()
        for key, e := range c.datos {
            if ahora.After(e.expira) {
                delete(c.datos, key)
            }
        }
        c.mu.Unlock()
    }
}

func (c *Cache) Stats() (hits, misses int64) {
    return c.metricas.hits, c.metricas.misses
}
```

**Tu review:** Escribe el review completo. Hay al menos 5 issues de distintos niveles
de severidad (blocker, major, minor, style). Para cada uno: línea, severidad, explicación, y código corregido.

**Pista para el reviewer:** Issues en orden de severidad:
1. **Blocker**: `limpiarLoop` no tiene mecanismo de terminación — goroutine leak.
2. **Blocker**: `Stats()` lee `c.metricas.hits/misses` sin lock — data race.
3. **Major**: evicción O(n) — con max=10,000 cada Set que requiere evicción itera todo el mapa.
4. **Major**: `limpiarLoop` adquiere el lock por el tiempo que tarda iterar todos los datos — bloquea Gets durante la limpieza.
5. **Minor**: `time.Now()` se llama múltiples veces en `Set` — puede ser inconsistente.
6. **Style**: `metricas` sin exportar con campos sin exportar — difícil de extender.

---

### Ejercicio 20.7.2 — PR: worker pool con prioridades

Haz el code review completo de este PR. Evalúa correctitud, liveness, y diseño:

```go
// PR: worker pool con tres niveles de prioridad
// Los jobs de alta prioridad siempre se procesan antes que los de baja

type Prioridad int
const (
    Alta   Prioridad = 3
    Media  Prioridad = 2
    Baja   Prioridad = 1
)

type Job struct {
    Fn        func()
    Prioridad Prioridad
}

type PriorityPool struct {
    alta  chan Job
    media chan Job
    baja  chan Job
    wg    sync.WaitGroup
}

func NuevoPriorityPool(workers int) *PriorityPool {
    p := &PriorityPool{
        alta:  make(chan Job, 100),
        media: make(chan Job, 100),
        baja:  make(chan Job, 100),
    }
    for i := 0; i < workers; i++ {
        p.wg.Add(1)
        go func() {
            defer p.wg.Done()
            for {
                // Prioridad estricta con select anidado:
                select {
                case job := <-p.alta:
                    job.Fn()
                default:
                    select {
                    case job := <-p.alta:
                        job.Fn()
                    case job := <-p.media:
                        job.Fn()
                    default:
                        select {
                        case job := <-p.alta:
                            job.Fn()
                        case job := <-p.media:
                            job.Fn()
                        case job := <-p.baja:
                            job.Fn()
                        }
                    }
                }
            }
        }()
    }
    return p
}

func (p *PriorityPool) Submit(job Job) {
    switch job.Prioridad {
    case Alta:
        p.alta <- job
    case Media:
        p.media <- job
    default:
        p.baja <- job
    }
}

func (p *PriorityPool) Shutdown() {
    close(p.alta)
    close(p.media)
    close(p.baja)
    p.wg.Wait()
}
```

**Tu review:** Hay issues de liveness, correctitud en shutdown, y un problema
de diseño en la implementación de prioridad. Identifícalos todos.

**Pista para el reviewer:** Liveness: el select anidado tiene busy-waiting — cuando
todos los canales están vacíos, el tercer `select` bloquea correctamente, pero
los dos primeros `default` hacen que el loop gire sin pausa cuando no hay trabajo.
Correctitud en shutdown: `close` de los tres canales puede causar panics si hay
goroutines enviando concurrentemente. Diseño: leer de un canal cerrado retorna
el zero value — `job.Fn` puede ser nil. Los workers deben verificar si el canal
fue cerrado.

---

### Ejercicio 20.7.3 — PR: rate limiter distribuido

Haz el code review de este PR. Enfócate en correctitud bajo condiciones de red adversas:

```go
// PR: rate limiter con estado en Redis para múltiples instancias del servicio

type RateLimiterRedis struct {
    redis  *redis.Client
    limite int
    window time.Duration
}

func (rl *RateLimiterRedis) Permitir(ctx context.Context, clienteID string) (bool, error) {
    key := fmt.Sprintf("rl:%s", clienteID)

    // Usar un Lua script para atomicidad:
    script := redis.NewScript(`
        local current = redis.call('INCR', KEYS[1])
        if current == 1 then
            redis.call('EXPIRE', KEYS[1], ARGV[1])
        end
        return current
    `)

    result, err := script.Run(ctx, rl.redis, []string{key},
        int(rl.window.Seconds())).Int()
    if err != nil {
        // Si Redis falla, ¿qué hacemos?
        return true, err  // fail open: permitir el request
    }

    return result <= rl.limite, nil
}
```

**Tu review:** El Lua script garantiza atomicidad en Redis. Pero hay un bug
en la lógica del script y un dilema de diseño en el error handling. Identifícalos.

**Pista para el reviewer:** Bug en el Lua script: si la key ya existe (creada por otro
proceso), `INCR` funciona correctamente. Pero si la key expira entre el `INCR` (que la
recrea con valor 1) y el `EXPIRE` — esto no puede pasar porque son parte del mismo
script Lua (atómico en Redis). El script es correcto. El bug real: si Redis tiene
alta latencia (~500ms), el rate limiter añade 500ms de latencia a cada request.
Necesita un timeout: `ctx, cancel := context.WithTimeout(ctx, 10*time.Millisecond)`.
Dilema de diseño: "fail open" (permitir cuando Redis falla) vs "fail closed" (rechazar).
Ambas son decisiones de negocio válidas pero deben documentarse explícitamente.

---

### Ejercicio 20.7.4 — PR: migración de sistema legacy

Haz el code review de este PR de migración. El riesgo principal es la transición:

```go
// PR: migrar de un lock global a sharded locks para mejorar throughput
// El sistema tiene 50,000 usuarios activos simultáneos

// ANTES:
type StoreLegacy struct {
    mu    sync.RWMutex
    datos map[string]Session
}

// DESPUÉS:
const numShards = 256

type StoreShard struct {
    mu    sync.RWMutex
    datos map[string]Session
}

type StoreSharded struct {
    shards [numShards]StoreShard
}

func (s *StoreSharded) shard(key string) *StoreShard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return &s.shards[h.Sum32()%numShards]
}

func (s *StoreSharded) Get(key string) (Session, bool) {
    shard := s.shard(key)
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    session, ok := shard.datos[key]
    return session, ok
}

func (s *StoreSharded) Set(key string, session Session) {
    shard := s.shard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    shard.datos[key] = session
}

// La migración se hace reemplazando StoreLegacy con StoreSharded
// en el constructor del servidor
```

**Tu review:** El diseño técnico es correcto. Pero hay consideraciones de migración
que el PR no aborda. Identifica los riesgos de la migración y propón cómo mitigarlos.

**Pista para el reviewer:** El PR no menciona: (1) cómo se inicializa `shard.datos`
(nil map = panic en `Get`). (2) el roll-out strategy — ¿se despliega de golpe o gradualmente?
(3) si hay tests de carga que demuestran el beneficio de rendimiento que justifica
el riesgo de la migración. (4) el plan de rollback si algo falla.
Para code review de cambios de infraestructura, el "¿qué pasa si sale mal?" es
tan importante como el "¿es correcto el código nuevo?".

---

### Ejercicio 20.7.5 — Construir: tu propio checklist

**Tipo: Construir/proceso**

A lo largo de este capítulo revisaste código con distintos tipos de bugs.
Construye tu checklist personal de code review de código concurrente,
basado en los bugs que encontraste:

**Formato sugerido:**

```markdown
# Mi Checklist de Code Review — Código Concurrente

## Bloqueadores (no apruebo si están presentes)
- [ ] [nombre del check] — [por qué es bloqueador] — [referencia: Ejercicio 20.X.Y]

## Problemas mayores (requieren corrección antes de merge)
- [ ] ...

## Mejoras (pueden ser seguimiento)
- [ ] ...

## Preguntas que siempre hago
- ¿Quién cierra este canal?
- ...
```

**Restricciones:** El checklist debe tener:
- Al menos 5 bloqueadores
- Al menos 5 problemas mayores
- Al menos 5 preguntas que haces aunque no hayas visto un bug

**Pista:** Los mejores checklists son los que puedes completar en 10 minutos
revisando un PR mediano. Un checklist de 50 items que nadie usa es peor que
uno de 15 items que se usa consistentemente. Prioriza los bugs que:
(1) son difíciles de detectar sin herramientas, (2) tienen consecuencias graves
en producción, (3) no son obvios para un revisor sin experiencia en concurrencia.

---

## Resumen del capítulo

**Los diez bugs más comunes en code review de código concurrente:**

```
1.  Map compartido sin lock (→ panic en producción, no en tests)
2.  Closure que captura variable de loop por referencia
3.  Canal cerrado múltiples veces (→ panic)
4.  Goroutine sin mecanismo de terminación (→ leak)
5.  context.Context ignorado en operaciones de red
6.  Sección crítica que incluye I/O (→ serialización innecesaria)
7.  check-then-act sin atomicidad (→ race silencioso)
8.  sync.Pool con buffer no reseteado (→ datos de request anterior)
9.  WaitGroup.Add() fuera del goroutine que controla el lifetime
10. Timeout global en lugar de timeout por operación
```

**El heurístico más útil para code review de concurrencia:**

> Por cada goroutine que se lanza, pregunta:
> "¿Cuándo termina esta goroutine y quién lo garantiza?"
>
> Si la respuesta no está clara en el código, es un bug o un riesgo de bug.
> La goroutine debe terminar de forma predecible — por cierre de canal,
> cancelación de contexto, o señal explícita — no por casualidad del scheduler.
