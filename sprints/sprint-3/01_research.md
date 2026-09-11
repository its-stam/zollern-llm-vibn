# ZOLLERN Research Report

**Datum:** 2026-05-11 | **Agent:** researcher (Pipeline Step 1/3)
**Korrigiert:** 16.05.2026

> **Korrekturhinweis (16.05.2026):** Das Erath-Architektur-Diagramm in Abschnitt 3
> war falsch dargestellt und wurde anhand des Original-Bildes korrigiert. Detail:
> `meetings/Erath_Agent_Architecture_2026-05-08.md`.

---

## 1. Current Workflow Analysis (SCRUM-40)

### Phase 1: Analyse-Workflow (`workflow_phase1_analyse.json`)
- **12 nodes**, 4 LLM calls (3 parallel + 1 summary)
- **Modell:** claude-sonnet-4-6 via Anthropic API (HTTP POST)
- **CAD:** Fiktiver "Foerderer_Drehtisch_v1" mit 6 Komponenten (K001-K006)
- Pipeline: Manual Trigger -> CAD vorbereiten -> 3 parallele Agenten (Klassifikation, Importformat, Simulationslogik) -> Tag-Wrapper (robust gegen Reihenfolge) -> Merge -> Aufbereiten -> Bericht-Generator -> persist to disk
- **Kosten:** ~$0.05 pro Lauf (6 Komponenten), ~$0.30-0.50 (50+ Komponenten geschaetzt)
- **MCP Tools:** Keine (Phase 1 ist reiner Analyse-Workflow, keine Transformation)
- **v2 Patches:** Tag-basiertes Mapping (statt mergeByPosition), Tool-Use erzwingt JSON, Retry 3x, System/User-Prompt-Split, B/C orthogonal, SPS-Halluzination geblockt, Disk-Persistenz

### Phase 2: Transform-Workflow (`workflow_phase2_transform.json`)
- **18 nodes**, 7 LLM calls (Plan, 3 parallel, Logic Builder, Validator + extra)
- **CAD:** "2-Foerderer_1-Drehtisch" mit 9 Objekten, Inventor User Defined Properties
- Pipeline: Manual Trigger -> CAD vorbereiten -> Agent 1: Plan -> 3 parallele Agenten (Renamer, Typer, Joint Configurer) + Tag Plan -> Merge Transform (4 Inputs) -> Aggregate -> Agent 5: Logic Builder -> Sammle alle Calls -> **MCP Executor (PLATZHALTER)** -> Agent 6: Validator -> persist (3 Dateien: .md + .calls.json + .meta.json)
- **11 MCP Tools spezifiziert:** import_cad_file, get_all_objects, rename_object, set_object_type, set_property, configure_motionjoint, create_button, create_logic_object, link_objects, documentation_execute (optional), content_help/object_help (optional)
- **Tool Calls fuer Foerderer-Szenario:** ~31 (6 rename + 6 type + 1 joint + 18 logic)
- **Kritische Ausfuehrungsreihenfolge:** Phase 0 (import+get) -> Phase 1 (rename) -> Phase 2 (type+property) -> Phase 3 (joint) -> Phase 4 (buttons+logic+links)
- **Status:** Architektonisch fertig. MCP Executor ist Platzhalter -- wartet auf Gruppe B (MCP) fuer live fe.screen-sim Endpunkt.

---

## 2. Scenario Mapping: Foerderer/Drehtisch -> Hochregallager

### Current (Foerderer/Drehtisch)
- 6-9 Komponenten, 1D/2D Bewegung (Foerderbaender, Drehtisch)
- Surface-Typen: Foerdertechnik, Drehtisch, Sensor, Antrieb, Tragstruktur
- Simple state machine: IDLE -> TRANSPORT -> HANDOFF -> ROTATION -> STOP
- 6 SPS-Variablen

### Target (Hochregallager -- aus Besprechungsprotokoll 06.05.2026)
- **Kran** als zentrales Bewegungselement, 3D-Gantry (X/Y/Z)
- **Regalplaetze** -- organisiertes Raster
- **FIFO-Prinzip** -- Einlagerstrategie
- **High Runner / Low Runner** -- frequenzbasierte Optimierung
- **Taktzeiten** -- Durchlaufzeitanalyse
- **Real use case:** Lagerung von Werkstuecktraegern in der LKW-Batteriefertigung

### Strukturelle Unterschiede
| Aspekt | Foerderer/Drehtisch | Hochregallager |
|--------|-------------------|----------------|
| Bewegungsachsen | 1D Foerderband + 1D Rotation | 3D Kran (X/Y/Z) |
| Komponenten | ~6-9 | Potentiell 50+ |
| Bewegungstyp | Kontinuierlich (Band) | Diskret positionierend (Kran) |
| Lagerlogik | Keine | FIFO-Queues, High/Low Runner |
| SPS-Komplexitaet | Einfach (State Machine) | Komplex (Positionsregler) |
| Oberflaechentypen | Foerderflaechen | KranGreifer, Regalboden |

### Neue MCP Tools fuer Hochregallager

**Kran-spezifisch (NEU):**
- `configure_crane(params)` -- Gantry-Kran initialisieren
- `set_crane_position(crane_id, target_x, target_y, target_z)` -- Position anfahren
- `set_crane_gripper(crane_id, mode)` -- Greifer oeffnen/schliessen
- `define_axis_params(axis, velocity, acceleration)` -- Achs-Parameter

**Lagerlogik (NEU):**
- `define_storage_rack(rack_id, grid_dimensions)` -- Regalraster definieren
- `register_shelf_position(rack_id, position_id, coordinates)` -- Regalplatz registrieren
- `configure_fifo_queue(queue_id, rack_ids)` -- FIFO-Logik definieren
- `set_high_low_runner(article_id, category)` -- Runner-Klassifikation

**Erweiterte bestehende Tools:**
- `set_object_type` braucht neue Typen: "CraneAxis", "Gripper", "Shelf", "Payload"
- `set_property` braucht: `DetectLoad`, `MaxHeight`, `PositionPrecision`
- Neue Logik-Typen jenseits von OR_Velocity: Sequence Logic, Position Comparison, FIFO Controller, Runner Router

---

## 3. Erath Architecture Mapping (08.05.2026)

### Erath Architecture (korrigiert 16.05.2026):
```
LLM (Agent) <-> MCP DOKU <-> API <-> C# Tool
```
- LLM (Agent) links; der LLM-Knoten ist als Agent aufgebaut: LLM + DB (Datenbank) + Rules (Regeln)
- MCP DOKU: zwei gestapelte Boxen (MCP oben, DOKU darunter), oben mittig
- API rechts; Core ueberlappt die obere rechte Ecke der API, ohne eigenen Pfeil
- C# Tool: zwei Boxen nebeneinander (C# und Tool), rechts unterhalb der API
- alle Verbindungen sind beidseitige Pfeile

Detail: `meetings/Erath_Agent_Architecture_2026-05-08.md`

### SCRUM-40 -> Erath Mapping

| Erath Component | SCRUM-40 Implementation | Status |
|-----------------|------------------------|--------|
| **LLM** | Claude (Anthropic API) via HTTP Request Nodes | DONE (aber hardcoded) |
| **MCP DOKU** | MCP_TOOLS_SPEC.md (static file) | PARTIAL -- kein live Server |
| **API** | Anthropic HTTP + (future) fe.screen-sim MCP | PARTIAL -- MCP Endpunkt fehlt |
| **Core** | n8n Orchestrierung (Merge, Aggregate, Code) | DONE |
| **C# Tool** | fe.screen-sim implementierung (Gruppe B (MCP)) | NOT IN SCOPE (extern) |
| **DB** | Keine persistente Datenbank in SCRUM-40 | MISSING |
| **Rules** | Regeln in System Prompts eingebettet | PARTIAL -- nicht als separate Engine |

### Kritische Gaps

1. **MCP DOKU:** Nur statische Markdown-Datei statt live Tool-Registry. LLM kann Tools nicht dynamisch entdecken.
2. **DB:** Kein persistentes Agentenwissen. Jeder Lauf ist stateless. Kein Cross-Session-Lernen.
3. **Rules:** Business-Regeln sind in Prompts versteckt. Nicht versionierbar, nicht unabhaengig testbar.
4. **Model-Agnostizitaet:** Hardcoded auf claude-sonnet-4-6.

---

## 4. US 9 Multi-Model Requirements

### Background (Sommer-Feedback 01.05.2026)
> "Selben Workflow isoliert mit verschiedenen APIs testen ob Ergebnis gleich ist -- GPT, Gemini. Berichte vergleichen."

**Modelle:** Claude Sonnet, ChatGPT-5, Gemini 3

### API-Unterschiede (massiv)

| Aspekt | Claude (Anthropic) | GPT-5 (OpenAI) | Gemini 3 (Google) |
|--------|-------------------|-----------------|-------------------|
| Endpoint | api.anthropic.com/v1/messages | api.openai.com/v1/chat/completions | generativelanguage.googleapis.com |
| Auth | x-api-key | Authorization: Bearer | API Key/Bearer |
| System Prompt | Eigenes Feld "system" | role: "system" in messages | In user messages (kein eigenes Feld) |
| Tool Calling | tools + tool_choice Block | tools + tool_choice | tools + toolConfig |
| Structured Output | tool_choice: {type: "tool"} | response_format: json_object + tools | response_mime_type: json + tools |
| Response Format | content[{type: "tool_use"}] | choices[0].message.tool_calls | candidates[0].content.parts[0].functionCall |

### n8n-Modell-Switching
- n8n hat keinen eingebauten LLM-Switcher (alle Calls sind raw HTTP)
- Aktuell: Jeder Modellwechsel erfordert manuelle Editierung von 4-7 HTTP-Nodes
- Moegliche Architektur: Switch-Node routet zu Sub-Workflow pro Modell

### Vergleichsmetriken (vorgeschlagen)
1. **Strukturelle Korrektheit** -- Wurden alle Komponenten klassifiziert?
2. **Halluzinationsrate** -- Erfundene SPS-Adressen vs TBD-Marker
3. **Token-Effizienz** -- Input/Output-Tokens pro Modell
4. **Antwortzeit** -- Latenz pro Modell
5. **Kosten pro Lauf** -- Tokens * modellspezifischer Preis
6. **Menschliche Bewertung** -- Welcher Bericht gefaellt Zollern besser?

---

## 5. Gaps & Open Questions

### Technische Gaps
1. MCP Executor ist Platzhalter (wartet auf Gruppe B, Grp B)
2. MCP DOKU nur statisches Markdown (kein live Server)
3. Kein DB/Wissen-System (stateless, kein Lernen)
4. Rules in Prompts eingebettet (nicht versioniert/testbar)
5. Nur Anthropic (kein GPT/Gemini-Vergleich)
6. Simulierte CAD-Daten (keine echten Zollern-Daten)
7. Manueller Trigger (keine Pipeline-Automatisierung)
8. Lokale Datei-Ausgabe (kein Team-Sharing)
9. Phase 2: Chain-Agent-Fehler brechen gesamte Pipeline ab (kein Checkpoint/Rollback)

### Szenario-Gaps (Hochregallager)
1. Aktuelle MCP-Tools haben keinen Kran/Gantry-Support
2. Aktuelle Logik-Typen haben kein FIFO/Sequencing
3. Kein Regalraster-Konzept in MCP-Tools
4. High/Low Runner nicht abbildbar
5. Komponenten-Anzahl steigt 5-10x (Token-Limits gefaehrdet)

### Offene Fragen fuer Gruppe B (MCP) / F.EE
1. Exakter API-String fuer `logic_type: "OR_Velocity"`?
2. JointScale [0.5,0.5,0.5] fuer RevoluteJoint und PrismaticJoint gleich?
3. Port-Name fuer Link 6 (Active -> Active) korrekt?
4. MCP-Server Base-URL und Auth-Methode?
5. Muss fe.screen-sim offen sein bei MCP-Call?
6. Batch-API moeglich?
7. Unterstuetzt MCP-Server Kran/Regal/Gantry-Tools?

### Offene Fragen fuer Product Owner (Kundenseite) / Zollern
1. Welche Inventor User Defined Properties in echten CAD-Dateien?
2. Ist Hochregallager das finale Szenario?
3. Gibt es reale CAD-Daten (STEP)?
4. Welche SPS-Steuerung (Siemens, Beckhoff)?
5. Taktzeiten: ms oder s-Bereich?

---

## 6. Raw Node Inventory (Alle Workflows)

| Workflow | Nodes | LLM Calls | MCP Tools | Status |
|----------|-------|-----------|-----------|--------|
| Phase 1 v1 (zollern_n8n_workflow_v1.json) | 10 | 4 | 0 | Deprecated |
| Phase 1 v2 (workflow_phase1_analyse.json) | 12 | 4 | 0 | Ready |
| Phase 2 v1 (zollern_n8n_workflow_phase2_v1.json) | 17 | 6 | 11 | Deprecated (Plan Bug) |
| Phase 2 v2 (workflow_phase2_transform.json) | 18 | 7 | 11 | Ready, kein MCP Endpunkt |
