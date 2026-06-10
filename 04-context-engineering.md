# 04 — Context Engineering

> Los modelos empeoran a razonar y completar tareas a medida que la ventana de contexto se llena. El contexto es escaso, y los harnesses son, en gran medida, mecanismos de entrega de buena ingeniería de contexto.

---

## Context Rot

**Context rot** es la degradación del rendimiento del modelo a medida que la ventana de contexto se llena de información. No es un bug del modelo — es una propiedad fundamental de la atención: más texto = menos atención por token = peor razonamiento.

### Síntomas

| Síntoma | Qué ves |
|---|---|
| **Olvido selectivo** | El agente ignora instrucciones que le diste al principio de la sesión |
| **Repetición** | El agente vuelve a hacer algo que ya hizo |
| **Sesgo de recencia** | Prioriza la información más reciente, aunque una más vieja sea más relevante |
| **Pérdida de coherencia** | Las decisiones del paso 20 contradicen las del paso 5 |
| **Scope drift** | El agente se desvía del objetivo original |

### Causas

1. **Overflow de ventana**: la sesión crece hasta llenar el contexto disponible
2. **Ruido acumulado**: tool outputs largos, errores de compilación extensos, discusiones tangenciales
3. **Complejidad creciente**: a medida que se implementan features, el estado mental del agente se hace más complejo
4. **Cambios externos**: otros developers hacen cambios que invalidan el contexto del agente

---

## Técnicas de mitigación

### 1. Compaction (resumen inteligente)

**Qué es**: cuando la ventana se acerca al límite, el harness resume las partes más viejas de la conversación para liberar espacio.

```
Sesión con 100K tokens (de 200K max)
        │
        ▼
Harness detecta que se acerca al límite
        │
        ▼
Primeros 60K tokens → resumidos a 5K
        │
        ▼
Agente continúa con 45K tokens de espacio libre
```

**Ventaja**: preserva continuidad — el agente sigue en la misma sesión.

**Limitación**: el resumen pierde detalle. Si había un matiz importante en los primeros 60K tokens, puede perderse. Además, si el modelo sufre "context anxiety" (ver abajo), compaction sola puede no ser suficiente.

### 2. Tool-call offloading

**Qué es**: cuando un tool call genera output muy largo (ej: un log de 2000 líneas), el harness no lo mete todo en contexto. En su lugar, mantiene solo la cabeza y la cola, y guarda el output completo en el filesystem.

```
Tool output: 2000 líneas de log
        │
        ▼
En contexto: primeras 20 líneas + últimas 20 líneas + 
             "[output completo guardado en /tmp/build-log.txt]"
        │
        ▼
Si el agente necesita más detalle: lee el archivo
```

**Principio**: el contexto es real-estate premium. No lo llenes con datos que el agente puede buscar on-demand.

### 3. Progressive disclosure (revelación progresiva)

**Qué es**: no cargar todas las herramientas e instrucciones al inicio. Revelar skills y tools solo cuando la tarea los necesita.

```
Inicio de sesión:
  - Instrucciones base (< 60 líneas)
  - Tools esenciales (filesystem, terminal)
        │
        ▼
Agente detecta que necesita hacer deploy:
  - Se carga skill de deployment
  - Se activan tools de CI/CD
        │
        ▼
Agente termina deploy:
  - Skill de deployment se descarga del contexto
```

**Ventaja**: el modelo arranca con un contexto limpio y enfocado. Las 50 herramientas no compiten por atención desde el primer token.

### 4. Full context resets con handoff files

**Qué es**: en vez de resumir, el harness **destruye** la sesión completa y crea una nueva desde cero, pasándole un archivo estructurado de handoff que contiene el estado relevante.

```
Sesión 1 (llena, 180K tokens)
        │
        ▼
Harness genera handoff file:
  - Objetivo original
  - Qué se completó
  - Estado actual del código
  - Próximos pasos
  - Decisiones tomadas
        │
        ▼
Sesión 2 (fresca, handoff = ~5K tokens)
  - Lee handoff
  - Continúa desde donde quedó Session 1
```

**Diferencia con compaction**: compaction resume in-place (misma sesión). Context reset destruye y recrea (sesión limpia). El reset es más drástico pero más efectivo para tareas realmente largas.

**Analogía**: es más parecido a cómo se onboardea un nuevo ingeniero que a cómo "recordamos" cosas. Le das un brief estructurado, no un volcado de toda la conversación.

**Cuándo usarlo**: cuando compaction sola no es suficiente, especialmente para tareas que se extienden por horas o múltiples sesiones.

---

## Context Drift (Deriva de Contexto)

**Qué es**: el agente pierde alineación con la tarea original durante ejecuciones largas. No es que el modelo "olvide" — es que la información reciente domina la atención y el objetivo original queda diluido.

### Técnicas de anclaje

| Técnica | Cómo funciona |
|---|---|
| **Anclaje al objetivo** | Re-inyectar el objetivo original periódicamente en el prompt |
| **Checkpoints de validación** | Cada N pasos, verificar: "¿sigo alineado con el objetivo?" |
| **Descomposición** | Dividir en sub-tareas con objetivos claros e independientes |
| **Plan file en disco** | Escribir el plan en un archivo que el agente re-lee antes de cada paso |
| **Done-condition explícita** | Definir QUÉ ES "terminado" antes de empezar — no después |

### El Plan File

Un patrón que aparece en todos los harnesses maduros: el agente escribe su plan en un archivo en disco antes de empezar. Antes de cada paso, re-lee el archivo para verificar alineación.

```markdown
# Plan — [Nombre de la tarea]

## Objetivo
[1-2 oraciones claras]

## Done condition
[Qué específicamente tiene que ser verdad para que esto esté terminado]

## Pasos
- [x] Paso 1: ...
- [x] Paso 2: ...
- [ ] Paso 3: ... ← ACTUAL
- [ ] Paso 4: ...

## Decisiones tomadas
- [Decisión]: [Razón]

## Notas
- [Contexto relevante que no cabe en los pasos]
```

---

## Context Anxiety

**Qué es**: un sesgo de algunos modelos donde empiezan a "cerrar" el trabajo prematuramente cuando perciben que se acercan al límite de su ventana de contexto. El agente simplifica, toma atajos, o declara "terminado" antes de tiempo.

**Síntomas**:
- El agente dice "ya terminé" pero falta funcionalidad
- Reduce la complejidad de lo que está implementando sin razón técnica
- Empieza a hacer stubs en vez de implementaciones completas

**Mitigación**:
- **Context resets** eliminan la percepción de estar "cerca del límite"
- **Modelos más nuevos** tienden a sufrir menos (Opus 4.6 eliminó mayormente este problema vs Opus 4.5)
- **Done-conditions explícitas** previenen que el agente se auto-declare "terminado"

---

## El contexto como el recurso más valioso

```
El agente tiene 2 recursos limitados:

1. Tokens de contexto    →  Cuánta información puede "ver"
2. Tokens de generación  →  Cuánto puede "escribir"

Todo el harness engineering de contexto optimiza el recurso #1.
```

Cada componente compite por espacio:
- Instrucciones del proyecto (necesario, pero breve)
- Descripciones de herramientas (necesario, pero limitado)
- Historial de conversación (crece sin control)
- Tool outputs (pueden ser enormes)
- Archivos del codebase (potencialmente gigantes)

El harness es el **curador** de ese espacio. Decide qué entra, cuándo, y cuánto se queda.

---

## Resumen

| Concepto | Definición |
|---|---|
| **Context rot** | El modelo empeora al llenar la ventana de contexto |
| **Compaction** | Resumir contexto viejo in-place |
| **Offloading** | Mover tool outputs grandes al filesystem, dejar solo cabeza/cola |
| **Progressive disclosure** | Cargar tools/skills on-demand, no al inicio |
| **Context reset** | Destruir sesión, crear nueva con handoff file |
| **Context drift** | Pérdida de alineación con el objetivo original |
| **Context anxiety** | Modelo cierra prematuramente por percibir límite de contexto |
| **Plan file** | Archivo en disco que ancla al agente a su objetivo |
| **Done-condition** | Definir "terminado" antes de empezar, no después |
