# Guía de Ejercicios — Cap.23: Arquitectura de Sistemas Distribuidos

> Lenguaje principal: **Go**. Los ejercicios usan gRPC, canales como proxies
> de red, y herramientas reales (etcd, Redis, PostgreSQL) donde aplica.
>
> Este es el capítulo final del repositorio. Su propósito es conectar
> todo lo anterior con los patrones que definen los sistemas distribuidos
> que se construyen hoy — microservicios, event sourcing, stream processing.
>
> Los ejercicios son los más grandes del repositorio: algunos son sistemas
> completos de 300–500 líneas. No hay atajos para esto.

---

## Lo que conecta este capítulo con el anterior

El Cap.22 explicó los **fundamentos**: por qué los sistemas distribuidos son difíciles
(fallos parciales, relojes no confiables, CAP theorem).

Este capítulo explica los **patrones** que la industria desarrolló para vivir
con esas dificultades:

```
Problema fundamental          →  Patrón arquitectónico
────────────────────────────────────────────────────────────
Los servicios fallan          →  Service mesh + circuit breaker
El estado se distribuye       →  Event sourcing + CQRS
La coordinación es cara       →  Saga pattern (sin 2PC)
El orden de mensajes importa  →  Kafka + consumers groups
La carga es variable          →  Backpressure + shedding de carga
El debugging es difícil       →  Distributed tracing (ya en Cap.18)
```

---

## Tabla de contenidos

- [Sección 23.1 — Service mesh y comunicación entre servicios](#sección-231--service-mesh-y-comunicación-entre-servicios)
- [Sección 23.2 — Event sourcing: el estado como secuencia de eventos](#sección-232--event-sourcing-el-estado-como-secuencia-de-eventos)
- [Sección 23.3 — CQRS: separar escritura de lectura](#sección-233--cqrs-separar-escritura-de-lectura)
- [Sección 23.4 — Stream processing: computación sobre eventos](#sección-234--stream-processing-computación-sobre-eventos)
- [Sección 23.5 — Coordinación distribuida con etcd](#sección-235--coordinación-distribuida-con-etcd)
- [Sección 23.6 — Particionamiento y consistent hashing](#sección-236--particionamiento-y-consistent-hashing)
- [Sección 23.7 — El sistema completo: integrando todo](#sección-237--el-sistema-completo-integrando-todo)

---

## Sección 23.1 — Service Mesh y Comunicación entre Servicios

Un service mesh abstrae la comunicación entre microservicios:
en lugar de que cada servicio implemente circuit breakers, retries, y tracing,
un proxy sidecar (Envoy, Linkerd) lo hace automáticamente.

```
Sin service mesh:
  Servicio A [código de retry] → red → Servicio B [código de rate limiting]
  Cada servicio implementa lógica de resiliencia → duplicación, inconsistencia

Con service mesh:
  Servicio A → [Envoy proxy] → red → [Envoy proxy] → Servicio B
  Los proxies manejan retry, circuit breaker, tracing, mTLS
  Los servicios solo implementan lógica de negocio
```

### Ejercicio 23.1.1 — Implementar un proxy sidecar simplificado

Sin usar Envoy (demasiado complejo para un ejercicio), implementa un
proxy TCP en Go que añade resiliencia a cualquier servicio:

```go
type SidecarProxy struct {
    listenAddr  string        // donde escucha el proxy (ej: :8080)
    upstreamAddr string       // donde está el servicio real (ej: :8081)
    breaker     *CircuitBreaker   // del Cap.21
    limiter     *RateLimiter      // del Cap.17
    tracer      trace.Tracer      // del Cap.18
}

func (p *SidecarProxy) Iniciar(ctx context.Context) error {
    ln, err := net.Listen("tcp", p.listenAddr)
    if err != nil { return err }

    for {
        conn, err := ln.Accept()
        if err != nil { return err }
        go p.manejarConexion(ctx, conn)
    }
}

func (p *SidecarProxy) manejarConexion(ctx context.Context, clientConn net.Conn) {
    defer clientConn.Close()

    // Rate limiting:
    if !p.limiter.Permitir(ctx, extraerClienteIP(clientConn)) {
        escribirError(clientConn, 429, "rate limit exceeded")
        return
    }

    // Circuit breaker:
    err := p.breaker.Call(func() error {
        upstreamConn, err := net.Dial("tcp", p.upstreamAddr)
        if err != nil { return err }
        defer upstreamConn.Close()

        // Proxy bidireccional:
        errCh := make(chan error, 2)
        go func() { _, e := io.Copy(upstreamConn, clientConn); errCh <- e }()
        go func() { _, e := io.Copy(clientConn, upstreamConn); errCh <- e }()
        <-errCh
        return nil
    })

    if errors.Is(err, ErrCircuitoAbierto) {
        escribirError(clientConn, 503, "service unavailable")
    }
}
```

**Restricciones:**
- El proxy debe ser transparente: el servicio real no sabe que existe
- Añadir métricas: latencia, tasa de error, estado del circuit breaker
- El proxy debe poder reiniciarse sin perder conexiones activas (graceful restart)

**Pista:** El proxy bidireccional usa dos goroutines que copian en paralelo.
Para el graceful restart: usar `SO_REUSEPORT` o pasar el file descriptor del
listener al proceso nuevo antes de apagar el viejo. En Kubernetes, esto no es
necesario (los pods se reemplazan uno a uno).

---

### Ejercicio 23.1.2 — Service discovery: encontrar servicios dinámicamente

En un entorno de microservicios, las instancias de un servicio van y vienen.
El service discovery permite encontrarlas:

```go
// Registro de servicios en etcd:
type RegistroServicios struct {
    cliente *clientv3.Client
}

func (r *RegistroServicios) Registrar(ctx context.Context, servicio, instanceID, addr string) error {
    // Crear un lease con TTL (si la instancia muere, el registro expira):
    lease, err := r.cliente.Grant(ctx, 10)  // TTL de 10 segundos
    if err != nil { return err }

    key := fmt.Sprintf("/servicios/%s/%s", servicio, instanceID)
    _, err = r.cliente.Put(ctx, key, addr, clientv3.WithLease(lease.ID))

    // Renovar el lease periódicamente (heartbeat):
    go r.renewLease(ctx, lease.ID)
    return err
}

// Descubrir instancias de un servicio:
func (r *RegistroServicios) Descubrir(ctx context.Context, servicio string) ([]string, error) {
    prefix := fmt.Sprintf("/servicios/%s/", servicio)
    resp, err := r.cliente.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil { return nil, err }

    addrs := make([]string, len(resp.Kvs))
    for i, kv := range resp.Kvs {
        addrs[i] = string(kv.Value)
    }
    return addrs, nil
}
```

**Restricciones:**
- Implementar un cliente HTTP que use el registro para distribuir carga (round-robin)
- Si una instancia muere (TTL expira), el cliente debe dejar de enviarle tráfico
- Implementar health checking: el cliente descarta instancias que no pasan el healthcheck

**Pista:** Watch en etcd permite recibir notificaciones cuando las instancias
se registran o desregistran: `r.cliente.Watch(ctx, prefix, clientv3.WithPrefix())`.
El cliente puede mantener una lista actualizada de instancias sin polling.

---

### Ejercicio 23.1.3 — Load balancing: distribuir la carga inteligentemente

Round-robin es simple pero no considera la carga de cada instancia:

```go
// Least connections: enviar al servidor con menos conexiones activas
type LeastConnLB struct {
    mu        sync.RWMutex
    servidores map[string]*InfoServidor
}

type InfoServidor struct {
    addr        string
    conexiones  atomic.Int64
    latenciaP99 atomic.Int64  // en microsegundos
}

func (lb *LeastConnLB) Seleccionar() *InfoServidor {
    lb.mu.RLock()
    defer lb.mu.RUnlock()

    var mejor *InfoServidor
    for _, srv := range lb.servidores {
        if mejor == nil || srv.conexiones.Load() < mejor.conexiones.Load() {
            mejor = srv
        }
    }
    return mejor
}

// Power of Two Choices: seleccionar 2 aleatorios, tomar el menos cargado
func (lb *LeastConnLB) SeleccionarP2C() *InfoServidor {
    // Seleccionar 2 servidores aleatoriamente, retornar el con menos conexiones
    // P2C da casi los mismos resultados que least-connections global
    // pero sin el bottleneck del lock global
}
```

**Restricciones:** Implementar y comparar los tres algoritmos bajo carga:
1. Round-robin
2. Least connections
3. Power of Two Choices (P2C)

Simular un servidor "lento" (10× la latencia normal) y medir cómo cada
algoritmo distribuye la carga ante esa instancia.

**Pista:** P2C (Power of Two Choices) es el algoritmo de Mitzenmacher (2001).
La intuición: elegir el mejor de 2 aleatorios es casi tan bueno como el mejor de N,
pero requiere solo 2 comparaciones en lugar de N. Usado por Nginx, HAProxy, y Envoy.
El algoritmo reduce drásticamente la probabilidad de sobrecargar una instancia
comparado con round-robin puro.

---

### Ejercicio 23.1.4 — mTLS: autenticación mutua entre servicios

En un service mesh, los servicios se autentican mutuamente con TLS:

```go
// Generar certificados para el servicio:
func generarCertificado(servicio string) (*tls.Certificate, error) {
    priv, _ := rsa.GenerateKey(rand.Reader, 2048)
    template := &x509.Certificate{
        SerialNumber: big.NewInt(1),
        Subject: pkix.Name{
            CommonName:   servicio,
            Organization: []string{"mi-empresa.internal"},
        },
        DNSNames: []string{
            servicio,
            servicio + ".default.svc.cluster.local",
        },
        NotBefore: time.Now(),
        NotAfter:  time.Now().Add(24 * time.Hour),
        KeyUsage:  x509.KeyUsageDigitalSignature,
        ExtKeyUsage: []x509.ExtKeyUsage{
            x509.ExtKeyUsageClientAuth,
            x509.ExtKeyUsageServerAuth,
        },
    }
    // ...
}
```

**Restricciones:** Implementar un servidor y cliente gRPC con mTLS donde:
- Cada servicio tiene su propio certificado firmado por una CA interna
- El servidor verifica que el cliente tiene un certificado de la CA correcta
- El cliente verifica que el servidor tiene el CN correcto (`servicio-b.internal`)
- Los certificados rotan automáticamente cada 24 horas sin downtime

**Pista:** La rotación de certificados sin downtime requiere que el servidor
acepte tanto el certificado viejo como el nuevo durante un período de transición.
`tls.Config.GetCertificate` permite cambiar el certificado por request —
útil para la rotación. Para la CA interna en producción, usar Vault o cert-manager.

---

### Ejercicio 23.1.5 — Leer: diagnosticar latencia entre servicios

**Tipo: Diagnosticar**

Este trace de Jaeger muestra un request lento entre tres servicios:

```
GET /api/checkout [1,240ms total]
│
├─ [0ms] gateway → order-service [1,230ms]
│   │
│   ├─ [0ms] order-service: validar request [2ms]
│   │
│   ├─ [2ms] order-service → inventory-service [15ms]
│   │   └─ inventory-service: verificar stock [13ms]
│   │
│   ├─ [17ms] order-service → payment-service [1,200ms] ← !!
│   │   ├─ [17ms] payment-service: preparar request [3ms]
│   │   ├─ [20ms] payment-service → banco-externo [8ms]
│   │   ├─ [28ms] payment-service: procesar respuesta [2ms]
│   │   └─ [30ms] payment-service: ESPERANDO... 1,170ms ← ??
│   │   └─ [1,200ms] payment-service: enviar respuesta a order-service
│   │
│   └─ [1,217ms] order-service: guardar pedido [13ms]
│
└─ [1,230ms] gateway: responder al cliente [10ms]
```

**Preguntas:**

1. ¿Dónde están los 1,170ms que `payment-service` pasa "esperando"?
2. Esto no está en los spans de Jaeger — ¿por qué?
3. ¿Qué tipo de instrumentación falta para visibilizar este tiempo?
4. Propón tres hipótesis para los 1,170ms.
5. ¿Qué cambio en el código de `payment-service` añadiría visibilidad?

**Pista:** El tiempo no capturado en spans es típicamente: (1) tiempo esperando
en una queue o semáforo interno, (2) tiempo en el garbage collector, (3) tiempo
esperando un lock de base de datos. Para visibilizarlo: añadir spans alrededor
de cada operación interna, incluyendo "esperando conexión de BD" y "esperando
semáforo". El GC se puede ver en el execution trace de Go (Cap.18 §18.3.5).

---

## Sección 23.2 — Event Sourcing: el Estado como Secuencia de Eventos

```
Base de datos tradicional:
  Estado actual: { saldo: 150, nombre: "Alice" }
  Historia: perdida (la última escritura sobreescribe)

Event sourcing:
  Log de eventos: [
    { tipo: "CuentaCreada", saldo_inicial: 0 },
    { tipo: "DepósitoRealizado", cantidad: 200 },
    { tipo: "RetiroRealizado", cantidad: 50 },
  ]
  Estado actual: calcular aplicando todos los eventos en orden
  Historia: completa y auditable
```

### Ejercicio 23.2.1 — Implementar el event store básico

```go
type Evento struct {
    ID          string
    AggregateID string      // a qué entidad pertenece este evento
    Tipo        string      // ej: "CuentaCreada", "DepósitoRealizado"
    Datos       interface{}
    Versión     int64       // número secuencial por aggregate
    Timestamp   time.Time
}

type EventStore struct {
    mu     sync.RWMutex
    eventos map[string][]Evento  // aggregateID → lista de eventos ordenados
}

func (es *EventStore) Guardar(ctx context.Context, aggregateID string, 
    nuevosEventos []Evento, versionEsperada int64) error {
    
    es.mu.Lock()
    defer es.mu.Unlock()
    
    historiaActual := es.eventos[aggregateID]
    
    // Optimistic concurrency control:
    // si la versión actual no es la esperada, hay un conflicto
    versionActual := int64(len(historiaActual))
    if versionActual != versionEsperada {
        return ErrConflictoVersiones{
            Esperada: versionEsperada,
            Actual:   versionActual,
        }
    }
    
    // Asignar versiones y guardar:
    for i, evento := range nuevosEventos {
        evento.Versión = versionActual + int64(i) + 1
        historiaActual = append(historiaActual, evento)
    }
    es.eventos[aggregateID] = historiaActual
    return nil
}

func (es *EventStore) Cargar(ctx context.Context, aggregateID string) ([]Evento, error) {
    es.mu.RLock()
    defer es.mu.RUnlock()
    return es.eventos[aggregateID], nil
}
```

**Restricciones:**
- Implementar el aggregate de `CuentaBancaria` que reconstruye su estado
  aplicando eventos desde el store
- Implementar `snapshot`: guardar el estado calculado cada N eventos para
  evitar reconstruir desde el principio cada vez
- Verificar con tests que el optimistic concurrency control previene actualizaciones perdidas

**Pista:** El optimistic concurrency control en event sourcing: cuando dos usuarios
modifican el mismo aggregate simultáneamente, solo uno puede guardar sus eventos
(el que tiene la versión correcta). El segundo recibe `ErrConflictoVersiones` y debe
releer el aggregate, aplicar su cambio de nuevo, e intentar guardar otra vez.
Esto es O(retry) pero garantiza que no se pierden actualizaciones silenciosamente.

---

### Ejercicio 23.2.2 — Proyecciones: derivar vistas del event log

```
El event log es la fuente de verdad.
Las proyecciones son vistas derivadas — pueden reconstruirse desde el log.

Event log de cuentas:
  { CuentaCreada: alice, 2020 }
  { Depósito: alice, 200 }
  { CuentaCreada: bob, 2021 }
  { Retiro: alice, 50 }
  { Depósito: bob, 100 }

Proyección "saldo actual":
  { alice: 150, bob: 100 }

Proyección "total por año de creación":
  { 2020: 150, 2021: 100 }

Proyección "historial de alice":
  [ Depósito(200), Retiro(50) ]
```

**Restricciones:** Implementar un sistema de proyecciones donde:
- Las proyecciones se construyen leyendo el event log desde el principio
- Las proyecciones se actualizan incrementalmente (no reconstruidas) cuando
  llegan nuevos eventos
- Si una proyección falla, puede reconstruirse desde cero sin afectar el event log
- Exponer las proyecciones como endpoints de lectura (para CQRS en la sección siguiente)

**Pista:** El punto de control (checkpoint) de una proyección: guardar hasta qué
evento procesó. Si la proyección falla y reinicia, retoma desde ese punto.
Sin checkpoint, reconstruye desde el principio (costoso pero correcto).
Las proyecciones son idempotentes — procesar el mismo evento dos veces produce
el mismo resultado (importante para el exactly-once en Kafka).

---

### Ejercicio 23.2.3 — Leer: las ventajas y costos del event sourcing

**Tipo: Analizar**

El equipo de un e-commerce está evaluando si migrar a event sourcing.
El sistema actual tiene:

```
Tabla de órdenes:
  id, user_id, status, total, created_at, updated_at

Las actualizaciones de status sobreescriben el valor anterior.
```

**Preguntas:**

1. ¿Qué información se pierde con el diseño actual que event sourcing capturaría?
2. Un cliente pregunta "¿por qué mi orden fue cancelada?". ¿Cómo respondes
   con el diseño actual vs con event sourcing?
3. ¿Qué ventaja tiene event sourcing para debugging de incidentes?
4. ¿Cuáles son los dos costos principales de event sourcing?
5. ¿Para qué tipos de dominio event sourcing tiene mayor ROI?

**Pista:** Event sourcing tiene mayor ROI en dominios donde la historia importa:
finanzas (auditoría), supply chain (trazabilidad), sistemas de reservas (¿quién
cambió qué y cuándo). Tiene menor ROI en dominios donde el estado actual es todo
lo que importa: contadores de visitas, temperatura actual de un sensor, posición
GPS en tiempo real. Los costos principales: (1) la complejidad de mantener proyecciones
actualizadas, (2) las queries sobre el estado histórico son más difíciles.

---

### Ejercicio 23.2.4 — Event sourcing con Kafka como event store

En producción, el event store suele ser Kafka (o Pulsar, o Kinesis):

```go
// Publicar un evento a Kafka:
func (es *EventStoreKafka) Guardar(ctx context.Context, evento Evento) error {
    msg := &kafka.Message{
        TopicPartition: kafka.TopicPartition{
            Topic:     &es.topicName,
            Partition: kafka.PartitionAny,
        },
        Key:   []byte(evento.AggregateID),   // misma key → misma partición → orden garantizado
        Value: serializar(evento),
        Headers: []kafka.Header{
            {Key: "tipo", Value: []byte(evento.Tipo)},
            {Key: "aggregate-version", Value: []byte(fmt.Sprintf("%d", evento.Versión))},
        },
    }
    return es.producer.Produce(msg, nil)
}

// Leer el historial de un aggregate:
// (más complejo — requiere leer desde el principio de la partición)
```

**Restricciones:** Implementar el event store sobre Kafka con:
- La misma partition key para eventos del mismo aggregate (garantiza orden)
- Compaction de Kafka para mantener el historial completo (log compaction)
- Un consumer group separado para cada proyección

**Pista:** En Kafka, usar el aggregateID como message key garantiza que todos
los eventos del mismo aggregate van a la misma partición — y por tanto están
ordenados dentro del aggregate. El log compaction de Kafka retiene al menos
el último mensaje por key — para event sourcing, NO usar compaction (se perderían
eventos históricos). En su lugar, configurar retención por tiempo o tamaño.

---

### Ejercicio 23.2.5 — Temporal queries: preguntas sobre el pasado

Una de las ventajas de event sourcing: responder preguntas temporales.

```go
// "¿Cuál era el saldo de Alice el 15 de enero de 2024?"
func (a *CuentaBancaria) EstadoEn(timestamp time.Time) EstadoCuenta {
    estado := EstadoCuenta{}
    for _, evento := range a.eventos {
        if evento.Timestamp.After(timestamp) {
            break  // no aplicar eventos después del timestamp pedido
        }
        estado = aplicar(estado, evento)
    }
    return estado
}

// "¿Cuándo fue el saldo de Alice mayor que $1000?"
func (a *CuentaBancaria) PeríodosConSaldoMayorQue(umbral float64) []Período {
    var períodos []Período
    estado := EstadoCuenta{}
    var inicio *time.Time

    for _, evento := range a.eventos {
        estadoAnterior := estado
        estado = aplicar(estado, evento)

        if estadoAnterior.Saldo <= umbral && estado.Saldo > umbral {
            t := evento.Timestamp
            inicio = &t
        } else if estadoAnterior.Saldo > umbral && estado.Saldo <= umbral && inicio != nil {
            períodos = append(períodos, Período{Inicio: *inicio, Fin: evento.Timestamp})
            inicio = nil
        }
    }
    return períodos
}
```

**Restricciones:** Implementar las dos queries y verificar que son correctas
con un test que incluye 100 eventos de depósito y retiro.

**Pista:** Las temporal queries son O(n) donde n es el número de eventos.
Para aggregates con millones de eventos, esto es lento. La optimización:
snapshots + binary search para encontrar el snapshot más cercano al timestamp
pedido, luego aplicar solo los eventos desde el snapshot hasta el timestamp.

---

## Sección 23.3 — CQRS: Separar Escritura de Lectura

```
CRUD tradicional (mismo modelo para leer y escribir):
  GET /orders → modelo de orden normalizado
  POST /orders → mismo modelo

CQRS (Command Query Responsibility Segregation):
  Command (escritura): "CrearOrden" → valida y genera eventos
  Query (lectura): "ObtenerOrdenPorID" → lee de una proyección optimizada para lectura

Ventaja: el modelo de lectura puede ser completamente diferente al de escritura.
  Write model: normalizado, consistente, usa event sourcing
  Read model: desnormalizado, rápido, puede ser Elasticsearch o Redis

Costo: eventual consistency entre el write model y el read model.
```

### Ejercicio 23.3.1 — Implementar el write side: comandos y validación

```go
// Un comando es una intención — puede fallar
type CrearOrdenCmd struct {
    UserID  string
    Items   []ItemOrden
    Método  string  // "tarjeta", "transferencia"
}

// El command handler valida y genera eventos
type OrdenCommandHandler struct {
    eventStore  *EventStore
    inventario  *InventarioService
}

func (h *OrdenCommandHandler) Manejar(ctx context.Context, cmd CrearOrdenCmd) error {
    // Validar:
    if len(cmd.Items) == 0 {
        return ErrOrdenSinItems
    }
    
    // Verificar invariantes de negocio:
    for _, item := range cmd.Items {
        disponible, err := h.inventario.Disponible(ctx, item.SKU, item.Cantidad)
        if err != nil { return err }
        if !disponible { return ErrStockInsuficiente{SKU: item.SKU} }
    }
    
    // Generar el evento:
    evento := Evento{
        AggregateID: nuevoID(),
        Tipo:        "OrdenCreada",
        Datos: OrdenCreadaDatos{
            UserID: cmd.UserID,
            Items:  cmd.Items,
            Total:  calcularTotal(cmd.Items),
        },
    }
    
    return h.eventStore.Guardar(ctx, evento.AggregateID, []Evento{evento}, 0)
}
```

**Restricciones:** Implementar el command handler con:
- Validación de negocio antes de generar eventos
- Idempotencia: si el mismo comando se envía dos veces, el segundo es no-op
- Los errores de validación son errores de dominio (no errores de infraestructura)

**Pista:** La idempotencia de comandos requiere una idempotency key que el cliente
genera y envía con el comando. El handler guarda las keys procesadas (en Redis
con TTL) y retorna el resultado anterior si la key ya fue procesada.

---

### Ejercicio 23.3.2 — Implementar el read side: proyecciones optimizadas

```go
// El read model está desnormalizado para responder queries rápidamente
type OrdenReadModel struct {
    ID          string
    UserNombre  string   // desnormalizado desde Users
    UserEmail   string   // desnormalizado desde Users
    Items       []ItemConPrecio
    Total       float64
    Status      string
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// La proyección mantiene el read model actualizado:
type OrdenProjection struct {
    store    map[string]*OrdenReadModel  // puede ser Redis, Elasticsearch, etc.
    userSvc  *UserService
}

func (p *OrdenProjection) ManejarEvento(ctx context.Context, evento Evento) error {
    switch evento.Tipo {
    case "OrdenCreada":
        datos := evento.Datos.(OrdenCreadaDatos)
        // Enriquecer con datos del usuario:
        usuario, _ := p.userSvc.ObtenerUsuario(ctx, datos.UserID)
        p.store[evento.AggregateID] = &OrdenReadModel{
            ID:         evento.AggregateID,
            UserNombre: usuario.Nombre,
            UserEmail:  usuario.Email,
            Items:      enriquecerItems(datos.Items),
            Total:      datos.Total,
            Status:     "pendiente",
            CreatedAt:  evento.Timestamp,
        }
    case "OrdenConfirmada":
        if modelo, ok := p.store[evento.AggregateID]; ok {
            modelo.Status = "confirmada"
            modelo.UpdatedAt = evento.Timestamp
        }
    }
    return nil
}
```

**Restricciones:**
- El read model debe poder reconstruirse desde cero procesando todos los eventos
- Implementar una query "órdenes del usuario U en los últimos 30 días" optimizada
- La proyección debe tolerar eventos fuera de orden (con vector clocks o versiones)

**Pista:** La query "órdenes del usuario U en los últimos 30 días" en el read model
puede ser O(1) si el store indexa por (userID, fecha). En Redis:
`ZADD orders:user:{userID} timestamp orderID`. Consultar con
`ZRANGEBYSCORE orders:user:{userID} hace30dias ahora`.

---

### Ejercicio 23.3.3 — Consistencia eventual entre write y read side

Cuando el write side acepta un comando, el read side puede tardar
milisegundos o segundos en actualizarse. El cliente necesita manejar esto:

```go
// Patrón: retornar el ID del aggregate y el número de versión
// El cliente puede poll hasta que el read side tiene esa versión

type ResultadoComando struct {
    AggregateID string
    Versión     int64
}

// El cliente consulta hasta que el read model tiene la versión esperada:
func (c *Cliente) EsperarProyección(ctx context.Context, id string, versionMin int64) (*OrdenReadModel, error) {
    ticker := time.NewTicker(50 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-ticker.C:
            modelo, err := c.readAPI.ObtenerOrden(ctx, id)
            if err == nil && modelo.Versión >= versionMin {
                return modelo, nil
            }
        }
    }
}
```

**Restricciones:** Implementar el sistema completo con el patrón de polling.
El cliente debe recibir el resultado actualizado en < 500ms en condiciones normales.
Añadir un mecanismo de webhook (notificación push) como alternativa al polling.

**Pista:** El webhook evita el polling: cuando la proyección se actualiza,
notifica al cliente con un HTTP POST. Para el webhook, el cliente necesita
exponer un endpoint de recepción — más complejo pero más eficiente que polling.
En interfaces de usuario modernas, WebSockets o Server-Sent Events reemplazan ambos.

---

### Ejercicio 23.3.4 — Leer: cuándo CQRS es demasiado

**Tipo: Analizar**

Un equipo está considerando CQRS para un sistema de gestión de contactos (CRM):

```
Operaciones del CRM:
  - Crear/editar contacto (nombre, email, teléfono, notas)
  - Buscar contactos por nombre o email
  - Ver historial de interacciones de un contacto
  - Exportar lista de contactos a CSV

Carga estimada:
  - 50 usuarios internos (no público)
  - ~200 operaciones/día
  - Latencia aceptable: < 2 segundos
```

**Preguntas:**

1. ¿CQRS añade valor para este sistema? ¿Por qué sí o por qué no?
2. ¿Event sourcing añade valor para este sistema?
3. ¿Cuál es el "wrong tool for the job" que indica este escenario?
4. ¿Para qué tamaño o tipo de sistema empezaría a tener sentido CQRS?
5. ¿Cómo puede el equipo evolucionar hacia CQRS más tarde sin reescribir?

**Pista:** CQRS y event sourcing son patrones para sistemas con:
(a) alta concurrencia que hace que el mismo modelo de lectura/escritura
sea un bottleneck, o (b) requisitos de auditoría y consultas históricas complejos.
Un CRM interno con 50 usuarios y 200 operaciones/día puede funcionar perfectamente
con PostgreSQL y un ORM simple durante años. La complejidad de CQRS en ese contexto
es deuda técnica desde el día uno. Martin Fowler llama esto "premature optimization"
a nivel arquitectónico.

---

### Ejercicio 23.3.5 — El "read model" como caché inteligente

CQRS no requiere event sourcing ni un store separado.
La forma más simple es usar el read model como una caché:

```go
// Write side: operaciones normales en PostgreSQL
// Read side: una vista materializada en Redis para queries frecuentes

type CachéReadModel struct {
    redis  *redis.Client
    db     *sql.DB
    ttl    time.Duration
}

func (c *CachéReadModel) ObtenerPerfil(ctx context.Context, userID string) (*Perfil, error) {
    // Intentar caché:
    key := "perfil:" + userID
    data, err := c.redis.Get(ctx, key).Bytes()
    if err == nil {
        return deserializar[Perfil](data), nil
    }

    // Cache miss: ir a la BD y cachear:
    perfil, err := c.db.obtenerPerfil(ctx, userID)
    if err != nil { return nil, err }

    c.redis.Set(ctx, key, serializar(perfil), c.ttl)
    return perfil, nil
}

// El write side invalida la caché cuando cambia el perfil:
func (c *CachéReadModel) ActualizarPerfil(ctx context.Context, perfil *Perfil) error {
    if err := c.db.actualizarPerfil(ctx, perfil); err != nil {
        return err
    }
    c.redis.Del(ctx, "perfil:"+perfil.ID)
    return nil
}
```

**Restricciones:** Implementar este patrón para el sistema de checkout del Cap.21
y medir la reducción de latencia con y sin el read model cacheado.

**Pista:** Esta forma "ligera" de CQRS (caché de lectura + invalidación en escritura)
captura el 80% del beneficio con el 20% de la complejidad del CQRS completo.
La trampa: la invalidación en escritura puede dejar la caché "stale" si la invalidación
falla (ej: el Redis está caído cuando se hace el delete). El patrón "write-through"
(actualizar caché y BD en la misma transacción) es más seguro pero más complejo.

---

## Sección 23.4 — Stream Processing: Computación sobre Eventos

```
Batch processing: procesar datos acumulados periódicamente
  → Resultado fresco cada hora/día
  → Tarda minutos/horas en completarse
  → Herramientas: Spark, Hadoop

Stream processing: procesar eventos conforme llegan
  → Resultado fresco en milisegundos/segundos
  → Procesa continuamente
  → Herramientas: Kafka Streams, Apache Flink, Go channels
```

### Ejercicio 23.4.1 — Implementar un pipeline de stream processing en Go

```go
// Los canales de Go son el primitivo de stream processing:
// cada stage del pipeline es un goroutine que lee de un canal y escribe a otro

type Evento struct {
    UserID    string
    Tipo      string
    Timestamp time.Time
    Datos     map[string]interface{}
}

// Stage 1: filtrar eventos relevantes
func filtrar(entrada <-chan Evento, predicate func(Evento) bool) <-chan Evento {
    salida := make(chan Evento, 100)
    go func() {
        defer close(salida)
        for e := range entrada {
            if predicate(e) {
                salida <- e
            }
        }
    }()
    return salida
}

// Stage 2: transformar eventos
func transformar[A, B any](entrada <-chan A, fn func(A) B) <-chan B {
    salida := make(chan B, 100)
    go func() {
        defer close(salida)
        for e := range entrada {
            salida <- fn(e)
        }
    }()
    return salida
}

// Stage 3: ventana de tiempo (time window)
func ventana(entrada <-chan Evento, tamaño time.Duration) <-chan []Evento {
    salida := make(chan []Evento, 10)
    go func() {
        defer close(salida)
        var buffer []Evento
        ticker := time.NewTicker(tamaño)
        defer ticker.Stop()
        for {
            select {
            case e, ok := <-entrada:
                if !ok {
                    if len(buffer) > 0 { salida <- buffer }
                    return
                }
                buffer = append(buffer, e)
            case <-ticker.C:
                if len(buffer) > 0 {
                    salida <- buffer
                    buffer = nil
                }
            }
        }
    }()
    return salida
}
```

**Restricciones:** Implementar un pipeline que:
1. Lee eventos de clicks de usuarios desde un canal
2. Filtra clicks en productos (ignora navegación)
3. Agrupa clicks por usuario en ventanas de 1 minuto
4. Calcula el producto más clickeado por usuario en cada ventana
5. Emite alertas cuando un usuario hace >100 clicks/minuto (bot detection)

**Pista:** El pipeline debe manejar backpressure naturalmente con canales buffered.
Si el stage de agregación es más lento que el de filtrado, el canal entre ellos
se llena y el filtrado reduce su velocidad. Esto es el modelo de concurrencia
del Cap.03 aplicado a streaming.

---

### Ejercicio 23.4.2 — Stateful stream processing: joins y aggregations

El stream processing stateful requiere mantener estado entre eventos:

```go
// Join de dos streams: unir eventos de distinto tipo por una clave común
type JoinStreams struct {
    izquierda map[string]Evento  // buffer de eventos del stream izquierdo
    derecha   map[string]Evento  // buffer de eventos del stream derecho
    ventana   time.Duration      // cuánto tiempo esperar por el otro stream
    mu        sync.Mutex
}

func (j *JoinStreams) Procesar(streamA, streamB <-chan Evento) <-chan ParDeEventos {
    resultado := make(chan ParDeEventos, 100)
    
    procesar := func(propio, otro map[string]Evento, e Evento) {
        j.mu.Lock()
        defer j.mu.Unlock()
        
        if par, ok := otro[e.UserID]; ok {
            // Encontrar el par — emitir el join:
            resultado <- ParDeEventos{A: e, B: par}
            delete(otro, e.UserID)
        } else {
            // Guardar para cuando llegue el par:
            propio[e.UserID] = e
        }
    }
    
    go func() {
        for e := range streamA {
            procesar(j.izquierda, j.derecha, e)
        }
    }()
    go func() {
        for e := range streamB {
            procesar(j.derecha, j.izquierda, e)
        }
    }()
    
    return resultado
}
```

**Restricciones:** Implementar un join entre:
- Stream de "usuario vio producto" 
- Stream de "usuario compró producto"

El join debe calcular el tiempo entre la vista y la compra (conversion funnel).
Si la compra no ocurre en 24 horas, el evento de vista debe expirar del buffer.

**Pista:** El buffer del join puede crecer indefinidamente si un stream es mucho
más rápido que el otro. La solución: TTL por evento en el buffer.
Implementar con un background goroutine que limpia eventos expirados cada minuto,
o con un heap de expiración (más eficiente para muchos eventos).

---

### Ejercicio 23.4.3 — Exactly-once en stream processing

El mayor desafío del stream processing: garantizar que cada evento
se procesa exactamente una vez, incluso cuando el procesador falla:

```
At-most-once (perdida de datos posible):
  Procesar → confirmar recepción → si falla antes de confirmar: el mensaje se pierde

At-least-once (duplicados posibles):
  Confirmar recepción → procesar → si falla entre confirmación y procesamiento:
  el mensaje se reenvía → puede procesarse dos veces

Exactly-once (sin pérdida, sin duplicados):
  Requiere que el procesamiento Y la confirmación sean atómicos.
  En la práctica: checkpointing del estado + idempotencia del procesamiento.
```

**Restricciones:** Implementar exactly-once para el pipeline del Ejercicio 23.4.1:
- El procesador guarda su posición en el stream (offset de Kafka) junto con su estado
- El guardado del offset y el estado es atómico (en la misma transacción)
- Si el procesador reinicia, retoma desde el último checkpoint

**Pista:** La solución estándar: guardar el offset y el estado en la misma
transacción de base de datos. Si el procesador procesa el evento 100 y guarda
el estado en PostgreSQL, el offset 100 también va en la misma transacción.
Si la transacción falla, el procesador reinicia desde el offset 99 y reprocesa
el evento 100 con el estado previo — el mismo resultado.

---

### Ejercicio 23.4.4 — Leer: watermarks y late arrivals

**Tipo: Diagnosticar**

Un pipeline de stream processing calcula el revenue por hora.
Los eventos tienen un `event_time` (cuándo ocurrió el click) y un
`processing_time` (cuándo llegó al sistema):

```
Hora 14:00–15:00 (event_time)

Processing_time → Event              Event_time
14:00:01        → click_user_a       14:00:00
14:00:05        → click_user_b       14:00:03
14:01:00        → purchase_user_c    14:00:58
15:00:10        → purchase_user_d    14:55:00  ← llegó tarde (5 min de retraso)
15:05:00        → purchase_user_e    14:59:00  ← llegó muy tarde (6 min de retraso)
16:00:00        → purchase_user_f    14:30:00  ← llegó muy muy tarde (90 min de retraso)
```

**Preguntas:**

1. Si la ventana cierra a las 15:00 (en processing_time), ¿qué purchases se incluyen?
2. Si usas watermarks con 10 minutos de tolerancia, ¿qué cambia?
3. ¿Cómo manejas `purchase_user_f` que llegó 90 minutos tarde?
4. ¿Cuál es el tradeoff entre latencia del resultado y completitud?
5. ¿Qué estrategia usaría Apache Flink para este problema?

**Pista:** El watermark dice "todos los eventos con event_time < T han llegado".
Con watermark = processing_time - 10 minutos, la ventana 14:00–15:00 cierra
cuando el watermark alcanza 15:00 = processing_time 15:10. Los eventos que llegan
después del cierre de la ventana son "late arrivals" — pueden ignorarse, ir a una
"late data" stream, o actualizar el resultado. `purchase_user_f` con 90 minutos de
retraso excede cualquier watermark razonable — generalmente se descarta o se procesa
en batch separado.

---

### Ejercicio 23.4.5 — Construir un mini-Kafka

Implementa una version simplificada de Kafka para entender su modelo de datos:

```go
// Kafka es esencialmente un commit log distribuido:
// - Los mensajes se añaden al final (append-only)
// - Los consumers leen desde un offset y avanzan
// - Los mensajes no se borran al leer — múltiples consumers pueden leer independientemente

type Partición struct {
    mensajes []Mensaje
    mu       sync.RWMutex
}

func (p *Partición) Producir(msg Mensaje) int64 {
    p.mu.Lock()
    defer p.mu.Unlock()
    offset := int64(len(p.mensajes))
    p.mensajes = append(p.mensajes, msg)
    return offset
}

func (p *Partición) Consumir(offset int64, maxMensajes int) []Mensaje {
    p.mu.RLock()
    defer p.mu.RUnlock()
    if offset >= int64(len(p.mensajes)) {
        return nil
    }
    fin := min(int(offset)+maxMensajes, len(p.mensajes))
    return p.mensajes[offset:fin]
}

type Topic struct {
    particiones []*Partición
}

// Elegir partición por key:
func (t *Topic) Partición(key []byte) *Partición {
    hash := fnv.New32a()
    hash.Write(key)
    return t.particiones[hash.Sum32()%uint32(len(t.particiones))]
}
```

**Restricciones:**
- Implementar consumer groups: múltiples consumers en el mismo grupo reparten las particiones
- Implementar el rebalanceo cuando un consumer entra o sale del grupo
- Implementar retención: borrar mensajes más viejos que N días

**Pista:** El consumer group rebalanceo es el problema más complejo:
cuando un consumer nuevo se une, las particiones se redistribuyen entre todos.
Durante el rebalanceo, ningún consumer procesa mensajes. En Kafka real,
el "group coordinator" (un broker especial) coordina el rebalanceo.
Para la versión simplificada: usar etcd para el liderazgo del coordinador.

---

## Sección 23.5 — Coordinación Distribuida con etcd

etcd es el sistema de coordinación del ecosistema Kubernetes. Internamente
usa Raft (del Cap.22 §22.5.1) para garantizar consistencia.

### Ejercicio 23.5.1 — Distributed lock con etcd

```go
// etcd ofrece distributed locks a través de su API de leases:

type DistributedLock struct {
    cliente  *clientv3.Client
    key      string
    ttl      int64  // segundos
    leaseID  clientv3.LeaseID
    session  *concurrency.Session
    mutex    *concurrency.Mutex
}

func (dl *DistributedLock) Lock(ctx context.Context) error {
    session, err := concurrency.NewSession(dl.cliente, concurrency.WithTTL(int(dl.ttl)))
    if err != nil { return err }
    dl.session = session
    dl.mutex = concurrency.NewMutex(session, dl.key)
    return dl.mutex.Lock(ctx)
}

func (dl *DistributedLock) Unlock(ctx context.Context) error {
    defer dl.session.Close()
    return dl.mutex.Unlock(ctx)
}
```

**Restricciones:** Implementar un job scheduler que usa distributed lock para
garantizar que solo una instancia ejecuta cada job:

```go
// 5 instancias del scheduler corren en paralelo
// Solo una debe ejecutar cada job — las demás deben saltarlo

func (s *Scheduler) EjecutarSiSoyLíder(ctx context.Context, jobID string, fn func()) {
    lock := s.nuevoLock("/jobs/" + jobID)
    if err := lock.TryLock(ctx); err != nil {
        return  // otro proceso tiene el lock
    }
    defer lock.Unlock(ctx)
    fn()
}
```

**Pista:** `TryLock` (no bloquea) vs `Lock` (espera): para un job scheduler,
`TryLock` es más apropiado — si no puedes adquirir el lock, el job ya está
siendo ejecutado por otra instancia. No tiene sentido esperar.
La duración del lease (TTL) debe ser mayor que la duración máxima del job —
si el job tarda más que el TTL, el lock expira y otra instancia puede empezar
el mismo job mientras el primero aún está corriendo.

---

### Ejercicio 23.5.2 — Leader election con etcd

La elección de líder es el caso de uso más común de etcd en Kubernetes:

```go
type LiderElecto struct {
    cliente   *clientv3.Client
    prefix    string
    campaign  *concurrency.Campaign
    session   *concurrency.Session
    esLíder   atomic.Bool
}

func (le *LiderElecto) Iniciar(ctx context.Context) error {
    session, _ := concurrency.NewSession(le.cliente)
    le.session = session
    le.campaign = concurrency.NewCampaign(session, le.prefix)

    go func() {
        // Campañar — bloquea hasta ganar la elección:
        if err := le.campaign.Campaign(ctx, le.nodeID); err != nil {
            return
        }
        le.esLíder.Store(true)
        log.Info("soy el líder")

        // Mantener el liderazgo hasta que el contexto se cancele:
        <-ctx.Done()
        le.campaign.Resign(context.Background())
        le.esLíder.Store(false)
    }()
    return nil
}
```

**Restricciones:** Implementar un servicio con liderazgo donde:
- Solo el líder acepta escrituras
- Los followers proxean las escrituras al líder
- Si el líder muere, se elige uno nuevo en < 5 segundos
- El nuevo líder está al día (leyó el estado desde etcd antes de aceptar escrituras)

**Pista:** El "al día" es el problema difícil. El nuevo líder debe leer el último
estado confirmado antes de empezar a procesar — de lo contrario puede aceptar
escrituras basándose en un estado obsoleto. En Kubernetes, los controllers resuelven
esto con la "informer cache" que se sincroniza con el API server al inicio.

---

### Ejercicio 23.5.3 — Watch: reaccionar a cambios en la configuración

```go
// etcd permite "watch" — recibir notificaciones cuando una key cambia:
func (s *Servicio) WatchConfig(ctx context.Context) {
    watchChan := s.etcd.Watch(ctx, "/config/", clientv3.WithPrefix())
    for resp := range watchChan {
        for _, event := range resp.Events {
            switch event.Type {
            case clientv3.EventTypePut:
                key := string(event.Kv.Key)
                valor := string(event.Kv.Value)
                s.actualizarConfig(key, valor)
                log.Info("config actualizada", "key", key, "valor", valor)
            case clientv3.EventTypeDelete:
                key := string(event.Kv.Key)
                s.resetearConfig(key)
            }
        }
    }
}
```

**Restricciones:** Implementar feature flags distribuidos con etcd:
- Los flags se almacenan en etcd bajo `/flags/{nombre}`
- Todos los servicios reciben la actualización en < 100ms cuando cambia un flag
- Si etcd no está disponible, los servicios usan el último valor conocido (fail-open)

**Pista:** El watch de etcd reconecta automáticamente si la conexión se pierde.
Para "fail-open": el servicio guarda en memoria el último valor de cada flag.
Si el watch se interrumpe (etcd no disponible), el servicio sigue funcionando
con los valores en memoria. Cuando etcd vuelve, el watch reanuda y sincroniza.
Esto es exactamente cómo Kubernetes maneja la indisponibilidad del API server.

---

### Ejercicio 23.5.4 — Distributed rate limiting con etcd

Un rate limiter distribuido necesita que todas las instancias compartan el estado:

```go
// El rate limiter centralized en etcd:
// No es la forma más eficiente, pero es correcta y simple

type RateLimiterEtcd struct {
    etcd   *clientv3.Client
    límite int
    ventana time.Duration
}

func (rl *RateLimiterEtcd) Permitir(ctx context.Context, clienteID string) (bool, error) {
    key := fmt.Sprintf("/rate-limits/%s/%d",
        clienteID,
        time.Now().Truncate(rl.ventana).UnixNano())

    // Incrementar con STM (Software Transactional Memory sobre etcd):
    var contador int64
    _, err := concurrency.NewSTM(rl.etcd, func(s concurrency.STM) error {
        v := s.Get(key)
        if v == "" {
            contador = 1
        } else {
            contador, _ = strconv.ParseInt(v, 10, 64)
            contador++
        }
        s.Put(key, strconv.FormatInt(contador, 10))
        return nil
    })

    return contador <= int64(rl.límite), err
}
```

**Restricciones:** Comparar el rate limiter centralizado (etcd) con el local (del Cap.17)
en términos de: correctitud bajo múltiples instancias, latencia añadida, disponibilidad
cuando etcd no está disponible.

**Pista:** El rate limiter centralizado es correcto pero añade ~5-10ms de latencia
por request (una transacción STM en etcd). Para sistemas de alta frecuencia (>1K req/s),
esto puede ser prohibitivo. La alternativa: rate limiter local con sincronización
periódica (token bucket distribuido) — cada instancia tiene su propio bucket y
sincroniza el estado global cada segundo.

---

### Ejercicio 23.5.5 — Leer: cuándo NO usar etcd

**Tipo: Analizar**

Un equipo usa etcd para almacenar datos de aplicación:

```
Uso actual de etcd:
  /sessions/{token}  → datos de sesión del usuario (1 MB cada una)
  /products/{id}     → catálogo de productos (10 KB cada uno, 100,000 productos)
  /cache/{key}       → caché de resultados (50 KB cada uno, 50,000 entradas)
  /locks/{job}       → distributed locks para jobs (pequeños, alta frecuencia)
  /config/{servicio} → configuración de servicios (pequeña, baja frecuencia)
  /leader/{servicio} → leader election (pequeño, baja frecuencia)
```

**Preguntas:**

1. ¿Cuáles usos son apropiados para etcd?
2. ¿Cuáles son inapropiados y por qué?
3. ¿Cuánto espacio ocuparía el catálogo de productos en etcd? ¿Es problemático?
4. ¿Qué herramienta es más apropiada para cada caso de uso inapropiado?
5. ¿Qué puede pasar al cluster de Kubernetes si etcd está lleno de datos de aplicación?

**Pista:** etcd está diseñado para datos de coordinación (pequeños, poca frecuencia,
alta consistencia): locks, leader election, configuración, service discovery.
El límite recomendado de etcd es 8 GB de datos y ~1,000 objetos. Almacenar
100,000 productos de 10 KB = 1 GB — al límite y con performance degradada.
Para sesiones de usuario: Redis. Para catálogo: PostgreSQL. Para caché: Redis.
Si etcd se sobrecarga, los componentes de Kubernetes que dependen de él
(scheduler, API server, controllers) dejan de funcionar.

---

## Sección 23.6 — Particionamiento y Consistent Hashing

Cuando los datos no caben en una sola máquina, se particionan:
cada nodo es responsable de un subconjunto de los datos.

```
Hash partitioning (naive):
  partition = hash(key) % N
  
  Problema: si N cambia (añadir o quitar un nodo), casi todos los datos
  deben moverse a una nueva partición.

Consistent hashing:
  Los nodos y las keys se mapean a un "anillo" de hashes.
  Cada key pertenece al primer nodo cuyo hash es >= al hash de la key.
  
  Ventaja: cuando se añade o quita un nodo, solo se mueven O(K/N) keys
  (donde K es el total de keys y N es el número de nodos).
  Solo las keys del rango afectado se mueven.
```

### Ejercicio 23.6.1 — Implementar consistent hashing

```go
type AnilloConsistente struct {
    hashes  []uint32           // hashes ordenados en el anillo
    nodos   map[uint32]string  // hash → nodoID
    mu      sync.RWMutex
    réplicas int               // réplicas virtuales por nodo (para distribución uniforme)
}

func (a *AnilloConsistente) AñadirNodo(nodoID string) {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    // Añadir réplicas virtuales para distribución uniforme:
    for i := 0; i < a.réplicas; i++ {
        hash := a.calcularHash(fmt.Sprintf("%s-%d", nodoID, i))
        a.hashes = append(a.hashes, hash)
        a.nodos[hash] = nodoID
    }
    sort.Slice(a.hashes, func(i, j int) bool { return a.hashes[i] < a.hashes[j] })
}

func (a *AnilloConsistente) ObtenerNodo(key string) string {
    a.mu.RLock()
    defer a.mu.RUnlock()
    
    if len(a.hashes) == 0 { return "" }
    
    hash := a.calcularHash(key)
    
    // Búsqueda binaria del primer hash >= hash de la key:
    idx := sort.Search(len(a.hashes), func(i int) bool {
        return a.hashes[i] >= hash
    })
    
    // Wrapping: si la key está después del último nodo, va al primero
    if idx == len(a.hashes) { idx = 0 }
    
    return a.nodos[a.hashes[idx]]
}
```

**Restricciones:**
- Verificar que añadir un nodo nuevo mueve aproximadamente 1/N de las keys
- Verificar que la distribución es uniforme con 150 réplicas virtuales por nodo
- Implementar `ObtenerNReplicasConsecutivas(key string, n int)` para replicación

**Pista:** Las réplicas virtuales son necesarias para distribución uniforme.
Sin ellas, los nodos pueden acaparar partes desiguales del anillo.
Con 150 réplicas virtuales por nodo (el default de librerías como groupcache),
la desviación estándar de la carga es ~10% — aceptable para la mayoría de usos.
Con menos réplicas, la distribución puede ser muy irregular.

---

### Ejercicio 23.6.2 — Rebalanceo: mover datos cuando cambia el cluster

Cuando se añade o quita un nodo, los datos deben moverse:

```go
type CacheDistribuida struct {
    anillo *AnilloConsistente
    datos  map[string][]byte  // datos locales de este nodo
    nodos  map[string]*ClienteNodo  // conexiones a otros nodos
}

// Cuando se añade un nuevo nodo:
func (c *CacheDistribuida) NodoUnido(nuevoNodoID, addr string) error {
    // Añadir al anillo:
    c.anillo.AñadirNodo(nuevoNodoID)

    // Mover las keys que ahora pertenecen al nuevo nodo:
    aReasignar := []string{}
    for key := range c.datos {
        if c.anillo.ObtenerNodo(key) == nuevoNodoID {
            aReasignar = append(aReasignar, key)
        }
    }

    // Transferir al nuevo nodo:
    nuevoNodo := c.conectar(nuevoNodoID, addr)
    for _, key := range aReasignar {
        nuevoNodo.Set(key, c.datos[key])
        delete(c.datos, key)
    }
    return nil
}
```

**Restricciones:** Implementar el rebalanceo sin downtime:
- Durante el rebalanceo, las keys en movimiento deben ser accesibles
- No perder datos si el nodo receptor falla durante el rebalanceo
- Minimizar el tráfico de red durante el rebalanceo

**Pista:** El protocolo de rebalanceo sin downtime: (1) copiar la key al nuevo nodo
mientras sigue accesible en el nodo original, (2) después de la copia exitosa,
hacer que el anillo apunte al nuevo nodo para esa key, (3) borrar del nodo original.
Si el nodo receptor falla en el paso (1), no pasa nada — la key sigue en el original.
Este es el mismo protocolo que usa Cassandra para mover datos (streaming).

---

### Ejercicio 23.6.3 — Ranged partitioning: alternativa al consistent hashing

El consistent hashing distribuye por hash de key. El range partitioning
distribuye por rango de key — más apropiado para queries de rango:

```
Hash partitioning:
  user_1 → nodo A
  user_2 → nodo C
  user_3 → nodo B
  Ventaja: distribución uniforme
  Desventaja: query "todos los usuarios entre user_100 y user_200" requiere consultar todos los nodos

Range partitioning:
  user_1 - user_100: nodo A
  user_101 - user_200: nodo B
  user_201 - user_300: nodo C
  Ventaja: query de rango va a un solo nodo
  Desventaja: puede crear "hot spots" (el nodo con los usuarios más activos)
```

**Restricciones:** Implementar range partitioning para una tabla de pedidos
donde las queries más frecuentes son "pedidos de la última semana".
Diseñar la partition key para que los pedidos recientes estén en el mismo nodo.

**Pista:** Usar el timestamp como parte de la partition key crea un "hot spot":
el nodo que tiene los pedidos más recientes recibe todo el tráfico.
La solución: composite key = (bucket_hash % N, timestamp) donde bucket_hash
distribuye uniformemente los usuarios entre N buckets, y dentro de cada bucket
los pedidos están ordenados por tiempo.

---

### Ejercicio 23.6.4 — Leer: hot spots y cómo evitarlos

**Tipo: Diagnosticar**

Un sistema de e-commerce tiene este patrón de carga:

```
Distribución de keys:
  product:iphone-15     → 45% del tráfico total
  product:macbook-air   → 20% del tráfico total
  product:airpods       → 15% del tráfico total
  product:* (otros)     → 20% del tráfico total distribuido

Consistent hashing con 3 nodos:
  iphone-15 → nodo A
  macbook-air → nodo B
  airpods → nodo B
  otros → distribuidos
  
Resultado:
  Nodo A: 45% de la carga
  Nodo B: 35% de la carga
  Nodo C: 20% de la carga
```

**Preguntas:**

1. ¿Por qué el consistent hashing no resuelve el hot spot en este caso?
2. Propón tres estrategias para distribuir el acceso a `product:iphone-15`.
3. ¿Cuál estrategia usarías y cuándo?
4. Si `product:iphone-15` tiene un precio que cambia veces al día,
   ¿qué complicación añade la estrategia de replicación?
5. ¿Amazon, con millones de productos con tráfico variable, cómo resuelve esto?

**Pista:** Las tres estrategias: (1) replicación en múltiples nodos con distribución
de lecturas (lectura de N copias), (2) caché local en cada instancia del servicio
(en memoria, sin red), (3) key sharding (product:iphone-15:shard-{0..9}, distribuir
entre 10 keys para distribuir en 10 nodos). La opción (2) es la más usada en
práctica para datos de lectura frecuente que cambian poco.

---

### Ejercicio 23.6.5 — Construir un mini-DynamoDB

DynamoDB es la base de datos de Amazon, diseñada para escalar horizontalmente
con consistent hashing. Implementa una versión simplificada:

```go
type MiniDynamo struct {
    anillo    *AnilloConsistente
    nodos     map[string]*Nodo
    escritura int  // quórum para escritura (ej: 2 de 3)
    lectura   int  // quórum para lectura (ej: 2 de 3)
}

func (d *MiniDynamo) Put(ctx context.Context, key string, valor []byte) error {
    nodos := d.anillo.ObtenerNReplicasConsecutivas(key, 3)
    confirmadas := 0
    var mu sync.Mutex
    var wg sync.WaitGroup

    for _, nodoID := range nodos {
        wg.Add(1)
        go func(id string) {
            defer wg.Done()
            if err := d.nodos[id].Set(key, valor); err == nil {
                mu.Lock()
                confirmadas++
                mu.Unlock()
            }
        }(nodoID)
    }

    wg.Wait()
    if confirmadas < d.escritura {
        return ErrQuorumNoAlcanzado
    }
    return nil
}
```

**Restricciones:**
- Replicación en 3 nodos con quórum configurable
- Si un nodo está caído, las escrituras van a los otros 2 (hinted handoff)
- Cuando el nodo vuelve, sincroniza los datos que se perdió (anti-entropy)

**Pista:** El "hinted handoff": si el nodo responsable de una key está caído,
el write va a otro nodo que "promete" entregarlo cuando el nodo original vuelva.
El nodo intermediario guarda el dato con metadata "para nodo X". Cuando X vuelve,
el intermediario le transfiere los datos prometidos. Esto garantiza durabilidad
incluso cuando el nodo primario está temporalmente caído.

---

## Sección 23.7 — El Sistema Completo: Integrando Todo

### Ejercicio 23.7.1 — Diseño de una plataforma de mensajería escalable

Diseña e implementa la arquitectura de una plataforma de mensajería tipo Slack:

```
Requisitos:
  - 1 millón de usuarios activos
  - Mensajes entregados en < 100ms en la misma región
  - Mensajes no se pierden (durable)
  - Historial de mensajes accesible (últimos 10,000 mensajes por canal)
  - Múltiples regiones (US, EU, APAC)

Componentes:
  - WebSocket server (conexiones persistentes de usuarios)
  - Message broker (routing de mensajes)
  - Message store (persistencia y historial)
  - Presence service (¿quién está online?)
  - Notification service (push notifications para usuarios offline)
```

**Restricciones:** Para cada componente, especificar:
- Qué tecnología (etcd, Kafka, PostgreSQL, Redis, etc.)
- Qué garantía de consistencia
- Cómo escala horizontalmente
- Cómo se comporta durante una partición de red entre regiones

**Pista:** El WebSocket server es stateful — la conexión del usuario está en una
instancia específica. Para entregar un mensaje a Alice (conectada al servidor 3),
desde Bob (conectado al servidor 7), necesitas routing entre servidores.
Redis Pub/Sub o Kafka resuelven esto: el servidor 7 publica el mensaje, el servidor 3
está suscrito al canal de Alice y lo entrega. La presencia (online/offline) puede
usar Redis con TTL (heartbeat del cliente actualiza el TTL).

---

### Ejercicio 23.7.2 — Diseño de un sistema de pagos distribuido

```
Requisitos:
  - Exactamente-una-vez: un pago nunca se procesa dos veces
  - Durabilidad: un pago confirmado nunca se pierde
  - Idempotencia: reintentar un pago fallido es seguro
  - Auditoría: historial completo de todos los cambios de estado
  - Multi-moneda: tipos de cambio actualizados en tiempo real

Componentes:
  - API de pagos (frontend del sistema)
  - Payment processor (orquestador del pago)
  - Ledger service (registro contable, inmutable)
  - Exchange rate service (tipos de cambio)
  - Notification service (confirmaciones)
```

**Restricciones:** Diseñar el flujo completo de un pago con:
- Idempotency keys para el API de pagos
- Saga pattern para coordinar los servicios (sin 2PC)
- Event sourcing para el ledger (auditoría completa)
- Compensaciones para cada paso de la saga

**Pista:** El flujo con saga: (1) reservar fondos (bloquear, no debitar), (2) llamar
al procesador externo, (3) debitar si éxito / liberar si error. La compensación de
(1) es liberar los fondos. La compensación de (2) es llamar al procesador para anular.
Si el procesador externo no soporta anulación, el sistema debe detectar el doble cobro
por otro medio (reconciliación con el extracto bancario).

---

### Ejercicio 23.7.3 — Operación: el runbook del sistema distribuido

Documenta el runbook de operación para el sistema de mensajería del Ejercicio 23.7.1:

```markdown
# Runbook: Plataforma de Mensajería

## Síntomas críticos y acciones inmediatas

### Los mensajes tardan más de 1 segundo en entregarse
Diagnóstico (< 2 minutos):
  1. Ver dashboard: ¿qué componente tiene alta latencia?
  2. Si Kafka: ver consumer lag — ¿el broker está saturado?
  3. Si WebSocket server: ver goroutines activas — ¿hay un goroutine leak?
  4. Si Redis: ver memory usage — ¿está cerca del límite?

Mitigación inmediata:
  [...]

### Usuarios reportan "no puedo conectarme"
[...]

### Mensajes duplicados reportados
[...]
```

**Restricciones:** El runbook debe cubrir los 5 fallos más probables con:
- Cómo detectarlos (métricas o logs)
- Diagnóstico en < 5 minutos
- Mitigación inmediata (sin root cause)
- Investigación del root cause
- Prevención a futuro

---

### Ejercicio 23.7.4 — Migración: de monolito a microservicios

Un sistema monolítico debe migrarse a microservicios sin downtime.
Describir el patrón "Strangler Fig" aplicado al sistema de e-commerce
que has construido a lo largo de este repositorio:

```
Monolito actual:
  - Orders
  - Inventory  
  - Payments
  - Users
  Todo en el mismo proceso, misma BD

Estrategia Strangler Fig:
  Fase 1: Añadir proxy delante del monolito (sin cambios en el monolito)
  Fase 2: Extraer un servicio (ej: Users) — el proxy enruta a uno u otro
  Fase 3: Una vez el nuevo servicio estable, eliminar el código del monolito
  Fase 4: Repetir para cada servicio
```

**Restricciones:** Diseñar el plan de migración especificando:
- Qué servicio extraer primero y por qué
- Cómo manejar las transacciones que cruzan servicios durante la migración
- Cómo hacer rollback si el nuevo servicio tiene problemas
- Cómo validar que el comportamiento es idéntico (shadowing)

**Pista:** El servicio más seguro para extraer primero: el que tiene menos
dependencias con el resto del sistema. Users es a menudo una buena opción —
es read-heavy, no tiene transacciones complejas, y los otros servicios
solo necesitan validar que el user_id existe. Payments es el más arriesgado
de extraer — tiene la mayor complejidad transaccional.

---

### Ejercicio 23.7.5 — Cierre: el sistema más difícil de depurar

**Tipo: Síntesis**

A lo largo de este repositorio implementaste o analizaste:
concurrencia, paralelismo, scheduling, locks, canales, race conditions,
deadlocks, observabilidad, debugging, code review, resiliencia,
sistemas distribuidos, consenso, y arquitectura.

El ejercicio final no tiene código que escribir. Tiene una sola pregunta
abierta que requiere integrar todo:

---

**Un sistema de e-commerce en producción empieza a mostrar el siguiente comportamiento
en horas punta:**

```
- Las órdenes se crean correctamente (HTTP 200)
- Los usuarios ven "orden creada" en la interfaz
- 30 minutos después, los emails de confirmación no llegan
- El equipo de soporte recibe quejas: "mi orden no aparece en el historial"
- Las métricas muestran: tasa de error = 0%, latencia p99 = 45ms (normal)
- El revenue sigue acumulando en el dashboard (las órdenes existen en algún lugar)
- Los logs muestran: "orden guardada exitosamente" para cada orden
- Después de 2 horas, las órdenes "aparecen" en el historial
```

**Preguntas:**

1. ¿De cuántos de los capítulos de este repositorio (18–23) pueden ser parte de la causa?
2. Propón tres hipótesis específicas (con nombres de componentes y patrones concretos).
3. ¿En qué orden las investigarías y por qué?
4. ¿Qué herramienta de cada capítulo usarías para confirmar o descartar cada hipótesis?
5. Si tuvieras que apostar: ¿cuál es la causa más probable y por qué?

**No hay una respuesta única correcta.** El valor del ejercicio está en el proceso:
cómo construyes hipótesis, cómo las priorizas, y cómo usas las herramientas del
repositorio para investigarlas sistemáticamente.

---

*Pista para la discusión:*

*El síntoma "las órdenes aparecen después de 2 horas" con "tasa de error = 0%"
sugiere que el sistema está funcionando, pero con consistencia eventual donde
se esperaba consistencia inmediata. Las hipótesis más probables involucran
la Sección 23.3 (CQRS con replication lag) o la Sección 23.2 (event sourcing
con proyección atrasada). El email de confirmación no llegando puede ser un
goroutine leak (Cap.19) en el servicio de notificaciones, o un bulkhead saturado
(Cap.21) que está descartando silenciosamente los envíos.*

*El ejercicio 23.7.5 existe porque los bugs más difíciles de producción
siempre cruzan múltiples capas del stack. No son bugs de concurrencia puros
ni bugs distribuidos puros — son la intersección de los dos, con un toque
de observabilidad insuficiente y un evento de carga inusual.*

---

## Epílogo del Repositorio

Este repositorio empezó con 5 problemas fundamentales de concurrencia
y termina con el diseño de sistemas distribuidos multi-región.

El camino fue:

```
Cap.01–07: Concurrencia — el problema base
  (goroutines, canales, locks, deadlocks, testing)

Cap.08–12: Paralelismo — escalar en una máquina
  (CPU cores, data parallelism, GPU, hardware effects)

Cap.13–16: Lenguajes — el mismo problema en distintos contextos
  (Rust, Java, Python, C#)

Cap.17: Arquitectura de producción — el sistema que sobrevive la realidad

Cap.18–21: Operaciones — cómo mantenerlo vivo y entenderlo cuando falla
  (observabilidad, debugging, code review, resiliencia)

Cap.22–23: Distribución — cuando una máquina ya no es suficiente
  (CAP, consenso, event sourcing, stream processing)
```

**Lo que no está en este repositorio** (el siguiente paso):

- Algoritmos de consenso formalmente verificados (TLA+)
- Byzantine fault tolerance (para sistemas sin confianza mutua)
- Distributed transactions con Spanner o CockroachDB en profundidad
- Formal methods para verificar propiedades de concurrencia

Para continuar: *The Art of Multiprocessor Programming* (Herlihy & Shavit)
cubre la teoría de estructuras de datos concurrentes. *Designing Data-Intensive
Applications* (Kleppmann) cubre el stack completo de sistemas distribuidos
con el mismo rigor que este repositorio intenta para concurrencia.

El objetivo de este repositorio nunca fue que dominaras todo. Fue que
cuando te encuentres con un bug de concurrencia, un sistema que no escala,
o un incidente de producción, tengas el vocabulario y las herramientas
para pensar sobre el problema sistemáticamente.

Eso es suficiente para empezar.
