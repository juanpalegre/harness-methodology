# Cheat Sheet — Harness Engineering

> Referencia rápida. Para profundizar, ir al módulo correspondiente.

---

## La ecuación

```
Agente = Modelo + Harness
Harness = Contexto + Enforcement + Observabilidad
```

---

## Componentes del harness

| Componente | Qué es | Ejemplo |
|---|---|---|
| **Instrucciones** | Archivos de texto inyectados en cada turno | `.ai/instructions.md`, `.context/` |
| **Herramientas** | Capacidades externas que el agente invoca | Filesystem, terminal, APIs, MCP servers |
| **Sandbox** | Entorno aislado de ejecución | Container, devcontainer, VM efímera |
| **Hooks** | Scripts en lifecycle points | Post-edit lint, pre-commit test, block destructive |
| **Loops** | Mecanismos de continuación y feedback | ReAct loop, Ralph loop, back-pressure |
| **Subagentes** | Agentes especializados coordinados | Planner, Generator, Evaluator, Reviewer |
| **Memoria** | Estado persistente entre sesiones | lessons.md, instrucciones, plan files |
| **Observabilidad** | Reconstrucción de qué hizo el agente | Commits, MR/PRs, logs, métricas |

---

## El Ratchet — regla de oro

```
Error → Diagnóstico → Regla/Hook/Test → Harness mejorado → Error no se repite
```

- Cada línea de instrucciones trazable a un fallo real
- < 60 líneas en archivo de instrucciones
- 3 niveles: sugerencia (instrucción) → validación (CI) → bloqueo (hook)
- Solo remover constraints cuando el modelo los hace redundantes

---

## Memoria — jerarquía de precedencia

```
1. Prompt actual (máxima prioridad)
2. Instrucciones locales (directorio)
3. Instrucciones globales (repo)
4. Memoria del servicio
5. Inferencia del código (mínima prioridad)
```

---

## Context Engineering — decisión rápida

| Problema | Técnica | Cuándo |
|---|---|---|
| Contexto se llena | **Compaction** | Sesiones largas (1-2 horas) |
| Tool output gigante | **Offloading** | Logs > 100 líneas |
| Muchos tools/skills | **Progressive disclosure** | > 10 herramientas |
| Sesión muy larga | **Context reset** | > 3 horas o tareas multi-sesión |
| Agente se desvía | **Plan file + anclaje** | Tareas > 10 pasos |
| Agente "termina" antes | **Done-condition explícita** | Siempre |

---

## Evaluación — 4 categorías de causa raíz

| Categoría | Señal típica | Solución típica |
|---|---|---|
| **Razonamiento** | Inventa APIs, lógica invertida | Mejorar instrucciones, dar ejemplos |
| **Herramientas** | Tool equivocada, parámetros mal | Mejorar tool descriptions, restringir |
| **Contexto** | No sabe convenciones, info obsoleta | Actualizar `.context/`, instrucciones |
| **Ambiente** | Permisos, deps faltantes, timeouts | Mejorar sandbox setup |

---

## Orquestación — 4 patrones

| Patrón | Cuándo | Riesgo principal |
|---|---|---|
| **Fan-out** | Tareas independientes | Merge conflicts |
| **Pipeline** | Etapas secuenciales | Eslabón débil propaga |
| **Orquestador** | Descomposición inteligente | Overhead de coordinación |
| **Peer review** | Siempre (mínimo viable) | — (bajo costo, alto retorno) |

---

## Autonomía — decision tree

```
¿Es reversible Y de bajo riesgo en las 3 dimensiones?
├── SÍ → Autónomo (auto-format, typos, docs)
└── NO
    ├── ¿Es implementación acotada con tests?
    │   └── SÍ → Semi-autónomo (MR/PR + review)
    ├── ¿Toca infra, datos sensibles, o producción?
    │   └── SÍ → Supervisado (plan + aprobación manual)
    └── ¿Es auditoría o análisis?
        └── SÍ → Restringido (solo lectura)
```

---

## Protección — 4 capas

```
1. PREVENCIÓN   → Allowlists, mínimo privilegio, firewall
2. DETECCIÓN    → Branch protection, SAST, secret scanning
3. SUPERVISIÓN  → Code review, CODEOWNERS, environment gates
4. AUDITORÍA    → VCS history, MR/PR timeline, pipeline logs
```

---

## Estructura mínima de un harness

```
proyecto/
├── .ai/
│   ├── instructions.md           # Reglas (< 60 líneas, ganadas con fallos)
│   └── agents/
│       ├── planner.md            # Descompone tareas antes de codear
│       ├── reviewer.md           # Revisa código, no re-escribe
│       ├── evaluator.md          # Evalúa output contra criterios
│       └── tester.md             # Genera tests
├── .context/
│   ├── architecture.md           # ADRs y decisiones de diseño
│   ├── conventions.md            # Naming, error handling, estilo
│   └── stack.md                  # Stack, deps, comandos
├── tasks/
│   ├── todo.md                   # Plan de trabajo visible
│   └── lessons.md                # Ratchet: errores → reglas → enforcement
└── [pipeline config]             # CI/CD según plataforma
```

---

## Anti-patrones

| Anti-patrón | Por qué es malo | Alternativa |
|---|---|---|
| **Sobre-autonomía** | Agente con merge directo sin review | Semi-autónomo siempre |
| **"Esperá al próximo modelo"** | El gap es de harness, no de modelo | Aplicar ratchet hoy |
| **Instrucciones kilométricas** | Más reglas = menos atención por regla | < 60 líneas, ganadas |
| **Auto-evaluación** | El generador sesga positivo | Evaluador separado |
| **Sin memory** | El agente repite errores en cada sesión | lessons.md + instrucciones |
| **Harness estático** | No evoluciona con el modelo ni el proyecto | Ratchet + prune periódico |
| **Approval fatigue** | Demasiadas gates → nadie lee | Gates proporcionales al riesgo |
| **Vibe coding en producción** | Sin spec, sin guardrails, sin review | SDD + hooks + CI |

---

## Checklist de setup

- [ ] `.ai/instructions.md` con reglas mínimas del proyecto
- [ ] `.context/` con architecture, conventions, stack
- [ ] `tasks/lessons.md` inicializado (vacío, se llena con ratchet)
- [ ] `tasks/todo.md` como superficie de tracking
- [ ] Al menos un agente especializado (reviewer o planner)
- [ ] CI/CD pipeline que ejecuta build + test + lint en cada MR/PR
- [ ] Branch protection: no merge sin aprobación + CI verde
- [ ] Hook post-edit: lint/typecheck (si el harness lo soporta)
- [ ] Hook pre-commit: tests (si el harness lo soporta)
