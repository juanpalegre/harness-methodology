# 05 — Evaluación Adversarial

> Separar al que genera del que evalúa. Los agentes consistentemente sesgan positivo cuando evalúan su propio trabajo.  
> — Anthropic

---

## Por qué la auto-evaluación falla

Cuando le pedís a un agente que evalúe su propio trabajo, tiende a responder **elogiándolo con confianza** — incluso cuando la calidad es obviamente mediocre para un humano.

Esto no es un bug aislado; es un patrón sistémico:

- El generador tiene **sesgo de confirmación**: ya invirtió tokens en producir la salida y está predispuesto a defenderla
- El modelo **sesga positivo por diseño**: fue entrenado para ser útil y optimista
- En tareas subjetivas (diseño, UX, legibilidad) no hay un check binario tipo "tests pasan/fallan"
- Incluso en tareas con checks verificables, el agente a veces exhibe **mal juicio**: declara "todo bien" cuando hay errores evidentes

**La separación entre generación y evaluación es el mecanismo más fuerte** para resolver esto.

---

## El patrón Generator/Evaluator (inspiración GAN)

Inspirado en las Generative Adversarial Networks (GANs), el patrón separa dos roles en agentes distintos:

```
                ┌──────────────┐
                │   GENERATOR  │
                │ (implementa) │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │  EVALUATOR   │
                │  (juzga)     │
                └──────┬───────┘
                       │
            ┌──────────┴──────────┐
            │                     │
        APRUEBA               RECHAZA
            │                     │
            ▼                     ▼
        Continúa          Feedback detallado
        al paso              → Generator
        siguiente              rehace
```

### Calibración del evaluador

Hacer un evaluador externo no elimina automáticamente la lenidad. Pero **calibrar un evaluador standalone para ser escéptico es mucho más tratable** que hacer que un generador sea crítico de sí mismo.

Técnicas de calibración:
- **Few-shot examples**: darle ejemplos de evaluaciones con scores detallados que reflejan tus estándares
- **Criterios explícitos**: no "¿está bien?" sino "¿cumple estos 4 criterios específicos?"
- **Umbrales duros**: si cualquier criterio baja de un threshold, el sprint falla
- **Penalización de patrones conocidos**: marcar explícitamente los patrones "AI slop" que querés evitar

### Ejemplo: criterios de evaluación para frontend (Anthropic)

| Criterio | Qué evalúa | Peso |
|---|---|---|
| **Calidad de diseño** | ¿Se siente como un todo coherente? Colores, tipografía, layout, identidad | Alto |
| **Originalidad** | ¿Hay decisiones creativas deliberadas o es template/defaults de librería? | Alto |
| **Craft** | Jerarquía tipográfica, espaciado consistente, armonía de color, contraste | Medio |
| **Funcionalidad** | ¿El usuario puede entender qué hace la interfaz y completar tareas? | Medio |

Se priorizan calidad y originalidad porque craft y funcionalidad tienden a salir bien por defecto con modelos capaces.

### Ejemplo: criterios de evaluación para código full-stack

| Criterio | Qué evalúa |
|---|---|
| **Completitud** | ¿Se implementaron todas las features del spec? ¿Hay stubs? |
| **Funcionalidad** | ¿Las features funcionan end-to-end? (verificado con browser headless) |
| **Diseño visual** | ¿La UI tiene identidad o es genérica? |
| **Calidad de código** | ¿Estructura limpia, manejo de errores, tests? |

---

## Sprint Contracts

Antes de que el generator empiece a codear, el generator y el evaluator **negocian qué "terminado" significa** para ese chunk de trabajo.

```
Generator propone:
  - Qué va a implementar
  - Cómo se verificará el éxito

Evaluator revisa:
  - ¿El scope está alineado con el spec?
  - ¿Los criterios de verificación son suficientes?
  - ¿Falta algo?

Iteran hasta acordar
        │
        ▼
Sprint contract firmado
        │
        ▼
Generator implementa contra el contrato
        │
        ▼
Evaluator evalúa contra el contrato
```

**Por qué funciona**: el spec del proyecto es intencionalmente de alto nivel (qué construir, no cómo). El sprint contract cierra la brecha entre historias de usuario y comportamiento testeable, sin sobre-especificar la implementación demasiado temprano.

**Ejemplo de sprint contract**:

```markdown
# Sprint Contract — Editor de Sprites

## Entregables
1. Canvas de edición con zoom y grilla
2. Paleta de colores con selector personalizado
3. Herramientas: pincel, borrador, fill, línea
4. Exportar sprite como PNG

## Criterios de verificación
- [ ] Canvas responde a mouse events (dibujar, borrar)
- [ ] Zoom funcional (in/out, scroll wheel)
- [ ] Al menos 16 colores en la paleta
- [ ] Cada herramienta produce el resultado visual esperado
- [ ] Export genera PNG válido descargable
- [ ] No hay errores en consola del browser

## Fuera de scope
- Animaciones (Sprint 4)
- Importación de imágenes (Sprint 5)
```

---

## Las 4 categorías de causa raíz

Cuando el agente falla, diagnosticar la causa raíz antes de aplicar el fix. Cada categoría tiene soluciones distintas:

### 1. Razonamiento

**Qué es**: el modelo razona mal — comete errores de lógica, alucina APIs que no existen, o toma un enfoque incorrecto.

| Señal | Ejemplo |
|---|---|
| Hallucination | Inventa `response.getData()` cuando el método real es `response.json()` |
| Lógica invertida | Condicional al revés que invierte el comportamiento |
| Sobre-ingeniería | Implementa un patrón complejo cuando bastaba un `if` |

**Soluciones**: mejorar instrucciones, dar ejemplos explícitos de lo que SÍ existe, verificar contra documentación real.

### 2. Herramientas

**Qué es**: el agente usa la herramienta incorrecta, con los parámetros mal, o en el orden incorrecto.

| Señal | Ejemplo |
|---|---|
| Tool equivocada | Usa `write_file` cuando necesita `replace_in_file` |
| Parámetros mal | Pasa path relativo cuando se necesita absoluto |
| Orden incorrecto | Intenta correr tests antes de instalar dependencias |

**Soluciones**: mejorar descripciones de tools, restringir tools disponibles, agregar validación de parámetros.

### 3. Contexto

**Qué es**: al agente le falta información, o la información que tiene es incorrecta o contradictoria.

| Señal | Ejemplo |
|---|---|
| Info faltante | No sabe que el proyecto usa TypeScript strict |
| Info obsoleta | Usa una API deprecada que cambió hace 2 versiones |
| Info contradictoria | Instrucciones dicen "usar REST" pero el código existente usa GraphQL |

**Soluciones**: actualizar instrucciones, agregar a `.context/`, dar acceso a documentación actualizada.

### 4. Ambiente

**Qué es**: el entorno de ejecución falla — permisos insuficientes, dependencias faltantes, límites de recursos.

| Señal | Ejemplo |
|---|---|
| Permisos | No puede escribir en un directorio protegido |
| Dependencias | `npm install` falla porque falta una system library |
| Timeouts | El test suite tarda más que el timeout del runner |

**Soluciones**: mejorar setup del sandbox, agregar pre-requisitos al entorno, ajustar límites.

---

## El ciclo de mejora continua

```
Evaluar (¿el output cumple los criterios?)
    │
    ├── SÍ → Continuar al próximo paso
    │
    └── NO → Diagnosticar
              │
              ├── Categorizar causa raíz
              │     (razonamiento / herramienta / contexto / ambiente)
              │
              └── Ajustar
                    │
                    ├── Instrucciones (agregar/modificar reglas)
                    ├── Memoria (agregar facts, actualizar docs)
                    └── Herramientas (agregar/restringir/configurar)
                    │
                    └── Re-evaluar
```

---

## Señales cuantitativas y cualitativas

### Cuantitativas (automatizables)

| Métrica | Umbral saludable | Qué indica si falla |
|---|---|---|
| CI pass rate | > 95% | El agente genera código que no compila/pasa lint |
| Approval rate (code review) | > 80% | El agente no sigue convenciones o comete errores de diseño |
| Merge rate | > 90% | El agente genera trabajo que no llega a producción |
| Cobertura de tests | > 80% | El agente no testea suficiente |
| Tiempo por tarea | Variable | El agente se pierde en loops o re-trabaja demasiado |

### Cualitativas (requieren juicio humano)

- **Alineación arquitectónica**: ¿el código sigue los patrones del proyecto?
- **Legibilidad**: ¿se entiende sin comentarios excesivos?
- **Edge cases**: ¿los maneja o solo cubre el happy path?
- **Scope**: ¿modificó solo lo necesario o tocó archivos que no debía?
- **Nombramiento**: ¿los nombres comunican intención?

---

## Resumen

| Concepto | Clave |
|---|---|
| **Auto-eval falla** | Los agentes sesgan positivo al evaluar su propio trabajo |
| **Generator/Evaluator** | Separar generación de evaluación; calibrar evaluador para ser escéptico |
| **Sprint contract** | Negociar "done" antes de codear; no sobre-especificar |
| **4 causa raíz** | Razonamiento, herramientas, contexto, ambiente |
| **Señales cuanti** | CI pass rate, approval rate, merge rate, cobertura |
| **Señales cuali** | Arquitectura, legibilidad, edge cases, scope |
| **Ciclo** | Evaluar → Diagnosticar → Ajustar → Re-evaluar |
