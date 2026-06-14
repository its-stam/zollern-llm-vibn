# SCRUM-30 Summary -- Datenformate und Preprocessing

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 2026-04-27

## Auftrag

Festlegung der unterstützten Eingabeformate sowie der Transformations- und Preprocessing-Schritte für die LLM-Anbindung an fe.screen-sim. Übernommen von Gruppe B (LLM-Stack).

## Eingabeformate (V4-Doku belegt)

`Station Import.html` listet 13 CAD-Formate explizit: `.sim3D`, `.aml`, `.dae`, `.stp`, `.step`, `.jt`, `.wrl`, `.x3d`, `.stl`, `.iges`, `.igs`, `.sldprt`, `.sldasm`. Zusätzlich `.sim-psz` über Process-Simulate-Plugin.

## Empfehlung Eingabe-Korridor

| Prio | Format | Begründung |
|------|--------|-----------|
| 1 | STEP | Industrie-Standard, ASCII, vollständige Topologie |
| 2 | AML | XML, kann Anlagenstruktur tragen |
| 3 | DAE | XML, Geometrie + optional Kinematik |
| 4 | JT (nach Konvertierung) | bei Schwerlastbereich verbreitet |

Endgültige Wahl hängt davon ab, was Zollern liefert. Information liegt nicht vor.

## Preprocessing-Pipeline (5 Phasen)

1. **Formaterkennung** -- Whitelist + Magic-Byte
2. **Normalisierung** -- Binär→Text via fe.screen-sim CAD-Konverter
3. **Strukturextraktion** -- Assembly-Baum, Bounding-Boxes, Komponentenmetadaten
4. **Reduktion** -- Token-bewusste Auswahl, Geometrie weglassen, Selektion
5. **Übergabe an LLM** -- strukturiertes JSON/YAML (Schema in SCRUM-10 festzulegen)

Konkrete Tool-Auswahl gehört in SCRUM-10 (prototypischer Workflow).

## Was offen bleibt

- Welches Format Zollern real liefert
- LLM-Stack und Tokenbudget (Gruppe B)
- Praktische Tests gegen fe.screen-sim
- V5-spezifisches Verhalten (Doku liegt nicht vor)

## Benötigte Inputs (per Jira/Teams anfragen)

1. PO/Zollern: Beispiel-CAD-Datei und reales Anlieferungsformat
2. Gruppe B (Gruppe B/Gruppe B/Gruppe B): LLM-Stack, Kontextfenster, Eingabe-Schema
3. Gruppe A (CAD-Parsing): Erkenntnisse SCRUM-29
4. Gruppe C (SCRUM-10): Verwendung des vorgeschlagenen Zwischenformats?

## Bearbeitungsstand

Definitionsteil abgeschlossen, Pipeline konzeptionell beschrieben. Eine vollständige Festlegung aller Pipeline-Bestandteile ist erst nach den oben genannten Inputs sinnvoll und gehört nicht mehr in den Scope dieses Tickets.
