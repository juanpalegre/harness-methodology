# 06 — Orquestación Multiagente

> El patrón más natural no es un agente que hace todo, sino múltiples agentes con contextos aislados, coordinados por el harness.

---

## Los 4 patrones de orquestación

### 1. Fan-out (Paralelo)

Múltiples agentes trabajan en tareas independientes simultáneamente. El más eficiente cuando las tareas no comparten estado.

```
         Tarea grande
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Agente A  Agente B  Agente C
 (auth)    (UI)      (API)
    │         │         │
    ▼         ▼         ▼
  MR/PR A   MR/PR B   MR/PR C
    │         │         │
    └─────────┼─────────┘
              ▼
         Integración
```

**Cuándo usar**: tareas descomponibles en unidades independientes (features distintas, módulos separados, microservicios).

**Riesgo principal**: conflictos de merge si dos agentes tocan los mismos archivos.

**Mitigación**: descomponer a nivel de archivo/módulo; definir boundaries claros antes de lanzar.

### 2. Pipeline (Secuencial)

Un agente pasa su output al siguiente en cadena. Cada eslabón tiene un rol específico.

```
Spec
  │
  ▼
Planner → Plan detallado
              │
              ▼
          Generator → Código implementado
                          │
                          ▼
                      Evaluator → Feedback/Aprobación
                                      │
                              ┌───────┴───────┐
                              │               │
                           APRUEBA         RECHAZA
                              │               │
                              ▼               ▼
                          Siguiente      Generator
                           sprint        re-trabaja
```

**Cuándo usar**: cuando cada etapa necesita contexto especializado y el output de una alimenta a la siguiente.

**Riesgo principal**: el pipeline es tan fuerte como su eslabón más débil. Un planner que genera specs vagos produce código vago.

### 3. Orquestador Central

Un agente coordinador analiza la tarea, la descompone, delega a workers especializados, y consolida los resultados.

```
         Tarea compleja
              │
              ▼
     ┌────────────────┐
     │  ORQUESTADOR   │
     │  (coordinador) │
     └───────┬────────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
 Worker 1  Worker 2  Worker 3
    │        │        │
    ▼        ▼        ▼
 Resultado  Resultado Resultado
    │        │        │
    └────────┼────────┘
             ▼
     ┌────────────────┐
     │  ORQUESTADOR   │
     │  (consolida)   │
     └────────────────┘
```

**Cuándo usar**: tareas donde la descomposición misma requiere inteligencia, o donde los workers necesitan contextos muy diferentes.

**Riesgo principal**: overhead del orquestador. Si la coordinación cuesta más que la ejecución, no vale la pena.

### 4. Peer Review

Un agente implementa, otro revisa. El patrón más simple de multi-agente y el más natural para SDLC.

```
Agente Generator → Código + PR
                        │
                        ▼
                  Agente Reviewer
                        │
                ┌───────┴───────┐
                │               │
             APRUEBA         COMENTA
                │               │
                ▼               ▼
             Merge         Generator
                           incorpora
                           feedback
```

**Cuándo usar**: siempre, como mínimo. Es el patrón de menor costo y mayor retorno.

**Implementación**: el reviewer es un agente con instrucciones de revisión (qué buscar, qué marcar como blocker, qué ignorar). No re-escribe código — señala problemas.

---

## Aislamiento

El principio fundamental de multi-agente es que **cada agente opera en un contexto aislado**:

| Mecanismo de aislamiento | Qué previene |
|---|---|
| **Ramas separadas** | Un agente no sobreescribe el trabajo de otro |
| **MR/PRs independientes** | Cada agente = su propia puerta de revisión |
| **CI/CD aislado** | Cada MR/PR tiene su propia ejecución de pipeline |
| **Contextos de ejecución separados** | No comparten memoria/archivos durante ejecución |

**Implementación agnóstica**:
```
repo/
├── feature/agent-1-auth       ← rama del agente 1
├── feature/agent-2-ui         ← rama del agente 2
└── feature/agent-3-api        ← rama del agente 3

Cada rama → MR/PR independiente → CI independiente → Review independiente
```

---

## Los 3 tipos de conflictos

### 1. Cambios superpuestos (Merge conflicts)

Dos agentes modifican los mismos archivos.

```
Agente A modifica → src/config.ts  ← Agente B modifica
                        │
                        ▼
                  MERGE CONFLICT
```

**Resolución**: rebase (si los cambios son compatibles) → merge manual (si requiere decisión) → arbitraje humano.

### 2. Esfuerzo duplicado

Dos agentes implementan funcionalidad equivalente sin saberlo.

**Resolución**: detectar en code review → elegir la mejor implementación → descartar la otra.

**Prevención**: descomponer a nivel de archivo/módulo con boundaries explícitos.

### 3. Salidas contradictorias

Un agente implementa con un enfoque (ej: REST), otro con otro (ej: GraphQL) para el mismo endpoint.

**Resolución**: arbitraje humano — requiere una decisión arquitectónica.

**Prevención**: definir estilo/enfoque en el spec antes de lanzar los agentes.

---

## Circuit Breaker

Patrón para prevenir que un agente entre en loop infinito o gaste recursos indefinidamente:

```
Agente intenta resolver tarea
        │
        ├── Éxito → Continuar
        │
        └── Fallo
              │
              ├── Intento 1: reintentar con contexto adicional
              ├── Intento 2: reintentar con enfoque diferente
              └── Intento 3: CIRCUIT BREAKER → detener + reportar + escalar a humano
```

**Implementación**:
- Definir número máximo de reintentos (ej: 3)
- Definir timeout máximo por tarea
- Definir condiciones de "stuck" (ej: no progreso en N minutos, mismo error 3 veces)
- Al disparar el circuit breaker: logging detallado + notificación

---

## Los 3 estados de fallo

| Estado | Qué pasó | Acción |
|---|---|---|
| **Fallida** | Error irrecuperable; el agente se detuvo | Diagnosticar causa raíz, ajustar harness, reintentar |
| **Parcial** | Completó parte pero no todo | Evaluar lo completo, crear nueva tarea para lo faltante |
| **Atascada** | Loop sin progreso o trabajo circular | Circuit breaker → escalar a humano |

---

## Observabilidad multiagente

Cuando hay múltiples agentes, la observabilidad se vuelve crítica. Los 3 pilares:

### 1. Trazabilidad

Poder seguir el flujo de trabajo entre agentes:

```
Issue #42 
  → Agent-1 creó rama feature/agent-1-auth (commit abc123)
  → Agent-1 creó MR/PR #15
  → Agent-2 creó rama feature/agent-2-ui (commit def456)
  → Agent-2 creó MR/PR #16 (depende de #15)
```

### 2. Registro

Cada agente documenta sus decisiones:

```
MR/PR description:
  - Qué implementó
  - Qué decisiones tomó y por qué
  - Qué dependencias tiene con otros MR/PRs
  - Qué está fuera de scope

Commits:
  - [agent-1] feat: implementar login con JWT
  - [agent-1] fix: manejar refresh token expirado
```

### 3. Artefactos

Evidencia tangible del trabajo:

| Artefacto | Propósito |
|---|---|
| Test results | Verificar correctitud |
| Build logs | Verificar que compila |
| Coverage reports | Verificar cobertura |
| Pipeline timeline | Verificar secuencia de ejecución |
| Audit log | Compliance y trazabilidad legal |

---

## Lifecycle del agente

```
Creación → Configuración → Activo → Actualización → Retiro
```

| Fase | Qué pasa |
|---|---|
| **Creación** | Se define el rol, instrucciones, herramientas |
| **Configuración** | Se conecta al repo, se le dan permisos |
| **Activo** | Ejecuta tareas, genera output |
| **Actualización** | Se actualizan instrucciones/tools (aplica en próxima ejecución) |
| **Retiro** | Se desactiva, **pero se preserva todo el historial** |

---

## Resumen

| Concepto | Clave |
|---|---|
| **4 patrones** | Fan-out (paralelo), Pipeline (secuencial), Orquestador (central), Peer Review |
| **Aislamiento** | Ramas + MR/PRs + CI + contextos separados por agente |
| **3 conflictos** | Superpuestos (merge), duplicados (esfuerzo), contradictorios (enfoque) |
| **Circuit breaker** | Max reintentos + timeout + detección de loop → escalar |
| **3 estados fallo** | Fallida, parcial, atascada |
| **Observabilidad** | Trazabilidad + registro + artefactos |
