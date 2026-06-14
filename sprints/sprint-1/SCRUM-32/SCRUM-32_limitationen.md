# SCRUM-32: Analyse CAD-Dateninterpretation durch LLMs und technische Limitationen

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 26.04.2026

## 1. Aufgabe (laut Jira)

Analyse der CAD-Dateninterpretation durch LLMs sowie Identifikation und Dokumentation technischer Limitationen. Erfassung von Einschränkungen hinsichtlich Performance, Datenkomplexität, Modellgrenzen und Integrationsaufwand. Strukturierte Dokumentation.

## 2. Quellen

- fe.screen-sim V4 User Documentation (englisch, lokal vorhanden)
  - `CAD Konverter.html`
  - `Modell Importer.html`
  - `CADAssembly.html`
  - `Projekt Erstellung und Import der CAD Files.html`
- ISO 10303 (STEP-Standard, allgemein dokumentiert)
- Khronos Collada-Spezifikation (DAE)
- AutomationML-Spezifikation (AML)
- Siemens JT-Open Spezifikation (JT)

Praktische Tests mit fe.screen-sim wurden nicht durchgeführt. Begründung siehe Abschnitt 6.

## 3. Sachstand laut V4-Doku

### 3.1 Unterstützte CAD-Formate

In der V4-Doku explizit benannt: `.aml`, `.dae`, `.stp`, `.step`, `.jt`. Die Doku ergänzt "and many more", ohne weitere Formate aufzuzählen.

### 3.2 Importverhalten

Beim Import konvertiert fe.screen-sim die Quelldatei in ein "file format developed by F.EE". Begründung laut Doku: "even if the project is passed on, access to the 3D data by third parties is not possible" (Quelle: `Modell Importer.html`).

### 3.3 CAD-Assembly

`CADAssembly` ist laut Doku ein reines Strukturelement zur Aufnahme importierter CAD-Dateien, ohne zusätzliche Funktionen (Quelle: `CADAssembly.html`).

### 3.4 Importdauer

"Depending on the size of the CAD model and the performance of the PC, the import process can take a few minutes." (Quelle: `Projekt Erstellung und Import der CAD Files.html`).

## 4. Eigenschaften der genannten Formate

| Format | Typ | LLM-Lesbarkeit (rohe Datei) |
|--------|-----|------------------------------|
| `.stp` / `.step` (ISO 10303) | ASCII-Text | direkt lesbar |
| `.dae` (Collada) | XML | direkt lesbar |
| `.aml` (AutomationML) | XML | direkt lesbar |
| `.jt` (Jupiter Tessellation) | binär | nicht direkt lesbar |

Quelle: jeweilige Standard-Spezifikationen.

## 5. Limitationen

### 5.1 Performance

- Importdauer fe.screen-sim: laut Doku "few minutes" je nach Modellgrösse und PC-Leistung. Konkrete Werte nicht dokumentiert.
- Latenz und Durchsatz eines LLM bei CAD-Verarbeitung: nicht analysiert. Erfordert praktische Tests, siehe Abschnitt 6.

### 5.2 Datenkomplexität

- Textbasierte Formate (STEP, DAE, AML) enthalten Geometrie und Topologie in vollständig serialisierter Form. Für grosse Baugruppen führt dies zu sehr umfangreichen Dateien. Genaue Grenzwerte für die LLM-Verarbeitbarkeit (Context-Limit) hängen vom eingesetzten LLM ab und wurden nicht gemessen.
- JT als Binärformat ist ohne vorherige Konvertierung für ein LLM nicht interpretierbar.

### 5.3 Modellgrenzen

Folgende Inhalte sind in CAD-Quelldaten typischerweise nicht enthalten und können von einem LLM aus diesen Daten allein nicht abgeleitet werden:

- Funktionssemantik (welche Komponente ist Förderer, Antrieb, Greifer)
- Steuerlogik (SPS-Programm, Sensorik)
- Bewegungs- und Achsdefinitionen, sofern nicht im Format hinterlegt (STEP enthält keine Kinematik, DAE optional)

Diese Aussagen folgen aus den Standard-Spezifikationen der jeweiligen Formate.

### 5.4 Integrationsaufwand

- Zugriff auf das proprietäre F.EE-Format ist laut Doku für Drittparteien nicht vorgesehen. Konsequenz: Eine LLM-Integration auf Datenebene kann ausschliesslich vor dem Importschritt ansetzen.
- Eine API-Beschreibung von fe.screen-sim für externe Integrationen ist in der V4-Doku nicht enthalten.
- Die Kommunikationsschnittstellen-Übersicht der V4-Doku (`Schnittstellen_grund.html`) beschreibt SPS- und Roboter-Interfaces, jedoch keine Schnittstelle zur Übergabe von extern generierten Modell- oder Simulationsdaten.

## 6. Was nicht analysiert wurde und warum

| Punkt | Grund |
|-------|-------|
| Praktische LLM-Tests mit echten CAD-Dateien | Keine CAD-Beispieldateien des Projekts zugänglich. Eigene Tests mit öffentlichen Dateien wurden im Rahmen dieses Tasks nicht durchgeführt. |
| Konkrete Token-Verbräuche pro CAD-Datei | Keine Messreihe vorhanden. Erfordert definierte Beispiel-Datei und festgelegtes LLM. |
| Latenz und Kosten LLM-API | Nicht im Scope dieses Tasks gemessen. |
| Verhalten der V5-Importpipeline | V5-Doku liegt nicht vor, nur V4. |
| Vollständige Liste unterstützter Formate | V4-Doku zählt nur Auswahl auf, vollständige Liste nicht zugänglich. |
| API-Möglichkeiten fe.screen-sim für externe Datenübergabe | Doku zur API nicht zugänglich. |

## 7. Fazit zum Bearbeitungsstand SCRUM-32

Die Analyse identifiziert auf Basis der vorliegenden V4-Doku und der bekannten Format-Spezifikationen die strukturellen Limitationen einer LLM-gestützten CAD-Verarbeitung in fe.screen-sim:

1. LLM-Zugriff nur vor dem Import möglich, da das interne F.EE-Format laut Doku abgeschottet ist.
2. Direkt LLM-lesbar sind nur textbasierte Formate (STEP, DAE, AML), nicht JT.
3. Funktionssemantik und Steuerlogik sind nicht Bestandteil der CAD-Daten.

Quantitative Aussagen zu Performance, Token-Verbrauch oder LLM-Genauigkeit erfordern praktische Tests mit definierter Beispiel-Datei und festgelegtem LLM. Dies wurde im Rahmen von SCRUM-32 nicht durchgeführt und ist offen.
