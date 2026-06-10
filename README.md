# Harness Engineering — Metodología Integrada de Desarrollo Agéntico

> Un modelo decente con un gran harness supera un gran modelo con un mal harness.

---

## La ecuación central

```
Agente = Modelo + Harness
```

**Si vos no sos el modelo, sos el harness.** — Viv Trivedy

El harness es todo lo que rodea al modelo: prompts, herramientas, hooks, sandboxes, subagentes, loops de feedback, políticas de contexto y paths de recuperación. La disciplina de diseñar, operar y evolucionar ese sistema se llama **Harness Engineering**.

Este repositorio integra tres corrientes en una metodología unificada y agnóstica de plataforma:

| Fuente | Qué aporta |
|---|---|
| **Spec Driven Development** (SDD) | Spec como artefacto primario; `.context/` + instrucciones + agentes especializados |
| **SDLC Agéntico** (GH-600) | Governance, multi-agente, protección, observabilidad en el ciclo de desarrollo |
| **Harness Engineering** (Osmani, Anthropic, Trivedy, HumanLayer) | Ratchet, hooks, context engineering, evaluación adversarial, HaaS |

## Mapa de navegación

| # | Módulo | Tema |
|---|---|---|
| 01 | [Fundamentos](01-fundamentos.md) | Principios, evolución TDD→SDD→HE, los 3 pilares |
| 02 | [Anatomía del Harness](02-anatomia-harness.md) | Componentes, taxonomía, mapeo comportamiento→componente |
| 03 | [Ratchet y Memoria](03-ratchet-y-memoria.md) | El trinquete, tipos de memoria, persistencia vía VCS |
| 04 | [Context Engineering](04-context-engineering.md) | Context rot, compaction, resets, progressive disclosure |
| 05 | [Evaluación Adversarial](05-evaluacion-adversarial.md) | Generator/Evaluator, sprint contracts, causa raíz |
| 06 | [Orquestación](06-orquestacion.md) | 4 patrones multiagente, conflictos, circuit breaker |
| 07 | [Protección y Autonomía](07-proteccion-autonomia.md) | Riesgo, autonomía, capas de protección, hooks |
| 08 | [Casos de Uso](08-casos-de-uso.md) | 6 escenarios prácticos con snippets agnósticos |
| — | [Cheat Sheet](cheat-sheet.md) | Referencia rápida, checklist, anti-patrones |
| — | [Mapeo de Plataformas](mapeo-plataformas.md) | Equivalencias GitHub / Azure DevOps / GitLab |

## Relación con agentic-init

[`agentic-init`](../agentic-init/) es la **implementación práctica** de esta metodología: un scaffold que genera la estructura de archivos, instrucciones y contexto. Esta guía es el **por qué** y el **cómo**; agentic-init es el **qué** concreto.

## Fuentes

- [Addy Osmani — Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/) (abril 2026)
- [Anthropic — Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) (marzo 2026)
- [Viv Trivedy — Anatomy of an Agent Harness](https://x.com/Vtrivedy10/status/2031408954517971368)
- [HumanLayer — Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- [Birgitta Böckeler / Martin Fowler — Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Simon Willison — Designing Agentic Loops](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/)
