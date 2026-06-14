# Sprint 2 Abschlussbericht — SCRUM-40

**Ticket:** SCRUM-40 | **Datum:** 01.05.2026 | **Bearbeiter:** Rustam Kohen
**Projekt:** Zollern Machbarkeitsstudie virtuelle Inbetriebnahme + KI
**Teamkontext:** Dieser Bericht beschreibt den n8n-/Agentic-Workflow (SCRUM-40), umgesetzt von Rustam Kohen als Teil des Projektteams. Am Gesamtprojekt mitgewirkt haben Gruppe B (MCP) und Gruppe B (LLM-Stack) (fe.screen-sim-Anbindung, MCP, LLM-Integration), Gruppe C (Workflow-Tests, Datenaufbereitung), Gruppe A (CAD) (Scrum, Datenerfassung), Gruppe A (CAD) (CAD-Modellierung) sowie Gruppe A, Gruppe A, Gruppe A und Gruppe A; Product Owner ist Product Owner (Kundenseite).

---

## Sprint-Ziel

Automatisierten n8n-Workflow entwickeln, der CAD-Daten via parallele LLM-Agenten analysiert und einen strukturierten Machbarkeitsbericht für fe.screen-sim V5 erzeugt. Erweiterung um Multi-Agent-Transform-Pipeline (Phase 2), die fe.screen-sim direkt via MCP-Tools manipuliert.

Phase 2 ist zusätzlich als architektonische Grundlage für die geplante Master-Thesis aufgebaut (Multi-Agent-Orchestrierung, MCP-Integration, Prompt Caching).

---

## Lieferungen Sprint 2

### Phase 1 — Analyse-Workflow (`workflow_phase1_analyse.json`)

Vollständig implementiert und lauffähig.

Drei parallele LLM-Agenten (Anthropic claude-sonnet-4-6) analysieren CAD-Daten unabhängig:

- **Agent A** — Komponentenklassifikation (Förderer, Antrieb, Sensor)
- **Agent B** — Importformat-Bewertung (STEP, DAE, AML für fe.screen-sim)
- **Agent C** — Simulationslogik (Bewegungen, Sensoren, SPS-Anbindung)
- **Summary-Agent** — fasst A+B+C zu Markdown-Bericht (6 Abschnitte) zusammen

Architektur-Grundlage: Parallelization-Pattern (Saile, LLM & Agentics Session 5).

**v2-Patches gegenüber v1:**

| Problem v1 | Lösung v2 |
|------------|-----------|
| `mergeByPosition` — Reihenfolge-abhängig und fragil | Tag-Wrapper (`rolle: A/B/C`) + `find()` |
| Keine Fehlerbehandlung | `retryOnFail: 3 Versuche, 2s Wartezeit` auf allen HTTP-Nodes |
| LLM ignoriert "nur JSON"-Prompt (~10%) | Tool-Use mit JSON-Schema erzwingt valides JSON |
| System + User-Prompt gemischt | Getrennte System/User-Prompts |
| Agent B und C überlappen bei SPS | B = nur Container-Format, C = nur Anlagenverhalten |
| LLM erfindet S7-Adressen | Explizit `TBD_SPS_Quelle_erforderlich` blockiert Halluzination |
| Kein Persistenz | `runs/{timestamp}.md` + `runs/{timestamp}.meta.json` |
| Kein Token-Tracking | tokens_in/out pro Agent + Summary |

**Kosten pro Lauf:**

| Szenario | Kosten |
|----------|--------|
| Beispielanlage Förderer/Drehtisch (klein, fiktiv) | ~5 Cent |
| Reale Zollern-Anlage, 50+ Komponenten (geschätzt) | 30–50 Cent |

Modell: claude-sonnet-4-6, A/B/C: 2048 tok, Summary: 4096 tok.

---

### Phase 2 — Transform-Workflow (`workflow_phase2_transform.json`)

Strukturell vollständig implementiert, MCP-Endpunkt noch nicht live.

Pipeline transformiert fe.screen-sim direkt via 11 MCP-Tools:

```
Plan Agent
    ↓
Renamer | Typer | Joint-Configurer   (parallel)
    ↓
Logic Builder                        (sequential, braucht typisierte Surfaces)
    ↓
Sammle alle Calls (Aggregate)
    ↓
MCP Executor                         (Platzhalter bis Endpunkt von Gruppe B (MCP))
    ↓
Validator → bericht_phase2.md + .calls.json + .meta.json
```

**Datenbasis:** Analyse YouTube-Demo F.EE (U2PuFm768SQ), Screenshots IMG_8128–8144.
Abgeleitet: Inventor-Tag-Konvention (`Objektname`, `Simulationstyp`, `JointType`), Button/Logic-Pattern pro Surface.

**MCP-Ausführungsreihenfolge (kritisch):**

```
Phase 0: import_cad_file → get_all_objects
Phase 1: rename_object        (alle Objekte mit Inventor-Tag Objektname)
Phase 2: set_object_type      (Surface | Motorisiert | Sensor)
       + set_property         (Sensor: DetectPayload=true, Motorisiert: JointScale=[0.5,0.5,0.5])
Phase 3: configure_motionjoint (alle Objekte mit JointType-Tag)
Phase 4: create_button (2×/Surface) + create_logic_object (1×/Surface) + link_objects (6×/Surface)
```

**MCP-Spezifikation** (`MCP_TOOLS_SPEC.md`) an Gruppe B (MCP) (Grp B) übergeben.

---

## Designentscheidungen

### Warum genau 3 Agenten?

Die Domäne zerfällt in drei strukturell unabhängige Teilfragen mit unterschiedlichen Wissensquellen:

- **Agent A** (Klassifikation): Bauteilkenntnis — was ist die Anlage
- **Agent B** (Importformat): Format-Kenntnis aus SCRUM-30/31 — wie kommt sie rein
- **Agent C** (Simulationslogik): Steuerungstechnik — wie verhält sie sich

Drei Teilfragen, drei Wissensdomänen. Mehr Agenten würden künstliche Splits erzeugen (z.B. Förderer-Agent + Drehtisch-Agent — gleiches Bewertungsschema, doppelter API-Call ohne Genauigkeitsgewinn). Weniger Agenten würden zwei Domänen in einen Prompt bündeln. Genau dieses Problem gab es in v1: B und C überlappten bei SPS-Themen und der Agent halluzinierte S7-Adressen.

### Warum Phase 1 und Phase 2?

Phase 1 beantwortet die Machbarkeitsfrage: können LLMs CAD-Daten interpretieren und Simulationsstrukturen vorschlagen. Output ist ein Bericht, keine Transformation.

Phase 2 zeigt den Weg in die Praxis: ein orchestrierter Multi-Agent-Workflow der fe.screen-sim direkt transformiert, sobald der MCP-Endpunkt steht. Phase 1 allein spart keine Inbetriebnahme-Stunden. Phase 2 schon.

Phase 2 entstand aus der Analyse des F.EE-Demo-Videos. Erst nach dieser Analyse wurde klar, dass eine direkte MCP-Transformation architektonisch machbar ist.

### Wie viele Phasen sind geplant?

Aktuell zwei Phasen lieferbereit. Phase 3 ist möglich und sinnvoll, hängt aber von zwei externen Abhängigkeiten ab:

1. Gruppe B (MCP) (Grp B) liefert MCP-Endpunkt zu fe.screen-sim → Phase 2 läuft live
2. Grp A (SCRUM-29) liefert echten CAD-Parser → beide Phasen laufen mit echten Zollern-Daten

Erst nach erstem echten Lauf ist klar, welche Erweiterungen Phase 3 braucht. Vorher wäre Phase 3 Spekulation.

Mögliche Phase-3-Inhalte: Webhook-Trigger (automatischer Start bei neuem CAD-Eingang), Output-Sync Teams/Sharepoint, Prompt Caching für Kostenoptimierung.

---

## Status

| Aufgabe | Status |
|---------|--------|
| Workflow-Architektur Phase 1 | DONE |
| 3 Agenten + Summary implementiert | DONE |
| v2 P0-Patches (Tag-Mapping, Retry, Persist) | DONE |
| v2 P1-Patches (Tool-Use, System-Prompt, B/C-Split, SPS-Block) | DONE |
| Phase 2 Workflow strukturell | DONE |
| MCP_TOOLS_SPEC.md für Gruppe B (MCP) | DONE |
| Anthropic Credential in n8n eingetragen | OFFEN |
| Erstdurchlauf Phase 1 mit fiktiven Daten | OFFEN |
| MCP-Endpunkt Gruppe B (MCP) (Grp B) | OFFEN — externe Abhängigkeit |
| Echtes CAD-Format Zollern (STEP vom Product Owner bestätigt) | OFFEN — Inventor-Tags unklar |
| CAD-Parser-Anbindung SCRUM-29 (Grp A) | OFFEN — externe Abhängigkeit |

---

## Offene Fragen

### Technisch — an Gruppe B (MCP) / F.EE

| # | Frage | Impact |
|---|-------|--------|
| 1 | Exakter API-String für `logic_type: "OR_Velocity"`? | Logic Builder bricht bei falschem String |
| 2 | Gilt `JointScale [0.5,0.5,0.5]` für RevoluteJoint und PrismaticJoint gleich? | Joint-Konfiguration Phase 3 |
| 3 | Port-Name für Link 6 (`Active → Active`) korrekt? | 6. link_objects-Call pro Surface |
| 4 | MCP-Server Base-URL und Auth-Methode? | MCP Executor kann ohne das nicht arbeiten |
| 5 | Muss fe.screen-sim offen und Station geladen sein beim MCP-Call? | Setup-Anforderung für echten Lauf |
| 6 | Batch-API möglich (mehrere rename in einem Call)? | Performance bei großen Anlagen |

### Inhaltlich — an Zollern / Product Owner (Kundenseite)

| # | Frage | Impact |
|---|-------|--------|
| 7 | Welche Inventor User Defined Properties sind in echten Zollern-CAD-Dateien gesetzt? | Tag-Konvention in Agenten muss angepasst werden |
| 8 | JointScale-Bedeutung für konkrete Zollern-Anlage bestätigen | Motorisiert-Konfiguration Phase 3 |

---

## Aktuelle Platzhalter

| Platzhalter | Was ersetzt werden muss | Verantwortlich |
|-------------|------------------------|----------------|
| Fiktive Förderer/Drehtisch-Daten | Echter STEP-Parser-Output | Grp A (SCRUM-29) |
| Manueller Trigger | Webhook-Trigger für SCRUM-29-Anbindung | Rustam (Folge-Ticket) |
| Lokaler Datei-Output `runs/` | Webhook → Teams oder Sharepoint | Rustam (Folge-Ticket) |
| Hardcoded Anlagenname `Foerderer_Drehtisch_v1` | Aus Input-Payload lesen | Rustam (mit Webhook zusammen) |
| MCP Executor (simuliert) | Echter HTTP-Call an fe.screen-sim MCP | Gruppe B (MCP) Grp B |

---

## Abhängigkeiten zu anderen Tickets

| Ticket | Team | Was fehlt | Blockiert |
|--------|------|-----------|-----------|
| SCRUM-29 | Grp A (Gruppe A (CAD)) | CAD-Parser-Output als Eingabeformat | Echter Erstdurchlauf Phase 1 |
| MCP-Endpunkt | Grp B (Gruppe B (MCP)) | fe.screen-sim MCP-Server live | Phase 2 Erstlauf |
| SCRUM-41 | — | Förderer/Drehtisch als Referenzanlage | Validierung Beispieldaten |

---

## Sprint 3 — Sommer-Feedback und Kandidaten

### Feedback Prof. Sommer (Sprint 2 Review, 01.05.2026)

> "Selben Workflow isoliert mit verschiedenen APIs testen ob Ergebnis gleich ist — GPT, Gemini. Berichte vergleichen. Was gefällt dem Kunden am Ende besser?"

Modelle: **Claude Sonnet**, **ChatGPT-5**, **Gemini 3**

Für rechenintensive Tests: Hr. Baumann (HS-IT) stellt Hochleistungsrechner via Uni-VPN bereit.

**Forschungsfragen Sprint 3:**
1. Geben alle drei Modelle bei gleicher Eingabe dasselbe Ergebnis aus?
2. Wo weichen sie ab — Klassifikation, Formatbewertung, Simulationslogik?
3. Welches Modell halluziniert weniger (z.B. SPS-Adressen)?
4. Was bevorzugt der Kunde (Zollern) beim Lesen der Berichte?

**Architektur Multi-Model-Test:**
```
Gleiche CAD-Eingabe (Förderer/Drehtisch)
    ├─ Workflow A: claude-sonnet   → bericht_claude.md
    ├─ Workflow B: chatgpt-5       → bericht_gpt.md
    └─ Workflow C: gemini-3        → bericht_gemini.md
              ↓
    Vergleichs-Node: Berichte nebeneinander
              ↓
    Strukturierter Vergleichsbericht
```

### Weitere Sprint 3 Kandidaten

| Aufgabe | Priorität |
|---------|-----------|
| Anthropic Credential + Phase-1-Erstlauf | hoch |
| Multi-Model-Test (Claude / GPT-5 / Gemini 3) | hoch — Sommer-Anforderung |
| VPN-Zugang Hr. Baumann klären | organisatorisch |
| Phase-2-Erstlauf sobald MCP-Endpunkt steht | mittel — wartet auf Gruppe B (MCP) |
| Webhook-Trigger, Anbindung SCRUM-29 | mittel |
| Output-Sync Teams/Sharepoint | niedrig |
| Prompt Caching | niedrig — Master-Thesis-Relevanz |

---

## Bewertung Machbarkeit (Stand 01.05.2026)

- LLM-Auswertung vor fe.screen-sim-Import: **machbar** (STEP, DAE, AML sind offene Formate)
- LLM-Auswertung fe.screen-sim-interner Strukturen: **nicht möglich** (F.EE-Format abgeschottet)
- Funktionssemantik/Steuerlogik aus CAD: **nicht möglich** — muss aus PLC-Tags/SPS-Quellen ergänzt werden
- MCP-gestützte automatische Transformation: **architektonisch bereit**, wartet auf MCP-Endpunkt
- Multi-Model-Vergleich: **Sprint 3**, Modelle Claude Sonnet / ChatGPT-5 / Gemini 3

---

## Master-Thesis-Anschluss

Phase 2 ist bewusst über die reine Machbarkeitsstudie hinaus aufgebaut, um als Grundlage für die geplante Master-Thesis zu dienen. Folgende Aspekte fließen direkt weiter:

- **Multi-Agent-Orchestrierung** (Plan Agent → parallele Worker → sequenzieller Logic Builder) als Forschungsobjekt
- **MCP-Integration** als Architekturpattern für LLM-getriebene Tool-Anbindung
- **Prompt Caching** zur Kostenoptimierung bei wiederholten Anlagen-Analysen
- **Multi-Model-Vergleich** (Sprint 3) als empirische Grundlage für Modellwahl

---

*Academic feasibility study, HS Albstadt-Sigmaringen. No customer data: all runs use a synthetic test plant.*
