# Guía de Ejercicios — C# y .NET: Donde async/await Nació

> Implementar todos los ejercicios en **C# 12 / .NET 8+**.
> Comparaciones con Go, Java, Rust, y Python donde iluminen el contraste.
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## Antes de la teoría: el inventor de async/await

```csharp
// C# 5.0 (2012) — el primer lenguaje con async/await integrado en el lenguaje:
public async Task<string> ObtenerDatosAsync(string url)
{
    using var client = new HttpClient();
    string respuesta = await client.GetStringAsync(url);  // ← esto
    return respuesta.ToUpper();
}
```

Antes de C# 5.0 (2012), el código asíncrono requería callbacks explícitos:

```csharp
// C# 4.0 y anterior — APM (Asynchronous Programming Model):
client.BeginGetResponse(result => {
    var response = client.EndGetResponse(result);
    response.BeginRead(readResult => {
        var bytes = response.EndRead(readResult);
        // y así, N niveles de callbacks...
    }, null);
}, null);
```

C# inventó `async/await`. JavaScript lo adoptó en ES2017. Python en 3.5.
Rust en 1.39. Java en 21. El modelo que todos los lenguajes modernos usan
fue diseñado en Microsoft Research y el equipo de C# a partir de 2010.

---

## El modelo de concurrencia de .NET

```
.NET Thread Pool:
  Thread pool gestionado automáticamente
  Escala basándose en la carga y el número de cores
  También maneja I/O Completion Ports (IOCP) en Windows
  en Linux/macOS: epoll/kqueue + thread pool

Task Parallel Library (TPL):
  Task<T>: la unidad de trabajo asíncrono
  Parallel.For, Parallel.ForEach: paralelismo de datos
  PLINQ (Parallel LINQ): consultas paralelas

async/await:
  Compilado a state machines (igual que en Rust)
  Zero allocation en el happy path (sin boxing de Tasks comunes)
  ConfigureAwait(false) para bibliotecas (no capturar el SynchronizationContext)

Channels (System.Threading.Channels):
  El equivalente de los canales de Go, en .NET
  BoundedChannel, UnboundedChannel
  Single/Multiple reader, Single/Multiple writer
```

---

## Tabla de contenidos

- [Sección 16.1 — Task y async/await: los fundamentos](#sección-161--task-y-asyncawait-los-fundamentos)
- [Sección 16.2 — Task Parallel Library: paralelismo de datos](#sección-162--task-parallel-library-paralelismo-de-datos)
- [Sección 16.3 — System.Threading.Channels: el canal de .NET](#sección-163--systemthreadingchannels-el-canal-de-net)
- [Sección 16.4 — ValueTask y Zero-Allocation async](#sección-164--valuetask-y-zero-allocation-async)
- [Sección 16.5 — CancellationToken: cancelación en .NET](#sección-165--cancellationtoken-cancelación-en-net)
- [Sección 16.6 — IAsyncEnumerable: streams asíncronos](#sección-166--iasyncenumerable-streams-asíncronos)
- [Sección 16.7 — Comparación final: los cuatro lenguajes](#sección-167--comparación-final-los-cuatro-lenguajes)

---

## Sección 16.1 — Task y async/await: Los Fundamentos

`Task<T>` es la unidad de trabajo asíncrono en .NET — el equivalente de
`Future<T>` en Java y `CompletableFuture` en su API completa.

**La transformación del compilador:**

```csharp
// Lo que escribes:
public async Task<int> CalcularAsync()
{
    int a = await ObtenerAAsync();
    int b = await ObtenerBAsync();
    return a + b;
}

// Lo que el compilador genera (simplificado):
public Task<int> CalcularAsync()
{
    var stateMachine = new CalcularAsyncStateMachine();
    stateMachine.builder = AsyncTaskMethodBuilder<int>.Create();
    stateMachine.state = -1;
    stateMachine.builder.Start(ref stateMachine);
    return stateMachine.builder.Task;
}

private struct CalcularAsyncStateMachine : IAsyncStateMachine
{
    public int state;
    public AsyncTaskMethodBuilder<int> builder;
    private int a, b;
    private TaskAwaiter<int> awaiter;

    public void MoveNext()
    {
        switch (state)
        {
            case -1:
                awaiter = ObtenerAAsync().GetAwaiter();
                if (!awaiter.IsCompleted) { state = 0; builder.AwaitUnsafeOnCompleted(ref awaiter, ref this); return; }
                goto case 0;
            case 0:
                a = awaiter.GetResult();
                awaiter = ObtenerBAsync().GetAwaiter();
                if (!awaiter.IsCompleted) { state = 1; builder.AwaitUnsafeOnCompleted(ref awaiter, ref this); return; }
                goto case 1;
            case 1:
                b = awaiter.GetResult();
                builder.SetResult(a + b);
                break;
        }
    }
}
```

---

### Ejercicio 16.1.1 — async/await básico y el SynchronizationContext

**Enunciado:** Uno de los conceptos más confusos de C# async: el `SynchronizationContext`.
En aplicaciones UI (WPF, WinForms, Blazor), el `await` captura el contexto
del thread UI y continúa en él — para poder actualizar la UI sin `Invoke`.
En bibliotecas de clase, esto puede causar deadlocks.

```csharp
// En una aplicación WPF (con SynchronizationContext):
private async void Button_Click(object sender, EventArgs e)
{
    // Estamos en el UI thread
    string datos = await ObtenerDatosAsync();  // await captura el SynchronizationContext
    lblResultado.Text = datos;  // continúa en el UI thread — correcto para UI
}

// En una biblioteca (sin necesidad de UI thread):
public async Task<string> ObtenerDatosAsync()
{
    await Task.Delay(100).ConfigureAwait(false);  // NO capturar el SynchronizationContext
    return "datos";  // continúa en cualquier thread del pool
}
```

**Restricciones:** Implementa los dos patrones y demuestra el deadlock clásico:
llamar `.Result` o `.Wait()` desde el UI thread en una Task que intentó
volver al UI thread después del await.

```csharp
// DEADLOCK clásico:
// UI thread llama .Result → espera la Task → Task intenta volver al UI thread
//                                          → UI thread está bloqueado esperando .Result
private void Button_Click_Incorrecto(object sender, EventArgs e)
{
    var datos = ObtenerDatosAsync().Result;  // ← DEADLOCK
}
```

**Pista:** La regla para evitar el deadlock: en bibliotecas, siempre usar
`ConfigureAwait(false)` para no capturar el SynchronizationContext.
En aplicaciones de consola y ASP.NET Core, no hay SynchronizationContext —
el deadlock no ocurre. Por eso el código que hace `.Result` a veces funciona
y a veces no, dependiendo del contexto.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.1.2 — Task.WhenAll y Task.WhenAny

**Enunciado:** Reimplementa el scatter/gather del Ejercicio 14.3.2 en C#:

```csharp
public async Task<List<string>> FetchTodosAsync(IEnumerable<string> urls)
{
    var tasks = urls.Select(url => FetchAsync(url)).ToList();

    // WhenAll: esperar todos — si alguno falla, propaga la primera excepción
    var resultados = await Task.WhenAll(tasks);
    return resultados.ToList();
}

public async Task<string> FetchPrimeroAsync(IEnumerable<string> urls)
{
    // WhenAny: retornar cuando el primero complete
    var tasks = urls.Select(url => FetchAsync(url)).ToList();
    var primera = await Task.WhenAny(tasks);
    return await primera;  // puede lanzar excepción si la Task falló
}

// Con manejo de errores individuales:
public async Task<List<Result<string>>> FetchConErroresAsync(IEnumerable<string> urls)
{
    var tasks = urls.Select(async url =>
    {
        try { return Result.Ok(await FetchAsync(url)); }
        catch (Exception e) { return Result.Error<string>(e); }
    });
    return (await Task.WhenAll(tasks)).ToList();
}
```

**Restricciones:** Implementa los tres patrones y mide la latencia total
vs fetching secuencial para 10 URLs con latencia variable (50-500ms).

**Pista:** `Task.WhenAll` con excepciones tiene un comportamiento especial:
si múltiples tasks fallan, `WhenAll` espera que todas completen (con error o éxito)
antes de lanzar la excepción. La excepción lanzada es solo la primera.
Para ver todos los errores: `task.Exception.InnerExceptions` o usar el patrón
de envolver en try/catch individualmente.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.1.3 — async void: el anti-patrón

**Enunciado:** `async void` es el anti-patrón más peligroso de C# async:

```csharp
// INCORRECTO — async void:
private async void ProcesarAsync()
{
    await Task.Delay(100);
    throw new InvalidOperationException("error");
    // Esta excepción NO puede ser capturada por el llamador
    // Causa UnhandledException en el AppDomain → crash de la app
}

// El llamador no puede await ni capturar:
try
{
    ProcesarAsync();  // No puede ser awaited — retorna void
    // ← La excepción ya se perdió
}
catch (Exception) { }  // ← Nunca se ejecuta

// CORRECTO — async Task:
private async Task ProcesarAsync()
{
    await Task.Delay(100);
    throw new InvalidOperationException("error");
    // Esta excepción se captura correctamente al awaitar
}

try { await ProcesarAsync(); }
catch (InvalidOperationException e) { /* manejado correctamente */ }
```

**Restricciones:** Demuestra el crash con `async void` y verifica que
`async Task` propaga la excepción correctamente.
La única excepción válida para `async void`: event handlers (Button.Click, etc.)
porque los event handlers tienen firma `void`.

**Pista:** `async void` existe porque los event handlers en .NET tienen firma
`EventHandler` que retorna `void`. Para soportar `async` en event handlers,
C# necesitaba `async void`. Pero fuera de event handlers, es siempre incorrecto.
El linter de C# (Roslyn analyzers) puede detectar `async void` fuera de event handlers.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.1.4 — Task continuations: ContinueWith y el equivalente moderno

**Enunciado:** `ContinueWith` es la API antigua de composición de Tasks.
`await` es el equivalente moderno pero más legible:

```csharp
// Antiguo (pre-C# 5):
Task<string> tarea = FetchAsync(url)
    .ContinueWith(t => {
        if (t.IsFaulted) return "error";
        return t.Result.ToUpper();
    }, TaskContinuationOptions.OnlyOnRanToCompletion);

// Moderno (C# 5+):
async Task<string> Procesar()
{
    try
    {
        string datos = await FetchAsync(url);
        return datos.ToUpper();
    }
    catch { return "error"; }
}
```

**Restricciones:** Reimplementa el Circuit Breaker del Cap.03 §3.6 usando
`ContinueWith` (para entender el API antiguo) y luego con `await` (versión moderna).
La versión con `await` debe ser notablemente más legible.

**Pista:** `ContinueWith` tiene peligros: por defecto, las continuaciones corren
en el scheduler del contexto que las registró (puede ser el UI thread).
`ContinueWith(..., TaskScheduler.Default)` fuerza el pool.
Con `await`, el comportamiento está determinado por `ConfigureAwait`.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.1.5 — Parallel.ForEachAsync: .NET 6+

**Enunciado:** `.NET 6` introdujo `Parallel.ForEachAsync` para paralelismo
de operaciones async con control de concurrencia:

```csharp
// Procesar 10,000 items con máximo 8 operaciones simultáneas:
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 8 },
    async (item, cancellationToken) =>
    {
        await ProcesarItemAsync(item, cancellationToken);
    }
);
```

**Restricciones:** Implementa el sistema de procesamiento de imágenes del
Ejercicio 10.7.1 con `Parallel.ForEachAsync`. Compara con:
- `Task.WhenAll` (sin límite de concurrencia)
- `SemaphoreSlim` manual (límite manual)
- `Parallel.ForEachAsync` (límite integrado)

**Pista:** `Task.WhenAll(items.Select(ProcesarItemAsync))` lanza TODOS los tasks
simultáneamente — sin límite. Para 10,000 items, esto puede saturar la red o la BD.
`SemaphoreSlim` permite implementar el límite manualmente.
`Parallel.ForEachAsync` hace lo mismo con menos código.

**Implementar en:** C# (.NET 6+)

---

## Sección 16.2 — Task Parallel Library: Paralelismo de Datos

El TPL de .NET tiene el API de paralelismo de datos más ergonómico de los
cuatro lenguajes — más completo que Rayon de Rust, más idiomático que los
streams paralelos de Java, más potente que multiprocessing de Python.

---

### Ejercicio 16.2.1 — Parallel.For y Parallel.ForEach

**Enunciado:** `Parallel.For` y `Parallel.ForEach` particionan el trabajo
automáticamente entre cores:

```csharp
// Parallel.For: índices enteros
var resultados = new double[N];
Parallel.For(0, N, i =>
{
    resultados[i] = Math.Sqrt(i);
});

// Parallel.ForEach: iterable arbitrario
var resultados2 = new ConcurrentBag<double>();
Parallel.ForEach(datos, item =>
{
    resultados2.Add(Procesar(item));
});

// Con opciones:
Parallel.For(0, N,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    i => resultados[i] = Math.Sqrt(i)
);

// Con estado local (para reducción sin contención):
double sumaTotal = 0;
Parallel.For(0, N,
    () => 0.0,              // inicializar estado local del thread
    (i, state, sumaLocal) => sumaLocal + datos[i],  // acumular localmente
    sumaLocal => Interlocked.Add(ref sumaTotal, (long)sumaLocal)  // combinar
);
```

**Restricciones:** Implementa la reducción paralela del Cap.08 §8.3 con el
patrón de estado local. Mide el speedup para 1M, 10M, 100M elementos.

**Pista:** El patrón de estado local en `Parallel.For` es idéntico al `fold+reduce`
de Rayon: cada thread acumula en su variable local, y al final combinan.
Esto evita la contención en la variable de resultado compartida.
Sin estado local, necesitarías `Interlocked.Add` en cada iteración — mucho más lento.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.2.2 — PLINQ: LINQ paralelo

**Enunciado:** PLINQ (Parallel LINQ) paraliza consultas LINQ con `.AsParallel()`:

```csharp
// LINQ secuencial:
var resultado = datos
    .Where(x => x > 0)
    .Select(x => x * x)
    .Sum();

// PLINQ — paralelo:
var resultado = datos
    .AsParallel()
    .Where(x => x > 0)
    .Select(x => x * x)
    .Sum();

// Con control de paralelismo:
var resultado = datos
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(x => x > 0)
    .Select(x => Calcular(x))  // operación costosa
    .ToList();

// Preservar el orden (más costoso):
var resultado = datos
    .AsParallel()
    .AsOrdered()  // fuerza el orden — overhead adicional
    .Select(x => x * 2)
    .ToList();
```

**Restricciones:** Implementa el word counter del Ejercicio 14.2.2 con PLINQ.
Mide el speedup para archivos de 10MB, 100MB, y 1GB.
Identificar cuándo PLINQ supera al LINQ secuencial.

**Pista:** PLINQ no siempre es más rápido. Para operaciones baratas (como `x * 2`),
el overhead de particionamiento y merge supera el beneficio.
PLINQ brilla cuando cada elemento requiere trabajo no-trivial (> ~1µs por elemento).
El `WithExecutionMode(ForceParallelism)` fuerza el paralelismo incluso cuando
PLINQ decide que sería más lento — útil para benchmarking pero no para producción.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.2.3 — Dataflow: pipelines con TPL Dataflow

**Enunciado:** `TPL Dataflow` (System.Threading.Tasks.Dataflow) permite construir
pipelines de procesamiento con backpressure integrado:

```csharp
// Crear bloques del pipeline:
var descargar = new TransformBlock<string, byte[]>(async url =>
    await httpClient.GetByteArrayAsync(url),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 8 }
);

var procesar = new TransformBlock<byte[], Resultado>(bytes =>
    ProcesarImagen(bytes),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4 }
);

var guardar = new ActionBlock<Resultado>(resultado =>
    GuardarEnDB(resultado),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 2 }
);

// Conectar los bloques (con backpressure automático):
descargar.LinkTo(procesar, new DataflowLinkOptions { PropagateCompletion = true });
procesar.LinkTo(guardar, new DataflowLinkOptions { PropagateCompletion = true });

// Enviar items al pipeline:
foreach (var url in urls)
    await descargar.SendAsync(url);

descargar.Complete();
await guardar.Completion;
```

**Restricciones:** Implementa el pipeline del Ejercicio 10.7.3 con TPL Dataflow.
Cada bloque debe tener un `MaxDegreeOfParallelism` diferente basándose en su tipo
(I/O-bound vs CPU-bound). El backpressure debe evitar que la memoria se llene.

**Pista:** `BoundedCapacity` en `ExecutionDataflowBlockOptions` limita el buffer
de cada bloque. Si el bloque siguiente no puede consumir más, `SendAsync` espera.
Este es el backpressure integrado del dataflow.
La diferencia con los pipelines manuales del Cap.03: TPL Dataflow maneja
automáticamente el threading, buffering, y completamiento de cada bloque.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.2.4 — ConcurrentDictionary y colecciones thread-safe

**Enunciado:** .NET tiene un set completo de colecciones thread-safe:

```csharp
// ConcurrentDictionary: el equivalente de ConcurrentHashMap de Java
var mapa = new ConcurrentDictionary<string, int>();

// Operaciones atómicas:
mapa.AddOrUpdate("key",
    addValueFactory: k => 1,          // si no existe
    updateValueFactory: (k, v) => v + 1  // si existe
);

// GetOrAdd: computar solo si no existe
var conexion = pool.GetOrAdd(serverId, id => new Conexion(id));

// ConcurrentQueue, ConcurrentStack, ConcurrentBag:
var queue = new ConcurrentQueue<Task>();
queue.Enqueue(nuevaTarea);
if (queue.TryDequeue(out var tarea)) procesarTarea(tarea);

// BlockingCollection: con backpressure
var buffer = new BlockingCollection<Item>(boundedCapacity: 100);
// Productor:
buffer.Add(item);  // bloquea si el buffer está lleno
// Consumidor:
foreach (var item in buffer.GetConsumingEnumerable()) { /* ... */ }
```

**Restricciones:** Implementa el HashMap concurrente del Cap.12 §12.3
usando `ConcurrentDictionary` y mide si supera al striped approach manual.
`ConcurrentDictionary` en .NET usa lock striping internamente — ¿cuántos stripes?

**Pista:** `ConcurrentDictionary` en .NET usa por defecto `4 × procesadores` stripes.
Para la mayoría de aplicaciones es suficiente. `ConcurrentDictionary` tiene
`GetOrAdd(key, factory)` que garantiza que la factory se llama at-most-once por key
bajo concurrencia — la misma garantía que `computeIfAbsent` de Java.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.2.5 — ImmutableCollections: seguridad sin locks

**Enunciado:** `System.Collections.Immutable` provee colecciones inmutables
que son inherentemente thread-safe:

```csharp
using System.Collections.Immutable;

// ImmutableList — thread-safe sin locks
ImmutableList<int> lista = ImmutableList<int>.Empty;
lista = lista.Add(1).Add(2).Add(3);  // crea nuevas versiones, no modifica

// Compartir entre threads sin sincronización:
var listaCompartida = ImmutableList<string>.Empty;
// Thread 1:
listaCompartida = listaCompartida.Add("a");  // crea nueva lista, no modifica la original
// Thread 2: ve la lista original o la nueva — ambas son válidas y consistentes

// ImmutableDictionary para config:
var config = ImmutableDictionary<string, string>.Empty
    .Add("host", "localhost")
    .Add("port", "5432");

// Actualizar atómicamente:
Interlocked.Exchange(ref configActual, config.SetItem("port", "5433"));
```

**Restricciones:** Implementa el sistema de configuración del Cap.13 §13.6.4
(RCU pattern) en C# usando `ImmutableDictionary` + `Interlocked.Exchange`.
Los lectores nunca deben bloquearse. El escritor hace swap atómico del diccionario.

**Pista:** `ImmutableDictionary.SetItem(key, value)` retorna un nuevo diccionario
con el key actualizado — la versión original no se modifica. Los threads que
tienen una referencia a la versión anterior siguen viendo sus datos consistentes.
Este es exactamente el patrón RCU del Cap.13 §13.6.4, implementado con
inmutabilidad estructural en lugar de atomics manuales.

**Implementar en:** C# (.NET 8)

---

## Sección 16.3 — System.Threading.Channels: El Canal de .NET

`System.Threading.Channels` (introducido en .NET Core 3.0) es la implementación
de Go channels para .NET — con tipos, backpressure, y cancellación integradas.

```csharp
// Crear un canal:
var canal = Channel.CreateBounded<int>(capacity: 100);
// o:
var canal = Channel.CreateUnbounded<int>();

// Escritor (productor):
ChannelWriter<int> writer = canal.Writer;
await writer.WriteAsync(42);  // espera si el canal está lleno (bounded)
bool exito = writer.TryWrite(42);  // no bloquea — false si lleno

// Lector (consumidor):
ChannelReader<int> reader = canal.Reader;
int valor = await reader.ReadAsync();  // espera si el canal está vacío
bool haValor = reader.TryRead(out var val);  // no bloquea

// Iterar el canal (como `for msg := range ch` en Go):
await foreach (var item in reader.ReadAllAsync())
{
    await ProcesarAsync(item);
}

// Cerrar el canal:
writer.Complete();  // señalizar fin — como `close(ch)` en Go
```

---

### Ejercicio 16.3.1 — Channels básicos: productor-consumidor

**Enunciado:** Reimplementa el Worker Pool del Cap.03 con `System.Threading.Channels`:

```csharp
public class WorkerPool<TInput, TOutput>
{
    private readonly Channel<TInput> _tareas;
    private readonly Channel<TOutput> _resultados;
    private readonly List<Task> _workers;

    public WorkerPool(int numWorkers, Func<TInput, Task<TOutput>> procesar)
    {
        _tareas = Channel.CreateBounded<TInput>(capacity: numWorkers * 2);
        _resultados = Channel.CreateUnbounded<TOutput>();

        _workers = Enumerable.Range(0, numWorkers)
            .Select(_ => Task.Run(async () =>
            {
                await foreach (var tarea in _tareas.Reader.ReadAllAsync())
                {
                    var resultado = await procesar(tarea);
                    await _resultados.Writer.WriteAsync(resultado);
                }
            }))
            .ToList();
    }

    public async Task SubmitAsync(TInput tarea) => await _tareas.Writer.WriteAsync(tarea);

    public async Task<List<TOutput>> CompletarAsync()
    {
        _tareas.Writer.Complete();
        await Task.WhenAll(_workers);
        _resultados.Writer.Complete();
        return await _resultados.Reader.ReadAllAsync().ToListAsync();
    }
}
```

**Restricciones:** El canal bounded proporciona backpressure — el productor
espera si el pool está saturado. Verificar que todos los items son procesados
exactamente una vez. Mide el throughput con 1, 4, 8 workers.

**Pista:** `ReadAllAsync()` itera el canal hasta que se llama `Complete()` en el escritor.
Es el equivalente perfecto de `for msg := range ch` en Go. La diferencia:
en Go el canal se cierra con `close(ch)`; en .NET se llama `writer.Complete()`.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.3.2 — Channel con múltiples consumidores (fan-out)

**Enunciado:** Múltiples consumidores pueden leer del mismo canal en .NET
(a diferencia de Go donde no hay garantía de cuál goroutine recibe):

```csharp
var canal = Channel.CreateBounded<Tarea>(capacity: 100);

// N consumidores leyendo del mismo canal:
var consumidores = Enumerable.Range(0, N)
    .Select(id => Task.Run(async () =>
    {
        await foreach (var tarea in canal.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumidor {id} procesando: {tarea}");
            await ProcesarAsync(tarea);
        }
    }));

// Distribuir trabajo:
for (int i = 0; i < 1000; i++)
    await canal.Writer.WriteAsync(new Tarea(i));
canal.Writer.Complete();

await Task.WhenAll(consumidores);
```

**Restricciones:** Implementar el fan-out del Cap.05 §5.4 con Channels.
Verificar que cada item es procesado exactamente una vez (no N veces).
Medir la distribución de trabajo entre consumidores (debería ser ~1/N cada uno).

**Pista:** En .NET, un `ChannelReader<T>` con múltiples consumers es safe —
cada item es recibido por exactamente un consumidor. El runtime coordina el acceso
al buffer del canal con CAS, similar al MPMC ring buffer del Cap.12 §12.5.2.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.3.3 — Channel pipeline con transformaciones

**Enunciado:** Encadena channels como un pipeline donde cada etapa lee
del canal anterior y escribe en el siguiente:

```csharp
static ChannelReader<TOutput> Transformar<TInput, TOutput>(
    ChannelReader<TInput> fuente,
    Func<TInput, Task<TOutput>> transformar,
    int maxConcurrencia = 1)
{
    var destino = Channel.CreateUnbounded<TOutput>();

    Task.Run(async () =>
    {
        var semaforo = new SemaphoreSlim(maxConcurrencia);
        var tareasPendientes = new List<Task>();

        await foreach (var item in fuente.ReadAllAsync())
        {
            await semaforo.WaitAsync();
            tareasPendientes.Add(Task.Run(async () =>
            {
                try { await destino.Writer.WriteAsync(await transformar(item)); }
                finally { semaforo.Release(); }
            }));
        }

        await Task.WhenAll(tareasPendientes);
        destino.Writer.Complete();
    });

    return destino.Reader;
}

// Uso:
var pipeline = Transformar(fuente.Reader, FiltrarAsync)
    |> Transformar(TransformarAsync, maxConcurrencia: 4)
    |> Transformar(GuardarAsync, maxConcurrencia: 2);
```

**Restricciones:** Implementar el pipeline del Cap.03 §3.2 con channels encadenados.
El `maxConcurrencia` de cada etapa permite ajustar el throughput.
Verificar que la cancelación se propaga de una etapa a la siguiente.

**Pista:** Este patrón es el equivalente en C# del pipeline con canales de Go.
La diferencia: en Go, los goroutines del pipeline son explícitos y cada etapa
es una goroutine; en C#, los Tasks del pipeline son implícitos en la función
`Transformar`. La abstracción de canal hace que la composición sea más limpia.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.3.4 — Channels vs BlockingCollection vs ConcurrentQueue

**Enunciado:** .NET tiene tres colecciones para producer-consumer con distintos tradeoffs:

```
ConcurrentQueue<T>:
  - Sin backpressure (crece ilimitado)
  - API de bajo nivel (Enqueue, TryDequeue)
  - Más rápida para throughput puro
  - No tiene soporte async nativo

BlockingCollection<T>:
  - Con backpressure (BoundedCapacity)
  - API bloqueante (Add bloquea el thread si lleno)
  - Ideal para threading (no async)
  - Tiene GetConsumingEnumerable() para iteración

Channel<T>:
  - Con o sin backpressure
  - API async nativa (WriteAsync, ReadAsync)
  - Ideal para async/await
  - Más moderna y ergonómica
```

**Restricciones:** Para el pipeline de procesamiento, implementa con las tres
y mide throughput, latencia, y uso de CPU para 1M items.

**Pista:** `Channel<T>` es la elección correcta para código async nuevo.
`BlockingCollection<T>` sigue siendo útil para código sincrónico con threads.
`ConcurrentQueue<T>` es la más rápida pero requiere un mecanismo externo
para el bloqueo/notificación (usualmente `SemaphoreSlim` o `ManualResetEventSlim`).

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.3.5 — Comparar: Channel de .NET vs chan de Go

**Enunciado:** Implementa el mismo sistema (pipeline con 4 etapas y backpressure)
en C# con Channels y en Go con canales. Compara:

```csharp
// C#:
var etapa1 = Channel.CreateBounded<Item>(100);
var etapa2 = Channel.CreateBounded<Procesado>(100);
// ... etc.
```

```go
// Go:
etapa1 := make(chan Item, 100)
etapa2 := make(chan Procesado, 100)
// ... etc.
```

**Restricciones:** Mide throughput, latencia p99, y overhead de creación del canal.
Compara la ergonomía del código (líneas, legibilidad, manejo de errores).

**Pista:** Los canales de Go tienen overhead de ~400-600 ns por operación.
`System.Threading.Channels` tiene overhead de ~100-200 ns.
La diferencia viene del scheduler de goroutines vs async/await del CLR.
En términos de API, los canales de Go son más simples (no hay distinción
entre `ChannelReader` y `ChannelWriter`); los de .NET son más ergonómicos
para async/await.

**Implementar en:** C# + Go (para la comparación)

---

## Sección 16.4 — ValueTask y Zero-Allocation async

`ValueTask<T>` es la optimización de .NET para métodos async que
completan sincrónicamente en el caso común (como caché hits):

```csharp
// Task<T>: siempre aloca un objeto Task en el heap
public async Task<int> ObtenerCacheAsync(string key)
{
    if (_cache.TryGetValue(key, out var valor))
        return valor;  // ← aloca un Task aunque complete sincrónicamente
    return await FetchAsync(key);
}

// ValueTask<T>: zero-allocation cuando completa sincrónicamente
public ValueTask<int> ObtenerCacheConValueTaskAsync(string key)
{
    if (_cache.TryGetValue(key, out var valor))
        return new ValueTask<int>(valor);  // ← NO aloca (struct en el stack)
    return new ValueTask<int>(FetchAsync(key));  // ← aloca solo si necesita I/O
}
```

---

### Ejercicio 16.4.1 — Medir la diferencia de allocations: Task vs ValueTask

**Enunciado:** Usa BenchmarkDotNet para medir las allocations:

```csharp
[MemoryDiagnoser]
public class BenchmarkTask
{
    private readonly Cache _cache = new Cache();

    [Benchmark]
    public async Task<int> ConTask()
    {
        return await _cache.GetWithTaskAsync("key");
    }

    [Benchmark]
    public async Task<int> ConValueTask()
    {
        return await _cache.GetWithValueTaskAsync("key");
    }
}
```

**Restricciones:** Para un método que completa sincrónicamente el 99% del tiempo,
`ValueTask` debe tener 0 allocations. Para el 1% asíncrono, debe tener las mismas
allocations que `Task`. Verificar con BenchmarkDotNet.

**Pista:** BenchmarkDotNet reporta `Gen 0` collections y allocations por operación.
Para `Task` que completa sincrónicamente: 1 allocation (el objeto Task en el heap).
Para `ValueTask` que completa sincrónicamente: 0 allocations.
Esta diferencia importa en hot paths con millones de calls por segundo.

**Implementar en:** C# (.NET 8, con BenchmarkDotNet)

---

### Ejercicio 16.4.2 — IValueTaskSource: reutilizar ValueTask sin allocations

**Enunciado:** Para máxima eficiencia, `IValueTaskSource` permite reutilizar
el mismo objeto para múltiples ValueTasks:

```csharp
// La mayoría de los casos: usar ValueTask<T> directamente
public ValueTask<int> LeerAsync()
{
    if (_buffer.Count > 0)
        return new ValueTask<int>(_buffer.Dequeue());  // síncrono — sin alloc
    return new ValueTask<int>(this, _token);  // IValueTaskSource — sin alloc
}

// IValueTaskSource implementado en la clase:
class MiCanal : IValueTaskSource<int>
{
    private ManualResetValueTaskSourceCore<int> _core;

    int IValueTaskSource<int>.GetResult(short token) => _core.GetResult(token);
    ValueTaskSourceStatus IValueTaskSource<int>.GetStatus(short token) => _core.GetStatus(token);
    void IValueTaskSource<int>.OnCompleted(Action<object?> continuation, object? state, short token, ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);

    public void SetResult(int result) => _core.SetResult(result);
}
```

**Restricciones:** Implementa un canal de I/O de alto rendimiento con `IValueTaskSource`.
Mide con BenchmarkDotNet que no hay allocations en el hot path.

**Pista:** `IValueTaskSource` es la herramienta de optimización más avanzada de .NET async.
Se usa internamente en `Socket`, `Pipe`, y `Channel<T>` del runtime.
Para código de aplicación, rara vez es necesario.
Solo considerar si el profiler muestra que las allocations de Task son
el cuello de botella — lo cual requiere millones de operations/segundo.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.4.3 — PooledValueTask con ObjectPool

**Enunciado:** Otra estrategia para reducir allocations: poolear los objetos Task:

```csharp
// Microsoft.Extensions.ObjectPool
var pool = new DefaultObjectPool<TaskCompletionSource<int>>(
    new DefaultPooledObjectPolicy<TaskCompletionSource<int>>()
);

public async ValueTask<int> OperacionConPoolAsync()
{
    var tcs = pool.Get();  // obtener del pool en lugar de new
    try
    {
        // usar tcs.Task como una Task normal
        return await tcs.Task;
    }
    finally
    {
        tcs = new TaskCompletionSource<int>();  // reset
        pool.Return(tcs);  // devolver al pool
    }
}
```

**Restricciones:** Implementa el server del Ejercicio 16.1.5 con object pooling
para los `TaskCompletionSource`. Mide la reducción en presión sobre el GC.

**Pista:** El object pooling reduce las allocations pero introduce complejidad.
La regla: usar pooling solo cuando el profiler muestra que el GC es el cuello
de botella. En la mayoría de servicios web, el GC del .NET moderno (generational,
concurrent) es suficientemente eficiente.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.4.4 — Benchmark holístico: Task vs ValueTask vs raw Thread

**Enunciado:** Compara el rendimiento de tres estilos de concurrencia
para el mismo problema (servidor de caché):

```csharp
// Estilo 1: async/await con Task
public async Task<string> ManejadorTask(HttpContext ctx)

// Estilo 2: async/await con ValueTask
public async ValueTask<string> ManejadorValueTask(HttpContext ctx)

// Estilo 3: síncrono con threads (como Java clásico)
public string ManejadorSincrono(HttpContext ctx)  // requiere más threads
```

**Restricciones:** Mide throughput, latencia p99, y memoria para 100K req/s.
El overhead de `ValueTask` vs `Task` debe ser medible en microbenchmarks
pero marginal en el sistema completo.

**Pista:** La diferencia entre `Task` y `ValueTask` importa en hot paths con
>1M ops/s. Para un servidor web con <100K req/s, la diferencia en latencia
total (incluyendo parseo HTTP, serialización, etc.) es < 1%.
Para el runtime de .NET, la diferencia entre async y síncrono con threads
depende del workload: I/O-bound → async gana; CPU-bound → similar.

**Implementar en:** C# (.NET 8, con BenchmarkDotNet y profiling)

---

### Ejercicio 16.4.5 — Span<T> y Memory<T>: cero copias en el procesamiento

**Enunciado:** `Span<T>` y `Memory<T>` permiten operar sobre subarrays
sin copiar datos — fundamental para parsers y procesadores de alta performance:

```csharp
// Sin Span: múltiples copias
string linea = "nombre,25,Madrid";
var partes = linea.Split(',');     // ← aloca un array y substrings
string nombre = partes[0];         // ← aloca un string
int edad = int.Parse(partes[1]);   // ← aloca un string

// Con Span: cero copias
ReadOnlySpan<char> linea = "nombre,25,Madrid";
int coma1 = linea.IndexOf(',');
var nombre = linea[..coma1];                    // ← Span — sin alloc
var resto = linea[(coma1 + 1)..];
int coma2 = resto.IndexOf(',');
int edad = int.Parse(resto[..coma2]);           // ← Span — sin alloc
```

**Restricciones:** Implementa un parser de CSV que procesa un archivo de 1GB
sin alocar strings por cada campo. Mide el uso de memoria comparando
con el parser basado en `string.Split`.

**Pista:** `Span<T>` solo existe en el stack (no puede escapar al heap, no puede
ser usado en async methods directamente). Para async, usar `Memory<T>` que
sí puede vivir en el heap pero mantiene la semántica de "slice" sin copia.
El parser de CSV sin allocations puede ser 3-5x más rápido que el que usa
`Split` por la reducción de presión sobre el GC.

**Implementar en:** C# (.NET 8)

---

## Sección 16.5 — CancellationToken: Cancelación en .NET

`CancellationToken` es el mecanismo de cancelación más sofisticado de los
cuatro lenguajes — integrado profundamente con async/await, Tasks, y la biblioteca estándar.

```csharp
// El patrón estándar:
public async Task ProcesarAsync(CancellationToken cancellationToken = default)
{
    // Verificar cancelación en el inicio:
    cancellationToken.ThrowIfCancellationRequested();

    // Pasar el token a operaciones await:
    await Task.Delay(1000, cancellationToken);  // cancela el delay si se cancela

    // Pasar a operaciones de la librería:
    await httpClient.GetAsync(url, cancellationToken);

    // Verificar en loops CPU-bound:
    for (int i = 0; i < N; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        Procesar(i);
    }
}

// Crear y cancelar:
var cts = new CancellationTokenSource();
var tarea = ProcesarAsync(cts.Token);

// Cancelar después de 5 segundos:
cts.CancelAfter(TimeSpan.FromSeconds(5));

// O cancelar manualmente:
cts.Cancel();

try { await tarea; }
catch (OperationCanceledException) { Console.WriteLine("cancelado"); }
```

---

### Ejercicio 16.5.1 — CancellationToken propagado por todo el sistema

**Enunciado:** Implementa el Circuit Breaker del Cap.03 §3.6 con `CancellationToken`
propagado correctamente a través de todas las capas:

```csharp
// Controlador HTTP:
[HttpGet("/datos/{id}")]
public async Task<IActionResult> ObtenerDatos(string id, CancellationToken ct)
{
    // ct se cancela automáticamente si el cliente desconecta
    var resultado = await _servicio.ObtenerAsync(id, ct);
    return Ok(resultado);
}

// Capa de servicio:
public async Task<Dato> ObtenerAsync(string id, CancellationToken ct)
{
    ct.ThrowIfCancellationRequested();
    return await _repositorio.BuscarAsync(id, ct);
}

// Capa de repositorio:
public async Task<Dato> BuscarAsync(string id, CancellationToken ct)
{
    return await _db.QueryFirstOrDefaultAsync<Dato>(
        "SELECT * FROM datos WHERE id = @id", new { id }, cancellationToken: ct
    );
}
```

**Restricciones:** Si el cliente HTTP desconecta, todas las operaciones pendientes
deben cancelarse en cadena. Verificar que no quedan threads/tasks "huérfanos"
después de la cancelación.

**Pista:** ASP.NET Core inyecta automáticamente el `CancellationToken` del request
HTTP en los parámetros de los controllers. Cuando el cliente desconecta,
el token se cancela — y si se propaga correctamente, toda la cadena de await
se cancela. Sin propagación correcta, el servidor desperdicia recursos en
requests cuyos clientes ya se fueron.

**Implementar en:** C# (.NET 8, con ASP.NET Core)

---

### Ejercicio 16.5.2 — LinkedCancellationToken: combinar múltiples tokens

**Enunciado:** A veces se necesita cancelar por múltiples razones:
timeout + cancelación del usuario + shutdown del servidor:

```csharp
public async Task ProcesarRequestAsync(
    Request request,
    CancellationToken userCancellationToken)
{
    // Combinar: timeout + usuario + shutdown
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        userCancellationToken,
        timeoutCts.Token,
        _shutdownToken  // del ApplicationLifetime
    );

    try
    {
        await ProcesarAsync(request, linkedCts.Token);
    }
    catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
    {
        throw new TimeoutException("operación expiró después de 30s");
    }
    catch (OperationCanceledException) when (userCancellationToken.IsCancellationRequested)
    {
        // Cliente desconectó — silenciar
    }
}
```

**Restricciones:** La cláusula `when` del catch permite distinguir la razón
de la cancelación. Implementar un servidor que maneja correctamente los tres
tipos de cancelación con comportamiento diferente para cada uno.

**Pista:** `CancellationTokenSource.CreateLinkedTokenSource` crea un token que
se cancela cuando CUALQUIERA de los tokens fuente se cancela.
El `when` en `catch (OperationCanceledException) when (...)` evalúa la condición
antes de entrar al catch — sin overhead si la condición es false.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.5.3 — CancellationToken.Register: acciones al cancelar

**Enunciado:** `CancellationToken.Register` ejecuta una acción cuando se cancela:

```csharp
public async Task ManejadorConLimpieza(CancellationToken ct)
{
    var recurso = AbrirRecurso();

    // Registrar limpieza al cancelar:
    using var registro = ct.Register(() =>
    {
        recurso.CerrarForzadamente();
        _metricas.IncrementarCancelaciones();
    });

    try
    {
        await UsarRecurso(recurso, ct);
    }
    catch (OperationCanceledException)
    {
        // El Register ya cerró el recurso
        throw;
    }
    finally
    {
        recurso.Dispose();
        // registro.Dispose() llama al Dispose del CancellationTokenRegistration
        // para evitar que se ejecute si no fue cancelado
    }
}
```

**Restricciones:** Implementar el Worker Pool con limpieza ordenada al recibir
señal de shutdown. Los workers activos deben completar su tarea actual,
los workers esperando deben detenerse inmediatamente.

**Pista:** `Register` es útil para integrar código que no soporta `CancellationToken`
nativo. Por ejemplo, `WebSocket.CloseAsync()` no recibe CT — puedes usar
`Register` para cerrarlo cuando se cancele. El `Dispose` del registro
evita memory leaks cuando el token nunca se cancela.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.5.4 — Graceful shutdown con IHostedService

**Enunciado:** ASP.NET Core tiene un mecanismo de shutdown ordenado integrado:

```csharp
public class WorkerBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // stoppingToken se cancela cuando el host recibe SIGTERM/Ctrl+C
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var tarea = await _cola.DequeueAsync(stoppingToken);
                await ProcesarAsync(tarea, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;  // shutdown ordenado
            }
        }
        
        // Limpiar recursos antes de salir
        await LimpiarAsync();
    }
}
```

**Restricciones:** Implementa el servicio de background del Ejercicio 14.7.3
(cron distribuido) como un `BackgroundService`. El shutdown debe ser ordenado:
completar la tarea actual, no iniciar nuevas, limpiar recursos.

**Pista:** `BackgroundService` ya implementa la lógica de `stoppingToken`.
Al recibir SIGTERM, ASP.NET Core cancela el `stoppingToken` y espera hasta
`ShutdownTimeout` (default: 30s) para que el servicio termine limpiamente.
Si tarda más, fuerza la terminación. El timeout es configurable en `appsettings.json`.

**Implementar en:** C# (.NET 8, con ASP.NET Core)

---

### Ejercicio 16.5.5 — CancellationToken vs Go context: filosofías distintas

**Enunciado:** Compara la cancelación en C# y Go para el mismo escenario
(cadena de llamadas con timeout + cancelación de usuario):

```go
// Go: context se pasa explícitamente como primer parámetro
func Procesar(ctx context.Context, id string) (Dato, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    return repositorio.Buscar(ctx, id)
}
```

```csharp
// C#: CancellationToken se pasa como último parámetro (convención)
public async Task<Dato> ProcesarAsync(string id, CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(
        ct, new CancellationTokenSource(TimeSpan.FromSeconds(30)).Token
    );
    return await _repositorio.BuscarAsync(id, cts.Token);
}
```

**Restricciones:** Implementa la misma lógica en ambos lenguajes y compara:
API ergonomía, composición de cancelaciones, deadlines, y propagación automática.

**Pista:** `context.Context` en Go combina cancelación + deadline + values (metadata).
`CancellationToken` en C# solo maneja cancelación; `DateTime` o `TimeSpan` se pasan separados.
La filosofía de Go (un objeto "contexto" que lo contiene todo) vs C#
(separar cancelación de otros concerns) refleja filosofías de diseño distintas.
Ninguna es objetivamente mejor — depende de las preferencias del equipo.

**Implementar en:** C# + Go

---

## Sección 16.6 — IAsyncEnumerable: Streams Asíncronos

`IAsyncEnumerable<T>` (C# 8 / .NET Core 3.0) es el equivalente asíncrono
de `IEnumerable<T>` — permite iterar datos que llegan asincrónicamente,
como rows de una base de datos o eventos de un stream:

```csharp
// Definir un stream asíncrono:
public async IAsyncEnumerable<Usuario> ObtenerUsuariosAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var row in _db.QueryAsync("SELECT * FROM usuarios", ct))
    {
        yield return new Usuario(row);
    }
}

// Consumir:
await foreach (var usuario in ObtenerUsuariosAsync(cancellationToken))
{
    await ProcesarUsuarioAsync(usuario);
}

// Con operadores LINQ async (System.Linq.Async):
await ObtenerUsuariosAsync()
    .Where(u => u.Activo)
    .Select(u => u.Email)
    .ForEachAsync(email => Console.WriteLine(email));
```

---

### Ejercicio 16.6.1 — IAsyncEnumerable vs List: streaming sin cargar todo en memoria

**Enunciado:** Procesa un stream de 10M registros de una BD sin cargar todo en memoria:

```csharp
// MAL — carga todo en memoria:
var todos = await _db.QueryAsync<Registro>("SELECT * FROM registros").ToListAsync();
foreach (var r in todos) Procesar(r);

// BIEN — streaming:
await foreach (var r in _db.QueryAsync<Registro>("SELECT * FROM registros"))
{
    await ProcesarAsync(r);
    // Solo un registro en memoria a la vez (más el buffer del driver de BD)
}
```

**Restricciones:** Mide el uso de memoria pico con `List` (todo en memoria) vs
`IAsyncEnumerable` (streaming). Para 10M registros de 100 bytes, la diferencia
debe ser > 100x en uso de memoria.

**Pista:** `IAsyncEnumerable` es lazy — no evalúa el siguiente elemento hasta que
se pide. La BD envía los rows en batches (configurable en el connection string).
Para EF Core, usar `AsAsyncEnumerable()` para streaming.
Para Dapper, `QueryUnbufferedAsync` devuelve `IAsyncEnumerable<T>`.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.6.2 — Channel<T> como IAsyncEnumerable

**Enunciado:** `Channel<T>.Reader` implementa `IAsyncEnumerable<T>` vía
`ReadAllAsync()`. Esto permite componer streams con LINQ async:

```csharp
var canal = Channel.CreateBounded<Evento>(capacity: 100);

// El reader como stream:
var stream = canal.Reader.ReadAllAsync();

// Procesar con LINQ async:
await foreach (var evento in stream.Where(e => e.Tipo == "error").Take(100))
{
    await LogarErrorAsync(evento);
}

// Pipeline de transformación:
IAsyncEnumerable<string> pipeline = canal.Reader
    .ReadAllAsync()
    .Select(e => e.Serializar())
    .Where(s => s.Length < 1024);
```

**Restricciones:** Implementa el sistema de events del Ejercicio 14.6.3
(Reactive Streams) usando `Channel<T>` + `IAsyncEnumerable` en lugar de Reactor.
Compara la ergonomía y rendimiento.

**Pista:** `System.Linq.Async` provee operadores LINQ para `IAsyncEnumerable`:
`Select`, `Where`, `Take`, `Skip`, `ToListAsync`, `ForEachAsync`, etc.
La diferencia con Reactive Streams (Rx): `IAsyncEnumerable` es pull-based
(el consumidor pide elementos); Rx es push-based (el productor empuja).
Para la mayoría de casos, pull-based es más simple de razonar.

**Implementar en:** C# (.NET 8, con System.Linq.Async)

---

### Ejercicio 16.6.3 — SignalR y streaming desde el servidor

**Enunciado:** SignalR (de ASP.NET Core) soporta server-to-client streaming
con `IAsyncEnumerable`:

```csharp
// Hub de SignalR:
public class StreamingHub : Hub
{
    public async IAsyncEnumerable<StockPrice> StreamPrecios(
        string simbolo,
        [EnumeratorCancellation] CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            yield return await _mercado.ObtenerPrecioAsync(simbolo);
            await Task.Delay(100, ct);  // actualizar cada 100ms
        }
    }
}

// Cliente JavaScript:
const connection = new signalR.HubConnectionBuilder()...
connection.stream("StreamPrecios", "AAPL")
    .subscribe({ next: precio => actualizarUI(precio) });
```

**Restricciones:** Implementa el sistema de time-series del Ejercicio 12.4.4
con streaming vía SignalR. El cliente debe ver actualizaciones en tiempo real.

**Pista:** El `[EnumeratorCancellation]` atributo conecta el `CancellationToken`
del caller con el iterator — cuando el cliente desconecta, SignalR cancela el token
y el stream para. Sin este atributo, el stream continuaría en el servidor
aunque el cliente se haya ido.

**Implementar en:** C# (.NET 8, con ASP.NET Core SignalR)

---

### Ejercicio 16.6.4 — Batching de IAsyncEnumerable

**Enunciado:** A veces es más eficiente procesar elementos en grupos:

```csharp
public static async IAsyncEnumerable<IReadOnlyList<T>> EnLotes<T>(
    this IAsyncEnumerable<T> fuente,
    int tamañoLote,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var lote = new List<T>(tamañoLote);
    await foreach (var item in fuente.WithCancellation(ct))
    {
        lote.Add(item);
        if (lote.Count >= tamañoLote)
        {
            yield return lote.AsReadOnly();
            lote = new List<T>(tamañoLote);
        }
    }
    if (lote.Count > 0)
        yield return lote.AsReadOnly();  // último lote parcial
}

// Uso:
await foreach (var lote in streamDeRegistros.EnLotes(1000))
{
    await _db.BulkInsertAsync(lote);  // inserción en lotes más eficiente
}
```

**Restricciones:** Implementar la extensión `EnLotes` y el procesamiento bulk
del Ejercicio 14.6.5 con el operador de batching.
El bulk insert debe ser > 10x más rápido que el insert de un registro a la vez.

**Pista:** El bulk insert usando `SqlBulkCopy` en SQL Server (o `COPY` en PostgreSQL)
puede insertar 10,000 filas en el mismo tiempo que 100 inserts individuales.
El batching con `IAsyncEnumerable` es la forma ergonómica de preparar los lotes
sin cargar todos los datos en memoria.

**Implementar en:** C# (.NET 8)

---

### Ejercicio 16.6.5 — IAsyncEnumerable vs Stream<T> de Reactor vs AsyncGenerator de Python

**Enunciado:** Los cuatro lenguajes tienen su versión de "stream asíncrono":

```python
# Python — async generator:
async def stream_usuarios():
    async with db.transaction():
        async for row in db.execute("SELECT * FROM usuarios"):
            yield Usuario(row)
```

```java
// Java — Flux de Reactor:
Flux<Usuario> streamUsuarios() {
    return Flux.fromIterable(db.queryForList("SELECT * FROM usuarios", Usuario.class));
}
```

```go
// Go — channel como stream:
func streamUsuarios(ctx context.Context) <-chan Usuario {
    ch := make(chan Usuario, 100)
    go func() {
        defer close(ch)
        rows, _ := db.QueryContext(ctx, "SELECT * FROM usuarios")
        for rows.Next() {
            var u Usuario
            rows.Scan(&u.ID, &u.Nombre)
            ch <- u
        }
    }()
    return ch
}
```

```csharp
// C# — IAsyncEnumerable:
async IAsyncEnumerable<Usuario> StreamUsuariosAsync([EnumeratorCancellation] CancellationToken ct = default) {
    await foreach (var row in db.QueryAsync("SELECT * FROM usuarios", ct))
        yield return new Usuario(row);
}
```

**Restricciones:** Implementa los cuatro y compara: ergonomía, backpressure,
cancelación, y composición con operadores (filter, map, take).

**Pista:** Python async generators son los más simples de escribir.
Go channels son los más simples de consumir pero no tienen operadores.
Reactor (Java) tiene los operadores más ricos pero la API más compleja.
C# `IAsyncEnumerable` tiene el balance entre simplicidad y riqueza de operadores.

**Implementar en:** Python + Java + Go + C#

---

## Sección 16.7 — Comparación Final: Los Cuatro Lenguajes

---

### Ejercicio 16.7.1 — El mismo servidor en cuatro lenguajes

**Enunciado:** Reimplementa el servidor de caché del Ejercicio 14.7.1
en C# y compara con Go, Java, y Rust:

```csharp
// C# con Channels y async/await:
var app = WebApplication.Create();

app.MapGet("/get/{key}", async (string key, ICacheService cache, CancellationToken ct) =>
    await cache.GetAsync(key, ct) is { } value
        ? Results.Ok(value)
        : Results.NotFound()
);

app.MapPost("/set/{key}", async (
    string key,
    [FromBody] string value,
    ICacheService cache,
    CancellationToken ct) =>
{
    await cache.SetAsync(key, value, ct);
    return Results.Ok();
});
```

**Restricciones:** Igual que el Ejercicio 14.7.1 — 10K req/s, 90% reads, 1% writes.
Añadir C# a la comparación de latencia p50/p99/p999, throughput, y memoria.

**Pista:** C# con ASP.NET Core Kestrel tiene un rendimiento similar a Go para
workloads de este tipo. La diferencia principal es el GC: C# tiene un GC
generacional con compactación, similar al de Java pero mejor ajustado para
baja latencia. El p999 de C# suele ser similar al de Go pero mejor que Java
(en configuración default).

**Implementar en:** C# (.NET 8) + comparación con los tres lenguajes anteriores

---

### Ejercicio 16.7.2 — Ecosistema: qué tiene cada lenguaje que los otros no

**Enunciado:** Cada lenguaje tiene características únicas en su ecosistema
de concurrencia. Identifica y demuestra una característica exclusiva de cada uno:

```
Go:    goroutines + scheduler M:N + select statement
       → millones de goroutines, comunicación por canales, CSP nativo

Rust:  ownership + Send/Sync en tiempo de compilación
       → races son imposibles de expresar, zero-cost futures

Java:  Virtual Threads (Project Loom) + StructuredTaskScope
       → 1M threads con semántica blocking, cancelación estructurada

Python: asyncio + ecosistema maduro para scripting/ML
        → numpy libera el GIL, integración con C extensions

C#:    ValueTask + Span<T> + Source Generators
       → zero-allocation async, procesamiento sin copias, generación de código
```

**Restricciones:** Para cada característica, implementa un ejemplo que demuestra
que el mismo comportamiento sería más difícil o imposible en los otros lenguajes.

**Pista:** Las características exclusivas reflejan las prioridades de diseño:
Go prioriza simplicidad y concurrencia comunicante;
Rust prioriza seguridad sin costo en runtime;
Java prioriza compatibilidad y evolución gradual;
Python prioriza la productividad del desarrollador;
C# prioriza la ergonomía y la performance del runtime.

**Implementar en:** Todos los lenguajes

---

### Ejercicio 16.7.3 — Performance shootout final

**Enunciado:** Un benchmark completo comparando los cuatro lenguajes
para cuatro tipos de workload:

```
Workload 1: I/O-bound, alta concurrencia
  1M conexiones HTTP simultáneas (10ms de latencia de "BD")
  
Workload 2: CPU-bound, paralelismo de datos
  Suma paralela de 1B números flotantes
  
Workload 3: Mixed, pipeline
  Fetch HTTP + procesamiento + almacenamiento en BD
  
Workload 4: Baja latencia
  Round-trip de mensaje en <100µs p99
```

**Restricciones:** Reportar para cada combinación (lenguaje × workload):
throughput, latencia p50/p99/p999, memoria pico.
Determinar qué lenguaje "gana" cada workload y por qué.

**Pista:** Los resultados esperados aproximados:
- Workload 1: Go ≈ C# > Java (con VTs) > Python (con asyncio)
- Workload 2: Rust > C# ≈ Java > Go > Python (con GIL)
- Workload 3: Go ≈ Rust ≈ C# > Java > Python
- Workload 4: Rust > C# > Go > Java > Python
Los números varían significativamente según la implementación y el hardware.

**Implementar en:** Go + Rust + Java + C#

---

### Ejercicio 16.7.4 — Errores de concurrencia: qué lenguaje los previene

**Enunciado:** Para cada tipo de error de concurrencia, determina si es:
(A) imposible de escribir, (B) detectado en compilación, (C) detectado en runtime,
(D) solo detectable con herramientas externas, o (E) silenciosamente incorrecto.

```
Error                     Go      Rust    Java    Python  C#
─────────────────────────────────────────────────────────────
Data race                  D       A       D       E       D
Use-after-free             E       A       N/A     N/A     N/A
Deadlock                   D       C       D       D       D
Memory leak                C       A       C       C       C
Iterator invalidation      E       A       C/D     C/D     B/C
Starvation                 D       D       D       D       D
Priority inversion         D       D       D       D       D
```

**Restricciones:** Para cada error marcado como (E) o (D), escribe el código
que lo introduce y el test/herramienta que lo detecta (si existe).
Para los marcados (A), confirma con un intento de compilación que falla.

**Pista:** "A" (imposible) es la ventaja central de Rust sobre los otros lenguajes.
"N/A" para Java/Python/C# en use-after-free porque tienen GC.
El Go race detector (-race) puede detectar data races pero no todas — solo
las que ocurren durante la ejecución del test, no las que podrían ocurrir
bajo otros schedulings.

**Implementar en:** Todos los lenguajes (para confirmar o refutar la tabla)

---

### Ejercicio 16.7.5 — La decisión arquitectónica: elegir el lenguaje correcto

**Enunciado:** Dado un sistema hipotético con los siguientes requisitos,
argumenta qué lenguaje(s) elegirías y por qué:

**Sistema: Plataforma de trading de alta frecuencia**

```
Requisitos:
  - Latencia: p99 < 500µs, p999 < 2ms
  - Throughput: 100K órdenes/segundo
  - Uptime: 99.999% (5.26 minutos de downtime por año)
  - Seguridad: auditoría de memory safety requerida por regulador
  - Equipo: 8 ingenieros, mezcla de senior/mid-level
  - Timeline: 18 meses para MVP, 36 meses para sistema completo
  - Integraciones: 15 exchanges, Bloomberg API (SDK en Java/C++/Python)

Restricciones adicionales:
  - El regulador requiere código auditado en un lenguaje con garantías de memory safety
  - El equipo tiene experiencia en Java y Python, poca en Go/Rust/C#
  - Budget limitado — no se puede contratar especialistas de Rust
```

**Restricciones:** La respuesta debe ser específica y justificada con referencias
a los ejercicios del repositorio. No vale "depende" sin especificar de qué depende.

**Pista:** La respuesta no tiene una única solución correcta. Consideraciones:
- Rust cumple los requisitos de latencia y safety pero el equipo no tiene experiencia
- Java 21 con VTs puede cumplir los requisitos con el equipo actual, pero el GC puede afectar p999
- C# tiene buen rendimiento y memory safety pero el SDK de Bloomberg no lo soporta
- Go tiene buen rendimiento pero no tiene las garantías de memory safety del regulador
- La arquitectura mixta (Rust core + Java/Python glue) añade complejidad
Una respuesta honesta reconoce estos tradeoffs y hace una recomendación justificada.

**Implementar en:** Documento técnico + implementación de la parte más crítica

---

## Resumen del capítulo y de la Parte 3

**Los cuatro lenguajes y sus modelos de concurrencia:**

| Lenguaje | Primitiva principal | Garantía de seguridad | Runtime |
|---|---|---|---|
| Go | Goroutines + canales | Race detector (runtime) | GC + scheduler M:N |
| Rust | Ownership + async | Compile-time (Send/Sync) | Sin GC, sin runtime obligatorio |
| Java | Virtual Threads + Structured Concurrency | Nada (historical) / -race (experimental) | JVM GC + JIT |
| Python | asyncio + multiprocessing | Nada (GIL protege en I/O) | CPython + GIL |
| C# | async/await + Channels + TPL | Nada built-in | CLR GC + JIT |

**La evolución del async/await en los cinco lenguajes:**

```
2012: C# 5.0 inventa async/await
2015: Python 3.5 adopta async/await (asyncio)
2017: JavaScript ES2017 adopta async/await
2019: Rust 1.39 adopta async/await (sin runtime)
2021: Java adopta corrutinas vía Project Loom (Virtual Threads)
```

**El principio que cierra la Parte 3:**

> No hay un lenguaje universalmente mejor para concurrencia.
> Cada uno optimiza para un conjunto diferente de constraints.
> La habilidad más valiosa no es dominar un lenguaje —
> es entender los tradeoffs de cada modelo lo suficientemente bien
> como para elegir el correcto para cada situación.

## La pregunta que guía la Parte Final (Cap.17)

> La Parte 4 (Cap.17) cierra el repositorio con los problemas más difíciles:
> los patrones arquitectónicos que unen todo —
> cómo diseñar sistemas que son correctos, eficientes, y operables bajo concurrencia real
> en producción durante años.
