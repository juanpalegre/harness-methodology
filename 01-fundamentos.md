# 01 — Fundamentos del Harness Engineering

> No es un problema del modelo. Es un problema de configuración.  
> — HumanLayer

---

## La ecuación que cambia todo

```
Agente = Modelo + Harness
```

Un modelo de lenguaje, por sí solo, no es un agente. Es un motor de predicción de texto. Se convierte en agente cuando un **harness** le da estado, ejecución de herramientas, loops de feedback y restricciones aplicables.

El harness es **todo lo que no es el modelo**:

- System prompts, archivos de instrucciones, skill files
- Herramientas, servidores MCP, CLIs
- Infraestructura de ejecución (filesystem, sandbox, browser)
- Lógica de orquestación (subagentes, handoffs, routing de modelos)
- Hooks y middleware para ejecución determinística
- Observabilidad (logs, trazas, métricas de costo y latencia)

**Herramientas como Copilot en VS Code, Claude Code, Cursor, Codex, Aider, Cline — son todos harnesses.** El modelo debajo puede ser el mismo. El comportamiento que experimentás está dominado por lo que el harness hace.

---

## La evolución: de TDD a Harness Engineering

```
TDD:   Tests → Código
BDD:   Comportamiento → Tests → Código
SDD:   Spec → Agente → Código + Tests
HE:    Harness → Agente → Código + Tests + Feedback → Harness mejorado
```

| Disciplina | Artefacto primario | Quién ejecuta | Loop de mejora |
|---|---|---|---|
| **TDD** | Tests unitarios | Humano | Refactor manual |
| **BDD** | Especificaciones de comportamiento | Humano | Refinamiento con stakeholders |
| **SDD** | Spec del proyecto (`.context/`, instrucciones) | Agente guiado por spec | Actualización manual del spec |
| **Harness Engineering** | El harness completo (spec + hooks + tools + loops) | Agente + harness | **Ratchet automático**: cada error genera una regla |

SDD es un **subconjunto** de Harness Engineering. SDD te enseña a preparar el contexto. HE agrega enforcement, observabilidad, evaluación adversarial, y la disciplina de evolución continua del sistema.

---

## El reframe "Skill Issue"

El insight más importante: **la mayoría de fallas de agentes no son problemas del modelo**.

Datos concretos:
- En Terminal Bench 2.0, Claude Opus 4.6 dentro de Claude Code obtiene un score **significativamente menor** que el mismo modelo en un harness custom optimizado
- El equipo de Viv Trivedy movió un coding agent del **Top 30 al Top 5** cambiando **solamente el harness**
- Los modelos se post-entrenan con harnesses específicos; moverlos a un harness diferente (con mejores herramientas para tu codebase, un prompt más ajustado, y back-pressure más firme) puede desbloquear capacidad que el harness original dejaba en el piso

Esto invierte la narrativa de "esperá al próximo modelo". **La brecha entre lo que los modelos pueden hacer y lo que ves que hacen es, en gran medida, una brecha de harness.**

---

## Los 3 pilares agnósticos

Todo harness efectivo se sostiene sobre tres pilares, independientes de la plataforma:

### Pilar 1: Contexto

> Un agente sin contexto es un agente que alucina.

El contexto es toda información que el agente necesita para operar dentro de los límites de tu proyecto:

- **Arquitectura**: qué patrones usa el proyecto, qué capas existen, qué está prohibido
- **Convenciones**: cómo se nombran las cosas, cómo se manejan errores, cómo se formatean commits
- **Stack**: qué tecnologías se usan, qué dependencias hay, qué comandos existen
- **Instrucciones**: qué puede hacer el agente, qué no puede hacer, cómo debe comportarse

El contexto se implementa como **archivos en el repositorio** que se inyectan en el prompt del agente en cada turno. Es el equivalente a un onboarding para un desarrollador junior: le das la documentación antes de que toque código.

### Pilar 2: Enforcement

> La diferencia entre "le dije al agente que haga X" y "el sistema fuerza X".

El enforcement es la capa de ejecución determinística: cosas que **no dependen de que el modelo se acuerde**, sino que se fuerzan programáticamente:

- **Hooks**: scripts que se ejecutan en puntos específicos del lifecycle (pre-commit, post-edit, on-session-start)
- **CI/CD**: validación automática de cada cambio (build, tests, lint, seguridad)
- **Branch protection**: reglas que impiden merge sin aprobación
- **Allowlists**: listas de herramientas/servicios/servidores permitidos

El principio: **éxito es silencioso, fallo es verboso**. Si el typecheck pasa, el agente no escucha nada. Si falla, el error se inyecta en el loop y el agente se autocorrige.

### Pilar 3: Observabilidad

> Si no podés ver qué hizo el agente, no podés mejorarlo.

La observabilidad es la capacidad de reconstruir qué hizo el agente, por qué, y cuál fue el resultado:

- **Commits** como checkpoints de progreso
- **Pull/Merge Requests** como registro de decisiones
- **Logs de CI/CD** como evidencia de validación
- **Trazas** de ejecución del agente (qué herramientas usó, qué generó, qué descartó)
- **Métricas**: tasa de aprobación, merge rate, tiempo por tarea, costo por ejecución

Sin observabilidad, el harness es una caja negra. Con observabilidad, cada ejecución es un dato que alimenta la mejora del harness.

---

## El principio más importante: el Ratchet

> Cada error del agente se convierte en una regla permanente.

El **ratchet** (trinquete) es el mecanismo por el cual el harness **solo se mueve hacia adelante**:

1. El agente comete un error
2. Analizás la causa raíz
3. Agregás una regla (instrucción), un hook (enforcement), o un test (validación)
4. Ese error no puede volver a ocurrir

```
Error observado → Diagnóstico → Regla/Hook/Test → Harness mejorado
                                                        ↓
                                      Próxima ejecución ya no tiene ese error
```

**Solo agregás constraints cuando ves un fallo real. Solo los removés cuando un modelo más capaz los hace redundantes.** Cada línea en un buen archivo de instrucciones debería ser trazable a un error específico que pasó.

Esto es lo que hace que harness engineering sea una **disciplina** y no un framework: el harness correcto para tu codebase está determinado por tu historial de fallos. No se descarga de un template.

> Los templates (como agentic-init) te dan la **estructura**. El ratchet te da el **contenido**.

---

## El harness como sistema vivo

Un harness no es un archivo de configuración que se escribe una vez. Es un sistema que co-evoluciona con el modelo y con el proyecto:

```
Harness diseñado para Modelo v1
        ↓
Modelo v2 sale (más capaz)
        ↓
Partes del harness ya no son load-bearing → REMOVER
        ↓
Nuevas capacidades del modelo desbloquean tareas más complejas
        ↓
Nuevas tareas tienen nuevos failure modes → AGREGAR scaffolding
        ↓
Harness evolucionado para Modelo v2
```

Ejemplo real de Anthropic: cuando pasaron de Opus 4.5 a Opus 4.6, pudieron **eliminar** toda la infraestructura de context resets porque el nuevo modelo ya no sufría "context anxiety". Pero a cambio, tuvieron que **agregar** scaffolding para tareas más complejas que antes eran inalcanzables.

> Cada componente del harness codifica una suposición sobre lo que el modelo **no puede hacer solo**. Cuando el modelo mejora, esa suposición se vuelve obsoleta y el componente debería salir.

---

## Resumen

| Concepto | Definición |
|---|---|
| **Harness** | Todo lo que no es el modelo: prompts, tools, hooks, sandbox, loops, observabilidad |
| **Harness Engineering** | La disciplina de diseñar, operar y evolucionar ese sistema |
| **Skill Issue** | La mayoría de fallas son de configuración, no del modelo |
| **Ratchet** | Cada error → regla permanente; el harness solo avanza |
| **3 Pilares** | Contexto + Enforcement + Observabilidad |
| **SDD ⊂ HE** | SDD cubre el pilar de contexto; HE cubre los tres |
| **Sistema vivo** | El harness co-evoluciona con el modelo y el proyecto |
