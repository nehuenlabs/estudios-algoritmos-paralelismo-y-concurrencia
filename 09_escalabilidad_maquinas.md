# Guía de Ejercicios — Escalabilidad: Cuándo una Máquina No Basta

> Implementar cada ejercicio en: **Go · Java · Python · Rust**
> según corresponda. Los ejercicios distribuidos usan HTTP/gRPC local.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: los tres límites de una máquina

Hay exactamente tres razones por las que una sola máquina no basta:

**1. Límite de cómputo (CPU)**

```
Problema: calcular el índice de Google sobre toda la web (petabytes de texto)
Una máquina con 128 cores tarda 3 años.
100,000 máquinas con 128 cores: 30 minutos.

Cuándo aplica: el trabajo es paralelizable y tarda demasiado en una máquina.
Solución: más máquinas (paralelismo horizontal).
```

**2. Límite de memoria (RAM)**

```
Problema: un grafo de redes sociales con 3 mil millones de nodos
Una máquina con 1 TB de RAM puede almacenar ~200 millones de usuarios.
Para 3 mil millones: necesitas al menos 15 máquinas.

Cuándo aplica: los datos no caben en la RAM de una máquina.
Solución: partición de datos (sharding).
```

**3. Límite de disponibilidad**

```
Problema: el servidor de un banco no puede tener downtime
Una sola máquina tiene ~99.9% uptime = 8.7 horas/año de downtime.
Con N máquinas redundantes: N × 9s de disponibilidad.

Cuándo aplica: el SLA requiere más disponibilidad de la que una máquina puede dar.
Solución: redundancia (replicación).
```

**El punto clave:** distribuir tiene costos. Latencia de red (100µs vs 100ns para RAM),
complejidad de sincronización, fallos parciales. Nunca distribuyas si no necesitas.

---

## La escala que importa en entrevistas

```
                    Datos       QPS típico    ¿Distribuido?
──────────────────  ──────────  ──────────    ─────────────
Startup pequeña     < 10 GB     < 1,000       No — una máquina basta
Empresa mediana     10 GB-1 TB  1,000-50,000  Tal vez — depende
Empresa grande      1 TB-1 PB   50,000+       Sí — necesariamente
Web-scale (Google)  Exabytes    Millones      Sí — siempre
```

Una pregunta de diseño de sistemas en entrevista que ignora el orden de magnitud
de los datos es una respuesta incompleta.

---

## Tabla de contenidos

- [Sección 9.1 — Los límites de una máquina: números concretos](#sección-91--los-límites-de-una-máquina-números-concretos)
- [Sección 9.2 — Sharding: partir los datos](#sección-92--sharding-partir-los-datos)
- [Sección 9.3 — Replicación: copias para disponibilidad](#sección-93--replicación-copias-para-disponibilidad)
- [Sección 9.4 — Consistent hashing: sharding sin rebalanceo total](#sección-94--consistent-hashing-sharding-sin-rebalanceo-total)
- [Sección 9.5 — El teorema CAP en la práctica](#sección-95--el-teorema-cap-en-la-práctica)
- [Sección 9.6 — Desde el paralelismo local al distribuido](#sección-96--desde-el-paralelismo-local-al-distribuido)
- [Sección 9.7 — Cuándo NO distribuir](#sección-97--cuándo-no-distribuir)

---

## Sección 9.1 — Los Límites de una Máquina: Números Concretos

El análisis de back-of-the-envelope (estimación de orden de magnitud) es
una habilidad crítica. Las entrevistas de diseño de sistemas la requieren.
Los números a memorizar:

```
Latencias (2024):
  Registro de CPU:        ~0.3 ns
  Caché L1:               ~1 ns
  Caché L2:               ~4 ns
  Caché L3:               ~20 ns
  RAM:                    ~100 ns
  SSD NVMe (local):       ~100 µs
  HDD (local):            ~10 ms
  Red local (mismo rack): ~0.5 ms  (500,000 ns — 5000x RAM)
  Red entre datacenters:  ~50-150 ms
  Red intercontinental:   ~150-250 ms

Throughput (2024):
  Disco NVMe:             ~7 GB/s lectura, ~5 GB/s escritura
  Red Ethernet (1 nodo):  ~10 Gbps = ~1.25 GB/s
  RAM:                    ~50 GB/s
  CPU (single core):      ~10 GFLOPS (float64)

Capacidades típicas de un nodo de producción (2024):
  RAM:     64-256 GB
  Cores:   32-128
  Disco:   2-32 TB NVMe
  Red:     10-100 Gbps
```

**Las reglas de escala:**

```
1 máquina puede manejar:
  ~100,000 req/s simples (solo RAM, sin DB)
  ~10,000 req/s con acceso a BD
  ~1,000 req/s con acceso a BD + cómputo moderado

1 máquina puede almacenar:
  ~256 GB en RAM (datos calientes)
  ~32 TB en NVMe (datos tibios)
  Sin límite en object storage (S3) — precio, no capacidad

Si necesitas más: escalar verticalmente (más RAM, más cores)
o escalar horizontalmente (más máquinas)
```

---

### Ejercicio 9.1.1 — Estimación back-of-the-envelope: ¿necesitas escalar?

**Enunciado:** Para cada sistema, estima si una sola máquina basta y por qué:

1. **Twitter/X**: 300M usuarios activos mensuales, 500M tweets/día, cada tweet ~280 bytes
2. **Netflix streaming**: 200M usuarios, pico 100M streams simultáneos, 5 Mbps por stream
3. **Ride-sharing GPS**: 1M drivers enviando ubicación cada 5 segundos
4. **E-commerce pequeño**: 10,000 productos, 1,000 usuarios/día, 100 orders/día
5. **Sistema de logs**: 1,000 servidores generando 100 logs/segundo cada uno

**Restricciones:** Para cada sistema, calcula:
- Datos totales que necesitan estar en memoria "caliente"
- QPS (queries per second) en pico
- Bandwidth de red necesario
- ¿Una máquina de 256 GB RAM, 128 cores, 100 Gbps basta?

**Pista:** Twitter: 500M tweets × 280 bytes = 140 GB/día solo datos nuevos.
El historial de tweets (10 años) ≈ 500 TB — no cabe en una máquina.
El sistema de búsqueda sí podría estar en varias máquinas con índice distribuido.

**Implementar en:** Cálculo escrito + código que valida las estimaciones

---

### Ejercicio 9.1.2 — Medir el overhead de red vs local

**Enunciado:** Mide empíricamente la diferencia entre llamadas locales
(en el mismo proceso) y llamadas por red (loopback):

```go
// Llamada local — memoria compartida
func llamadaLocal(srv *Servicio, req Request) Response {
    return srv.Procesar(req)  // ~1 µs
}

// Llamada HTTP local — loopback (127.0.0.1)
func llamadaHTTP(client *http.Client, url string, req Request) Response {
    // serializar, enviar, recibir, deserializar — ~500 µs
}

// Llamada gRPC local
func llamadagRPC(client pb.ServicioClient, req *pb.Request) *pb.Response {
    // similar a HTTP pero más eficiente — ~200 µs
}
```

**Restricciones:** Mide el overhead de: serialización (JSON/protobuf),
latencia de red (loopback), y deserialización por separado.
El resultado debe mostrar que la red es 100-500x más costosa que la RAM.

**Pista:** La latencia de loopback (127.0.0.1) incluye: syscall (send),
stack de red del kernel, syscall (recv) × 2. Esto es ~50µs mínimo.
La serialización/deserialización JSON agrega ~10-100µs según el tamaño.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 9.1.3 — El límite de una máquina para un servidor web

**Enunciado:** Implementa un servidor HTTP que responde con datos de una
caché en memoria y encuentra el límite de QPS de una sola máquina:

```go
func main() {
    cache := NuevaCache(1_000_000)  // 1M entradas precargadas
    http.HandleFunc("/get", func(w http.ResponseWriter, r *http.Request) {
        key := r.URL.Query().Get("key")
        valor, _ := cache.Get(key)
        fmt.Fprint(w, valor)
    })
    http.ListenAndServe(":8080", nil)
}
```

Usa `wrk` o `hey` para medir el QPS máximo con distintas concurrencias:
1, 10, 100, 1000 connections concurrentes.

**Restricciones:** Mide hasta que la latencia p99 supera 10ms — ese es el límite práctico.
Reporta: QPS máximo, el número de connections donde empieza a degradarse, y por qué.

**Pista:** El límite no es siempre el CPU. Para handlers muy simples (solo lectura de caché),
el límite suele ser el número de goroutines activas compitiendo por el scheduler.
Para handlers con más trabajo, el CPU satura primero.
Un servidor Go bien escrito puede manejar ~100,000 req/s para handlers simples.

**Implementar en:** Go · Java (Netty/Spring) · Python (FastAPI) · Rust (Actix)

---

### Ejercicio 9.1.4 — Cuándo la RAM es el límite

**Enunciado:** Un sistema de recomendaciones necesita almacenar el vector de
preferencias de cada usuario (100 dimensiones float32 = 400 bytes por usuario).

¿Cuántos usuarios caben en una máquina con 256 GB de RAM
si también necesitas espacio para el índice, el sistema operativo,
y el código del servidor?

```
Presupuesto de memoria:
  OS + overhead:        ~10 GB
  Servidor + caché JVM: ~20 GB
  Datos disponibles:    ~226 GB

  Usuarios que caben:   226 GB / 400 bytes = ~565 millones
```

Si tienes 2 mil millones de usuarios, ¿cuántas máquinas necesitas?
¿Cómo distributes los usuarios entre máquinas?

**Restricciones:** Implementa la distribución con consistent hashing (previa Sección 9.4).
Verifica que ninguna máquina excede el 110% de la carga media (balanceo).

**Pista:** La respuesta "4 máquinas" es incorrecta — necesitas redundancia (al menos 2x)
y headroom para crecimiento (20-30% extra). La respuesta práctica es ~12 máquinas
(4 para los datos, 2x replicación, 1.5x headroom) → redondear a la siguiente potencia de 2.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.1.5 — El costo de la consistencia distribuida

**Enunciado:** Una operación que tarda 1µs localmente puede tardar 10ms
distribuida si requiere consenso (Paxos/Raft con 3 nodos). 
Mide este overhead empíricamente implementando:

1. Contador local con atómico: ~10ns por operación
2. Contador distribuido con 3 nodos (2-phase commit simplificado): mide
3. Contador distribuido con CRDTs (eventual consistency): mide

**Restricciones:** Los 3 nodos corren como procesos separados en la misma máquina
(usando diferentes puertos). La latencia incluye la comunicación TCP/HTTP.

**Pista:** El 2-phase commit requiere 2 round-trips (prepare + commit) —
con 3 nodos y latencia de loopback de ~100µs, son ~400µs mínimo.
Los CRDTs no requieren coordinación para escribir — solo para leer el valor consistente.
Para escrituras frecuentes y lecturas ocasionales, los CRDTs son mucho más rápidos.

**Implementar en:** Go · Java · Python

---

## Sección 9.2 — Sharding: Partir los Datos

El sharding divide los datos en particiones (shards) que viven en distintas máquinas.
Permite escalar más allá del límite de una sola máquina.

**Las tres estrategias de sharding:**

```
1. Range sharding:
   Shard 0: usuarios con ID 0-999,999
   Shard 1: usuarios con ID 1,000,000-1,999,999
   Shard N: usuarios con ID N*1M - (N+1)*1M - 1

   Ventaja: range queries eficientes (dame usuarios del 1M al 2M)
   Desventaja: hotspots (los IDs nuevos siempre van al mismo shard)

2. Hash sharding:
   shard = hash(userID) % N_shards

   Ventaja: distribución uniforme
   Desventaja: range queries ineficientes, rebalanceo costoso

3. Directory sharding:
   Una tabla de directorio mapea userID → shard
   Ventaja: flexible, puede mover datos entre shards
   Desventaja: el directorio es un cuello de botella y punto de fallo
```

---

### Ejercicio 9.2.1 — Implementar hash sharding

**Enunciado:** Implementa un router de sharding que dirige requests al shard correcto:

```go
type RouterSharding struct {
    shards []*Shard
    n      int
}

func (r *RouterSharding) ShardPara(key string) *Shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return r.shards[h.Sum32() % uint32(r.n)]
}
```

**Restricciones:** Verifica que la distribución es uniforme para 1M claves.
La desviación estándar del número de claves por shard debe ser <5% de la media.
Implementa Get, Set, y Delete que van automáticamente al shard correcto.

**Pista:** La función de hash debe distribuir bien. FNV32a es una opción común
para sharding porque es rápida y tiene buena distribución. SHA-256 es más fuerte
pero innecesariamente lento para sharding. Prueba la distribución con un histograma.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 9.2.2 — El problema del rebalanceo

**Enunciado:** Con hash sharding y 4 shards, si añades un quinto shard,
necesitas mover ~80% de los datos (de `hash(key) % 4` a `hash(key) % 5`).

Mide este impacto y el downtime que causaría:

```go
// Con N=4 shards:
shard4 := hash(key) % 4  // asignación original

// Con N=5 shards:
shard5 := hash(key) % 5  // nueva asignación

// Porcentaje de claves que cambian de shard:
// (claves donde shard4 != shard5) / total
```

**Restricciones:** Calcula el porcentaje de claves que se mueven para
N → N+1 con N = {2, 4, 8, 16, 32}. Verifica que siempre es ~(N-1)/N * 100%.
Explica por qué esto hace que el rebalanceo sea costoso en producción.

**Pista:** Para N=4 → N=5: se mueven ~(4/5) = 80% de las claves.
Para N=100 → N=101: se mueven ~(100/101) ≈ 99% de las claves.
Este es el problema que el consistent hashing (Sección 9.4) resuelve:
con N → N+1, solo se mueve ~(1/N) de las claves (el mínimo posible).

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.2.3 — Range sharding con hotspots

**Enunciado:** Un sistema de analytics tiene range sharding por fecha:
- Shard 0: datos de 2020
- Shard 1: datos de 2021
- Shard 2: datos de 2022
- Shard 3: datos de 2023-2024 (shard "actual")

Todos los nuevos datos van al shard 3. Diseña una estrategia para
distribuir la carga del shard caliente:

**Restricciones:** Implementa "split sharding": cuando el shard actual supera
un umbral de tamaño o QPS, dividirlo en dos. Verifica que los datos existentes
son accesibles después del split.

**Pista:** El hotspot de range sharding se mitiga con "pre-splitting":
crear múltiples shards para el periodo actual desde el inicio,
y distribuir las escrituras entre ellos usando hash(timestamp % N_subshards).
Las lecturas de range queries necesitan consultar todos los sub-shards del periodo.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.2.4 — Transacciones cross-shard

**Enunciado:** Una transferencia bancaria entre dos cuentas que están en
distintos shards no puede hacerse con una transacción local.

```go
// Cuenta A está en Shard 1, Cuenta B está en Shard 2
// Transferir 100 de A a B de forma atómica

func Transferir(shardA, shardB *Shard, idA, idB string, monto int) error {
    // Opción 1: Two-Phase Commit (2PC)
    // Opción 2: Saga (compensación)
    // Opción 3: Diseñar para evitar cross-shard (co-localización)
}
```

**Restricciones:** Implementa la Saga del Ejercicio 6.7.3 para este caso.
Verifica que si la Fase 2 falla (el crédito en B falla), la Fase 1 se revierte (el débito en A).

**Pista:** Las transacciones cross-shard son el mayor dolor del sharding.
La solución de diseño más elegante es evitarlas: poner las dos cuentas de una
transferencia frecuente en el mismo shard (co-localización por patrón de acceso).
Cuando no es posible evitarlas, la Saga es preferible a 2PC por su disponibilidad.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.2.5 — Shard key design: el arte de elegir la clave

**Enunciado:** Para un sistema de mensajes directos (como Twitter DMs o WhatsApp),
diseña tres estrategias de shard key y analiza sus tradeoffs:

1. `shard_key = hash(sender_id)`
2. `shard_key = hash(conversation_id)`
3. `shard_key = hash(min(userA, userB) + max(userA, userB))`

Para cada estrategia, analiza:
- ¿Los mensajes de una conversación están en el mismo shard? (locality)
- ¿Hay hotspots si un usuario es muy activo? (distribution)
- ¿Se pueden recuperar todos los mensajes de un usuario eficientemente? (query pattern)

**Restricciones:** Implementa las tres y mide la distribución con un dataset
sintético de 1M mensajes con distribución Pareto de usuarios activos.

**Pista:** La estrategia 3 (basada en la conversación) garantiza que todos los
mensajes de una conversación están en el mismo shard — queries eficientes.
Pero si hay 100 usuarios que dominan el 80% de los mensajes (distribución Pareto),
el 20% de los shards tendrá el 80% de la carga.

**Implementar en:** Go · Java · Python

---

## Sección 9.3 — Replicación: Copias para Disponibilidad

La replicación mantiene múltiples copias de los datos en distintas máquinas.
El objetivo es disponibilidad y tolerancia a fallos, no escalar el almacenamiento.

**Los tres modos de replicación:**

```
1. Sincrónica (strong consistency):
   Escritura → Primario
   Primario → espera confirmación de todas las réplicas
   Primario → confirma al cliente
   
   Latencia: alta (espera todas las réplicas)
   Durabilidad: máxima (todos tienen el dato antes de confirmar)
   Disponibilidad: baja si una réplica está caída

2. Semi-sincrónica (compromiso):
   Escritura → Primario
   Primario → espera confirmación de al menos 1 réplica
   Primario → confirma al cliente
   
   Latencia: moderada
   Durabilidad: buena (al menos 2 copias antes de confirmar)
   Disponibilidad: moderada

3. Asíncrona (eventual consistency):
   Escritura → Primario
   Primario → confirma al cliente INMEDIATAMENTE
   Primario → replica en background
   
   Latencia: mínima
   Durabilidad: puede perderse datos si el primario cae antes de replicar
   Disponibilidad: alta
```

---

### Ejercicio 9.3.1 — Replicación primario-réplica

**Enunciado:** Implementa un sistema de replicación primario-réplica
con los tres modos de consistencia:

```go
type Cluster struct {
    primario  *Nodo
    replicas  []*Nodo
    modo      ModoReplicacion
}

type ModoReplicacion int
const (
    Sincronica     ModoReplicacion = iota
    SemiSincronica
    Asincronica
)

func (c *Cluster) Escribir(key, value string) error
func (c *Cluster) Leer(key string) (string, error)
```

**Restricciones:** Simula el fallo del primario durante una escritura y verifica
qué datos se pierden en cada modo. Los nodos son goroutines con canales de comunicación.

**Pista:** Para el modo sincrónico, el primario espera ACK de todas las réplicas
antes de responder al cliente. Usa `errgroup` para esperar todas las réplicas.
Para el modo asíncrono, el primario usa un canal con buffer para las réplicas
— si el buffer está lleno, las escrituras de replicación se descartan (lag de replicación).

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.3.2 — Detección de fallo y failover

**Enunciado:** Cuando el primario falla, el sistema debe elegir un nuevo primario
(failover). Implementa detección de fallo por heartbeat y elección de réplica:

```go
type Monitor struct {
    cluster *Cluster
    intervalo time.Duration
    timeout   time.Duration
}

func (m *Monitor) Monitorear(ctx context.Context)
func (m *Monitor) ElegirNuevoPrimario() (*Nodo, error)
```

**Restricciones:** El failover debe ocurrir en menos de 30 segundos.
La réplica con más datos actualizados debe ser elegida como nuevo primario.
Verifica que no hay "split brain" (dos nodos creyendo ser el primario).

**Pista:** El split brain es el problema más serio del failover.
La solución clásica: quorum — solo se puede elegir primario si más de la mitad
de los nodos acuerdan. Con 3 nodos, necesitas 2 votos. Con 5 nodos, necesitas 3.
Raft y Paxos formalizan este mecanismo.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.3.3 — Replication lag: leer datos desactualizados

**Enunciado:** Con replicación asíncrona, una réplica puede estar varios
segundos detrás del primario. Un usuario que escribe y luego lee
puede obtener el valor antiguo (si lee de la réplica).

```
t=0: Usuario escribe X=100 (va al primario)
t=1: Usuario lee X (va a la réplica)
t=2: La réplica recibe X=100 del primario
     → El usuario vio X=0 (valor anterior) aunque escribió X=100
```

Implementa dos estrategias para mitigar esto:
1. **Read-your-writes**: siempre leer del primario después de una escritura reciente
2. **Sticky sessions**: el mismo usuario siempre lee del mismo nodo

**Restricciones:** La estrategia 1 usa un timestamp de la última escritura
por usuario, almacenado en el cliente (cookie o token).
La estrategia 2 usa afinidad de sesión en el load balancer.

**Pista:** Read-your-writes es la garantía mínima para una buena UX.
Sin ella, el usuario escribe un comentario y al refrescar la página no lo ve
(está leyendo de una réplica que no tiene la escritura aún).
DynamoDB, Cassandra, y la mayoría de bases de datos distribuidas ofrecen
esta garantía como opción configurable.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.3.4 — Quorum reads y writes

**Enunciado:** El sistema de quorum (W + R > N) garantiza consistencia
sin requerir sincronización estricta:

```
N = número total de réplicas
W = cuántas réplicas deben confirmar una escritura
R = cuántas réplicas deben responder una lectura

Consistencia fuerte: W + R > N

Ejemplo con N=3:
  W=2, R=2: consistente — al menos una réplica tiene el dato más reciente
  W=1, R=1: eventual consistency — puede leer dato antiguo
  W=3, R=1: escritura lenta (todas deben confirmar), lectura rápida
```

Implementa el sistema de quorum y verifica las garantías de consistencia.

**Restricciones:** Para W+R > N, la lectura debe retornar el valor más reciente
(comparando versiones/timestamps). Para W+R ≤ N, puede retornar versiones distintas.
Simula un nodo lento y verifica que el quorum todavía funciona.

**Pista:** El protocolo de Dynamo de Amazon (2007) popularizó los quorums
con N=3, W=2, R=2. Cassandra, Riak, y otros NoSQL lo usan.
El "valor más reciente" se determina por timestamp o número de versión (vector clocks).

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.3.5 — Replicación multi-región con conflict resolution

**Enunciado:** Con réplicas en múltiples regiones geográficas (US, EU, Asia),
dos usuarios pueden escribir el mismo dato simultáneamente en distintas regiones.
Hay un conflicto de escritura.

```
t=0: Usuario A (US) escribe nombre="Alice"
t=0: Usuario B (EU) escribe nombre="Alicia"
t=1: Las regiones sincronizan — ¿cuál valor es el correcto?
```

Implementa tres estrategias de resolución de conflictos:
1. **Last-write-wins (LWW)**: el timestamp más reciente gana
2. **Multi-value (versión de DynamoDB)**: retornar ambos valores al cliente
3. **CRDT (Conflict-free Replicated Data Type)**: estructura de datos que se fusiona automáticamente

**Restricciones:** Para la estrategia CRDT, implementa un G-Counter (counter que solo crece)
y un 2P-Set (set que permite agregar y eliminar).
Verifica que las operaciones concurrentes producen el mismo resultado
independientemente del orden de sincronización.

**Pista:** LWW es simple pero puede perder datos si los relojes están desincronizados
(NTP tiene ±1ms de precisión). CRDTs son la solución más elegante para tipos específicos
de datos. Para datos generales (strings, objetos complejos), multi-value es honesto
pero requiere que el cliente resuelva el conflicto.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 9.4 — Consistent Hashing: Sharding sin Rebalanceo Total

El consistent hashing resuelve el problema del Ejercicio 9.2.2: cuando cambia
el número de shards, solo se mueve una fracción mínima de los datos.

**La idea:**

```
Imagina un anillo circular de valores hash de 0 a 2^32-1.
Los shards se ubican en puntos del anillo.
Cada clave va al primer shard que encuentras en el sentido de las agujas del reloj.

Con N=4 shards:
  Shard 0 en posición 100
  Shard 1 en posición 200
  Shard 2 en posición 300
  Shard 3 en posición 400

  Key "alice" → hash = 150 → Shard 1 (primer shard > 150)
  Key "bob"   → hash = 350 → Shard 3 (primer shard > 350)

Cuando añades Shard 4 en posición 250:
  Solo las claves entre 200 y 250 se mueven de Shard 2 a Shard 4
  ~(1/N) de las claves = 25% (vs 80% con hash simple)
```

**Nodos virtuales (vnodes) — para distribución uniforme:**

Un solo punto en el anillo por shard da distribución poco uniforme.
Con K vnodes por shard (K=100-200), la distribución es mucho más uniforme:

```
Shard 0 en posiciones: 50, 180, 320, 450, ...  (100 posiciones)
Shard 1 en posiciones: 30, 150, 290, 410, ...  (100 posiciones)
```

---

### Ejercicio 9.4.1 — Implementar consistent hashing

**Enunciado:** Implementa el anillo de consistent hashing:

```go
type AnilloHash struct {
    vnodes  int       // número de vnodes por shard
    anillo  []Punto   // puntos en el anillo, ordenados
    shards  map[uint32]*Shard
}

type Punto struct {
    hash  uint32
    shard *Shard
}

func (a *AnilloHash) AgregarShard(id string, shard *Shard)
func (a *AnilloHash) EliminarShard(id string)
func (a *AnilloHash) ShardPara(key string) *Shard
```

**Restricciones:** `ShardPara` debe ser O(log N) usando búsqueda binaria en el anillo.
`AgregarShard` y `EliminarShard` deben ser O(K log N) donde K es el número de vnodes.
Verifica que la distribución con 100 vnodes tiene desviación estándar < 10%.

**Pista:** La búsqueda del primer shard en sentido horario es una búsqueda
de lower_bound en un array ordenado. El módulo (anillo) se implementa con
`if idx == len(anillo): idx = 0`.
Con 100 vnodes por shard y 10 shards: 1000 puntos en el anillo.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 9.4.2 — Rebalanceo cuando se añade un shard

**Enunciado:** Implementa la migración de datos cuando se añade un shard nuevo
al anillo de consistent hashing. Solo las claves que le corresponden al nuevo shard
deben migrar desde el shard anterior.

```go
func (a *AnilloHash) MigrarParaNuevoShard(nuevoShard *Shard) []MigracionItem {
    // retorna las (key, value) que deben moverse al nuevo shard
}
```

**Restricciones:** Mide el porcentaje de claves migradas con N=4 y N=10 shards.
Con consistent hashing, debe ser ~1/N. Con hash simple, era ~(N-1)/N.

**Pista:** Las claves que migran al nuevo shard son las que estaban en el
shard anterior cuyo hash cae entre los dos vnodes del shard nuevo más próximos.
Con 100 vnodes por shard nuevo = 100 rangos migran = 100/total_vnodes porcentaje.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.4.3 — Consistent hashing con pesos

**Enunciado:** No todos los shards tienen el mismo hardware. Un shard con
128 GB de RAM debería tener más vnodes que uno con 64 GB (recibir más datos).

```go
func (a *AnilloHash) AgregarShardConPeso(id string, shard *Shard, peso float64)
// peso = 2.0 → doble de vnodes → doble de datos
```

**Restricciones:** Con shards de pesos [1, 2, 4], verifica que la distribución
de datos es proporcional a los pesos (dentro de ±10%).

**Pista:** El número de vnodes para un shard es proporcional a su peso:
`vnodes = int(pesoPorVnode * peso)` donde `pesoPorVnode = 100` para pesos normalizados.
Un shard con peso 2 tiene 200 vnodes, uno con peso 1 tiene 100 vnodes.
La distribución resultante es proporcional al número de vnodes.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.4.4 — Consistent hashing en una caché distribuida

**Enunciado:** Implementa una caché distribuida tipo memcached usando
consistent hashing para distribuir las claves entre N nodos:

```go
type CacheDistribuida struct {
    anillo *AnilloHash
    nodos  map[string]*NodoCache
}

func (c *CacheDistribuida) Get(key string) ([]byte, bool)
func (c *CacheDistribuida) Set(key string, value []byte, ttl time.Duration) error
func (c *CacheDistribuida) Delete(key string) error
func (c *CacheDistribuida) AgregarNodo(addr string)
func (c *CacheDistribuida) EliminarNodo(addr string)
```

**Restricciones:** Cuando se elimina un nodo (fallo), las claves que tenía
van automáticamente al siguiente nodo del anillo. Los clientes hacen fallover
transparentemente. Las claves del nodo eliminado se pierden (es caché — aceptable).

**Pista:** La eliminación de un nodo en una caché distribuida es diferente
a una base de datos: no necesitas migrar datos explícitamente — simplemente
las claves ahora apuntan a otro nodo. La próxima vez que se pidan, se recalculan
y se guardan en el nuevo nodo. El costo es un aumento temporal de cache misses.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.4.5 — Consistent hashing vs rendezvous hashing

**Enunciado:** El rendezvous hashing (HRW — Highest Random Weight) es
una alternativa al consistent hashing con propiedades distintas:

```go
// Para cada clave, calcular un "score" por shard:
// score(key, shard) = hash(key + shard_id)
// El shard con mayor score recibe la clave

func (r *RendezvousHash) ShardPara(key string) *Shard {
    var mejorShard *Shard
    var mejorScore uint32

    for _, shard := range r.shards {
        score := hash(key + shard.ID)
        if score > mejorScore {
            mejorScore = score
            mejorShard = shard
        }
    }
    return mejorShard
}
```

**Restricciones:** Compara consistent hashing vs rendezvous hashing en:
- Distribución (uniformidad)
- Overhead de añadir/eliminar un shard (qué porcentaje de claves se mueve)
- Complejidad de implementación
- Performance de lookup (O(log N) vs O(N))

**Pista:** Rendezvous hashing es O(N) — para cada clave, evalúa todos los shards.
Es más simple de implementar y tiene igual distribución.
Consistent hashing es O(log N) — preferible para N grande (>100 shards).
Para N pequeño (<20), rendezvous es más práctico.

**Implementar en:** Go · Java · Python · Rust

---

## Sección 9.5 — El Teorema CAP en la Práctica

El teorema CAP (Brewer, 2000) dice que un sistema distribuido no puede
garantizar simultáneamente:

```
C — Consistency: todos los nodos ven los mismos datos al mismo tiempo
A — Availability: toda request recibe una respuesta (aunque no la más actualizada)
P — Partition tolerance: el sistema funciona aunque haya partición de red
```

**La realidad del CAP:**

```
P es obligatorio en sistemas distribuidos reales.
La red siempre puede fallar — si no toleras particiones, no tienes un sistema distribuido.
La elección real es: durante una partición de red, ¿prefieres C o A?

CP (Consistency + Partition tolerance):
  Durante una partición, rechaza requests que podrían ser inconsistentes.
  Prioriza la corrección sobre la disponibilidad.
  Ejemplos: HBase, Zookeeper, etcd, Google Spanner (en cierta medida)

AP (Availability + Partition tolerance):
  Durante una partición, responde aunque el dato pueda estar desactualizado.
  Prioriza la disponibilidad sobre la consistencia.
  Ejemplos: DynamoDB, Cassandra, CouchDB, DNS
```

**El "PACELC" (más preciso que CAP):**

```
If Partition: C vs A  (como en CAP)
Else (sin partición): Latency vs Consistency

Ejemplo:
  Google Spanner: CP + EC (alto costo de consistencia incluso sin partición)
  DynamoDB: AP + EL (disponibilidad alta, baja latencia, consistencia eventual)
```

---

### Ejercicio 9.5.1 — Simular una partición de red

**Enunciado:** Implementa un sistema de 3 nodos con partición de red simulada.
Verifica el comportamiento del sistema en modo CP vs AP:

```go
type Red struct {
    nodos     []*Nodo
    particion bool  // simular partición
}

func (r *Red) Particionar()  // dividir en dos grupos
func (r *Red) Sanar()        // restaurar la red
```

**Restricciones:** Durante la partición, el sistema CP debe rechazar escrituras
en el grupo minoritario. El sistema AP debe aceptar escrituras en ambos grupos
y reconciliar al sanar.

**Pista:** La partición divide los 3 nodos en grupos de 2 y 1 (o 1 y 2).
El grupo de 2 tiene quorum (mayoría) y puede seguir operando.
El grupo de 1 no tiene quorum — en modo CP, debe rechazar; en modo AP, puede continuar.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.5.2 — Consistency models: un espectro

**Enunciado:** El teorema CAP presenta un falso binario. En la práctica
hay un espectro de modelos de consistencia:

```
Strong (Linearizable)
  └─ Sequential consistency
       └─ Causal consistency
            └─ Read-your-writes
                 └─ Monotonic reads
                      └─ Eventual consistency
```

Implementa los tres modelos más comunes:

1. **Linearizable**: cada operación parece ejecutarse instantáneamente
2. **Causal**: las operaciones causalmente relacionadas se ven en orden
3. **Eventual**: eventualmente todos los nodos convergen al mismo valor

**Restricciones:** Para cada modelo, implementa un test que distingue
ese modelo del siguiente más débil (linearizable es detectablemente diferente
de causal, etc.).

**Pista:** La linearizabilidad requiere que las operaciones tengan un orden
total consistente con sus timestamps reales. La causalidad es más débil:
si A causa B, B se ve después de A, pero operaciones no relacionadas pueden
verse en cualquier orden. La eventual consistency solo garantiza convergencia
— sin orden garantizado.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.5.3 — Diseñar para AP: CRDTs en la práctica

**Enunciado:** Los CRDTs permiten que múltiples nodos actualicen datos
sin coordinación y converjan automáticamente. Implementa:

1. **G-Counter**: contador que solo crece (distributed counter)
2. **PN-Counter**: contador que puede crecer y decrecer
3. **G-Set**: conjunto que solo permite agregar elementos
4. **2P-Set**: conjunto que permite agregar y eliminar (usando dos G-Sets)
5. **LWW-Register**: registro con last-write-wins semántica

```go
type GCounter struct {
    contadores map[string]int64  // nodeID → valor local
}

func (c *GCounter) Incrementar(nodeID string)
func (c *GCounter) Valor() int64
func (c *GCounter) Fusionar(otro *GCounter) *GCounter  // merge sin conflictos
```

**Restricciones:** El método `Fusionar` debe ser conmutativo, asociativo, e idempotente.
Verifica que dos nodos que reciben las mismas operaciones en distinto orden
convergen al mismo valor después de la fusión.

**Pista:** La fusión de un G-Counter toma el máximo de cada contador por nodeID.
`merge(A, B).contadores[node] = max(A.contadores[node], B.contadores[node])`.
Esto es conmutativo (max(a,b) = max(b,a)), asociativo, e idempotente (max(a,a)=a).

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 9.5.4 — Google Spanner: consistencia fuerte a escala global

**Enunciado:** Google Spanner logra consistencia linearizable a escala global
usando relojes atómicos + GPS (TrueTime). Implementa una versión simplificada
del concepto de TrueTime:

```go
// TrueTime retorna un intervalo [earliest, latest] donde el tiempo real está
type TrueTime struct {
    epsilon time.Duration  // incertidumbre del reloj
}

func (tt *TrueTime) Now() (earliest, latest time.Time)
func (tt *TrueTime) After(t time.Time) bool  // ¿seguro que pasó t?
func (tt *TrueTime) Before(t time.Time) bool // ¿seguro que no llegó t?
```

Spanner espera `2*epsilon` antes de confirmar una transacción para garantizar
que su timestamp es el más alto visto por cualquier nodo.

**Restricciones:** Simula TrueTime con `epsilon = 7ms` (el valor real de Google).
Verifica que dos transacciones que usan TrueTime correctamente siempre tienen
el orden correcto incluso sin coordinación directa.

**Pista:** La clave del insight de Spanner: si esperamos 2ε antes de commitear,
cualquier otra transacción que también espera 2ε tendrá un timestamp
inequívocamente mayor o menor que el nuestro — sin ambigüedad.
Con GPS + relojes atómicos, ε es ~7ms. Sin ellos (usando NTP), ε es ~10ms.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.5.5 — Elegir CP vs AP para diferentes partes del sistema

**Enunciado:** Un sistema de e-commerce tiene distintos requisitos de consistencia
para distintas partes. Diseña y justifica la elección para cada:

1. **Inventario de productos**: ¿puedes vender el mismo producto dos veces?
2. **Carrito de compras**: ¿es crítico ver el carrito actualizado inmediatamente?
3. **Precios**: ¿puede un usuario ver un precio desactualizado?
4. **Historial de pedidos**: ¿el usuario necesita ver su pedido inmediatamente?
5. **Métricas y analytics**: ¿importa si hay un retraso de 30 segundos?

**Restricciones:** Para cada caso, implementa un sistema mínimo que demuestra
la elección (CP o AP) y sus consecuencias cuando hay una partición.

**Pista:** El inventario es claramente CP — vender más de lo disponible es un
problema de negocio serio. El carrito puede ser AP — perder un item del carrito
es un inconveniente, no un desastre. Las métricas son AP — un delay de 30s
no importa. Esta clasificación es exactamente lo que hacen los diseñadores de
sistemas como Amazon en la práctica.

**Implementar en:** Go · Java · Python

---

## Sección 9.6 — Desde el Paralelismo Local al Distribuido

El paralelismo local (Cap.08) y el distribuido (Cap.22-23) comparten
los mismos patrones conceptuales pero con diferentes garantías.

**La tabla de equivalencias:**

```
Local (Cap.08)                        Distribuido (Cap.22-23)
──────────────────                    ──────────────────────
goroutine/thread                      microservicio/proceso
canal (canal Go)                      queue (Kafka, RabbitMQ)
WaitGroup/errgroup                    distributed barrier / checkpoint
mutex                                 distributed lock (Redis, ZooKeeper)
atómico                               CAS en base de datos
Worker Pool                           task queue (Celery, Temporal)
Pipeline                              streaming pipeline (Flink, Spark Streaming)
MapReduce local                       MapReduce distribuido (Hadoop, Spark)
consistent hashing local              consistent hashing de cluster (Cassandra)
STM local                             distributed transaction (Saga, 2PC)
```

---

### Ejercicio 9.6.1 — Del Worker Pool local al distribuido

**Enunciado:** Refactoriza el Worker Pool del Cap.03 para que pueda distribuir
trabajo entre múltiples instancias del servicio:

```go
// Worker Pool local — todo en el mismo proceso
type WorkerPoolLocal struct {
    workers int
    jobs    chan Job
    results chan Result
}

// Worker Pool distribuido — workers en distintas máquinas
type WorkerPoolDistribuido struct {
    coordinador *Coordinador  // distribuye trabajo
    workers     []*WorkerRemoto  // workers remotos via gRPC
}
```

**Restricciones:** El Worker Pool distribuido debe tener la misma API externa
que el local. El coordinador usa un canal HTTP/gRPC para enviar jobs a los workers.

**Pista:** El "canal" entre el coordinador y los workers es una cola de mensajes
o una cola de tareas. Redis, RabbitMQ, o simplemente HTTP/gRPC pueden ser
la implementación. La diferencia con el canal local: los mensajes pueden perderse,
duplicarse, o llegar fuera de orden.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.6.2 — MapReduce distribuido (simplificado)

**Enunciado:** Extiende el MapReduce local del Ejercicio 8.3.4 para que
los workers sean procesos separados (en distintos puertos):

```go
// Coordinador
type CoordinadorMapReduce struct {
    workers  []string  // direcciones de workers
    tareas   []Tarea
    estado   map[int]EstadoTarea
}

// Worker
type WorkerMapReduce struct {
    addr string
}
```

**Restricciones:** El coordinador reasigna tareas de workers que fallan.
Un worker que tarda más del doble del tiempo promedio es reiniciado.
Los resultados son correctos aunque un worker falle a mitad de su tarea.

**Pista:** Este es el modelo del paper de Google MapReduce (2004).
El coordinador mantiene el estado de cada tarea (pendiente, en progreso, completado).
Las tareas "en progreso" se reasignan si el worker no reporta progreso en X segundos.
La idempotencia de las tareas es clave — ejecutar la misma tarea dos veces
produce el mismo resultado.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.6.3 — Consistent hashing en un cluster real

**Enunciado:** Combina el consistent hashing del Ejercicio 9.4.1 con
un cluster de servicios HTTP reales (distintos puertos en la misma máquina):

```go
type ClusterDistribuido struct {
    anillo   *AnilloHash
    nodos    map[string]*http.Client  // addr → cliente HTTP
}

func (c *ClusterDistribuido) Get(key string) ([]byte, error)
func (c *ClusterDistribuido) Set(key string, value []byte) error
func (c *ClusterDistribuido) AgregarNodo(addr string)
func (c *ClusterDistribuido) EliminarNodo(addr string)
```

**Restricciones:** Cuando se elimina un nodo, el sistema sigue funcionando
(las claves de ese nodo van al siguiente del anillo). Implementa gossip protocol
básico para que los nodos se notifiquen entre sí cuando alguien se une o sale.

**Pista:** El gossip protocol es cómo los sistemas distribuidos diseminan
información sin un coordinador central. Cada nodo periodicamente elige
un nodo aleatorio y le envía su lista de nodos conocidos. En pocas rondas,
todos los nodos tienen la información actualizada.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.6.4 — Pipeline de datos: local vs distribuido

**Enunciado:** El pipeline del Cap.03 §3.2 (etapas conectadas por canales)
tiene un análogo distribuido con Kafka entre etapas:

```
Local:
  Leer → [canal] → Parsear → [canal] → Validar → [canal] → Escribir

Distribuido (con Kafka):
  Servicio Lector → [topic raw-data] → Servicio Parser → [topic parsed] → ...
```

Implementa ambos y compara:
- Throughput máximo
- Latencia end-to-end
- Comportamiento cuando una etapa falla
- Escalabilidad horizontal de cada etapa

**Restricciones:** El pipeline distribuido debe usar HTTP como sustituto de Kafka
(un endpoint `/publish` y un endpoint `/subscribe`). Cada etapa puede tener
múltiples instancias.

**Pista:** La diferencia clave: en el pipeline local, si una etapa falla,
el pipeline entero se detiene. En el pipeline distribuido con una queue,
las otras etapas siguen procesando los mensajes ya encolados. El pipeline
distribuido puede tener distintos números de instancias por etapa (escala independiente).

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.6.5 — Diseño: ¿cuándo distribuir el Worker Pool?

**Enunciado:** El Worker Pool local es suficiente para muchos casos.
Define los criterios de cuándo necesitas el Worker Pool distribuido:

**Implementa un sistema que automáticamente decide:**
- Si el trabajo cabe en una máquina → Worker Pool local
- Si necesita más de una máquina → Worker Pool distribuido

```go
type AutoScaler struct {
    local      *WorkerPoolLocal
    distribuido *WorkerPoolDistribuido
    umbralCPU  float64
    umbralMem  float64
}

func (a *AutoScaler) Submit(job Job) error
func (a *AutoScaler) EscalarSiNecesario()
```

**Restricciones:** El AutoScaler monitorea CPU y memoria cada 10 segundos.
Si cualquiera supera el umbral, activa el modo distribuido.
Cuando la carga baja, vuelve al modo local.

**Pista:** El umbral de CPU es claro. El de memoria es más sutil — la memoria
se libera lentamente por el GC. Una mejor señal: latencia p99 del Worker Pool.
Si la latencia supera el SLA, necesitas más recursos (más workers locales o distribuidos).

**Implementar en:** Go · Java · Python

---

## Sección 9.7 — Cuándo NO Distribuir

La distribución resuelve problemas de escala pero introduce complejidad.
La regla es: no distribuyas hasta que sea necesario.

**Los costos ocultos de distribuir:**

```
Latencia de red:
  Llamada local:     ~1 µs
  Llamada HTTP:      ~500 µs
  Llamada cross-DC:  ~50 ms
  → Cada hop añade latencia no recuperable

Consistencia:
  Transacciones locales son ACID gratuitas
  Transacciones distribuidas requieren Saga o 2PC — complejidad enorme

Observabilidad:
  Un servicio: 1 log, 1 trace
  10 servicios: distributed tracing, correlación de logs — mucho más difícil

Operaciones:
  1 servicio: 1 deployment, 1 runbook
  10 servicios: 10 deployments, gestión de versiones, compatibility matrix

Testing:
  1 servicio: tests unitarios + integración simples
  10 servicios: tests de integración entre servicios, contract testing
```

---

### Ejercicio 9.7.1 — Monolito vs microservicios: el experimento

**Enunciado:** Implementa el mismo sistema (un procesador de pedidos simple)
como monolito y como dos microservicios. Mide:

1. Latencia end-to-end para procesar 1 pedido
2. Throughput máximo (pedidos/segundo)
3. Tiempo de deployment
4. Dificultad de debugging cuando algo falla

**Restricciones:** El monolito y los microservicios tienen exactamente la
misma lógica de negocio. La única diferencia es el mecanismo de comunicación
(llamada local vs HTTP).

**Pista:** La latencia del monolito será ~100x menor que los microservicios
para llamadas simples. El throughput del monolito será mayor (no hay overhead de red).
El único argumento a favor de los microservicios en este nivel es el despliegue
independiente — que solo importa si los equipos son grandes y trabajan en paralelo.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.7.2 — El escalado vertical: ¿cuánto puedes crecer sin distribuir?

**Enunciado:** Antes de distribuir, ¿puedes simplemente comprar una máquina más grande?

Para un servidor web con 10,000 req/s y 1TB de datos:
- Una máquina de $500/mes con 8 cores, 32 GB RAM → ¿soporta la carga?
- Una máquina de $5,000/mes con 96 cores, 512 GB RAM → ¿soporta la carga?
- Una máquina de $50,000/mes con 192 cores, 4 TB RAM → ¿soporta la carga?

Implementa el servidor y mide qué tamaño de máquina (simulada) es el cuello de botella.

**Restricciones:** Simula distintas "máquinas" ajustando `GOMAXPROCS` y el
tamaño máximo de la caché. Determina el punto donde el escalado vertical no basta.

**Pista:** Para la mayoría de startups, el escalado vertical es suficiente
hasta millones de usuarios. WhatsApp servía 900M usuarios con 50 ingenieros y
pocas máquinas bien verticalmente escaladas. La distribución prematura es
"la raíz de todos los males" aplicada a arquitectura.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.7.3 — Identificar los cuellos de botella reales

**Enunciado:** Antes de distribuir, instrumenta el sistema para identificar
qué parte es el cuello de botella:

```go
type InstrumentacionPerfil struct {
    tiemposBD     []time.Duration
    tiemposCache  []time.Duration
    tiemposCPU    []time.Duration
    tiemposRed    []time.Duration
}
```

Para el sistema de e-commerce del Ejercicio 9.5.5, determina:
¿Es el cuello de botella la BD, la caché, el CPU, o la red?
¿Qué parte se beneficia más de distribuir y cuál de optimizar?

**Restricciones:** El perfil debe tener overhead < 1% (no degradar el sistema al medirlo).
Usa sampling (medir 1 de cada 100 requests) si es necesario.

**Pista:** La mayoría de los sistemas web son I/O-bound — la BD es el cuello de botella.
La solución es a menudo agregar caché (Redis), no distribuir la BD.
Distribuir la BD (sharding) es el último recurso — primero optimiza las queries,
luego agrega réplicas de lectura, luego caché, y solo si todo eso falla considera sharding.

**Implementar en:** Go · Java · Python

---

### Ejercicio 9.7.4 — El costo de la complejidad distribuida

**Enunciado:** Implementa un sistema distribuido simple (3 servicios, 1 cola de mensajes)
y mide el tiempo que tarda en:
1. Hacer un deployment completo
2. Debuggear un fallo en producción (con logs distribuidos)
3. Escribir y ejecutar un test de integración
4. Entender el comportamiento end-to-end de una request

**Restricciones:** Usa docker-compose para simular el entorno distribuido.
Incluye distributed tracing (OpenTelemetry) para el debugging.
Mide el tiempo real, no estimado.

**Pista:** El ejercicio es deliberadamente "doloroso" — ese es el punto.
La complejidad operacional de los microservicios es real y frecuentemente
subestimada. La pregunta no es "¿microservicios son malos?" sino
"¿el costo de esta complejidad está justificado por los beneficios en mi caso?"

**Implementar en:** Go · Java · Python (con docker-compose)

---

### Ejercicio 9.7.5 — Diseño holístico: sistema completo de inicio a escala

**Enunciado:** Diseña la arquitectura de un sistema de notificaciones en
tres fases, comenzando simple y distribuyendo solo cuando sea necesario:

**Fase 1 (0-10K usuarios): monolito**
```
- Servidor HTTP único
- BD local (PostgreSQL)
- Envío de emails síncronos
- Una máquina, despliegue simple
```

**Fase 2 (10K-1M usuarios): separar por dominio**
```
- ¿Qué cambiar primero y por qué?
- ¿Cuál es el primer cuello de botella real?
```

**Fase 3 (1M-100M usuarios): distribuir lo necesario**
```
- ¿Qué no puede escalar verticalmente?
- ¿Qué si puede?
```

**Restricciones:** Implementa la Fase 1 completa. Para las Fases 2 y 3,
escribe el diseño técnico justificando cada decisión con números concretos.

**Pista:** El primer cuello de botella de un sistema de notificaciones suele ser
el envío de emails (rate limiting del proveedor SMTP, no el servidor).
La solución es una cola asíncrona de emails — sin necesidad de distribuir el servidor.
El segundo cuello de botella suele ser la BD — antes de sharding, agrega read replicas
y optimiza queries. El servidor HTTP raramente es el cuello de botella primero.

**Implementar en:** Go · Java · Python

---

## Resumen del capítulo

**Los tres límites y cuándo distribuir:**

| Límite | Síntoma | Solución |
|---|---|---|
| CPU | CPU 100% en pico, latencia alta | Escalar verticalmente primero, luego distribuir |
| Memoria | RAM llena, OOM | Sharding (si los datos no caben en ninguna máquina grande) |
| Disponibilidad | Downtime inaceptable | Replicación + failover |

**El árbol de decisión antes de distribuir:**

```
¿Hay un problema real de escala?
    No → no distribuyas
    Sí → ¿puedes resolver con una máquina más grande? (escala vertical)
           Sí → hazlo (más simple, más barato, más rápido)
           No → ¿es el cuello de botella la BD, la caché, el CPU, o la red?
                  BD → añade réplicas de lectura, caché, optimiza queries
                  CPU → sharding de compute (Worker Pool distribuido)
                  Memoria → sharding de datos (consistent hashing)
                  Disponibilidad → replicación + failover
```

**La Ley de Conway aplicada:**

> "Las organizaciones que diseñan sistemas están condicionadas a producir
> diseños que copian las estructuras de comunicación de esas organizaciones."

Si tienes 50 ingenieros en un equipo, un monolito bien estructurado es
probablemente mejor que 50 microservicios. Los microservicios tienen sentido
cuando los equipos son independientes y necesitan desplegar de forma autónoma.

## La pregunta que guía el Cap.10

> La Parte 2 cubre el paralelismo local (Cap.08) y el límite entre local y distribuido (Cap.09).
> Los Cap.10-13 profundizan en patrones de paralelismo avanzados:
> paralelismo de tareas heterogéneas, GPU computing, algoritmos numéricos paralelos,
> y el puente final hacia sistemas distribuidos de producción.
>
> El Cap.10 entra en el paralelismo de tareas: cómo paralelizar trabajo que
> no es fácilmente divisible en chunks iguales — el problema más común en producción.
