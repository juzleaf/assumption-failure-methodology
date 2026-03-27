# AFM Falsify Report: OpenBSE v31

**Target**: OpenBSE — Open Building Simulation Engine (Rust)
**Location**: `/tmp/afm-benchmark-round3/openbse-v31/`
**Method**: AFM Falsify (blind, no hints)
**Date**: 2026-03-27

---

## Phase 1: Seed Map

Codebase sweep across 8 crates (~25 source files). Seeds extracted from structural, physical, and logical anomalies.

| # | Seed | File:Line | Category |
|---|------|-----------|----------|
| S01 | Convergence config exists but simulate_timestep runs graph once — no iteration | simulation.rs:262-333 | Logic |
| S02 | cp_water inconsistency: 4180.0 (psychrometrics) vs 4186.0 (chiller, water_heater) | psychrometrics/lib.rs:408, chiller.rs:190, water_heater.rs:24 | Physics |
| S03 | Hardcoded non-leap year (365 days, Feb=28) in 6+ locations | simulation.rs:122,184; types.rs:33; schedule.rs:441; heat_balance.rs | Design |
| S04 | AUTOSIZE sentinel -99999.0 with tolerance 1.0 — any value in [-100000,-99998] triggers | types.rs:57-61 | Robustness |
| S05 | month=0 causes u32 underflow panic in day_of_year (self.month - 1) | types.rs:35 | Crash |
| S06 | Graph cycle-breaking comment says "most incoming edges" but code picks min_by_key (fewest) | graph.rs:129 vs 155 | Doc/Logic |
| S07 | Duplicate component name silently overwrites in graph | graph.rs:72-74 | Data integrity |
| S08 | ControlSignals.sat_setpoint defaults to 0.0 — valid temperature, not an "unset" sentinel | simulation.rs:86 | Default |
| S09 | Chiller forces min mass_flow 0.001 kg/s even at zero load | chiller.rs:242 | Physics |
| S10 | Chiller uses hardcoded cp_water=4186.0 instead of inlet.state.cp (4180.0) | chiller.rs:190 | Consistency |
| S11 | Heat recovery bypass check ignores humidity — misses beneficial latent recovery | heat_recovery.rs:134 | Physics |
| S12 | VAV box set_setpoint used as control_signal setter — semantic mismatch | vav_box.rs:302-305 | API design |
| S13 | EPW parser silently skips malformed data lines (Err => continue) | weather/lib.rs:275 | Silent failure |
| S14 | Controls state zone_temp() returns hardcoded 21.0 for unknown zones | controls/state.rs:67 | Silent fallback |
| S15 | Boiler LeavingSetpointModulated ignores inlet mass_flow — computes its own | boiler.rs:184-220 | Design |
| S16 | Fan heat formula: ShaftPower = MotorEff * Power | fan.rs:151 | Physics (verify) |

---

## Phase 2: Interrogate + Falsify

### S01 — Convergence iteration configured but never executed

**INTERROGATE**

- **Premise**: SimulationConfig stores `max_air_loop_iterations: 20`, `max_plant_loop_iterations: 10`, and `convergence_tolerance: 0.01`. The assumption is that these parameters are used by the simulation loop.
- **Mechanism**: `simulate_timestep()` (line 262-333) walks the component graph exactly once via `for &node_idx in &order`. No outer loop, no convergence check, no comparison of inlet/outlet states between iterations.
- **Expected Effect**: For systems with feedback (e.g., return air mixing with outdoor air, plant loops where boiler outlet feeds coils that determine load), a single-pass solution will use stale values from the previous timestep. Temperature errors accumulate.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

Consider a simple AHU with outdoor air + return air mixing → cooling coil → supply fan → zone → return duct → mixer. In EnergyPlus, this requires 2-20 iterations per timestep because:
1. The mixer output depends on return air temperature
2. Return air temperature depends on zone temperature
3. Zone temperature depends on supply air conditions
4. Supply air depends on the mixer output

With ONE pass: the mixer uses the PREVIOUS timestep's return air state. On a step change (e.g., morning startup, occupancy jump, setpoint change), the coil sees incorrect mixed air temperature, delivers wrong capacity, zone temperature diverges from expected, and the error persists for multiple timesteps until it "catches up" through time-lag averaging.

For plant loops: a chiller's leaving water temperature affects the coil's capacity, which affects the load request back to the chiller — a classic feedback loop that EnergyPlus iterates to convergence.

The config parameters (`max_air_loop_iterations`, `convergence_tolerance`) are dead code — they exist in the struct but nothing reads them during simulation.

**Final verdict**: **PROVEN**

The code configures convergence iteration parameters that are never used. The `simulate_timestep` function provably runs each component exactly once. Any system with feedback paths (mixing, plant loops) will produce first-order lag errors that EnergyPlus handles through iteration.

**Chain**: Leads to S15 (boiler computes own flow independently of network).

---

### S02 — cp_water inconsistency (4180 vs 4186 J/kg-K)

**INTERROGATE**

- **Premise**: The specific heat of water should be consistent throughout the simulation for energy conservation.
- **Mechanism**: `openbse_psychrometrics::CP_WATER = 4180.0` (lib.rs:408) is used by `FluidState::water()` to initialize plant loop states. But `chiller.rs:190` and `water_heater.rs:24` use hardcoded `4186.0`.
- **Expected Effect**: When a chiller computes `Q = m * 4186 * dT` and the downstream coil uses `FluidState.cp = 4180`, the energy exchanged across the interface is inconsistent. Over a timestep: `dQ_error = m * 6.0 * dT`.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

For a chiller serving a cooling coil:
- Flow rate: 5 kg/s (typical small commercial)
- Delta-T: 5°C (typical chiller)
- Chiller thinks it delivers: 5 * 4186 * 5 = 104,650 W
- Coil receives (using cp=4180): 5 * 4180 * 5 = 104,500 W
- Discrepancy: 150 W per timestep

150W is 0.14% error — small but systematic. Over a year-long simulation with 8760 hours, this is a persistent bias. More critically, it violates the fundamental principle that energy entering one component equals energy leaving the previous one. In ASHRAE Standard 140 validation, such systematic biases can push results outside acceptable tolerance bands.

However: the PRACTICAL impact is small (0.14%) and unlikely to cause simulation failures. The issue is real but not catastrophic.

**Final verdict**: **SUSPECTED**

The inconsistency is provably present in code. The energy conservation violation is real but quantitatively small (~0.14%). It's a code quality / maintenance issue rather than a correctness-breaking bug.

**Chain**: Check all hardcoded cp values across the codebase → found in chiller.rs:190, water_heater.rs:24. The canonical constant should be used everywhere.

---

### S03 — Hardcoded 365-day non-leap year

**INTERROGATE**

- **Premise**: Building simulations typically use TMY (Typical Meteorological Year) data which is always 365 days. Leap years are not typically modeled.
- **Mechanism**: `days_in_months = [31, 28, ...]` appears in simulation.rs (lines 122, 184), types.rs:33, schedule.rs:441, and several other locations. The schedule module explicitly states "Non-leap year assumed (365 days)".
- **Expected Effect**: If someone feeds 8784-hour (leap year) weather data, the simulation would either access out-of-bounds or skip Feb 29 data.

**Initial verdict**: HOLDS — this is a deliberate, documented design decision

**FALSIFY — Challenge**: Could this cause problems even within its own documented constraint?

The weather array is indexed by `hour_idx` which is bounded by `end_hour.min(weather_hours.len() as u32)` (simulation.rs:135). If weather data is exactly 8760 entries and the simulation requests Jan 1 to Dec 31, it works. If weather data is from a leap year file with 8784 entries, the extra 24 hours (Feb 29) would be silently skipped — the day mapping would be wrong for all dates after February.

However, this matches EnergyPlus convention (TMY = 365 days always). Standard weather files (EPW, TMY3) are always 8760 hours. The codebase is self-consistent.

**Final verdict**: **HARDENED**

Deliberate design choice matching industry convention. The 6 locations are all consistent with each other. Risk only exists if someone manually constructs a leap-year weather file, which is outside the intended use case.

---

### S04 — AUTOSIZE sentinel tolerance collision

**INTERROGATE**

- **Premise**: The sentinel value -99999.0 with tolerance 1.0 means `is_autosize(v)` returns true for any v in [-100000.0, -99998.0].
- **Mechanism**: `is_autosize(val: f64) -> bool { (val - AUTOSIZE).abs() < 1.0 }` at types.rs:60-61. Used in 15+ locations to decide whether a parameter should be auto-sized.
- **Expected Effect**: If any engineering parameter happens to fall in [-100000, -99998], it would be falsely treated as "autosize me".

**Initial verdict**: HOLDS (initially)

**FALSIFY — Challenge**: Can any real parameter be in this range?

Building simulation parameters:
- Capacities: typically 1,000 to 10,000,000 W (positive)
- Flow rates: 0.001 to 100 m3/s (positive)
- Temperatures: -50 to 100°C (positive or small negative)
- Pressures: 50 to 300,000 Pa (positive)

No physical building parameter is near -99999. The tolerance of 1.0 exists to handle floating-point serialization/deserialization roundoff. EnergyPlus uses the same convention.

However, the design is fragile: there is NO type-level guarantee. A future developer could use -99999 in a different context (e.g., as an error code, or in a formula intermediate). The tolerance-based approach is less safe than Rust's `Option<f64>` pattern.

**Final verdict**: **HARDENED**

No practical collision with physical parameters. The sentinel pattern, while not idiomatic Rust, matches EnergyPlus convention and functions correctly within the building simulation domain. The `AutosizeValue` enum (types.rs:72-105) provides a safer wrapper for new code.

---

### S05 — month=0 underflow panic in day_of_year

**INTERROGATE**

- **Premise**: `TimeStep.month` is u32. If month is 0, the expression `(self.month - 1) as usize` underflows (u32 subtraction wrapping to 4294967295).
- **Mechanism**: `day_of_year()` at types.rs:35 uses `self.month - 1` to index into `days_in_months[..(self.month - 1) as usize]`. With month=0, this becomes `days_in_months[..4294967295]` which panics on out-of-bounds slice.
- **Expected Effect**: Runtime panic if TimeStep is constructed with month=0.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

```rust
let ts = TimeStep { month: 0, day: 1, hour: 1, sub_hour: 1,
                    timesteps_per_hour: 1, sim_time_s: 0.0, dt: 3600.0 };
ts.day_of_year(); // PANIC: u32 underflow then out-of-bounds
```

However: month is set internally by `SimulationRunner` via `month_day_from_hour()` which derives month from the `days_in_months` array — it will always produce 1-12. External callers could construct an invalid TimeStep, but the struct is primarily used internally.

Check for month=13: `days_in_months[..12]` = entire array, works but returns wrong value. month=14: `days_in_months[..13]` = out of bounds, panic.

**Final verdict**: **SUSPECTED**

The panic is real and trivially reproducible with `month=0` or `month > 13`. In practice, internal simulation code always produces valid months 1-12. The vulnerability is in the public API — any caller constructing TimeStep manually could trigger it. A defensive `assert!(self.month >= 1 && self.month <= 12)` or using 0-based indexing with checked operations would fix it.

---

### S06 — Graph cycle-breaking: comment/code mismatch

**INTERROGATE**

- **Premise**: Comment at graph.rs:129 says "component with the most incoming edges" but code at line 155 uses `min_by_key(|(_, &d)| d)` which selects the FEWEST remaining in-degree.
- **Mechanism**: In Kahn's algorithm with cycle-breaking, you pick an unvisited node to force into the queue when all remaining nodes have non-zero in-degree. The choice of WHICH node affects simulation order.
- **Expected Effect**: Either the comment is wrong (benign) or the code is wrong (affects simulation order for cyclic plant loops).

**Initial verdict**: UNCLEAR

**FALSIFY — Which is correct?**

Standard Kahn's algorithm breaks cycles by removing an arbitrary node. Common heuristics:
- **Fewest in-degree** (code behavior): picks the node closest to being "ready" — this is the standard optimization because it minimizes the number of edges that need to be conceptually "broken"
- **Most in-degree** (comment): would pick the most-connected node, which is a less common choice

The CODE (min_by_key = fewest remaining in-degree) follows standard graph algorithm practice. The COMMENT is wrong.

Impact: the comment misleads developers but the algorithm is correct. For plant loops with cycles, the fewest-in-degree heuristic produces a reasonable simulation order.

**Final verdict**: **HARDENED**

The code implements the standard and correct heuristic. The comment is factually wrong (says "most" when code does "fewest") but this is a documentation bug, not a logic bug. The simulation order produced is correct for the intended use case.

**Chain**: None — contained issue.

---

### S07 — Duplicate component name silently overwrites

**INTERROGATE**

- **Premise**: `add_air_component` at graph.rs:72-74 inserts the component name into `self.name_map`. HashMap::insert silently replaces the old value if the key already exists.
- **Mechanism**: If two components have the same name, the second registration overwrites the first's node index in the name_map. The first component still exists in the graph but becomes unreachable by name.
- **Expected Effect**: Control signals addressed by component name would only reach the second component. The first would receive no setpoint overrides.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

```rust
graph.add_air_component(Box::new(coil_1)); // name = "CoolingCoil"
graph.add_air_component(Box::new(coil_2)); // name = "CoolingCoil" (duplicate)
// name_map["CoolingCoil"] now points to coil_2
// coil_1 exists in graph but cannot be looked up by name
// signals.coil_setpoints["CoolingCoil"] = 13.0 → only reaches coil_2
```

In practice: component names come from user input (JSON model files). The input validation module (`openbse-io`) should catch duplicates. Let me check.

The `validate_model` function in `openbse-io/src/input.rs` would need to check for duplicate names across all component types.

However, even if input validation exists, the graph layer itself has no guard — it's a defense-in-depth failure. If validation is bypassed (programmatic API, testing), duplicates corrupt the simulation silently.

**Final verdict**: **SUSPECTED**

The silent overwrite is real. Impact depends on whether input validation catches duplicates upstream. The graph API itself provides no protection, violating defense-in-depth principles.

---

### S08 — ControlSignals.sat_setpoint defaults to 0.0

**INTERROGATE**

- **Premise**: `sat_setpoint` (Supply Air Temperature setpoint) defaults to 0.0 in the `ControlSignals` struct. 0.0°C is a valid temperature — it's freezing point.
- **Mechanism**: The default is set in the struct definition (simulation.rs:86). If a controller doesn't explicitly set `sat_setpoint`, components reading it will see 0.0°C as the target.
- **Expected Effect**: If heat recovery or other components use `sat_setpoint` for capacity limiting, a 0°C default would cause extreme overcooling.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

The field comment says "used for HR credit cap" — heat recovery credit capping. If a heat recovery unit reads `signals.sat_setpoint = 0.0`, it would limit its recovery to achieve a 0°C supply air temperature, which is absurd in heating mode.

Let me check how sat_setpoint is actually consumed.

From the grep results, `sat_setpoint` appears in simulation.rs:86 (definition) and in heat_recovery.rs context. If heat recovery uses this as: "don't recover more heat than needed to reach SAT setpoint", then 0°C default means in heating season, the HR would try to cool supply air to 0°C — defeating its purpose.

However: looking at the `run()` and `run_with_controls()` methods, `run()` passes `&ControlSignals::default()` which means the 0.0 default is the ACTIVE value when no controls framework is used. The question is: do any components actually read `signals.sat_setpoint`?

From the `simulate_timestep` code (lines 262-333), control signals are applied via `signals.coil_setpoints`, `signals.air_mass_flows`, and `signals.plant_loads`. The `sat_setpoint` field is NOT directly consumed in `simulate_timestep` — it's available on the struct but there's no code path that reads it during component simulation.

**Revised finding**: `sat_setpoint` is defined but appears unused in the simulation loop. It's dead code on the struct.

**Final verdict**: **UNCLEAR**

The 0.0 default is problematic IF the field is consumed, but current evidence suggests it may be unused dead code. If future development reads this field without setting it, the 0°C default would cause incorrect behavior. This is a latent hazard rather than a current bug.

---

### S09 — Chiller forces minimum mass_flow 0.001 kg/s at zero load

**INTERROGATE**

- **Premise**: At chiller.rs:242, `mass_flow = inlet.state.mass_flow.max(0.001)` forces a minimum flow of 0.001 kg/s even when the system should be off.
- **Mechanism**: When the chiller has zero load, it should produce zero flow. Forcing 0.001 kg/s means cold water circulates through the plant loop even when no cooling is requested.
- **Expected Effect**: Parasitic cooling in plant loops during periods when the chiller should be off. Small but systematic energy accounting error.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

Scenario: Winter operation, chiller off, zero cooling load.
- The chiller still pushes 0.001 kg/s of chilled water through the loop
- If chiller leaving water temp is, say, 7°C and return water is 20°C:
  - Parasitic cooling: 0.001 * 4180 * 13 = 54 W
  - Over 4000 heating-season hours: 54 * 4000 = 216 kWh/yr

This is small but nonzero. More importantly, it prevents the plant loop from truly "off" state, which could confuse controls logic that checks for zero flow as an indication of equipment off-state.

However: this is a common numerical guard to prevent division-by-zero in downstream calculations. If `mass_flow = 0`, any `Q / (m * cp * dT)` calculation would produce infinity. The 0.001 kg/s is a numerical stabilizer, not a physics choice.

**Final verdict**: **SUSPECTED**

The minimum flow guard exists for numerical stability (preventing div-by-zero). The physics impact is small (~54W parasitic). Better approaches exist (check for zero flow and short-circuit the calculation), but the current approach is a common pattern in building simulation code. It's a code quality issue, not a correctness bug for practical simulations.

---

### S10 — Chiller hardcoded cp_water vs FluidState.cp

**INTERROGATE**

This is a specific instance of S02. The chiller uses `4186.0` at line 190 while the FluidState infrastructure provides `4180.0`. Same analysis as S02 — 0.14% systematic bias.

**Final verdict**: **SUSPECTED** (same as S02)

---

### S11 — Heat recovery bypass ignores latent benefit

**INTERROGATE**

- **Premise**: Heat recovery wheels can recover both sensible and latent energy. The bypass decision should consider total energy savings, not just temperature.
- **Mechanism**: At heat_recovery.rs:134, bypass is triggered when the temperature difference is within a 1°C deadband. For enthalpy wheels in hot-humid climates, the outdoor air might be at the same temperature as return air but with much higher humidity — latent recovery is still beneficial.
- **Expected Effect**: In subtropical/tropical climates, the heat recovery would bypass when it should be operating, losing significant dehumidification benefit.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

Miami summer conditions:
- Outdoor: 32°C, 80% RH → h ≈ 86 kJ/kg
- Return: 24°C, 50% RH → h ≈ 48 kJ/kg
- Temperature difference: 8°C (bypass NOT triggered)

But consider a shoulder-season scenario:
- Outdoor: 25°C, 90% RH → h ≈ 71 kJ/kg, w ≈ 0.018 kg/kg
- Return: 24°C, 50% RH → h ≈ 48 kJ/kg, w ≈ 0.009 kg/kg
- Temperature difference: 1°C → BYPASS TRIGGERED
- Enthalpy difference: 23 kJ/kg → significant latent recovery benefit lost

At 1 kg/s airflow with 70% effectiveness:
- Latent recovery lost: 1.0 * 0.70 * 23000 = 16,100 W of dehumidification
- This is a 5-ton cooling load that the cooling coil must now handle

This is a significant energy impact for humid climates with enthalpy wheels.

However: the code may model sensible-only heat recovery (no enthalpy wheel). Let me check.

The heat_recovery component would need to distinguish between sensible-only (plate/pipe) and total-energy (wheel/membrane) types. If it only models sensible recovery, the temperature-based bypass is correct. If it models enthalpy wheels, the bypass logic is incomplete.

**Final verdict**: **SUSPECTED**

The bypass logic is correct for sensible-only heat recovery but incomplete for enthalpy wheels. The concrete scenario shows 16 kW of lost recovery in humid climates. Whether this is a bug depends on whether the component is intended to model total-energy recovery — the 1°C temperature deadband suggests sensible-only design intent, but this should be explicit.

---

### S12 — VAV box set_setpoint semantic overload

**INTERROGATE**

- **Premise**: The `set_setpoint` method on the AirComponent trait is semantically "set the temperature setpoint". In vav_box.rs:302-305, it's repurposed to set the zone thermostat control signal.
- **Mechanism**: The VAV box receives a setpoint via `set_setpoint()` which it interprets as a damper position or flow fraction rather than a temperature.
- **Expected Effect**: If the controls framework sends a temperature (e.g., 22°C) thinking it's setting a temperature setpoint, the VAV box interprets it as a flow fraction — resulting in maximum or near-maximum flow.

**Initial verdict**: VULNERABLE

**FALSIFY — Challenge**:

Looking at the code flow: `signals.coil_setpoints` is a HashMap of component name to f64. In simulate_timestep (line 284-286):
```rust
if let Some(&sp) = signals.coil_setpoints.get(&comp_name) {
    component.set_setpoint(sp);
}
```

This sends the same signal type to ALL air components — coils get temperature setpoints, VAV boxes get... what? The caller must know that for a VAV box, the "setpoint" is actually a control signal (0-1 damper position or similar).

This is a type-safety violation — the same f64 parameter carries different semantic meaning depending on the component type. A coil expects °C, a VAV box expects a fraction.

However: the controls framework IS the caller, and it presumably knows the component types. This is "works by convention" rather than "works by contract."

**Final verdict**: **SUSPECTED**

The semantic overload is real — the trait interface doesn't distinguish between temperature setpoints and control signals. It works only because callers maintain the convention. A refactor to typed enums or separate methods would eliminate the class of errors.

---

### S13 — EPW parser silently skips malformed data

**INTERROGATE**

- **Premise**: Weather data is a critical simulation input. Silently dropping data points creates gaps that propagate through the entire simulation.
- **Mechanism**: At weather/lib.rs:275, `Err(_) => continue` means any malformed line in the EPW file is silently skipped. The resulting weather array has fewer than 8760 entries.
- **Expected Effect**: If the weather array has, say, 8755 entries, the simulation will run 5 fewer hours than expected. The mapping from `hour_idx` to calendar date becomes incorrect for all subsequent hours.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

If EPW line 2000 (≈ March 24, hour 8) is malformed:
- Weather array has 8759 entries instead of 8760
- `hour_idx = 2000` now reads the data from original hour 2001
- All hours after March 24 08:00 are shifted by one hour
- The simulation maps `hour_idx` to (month, day) using `month_day_from_hour()` which counts from the array start
- The last hour of Dec 31 (hour_idx=8759) now accesses `weather_hours[8759]` which is out of bounds... wait, `end_hour.min(weather_hours.len() as u32)` catches this — simulation stops one hour early

So: missing one hour means the simulation ends one hour before Dec 31 23:00. Missing five hours means losing five hours of December data. The calendar mapping is still correct (it's based on hour_idx not array index), but the weather data is shifted.

Actually, re-reading the code: `hour_idx` IS the array index at line 136: `weather_hours[hour_idx as usize]`. So hour_idx=2000 reads weather_hours[2000], which after a skip at position 2000, contains the data from original file line 2001. The simulation THINKS it's March 24 hour 8 but uses March 24 hour 9 data. Every subsequent hour is off by one.

**Final verdict**: **SUSPECTED**

Silent data loss is real. For a single malformed line, the impact is a one-hour shift in weather data alignment for the rest of the year. For multiple malformed lines, the shift accumulates. The simulation runs without error but with systematically wrong weather mapping. A warning or error on parse failure would be more appropriate than silent skip.

---

### S14 — Controls state returns hardcoded 21.0 for unknown zones

**INTERROGATE**

- **Premise**: `zone_temp()` at controls/state.rs:67 returns 21.0°C if the zone name is not found. 21°C is roughly room temperature.
- **Mechanism**: `self.zone_temps.get(zone).copied().unwrap_or(21.0)` — any typo in zone name, or query before the zone is simulated, returns a plausible temperature.
- **Expected Effect**: A controller querying a misspelled zone name gets 21°C, concludes the zone is at setpoint, and takes no action. The actual zone could be at 35°C or 5°C.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

```rust
// Controller queries zone temperature
let t = state.zone_temp("Conference_Room"); // typo — actual name is "ConferenceRoom"
// Returns 21.0 — controller thinks zone is comfortable
// Actual zone temp might be 30°C in summer
// HVAC stays off, zone overheats
```

The failure is silent — no warning, no error. The controller operates on fabricated data. This violates the "fail-fast" principle: an unknown zone should be an error, not a plausible default.

However: this only matters if:
1. A zone name mismatch exists between the model and controls configuration
2. The first timestep before envelope simulation has run (bootstrap problem)

For case 2: returning 21°C as an initial guess during the first timestep is actually reasonable — it's a typical indoor temperature. The problem is case 1, where it masks configuration errors permanently.

**Final verdict**: **SUSPECTED**

The silent fallback is a real hazard for configuration errors. A better design would be `zone_temp() -> Option<f64>` or `zone_temp_or_default()` with explicit naming. The current implementation masks bugs that would otherwise surface immediately.

---

### S15 — Boiler LeavingSetpointModulated ignores inlet mass_flow

**INTERROGATE**

- **Premise**: In a plant loop, mass flow is determined by the pump and distribution network. The boiler should heat whatever water flows through it, not compute its own flow.
- **Mechanism**: At boiler.rs:184-220, the LeavingSetpointModulated mode computes `m_required = boiler_load / (cp * dt_set)` independently of `inlet.state.mass_flow`. It creates its own flow rate.
- **Expected Effect**: The boiler's outlet flow differs from its inlet flow, violating mass conservation in the plant loop.

**Initial verdict**: VULNERABLE

**FALSIFY — Concrete failure construction**:

Plant loop scenario:
- Pump delivers 2.0 kg/s
- Boiler receives inlet at 60°C, setpoint 80°C, load demand 50 kW
- Boiler computes: m_required = 50000 / (4180 * 20) = 0.598 kg/s
- Boiler outputs 0.598 kg/s at 80°C
- But the pump sent 2.0 kg/s — where did the other 1.4 kg/s go?

The outlet WaterPort has the boiler's computed mass_flow (0.598), not the inlet's (2.0). Downstream components see a different flow rate than what the pump provides. This violates mass conservation.

However: EnergyPlus's boiler model in "LeavingSetpointModulated" mode ALSO computes its own flow — it requests flow from the plant manager and the plant solver reconciles. The difference is that E+ has a plant solver that iterates to convergence (see S01). Without iteration, OpenBSE's boiler just outputs whatever flow it wants.

This is directly connected to S01: the single-pass architecture means the boiler's flow decision is never reconciled with the pump's flow decision. In EnergyPlus, the plant loop solver would iterate until flow and temperatures converge.

**Final verdict**: **PROVEN**

The boiler outputs a mass flow rate that differs from its inlet, violating mass conservation. This is not caught because the simulation doesn't iterate (S01). In EnergyPlus, the plant solver reconciles these flow mismatches through iteration. The combination of S01 + S15 means plant loop simulations will have systematic energy imbalances.

**Chain**: Directly caused by S01 (no convergence iteration).

---

### S16 — Fan heat formula verification

**INTERROGATE**

- **Premise**: EnergyPlus defines: ShaftPower = MotorEff * TotalPower.
- **Mechanism**: fan.rs:151: `shaft_power = self.motor_efficiency * self.power`. Then `heat_to_air = shaft_power + (self.power - shaft_power) * self.motor_in_airstream_fraction`.
- **Expected Effect**: All shaft power becomes airstream heat (friction/turbulence). Motor waste heat only enters airstream if motor is in the duct.

**FALSIFY**:

The formula decomposition:
- Shaft power = work delivered to impeller = MotorEff * ElecPower (correct — motor efficiency converts electrical to mechanical)
- Motor waste = (1 - MotorEff) * ElecPower = Power - ShaftPower
- Heat to air = ShaftPower (all becomes heat via friction) + MotorWaste * MotorInAirFrac

This matches EnergyPlus Engineering Reference exactly. The code is correct.

**Final verdict**: **HARDENED**

The formula matches EnergyPlus exactly. The comment and code are consistent. No error found.

---

## Proof Catalog

Issues with concrete, reproducible failure paths.

| ID | Finding | Verdict | Impact |
|----|---------|---------|--------|
| S01 | **Convergence iteration dead code** — max_air_loop_iterations and convergence_tolerance are configured but never used. simulate_timestep runs graph exactly once. | PROVEN | Systems with feedback (mixing, plant loops) produce first-order lag errors. Plant loop energy balances are incorrect. |
| S15 | **Boiler mass flow conservation violation** — LeavingSetpointModulated mode outputs self-computed flow, ignoring inlet flow. Combined with S01, flow mismatches are never resolved. | PROVEN | Plant loop mass conservation violated. Downstream components see wrong flow rate. Energy accounting incorrect. |

## Suspicion List

Issues with strong evidence but some mitigating factors.

| ID | Finding | Verdict | Impact |
|----|---------|---------|--------|
| S02 | cp_water inconsistency (4180 vs 4186) | SUSPECTED | 0.14% systematic energy bias across water/plant interfaces |
| S05 | month=0 u32 underflow panic in day_of_year | SUSPECTED | Runtime panic on invalid input; internal code produces valid months |
| S07 | Duplicate component name silently overwrites graph entry | SUSPECTED | First component becomes unreachable by name; depends on input validation |
| S09 | Chiller forces 0.001 kg/s minimum flow at zero load | SUSPECTED | Parasitic cooling ~54W in off-season; numerical stability guard |
| S10 | Chiller hardcoded cp vs FluidState.cp | SUSPECTED | Same as S02 — 0.14% bias |
| S11 | Heat recovery bypass ignores latent benefit | SUSPECTED | Up to 16 kW lost dehumidification in humid climates |
| S12 | VAV box set_setpoint semantic overload | SUSPECTED | Type-safety gap — works by convention not contract |
| S13 | EPW parser silently skips malformed lines | SUSPECTED | Weather data shifts by N hours for N malformed lines |
| S14 | Controls state returns 21.0 for unknown zones | SUSPECTED | Masks configuration errors with plausible default |

## Defense Map

Issues that were investigated and found to be correctly implemented or deliberately designed.

| ID | Finding | Verdict | Reason |
|----|---------|---------|--------|
| S03 | Hardcoded 365-day year | HARDENED | Matches TMY convention; all 6 locations consistent |
| S04 | AUTOSIZE sentinel tolerance | HARDENED | No physical parameter near -99999; AutosizeValue enum provides safer API |
| S06 | Graph cycle-breaking comment/code mismatch | HARDENED | Code is correct (fewest in-degree); comment is wrong but harmless |
| S16 | Fan heat formula | HARDENED | Matches EnergyPlus exactly |

## Unclear

| ID | Finding | Verdict | Reason |
|----|---------|---------|--------|
| S08 | ControlSignals.sat_setpoint defaults to 0.0°C | UNCLEAR | Field appears unused in current simulation loop — latent hazard |

## Root Causes

1. **Missing iteration solver** (root of S01, S15): The simulation architecture runs components in a single topological pass without iteration. EnergyPlus uses iterative solvers for both air and plant loops. This is the single most impactful architectural gap — it affects every system with feedback paths (which is virtually every real building model).

2. **Inconsistent constants** (root of S02, S10): Hardcoded physical constants in individual component files instead of referencing the canonical crate constant. A simple code hygiene issue with a clear fix: replace all `4186.0` with `openbse_psychrometrics::CP_WATER`.

3. **Weak API contracts** (root of S07, S08, S12, S14): Public APIs use f64 where typed enums or Option would enforce correctness. HashMap silently overwrites, defaults mask errors, trait methods carry ambiguous semantics. This is a pattern of "works in happy path" but provides no guardrails for edge cases or misconfiguration.

4. **Silent failure preference** (root of S13, S14): The codebase prefers to continue with defaults/fallbacks rather than fail on bad data. This makes debugging extremely difficult — simulation completes without error but produces wrong results.

## Seed Map

```
S01 (no iteration) ──→ S15 (boiler flow mismatch) ──→ plant loop energy errors
     │
     └──→ S09 (chiller min flow) ──→ parasitic loads in off-state

S02 (cp_water) ──→ S10 (chiller cp) ──→ water-side energy imbalance

S07 (duplicate names) ┐
S12 (semantic overload)├──→ Weak API contracts root cause
S14 (silent defaults)  │
S08 (0.0 default)     ┘

S13 (EPW skip) ──→ weather/simulation alignment error

S03 (365 days) ──→ HARDENED (deliberate)
S04 (AUTOSIZE) ──→ HARDENED (convention)
S06 (comment)  ──→ HARDENED (code correct)
S16 (fan heat) ──→ HARDENED (matches E+)
```

## Unexplored Sections

The following areas were not fully interrogated due to file size constraints:

1. **heat_balance.rs** (~265KB) — Envelope heat balance with solar calculations, CTF solver, zone coupling. Only partially read. May contain:
   - CTF coefficient computation issues (Seem 1987 method)
   - Solar distribution errors
   - Zone air balance convergence issues

2. **solar.rs** — Solar geometry and radiation calculations. Not fully read. May contain:
   - Clearness index / sky model issues
   - Window transmission calculation errors
   - Shading algorithm edge cases

3. **ctf.rs** — Conduction Transfer Function coefficient generator. Not fully read. May contain:
   - Numerical stability issues for thin/high-conductivity layers
   - Time step coupling constraints
   - Matrix conditioning problems

4. **openbse-io/src/input.rs** — Input validation. Partially read. May contain:
   - Missing validation for duplicate component names (affects S07)
   - Incomplete range checking for physical parameters

5. **openbse-cli/src/main.rs** — CLI and autosizing logic. Partially read. The autosizing logic (>1000 lines) may contain:
   - Sizing calculation errors
   - Missing safety factors
   - Component interdependency during sizing

---

*Report generated by AFM Falsify (blind test, no hints). 16 seeds extracted, 2 PROVEN, 9 SUSPECTED, 4 HARDENED, 1 UNCLEAR.*
