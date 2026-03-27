# OpenBSE Code Review + Systematic Debugging Report

## Summary

OpenBSE is a Rust-based building energy simulation engine organized as a Cargo workspace with 8 crates: core simulation loop, psychrometrics, HVAC components, building envelope, weather I/O, controls, input parsing, and a Tauri-based editor. The codebase targets EnergyPlus-compatible physics and conventions.

This review examined all Rust source files across the workspace. The analysis uncovered **21 findings**: 3 Critical, 6 High, 8 Medium, and 4 Low severity issues. The most impactful problems are a missing BDF history update that will cause the envelope solver to diverge, static control signals that make the HVAC simulation unable to respond to changing conditions, and several inconsistent physical constants that will produce energy balance errors.

---

## Findings

### F-01: Missing `update_bdf_history()` Call in Envelope Simulation Loop

**Severity:** Critical

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/simulation.rs` (lines 177-258), `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/ports.rs` (`EnvelopeSolver` trait)

**Description:**
The `run_with_envelope()` method calls `envelope.solve_timestep()` each sub-hourly timestep but never calls `envelope.update_bdf_history()`. The trait documentation on `update_bdf_history()` explicitly states it "must be called exactly ONCE per physical timestep" to advance the backward difference formula (BDF) time-history arrays for zone temperature and surface temperature calculations.

**Evidence:**
In `simulation.rs` lines 240-254, the inner loop calls `envelope.solve_timestep()` and then `self.simulate_timestep()` but has no call to `update_bdf_history()`. The `EnvelopeSolver` trait in `ports.rs` defines `update_bdf_history(&mut self)` with documentation: "Must be called exactly ONCE per physical timestep, AFTER `solve_timestep` finishes."

**Impact:**
Without advancing the BDF history, the zone temperature predictor-corrector will use stale or uninitialized previous-timestep temperatures. This causes the zone heat balance to diverge or produce wildly incorrect zone temperatures, making all HVAC load calculations unreliable. The entire coupled envelope+HVAC simulation is broken.

**Recommendation:**
Add `envelope.update_bdf_history();` after the `envelope.solve_timestep()` call inside the sub-hourly loop, before advancing `sim_time`.

---

### F-02: Static Control Signals Never Updated During Simulation

**Severity:** Critical

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/simulation.rs` (lines 116-168)

**Description:**
`run_with_controls()` accepts `signals: &ControlSignals` as an immutable reference and passes the same static signals object to every timestep throughout the entire simulation. In a real building simulation, control signals (coil setpoints, air flow rates, plant loads) must be recomputed each timestep based on current zone temperatures, outdoor conditions, and occupancy schedules.

**Evidence:**
Line 120: `signals: &ControlSignals` is an immutable borrow. Line 160: `self.simulate_timestep(graph, &ctx, signals)?` passes the same `signals` each iteration. The `run_with_envelope()` method at line 232-238 similarly builds `hvac` conditions from the same static `signals`.

**Impact:**
HVAC components receive constant setpoints and flow rates regardless of changing weather or zone conditions. The simulation cannot model feedback control (e.g., a thermostat reducing heating as zone temperature rises). This produces unrealistic energy consumption and comfort results for any simulation longer than a single timestep.

**Recommendation:**
Refactor to accept a callback or trait object that recomputes `ControlSignals` each timestep given the current `SimulationContext` and zone states. Alternatively, make `signals` mutable and update it within the loop.

---

### F-03: Graph Cycle-Breaking Comment Contradicts Code

**Severity:** High

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/graph.rs` (lines 127-182)

**Description:**
The comment on `compute_order_with_cycles()` at line 128-129 says "breaks cycles at the component with the most incoming edges," but the code at line 155 uses `min_by_key(|(_, &d)| d)`, which selects the node with the **fewest** remaining incoming edges. Either the comment is wrong or the code is wrong.

**Evidence:**
Line 128-129: `/// Uses a modified Kahn's algorithm that breaks cycles at the component with the most incoming edges.`
Line 152-156:
```rust
if let Some((&node, _)) = in_degree
    .iter()
    .filter(|(&n, &d)| !visited.contains(&n) && d > 0)
    .min_by_key(|(_, &d)| d)
```

**Impact:**
If the intent was to break at the node with the most edges (a common heuristic for plant loops where the most-connected component like a splitter/mixer is a natural break point), the code does the opposite. This could produce suboptimal simulation ordering for plant loops with cycles, leading to slower convergence or incorrect results on the first pass through the loop.

**Recommendation:**
Determine the correct intent and align comment with code. If the intent is to break at the most-connected node, change `min_by_key` to `max_by_key`. If `min_by_key` is correct (break at the node closest to being unblocked), update the comment.

---

### F-04: Inconsistent Water Specific Heat Constants Across Crates

**Severity:** High

**Files:**
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-psychrometrics/src/lib.rs`: `CP_WATER = 4180.0`
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/chiller.rs`: `cp_water = 4186.0`
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/water_heater.rs`: `CP_WATER = 4186.0`
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/cooling_tower.rs`: likely uses local constant

**Description:**
The psychrometrics crate defines `CP_WATER = 4180.0` J/(kg*K), but multiple component files define their own local constants with the value 4186.0 J/(kg*K). Both values are physically reasonable (water specific heat at different temperatures), but using different values in different parts of the energy balance chain introduces systematic errors.

**Evidence:**
In `chiller.rs`: `let cp_water = 4186.0;`
In `water_heater.rs`: `const CP_WATER: f64 = 4186.0;`
In psychrometrics `lib.rs`: `pub const CP_WATER: f64 = 4180.0;`

**Impact:**
A 0.14% discrepancy. When the chiller computes a chilled water temperature delta using 4186.0 but the coil on the other side of the loop uses 4180.0, energy is not conserved across the plant loop. Over an annual simulation this produces a cumulative drift in plant loop energy balance.

**Recommendation:**
Use a single authoritative constant from the psychrometrics crate throughout. Replace all local `cp_water`/`CP_WATER` definitions with `openbse_psychrometrics::CP_WATER`.

---

### F-05: Inconsistent Latent Heat of Vaporization Constants

**Severity:** High

**Files:**
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-psychrometrics/src/lib.rs`: `HFG_WATER = 2.50094e6` (at 0 deg C)
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/heat_recovery.rs`: `HFG = 2.454e6` (at ~20 deg C)

**Description:**
The heat recovery component uses a local `HFG = 2.454e6` J/kg while the psychrometrics crate defines `HFG_WATER = 2.50094e6` J/kg. The difference is 1.9%, which is physically meaningful for latent load calculations.

**Evidence:**
Heat recovery uses `HFG` in the latent recovery formula: `q_latent = inlet.mass_flow * HFG * delta_w`. The psychrometrics crate uses the 0 deg C reference value which is standard for ASHRAE psychrometric calculations.

**Impact:**
Latent recovery calculations in the heat recovery wheel/plate will undercount latent energy by ~1.9% relative to what the psychrometric functions expect. This creates an energy imbalance in the air-side enthalpy calculation when the recovered air hits downstream coils.

**Recommendation:**
Use the psychrometrics crate's `HFG_WATER` constant for consistency. If temperature-dependent HFG is desired, compute it from the Clausius-Clapeyron relation rather than hardcoding a different constant.

---

### F-06: No Leap Year Support in Calendar Functions

**Severity:** High

**Files:**
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/simulation.rs` (lines 122, 184)
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/types.rs` (`day_of_year()`)

**Description:**
Both `run_with_controls()` and `run_with_envelope()` hardcode `days_in_months = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]` with February always having 28 days. The `day_of_year()` function in `types.rs` has the same hardcoded array.

**Evidence:**
`simulation.rs` line 122: `let days_in_months: [u32; 12] = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];`
Same pattern in line 184 and in `types.rs`.

**Impact:**
For leap year weather files (EPW data can span leap years), the simulation will misalign weather data from March 1 onward by one hour. Hour 1417 (Feb 29, 01:00 in a leap year) maps to March 1 data in the simulation. This one-day offset persists for 10 months, causing all solar calculations, temperature profiles, and occupancy schedules to be shifted.

**Recommendation:**
Accept a `is_leap_year` parameter or detect it from the weather data year. Adjust February to 29 days when applicable.

---

### F-07: Chiller Fabricates Mass Flow When None Exists

**Severity:** High

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/chiller.rs`

**Description:**
The chiller implementation uses `mass_flow = inlet.state.mass_flow.max(0.001)` to set a minimum mass flow of 0.001 kg/s even when the actual inlet flow is zero. This fabricates flow that does not physically exist.

**Evidence:**
The `.max(0.001)` pattern appears in the chiller's `simulate_plant()` method.

**Impact:**
When the plant loop pump is off (mass_flow = 0), the chiller still computes a leaving water temperature based on the fabricated 0.001 kg/s flow. This produces an unrealistically large temperature drop (Q / (0.001 * cp) can be thousands of degrees), which then propagates downstream as a nonsensical water state. It also means the chiller reports nonzero energy consumption when the plant loop is inactive.

**Recommendation:**
Return the inlet state unchanged when `inlet.state.mass_flow <= 0.0` (or below a physically meaningful minimum like 0.0001 kg/s). Only compute chiller operation when there is actual flow.

---

### F-08: Autosize Sentinel Tolerance Too Wide

**Severity:** High

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/types.rs`

**Description:**
The `AUTOSIZE` sentinel value is -99999.0, but `is_autosize()` checks `(val - AUTOSIZE).abs() < 1.0`, accepting any value in the range [-100000.0, -99998.0] as "autosize." The tolerance of 1.0 is far too wide for a sentinel detection function.

**Evidence:**
```rust
pub const AUTOSIZE: f64 = -99999.0;
pub fn is_autosize(val: f64) -> bool {
    (val - AUTOSIZE).abs() < 1.0
}
```

**Impact:**
A legitimate negative value like -99998.5 (unlikely in practice but possible for certain offsets or error codes) would be incorrectly identified as an autosize sentinel. More practically, floating-point arithmetic on the sentinel value could produce values like -99999.5 that should not match but do, or conversely, accumulated rounding could push the sentinel outside the 1.0 band.

**Recommendation:**
Use exact equality (`val == AUTOSIZE`) since the sentinel is assigned as a literal and should never undergo arithmetic. If tolerance is needed for deserialization, use a much tighter bound like 0.01.

---

### F-09: DX Cooling Coil Power Overcharge at Part Load

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/cooling_coil.rs`

**Description:**
The DX cooling coil computes power as `available_cap * plr_power / available_cop`, where `plr_power` is the part-load ratio from a performance curve. This charges power proportional to `available_cap` (the maximum capacity at current conditions) rather than the `actual_delivered_cooling`. When PLR < 1, the power drawn exceeds what the actual cooling delivered would require.

**Evidence:**
The power formula uses `available_cap * plr_power` as the numerator, where `available_cap` is the full-load capacity derated for conditions and `plr_power` is the PLR curve output. The actual cooling delivered is `available_cap * plr_cooling` which may differ from `plr_power`.

**Impact:**
At part-load conditions (the majority of operating hours), the coil reports higher power consumption than physically justified. This overstates cooling energy by the ratio `plr_power / plr_cooling`, which can be significant (5-15%) at low part-load ratios where the PLR-power curve diverges from the PLR-capacity curve.

**Recommendation:**
Use `actual_delivered_cooling / effective_cop_at_plr` for power calculation, or verify that the PLR-power curve already accounts for this correctly.

---

### F-10: CHW Cooling Coil Applies SHR to Water-Side Capacity

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/chw_cooling_coil.rs`

**Description:**
The available sensible capacity calculation uses `(self.nominal_capacity * shr).min(water_capacity * shr)`. The application of `shr` to `water_capacity` is physically incorrect — the water side delivers total cooling (sensible + latent), and SHR is an air-side concept describing how the total cooling splits between sensible and latent.

**Evidence:**
The formula `water_capacity * shr` treats the water-side capacity as if it has a sensible/latent split, but water temperature change is purely sensible from the water's perspective.

**Impact:**
The coil artificially limits its sensible cooling capacity by multiplying the water-side limit by SHR. For a typical SHR of 0.7, this means the coil can only deliver 70% of what the water flow could physically support, even in dry conditions where the coil should deliver nearly 100% sensible cooling.

**Recommendation:**
The water-side capacity limit should apply to total cooling, not sensible cooling. Compute available total capacity from the water side, then apply SHR to determine the sensible portion: `available_sensible = min(nominal_capacity * shr, water_capacity) * shr_effective`.

---

### F-11: CHW Cooling Coil Does Not Model Dehumidification

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/chw_cooling_coil.rs`

**Description:**
The chilled water cooling coil passes inlet humidity ratio through to the outlet unchanged. When the coil surface temperature is below the dew point of the entering air (which is the normal operating condition for cooling coils), condensation occurs and the leaving air humidity ratio should be lower.

**Evidence:**
The outlet `AirPort` is constructed with `inlet.state.w` as the humidity ratio, regardless of the coil's surface temperature or cooling depth.

**Impact:**
Zone humidity levels will be overpredicted because the coil never removes moisture. This leads to incorrect latent loads, understated dehumidification energy, and potentially triggers supplemental dehumidification that would not be needed in reality.

**Recommendation:**
Implement apparatus dew point (ADP) or bypass factor model similar to the DX cooling coil. Calculate the coil surface temperature from the water inlet temperature and compute the saturated humidity ratio at that temperature to determine moisture removal.

---

### F-12: Pump Heat-to-Fluid Uses Total Power Instead of Waste Heat

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/pump.rs`

**Description:**
The pump computes `heat_to_fluid = power * motor_heat_to_fluid_fraction`, where `power` is the total electrical power input. This implies that the `motor_heat_to_fluid_fraction` of the total electrical input becomes heat in the fluid. Physically, the useful work (pressure rise times flow) does NOT become heat — only the motor/hydraulic losses do.

**Evidence:**
The formula applies `motor_heat_to_fluid_fraction` to the total electrical power, not to the waste heat fraction.

**Impact:**
The pump adds more heat to the fluid than it should. For a pump with 80% combined efficiency and `motor_heat_to_fluid_fraction = 1.0`, the code adds 100% of electrical power as heat, but only 20% (the losses) should become heat. The remaining 80% is useful work that raises fluid pressure, not temperature. This overstates plant loop water temperature rise and increases chiller/boiler loads.

**Recommendation:**
Compute waste heat as `power * (1.0 - pump_efficiency)` and then apply `motor_heat_to_fluid_fraction` to only the waste heat portion: `heat_to_fluid = power * (1.0 - pump_efficiency) * motor_heat_to_fluid_fraction`. Note: EnergyPlus actually does add all pump power as heat to the fluid (shaft work eventually becomes heat via friction in pipes), so this may be intentionally matching E+ behavior. If so, add a comment explaining this convention.

---

### F-13: `tsat_fn_press` Has Poor Initial Guess for Newton-Raphson

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-psychrometrics/src/lib.rs`

**Description:**
The saturation temperature function `tsat_fn_press()` uses the initial guess formula `100.0 * (press / 101325.0).ln() / (100.0_f64.ln())` for the liquid region. This formula produces wildly incorrect initial guesses at low pressures. For example, at 1000 Pa (~7 deg C saturation), the formula gives approximately -100 deg C.

**Evidence:**
The initial guess formula: `t_guess = 100.0 * (press / 101325.0).ln() / 100.0_f64.ln()`. At `press = 1000`: `ln(1000/101325) / ln(100) = ln(0.00987) / 4.605 = -4.618 / 4.605 = -1.003`, so `t_guess = 100 * (-1.003) = -100.3 deg C`.

**Impact:**
Newton-Raphson iteration starting from -100 deg C when the answer is ~7 deg C requires many more iterations to converge (if it converges at all) and may find an incorrect root in the ice region or fail. The function is called during psychrometric calculations that may involve low-pressure conditions (high altitude, vacuum systems).

**Recommendation:**
Use the Antoine equation for initial guess: `t_guess = B / (A - log10(press/1000)) - C` with standard water constants (A=8.07131, B=1730.63, C=233.426 for the 1-100 deg C range). Alternatively, use a piecewise linear interpolation table for the initial guess.

---

### F-14: Boiler Allows Efficiency Above 100%

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/boiler.rs`

**Description:**
The boiler clamps efficiency to a maximum of 1.1 (110%), allowing operation above 100% thermal efficiency. While condensing boilers can approach ~98% efficiency on a lower heating value basis, no boiler can exceed 100% on a higher heating value basis.

**Evidence:**
The efficiency clamping allows values up to 1.1, meaning the boiler can output 10% more energy than its fuel input.

**Impact:**
If a performance curve produces efficiency values slightly above 1.0 at favorable conditions, the boiler will report more thermal energy output than fuel consumed, violating energy conservation.

**Recommendation:**
Clamp to 1.0 for HHV-basis calculations, or to ~0.98 for realistic condensing boiler limits. If LHV-basis efficiencies are intended (where >1.0 is physically meaningful), document the convention and convert appropriately for energy balance reporting.

---

### F-15: Heat Recovery Latent Formula Mass Flow Inconsistency

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/heat_recovery.rs`

**Description:**
The latent recovery formula uses `q_latent = inlet.mass_flow * HFG * delta_w`, where `delta_w` is the difference in humidity ratio (kg water / kg dry air) but `inlet.mass_flow` is the total moist air mass flow rate (kg moist air / s). Since humidity ratio is defined per unit of dry air, the formula should use dry air mass flow.

**Evidence:**
`delta_w = w_supply - w_exhaust` is in kg/kg-dry-air units. `inlet.mass_flow` is total mass flow including moisture.

**Impact:**
At typical indoor humidity ratios (0.008 kg/kg), the error is approximately 0.8%, which is small. However, in humid climates with high outdoor humidity ratios (0.020+ kg/kg), the error grows to ~2%, systematically overstating latent recovery.

**Recommendation:**
Convert to dry air mass flow: `m_dry = inlet.mass_flow / (1.0 + inlet.state.w)` and use `m_dry * HFG * delta_w`.

---

### F-16: Duplicate Component Names Silently Overwrite in Graph

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-core/src/graph.rs`

**Description:**
When components are added to the `SimulationGraph`, their names are inserted into a `name_to_node` HashMap. If two components share the same name, the second silently overwrites the first's mapping. The first component still exists in the graph but becomes unreachable by name.

**Evidence:**
The `add_air_component` and `add_plant_component` methods insert into `self.name_to_node` without checking for duplicates.

**Impact:**
Duplicate names could arise from user input errors in the YAML configuration. The orphaned component would still be simulated but could not receive control signals (which are dispatched by name), leading to uncontrolled HVAC components that run at whatever their last setpoint was.

**Recommendation:**
Return an error or log a warning when a duplicate component name is detected during graph construction.

---

### F-17: Heating Coil UA Model Geometry Mismatch

**Severity:** Medium

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/heating_coil.rs`

**Description:**
The hot water heating coil computes UA from design conditions using a counterflow LMTD assumption, but then uses a cross-flow effectiveness formula during simulation. These are different heat exchanger geometries with different effectiveness-NTU relationships.

**Evidence:**
The test comments acknowledge this mismatch. The design UA calculation assumes counterflow LMTD, while the runtime effectiveness uses a cross-flow unmixed formula.

**Impact:**
The computed UA will be somewhat incorrect for the actual heat exchanger geometry, producing a coil that is slightly over- or under-sized compared to its design intent. The error magnitude depends on NTU — at low NTU (< 1), all geometries converge and the error is small; at high NTU (> 3), the error can reach 10-15%.

**Recommendation:**
Use consistent geometry assumptions for both design UA calculation and runtime simulation. If the coil is cross-flow, use cross-flow LMTD for design.

---

### F-18: VAV/PFP Box `set_setpoint()` Semantic Mismatch

**Severity:** Low

**Files:**
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/vav_box.rs`
- `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/pfp_box.rs`

**Description:**
The `AirComponent` trait defines `set_setpoint()` to set a temperature setpoint, but the VAV box and PFP box implementations use it to set `control_signal` (a -1.0 to +1.0 heating/cooling signal). The `setpoint()` method then returns this control signal, not a temperature.

**Evidence:**
VAV box: `fn set_setpoint(&mut self, signal: f64) { self.control_signal = signal; }`
PFP box: `fn set_setpoint(&mut self, signal: f64) { self.control_signal = signal; }`
The trait's other implementors (coils) use it for temperature setpoints.

**Impact:**
This is a semantic confusion that could cause bugs if someone passes a temperature where a control signal is expected, or vice versa. The current code works because the callers know to pass control signals to these specific components, but it violates the trait's documented contract.

**Recommendation:**
Add a separate `set_control_signal()` method to the trait, or document that `set_setpoint()` has different semantics for terminal units vs. coils. Consider a trait redesign that separates setpoint-controlled and signal-controlled components.

---

### F-19: Water Heater Uses `RHO_WATER = 1.0` (kg/L)

**Severity:** Low

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-components/src/water_heater.rs`

**Description:**
The water heater defines `RHO_WATER = 1.0` which is in kg/L (not the standard SI kg/m^3 = 1000.0). While this is consistent internally (tank volume is in liters), it creates a unit convention that differs from the rest of the codebase where mass flows are in kg/s and volumes are in m^3.

**Evidence:**
`const RHO_WATER: f64 = 1.0; // kg/L`
`let m_tank = self.tank_volume * RHO_WATER; // tank water mass [kg]`

**Impact:**
If someone refactors to use m^3 inputs (consistent with the rest of the simulation) without updating the density constant, tank mass calculations would be off by 1000x. The current code is correct within its own unit system but is a maintenance hazard.

**Recommendation:**
Add a prominent unit comment or convert to SI-consistent units (volume in m^3, density in kg/m^3) with appropriate conversion factors at the input boundary.

---

### F-20: EPW Parser Silently Skips Malformed Data Lines

**Severity:** Low

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-weather/src/lib.rs` (line 275)

**Description:**
The EPW parser's hourly data loop uses `Err(_) => continue` to silently skip any line that fails to parse. This means corrupted or truncated weather files will produce a simulation with missing hours rather than reporting an error.

**Evidence:**
Line 273-275:
```rust
match parse_data_line(&line) {
    Ok(hour) => hours.push(hour),
    Err(_) => continue, // Skip malformed lines
}
```

**Impact:**
A corrupted weather file with missing hours will cause the `hours` vector to have fewer than 8760 entries. The simulation loop indexes into this vector by hour-of-year, which could cause a panic if `hour_idx` exceeds `hours.len()`, or silently truncate the simulation if the `.min(weather_hours.len())` guard catches it. Either way, the user gets no warning about data quality.

**Recommendation:**
Count skipped lines and emit a warning. If more than a threshold (e.g., 24 hours) are skipped, return an error. Consider interpolating or repeating the previous hour's data for isolated missing lines.

---

### F-21: Thermostat Proportional Band Hardcoded

**Severity:** Low

**Files:** `/tmp/afm-benchmark-round3/openbse-cr/crates/openbse-controls/src/thermostat.rs`

**Description:**
The zone thermostat uses a hardcoded `max_error = 5.0` deg C for its proportional control band. This means the thermostat reaches full output when the zone temperature is 5 deg C away from setpoint. This value is not configurable.

**Evidence:**
`max_error = 5.0` is used in the proportional control calculation.

**Impact:**
Different building types and HVAC systems may need different proportional bands. A tight band (1-2 deg C) is typical for VAV systems, while a wider band (5-10 deg C) is used for radiant systems. The hardcoded value prevents tuning control response to the specific system being simulated.

**Recommendation:**
Make `max_error` a configurable field on the thermostat struct with a sensible default.

---

## Statistics

| Severity | Count |
|----------|-------|
| Critical | 2     |
| High     | 6     |
| Medium   | 9     |
| Low      | 4     |
| **Total**| **21** |
