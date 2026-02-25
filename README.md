# Repositorio de Concurrencia y Paralelismo

> Compañero del [Repositorio de Algoritmos y Sistemas](https://github.com/nehuenlabs/estudios-algoritmos-recursividad).
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

## 🎉 Repositorio completo

**Total: 595 ejercicios en 17 capítulos**

| Parte | Capítulos | Ejercicios | Tema |
|---|---|---|---|
| 1 — Concurrencia | Cap.01-07 | 245 | Fundamentos en Go |
| 2 — Paralelismo | Cap.08-12 | 175 | Hardware + estructuras lock-free |
| 3 — Lenguajes | Cap.13-16 | 140 | Rust, Java, Python, C# |
| 4 — Producción | Cap.17 | 35 | Observabilidad, resilience, operabilidad |

---

## Prerrequisitos

El repositorio asume familiaridad con:
- Recursión, estructuras de datos y algoritmos básicos [Cap.01–15 del repo de algoritmos](https://github.com/nehuenlabs/estudios-algoritmos-recursividad)
- Un lenguaje de la lista de referencia a nivel intermedio
- El modelo de ejecución básico: qué es un proceso, un hilo, y una goroutine

No asume conocimiento previo de concurrencia — ese es el propósito del Cap.01.

---
