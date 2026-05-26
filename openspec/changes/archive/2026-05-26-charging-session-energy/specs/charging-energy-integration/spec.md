## ADDED Requirements

### Requirement: Agent computes energy by integrating charge power over time

The `ChargeAgent` SHALL accumulate charged energy (kWh) during each charging session by multiplying the current charge power (kW) by the time delta since the last power reading, then store the final accumulated value in `ChargingSession.realCharged_kWh` when the session ends.

#### Scenario: Power integration during a charging session

- **WHEN** the `__onChargePowerChange` observer fires with a new power value
- **THEN** the agent SHALL compute `delta_hours = (current_timestamp - last_timestamp) / 3600` and add `last_power_kW * delta_hours` to an in-memory accumulator for that session

#### Scenario: Accumulator reset on new session

- **WHEN** a new `ChargingSession` is created (charging starts or plug connects)
- **THEN** the in-memory accumulator SHALL be reset to zero for that session

#### Scenario: Store energy on session end

- **WHEN** charging ends (state transitions from CHARGING to OFF/READY_FOR_CHARGING/ERROR) or plug disconnects
- **THEN** the agent SHALL write the accumulated `_accumulated_energy_kWh` to `chargingSession.realCharged_kWh`

#### Scenario: Preserve manually entered values

- **WHEN** `chargingSession.realCharged_kWh` already has a non-null value at session end
- **THEN** the agent SHALL NOT overwrite it with the computed value

#### Scenario: Session resumption carries forward partial accumulator

- **WHEN** an interrupted session is resumed within the configured interruption window (24h for normal, 300h for conservation)
- **THEN** the accumulator SHALL continue from its previous value, and the session end fields (ended, endSOC_pct) SHALL be cleared to allow continued accumulation
