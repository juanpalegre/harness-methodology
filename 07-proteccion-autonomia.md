# 07 — Protección y Autonomía

> La cantidad de supervisión debe ser proporcional al riesgo. No más, no menos.

---

## Las 3 dimensiones de riesgo

Toda acción de un agente tiene riesgo en tres ejes independientes:

| Dimensión | Pregunta | Ejemplo alto riesgo |
|---|---|---|
| **Operacional** | ¿Puede romper estabilidad/disponibilidad? | Despliegues, cambios de infraestructura, migraciones de BD |
| **Seguridad** | ¿Puede crear vulnerabilidades o exponer datos? | Manejo de secrets, configuración de red, permisos |
| **Compliance** | ¿Puede violar normativas? | Datos personales, retención de logs, auditoría |

Una acción puede ser de bajo riesgo en una dimensión y alto en otra. Ej: corregir un typo en un README es bajo en las tres. Cambiar un endpoint que maneja datos de salud es bajo operacional, alto seguridad, alto compliance.

---

## Los 4 niveles de autonomía

El nivel de autonomía debe ser proporcional al riesgo, no fijo:

| Nivel | Supervisión | Cuándo aplicar | Ejemplo |
|---|---|---|---|
| **Totalmente autónomo** | Ninguna | Riesgo BAJO en las 3 dimensiones | Auto-format, fix typos, actualizar docs |
| **Semi-autónomo** | Ejecuta + espera validación | Riesgo MEDIO | Crear MR/PR, implementar features acotadas |
| **Supervisado** | Genera plan, humano ejecuta | Riesgo ALTO | Migración de BD, cambio de arquitectura |
| **Restringido** | Solo lectura/análisis | Cualquier riesgo | Análisis de código, auditoría, reporting |

**El estándar para la mayoría de tareas de desarrollo es semi-autónomo**: el agente implementa y crea el MR/PR, pero un humano revisa y aprueba antes del merge.

### Anti-patrón: sobre-autonomía

El anti-patrón más peligroso es un agente con permisos de merge directo a la rama principal sin supervisión. Incluso si el agente es 99% correcto, el 1% que falla puede ser catastrófico.

### Anti-patrón: approval fatigue

El opuesto es igualmente peligroso: **demasiadas puertas de aprobación**. Cuando hay tantos pasos de aprobación que los revisores dejan de leer y aprueban mecánicamente, la seguridad efectiva es **menor** que con menos puertas.

**Principio**: solo puertas proporcionales al riesgo.

---

## Las 4 capas de protección

### Capa 1: Prevención

Impedir que acciones riesgosas se ejecuten.

| Mecanismo | Qué previene | Implementación agnóstica |
|---|---|---|
| **Allowlists de herramientas** | Agente conecta a servicios no autorizados | Lista de servidores/APIs permitidos a nivel org |
| **Tokens con scope mínimo** | Agente accede a recursos no necesarios | Solo permisos mínimos: leer código + escribir MR/PRs |
| **Firewall de agente** | Exfiltración de datos, contacto con servicios maliciosos | Default-deny para conexiones de red |
| **Permisos restrictivos** | Modificación de config crítica | Sin acceso a admin, infra, secrets |

**Principio de mínimo privilegio**: el agente debe tener exactamente los permisos necesarios para su tarea, ninguno más.

### Capa 2: Detección y Bloqueo

Detectar y bloquear acciones riesgosas antes de que impacten.

| Mecanismo | Qué detecta |
|---|---|
| **Branch protection / reglas de rama** | Merge sin aprobación, push directo a main |
| **Status checks obligatorios** | Código que no compila, no pasa lint, tiene vulnerabilidades |
| **Análisis de seguridad (SAST)** | Patrones de código inseguro |
| **Revisión de dependencias** | Dependencias con vulnerabilidades conocidas |
| **Secret scanning** | Secrets hardcodeados en el código |

### Capa 3: Supervisión Humana

Puntos donde un humano debe aprobar explícitamente.

| Mecanismo | Cuándo aplica |
|---|---|
| **Code review en MR/PR** | Siempre — es la puerta principal |
| **CODEOWNERS / ownership rules** | Cambios en áreas sensibles requieren aprobación de expertos |
| **Environment protection** | Deploy a producción requiere aprobación manual |
| **Workflow manual** | Agente prepara la acción, humano la ejecuta |

### Capa 4: Auditoría

Registro para reconstrucción y compliance.

| Mecanismo | Qué registra |
|---|---|
| **Historial de VCS** | Todos los cambios, quién, cuándo, por qué |
| **Timeline de MR/PR** | Discusiones, aprobaciones, cambios de estado |
| **Logs de pipeline** | Resultados de build, test, deploy |
| **Logs de sesión del agente** | Qué tools usó, qué generó, qué errores encontró |
| **Audit log organizacional** | Cambios de permisos, configuración, accesos |

---

## Hooks como capa de enforcement

Los hooks son el mecanismo que convierte intenciones en restricciones ejecutables. No dependen de que el modelo "se acuerde" — se ejecutan programáticamente.

### Hooks comunes y su implementación

| Hook | Trigger | Acción |
|---|---|---|
| **post-edit lint** | Después de cada edición de archivo | Ejecutar linter + typecheck; inyectar errores al agente |
| **block destructive** | Antes de ejecutar comando en terminal | Bloquear `rm -rf`, `git push --force`, `DROP TABLE` |
| **no-skipped-tests** | Antes de commit | Buscar `.skip(`, `xit(`, `@Ignore`; rechazar si encuentra |
| **auto-format** | Después de escribir archivo | Formatear con el formatter del proyecto |
| **test-on-edit** | Después de editar código fuente | Ejecutar tests relacionados |
| **approval-gate** | Antes de merge | Verificar N aprobaciones + CI verde |

### Principio de diseño

```
Éxito → silencioso (el agente no escucha nada)
Fallo  → verboso  (error inyectado al loop para autocorrección)
```

Esto hace que el cost del hook sea casi cero en el caso feliz, y directamente accionable cuando algo falla.

---

## Acciones irreversibles

Para acciones que no se pueden deshacer, el protocolo es más estricto:

| Requisito | Ejemplo |
|---|---|
| **2+ aprobaciones** | No alcanza con un solo reviewer |
| **Período de gracia** | Esperar N horas/días antes de ejecutar |
| **Plan de rollback** | Documentar cómo revertir antes de ejecutar |
| **Logging explícito** | Registrar quién aprobó, cuándo, por qué |

**Ejemplos de acciones irreversibles**:
- Eliminación de datos de producción
- Cambios de permisos a nivel organización
- Rotación de secrets/certificados
- Migración destructiva de base de datos
- Deploy sin rollback automático

---

## Modelo de confianza en capas

La confianza se asigna en capas, de más general a más específica:

```
Organización
  │ Políticas globales: qué tools están permitidos, 
  │ qué servidores de extensión son confiables
  │
  └── Repositorio
        │ Reglas de branch protection, instrucciones del proyecto,
        │ quién puede aprobar MR/PRs
        │
        └── Agente individual
              │ Herramientas específicas, restricciones por rol,
              │ nivel de autonomía asignado
```

Cada capa hereda las restricciones de la capa superior. Un agente no puede tener más permisos que su repositorio, y un repositorio no puede violar políticas de la organización.

---

## IA Responsable en el harness

| Principio | Implementación |
|---|---|
| **Seguridad del contenido** | El agente no genera código malicioso, exploits, o bypass de seguridad |
| **Transparencia** | Cambios del agente son identificables (commit author, labels en MR/PR) |
| **Trazabilidad** | Se puede reconstruir el razonamiento del agente para cualquier cambio |
| **Monitoreo** | Detectar comportamiento anómalo (scope creep, acceso inusual, output sospechoso) |
| **Control humano** | El humano siempre puede detener, revertir, y override al agente |

---

## Resumen

| Concepto | Clave |
|---|---|
| **3 dimensiones** | Operacional + Seguridad + Compliance |
| **4 niveles autonomía** | Autónomo → Semi-autónomo → Supervisado → Restringido |
| **4 capas protección** | Prevención → Detección → Supervisión → Auditoría |
| **Mínimo privilegio** | Exactamente los permisos necesarios, ninguno más |
| **Hooks** | Enforcement programático; éxito silencioso, fallo verboso |
| **Approval fatigue** | Demasiadas puertas → revisores no leen → MENOS seguridad |
| **Irreversibles** | 2+ aprobaciones + gracia + rollback plan + logging |
| **Confianza en capas** | Org → Repo → Agente; cada capa hereda restricciones |
