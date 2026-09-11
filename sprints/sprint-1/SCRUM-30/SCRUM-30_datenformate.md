# SCRUM-30: Definition geeigneter Datenformate und Preprocessing-Schritte

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 2026-04-27

## 1. Aufgabe (laut Jira)

> Festlegung der unterstützten Eingabeformate sowie notwendiger Transformations- und Preprocessing-Pipelines zur Vorbereitung der CAD-Daten für ein LLM.

Übernommen von Gruppe B (LLM-Stack) am 27.04.2026.

## 2. Quellen

- fe.screen-sim V4 User Documentation (englisch, lokal vorhanden)
  - `Station Import.html`
  - `Modell Importer.html`
  - `CAD Konverter.html`
  - `CADAssembly.html`
  - `Process Simulate Import.html`
  - `Projekt Erstellung und Import der CAD Files.html`
- ISO 10303 (STEP)
- Khronos Collada-Spezifikation (DAE)
- AutomationML-Spezifikation (AML, IEC 62714)
- Siemens JT-Open Spezifikation (JT)
- Vorergebnisse SCRUM-31 (Generierungsfähigkeit pro Format)
- Vorergebnisse SCRUM-32 (CAD-Limitationen)

Praktische Tests gegen fe.screen-sim wurden im Rahmen dieses Tasks nicht durchgeführt. Begründung siehe Abschnitt 8.

## 3. Eingabeformate laut V4-Doku

### 3.1 CAD-Import (`Station Import.html`)

In der V4-Doku explizit aufgelistet:

| Ext. | Typ | Standard / Quelle |
|------|-----|-------------------|
| `.sim3D` | proprietär (F.EE) | nur fe.screen-sim intern |
| `.aml` | XML | AutomationML / IEC 62714 |
| `.dae` | XML | Khronos Collada |
| `.stp` / `.step` | ASCII-Text | ISO 10303 |
| `.jt` | binär | Siemens JT-Open |
| `.wrl` | ASCII-Text | VRML 97 |
| `.x3d` | XML | ISO/IEC 19775 |
| `.stl` | ASCII oder binär | de-facto Standard |
| `.iges` / `.igs` | ASCII-Text | IGES 5.3 |
| `.sldprt` / `.sldasm` | binär | SolidWorks proprietär |

Die Doku ergänzt "and much more", führt aber keine vollständige Liste auf.

### 3.2 Process-Simulate-Import (`Process Simulate Import.html`)

Sondereingang `.sim-psz`, Export aus dem F.EE-Plugin für Process Simulate. Setzt das separat verfügbare Plugin voraus.

### 3.3 Begleitformate (kein CAD)

Aus SCRUM-31 belegt, hier nur zur Abgrenzung:

- Connectors-XML, Payloads-XML
- Schaltschrank-Excel, PLCTAGS-Excel
- TIA-AML

Diese sind nicht Gegenstand der CAD-Preprocessing-Pipeline, ergänzen die Simulationsstruktur jedoch nach dem CAD-Import.

## 4. Klassifikation der CAD-Formate für LLM-Verarbeitung

| Format | Direkt LLM-lesbar | Hinweis |
|--------|-------------------|---------|
| STEP (`.stp`/`.step`) | ja | Klartext, ISO 10303-21 |
| DAE (Collada) | ja | XML |
| AML | ja | XML |
| X3D | ja | XML |
| WRL | ja | ASCII |
| IGES | ja | ASCII |
| STL ASCII | ja | reine Geometrie, keine Struktur |
| STL binär | nein | Konvertierung erforderlich |
| JT | nein | binär, Konvertierung erforderlich |
| SLDPRT/SLDASM | nein | binär, Konvertierung erforderlich |
| sim3D | nein | F.EE-proprietär, laut V4-Doku kein Drittparteizugriff (siehe SCRUM-32, 3.2) |

"Direkt LLM-lesbar" meint: das Roh-File kann ohne Binärkonvertierung in einen Text-Tokenstrom überführt werden. Es bedeutet nicht, dass das Modell semantisch korrekt interpretiert wird (siehe SCRUM-32, 5.3).

## 5. Empfohlener Eingabe-Korridor

Auf Basis der V4-Doku und der Format-Eigenschaften ergibt sich folgende Priorisierung. Die Auswahl ist konzeptionell, nicht durch Tests validiert.

| Prio | Format | Begründung |
|------|--------|-----------|
| 1 | STEP | weit verbreiteter Industrie-Standard, ASCII, vollständige Geometrie und Topologie, in V4-Doku gelistet |
| 2 | AML | XML-basiert, IEC 62714, kann zusätzlich Anlagenstruktur und Komponentenrollen tragen |
| 3 | DAE | XML, gut für Geometrie + optionale Kinematik |
| 4 | JT (nach Konvertierung) | bei Zollern in Schwerlastbereich verbreitet, Konvertierung über `CAD Konverter` möglich |

Voraussetzung der finalen Festlegung: Zollern-seitige Klärung, in welchem Format CAD tatsächlich angeliefert wird. Diese Information liegt aktuell nicht vor.

## 6. Preprocessing-Pipeline (konzeptuell)

Die Pipeline ist in fünf Phasen gegliedert. Jede Phase ist als Black-Box-Beschreibung formuliert; die konkrete Tool-Wahl ist nicht Bestandteil dieses Tasks und gehört in SCRUM-10 (prototypischer Workflow).

```
[CAD-Datei] --> 1. Formaterkennung
              --> 2. Normalisierung (Binär -> Text falls nötig)
              --> 3. Strukturextraktion
              --> 4. Reduktion
              --> 5. Übergabe an LLM
```

### 6.1 Phase 1 -- Formaterkennung

- Prüfung der Dateiendung gegen die in 3.1 gelistete Whitelist.
- Magic-Byte-Prüfung für STL (ASCII vs. binär).
- Ablehnung nicht unterstützter Formate mit Fehlerausgabe.

### 6.2 Phase 2 -- Normalisierung

- Binärformate (JT, SLDPRT/SLDASM, STL-binär) werden zunächst in ein textbasiertes Zwischenformat überführt. fe.screen-sim bringt hierfür den `CAD Konverter` mit, der Konvertierung zwischen den in 3.1 gelisteten Formaten erlaubt (Quelle: `CAD Konverter.html`).
- Textbasierte Formate werden unverändert weitergereicht.
- Kein semantischer Eingriff in dieser Phase.

### 6.3 Phase 3 -- Strukturextraktion

Ziel: Aus dem CAD-File ein reduziertes, semantisch annotiertes Modell erzeugen, das ein LLM verarbeiten kann.

- Extraktion der Baumstruktur (Assemblies, Subassemblies, Parts) aus STEP/DAE/AML.
- Extraktion benannter Bauteile (Naming aus dem CAD-Modell, sofern vorhanden).
- Extraktion von Bounding-Boxes und Ankerpunkten je Komponente.
- Optional: Extraktion vorhandener Kinematik aus AML / DAE, sofern modelliert.

Was hier nicht extrahiert werden kann:

- Funktionssemantik (Förderer, Antrieb, Greifer) -- ist nicht zwingend in CAD enthalten (SCRUM-32, 5.3).
- Steuerlogik und Sensorik -- gehört in PLCTAGS / TIA-AML, nicht ins CAD (SCRUM-31, 4).

### 6.4 Phase 4 -- Reduktion

LLMs haben pro Anfrage ein Token-Limit. Rohe STEP/DAE-Dateien grosser Anlagen sprengen typische Kontextfenster (siehe SCRUM-32, 5.2). Massnahmen:

- Geometrie-Dezimierung: fe.screen-sim bietet im Modell-Importer eine "Decimate/Reduce"-Funktion mit prozentualer Qualität (Quelle: `Modell Importer.html`). Diese ist auf Visualisierung ausgelegt; für LLM-Vorbereitung muss eine eigene Reduktion vor dem fe.screen-sim-Import erfolgen.
- Strukturreduktion: nur Hierarchie und Komponentenmetadaten an LLM weitergeben, Geometrie-Vertices nicht.
- Selektion: nur die für die Aufgabe relevanten Subassemblies übergeben.

Konkrete Tool-Wahl (z. B. CAD-Toolkette zur Programmatik) ist offen und gehört in SCRUM-10.

### 6.5 Phase 5 -- Übergabe an LLM

- Format der LLM-Eingabe ist nicht abschliessend festgelegt. Vorschlag: strukturiertes JSON oder YAML mit Komponentenbaum, Bounding-Boxes und Metadaten.
- Begründung des Vorschlags: textbasiert, schemafest, in SCRUM-31 als für LLMs gut generierbar bestätigt.
- Festlegung des konkreten Schemas erfordert Abstimmung mit Gruppe B (LLM-Stack, Tokenbudget, Prompttemplate). Aktuell offen.

## 7. Vorschlag Zwischenformat

Als Schnittstellenformat zwischen Preprocessing und LLM-Stufe wird ein strukturiertes Komponentenmodell vorgeschlagen. Beispielhafter Aufbau:

```yaml
assembly:
  id: "asm_root"
  name: "string aus CAD"
  bbox: [xmin, ymin, zmin, xmax, ymax, zmax]
  children:
    - id: "asm_001"
      name: "string"
      bbox: [...]
      type: "unknown"     # optional, falls Klassifikation vor LLM
      children: [...]
parts:
  - id: "part_001"
    name: "string"
    bbox: [...]
    parent: "asm_001"
```

Dieses Schema ist ein Vorschlag und nicht verbindlich. Festlegung gehört in SCRUM-10. Die Datentiefe (mit/ohne Mesh, mit/ohne Material) richtet sich nach Tokenbudget und LLM-Wahl.

## 8. Was nicht festgelegt wurde und warum

| Punkt | Grund |
|-------|-------|
| Endgültige Auswahl der CAD-Eingabeformate | Hängt davon ab, in welchem Format Zollern Modelle anliefert. Information liegt nicht vor. |
| Konkrete Toolkette für Phase 2-4 | Erfordert Test mit echtem Beispielmodell. Gehört in SCRUM-10. |
| Schema des Zwischenformats | Erfordert Abstimmung mit Gruppe B (LLM-Stack, Tokenbudget). |
| Token-Verbrauch pro Format | Keine Messreihe vorhanden; ohne Beispieldatei nicht ermittelbar. |
| Verhalten der V5-Importpipeline | V5-Doku liegt nicht vor, nur V4. |
| API-basierter Übergabeweg an fe.screen-sim | API-Beschreibung in V4-Doku nicht enthalten (SCRUM-32, 5.4). |

## 9. Offene Fragen / benötigte Inputs

Folgende Punkte sind für die Umsetzung der Pipeline (SCRUM-10) erforderlich und sollen über Jira-Kommentar bzw. Teams angefragt werden:

1. **An Zollern (über Product Owner (Kundenseite)):** Welche CAD-Formate werden im realen Anlagenkontext angeliefert? Liegt eine Beispieldatei (Förderer / Drehtisch o. ä.) vor?
2. **An Gruppe B (LLM-Stack):** Welcher LLM-Stack wird in SCRUM-40 angebunden? Kontextfenster? Strukturierte JSON-Eingabe oder Freitext-Prompt?
3. **An Gruppe A (CAD-Parsing):** Stand SCRUM-29 (CAD-LLM-Interpretation) -- gibt es Erkenntnisse zu funktionierenden Eingabeformaten aus deren Tests?
4. **An Gruppe C (SCRUM-10):** Soll das in 7. vorgeschlagene Zwischenformat als Ausgangsschema für den fiktiven Workflow verwendet werden?
5. **An PO:** Liegt eine V5-Doku vor oder bleibt V4 die Referenz?

## 10. Fazit

Die Definition der Eingabeformate ist auf Basis der V4-Doku belegbar abgeschlossen: 13 Formate sind explizit unterstützt, vier davon (STEP, AML, DAE, JT mit Konvertierung) werden für die LLM-Pipeline empfohlen.

Die Preprocessing-Pipeline ist in fünf konzeptionellen Phasen beschrieben. Die Phasen 1-2 sind anhand der V4-Doku belegbar. Die Phasen 3-5 sind als Konzept formuliert; die konkrete Tool-Auswahl, das LLM-Eingabeschema und das Zwischenformat erfordern Abstimmung mit Gruppe B und einen praktischen Test im Rahmen von SCRUM-10.

Damit liegt für SCRUM-30 ein dokumentierter, belegbarer Stand vor. Eine vollständige Festlegung aller Pipeline-Bestandteile ist erst nach den unter 9. genannten Inputs sinnvoll und gehört nicht mehr in den Scope dieses Tickets.
