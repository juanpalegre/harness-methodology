# 02 — Anatomía del Harness

> Empezá por el comportamiento que querés. Derivá el componente del harness que lo entrega.  
> — Viv Trivedy

---

## Mapeo comportamiento → componente

La forma más útil de diseñar un harness es preguntarse: **¿qué quiero que el agente haga (o deje de hacer)?** y derivar el componente que lo garantiza.

| Comportamiento deseado | Componente del harness |
|---|---|
| Trabajar con datos reales de forma durable | Filesystem + Control de versiones |
| Escribir y ejecutar código | Bash / terminal / code execution |
| Ejecutar de forma segura sin romper nada | Sandbox con aislamiento |
| Recordar conocimiento entre sesiones | Archivos de memoria + instrucciones persistentes |
| No degradarse en tareas largas | Compaction + offloading + context resets |
| Completar trabajo de largo aliento | Loops de continuación + planificación + verificación |
| No hacer cosas prohibidas | Hooks de enforcement |
| Operar dentro del stack del proyecto | Instrucciones de contexto (spec) |
| Coordinarse con otros agentes | Orquestación multiagente |

Si no podés nombrar el comportamiento que un componente existe para entregar, probablemente no debería estar ahí.

---

## Taxonomía completa de componentes

### 1. System Prompts e Instrucciones

**Qué son**: archivos de texto que se inyectan en el prompt del agente en cada turno. Definen comportamiento, restricciones, convenciones y contexto del proyecto.

**Jerarquía típica**:
```
Organización (políticas globales)
    └── Repositorio (instrucciones del proyecto)
        └── Directorio (instrucciones específicas por área)
            └── Agente individual (restricciones por rol)
```

**Implementación agnóstica**:
```
repo/
├── .ai/instructions.md            # Instrucciones globales del repo
├── .ai/agents/
│   ├── planner.md                  # Agente planificador
│   ├── reviewer.md                 # Agente revisor
│   └── tester.md                   # Agente tester
├── .context/
│   ├── architecture.md             # Decisiones de arquitectura
│   ├── conventions.md              # Convenciones de código
│   └── stack.md                    # Stack tecnológico
└── src/
    └── .ai/instructions.md         # Instrucciones específicas para /src/
```

**Regla clave**: mantener instrucciones en **menos de 60 líneas**. Cada línea compite por atención del modelo. Más reglas hacen que cada regla importe menos. **Checklist de piloto, no guía de estilo.**

### 2. Herramientas (Tools)

**Qué son**: capacidades externas que el agente puede invocar. Sin herramientas, es un chatbot. Con herramientas, es un actor autónomo.

**4 categorías** (agnósticas):

| Categoría | Ejemplos | Cuándo la agrega el harness |
|---|---|---|
| **Built-in** | Lectura/escritura de archivos, terminal, búsqueda de código | Siempre disponibles |
| **Extensión** | APIs externas, bases de datos, servicios cloud (via MCP u otro protocolo) | Cuando el agente necesita datos/servicios externos |
| **Workflow** | Pipelines de CI/CD, checks de calidad, deployments | Cuando el agente interactúa con el ciclo de entrega |
| **Contexto** | Archivos de instrucciones, documentación, memoria | Siempre disponibles |

**Principio de diseño**: cada tool tiene un nombre, descripción y schema que se estampan en el prompt cada request. **Diez herramientas enfocadas superan a cincuenta superpuestas**, porque el modelo puede mantener el menú completo en su cabeza.

**Riesgo de seguridad**: las descripciones de herramientas pueblan el prompt, así que cualquier servidor de herramientas externo que instalés es texto confiado que el modelo va a leer. Un servidor malicioso puede inyectar instrucciones en tu agente antes de que hayas escrito nada.

### 3. Sandbox (Entorno de Ejecución)

**Qué son**: entornos aislados donde el agente ejecuta código de forma segura. El modelo no configura su entorno de ejecución; decidir dónde corre el agente, qué está disponible, y cómo verifica su trabajo son decisiones del harness.

**Características de un buen sandbox**:
- Runtimes pre-instalados (lenguaje, paquetes, test frameworks)
- CLI de versionado y tests
- Browser headless para interacción web
- Allow-list de comandos
- Aislamiento de red
- Creación/destrucción on-demand

**Implementación por contexto**:

| Contexto | Sandbox |
|---|---|
| Desarrollo local | Container Docker, devcontainer, VM local |
| CI/CD | Runner efímero (pipeline agent, GitHub Actions runner, GitLab runner) |
| Agente cloud | VM sandbox efímera provista por el servicio |

### 4. Hooks (Capa de Enforcement)

**Qué son**: scripts que se ejecutan en puntos específicos del lifecycle del agente. Son lo que separa "le dije al agente que haga X" de "el sistema fuerza X".

**Puntos de lifecycle comunes**:

| Momento | Hook típico |
|---|---|
| Inicio de sesión | Cargar instrucciones + memoria + reglas activas |
| Post-edición de archivo | Ejecutar linter + typecheck; inyectar errores al agente |
| Pre-commit | Grep por patrones prohibidos (`.skip(`, `xit(`, `TODO:`, `console.log`) |
| Pre-push | Ejecutar suite de tests completa |
| Pre-merge/PR | Validar que CI pasó, que hay reviews suficientes |
| Comando destructivo | Bloquear `rm -rf`, `git push --force`, `DROP TABLE` |

**Principio**: éxito es silencioso, fallo es verboso. Si typecheck pasa → el agente no escucha nada. Si falla → el error se inyecta en el loop y el agente se autocorrige. Esto hace el feedback loop casi gratis en el caso común y directamente accionable cuando algo sale mal.

### 5. Loops de Feedback y Continuación

**Qué son**: mecanismos que mantienen al agente trabajando más allá de una sesión única.

**El loop agéntico básico** (ReAct):
```
Razonar → Actuar (tool call) → Observar resultado → Repetir
```

**Loop de continuación (Ralph Loop)**:
Un hook intercepta el intento del modelo de terminar y re-inyecta el prompt original en una ventana de contexto fresca, forzando al agente a continuar contra un objetivo de completitud. Cada iteración arranca limpia pero lee estado de la anterior via filesystem.

**Back-pressure**: cuando un hook detecta un fallo (tests rotos, typecheck fallido), inyecta el error de vuelta al loop. El agente se autocorrige sin intervención humana.

### 6. Subagentes y Orquestación

**Qué son**: agentes especializados que operan en contextos separados, coordinados por el harness.

**Separaciones comunes**:
- **Planner** / **Generator** / **Evaluator** — planifica, implementa, evalúa por separado
- **Coder** / **Reviewer** — uno implementa, otro revisa
- **Coordinador** / **Workers** — uno descompone, muchos ejecutan en paralelo

La separación entre generación y evaluación es clave: los agentes consistentemente sesgan positivo cuando evalúan su propio trabajo. Un evaluador externo, calibrado para ser escéptico, es mucho más tratable que hacer que un generador sea crítico de sí mismo.

### 7. Observabilidad

**Qué es**: la capacidad de reconstruir qué hizo el agente, por qué y cuál fue el resultado.

**3 pilares de observabilidad**:

| Pilar | Qué captura | Implementación |
|---|---|---|
| **Trazabilidad** | Flujo de ejecución entre agentes y herramientas | Logs estructurados, event bus |
| **Registro** | Decisiones y acciones tomadas | Commits, MR/PR descriptions, comments |
| **Artefactos** | Evidencia tangible del trabajo | Test results, build logs, coverage reports |

---

## Diagrama de un harness completo

```
┌─────────────────────────────────────────────────────────────┐
│                       HARNESS                                │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Instrucciones│  │  Herramientas│  │  Observabil. │       │
│  │  (.ai/, .ctx/)│  │  (tools/MCP) │  │  (logs/VCS)  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                  │               │
│         ▼                 ▼                  ▼               │
│  ┌──────────────────────────────────────────────────┐       │
│  │              LOOP AGÉNTICO                        │       │
│  │                                                   │       │
│  │   Razonar → Actuar → Observar → Repetir          │       │
│  │       ▲                              │            │       │
│  │       │         BACK-PRESSURE        │            │       │
│  │       └──────────────────────────────┘            │       │
│  └──────────────────────────────────────────────────┘       │
│         │                 │                                  │
│         ▼                 ▼                                  │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │    Sandbox    │  │    Hooks     │                         │
│  │  (ejecución)  │  │ (enforcement)│                         │
│  └──────────────┘  └──────────────┘                         │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │           SUBAGENTES                              │       │
│  │  Planner │ Generator │ Evaluator │ Reviewer       │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │           MEMORIA                                 │       │
│  │  Sesión (corto plazo) │ Instrucciones (largo)     │       │
│  │  Filesystem (externo) │ Lessons (ratchet)         │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

---

## Cuándo agregar cada componente

No todo harness necesita todos los componentes. Empezá simple y agregá complejidad solo cuando ves un fallo real:

| Nivel | Componentes | Para qué |
|---|---|---|
| **Mínimo** | Instrucciones + 1 tool (filesystem) | Proyecto personal, prototipos |
| **Proyecto serio** | + Hooks (lint/test) + CI/CD + Memoria | Equipo chico, código que se mantiene |
| **Producción** | + Sandbox + Subagentes + Observabilidad | Equipo grande, código en producción |
| **Empresa** | + Allowlists + Auditoría + Multi-agente | Organización con compliance |

> Encontrar la solución más simple posible y solo aumentar complejidad cuando se necesita.  
> — Anthropic, "Building Effective Agents"
