# MCP Tools Specification – fe.screen-sim V5
**Für:** Gruppe B (MCP) (Grp B LLM) | **Von:** Rustam Kohen (Grp C) | **Stand:** 2026-04-28

Basis: Analyse YouTube-Demo U2PuFm768SQ (F.EE GmbH, Screenshots IMG_8128–8144).

---

## Überblick

Phase 2 des SCRUM-40 Workflows ruft 11 MCP-Tools in fester Reihenfolge auf:

```
1. import_cad_file       ← STEP einlesen
2. get_all_objects       ← Objekt-Liste abrufen
3. rename_object         ← (n-mal, je Objekt mit Inventor-Tag)
4. set_object_type       ← (n-mal, je Objekt)
5. set_property          ← (n-mal, für Sensor+Motorisiert Extras)
6. configure_motionjoint ← (n-mal, nur Motorisiert mit JointType)
7. create_button         ← (pro Surface 2x: Forward + Reverse)
8. create_logic_object   ← (pro Surface 1x: OR_Velocity)
9. link_objects          ← (pro Surface 6x: Buttons→Logic→Surface)
10. documentation_execute ← optional, für Doku-Output
11. content_help / object_help ← optional, für Debugging
```

---

## Tool-Definitionen

### 1. import_cad_file

Lädt eine STEP-Datei in die aktive fe.screen-sim Station.

```
Request:
  tool: "import_cad_file"
  params:
    stationFilePath: string   # absoluter Pfad zur .stp Datei
                              # Beispiel: "C:\CAD\Conveyor\anlage.stp"

Response:
  success: boolean
  station_id: string          # ID der geladenen Station
  message: string
```

### 2. get_all_objects

Gibt alle Objekte der aktuell geladenen Station zurück inkl. Inventor User Defined Properties falls vorhanden.

```
Request:
  tool: "get_all_objects"
  params: {}                  # keine Parameter

Response:
  objects: Array<{
    id: string                # interne fe.screen-sim ID, z.B. "obj_010"
    name: string              # aktueller Anzeigename
    inventor_tags: {
      Objektname:     string | null
      Simulationstyp: string | null   # "Surface" | "Motorisiert" | "Sensor" | null
      JointType:      string | null   # "RevoluteJoint" | "PrismaticJoint" | null
    }
    position: { x: number, y: number, z: number }
    bounding_box: { x: number, y: number, z: number }
  }>
```

### 3. rename_object

Benennt ein einzelnes Objekt um.

```
Request:
  tool: "rename_object"
  params:
    object_id: string         # ID aus get_all_objects
    new_name: string          # neuer Anzeigename (aus Inventor-Tag "Objektname")

Response:
  success: boolean
  object_id: string
  old_name: string
  new_name: string
```

**Hinweis:** Muss BEFORE set_object_type aufgerufen werden (Name wird bei Typ-Setzung als Referenz verwendet).

### 4. set_object_type

Setzt den Simulationstyp eines Objekts in fe.screen-sim.

```
Request:
  tool: "set_object_type"
  params:
    object_id: string
    type: string              # "Surface" | "Motorisiert" | "Sensor"

Response:
  success: boolean
  object_id: string
  type_set: string
```

**Reihenfolge:** Nach rename_object, vor configure_motionjoint.

### 5. set_property

Setzt eine zusätzliche Eigenschaft auf einem Objekt (verwendet nach set_object_type).

```
Request:
  tool: "set_property"
  params:
    object_id: string
    property: string          # Eigenschaftsname
    value: any                # Wert (boolean | number | array)

Bekannte Kombinationen (aus F.EE Demo):
  Sensor:     property="DetectPayload"  value=true
  Motorisiert: property="JointScale"   value=[0.5, 0.5, 0.5]

Response:
  success: boolean
  object_id: string
  property: string
  value_set: any
```

**Offen:** JointScale-Bedeutung für RevoluteJoint vs PrismaticJoint noch zu bestätigen (Zollern).

### 6. configure_motionjoint

Konfiguriert das Motion-Joint eines Motorisiert-Objekts.

```
Request:
  tool: "configure_motionjoint"
  params:
    object_id: string
    joint_type: string        # "RevoluteJoint" | "PrismaticJoint"
    motion_source: string     # Name des Motorisiert-Objekts als Antriebsquelle

Response:
  success: boolean
  object_id: string
  joint_configured: string

Beispiel:
  object_id: "obj_020"        # DT01 (Drehtisch)
  joint_type: "RevoluteJoint"
  motion_source: "Motorisiert"
```

**Abhängigkeit:** set_object_type muss vorher gelaufen sein.

### 7. create_button

Erstellt einen Steuerungs-Button (für Surface Forward/Reverse).

```
Request:
  tool: "create_button"
  params:
    name: string              # z.B. "RB01_Forward"
    parent: string            # object_id der Surface
    scale: [number, number, number]   # [0.5, 0.5, 0.5] aus F.EE Demo
    offset_y: number          # +0.5 (Button erscheint 0.5m über Surface)
    on_press_velocity: [number, number, number]
                              # Forward: [1.0, 0, 0]
                              # Reverse: [-1.0, 0, 0]

Response:
  success: boolean
  button_id: string           # WICHTIG: ID für nachfolgende link_objects Calls
  button_name: string
```

**Wichtig:** Response-`button_id` direkt für `link_objects` verwenden.

### 8. create_logic_object

Erstellt ein LogicObject, das Button-Inputs zu Surface-Velocity verbindet.

```
Request:
  tool: "create_logic_object"
  params:
    name: string              # z.B. "RB01_Controller"
    logic_type: string        # "OR_Velocity" (aus F.EE Demo)
    inputs: string[]          # Namen der Inputs [button_forward_id, button_reverse_id]
    output_target: string     # object_id der Surface

Response:
  success: boolean
  logic_object_id: string     # WICHTIG: ID für link_objects
  logic_object_name: string
```

**Offen:** "OR_Velocity" – exakter fe.screen-sim API-String zu bestätigen.

### 9. link_objects

Verbindet zwei Objekte über einen definierten Port.

```
Request:
  tool: "link_objects"
  params:
    from: string              # source object_id oder button_id
    from_port: string         # z.B. "Pressed", "VelocityX", "ButtonForward"
    to: string                # target object_id
    to_port: string           # z.B. "ButtonForward", "ButtonReverse", "InVelocityX"

Response:
  success: boolean
  link_id: string
```

Pro Surface 6 link_objects Calls (aus F.EE Demo):

```
1. { from: <forward_button_id>, from_port: "Pressed",
     to: <logic_id>,           to_port: "ButtonForward" }

2. { from: <reverse_button_id>, from_port: "Pressed",
     to: <logic_id>,            to_port: "ButtonReverse" }

3. { from: <logic_id>, from_port: "VelocityX",
     to: <surface_id>, to_port: "InVelocityX" }

4. { from: <logic_id>, from_port: "VelocityY",
     to: <surface_id>, to_port: "InVelocityY" }

5. { from: <logic_id>, from_port: "VelocityZ",
     to: <surface_id>, to_port: "InVelocityZ" }

6. { from: <logic_id>, from_port: "Active",
     to: <surface_id>, to_port: "Active" }
```

**Offen:** Link 6 (Active→Active) aus Demo angenommen, zu bestätigen.

### 10. documentation_execute

Führt einen Dokumentations-Export aus (optional, für Bericht-Anhang).

```
Request:
  tool: "documentation_execute"
  params:
    output_format: string     # "markdown" | "pdf" | "html"
    output_path: string       # Ziel-Pfad

Response:
  success: boolean
  output_path: string
```

### 11. content_help / object_help

Debugging-Tools. Geben Kontext-Hilfe zu fe.screen-sim Elementen.

```
content_help:
  params: { topic: string }
  response: { content: string }

object_help:
  params: { object_id: string }
  response: { help_text: string, available_types: string[], available_ports: string[] }
```

---

## Ausführungsreihenfolge (kritisch)

```
Phase 0: import_cad_file → get_all_objects
Phase 1: rename_object    (alle Objekte mit Inventor-Tag Objektname)
Phase 2: set_object_type  (alle Objekte mit Inventor-Tag Simulationstyp)
         + set_property   (Sensor: DetectPayload, Motorisiert: JointScale)
Phase 3: configure_motionjoint (alle Objekte mit JointType-Tag)
Phase 4: create_button + create_logic_object + link_objects (alle Surfaces)
```

**Warum diese Reihenfolge:**
- rename vor type: Name-Referenzen in type-Calls müssen bereits korrekt sein
- type vor joint: Joint-Config erfordert Typ "Motorisiert" bereits gesetzt
- type vor logic: Logic-Builder braucht Surface-Typ bestätigt
- create_button vor link_objects: link_objects braucht button_id aus create_button Response

---

## ID-Tracking im MCP Executor (für echte Implementierung)

```javascript
// Wenn MCP Executor real wird (Gruppe B (MCP) liefert Endpunkt):
const id_map = {};

// Phase 4 Beispiel:
const btn_fwd = await mcp.create_button({ name: "RB01_Forward", ... });
id_map["RB01_Forward"] = btn_fwd.button_id;

const logic = await mcp.create_logic_object({ name: "RB01_Controller", ... });
id_map["RB01_Controller"] = logic.logic_object_id;

await mcp.link_objects({
  from: id_map["RB01_Forward"],
  from_port: "Pressed",
  to: id_map["RB01_Controller"],
  to_port: "ButtonForward"
});
```

---

## Offene Fragen (für Gruppe B (MCP) / F.EE)

| # | Frage | Wer klärt | Impact |
|---|-------|-----------|--------|
| 1 | Exakter API-String für `logic_type: "OR_Velocity"`? | Gruppe B (MCP) / F.EE Doku | Logic Builder |
| 2 | JointScale `[0.5,0.5,0.5]` gilt für RevoluteJoint (rad/s) und PrismaticJoint (m/s) gleich? | Gruppe B (MCP) / Zollern | Joint Configurer |
| 3 | Link 6 (Active→Active) — Port-Namen korrekt? | F.EE Demo-Analyse | link_objects |
| 4 | MCP-Server Base-URL + Auth-Methode? | Gruppe B (MCP) | MCP Executor |
| 5 | fe.screen-sim muss offen + Station geladen sein bei MCP-Call? | Gruppe B (MCP) | Setup-Doku |
| 6 | Batch-API möglich (mehrere rename in einem Call)? | F.EE API | Performance |

---

## Beispiel-Durchlauf (Förderer RB01)

```
1. rename_object(obj_010, "RB01")
2. set_object_type(obj_010, "Surface")
3. create_button("RB01_Forward", parent=obj_010, velocity=[1.0,0,0])
   → response: { button_id: "btn_42" }
4. create_button("RB01_Reverse", parent=obj_010, velocity=[-1.0,0,0])
   → response: { button_id: "btn_43" }
5. create_logic_object("RB01_Controller", logic_type="OR_Velocity")
   → response: { logic_object_id: "logic_07" }
6. link_objects(btn_42, "Pressed" → logic_07, "ButtonForward")
7. link_objects(btn_43, "Pressed" → logic_07, "ButtonReverse")
8. link_objects(logic_07, "VelocityX" → obj_010, "InVelocityX")
9. link_objects(logic_07, "VelocityY" → obj_010, "InVelocityY")
10. link_objects(logic_07, "VelocityZ" → obj_010, "InVelocityZ")
11. link_objects(logic_07, "Active"    → obj_010, "Active")
```

---

*Academic feasibility study, HS Albstadt-Sigmaringen. No customer data: all runs use a synthetic test plant.*
