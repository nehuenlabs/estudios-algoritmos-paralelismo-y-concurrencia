# Guía de Ejercicios — Cap.19: Debugging de Sistemas Concurrentes

> Lenguaje principal: **Go**. Las técnicas aplican a cualquier lenguaje.
>
> Este capítulo tiene tres tipos de ejercicios:
> - **Leer**: dado un síntoma, un dump, o un log — ¿qué bug es?
> - **Reproducir**: hacer que un bug no determinista sea reproducible
> - **Corregir**: encontrar la línea exacta del problema y arreglarlo
>
> La habilidad central de este capítulo es diferente a los anteriores:
> no es implementar algo nuevo — es entender código que ya existe y está roto.

---

## Por qué el debugging concurrente es diferente

Debuggear código secuencial es difícil. Debuggear código concurrente es
cualitativamente más difícil por cuatro razones:

```
1. El bug puede desaparecer al observarlo
   Añadir un log o un breakpoint cambia el timing.
   El bug que ocurría 1 vez en 1000 puede pasar a 0 veces en 1000
   — o a 500 veces en 1000, dependiendo del overhead.

2. El bug no es reproducible de forma determinista
   "Se colgó una vez a las 3AM" no es suficiente para reproducirlo en dev.
   El único estado capturado es lo que tenías configurado que capturara.

3. La causa y el síntoma están separados en el tiempo
   Una goroutine escribe datos corruptos ahora.
   Otra goroutine lee esos datos corruptos 2 segundos después.
   El crash ocurre en la goroutine lectora — pero el bug está en la escritora.

4. Los tests no cubren todos los interleavings posibles
   Un test puede pasar 10,000 veces y fallar en producción
   porque el scheduling del OS es diferente bajo carga real.
```

**La estrategia:**

```
Paso 1: Capturar el estado (antes de que desaparezca)
  → goroutine dumps, heap profiles, logs con timestamps

Paso 2: Reproducir (hacer el bug determinista)
  → race detector, stress tests, -count=1000

Paso 3: Aislar (reducir al mínimo que muestra el bug)
  → bisect (git bisect), reducción del caso de prueba

Paso 4: Entender (por qué ocurre, no solo qué ocurre)
  → leer el código con el modelo mental del interleaving

Paso 5: Corregir y verificar
  → la corrección debe hacer el test imposible de fallar, no solo difícil
```

---

## Tabla de contenidos

- [Sección 19.1 — Data races: encontrarlos y entenderlos](#sección-191--data-races-encontrarlos-y-entenderlos)
- [Sección 19.2 — Deadlocks: diagnóstico y resolución](#sección-192--deadlocks-diagnóstico-y-resolución)
- [Sección 19.3 — Goroutine leaks: detectar y corregir](#sección-193--goroutine-leaks-detectar-y-corregir)
- [Sección 19.4 — Heisenbugs: bugs que desaparecen al observarlos](#sección-194--heisenbugs-bugs-que-desaparecen-al-observarlos)
- [Sección 19.5 — Bugs de liveness: starvation y livelock](#sección-195--bugs-de-liveness-starvation-y-livelock)
- [Sección 19.6 — Memory corruption y uso incorrecto de unsafe](#sección-196--memory-corruption-y-uso-incorrecto-de-unsafe)
- [Sección 19.7 — Estrategias sistemáticas de debugging](#sección-197--estrategias-sistemáticas-de-debugging)

---

## Sección 19.1 — Data Races: Encontrarlos y Entenderlos

### Ejercicio 19.1.1 — Leer: ¿qué dice el race detector?

**Tipo: Leer/diagnosticar**

El race detector de Go produce este output al ejecutar con `-race`:

```
==================
WARNING: DATA RACE
Read at 0x00c000126010 by goroutine 7:
  main.(*Cache).Get()
      /app/cache.go:34 +0x6c
  main.handleRequest()
      /app/handler.go:28 +0x84

Previous write at 0x00c000126010 by goroutine 6:
  main.(*Cache).Set()
      /app/cache.go:51 +0x8c
  main.handleRequest()
      /app/handler.go:35 +0xa0

Goroutine 7 (running) created at:
  main.(*Server).Serve()
      /app/server.go:67 +0x118

Goroutine 6 (running) created at:
  main.(*Server).Serve()
      /app/server.go:67 +0x118
==================
```

Y el código es este:

```go
type Cache struct {
    data map[string][]byte
}

func (c *Cache) Get(key string) ([]byte, bool) {
    v, ok := c.data[key]  // línea 34
    return v, ok
}

func (c *Cache) Set(key string, value []byte) {
    c.data[key] = value  // línea 51
}
```

**Preguntas:**

1. ¿Qué operación concreta tiene el race? Describe el interleaving exacto.
2. ¿Por qué leer y escribir un map concurrentemente es un data race en Go
   aunque en otros lenguajes podría "funcionar"?
3. ¿Puede este race causar un crash? ¿Bajo qué condiciones?
4. Propón tres formas de corregirlo con distintos tradeoffs de rendimiento.
5. ¿El race detector siempre detecta este race? ¿Cuándo podría no detectarlo?

**Pista:** En Go, el runtime detecta escrituras concurrentes a un map y lanza
un panic ("concurrent map read and map write"). El race detector lo detecta
*antes* del crash. No siempre lo detecta: solo cuando el interleaving específico
ocurre durante la ejecución del test. Si el test termina antes de que ocurra
el interleaving, el race detector no lo reporta. Las tres correcciones:
`sync.RWMutex`, `sync.Map`, y sharding (múltiples maps con múltiples locks).

---

### Ejercicio 19.1.2 — Reproducir: hacer un race determinista

**Tipo: Reproducir**

Este código tiene un data race que ocurre "de vez en cuando" en producción
pero nunca en los tests:

```go
type Contador struct {
    n int
}

func (c *Contador) Incrementar() {
    c.n++
}

func (c *Contador) Valor() int {
    return c.n
}

func TestContador(t *testing.T) {
    c := &Contador{}
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Incrementar()
        }()
    }
    wg.Wait()
    // El test a veces pasa (valor = 100) y a veces falla (valor < 100)
    // Con -race: siempre detecta el race
}
```

**Restricciones:**

1. Ejecutar con `-race` y confirmar que el race detector lo reporta
2. Hacer el race *visible* (que cause un valor incorrecto) en al menos
   el 50% de las ejecuciones sin `-race`, usando `runtime.Gosched()`
   para forzar el context switch en el momento exacto del race:

```go
func (c *Contador) IncrementarConRace() {
    valor := c.n          // leer
    runtime.Gosched()     // ← forzar switch aquí, entre la lectura y la escritura
    c.n = valor + 1       // escribir
}
```

3. Verificar que la corrección (`atomic.Int64`) elimina tanto el race
   como el valor incorrecto

**Pista:** `runtime.Gosched()` cede el CPU al scheduler de Go, permitiendo
que otra goroutine corra. Colocado entre la lectura y la escritura del contador,
garantiza que otra goroutine lea el mismo valor antes de que la primera lo actualice.
Esto hace el race completamente determinista. En producción, el OS hace esto
automáticamente pero en momentos impredecibles.

---

### Ejercicio 19.1.3 — Corregir: race en el patrón singleton

**Tipo: Corregir**

Este singleton tiene un data race clásico:

```go
var instancia *Servicio
var mu sync.Mutex

func ObtenerServicio() *Servicio {
    if instancia != nil {  // primera verificación — sin lock
        return instancia
    }
    mu.Lock()
    defer mu.Unlock()
    if instancia == nil {  // segunda verificación — con lock
        instancia = &Servicio{configurar()}
    }
    return instancia
}
```

**Preguntas:**

1. ¿Hay un data race aquí? El patrón tiene nombre — ¿cuál es?
2. Describe el interleaving exacto que causa el race.
3. ¿El race detector de Go lo detectaría? ¿Siempre?
4. Corrige usando `sync.Once` (la forma correcta en Go).
5. ¿Por qué `sync.Once` es correcto aunque internamente use el mismo
   double-check locking que el código incorrecto?

**Pista:** La primera verificación `if instancia != nil` sin lock es el race:
goroutine A puede estar escribiendo `instancia` (dentro del lock) mientras
goroutine B la lee (fuera del lock). En muchas arquitecturas, una escritura
parcial de un puntero es visible como un puntero inválido.
`sync.Once` usa memory barriers (instrucciones de CPU que garantizan que las
escrituras son visibles antes de que se lea el flag "done") que el double-check
manual no tiene.

---

### Ejercicio 19.1.4 — Leer: race en código de producción real

**Tipo: Leer/diagnosticar**

Este código viene de un sistema de caché con TTL en producción:

```go
type EntradaCache struct {
    valor     interface{}
    expira    time.Time
    accesos   int  // estadística
}

type Cache struct {
    entradas map[string]*EntradaCache
    mu       sync.RWMutex
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    entrada, ok := c.entradas[key]
    c.mu.RUnlock()

    if !ok || time.Now().After(entrada.expira) {
        return nil, false
    }

    entrada.accesos++  // ← ¿race aquí?
    return entrada.valor, true
}

func (c *Cache) Set(key string, valor interface{}, ttl time.Duration) {
    c.mu.Lock()
    c.entradas[key] = &EntradaCache{
        valor:  valor,
        expira: time.Now().Add(ttl),
    }
    c.mu.Unlock()
}
```

**Preguntas:**

1. ¿Hay un data race? ¿Dónde exactamente?
2. ¿El race detector lo detecta siempre?
3. ¿Qué consecuencia práctica tiene este race? (¿puede causar crash o solo dato incorrecto?)
4. Propón tres correcciones con distintos tradeoffs:
   - Sin cambiar la estructura de datos
   - Cambiando `accesos` a `atomic.Int64`
   - Eliminando la estadística del hot path
5. ¿Cuál corrección elegirías para un caché de alto rendimiento?

**Pista:** `entrada.accesos++` después de liberar el RLock es un race:
otra goroutine podría estar modificando la misma entrada (aunque Set reemplaza
el puntero, no modifica la entrada). El race es en la *entrada* apuntada,
no en el map. La consecuencia: `accesos` puede contar de más o de menos —
pero no causa crash porque es un int, no una estructura compleja.
Para alto rendimiento: `atomic.Int64` es la corrección con menor overhead.

---

### Ejercicio 19.1.5 — Race en el cierre de canales

**Tipo: Leer/corregir**

Este código tiene un race clásico sobre el cierre de canales:

```go
func procesarItems(items []Item) <-chan Resultado {
    resultados := make(chan Resultado, len(items))

    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        go func(i Item) {
            defer wg.Done()
            resultados <- procesar(i)
        }(item)
    }

    // Bug: cerrar el canal cuando todos terminen
    go func() {
        wg.Wait()
        close(resultados)
    }()

    return resultados
}

// El caller puede cerrar también:
func main() {
    ch := procesarItems(items)
    for resultado := range ch {
        usar(resultado)
    }
    close(ch)  // ← ¡race! procesarItems ya lo cerró
}
```

**Preguntas:**

1. ¿Cuántos lugares cierran el canal? ¿Cuál es la consecuencia?
2. Go produce un panic en `close` de un canal ya cerrado — ¿es este un race
   o es un bug de lógica? ¿Cuál es la diferencia?
3. ¿Qué convención de propiedad previene este tipo de bugs?
4. Corrige el código estableciendo claramente quién es dueño del canal.
5. ¿Cómo verificarías con un test que el canal nunca se cierra dos veces?

**Pista:** La convención en Go: **quien crea el canal es quien lo cierra**.
En este caso, `procesarItems` crea y cierra — el caller solo lee.
El patrón `for resultado := range ch` termina automáticamente cuando el canal
se cierra, por lo que el caller no necesita cerrarlo.
Para el test: `close` en un canal ya cerrado produce un panic detectable con `recover`.

---

## Sección 19.2 — Deadlocks: Diagnóstico y Resolución

### Ejercicio 19.2.1 — Leer: identificar deadlock en goroutine dump

**Tipo: Leer/diagnosticar**

Este es el goroutine dump de un servidor que "se congeló":

```
goroutine 1 [chan receive]:
main.main()
    /app/main.go:89

goroutine 34 [semacquire]:
sync.runtime_SemacquireMutex(0xc000124080)
    runtime/sema.go:77
sync.(*Mutex).Lock()
    sync/mutex.go:102
main.(*Inventario).Reservar(0xc000124000, ...)
    /app/inventario.go:45
main.(*Pedido).Confirmar(0xc000118000, ...)
    /app/pedido.go:78

goroutine 35 [semacquire]:
sync.runtime_SemacquireMutex(0xc000118080)
    runtime/sema.go:77
sync.(*Mutex).Lock()
    sync/mutex.go:102
main.(*Pedido).Actualizar(0xc000118000, ...)
    /app/pedido.go:102
main.(*Inventario).Liberar(0xc000124000, ...)
    /app/inventario.go:67
```

**Preguntas:**

1. Identifica los dos recursos en deadlock y qué goroutine tiene cada uno.
2. Dibuja el "wait-for graph" (quién espera a quién).
3. ¿Qué función del código de producción tiene el deadlock real?
4. Describe la secuencia de eventos que llevó a este estado.
5. Propón la corrección mínima (una línea o un reordenamiento).

**Pista:** Goroutine 34 tiene el lock de `Pedido` (lo adquirió en `Confirmar`)
y espera el lock de `Inventario`. Goroutine 35 tiene el lock de `Inventario`
(lo adquirió en `Liberar`) y espera el lock de `Pedido`.
La corrección: ordenar la adquisición de locks consistentemente.
Si siempre adquieres `Pedido` antes que `Inventario` (o viceversa), el deadlock
es imposible. El orden puede ser por dirección de memoria (`id(lock)`) o por convención.

---

### Ejercicio 19.2.2 — Reproducir: hacer un deadlock determinista en tests

**Tipo: Reproducir**

Este código tiene un deadlock intermitente — ocurre 1 vez en ~500 requests:

```go
type SistemaTransacciones struct {
    cuentaMu sync.Mutex
    stockMu  sync.Mutex
}

func (s *SistemaTransacciones) Comprar(userID, itemID string) error {
    s.cuentaMu.Lock()
    defer s.cuentaMu.Unlock()

    if !verificarSaldo(userID) {
        return ErrSaldoInsuficiente
    }

    s.stockMu.Lock()  // adquirir segundo lock mientras se tiene el primero
    defer s.stockMu.Unlock()

    return procesarCompra(userID, itemID)
}

func (s *SistemaTransacciones) Devolver(userID, itemID string) error {
    s.stockMu.Lock()  // orden inverso: stock primero
    defer s.stockMu.Unlock()

    s.cuentaMu.Lock()  // luego cuenta
    defer s.cuentaMu.Unlock()

    return procesarDevolucion(userID, itemID)
}
```

**Restricciones:**

1. Escribe un test que reproduce el deadlock en < 1 segundo de forma determinista
   usando `time.Sleep` o canales para forzar el interleaving exacto.
2. Verifica que el test falla (deadline del test context) cuando hay deadlock.
3. Corrige el código y verifica que el test pasa.

**Pista:** Para hacer el deadlock determinista:
```go
// Goroutine A: empieza Comprar, adquiere cuentaMu, luego pausa
// Goroutine B: empieza Devolver, adquiere stockMu, luego pausa
// Ahora ambas intentan adquirir el segundo lock → deadlock

ready := make(chan struct{})
go func() {
    s.cuentaMu.Lock()
    close(ready)           // señalizar que tiene cuentaMu
    s.stockMu.Lock()       // esperar stockMu — deadlock si B ya lo tiene
    // ...
}()
<-ready
s.stockMu.Lock()           // adquirir stockMu después de que A tiene cuentaMu
s.cuentaMu.Lock()          // deadlock aquí
```

---

### Ejercicio 19.2.3 — Corregir: deadlock en pipeline con canales

**Tipo: Corregir**

Este pipeline con canales tiene un deadlock cuando hay errores:

```go
func pipeline(entrada <-chan Item) (<-chan Resultado, <-chan error) {
    resultados := make(chan Resultado)
    errores := make(chan error)

    go func() {
        for item := range entrada {
            resultado, err := procesar(item)
            if err != nil {
                errores <- err  // ← si nadie lee errores, bloquea aquí
                continue
            }
            resultados <- resultado  // ← si nadie lee resultados, bloquea aquí
        }
        close(resultados)
        close(errores)
    }()

    return resultados, errores
}

// El caller solo lee resultados, ignora errores:
func main() {
    resultados, _ := pipeline(entrada)  // ← bug: nadie lee el canal de errores
    for r := range resultados {
        usar(r)
    }
}
```

**Preguntas:**

1. ¿Cuándo ocurre el deadlock exactamente?
2. ¿Por qué ignorar un canal de retorno puede causar un deadlock?
3. Propón tres formas de corregirlo:
   - Usando un canal buffered para errores
   - Unificando resultados y errores en un solo canal
   - Usando `errgroup` de `golang.org/x/sync`
4. ¿Cuál es la convención más idiomática en Go para este patrón?
5. ¿Qué herramienta detectaría este deadlock automáticamente?

**Pista:** Go detecta deadlocks "totales" (todos los goroutines bloqueados) y
produce `fatal error: all goroutines are asleep - deadlock!`. Pero un deadlock
"parcial" (solo algunas goroutines bloqueadas, otras siguen corriendo) no lo detecta.
Este es un deadlock parcial: la goroutine del pipeline está bloqueada enviando a `errores`,
pero `main` sigue corriendo (procesando `resultados`). La corrección con `errgroup`:
la goroutine cancela el contexto al primer error, y el caller solo recibe un error final.

---

### Ejercicio 19.2.4 — Leer: deadlock por context mal implementado

**Tipo: Leer/diagnosticar**

Este código intenta implementar cancelación con context pero tiene un deadlock:

```go
func obtenerConTimeout(ctx context.Context, key string) ([]byte, error) {
    resultado := make(chan []byte)
    errCh := make(chan error)

    go func() {
        data, err := bdLenta.Get(key)  // puede tardar 30s
        if err != nil {
            errCh <- err
        } else {
            resultado <- data
        }
    }()

    select {
    case data := <-resultado:
        return data, nil
    case err := <-errCh:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err()
        // ← la goroutine interna queda bloqueada enviando a resultado o errCh
        //   que nadie leerá después de este return
    }
}
```

**Preguntas:**

1. ¿Es este un deadlock o un goroutine leak? ¿Cuál es la diferencia?
2. ¿Cuántas goroutines quedan "colgadas" por cada llamada cancelada?
3. ¿Cuándo el sistema se vería afectado por este bug?
4. Corrige el código para que la goroutine interna siempre termine.
5. ¿Qué test verificaría que no hay goroutine leaks?

**Pista:** Es un goroutine leak (no un deadlock): `main` puede seguir,
pero la goroutine interna queda bloqueada para siempre en `resultado <- data`.
Con 1000 requests canceladas por hora, después de 24 horas hay 24,000 goroutines
colgadas. La corrección: hacer los canales buffered de tamaño 1, así la goroutine
interna puede enviar y terminar aunque nadie esté leyendo.
```go
resultado := make(chan []byte, 1)  // buffered
errCh := make(chan error, 1)       // buffered
```

---

### Ejercicio 19.2.5 — Construir: detector automático de deadlocks potenciales

**Tipo: Construir**

Implementa un detector de orden de locks que alerta cuando se adquieren
locks en orden inconsistente (que puede llevar a deadlock):

```go
type LockOrderDetector struct {
    mu      sync.Mutex
    ordenes map[string][]string  // para cada lock, qué locks ya se tienen
}

var detector = &LockOrderDetector{
    ordenes: make(map[string][]string),
}

type LockTrackeado struct {
    mu   sync.Mutex
    name string
}

func (l *LockTrackeado) Lock() {
    // Registrar que estamos adquiriendo 'l.name'
    // mientras ya tenemos los locks que están en el goroutine-local state
    detector.VerificarOrden(l.name, locksActualesDeEstaGoroutine())
    l.mu.Lock()
}
```

**Restricciones:** El detector debe:
- Registrar en qué orden se adquieren los locks
- Alertar (log.Warning o panic en tests) cuando detecta un orden inverso
- Funcionar sin overhead significativo en producción (desactivable con un flag)

**Pista:** Go no tiene thread-local storage, pero se puede simular con un
`sync.Map` indexado por goroutine ID (obtenible con `runtime.Stack` parseando
el output). En la práctica, este tipo de detector se implementa en Java (con
ThreadLocal) más fácilmente. Para Go, la alternativa más práctica es un
análisis estático o un lock hierarchy con comentarios en el código.

---

## Sección 19.3 — Goroutine Leaks: Detectar y Corregir

### Ejercicio 19.3.1 — Leer: identificar el leak en el código

**Tipo: Leer/diagnosticar**

Cada uno de estos snippets tiene un goroutine leak diferente. Identifica el leak
en cada uno sin ejecutar el código:

**Snippet A:**
```go
func procesarBatch(items []Item) []Resultado {
    ch := make(chan Resultado)
    for _, item := range items {
        go func(i Item) {
            ch <- procesar(i)
        }(item)
    }
    // Solo tomar el primer resultado:
    return []Resultado{<-ch}
    // ← len(items)-1 goroutines quedan esperando para enviar a ch
}
```

**Snippet B:**
```go
func monitorear(ctx context.Context) {
    go func() {
        ticker := time.NewTicker(time.Second)
        for {
            select {
            case <-ticker.C:
                verificarSalud()
            }
        }
        // ← nunca termina: no hay case para ctx.Done()
    }()
}
```

**Snippet C:**
```go
func suscribir(topic string) <-chan Mensaje {
    ch := make(chan Mensaje)
    broker.Subscribe(topic, func(msg Mensaje) {
        ch <- msg  // ← si el caller deja de leer ch, esta goroutine bloquea
    })
    return ch
}
```

**Restricciones:** Para cada snippet:
1. Describir exactamente qué goroutine queda colgada y por qué
2. Calcular cuántas goroutines se acumulan en 1 hora con 100 calls/min
3. Proponer la corrección mínima

**Pista:** Los tres patrones de leak más comunes:
A: "Fire and forget" con canal no buffered — las goroutines quedan esperando
para enviar a un canal que nadie leerá.
B: Goroutine de background sin mecanismo de terminación — vive para siempre.
C: Callback que bloquea — el suscriptor bloquea si el canal del caller está lleno.

---

### Ejercicio 19.3.2 — Reproducir: hacer el leak medible

**Tipo: Reproducir**

Escribe un test que:
1. Detecta si hay un goroutine leak en una función
2. Mide la tasa de crecimiento de goroutines bajo carga
3. Falla si las goroutines crecen más de N por 100 calls

```go
func TestSinLeak(t *testing.T) {
    baseline := runtime.NumGoroutine()

    for i := 0; i < 100; i++ {
        funcionBajoTest()
    }

    // Dar tiempo para que las goroutines terminen:
    time.Sleep(100 * time.Millisecond)
    runtime.GC()

    final := runtime.NumGoroutine()
    leak := final - baseline

    if leak > 5 {  // tolerancia pequeña por goroutines del sistema
        t.Errorf("posible goroutine leak: %d goroutines extra después de 100 calls", leak)
    }
}
```

**Restricciones:** Aplicar este patrón a las tres funciones del Ejercicio 19.3.1
para confirmar que tienen leaks. Luego aplicarlo a las versiones corregidas
para verificar que el leak está resuelto.

**Pista:** `runtime.GC()` antes de contar fuerza la recolección de goroutines
que el GC podría no haber reclamado aún. `time.Sleep()` es necesario porque
algunas goroutines pueden estar terminando en el momento del conteo.
La librería `goleak` (del Cap.07) hace esto de forma más robusta: usa
`goleak.VerifyNone(t)` que espera hasta que todas las goroutines terminan
o hasta un timeout.

---

### Ejercicio 19.3.3 — Corregir: leak en un servidor HTTP

**Tipo: Corregir**

Este servidor tiene un goroutine leak que se manifiesta después de horas de carga:

```go
func (s *Servidor) manejarRequest(w http.ResponseWriter, r *http.Request) {
    resultCh := make(chan string)

    go func() {
        // Operación potencialmente lenta:
        resultado := s.procesarLento(r.Context(), r.URL.Query().Get("id"))
        resultCh <- resultado
    }()

    select {
    case resultado := <-resultCh:
        fmt.Fprintln(w, resultado)
    case <-time.After(5 * time.Second):
        http.Error(w, "timeout", http.StatusGatewayTimeout)
        // ← la goroutine interna queda viva esperando enviar a resultCh
    }
}
```

**Restricciones:**

1. Identificar cuándo ocurre el leak (no en todos los requests)
2. Corregir sin cambiar la lógica de timeout
3. Verificar con `goleak` que el test no deja goroutines activas
4. Explicar por qué `r.Context()` no soluciona el problema por sí solo

**Pista:** `r.Context()` se cancela cuando el cliente HTTP desconecta — pero no
cuando el handler retorna. Para que la cancelación funcione, `procesarLento`
debe verificar el context (con `ctx.Done()`). La corrección completa:
pasar un contexto con timeout derivado del request context:
```go
ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
defer cancel()
// Ahora pasar ctx a procesarLento — se cancela al timeout O al disconnect
```
Y hacer resultCh buffered para que la goroutine pueda terminar aunque nadie lea.

---

### Ejercicio 19.3.4 — Leer: goroutine leak en un sistema de suscripción

**Tipo: Leer/diagnosticar**

Este sistema de pub-sub tiene un leak que crece lentamente:

```go
type EventBus struct {
    subs map[string][]chan Event
    mu   sync.RWMutex
}

func (b *EventBus) Subscribe(topic string) <-chan Event {
    ch := make(chan Event, 100)
    b.mu.Lock()
    b.subs[topic] = append(b.subs[topic], ch)
    b.mu.Unlock()
    return ch
}

func (b *EventBus) Publish(topic string, event Event) {
    b.mu.RLock()
    subs := b.subs[topic]
    b.mu.RUnlock()

    for _, ch := range subs {
        ch <- event  // ← bloquea si el subscriber no está leyendo
    }
}
```

**Preguntas:**

1. ¿Hay un goroutine leak aquí? (Pista: mira `Publish`, no `Subscribe`)
2. ¿Qué pasa si un subscriber deja de leer su canal pero no llama a Unsubscribe?
3. ¿Qué pasa si el canal buffered (100) se llena?
4. Propón un `Unsubscribe` que no causa leaks.
5. ¿Cómo `Publish` podría evitar bloquearse si el subscriber es lento,
   sin perder eventos?

**Pista:** No hay goroutine leak en el código mostrado — pero hay un bug de "canal lleno":
si un subscriber no lee y el canal (buffered de 100) se llena, `Publish` bloquea
el goroutine que llama `Publish`. Si Publish se llama desde un goroutine del pool,
ese goroutine queda inutilizable. La solución: `select { case ch <- event: default: metricas.MensajePerdido.Inc() }` — publicar sin bloquear, descartando si el subscriber es lento.

---

### Ejercicio 19.3.5 — Construir: wrapper que detecta leaks automáticamente

**Tipo: Construir**

Implementa un wrapper de `sync.WaitGroup` que detecta cuando las goroutines
no terminan en el tiempo esperado:

```go
type WaitGroupConTimeout struct {
    wg      sync.WaitGroup
    timeout time.Duration
    nombre  string  // para el mensaje de error
}

func (w *WaitGroupConTimeout) Add(n int) { w.wg.Add(n) }
func (w *WaitGroupConTimeout) Done()     { w.wg.Done() }

func (w *WaitGroupConTimeout) WaitConTimeout() error {
    done := make(chan struct{})
    go func() {
        w.wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        return nil
    case <-time.After(w.timeout):
        // Capturar el dump de goroutines para diagnóstico:
        buf := make([]byte, 64*1024)
        n := runtime.Stack(buf, true)
        return fmt.Errorf("%s: timeout después de %v\ngoroutines activas:\n%s",
            w.nombre, w.timeout, buf[:n])
    }
}
```

**Restricciones:** El wrapper debe:
- En timeout: capturar y loguear los stacks de las goroutines que no terminaron
- Funcionar correctamente con el contexto de cancelación
- Ser usable como reemplazo drop-in de `sync.WaitGroup`

**Pista:** El dump completo de goroutines puede ser grande (MB). Para producción,
considerar limitar el dump a las primeras N goroutines o filtrar por nombre de función.
`runtime.Stack(buf, true)` con `true` captura *todos* los goroutines, no solo el actual.
Para identificar cuáles goroutines son las problemáticas: buscar las que están en
`semacquire` o `chan send/receive` con los nombres de función del componente.

---

## Sección 19.4 — Heisenbugs: Bugs que Desaparecen al Observarlos

### Ejercicio 19.4.1 — Leer: por qué el log hace desaparecer el bug

**Tipo: Leer/diagnosticar**

Un equipo reporta: "El bug desaparece cuando añadimos logging".
Dado este código:

```go
func procesarItem(item Item) Resultado {
    go actualizar(item.ID, "procesando")  // goroutine sin sincronización

    resultado := calcular(item)

    // Con este log, el bug desaparece:
    log.Printf("calculado: %v", resultado)  // ← ¿por qué este log ayuda?

    go actualizar(item.ID, "completado")
    return resultado
}
```

**Preguntas:**

1. ¿Por qué añadir un `log.Printf` puede hacer desaparecer un race condition?
2. ¿Qué overhead tiene `log.Printf` en Go? ¿Por qué ese overhead importa aquí?
3. Describe el interleaving que el log previene (no elimina — previene).
4. ¿Es el log una "corrección" válida? ¿Por qué sí o por qué no?
5. ¿Cómo reproducirías el bug original sin el log para poder corregirlo correctamente?

**Pista:** `log.Printf` en Go adquiere un mutex interno y hace una syscall (write).
Ambas operaciones ceden el CPU al scheduler, dando tiempo a que la goroutine
de `actualizar` complete antes de que continue `procesarItem`. Esto no
*elimina* el race — lo hace menos probable. La corrección real requiere sincronización
explícita, no depender del timing de los logs.

---

### Ejercicio 19.4.2 — Técnicas para hacer bugs no-deterministas reproducibles

**Tipo: Construir**

Implementa un conjunto de herramientas para forzar interleavings específicos
en tests de concurrencia:

```go
// Técnica 1: Inyección de latencia en puntos específicos
type PuntoDeSincronizacion struct {
    nombre  string
    activo  bool
    barrera chan struct{}
}

func (p *PuntoDeSincronizacion) Llegar() {
    if !p.activo { return }
    p.barrera <- struct{}{}  // señalizar al test que llegamos aquí
    <-p.barrera              // esperar que el test nos libere
}

// Técnica 2: Forzar GC para exponer bugs de timing
func ForzarContextSwitch() {
    runtime.Gosched()
    runtime.GC()
}

// Técnica 3: Reducir el quantum del scheduler
func IniciarConSchedulerAgresivo() {
    // En tests, GOMAXPROCS=1 hace los races más deterministas
    // porque hay un solo procesador y el switch ocurre en puntos conocidos
    runtime.GOMAXPROCS(1)
}
```

**Restricciones:** Implementar las tres técnicas y aplicarlas a los bugs
de los ejercicios 19.1.2 y 19.2.2 para hacer los tests 100% reproducibles.

**Pista:** `GOMAXPROCS(1)` es la técnica más poderosa para hacer races deterministas:
con un solo procesador, el switch ocurre solo en puntos de cooperación (`Gosched`,
channel ops, syscalls). Los races que requieren preemption (el timer interrumpiendo
en medio de un `n++`) se vuelven reproducibles porque el switch solo ocurre
donde tú lo fuerzas con `runtime.Gosched()`.

---

### Ejercicio 19.4.3 — El bug que solo ocurre bajo carga

**Tipo: Reproducir**

El bug del Ejercicio 19.4.1 solo ocurre cuando el sistema procesa >500 req/s.
En el benchmark local con una sola goroutine, nunca falla.

**Restricciones:**

1. Escribe un stress test que simula la carga necesaria para reproducir el bug:

```go
func TestBajoAltaCarga(t *testing.T) {
    // Lanzar suficientes goroutines para saturar el scheduler
    // y crear el interleaving que revela el bug:
    const goroutines = 500
    var wg sync.WaitGroup
    errores := make(chan error, goroutines)

    for i := 0; i < goroutines; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            if err := verificarInvariante(procesarItem(generarItem(id))); err != nil {
                errores <- err
            }
        }(i)
    }

    wg.Wait()
    close(errores)

    for err := range errores {
        t.Error(err)
    }
}
```

2. El test debe fallar consistentemente antes de la corrección y pasar después.
3. Correr con `-race -count=100` para confirmar que el race detector lo ve.

**Pista:** `-count=100` ejecuta el test 100 veces. Para bugs de timing,
combinar `-count=100 -race` es muy efectivo: el race detector instrumenta
el código (haciéndolo ~2x más lento y cambiando el timing), lo que a veces
hace que bugs que no ocurrían sin `-race` se vuelvan reproducibles.
Y viceversa: algunos bugs desaparecen con `-race` por el overhead.

---

### Ejercicio 19.4.4 — Heisenbug por variable capturada en closure

**Tipo: Leer/corregir**

Este es uno de los bugs más comunes en Go para principiantes — y para expertos:

```go
func procesarTodos(items []Item) []Resultado {
    resultados := make([]Resultado, len(items))
    var wg sync.WaitGroup

    for i, item := range items {
        wg.Add(1)
        go func() {  // ← bug clásico
            defer wg.Done()
            resultados[i] = procesar(item)
            // i e item son las variables del loop — se comparten con el loop
        }()
    }

    wg.Wait()
    return resultados
}
```

**Preguntas:**

1. ¿Qué imprime este código si items = ["a", "b", "c"]?
   (Pista: casi nunca ["a_procesado", "b_procesado", "c_procesado"])
2. ¿Es esto un data race, un bug de semántica, o ambos?
3. El race detector ¿lo detecta?
4. Corrige de las tres formas idiomáticas en Go:
   - Pasando como parámetro
   - Creando variable local en el loop (`:= item` dentro del loop)
   - Con Go 1.22+ (el comportamiento del loop cambió)
5. ¿Cuándo Go 1.22 lo arregla automáticamente y cuándo no?

**Pista:** Antes de Go 1.22, las variables del `for range` se reusan en cada iteración.
La goroutine captura la variable (no el valor) — para cuando la goroutine corre,
la variable ya tiene el valor de la última iteración. El race detector puede o no
detectarlo dependiendo del interleaving. Go 1.22 cambió esto: cada iteración
tiene sus propias variables, haciendo este bug imposible para nuevos programas.
Pero código con `//go:build go1.21` sigue teniendo el bug antiguo.

---

### Ejercicio 19.4.5 — Estrategia: "smallest failing example"

**Tipo: Proceso**

Dado un bug de concurrencia en un sistema de 10,000 líneas, la estrategia
más eficiente es reducirlo al "smallest failing example" (SFE) —
el programa más pequeño que muestra el bug.

**Ejercicio:** El Ejercicio 19.1.4 (race en el caché con TTL) forma parte
de un sistema de 5000 líneas. Practica la reducción:

```
Sistema original: 5000 líneas, 20 archivos, 3 dependencias externas

Paso 1: Identificar el componente (cache.go, 200 líneas)
Paso 2: Extraer solo ese componente con un main() que reproduce el bug
Paso 3: Eliminar campos, métodos, y funciones no necesarios para el bug
Paso 4: El SFE debe ser < 50 líneas y reproducir el bug en < 1 segundo
```

**Restricciones:**

1. Crear el SFE para el bug del Ejercicio 19.1.4
2. El SFE debe ser ejecutable con `go run sfep.go` sin dependencias externas
3. El race detector debe detectar el race en el SFE
4. Documentar qué eliminaste y por qué no era necesario para el bug

**Pista:** El SFE es valioso por dos razones: primero, el proceso de reducción
a veces revela la causa raíz (al eliminar un componente, el bug desaparece —
ese componente era necesario). Segundo, el SFE es lo que envías en un bug report
o lo que muestras en un code review. Un SFE de 30 líneas es mucho más fácil
de razonar que el sistema completo.

---

## Sección 19.5 — Bugs de Liveness: Starvation y Livelock

### Ejercicio 19.5.1 — Leer: identificar starvation en métricas

**Tipo: Leer/diagnosticar**

Un sistema de prioridades tiene este comportamiento visible en las métricas:

```
Tarea prioridad ALTA:    procesadas=1,247  latencia_p99=45ms
Tarea prioridad MEDIA:   procesadas=891    latencia_p99=120ms
Tarea prioridad BAJA:    procesadas=12     latencia_p99=47,000ms (47 segundos!)

Tasa de arrival:
  ALTA:  50/s
  MEDIA: 30/s
  BAJA:  20/s

Workers: 8 (capacidad total: ~80 tareas/s a 10ms cada una)
```

**Preguntas:**

1. ¿Hay starvation? ¿Qué lo confirma?
2. La carga total (50+30+20 = 100/s) supera la capacidad (80/s). ¿Qué relación
   tiene esto con el starvation?
3. ¿Cuál sería la latencia p99 de BAJA si hubiera 0 starvation bajo esta carga?
4. Propón tres estrategias para mitigar el starvation sin eliminar las prioridades.
5. Si añades 2 workers más (total 10, capacidad ~100/s), ¿desaparece el starvation?

**Pista:** Con carga total = capacidad del sistema, cualquier variación en el arrival
rate de ALTA puede "robarle" capacidad a BAJA indefinidamente. Si ALTA llega en bursts,
puede monopolizar todos los workers. Las estrategias: (1) capacidad mínima garantizada
por prioridad (ej: 2 workers siempre para BAJA), (2) aging (incrementar la prioridad
de tareas que llevan mucho tiempo esperando), (3) weighted fair queuing.

---

### Ejercicio 19.5.2 — Reproducir: livelock detectable

**Tipo: Reproducir**

Un livelock es cuando dos goroutines se detectan mutuamente y se "echan atrás"
continuamente, sin progresar:

```go
// Dos procesos intentando "ser educados":
func procesoCortés(yo, otro *sync.Mutex, nombre string) {
    for intentos := 0; intentos < 1000; intentos++ {
        yo.Lock()

        // Verificar si el otro también está intentando:
        if !otro.TryLock() {
            // "Ser educado": soltar el mío para dejar pasar al otro
            yo.Unlock()
            time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond)
            continue  // intentar de nuevo
        }

        // Ambos locks adquiridos — hacer el trabajo:
        hacerTrabajo()
        otro.Unlock()
        yo.Unlock()
        return  // éxito
    }
    log.Println(nombre, ": agotados los intentos — livelock")
}
```

**Restricciones:**

1. Demostrar que este código puede livelockear (ambas goroutines
   se "echan atrás" simultáneamente indefinidamente)
2. Medir el porcentaje de tiempo que se gasta en "intentos fallidos"
3. Corregir con una estrategia que garantiza progreso

**Pista:** El livelock ocurre cuando ambas goroutines adquieren su lock,
detectan que el otro está ocupado, y se retiran al mismo tiempo.
Si los sleeps están sincronizados (por el scheduler), pueden seguir
retirándose al mismo tiempo indefinidamente. La corrección: jitter asimétrico
(uno espera siempre más que el otro) o backoff exponencial con jitter aleatorio
suficientemente grande para romper la sincronía.

---

### Ejercicio 19.5.3 — Corregir: starvation en un reader-writer lock

**Tipo: Corregir**

Este reader-writer lock manual tiene writer starvation:

```go
type RWLockConStarvation struct {
    readers atomic.Int32
    writing atomic.Bool
}

func (l *RWLockConStarvation) RLock() {
    for l.writing.Load() {
        runtime.Gosched()
    }
    l.readers.Add(1)
}

func (l *RWLockConStarvation) RUnlock() {
    l.readers.Add(-1)
}

func (l *RWLockConStarvation) Lock() {
    l.writing.Store(true)
    for l.readers.Load() > 0 {
        runtime.Gosched()  // esperar a que todos los readers terminen
    }
}

func (l *RWLockConStarvation) Unlock() {
    l.writing.Store(false)
}
```

**Preguntas:**

1. ¿Hay writer starvation o reader starvation? Describe el escenario exacto.
2. ¿Hay también un data race en este código?
3. ¿Cómo `sync.RWMutex` de Go resuelve el starvation?
4. Corrige el lock manual para eliminar el starvation (pista: queue de escritores pendientes).
5. ¿Cuándo un lock manual valdría la pena sobre `sync.RWMutex`?

**Pista:** La secuencia de writer starvation:
- Writer llama Lock(), establece writing=true
- Mientras espera a los readers existentes, nuevos readers pueden entrar
  (hay una ventana entre cuando writer establece writing=true y cuando readers
  verifican la condición)
- Si hay un stream continuo de readers, el writer puede nunca adquirir el lock
`sync.RWMutex` resuelve esto registrando writers pendientes: una vez que hay un
writer esperando, nuevos readers deben esperar en lugar de adquirir inmediatamente.

---

### Ejercicio 19.5.4 — Leer: starvation en un connection pool

**Tipo: Leer/diagnosticar**

Un connection pool a la base de datos tiene este comportamiento bajo carga:

```
Conexiones en pool: 20
Requests en vuelo: 450

Distribución de tiempo esperando conexión:
  p50: 2ms
  p90: 15ms
  p99: 8,400ms  ← !!
  p999: 30,000ms (timeout)

CPU del servidor de BD: 45% (no saturado)
Queries activas en BD: 18-20 (el pool está siendo usado, no hay queries lentas)
```

**Preguntas:**

1. ¿Por qué p99 es 8400ms cuando p50 es solo 2ms?
2. ¿El problema está en el pool, en la BD, o en el cliente?
3. ¿Qué tipo de distribución de colas produce este patrón de latencia?
4. ¿Qué cambio en la configuración del pool resolvería el p99?
5. ¿Cuántas conexiones necesitarías para que p99 < 100ms con 450 requests en vuelo
   a 10ms promedio de query? (Usar Little's Law del Cap.17 §17.5.3)

**Pista:** Con 450 requests y 20 conexiones, hay 430 requests esperando en la cola
en cualquier momento. Los 2ms de p50 son para los requests que tienen suerte
de encontrar una conexión libre inmediatamente. Los 8400ms de p99 son para
los requests que llegan cuando la cola ya tiene ~430 items. Con Little's Law:
`L = λW` → `450 = λ × 0.010` → `λ = 45,000 req/s`... pero tenemos 20 conexiones.
El tiempo de espera = (items en cola) / (throughput del pool) = 430 / (20/0.010) = 215ms.
Si p99 es 8400ms, hay momentos de spike donde la cola es mucho más larga.

---

### Ejercicio 19.5.5 — Construir: priority queue con aging anti-starvation

**Tipo: Construir**

Implementa una priority queue para el Worker Pool del Cap.03 con aging:
las tareas de baja prioridad incrementan su prioridad con el tiempo
para garantizar que eventualmente se ejecuten:

```go
type TareaConPrioridad struct {
    tarea         Task
    prioridad     int       // 1=baja, 2=media, 3=alta
    encolada      time.Time // cuando se añadió a la cola
}

func (t *TareaConPrioridad) PrioridadEfectiva() int {
    // El "aging": por cada 5 segundos de espera, subir 1 nivel de prioridad
    espera := time.Since(t.encolada)
    incremento := int(espera.Seconds() / 5)
    return min(t.prioridad+incremento, 3)  // máximo nivel 3
}
```

**Restricciones:**

- La prioridad efectiva debe calcularse en O(1)
- Garantía: una tarea de prioridad 1 debe ejecutarse en máximo 15 segundos
  aunque lleguen continuamente tareas de prioridad 3
- Verificar con un test de carga que no hay starvation

**Pista:** Un heap (`container/heap`) con prioridad efectiva que se recalcula
al extraer (no al insertar) es suficiente para este aging. El problema:
si la queue tiene 10,000 items, recalcular la prioridad efectiva de todos
al cambiar el tiempo es O(n). La solución: recalcular solo al extraer (`Pop`)
y usar el tiempo de encolado + prioridad original para ordenar. Para garantía
de 15 segundos con aging de 5s/nivel: prioridad 1 → nivel 2 a los 5s → nivel 3 a los 10s → ejecutado antes de los 15s.

---

## Sección 19.6 — Memory Corruption y Uso Incorrecto de unsafe

### Ejercicio 19.6.1 — Leer: panic por map concurrent access

**Tipo: Leer/diagnosticar**

Este es el stack trace de un panic en producción:

```
goroutine 1847 [running]:
runtime.throw2({0x6f4a20?, 0x0?})
    runtime/panic.go:1023
runtime.mapaccess2_faststr(0xc00013e180, {0xc000234010, 0xa})
    runtime/map_faststr.go:108
main.(*SessionStore).Get(...)
    /app/session.go:34
main.handleRequest(...)
    /app/handler.go:67

goroutine 1848 [running]:
runtime.throw2({0x6f4a20?, 0x0?})
    runtime/panic.go:1023
runtime.mapassign_faststr(0xc00013e180, {0xc000278044, 0xe})
    runtime/map.go:579
main.(*SessionStore).Set(...)
    /app/session.go:51
main.handleRequest(...)
    /app/handler.go:72
```

**Preguntas:**

1. ¿Qué tipo de operación causa este panic? (pista: `mapaccess2_faststr`, `mapassign_faststr`)
2. ¿Por qué Go hace panic en lugar de retornar datos corruptos silenciosamente?
3. ¿El race detector habría detectado esto antes del panic?
4. ¿Qué código de SessionStore causó esto?
5. Propón la corrección y explica por qué sync.Map podría ser mejor que mutex+map
   para un SessionStore con muchas lecturas.

**Pista:** `mapaccess2_faststr` es la función interna de Go para leer de un map
con key de tipo string. `mapassign_faststr` es para escribir. El panic
"concurrent map read and map write" ocurre porque el runtime detecta el acceso
concurrente y prefiere hacer panic (reproducible) en lugar de retornar datos corruptos
(silencioso y difícil de debuggear). El race detector lo habría detectado antes,
pero solo si el interleaving específico ocurrió durante los tests con `-race`.

---

### Ejercicio 19.6.2 — Leer: slice compartido entre goroutines

**Tipo: Leer/diagnosticar**

Este código tiene un bug sutil de memoria compartida:

```go
func procesarEnParalelo(datos []byte) ([]Resultado, error) {
    n := len(datos) / 4
    resultados := make([]Resultado, 4)

    var wg sync.WaitGroup
    for i := 0; i < 4; i++ {
        wg.Add(1)
        chunk := datos[i*n : (i+1)*n]  // slice del mismo array subyacente
        go func(idx int, data []byte) {
            defer wg.Done()
            resultados[idx] = procesar(data)
        }(i, chunk)
    }

    wg.Wait()
    return resultados, nil
}
```

**Preguntas:**

1. ¿Hay un data race aquí? Examina tanto `datos` como `resultados`.
2. ¿Las goroutines modifican `datos`? Si `procesar` es read-only, ¿hay race?
3. ¿Hay race en `resultados`? Cada goroutine escribe en un índice diferente.
4. Si `procesar` modifica el slice (normaliza los datos in-place), ¿cambia la respuesta?
5. ¿Cómo verificarías que `procesar` es truly read-only sin leer su implementación?

**Pista:** Escrituras a índices *diferentes* de un slice no son un data race
(cada goroutine escribe en `resultados[0]`, `resultados[1]`, etc. — memoria separada).
Pero si `procesar` modifica `data` (el chunk del slice original), y los chunks
se solapan (por redondeo), podría haber un race. Verificar con `-race` y una
implementación de `procesar` que modifica in-place. Para garantizar que
`procesar` es read-only: cambia el parámetro a `data []byte` y pásale una copia
— si el test falla, `procesar` sí modifica los datos.

---

### Ejercicio 19.6.3 — Corregir: uso incorrecto de sync.Pool

**Tipo: Corregir**

`sync.Pool` es un pool de objetos reutilizables — pero tiene semántica sutil:

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}

func procesarRequest(data []byte) []byte {
    buf := bufferPool.Get().([]byte)
    // BUG: el buffer puede contener datos de un request anterior:
    buf = append(buf, data...)
    resultado := calcular(buf)

    bufferPool.Put(buf)  // devolver al pool
    return resultado
}
```

**Preguntas:**

1. ¿Cuál es el bug? (Pista: no es un data race — es un bug de lógica)
2. ¿Cuándo el bug produce resultados incorrectos silenciosamente?
3. ¿Puede producir un data race además del bug de lógica?
4. Corrige el código correctamente.
5. ¿Cuándo `sync.Pool` *no* es una buena idea (a pesar de reducir las allocations)?

**Pista:** El bug: `bufferPool.Get()` retorna el buffer con `len=0` pero `cap=4096` —
el contenido del slice desde la última vez que se usó puede estar ahí si la
capacidad no se reseteó. El `append` empieza a añadir después de `len=0`,
lo que parece correcto, pero si el buffer se devuelve con `len > 0`, el próximo
Get() obtiene un buffer "con contenido" que el usuario puede no esperar.
La corrección: `bufferPool.Put(buf[:0])` — devolver con len=0 para resetear.

---

### Ejercicio 19.6.4 — Leer: use-after-free simulado en Go con finalizers

**Tipo: Leer/diagnosticar**

Go tiene GC, pero `runtime.SetFinalizer` puede crear un patrón similar
al use-after-free de C:

```go
type Recurso struct {
    conn net.Conn
    closed bool
}

func NuevoRecurso() *Recurso {
    r := &Recurso{conn: abrirConexion()}
    runtime.SetFinalizer(r, func(r *Recurso) {
        r.conn.Close()  // cerrar cuando el GC recolecte r
        r.closed = true
    })
    return r
}

func (r *Recurso) Usar() error {
    if r.closed {
        return ErrRecursoCerrado
    }
    return r.conn.Write([]byte("datos"))
    // BUG: si el GC corre entre el check y el Write,
    // el finalizer puede cerrar la conexión
}
```

**Preguntas:**

1. ¿Cuándo puede el finalizer correr concurrentemente con `Usar()`?
2. ¿Hay un race condition entre `r.closed` y el finalizer?
3. ¿Por qué confiar en finalizers para cerrar recursos es generalmente una mala idea?
4. Corrige usando el patrón correcto para recursos que necesitan cierre explícito.
5. ¿Cuál es la relación entre este bug y el patrón de `defer r.Close()`?

**Pista:** El finalizer puede correr en *cualquier* momento después de que el GC
determine que el objeto no tiene referencias alcanzables. Si la goroutine
solo tiene un puntero local a `r`, el GC puede decidir que `r` es recolectable
incluso mientras `Usar()` está ejecutando (el puntero local puede no estar
en el stack en ese punto del análisis). La solución correcta: usar `defer r.Close()`
explícitamente y no usar finalizers para lógica de cierre crítica.

---

### Ejercicio 19.6.5 — Construir: test suite para verificar thread safety

**Tipo: Construir**

Implementa una suite de tests que verifica que una estructura de datos
es correctamente thread-safe bajo distintas condiciones de carga:

```go
// ThreadSafetyTest ejecuta una serie de operaciones concurrentes
// y verifica que el resultado final es correcto:
func ThreadSafetyTest(t *testing.T, estructura interface{
    Set(key string, value int)
    Get(key string) (int, bool)
    Delete(key string)
}) {
    t.Helper()

    // Test 1: 1000 writes concurrentes, resultado debe ser consistente
    // Test 2: reads y writes mezclados, no debe haber panic
    // Test 3: con -race, no debe reportar data races
    // Test 4: resultado final debe matchear la versión secuencial
    // Test 5: bajo carga de 8 goroutines, no debe haber valores perdidos
}
```

**Restricciones:**

- Los tests deben poder aplicarse a cualquier implementación de la interfaz
- Deben comparar el resultado con una implementación de referencia secuencial
- Deben ser deterministas (mismo resultado en cada ejecución)
- Deben completarse en < 5 segundos

**Pista:** La comparación con una implementación de referencia secuencial es
la técnica más efectiva para detectar bugs de concurrencia en estructuras de datos:
aplica el mismo conjunto de operaciones en orden secuencial y en paralelo,
y verifica que el resultado final es el mismo. Esto no verifica que no hay races
(puede haber races que producen el resultado correcto "de casualidad"), pero
verifica que el comportamiento observable es correcto. Combinar con `-race`
cubre ambos casos.

---

## Sección 19.7 — Estrategias Sistemáticas de Debugging

### Ejercicio 19.7.1 — El método "cambiar una cosa a la vez"

**Tipo: Proceso**

Un servicio en producción tiene un bug de concurrencia que no se reproduce en dev.
Las diferencias entre producción y dev son:

```
Producción:            Dev:
  GOMAXPROCS: 32         GOMAXPROCS: 8
  Carga: 10,000 req/s    Carga: 10 req/s
  Memoria: 64 GB         Memoria: 16 GB
  OS: Linux 5.15         OS: macOS 14
  Go: 1.21               Go: 1.21
  Dependencias:          Dependencias:
    Redis (remoto)           Redis (local, docker)
    BD PostgreSQL (remoto)   BD PostgreSQL (local, docker)
```

**Restricciones:**

Diseña el plan de debugging siguiendo el principio de "cambiar una cosa a la vez":
1. Prioriza las diferencias por probabilidad de causar el bug
2. Para cada diferencia, describe cómo la cambiarías en dev para matchear producción
3. Explica qué descartarías primero y por qué
4. Propón cuál diferencia es más probable que sea relevante para un bug de concurrencia

**Pista:** `GOMAXPROCS: 32` es la diferencia más relevante para bugs de concurrencia —
con 32 goroutines corriendo en paralelo real, hay más interleavings posibles que con 8.
Para reproducir: `GOMAXPROCS=32 go test -race -count=100 ./...`.
La latencia de red (Redis/BD remotos) también importa: cambia el timing de las
operaciones concurrentes. Para reproducir: añadir latencia artificial con un proxy.

---

### Ejercicio 19.7.2 — git bisect para bugs de concurrencia

**Tipo: Proceso**

`git bisect` automatiza la búsqueda del commit que introdujo un bug:

```bash
git bisect start
git bisect bad HEAD          # el bug existe aquí
git bisect good v1.2.0       # no existía en esta versión

# Git hace checkout de commits intermedios:
# → ejecutar el test de reproducción
# → si el test falla: git bisect bad
# → si el test pasa: git bisect good
# → git bisect encuentra el commit exacto

# Automatizar con un script:
git bisect run go test -race -run TestRaceCondicion -count=100 ./...
```

**Restricciones:**

Para un bug de concurrencia intermitente donde el test falla el ~30% de las veces:

1. El script de `git bisect run` debe ejecutar el test múltiples veces
   para distinguir "bug presente" de "falso negativo":

```bash
#!/bin/bash
# bisect.sh
PASS=0; FAIL=0
for i in $(seq 1 10); do
    if go test -race -run TestRaceCondicion -count=10 ./...; then
        PASS=$((PASS+1))
    else
        FAIL=$((FAIL+1))
    fi
done
# Si más de 3 de 10 runs fallan: el bug está presente
[ $FAIL -gt 3 ] && exit 1 || exit 0
```

2. Calcular cuántos commits revisará git bisect dado un rango de 100 commits.
3. Estimar el tiempo total si cada run del script toma 2 minutos.

**Pista:** `git bisect` hace búsqueda binaria: con 100 commits necesita
log₂(100) ≈ 7 iteraciones. Con 10 runs de 2 minutos cada una = 20 min por commit.
Total: 7 × 20 = ~140 minutos. Para bugs de concurrencia donde el test falla
ocasionalmente, ajustar el número de runs y el umbral de fallo para balancear
la confianza con el tiempo total del bisect.

---

### Ejercicio 19.7.3 — Reducción de casos de prueba automatizada

**Tipo: Construir**

Implementa un reductor de casos de prueba para el Worker Pool:
dado un test que falla intermitentemente, encontrar el subset mínimo
de operaciones que reproduce el fallo:

```go
type OperacionPool struct {
    tipo     string // "submit", "shutdown", "resize"
    delay    time.Duration
    goroutines int
}

func reducirCasoFallido(operaciones []OperacionPool) []OperacionPool {
    // Algoritmo: delta debugging
    // 1. Intentar con la mitad de las operaciones
    // 2. Si la mitad reproduce el bug: reducir esa mitad
    // 3. Si no: intentar con la otra mitad
    // 4. Repetir hasta no poder reducir más
}
```

**Restricciones:** El reductor debe encontrar el caso mínimo en O(n log n) runs.
El caso mínimo no debe ser mayor de 2x el tamaño óptimo.

**Pista:** El algoritmo de delta debugging (Andreas Zeller) es el estándar para
esto. Para bugs deterministas, reduce el caso rápidamente. Para bugs intermitentes,
necesita ejecutar cada candidato múltiples veces para distinguir "reproduce el bug"
de "no lo reproduce por casualidad". La librería `github.com/dvyukov/go-fuzz`
implementa una versión para fuzzing que puede adaptarse a este uso.

---

### Ejercicio 19.7.4 — Construir el "concurrency regression suite"

**Tipo: Construir**

Implementa una suite de tests de regresión para bugs de concurrencia ya corregidos.
El objetivo: cada bug que se encuentra y se corrige debe dejar un test que falla
si el bug regresa:

```go
// Cada test de regresión documenta:
// 1. El bug original (descripción)
// 2. El escenario que lo reproducía
// 3. La corrección aplicada
// 4. El test que verifica que no regresa

// Ejemplo:
func TestRegressionRace_CacheSet_20240115(t *testing.T) {
    // Bug: data race en Cache.Set cuando dos goroutines llaman Set
    // simultáneamente con la misma key.
    // Corregido en commit abc123 añadiendo sync.RWMutex.
    // Este test usa -race para verificar que la corrección no regresa.

    cache := NewCache()
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            cache.Set("key", []byte(fmt.Sprintf("value%d", n)))
        }(i)
    }
    wg.Wait()
    // Si hay race, go test -race falla aquí
}
```

**Restricciones:** Implementar la suite con los bugs de los ejercicios
19.1.1, 19.2.1, y 19.3.1. Cada test debe:
- Pasar con la corrección aplicada
- Fallar (con `-race`) sin la corrección
- Completarse en < 1 segundo

**Pista:** Los tests de regresión de bugs de concurrencia son especialmente valiosos
porque los bugs de concurrencia tienen alta tasa de recurrencia — un cambio en otra
parte del sistema puede "destapar" el mismo race de una nueva forma. Mantener
estos tests en CI con `-race` es la red de seguridad más efectiva.

---

### Ejercicio 19.7.5 — El debugging journal: documentar el proceso

**Tipo: Proceso/documentar**

Documenta el proceso completo de debugging de un bug de concurrencia
en formato "debugging journal" — el registro de lo que pensaste, probaste,
y aprendiste en cada paso:

```markdown
# Debugging Journal: Race en Cache TTL
**Fecha:** 2024-01-15
**Sistema:** Servicio de sesiones
**Síntoma:** Crash esporádico en producción, 1-2 veces por semana

## Hipótesis inicial
El crash ocurre en `session.go:34` — sospecho acceso concurrente al mapa.

## Paso 1: Reproducir
Ejecuté `go test -race ./...` → encontré el race (Ejercicio 19.1.4).
Tiempo: 5 minutos.

## Paso 2: Entender
Tracé el interleaving exacto: Get lee `accesos` sin lock mientras Set puede
estar reemplazando la entrada. El race es en la *entrada* (struct), no en el mapa.

## Paso 3: Corregir
Opción elegida: `atomic.Int32` para `accesos`.
Razón: mínimo overhead, sin cambiar la estructura del lock.

## Paso 4: Verificar
`go test -race -count=1000 ./...` → 0 races reportados.
Load test 30min con `-race` → 0 races.

## Lección aprendida
RWMutex protege el mapa pero no los campos de las structs en el mapa.
Añadir al checklist de code review: "¿se modifican campos de structs que
están en mapas protegidos por RWMutex?"
```

**Restricciones:** Escribe el journal para el debugging del bug
del Ejercicio 19.2.1 (deadlock en el goroutine dump). El journal debe
incluir los caminos equivocados que tomaste antes de encontrar la causa raíz.

**Pista:** Los caminos equivocados son la parte más valiosa del journal —
documentan qué hipótesis descartaste y por qué. Esto es útil para:
(1) no repetir el mismo callejón sin salida en un bug similar,
(2) enseñar al equipo a reconocer las pistas que apuntan en la dirección correcta,
(3) escribir el post-mortem si el bug causó un incidente.

---

## Resumen del capítulo

**Los cinco bugs de concurrencia más comunes y cómo detectarlos:**

```
1. Data race
   Detectar: go test -race
   Síntoma: valores incorrectos, panic "concurrent map write"
   Causa raíz: acceso a memoria compartida sin sincronización

2. Deadlock
   Detectar: goroutine dump (SIGQUIT o /debug/pprof/goroutine)
   Síntoma: el sistema se "congela", CPU al 0%
   Causa raíz: ciclo en el grafo de espera de locks

3. Goroutine leak
   Detectar: goleak, runtime.NumGoroutine() creciente
   Síntoma: memoria que crece lentamente, goroutines que no terminan
   Causa raíz: canal sin leer, context sin cancelar, goroutine sin join

4. Starvation
   Detectar: latencia p99 vs p999 muy dispares, métricas por prioridad
   Síntoma: algunas tareas tardan mucho aunque el sistema "está bien"
   Causa raíz: prioridades sin aging, recursos no compartidos equitativamente

5. Heisenbug
   Detectar: desaparece con logging/profiling → bug de timing
   Síntoma: intermitente, no reproducible en dev
   Causa raíz: dependencia en el timing del scheduler, closure mal capturada
```

**La herramienta más importante es la que no necesitas usar:**

> Un sistema con buen diseño (canales con backpressure, ownership claro de recursos,
> contextos propagados correctamente) tiene pocos bugs de concurrencia que debuggear.
> Invertir en el diseño correcto desde el principio es más eficiente que
> ser experto en debuggear el diseño incorrecto.
