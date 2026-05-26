## Why

The Grafana charges dashboard and charge session detail view currently show charging sessions but do not display the total energy (kWh) charged during each session for vehicles where the battery capacity is unknown or the SoC-based estimation is unreliable. Users must manually inspect raw data or navigate to the edit page to see this value.

## What Changes

- Compute actual charged energy (kWh) in the charge agent by integrating charge power over time during each session
- Store the computed value in the existing `realCharged_kWh` field on `ChargingSession`
- Update the charges dashboard table to display the computed kWh prominently per session
- Update the charge session detail panel to reference the computed value

## Capabilities

### New Capabilities
- `charging-energy-integration`: Compute and persist real charged energy (kWh) by accumulating `chargePower_kW * time_delta` during the charging session, stored in `realCharged_kWh`

### Modified Capabilities

None — no existing specs to modify.

## Impact

- **Model**: `ChargingSession.realCharged_kWh` (already exists, will be auto-populated)
- **Agent**: `agents/charge_agent.py` — add power integration logic that accumulates energy during charging
- **Grafana dashboard**: `charges.json` — ensure Amount column panel shows computed value; `charge_session.json` — ensure detail panel references auto-computed energy
- Database migration: none needed (column already exists)
