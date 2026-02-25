# Guía de Ejercicios — Cap.21: Resiliencia en Sistemas Concurrentes

> Lenguaje principal: **Go**. Los patrones aplican a cualquier lenguaje.
>
> Este capítulo cierra la Parte 3b. La resiliencia es la propiedad de
> un sistema de degradarse de forma controlada bajo condiciones adversas
> — en lugar de fallar completamente.
>
> Los tres tipos de ejercicios:
> - **Implementar**: construir los patrones de resiliencia desde cero
> - **Calibrar**: ajustar los parámetros para que los patrones funcionen
> - **Componer**: combinar múltiples patrones en un sistema coherente

---

## El problema central

Los sistemas concurrentes fallan de formas que los sistemas secuenciales no:

```
Sistema secuencial bajo carga alta:
  → Se vuelve lento
  → Las requests tardan más
  → Eventualmente responde (solo lento)

Sistema concurrente bajo carga alta (sin resiliencia):
  → Las goroutines se acumulan esperando recursos
  → El memory crece hasta OOM
  → El sistema muere completamente
  → Todas las requests fallan (no solo las nuevas)

Sistema concurrente bajo carga alta (con resiliencia):
  → Rechaza requests nuevas (backpressure)
  → Sirve las requests en vuelo con degradación elegante
  → Se recupera automáticamente cuando baja la carga
```

La diferencia entre el segundo y el tercer escenario es el conjunto de patrones
que cubre este capítulo.

---

## Tabla de contenidos

- [Sección 21.1 — Circuit Breaker: detener la cascada](#sección-211--circuit-breaker-detener-la-cascada)
- [Sección 21.2 — Retry con backoff: reintentar con inteligencia](#sección-212--retry-con-backoff-reintentar-con-inteligencia)
- [Sección 21.3 — Timeout hierarchy: contratos de tiempo](#sección-213--timeout-hierarchy-contratos-de-tiempo)
- [Sección 21.4 — Bulkhead: aislar los fallos](#sección-214--bulkhead-aislar-los-fallos)
- [Sección 21.5 — Hedging: la apuesta de cobertura](#sección-215--hedging-la-apuesta-de-cobertura)
- [Sección 21.6 — Degradación elegante: funcionar con menos](#sección-216--degradación-elegante-funcionar-con-menos)
- [Sección 21.7 — Componer: el sistema resiliente completo](#sección-217--componer-el-sistema-resiliente-completo)

---

## Sección 21.1 — Circuit Breaker: Detener la Cascada

El circuit breaker evita que las llamadas a un servicio externo que está
fallando saturen el sistema llamante. En lugar de acumular goroutines esperando
un timeout de 30 segundos, el breaker abre y devuelve error inmediatamente.

```
Estado CERRADO (normal):
  → Pasa todas las llamadas
  → Monitorea la tasa de errores
  → Si errores > umbral → abre

Estado ABIERTO (fallando):
  → Rechaza todas las llamadas inmediatamente
  → No llama al servicio externo
  → Después de un tiempo de espera → pasa a HALF-OPEN

Estado HALF-OPEN (probando):
  → Pasa un número limitado de llamadas de prueba
  → Si tienen éxito → cierra
  → Si fallan → vuelve a abrir
```

### Ejercicio 21.1.1 — Implementar el circuit breaker básico

Implementa un circuit breaker con los tres estados y la lógica de transición:

```go
type Estado int

const (
    Cerrado  Estado = iota
    Abierto
    HalfOpen
)

type CircuitBreaker struct {
    mu             sync.Mutex
    estado         Estado
    fallos         int       // fallos consecutivos en estado Cerrado
    umbralFallos   int       // fallos antes de abrir
    ultimoFallo    time.Time
    tiempoEspera   time.Duration // tiempo en Abierto antes de HalfOpen
    pruebasHalfOpen int      // llamadas de prueba en HalfOpen
    exitosHalfOpen  int      // éxitos acumulados en HalfOpen
}

// Call ejecuta fn si el breaker lo permite.
// Retorna ErrCircuitoAbierto si el breaker está Abierto.
func (cb *CircuitBreaker) Call(fn func() error) error {
    // TODO: implementar
}
```

**Restricciones:**
- La transición Cerrado→Abierto ocurre cuando `fallos >= umbralFallos`
- La transición Abierto→HalfOpen ocurre después de `tiempoEspera`
- La transición HalfOpen→Cerrado ocurre cuando `exitosHalfOpen >= pruebasHalfOpen`
- La transición HalfOpen→Abierto ocurre al primer fallo en HalfOpen
- Exponer el estado actual como métrica (gauge: 0=Cerrado, 1=Abierto, 2=HalfOpen)

**Pista:** El estado y los contadores deben estar bajo el mismo mutex para que
las transiciones sean atómicas. Si dos goroutines llegan simultáneamente al umbral,
solo una debe abrir el breaker (la segunda encuentra ya el estado Abierto).
La transición Abierto→HalfOpen es "lazy" — ocurre cuando alguien intenta
una llamada, no en un ticker separado.

---

### Ejercicio 21.1.2 — Calibrar: los umbrales que importan

Un circuit breaker mal calibrado es peor que no tener uno:

```
Demasiado sensible (umbral bajo, tiempo de espera corto):
  → Se abre con errores transitorios normales
  → Oscila entre Abierto y Cerrado (flapping)
  → Los usuarios ven errores innecesarios

Demasiado insensible (umbral alto, tiempo de espera largo):
  → No protege cuando el servicio externo está degradado
  → Las goroutines se acumulan durante el tiempo extra
```

**Restricciones:** Para un servicio externo con estas características:
- P99 de latencia normal: 50ms
- Timeout configurado: 5 segundos
- Tasa de error normal: 0.1%
- Tasa de error durante fallo: 80%
- Tiempo típico de recuperación del servicio: 2 minutos

Calcula y justifica los valores óptimos para:
- `umbralFallos`: ¿cuántos fallos consecutivos antes de abrir?
- `tiempoEspera`: ¿cuánto tiempo en Abierto antes de probar?
- `pruebasHalfOpen`: ¿cuántas llamadas de prueba en HalfOpen?

Luego verifica con un test de simulación que el breaker no hace flapping
con la tasa de error normal (0.1%) pero sí abre con la tasa de fallo (80%).

**Pista:** Para `umbralFallos`: con 0.1% de error normal y requests en ráfagas,
un umbral de 5 fallos consecutivos puede abrirse por casualidad. Umbral de 10
es más seguro. Para `tiempoEspera`: 2 minutos de recuperación del servicio →
empezar a probar a los 30-60s (no al 100% del tiempo de recuperación).
Para `pruebasHalfOpen`: 3-5 pruebas exitosas da suficiente confianza sin
exponer demasiado tráfico real a un servicio inestable.

---

### Ejercicio 21.1.3 — Circuit breaker basado en latencia

El circuit breaker del Ejercicio 21.1.1 reacciona a errores. Un sistema
que responde correctamente pero muy lentamente también es un problema:

```go
type CircuitBreakerLatencia struct {
    // Además de los fallos, monitorear la latencia:
    ventana     []time.Duration // últimas N latencias (sliding window)
    umbralLento time.Duration   // qué es "lento" (ej: 2s)
    rateLento   float64         // % de llamadas lentas para abrir (ej: 0.5 = 50%)
    mu          sync.Mutex
    // ... resto del estado del breaker básico
}
```

**Restricciones:**
- El breaker abre si: `(fallos / total) > umbralErrores` O `(llamadasLentas / total) > rateLento`
- La ventana deslizante debe ser de las últimas 100 llamadas
- Una llamada que tarda más del timeout se cuenta como fallo Y como llamada lenta

**Pista:** El percentil P99 no es directamente calculable sin ordenar la ventana
(O(n log n)). Para un breaker en producción, una aproximación más eficiente:
contar cuántas llamadas superan el umbral de latencia (operación O(1)) en lugar
de calcular el percentil exacto. La librería `resilience4j` de Java usa exactamente
este enfoque: cuenta la tasa de "slow calls" en lugar del percentil.

---

### Ejercicio 21.1.4 — Leer: diagnosticar un breaker que flappea

**Tipo: Diagnosticar**

Este gráfico de métricas muestra el estado del circuit breaker a lo largo del tiempo:

```
Tiempo    Estado    Tasa de error    Latencia P99
00:00     Cerrado   0.8%             45ms
00:05     Cerrado   1.2%             52ms
00:10     Cerrado   2.1%             78ms
00:15     Abierto   --               --         ← breaker abre
00:16     HalfOpen  --               --
00:17     Cerrado   --               --         ← breaker cierra (3 pruebas ok)
00:18     Cerrado   1.8%             65ms
00:20     Cerrado   2.3%             89ms
00:22     Abierto   --               --         ← vuelve a abrir
00:23     HalfOpen  --               --
00:24     Cerrado   --               --         ← vuelve a cerrar
[patrón se repite cada 7-10 minutos]
```

**Preguntas:**

1. ¿Qué nombre tiene este comportamiento?
2. ¿Qué indica que el servicio externo no se ha recuperado completamente?
3. ¿Qué parámetro del breaker está mal calibrado?
4. ¿Cómo cambia el impacto en los usuarios durante el flapping vs durante un Abierto sostenido?
5. Propón los ajustes de configuración que estabilizarían el comportamiento.

**Pista:** El flapping ocurre cuando el servicio externo se recupera parcialmente:
suficiente para pasar las `pruebasHalfOpen` (3 llamadas exitosas) pero no suficiente
para mantenerse estable bajo carga completa. La corrección: aumentar `pruebasHalfOpen`
a 10-20 y hacerlas gradualmente (1% del tráfico, luego 5%, luego 10%) en lugar
de pasar inmediatamente al 100%.

---

### Ejercicio 21.1.5 — Circuit breaker con fallback integrado

Un circuit breaker solo tiene valor si hay algo útil que hacer cuando está abierto:

```go
type BreakerConFallback[T any] struct {
    breaker  *CircuitBreaker
    fallback func(ctx context.Context) (T, error)
}

func (b *BreakerConFallback[T]) Ejecutar(
    ctx context.Context,
    fn func(ctx context.Context) (T, error),
) (T, error) {
    err := b.breaker.Call(func() error {
        resultado, e := fn(ctx)
        if e == nil {
            // guardar resultado para retornar
        }
        return e
    })

    if errors.Is(err, ErrCircuitoAbierto) {
        return b.fallback(ctx)
    }
    // retornar resultado o error
}
```

**Restricciones:** Implementar con tres niveles de fallback en cascada:
1. Caché local (TTL de 5 minutos)
2. Datos degradados (respuesta simplificada sin campos opcionales)
3. Error explícito con mensaje útil para el usuario

El nivel de fallback activo debe ser visible como métrica:
`fallback_level{level="cache|degraded|error"}`.

**Pista:** El fallback en cascada requiere que cada nivel indique si puede
servir la request. La caché puede tener un miss (no hay datos para esa key).
Los datos degradados siempre están disponibles pero pueden ser menos útiles.
El error explícito es siempre el último recurso. La métrica permite monitorear
qué porcentaje del tráfico está siendo servido por cada nivel.

---

## Sección 21.2 — Retry con Backoff: Reintentar con Inteligencia

### Ejercicio 21.2.1 — Implementar backoff exponencial con jitter

El retry sin jitter causa "thundering herd" — todos los clientes reintentan
al mismo tiempo, generando un spike en el servicio que acaba de recuperarse:

```go
type ConfigBackoff struct {
    DelayInicial  time.Duration // ej: 100ms
    DelayMaximo   time.Duration // ej: 30s
    Multiplicador float64       // ej: 2.0
    Jitter        float64       // ej: 0.2 (±20% de variación)
    MaxIntentos   int           // ej: 5
}

type ReintentadorBackoff struct {
    config ConfigBackoff
}

// Calcular el delay para el intento N (base 0):
func (r *ReintentadorBackoff) calcularDelay(intento int) time.Duration {
    // delay = min(DelayInicial * Multiplicador^intento, DelayMaximo)
    // jitter = delay * Jitter * (rand en [-1, 1])
    // resultado = delay + jitter (clamped a [0, DelayMaximo])
}

func (r *ReintentadorBackoff) Ejecutar(ctx context.Context, fn func() error) error {
    // TODO: implementar el loop de reintentos con el backoff calculado
}
```

**Restricciones:**
- No reintentar errores no-retryables (definidos por una función `EsRetryable(error) bool`)
- El jitter debe distribuirse uniformemente (no gaussiano)
- Respetar la cancelación del contexto en el sleep del backoff
- Exponer el número de reintentos como métrica

**Pista:** El sleep con cancelación:
```go
select {
case <-time.After(delay):
    // continuar con el siguiente intento
case <-ctx.Done():
    return ctx.Err()
}
```
Para el jitter: `delay + time.Duration(rand.Float64()*2*jitterMax - jitterMax)`
donde `jitterMax = delay * config.Jitter`.

---

### Ejercicio 21.2.2 — Clasificar errores retryables

No todos los errores se deben reintentar. Reintentar un error 4xx es inútil;
reintentar un timeout puede ser la respuesta correcta:

```go
// Implementar EsRetryable para estos tipos de error:

func EsRetryable(err error) bool {
    // Errores de red transitórios → retryable
    var netErr *net.OpError
    if errors.As(err, &netErr) {
        return netErr.Temporary()  // ← ¿es Temporary() confiable?
    }

    // Errors de HTTP:
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        switch httpErr.StatusCode {
        case 429: return true   // Too Many Requests → retry (con backoff)
        case 500: return true   // Internal Server Error → retry
        case 502: return true   // Bad Gateway → retry
        case 503: return true   // Service Unavailable → retry
        case 504: return true   // Gateway Timeout → retry
        case 400: return false  // Bad Request → no retry (el request está mal)
        case 401: return false  // Unauthorized → no retry (renovar credenciales)
        case 404: return false  // Not Found → no retry (el recurso no existe)
        case 409: return false  // Conflict → depende del contexto
        }
    }

    return false
}
```

**Restricciones:** Implementar la clasificación y verificar con tests
para cada código de HTTP. Documentar qué hacer con los casos ambiguos
(409 Conflict puede ser retryable en algunos contextos y no en otros).

**Pista:** `net.OpError.Temporary()` fue deprecado en Go 1.18 porque era
poco confiable — muchos errores temporales no lo implementaban.
La alternativa moderna: verificar si el error implementa la interface
`interface { Timeout() bool }` para timeouts, y tratar los errores de
conexión rechazada como retryables. El 409 Conflict en APIs REST generalmente
indica un conflicto de estado que no desaparece con un retry — pero en
sistemas de consenso distribuido puede ser retryable.

---

### Ejercicio 21.2.3 — Retry con idempotency key

El retry de operaciones no-idempotentes (como crear un pedido) puede
causar duplicados:

```go
// Sin idempotency key:
// POST /orders → crea el pedido
// POST /orders → timeout (el pedido puede haberse creado o no)
// POST /orders → retry → crea un segundo pedido (duplicado!)

// Con idempotency key:
// POST /orders + X-Idempotency-Key: abc123 → crea el pedido
// POST /orders + X-Idempotency-Key: abc123 → timeout
// POST /orders + X-Idempotency-Key: abc123 → retry →
//   el servidor detecta que abc123 ya procesó → retorna el resultado original

type ClienteIdempotente struct {
    httpClient  *http.Client
    reintentador *ReintentadorBackoff
}

func (c *ClienteIdempotente) Post(ctx context.Context, url string, body interface{}) (*http.Response, error) {
    idempotencyKey := uuid.New().String()  // generar una vez, reusar en todos los reintentos

    return c.reintentador.EjecutarHTTP(ctx, func() (*http.Response, error) {
        req, _ := http.NewRequestWithContext(ctx, "POST", url, marshal(body))
        req.Header.Set("X-Idempotency-Key", idempotencyKey)
        return c.httpClient.Do(req)
    })
}
```

**Restricciones:** Implementar el cliente idempotente y el servidor que
procesa las idempotency keys. El servidor debe:
- Guardar los resultados de requests procesados (Redis o mapa en memoria con TTL)
- Si la misma key llega dos veces, retornar el resultado guardado
- Si la key llega mientras la primera está procesando, esperar y retornar el mismo resultado

**Pista:** El servidor necesita manejar el caso de "la key está siendo procesada":
si dos requests llegan simultáneamente con la misma key, el segundo debe
esperar al primero en lugar de procesar en paralelo. Esto requiere un lock
por idempotency key — `sync.Map` con `singleflight` es el patrón correcto.

---

### Ejercicio 21.2.4 — Leer: el retry que empeoró el problema

**Tipo: Diagnosticar**

Un sistema tiene este comportamiento bajo un fallo del servicio externo:

```
Tasa de requests al servicio externo (normales):
  14:00 - 14:05: 1000 req/min

Fallo del servicio externo a las 14:05:

Tasa de requests al servicio externo (con retry sin jitter):
  14:05 - 14:06: 3000 req/min  ← !! (1000 originales + 2000 reintentos)
  14:06 - 14:07: 5000 req/min  ← !! (creciendo)
  14:07:         servicio externo muere completamente (OOM)

Con retry + jitter:
  14:05 - 14:10: 1200 req/min  ← ligeramente elevado pero estable
  14:10:         servicio externo se recupera
```

**Preguntas:**

1. ¿Qué nombre tiene el fenómeno donde el retry empeora la situación?
2. ¿Por qué sin jitter los reintentos se sincronizan?
3. Calcula: con 1000 req/min originales y 3 reintentos a intervalos fijos de 1s,
   ¿cuántos requests por minuto llegan al servicio externo en el peor caso?
4. ¿Cuándo el circuit breaker debería haber abierto para prevenir esto?
5. ¿El retry con jitter elimina el thundering herd completamente o solo lo mitiga?

**Pista:** Sin jitter, todos los clientes que fallaron al mismo tiempo (cuando el
servicio cayó) reintentan al mismo tiempo (1 segundo después), luego de nuevo
al mismo tiempo (2 segundos después), etc. El jitter dispersa los reintentos
en el tiempo, reduciendo el spike pero no eliminándolo completamente — todavía
hay más tráfico que en condiciones normales. El circuit breaker es la segunda
línea de defensa: cuando el error rate supera el umbral, corta el tráfico en lugar
de seguir reintentando.

---

### Ejercicio 21.2.5 — Retry budget: cuánto reintentar en total

En un sistema con múltiples capas, los reintentos se multiplican:

```
Cliente → Servicio A (3 reintentos) → Servicio B (3 reintentos) → BD (3 reintentos)

Si BD falla: BD hace 3 intentos
Servicio B ve el error y hace 3 reintentos de su llamada a BD:
  → 3 × 3 = 9 requests a BD

Servicio A ve el error y hace 3 reintentos de su llamada a Servicio B:
  → 3 × 9 = 27 requests a BD

Si hay 100 clientes simultáneos:
  → 2700 requests a BD por cada request original
```

**Restricciones:** Implementar un "retry budget" que limita el número total
de reintentos del sistema, no por capa:

```go
type RetryBudget struct {
    // El token bucket controla los reintentos:
    // - Se llena a una tasa de N tokens por segundo
    // - Cada reintento consume 1 token
    // - Si no hay tokens disponibles, no reintentar
    tokens     atomic.Int64
    maxTokens  int64
    refillRate float64  // tokens por segundo
}

func (rb *RetryBudget) PermitirReintento() bool {
    // Retorna true si hay tokens disponibles (consume uno)
    // Retorna false si el budget está agotado
}
```

**Pista:** El token bucket se puede implementar con un timer que añade tokens
periódicamente, o de forma "lazy" calculando cuántos tokens se han acumulado
desde la última llamada. El approach lazy es más eficiente:
```go
ahora := time.Now()
tokensAcumulados := int64(ahora.Sub(ultimaRecarga).Seconds() * refillRate)
// añadir tokensAcumulados al bucket, sin superar maxTokens
```

---

## Sección 21.3 — Timeout Hierarchy: Contratos de Tiempo

### Ejercicio 21.3.1 — Diseñar la jerarquía de timeouts

Para un sistema con esta arquitectura, diseña los timeouts de cada capa:

```
Usuario (browser/app)
  └── API Gateway           [timeout: ?]
        └── Servicio A      [timeout: ?]
              ├── Redis      [timeout: ?]
              └── Servicio B [timeout: ?]
                    └── BD PostgreSQL [timeout: ?]
```

**Restricciones:** Los timeouts deben satisfacer:
- El usuario nunca debe esperar más de 3 segundos
- Cada capa debe tener al menos 100ms de overhead para manejo de errores
- Los timeouts deben disminuir de afuera hacia adentro
- Si la BD hace timeout, el error debe propagarse hasta el usuario en < 500ms
- Redis debe responder en < 10ms (si no, algo está muy mal)

**Pista:** Estructura típica:
```
Usuario:       3000ms
API Gateway:   2800ms (200ms de overhead)
Servicio A:    2500ms (300ms de overhead)
  Redis:         10ms
  Servicio B:  2000ms (500ms de overhead)
    BD:         1500ms
```
El overhead de cada capa incluye: serialización, routing de red, y tiempo
para loguear/métricas el error antes de propagarlo.

---

### Ejercicio 21.3.2 — Implementar timeout propagado con context

El timeout del request original debe propagarse a todas las capas:

```go
func manejarRequest(w http.ResponseWriter, r *http.Request) {
    // El contexto ya tiene el deadline del request HTTP
    // (configurado por el servidor con ReadTimeout/WriteTimeout)
    ctx := r.Context()

    // Cada llamada downstream debe respetar el deadline restante:
    resultado, err := servicioA.Llamar(ctx, datos)
    // Si el deadline del request es en 200ms, servicioA lo sabe
    // y no intenta hacer una operación que tarda 500ms
}

// En servicioA:
func (s *ServicioA) Llamar(ctx context.Context, datos Datos) (Resultado, error) {
    // Verificar cuánto tiempo queda:
    deadline, ok := ctx.Deadline()
    if ok && time.Until(deadline) < 100*time.Millisecond {
        // No hay suficiente tiempo — fallar rápido
        return Resultado{}, ErrTiempoInsuficiente
    }

    // Añadir un timeout propio más estricto que el del padre:
    ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()

    return s.procesarInterno(ctx, datos)
}
```

**Restricciones:** Implementar el sistema con los timeouts del Ejercicio 21.3.1.
Si el deadline restante es menor que el timeout de la siguiente capa,
usar el deadline restante (no el timeout configurado). Verificar con tests
que una request que llega con 50ms de deadline restante falla rápido en cada capa.

**Pista:** `context.WithTimeout` vs `context.WithDeadline`: si el padre ya tiene
un deadline más cercano, el hijo hereda el deadline del padre aunque el timeout
del hijo sea más largo. Go garantiza que el contexto hijo expira cuando el padre
expira O cuando el timeout del hijo vence, lo que ocurra primero.

---

### Ejercicio 21.3.3 — Detectar orphan requests

Una "orphan request" es un request cuyo cliente ya hizo timeout pero el servidor
sigue procesando. Es trabajo inútil que consume recursos:

```go
// Detectar y cancelar orphan requests:
func manejarRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    procesarDatos := func() {
        // Operación larga:
        for _, chunk := range obtenerChunks() {
            select {
            case <-ctx.Done():
                // El cliente desconectó o hizo timeout — parar aquí
                log.Info("request cancelado por cliente", "reason", ctx.Err())
                metricas.OrphansCancelados.Inc()
                return
            default:
                procesarChunk(chunk)
            }
        }
    }

    procesarDatos()
}
```

**Restricciones:** Implementar un middleware que:
1. Mide cuántos requests se cancelan antes de completarse
2. Distingue entre cancelación por timeout vs desconexión del cliente
3. Loguea el tiempo invertido en la request antes de la cancelación

**Pista:** `ctx.Err()` retorna `context.DeadlineExceeded` (timeout) o
`context.Canceled` (cancelación explícita o desconexión del cliente).
En Go's `net/http`, `r.Context()` se cancela automáticamente cuando el cliente
cierra la conexión. Para distinguir timeout de desconexión: el servidor puede
tener un deadline configurado con `http.Server.WriteTimeout`, que causa
`DeadlineExceeded`. Una desconexión del cliente causa `Canceled`.

---

### Ejercicio 21.3.4 — El timeout adaptativo

Un timeout fijo no es óptimo: si el p99 normal es 45ms, un timeout de 5s
es demasiado permisivo. Un timeout de 100ms puede ser demasiado estricto
durante picos de carga normales:

```go
type TimeoutAdaptativo struct {
    // El timeout se ajusta basándose en el p99 observado:
    p99Observado time.Duration
    multiplicador float64  // ej: 2.0 → timeout = p99 × 2
    minTimeout   time.Duration
    maxTimeout   time.Duration

    // Histograma de latencias recientes:
    histograma   *HistogramaExponencial  // del Cap.10
}

func (t *TimeoutAdaptativo) Timeout() time.Duration {
    p99 := t.histograma.Percentil(0.99)
    timeout := time.Duration(float64(p99) * t.multiplicador)
    return clamp(timeout, t.minTimeout, t.maxTimeout)
}
```

**Restricciones:** El timeout adaptativo debe:
- Actualizarse con las últimas 1000 observaciones (ventana deslizante)
- Tener un mínimo y máximo fijo (para evitar timeouts extremos)
- No cambiar más rápido que cada 10 segundos (para estabilidad)

**Pista:** El timeout adaptativo es útil para servicios con variabilidad alta
en la latencia (microservicios compartidos, servicios en hardware variable).
Un timeout de `p99 × 2` significa que el 1% de los requests normales harán timeout —
puede ser aceptable si el servicio es interno. Para servicios de cara al usuario,
`p99 × 3` o `p999 × 1.5` es más conservador.

---

### Ejercicio 21.3.5 — Leer: encontrar el timeout incorrecto

**Tipo: Diagnosticar**

Este trace muestra un request que falló con timeout:

```
GET /checkout [timeout after 3000ms]
│
├─ [0ms] validate_cart [8ms]
│
├─ [8ms] check_inventory [12ms]
│
├─ [20ms] process_payment [2980ms → timeout]
│   └─ [20ms] http.POST payment-gateway/charge
│       [esperando... 2980ms... timeout del request padre]
│       [payment-gateway interno: procesó en 150ms y retornó 200 OK]
│       [pero el caller ya hizo timeout y no leyó la respuesta]
│
└─ [timeout] — usuario ve error
   [payment-gateway: el pago SÍ se procesó]
```

**Preguntas:**

1. ¿El pago se procesó o no? ¿Cómo lo sabrías?
2. ¿Qué está mal en la jerarquía de timeouts?
3. ¿Por qué el payment-gateway tardó 2980ms si normalmente tarda 150ms?
4. ¿Cómo el uso de idempotency keys (Ejercicio 21.2.3) habría ayudado aquí?
5. ¿Qué cambio de arquitectura previene este tipo de situación?

**Pista:** El payment-gateway tardó 2980ms porque estaba procesando correctamente —
el problema es que el timeout del request padre (3000ms) era demasiado corto
para incluir el tiempo de validación + inventario (20ms) + el tiempo normal
del gateway (150ms) + cualquier latencia de red o variabilidad. Los 2980ms
sugieren que el gateway estaba esperando una respuesta de su banco adquiriente
que tardó más de lo normal. La solución: el timeout del request padre debe ser
mayor que la suma de todos los timeouts de las operaciones internas.

---

## Sección 21.4 — Bulkhead: Aislar los Fallos

### Ejercicio 21.4.1 — Implementar bulkhead con pools separados

El bulkhead impide que el fallo de un componente consuma todos los recursos
del sistema, afectando a otros componentes:

```go
// Sin bulkhead: un pool compartido para todo
// Si el servicio B está lento, puede agotar el pool → el servicio A también falla

// Con bulkhead: pools separados por dependencia
type BulkheadManager struct {
    pools map[string]*WorkerPool
}

func NuevoBulkheadManager(config map[string]BulkheadConfig) *BulkheadManager {
    bm := &BulkheadManager{
        pools: make(map[string]*WorkerPool),
    }
    for nombre, cfg := range config {
        bm.pools[nombre] = NuevoWorkerPool(cfg.MaxConcurrentes, cfg.QueueSize)
    }
    return bm
}

func (bm *BulkheadManager) Ejecutar(nombre string, ctx context.Context, fn func()) error {
    pool, ok := bm.pools[nombre]
    if !ok {
        return fmt.Errorf("bulkhead desconocido: %s", nombre)
    }
    return pool.SubmitConContext(ctx, fn)
}
```

**Restricciones:** Configurar los bulkheads para este sistema:
- Llamadas a BD: máximo 20 concurrentes, queue de 50
- Llamadas a servicio de pagos: máximo 10 concurrentes, queue de 20
- Llamadas a servicio de emails: máximo 5 concurrentes, queue de 100

Verificar con un test de carga que el fallo del servicio de pagos (todos los
workers bloqueados) no afecta la capacidad de hacer llamadas a BD.

**Pista:** Los tamaños de bulkhead se calculan con Little's Law:
`concurrentes = throughput_objetivo × latencia_p99`
Para BD: si quieres 200 queries/s y la latencia p99 es 10ms →
`concurrentes = 200 × 0.010 = 2`... lo cual parece bajo. La fórmula
da el estado estable; el pool más grande (20) acomoda los spikes.
El queue size determina cuántas requests se pueden acumular antes de rechazar.

---

### Ejercicio 21.4.2 — Bulkhead por tenant

En un sistema multi-tenant, un tenant que genera mucho tráfico no debe
degradar la experiencia de otros tenants:

```go
type BulkheadTenant struct {
    pools  map[string]*WorkerPool  // un pool por tenant
    poolMu sync.RWMutex
    config BulkheadConfig
}

func (bt *BulkheadTenant) Ejecutar(tenantID string, ctx context.Context, fn func()) error {
    bt.poolMu.RLock()
    pool, ok := bt.pools[tenantID]
    bt.poolMu.RUnlock()

    if !ok {
        // Crear pool para nuevo tenant:
        bt.poolMu.Lock()
        // Double-check dentro del write lock:
        if pool, ok = bt.pools[tenantID]; !ok {
            pool = NuevoWorkerPool(bt.config.MaxConcurrentes, bt.config.QueueSize)
            bt.pools[tenantID] = pool
        }
        bt.poolMu.Unlock()
    }

    return pool.SubmitConContext(ctx, fn)
}
```

**Restricciones:**
- Los tenants "premium" deben tener pools más grandes que los "free"
- Los pools de tenants inactivos deben limpiarse periódicamente (evitar memory leak)
- Debe haber un límite de tenants simultáneos (para evitar OOM con tenants maliciosos)

**Pista:** La limpieza de pools inactivos requiere rastrear cuándo se usó por
última vez cada pool. Un LRU cache de pools es la estructura correcta:
cuando el cache está lleno (límite de tenants), evictar el pool menos
recientemente usado (y hacer `Shutdown()` del pool antes de eliminarlo).

---

### Ejercicio 21.4.3 — Bulkhead semafórico

El bulkhead no siempre requiere un pool completo. A veces un semáforo
que limita la concurrencia es suficiente:

```go
type BulkheadSemaforo struct {
    sem chan struct{}
}

func NuevoBulkheadSemaforo(max int) *BulkheadSemaforo {
    bs := &BulkheadSemaforo{sem: make(chan struct{}, max)}
    for i := 0; i < max; i++ {
        bs.sem <- struct{}{}
    }
    return bs
}

func (bs *BulkheadSemaforo) Ejecutar(ctx context.Context, fn func() error) error {
    // Adquirir:
    select {
    case <-bs.sem:
        defer func() { bs.sem <- struct{}{} }()
    case <-ctx.Done():
        return ErrBulkheadLleno
    }
    return fn()
}
```

**Restricciones:** Comparar el bulkhead semafórico con el bulkhead de pool
en términos de throughput, latencia, y overhead de memoria con 1000 goroutines
concurrentes y 10 de concurrencia máxima.

**Pista:** La diferencia principal: el bulkhead semafórico bloquea la goroutine
llamante (que puede ser un goroutine del servidor HTTP — goroutines de Go son baratas).
El bulkhead de pool tiene sus propios goroutines workers — la goroutine llamante
solo espera en la queue del canal. Para Go, donde los goroutines son ligeros,
el bulkhead semafórico es frecuentemente suficiente y más simple.

---

### Ejercicio 21.4.4 — Leer: identificar la ausencia de bulkhead

**Tipo: Diagnosticar**

Este goroutine dump muestra un sistema sin bulkhead bajo fallo de un servicio externo:

```
goroutine 1 [chan receive]: main.main
goroutine 18 [running]: http.(*Server).Serve

[847 goroutines en estado "IO wait":]
goroutine 19..866 [IO wait]:
  net.(*netFD).Read
  net.(*conn).Read
  net/http.(*connReader).Read
  net/http.(*response).finishRequest
  main.manejarRequest
    /app/handler.go:34  ← todos en la misma línea

[La línea 34 de handler.go:]
34:    resp, err := servicioExterno.Get(ctx, id)
```

**Preguntas:**

1. ¿Qué está pasando? ¿Por qué hay 847 goroutines en IO wait?
2. ¿Qué servicio externo está fallando?
3. ¿Qué recursos del sistema están siendo consumidos por estas goroutines?
4. ¿Qué pasa con los nuevos requests que llegan mientras esto ocurre?
5. ¿Cómo un bulkhead de 20 concurrentes habría cambiado este escenario?

**Pista:** Sin bulkhead, cada request HTTP lanza una goroutine que llama al servicio
externo. Si el servicio externo está lento (respondiendo en 30s en lugar de 50ms),
todas las goroutines se acumulan esperando. Con 100 req/s de tráfico y 30s de
espera, hay `100 × 30 = 3000` goroutines en vuelo. El dump muestra 847 porque
el sistema empezó a rechazar conexiones antes. Con bulkhead de 20: los primeros
20 requests esperan al servicio lento; los siguientes reciben ErrBulkheadLleno
inmediatamente (fail fast) — mucho mejor que 847 goroutines colgadas.

---

### Ejercicio 21.4.5 — Bulkhead adaptativo

Un bulkhead con límite fijo es un tradeoff: muy pequeño y throttlea en condiciones
normales; muy grande y no protege bajo carga:

```go
type BulkheadAdaptativo struct {
    actual     atomic.Int64     // concurrencia actual
    limite     atomic.Int64     // límite actual (se ajusta)
    limiteMin  int64
    limiteMax  int64
    latencias  *HistogramaExponencial
    sloLatencia time.Duration   // objetivo de latencia
}

// Ajustar el límite cada 5 segundos basándose en el p99:
func (ba *BulkheadAdaptativo) ajustar() {
    p99 := ba.latencias.Percentil(0.99)
    if p99 > ba.sloLatencia {
        // Latencia alta → reducir límite (menos concurrencia)
        nuevoLimite := max(ba.limiteMin, ba.limite.Load()-1)
        ba.limite.Store(nuevoLimite)
    } else if p99 < ba.sloLatencia/2 {
        // Latencia muy baja → podemos añadir más concurrencia
        nuevoLimite := min(ba.limiteMax, ba.limite.Load()+1)
        ba.limite.Store(nuevoLimite)
    }
}
```

**Restricciones:** El límite adaptativo debe converger al valor óptimo en
< 2 minutos desde el arranque. Verificar con un test de simulación que
el bulkhead se ajusta correctamente ante cambios de carga.

**Pista:** Este es el algoritmo AIMD (Additive Increase, Multiplicative Decrease)
del Cap.10 §10.6.1 aplicado al bulkhead. El ajuste "lento subiendo, rápido bajando"
previene que el límite suba demasiado durante una recuperación de carga.

---

## Sección 21.5 — Hedging: La Apuesta de Cobertura

El hedging lanza una segunda request si la primera no responde en tiempo:
en lugar de esperar el timeout completo, se hace la misma operación
en paralelo con un pequeño delay.

```
Sin hedging:
  Request 1 → lento (p99 = 500ms) → timeout a los 2s → error

Con hedging (hedge delay = 100ms):
  Request 1 → [100ms] → todavía no respondió
  Request 2 → lanzar segunda request en paralelo
  Request 2 → responde en 50ms (total: 150ms desde el inicio)
  Request 1 → cancelar
```

### Ejercicio 21.5.1 — Implementar hedged requests

```go
type ClienteHedged struct {
    cliente     *http.Client
    hedgeDelay  time.Duration  // esperar este tiempo antes de lanzar la segunda
    maxHedges   int            // máximo de requests paralelas (generalmente 2-3)
}

func (c *ClienteHedged) Get(ctx context.Context, url string) (*http.Response, error) {
    tipo resultado struct {
        resp *http.Response
        err  error
    }

    resultados := make(chan resultado, c.maxHedges)

    lanzarRequest := func() {
        resp, err := c.cliente.Get(url)
        resultados <- resultado{resp, err}
    }

    // Primera request:
    go lanzarRequest()

    // Si no hay respuesta en hedgeDelay, lanzar más:
    for hedge := 1; hedge < c.maxHedges; hedge++ {
        select {
        case r := <-resultados:
            return r.resp, r.err  // primera respuesta gana
        case <-time.After(c.hedgeDelay):
            go lanzarRequest()  // lanzar segunda request
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }

    // Esperar la primera respuesta exitosa:
    select {
    case r := <-resultados:
        return r.resp, r.err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

**Restricciones:**
- Cancelar las requests perdedoras cuando una gana (para no desperdiciar recursos)
- Solo retornar la primera respuesta exitosa (ignorar errores si al menos una triunfa)
- Métricas: tasa de hedges activados, reducción de latencia p99

**Pista:** Para cancelar las requests perdedoras: usar un contexto con cancel y
pasarlo a cada request. Cuando una gana, llamar `cancel()`. Las requests restantes
verán `ctx.Done()` y terminarán. El problema: si el servidor es idempotente
pero costoso, las requests canceladas pueden haber iniciado trabajo que no se
puede revertir. El hedging es apropiado para operaciones de lectura; con escrituras,
requiere idempotency keys.

---

### Ejercicio 21.5.2 — Calibrar el hedge delay

El hedge delay es el parámetro más importante del hedging:

```
Demasiado corto (ej: 1ms):
  → Casi siempre se lanza la segunda request
  → El tráfico al servicio se duplica
  → El beneficio de latencia es mínimo (la segunda request no es más rápida)

Demasiado largo (ej: igual al timeout):
  → Solo se activa para los casos muy lentos
  → No ayuda con el tail latency normal

Óptimo: el p95 de latencia normal
  → El 95% de requests responden antes del hedge
  → El 5% más lento genera una segunda request
  → El tráfico aumenta solo un 5%
```

**Restricciones:** Dado este histograma de latencias:

```
p50:  12ms
p75:  18ms
p90:  45ms
p95:  89ms
p99:  340ms
p999: 2100ms
```

Calcula el `hedgeDelay` óptimo si:
- El objetivo es reducir el p99 a < 100ms
- El presupuesto de tráfico adicional es máximo el 10%
- El servicio es stateless e idempotente

**Pista:** Con `hedgeDelay = p95 = 89ms`, el 5% de las requests activan el hedge.
La segunda request tiene p50 = 12ms adicional → el p99 efectivo sería ~89ms + 12ms = 101ms.
Para llegar a < 100ms consistentemente: `hedgeDelay = p90 = 45ms` activa hedges
para el 10% de requests (dentro del presupuesto), y el p99 efectivo sería ~45ms + 12ms = 57ms.

---

### Ejercicio 21.5.3 — Hedging en un pipeline concurrente

El hedging aplica no solo a requests HTTP sino a cualquier operación con tail latency alta:

```go
// Hedging para queries a BD:
func (r *Repositorio) ObtenerUsuarioHedged(ctx context.Context, id int64) (*Usuario, error) {
    // Si hay múltiples réplicas de lectura, hacer hedge entre ellas:
    réplicas := r.réplicasDisponibles()

    resultados := make(chan struct {
        usuario *Usuario
        err     error
    }, len(réplicas))

    lanzar := func(réplica *bd.Conn) {
        usuario, err := réplica.ObtenerUsuario(ctx, id)
        resultados <- struct {
            usuario *Usuario
            err     error
        }{usuario, err}
    }

    // Lanzar al primero, hedge a los demás si es necesario:
    go lanzar(réplicas[0])
    // [... mismo patrón del Ejercicio 21.5.1 ...]
}
```

**Restricciones:** Implementar para un repositorio con 3 réplicas de lectura.
El hedge debe seleccionar réplicas distintas (no repetir la misma).
Verificar que el hedging reduce el p99 de queries sin aumentar el p50.

**Pista:** Para seleccionar réplicas distintas: shuffle de la lista de réplicas
antes de asignar. La primera request va a `réplicas[0]`, el primer hedge a
`réplicas[1]`, el segundo hedge a `réplicas[2]`. Sin shuffle, siempre la
misma réplica recibe las requests lentas — potencialmente sobreargándola.

---

### Ejercicio 21.5.4 — Leer: cuándo el hedging hace daño

**Tipo: Diagnosticar**

Un equipo habilitó hedging en su sistema de pagos y observó:

```
Antes del hedging:
  Tráfico al proveedor de pagos: 1000 req/min
  Tasa de éxito: 99.2%
  Latencia p99: 450ms

Después del hedging (hedge delay = p90 = 200ms):
  Tráfico al proveedor de pagos: 1800 req/min  (+80%)
  Tasa de éxito: 97.1%  (bajó!)
  Latencia p99: 380ms   (mejoró algo)
  Duplicados en base de datos: 23 en 1 semana
```

**Preguntas:**

1. ¿Por qué el tráfico subió un 80% si el hedge delay es el p90?
2. ¿Por qué bajó la tasa de éxito?
3. ¿Qué causó los 23 duplicados?
4. ¿Es el hedging apropiado para operaciones de pago?
5. ¿Qué cambio haría el hedging seguro en este contexto?

**Pista:** El hedge al p90 activa hedges para el 10% de requests, no el 80%.
El 80% de aumento sugiere que el hedge delay está mal calibrado o que hay
un problema de amplificación (un request hedged genera múltiples downstream requests).
Los duplicados: el proveedor de pagos no es idempotente — dos requests con los
mismos datos crean dos cargos. La solución: idempotency keys que el proveedor
de pagos soporte, permitiendo que el segundo request sea un no-op si el primero
ya se procesó.

---

### Ejercicio 21.5.5 — Combinar hedging con circuit breaker

El hedging y el circuit breaker se complementan:

```
Circuit breaker: evita llamar a un servicio que claramente está fallando
Hedging: reduce la latencia para el tail de requests lentos

Interacción:
  - Si el circuit breaker está Abierto, no hacer hedges (el servicio está down)
  - Si el circuit breaker está Cerrado, usar hedging normalmente
  - Si el circuit breaker está HalfOpen, NO hacer hedges (las requests de prueba
    deben llegar al servicio, no ser evitadas con el resultado de otra)
```

**Restricciones:** Implementar el cliente que combina ambos patrones correctamente.
El estado del circuit breaker debe afectar el comportamiento del hedging.

**Pista:** La composición es más compleja que usar los dos independientemente.
Si el circuit breaker está HalfOpen y el hedge gana con la segunda request,
¿cuenta eso como un éxito para el breaker? La respuesta: sí — lo que importa
es que la operación tuvo éxito, independientemente de qué request "ganó".
El breaker debe contar éxitos/fallos basándose en el resultado final de la
operación compuesta (hedging + breaker), no en cada request individual.

---

## Sección 21.6 — Degradación Elegante: Funcionar con Menos

### Ejercicio 21.6.1 — Feature flags para degradación gradual

La degradación elegante requiere poder desactivar funcionalidades
individualmente sin reiniciar el sistema:

```go
type FeatureFlags struct {
    flags sync.Map  // string → bool, actualizable en runtime
}

func (ff *FeatureFlags) Habilitado(feature string) bool {
    v, ok := ff.flags.Load(feature)
    if !ok { return true }  // habilitado por defecto
    return v.(bool)
}

func (ff *FeatureFlags) Deshabilitar(feature string) {
    ff.flags.Store(feature, false)
}

// Uso:
func manejarRequest(w http.ResponseWriter, r *http.Request) {
    usuario := obtenerUsuario(r)

    if flags.Habilitado("recomendaciones") {
        recomendaciones := obtenerRecomendaciones(usuario)
        renderRecomendaciones(w, recomendaciones)
    }

    if flags.Habilitado("analytics") {
        registrarEvento(usuario, "page_view")
    }

    renderContenidoPrincipal(w, usuario)
}
```

**Restricciones:** Implementar un sistema de feature flags que:
- Se actualiza via HTTP (`POST /admin/flags` con JSON)
- Persiste los cambios en Redis (para que sobrevivan reinicios)
- Permite habilitación gradual (ej: solo para el 10% de usuarios)
- Expone el estado de todos los flags como métrica

**Pista:** La habilitación gradual requiere una función determinista:
el mismo usuario siempre debe ver el flag habilitado o deshabilitado.
`hash(userID + flagName) % 100 < porcentaje` da un resultado determinista
que distribuye el 10% de usuarios de forma consistente.

---

### Ejercicio 21.6.2 — Caché como degradación de BD

Cuando la BD está lenta o no disponible, una caché con datos ligeramente
desactualizados es mejor que un error:

```go
type ServicioConCache struct {
    bd     *bd.Client
    cache  *Cache
    breaker *CircuitBreaker
}

func (s *ServicioConCache) ObtenerProducto(ctx context.Context, id string) (*Producto, error) {
    // Intentar caché primero (siempre disponible):
    if producto, ok := s.cache.Get(id); ok {
        return producto.(*Producto), nil
    }

    // Caché miss: intentar BD con circuit breaker:
    var producto *Producto
    err := s.breaker.Call(func() error {
        var e error
        producto, e = s.bd.ObtenerProducto(ctx, id)
        return e
    })

    if err == nil {
        // Guardar en caché con TTL apropiado:
        s.cache.Set(id, producto, 5*time.Minute)
        return producto, nil
    }

    // BD no disponible: retornar caché expirada si existe:
    if stale, ok := s.cache.GetStale(id); ok {
        // Retornar datos desactualizados con advertencia:
        return stale.(*Producto), ErrDatosDesactualizados
    }

    return nil, fmt.Errorf("servicio no disponible: %w", err)
}
```

**Restricciones:** Implementar `Cache.GetStale` que retorna datos aunque hayan
expirado su TTL normal. La caché debe distinguir entre "expirado pero disponible"
y "nunca fue cacheado". Añadir métricas para cada ruta (cache hit, cache stale, BD, error).

**Pista:** Para implementar "stale" sin duplicar el almacenamiento: guardar
la entrada con el TTL normal más un "grace period" más largo. El TTL normal
determina si la entrada es "fresca". El grace period determina si todavía está
disponible como stale. Ejemplo: TTL = 5min, grace = 30min. La entrada se
considera fresca los primeros 5min, stale entre 5min y 30min, y ausente después.

---

### Ejercicio 21.6.3 — Respuesta degradada: menos campos, misma estructura

Cuando un servicio dependiente falla, retornar una respuesta con campos
opcionales vacíos es mejor que fallar completamente:

```go
type PerfilUsuario struct {
    // Campos esenciales (desde BD principal — siempre disponibles):
    ID     string `json:"id"`
    Nombre string `json:"nombre"`
    Email  string `json:"email"`

    // Campos opcionales (desde servicios externos — pueden fallar):
    Foto          *string  `json:"foto,omitempty"`           // servicio de imágenes
    Recomendaciones []Item `json:"recomendaciones,omitempty"` // ML service
    Notificaciones  int    `json:"notificaciones,omitempty"`  // servicio de notifs
}

func (s *Servicio) ObtenerPerfil(ctx context.Context, id string) (*PerfilUsuario, error) {
    // Los campos esenciales deben obtenerse primero — si fallan, fallar todo:
    perfil, err := s.bd.ObtenerPerfil(ctx, id)
    if err != nil { return nil, err }

    // Los campos opcionales se obtienen en paralelo — si fallan, omitir:
    var wg sync.WaitGroup
    wg.Add(3)

    go func() {
        defer wg.Done()
        foto, err := s.imagenes.ObtenerFoto(ctx, id)
        if err == nil { perfil.Foto = &foto }
        // Si falla: perfil.Foto queda nil → omitempty lo excluye del JSON
    }()
    // ... similarmente para Recomendaciones y Notificaciones

    wg.Wait()
    return perfil, nil
}
```

**Restricciones:** Implementar con un timeout corto para los campos opcionales
(si no responden en 200ms, continuar sin ellos). Añadir métricas que indiquen
qué porcentaje de perfiles se devuelven con cada campo presente.

**Pista:** El timeout para los campos opcionales debe ser mucho más corto
que el timeout del request completo. Si el request tiene 2s de timeout y
los campos opcionales tardan hasta 200ms, el perfil "degradado" se retorna
en < 300ms (200ms de espera + overhead). Sin timeout en los opcionales,
un servicio lento puede hacer que el perfil completo tarde tanto como si
los campos fueran obligatorios.

---

### Ejercicio 21.6.4 — Shed load por prioridad de usuario

Cuando el sistema está saturado, no todos los usuarios son iguales:

```go
type NivelServicio int

const (
    Premium  NivelServicio = 3  // siempre servir
    Standard NivelServicio = 2  // servir si hay capacidad
    Free     NivelServicio = 1  // servir solo con capacidad sobrante
)

type LoadShedder struct {
    carga         *CargaActual    // métricas de carga del sistema
    umbralWarning float64         // ej: 0.7 (70% de capacidad)
    umbralCritico float64         // ej: 0.9 (90% de capacidad)
}

func (ls *LoadShedder) DebeAtender(usuario *Usuario) bool {
    carga := ls.carga.Actual()

    switch {
    case carga < ls.umbralWarning:
        return true  // todos son atendidos

    case carga < ls.umbralCritico:
        // Solo standard y premium:
        return usuario.Nivel >= Standard

    default:
        // Solo premium:
        return usuario.Nivel == Premium
    }
}
```

**Restricciones:** El shed de carga debe ser:
- Proporcional (rechazar gradualmente, no de golpe)
- Medible (métricas de cuánto tráfico se shed por nivel)
- Reversible (al bajar la carga, volver a aceptar requests gradualmente)

**Pista:** La reversibilidad es importante: si el load shedder rechaza hasta que
la carga baja al 70%, cuando la carga baja al 69% hay un spike de requests
acumuladas que pueden volver a subir la carga al 90%. Solución: hysteresis —
el umbral para empezar a aceptar de nuevo (65%) es menor que el umbral para
empezar a rechazar (70%).

---

### Ejercicio 21.6.5 — El degradation budget

Similar al error budget del SLO (Cap.17 §17.6.3), el degradation budget
mide cuánta degradación es aceptable:

```
SLO normal:     100% de usuarios reciben el perfil completo (con recomendaciones)
SLO degradado:  95% de usuarios reciben perfil completo, 5% sin recomendaciones
Budget mensual: 5% × 30 días × 24h = 36 horas de degradación aceptable

Si las recomendaciones fallan durante 2 horas/día (normal):
  Burn rate = 2h/día ÷ (36h/mes ÷ 30días) = 2 ÷ 1.2 = 1.67× el budget normal
  → En 18 días el budget mensual se agota → alertar al equipo de ML
```

**Restricciones:** Implementar el degradation budget tracker para el sistema
del Ejercicio 21.6.3. El tracker debe:
- Registrar cuándo cada campo opcional está ausente
- Calcular el "burn rate" del degradation budget en tiempo real
- Alertar cuando el burn rate supera 2× el esperado

**Pista:** El degradation budget es una extensión natural del error budget de SLO.
La diferencia: el error budget cuenta requests fallidas; el degradation budget
cuenta requests "degradadas" (exitosas pero con menos información).
Para el cálculo: `burn_rate = (requests_degradadas_últimas_1h / total_requests_últimas_1h) / (budget_mensual / horas_en_mes)`.

---

## Sección 21.7 — Componer: El Sistema Resiliente Completo

### Ejercicio 21.7.1 — Orden de los patrones en la cadena de llamada

Cuando se componen múltiples patrones de resiliencia, el orden importa:

```
Opción A: Retry → Circuit Breaker → Timeout → Función
  Si la función falla:
    1. Timeout la cancela a los N segundos
    2. Circuit Breaker registra el fallo
    3. Retry intenta de nuevo
  Problema: si el CB abre, Retry sigue intentando → Retry intenta CB.Call → CB retorna error → Retry lo ve como error retryable

Opción B: Circuit Breaker → Retry → Timeout → Función
  Si la función falla:
    1. Timeout la cancela
    2. Retry intenta de nuevo (mismo intento le cuenta al CB)
    3. Si el CB abre, Retry ve ErrCircuitoAbierto → debe tratarlo como NOT retryable
  Más correcto para la mayoría de casos

Opción C: Circuit Breaker → Timeout → Hedging → Función
  El hedging reduce el tail latency antes de que el CB vea el fallo
  Apropiado para servicios de lectura con tail latency alta
```

**Restricciones:** Implementar las tres opciones y comparar su comportamiento
con un servicio externo simulado que:
- Falla el 20% de las requests
- El 5% de las requests tardan 10× lo normal (tail latency)
- Se recupera después de 30 segundos de fallo sostenido

**Pista:** La Opción B es el orden correcto para la mayoría de casos.
La clave: `ErrCircuitoAbierto` debe ser clasificado como NOT retryable
por la función `EsRetryable` del Ejercicio 21.2.2. Si el CB está abierto,
reintentar inmediatamente es inútil — el CB seguirá abierto.

---

### Ejercicio 21.7.2 — El cliente resiliente completo

Implementa un cliente HTTP que compone todos los patrones del capítulo:

```go
type ClienteResiliente struct {
    httpClient  *http.Client
    breaker     *CircuitBreakerLatencia   // Sección 21.1.3
    reintentador *ReintentadorBackoff     // Sección 21.2.1
    bulkhead    *BulkheadSemaforo         // Sección 21.4.3
    hedger      *ClienteHedged            // Sección 21.5.1
    timeout     time.Duration             // Sección 21.3
}

func (c *ClienteResiliente) Get(ctx context.Context, url string) (*http.Response, error) {
    // 1. Bulkhead: limitar la concurrencia total
    return c.bulkhead.Ejecutar(ctx, func() error {
        // 2. Circuit breaker: fallar rápido si el servicio está down
        return c.breaker.Call(func() error {
            // 3. Retry: reintentar fallos transitorios
            return c.reintentador.Ejecutar(ctx, func() error {
                // 4. Timeout: cada intento tiene su propio timeout
                reqCtx, cancel := context.WithTimeout(ctx, c.timeout)
                defer cancel()

                // 5. Hedging: reducir tail latency
                _, err := c.hedger.Get(reqCtx, url)
                return err
            })
        })
    })
}
```

**Restricciones:** La composición debe manejar correctamente:
- La cancelación del contexto padre se propaga a todos los niveles
- Los errores del bulkhead (lleno) no se reintentan
- Los errores del circuit breaker (abierto) no se reintentan
- Solo los errores de red transitorios se reintentan

**Pista:** Cada patrón debe distinguir entre errores que son "suyos" (bulkhead lleno,
CB abierto) y errores de la operación subyacente. Solo los segundos deben
propagarse hacia arriba para que el retry los maneje. Una forma limpia:
definir sentinel errors (`ErrBulkheadLleno`, `ErrCircuitoAbierto`) y en
`EsRetryable`, retornar false para estos específicamente.

---

### Ejercicio 21.7.3 — Testing de resiliencia: chaos engineering simplificado

Implementa un proxy de caos para testear que el cliente resiliente
se comporta correctamente ante cada tipo de fallo:

```go
type ProxyCaos struct {
    real          http.Handler
    config        CaosConfig
    mu            sync.RWMutex
}

type CaosConfig struct {
    TasaError     float64        // % de requests que fallan con 500
    TasaTimeout   float64        // % de requests que tardan muchísimo
    TasaLatencia  float64        // % de requests con latencia extra
    LatenciaExtra time.Duration
    Activo        bool
}

func (p *ProxyCaos) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    p.mu.RLock()
    config := p.config
    p.mu.RUnlock()

    if !config.Activo {
        p.real.ServeHTTP(w, r)
        return
    }

    rng := rand.Float64()
    switch {
    case rng < config.TasaError:
        http.Error(w, "chaos error", 500)
    case rng < config.TasaError+config.TasaTimeout:
        time.Sleep(30 * time.Second)  // simular timeout
    case rng < config.TasaError+config.TasaTimeout+config.TasaLatencia:
        time.Sleep(config.LatenciaExtra)
        p.real.ServeHTTP(w, r)
    default:
        p.real.ServeHTTP(w, r)
    }
}
```

**Restricciones:** Escribir tests que verifiquen, usando el ProxyCaos, que:
1. Con 20% de error rate: el circuit breaker abre después de N fallos
2. Con 5% de tail latency: el hedging reduce el p99 observado
3. Con servicio totalmente caído: el bulkhead rechaza rápido (< 10ms por request)
4. Después del caos: el sistema se recupera sin intervención manual

**Pista:** Los tests de resiliencia deben ser más lentos que los unit tests
(el circuit breaker necesita tiempo para acumular fallos y para el half-open).
Usar timeouts de test generosos (`t.Setenv` o `testing.T.Deadline()`) y
configurar los parámetros del breaker para el test (umbral bajo, tiempo de espera corto).

---

### Ejercicio 21.7.4 — Runbook de resiliencia

Documenta el runbook para operar el sistema resiliente del Ejercicio 21.7.2:

```markdown
# Runbook: Sistema con Patrones de Resiliencia

## Señales de que los patrones están funcionando correctamente
- Circuit breaker: `circuit_breaker_state` oscila entre 0 (cerrado) y 1 (abierto)
  mientras el servicio externo está degradado, luego vuelve a 0 cuando se recupera.
- Hedging: `hedge_activations_total` aumenta cuando hay tail latency, y la
  latencia p99 observada baja de forma correspondiente.
- Retry: `retry_attempts_total` aumenta durante fallos transitorios,
  pero la tasa de éxito final se mantiene alta.

## Señales de que algo está mal

### El circuit breaker no abre cuando debería
Posible causa: umbral demasiado alto
Diagnóstico: ver `circuit_breaker_failures_total` vs `umbral_configurado`
Acción: reducir umbral vía feature flag

### El circuit breaker flappea
[... continuar el runbook ...]
```

**Restricciones:** El runbook debe cubrir los 5 escenarios más probables
de mal funcionamiento de los patrones de resiliencia, con:
- Cómo detectarlo (qué métrica o log)
- Cómo diagnosticarlo (qué herramienta usar)
- Cómo mitigarlo (acción inmediata)
- Cómo prevenirlo en el futuro (mejora a largo plazo)

---

### Ejercicio 21.7.5 — Diseño integral: sistema de checkout resiliente

Diseña e implementa el sistema de checkout de un e-commerce con todos los
patrones de resiliencia del capítulo:

```
Dependencias del checkout:
  1. Servicio de inventario      [latencia p99: 20ms, disponibilidad: 99.5%]
  2. Servicio de pagos           [latencia p99: 200ms, disponibilidad: 99.9%]
  3. Servicio de shipping        [latencia p99: 50ms, disponibilidad: 98%]
  4. Servicio de notificaciones  [latencia p99: 100ms, disponibilidad: 95%]
  5. BD de pedidos               [latencia p99: 10ms, disponibilidad: 99.99%]

SLO del checkout: 99.5% de requests exitosas, p99 < 500ms
```

**Restricciones:** Para cada dependencia, diseñar y justificar:
- ¿Circuit breaker? ¿Con qué umbrales?
- ¿Retry? ¿Cuántos intentos? ¿Qué errores son retryables?
- ¿Timeout? ¿Cuánto?
- ¿Hedging? ¿Con qué delay?
- ¿Degradación elegante? ¿Qué pasa si falla?

**Pista:** El servicio de notificaciones (disponibilidad 95%) es claramente
un candidato para degradación elegante — el pedido debe completarse aunque
el email de confirmación falle. El servicio de pagos (disponibilidad 99.9%)
requiere idempotency keys para el retry. El servicio de shipping puede
hacer fallback a "envío calculado al recoger el pedido" si falla.
La BD de pedidos (99.99%) es el único servicio que, si falla, debe
fallar el checkout completo — no hay degradación posible sin persistir el pedido.

---

## Resumen del capítulo y de la Parte 3b

**Los seis patrones de resiliencia y cuándo usar cada uno:**

```
Circuit Breaker:
  Úsalo cuando: una dependencia externa puede fallar por períodos sostenidos.
  No lo uses cuando: la operación es local (memoria, CPU) — el overhead es innecesario.

Retry con Backoff:
  Úsalo cuando: los fallos son transitorios (timeouts de red, spikes de carga).
  No lo uses cuando: los fallos son permanentes (validación, not found) o cuando
  la operación no es idempotente sin mecanismo adicional.

Timeout Hierarchy:
  Siempre: cada operación de red debe tener un timeout. Sin excepción.
  El timeout del padre debe ser mayor que el del hijo.

Bulkhead:
  Úsalo cuando: tienes múltiples dependencias externas y el fallo de una
  no debe consumir recursos para las otras.
  El tamaño del pool determina el aislamiento — muy pequeño y es un cuello de botella.

Hedging:
  Úsalo cuando: tienes tail latency alta en operaciones idempotentes de lectura.
  No lo uses cuando: la operación tiene efectos secundarios que no puedes deduplicar.

Degradación Elegante:
  Diseña siempre con degradación en mente: ¿qué pasa si X falla?
  La respuesta "fallo completo" rara vez es la correcta para funcionalidades opcionales.
```

**El principio unificador de la Parte 3b:**

> La observabilidad (Cap.18) te dice cuándo el sistema está fallando.
> El debugging (Cap.19) te dice por qué está fallando.
> El code review (Cap.20) previene que los bugs lleguen a producción.
> La resiliencia (Cap.21) hace que el sistema funcione aunque algo falle.
>
> Ninguno de los cuatro es suficiente solo. Un sistema resiliente sin
> observabilidad no sabe que está degradado. Un sistema observable sin
> resiliencia colapsa bajo la primera falla. El objetivo es los cuatro juntos.
