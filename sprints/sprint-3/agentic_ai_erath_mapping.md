# Agentic AI & Erath Architecture Mapping

**Datum:** 2026-05-11 | **Agent:** coder (Pipeline Step 3/3)
**Basis:** Erath Agent Architecture (08.05.2026), Sprint 3 Design
**Korrigiert:** 16.05.2026

> **Korrekturhinweis (16.05.2026):** (1) Das Erath-Diagramm in Abschnitt 1 war
> falsch herum dargestellt — korrigiert anhand des Original-Bildes. (2) US9 war als
> interner Fan-out (3 Modelle parallel pro Rolle) beschrieben — korrigiert auf
> isolierte Laeufe (derselbe Workflow je einmal pro Modell), gemaess Sommer-Vorgabe.
> Detail zum Diagramm: `meetings/Erath_Agent_Architecture_2026-05-08.md`.

---

## 1. Erath Architecture (Referenz)

```
LLM (Agent) <-> MCP DOKU <-> API <-> C# Tool
```

- LLM (Agent) links; der LLM-Knoten ist als Agent aufgebaut: LLM + DB + Rules
- MCP DOKU: zwei gestapelte Boxen (MCP oben, DOKU darunter), oben mittig
- API rechts; Core ueberlappt die obere rechte Ecke der API, ohne eigenen Pfeil
- C# Tool: zwei Boxen nebeneinander (C# und Tool), rechts unterhalb der API
- alle Verbindungen sind beidseitige Pfeile

### Erath Layer-Definitionen

> Hinweis: Erath lieferte nur das Diagramm, keine Layer-Definitionen. Die folgende
> Tabelle ist die Arbeitsinterpretation des Teams.

| Layer | Funktion | Verantwortung |
|-------|----------|---------------|
| **LLM** | Sprachmodell-Kern | Reasoning, Klassifikation, Planung |
| **DB** | Wissensdatenbank | Persistenz, Cross-Session-Lernen |
| **Rules** | Regel-Engine | Business-Regeln, Validierung |
| **MCP DOKU** | Tool-Registry | Dynamische Tool-Entdeckung |
| **API** | Kommunikationsschicht | n8n HTTP, MCP-Protokoll |
| **Core** | Orchestrierung | n8n Workflow-Steuerung |
| **C# Tool** | fe.screen-sim | CAD-Transformation (extern) |

---

## 2. Sprint 3 -> Erath Mapping

### Layer-Mapping

| Erath Component | Sprint 3 Implementation | Status |
|----------------|------------------------|--------|
| **LLM** | Sprachmodell-Kern via HTTP Request; fuer US9 isoliert mit Claude Sonnet / GPT-5 / Gemini 3 ausgefuehrt | ADAPTIERT |
| **DB (Wissen)** | n8n-Code-Node: In-Memory-JSON | TEILWEISE |
| **DB (Persistenz)** | Dateisystem (.md, .json, .meta) | TEILWEISE |
| **Rules (Klassifikation)** | System-Prompt: Komponenten-Taxonomie | EINGEBETTET |
| **Rules (Logik)** | System-Prompt: Transform-Regeln | EINGEBETTET |
| **Rules (FIFO)** | System-Prompt: Lagerstrategie | NEU |
| **MCP DOKU** | MCP_TOOLS_SPEC.md (statisch) | TEILWEISE |
| **MCP DOKU (live)** | fe.screen-sim MCP Endpunkt | BLOCKED |
| **API (n8n)** | n8n HTTP Request Nodes | DONE |
| **API (MCP)** | fe.screen-sim Bridge | BLOCKED |
| **Core** | n8n Orchestrierung (Merge, Aggregate, Code) | DONE |
| **C# Tool** | fe.screen-sim V5 | EXTERN (Grp B) |

GPT-5 und Gemini 3 sind keine zusaetzlichen LLM-Komponenten, sondern alternative
Modelle fuer die drei isolierten US9-Laeufe.

### Detaillierte Coverage

#### Phase 1 (Analyse) — Erath Coverage

| Node | Erath Layer | Covered | Bemerkung |
|------|------------|---------|-----------|
| Manual Trigger | Core | Voll | n8n Trigger |
| CAD vorbereiten | Core | Voll | Code-Node |
| Agent A (Klassifikation) | LLM + Rules | Voll | Prompt-basierte Rules |
| Agent B (Importformat) | LLM | Voll | Keine Rules noetig |
| Agent C (Simulationslogik) | LLM + Rules | Voll | FIFO-Regeln in Prompt |
| Tag A/B/C | API | Voll | Response-Parsing |
| Merge | Core | Voll | n8n Merge |
| Aufbereiten | DB (fluechtig) | Teilweise | In-Memory, keine Persistenz |
| Bericht-Generator | LLM | Voll | |
| persist | DB (Persistenz) | Voll | Datei-System |

#### Phase 2 (Transform) — Erath Coverage

| Node | Erath Layer | Covered | Bemerkung |
|------|------------|---------|-----------|
| Manual Trigger | Core | Voll | |
| CAD vorbereiten | Core | Voll | |
| Agent 1: Plan | LLM + Rules | Voll | Plan-Prompt mit Rules |
| Agent 2a: Renamer | LLM | Voll | |
| Agent 2b: Typer | LLM | Voll | |
| Agent 2c: Joint Config | LLM | Voll | |
| Tag Plan / Tag Parallel | API | Voll | |
| Merge Transform | Core | Voll | |
| Aggregate | Core | Voll | |
| Agent 5: Logic Builder | LLM + Rules | Voll | FIFO + Sequence in Prompt |
| Sammle alle Calls | Core | Voll | |
| MCP Executor | API <-> C# Tool | PLATZHALTER | Wartet auf Gruppe B |
| Agent 6: Validator | LLM + Rules | Voll | |
| persist (3 Dateien) | DB (Persistenz) | Voll | .md + .calls.json + .meta.json |

---

## 3. Multi-Agent-Rollen

> Pro US9-Lauf laeuft der gesamte Workflow mit genau einem Modell. Die drei Modelle
> (Claude Sonnet, GPT-5, Gemini 3) werden in drei isolierten Laeufen verglichen,
> nicht parallel pro Rolle.

| Rolle | Erath Layer | Prompt-Komplexitaet |
|-------|------------|-------------------|
| Plan Agent | LLM + Rules | Hoch (Erstellt Plan) |
| Renamer | LLM | Niedrig (Bulk-Rename) |
| Typer | LLM | Niedrig (Type-Set) |
| Joint Config | LLM | Mittel (Joint-Params) |
| Logic Builder | LLM + Rules | Hoch (Buttons + Logik) |
| Validator | LLM + Rules | Mittel (Validierung) |
| Klassifikation (Phase 1) | LLM + Rules | Mittel (Taxonomie) |
| Importformat (Phase 1) | LLM | Niedrig (Gantry-Struktur) |
| Simulationslogik (Phase 1) | LLM + Rules | Hoch (FIFO + Sequenz) |

---

## 4. Kritische Gaps (aus Research bestaetigt)

| Gap | Erath Layer | Schwere | Workaround | Timeline |
|-----|------------|---------|-----------|----------|
| **MCP Executor fehlt** | API <-> C# Tool | HOCH | Platzhalter-Node | Wartet auf Gruppe B |
| **Live MCP DOKU** | MCP DOKU | MITTEL | Statische MCP_TOOLS_SPEC.md | Wartet auf Gruppe B |
| **Keine persistente DB** | DB | NIEDRIG | Dateisystem-Persistenz | Sprint 4+ |
| **Rules in Prompts** | Rules | NIEDRIG | Kein separater Workaround | Sprint 4+ |
| **Modellvergleich (US9)** | LLM | OFFEN | drei isolierte Laeufe, ein Modell pro Lauf | Sprint 3 |

### Gap: MCP Executor

**Problem:** MCP Executor Node ist ein Platzhalter. Die tatsaechliche fe.screen-sim MCP Bridge existiert noch nicht.

**Auswirkung:**
- Phase 2 kann keine CAD-Transformation ausfuehren
- Validator kann nur auf Plan-Ebene pruefen (keine echte Simulation)
- Kein End-to-End-Test moeglich

**Workaround:**
- MCP Executor gibt Mock-Erfolg zurueck
- Validator prueft nur, ob Calls generiert wurden (nicht ob sie ausgefuehrt wurden)
- Manuelle Ueberpruefung der .calls.json durch Zollern

### Gap: Live MCP DOKU

**Problem:** Aktuell statische Markdown-Datei statt dynamischer Tool-Registry.

**Auswirkung:**
- LLM kann Tools nicht dynamisch entdecken
- Tool-Spezifikationen muessen manuell in Prompts gepflegt werden
- Aenderungen an fe.screen-sim erfordern Prompt-Updates

**Workaround:**
- MCP_TOOLS_SPEC.md als zentrale Referenz
- Manuelle Synchronisation bei Tool-Aenderungen

---

## 5. Coverage Matrix (Vollstandigkeit)

| Anforderung | Phase 1 | Phase 2 | US 9 | Erath | Status |
|------------|---------|---------|------|-------|--------|
| Hochregallager-Klassifikation | x | | | x | DESIGN |
| Kran-Kinematik (3D) | | x | | x | DESIGN |
| FIFO-Logik | | x | | x | DESIGN |
| Multi-Model-Vergleich (isolierte Laeufe) | | | x | x | DESIGN |
| MCP-Integration | | Platzhalter | | x | BLOCKED |
| Agentic AI-Dokumentation | | | | x | DONE |
| Persistente DB | | | | Gap | OPEN |
| Regel-Engine (extern) | | | | Gap | OPEN |
| Live-MCP-DOKU | | | | Gap | OPEN |
| US9-Lauf GPT-5 | | | x | | DESIGN |
| US9-Lauf Gemini 3 | | | x | | DESIGN |
| RBG/3-Achs-Kinematik | | x | | x | DESIGN |
| Standardisierte Prompts | x | x | x | | DONE |
| Vergleichs-Schema | | | x | | DONE |

---

## 6. Naechste Schritte

1. **MCP Executor bereitstellen** (Gruppe B (MCP)) — ermoeglicht End-to-End
2. **US 9 Testlauf** — drei isolierte Laeufe, je ein Modell (Claude Sonnet, GPT-5, Gemini 3), gleiche CAD-Eingabe
3. **Auswertung** — gewichtete Bewertungsformel auf die drei Berichte anwenden
4. **DB-Konzept** — persistente Agenten-Wissensdatenbank (Sprint 4)
5. **Rule-Engine** — externe, versionierbare Regeln (Sprint 4)
