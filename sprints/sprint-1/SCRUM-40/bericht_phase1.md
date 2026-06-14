# Machbarkeitsbericht: Foerderer_Drehtisch_v1

**Anlage:** Foerderer_Drehtisch_v1
**Datum:** 2026-04-29
**Projekt:** Zollern – Machbarkeitsstudie Virtuelle Inbetriebnahme + KI
**Status:** Machbar mit Einschränkungen

---

## 1. Executive Summary

Die Anlage Foerderer_Drehtisch_v1 ist eine sechskomponentige Förder- und Positionieranlage mit zwei synchronisierten Förderbändern und einem zentral gesteuerten Drehtisch. Die hierarchische Komponentenstruktur ist klar definiert und vollständig klassifiziert. Eine Simulation in fe.screen-sim V5 ist technisch machbar, erfordert jedoch eine strukturierte Datenübergabe und Klarstellung der SPS-Schnittstellen.

---

## 2. Komponentenübersicht

| ID | Name | Funktionsklasse | Material | Abmessungen (mm) | Parent | Funktion |
|---|---|---|---|---|---|---|
| K001 | Hauptrahmen | Tragstruktur | Stahl | 2000 × 800 × 600 | – | Träger aller Unterkomponenten |
| K002 | Foerderband_Links | Fördertechnik | Gummi/Stahl | 1800 × 200 × 100 | K001 | Transport Links, 500 mm/s |
| K003 | Foerderband_Rechts | Fördertechnik | Gummi/Stahl | 1800 × 200 × 100 | K001 | Transport Rechts, 500 mm/s, synchron |
| K004 | Drehtisch_Zentrum | Drehtisch | Stahl | 600 × 600 × 200 | K001 | Rotation um Z-Achse, 0–100 rpm |
| K005 | Antriebsmotor_Drehtisch | Antrieb | Metall | 200 × 200 × 300 | K004 | Antriebserzeugung für K004 |
| K006 | Sensor_Endlage | Sensor | Kunststoff/Metall | 50 × 50 × 80 | K004 | Erkennung Endlagen (0°/90°/180°/270°) |

---

## 3. Empfehlung Importformat für fe.screen-sim V5

**Priorität 1 (Empfohlen): AutomationML (AML)**

- **Machbarkeit:** Hoch
- **Begründung:** AML bietet nativen Support für SystemUnitClass-Definitionen und InternalElements für Komponenteninstanzen. Die hierarchische Baumstruktur der Anlage wird durch verschachtelte Elemente direkt abgebildet. Ermöglicht zukünftige Erweiterungen für SPS-Variablenbindung und Kinematikdaten.
- **Umsetzungsaufwand:** Mittel

**Priorität 2 (Alternative): Connectors-XML**

- **Machbarkeit:** Hoch
- **Begründung:** XML bildet die Komponenten-Kind-Beziehungen natürlich ab. Keine ID-basierte Referenzverwaltung erforderlich. Erweiterbar und toolunabhängig.
- **Umsetzungsaufwand:** Niedrig

**Nicht empfohlen:** Schaltschrank-Excel (flache Struktur), PLCTAGS-Excel (fokussiert auf SPS-Variablen, nicht Geometrie).

---

## 4. Simulationsstruktur-Vorschlag

### 4.1 Bewegungsmodell

| Komponente | Bewegungsart | Parameter | Steuerung |
|---|---|---|---|
| K002 (Foerderband_Links) | Linear, X-Achse | v = 500 mm/s, konstant | `FB_Links_Laufen` (BOOL) |
| K003 (Foerderband_Rechts) | Linear, X-Achse | v = 500 mm/s, synchron mit K002 | `FB_Rechts_Laufen` (BOOL) |
| K004 (Drehtisch_Zentrum) | Rotativ, Z-Achse | n = 0–100 rpm | `Drehtisch_Freigabe` (BOOL), `Drehtisch_Drehzahl` (INT) |
| K005 (Antriebsmotor) | Rotativ | Abhängig von K004 | Untergeordnet K004 |

### 4.2 Sensormodell

| Sensor-ID | Auslöser | SPS-Variable |
|---|---|---|
| K006 (Endlage) | Drehtisch bei 0°/90°/180°/270° | `Endlage_erreicht` (BOOL) |
| Sensor_Foerderband_Links | Werkstück auf K002 | `Werkstück_Links_erkannt` (BOOL) |
| Sensor_Foerderband_Rechts | Werkstück auf K003 | `Werkstück_Rechts_erkannt` (BOOL) |

### 4.3 SPS-Schnittstelle

**Eingänge (Stellbefehle):**

```
FB_Links_Laufen          [BOOL]   Start/Stop Förderband Links
FB_Rechts_Laufen         [BOOL]   Start/Stop Förderband Rechts
Drehtisch_Freigabe       [BOOL]   Aktivierung Drehbetrieb
Drehtisch_Drehzahl       [INT]    Sollwert 0–100 rpm
Drehtisch_Zielwinkel     [INT]    Sollwinkel in Grad (0/90/180/270)
```

**Ausgänge (Rückmeldungen):**

```
Endlage_erreicht             [BOOL]   Drehtisch-Position erreicht
Werkstück_Links_erkannt      [BOOL]   Detektiert auf Förderband Links
Werkstück_Rechts_erkannt     [BOOL]   Detektiert auf Förderband Rechts
Betriebszustand              [INT]    0=Stillstand, 1=Lauf, 2=Fehler
```

### 4.4 Ablauflogik

```
[START]
  ↓
[IDLE: Warten auf SPS-Freigabe]
  ├─→ FB_Links_Laufen = true  → K002 linear X-Richtung
  ├─→ FB_Rechts_Laufen = true → K003 linear X-Richtung (synchron)
  ↓
[TRANSPORT: Werkstück auf Förderbändern]
  ├─→ Werkstück erkannt (Sensor) → SPS informieren
  ↓
[HANDOFF: Werkstück auf Drehtisch positioniert]
  ↓
[DREHBETRIEB: Drehtisch_Freigabe = true]
  ├─→ K004 rotiert bis Zielwinkel
  ├─→ K006 detektiert Endlage → Endlage_erreicht = true
  ↓
[STOP → Rückkehr zu IDLE]
```

---

## 5. Limitationen und offene Punkte

| Punkt | Status | Erforderlich für |
|---|---|---|
| SPS-Adressierung | TBD | Hardware-Integration |
| Sensor-Typ (IR/Taster) | Nicht spezifiziert | Physikalisches Sensormodell |
| Mechanische Kopplungsdetails K005→K004 | Nicht definiert | Kinematik-Simulation |
| Werkstückgeometrie | Nicht vorhanden | Kollisionserkennung |
| Echtes CAD-Format von Zollern | STEP bestätigt | Realer Import |

**Annahmen für Simulation:**
1. Beide Förderbänder starten/stoppen gleichzeitig ohne Verzögerung.
2. Drehtisch-Rotation ohne Schaukelbewegung; Endlagenerkennung unverzögert.
3. SPS-Zykluszeit mind. 50 ms ausreichend für Rückkopplung.

---

## 6. Nächste Schritte

**Phase 1 – Datenaufbereitung:**
- AML-Export basierend auf Komponentendefinitionen generieren
- SPS-Adressen konkretisieren (Hersteller, Modell, Speicheradressen)
- Sensor-Typ für K002/K003 festlegen

**Phase 2 – Simulationsimplementierung:**
- AML-Datei in fe.screen-sim V5 importieren
- Bewegungsprofile parametrieren (500 mm/s, 0–100 rpm)
- SPS-Schnittstellenanbindung via OPC-UA oder TCP/IP

**Phase 3 – Validierung:**
- Testfall: Beide Förderbänder starten, Werkstück erkannt, Drehtisch rotiert zu 90°
- Performance-Check bei 20 Hz Simulationsfrequenz

---

## Fazit

Die Anlage Foerderer_Drehtisch_v1 eignet sich für eine fe.screen-sim-V5-Simulation. **AutomationML ist das bevorzugte Importformat.** Kritische Erfolgsfaktoren: vollständige SPS-Adressierung, Sensor-Typ-Konkretisierung, Validierung der Hierarchie-Unterstützung in fe.screen-sim V5. Umsetzung in 4–5 Wochen realistisch.

---

*Academic feasibility study, HS Albstadt-Sigmaringen. No customer data: all runs use a synthetic test plant.*
