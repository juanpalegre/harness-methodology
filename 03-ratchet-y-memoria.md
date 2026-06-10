# 03 — El Ratchet y la Memoria

> Solo agregás constraints cuando ves un fallo real. Solo los removés cuando un modelo más capaz los hace redundantes.

---

## El Ratchet (Trinquete)

El ratchet es **el hábito más importante** de harness engineering. Consiste en tratar los errores del agente como señales permanentes — no como anécdotas, no como "malos runs" para reintentar. Señales.

### El ciclo

```
Agente comete error
        │
        ▼
Diagnosticar causa raíz
        │
        ├── ¿El agente no sabía una convención?
        │     └── Agregar a instrucciones (.ai/instructions.md)
        │
        ├── ¿El agente ejecutó algo destructivo?
        │     └── Agregar hook que lo bloquee
        │
        ├── ¿El agente se perdió en tarea larga?
        │     └── Descomponer en planner + executor
        │
        ├── ¿El agente ignoró un test?
        │     └── Agregar hook post-edit que corra tests
        │
        └── ¿El agente "terminó" con código roto?
              └── Agregar back-pressure: typecheck inyectado al loop
        │
        ▼
Error no puede volver a ocurrir
```

### Reglas del ratchet

1. **Cada línea del archivo de instrucciones debe ser trazable a un error específico**. Si no podés nombrar el fallo que la originó, es ruido.

2. **Ganar cada línea**. HumanLayer mantiene su archivo de instrucciones en menos de 60 líneas. Cada línea compite por atención. Más reglas hacen que cada regla importe menos.

3. **El ratchet tiene 3 niveles de enforcement**:

| Nivel | Mecanismo | Ejemplo |
|---|---|---|
| **Sugerencia** | Instrucción en archivo de reglas | "Usar camelCase para variables" |
| **Validación** | CI/CD check | Linter que falla si hay snake_case |
| **Bloqueo** | Hook que impide la acción | Pre-commit hook que rechaza el commit |

4. **Solo remover constraints cuando el modelo los hace redundantes**. Si Modelo v2 ya no comete el error que motivó una regla, remover la regla. Cada componente del harness codifica una suposición sobre lo que el modelo no puede hacer solo.

### Ejemplo concreto de ratchet

**Fallo observado**: el agente comenta tests en vez de arreglarlos para que CI pase.

**Respuesta en 3 niveles**:

```markdown
# En instrucciones (.ai/instructions.md)
- Nunca comentar tests. Eliminarlos o arreglarlos.
```

```yaml
# Hook pre-commit (agnóstico — se implementa según plataforma)
pre-commit:
  - name: no-skipped-tests
    run: |
      # Buscar patrones de tests comentados/skippeados
      grep -rn "\.skip\(\|xit(\|xdescribe(\|@pytest.mark.skip\|@Ignore" --include="*.test.*" --include="*_test.*" --include="*.spec.*"
      if [ $? -eq 0 ]; then
        echo "ERROR: Tests skippeados o comentados detectados"
        exit 1
      fi
```

```markdown
# En el agente reviewer (.ai/agents/reviewer.md)
## Regla: Tests comentados
Si detectás tests comentados, skippeados o con `.skip()`:
- Marcar como BLOCKER
- No aprobar hasta que se eliminen o se arreglen
```

---

## Tipos de Memoria

Un agente tiene tres mecanismos para "saber cosas", cada uno con características distintas:

### 1. Memoria de corto plazo (sesión)

**Qué es**: toda la información dentro de la ventana de contexto actual.

| Característica | Detalle |
|---|---|
| **Duración** | Solo la sesión activa |
| **Contenido** | Historial de chat, archivos abiertos, resultados de herramientas |
| **Límite** | Ventana de contexto del modelo (varía: 128K–200K tokens) |
| **Volatilidad** | Se pierde al cerrar sesión |

**Riesgo**: a medida que la sesión crece, la información vieja queda "lejos" y el modelo la pierde (context rot). Ver [04-context-engineering](04-context-engineering.md).

### 2. Memoria de largo plazo (instrucciones persistentes)

**Qué es**: archivos en el repositorio que se inyectan en cada sesión.

| Característica | Detalle |
|---|---|
| **Duración** | Permanente (vive en el repo) |
| **Contenido** | Instrucciones, convenciones, reglas del ratchet, lecciones aprendidas |
| **Límite** | Sin límite técnico, pero limitado por atención del modelo |
| **Mantenimiento** | Manual — requiere que alguien actualice los archivos |

**Archivos típicos**:
```
.ai/instructions.md          # Reglas globales (< 60 líneas)
.context/architecture.md      # Decisiones de arquitectura
.context/conventions.md       # Convenciones de código
.context/stack.md             # Stack tecnológico
tasks/lessons.md              # Lecciones del ratchet
```

**Jerarquía de precedencia** (de mayor a menor prioridad):
```
1. Prompt actual del usuario
2. Instrucciones locales (directorio específico)
3. Instrucciones globales del repo
4. Memoria del servicio (si el proveedor la soporta)
5. Inferencia del código existente
```

### 3. Memoria externa

**Qué es**: datos que el agente accede activamente via herramientas.

| Característica | Detalle |
|---|---|
| **Duración** | Depende de la fuente |
| **Contenido** | Archivos del repo, bases de datos, APIs, documentación, web |
| **Límite** | Sin límite de tamaño |
| **Acceso** | Requiere herramientas configuradas |

**Ejemplos**: buscar en la documentación de una librería, consultar una base de datos, leer un archivo largo que no cabe en contexto.

---

## Persistencia de estado vía control de versiones

El control de versiones (Git, etc.) es el mecanismo de persistencia más robusto para el trabajo del agente:

| Artefacto VCS | Función para el agente |
|---|---|
| **Commits** | Checkpoints de progreso — puntos de recuperación |
| **Merge/Pull Requests** | Registro acumulado de lo realizado + puerta de aprobación |
| **Comentarios en MR/PR** | Historial de decisiones y hallazgos |
| **Ramas** | Aislamiento de trabajo (un agente = una rama) |
| **Historial** | Línea temporal completa para reconstruir contexto |

### Reanudación sin repetición

Cuando un agente necesita continuar trabajo que empezó en otra sesión:

```
1. Leer commits recientes → entender qué se hizo
2. Leer MR/PR description → entender el objetivo global
3. Leer tasks/todo.md → ver qué falta
4. Leer tasks/lessons.md → cargar ratchet
5. Continuar desde el último punto válido
```

Esto es análogo a cómo un humano se onboardea un lunes: revisa los commits del viernes, lee los comentarios del PR, y continúa.

---

## El archivo `lessons.md` como implementación del ratchet

El formato recomendado para el archivo de lecciones (ratchet manual):

```markdown
# Lecciones — [Proyecto]

> Cada entrada viene de un error real. El agente consulta este archivo
> al inicio de cada sesión.

## Reglas activas

### [Fecha] — [Categoría]
- **Error**: [qué pasó concretamente]
- **Causa raíz**: [por qué pasó — de las 4 categorías: razonamiento/herramientas/contexto/ambiente]
- **Regla**: [qué hacer distinto]
- **Enforcement**: [cómo se fuerza — instrucción/hook/test/CI]

## Reglas retiradas

### [Fecha retirada] — [Categoría]
- **Regla original**: [qué decía]
- **Razón de retiro**: [el modelo ya no comete este error / ya no aplica]
```

**Diferencia con un `lessons.md` simple**: el formato de ratchet incluye:
- **Causa raíz** categorizada (no solo "qué pasó")
- **Enforcement** explícito (no solo "qué hacer distinto" sino "cómo se fuerza")
- **Sección de reglas retiradas** (para evolucionar con el modelo)

---

## Resumen

| Concepto | Clave |
|---|---|
| **Ratchet** | Error → regla → enforcement; solo avanza, nunca retrocede (salvo por mejora del modelo) |
| **60 líneas** | Límite práctico de instrucciones; cada línea ganada con un fallo real |
| **3 niveles** | Sugerencia (instrucción) → Validación (CI) → Bloqueo (hook) |
| **3 memorias** | Corto plazo (sesión) → Largo plazo (archivos repo) → Externa (tools) |
| **VCS = persistencia** | Commits = checkpoints, MR/PR = registro, ramas = aislamiento |
| **Reanudación** | Leer commits + MR + todo + lessons → continuar sin repetir |
