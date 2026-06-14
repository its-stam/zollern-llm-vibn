# SCRUM-31: Bewertung der Generierungsfähigkeit von Simulationsstrukturen durch LLMs

**Bearbeitung:** Rustam Kohen | **Sprint:** 1 | **Stand:** 2026-04-27

## 1. Aufgabe (laut Jira)

Analyse, ob und wie ein LLM aus gegebenen Eingabedaten valide Simulationsstrukturen ableiten oder generieren kann. Ergebnisbewertung anhand definierter Kriterien.

## 2. Quellen

- fe.screen-sim V4 User Documentation (englisch, lokal vorhanden)
  - `Import und Export von Connectoren.html`
  - `Import und Export von Foerdergut.html`
  - `Import und Export von Schaltschraenken.html`
  - `Import aus Excel Tabelle.html`
  - `Export und Import der SPS Variablen.html`
  - `Export einer AML aus dem TIA Portal.html`
  - `CADAssembly.html`, `Modell Importer.html`
  - `Schnittstellen_grund.html`
- AutomationML-Spezifikation (AML, IEC 62714)
- Vorergebnisse SCRUM-32 (CAD-Limitationen)

Praktische LLM-Generierungstests gegen fe.screen-sim wurden nicht durchgeführt. Begründung siehe Abschnitt 7.

## 3. Definition Simulationsstruktur in fe.screen-sim

Unter "Simulationsstruktur" wird in dieser Bewertung jede über die V4-Doku belegbare extern importierbare Datei verstanden, die Bestandteile einer Simulation in fe.screen-sim definiert oder konfiguriert. Nicht eingeschlossen ist das proprietäre F.EE-Format, da laut V4-Doku kein Drittparteizugriff vorgesehen ist (siehe SCRUM-32, Abschnitt 3.2).

## 4. Übersicht extern importierbarer Strukturen

| # | Strukturtyp | Format | Quelle V4-Doku |
|---|-------------|--------|----------------|
| 1 | Connectors (Steckverbinder + Kindelemente) | XML | `Import und Export von Connectoren.html` |
| 2 | Payloads (Fördergut) | XML | `Import und Export von Foerdergut.html` |
| 3 | Schaltschrank-Konfiguration | Excel (Spalten A–E, definierte Element-Typen) | `Import und Export von Schaltschraenken.html` |
| 4 | SPS-Variablen ("PLCTAGS") | Excel (Name, Path, Data Type, Logical Address, Comment) | `Import aus Excel Tabelle.html` |
| 5 | TIA-Portal-Export | AML (AutomationML) | `Export einer AML aus dem TIA Portal.html` |
| 6 | CAD-Modell (Geometrie) | `.stp/.step/.dae/.aml/.jt` | `Modell Importer.html` |

Strukturen 1–4 haben ein dokumentiertes Schema mit benannten Feldern. Struktur 5 (AML) folgt einem öffentlich spezifizierten XML-Schema. Struktur 6 ist Geometrie-Import und in SCRUM-32 vollständig behandelt; sie ist in dieser Bewertung daher zweitrangig.

## 5. Bewertungskriterien

Für die Bewertung der LLM-Generierungsfähigkeit werden vier Kriterien verwendet. Die Kriterien sind aus der V4-Doku ableitbar (Pflichtfelder, Schemavorgaben) bzw. ergeben sich aus dem Generierungsziel.

| Krit. | Bezeichnung | Definition |
|-------|-------------|------------|
| K1 | Schema-Konformität | Generierte Datei entspricht der in der V4-Doku oder der jeweiligen Format-Spezifikation vorgegebenen Struktur (Pflichtfelder, Spalten-Reihenfolge, Datentypen). |
| K2 | Importierbarkeit | fe.screen-sim akzeptiert die Datei beim Import ohne Fehler. Setzt praktischen Test gegen fe.screen-sim voraus. |
| K3 | Semantische Korrektheit | Inhalt entspricht der intendierten Anlage (z. B. ein Schaltschrank mit korrekt benannten Schützen, Tastern, Hauptschalter). |
| K4 | Vollständigkeit | Alle laut Doku zwingend erforderlichen Felder sind besetzt. |

Quantitative Schwellenwerte (z. B. „Importrate ≥ 95 %") werden in dieser theoretischen Bewertung nicht festgelegt, da kein Messsetup vorliegt.

## 6. Bewertung je Strukturtyp

### 6.1 Connectors (XML)

| Krit. | Bewertung | Begründung |
|-------|-----------|------------|
| K1 | Theoretisch erfüllbar | XML-Schema laut V4-Doku nicht vollständig publiziert. Generierung erfordert Beispieldatei aus fe.screen-sim als Referenz. |
| K2 | Nicht prüfbar | Praktischer Test ausstehend. |
| K3 | LLM-geeignet | Connectors sind benannte Objekte mit Position; LLM kann strukturiert ableiten, sofern Spezifikation oder Beispiel vorliegt. |
| K4 | Theoretisch erfüllbar | Hängt von vollständig vorliegender Beispieldatei ab. |

**Voraussetzung für Generierung:** Mindestens eine exportierte Beispiel-XML als Schema-Referenz.

### 6.2 Payloads (XML)

Bewertung analog zu 6.1. Die V4-Doku beschreibt das Format als XML mit Anzahl, Position und „data" der Payloads. Vollständiges Schema wird in der Doku nicht aufgelistet.

### 6.3 Schaltschrank (Excel)

| Krit. | Bewertung | Begründung |
|-------|-----------|------------|
| K1 | Erfüllbar | Spalten A–E sind in der Doku exakt benannt: Name, Type, Label, X-Position, Y-Position. |
| K2 | Nicht prüfbar | Praktischer Test ausstehend. |
| K3 | LLM-geeignet | Element-Typen sind eine geschlossene Liste laut Doku (u. a. MainSwitch, MotorProtector, Push button KP32, NumericInput, Seven segments display). LLM kann aus Anforderungstext eine konforme Tabelle erzeugen. |
| K4 | Erfüllbar | Pflichtfelder vollständig dokumentiert. |

**Generierungsbeispiel (illustrativ, nicht getestet):**

| A (Name) | B (Type) | C (Label) | D (X) | E (Y) |
|----------|----------|-----------|-------|-------|
| MS1 | MainSwitch | Hauptschalter | 0 | 0 |
| BT1 | Push button KP32 | Start | 150 | 0 |
| BT2 | Push button KP32 | Stop | 300 | 0 |

### 6.4 SPS-Variablen (Excel "PLCTAGS")

| Krit. | Bewertung | Begründung |
|-------|-----------|------------|
| K1 | Erfüllbar | Spalten A–E exakt definiert: Name, Path, Data Type, Logical Address, Comment. Dateiname muss "PLCTAGS" lauten. |
| K2 | Nicht prüfbar | Praktischer Test ausstehend. |
| K3 | LLM-geeignet bei Nomenklatur-Vorgabe | Logical Address und Data Type folgen Siemens-S7-Konventionen (z. B. `%I0.0`, `BOOL`). LLM kann konforme Tags erzeugen, sofern Adressraum und Naming-Konvention vorgegeben sind. |
| K4 | Erfüllbar | Schema vollständig dokumentiert. |

**Hinweis:** Die V4-Doku beschreibt zusätzlich einen TIA-Connector-Workflow, der Variablen direkt aus dem TIA Portal überträgt. Eine LLM-Generierung ist somit eine Alternative für Fälle ohne TIA-Projekt.

### 6.5 AML aus TIA Portal

| Krit. | Bewertung | Begründung |
|-------|-----------|------------|
| K1 | Erfüllbar | AML ist ein öffentliches XML-Schema (IEC 62714). LLM kann konforme Strukturen erzeugen. |
| K2 | Nicht prüfbar | fe.screen-sim akzeptiert laut Doku AML aus TIA-Portal-Export; ob extern generierte AML akzeptiert werden, ist nicht dokumentiert. |
| K3 | Eingeschränkt geeignet | TIA-Portal-AML enthält PLC-spezifische Profile, die ohne TIA-Quelle nicht trivial nachbildbar sind. |
| K4 | Mit Profilbeschreibung erfüllbar | Erfordert zusätzlich Profil-Spezifikation des TIA-Exports. |

### 6.6 CAD-Modell

Auf SCRUM-32 verwiesen. STEP/DAE/AML als Text generierbar, jedoch geometrisch konsistente CAD-Erzeugung ist Modellierungsaufgabe und liegt ausserhalb typischer LLM-Generierungsmuster ohne CAD-Toolkette.

## 7. Was nicht bewertet wurde und warum

| Punkt | Grund |
|-------|-------|
| K2 für alle Strukturen (Importierbarkeit) | Kein praktischer Test gegen fe.screen-sim durchgeführt. Erfordert Lizenz, Installation, Beispiel-Importroutine. |
| Vollständige XML-Schemata Connectors / Payloads | Nicht in V4-Doku publiziert. Erfordert exportierte Referenzdateien. |
| API-basierte Generierung | Eine API-Beschreibung von fe.screen-sim ist in der V4-Doku nicht enthalten (siehe SCRUM-32, 5.4). |
| LLM-Auswahl (Modell, Token-Verbrauch, Kosten) | Out of scope dieses Tasks. Wird in SCRUM-9 / SCRUM-10 betrachtet. |
| V5-spezifisches Importverhalten | V5-Doku liegt nicht vor. |

## 8. Fazit

Die Bewertung führt zu folgenden belegbaren Aussagen:

1. fe.screen-sim akzeptiert laut V4-Doku mindestens fünf extern importierbare Strukturtypen (Connectors-XML, Payloads-XML, Schaltschrank-Excel, PLCTAGS-Excel, TIA-AML).
2. Für die beiden Excel-Formate (Schaltschrank, PLCTAGS) ist die V4-Doku schemaseitig vollständig genug, um eine LLM-gestützte Generierung schemakonform (K1) und vollständig (K4) zu ermöglichen.
3. Für Connectors-XML und Payloads-XML ist eine LLM-Generierung erst nach Vorlage einer exportierten Beispieldatei sinnvoll, da die V4-Doku das vollständige XML-Schema nicht offenlegt.
4. AML-Generierung ist syntaktisch möglich, semantisch jedoch durch fehlende Profil-Spezifikationen eingeschränkt.
5. Importierbarkeit (K2) und damit die Validierung jeder generierten Struktur erfordert einen praktischen Test gegen fe.screen-sim und ist im Rahmen dieses Tasks nicht erbracht.

Damit liegt eine theoretische Generierungsfähigkeit für die Excel-basierten Strukturen vor (K1, K3, K4 erfüllbar). Für XML- und AML-Strukturen ist sie an die Bereitstellung von Schema oder Beispieldateien gebunden. Eine abschließende Validierung steht für alle Strukturen aus.
