# Propuesta: Software Factory (modelo Matt Pocock) para claude-forge

> **Estado:** 📝 PROPUESTA — pendiente de aprobación (HARD-GATE, golden principle #9). No se ha escrito código todavía.
> **Fecha:** 2026-10-28
> **Relación con SDD:** esta propuesta es la continuación natural del flujo Spec-Driven Development ya implantado (spec → plan → implementación revisada contra la spec). SDD resuelve *qué* construir y *cómo comprobarlo*; la Software Factory resuelve *quién arranca el trabajo y cuándo*.

---

## 1. Qué es una Software Factory (según Matt Pocock)

Definición literal de su *Dictionary of AI Coding* (`mattpocock/dictionary-of-ai-coding`, entrada *Software factory*):

> Un sistema de trabajo en el que las **sesiones de agente las arrancan disparadores (triggers)** — un issue creado o etiquetado, un cron, un fallo de CI, otra sesión que termina — en lugar de un humano, de modo que más trabajo corre **AFK** y la atención humana se reserva para las decisiones **human-in-the-loop** que quedan.

Ideas clave que hay que respetar:

| Concepto | Qué significa | Consecuencia de diseño |
|---|---|---|
| **Trigger → sesión** | Un evento, no una persona, abre la sesión del agente | Necesitamos disparadores: etiqueta `ready-for-agent`, cron, CI rojo, PR abierto |
| **AFK vs HITL** | AFK = el agente trabaja sin supervisión; HITL = tú estás en el bucle | Clasificar cada ticket como `ready-for-agent` o `ready-for-human` |
| **Turno de día / turno de noche** | Día: grilling, spec, tickets, revisión, QA. Noche: implementación AFK | El humano invierte su tiempo *antes* (alinear) y *después* (revisar), nunca *durante* |
| **El output es revisable** | Un AFK termina en un PR, no en cambios ya mergeados | Prohibido auto-merge a `main` |
| **No es un *dark factory*** | Dark factory = nadie lee el código producido | Siempre hay revisión humana antes de merge (al menos en código de riesgo) |
| **Empezar pequeño** | "Un cron que corre un tipo de sesión y abre un PR revisable ya es una factory" | Fase 1 = un único bucle estrecho |
| **Tres frenos contra el "slop cannon"** | 1) checks deterministas, 2) revisor IA con estándares, 3) atención humana solo donde hay riesgo | Pipeline de revisión en capas |

Piezas públicas de su implementación que tomamos como referencia:

- **`mattpocock/skills`** — `grill-with-docs` / `grilling`, `to-spec`, `to-tickets` (tracer bullets con *blocking edges*), `triage` (máquina de estados de etiquetas), `implement`, `implement-spec` (grafo de tickets + *frontier* + subagentes en worktrees + rama de integración), `code-review` (dos ejes: **Standards** y **Spec**, en subagentes paralelos), `tdd`, `retro`, `pr`.
- **`mattpocock/sandcastle`** — librería TS que ejecuta agentes en sandboxes (Docker/Podman/Vercel) sobre su propia rama y mergea los commits. Su plantilla `parallel-planner-with-review` es exactamente el turno de noche: **Planner** (calcula issues desbloqueados → JSON `<plan>`) → **Implementer** por issue (RGR, hasta 100 iteraciones, señal `<promise>COMPLETE</promise>`) → **Reviewer** en la misma sandbox → **Merger** (resuelve conflictos, pasa tests, cierra issues). Bucle externo hasta que no queda frontier.
- El repo `mattpocock/software-factory` y la rama `software-factory-skill` no son públicos ahora mismo; la propuesta se basa en las piezas públicas anteriores.

---

## 2. Punto de partida en claude-forge

Lo que ya tenemos y se reutiliza (no se duplica):

| Pieza existente | Rol en la factory |
|---|---|
| SDD: `spec.md`, `cc-dev-agent`, HARD-GATE de `plan.md` (principio #9), "SDD review obligatoria" (principio #11) | La **spec** de la factory. Se amplía con tickets y seams de test |
| `agents/planner.md`, `/plan` | Base del **Planner** diurno (HITL) |
| `agents/tdd-guide.md`, `/tdd` | Disciplina del **Implementer** |
| `agents/code-reviewer.md`, `/handoff-verify` (8 revisores) | **Freno 2** (revisor IA). Falta el eje *Spec* explícito |
| `agents/verify-agent.md`, `/verify-loop` | **Freno 1** (checks deterministas) |
| `/orchestrate` + `team-orchestrator`, `/worktree-start` | Paralelismo; se sustituye por worktrees + subagentes en background para no depender de Agent Teams experimental |
| `rules/verification.md` (evidencia antes de "hecho") | Criterio de salida de cada ticket |
| `continuous-learning-v2`, meta-mejora (principio #13) | Equivalente a `/retro` de Pocock |
| `setup/GLOSSARY.md` | Hoy es un glosario de usuario; la factory necesita un **glosario de dominio por proyecto** (`GLOSSARY.md` + ADRs) |

Huecos a cubrir: **disparadores**, **tracker con estados**, **tickets con dependencias**, **frontier/grafo**, **revisión en dos ejes**, **sandbox AFK**, **política HITL/AFK** y **toolchain Java/Python** (todo lo de Pocock asume TypeScript/npm).

---

## 3. Arquitectura propuesta

```
            TURNO DE DÍA (HITL)                               TURNO DE NOCHE (AFK)
 ┌──────────────────────────────────────┐        ┌──────────────────────────────────────────────┐
 │ /grill        → GLOSSARY.md + ADRs   │        │ Trigger (label / cron / CI rojo / PR abierto)│
 │ /to-spec      → spec (SDD) en issue  │        │            │                                 │
 │ /to-tickets   → tickets + blocked-by │──────▶ │  PLANNER   → frontier JSON (desbloqueados)   │
 │ /triage       → ready-for-agent |    │ label  │  IMPLEMENTER×N (worktree/sandbox, TDD)       │
 │                 ready-for-human      │        │  REVIEWER  (Standards ‖ Spec, en paralelo)   │
 └──────────────────────────────────────┘        │  MERGER    → rama de integración + PR draft  │
                 ▲                               └──────────────────────────────────────────────┘
                 │                                                │
 ┌──────────────────────────────────────┐                         ▼
 │ Revisión humana del PR (merge-danger │◀──── Freno 1: mvn/gradle/pytest/ruff/mypy (CI)
 │ call: puerta 1-vía / 2-vías) + /retro│◀──── Freno 2: revisor IA con CODING_STANDARDS.md
 └──────────────────────────────────────┘◀──── Freno 3: humano solo en riesgo (schema, auth, prod)
```

### 3.1 Máquina de estados del tracker (etiquetas)

Categoría: `bug` | `enhancement`. Estado: `needs-triage` → `needs-info` | `ready-for-agent` | `ready-for-human` | `wontfix`. Añadimos dos estados de factory: `agent-in-progress` (lock para que dos sesiones no cojan el mismo ticket) y `agent-failed` (la sesión agotó iteraciones o checks; vuelve a un humano).

### 3.2 Disparadores (por orden de implantación)

| # | Trigger | Sesión | Mecanismo |
|---|---|---|---|
| T1 | Cron nocturno | Recorre el frontier de `ready-for-agent` | Orquestador local o tarea programada |
| T2 | Issue etiquetado `ready-for-agent` | Implementación de ese ticket → PR | GitHub Action (`anthropics/claude-code-action`) |
| T3 | PR abierto por la factory | Revisión automática en dos ejes; sus comentarios disparan sesión de fix-up | GitHub Action |
| T4 | CI rojo en `main` | Diagnóstico (bisect del commit) + PR de propuesta | GitHub Action `workflow_run` |
| T5 | Mantenimiento por cron | "Una regla de lint / un warning de SpotBugs por noche" | Cron |

### 3.3 Motor de orquestación — decisión recomendada

| Opción | Pros | Contras |
|---|---|---|
| A. Sandcastle (TS) tal cual | Probado por Pocock, sandbox Docker resuelto | Añade Node/TS a proyectos Java/Python; plantillas asumen `npm run test` |
| **B. Orquestador Python propio** (`factory/run.py`) sobre `claude -p` headless + `git worktree` + Docker opcional | Encaja con tu stack (Java/Python), sin dependencias nuevas, mismo diseño de 4 fases y mismos prompts | Hay que mantenerlo (~300-500 líneas) |
| C. Solo GitHub Actions | Cero infraestructura local, triggers nativos | Minutos de CI, menos control del paralelismo, no vale para repos sin GitHub (p. ej. GitLab interno) |

**Recomendación:** B como motor (portando la lógica de `parallel-planner-with-review`: plan JSON validado con `pydantic`, `asyncio.gather` por ticket, señal de completado, `maxIterations`, timeout de inactividad) + C solo para los triggers T2–T4. Sandcastle queda como referencia de diseño y alternativa si algún proyecto es TS.

---

## 4. Entregables (archivos)

> claude-forge es un plugin distribuible: lo que va a los proyectos del usuario se entrega como **plantillas** en `setup/factory/`, no como workflows activos de este repo.

| Archivo | Tipo | Responsabilidad |
|---|---|---|
| `skills/factory-setup/SKILL.md` | Skill (user-invoked) | Configura un proyecto una vez: tracker (GitHub / GitLab / ficheros locales), mapa de etiquetas, ubicación de `GLOSSARY.md`/ADRs, comandos de checks (Maven/Gradle/pytest) |
| `skills/grilling/SKILL.md` | Skill (model-invoked) | Entrevista pregunta a pregunta hasta resolver el árbol de diseño; actualiza `GLOSSARY.md` y ADRs inline |
| `commands/grill.md` | Command | Punto de entrada HITL que invoca `grilling` + dominio |
| `commands/to-spec.md` | Command | Sintetiza la conversación en la **spec SDD existente** + seams de test, la publica en el tracker con `ready-for-agent` |
| `commands/to-tickets.md` | Command | Tracer bullets verticales con *blocked-by*, tamaño "cabe en una ventana de contexto"; expand–contract para refactors anchos |
| `commands/triage.md` | Command | Máquina de estados de §3.1 + `AGENT-BRIEF` + `.out-of-scope/` |
| `skills/implement-spec/SKILL.md` | Skill | Grafo de tickets → frontier → implementers en worktrees en background → merger → rama de integración → PR draft |
| `agents/spec-reviewer.md` | Agent | **Eje Spec**: requisitos ausentes/parciales, scope creep, implementado-pero-mal (citando la spec). Hace cumplir el principio #11 |
| `agents/code-reviewer.md` | Modificar | Añadir baseline de *smells* de Fowler y carga de `CODING_STANDARDS.md`; se ejecuta en paralelo con `spec-reviewer` sin mezclar ni re-rankear hallazgos |
| `rules/factory.md` | Rule | Política AFK/HITL, prohibición de auto-merge, cuándo un ticket es `ready-for-human` (migraciones, auth, datos personales, prod) |
| `setup/factory/run.py` + `prompts/{plan,implement,review,merge}.md` | Plantilla | Orquestador Python de 4 fases (§3.3 B) |
| `setup/factory/CODING_STANDARDS.md` | Plantilla | Estándares Java/Python que solo carga el revisor (no consume tokens al implementar) |
| `setup/factory/Dockerfile` | Plantilla | Sandbox con JDK (8/17/21 configurable), Maven/Gradle, Python + uv, `git`, `claude` |
| `setup/factory/github/*.yml` | Plantilla | Workflows T2–T4 |
| `commands/retro.md` | Command | Tras cada noche: propone mejoras del entorno (checks, estándares, CLAUDE.md), de más a menos grave; se conecta con `continuous-learning-v2` |
| `docs/SOFTWARE-FACTORY.md` | Doc | Guía de uso + glosario (trigger, AFK, HITL, frontier, tracer bullet, dark factory) |

---

## 5. Plan por fases

Cada fase termina en algo revisable y se puede parar ahí con valor entregado.

### Fase 0 — Alineación (HITL, ½ día)
- [ ] Responder las preguntas abiertas de §7.
- [ ] Localizar la implementación SDD actual (spec template, UI `sddui`) para que `to-spec` reutilice exactamente su formato.
- **Criterio:** esta propuesta pasa a `APPROVED`.

### Fase 1 — Turno de día (HITL), sin automatización (2-3 días)
- [ ] `factory-setup`, `grilling` + `/grill`, `/to-spec`, `/to-tickets`, `/triage`, `rules/factory.md`.
- [ ] Probar en un proyecto real Java y uno Python: de una idea a N tickets `ready-for-agent` con dependencias.
- **Criterio:** un ticket cualquiera puede implementarse en una sesión nueva *solo* con el ticket + punteros (spec, glosario, ADRs), sin preguntas.

### Fase 2 — Primera factory mínima: un trigger, un PR (2 días)
- [ ] Trigger T2 (etiqueta `ready-for-agent` → sesión → PR) con plantilla de workflow.
- [ ] `agents/spec-reviewer.md` + `code-reviewer` en dos ejes; revisión automática en el PR (T3, sin fix-up todavía).
- [ ] Lock `agent-in-progress` y salida `agent-failed`.
- **Criterio:** 5 tickets reales procesados AFK; se mide % de PRs aceptados sin cambios, con cambios menores y rechazados. Esto calibra cuánto se puede confiar antes de ampliar.

### Fase 3 — Turno de noche en paralelo (3-4 días)
- [ ] `setup/factory/run.py`: Planner → Implementers paralelos (worktrees, opcional Docker) → Reviewer → Merger → PR draft sobre rama de integración. Bucle hasta frontier vacío o `MAX_ITERATIONS`.
- [ ] `skills/implement-spec` para la variante interactiva (dentro de una sesión de Claude Code).
- [ ] Freno 1 configurable por proyecto: `mvn -q verify` / `gradle check` / `pytest -q && ruff check && mypy`.
- **Criterio:** una spec de ≥5 tickets con al menos 2 ramas paralelas termina en un único PR draft verde, revisable por la mañana.

### Fase 4 — Ampliar disparadores y cerrar el ciclo (2 días)
- [ ] T4 (CI rojo en `main` → diagnóstico), T5 (mantenimiento nocturno: una regla por noche), fix-up a partir de comentarios de revisión.
- [ ] `/retro` conectado a `continuous-learning-v2`.
- [ ] `docs/SOFTWARE-FACTORY.md`, actualizar inventario del README (commands/skills/agents) y `plugin.json`.
- **Criterio:** una semana de uso con informe: tickets procesados, PRs aceptados, coste en tokens, incidencias.

---

## 6. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| "Slop cannon": más PRs de los que se pueden revisar | Empezar con un solo trigger; tickets pequeños (una ventana de contexto); límite de PRs abiertos por la factory |
| AFK construye horas sobre una decisión errónea | Grilling + spec antes; ambigüedad ⇒ `ready-for-human`, nunca `ready-for-agent` |
| Deriva hacia *dark factory* | `rules/factory.md`: sin auto-merge; el PR incluye *merge-danger call* (puerta 1-vía/2-vías, radio de impacto) |
| Seguridad (permisos amplios en AFK) | Sandbox/worktree aislado, sin secretos de producción, hooks existentes (`remote-command-guard`, `db-guard`, `output-secret-filter`) activos también en AFK |
| Coste de tokens | Modelo potente solo para Planner; implementers con modelo más barato; `MAX_ITERATIONS` y timeout de inactividad |
| Proyectos legacy (JDK 8) | Imagen de sandbox con JDK configurable; checks definidos en `factory-setup`, no hardcodeados |

---

## 7. Preguntas abiertas (necesito tu respuesta para cerrar la Fase 0)

1. **¿Dónde vive la implementación SDD de referencia?** En este repo solo hay rastros (`cc-dev-agent`, principios #9 y #11). Si es `sddui-edu` (local, Windows), ¿la factory debe integrarse con esa UI (p. ej. un tablero de tickets/estado de la noche)?
2. **Tracker:** ¿GitHub Issues, GitLab interno, Jira, o ficheros locales (`.scratch/<feature>/issues/NN-*.md`)? Cambia cómo se implementan los triggers.
3. **Dónde corre el turno de noche:** ¿tu PC Windows (Docker Desktop disponible?), un servidor, o GitHub Actions?
4. **Alcance:** ¿la factory es parte del plugin claude-forge (plantillas reutilizables) o solo para tus proyectos concretos?
5. **Idioma** de prompts/skills: el repo mezcla coreano e inglés; ¿escribimos las nuevas piezas en inglés (mejor para el modelo) con docs en español?

---

## 8. Fuentes

Primarias (leídas directamente):
- `mattpocock/dictionary-of-ai-coding` — entradas *Software factory*, *Dark factory*, *AFK*, *Human-in-the-loop*, *Ticket*.
- `mattpocock/skills` — README y `to-spec`, `to-tickets`, `triage`, `implement`, `implement-spec`, `code-review`.
- `mattpocock/sandcastle` — README y plantilla `src/templates/parallel-planner-with-review` (`main.mts`, `plan/implement/review/merge-prompt.md`, `CODING_STANDARDS.md`).

Secundarias (resúmenes vía búsqueda, no verificadas en detalle):
- Posts de Matt Pocock en X sobre Sandcastle, AFK agents y la entrada *Software Factory* del diccionario.
- Charla "Fixing the PR Bottleneck" (AI Engineer) — los tres frenos contra el *slop cannon*.
- Algomatic Tech Blog (2026-10-22) sobre la skill `software-factory`.
