# MCP-Schnittstelle: Anforderungen an Gruppe B (MCP)

**Datum:** 2026-05-11 | **Erstellt von:** Rustam (n8n-Agent-Layer)
**Status:** GELIEFERT ✅ (2026-05-11 15:41)

---

## Kontext

n8n Phase 2 Workflow (`workflow_phase2_hochregallager.json`, 895 Zeilen, 46 Nodes) nutzt 6 MCP-Tools zur Automatisierung des Hochregal-Szenarios. Der MCP Executor Node (Zeile 656) ist aktuell Platzhalter.

**Blockiert:** Ohne echte MCP-Endpunkte kann der n8n-Agent-Layer nicht gegen F.EE getestet werden.

---

## Benötigte MCP-Tools (aus Workflow extrahiert)

### 1. `rename_object`
- **Input:** `object_id` (string), `new_name` (string)
- **Output:** `success` (bool), `renamed_object` (object)
- **Workflow-Node:** Claude/GPT/Gemini Agent 2a: Renamer
- **Aufrufe pro Lauf:** ~9 (ein Objekt pro CAD-Komponente)

### 2. `set_object_type`
- **Input:** `object_id` (string), `object_type` (string, Enum)
- **Output:** `success` (bool)
- **Workflow-Node:** Claude/GPT/Gemini Agent 2b: Typer
- **Aufrufe pro Lauf:** ~9

### 3. `configure_motionjoint`
- **Input:** `joint_name` (string), `joint_type` (string: "PrismaticJoint"), `axis` (string: X/Y/Z/G), `range` ([min, max]), `velocity` (float)
- **Output:** `success` (bool), `joint_id` (string)
- **Workflow-Node:** Claude/GPT/Gemini Agent 2c: Joint Configurer
- **Aufrufe pro Lauf:** 4 (X_Axis, Y_Axis, Z_Axis, Gripper)

### 4. `create_button`
- **Input:** `surface_id` (string), `button_name` (string), `direction` (string: "Forward"/"Reverse"), `velocity` (float)
- **Output:** `success` (bool), `button_id` (string)
- **Workflow-Node:** Agent 5: Logic Builder
- **Aufrufe pro Lauf:** ~8 (2 Buttons pro Surface)

### 5. `create_logic_object`
- **Input:** `logic_type` (string), `inputs` (array), `config` (object)
- **Output:** `success` (bool), `logic_id` (string)
- **Workflow-Node:** Agent 5: Logic Builder
- **Aufrufe pro Lauf:** ~4 (FIFO, PositionCompare, RunnerRouter, Sequence)

### 6. `link_objects`
- **Input:** `source_id` (string), `target_id` (string), `link_type` (string)
- **Output:** `success` (bool)
- **Workflow-Node:** Agent 5: Logic Builder
- **Aufrufe pro Lauf:** ~10

---

## Benötigte Infos von euch

| # | Frage | Prio |
|---|-------|------|
| 1 | MCP-Endpunkt-URL (Basis-URL) | Kritisch |
| 2 | Authentifizierungs-Methode (API-Key? OAuth? Header?) | Kritisch |
| 3 | Verfügbare Tool-Namen (falls andere als oben) | Hoch |
| 4 | Request/Response-Format (JSON-RPC? REST?) | Hoch |
| 5 | Tool-Schemas (JSON Schema für jedes Tool) | Hoch |
| 6 | Rate Limits (max calls/sec) | Mittel |
| 7 | Test-Endpunkt vorhanden? (Sandbox) | Mittel |

---

## Nächste Schritte

1. ~~Gruppe B füllt aus, was vorhanden ist~~ ✅ (2026-05-11 15:41)
2. Rustam baut MCP-Client-Node gegen echte Endpunkte ✅ (workflow_phase2_hochregallager.json updated)
3. Integrationstest: n8n → MCP → F.EE → Hochregal-Simulation

**Bei Klärungsbedarf oder Verzögerung:** Rustam eskaliert an Scrum Master oder PO (Product Owner (Kundenseite)).

---

## Gelieferte MCP-Konfiguration (Gruppe B, 2026-05-11T15:41)

```json
{
  "mcpServers": {
    "fe.screen-sim-MCP-server": {
      "url": "http://127.0.0.1:3001/sse",
      "headers": {
        "x-auth-user": "admin",
        "x-auth-password": "admin"
      }
    }
  }
}
```

**Transport:** SSE (Server-Sent Events)  
**Auth:** Custom Headers (x-auth-user, x-auth-password)  
**Protokoll:** MCP JSON-RPC 2.0 (Streamable HTTP Transport)

**Wichtig:** `127.0.0.1` ist Humans lokaler Rechner. Für n8n-Integration braucht es:
- Entweder n8n läuft auf Humans Rechner
- Oder Gruppe B stellt den MCP-Server via Tunnel/Network bereit
- URL in n8n konfigurierbar via Environment Variable `MCP_FESIM_URL`
