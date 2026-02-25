# Guía de Ejercicios — Cap.22: De Memoria Compartida a Paso de Mensajes

> Lenguaje principal: **Go**. Los ejemplos usan la red real (TCP/gRPC) o
> la simulan con goroutines + canales cuando el objetivo es el algoritmo,
> no la infraestructura.
>
> Este capítulo asume que dominas los capítulos anteriores.
> Los ejercicios son más grandes — algunos son sistemas completos de 200–400 líneas.
> El objetivo no es memorizar los algoritmos: es entender por qué existen.

---

## El salto conceptual

Todo lo anterior asumía una propiedad que en este capítulo desaparece:

```
Concurrencia (Cap.01–21):
  - Múltiples goroutines en el mismo proceso
  - Comparten la misma memoria (heap)
  - La comunicación es instantánea (nanosegundos)
  - Los fallos son totales: si el proceso muere, muere todo junto
  - sync.Mutex garantiza que solo una goroutine accede a un dato a la vez

Distribución (Cap.22–23):
  - Múltiples procesos en distintas máquinas
  - NO comparten memoria — cada proceso tiene su propio heap
  - La comunicación es por red (milisegundos, puede fallar, puede reordenarse)
  - Los fallos son parciales: un nodo puede caer sin que los demás lo sepan
  - No hay un mutex global — coordinar requiere protocolos explícitos
```

Este cambio hace que problemas "resueltos" en concurrencia vuelvan a ser difíciles:

```
Problema: "¿cuál es el valor actual del contador?"

Con memoria compartida:
  mutex.Lock()
  valor := contador
  mutex.Unlock()
  // 'valor' es el valor actual — garantizado

Con memoria distribuida:
  valor := pedirValorAlNodo()
  // 'valor' era el valor cuando el nodo respondió
  // ¿Qué pasa si otro nodo lo cambió mientras la respuesta viajaba por la red?
  // ¿Qué pasa si el nodo que respondió está desconectado del quórum?
  // No hay respuesta simple — depende de qué garantías quieres
```

---

## La taxonomía de los fallos distribuidos

```
Fallo de crash (crash fault):
  El nodo para de responder completamente.
  Los otros nodos eventualmente detectan que no responde.
  Es el fallo más simple — y el que la mayoría de algoritmos asume.

Fallo de red (network partition):
  El nodo sigue funcionando pero no puede comunicarse con algunos otros.
  Los nodos en las dos particiones no saben que el otro sigue vivo.
  Es el fallo más peligroso — causa "split brain".

Fallo bizantino (Byzantine fault):
  El nodo responde, pero con datos incorrectos o maliciosos.
  El fallo más difícil — requiere protocolos especiales (BFT).
  Relevante en blockchains; raro en sistemas internos de confianza.

Fallo de omisión (omission fault):
  El nodo recibe mensajes pero no responde a algunos.
  Difícil de distinguir de un fallo de red desde fuera.
```

Los ejercicios de este capítulo asumen crash faults y network partitions.
Los fallos bizantinos quedan para el libro de Herlihy (mencionado en el epílogo).

---

## Tabla de contenidos

- [Sección 22.1 — Comunicación por red: TCP, gRPC, y mensajes](#sección-221--comunicación-por-red-tcp-grpc-y-mensajes)
- [Sección 22.2 — Detección de fallos: heartbeats y timeouts](#sección-222--detección-de-fallos-heartbeats-y-timeouts)
- [Sección 22.3 — Relojes y causalidad: el tiempo en sistemas distribuidos](#sección-223--relojes-y-causalidad-el-tiempo-en-sistemas-distribuidos)
- [Sección 22.4 — Consistencia: el espectro de garantías](#sección-224--consistencia-el-espectro-de-garantías)
- [Sección 22.5 — Consenso: acordar en presencia de fallos](#sección-225--consenso-acordar-en-presencia-de-fallos)
- [Sección 22.6 — Replicación: mantener copias sincronizadas](#sección-226--replicación-mantener-copias-sincronizadas)
- [Sección 22.7 — El teorema CAP: eligiendo dos de tres](#sección-227--el-teorema-cap-eligiendo-dos-de-tres)

---

## Sección 22.1 — Comunicación por Red: TCP, gRPC, y Mensajes

La primera diferencia con la memoria compartida: la comunicación puede fallar,
reordenarse, y duplicarse.

```
Propiedades de TCP:
  ✓ Confiable: los bytes llegan o la conexión se rompe (no hay pérdida silenciosa)
  ✓ Ordenado: los bytes llegan en el orden en que se enviaron
  ✗ No atómico: puede fallar a mitad de un mensaje
  ✗ No tiene semántica de "exactamente una vez" — puede enviar el mismo byte dos veces
    si hay reconexión después de un fallo parcial

Propiedades de un mensaje sobre TCP (ej: gRPC):
  ✓ Delimita mensajes (TCP es stream, gRPC añade framing)
  ✓ At-most-once si la conexión falla: el mensaje se envió o no, no ambas
  ✗ Si el servidor procesa el mensaje Y luego la conexión falla ANTES de responder,
    el cliente no sabe si el servidor procesó → al reintentar puede haber duplicado
```

### Ejercicio 22.1.1 — Implementar un servidor TCP que maneja reconexiones

```go
// El problema: un cliente envía "incrementar contador" y la conexión falla.
// ¿El servidor procesó el mensaje? El cliente no sabe.
// Al reconectar y reenviar, ¿el contador se incrementa una o dos veces?

type ServidorTCP struct {
    ln      net.Listener
    contador atomic.Int64
}

func (s *ServidorTCP) Escuchar(addr string) error {
    ln, err := net.Listen("tcp", addr)
    if err != nil { return err }
    s.ln = ln

    for {
        conn, err := ln.Accept()
        if err != nil { return err }
        go s.manejarConexion(conn)
    }
}

func (s *ServidorTCP) manejarConexion(conn net.Conn) {
    defer conn.Close()
    decoder := json.NewDecoder(conn)
    encoder := json.NewEncoder(conn)

    for {
        var msg Mensaje
        if err := decoder.Decode(&msg); err != nil {
            return  // conexión cerrada o error — no sabe qué procesó
        }

        resultado := s.procesarMensaje(msg)
        if err := encoder.Encode(resultado); err != nil {
            return  // procesó pero no pudo responder
        }
    }
}
```

**Restricciones:**
1. Implementar el servidor y el cliente
2. Simular un fallo de red justo después de que el servidor procese pero antes de responder
3. Mostrar que sin idempotency keys el contador puede incrementarse dos veces
4. Implementar idempotency keys para hacer el incremento exactamente-una-vez

**Pista:** La idempotency key en el servidor requiere guardar qué keys ya procesó.
Si `key = uuid` del cliente y el servidor ya procesó esa key, retornar el resultado
guardado en lugar de procesar de nuevo. El servidor necesita un mapa de
`key → resultado` con TTL (las keys viejas se pueden limpiar).

---

### Ejercicio 22.1.2 — gRPC: comunicación tipada con protobuf

gRPC resuelve el framing (delimitar mensajes en TCP) y añade tipos:

```protobuf
// contador.proto
syntax = "proto3";

service ContadorService {
  rpc Incrementar(IncrRequest) returns (IncrResponse);
  rpc ObtenerValor(Empty) returns (ValorResponse);
  rpc StreamCambios(Empty) returns (stream CambioEvent);  // servidor → cliente streaming
}

message IncrRequest {
  string idempotency_key = 1;
  int64 delta = 2;
}

message IncrResponse {
  int64 nuevo_valor = 1;
  bool fue_duplicado = 2;
}
```

**Restricciones:** Implementar el servidor gRPC del ContadorService con:
- Idempotency keys guardadas en Redis (con TTL de 24 horas)
- Streaming de cambios: cuando el contador cambia, notificar a todos los subscribers
- Manejo correcto de desconexiones de clientes en streaming

**Pista:** El servidor streaming de gRPC usa `stream.Send()` que retorna error
cuando el cliente desconecta. Manejar el error limpiamente (sin panic).
Para múltiples subscribers: un fan-out pattern como el del Cap.05 §5.4,
pero donde cada subscriber es una conexión gRPC activa.

---

### Ejercicio 22.1.3 — Leer: ¿qué puede salir mal con los mensajes?

**Tipo: Diagnosticar**

Para cada escenario, determina si el sistema queda en un estado correcto
o incorrecto, y por qué:

```
Sistema: dos nodos (A y B). A envía "transferir $100 de cuenta X a cuenta Y" a B.

Escenario 1:
  A envía el mensaje → B lo recibe → B procesa (debita X, acredita Y) → B responde OK
  → A recibe OK
  Resultado: ???

Escenario 2:
  A envía el mensaje → B lo recibe → B procesa → [fallo de red] → A no recibe respuesta
  → A reintenta después de timeout → B recibe el mensaje de nuevo
  Resultado sin idempotency keys: ???
  Resultado con idempotency keys: ???

Escenario 3:
  A envía el mensaje → [fallo de red] → B nunca recibe el mensaje
  → A reintenta → B recibe y procesa → B responde OK → A recibe OK
  Resultado: ???

Escenario 4:
  A envía el mensaje → B lo recibe → B debita X → [crash de B antes de acreditar Y]
  → B reinicia → A reintenta → B recibe el mensaje de nuevo
  Resultado sin transacción: ???
  Resultado con transacción: ???
```

**Pista:** El Escenario 4 muestra el problema de la transacción distribuida:
debitar una cuenta y acreditar otra son dos operaciones que deben ser atómicas.
Si B crashea entre ellas, el dinero desaparece. Las soluciones: saga pattern
(con compensación), two-phase commit (2PC), o diseñar el sistema para que sea
idempotente y el reintento del Escenario 4 no duplique el crédito.

---

### Ejercicio 22.1.4 — Serialización y versionado de mensajes

Los sistemas distribuidos viven durante años. Los mensajes deben poder
evolucionar sin romper compatibilidad:

```go
// Versión 1 del mensaje (2023):
type PedidoV1 struct {
    ID       string
    UserID   string
    Items    []Item
    Total    float64
}

// Versión 2 del mensaje (2024) — añade campos:
type PedidoV2 struct {
    ID       string
    UserID   string
    Items    []Item
    Total    float64
    Moneda   string  // nuevo campo — ¿qué valor tiene para mensajes V1?
    Dirección *Dirección  // nuevo campo opcional
}
```

**Restricciones:** Implementar un sistema de serialización que:
- Permite que un servidor V2 procese mensajes V1 (con valores por defecto)
- Permite que un servidor V1 ignore los campos V2 sin crashear
- Detecta mensajes incompatibles (breaking changes) en tiempo de compilación si es posible

**Pista:** Protocol Buffers (protobuf) maneja esto automáticamente: campos nuevos
son opcionales y tienen valores por defecto; campos desconocidos se ignoran.
Para JSON: usar `json.RawMessage` para campos opcionales que pueden no estar presentes.
El "forward compatibility" (V1 ignora campos V2) es más difícil que el
"backward compatibility" (V2 provee defaults para campos V1 ausentes).

---

### Ejercicio 22.1.5 — Message queue: desacoplar productor de consumidor

En lugar de RPC síncrono (esperar respuesta), el message queue permite
comunicación asíncrona:

```
RPC síncrono:
  Productor → [espera] → Consumidor → [espera] → Productor
  Ventaja: el productor sabe inmediatamente si tuvo éxito
  Desventaja: el productor y consumidor deben estar disponibles simultáneamente

Message queue:
  Productor → Queue → [consumidor puede estar caído] → Consumidor (cuando vuelve)
  Ventaja: desacoplamiento temporal — el productor no necesita esperar
  Desventaja: el productor no sabe cuándo se procesó; entrega at-least-once por defecto
```

**Restricciones:** Implementar una message queue simplificada en Go:
- Persistencia en disco (los mensajes sobreviven un restart del proceso)
- Al menos una semántica: garantizar que cada mensaje se entrega al menos una vez
- Acknowledgment: el consumidor confirma que procesó antes de que el mensaje se elimine
- Dead letter queue: mensajes que fallan N veces van a una queue separada

**Pista:** La persistencia más simple: un append-only log en disco (un archivo
donde cada línea es un mensaje JSON). Los mensajes "procesados" se marcan
(no se borran — borrar en el medio es costoso) y periódicamente se compacta el archivo.
Este es exactamente el diseño de Kafka y de WAL (Write-Ahead Log) en bases de datos.

---

## Sección 22.2 — Detección de Fallos: Heartbeats y Timeouts

En un sistema distribuido, saber si un nodo está caído es más difícil de lo que parece.

```
El problema de la detección de fallos:
  Si el nodo B no responde en 5 segundos, ¿está:
    (a) caído?
    (b) muy lento (GC pause, CPU saturado)?
    (c) procesando tu mensaje pero la respuesta se perdió?
    (d) en una partición de red pero funcionando normalmente?

  Desde la perspectiva de A, (a), (b), (c), y (d) son indistinguibles.
  No hay forma de saber cuál es sin información adicional.
```

### Ejercicio 22.2.1 — Implementar heartbeat con detección de fallos

```go
type DetectorFallos struct {
    nodos    map[string]*EstadoNodo
    mu       sync.RWMutex
    timeout  time.Duration
    intervalo time.Duration
}

type EstadoNodo struct {
    ultimoHeartbeat time.Time
    activo          bool
}

// El nodo B envía heartbeats al detector:
func (d *DetectorFallos) RecibirHeartbeat(nodoID string) {
    d.mu.Lock()
    defer d.mu.Unlock()
    if estado, ok := d.nodos[nodoID]; ok {
        estado.ultimoHeartbeat = time.Now()
        estado.activo = true
    }
}

// El detector verifica periódicamente:
func (d *DetectorFallos) verificar() {
    d.mu.Lock()
    defer d.mu.Unlock()
    ahora := time.Now()
    for id, estado := range d.nodos {
        if ahora.Sub(estado.ultimoHeartbeat) > d.timeout {
            if estado.activo {
                log.Warn("nodo posiblemente caído", "nodo", id)
                estado.activo = false
                // Notificar a quien corresponda
            }
        }
    }
}
```

**Restricciones:**
- Implementar con phi accrual failure detection: en lugar de un umbral binario,
  calcular la probabilidad de que el nodo esté caído basándose en el historial
  de intervalos entre heartbeats
- Verificar con un test que el detector no hace falsos positivos durante
  un GC pause de 500ms con timeout configurado de 1 segundo

**Pista:** El phi accrual detector (usado por Akka y Cassandra) calcula `φ(t)`:
si `φ > umbral` (ej: φ > 8), el nodo se considera caído. φ se calcula
basándose en la distribución histórica de los intervalos entre heartbeats.
Un GC pause causa un heartbeat tardío — φ sube pero puede no superar el umbral.
Un nodo caído causa que φ crezca indefinidamente.

---

### Ejercicio 22.2.2 — Leer: falso positivo de detección de fallos

**Tipo: Diagnosticar**

Este log muestra un incidente en un cluster de 3 nodos:

```
[14:00:00] Nodo A: heartbeat de B recibido
[14:00:00] Nodo A: heartbeat de C recibido
[14:00:01] Nodo A: heartbeat de B recibido
[14:00:01] Nodo A: heartbeat de C recibido
[14:00:02] Nodo A: heartbeat de B recibido
[14:00:02] Nodo A: heartbeat de C recibido
[14:00:03] Nodo A: heartbeat de B recibido
[14:00:03] Nodo A: NO heartbeat de C  ← C está haciendo GC (500ms pause)
[14:00:04] Nodo A: NO heartbeat de C  ← timeout configurado = 1s
[14:00:04] Nodo A: !! C marcado como CAÍDO
[14:00:04] Nodo A: iniciando failover de C → asumiendo liderazgo de datos de C
[14:00:04] Nodo C: GC pause terminada, enviando heartbeat
[14:00:04] Nodo C: recibiendo requests que eran de A (confusión)
[14:00:04] Nodo A: recibiendo heartbeat de C  ← C "resucitó"
[14:00:04] Nodo A: C marcado como ACTIVO
[14:00:04] !! Dos nodos asumieron ser líder simultáneamente → split brain
```

**Preguntas:**

1. ¿Qué evento causó el falso positivo?
2. ¿Por qué el timeout de 1 segundo es demasiado corto para este sistema?
3. ¿Qué consecuencias tiene el "split brain" en un sistema de base de datos?
4. ¿Cómo el phi accrual detector habría manejado esto mejor?
5. ¿Qué garantía mínima necesitas para prevenir el split brain incluso con
   detección de fallos imperfecta?

**Pista:** El split brain es el mayor riesgo de un detector de fallos agresivo.
Si dos nodos piensan que son el líder, ambos pueden aceptar escrituras —
y el estado diverge. La garantía para prevenir split brain: el líder necesita
una "lease" del quórum (mayoría de nodos deben confirmar que sigue siendo líder
periódicamente). Sin la confirmación del quórum, el líder deja de aceptar escrituras.
Esto es lo que Raft llama el "leader lease".

---

### Ejercicio 22.2.3 — SWIM: detección de fallos escalable

El heartbeat centralizado (todos reportan a un servidor) no escala:

```
Con N nodos y heartbeats cada segundo:
  Mensajes por segundo = N (un heartbeat por nodo)
  Estado del servidor central: O(N)
  Si el servidor cae, la detección falla

SWIM (Scalable Weakly-consistent Infection-style Membership):
  Cada nodo hace ping a un nodo aleatorio cada T segundos
  Si no responde, pide a K nodos que hagan ping de su parte (indirect ping)
  Si ninguno responde, el nodo se marca como "sospechoso"
  Si después de un tiempo sigue sospechoso, se marca como "caído"

  Mensajes por segundo = N (distribuidos entre todos los nodos)
  Sin servidor central
  Usado por: Consul, memberlist, Kubernetes
```

**Restricciones:** Implementar una versión simplificada de SWIM con:
- Ping directo cada 1 segundo a un nodo aleatorio
- Indirect ping a 3 nodos si el directo falla
- Estado: vivo → sospechoso → caído (con tiempos configurables)
- Propagación del estado por "gossip": cada mensaje incluye actualizaciones recientes

**Pista:** El gossip protocol se implementa piggy-backing: cuando A envía un ping a B,
incluye en el mismo mensaje las últimas K actualizaciones de estado que conoce
(ej: "C está sospechoso", "D resucitó"). B procesa el ping Y las actualizaciones.
Cada actualización tiene un counter de "generación" para que actualizaciones
viejas no sobreescriban a las nuevas.

---

### Ejercicio 22.2.4 — El timeout como contrato

En un sistema distribuido, el timeout es un contrato entre el cliente y el servidor:

```go
// El cliente promete: si no recibes respuesta en T, puedes asumir que fallé.
// El servidor puede: reintentar si ve que T no ha pasado.

type RequestConDeadline struct {
    Payload  interface{}
    Deadline time.Time  // cuándo el cliente dejará de esperar
}

func servidor(req RequestConDeadline) Response {
    // Si el deadline ya pasó, no procesar (el cliente ya no espera):
    if time.Now().After(req.Deadline) {
        return Response{Err: "deadline already passed"}
    }

    // Calcular cuánto tiempo queda:
    tiempoRestante := time.Until(req.Deadline)

    // Crear contexto con el tiempo restante:
    ctx, cancel := context.WithDeadline(context.Background(), req.Deadline)
    defer cancel()

    return procesarConContexto(ctx, req.Payload)
}
```

**Restricciones:** Implementar el sistema donde el deadline se propaga como
parte del mensaje (no solo como timeout de conexión TCP). Verificar que:
- Si el deadline llega durante el procesamiento, el servidor para de procesar
- El cliente no espera más allá del deadline
- Los recursos del servidor se liberan cuando el deadline vence

**Pista:** Pasar el deadline en el mensaje es necesario cuando hay múltiples
saltos (cliente → servicio A → servicio B). Si A sabe que el deadline del cliente
original es en 500ms, puede configurar el timeout de su llamada a B
en `deadline - overhead_de_A` en lugar de usar un timeout fijo propio.
Esto es exactamente lo que hace gRPC con `grpc.MethodConfig.Timeout`.

---

### Ejercicio 22.2.5 — Liveness vs safety en la detección de fallos

**Tipo: Diseñar**

Todo detector de fallos tiene un tradeoff fundamental:

```
Completeness (liveness): todo nodo caído eventualmente es detectado
Accuracy (safety): ningún nodo vivo es declarado caído

No puedes tener ambas propiedades perfectas en un sistema asíncrono
(esto es el resultado de Fischer, Lynch y Paterson, 1985 — el teorema FLP).

En la práctica:
  Sistema CP-biased (prefiere safety):
    → Timeout largo, pocas falsas alarmas, pero detección lenta
    → Apropiado cuando el costo de un falso positivo es alto (ej: BD)

  Sistema AP-biased (prefiere liveness):
    → Timeout corto, detección rápida, pero más falsos positivos
    → Apropiado cuando el costo de no detectar es alto (ej: load balancer)
```

**Restricciones:** Para cada uno de estos sistemas, determina cuál bias
es más apropiado y justifica los valores de timeout:

1. Base de datos distribuida con replicación síncrona
2. Load balancer que distribuye tráfico HTTP
3. Sistema de monitoreo de salud de microservicios
4. Coordinador de transacciones distribuidas
5. Caché distribuido con replicación asíncrona

**Pista:** La regla general: si el costo de declarar vivo a un nodo caído
(false negative) es mayor que el costo de declarar caído a un nodo vivo
(false positive), prefiere liveness (timeout corto). Para BD: declarar
caído a un nodo que está vivo puede causar split brain — prefiere safety.
Para load balancer: no detectar un backend caído significa mandar tráfico
a un servidor que no responde — prefiere liveness.

---

## Sección 22.3 — Relojes y Causalidad: el Tiempo en Sistemas Distribuidos

```
El problema de los relojes distribuidos:
  Nodo A envía mensaje a las 10:00:00.000 (su reloj)
  Nodo B recibe el mensaje a las 09:59:59.998 (su reloj)

  ¿El mensaje llegó ANTES de que se enviara?
  Imposible físicamente, pero posible según los relojes.

  Causa: los relojes de distintas máquinas no están sincronizados perfectamente.
  NTP puede sincronizar con precisión de ~1-10ms.
  Google Spanner usa relojes atómicos + GPS: ~7ns de error.
  La mayoría de sistemas: asumir que los relojes no son confiables para ordenar eventos.
```

### Ejercicio 22.3.1 — Implementar Lamport timestamps

Leslie Lamport (1978) propuso una forma de ordenar eventos causalmente sin
depender de los relojes del sistema:

```
Reglas:
  1. Cada nodo tiene un contador local (Lamport clock), inicialmente 0.
  2. Antes de un evento local: incrementar el contador.
  3. Antes de enviar un mensaje: incrementar y adjuntar el contador al mensaje.
  4. Al recibir un mensaje con timestamp T:
     reloj_local = max(reloj_local, T) + 1

Propiedad: si A → B (A causó B), entonces timestamp(A) < timestamp(B).
Nota: timestamp(A) < timestamp(B) NO implica que A causó B.
```

```go
type LamportClock struct {
    tiempo atomic.Int64
}

func (lc *LamportClock) Tick() int64 {
    return lc.tiempo.Add(1)
}

func (lc *LamportClock) Update(tiempoRecibido int64) int64 {
    for {
        actual := lc.tiempo.Load()
        nuevo := max(actual, tiempoRecibido) + 1
        if lc.tiempo.CompareAndSwap(actual, nuevo) {
            return nuevo
        }
    }
}
```

**Restricciones:** Implementar tres nodos que se comunican y demostrar:
1. Si A envía a B, el timestamp de B > timestamp de A
2. Los timestamps son totalmente ordenados (con tie-breaking por nodo ID)
3. Dos eventos concurrentes (sin relación causal) pueden tener cualquier orden

**Pista:** Para el punto 3, "concurrente" significa que A no causó B ni B causó A.
Dos escrituras simultáneas a la BD desde nodos distintos son concurrentes.
Los Lamport timestamps dan un orden total pero arbitrario para eventos concurrentes —
no hay un orden "correcto" para eventos sin relación causal.

---

### Ejercicio 22.3.2 — Vector clocks: causalidad precisa

Los Lamport clocks ordenan eventos pero no detectan la concurrencia.
Los vector clocks sí:

```go
// Un vector clock es un array de contadores, uno por nodo:
type VectorClock struct {
    relojes map[string]int64  // nodeID → tiempo lógico
    nodeID  string
}

func (vc *VectorClock) Tick() VectorClock {
    copia := vc.copia()
    copia.relojes[vc.nodeID]++
    return copia
}

func (vc *VectorClock) Merge(otro VectorClock) VectorClock {
    copia := vc.copia()
    for nodo, tiempo := range otro.relojes {
        if tiempo > copia.relojes[nodo] {
            copia.relojes[nodo] = tiempo
        }
    }
    copia.relojes[vc.nodeID]++
    return copia
}

// Comparación:
// A happensBefore B: para todo nodo, A[nodo] <= B[nodo], y existe nodo donde A[nodo] < B[nodo]
// A concurrent B: ni A happensBefore B, ni B happensBefore A
func (a VectorClock) HappensBefore(b VectorClock) bool {
    menorEnAlguno := false
    for nodo, tiempoA := range a.relojes {
        tiempoB := b.relojes[nodo]
        if tiempoA > tiempoB { return false }  // a no puede haber ocurrido antes
        if tiempoA < tiempoB { menorEnAlguno = true }
    }
    return menorEnAlguno
}
```

**Restricciones:** Implementar un sistema de versionado de documentos con
vector clocks. Cuando dos usuarios editan el mismo documento simultáneamente
(conflicto), el sistema debe detectarlo (en lugar de silenciosamente sobreescribir).

**Pista:** Los vector clocks son la base del control de versiones distribuido
(Git usa un concepto similar). En sistemas de bases de datos como Dynamo/Riak,
los vector clocks detectan conflictos que luego se resuelven por "last write wins"
o manualmente. El tamaño del vector crece con el número de nodos — para sistemas
grandes, se usan versiones comprimidas (dotted version vectors).

---

### Ejercicio 22.3.3 — Leer: detectar causalidad con vector clocks

**Tipo: Leer/diagnosticar**

Dado este historial de eventos en un sistema de 3 nodos (A, B, C):

```
Evento  Nodo  Vector Clock después del evento  Descripción
e1      A     {A:1, B:0, C:0}                  A escribe X=1
e2      B     {A:0, B:1, C:0}                  B escribe X=2 (concurrent con e1)
e3      A     {A:2, B:0, C:0}                  A lee X (¿qué valor?)
e4      B     {A:1, B:2, C:0}                  B recibe e1, luego escribe X=3
e5      C     {A:1, B:2, C:1}                  C recibe e4, luego lee X
e6      A     {A:3, B:2, C:0}                  A recibe e4, luego escribe X=4
```

**Preguntas:**

1. ¿Cuáles pares de eventos son causalmente relacionados (uno ocurrió antes del otro)?
2. ¿Cuáles pares de eventos son concurrentes?
3. En `e3`, ¿qué valor ve A cuando lee X? ¿Depende del modelo de consistencia?
4. ¿Hay un conflicto entre `e4` y `e6`?
5. Si el sistema usa "last write wins" con Lamport timestamps como tiebreaker,
   ¿qué valor tiene X después de todos los eventos?

**Pista:** Para determinar si dos eventos son causalmente relacionados, comparar
sus vector clocks. e1={A:1,B:0,C:0} y e2={A:0,B:1,C:0}: ninguno es ≤ al otro
en todas las dimensiones → son concurrentes. e4={A:1,B:2,C:0} tiene A:1 ≥ e1's A:1
y B:2 > e1's B:0 → e1 happened-before e4.

---

### Ejercicio 22.3.4 — Hybrid Logical Clocks (HLC)

Los Lamport clocks preservan causalidad pero no el tiempo real.
Los relojes físicos preservan tiempo real pero no la causalidad.
Los HLC combinan ambos:

```go
type HLC struct {
    // l: la parte de tiempo físico (truncada al segundo)
    // c: el counter de Lamport dentro del mismo segundo físico
    l int64  // unix timestamp en milisegundos
    c int64  // counter
    mu sync.Mutex
}

func (h *HLC) Now() (l, c int64) {
    h.mu.Lock()
    defer h.mu.Unlock()
    pt := time.Now().UnixMilli()  // physical time
    if pt > h.l {
        h.l = pt
        h.c = 0
    } else {
        h.c++
    }
    return h.l, h.c
}

func (h *HLC) Update(lMsg, cMsg int64) (l, c int64) {
    h.mu.Lock()
    defer h.mu.Unlock()
    pt := time.Now().UnixMilli()
    h.l = max(h.l, lMsg, pt)
    if h.l == lMsg && h.l == pt {
        h.c = max(h.c, cMsg) + 1
    } else if h.l == lMsg {
        h.c = cMsg + 1
    } else if h.l == pt {
        h.c = h.c + 1
    } else {
        h.c = 0
    }
    return h.l, h.c
}
```

**Restricciones:** Demostrar que los HLC:
1. Preservan causalidad (como Lamport)
2. Aproximan el tiempo real (como relojes físicos)
3. Permiten hacer range queries por tiempo: "todos los eventos entre las 14:00 y las 15:00"

**Pista:** Los HLC son usados por CockroachDB y YugabyteDB para transacciones
distribuidas sin necesidad de relojes atómicos (como Google Spanner).
La propiedad clave: HLC(e) ≈ tiempo físico de e, y si A → B entonces HLC(A) < HLC(B).
Esto permite range queries por tiempo que respetan causalidad.

---

### Ejercicio 22.3.5 — Leer: el peligro del tiempo físico en sistemas distribuidos

**Tipo: Diagnosticar**

Un sistema de base de datos distribuida usa timestamps físicos para ordenar escrituras.
Dado este escenario:

```
Nodo A (reloj ligeramente adelantado, +2ms):
  14:00:00.002 — cliente escribe X=1 en A

Nodo B (reloj en hora, referencia):
  14:00:00.000 — cliente escribe X=2 en B

La escritura de B tiene timestamp 14:00:00.000, que es MENOR que el timestamp
de A (14:00:00.002), aunque B escribió DESPUÉS.

Con "last write wins por timestamp físico":
  X = 1 (la escritura de A "ganó" porque tiene timestamp mayor)

¿Esto es correcto?
```

**Preguntas:**

1. ¿Qué valor debería tener X según el usuario que escribió B?
2. ¿Por qué los timestamps físicos no son confiables para ordenar escrituras?
3. ¿Qué garantía de NTP ayudaría a reducir (pero no eliminar) este problema?
4. ¿Cómo Google Spanner resuelve esto?
5. ¿Cuándo es aceptable usar timestamps físicos para ordenar eventos?

**Pista:** Google Spanner usa TrueTime: en lugar de un timestamp puntual,
cada reloj reporta un intervalo [earliest, latest] que garantiza contiene
el tiempo real. Si dos operaciones tienen intervalos que no se solapan,
están ordenadas. Si se solapan, Spanner espera hasta que los intervalos
ya no se solapan antes de confirmar. Esto garantiza el orden a costa de latencia adicional.

---

## Sección 22.4 — Consistencia: el Espectro de Garantías

```
Consistencia linealizable (la más fuerte):
  Cada operación parece ejecutarse atómicamente en un punto entre su inicio y fin.
  Leer después de escribir siempre ve el valor escrito.
  Costosa: requiere coordinación entre todos los nodos para cada operación.

Consistencia secuencial:
  Todas las operaciones parecen ejecutarse en algún orden secuencial,
  y el orden es consistente con el orden en cada nodo individual.
  Más débil que linealizable: no requiere que el punto sea dentro del intervalo real.

Consistencia causal:
  Las operaciones causalmente relacionadas son vistas en el mismo orden por todos.
  Las operaciones concurrentes pueden verse en cualquier orden.
  Usada por: sistemas de mensajería, feeds sociales.

Eventual consistency (la más débil práctica):
  Si no hay nuevas escrituras, eventualmente todos los nodos verán el mismo valor.
  No hay garantía de cuándo.
  Usada por: DNS, algunos sistemas de caché, replicación asíncrona.
```

### Ejercicio 22.4.1 — Implementar un registro linearizable

Un registro (variable) linearizable garantiza que cada read ve el último write
confirmado, sin importar en qué nodo se ejecute:

```go
// Implementar con Raft (simplificado) o con un protocolo de quórum simple:

type RegistroLinearizable struct {
    nodos    []*Nodo
    quorum   int  // mayoria de nodos
}

func (r *RegistroLinearizable) Write(valor string) error {
    // Para escribir: necesitas confirmación del quórum
    confirmaciones := 0
    for _, nodo := range r.nodos {
        if err := nodo.Proponer(valor); err == nil {
            confirmaciones++
        }
    }
    if confirmaciones < r.quorum {
        return ErrQuorumNoAlcanzado
    }
    return nil
}

func (r *RegistroLinearizable) Read() (string, error) {
    // Para leer: también necesitas quórum (para garantizar linearizabilidad)
    // Un read de un solo nodo puede leer un valor obsoleto
    // ...
}
```

**Restricciones:** Demostrar la diferencia entre:
- Read del quórum (linearizable pero costoso)
- Read de un solo nodo (rápido pero puede ser obsoleto)

Con un test donde un nodo tiene un valor viejo y el quórum tiene el nuevo.

**Pista:** El read del quórum: preguntar a todos los nodos y retornar el valor
con el número de versión más alto (que al menos el quórum confirma).
Si solo preguntas a un nodo, puedes leer de un nodo que no recibió la última
escritura (por lentitud de red, partición temporal, etc.).

---

### Ejercicio 22.4.2 — Eventual consistency: convergencia garantizada

Los sistemas con eventual consistency deben converger — pero ¿cómo garantizarlo
cuando dos nodos tienen estados distintos?

```go
// CRDTs (Conflict-free Replicated Data Types):
// estructuras de datos que siempre convergen, sin coordinación

// G-Counter (Grow-only Counter): un counter que solo puede incrementar
type GCounter struct {
    relojes map[string]int64  // nodeID → valor en ese nodo
    nodeID  string
}

func (g *GCounter) Incrementar(delta int64) {
    g.relojes[g.nodeID] += delta
}

func (g *GCounter) Valor() int64 {
    total := int64(0)
    for _, v := range g.relojes {
        total += v
    }
    return total
}

// Merge: tomar el máximo de cada nodo (operación idempotente y conmutativa)
func (g *GCounter) Merge(otro *GCounter) {
    for nodo, valor := range otro.relojes {
        if valor > g.relojes[nodo] {
            g.relojes[nodo] = valor
        }
    }
}
```

**Restricciones:** Implementar:
1. G-Counter (solo incrementa)
2. PN-Counter (incrementa y decrementa — usando dos G-Counters)
3. LWW-Register (Last-Write-Wins Register — usando HLC como timestamp)
4. OR-Set (Observed-Remove Set — permite add y remove sin conflictos)

Para cada uno, demostrar que dos réplicas con estados distintos convergen
al mismo valor después de un Merge, independientemente del orden de los Merges.

**Pista:** El OR-Set es el CRDT más interesante: para que `remove(x)` funcione
correctamente, solo debe remover los adds de x que el removedor vio.
Si simultáneamente alguien hace `add(x)` y otro hace `remove(x)`,
el add "gana" porque el removedor no lo observó. Esto se implementa
con un set de "unique tags" por elemento.

---

### Ejercicio 22.4.3 — Leer: qué modelo de consistencia necesita tu sistema

**Tipo: Diseñar**

Para cada uno de estos sistemas, determina el modelo de consistencia
mínimo necesario y justifica la elección:

```
1. Sistema de saldo bancario:
   Operación: transferir $100 de cuenta A a cuenta B
   Pregunta: ¿puede leer un saldo desactualizado y aprobar una transferencia
             que deja el saldo negativo?

2. Sistema de likes en redes sociales:
   Operación: ver cuántos likes tiene un post
   Pregunta: ¿importa si el número que ves está desactualizado en 2 segundos?

3. Sistema de reservas de asientos:
   Operación: dos usuarios intentan reservar el asiento 15A simultáneamente
   Pregunta: ¿puede el sistema asignar el mismo asiento a dos personas?

4. Sistema de caché de configuración:
   Operación: leer la configuración del servicio (timeout, feature flags)
   Pregunta: ¿importa si un servidor tiene una versión ligeramente vieja?

5. Sistema de inventario:
   Operación: reservar el último item en stock simultáneamente desde dos regiones
   Pregunta: ¿puede oversell (vender más de lo disponible)?
```

**Pista:** La elección de consistencia es una decisión de negocio disfrazada de
decisión técnica. Para saldos bancarios: la pérdida de un cliente por saldo
negativo es mucho más costosa que la latencia adicional de linearizabilidad.
Para likes: un número desactualizado en 2 segundos no afecta al negocio.
Para asientos: depende del costo de un oversell — en vuelos, el costo es alto
(linearizable). En restaurantes con overbooking planificado, puede ser aceptable.

---

### Ejercicio 22.4.4 — Read-your-writes y monotonic reads

Dos garantías de consistencia "sesión" que son más débiles que linearizable
pero más fuertes que eventual:

```
Read-your-writes:
  Después de escribir, siempre leerás tu propia escritura.
  (Pero otro usuario puede no verla todavía.)

Monotonic reads:
  Una vez que lees un valor, nunca verás un valor anterior.
  (No retrocederás en el tiempo.)

Ejemplo sin estas garantías:
  1. Usuario actualiza su perfil: nombre → "Alice Smith"
  2. Usuario recarga la página: ve nombre → "Alice Jones" (¡el anterior!)
     (Porque la segunda request fue a un nodo diferente que no recibió la escritura)
  3. Usuario recarga de nuevo: ve nombre → "Alice Smith"
  Esto es confuso y parece un bug — aunque técnicamente es "eventual consistency"
```

**Restricciones:** Implementar un proxy de base de datos que garantiza
read-your-writes usando "sticky sessions" por usuario:
- Las escrituras del usuario U van siempre al nodo primario
- Las lecturas del usuario U van al nodo que tiene la versión más reciente de sus datos

**Pista:** La implementación simple: después de una escritura, guardar el timestamp
de la escritura en la cookie del usuario. En las lecturas siguientes, el proxy
envía al nodo primario hasta que la réplica secundaria confirma que tiene
el timestamp o más reciente. Esto se llama "read-your-writes with a causal token".

---

### Ejercicio 22.4.5 — Comparar consistencia en la práctica

**Tipo: Medir**

Configura tres nodos de una base de datos (puede ser etcd, Redis Cluster, o simulado)
y mide el costo de consistencia:

```
Experimento:
  1. Escribe 10,000 values con consistencia linearizable → mide latencia
  2. Escribe 10,000 values con consistencia eventual → mide latencia
  3. Lee 10,000 values con quórum → mide latencia y % de "stale reads"
  4. Lee 10,000 values sin quórum → mide latencia y % de "stale reads"
```

**Restricciones:** El experimento debe ejecutarse con latencia de red simulada
de 10ms entre nodos (usando `tc netem` en Linux o simulación en Go).
Documentar los resultados en una tabla y analizar el tradeoff.

**Pista:** La latencia de escritura linearizable ≈ 2 × latencia de red × número de hops.
Con 3 nodos y 10ms entre ellos: ~20-30ms por escritura. Con eventual consistency:
~10ms (solo espera la confirmación del primario). El factor 2-3x de diferencia
explica por qué muchos sistemas eligen eventual consistency para operaciones
de alta frecuencia.

---

## Sección 22.5 — Consenso: Acordar en Presencia de Fallos

El problema del consenso: N nodos deben acordar un único valor,
incluso si algunos nodos fallan.

```
Propiedades requeridas:
  Validity: el valor acordado fue propuesto por algún nodo (no uno inventado)
  Agreement: todos los nodos que deciden, deciden el mismo valor
  Termination: eventualmente todos los nodos que no fallan deciden

El resultado FLP (Fischer, Lynch, Paterson, 1985):
  En un sistema asíncrono, es imposible garantizar las tres propiedades
  con incluso UN fallo de crash.

La solución práctica: relajar "termination" usando tiempos (relojes físicos).
Raft y Paxos no son "puros" — usan timeouts para garantizar progreso.
```

### Ejercicio 22.5.1 — Implementar Raft: elección de líder

Raft divide el consenso en tres problemas independientes:
1. Elección de líder (leader election)
2. Replicación de log (log replication)
3. Seguridad (safety)

Este ejercicio implementa solo la elección de líder:

```go
type EstadoRaft int

const (
    Follower  EstadoRaft = iota
    Candidate
    Leader
)

type NodoRaft struct {
    id          string
    estado      EstadoRaft
    termActual  int64     // el "mandato" actual
    votadoPara  string    // a quién voté en este término
    votos       int       // votos recibidos como candidato

    // Canales para simular la red:
    mensajes chan MensajeRaft

    // Timeout de elección aleatorio (evita empates):
    timeoutEleccion time.Duration
}

// Cuando el follower no escucha del líder:
func (n *NodoRaft) iniciarEleccion() {
    n.termActual++
    n.estado = Candidate
    n.votadoPara = n.id
    n.votos = 1  // votar por uno mismo

    // Solicitar votos a todos los demás:
    for _, peer := range n.peers {
        go n.pedirVoto(peer, n.termActual)
    }
}
```

**Restricciones:** Implementar la elección de líder con estas garantías:
- Solo un líder por término (safety)
- Eventualmente se elige un líder (liveness, con timeouts aleatorios)
- Si el líder cae, se elige uno nuevo (en máximo 2 × timeout_eleccion)

**Pista:** El timeout de elección aleatorio (ej: entre 150ms y 300ms) es la clave
de Raft para evitar split votes. Si todos los followers tienen el mismo timeout,
todos se convierten en candidates simultáneamente y nadie gana. Con timeouts
aleatorios, uno se convierte en candidate primero y gana antes de que los demás
empiecen su elección. Verificar con un test que el sistema elige un líder
en < 1 segundo después de que el líder anterior cae.

---

### Ejercicio 22.5.2 — Raft: replicación de log

Una vez que hay un líder, debe replicar sus decisiones a los followers:

```go
type EntradaLog struct {
    Termino int64       // en qué término se creó
    Índice  int64       // posición en el log
    Comando interface{} // el valor a acordar
}

type NodoRaft struct {
    // ... (del ejercicio anterior)
    log         []EntradaLog
    commitIndex int64  // última entrada comprometida
    lastApplied int64  // última entrada aplicada a la máquina de estado
}

// El líder envía AppendEntries a los followers:
type AppendEntriesArgs struct {
    Termino      int64
    LiderID      string
    PrevLogIndice int64
    PrevLogTermino int64
    Entradas     []EntradaLog
    LiderCommit  int64
}
```

**Restricciones:** Implementar la replicación con:
- El líder espera confirmación del quórum antes de commitear
- Los followers verifican la consistencia del log (PrevLogIndice y PrevLogTermino)
- Si hay inconsistencias, el follower rechaza y el líder retrocede y reintenta

**Pista:** La garantía de Raft: una vez que una entrada está en el log de la mayoría
de nodos, nunca se pierde (incluso si el líder falla). La propiedad se mantiene
porque cualquier nuevo líder debe haber ganado votos de la mayoría — y por lo
tanto tiene el log más actualizado de la mayoría.

---

### Ejercicio 22.5.3 — Leer: Paxos vs Raft

**Tipo: Comparar**

Paxos (Lamport, 1989) fue el primer protocolo de consenso práctico.
Raft (Ongaro & Ousterhout, 2014) fue diseñado explícitamente para ser más comprensible.

```
Paxos:
  Dos fases: Prepare y Accept
  Roles: Proposer, Acceptor, Learner
  Permite múltiples proposers simultáneos (requiere más rondas de mensajes)
  El paper original es famoso por ser difícil de entender
  Muchas variantes: Multi-Paxos, Fast Paxos, Cheap Paxos

Raft:
  Tres sub-problemas bien definidos: elección, replicación, membership
  Un líder a la vez (simplifica el flujo de mensajes)
  Log continuamente ordered (más fácil de razonar)
  Diseñado para implementaciones: tiene detalles que Paxos omite
```

**Preguntas:**

1. ¿Por qué Raft limita a un único líder si Multi-Paxos permite múltiples proposers?
2. ¿Cuántos fallos puede tolerar un cluster de 5 nodos? ¿Y de 6?
3. ¿Qué pasa con la disponibilidad durante una elección de líder en Raft?
4. ¿Cuándo elegirías Paxos sobre Raft?
5. Nombra tres sistemas distribuidos que usan Raft internamente.

**Pista:** La fórmula de tolerancia a fallos: un cluster de N nodos tolera
`f = (N-1)/2` fallos (redondeado hacia abajo). 5 nodos → tolera 2 fallos.
6 nodos → todavía tolera 2 fallos (quórum = 4). Por eso los clusters tienen
siempre número impar de nodos — agregar el 6to no mejora la tolerancia.
etcd, CockroachDB, y TiKV usan Raft. Cassandra usa una variante de Paxos.

---

### Ejercicio 22.5.4 — Consenso sin Raft: el algoritmo 2PC

Two-Phase Commit (2PC) es el protocolo estándar para transacciones distribuidas:

```
Fase 1 (Prepare):
  Coordinador → todos los participantes: "¿puedes commitear?"
  Participantes: bloquearse (adquirir locks) y responder "sí" o "no"

Fase 2 (Commit o Abort):
  Si todos dijeron "sí": Coordinador → todos: "commitea"
  Si alguno dijo "no": Coordinador → todos: "aborta"

El problema: el coordinador puede fallar entre las dos fases.
  → Los participantes están bloqueados esperando la decisión
  → No pueden commitear ni abortar sin saber qué decidió el coordinador
  → Este es el "blocking problem" de 2PC
```

**Restricciones:** Implementar 2PC con un coordinador y 3 participantes.
Simular el fallo del coordinador después de la Fase 1 y mostrar que los
participantes quedan bloqueados. Luego implementar la recuperación:
cuando el coordinador reinicia, consulta los participantes para reconstruir
el estado y completar la transacción.

**Pista:** La recuperación de 2PC: el coordinador guarda su decisión en un WAL
(write-ahead log) antes de enviarla. Si reinicia, lee el WAL y reenvía la
decisión. Si el WAL no tiene la decisión (el coordinador crasheó antes de decidir),
el coordinador puede abortar (conservador). Para sistemas de producción,
el coordinador debe ser replicado (con Raft) para evitar el single point of failure.

---

### Ejercicio 22.5.5 — Saga pattern: transacciones sin 2PC

Las sagas reemplazan las transacciones ACID distribuidas con una secuencia
de transacciones locales compensables:

```
Saga: crear un pedido de viaje
  T1: reservar vuelo        → C1: cancelar vuelo
  T2: reservar hotel        → C2: cancelar hotel
  T3: reservar auto         → C3: cancelar auto
  T4: cobrar tarjeta        → C4: reembolsar tarjeta

Si T3 falla:
  Ejecutar compensaciones en orden inverso: C2, C1
  (T4 nunca se ejecutó, T3 falló, T2 y T1 se compensan)

Garantía: el sistema eventual llega a un estado consistente.
NO garantiza aislamiento (otro usuario puede ver el estado intermedio).
```

**Restricciones:** Implementar el saga orchestrator:
- Ejecuta las transacciones en orden
- Si alguna falla, ejecuta las compensaciones de las anteriores
- Es idempotente: si el orchestrator cae y reinicia, puede retomar desde donde quedó
- Expone el estado de la saga como una máquina de estados visible

**Pista:** La idempotencia del orchestrator requiere guardar qué paso ejecutó último
(y su resultado) en almacenamiento durable antes de ejecutar el siguiente paso.
Si el orchestrator cae y reinicia, lee el estado guardado y retoma desde el último
paso completado. Esto requiere que cada transacción Ti sea idempotente también
(o use idempotency keys).

---

## Sección 22.6 — Replicación: Mantener Copias Sincronizadas

### Ejercicio 22.6.1 — Replicación síncrona vs asíncrona

```
Replicación síncrona:
  El leader espera confirmación de todas las réplicas antes de confirmar al cliente.
  Garantía: si el leader cae, ningún dato se pierde.
  Costo: la latencia de escritura = max(latencia de todas las réplicas)

Replicación asíncrona:
  El leader confirma al cliente sin esperar las réplicas.
  Garantía: baja latencia de escritura.
  Costo: si el leader cae antes de replicar, se pierde la escritura reciente.

Replicación semi-síncrona (MySQL, PostgreSQL):
  El leader espera confirmación de AL MENOS UNA réplica.
  Tradeoff: tolera la pérdida de una réplica pero no del leader.
```

**Restricciones:** Implementar los tres modos y medir:
- Latencia de escritura con 3 nodos y 5ms de latencia de red
- RPO (Recovery Point Objective): ¿cuántos datos se pierden si el leader cae?
- RTO (Recovery Time Objective): ¿cuánto tarda en recuperarse?

**Pista:** Con replicación síncrona y 3 nodos a 5ms de latencia:
escritura ≈ 10ms (2 RTT: client→leader→replica, replica→leader→client).
Con asíncrona: escritura ≈ 5ms (1 RTT). El RPO de asíncrona ≈ el intervalo
entre replicaciones (puede ser milisegundos o segundos según la carga).

---

### Ejercicio 22.6.2 — Read replicas: escalar lecturas

Las read replicas permiten distribuir la carga de lecturas:

```
Configuración:
  1 primary (acepta escrituras)
  N replicas (solo lecturas, asíncronas)

Problema: replication lag
  Una escritura en el primary puede tardar 100ms en llegar a las replicas.
  Si un cliente escribe y luego lee de una replica, puede ver el estado anterior.
```

**Restricciones:** Implementar un proxy que:
- Envía escrituras al primary
- Distribuye lecturas entre replicas (round-robin)
- Garantiza "read-your-writes" para el mismo cliente (sticky routing post-escritura)
- Monitorea el replication lag y alerta si supera 1 segundo

**Pista:** Para "read-your-writes" con replicas: después de una escritura, el proxy
guarda el LSN (Log Sequence Number) de esa escritura. Las siguientes lecturas del mismo
cliente van a una replica que confirme tener ese LSN o más reciente. Esto requiere
que las replicas exponen su LSN actual (PostgreSQL tiene `pg_last_wal_replay_lsn()`).

---

### Ejercicio 22.6.3 — Conflict resolution: qué pasa cuando dos nodos divergen

En sistemas con replicación multi-master o con particiones de red,
dos nodos pueden aceptar escrituras conflictivas:

```
Nodo A (región US): usuario actualiza email → alice@new.com
Nodo B (región EU): usuario actualiza email → alice@work.com
(Ocurre simultáneamente, durante una partición de red)

Cuando la red se recupera, ¿cuál email es el correcto?
```

**Estrategias de resolución:**

```go
// 1. Last Write Wins (LWW): el timestamp más reciente gana
func lww(a, b Versión) Versión {
    if a.Timestamp.After(b.Timestamp) { return a }
    return b
}

// 2. Merge: combinar cuando es posible
// Para un conjunto de tags, hacer la unión
func mergeTags(a, b []string) []string {
    union := make(map[string]bool)
    for _, t := range append(a, b...) { union[t] = true }
    // ...
}

// 3. Resolver manualmente: marcar el conflicto para que el usuario decida
func marcarConflicto(a, b Versión) ConflictoParaResolver {
    return ConflictoParaResolver{Opcion1: a, Opcion2: b}
}
```

**Restricciones:** Para cada tipo de dato, implementar la estrategia correcta:
- Email del usuario: LWW (solo puede haber uno)
- Lista de favoritos: merge (unión)
- Documento de texto: diff3 merge (como Git)
- Saldo de cuenta: nunca multi-master (demasiado arriesgado)

**Pista:** LWW parece simple pero tiene el problema de los relojes distribuidos
(Ejercicio 22.3.5): dos escrituras simultáneas pueden tener timestamps idénticos
o en el orden incorrecto. La alternativa: LWW con vector clocks para detectar
escrituras genuinamente concurrentes y tratarlas como conflicto en lugar de
silenciosamente descartar una.

---

### Ejercicio 22.6.4 — Change data capture (CDC)

CDC captura los cambios de la BD y los propaga a otros sistemas:

```
Base de datos → Binlog/WAL → CDC connector → Kafka topic → Consumidores

Ventaja sobre polling:
  Polling: SELECT * FROM tabla WHERE updated_at > ? (cada N segundos)
  - Puede perder cambios si updated_at no se actualiza
  - Carga en la BD proporcional a la frecuencia del poll
  
  CDC: leer el WAL de la BD (como una replica)
  - Captura TODOS los cambios, incluyendo deletes
  - Overhead mínimo en la BD (el WAL ya existe)
  - Latencia baja (milisegundos)
```

**Restricciones:** Implementar un CDC simplificado:
- Leer el WAL de PostgreSQL usando `pg_logical_slot_get_changes()`
- Convertir los cambios a eventos (INSERT/UPDATE/DELETE con los datos)
- Publicar los eventos a un canal de Go (o simulación de Kafka)
- El CDC debe ser "at-least-once": si falla, retomar desde el último punto procesado

**Pista:** La posición en el WAL se llama LSN (Log Sequence Number).
El CDC guarda el último LSN procesado. Al reiniciar, solicita cambios desde ese LSN.
Esto garantiza que no pierde cambios (at-least-once) pero puede duplicar el
último lote si el CDC falló después de leer pero antes de guardar el LSN.
Los consumidores deben ser idempotentes para manejar duplicados.

---

### Ejercicio 22.6.5 — Leer: diagnosticar replication lag

**Tipo: Diagnosticar**

Este dashboard muestra las métricas de un cluster PostgreSQL con 1 primary y 2 replicas:

```
Primary:
  Escrituras/s: 850
  WAL generation rate: 12 MB/s
  Connections activas: 180/200

Replica 1 (misma datacenter):
  Replication lag: 45ms
  WAL apply rate: 12 MB/s  ← igual que la generación → al día
  Connections activas: 95/100

Replica 2 (otra región, 80ms de latencia de red):
  Replication lag: 8,400ms  ← !!
  WAL apply rate: 8 MB/s    ← más lento que la generación
  Connections activas: 98/100

Alerta disparada: Replica 2 replication lag > 5s
```

**Preguntas:**

1. ¿Por qué Replica 1 está al día pero Replica 2 no?
2. Si el primary crashea ahora, ¿cuántos datos se perderían al hacer failover a Replica 2?
3. ¿La saturación de conexiones (98/100) en Replica 2 puede ser la causa del lag?
4. ¿Qué pasaría con las lecturas "read-your-writes" que van a Replica 2?
5. Propón dos acciones inmediatas y una a largo plazo.

**Pista:** El WAL apply rate de Replica 2 (8 MB/s) es menor que la generación del primary (12 MB/s).
El backlog crece a 4 MB/s. Si el primary genera 12 MB/s durante una hora, el backlog
acumulado es ~14.4 GB. La latencia de red de 80ms limita cuántos WAL records puede
procesar por segundo. Las conexiones saturadas pueden estar esperando I/O — verificar
con `pg_stat_replication` si el apply worker está esperando disco.

---

## Sección 22.7 — El Teorema CAP: Eligiendo Dos de Tres

```
CAP Theorem (Brewer, 2000):
  En presencia de una partición de red (P), un sistema distribuido debe elegir entre:
  
  Consistency (C): todos los nodos ven el mismo dato al mismo tiempo
  Availability (A): cada request recibe una respuesta (puede ser desactualizada)
  
  No puede garantizar ambas durante una partición.

La formulación más precisa (Gilbert & Lynch, 2002):
  Es imposible que un sistema distribuido garantice simultáneamente:
  - Linearizability (C)
  - Availability total (A)
  - Partition tolerance (P)

Interpretación práctica:
  Las particiones de red OCURREN (no son opcionales).
  Por tanto, el sistema debe elegir entre C y A cuando hay una partición.
  
  CP: durante una partición, el sistema rechaza requests para mantener consistencia
  AP: durante una partición, el sistema responde con datos posiblemente desactualizados
```

### Ejercicio 22.7.1 — Observar CAP en un sistema real

**Tipo: Experimento**

Configura un cluster de 3 nodos de etcd (CP) o Cassandra (AP) y simula
una partición de red:

```bash
# Simular partición de red con iptables:
sudo iptables -A INPUT -s <IP_NODO_2> -j DROP
sudo iptables -A INPUT -s <IP_NODO_3> -j DROP

# Con etcd (CP):
# Durante la partición, el nodo aislado rechaza writes (mantiene consistencia)
# El quórum (nodos 2 y 3) sigue funcionando

# Con Cassandra (AP) y quórum = 1:
# Durante la partición, todos los nodos aceptan writes (disponibilidad)
# Los datos divergen — se reconcilian cuando la red se recupera
```

**Restricciones:** Documentar el comportamiento de ambos sistemas durante y después
de la partición, midiendo:
- Disponibilidad durante la partición (% de requests exitosos)
- Consistencia después de la recuperación (¿hay datos perdidos o conflictos?)
- Tiempo de recuperación (cuánto tarda en volver al estado normal)

**Pista:** Para etcd: el nodo aislado devuelve `context deadline exceeded` o
`etcdserver: request timed out` para escrituras durante la partición.
Para Cassandra con `consistency_level = ONE`: todas las escrituras tienen éxito
durante la partición, pero al recuperar, los datos se reconcilian con LWW.

---

### Ejercicio 22.7.2 — PACELC: más allá del CAP

El teorema PACELC (Abadi, 2012) extiende CAP:

```
CAP se aplica solo durante una partición (P).
Pero la mayoría del tiempo NO hay partición.
¿Cuál es el tradeoff entonces?

PACELC:
  Si hay Partición (P): elegir entre Availability (A) y Consistency (C)
  Else (sin partición): elegir entre Latency (L) y Consistency (C)

Ejemplos:
  DynamoDB: PA/EL (disponible durante partición, latencia baja sin partición)
  HBase: PC/EC (consistente durante partición, consistente sin partición)
  MySQL cluster (sync replication): PC/EC
  Cassandra (tunable): PA/EL por defecto, puede configurarse hacia PC/EC
```

**Restricciones:** Para cada sistema, clasificarlo según PACELC y verificar
experimentalmente la clasificación midiendo:
1. Comportamiento durante una partición simulada
2. Tradeoff latencia/consistencia en condiciones normales

**Pista:** La mayoría del tiempo tu sistema NO está bajo partición de red.
El tradeoff EL vs EC (Else: Latency vs Consistency) es más relevante día a día
que el tradeoff durante particiones. Un sistema que elige "EC" (consistencia incluso
sin partición) paga en latencia: cada operación requiere coordinación del quórum.

---

### Ejercicio 22.7.3 — Diseñar el modelo de consistencia para un sistema específico

**Tipo: Diseñar**

Diseña el modelo de consistencia para una plataforma de trading de criptomonedas:

```
Componentes:
  1. Order book: qué órdenes de compra/venta están abiertas
  2. Saldo de usuarios: cuánto dinero/crypto tiene cada usuario
  3. Historial de trades: registro inmutable de transacciones
  4. Precio de mercado: el precio actual de cada par (BTC/USD, etc.)
  5. Notificaciones: avisar al usuario cuando su orden se ejecuta
```

**Para cada componente, decide:**
- Modelo de consistencia (linearizable, causal, eventual)
- Justificación de negocio
- Consecuencia de elegir un modelo más débil
- Tecnología que implementa ese modelo

**Pista:** Order book y saldo de usuario requieren linearizabilidad — un trade
ejecutado dos veces o un saldo que permite comprar más de lo que se tiene
son errores de negocio graves. El historial es un append-only log que puede
ser eventual. El precio de mercado puede ser eventual — un precio desactualizado
en 100ms es aceptable si el order book es consistente. Las notificaciones son
best-effort (eventual).

---

### Ejercicio 22.7.4 — Implementar "tunable consistency"

Cassandra permite elegir el nivel de consistencia por operación:

```go
type NivelConsistencia int

const (
    One       NivelConsistencia = 1  // confirmar 1 replica
    Quorum    NivelConsistencia = 2  // confirmar mayoría
    All       NivelConsistencia = 3  // confirmar todas las replicas
    LocalOne  NivelConsistencia = 4  // confirmar 1 replica en la región local
)

type ClienteTunable struct {
    nodos   []*Nodo
    quorum  int
}

func (c *ClienteTunable) Write(key, valor string, nivel NivelConsistencia) error {
    confirmacionesNecesarias := c.confirmsNecesarias(nivel)
    confirmadas := 0

    // Enviar a todos, esperar las necesarias:
    resultados := make(chan error, len(c.nodos))
    for _, nodo := range c.nodos {
        go func(n *Nodo) {
            resultados <- n.Set(key, valor)
        }(nodo)
    }

    for range c.nodos {
        if err := <-resultados; err == nil {
            confirmadas++
            if confirmadas >= confirmacionesNecesarias {
                return nil  // no esperar a las demás
            }
        }
    }
    return ErrConsistenciaInsuficiente
}
```

**Restricciones:** Implementar con las cuatro opciones y demostrar que:
- `One` + `All` para leer-escribir garantiza consistencia (W + R > N)
- `Quorum` + `Quorum` garantiza consistencia (W + R > N)
- `One` + `One` no garantiza consistencia (puede leer de una replica sin el último write)

**Pista:** La regla W + R > N garantiza que el quórum de lectura y el de escritura
se solapan — al menos un nodo tiene el valor más reciente. Con 3 nodos:
`All(3) + One(1) = 4 > 3` ✓, `Quorum(2) + Quorum(2) = 4 > 3` ✓,
`One(1) + One(1) = 2 ≤ 3` ✗.

---

### Ejercicio 22.7.5 — El sistema que cambió de modelo de consistencia

**Tipo: Leer/analizar**

Amazon DynamoDB fue lanzado en 2007 con eventually consistent como único modelo.
En 2018 añadió "strongly consistent reads" como opción.

```
Razones para el eventual consistency original:
  - Baja latencia global (leer de la replica local)
  - Alta disponibilidad (no bloquear durante particiones)
  - Escalabilidad (sin coordinación de quórum)

Razones para añadir strongly consistent:
  - Muchos casos de uso requerían read-after-write
  - Los workarounds (usar la primary key para forzar consistency) eran complicados
  - Microservicios que dependen unos de otros necesitan ver el estado actualizado

Costo:
  - Strongly consistent reads cuestan 2× el precio de eventually consistent
  - Mayor latencia (requiere leer del nodo líder, no de la réplica local)
```

**Preguntas:**

1. ¿Por qué Amazon no hizo strongly consistent el default?
2. ¿Qué casos de uso justifican pagar 2× el precio?
3. ¿Qué casos de uso pueden usar eventually consistent sin problemas?
4. ¿Cómo implementarías el "strongly consistent read" en tu implementación de tunable consistency?
5. ¿Qué lección general sobre diseño de sistemas ilustra esta evolución?

**Pista:** La lección general: el modelo de consistencia es una decisión de API
que es muy difícil de cambiar retroactivamente. Amazon tardó 11 años en añadir
strongly consistent porque requirió cambios en la arquitectura interna.
El consejo: diseñar la API para soportar ambos modos desde el inicio, aunque
solo implementes uno inicialmente. Cambiar la API es más difícil que cambiar la implementación.

---

## Resumen del capítulo

**Los cinco conceptos que hacen los sistemas distribuidos fundamentalmente distintos:**

```
1. Sin memoria compartida
   La comunicación es por mensajes — pueden perderse, reordenarse, duplicarse.
   Cada operación en la red puede fallar silenciosamente.

2. Fallos parciales
   Un nodo puede caer sin que los demás lo sepan inmediatamente.
   La detección de fallos es probabilística, no determinista.

3. Sin tiempo global
   Los relojes de distintas máquinas no están sincronizados perfectamente.
   El orden de los eventos se debe inferir de la causalidad, no del tiempo físico.

4. El teorema CAP
   Durante una partición, debes elegir entre consistencia y disponibilidad.
   No hay opción "correcta" — depende de lo que tu sistema no puede permitirse perder.

5. El consenso es caro
   Acordar un valor entre N nodos requiere múltiples rondas de mensajes.
   Los sistemas que lo hacen bien (etcd, ZooKeeper) son los más complejos del ecosistema.
```

**Lo que los capítulos 1–21 dan por sentado que este capítulo no puede:**

```
Cap.01–21:        Este capítulo:
─────────────     ─────────────────────────────
mutex.Lock()   → protocolo de consenso (varios RTT de red)
canal <- valor → mensaje TCP (puede perderse o llegar tarde)
time.Now()     → timestamp físico no confiable para ordenar eventos
el proceso     → quórum de nodos (algunos pueden estar caídos)
goroutine      → nodo en una región geográfica diferente
```

El Cap.23 construye sobre esto — los patrones de arquitectura que hacen
que los sistemas distribuidos sean operables en producción.
