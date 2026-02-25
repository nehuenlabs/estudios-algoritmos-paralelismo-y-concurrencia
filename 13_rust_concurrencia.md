# Guía de Ejercicios — Rust: Concurrencia Garantizada por el Compilador

> Implementar todos los ejercicios en **Rust**.
> Comparaciones con Go, Java, y C++ donde ilustren el contraste.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el error que Rust hace imposible

```rust
// Esto compila en Go, Java, C++... pero no en Rust:

fn main() {
    let datos = vec![1, 2, 3, 4, 5];

    let hilo = std::thread::spawn(|| {
        println!("{:?}", datos);  // ← ERROR DE COMPILACIÓN
    });

    println!("{:?}", datos);  // ← si esto compilara, dos threads usarían `datos`
    hilo.join().unwrap();
}
```

```
error[E0382]: borrow of moved value: `datos`
  --> src/main.rs:9:22
   |
 3 |     let datos = vec![1, 2, 3, 4, 5];
   |         ----- move occurs because `datos` has type `Vec<i32>`
 5 |     let hilo = std::thread::spawn(|| {
   |                               -- value moved into closure here
 6 |         println!("{:?}", datos);
   |                          ----- variable moved due to use in closure
...
 9 |     println!("{:?}", datos);
   |                      ^^^^^ value borrowed here after move
```

El compilador detecta una potencial race condition en tiempo de compilación.
La solución en Rust no es "tener más cuidado" — es cambiar el código para que
exprese exactamente el ownership:

```rust
fn main() {
    let datos = vec![1, 2, 3, 4, 5];

    // Opción 1: mover la propiedad al thread
    let hilo = std::thread::spawn(move || {
        println!("{:?}", datos);  // datos fue movido → el thread es el dueño
    });
    // println!("{:?}", datos);  ← ya no existe aquí — fue movido
    hilo.join().unwrap();
}
```

```rust
fn main() {
    let datos = vec![1, 2, 3, 4, 5];

    // Opción 2: compartir con Arc (Atomic Reference Count)
    let datos = Arc::new(datos);
    let datos_clone = Arc::clone(&datos);

    let hilo = std::thread::spawn(move || {
        println!("{:?}", datos_clone);  // clone tiene su propio Arc
    });

    println!("{:?}", datos);  // el Arc original sigue siendo válido
    hilo.join().unwrap();
}
```

La diferencia con Go: en Go, si omites el mutex, el programa compila y los bugs
aparecen en producción. En Rust, el programa no compila hasta que el código
exprese explícitamente el ownership y los permisos de acceso.

---

## Los dos traits que gobiernan la concurrencia en Rust

```rust
pub unsafe trait Send {
    // Implementado por tipos que pueden transferirse entre threads
    // Vec<T> es Send si T es Send
    // Rc<T> NO es Send (contador de referencias no-atómico)
}

pub unsafe trait Sync {
    // Implementado por tipos que pueden ser accedidos desde múltiples threads
    // &T es Sync si T es Sync
    // Cell<T> NO es Sync (mutabilidad interior no-atómica)
}
```

**La tabla que importa:**

```
Tipo           Send     Sync     Notas
──────────────────────────────────────────────────────
i32, f64       ✓        ✓        tipos primitivos
Vec<T>         ✓ (si T) ✗        puede mutar → solo un thread puede tenerlo
Arc<T>         ✓ (si T) ✓ (si T) referencia contada atómica → compartir entre threads
Rc<T>          ✗        ✗        referencia contada no-atómica → solo un thread
Mutex<T>       ✓ (si T) ✓ (si T) permite mutación compartida de forma segura
RwLock<T>      ✓ (si T) ✓ (si T) lecturas múltiples, escritura exclusiva
Cell<T>        ✓ (si T) ✗        mutabilidad interior sin lock → no compartir
RefCell<T>     ✗        ✗        borrow checking en runtime → solo un thread
```

---

## Tabla de contenidos

- [Sección 13.1 — Ownership y borrowing bajo concurrencia](#sección-131--ownership-y-borrowing-bajo-concurrencia)
- [Sección 13.2 — Arc y Mutex: compartir con seguridad](#sección-132--arc-y-mutex-compartir-con-seguridad)
- [Sección 13.3 — Canales en Rust: mpsc y crossbeam](#sección-133--canales-en-rust-mpsc-y-crossbeam)
- [Sección 13.4 — Rayon: paralelismo de datos idiomático](#sección-134--rayon-paralelismo-de-datos-idiomático)
- [Sección 13.5 — Async/await: concurrencia sin threads del OS](#sección-135--asyncawait-concurrencia-sin-threads-del-os)
- [Sección 13.6 — Atomics y lock-free en Rust](#sección-136--atomics-y-lock-free-en-rust)
- [Sección 13.7 — Rust vs Go: cuándo elegir cada uno](#sección-137--rust-vs-go-cuándo-elegir-cada-uno)

---

## Sección 13.1 — Ownership y Borrowing bajo Concurrencia

El sistema de ownership de Rust garantiza que en todo momento:
- Hay exactamente un propietario de cada dato (ownership)
- Múltiples lecturas concurrentes son seguras (`&T` — shared reference)
- Una escritura es exclusiva (`&mut T` — exclusive reference)

Estas reglas se verifican en tiempo de compilación. Las races son
literalmente imposibles de expresar en código Rust seguro.

---

### Ejercicio 13.1.1 — Errores de ownership como documentación

**Enunciado:** Estudia y resuelve estos seis errores de compilación, entendiendo
qué garantía de seguridad viola cada uno:

```rust
// Error 1: usar después de move
let v = vec![1, 2, 3];
let h = thread::spawn(move || println!("{v:?}"));
println!("{v:?}");  // ← ERROR: v fue movido

// Error 2: enviar referencia no-Send
let rc = Rc::new(42);
thread::spawn(move || println!("{rc}"));  // ← ERROR: Rc no es Send

// Error 3: Mutex sin Arc no puede compartirse
let m = Mutex::new(0);
let h = thread::spawn(|| *m.lock().unwrap() += 1);  // ← ERROR: borrow inválido
*m.lock().unwrap() += 1;
h.join().unwrap();

// Error 4: RefCell entre threads
let cell = RefCell::new(0);
let h = thread::spawn(move || *cell.borrow_mut() += 1);  // ← ERROR: RefCell no es Send

// Error 5: data race con multiple &mut
let mut v = vec![1, 2, 3];
let a = &mut v[0];  // mutable borrow
let b = &mut v[1];  // ← ERROR: segundo mutable borrow del mismo Vec
*a = 10;
*b = 20;

// Error 6: lifetime más corta que el thread
let datos = vec![1, 2, 3];
let h = thread::spawn(|| println!("{datos:?}"));  // ← ERROR: datos puede ser dropped
h.join().unwrap();  // antes de aquí
```

**Restricciones:** Para cada error, escribir: (1) el error exacto del compilador,
(2) qué race condition / bug prevendría si compilara, (3) la corrección mínima.

**Pista:** El Error 6 es el más sutil: el thread podría vivir más que `datos` si el Join
está en otro branch o si el programa hace `std::process::exit`. El compilador no puede
saber que el Join siempre ocurre antes del drop de `datos`.
La solución: `thread::scope` (Rust 1.63) que garantiza que el thread no vive más que su scope.

**Implementar en:** Rust

---

### Ejercicio 13.1.2 — `thread::scope` para referencias temporales

**Enunciado:** `thread::scope` permite lanzar threads que acceden a datos del stack
porque el compilador garantiza que todos los threads terminan antes de que el scope salga:

```rust
fn procesar_en_paralelo(datos: &[i64]) -> Vec<i64> {
    let mut resultados = vec![0i64; datos.len()];

    thread::scope(|s| {
        // Podemos prestar `datos` y `resultados` a los threads
        // porque el scope garantiza que todos terminan aquí
        for (chunk_datos, chunk_res) in
            datos.chunks(datos.len() / 4)
            .zip(resultados.chunks_mut(resultados.len() / 4))
        {
            s.spawn(|| {
                for (d, r) in chunk_datos.iter().zip(chunk_res.iter_mut()) {
                    *r = d * d;  // sin Arc, sin Clone, solo borrowing
                }
            });
        }
        // Todos los threads terminan aquí automáticamente
    });

    resultados
}
```

**Restricciones:** Implementar `procesar_en_paralelo` para estas cuatro operaciones:
suma de cuadrados, normalización (dividir por la media), transformación FFT simplificada,
y filtro de ventana deslizante. Cada una con 8 threads usando `thread::scope`.

**Pista:** `thread::scope` existe en Rust estable desde 1.63. Antes, se necesitaba
`crossbeam::scope`. La diferencia con `thread::spawn`: el scope tiene un lifetime
vinculado al closure, por lo que los threads dentro pueden prestar referencias
al stack del scope. `thread::spawn` requiere `'static` — los datos deben vivir "para siempre".

**Implementar en:** Rust

---

### Ejercicio 13.1.3 — Rayon: paralelismo sin gestionar threads

**Enunciado:** Rayon extiende el ownership de Rust al paralelismo automático:
si el tipo es `Send`, `par_iter()` puede dividir el trabajo entre threads automáticamente.

```rust
use rayon::prelude::*;

fn suma_paralela(v: &[f64]) -> f64 {
    v.par_iter().sum()  // paralelismo automático — correcto por los tipos
}

fn transformar_paralelo(v: &mut [f64]) {
    v.par_iter_mut()
     .for_each(|x| *x = x.sqrt());  // mutación paralela segura
}
```

**Restricciones:** Implementa con Rayon las operaciones del Ejercicio 13.1.2
y compara con la versión manual de `thread::scope`.
La versión de Rayon debe ser igual de correcta y dentro de 1.5x de la velocidad.

**Pista:** Rayon usa work-stealing internamente (como el ForkJoinPool de Java).
La corrección en tiempo de compilación viene de `Send + Sync`: Rayon requiere
que los tipos de los datos sean `Send` para distribuirlos entre threads,
y que el closure sea `Send` para ejecutarlo en threads distintos.
Si alguno no es `Send`, el código no compila — sin posibilidad de race condition.

**Implementar en:** Rust

---

### Ejercicio 13.1.4 — Interior mutability: Cell, RefCell, Mutex

**Enunciado:** Rust tiene tres formas de mutabilidad interior, con garantías distintas:

```rust
// Cell<T>: para tipos Copy, sin locks, sin runtime overhead
// Solo funciona con un thread (no es Sync)
let cell = Cell::new(0i32);
cell.set(cell.get() + 1);  // no hay borrow — solo copiar valores

// RefCell<T>: borrow checking en runtime
// Panics si se viola el borrow checker en runtime
// Solo funciona con un thread (no es Sync)
let refcell = RefCell::new(vec![1, 2, 3]);
refcell.borrow_mut().push(4);  // puede panic si ya hay un borrow activo

// Mutex<T>: para múltiples threads, con overhead de lock
// Es Sync — puede compartirse entre threads
let mutex = Arc::new(Mutex::new(vec![1, 2, 3]));
mutex.lock().unwrap().push(4);  // bloquea hasta adquirir el lock
```

**Restricciones:** Implementa un sistema de estadísticas concurrente donde:
- Los contadores por thread usan `Cell<u64>` (sin overhead)
- Los contadores globales usan `Mutex<HashMap<String, u64>>`
- El estado temporal de procesamiento por item usa `RefCell`
Verifica que el compilador rechaza el uso incorrecto de cada uno.

**Pista:** El truco de los tres: `Cell` y `RefCell` están diseñados para ser usados
dentro de un solo thread (tienen semántica de un thread). `Mutex` convierte
cualquier tipo en uno que puede compartirse entre threads. El patrón común:
`Arc<Mutex<T>>` para compartir estado mutable entre múltiples threads.

**Implementar en:** Rust

---

### Ejercicio 13.1.5 — Unsafe: romper las reglas con responsabilidad

**Enunciado:** `unsafe` permite bypassear las garantías del compilador.
Es necesario para implementar las primitivas que las garantías usan internamente:

```rust
// Implementar split_at_mut sin la versión stdlib
// (la stdlib la implementa con unsafe internamente)
fn split_en_mitad<T>(slice: &mut [T]) -> (&mut [T], &mut [T]) {
    let mid = slice.len() / 2;
    let ptr = slice.as_mut_ptr();
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), slice.len() - mid),
        )
        // Correcto: los dos slices no se solapan
        // El compilador no puede verificarlo → unsafe
    }
}
```

**Restricciones:** Implementa un `Vec` concurrente simplificado que permite
múltiples lectores y un escritor, usando `unsafe` mínimo.
Documenta con comentarios `// SAFETY: ...` cada uso de `unsafe` explicando por qué es correcto.

**Pista:** La regla de oro del `unsafe` en Rust: "unsafe no desactiva el borrow checker,
lo cambia de compilador a programador". El código unsafe debe ser tan pequeño como sea posible
(encapsulado en funciones pequeñas) y estar rodeado de código safe que lo usa.
Los comentarios `// SAFETY:` no son opcionales en código de producción — son la documentación
de por qué el unsafe es correcto.

**Implementar en:** Rust

---

## Sección 13.2 — Arc y Mutex: Compartir con Seguridad

El patrón `Arc<Mutex<T>>` es el equivalente en Rust del `sync.Mutex` de Go
envolviendo un dato. La diferencia: en Go, el mutex y el dato que protege
son convencionalmente cercanos pero no hay enforcement. En Rust, el dato
está dentro del `Mutex` — no se puede acceder al dato sin adquirir el lock.

```rust
// Go: el dato y el mutex son independientes — la protección es convención
type Cuenta struct {
    mu    sync.Mutex
    saldo int64
}

// Rust: el dato está dentro del Mutex — la protección es enforced
let cuenta = Arc::new(Mutex::new(Cuenta { saldo: 1000 }));

// Para acceder al saldo, DEBES adquirir el lock:
let guard = cuenta.lock().unwrap();
let saldo = guard.saldo;  // guard es el único acceso al dato
// guard se libera automáticamente al salir del scope
```

---

### Ejercicio 13.2.1 — El Mutex "envuelve" el dato — no lo acompaña

**Enunciado:** Reimplementa el sistema bancario del Cap.06 en Rust.
Demuestra que es imposible acceder al saldo sin adquirir el lock:

```rust
struct Banco {
    cuentas: Arc<Mutex<HashMap<String, i64>>>,
}

impl Banco {
    fn transferir(&self, de: &str, a: &str, monto: i64) -> Result<(), String> {
        let mut cuentas = self.cuentas.lock().map_err(|e| e.to_string())?;
        // cuentas es un MutexGuard — acceso exclusivo garantizado
        let saldo_de = cuentas.get(de).copied().ok_or("cuenta no encontrada")?;
        if saldo_de < monto {
            return Err("saldo insuficiente".to_string());
        }
        *cuentas.get_mut(de).unwrap() -= monto;
        *cuentas.get_mut(a).unwrap() += monto;
        Ok(())
    }
}
```

**Restricciones:** El compilador debe rechazar cualquier intento de acceder
al saldo directamente (sin `lock()`). Demuestra esto con un test de compilación.
Implementa 10,000 transferencias concurrentes y verifica que la suma de saldos
se mantiene constante.

**Pista:** El `MutexGuard<T>` implementa `Deref<Target=T>` — por eso puedes usar
`*guard` o `guard.campo` para acceder al dato. Cuando el `MutexGuard` se droppea
(fin de scope), el lock se libera automáticamente. No hay `Unlock()` explícito
— no se puede "olvidar" liberar el lock.

**Implementar en:** Rust

---

### Ejercicio 13.2.2 — Deadlock: Rust no lo previene en tiempo de compilación

**Enunciado:** Rust previene races, pero no deadlocks. Demuestra el deadlock
del transfer bancario cuando los locks se adquieren en orden diferente:

```rust
// Thread 1: lock cuenta A, luego cuenta B
// Thread 2: lock cuenta B, luego cuenta A
// → Deadlock potencial

fn transferir_deadlock(
    cuenta_a: Arc<Mutex<i64>>,
    cuenta_b: Arc<Mutex<i64>>,
    monto: i64,
) {
    let a = cuenta_a.lock().unwrap();
    // aquí: thread 2 puede adquirir b y quedar esperando a
    let b = cuenta_b.lock().unwrap();  // ← posible deadlock
    *a - monto; // Nota: Rust tiene MutexGuard, a es inmutable aquí — ver pista
    // ...
}
```

**Restricciones:** Implementa la detección de deadlock con `try_lock()` y backoff.
Implementa la solución idiomática: ordenar los locks por dirección de memoria.

```rust
fn transferir_seguro(a: Arc<Mutex<i64>>, b: Arc<Mutex<i64>>, monto: i64) {
    // Siempre adquirir el lock con menor dirección primero
    let (primero, segundo) = if Arc::as_ptr(&a) < Arc::as_ptr(&b) {
        (&a, &b)
    } else {
        (&b, &a)
    };
    // Adquirir en orden consistente...
}
```

**Pista:** Rust previene races en tiempo de compilación, pero los deadlocks son
un problema de lógica de programa, no de accesos a memoria. El compilador no puede
saber si dos locks siempre se adquieren en el mismo orden.
`parking_lot::Mutex` tiene `try_lock_for(duration)` que evita deadlocks infinitos.
La biblioteca `deadlock` de Rust puede detectar deadlocks en runtime.

**Implementar en:** Rust

---

### Ejercicio 13.2.3 — RwLock: lecturas múltiples, escritura exclusiva

**Enunciado:** Implementa una caché de configuración con `RwLock`:

```rust
struct ConfigCache {
    inner: Arc<RwLock<HashMap<String, String>>>,
}

impl ConfigCache {
    fn get(&self, key: &str) -> Option<String> {
        // Múltiples threads pueden leer simultáneamente
        self.inner.read().unwrap().get(key).cloned()
    }

    fn set(&self, key: String, value: String) {
        // Solo un thread puede escribir, ninguno puede leer
        self.inner.write().unwrap().insert(key, value);
    }
}
```

**Restricciones:** Mide el throughput con 90% reads y 10% writes para 1, 4, 8 threads.
El `RwLock` debe superar al `Mutex` para 90% reads.
Compara con `parking_lot::RwLock` (implementación más rápida).

**Pista:** `std::sync::RwLock` de Rust puede tener "writer starvation" — si constantemente
llegan nuevos lectores, el escritor nunca obtiene el lock.
`parking_lot::RwLock` tiene política de fairness configurable.
Para la mayoría de uso, `std::sync::RwLock` es suficiente; para alta contención
con muchos writes, `parking_lot` es notablemente mejor.

**Implementar en:** Rust

---

### Ejercicio 13.2.4 — Condvar: coordinación sin polling

**Enunciado:** `Condvar` en Rust funciona igual que en C++/Java pero con
garantías de ownership:

```rust
struct Cola<T> {
    datos: Mutex<VecDeque<T>>,
    no_vacia: Condvar,
}

impl<T> Cola<T> {
    fn push(&self, item: T) {
        self.datos.lock().unwrap().push_back(item);
        self.no_vacia.notify_one();  // despertar a un consumidor
    }

    fn pop(&self) -> T {
        let mut datos = self.datos.lock().unwrap();
        loop {
            if let Some(item) = datos.pop_front() {
                return item;
            }
            // Libera el lock y espera la notificación
            datos = self.no_vacia.wait(datos).unwrap();
        }
    }
}
```

**Restricciones:** Implementar también `pop_timeout(d: Duration) -> Option<T>`.
Verificar que no hay spurious wakeups que generen comportamiento incorrecto.
El `Condvar` siempre debe usarse junto con el `Mutex` que protege la condición.

**Pista:** `Condvar::wait` toma el `MutexGuard` y lo libera atómicamente mientras espera.
El tipo del parámetro hace esto explícito — no puedes usar `wait` sin tener el lock.
En Go, `sync.Cond` funciona igual pero sin el enforcement del tipo.

**Implementar en:** Rust

---

### Ejercicio 13.2.5 — Patrón "typestate" para protocolos concurrentes

**Enunciado:** El sistema de tipos de Rust puede encodear estados de un protocolo
concurrente, haciendo imposible las transiciones incorrectas:

```rust
// Un socket que solo puede enviarse en estado Conectado,
// y solo puede cerrarse una vez
struct Socket<Estado> {
    fd: std::os::unix::io::RawFd,
    _estado: std::marker::PhantomData<Estado>,
}

struct Desconectado;
struct Conectado;

impl Socket<Desconectado> {
    fn conectar(self, addr: &str) -> Result<Socket<Conectado>, IoError> {
        // ...
        Ok(Socket { fd: self.fd, _estado: PhantomData })
    }
}

impl Socket<Conectado> {
    fn enviar(&self, data: &[u8]) -> Result<usize, IoError> { /* ... */ }
    fn cerrar(self) -> Socket<Desconectado> { /* consume self → no se puede usar después */ }
}

// Esto no compila — el tipo lo hace imposible:
// let s = Socket::nuevo();
// s.enviar(b"hola");  ← ERROR: Socket<Desconectado> no tiene método `enviar`
```

**Restricciones:** Implementa un Worker Pool con typestate:
- `PoolBuilder` → `Pool<Iniciando>` → `Pool<Activo>` → `Pool<Detenido>`
- `submit` solo existe en `Pool<Activo>`
- `detener` transiciona a `Pool<Detenido>` y consume el pool
- `Pool<Detenido>` no tiene ningún método que pueda enviar trabajo

**Pista:** El typestate pattern codifica el estado de un objeto en su tipo,
haciendo que las operaciones inválidas en ciertos estados sean errores de compilación.
En Go, esto no es posible directamente — se maneja con panics o errores en runtime.
En Rust, el `PhantomData<Estado>` hace que el compilador trate los distintos
estados como tipos incompatibles aunque compartan la misma representación en memoria.

**Implementar en:** Rust

---

## Sección 13.3 — Canales en Rust: mpsc y Crossbeam

Rust tiene canales en su biblioteca estándar (`std::sync::mpsc`) y en
la biblioteca `crossbeam` (más completa). La diferencia fundamental con Go:
los tipos que fluyen por el canal deben ser `Send`.

```rust
// En Go, puedes enviar cualquier cosa por un canal:
ch := make(chan interface{})  // ← puede ser un race condition

// En Rust, el compilador verifica que el tipo puede enviarse entre threads:
let (tx, rx) = mpsc::channel::<Vec<i64>>();  // Vec<i64> es Send ✓
// let (tx, rx) = mpsc::channel::<Rc<i64>>();  // Rc<i64> no es Send ✗ → error de compilación
```

---

### Ejercicio 13.3.1 — mpsc: el canal estándar de Rust

**Enunciado:** Reimplementa el pipeline del Cap.03 §3.2 en Rust usando canales mpsc:

```rust
fn pipeline(entrada: Vec<i64>) -> Vec<i64> {
    let (tx1, rx1) = mpsc::channel();
    let (tx2, rx2) = mpsc::channel();
    let (tx3, rx3) = mpsc::channel();

    // Etapa 1: filtrar negativos
    thread::spawn(move || {
        for item in entrada {
            if item > 0 { tx1.send(item).unwrap(); }
        }
    });

    // Etapa 2: transformar
    thread::spawn(move || {
        for item in rx1 {
            tx2.send(item * 2).unwrap();
        }
    });

    // Etapa 3: acumular
    thread::spawn(move || {
        for item in rx2 {
            tx3.send(item + 1).unwrap();
        }
    });

    rx3.iter().collect()
}
```

**Restricciones:** El canal se cierra automáticamente cuando todos los `Sender` son dropeados.
El receiver itera hasta que el canal está cerrado y vacío.
Demuestra que en Rust no es posible enviar por un canal cerrado sin error explícito
(a diferencia de Go donde es un panic).

**Pista:** En Rust, `tx.send(item)` retorna `Result<(), SendError<T>>`.
Si el receiver fue dropeado, el send falla. Esto es detectable y manejable.
En Go, enviar a un canal cerrado es un panic no recuperable en la mayoría de los casos.
La diferencia refleja la filosofía: Rust prefiere errores explícitos; Go prefiere fallar rápido.

**Implementar en:** Rust

---

### Ejercicio 13.3.2 — crossbeam: canales más poderosos

**Enunciado:** `crossbeam-channel` tiene canales que no existen en `std::sync::mpsc`:
- Multi-producer multi-consumer (mpmc)
- Select sobre múltiples canales
- After (timeout) y never (nunca ready)

```rust
use crossbeam_channel::{select, tick, after};

fn con_timeout<T>(rx: &Receiver<T>, timeout: Duration) -> Option<T> {
    let timeout_ch = after(timeout);
    select! {
        recv(rx) -> msg => msg.ok(),
        recv(timeout_ch) -> _ => None,
    }
}

fn combinar<T>(rx1: &Receiver<T>, rx2: &Receiver<T>) -> T {
    select! {
        recv(rx1) -> msg => msg.unwrap(),
        recv(rx2) -> msg => msg.unwrap(),
    }
}
```

**Restricciones:** Reimplementa el Or-channel del Ejercicio 5.3.1 usando
`crossbeam::select!`. El select de Rust es más eficiente que el select de Go
para muchos canales porque no requiere goroutines adicionales.

**Pista:** `crossbeam::select!` es una macro que compila a código eficiente —
no hay goroutines adicionales como en el `select` de Go. El `tick(duration)`
crea un canal que recibe un mensaje cada `duration` — útil para heartbeats y timeouts.

**Implementar en:** Rust

---

### Ejercicio 13.3.3 — Rendezvous vs buffered vs unbounded

**Enunciado:** Rust tiene tres tipos de canales con semánticas distintas:

```rust
// Rendezvous (capacidad 0): sender se bloquea hasta que receiver recibe
let (tx, rx) = crossbeam_channel::bounded(0);

// Buffered: sender se bloquea solo si el buffer está lleno
let (tx, rx) = crossbeam_channel::bounded(100);

// Unbounded: sender nunca se bloquea (riesgo de memoria infinita)
let (tx, rx) = crossbeam_channel::unbounded();
```

**Restricciones:** Implementa un sistema de backpressure que:
- Usa bounded(N) para limitar la memoria
- Mide la latencia cuando el canal está 80%, 90%, 100% lleno
- Compara con Go's `chan T` buffered

**Pista:** El canal bounded(0) (rendezvous) garantiza que el sender y receiver
están sincronizados para cada item — como CSP puro. Pero tiene throughput mínimo
porque requiere coordinación por cada item. El unbounded puede usar memoria
sin límite si el producer es más rápido que el consumer — úsalo solo cuando
estás seguro de que la diferencia de velocidad es temporal.

**Implementar en:** Rust

---

### Ejercicio 13.3.4 — Patrón Actor con canales en Rust

**Enunciado:** Implementa el modelo de actores del Cap.06 §6.3 en Rust:

```rust
enum MensajeActor {
    Depositar(i64),
    Retirar(i64, oneshot::Sender<Result<i64, String>>),
    Consultar(oneshot::Sender<i64>),
    Detener,
}

fn actor_cuenta(mut saldo: i64, rx: Receiver<MensajeActor>) {
    for mensaje in rx {
        match mensaje {
            MensajeActor::Depositar(monto) => saldo += monto,
            MensajeActor::Retirar(monto, respuesta) => {
                if saldo >= monto {
                    saldo -= monto;
                    respuesta.send(Ok(saldo)).ok();
                } else {
                    respuesta.send(Err("saldo insuficiente".into())).ok();
                }
            }
            MensajeActor::Consultar(respuesta) => {
                respuesta.send(saldo).ok();
            }
            MensajeActor::Detener => break,
        }
    }
}
```

**Restricciones:** Los actores se comunican solo por mensajes — nunca hay acceso
directo al estado. El compilador garantiza que `saldo` no puede ser accedido
desde fuera del thread del actor (está owned por el closure del actor).

**Pista:** El canal `oneshot` (de `crossbeam` o `tokio`) permite enviar exactamente
un valor de respuesta. Es el equivalente a la semántica ask/reply del actor model.
El `Sender<T>` del oneshot implementa `Send` si `T: Send`, así que el compilador
garantiza que la respuesta puede enviarse entre threads.

**Implementar en:** Rust

---

### Ejercicio 13.3.5 — Benchmark: canales de Rust vs canales de Go

**Enunciado:** Compara el rendimiento de los canales de ambos lenguajes para:
1. SPSC con mensaje de 8 bytes (latencia de round-trip)
2. MPMC con 8 threads (throughput)
3. Select sobre 4 canales (overhead del select)
4. Canal cerrado: detección de cierre

**Restricciones:** Los benchmarks deben ser equivalentes en lógica.
La comparación debe controlarse por el overhead del language runtime
(reportar tanto ns/op como el número de operaciones por segundo).

**Pista:** Go channels tienen overhead de ~400-600 ns por operación.
`crossbeam::bounded` Rust tiene overhead de ~50-100 ns para los mismos patrones.
La diferencia se debe en parte al GC de Go (escaneando referencias) y al
scheduler de goroutines (más pesado que threads del OS en overhead per-message).
Para throughput de millones de mensajes por segundo, la diferencia importa.

**Implementar en:** Rust + Go (para la comparación)

---

## Sección 13.4 — Rayon: Paralelismo de Datos Idiomático

Rayon es la biblioteca de paralelismo de datos de Rust. Transforma código
secuencial en paralelo cambiando `iter()` por `par_iter()`.

**La magia del compile-time:**

```rust
// Secuencial
let suma: f64 = datos.iter().map(|x| x.sqrt()).sum();

// Paralelo — literalmente el mismo código con par_iter
let suma: f64 = datos.par_iter().map(|x| x.sqrt()).sum();

// Si x.sqrt() accede a datos compartidos incorrectamente:
// → Error de compilación (si viola Send/Sync)
// No es posible introducir un race condition silencioso
```

---

### Ejercicio 13.4.1 — Transformaciones paralelas con par_iter

**Enunciado:** Implementa las operaciones del Cap.08 §8.3 (reducción paralela)
usando Rayon:

```rust
use rayon::prelude::*;

fn suma_paralela(v: &[f64]) -> f64 {
    v.par_iter().sum()
}

fn histograma_paralelo(datos: &[i64], bins: usize) -> Vec<usize> {
    datos.par_iter()
         .fold(|| vec![0usize; bins], |mut acc, &x| {
             acc[(x as usize) % bins] += 1;
             acc
         })
         .reduce(|| vec![0usize; bins], |mut a, b| {
             a.iter_mut().zip(b.iter()).for_each(|(ai, bi)| *ai += bi);
             a
         })
}
```

**Restricciones:** El histograma usa fold+reduce para evitar contención:
cada thread tiene su acumulador local, luego se combinan.
Mide el speedup real vs la versión secuencial para 1M, 10M, y 100M elementos.

**Pista:** `par_iter().fold()` en Rayon crea un acumulador local por thread.
`reduce()` combina los acumuladores al final. Este es el patrón "tree reduction"
del Cap.08 §8.3 pero expresado funcionalmente.
La corrección está garantizada por el tipo: si el acumulador no es `Send`,
el código no compila.

**Implementar en:** Rust

---

### Ejercicio 13.4.2 — Rayon con estructuras personalizadas

**Enunciado:** Para usar `par_iter()` con un tipo personalizado, debes implementar
`IntoParallelIterator`:

```rust
struct ArbolBinario<T> {
    valor: T,
    izq: Option<Box<ArbolBinario<T>>>,
    der: Option<Box<ArbolBinario<T>>>,
}

// Para permitir par_iter() en el árbol:
impl<T: Send> rayon::iter::IntoParallelIterator for ArbolBinario<T> {
    type Item = T;
    type Iter = /* ... */;
    fn into_par_iter(self) -> Self::Iter { /* ... */ }
}

// Uso:
let suma: i64 = arbol.into_par_iter().sum();
```

**Restricciones:** El iterador paralelo debe dividir el árbol en subárboles
y asignarlos a distintos threads. Usar el `split_at` de Rayon para la división.

**Pista:** Implementar `IntoParallelIterator` requiere implementar `Producer`
(que puede dividirse) y `UnindexedProducer` (para divisiones arbitrarias).
El árbol binario se divide naturalmente por sus subárboles.
Para árboles desbalanceados, Rayon usará work-stealing para balancear la carga.

**Implementar en:** Rust

---

### Ejercicio 13.4.3 — par_sort y par_sort_unstable

**Enunciado:** Rayon tiene sort paralelo integrado:

```rust
let mut datos: Vec<f64> = /* ... */;
datos.par_sort_by(|a, b| a.partial_cmp(b).unwrap());
// o la versión más rápida pero inestable:
datos.par_sort_unstable_by(|a, b| a.partial_cmp(b).unwrap());
```

Mide el speedup de `par_sort_unstable` vs `sort_unstable` para:
- 1M floats aleatorios
- 1M floats casi ordenados (1% out of order)
- 1M strings de longitud variable
- 1M structs comparados por múltiples campos

**Restricciones:** El comparador puede ser arbitrario (sin restricciones de tipo
más allá de los traits), pero debe ser determinista (mismo resultado para el mismo input).

**Pista:** Para floats, `f64` no implementa `Ord` (por NaN) — debes usar `partial_cmp`
y decidir cómo manejar NaN. Para strings, el sort es más lento que para números
por la longitud variable y la comparación lexicográfica por bytes.
`par_sort_unstable` usa un introsort paralelo similar al de la STL de C++.

**Implementar en:** Rust

---

### Ejercicio 13.4.4 — Rayon scope: el equivalente de thread::scope para Rayon

**Enunciado:** `rayon::scope` permite lanzar tareas de Rayon con lifetimes
no `'static`, similar a `thread::scope`:

```rust
fn procesar_con_referencias(datos: &[i64], resultado: &mut Vec<i64>) {
    rayon::scope(|s| {
        for (chunk_in, chunk_out) in
            datos.chunks(1000).zip(resultado.chunks_mut(1000))
        {
            s.spawn(|_| {
                for (input, output) in chunk_in.iter().zip(chunk_out.iter_mut()) {
                    *output = input * input;
                }
            });
        }
    });
}
```

**Restricciones:** Las referencias prestadas a las tareas deben tener lifetime
mayor que el scope. El compilador debe rechazar referencias con lifetime menor.

**Pista:** `rayon::scope` funciona como `thread::scope` pero usa el thread pool
de Rayon en lugar de threads del OS. La ventaja: menos overhead de creación de threads.
La desventaja: las tareas dentro del scope comparten el thread pool con otras tareas
de Rayon — puede haber interferencia si el pool está saturado.

**Implementar en:** Rust

---

### Ejercicio 13.4.5 — Rayon vs std::thread vs tokio::task

**Enunciado:** Rust tiene tres modelos de paralelismo/concurrencia:
- `std::thread`: threads del OS, para trabajo CPU-bound de larga duración
- `rayon`: fork-join sobre thread pool, para paralelismo de datos
- `tokio::task`: async tasks sobre runtime, para I/O-bound

Implementa el mismo sistema (procesador de archivos) con los tres modelos
y compara:
- Throughput para 1000 archivos de 1MB (CPU-bound: compresión)
- Throughput para 1000 requests HTTP (I/O-bound)
- Overhead de scheduling
- Uso de memoria

**Restricciones:** Los tres modelos deben producir el mismo resultado.
Para la compresión (CPU-bound), `rayon` debe ganar.
Para HTTP (I/O-bound), `tokio::task` debe ganar.

**Pista:** Mezclar Rayon y Tokio requiere cuidado: ejecutar código de Rayon
dentro de un async Tokio bloquea el runtime. La solución:
`tokio::task::spawn_blocking(|| { rayon_code... })` ejecuta código bloqueante
en un thread pool separado, sin bloquear el runtime de Tokio.

**Implementar en:** Rust

---

## Sección 13.5 — Async/Await: Concurrencia sin Threads del OS

El async/await de Rust es diferente al de otros lenguajes:

```
Python async: event loop integrado, await es explícito, runtime fijo
JavaScript: event loop del browser/Node.js, promesas automáticas
C# async: integrado con el runtime .NET, transparent scheduling
Go: goroutines (más ligero que threads del OS), canales para comunicación

Rust async:
  - NO hay runtime incluido en la biblioteca estándar
  - Debes elegir un runtime (Tokio, async-std, smol)
  - Futures son lazy: no hacen nada hasta que son polled
  - Zero-cost abstraction: el compilador transforma async fn en state machines
  - Las state machines se alinean en memoria — sin allocation por Future
```

**La diferencia fundamental: Futures son lazy**

```rust
// En Python/JavaScript, esto inicia la operación:
let future = fetch("http://example.com");  // ← ya empezó

// En Rust, esto solo crea la state machine — no hace nada todavía:
let future = fetch("http://example.com");  // ← nada ocurrió aún

// Solo al "awaitear" o al dársela al runtime, empieza:
let resultado = future.await;  // ← ahora ocurre
```

---

### Ejercicio 13.5.1 — Futures como state machines

**Enunciado:** Implementa un Future manualmente para entender cómo funciona el compilador:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// El compilador transforma `async fn esperar(n: u64) -> u64`
// en algo equivalente a esto:
struct FutureEsperar {
    estado: EstadoEsperar,
    n: u64,
}

enum EstadoEsperar {
    Inicio,
    EsperandoTimer(/* timer future */),
    Completado,
}

impl Future for FutureEsperar {
    type Output = u64;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u64> {
        match self.estado {
            EstadoEsperar::Inicio => {
                // Iniciar el timer
                self.estado = EstadoEsperar::EsperandoTimer(/* ... */);
                Poll::Pending
            }
            EstadoEsperar::EsperandoTimer(ref mut timer) => {
                // Verificar si el timer terminó
                match Pin::new(timer).poll(cx) {
                    Poll::Ready(_) => {
                        self.estado = EstadoEsperar::Completado;
                        Poll::Ready(self.n)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
            EstadoEsperar::Completado => panic!("polled after completion"),
        }
    }
}
```

**Restricciones:** Implementar un Future que hace tres operaciones secuenciales
sin usar `async/await`, para entender la transformación que hace el compilador.

**Pista:** El compilador transforma `async fn` en una struct que implementa `Future`.
Cada punto de `await` es un estado en el enum. El `poll` avanza la state machine
hasta el próximo punto de `await` o hasta completar. Implementar esto manualmente
una vez es el mejor tutorial sobre cómo funciona async/await en Rust.

**Implementar en:** Rust

---

### Ejercicio 13.5.2 — Tokio: el runtime más popular

**Enunciado:** Reimplementa el servidor HTTP del Ejercicio 9.1.3 con Tokio:

```rust
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    let cache = Arc::new(Cache::new(1_000_000));

    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let cache = Arc::clone(&cache);
        tokio::spawn(async move {
            manejar_conexion(stream, cache).await;
        });
    }
}

async fn manejar_conexion(stream: TcpStream, cache: Arc<Cache>) {
    // Leer request, buscar en cache, responder
}
```

**Restricciones:** Mide el throughput con 1000, 10000, 100000 conexiones concurrentes.
Compara con el servidor de Go. El servidor de Rust con Tokio debe tener menor
uso de memoria por conexión que el servidor de Go.

**Pista:** Tokio tasks son mucho más ligeras que goroutines de Go en términos de memoria:
~100 bytes vs ~2 KB para una goroutine nueva. Para 100,000 conexiones:
Tokio: ~10 MB; Go: ~200 MB. La diferencia viene de que las Futures de Rust son
state machines en el stack (o como struct si son grandes), mientras que las goroutines
de Go tienen un stack separado que crece dinámicamente.

**Implementar en:** Rust

---

### Ejercicio 13.5.3 — tokio::select! y cancelación

**Enunciado:** `tokio::select!` es el equivalente del `select` de Go para async:

```rust
async fn con_timeout<F, T>(future: F, timeout: Duration) -> Option<T>
where
    F: Future<Output = T>,
{
    tokio::select! {
        resultado = future => Some(resultado),
        _ = tokio::time::sleep(timeout) => None,
    }
}
```

**La diferencia crítica con Go:** cuando un branch de `select!` es cancelado,
la Future correspondiente recibe una señal de cancelación a través del `drop`.
Los Futures deben ser "cancellation-safe" — no pueden estar a mitad de una
operación crítica cuando son cancelados.

**Restricciones:** Implementa el Circuit Breaker del Cap.03 §3.6 con async/await y
`select!` para el timeout. Verifica que la cancelación es safe: no hay state corruption
si el Future es cancelado mientras espera una respuesta.

**Pista:** No todos los Futures son cancellation-safe. Por ejemplo, un Future que
hace `recv()` en un canal y luego procesa el mensaje: si es cancelado después de
`recv()` pero antes de procesar, el mensaje se pierde. Tokio documenta cuáles
de sus tipos son cancellation-safe. Para Futures propios, debes verificarlo.

**Implementar en:** Rust

---

### Ejercicio 13.5.4 — Streams: async iteradores

**Enunciado:** Un `Stream` es el equivalente async de un `Iterator`:
produce items uno a uno, yielding el control entre items:

```rust
use tokio_stream::StreamExt;

async fn procesar_stream(stream: impl Stream<Item = Resultado>) {
    let mut stream = stream;
    while let Some(item) = stream.next().await {
        procesar(item).await;
    }
}

// Con operadores funcionales:
stream
    .filter(|x| x.es_valido())
    .map(|x| transformar(x))
    .take(1000)
    .collect::<Vec<_>>()
    .await
```

**Restricciones:** Implementa el pipeline del Cap.03 §3.2 como un Stream:
cada etapa es una transformación del stream. Las etapas corren concurrentemente
pero no en paralelo (un stream es single-threaded en el runtime).

**Pista:** La diferencia entre `Stream` e `Iterator`: el `Stream` puede yieldar
al runtime entre items, permitiendo que otros tasks progresen.
Para paralelismo real (múltiples items procesados simultáneamente),
usa `buffer_unordered(N)` que mantiene N Futures activos simultáneamente.

**Implementar en:** Rust

---

### Ejercicio 13.5.5 — async trait y el overhead cero

**Enunciado:** Hasta Rust 1.75, `async fn` en traits requería la biblioteca
`async-trait` que usa boxing. Desde Rust 1.75, es estable y sin boxing:

```rust
// Antes de Rust 1.75 (con boxing, ~20ns overhead):
#[async_trait::async_trait]
trait Servicio {
    async fn procesar(&self, req: Request) -> Response;
}

// Rust 1.75+ (sin boxing, zero-cost):
trait Servicio {
    async fn procesar(&self, req: Request) -> Response;
    // El compilador genera el Return Position Impl Trait
}
```

**Restricciones:** Implementa el Worker Pool del Cap.03 como un trait async.
Mide el overhead del boxing vs el zero-cost implementation.
La diferencia debe ser visible en microbenchmarks pero insignificante en macrobenchmarks.

**Pista:** El overhead del boxing en `async_trait` es ~20-50 ns por llamada.
Para un servidor que maneja 1M req/s, 20 ns × 1M = 20 ms extra por segundo = 2% overhead.
Para sistemas de alta frecuencia, esto importa. Para la mayoría de sistemas web, no.

**Implementar en:** Rust

---

## Sección 13.6 — Atomics y Lock-Free en Rust

Rust expone los atomics con `Ordering` explícito, similar a C++11.
A diferencia de Go (que siempre usa seq_cst), Rust permite especificar
exactamente qué garantías necesitas.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

let contador = AtomicUsize::new(0);

// Relaxed: solo atomicidad, sin ordering garantizado
contador.fetch_add(1, Ordering::Relaxed);

// Release/Acquire: para sincronizar con otra thread
contador.store(1, Ordering::Release);
let v = contador.load(Ordering::Acquire);

// SeqCst: orden total global — más costoso
contador.fetch_add(1, Ordering::SeqCst);
```

---

### Ejercicio 13.6.1 — Implementar spinlock con Ordering explícito

**Enunciado:** Implementa el spinlock del Cap.11 §11.3.4 en Rust con
Ordering correcto:

```rust
pub struct Spinlock {
    estado: AtomicBool,
}

impl Spinlock {
    pub fn lock(&self) -> SpinlockGuard<'_> {
        while self.estado.compare_exchange_weak(
            false, true,
            Ordering::Acquire,   // éxito: acquire — ver las escrituras antes del unlock
            Ordering::Relaxed,   // fallo: relaxed — solo reintentando
        ).is_err() {
            // Hint al CPU para reducir contención en el bus
            std::hint::spin_loop();
        }
        SpinlockGuard { lock: self }
    }
}

impl Drop for SpinlockGuard<'_> {
    fn drop(&mut self) {
        self.lock.estado.store(false, Ordering::Release);
        // Release: las escrituras anteriores son visibles a quien hace Acquire
    }
}
```

**Restricciones:** El `SpinlockGuard` implementa RAII — no se puede "olvidar"
liberar el lock. Verifica que `Acquire`/`Release` son los orderings correctos
(no `Relaxed`, que permitiría ver estado inconsistente después del lock).

**Pista:** `compare_exchange_weak` puede fallar spuriously (permite instrucciones LL/SC
en ARM que son más eficientes). Para un spinlock, esto es correcto porque
igual reintentaría. `compare_exchange` (strong) no falla spuriously pero es menos
eficiente en ARM. En x86, ambos tienen el mismo comportamiento.

**Implementar en:** Rust

---

### Ejercicio 13.6.2 — Stack lock-free en Rust con Ordering correcto

**Enunciado:** Reimplementa el Stack lock-free del Cap.12 §12.1 en Rust
con Orderings explícitos:

```rust
pub struct StackLF<T> {
    head: AtomicPtr<Nodo<T>>,
}

struct Nodo<T> {
    valor: T,
    next: *mut Nodo<T>,
}

impl<T: Send> StackLF<T> {
    pub fn push(&self, valor: T) {
        let nodo = Box::into_raw(Box::new(Nodo {
            valor,
            next: std::ptr::null_mut(),
        }));
        loop {
            let head = self.head.load(Ordering::Acquire);
            unsafe { (*nodo).next = head; }
            if self.head.compare_exchange(
                head, nodo,
                Ordering::Release,
                Ordering::Relaxed,
            ).is_ok() {
                break;
            }
        }
    }
}
```

**Restricciones:** El código usa `unsafe` para los punteros raw. Cada `unsafe`
debe estar documentado con `// SAFETY:`. La implementación debe ser correcta
bajo el race detector de Rust (ThreadSanitizer).

**Pista:** La memory reclamation es el problema: cuando haces `pop()`, retornas
el valor pero ¿cuándo liberas el nodo? Con GC (Go/Java) es automático.
Sin GC (Rust/C++), necesitas hazard pointers o epoch-based reclamation.
La solución pragmática en Rust: usar `crossbeam::epoch` para la reclamación.

**Implementar en:** Rust

---

### Ejercicio 13.6.3 — epoch-based reclamation con crossbeam

**Enunciado:** `crossbeam::epoch` resuelve el problema de memory reclamation
para estructuras lock-free:

```rust
use crossbeam::epoch::{self, Atomic, Owned, Shared};

struct StackEpoch<T> {
    head: Atomic<NodoEpoch<T>>,
}

impl<T: Send + Sync> StackEpoch<T> {
    fn push(&self, valor: T) {
        let guard = epoch::pin();  // "anclar" la epoch actual
        let nodo = Owned::new(NodoEpoch { valor, next: Atomic::null() });
        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            nodo.next.store(head, Ordering::Relaxed);  // incorrecto, ver pista
            // ...
        }
    }

    fn pop(&self) -> Option<T> {
        let guard = epoch::pin();
        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            // Si pop exitoso, marcar el nodo para liberación:
            // unsafe { guard.defer_destroy(head); }
            // El nodo se liberará cuando ningún thread tenga la epoch "pinned"
        }
    }
}
```

**Restricciones:** Implementar el stack completo con epoch-based reclamation.
No debe haber memory leaks detectados por `valgrind` o `AddressSanitizer`.

**Pista:** `epoch::pin()` incrementa el contador de referencia de la epoch actual.
Mientras un thread está "pinned", el GC no puede liberar objetos de esa epoch.
`guard.defer_destroy(ptr)` marca el puntero para liberación cuando todos los
threads que la tenían "pinned" avancen a una epoch posterior.

**Implementar en:** Rust (con crossbeam)

---

### Ejercicio 13.6.4 — Compare-and-exchange con tagged pointers

**Enunciado:** Implementa tagged pointers en Rust para resolver el ABA problem
del Stack lock-free:

```rust
// Los punteros en x86-64 solo usan 48 bits
// Podemos usar los 16 bits superiores como tag/versión

struct TaggedPtr<T> {
    ptr: AtomicU64,  // los 48 bits bajos son el puntero, 16 altos son el tag
}

impl<T> TaggedPtr<T> {
    fn load(&self) -> (*mut T, u16) {
        let raw = self.ptr.load(Ordering::Acquire);
        let tag = (raw >> 48) as u16;
        let ptr = (raw & 0x0000_FFFF_FFFF_FFFF) as *mut T;
        (ptr, tag)
    }

    fn cas(&self, old_ptr: *mut T, old_tag: u16, new_ptr: *mut T, new_tag: u16) -> bool {
        let old = ((old_tag as u64) << 48) | (old_ptr as u64);
        let new = ((new_tag as u64) << 48) | (new_ptr as u64);
        self.ptr.compare_exchange(old, new, Ordering::AcqRel, Ordering::Relaxed).is_ok()
    }
}
```

**Restricciones:** Toda la operación con punteros raw está en `unsafe`.
Documenta con `// SAFETY:` por qué cada operación es correcta.
Verifica que el ABA problem del Ejercicio 12.1.2 no ocurre con tagged pointers.

**Pista:** En ARM64 (Apple M1/M2), los punteros tienen 48 bits de dirección virtual.
En x86-64, también. Pero en sistemas con mode de dirección de 5 niveles (Linux 5-level paging),
los punteros pueden usar 57 bits, dejando solo 7 bits para el tag — suficiente para muchos usos.
Verificar las capacidades del CPU antes de asumir 16 bits disponibles.

**Implementar en:** Rust

---

### Ejercicio 13.6.5 — Benchmark: atomics Rust vs Go vs C++

**Enunciado:** Compara el rendimiento de atomics en los tres lenguajes para:
1. Contador atómico: `fetch_add` con Relaxed (Rust) vs `sync/atomic.AddInt64` (Go)
2. CAS loop: compare_exchange tight loop
3. SPSC ring buffer (del Cap.12 §12.5.1)
4. Spinlock: Rust vs sync.Mutex de Go

**Restricciones:** Los benchmarks deben ser equivalentes. Controlar por el overhead
del compilador: verificar con el ensamblador generado que las instrucciones son las esperadas.

**Pista:** Para Rust: `cargo objdump --release` o `cargo asm` muestra el ensamblador.
Para Go: `go tool compile -S`. Para C++: `gcc -S -O2`.
La instrucción atómica en x86 es `lock add`, `lock cmpxchg`, etc.
En ARM: `ldaxr`/`stlxr` (load-acquire exclusive, store-release exclusive).

**Implementar en:** Rust · Go · C++ (para comparación)

---

## Sección 13.7 — Rust vs Go: Cuándo Elegir Cada Uno

La pregunta más práctica del capítulo: ¿para qué tipo de proyectos elegir Rust
y para cuáles Go?

**La tabla de decisión:**

```
                            Go              Rust
──────────────────────────────────────────────────────────────────
Velocidad de desarrollo     ████████████    ████████
Seguridad en concurrencia   ████████        ████████████ (compile-time)
Rendimiento                 ████████        ████████████
Uso de memoria              ████████        ████████████ (sin GC)
Ecosistema web/cloud        ████████████    ████████
Ecosistema sistemas         ████████        ████████████
Curva de aprendizaje        ████            ████████████ (más difícil)
Latencia consistente        ████████        ████████████ (sin GC pauses)
```

**Los casos de uso donde Rust gana claramente:**
- Sistemas embebidos (sin GC, sin runtime)
- Drivers de dispositivos, código del kernel
- Latencia < 1ms consistente (sin GC pauses)
- Seguridad crítica (memory safety + thread safety por tipos)
- Alta performance con baja memoria (servidores de alta densidad)

**Los casos de uso donde Go gana claramente:**
- Servicios web y microservicios (productividad > rendimiento máximo)
- CLIs y herramientas de administración
- Sistemas distribuidos (mejor ecosistema: gRPC, Kubernetes client, etc.)
- Equipos que necesitan moverse rápido
- Prototipado e iteración rápida

---

### Ejercicio 13.7.1 — El mismo sistema en Go y Rust: medir la diferencia real

**Enunciado:** Implementa el sistema de procesamiento de mensajes del Ejercicio 10.7.3
(Workflow Engine) en ambos lenguajes y mide:
- Latencia p50, p99, p999
- Throughput máximo (requests/segundo)
- Uso de memoria bajo carga
- Tiempo de desarrollo estimado (líneas de código, complejidad)

**Restricciones:** Los sistemas deben ser funcionalmente equivalentes.
Las mediciones de latencia deben controlar por el GC de Go (medir durante GC pauses).

**Pista:** La diferencia más notable suele ser el p999 (percentil 99.9).
En Go, el GC puede causar pauses de 0.5-5 ms que aparecen en el p999.
En Rust, sin GC, el p999 es similar al p99. Para sistemas donde el p999 importa
(trading, gaming, tiempo real), esta diferencia es significativa.

**Implementar en:** Go + Rust (comparación directa)

---

### Ejercicio 13.7.2 — Memory safety: demostrar qué previene Rust

**Enunciado:** Implementa estos bugs comunes en C/C++ y verifica que Rust
los previene en tiempo de compilación o con AddressSanitizer:

```c
// Bug 1: use-after-free
int *ptr = malloc(sizeof(int));
free(ptr);
*ptr = 42;  // use-after-free — undefined behavior en C, error de compilación en Rust

// Bug 2: double-free
free(ptr);
free(ptr);  // undefined behavior en C, error de compilación en Rust (solo un owner)

// Bug 3: data race
// thread 1: x++;
// thread 2: x++;  // data race en C — error de compilación en Rust

// Bug 4: iterator invalidation
std::vector<int> v = {1,2,3};
auto it = v.begin();
v.push_back(4);  // puede invalidar iteradores
*it;  // undefined behavior en C++ — error de compilación en Rust
```

**Restricciones:** Para cada bug: (a) código en C/C++ que tiene el bug,
(b) el código Rust equivalente que no compila, (c) el error de compilación exacto,
(d) el código Rust correcto.

**Pista:** Rust hace que estos bugs sean errores de compilación a través del
sistema de ownership: (1) use-after-free: no puedes usar después de move/drop;
(2) double-free: solo un owner, se libera una vez; (3) data race: Send/Sync;
(4) iterator invalidation: el iterator tiene un borrow del vector — no puedes
mutar el vector mientras el borrow está activo.

**Implementar en:** C++ (el código con bugs) + Rust (los errores de compilación)

---

### Ejercicio 13.7.3 — El costo del borrow checker: cuánto ralentiza el desarrollo

**Enunciado:** El borrow checker previene bugs, pero también requiere que el
programador piense más cuidadosamente sobre el ownership. ¿Cuánto tiempo toma
aprender a trabajar con él eficientemente?

Implementa estos cinco patrones y mide cuántas iteraciones de compilación necesitas:

1. Grafo con ciclos (nodos que se apuntan mutuamente)
2. Cache con referencias a datos en otra estructura
3. Builder pattern con herencia de estado
4. Event system con múltiples observers
5. Parser recursivo con estado compartido

**Restricciones:** Registra el número de errores de compilación de borrow checker
antes de llegar a una solución que compile. Clasifica los errores: ¿son bugs reales
que el compilador detectó, o limitaciones del borrow checker?

**Pista:** El borrow checker rechaza a veces código que sería correcto en la práctica
(false positives). Las soluciones: `Arc<RefCell<T>>` para grafo con ciclos (runtime safety),
`Rc<Cell<T>>` para referencia self-referential (runtime safety), o restructurar el código.
El "fighting the borrow checker" es una experiencia común para programadores nuevos en Rust
— y una señal de que el diseño puede mejorarse.

**Implementar en:** Rust (con conteo de iteraciones)

---

### Ejercicio 13.7.4 — Performance-critical code: Rust donde importa, Go donde es suficiente

**Enunciado:** Diseña un sistema donde el core de alta performance está en Rust
y la lógica de negocio en Go, comunicados por FFI (Foreign Function Interface):

```go
// Go: la interfaz de alto nivel
// #cgo LDFLAGS: -L. -lcore_rust
// #include "core_rust.h"
import "C"

func ProcesarDatos(datos []byte) []byte {
    // Llamar a la función de Rust para el procesamiento intensivo
    resultado := C.procesar_core(
        (*C.uint8_t)(unsafe.Pointer(&datos[0])),
        C.size_t(len(datos)),
    )
    return C.GoBytes(resultado.ptr, C.int(resultado.len))
}
```

```rust
// Rust: el core de alta performance
#[no_mangle]
pub extern "C" fn procesar_core(data: *const u8, len: usize) -> SliceResult {
    let datos = unsafe { std::slice::from_raw_parts(data, len) };
    let resultado = procesar_rust(datos);
    // ...
}
```

**Restricciones:** La interfaz FFI debe ser minimal (solo los datos necesarios).
El overhead del cruce FFI debe ser medido y documentado.

**Pista:** El overhead del FFI es ~5-20 ns por llamada (similar a una llamada de función normal).
Para llamadas de alta frecuencia (>1M/s), el overhead importa.
Para llamadas de baja frecuencia (<1K/s), es completamente irrelevante.
Este patrón (Go para la lógica, Rust para el hot path) es común en sistemas de ML
donde el modelo en Rust es llamado desde un servidor en Go.

**Implementar en:** Rust + Go (con CGo)

---

### Ejercicio 13.7.5 — Guía de decisión: Go vs Rust para tu próximo proyecto

**Enunciado:** Basándote en los ejercicios del capítulo, escribe una guía de decisión
personal con los siguientes elementos:

1. **5 señales de que deberías elegir Rust:**
   - Ejemplo concreto de cada señal
   - El costo que evitas con Rust en ese caso

2. **5 señales de que deberías elegir Go:**
   - Ejemplo concreto de cada señal
   - El costo que evitas con Go en ese caso

3. **3 casos donde ambos son adecuados:**
   - El criterio de desempate para tu equipo/proyecto específico

4. **Un caso real de tu experiencia (o hipotético detallado):**
   - La decisión que tomarías y por qué

**Restricciones:** La guía debe ser específica y opinionada, no una lista de
"depende". Cada recomendación debe tener una justificación técnica concreta.

**Pista:** La decisión más honesta: para la mayoría de proyectos de aplicación web,
Go es la elección correcta por la velocidad de desarrollo y el ecosistema.
Para sistemas donde la seguridad de memoria es crítica (cryptografía, parsers de datos
de fuentes no confiables) o donde la latencia consistente importa más que el throughput
promedio, Rust justifica su curva de aprendizaje más alta.

**Implementar en:** Documento técnico + implementaciones de respaldo

---

## Resumen del capítulo

**Lo que Rust hace diferente en concurrencia:**

| Go | Rust |
|---|---|
| Race conditions detectadas en runtime (con -race) | Races son errores de compilación |
| Goroutines con GC — simples de crear | Threads del OS + async tasks — más control |
| Canales son la primitiva principal | Canales + ownership + async/await |
| sync.Mutex no está vinculada al dato | Mutex<T> envuelve el dato — no se puede acceder sin lock |
| No hay Ordering en atomics (siempre seq_cst) | Ordering explícito — puede optimizarse |
| GC pauses visibles en p99/p999 | Sin GC — latencia más consistente |
| Productividad alta, especialmente para servicios web | Productividad menor, pero errores menores |

**El principio central:**

> Rust traslada la responsabilidad de la concurrencia correcta del programador
> al compilador. El código correcto es el único que compila.
> El costo es un sistema de tipos más complejo y una curva de aprendizaje más alta.
> La ganancia es que una clase entera de bugs — races, use-after-free, iterator
> invalidation — son imposibles de introducir inadvertidamente.

## La pregunta que guía el Cap.14

> El Cap.14 entra en Java: el lenguaje donde la concurrencia se inventó
> para la JVM, el ecosistema más rico de primitivas concurrentes,
> y los Virtual Threads de Java 21 que cambiaron el panorama.
