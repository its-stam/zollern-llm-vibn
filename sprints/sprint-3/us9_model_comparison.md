# US 9: Multi-Model Comparison

**Datum:** 2026-05-11 | **Agent:** coder (Pipeline Step 3/3)
**Status:** Implemented

---

## 1. Ziel

Vergleich von drei LLMs (Claude Sonnet 4, GPT-5, Gemini 3) bei der CAD-Klassifikation und Workflow-Transformation fuer das Hochregallager-Szenario. Gleicher Input, gleiche Tasks, unterschiedliche Modelle.

## 2. Modelle

| Modell | API Endpoint | Modell-ID | Kosten Input | Kosten Output |
|--------|-------------|-----------|-------------|--------------|
| Claude Sonnet 4 | api.anthropic.com/v1/messages | claude-sonnet-4-20250506 | $3.00/Mtok | $15.00/Mtok |
| GPT-5 | api.openai.com/v1/chat/completions | gpt-5-turbo | $2.50/Mtok | $10.00/Mtok |
| Gemini 3 | generativelanguage.googleapis.com | gemini-3-pro | $1.50/Mtok | $7.50/Mtok |

## 3. Test-Setup

### Phase 1 (Analyse)
- **CAD:** Hochregallager mit RBG + 13 Komponenten
- **Task:** Klassifikation, Importformat, Simulationslogik
- **Output:** Strukturierter Analyse-Bericht

### Phase 2 (Transform)
- **CAD:** Hochregallager_v1 mit 9 Objekten
- **Task:** Plan, Rename, Type, Joint Config, Logic
- **Output:** fe.screen-sim Tool-Calls + Validierung

## 4. Pipeline-Struktur

```
                +--> [Claude Agent A] --> [Tag Claude A] --+
                |--> [GPT Agent A]    --> [Tag GPT A]    --|
                |--> [Gemini Agent A] --> [Tag Gemini A] --|
                |                                          |
[Trigger] --> [CAD Prep] --+--> [Claude Agent B] --> [Tag Claude B] --+--> [Merge All] --> [Aufbereitung] --> [Bericht]
                           |--> [GPT Agent B]    --> [Tag GPT B]    --|
                           |--> [Gemini Agent B] --> [Tag Gemini B] --|
                           |                                          |
                           +--> [Claude Agent C] --> [Tag Claude C] --+
                            --> [GPT Agent C]    --> [Tag GPT C]    --
                            --> [Gemini Agent C] --> [Tag Gemini C] --
```

## 5. Gewichtete Bewertungsformel

```
Score = 0.30 * strukturelle_korrektheit
      + 0.20 * (1 - halluzinationsrate)
      + 0.10 * token_effizienz
      + 0.10 * (1 - latency/max_latency)
      + 0.10 * (1 - cost/max_cost)
      + 0.20 * qualitaet
```

### Metrik-Definitionen

1. **Strukturelle Korrektheit** = korrekt_klassifiziert / gesamt_komponenten
   - Alle 13 Komponenten korrekt getypt?
   - Keine Fehler in der Taxonomie?

2. **Halluzinationsrate** = erfundene_sps_adressen / gesamt_sps_adressen
   - Werden SPS-Adressen erfunden statt TBD-Marker gesetzt?

3. **Token-Effizienz** = output_tokens_nutzbar / output_tokens_gesamt
   - Wie viel Output ist tatsaechlich verwertbar?

4. **Latenz** = Zeit von Request bis Response
   - Modellvergleich bei gleichem Input

5. **Kosten pro Lauf** = Tokens * modellspezifischer Preis
   - Wirtschaftlichkeitsvergleich

6. **Qualitaet** = Menschliche Bewertung (Zollern-Feedback)
   - Welcher Bericht ist brauchbarer?

## 6. Output-Struktur

Pro Lauf werden 5 Dateien geschrieben:

| Datei | Inhalt |
|-------|--------|
| `us9_report_claude_{timestamp}.md` | Claude-Bericht |
| `us9_report_gpt_{timestamp}.md` | GPT-Bericht |
| `us9_report_gemini_{timestamp}.md` | Gemini-Bericht |
| `us9_comparison_{timestamp}.md` | Vergleichsbericht mit Scores |
| `us9_comparison_{timestamp}.json` | Maschinenlesbare Ergebnisse |

## 7. Vergleichsbericht (Markdown)

Enthaelt:
- Metrik-Tabelle (alle 6 Metriken pro Modell)
- Score-Tabelle (gewichtete Formel)
- Gewinner pro Metrik (fett markiert)
- Ranking (1./2./3.)
- Empfehlung

## 8. Implementierte Nodes

### Phase 1 Agenten (4 Rollen x 3 Modelle = 12 HTTP Nodes)

| Rolle | Claude | GPT | Gemini |
|-------|--------|-----|--------|
| Agent A (Klassifikation) | claude_agent_a | gpt_agent_a | gemini_agent_a |
| Agent B (Importformat) | claude_agent_b | gpt_agent_b | gemini_agent_b |
| Agent C (Simulationslogik) | claude_agent_c | gpt_agent_c | gemini_agent_c |
| Bericht | claude_bericht | gpt_bericht | gemini_bericht |

### Tag-Wrapper (3 Modelle x 3 Rollen + 1 Bericht = 12 Tag Nodes)

Jeder Tag-Wrapper normalisiert die modellspezifische Response in ein einheitliches Format:

```typescript
interface TaggedResponse {
  modell: "claude" | "gpt" | "gemini";
  rolle: "agent_a" | "agent_b" | "agent_c" | "bericht";
  payload: object;  // normalisierte tool_use/tool_calls/functionCall
  tokens_in: number;
  tokens_out: number;
  tokens_total: number;
}
```

### Merge & Vergleich (3 Merge Nodes)

| Node | Inputs | Funktion |
|------|--------|----------|
| merge_alle_modelle | 9 (3 Modelle x 3 Rollen) | Vereinigt alle Tag-Outputs |
| ergebnisse_aufbereiten | 1 (von merge) | Baut token_vergleich + ergebnisse |
| merge_berichte | 3 (Claude, GPT, Gemini) | Vereinigt alle Berichte |
