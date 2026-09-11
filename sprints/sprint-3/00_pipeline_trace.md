# ZOLLERN Sprint 3 — Pipeline Trace

**Datum:** 2026-05-11
**Pipeline:** 3-Agent (researcher → architect → coder)
**Korrigiert:** 16.05.2026

> **Korrekturhinweis (16.05.2026):** Dieser Trace dokumentiert einen Pipeline-Lauf
> mit zwei Designfehlern, die spaeter korrigiert wurden: (1) das Erath-Diagramm war
> falsch herum, (2) US9 wurde als interner Fan-out (3 Modelle parallel in einem
> Workflow) entworfen statt als isolierte Laeufe (derselbe Workflow je einmal pro
> Modell). Betroffene Stellen sind unten mit [KORRIGIERT] markiert.

---

## Agent Pipeline

### Step 1: Researcher

| Feld | Wert |
|------|------|
| **Agent** | `researcher` |
| **Input** | SCRUM-40 Workflows (Phase 1 + 2), Erath Architecture, Besprechungsprotokolle |
| **Output** | `01_research.md` |
| **Status** | DONE |

**Key Findings:**
- Foerderer/Drehtisch → Hochregallager: 6 strukturelle Unterschiede
- Erath Architecture: 7 Layer, 4 Gaps (MCP DOKU, DB, Rules, Model-Agnostizitaet)
- US 9: 5 API-Unterschiede, 6 Metriken, 10 technische Gaps
- Phase 2: 18 Nodes, 7 LLM Calls, MCP Executor ist Platzhalter

### Step 2: Architect

| Feld | Wert |
|------|------|
| **Agent** | `architect` |
| **Input** | `01_research.md` |
| **Output** | `02_architecture.md` |
| **Status** | DONE |

**Key Decisions:**
- Hochregallager-Klassifikation: 11 Typen (crane_system, storage_system, material_flow, control, sensor)
- Kran-Kinematik: 3x PrismaticJoint (X/Y/Z) + Gripper
- US 9: [KORRIGIERT 16.05.] isolierte Laeufe — derselbe Workflow je einmal pro Modell, kein Parallel-HTTP-Fan-out
- Gewichtete Bewertungsformel: 6 Metriken

### Step 3: Coder (current)

| Feld | Wert |
|------|------|
| **Agent** | `coder` |
| **Input** | `02_architecture.md` + `01_research.md` |
| **Output** | 7 Deliverables |
| **Status** | IMPLEMENTING |

---

## Deliverable Manifest

| # | File | Type | Status | Description |
|---|------|------|--------|-------------|
| 1 | `workflow_phase1_hochregallager.json` | n8n Workflow | WRITING | Phase 1 Analyse — [KORRIGIERT] US9 nicht intern eingebaut; US9 = isolierte Laeufe |
| 2 | `workflow_phase2_hochregallager.json` | n8n Workflow | WRITING | Phase 2 Transform — [KORRIGIERT] US9 nicht intern eingebaut; US9 = isolierte Laeufe |
| 3 | `us9_model_comparison.md` | Dokumentation | DONE | US 9 Spezifikation + Metriken |
| 4 | `us9_comparison_schema.json` | JSON Schema | DONE | Validierungsschema fuer Ergebnisse |
| 5 | `us9_prompts.md` | Dokumentation | WRITING | Standardisierte Prompts |
| 6 | `agentic_ai_erath_mapping.md` | Dokumentation | WRITING | Erath Architektur-Mapping |
| 7 | `00_pipeline_trace.md` | Dokumentation | WRITING | Pipeline-Trace (diese Datei) |

---

## Architektur-Entscheidungen (getroffen)

| ID | Entscheidung | Begruendung |
|----|-------------|-------------|
| AD-1 | Parallel HTTP statt Sub-Workflow | Debugging, Transparenz |
| AD-2 | In-Memory-JSON → Datei statt SQLite | Einfach, kein externes Setup |
| AD-3 | [KORRIGIERT 16.05.] US9 = isolierte Laeufe, ein Modell pro Lauf | Sommer-Vorgabe: selben Workflow isoliert testen, kein Fan-out |
| AD-4 | Node-Struktur identisch zu SCRUM-40 | Nur CAD-Fixture + Taxonomy geaendert |
| AD-5 | Modell-spezifische Tag-Wrapper | 3 verschiedene Response-Formate normalisieren |

---

## Open Points

| Punkt | Verantwortlich | Status |
|-------|---------------|--------|
| MCP Executor Endpunkt | Gruppe B (MCP) | BLOCKED |
| Live MCP DOKU | Gruppe B (MCP) | BLOCKED |
| Jira Tickets (SCRUM-44 ff.) | Product Owner (Kundenseite) | OPEN |
| Peer Review | Product Owner (Kundenseite) | PENDING |
| Echte CAD-Daten von Zollern | PO | OPEN |
| API Keys GPT + Gemini | Rustam | OPEN |
| fe.screen-sim V5 verfuegbar | Gruppe B (MCP) | OPEN |

---

## Commit History (sprints/sprint-3/)

| Date | File | Description |
|------|------|-------------|
| 2026-05-11 | `01_research.md` | Research Report |
| 2026-05-11 | `02_architecture.md` | Architecture Design |
| 2026-05-11 | `us9_model_comparison.md` | US 9 Spec |
| 2026-05-11 | `us9_comparison_schema.json` | Comparison Schema |
| 2026-05-11 | `00_pipeline_trace.md` | Pipeline Trace |
| 2026-05-11 | `workflow_phase1_hochregallager.json` | Phase 1 + US 9 |
| 2026-05-11 | `workflow_phase2_hochregallager.json` | Phase 2 + US 9 |
| 2026-05-11 | `us9_prompts.md` | Standardized Prompts |
| 2026-05-11 | `agentic_ai_erath_mapping.md` | Erath Architecture Mapping |
