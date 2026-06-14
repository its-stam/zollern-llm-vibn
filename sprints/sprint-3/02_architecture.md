# ZOLLERN Sprint 3 Architecture Design

**Datum:** 2026-05-11 | **Agent:** architect (Pipeline Step 2/3)
**Basis:** Research Report (01_research.md)
**Korrigiert:** 16.05.2026

> **Korrekturhinweis (16.05.2026):** Zwei Korrekturen gegenueber der Erstfassung.
> (1) Section 2: US9 war als interner Fan-out (3 Modelle parallel in einem Workflow)
> beschrieben — korrigiert auf isolierte Laeufe (derselbe Workflow je einmal pro
> Modell), gemaess Sommer-Vorgabe. (2) Section 3: das Erath-Diagramm war falsch
> herum — korrigiert anhand des Original-Bildes
> (`meetings/Erath_Agent_Architecture_2026-05-08.md`).

---

## Architecture Overview

```
[CAD-Daten (Hochregallager)] --> [n8n Orchestrator]
                                       |
                    +------------------+------------------+
                    |                  |                  |
             Phase 1 Analyse    US 9 Multi-Model   Phase 2 Transform
                    |                  |                  |
            Analyse-Bericht    Vergleichsbericht   MCP Executor
                                                         |
                                                  fe.screen-sim V5
                                                         |
                                                  Validator Output
```

---

## Section 1: Hochregallager Workflow Design

### Phase 1: Analyse-Workflow (12 Nodes, adapted from SCRUM-40)

**Basis:** `workflow_phase1_analyse.json`

| Node | SCRUM-40 (Foerderer/Drehtisch) | Sprint 3 (Hochregallager) |
|------|-------------------------------|--------------------------|
| 1. Manual Trigger | Keine Aenderung | Unchanged |
| 2. CAD vorbereiten | Foerderer_Drehtisch_v1 | Hochregallager_mit_RBG_v1 |
| 3. Agent A (Klassifikation) | 6 Foerderer-Typen | Kran, Regalplatz, WT, EAS, Sensor |
| 4. Agent B (Importformat) | Foerdertechnik-Struktur | Gantry-Struktur (X/Y/Z) |
| 5. Agent C (Simulationslogik) | State Machine (IDLE->TRANSPORT) | FIFO-Queue + Positionierung |
| 6-8. Tag wrapper A/B/C | Unchanged | Unchanged |
| 9. Merge | Tag-basiert | Unchanged |
| 10. Aufbereiten | Property-Name Mapping | Gleiches Pattern |
| 11. Bericht-Generator | Markdown-Bericht | Markdown + Komponenten-Tabelle |
| 12. persist | .md + Meta | .md + .json + Meta |

### Component Type Mapping: Foerderer -> Hochregallager

| Alt (Foerderer/Drehtisch) | Neu (Hochregallager) | Properties |
|---------------------------|----------------------|------------|
| Foerderband gerade | Kran X-Achse (Fahrschiene) | Laenge, Geschw, Position |
| Foerderband Kurve | Kran Y-Achse (Katzfahrt) | Hub, Geschw, Position |
| Drehtisch | Kran Z-Achse (Hub) | Hoehe, Geschw, Hubkraft |
| Sensor | Greifer (oeffnen/schliessen) | Typ, OEffnungsweite |
| Antrieb | Regalplatz-Array | Raster, Kapazitaet |
| Tragstruktur | Regalboden | Traglast, Position |
| NEU | Werkstuecktraeger (WT) | Typ, Gewicht, Groesse |
| NEU | Ein-/Ausgabeplatz (EAS) | Typ, Position |
| Steuerung | FIFO-Controller | Strategie (FIFO) |
| Steuerung | Runner-Klassifizierer | High/Low Threshold |

### Neue Klassifikations-Taxonomie

```
crane_system:
  - crane_x_axis (Fahrschiene)
  - crane_y_axis (Katzfahrt)
  - crane_z_axis (Hub)
  - gripper (Greifer)
storage_system:
  - rack (Regal)
  - shelf (Regalboden)
  - shelf_position (Regalplatz)
material_flow:
  - workpiece_carrier (Werkstuecktraeger)
  - input_output_station (Ein-/Ausgabe)
control:
  - fifo_controller (FIFO-Steuerung)
  - runner_classifier (Runner-Klassifikation)
sensor:
  - position_sensor (Positionssensor)
  - load_sensor (Lastsensor)
```

### Phase 2: Transform-Workflow (18 Nodes, adapted from SCRUM-40)

**Basis:** `workflow_phase2_transform.json`

| Node | SCRUM-40 | Sprint 3 |
|------|----------|----------|
| 1. Manual Trigger | Unchanged | Unchanged |
| 2. CAD vorbereiten | 2-Foerderer-1-Drehtisch | Hochregallager_v1 |
| 3. Agent 1: Plan | alter Plan-Prompt | erweiterter Plan-Prompt (3D + FIFO) |
| 4. Agent 2a: Renamer | alte Namen | Kran-Komponenten |
| 5. Agent 2b: Typer | alte Typen | Gantry-Typen |
| 6. Agent 2c: Joint Config | 1 Drehtisch (Revolute) | 3 Achsen (Prismatic) |
| 7. Tag Plan | Unchanged | Unchanged |
| 8. Merge Transform | 4 Inputs | Unchanged |
| 9. Aggregate | aggregateAllItemData | Unchanged |
| 10. Agent 3: Logic Builder | altes Logic-Prompt | erweitert (FIFO + Sequenz) |
| 11. Sammle alle Calls | Unchanged | Unchanged |
| 12. MCP Executor | PLATZHALTER | PLATZHALTER (wartet auf Gruppe B) |
| 13. Agent 4: Validator | alter Validator | erweiterter Validator (3D-Gantry-Check) |
| 14-16. persist | .md + .calls.json + .meta.json | Unchanged |

### Neue Kran-Joint-Konfiguration

```json
{
  "x_axis": {"joint_type": "PrismaticJoint", "axis": "X", "range": [0, 30000], "velocity": 2.0},
  "y_axis": {"joint_type": "PrismaticJoint", "axis": "Y", "range": [0, 15000], "velocity": 1.5},
  "z_axis": {"joint_type": "PrismaticJoint", "axis": "Z", "range": [0, 10000], "velocity": 0.8},
  "gripper": {"joint_type": "PrismaticJoint", "axis": "G", "range": [0, 500], "velocity": 0.3}
}
```

### Neue Logic-Typen

| Logic Type | Beschreibung |
|-----------|-------------|
| `Sequence` | Abfolge von Bewegungen |
| `PositionCompare` | Positionsvergleich |
| `FIFOController` | FIFO-Queue-Logik |
| `RunnerRouter` | High/Low-Routing |
| `GantryMove` | 3D-Koordinaten-Bewegung |
| `GripperControl` | Greifer steuern |

---

## Section 2: US 9 Multi-Model-Vergleich (isolierte Laeufe)

> **Korrigiert 16.05.2026:** Die Erstfassung beschrieb US9 als "Parallel HTTP
> Request Nodes" — drei Modelle parallel in einem Workflow. Das widerspricht
> Sommers Vorgabe. US9 = derselbe Workflow, isoliert je einmal pro Modell.

### Vorgabe (Sommer, Sprint-2-Review 01.05.2026)

> "Selben Workflow isoliert mit verschiedenen APIs testen ob Ergebnis gleich ist
> — GPT, Gemini. Berichte vergleichen. Was gefaellt dem Kunden am Ende besser?"

Modelle: Claude Sonnet, ChatGPT-5, Gemini 3.

### Architektur: isolierte Laeufe (kein Fan-out)

```
Gleiche CAD-Eingabe (Hochregallager)
  |
  +-- Lauf A: Workflow mit claude-sonnet  -> bericht_claude.md
  +-- Lauf B: Workflow mit chatgpt-5      -> bericht_gpt.md
  +-- Lauf C: Workflow mit gemini-3       -> bericht_gemini.md
  |
  v
  Vergleichs-Schritt -> strukturierter Vergleichsbericht
```

Derselbe Workflow wird dreimal isoliert ausgefuehrt, ein Modell pro Lauf, gleiche
CAD-Eingabe. Jeder Lauf erzeugt einen eigenen Bericht. Ein Vergleichs-Schritt
stellt die drei Berichte gegenueber. Es laufen nie mehrere Modelle gleichzeitig
in einem Workflow.

### Implementierung

Die drei Provider haben unterschiedliche Endpunkte und Request-Bodies (siehe
API-Konfiguration). Daher drei separate Workflow-Varianten, je eine pro Provider,
oder ein Workflow dessen HTTP-Agent-Nodes pro Lauf auf den Provider umgestellt
werden. Pro Lauf laeuft nur ein Modell.

### API-Konfiguration

| Parameter | Claude | GPT | Gemini |
|-----------|--------|-----|--------|
| Endpoint | /v1/messages | /v1/chat/completions | /v1/models/gemini-3-pro:generateContent |
| Auth | x-api-key | Authorization: Bearer | x-goog-api-key |
| System | eigenes Feld | role: "system" | In user content |
| Tool-Call | tool_choice: {type: "tool"} | tool_choice: "required" | toolConfig: {...} |
| Parse | content[].tool_use | choices[0].message.tool_calls | candidates[0].content.parts[0].functionCall |

### Gewichtete Bewertungsformel

```
Score = 0.30 * strukturelle_korrektheit
      + 0.20 * (1 - halluzinationsrate)
      + 0.10 * token_effizienz
      + 0.10 * (1 - latency/max_latency)
      + 0.10 * (1 - cost/max_cost)
      + 0.20 * qualitaet
```

### Vergleichsmetriken

1. **Strukturelle Korrektheit** = korrekt_klassifiziert / gesamt_komponenten
2. **Halluzinationsrate** = erfundene_sps_adressen / gesamt_sps_adressen
3. **Token-Effizienz** = output_tokens_nutzbar / output_tokens_gesamt
4. **Antwortzeit** (Latenz pro Modell)
5. **Kosten pro Lauf**
6. **Menschliche Bewertung** (Zollern-Feedback)

---

## Section 3: Agentic AI & Erath Architecture Mapping

### Erath Architecture (Wolfgang Erath, 08.05.2026 — korrigiert 16.05.2026)

```
LLM (Agent) <-> MCP DOKU <-> API <-> C# Tool
```

- LLM (Agent) links; der LLM-Knoten ist als Agent aufgebaut: LLM + DB + Rules
- MCP DOKU: zwei gestapelte Boxen (MCP oben, DOKU darunter), oben mittig
- API rechts; Core ueberlappt die obere rechte Ecke der API, ohne eigenen Pfeil
- C# Tool: zwei Boxen nebeneinander (C# und Tool), rechts unterhalb der API
- alle Verbindungen sind beidseitige Pfeile

Detail: `meetings/Erath_Agent_Architecture_2026-05-08.md`

### Sprint 3 -> Erath Mapping

| Erath Component | Sprint 3 Implementation | Status |
|----------------|------------------------|--------|
| LLM | Sprachmodell-Kern via HTTP Request; fuer US9 isoliert mit Claude Sonnet / GPT-5 / Gemini 3 | Adaptiert |
| DB (Wissen) | n8n-Code-Node: In-Memory-JSON | Teilweise |
| DB (Persistenz) | Dateisystem (.md, .json, .meta) | Teilweise |
| Rules (Klassifikation) | System-Prompt: Komponenten-Taxonomie | Eingebettet |
| Rules (Logik) | System-Prompt: Transform-Regeln | Eingebettet |
| Rules (FIFO) | System-Prompt: Lagerstrategie | Neu |
| MCP DOKU | MCP_TOOLS_SPEC.md (statisch) | Teilweise |
| MCP DOKU (live) | fe.screen-sim MCP Endpunkt | Wartet auf Gruppe B |
| API (n8n) | n8n HTTP Request Nodes | Done |
| API (MCP) | fe.screen-sim Bridge | Wartet auf Gruppe B |
| Core | n8n Orchestrierung | Adaptiert |
| C# Tool | fe.screen-sim V5 | Extern (Grp B) |

GPT-5 und Gemini 3 sind keine zusaetzlichen LLM-Komponenten, sondern alternative
Modelle fuer die drei isolierten US9-Laeufe.

### Multi-Agent-Rollen (SCRUM-40 + Sprint 3)

| Rolle | Erath Layer | Prompt-Komplexitaet |
|-------|------------|-------------------|
| Plan Agent | LLM + Rules | Hoch (Erstellt Plan) |
| Renamer | LLM | Niedrig (Bulk-Rename) |
| Typer | LLM | Niedrig (Type-Set) |
| Joint Config | LLM | Mittel (Joint-Params) |
| Logic Builder | LLM + Rules | Hoch (Buttons + Logik) |
| Validator | LLM + Rules | Mittel (Validierung) |

### Coverage Matrix

| Anforderung | Phase 1 | Phase 2 | US 9 | Erath | Status |
|------------|---------|---------|------|-------|--------|
| Hochregallager-Klassifikation | x | | | x | Design |
| Kran-Kinematik (3D) | | x | | x | Design |
| FIFO-Logik | | x | | x | Design |
| Multi-Model-Vergleich (isolierte Laeufe) | | | x | x | Design |
| MCP-Integration | | Platzhalter | | x | Blocked |
| Agentic AI-Dokumentation | | | | x | Done |
| Persistente DB | | | | Gap | Open |
| Regel-Engine (extern) | | | | Gap | Open |
| Live-MCP-DOKU | | | | Gap | Open |

---

## Section 4: Key Architecture Decisions

| Entscheidung | Gewaehlt | Alternative | Grund |
|-------------|---------|-------------|-------|
| US9-Modellvergleich | isolierte Laeufe, ein Modell pro Lauf | interner Fan-out (3 Modelle/Workflow) | Sommer-Vorgabe; sauberer, isolierter Vergleich |
| DB-System | In-Memory-JSON -> Datei | SQLite/Postgres | Einfach, kein externes Setup |
| MCP DOKU | Statisch + evtl live | Nur live | Gruppe B liefert live, statisch als Fallback |
| Rules | In Prompts (aktuell) | Externes JSON | Iterative Migration |

### Implementation Checklist

#### Phase A: Workflow-Adaption
- [ ] Phase 1 CAD-Daten auf Hochregallager umstellen
- [ ] Alle 12 Nodes adaptieren
- [ ] Phase 2 CAD-Daten auf Hochregallager umstellen
- [ ] Alle 18 Nodes adaptieren
- [ ] Neue Joint-Konfiguration (3x Prismatic)
- [ ] Neue Logic-Typen (FIFO, Sequence)

#### Phase B: US 9 Multi-Model (isolierte Laeufe)
- [ ] Workflow-Variante Claude (claude-sonnet)
- [ ] Workflow-Variante GPT-5
- [ ] Workflow-Variante Gemini 3
- [ ] drei isolierte Laeufe mit gleicher CAD-Eingabe
- [ ] Vergleichs-Schritt ueber die drei Berichte

#### Phase C: Dokumentation
- [ ] workflow_phase1_hochregallager.json
- [ ] workflow_phase2_hochregallager.json
- [ ] us9_model_comparison.md
- [ ] us9_comparison_schema.json
- [ ] us9_prompts.md
- [ ] agentic_ai_erath_mapping.md
- [ ] 00_pipeline_trace.md

#### Phase D: Review
- [ ] Peer-Review mit Product Owner (Kundenseite)
- [ ] Abstimmung mit Gruppe B (MCP) (MCP)
- [ ] Praesentation 18.05.2026 Meeting
