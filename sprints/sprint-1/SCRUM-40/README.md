# SCRUM-40: n8n-Workflow CAD → LLM → Simulationsbericht

**Ticket:** SCRUM-40 | **Bearbeitung:** Rustam Kohen | **Stand:** 2026-04-28 (v2)
**Projekt:** Zollern – Machbarkeitsstudie virtuelle Inbetriebnahme + KI

## Zweck

Automatisierter Workflow, der eine CAD-Assemblierung durch drei parallele LLM-Agenten analysiert und einen strukturierten Simulationsbericht für fe.screen-sim V5 erzeugt. Ziel: zeigen, dass LLMs CAD-Daten interpretieren und Simulationsstrukturen vorschlagen können.

## Architektur

```
                     ┌──► Agent A: Klassifikation ──► Tag A ──┐
                     │                                        │
CAD-Daten ───────────┼──► Agent B: Importformat   ──► Tag B ──┼──► Merge (append) ──► Aufbereiten (find by Tag) ──► Bericht-Generator ──► Bericht ausgeben + speichern
                     │                                        │
                     └──► Agent C: Simulationslogik ► Tag C ──┘
```

## Änderungen v1 → v2 (P0+P1 Patches, 2026-04-28)

| Bereich | v1 | v2 |
|---------|----|-----|
| Agenten-Mapping | mergeByPosition + `data[0..2]` (fragil) | Tag-Wrapper pro Agent + `find(rolle === ...)` |
| Fehlerbehandlung | keine | `retryOnFail: 3 Versuche` auf allen HTTP-Nodes, try/catch in Tag-Nodes |
| JSON-Output | "Nur JSON zurückgeben" als Prompt (LLM ignoriert ~10%) | Tool-Use mit JSON-Schema erzwingt validen Output |
| System-Prompt | alles in user | system + user getrennt |
| max_tokens | 1024/2048 | 2048/4096 |
| B vs C Überlappung | beide arbeiten an SPS/Format | B = nur Container-Format, C = nur Anlagenverhalten + SPS |
| SPS-Adressen | LLM erfindet S7-Adressen | explizit `TBD_SPS_Quelle_erforderlich` |
| Persistierung | `console.log` → weg | Markdown-Bericht + Sidecar-Meta-JSON in `runs/{timestamp}.{md,meta.json}` |
| Token-Tracking | nein | tokens_in/tokens_out pro Agent + Summary |

Backup v1: `zollern_n8n_workflow.v1.json`

**Muster:** Parallelization Workflow (Saile, LLM & Agentics – Session 5).
Drei parallele Spezial-Agenten, dann ein Summary-Agent fasst zusammen.
Vorteil gegenüber Single-Agent: höhere Genauigkeit, klare Verantwortlichkeiten, parallel = schneller.

## Agenten-Rollen

| Agent | Aufgabe | Input | Output |
|-------|---------|-------|--------|
| A | Komponenten klassifizieren (Fördertechnik, Antrieb, Sensor, ...) | Assembly-JSON | JSON-Array mit Klassifikationen |
| B | Importformate für fe.screen-sim bewerten | Assembly-JSON | JSON mit Machbarkeit pro Format |
| C | Simulationslogik ableiten (Bewegungen, Sensoren, SPS) | Assembly-JSON | Strukturiertes JSON mit Logik |
| Summary | Bericht generieren (Markdown, 6 Abschnitte) | Output A+B+C | Vollständiger Machbarkeitsbericht |

## Modell und Kosten

- **Modell:** `claude-sonnet-4-6` (Anthropic API)
- **Token-Limit pro Agent:** 2048 (Summary: 4096)
- **Kosten pro Durchlauf (geschätzt):** unter 0,10 USD bei aktueller Beispielanlage
- **Strukturierter Output:** Tool-Use (Anthropic structured outputs) für Agent A/B/C, freier Markdown für Summary

## Persistierung

Jeder Lauf schreibt automatisch:
- `runs/YYYY-MM-DD_HH-MM-SS.md` – finaler Markdown-Bericht
- `runs/YYYY-MM-DD_HH-MM-SS.meta.json` – Metadaten (Modell, Tokens, Anlage, Pfad)

## Quick Start

1. n8n öffnen → Import from JSON → **`zollern_n8n_workflow_v2.json`** (das ist der aktuelle, importieren!)
2. Credential `Anthropic API Key` anlegen (siehe `SCRUM-40_setup_anleitung.md`)
3. Execute Workflow → Bericht erscheint im Output-Node und wird in `runs/` persistiert

Detaillierte Setup-Schritte: `SCRUM-40_setup_anleitung.md`
Vollständige technische Erklärung der v2-Änderungen: `SCRUM-40_erklaerung_v2.txt`
Team-Kommunikation (Entwürfe): `SCRUM-40_teams_nachrichten.txt`

## Aktuelle Platzhalter (für späteren Ersatz)

| Platzhalter | Was ersetzt werden muss | Verantwortlich |
|-------------|------------------------|----------------|
| Fiktive Förderer/Drehtisch-Daten in `CAD-Daten vorbereiten` | Echter STEP/AML-Parser-Output | Grp A (SCRUM-29) |
| Manueller Trigger | Webhook-Trigger für SCRUM-29-Anbindung | Rustam (Folge-Ticket P2) |
| Lokaler Datei-Output `runs/` | Webhook → Teams oder Sharepoint-Sync | Rustam (Folge-Ticket P2) |
| Hardcoded Anlagenname `Foerderer_Drehtisch_v1` | aus Input-Payload lesen | Rustam (mit Webhook-Trigger zusammen) |

## Verbindung zu anderen Tickets

- **SCRUM-10** (Gruppe C): Prototypischer Workflow mit fiktiven Daten – SCRUM-40 ist die n8n-Umsetzung dieses Tickets
- **SCRUM-29** (Gruppe A): CAD-Dateninterpretation durch LLMs – liefert das Eingabeformat
- **SCRUM-30** (Rustam, fertig): Datenformate-Analyse – Grundlage für Agent B
- **SCRUM-31** (Rustam, fertig): Importformat-Bewertung – Logik in Agent B integriert
- **SCRUM-32** (Gruppe C, fertig): Limitationen – fließen in Bericht-Abschnitt 5 ein
- **SCRUM-41**: Förderer/Drehtisch als Beispielanlage – fiktive Daten basieren darauf

## Status

- [x] Workflow-Architektur definiert
- [x] Saile-Muster adaptiert (Parallelization + Summary)
- [x] Drei Agenten + Summary-Agent implementiert
- [x] Fiktive Beispielanlage (Förderer/Drehtisch) integriert
- [x] JSON-Schema valide, importierbar in n8n
- [x] **v2 P0:** Tag-basiertes Mapping (robust gegen Reihenfolge)
- [x] **v2 P0:** Retry on fail (3 Versuche, 2s Wartezeit)
- [x] **v2 P0:** Markdown + Meta-Sidecar persistiert in `runs/`
- [x] **v2 P1:** Tool-Use erzwingt valide JSON-Outputs
- [x] **v2 P1:** System-Prompts getrennt
- [x] **v2 P1:** B/C-Aufgaben orthogonalisiert
- [x] **v2 P1:** SPS-Adressen-Halluzination geblockt
- [ ] Anthropic Credential in n8n eingetragen
- [ ] Erstdurchlauf mit fiktiven Daten getestet
- [ ] Echtes CAD-Format von Zollern geklärt (Product Owner (Kundenseite))
- [ ] CAD-Parser-Anbindung (Grp A SCRUM-29)
- [ ] Webhook-Trigger statt Manual (P2)
- [ ] Output-Anbindung an Teams/Sharepoint (P2)
- [ ] Prompt Caching (P2, Master-Thesis-Vorbereitung)
- [ ] MCP-Architektur skizzieren für Bericht (P2, Master-Thesis-Roadmap)

## Bewertung der Machbarkeit (auf Basis SCRUM-30/31/32)

- LLM-Auswertung **vor** dem fe.screen-sim-Import: machbar (offene Formate STEP/DAE/AML)
- LLM-Auswertung der fe.screen-sim-internen Strukturen: nicht möglich (F.EE-Format abgeschottet)
- Funktionssemantik/Steuerlogik: nicht in CAD enthalten – muss aus PLCTAGS/SPS-Quellen ergänzt werden

## Phase 1 vs Phase 2 (2026-04-28, nach Demo-Video-Analyse)

**Phase 1** (`zollern_n8n_workflow_v2.json`): Analyse-Workflow. 3 parallele Agenten erzeugen Machbarkeits-Bericht über CAD-Daten. Liefert Erkenntnisse, transformiert fe.screen-sim NICHT.

**Phase 2** (`zollern_n8n_workflow_phase2.json`): Multi-Agent-Pipeline die fe.screen-sim direkt transformiert via MCP-Tools. Architektur:

```
Plan Agent (analysiert CAD + erstellt Plan)
        ↓
   ┌────┼────┐
   ▼    ▼    ▼
Renamer Typer Joint   ← parallel (Saile-Pattern)
   └────┼────┘
        ▼
   Logic Builder (sequential, braucht Surfaces typed)
        ▼
   MCP Executor (Platzhalter, später HTTP an fe.screen-sim MCP)
        ▼
   Validator → Bericht + persist
```

**Erkenntnisse aus Demo-Video** (YouTube U2PuFm768SQ, F.EE):
- CAD-Tags: `Inventor User Defined Properties/{Objektname,Simulationstyp,JointType}`
- Pro Surface: 2 Buttons (Forward/Reverse, ±1.0 m/s) + 1 LogicObject (OR-Logik) + 6 Verknüpfungen
- Sensor → DetectPayload=true; Motorisiert → JointScale=[0.5,0.5,0.5]
- MCP-Tools: documentation_execute, content_help, object_help, import_cad_file, get_all_objects, rename_object, set_object_type, configure_motionjoint, create_button, create_logic_object, link_objects

**Phase 2 ist noch nicht voll lauffähig** -- MCP-Endpunkt zu fe.screen-sim fehlt (Aufgabe Gruppe B (MCP), Grp B). Workflow läuft mit simuliertem Executor (loggt Calls, schreibt Bericht).

## Repo-Struktur

```
SCRUM-40/
├── README.md                              ← diese Datei
├── README.txt                             ← gleiche Inhalte, plain text
├── SCRUM-40_setup_anleitung.md            ← Setup für Bediener
├── SCRUM-40_erklaerung_v2.txt             ← v2-Änderungen erklärt
├── SCRUM-40_teams_nachrichten.txt         ← Entwürfe für 4 Teams-Nachrichten
├── zollern_n8n_workflow_v1.json           ← Backup (nur Analyse, keine Persist)
├── zollern_n8n_workflow_v2.json           ← Phase 1 (Analyse-Workflow, 3 parallele Agenten)
├── zollern_n8n_workflow_phase2.json       ← Phase 2 (Multi-Agent + MCP, Demo-Video-konform)
├── runs/                                  ← Phase-1-Läufe (.md + .meta.json)
└── runs_phase2/                           ← Phase-2-Läufe (.md + .calls.json + .meta.json)
```

## Scope

Academic feasibility study. Contains no customer data: all runs use a synthetic test plant.
