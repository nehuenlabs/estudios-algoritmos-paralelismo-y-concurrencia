# Repositorio de Concurrencia y Paralelismo

> Compañero del [Repositorio de Algoritmos y Sistemas](../algoritmos/).
> Ese repositorio termina donde este empieza: en el borde donde los algoritmos
> puros se encuentran con múltiples agentes ejecutando simultáneamente.

---

## Estructura

```
PARTE 1 — Fundamentos de concurrencia (Cap.01–07)
  Los problemas y las soluciones primitivas.
  Base obligatoria para todo lo demás.

PARTE 2 — Paralelismo (Cap.08–13)
  Múltiples núcleos, división de trabajo, rendimiento.

PARTE 3a — Entrevistas técnicas (Cap.14–17)
  Patrones que aparecen en entrevistas FAANG y similares.
  Go y Java como lenguajes de referencia.

PARTE 3b — Producción (Cap.18–21)
  Observabilidad, debugging, code review, resiliencia.

PARTE 4 — Sistemas distribuidos (Cap.22–23)
  De memoria compartida a paso de mensajes.
  El puente con los sistemas del Cap.17 del repo de algoritmos.
```

---

## Lenguajes de referencia

**Go** es el lenguaje principal de este repositorio. Tiene concurrencia integrada
en el lenguaje (goroutines, canales, select) y el mejor detector de races del ecosistema.

**Rust** aparece donde la seguridad en compilación es el punto — el compilador
rechaza data races, no las detecta en runtime.

**Java** y **Python** están presentes para entrevistas en esos ecosistemas.

---

## Progreso

| Archivo | Estado | Ejercicios |
|---|---|---|
| `01_cinco_problemas_fundamentales.md` | ✅ Listo | 35 ejercicios (7 secciones × 5) |

| `02_herramientas_sincronizacion.md` | ✅ Listo | 35 ejercicios (7 secciones × 5) |

| `03_patrones_clasicos.md` | ✅ Listo | 35 ejercicios (7 secciones × 5) |

**Total: 105 ejercicios**

---

## Prerrequisitos

El repositorio asume familiaridad con:
- Recursión, estructuras de datos y algoritmos básicos (Cap.01–15 del repo de algoritmos)
- Un lenguaje de la lista de referencia a nivel intermedio
- El modelo de ejecución básico: qué es un proceso, un hilo, y una goroutine

No asume conocimiento previo de concurrencia — ese es el propósito del Cap.01.

---

## Cómo usar este repositorio

El mismo criterio que el repositorio de algoritmos:
no uses IA para generar soluciones. El objetivo es ver el problema,
entender por qué ocurre, y razonar sobre la solución.

Para el Cap.01 en particular: antes de leer la sección de teoría,
intenta responder por qué el contador del ejemplo inicial no siempre da 2000.
Si puedes explicarlo sin leer la sección, el resto del capítulo será más ligero.
