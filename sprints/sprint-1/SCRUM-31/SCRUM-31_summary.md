# SCRUM-31 Summary

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 2026-04-27

## Aufgabe (laut Jira)

Analyse, ob und wie ein LLM aus gegebenen Eingabedaten valide Simulationsstrukturen ableiten oder generieren kann. Ergebnisbewertung anhand definierter Kriterien.

## Vorgehen

Bewertung der durch fe.screen-sim akzeptierten externen Importstrukturen anhand vier Kriterien: Schema-Konformität (K1), Importierbarkeit (K2), Semantische Korrektheit (K3), Vollständigkeit (K4). Quellen: V4 User Documentation, AutomationML-Spezifikation, Vorergebnisse SCRUM-32.

## Untersuchte Strukturen

| # | Struktur | Format | Schema in V4-Doku |
|---|----------|--------|--------------------|
| 1 | Connectors | XML | unvollständig |
| 2 | Payloads | XML | unvollständig |
| 3 | Schaltschrank | Excel A–E | vollständig |
| 4 | SPS-Variablen "PLCTAGS" | Excel A–E | vollständig |
| 5 | TIA-Portal-Export | AML | öffentlich (IEC 62714) |

## Ergebnis

**LLM-generierungsfähig (K1, K3, K4 theoretisch erfüllbar):**

- Schaltschrank-Excel: Pflichtfelder und Element-Typen vollständig dokumentiert.
- PLCTAGS-Excel: Schema vollständig dokumentiert, S7-Konventionen anwendbar.

**Generierungsfähig nur mit Zusatzinput:**

- Connectors-XML, Payloads-XML: Schema-Referenz (Beispieldatei) erforderlich.
- AML: Profil-Spezifikation des TIA-Exports erforderlich.

**K2 (Importierbarkeit) ist für alle Strukturen offen**, da kein praktischer Test gegen fe.screen-sim durchgeführt wurde (kein Setup im Scope).

## Nicht analysiert

- Praktische Imports gegen fe.screen-sim (kein Setup)
- Vollständige XML-Schemata Connectors / Payloads (nicht publiziert)
- API-basierte Generierung (keine API-Doku zugänglich)
- LLM-Auswahl, Token-Verbrauch, Kosten (Scope SCRUM-9 / SCRUM-10)
- V5-Importverhalten (V5-Doku liegt nicht vor)

**Detailbericht:** SCRUM-31_bewertung.md
