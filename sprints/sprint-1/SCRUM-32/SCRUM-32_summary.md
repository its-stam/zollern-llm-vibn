# SCRUM-32 Summary

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 26.04.2026

## Aufgabe (laut Jira)

Analyse der CAD-Dateninterpretation durch LLMs sowie Identifikation und Dokumentation technischer Limitationen. Erfassung von Einschränkungen hinsichtlich Performance, Datenkomplexität, Modellgrenzen und Integrationsaufwand.

## Quellen

fe.screen-sim V4 User Documentation (lokal) sowie öffentliche Standard-Spezifikationen der genannten CAD-Formate (ISO 10303, Collada, AutomationML, JT-Open).

## Ergebnisse

**Unterstützte Formate (V4-Doku):** `.aml`, `.dae`, `.stp`, `.step`, `.jt`. Doku ergänzt "and many more" ohne weitere Auflistung.

**Importverhalten:** Die Quelldatei wird in ein proprietäres F.EE-Format konvertiert. Drittparteizugriff laut Doku nicht vorgesehen.

**Limitationen (aus Doku und Format-Spezifikationen ableitbar):**

1. **Eingriffspunkt LLM:** nur vor dem Import möglich. Internes Format ist abgeschottet.
2. **LLM-Lesbarkeit:** STEP, DAE und AML sind textbasiert und direkt lesbar. JT ist binär und nicht direkt verarbeitbar.
3. **Modellgrenzen:** Funktionssemantik (Förderer, Antrieb, Greifer), Steuerlogik und teilweise Kinematik sind in CAD-Quelldaten nicht enthalten und können nicht aus diesen abgeleitet werden.
4. **Integrationsaufwand:** API-Beschreibung von fe.screen-sim für externe Integrationen ist in der V4-Doku nicht enthalten.
5. **Performance (Import):** "few minutes" je nach Modellgrösse laut Doku. Konkrete Werte nicht dokumentiert.

## Nicht analysiert

- Praktische Tests mit CAD-Dateien (keine Beispieldatei zugänglich)
- Konkrete LLM-Token-Verbräuche, Latenz, Kosten (kein Messsetup im Scope)
- V5-Importpipeline (V5-Doku liegt nicht vor)
- Vollständige Formatliste (V4 zählt nur Auswahl auf)
- API-Möglichkeiten fe.screen-sim für externe Datenübergabe (Doku zur API nicht zugänglich)

**Detailbericht:** SCRUM-32_limitationen.md
