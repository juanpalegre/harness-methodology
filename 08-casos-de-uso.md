# 08 — Casos de Uso

> Un buen harness no se descarga de un template. Está determinado por tu historial de fallos. Pero los patrones se repiten.

---

## Caso 1: Onboarding de proyecto existente (crear harness desde cero)

### Situación

Tenés un proyecto con 50K líneas de código, 3 desarrolladores, CI/CD funcionando, pero sin ninguna estructura para agentes. El agente alucina, ignora convenciones, y genera código inconsistente.

### Componentes del harness involucrados

- Contexto (instrucciones + spec)
- Memoria (lessons)
- Enforcement (hooks básicos)

### Pasos

**1. Auditar el proyecto (15 min)**

Antes de escribir una sola instrucción, entender qué hay:
- ¿Qué stack? ¿Qué framework de tests?
- ¿Qué convenciones se siguen (implícitamente)?
- ¿Qué errores comete el agente hoy?
- ¿Qué archivos son intocables?

**2. Crear la estructura mínima**

```
proyecto/
├── .ai/
│   └── instructions.md         # Instrucciones globales (< 60 líneas)
├── .context/
│   ├── architecture.md         # Patrones y capas del proyecto
│   ├── conventions.md          # Naming, error handling, commit format
│   └── stack.md                # Stack, deps, comandos útiles
└── tasks/
    └── lessons.md              # Vacío — se llena con el ratchet
```

**3. Escribir instrucciones mínimas**

```markdown
# Instrucciones del proyecto

## Stack
- TypeScript strict, Node.js 20, Express, PostgreSQL
- Tests: Vitest. Correr con `npm test`
- Lint: ESLint + Prettier. Correr con `npm run lint`

## Reglas
- Nunca modificar archivos en /infrastructure/ o /migrations/
- Usar camelCase para variables, PascalCase para tipos
- Cada función pública debe tener test
- Máximo 300 líneas por archivo
- Manejar errores con try/catch, nunca swallow silencioso
- Commits: tipo(scope): descripción — ej: feat(auth): add JWT refresh

## Proceso
- SIEMPRE leer tasks/lessons.md al inicio de sesión
- Planificar ANTES de implementar
- Correr tests después de cada cambio
```

**4. Empezar a usar y aplicar ratchet**

Cada vez que el agente comete un error, agregar a lessons.md y, si aplica, a las instrucciones.

---

## Caso 2: Bug fix con ratchet

### Situación

El agente recibe un issue: "Login falla cuando el email tiene mayúsculas". Lo arregla, pero su fix introduce un bug de regresión porque no corrió los tests existentes.

### Componentes del harness involucrados

- Ratchet (lessons → instrucción → hook)
- Back-pressure (test failure inyectado al loop)

### Secuencia

**1. Primera ejecución (fallo)**
```
Agente lee issue → implementa fix (normalizar email a lowercase)
→ Commit sin correr tests → CI falla: test "email display should preserve case" roto
```

**2. Diagnóstico (causa raíz: ambiente/herramientas)**
- El agente no corrió tests antes de commitear
- El fix era correcto para auth pero rompió display

**3. Aplicar ratchet (3 niveles)**

```markdown
# tasks/lessons.md
### 2026-05-30 — Herramientas
- **Error**: Commit sin correr tests → regresión en email display
- **Causa raíz**: Herramientas — no usó test runner antes de commit
- **Regla**: SIEMPRE correr tests antes de commit
- **Enforcement**: Hook post-edit que ejecuta tests relacionados
```

```markdown
# .ai/instructions.md (agregar línea)
- Después de cada cambio en src/, correr `npm test` antes de commitear
```

```yaml
# Hook pre-commit (agnóstico)
pre-commit:
  - name: run-tests
    run: npm test
    on-failure: block-commit
```

**4. Segunda ejecución (éxito)**
```
Agente lee issue → implementa fix → hook ejecuta tests → test falla 
→ agente ve el error → ajusta fix (normalizar solo para auth, preservar para display) 
→ tests pasan → commit exitoso
```

---

## Caso 3: Feature nueva con ciclo completo

### Situación

Feature request: "Agregar sistema de notificaciones push". Es una feature compleja que toca backend, frontend y configuración de servicios.

### Componentes del harness involucrados

- Planner (descomposición)
- Generator/Evaluator (implementación + validación)
- Sprint contracts (definición de "done")
- Observabilidad (MR/PR + commits + CI)

### Secuencia

**1. Planificación**

```markdown
# tasks/todo.md — Plan generado por el planner

## Feature: Notificaciones Push

### Sprint 1: Backend API
- [ ] Modelo de datos: Notification (id, userId, type, payload, read, createdAt)
- [ ] Endpoint POST /notifications — crear notificación
- [ ] Endpoint GET /notifications — listar por usuario
- [ ] Endpoint PATCH /notifications/:id/read — marcar leída
- [ ] Tests unitarios para cada endpoint

### Sprint 2: Integración con servicio push
- [ ] Configurar cliente del servicio push (agnóstico)
- [ ] Queue de envío async
- [ ] Retry con backoff exponencial
- [ ] Tests de integración

### Sprint 3: Frontend
- [ ] Componente NotificationBell con badge counter
- [ ] Panel desplegable con lista de notificaciones
- [ ] Mark as read on click
- [ ] Polling o WebSocket para actualizaciones
- [ ] Tests de componentes
```

**2. Sprint contract (Sprint 1)**

```markdown
# Sprint Contract — Backend API de Notificaciones

## Entregables
API REST con 3 endpoints + modelo de datos + tests

## Criterios de verificación
- [ ] POST /notifications crea notificación y retorna 201
- [ ] GET /notifications retorna lista paginada filtrada por userId
- [ ] PATCH /notifications/:id/read retorna 200 y marca read=true
- [ ] Validación de input: retorna 400 en payload inválido
- [ ] Tests: cobertura > 80% de los endpoints
- [ ] No hay errores de lint/typecheck
```

**3. Generator implementa → Evaluator valida**

```
Generator:
  - Implementa modelo + endpoints + tests
  - Self-check: "¿cumple el contract?"
  - Crea MR/PR con description detallada

Evaluator:
  - Lee sprint contract
  - Ejecuta la app (o tests) para verificar cada criterio
  - Resultado: 
    - "POST /notifications: PASS"
    - "GET /notifications: FAIL — no implementó paginación"
    - "Cobertura: 72% — falta test para 400 en payload inválido"

Generator recibe feedback → arregla → Evaluator re-valida → PASS
```

**4. Merge + siguiente sprint**

---

## Caso 4: Multi-agente para refactor grande

### Situación

Refactorizar un monolito: separar el módulo de auth (actualmente acoplado a todo) en un servicio independiente.

### Componentes del harness involucrados

- Fan-out (paralelo) con boundaries claros
- Peer review entre agentes
- Aislamiento por ramas

### Secuencia

```markdown
# Descomposición (humano o planner)

## Agente A: Extraer interfaces
- Rama: feature/refactor-auth-interfaces
- Scope: definir interfaces/contratos entre auth y el resto
- NO modificar implementaciones
- Output: interfaces + types

## Agente B: Implementar servicio auth standalone
- Rama: feature/refactor-auth-service
- Scope: implementar el servicio de auth usando las interfaces de A
- Depende de: merge de MR/PR de Agente A
- Output: servicio completo con tests

## Agente C: Adaptar consumidores
- Rama: feature/refactor-auth-consumers  
- Scope: reemplazar llamadas directas a auth por las nuevas interfaces
- Depende de: merge de MR/PR de Agente A
- Output: consumidores refactorizados con tests
```

```
Agente A (interfaces)
    │
    ▼ merge
    ├──────────────────┐
    ▼                  ▼
Agente B (servicio)  Agente C (consumidores)  ← paralelo
    │                  │
    ▼ merge            ▼ merge
    └──────────────────┘
              │
              ▼
    Integración final
```

---

## Caso 5: Agente de largo aliento con context resets

### Situación

Construir una aplicación web completa desde un prompt de una línea: "Crear un editor de diagramas colaborativo en tiempo real". Estimación: 4+ horas de ejecución autónoma.

### Componentes del harness involucrados

- Planner (spec detallado desde 1 línea)
- Context resets con handoff files
- Evaluator con Playwright/browser headless
- Ralph Loop para continuación

### Secuencia

**1. Planner expande el prompt**

```
Input: "Crear un editor de diagramas colaborativo en tiempo real"

Output: Spec de 15 features en 8 sprints
  1. Canvas con shapes básicas (rect, circle, line, text)
  2. Toolbar con herramientas de dibujo
  3. Drag, resize, rotate de shapes
  4. Undo/redo
  5. Export PNG/SVG
  6. Backend con WebSocket para colaboración
  7. Cursors de otros usuarios en tiempo real
  8. Sistema de rooms/sesiones
  ...
```

**2. Generator trabaja sprint por sprint**

Cada sprint puede durar 30-60 minutos. Cuando el contexto se llena:

```
Sesión 1 (sprints 1-3, 120K tokens)
    │
    ▼ contexto lleno
    │
Harness genera handoff:

# Handoff — Sesión 1 → Sesión 2
## Completado
- Sprint 1: Canvas funcional con 4 shapes ✓
- Sprint 2: Toolbar con 6 herramientas ✓
- Sprint 3: Drag/resize funcional ✓
## Estado actual
- Frontend: React + Canvas API, corriendo en localhost:3000
- 23 archivos, 2400 LOC
- Tests: 18 pasando, 0 fallando
## Próximo: Sprint 4 (Undo/Redo)
## Decisiones clave
- Canvas API (no SVG) por performance
- State management con Zustand (no Redux)
    │
    ▼
Sesión 2 (fresca, lee handoff + código) → continúa Sprint 4
```

**3. Evaluator verifica con browser**

```
Evaluator:
  - Abre localhost:3000 con browser headless
  - Navega la UI, clickea herramientas, dibuja shapes
  - Verifica cada criterio del sprint contract
  - Screenshot + reporte

Resultado:
  - "Canvas shapes: PASS"
  - "Drag & drop: PASS" 
  - "Resize: FAIL — no mantiene aspect ratio con Shift"
  - "Undo: FAIL — deshace de a 2 pasos en vez de 1"
```

---

## Caso 6: Migración de harness entre plataformas

### Situación

Tu equipo trabaja con GitHub y decide migrar a GitLab (o viceversa). Tenés un harness completo configurado. ¿Cómo migrar sin perder la inversión?

### Componentes del harness involucrados

- Tabla de equivalencias de plataformas
- Separación entre lo agnóstico y lo específico

### Principio clave

El harness tiene dos capas:

```
┌──────────────────────────────────┐
│  CAPA AGNÓSTICA (se migra 1:1)  │
│                                  │
│  .context/ (architecture, etc.)  │
│  .ai/instructions.md             │
│  .ai/agents/*.md                 │
│  tasks/lessons.md                │
│  tasks/todo.md                   │
│  Hooks (lógica)                  │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  CAPA ESPECÍFICA (traducir)     │
│                                  │
│  CI/CD pipeline syntax           │
│  Branch protection config        │
│  MR/PR templates                 │
│  Permisos y roles                │
│  Integration con servicios       │
└──────────────────────────────────┘
```

### Checklist de migración

| Componente | Acción |
|---|---|
| `.context/` | Copiar tal cual (100% agnóstico) |
| Instrucciones de agente | Copiar contenido; adaptar path si la herramienta lo requiere |
| `tasks/lessons.md` | Copiar tal cual (100% agnóstico) |
| CI/CD | Traducir syntax: Actions → GitLab CI / Azure Pipelines / etc. |
| Branch protection | Recrear en la nueva plataforma con reglas equivalentes |
| Hooks | Misma lógica; puede cambiar el mecanismo de trigger |
| Secrets/variables | Reconfigurar en la nueva plataforma |

Ver [mapeo-plataformas.md](mapeo-plataformas.md) para la tabla completa de equivalencias.

---

## Patrones recurrentes en todos los casos

| Patrón | Aparece en |
|---|---|
| **Definir "done" antes de empezar** | Casos 3, 4, 5 |
| **Descomponer antes de ejecutar** | Casos 3, 4, 5 |
| **Cada error → regla permanente** | Caso 2 (explícito), todos (implícito) |
| **Separar generación de evaluación** | Casos 3, 5 |
| **Aislamiento por ramas** | Casos 3, 4 |
| **Handoff estructurado entre sesiones** | Caso 5 |
| **Lo agnóstico se migra; lo específico se traduce** | Caso 6 |
