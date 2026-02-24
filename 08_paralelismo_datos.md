# Guía de Ejercicios — Paralelismo de Datos: Múltiples Núcleos para Trabajo CPU-bound

> Implementar cada ejercicio en: **Go · Rust · Java · C#**
> Equivalencias en Python donde aplique (multiprocessing, no threads).
>
> Regla: no uses IA para generar soluciones. El objetivo es ver el problema,
> no memorizar la solución.

---

## La distinción que inaugura la Parte 2

Los Cap.01–07 cubrieron **concurrencia**: la estructura de un programa que maneja
múltiples tareas que progresan al mismo tiempo. El énfasis es la coordinación.

La Parte 2 cubre **paralelismo**: usar múltiples núcleos físicos simultáneamente
para hacer trabajo más rápido. El énfasis es la velocidad.

```
Concurrencia:
  "Tengo 1000 requests HTTP que manejar."
  → Goroutines bloqueadas en I/O mientras otras corren
  → Un solo núcleo puede manejar miles de conexiones concurrentes
  → El límite es la latencia de I/O, no el CPU

Paralelismo:
  "Tengo que calcular el hash SHA-256 de 1 millón de archivos."
  → Trabajo CPU-bound: cada hash usa el 100% de un núcleo
  → Necesito múltiples núcleos para hacerlo más rápido
  → El límite es el número de núcleos físicos disponibles
```

**El mismo código, distintos escenarios:**

```go
// Escenario 1: I/O-bound (concurrencia, no paralelismo)
for _, url := range urls {
    go func(u string) {
        resp, _ := http.Get(u)     // pasa el 99% del tiempo esperando la red
        procesar(resp)
    }(url)
}
// Con 1 núcleo y 1000 goroutines: funciona perfectamente
// Con 8 núcleos: no mejora mucho — el cuello de botella es la red

// Escenario 2: CPU-bound (paralelismo, no solo concurrencia)
for _, archivo := range archivos {
    go func(f string) {
        datos, _ := os.ReadFile(f)
        calcularHash(datos)        // pasa el 99% del tiempo en CPU
    }(archivo)
}
// Con 1 núcleo: las goroutines se turnan en el mismo núcleo
// Con 8 núcleos: 8x más rápido (idealmente) — aquí el paralelismo importa
```

---

## Antes de la teoría: ¿cuándo el paralelismo ayuda?

```go
// Benchmark: calcular suma de cuadrados de 10 millones de números

// Versión secuencial
func sumaSecuencial(nums []int) int {
    sum := 0
    for _, n := range nums {
        sum += n * n
    }
    return sum
}

// Versión paralela con 8 workers
func sumaParalela(nums []int, workers int) int {
    chunk := len(nums) / workers
    partial := make(chan int, workers)

    for i := 0; i < workers; i++ {
        start := i * chunk
        end := start + chunk
        if i == workers-1 { end = len(nums) }

        go func(slice []int) {
            sum := 0
            for _, n := range slice {
                sum += n * n
            }
            partial <- sum
        }(nums[start:end])
    }

    total := 0
    for i := 0; i < workers; i++ {
        total += <-partial
    }
    return total
}
```

```
Resultado en una máquina con 8 cores:

Secuencial (1 core):    52.3 ms
Paralela (2 workers):   27.1 ms  (1.93x — casi 2x)
Paralela (4 workers):   14.2 ms  (3.68x — casi 4x)
Paralela (8 workers):    8.1 ms  (6.46x — no 8x, ¿por qué?)
Paralela (16 workers):   8.3 ms  (6.29x — más workers no ayuda)
```

El speedup no es lineal porque existe una parte secuencial (dividir el trabajo,
juntar los resultados) que no se paraleliza. Ese límite se llama **Ley de Amdahl**.

---

## Tabla de contenidos

- [Sección 8.1 — La Ley de Amdahl y el límite teórico del speedup](#sección-81--la-ley-de-amdahl-y-el-límite-teórico-del-speedup)
- [Sección 8.2 — Paralelismo fork-join](#sección-82--paralelismo-fork-join)
- [Sección 8.3 — Reducción paralela](#sección-83--reducción-paralela)
- [Sección 8.4 — Paralelismo de datos con SIMD y el runtime](#sección-84--paralelismo-de-datos-con-simd-y-el-runtime)
- [Sección 8.5 — Cache efficiency en código paralelo](#sección-85--cache-efficiency-en-código-paralelo)
- [Sección 8.6 — Work stealing y balanceo dinámico](#sección-86--work-stealing-y-balanceo-dinámico)
- [Sección 8.7 — Paralelismo en la práctica: casos reales](#sección-87--paralelismo-en-la-práctica-casos-reales)

---

## Sección 8.1 — La Ley de Amdahl y el Límite Teórico del Speedup

La **Ley de Amdahl** (1967) dice que si una fracción `f` de tu programa
no puede paralelizarse, el speedup máximo con `N` procesadores es:

```
Speedup(N) = 1 / (f + (1-f)/N)

Cuando N → ∞:
Speedup(∞) = 1 / f

Si f = 0.10 (10% secuencial):
  Speedup(∞) = 10x — nunca más de 10x, independientemente de los núcleos
```

**La tabla que todo desarrollador debe tener en mente:**

```
Fracción secuencial f    Speedup máximo (infinitos núcleos)
─────────────────────    ──────────────────────────────────
0.50  (50% secuencial)   2x
0.25  (25% secuencial)   4x
0.10  (10% secuencial)   10x
0.05  (5% secuencial)    20x
0.01  (1% secuencial)    100x
0.001 (0.1% secuencial)  1000x
```

**La Ley de Gustafson** (perspectiva alternativa, 1988):

Amdahl asume que el tamaño del problema es fijo. Gustafson observó que
en la práctica, con más procesadores, los problemas crecen:

```
Speedup de Gustafson = N - f(N-1)

Con N=8 procesadores y f=0.10:
  Amdahl: 1 / (0.1 + 0.9/8) = 4.7x
  Gustafson: 8 - 0.1*7 = 7.3x

La diferencia: Gustafson asume que usas los 8 procesadores para un problema
8x más grande, no para el mismo problema 8x más rápido.
```

---

### Ejercicio 8.1.1 — Medir la fracción secuencial de tu código

**Enunciado:** Para cualquier algoritmo que paralelices, mide empíricamente
la fracción secuencial usando la Ley de Amdahl inversa.

Dada la tabla de speedups medidos:
```
1 worker:  T₁ = 100ms
2 workers: T₂ = 54ms  → speedup = 1.85x
4 workers: T₄ = 29ms  → speedup = 3.45x
8 workers: T₈ = 18ms  → speedup = 5.56x
```

Calcula `f` usando la fórmula inversa y verifica que es consistente
entre las distintas mediciones.

**Restricciones:** Implementa la fórmula inversa:
`f = (1/Speedup - 1/N) / (1 - 1/N)`.
La fracción secuencial `f` debe ser consistente (±5%) entre todas las mediciones.
Si no es consistente, el modelo de Amdahl no aplica — documenta por qué.

**Pista:** Una fracción secuencial inconsistente sugiere superlineal speedup
(improbable) o que la paralelización tiene overhead creciente con N.
El overhead del scheduler, la contención en locks compartidos, y el false
sharing pueden hacer que el speedup sea sublineal de forma no uniforme.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.1.2 — La fracción secuencial oculta

**Enunciado:** Identifica las partes secuenciales ocultas en este pipeline paralelo:

```go
func procesarParalelo(datos []Item, workers int) []Resultado {
    // 1. Dividir — ¿cuánto tiempo tarda? ¿es paralelo?
    chunks := dividir(datos, workers)

    // 2. Procesar — paralelo
    resultadosParciales := make([][]Resultado, workers)
    var wg sync.WaitGroup
    for i, chunk := range chunks {
        wg.Add(1)
        go func(id int, c []Item) {
            defer wg.Done()
            resultadosParciales[id] = procesarChunk(c)
        }(i, chunk)
    }
    wg.Wait()

    // 3. Juntar — ¿cuánto tiempo tarda? ¿es paralelo?
    return juntar(resultadosParciales)
}
```

**Restricciones:** Mide el tiempo de cada fase (dividir, procesar, juntar)
con distintos tamaños de input. Para inputs pequeños, ¿cuál fase domina?

**Pista:** Para inputs de 100 elementos, dividir y juntar pueden dominar
el tiempo total — el "overhead de paralelismo". Para inputs de 1 millón,
la fase de procesamiento domina. El break-even point (donde el paralelismo
empieza a ganar) depende del ratio entre overhead y trabajo real.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.1.3 — Identificar el cuello de botella con profiling

**Enunciado:** Usa `go tool pprof` para identificar qué parte de este
procesamiento de imágenes no se beneficia del paralelismo:

```go
func procesarImagenes(imagenes []Imagen, workers int) {
    jobs := make(chan Imagen, len(imagenes))
    for _, img := range imagenes {
        jobs <- img
    }
    close(jobs)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for img := range jobs {
                redimensionar(img)    // CPU-bound
                comprimir(img)        // CPU-bound
                guardar(img)          // I/O-bound ← cuello de botella?
            }
        }()
    }
    wg.Wait()
}
```

**Restricciones:** Genera un CPU profile y un trace. Identifica el tiempo
gastado en cada fase y el tiempo de espera en I/O.

**Pista:** `guardar` escribe al disco — I/O-bound. Todos los workers compiten
por el mismo disco. El paralelismo no ayuda para esta fase (un HDD solo hace
una operación a la vez). Un SSD NVMe puede hacer operaciones paralelas,
pero el kernel puede serializar las writes de todas formas.

**Implementar en:** Go · Java · Rust

---

### Ejercicio 8.1.4 — Ley de Gustafson: escalar el problema con los núcleos

**Enunciado:** En lugar de medir cuánto más rápido resuelves el mismo problema,
mide cuánto problema más grande resuelves en el mismo tiempo.

Para el algoritmo de N-cuerpos (simulación gravitacional):
- Con 1 núcleo, 5000 cuerpos en 1 segundo
- Con 4 núcleos, ¿cuántos cuerpos puedes simular en 1 segundo?
- Con 8 núcleos, ¿cuántos?

Verifica que la Ley de Gustafson predice mejor el resultado que la de Amdahl
cuando escalas el problema con los núcleos.

**Restricciones:** Implementa la simulación N-cuerpos con O(N²) naive.
El speedup se mide en "cuerpos procesados por segundo", no en "tiempo para N fijo".

**Pista:** La simulación N-cuerpos es `O(N²)` — cada cuerpo interactúa con
todos los demás. El parallelismo divide los pares de cuerpos entre los workers.
Con 4 workers, puedes procesar ~4x más pares por segundo → ~2x más cuerpos
(porque 4x cuerpos = 16x pares, pero con 4x velocidad = 4x pares → sqrt(4)x cuerpos).

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.1.5 — Overhead de paralelismo: el "break-even" point

**Enunciado:** Mide el break-even point para la suma paralela del inicio del capítulo:
el tamaño mínimo de array donde la versión paralela es más rápida que la secuencial.

```
Para arrays pequeños: overhead de lanzar goroutines > speedup de paralelismo
Para arrays grandes:  speedup de paralelismo > overhead
El break-even está en algún punto entre los dos.
```

**Restricciones:** Mide para arrays de tamaño `10^k` con k=2,3,4,5,6,7.
El break-even debe reportarse en número de elementos, no en tiempo.
Prueba con 1, 2, 4, y 8 workers para ver si el break-even cambia con más workers.

**Pista:** El overhead de lanzar N goroutines es aproximadamente `N * 400ns`.
El trabajo secuencial para un array de M elementos es `M * ~2ns`.
El break-even teórico es cuando `M * 2ns / N ≈ 400ns`, es decir `M ≈ 200N`.
Para 8 workers: break-even ≈ 1600 elementos. Verifica si el medido coincide.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 8.2 — Paralelismo Fork-Join

El patrón fork-join divide recursivamente un problema en subproblemas
hasta que son lo suficientemente pequeños para resolver secuencialmente,
y luego combina los resultados.

```
fork-join:

problema(1M elementos)
  ├─ fork: problema(500K)
  │     ├─ fork: problema(250K)
  │     │     ├─ fork: problema(125K)  → resolver (umbral)
  │     │     └─ fork: problema(125K)  → resolver (umbral)
  │     │     └─ join: combinar 125K + 125K
  │     └─ ...
  └─ fork: problema(500K)
        └─ ...
  └─ join: combinar todo
```

**El umbral de corte — la decisión crítica:**

```go
const UMBRAL = 10_000  // por debajo de este tamaño, resolver secuencialmente

func sumaForkJoin(nums []int) int {
    if len(nums) <= UMBRAL {
        // resolver secuencialmente — sin más forks
        sum := 0
        for _, n := range nums { sum += n * n }
        return sum
    }

    // fork: dividir en dos mitades
    mid := len(nums) / 2
    ch := make(chan int, 1)
    go func() {
        ch <- sumaForkJoin(nums[:mid])  // fork
    }()
    derecha := sumaForkJoin(nums[mid:]) // resolver la derecha en este goroutine

    // join: esperar la izquierda y combinar
    return <-ch + derecha
}
```

**Por qué resolver la derecha en el goroutine actual:**

Si lanzas dos goroutines (izquierda y derecha) y esperas ambas, el goroutine
actual se bloquea inmediatamente. Es más eficiente resolver la mitad derecha
en el goroutine actual mientras la izquierda corre en paralelo.
Esto es el "continuation passing" del fork-join.

---

### Ejercicio 8.2.1 — Merge sort paralelo

**Enunciado:** Implementa merge sort usando fork-join. La división es natural:
ordena cada mitad en paralelo, luego mezcla secuencialmente.

```go
func MergeSortParalelo(nums []int, umbral int) []int
```

**Restricciones:** Para `len(nums) <= umbral`, usa sort.Slice secuencial.
Para slices más grandes, fork dos goroutines. La fase de merge es siempre secuencial.
Mide el speedup real vs el sort secuencial para arrays de 1M y 10M elementos.

**Pista:** El merge de dos arrays ya ordenados es `O(n)` — rápido pero secuencial.
Para maximizar el paralelismo, el umbral debe ser lo suficientemente grande
para que la fase de merge no domine. Un umbral de `len(nums)/NumCPU` para
el nivel superior, y `10000` para los niveles internos, suele funcionar bien.

**Implementar en:** Go · Java (`ForkJoinPool`) · Rust (`rayon`) · C#

---

### Ejercicio 8.2.2 — QuickSort paralelo con elección de pivote

**Enunciado:** QuickSort paralelo divide alrededor de un pivote y ordena
ambas particiones en paralelo. El balance del trabajo depende del pivote.

```go
func QuickSortParalelo(nums []int, umbral int)
```

Implementa tres estrategias de elección de pivote y mide su impacto
en el balance del trabajo paralelo:
1. Primer elemento (peor caso en arrays ya ordenados)
2. Mediana de tres (primer, medio, último)
3. Pivote aleatorio

**Restricciones:** Mide la distribución de tamaños de las particiones
para arrays aleatorios, casi-ordenados, y en orden inverso.
Un buen pivote produce particiones de tamaño similar → trabajo balanceado.

**Pista:** Con el pivote peor (primer elemento) en un array ya ordenado,
QuickSort degrada a `O(n²)` y el paralelismo no ayuda (una partición siempre
tiene 0 elementos, la otra tiene n-1). La mediana de tres es `O(n log n)`
en casi todos los casos prácticos.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.2.3 — Árbol de decisión paralelo

**Enunciado:** La evaluación de un árbol de decisión (como en Random Forest)
es naturalmente recursiva. Paraleliza la evaluación de un bosque de N árboles:

```go
type Arbol struct {
    izquierda, derecha *Arbol
    caracteristica int
    umbral float64
    prediccion int  // en hojas
}

func EvaluarBosqueParalelo(bosque []*Arbol, muestra []float64, umbral int) int
// Retorna la predicción por mayoría de votos de todos los árboles
```

**Restricciones:** El umbral define cuándo evaluar secuencialmente vs en paralelo.
Mide el speedup con bosques de 10, 100, y 1000 árboles.

**Pista:** Cada árbol es independiente — no hay estado compartido entre ellos.
Esto hace que la paralelización sea trivialmente correcta. El desafío es el overhead:
evaluar un árbol individual toma ~microsegundos; lanzar una goroutine toma ~400ns.
Para árboles pequeños, el overhead domina.

**Implementar en:** Go · Java · Python (`concurrent.futures`) · Rust

---

### Ejercicio 8.2.4 — Fibonacci paralelo: cuándo el fork-join se vuelve ineficiente

**Enunciado:** El Fibonacci paralelo es el ejemplo clásico de por qué
el fork-join ingenuo puede ser peor que el secuencial:

```go
// Fork-join ingenuo — exponencialmente muchas goroutines
func FibParalelo(n int) int {
    if n <= 1 { return n }
    ch := make(chan int, 1)
    go func() { ch <- FibParalelo(n - 1) }()
    return FibParalelo(n-2) + <-ch
    // Para n=30: ~1 millón de goroutines lanzadas
}

// Fork-join con umbral — solo paraleliza el nivel superior
func FibParaleloUmbral(n, umbral int) int {
    if n <= umbral { return fibSecuencial(n) }
    // ...
}
```

Mide el número de goroutines creadas para distintos valores de n y umbral.
Encuentra el umbral óptimo que maximiza el speedup sin exceder X goroutines.

**Restricciones:** Verifica que el resultado es correcto para n=35.
El overhead de goroutines sin umbral debe ser medible y significativo.

**Pista:** El árbol de llamadas de Fibonacci tiene `O(φⁿ)` nodos (φ ≈ 1.618).
Para n=30: ~1.3M goroutines. Para n=35: ~15M goroutines — claramente ineficiente.
El umbral óptimo es el que produce `NumCPU * K` goroutines activas simultáneamente,
donde K=2-4. Para n=35 y 8 núcleos: umbral ≈ 20-25 (produce ~16-32 goroutines activas).

**Implementar en:** Go · Java (`ForkJoinPool.ManagedBlocker`) · Rust

---

### Ejercicio 8.2.5 — Fork-join con resultados heterogéneos

**Enunciado:** En un análisis de texto, el trabajo de cada chunk es diferente:
un chunk con muchas palabras raras tarda más que uno con palabras comunes.
El trabajo desbalanceado hace que algunos workers terminen antes que otros
(problema de "stragglers").

Implementa fork-join con work stealing para manejar trabajo desbalanceado:
cuando un worker termina su chunk, "roba" trabajo del chunk más grande
de otro worker que sigue ocupado.

**Restricciones:** Mide la distribución de tiempos de completitud de los workers.
Con work stealing, los workers deben terminar en un rango de ±10% del tiempo total.
Sin work stealing, algunos pueden terminar en el 30% del tiempo mientras otros
siguen corriendo.

**Pista:** Work stealing para fork-join: cada worker tiene su propia deque
(double-ended queue). Trabaja desde el frente (LIFO). Los workers ociosos
roban desde el fondo de la deque de un worker ocupado (FIFO del ladrón).
Esta asimetría minimiza la contención: el propietario y el ladrón rara vez
acceden al mismo extremo simultáneamente.

**Implementar en:** Go · Java (`ForkJoinPool` tiene work stealing nativo) · Rust (`rayon`)

---

## Sección 8.3 — Reducción Paralela

La reducción es la operación de combinar N elementos en uno: suma, producto,
máximo, concatenación. Es paralelizable porque la operación de combinación
es asociativa.

**La reducción en árbol — O(log N) pasos con N/2 operaciones por paso:**

```
N=8 elementos:
  Paso 1: [a+b] [c+d] [e+f] [g+h]  ← 4 operaciones en paralelo
  Paso 2: [(a+b)+(c+d)] [(e+f)+(g+h)]  ← 2 operaciones en paralelo
  Paso 3: [(a+b+c+d)+(e+f+g+h)]  ← 1 operación

  Total: log₂(8) = 3 pasos (vs 7 pasos secuenciales)
  Trabajo total: 7 operaciones (mismo que secuencial)
  Tiempo paralelo: 3 pasos (vs 7 pasos)
```

**La condición para que la reducción en árbol sea correcta:**

```
La operación ⊕ debe ser asociativa: (a ⊕ b) ⊕ c = a ⊕ (b ⊕ c)

Ejemplos asociativos:
  +  (suma)           ← ✓
  *  (producto)        ← ✓
  max, min             ← ✓
  AND, OR, XOR         ← ✓
  string concatenation ← ✓

No asociativos (no paralelizables en árbol):
  -  (resta)           ← ✗ (a - b - c ≠ a - (b - c))
  /  (división)        ← ✗
  potencia             ← ✗
```

---

### Ejercicio 8.3.1 — Reducción paralela genérica

**Enunciado:** Implementa una función de reducción paralela genérica:

```go
func Reducir[T any](
    datos []T,
    neutro T,
    op func(T, T) T,
    workers int,
) T
```

Donde `neutro` es el elemento neutro de la operación (0 para suma, 1 para producto,
`math.MinInt64` para máximo, etc.).

**Restricciones:** La función debe funcionar correctamente para cualquier
operación asociativa. Verifica con suma, producto, y máximo.
El resultado debe ser idéntico al de la reducción secuencial.

**Pista:** La correctness depende de que `op` sea asociativa. Si el usuario
pasa una operación no asociativa (como resta), el resultado puede ser incorrecto —
documenta esta precondición. Para verificar la associatividad en tests,
usa property-based testing: `op(op(a,b),c) == op(a,op(b,c))` para valores aleatorios.

**Implementar en:** Go · Java (`Stream.reduce`) · Rust (`rayon::iter::ParallelIterator::reduce`) · C#

---

### Ejercicio 8.3.2 — Prefix sum (scan) paralelo

**Enunciado:** El prefix sum calcula, para cada posición i, la suma de todos
los elementos hasta i. La versión secuencial es O(N). La versión paralela
también es O(N) trabajo pero O(log N) en tiempo paralelo.

```
Input:  [3, 1, 7, 0, 4, 1, 6, 3]
Output: [3, 4, 11, 11, 15, 16, 22, 25]  ← suma acumulada
```

**El algoritmo de Blelloch (1990):**
```
Fase up-sweep (reducción):   construir árbol de sumas parciales
Fase down-sweep (distribución): propagar los prefijos de arriba hacia abajo
```

**Restricciones:** Implementa el algoritmo de Blelloch in-place en un array.
El resultado debe ser idéntico al de la versión secuencial.
Mide el speedup real vs la versión secuencial para arrays de 10M elementos.

**Pista:** El prefix sum paralelo es menos intuitivo que la reducción simple
porque tiene dependencias de datos más complejas. El algoritmo de Blelloch
las resuelve con dos pases sobre el árbol implícito del array.
Este algoritmo aparece en GPU programming (CUDA) donde el speedup es enorme.

**Implementar en:** Go · Java · Rust · C (para comparar con SIMD manualmente)

---

### Ejercicio 8.3.3 — Reducción de histograma

**Enunciado:** Un histograma cuenta la frecuencia de cada valor en un array.
La versión paralela necesita combinar histogramas parciales:

```go
func HistogramaParalelo(datos []int, bins int, workers int) []int
```

**El problema:** si todos los workers actualizan el mismo histograma,
hay alta contención. Si cada worker tiene su propio histograma, necesitas
combinarlos al final.

**Restricciones:** Implementa tres estrategias:
1. Histograma compartido con mutex (contención alta)
2. Histograma compartido con operaciones atómicas (mejor, pero aún hay contención)
3. Histogramas privados por worker + reducción al final (mejor para bins pequeños)

Mide el speedup de cada estrategia para bins = {10, 100, 1000}.

**Pista:** Para bins pequeños (10), la estrategia 3 gana porque la reducción
final es rápida (combinar 10 valores por worker). Para bins grandes (1000),
la memoria de los histogramas privados puede ser un problema (8*workers*1000 bytes).
La elección óptima depende del número de bins y de la contención.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.3.4 — MapReduce simplificado

**Enunciado:** MapReduce (Google, 2004) es una reducción paralela a gran escala.
Implementa una versión local (sin red):

```go
type MapReduceJob[K comparable, V, R any] struct {
    Datos    []V
    Map      func(V) (K, R)          // transforma cada elemento en (clave, valor)
    Reduce   func(K, []R) R          // combina valores de la misma clave
    Workers  int
}

func (j *MapReduceJob[K, V, R]) Ejecutar() map[K]R
```

**Casos de uso a implementar:**
1. Contar palabras en un corpus de documentos
2. Encontrar la palabra más frecuente por capítulo
3. Calcular la suma total de ventas por categoría

**Restricciones:** La fase Map debe correr en paralelo (un worker por chunk).
La fase Reduce puede ser paralela o secuencial — decide cuál es mejor.
Verifica que el resultado es idéntico al de la implementación secuencial.

**Pista:** La fase Map produce pares (clave, valor). El framework agrupa los
pares por clave (shuffle). La fase Reduce procesa cada grupo.
El shuffle es inherentemente secuencial si no hay particionado previo —
documenta este cuello de botella.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 8.3.5 — Reducción con orden parcial (semilattice)

**Enunciado:** Algunas operaciones de reducción no son completamente asociativas
pero tienen un orden parcial útil. Por ejemplo, en un sistema de votación:
combinar votos es asociativo, pero el resultado final depende del orden de los ties.

Implementa una reducción paralela para un sistema de ranking con estas propiedades:
- Asociativa para la mayoría de casos
- Determinista cuando no hay ties
- Estable (mismo input siempre produce mismo output)

**Restricciones:** Con 1000 items rankeados por 10,000 voters, el resultado
debe ser el mismo con 1, 4, y 8 workers. Documenta cómo se rompen los ties.

**Pista:** La forma más simple de garantizar determinismo en reducción paralela
con ties: asignar un ID único a cada elemento y usarlo como desempate.
La reducción paralela puede usar cualquier orden porque el desempate por ID
es determinista y no depende del orden de procesamiento.

**Implementar en:** Go · Java · Python · Rust · C#

---

## Sección 8.4 — Paralelismo de Datos con SIMD y el Runtime

SIMD (Single Instruction, Multiple Data) ejecuta la misma operación sobre
múltiples datos simultáneamente usando registros especiales del CPU.

```
Suma escalar (normal):
  r = a[0] + b[0]  ← 1 suma por instrucción

Suma SIMD (AVX-256):
  r[0:4] = a[0:4] + b[0:4]  ← 4 sumas por instrucción (int64)
  r[0:8] = a[0:8] + b[0:8]  ← 8 sumas por instrucción (int32)

Speedup teórico: 4x-8x solo por SIMD, sin threads adicionales
```

**SIMD en Go:**

Go no expone SIMD directamente. Pero el compilador de Go **auto-vectoriza**
algunos loops cuando detecta el patrón correcto:

```go
// Este loop puede ser auto-vectorizado:
for i := range a {
    c[i] = a[i] + b[i]  // suma elemento a elemento, sin dependencias
}

// Este no (dependencia entre iteraciones):
for i := 1; i < len(a); i++ {
    a[i] = a[i] + a[i-1]  // depende del resultado anterior
}
```

**Para control explícito de SIMD en Go:**
- `golang.org/x/sys/cpu` — detectar capacidades SIMD del CPU
- Assembly en Go (`_amd64.s`) — instrucciones SIMD manuales
- `github.com/klauspost/cpuid` — información detallada del CPU

---

### Ejercicio 8.4.1 — Verificar auto-vectorización del compilador

**Enunciado:** Verifica qué loops vectoriza el compilador de Go automáticamente.

```go
// Test 1: suma vectorizable
func SumaSlices(a, b, c []float64) {
    for i := range a { c[i] = a[i] + b[i] }
}

// Test 2: suma no vectorizable (dependencia)
func SumaAcumulada(a []float64) {
    for i := 1; i < len(a); i++ { a[i] += a[i-1] }
}

// Test 3: condicional — ¿se vectoriza?
func FiltroPositivos(a, b []float64) {
    for i := range a {
        if a[i] > 0 { b[i] = a[i] } else { b[i] = 0 }
    }
}
```

**Restricciones:** Usa `go tool compile -S -gcflags="-S"` para ver el ensamblador
generado. Identifica las instrucciones SIMD (XMM, YMM en x86).
Compara el rendimiento con y sin auto-vectorización desactivando con
`//go:noescape` o cambiando el patrón del loop.

**Pista:** Las instrucciones SIMD en x86 incluyen: MOVAPS, ADDPS, MULPS (SSE),
VADDPD, VMULPD (AVX). Si el ensamblador generado no las contiene, el loop
no fue vectorizado. Otra forma: benchmark — un loop vectorizado es ~4-8x
más rápido que el escalar equivalente para el mismo número de iteraciones.

**Implementar en:** Go · Rust (`cargo asm`)

---

### Ejercicio 8.4.2 — Suma de punto flotante con SIMD manual en Go

**Enunciado:** Implementa la suma de un slice de float64 usando ensamblador
de Go con instrucciones AVX2:

```asm
// sum_amd64.s
TEXT ·SumaAVX2(SB),NOSPLIT,$0-32
    // Suma 4 float64 por iteración usando registros YMM
    // ...
    RET
```

Y en Go:
```go
//go:noescape
func SumaAVX2(data []float64) float64
```

**Restricciones:** La implementación debe detectar en tiempo de ejecución
si AVX2 está disponible y caer back a la versión escalar si no.
El resultado debe ser numéricamente idéntico (±epsilon) a la suma secuencial.

**Pista:** La suma de punto flotante no es perfectamente asociativa —
el orden de las sumas afecta el error de redondeo. Para la mayoría de
aplicaciones, la diferencia es insignificante. Para aplicaciones de alta
precisión (computación científica), documenta la diferencia de error.

**Implementar en:** Go + Assembly · Rust (con `std::arch`) · C (para comparar)

---

### Ejercicio 8.4.3 — Búsqueda paralela con SIMD y goroutines

**Enunciado:** Combina SIMD (para eficiencia por núcleo) y goroutines
(para múltiples núcleos) para una búsqueda de substring ultra-rápida:

```go
// Buscar un patrón de 16 bytes en un corpus de 1GB
func BuscarParalelo(corpus []byte, patron []byte, workers int) []int
```

El algoritmo:
1. Divide el corpus en chunks entre workers (goroutines)
2. Cada worker usa SIMD para comparar 16 bytes a la vez

**Restricciones:** Mide el throughput en GB/s. La búsqueda secuencial
naive es ~500 MB/s. Con SIMD, debería superar 2 GB/s. Con 8 cores y SIMD,
debería acercarse al ancho de banda de memoria (típicamente 40-80 GB/s).

**Pista:** El cuello de botella para búsquedas en datos grandes es el ancho
de banda de memoria (memory bandwidth), no el compute. Más workers que
núcleos no ayuda cuando el limitante es la RAM.
`strings.Index` en Go ya usa SIMD internamente en plataformas modernas.

**Implementar en:** Go · Rust · C (para baseline con `memmem`)

---

### Ejercicio 8.4.4 — Detectar capacidades del CPU en tiempo de ejecución

**Enunciado:** Implementa un dispatcher que elige la mejor implementación
de una operación según las capacidades del CPU:

```go
type Implementacion struct {
    Nombre    string
    Disponible func() bool
    Ejecutar   func([]float64) float64
}

var impls = []Implementacion{
    {"AVX-512", func() bool { return cpu.X86.HasAVX512F }, sumaAVX512},
    {"AVX2",    func() bool { return cpu.X86.HasAVX2 }, sumaAVX2},
    {"SSE4.1",  func() bool { return cpu.X86.HasSSE41 }, sumaSSE41},
    {"escalar", func() bool { return true }, sumaEscalar},
}

func SumaAdaptiva(data []float64) float64 {
    for _, impl := range impls {
        if impl.Disponible() {
            return impl.Ejecutar(data)
        }
    }
    panic("sin implementación disponible")
}
```

**Restricciones:** El dispatcher debe ser thread-safe (la selección ocurre una vez
al inicio y se cachea). Verifica que el resultado es idéntico para todas las implementaciones.

**Pista:** Este patrón es exactamente cómo funcionan las librerías optimizadas
como OpenBLAS, oneDNN, y la librería estándar de Go en plataformas x86.
La detección de CPU con `golang.org/x/sys/cpu` es cero-cost después del
primer uso si se cachea el resultado.

**Implementar en:** Go · Rust · C

---

### Ejercicio 8.4.5 — Límite de ancho de banda de memoria

**Enunciado:** Mide el ancho de banda de memoria de tu máquina y verifica
que tus operaciones paralelas se acercan a ese límite:

```go
// Bandwidth test — operación que solo lee/escribe memoria, sin compute
func CopiarMemoria(dst, src []byte) {
    copy(dst, src)  // limitado por bandwidth, no por CPU
}

// Compute test — operación que usa el CPU intensamente
func MultiplicarMatrices(a, b, c [][]float64) {
    // limitado por FLOPS del CPU, no por bandwidth
}
```

**Restricciones:** Mide el throughput de `CopiarMemoria` con 1, 2, 4, 8 goroutines.
El throughput debe saturar cuando las goroutines agotan el ancho de banda de memoria
(~50-100 GB/s para RAM moderna, ~1 TB/s para caché L3).

**Pista:** Para operaciones memory-bound, más threads no ayuda después de
cierto punto — todas compiten por el mismo bus de memoria. Para operaciones
compute-bound, más threads sí ayuda hasta NumCPU.
`lmbench`, `stream`, y `sysbench` son herramientas estándar para medir bandwidth.

**Implementar en:** Go · Rust · C (benchmark de referencia con STREAM)

---

## Sección 8.5 — Cache Efficiency en Código Paralelo

El rendimiento del código paralelo en hardware moderno depende casi tanto
de la eficiencia de caché como del número de núcleos. Un código que ignora
la jerarquía de caché puede ser 10x más lento del teórico.

**La jerarquía de caché (valores típicos para 2024):**

```
Registro:   ~0.3 ns, ~TB/s bandwidth
L1 caché:   ~1 ns, 1-5 MB, ~2 TB/s bandwidth
L2 caché:   ~4 ns, 8-32 MB, ~500 GB/s bandwidth
L3 caché:   ~20 ns, 32-64 MB (compartida entre núcleos), ~200 GB/s
RAM:        ~100 ns, GB-TB, ~50-100 GB/s bandwidth
Disco NVMe: ~100 µs, ~7 GB/s bandwidth
```

**El problema de false sharing en código paralelo:**

```go
// Datos de workers adyacentes en la misma línea de caché
type Worker struct {
    count int64  // 8 bytes
}
workers := make([]Worker, 8)  // 8 * 8 = 64 bytes — ¡todo en una línea de caché!

// Cuando Worker 0 actualiza workers[0].count,
// invalida la línea de caché de Workers 1-7.
// Cada worker ve su propio dato inválido y debe re-leerlo.
// Con 8 workers: 7 invalidaciones por cada escritura de cualquier worker.
```

---

### Ejercicio 8.5.1 — Medir el impacto del false sharing

**Enunciado:** Implementa un microbenchmark que muestra el impacto del
false sharing de forma clara:

```go
// Con false sharing
type ConFlaso [8]struct{ val int64 }

// Sin false sharing (padding a 64 bytes)
type SinFalso [8]struct {
    val int64
    _   [56]byte  // padding
}
```

**Restricciones:** Con 8 goroutines actualizando su propio campo, la versión
sin false sharing debe ser al menos 3x más rápida. Verifica con GOMAXPROCS=8.

**Pista:** El false sharing solo ocurre con múltiples núcleos reales.
Con GOMAXPROCS=1, no hay false sharing (solo un núcleo). Con GOMAXPROCS=8,
los 8 núcleos invalidan la misma línea de caché constantemente — el overhead
de coherencia de caché domina el tiempo de ejecución.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.5.2 — Acceso a datos: row-major vs column-major

**Enunciado:** La multiplicación de matrices tiene dos formas de acceder
a los datos que tienen rendimiento radicalmente distinto por la caché:

```go
// Row-major (C, Go): los elementos de una fila son contiguos
// a[i][j] está en memoria en la posición i*cols + j

// Multiplicación ijk (acceso row-major para A, column-major para B)
func MultiplicarIJK(a, b, c [][]float64, n int) {
    for i := range c {
        for j := range c[i] {
            for k := 0; k < n; k++ {
                c[i][j] += a[i][k] * b[k][j]  // b[k][j] — saltos en memoria!
            }
        }
    }
}

// Multiplicación ikj (mejor localidad para B)
func MultiplicarIKJ(a, b, c [][]float64, n int) {
    for i := range a {
        for k := 0; k < n; k++ {
            for j := range b[k] {
                c[i][j] += a[i][k] * b[k][j]  // b[k][j] — acceso contiguo!
            }
        }
    }
}
```

**Restricciones:** Mide la diferencia de rendimiento para matrices de 512x512,
1024x1024, y 2048x2048. La diferencia debe ser al menos 2x para matrices grandes.
Implementa las 6 permutaciones (ijk, ikj, jik, jki, kij, kji) y rankéalas por rendimiento.

**Pista:** La mejor permutación para Go (row-major) es ikj o jki.
Para matrices que no caben en L3 caché (>64 MB), la diferencia se amplifica.
La multiplicación optimizada (BLAS, OpenBLAS) usa blocking/tiling para
maximizar la reutilización de datos en caché — un tema para Cap.09.

**Implementar en:** Go · Java · Rust · C

---

### Ejercicio 8.5.3 — Cache-oblivious algorithms

**Enunciado:** Los algoritmos "cache-oblivious" son eficientes en todas las
capas de caché sin necesitar saber el tamaño de ninguna.
Implementa merge sort cache-oblivious vs la versión estándar:

```go
// Versión estándar: divide en dos mitades
func MergeSortEstandar(a []int)

// Versión cache-oblivious: divide siempre en mitades recursivamente
// hasta llegar a arrays pequeños — el tamaño de corte no necesita
// ser ajustado al tamaño de caché
func MergeSortCacheOblivious(a []int)
```

**Restricciones:** Mide el rendimiento para arrays que caben en L1, L2, L3, y RAM.
La versión cache-oblivious debe ser comparable a la estándar en todos los casos
sin ajustar parámetros.

**Pista:** La diferencia con el merge sort estándar es sutil — ambos son O(N log N).
La ventaja del cache-oblivious es que la localidad de referencia se mantiene
para cualquier tamaño de caché, no solo para el que fue optimizado.
Para sistemas con cachés de distintos tamaños (cloud, mobile), esto importa más.

**Implementar en:** Go · Rust · C (para comparar con la implementación de glibc)

---

### Ejercicio 8.5.4 — Blocking para multiplicación de matrices

**Enunciado:** La multiplicación de matrices blockificada divide las matrices
en submatrices (bloques) que caben en caché, reduciendo los cache misses:

```go
func MultiplicarBlocking(a, b, c [][]float64, n, blockSize int) {
    for ii := 0; ii < n; ii += blockSize {
        for jj := 0; jj < n; jj += blockSize {
            for kk := 0; kk < n; kk += blockSize {
                // multiplicar el bloque a[ii:ii+bs][kk:kk+bs]
                // por b[kk:kk+bs][jj:jj+bs]
                // y acumular en c[ii:ii+bs][jj:jj+bs]
                for i := ii; i < min(ii+blockSize, n); i++ {
                    for k := kk; k < min(kk+blockSize, n); k++ {
                        for j := jj; j < min(jj+blockSize, n); j++ {
                            c[i][j] += a[i][k] * b[k][j]
                        }
                    }
                }
            }
        }
    }
}
```

**Restricciones:** Encuentra el blockSize óptimo experimentalmente para tu máquina.
La versión blocking debe ser al menos 2x más rápida que la versión ingenua
para matrices de 1024x1024.

**Pista:** El blockSize óptimo es el mayor que cabe en L1/L2 caché.
Para L1 de 32KB: 3 bloques (A, B, C) de blockSize×blockSize×8 bytes ≤ 32KB →
blockSize ≤ sqrt(32768/3/8) ≈ 37. Valores típicos: 32, 64, 128.

**Implementar en:** Go · Java · Rust · C

---

### Ejercicio 8.5.5 — Prefetching manual

**Enunciado:** El hardware hace prefetch automático para accesos secuenciales.
Para accesos con stride (saltos), el prefetch puede hacerse manualmente:

```go
// Acceso secuencial — hardware prefetch funciona bien
for i := range a { sum += a[i] }

// Acceso con stride — hardware prefetch falla
for i := 0; i < len(a); i += 16 { sum += a[i] }

// Con prefetch manual (instrucción PREFETCHT0):
// Cargar en caché el próximo elemento antes de necesitarlo
```

**Restricciones:** Implementa una función de suma con stride usando
`runtime/internal/sys.Prefetch` o assembly manual.
Mide la mejora para strides de 16, 64, y 256.

**Pista:** El prefetch manual solo ayuda cuando el acceso tiene un patrón
predecible. Para accesos completamente aleatorios, el prefetch no puede
anticipar qué cacheline será necesaria. Para strides regulares, el hardware
moderno en realidad puede detectar el patrón y hacer prefetch automático.

**Implementar en:** Go + Assembly · Rust (`std::intrinsics::prefetch_read_data`) · C

---

## Sección 8.6 — Work Stealing y Balanceo Dinámico

El balanceo estático (dividir el trabajo en chunks iguales antes de empezar)
funciona cuando todos los items tienen el mismo costo. Cuando los costos
varían, algunos workers terminan antes y quedan ociosos.

**Work stealing — el balanceo dinámico:**

```
Worker 0: [tarea A (10ms)] [tarea E (10ms)]  ← termina en 20ms
Worker 1: [tarea B (30ms)]                    ← termina en 30ms
Worker 2: [tarea C (10ms)] [roba de W1!]      ← roba tarea B a la mitad

Con work stealing:
  Worker 2 termina tarea C en 10ms
  Roba parte de la cola de Worker 1 (que tiene work pendiente)
  Todos terminan más cerca del tiempo óptimo
```

**Go's scheduler ya hace work stealing internamente (Cap.04 §4.1).**
Para el nivel de aplicación, el balanceo dinámico se implementa con
una cola compartida o con work stealing manual.

---

### Ejercicio 8.6.1 — Work stealing con deque

**Enunciado:** Implementa una deque (double-ended queue) thread-safe
para work stealing:

```go
type Deque[T any] struct {
    // ...
}

func (d *Deque[T]) PushFront(item T)         // el propietario agrega trabajo
func (d *Deque[T]) PopFront() (T, bool)      // el propietario toma trabajo
func (d *Deque[T]) PopBack() (T, bool)       // el ladrón roba trabajo
```

La asimetría (propietario por el frente, ladrón por el fondo) minimiza
la contención: el propietario y el ladrón raramente acceden al mismo extremo.

**Restricciones:** `PopFront` y `PushFront` deben ser O(1) amortizado.
`PopBack` (robo) puede tener contención con `PopFront` — usa CAS para resolverlo.

**Pista:** El algoritmo clásico de Arora, Blumofe y Plaxton (1998) usa
un array circular con dos punteros: `top` (ladrón, atómico) y `bottom`
(propietario, relajado). El CAS en `PopBack` resuelve el único caso
de contención: cuando el ladrón y el propietario intentan tomar el último elemento.

**Implementar en:** Go · Java · Rust · C

---

### Ejercicio 8.6.2 — Balanceo dinámico con work stealing

**Enunciado:** Implementa un sistema de procesamiento con work stealing
para trabajo heterogéneo (tareas de distinta duración):

```go
type SistemaWorkStealing struct {
    deques  []*Deque[Tarea]  // una por worker
    workers int
}

func (s *SistemaWorkStealing) Ejecutar(tareas []Tarea)
```

Compara con el balanceo estático (chunks iguales) para una distribución
de tareas donde el 10% tarda 10x más que el promedio (distribución Pareto).

**Restricciones:** Mide el tiempo de completitud y la utilización de cada worker.
Con work stealing, la utilización debe ser >90% para todos los workers.
Con balanceo estático, algunos workers pueden estar ociosos >30% del tiempo.

**Pista:** La distribución Pareto es común en sistemas reales: el 20% de los
documentos tiene el 80% del texto, el 20% de los usuarios hace el 80% de las
requests. El work stealing es la respuesta estándar a esta distribución.

**Implementar en:** Go · Java (`ForkJoinPool`) · Rust (`rayon`) · C# (`PLINQ`)

---

### Ejercicio 8.6.3 — Thread pool con afinidad de NUMA

**Enunciado:** En servidores multi-socket, los accesos a memoria de otro
socket son ~2-3x más lentos que a memoria local (NUMA — Non-Uniform Memory Access).
Un thread pool NUMA-aware asigna workers al socket donde están los datos.

```go
type PoolNUMA struct {
    nodos  int
    workers []*WorkerGroup  // un grupo por nodo NUMA
}

func (p *PoolNUMA) Submit(tarea Tarea, nodoPreferido int)
```

**Restricciones:** Implementa con `runtime.LockOSThread()` y afinidad de CPU
(usando `syscall` en Linux: `sched_setaffinity`). Mide la diferencia de
latencia entre accesos locales y remotos en tu máquina.

**Pista:** En una laptop o desktop con un solo socket, NUMA no aplica.
Para simular NUMA, usa `numactl --membind=1 --cpunodebind=0` en Linux
para forzar accesos cross-node. En cloud (AWS c5.metal, por ejemplo),
hay múltiples nodos NUMA y la diferencia es medible sin simulación.

**Implementar en:** Go (Linux) · C (para comparar con `libnuma`)

---

### Ejercicio 8.6.4 — Throttling dinámico según temperatura del CPU

**Enunciado:** Los CPUs modernos reducen su frecuencia cuando se sobrecalientan
(thermal throttling). Un sistema paralelo que lo ignora puede tener
rendimiento inconsistente.

Implementa un monitor de temperatura y un throttler dinámico:

```go
type ThrottlerTermico struct {
    maxTemp   float64  // Celsius — reducir workers si se supera
    workers   int
    activos   atomic.Int32
}

func (t *ThrottlerTermico) AjustarSegunTemperatura()
```

**Restricciones:** En Linux, la temperatura del CPU está en
`/sys/class/thermal/thermal_zone*/temp`.
Si la temperatura supera el umbral, reduce los workers activos.
Si baja, incrementa gradualmente.

**Pista:** El throttling térmico es relevante para benchmarks de larga duración
(>30 segundos). Los CPUs modernos (Intel, AMD) pueden bajar de 3.5 GHz a 1.5 GHz
bajo carga sostenida en laptops con refrigeración insuficiente.
Un benchmark que no controla la temperatura puede reportar resultados inconsistentes.

**Implementar en:** Go (Linux) · C (para herramientas de baseline)

---

### Ejercicio 8.6.5 — Comparar work stealing vs round-robin vs least-loaded

**Enunciado:** Implementa tres estrategias de distribución de trabajo y
compáralas para distintas distribuciones de tamaños de tarea:

1. **Round-robin**: asignar tareas secuencialmente a workers en ciclo
2. **Least-loaded**: asignar al worker con menos trabajo pendiente
3. **Work stealing**: workers toman trabajo de la cola global o roban de otros

Para distribuciones:
- Uniforme: todas las tareas duran lo mismo
- Bimodal: 50% duran 1ms, 50% duran 10ms
- Pareto: 80% duran 1ms, 20% duran 50ms

**Restricciones:** Mide: tiempo total de completitud, utilización promedio
de workers, varianza de carga entre workers, y overhead de la estrategia.

**Pista:** Round-robin es simple y funciona bien para distribución uniforme.
Least-loaded introduce overhead de sincronización para consultar las cargas.
Work stealing tiene overhead bajo pero latencia de respuesta variable.
No hay un ganador universal — el mejor depende de la distribución y del overhead aceptable.

**Implementar en:** Go · Java · Rust · C#

---

## Sección 8.7 — Paralelismo en la Práctica: Casos Reales

Los patrones abstractos de las secciones anteriores resuelven problemas reales.
Esta sección los conecta con casos concretos de producción.

---

### Ejercicio 8.7.1 — Procesamiento paralelo de CSV grande

**Enunciado:** Un archivo CSV de 10 GB con 100 millones de filas necesita
procesarse en menos de 60 segundos en una máquina de 8 cores.

```go
func ProcesarCSVParalelo(
    path string,
    transformar func([]string) (Registro, error),
    workers int,
) ([]Registro, error)
```

**Restricciones:** El archivo no debe cargarse entero en memoria — streaming.
La lectura del archivo es I/O-bound, el parsing es CPU-bound. Diseña el pipeline
para paralelizar el parsing sin parallelizar la lectura (que ya está saturada).

**Pista:** El cuello de botella cambia según el hardware:
- HDD: lectura es el cuello (~100 MB/s → 100s para 10GB)
- SSD: lectura puede ser comparable al parsing (~500MB/s → 20s)
- NVMe RAID: parsing puede ser el cuello (~3GB/s → 3s)
El diseño correcto: un goroutine lee, N goroutines parsean chunks leídos.

**Implementar en:** Go · Java · Python (`multiprocessing.Pool`) · Rust

---

### Ejercicio 8.7.2 — Renderizado de imágenes por tiles

**Enunciado:** El renderizado de una imagen (ray tracing, fractales, etc.)
puede dividirse en tiles independientes. Implementa el renderizado paralelo
de un fractal de Mandelbrot:

```go
func RenderMandelbrot(
    ancho, alto int,
    tileSize int,
    maxIter int,
    workers int,
) [][]int  // intensidad de cada píxel
```

**Restricciones:** Divide la imagen en tiles de `tileSize × tileSize` píxeles.
El trabajo por tile varía significativamente (algunos tiles están en el borde del
fractal — muy costosos; otros están fuera — calculados inmediatamente).
Usa work stealing para balancear el trabajo desigual.

**Pista:** El borde del Mandelbrot es donde pasan las iteraciones — puede ser
100x más costoso que el interior o el exterior. Con balanceo estático,
los workers con tiles en el borde terminan mucho después que los demás.
Con work stealing, el balanceo dinámico distribuye los tiles caros.

**Implementar en:** Go · Java · Rust · C#

---

### Ejercicio 8.7.3 — Compilador paralelo: procesar múltiples archivos

**Enunciado:** Un compilador que procesa múltiples archivos fuente puede
compilarlos en paralelo (cada archivo es independiente). Implementa la
fase de análisis léxico/sintáctico paralelo:

```go
type CompiladorParalelo struct {
    workers int
}

func (c *CompiladorParalelo) Compilar(archivos []string) ([]AST, []error)
```

**Restricciones:** Los errores de compilación de un archivo no deben afectar
a los otros. Si N archivos tienen errores, retornar todos los N errores.
El orden de los ASTs en el resultado debe corresponder al orden de los archivos
de entrada (aunque se procesen en orden diferente).

**Pista:** Este es exactamente lo que hace `go build` internamente — los paquetes
sin dependencias entre sí se compilan en paralelo. Los paquetes con dependencias
se compilan en orden topológico. El Ejercicio 8.7.3 solo cubre el caso
sin dependencias (todos los archivos son independientes).

**Implementar en:** Go · Java · Rust

---

### Ejercicio 8.7.4 — Búsqueda paralela en índice invertido

**Enunciado:** Un motor de búsqueda simple tiene un índice invertido:
`palabra → [documentos que la contienen]`. Para una query de N términos,
busca todos los documentos que contienen todos los términos.

```go
type IndiceInvertido struct {
    indice map[string][]int  // palabra → IDs de documentos
}

func (idx *IndiceInvertido) BuscarAND(terminos []string, workers int) []int
// Retorna documentos que contienen TODOS los términos
```

**Restricciones:** Busca cada término en paralelo, luego calcula la intersección.
Para queries de alta frecuencia (palabras comunes), el índice puede tener millones
de documentos por término — la intersección es el cuello de botella.

**Pista:** La estrategia óptima: ordenar los términos por frecuencia ascendente
(el término más raro primero). La intersección se hace de menos frecuente
a más frecuente — los conjuntos se reducen rápidamente.
Con paralelismo: buscar todos los términos en paralelo, luego intersección secuencial.

**Implementar en:** Go · Java · Python · Rust

---

### Ejercicio 8.7.5 — Benchmark completo: ¿cuándo paralelizar?

**Enunciado:** Para cada operación, determina empíricamente si el paralelismo
vale la pena y con cuántos workers:

1. Ordenar un array de 1M strings
2. Calcular SHA-256 de 10,000 archivos de 1MB
3. Parsear 100,000 documentos JSON de 10KB
4. Comprimir 1,000 archivos de 1MB con gzip
5. Calcular la multiplicación de 1,000 matrices de 100×100

**Restricciones:** Para cada operación, reporta:
- Tiempo secuencial (1 worker)
- Tiempo óptimo (N workers donde N maximiza el speedup)
- Speedup máximo observado
- Número de workers óptimo y por qué

**Pista:** Las operaciones 1 y 3 son probablemente memory-bound — saturan rápido.
La 2 y 4 dependen del throughput de I/O del disco. La 5 es compute-bound.
El número óptimo de workers no siempre es NumCPU — para operaciones memory-bound,
puede ser 2 o 4 aunque tengas 16 núcleos.

**Implementar en:** Go · Java · Rust · C#

---

## Resumen del capítulo

**La decisión de paralelizar:**

```
¿Es el trabajo CPU-bound?
    No → usar concurrencia (goroutines + I/O async) — el paralelismo no ayuda
    Sí → ¿cuál es la fracción secuencial?
           > 50% → no vale la pena (speedup máximo: 2x)
           10-50% → paralelismo moderado (speedup: 2-10x)
           < 10%  → alto paralelismo potencial (speedup: 10x+)
```

**Los números que importan:**

| Operación | Bound | Workers óptimos |
|---|---|---|
| Suma de arrays | Memory-bound | 2-4 |
| Multiplicación de matrices (pequeñas) | Cache-bound | NumCPU |
| SHA-256 de archivos | Compute + I/O | NumCPU |
| Parsing de JSON | Compute-bound | NumCPU |
| Compresión gzip | Compute-bound | NumCPU |
| Lectura de disco (HDD) | I/O-bound | 1-2 |
| Lectura de disco (NVMe) | I/O + Compute | 4-8 |

## La pregunta que guía el Cap.09

> El Cap.08 cubrió cómo usar múltiples núcleos dentro de un proceso.
> El Cap.09 sube un nivel: cómo diseñar algoritmos que escalen en
> múltiples máquinas — el puente entre el paralelismo local y los sistemas distribuidos.
> La pregunta central: ¿qué problemas requieren más de una máquina, y por qué?
