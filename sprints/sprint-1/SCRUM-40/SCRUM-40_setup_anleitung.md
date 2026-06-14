# SCRUM-40: n8n Workflow – Setup-Anleitung

**Stand:** 2026-04-28 (v2) | **Bearbeitung:** Rustam Kohen

## Was dieser Workflow macht

```
CAD-Daten (fiktiv/real)
        |
        v
[ Agent A ] Klassifikation     → Tag A ─┐
[ Agent B ] Importformat       → Tag B ─┼─► Merge (append) ─► Aufbereiten ─► Bericht-Generator ─► Bericht ausgeben + speichern
[ Agent C ] Simulationslogik   → Tag C ─┘
```

Drei Claude-Agenten laufen parallel, jeder mit eigenem Tool-Schema (Tool-Use erzwingt valides JSON). Tag-Wrapper markieren jeden Output mit `rolle`. Aufbereitung mappt per Tag (robust gegen Reihenfolge). Summary-Agent erzeugt Markdown-Bericht. Letzter Code-Node persistiert .md + .meta.json in `runs/`.

Muster: Saile (LLM & Agentics) – Parallelization + Summary-Agent.

## Credentials einrichten (einmalig)

In n8n unter **Settings → Credentials → New**:

| Typ | Name | Feld | Wert |
|-----|------|------|------|
| Header Auth | `Anthropic API Key` | Header Name | `x-api-key` |
| | | Header Value | dein Anthropic API Key |

## Workflow importieren

1. n8n öffnen
2. **New Workflow → Import from JSON**
3. Datei `zollern_n8n_workflow_v2.json` hochladen ⭐ (NICHT die v1-Datei, das ist nur Backup)
4. Credential `Anthropic API Key` in allen 4 HTTP-Request-Nodes verknüpfen
5. **Execute Workflow** (Manual Trigger)

## Was jetzt noch Platzhalter ist

| Node | Was ersetzen | Womit |
|------|-------------|-------|
| `CAD-Daten vorbereiten` | Fiktive JSON-Assemblierung | Echter STEP/AML-Parser Output (pythonOCC oder fe.screen-sim CAD-Konverter) |
| Alle Agent-Nodes | `claude-sonnet-4-6` | Bei Bedarf anderes Modell |
| `Start (Manuell)` | Manueller Trigger | Webhook-Trigger für Auto-Start aus SCRUM-29-Pipeline |
| `Bericht ausgeben + speichern` | Lokale Datei `runs/...md` | Zusätzlich: Webhook → Teams-Channel oder Sharepoint-Sync |

## Welche Credentials du brauchst

- **Anthropic API Key** → von platform.anthropic.com
- Später für echte Anbindung: **fe.screen-sim MCP Credentials** (Gruppe B klärt das)

## Was der Bericht enthält

Claude generiert automatisch:
1. Executive Summary (3 Sätze)
2. Komponentenübersicht mit Klassifikation (Tabelle)
3. Empfehlung welche fe.screen-sim Importstrukturen erstellt werden sollen
4. Simulationsstruktur-Vorschlag (Bewegungsabläufe, Sensoren, SPS-Variablen mit Adress-Platzhaltern)
5. Limitationen und offene Punkte
6. Nächste Schritte

## Persistierung

Pro Lauf werden zwei Dateien geschrieben:
- `runs/2026-04-28_15-30-45.md` – finaler Bericht
- `runs/2026-04-28_15-30-45.meta.json` – Metadaten (Modell, Token-Verbrauch, Anlage)

`runs/` wird beim ersten Lauf automatisch angelegt.

## Fehlertoleranz

- Alle 4 HTTP-Nodes: `retryOnFail: 3 Versuche, 2s Wartezeit`
- Tag-Code-Nodes: try/catch, Fehler werden als `_error` ins Payload geschrieben statt Pipeline zu killen
- Bei Tool-Use-Fail (sehr selten): Fallback auf Roh-Text in `_raw_text`

## Verbindung zu anderen Tickets

- **SCRUM-30** (Rustam): Eingabeformate definiert → CAD-Daten-Node hier direkt verwertbar
- **SCRUM-31** (Rustam): Importformat-Bewertung → Agent B nutzt dieselbe Logik
- **SCRUM-32** (Rustam/Gruppe C/Karim): Limitationen → fließen in Bericht-Abschnitt 5 ein
- **SCRUM-10** (Gruppe C): Prototypischer Workflow → dieser Workflow ist die n8n-Umsetzung
