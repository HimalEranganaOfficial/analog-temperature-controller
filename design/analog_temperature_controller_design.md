# Pure Analog Temperature Control Circuit (Proteus)

## 1) Complete Schematic (Labeled Connections, References, Net Names)

### 1.1 Net Names

- `+12V` : main regulated supply (>= 3 A)
- `GND` : common ground
- `VSENSE` : filtered thermistor divider voltage (temperature signal)
- `VREF` : filtered potentiometer/reference voltage (setpoint signal)
- `COOL_CMD` : LM393 channel-A open-collector output (cooling command)
- `HEAT_CMD` : LM393 channel-B open-collector output (heating command)
- `FAN1_GATE`, `FAN2_GATE`, `HEAT_GATE` : MOSFET gate nodes

### 1.2 Component References

- Sensor/Input:
  - `TH1` = 10 kΩ NTC @ 25°C
  - `R1` = 10 kΩ (thermistor divider lower resistor)
  - `RV1` = 10 kΩ linear potentiometer (setpoint)
  - `R2`, `R3` = 68 kΩ (setpoint range shaping network)
  - `C1` = 10 nF (VSENSE filter)
  - `C2` = 10 nF (VREF filter)
- Comparator:
  - `U1` = LM393 dual comparator
  - `R4`, `R5` = 10 kΩ pull-up resistors for LM393 outputs
  - `C3` = 100 nF decoupling at LM393 VCC
- Power Stage:
  - `Q1` = IRF540N (Fan 1 low-side switch)
  - `Q2` = IRF540N (Fan 2 low-side switch)
  - `Q3` = IRF540N (Heater low-side switch)
  - `R6`, `R7`, `R8` = 100 Ω gate series resistors
  - `R9`, `R10`, `R11` = 10 kΩ gate pull-down resistors
  - `D1`, `D2`, `D3` = 1N4007 flyback diodes
- Loads:
  - `M1` = 12 V fan (exhaust, model 40 Ω or DC MOTOR)
  - `M2` = 12 V fan (intake, model 40 Ω or DC MOTOR)
  - `RH1` = heater element model 6 Ω

### 1.3 Electrical Connections (Pin-Level Netlist)

#### Sensor Divider
- `+12V -> TH1(top)`
- `TH1(bottom) -> VSENSE`
- `VSENSE -> R1(top)`
- `R1(bottom) -> GND`
- `VSENSE -> C1 -> GND`

#### Setpoint Network
- `RV1(end_A) -> +12V`
- `RV1(end_B) -> GND`
- `RV1(wiper) -> VREF_RAW`
- `R2` and `R3` used as divider scaling around `VREF_RAW` to compress effective range to 20–26°C equivalent:
  - `VREF_RAW -> R2 -> VREF`
  - `VREF -> R3 -> GND`
- `VREF -> C2 -> GND`

> In Proteus, adjust RV1 during calibration so `VREF` corresponds to 20–26°C equivalent `VSENSE` voltage (see calculations section).

#### Comparator Logic (LM393)
- `U1(VCC) -> +12V`
- `U1(GND) -> GND`
- `C3` placed directly across `U1(VCC)` and `U1(GND)`

**Cooling comparator (U1A):**
- `U1A(+) <- VREF`
- `U1A(-) <- VSENSE`
- `U1A(OUT) -> COOL_CMD`
- `COOL_CMD -> R4 -> +12V` (mandatory pull-up)

**Heating comparator (U1B):**
- `U1B(+) <- VSENSE`
- `U1B(-) <- VREF`
- `U1B(OUT) -> HEAT_CMD`
- `HEAT_CMD -> R5 -> +12V` (mandatory pull-up)

This complementary arrangement implements:
- `VSENSE > VREF` -> `COOL_CMD` active -> fans ON, heater OFF
- `VSENSE < VREF` -> `HEAT_CMD` active -> heater ON, fans OFF

#### MOSFET Gate and Load Wiring

**Fan 1 path**
- `COOL_CMD -> R6 -> FAN1_GATE -> Q1(G)`
- `FAN1_GATE -> R9 -> GND`
- `Q1(S) -> GND`
- `Q1(D) -> M1(-)`
- `M1(+) -> +12V`
- `D1` across M1: cathode to `+12V`, anode to `M1(-)/Q1(D)`

**Fan 2 path**
- `COOL_CMD -> R7 -> FAN2_GATE -> Q2(G)`
- `FAN2_GATE -> R10 -> GND`
- `Q2(S) -> GND`
- `Q2(D) -> M2(-)`
- `M2(+) -> +12V`
- `D2` across M2: cathode to `+12V`, anode to `M2(-)/Q2(D)`

**Heater path**
- `HEAT_CMD -> R8 -> HEAT_GATE -> Q3(G)`
- `HEAT_GATE -> R11 -> GND`
- `Q3(S) -> GND`
- `Q3(D) -> RH1(-)`
- `RH1(+) -> +12V`
- `D3` across RH1: cathode to `+12V`, anode to `RH1(-)/Q3(D)`

---

## 2) Operational Principle

1. `TH1 + R1` converts temperature to voltage at `VSENSE`.
   - For NTC: temperature increase -> resistance decreases -> `VSENSE` increases if NTC is top leg.
2. `RV1 (+ R2/R3 shaping)` creates `VREF` corresponding to desired 20–26°C setpoint.
3. LM393 compares `VSENSE` and `VREF` with two channels wired as complementary comparators.
4. Open-collector outputs pull low when active; pull-ups generate valid high level.
5. `COOL_CMD` drives Q1 and Q2 together (both fans).
6. `HEAT_CMD` drives Q3 (heater).
7. Because channels are complementary, only one actuator group is active at a time.

---

## 3) Design Calculations

### 3.1 Thermistor Divider (VSENSE)

For `TH1` (top leg) and `R1 = 10 kΩ` (bottom leg):

\[
V_{SENSE} = V_{CC} \cdot \frac{R_1}{R_{NTC}(T) + R_1}
\]

Using typical 10 kΩ NTC with \( B \approx 3950 \):

\[
R(T)=R_{25}\cdot e^{B\left(\frac{1}{T}-\frac{1}{T_{25}}\right)}
\]

where \(T\) in Kelvin, \(R_{25}=10k\Omega\), \(T_{25}=298.15K\).

Approximate values:
- at 20°C (293.15K): \(R_{NTC}\approx12.5k\Omega\) -> \(V_{SENSE}\approx 5.33V\)
- at 26°C (299.15K): \(R_{NTC}\approx9.56k\Omega\) -> \(V_{SENSE}\approx 6.13V\)

So 20–26°C corresponds to approximately **5.33 V to 6.13 V** on `VSENSE` at 12 V supply.

### 3.2 Setpoint Scaling (VREF)

`RV1` provides a tunable voltage, then `R2/R3` compress and shift the usable span.
Practical setup:
1. Place TH1 model at 20°C and read `VSENSE_20`.
2. Adjust RV1 minimum position and R2/R3 proportion so `VREF_min = VSENSE_20`.
3. Place TH1 model at 26°C and adjust RV1 maximum so `VREF_max = VSENSE_26`.
4. Verify mid-scale at 23°C aligns with midpoint voltage.

Given both fixed legs are 68 kΩ, loading is symmetric and stable; fine trim is done by RV1 in simulation.

### 3.3 MOSFET Gate Drive

- LM393 pull-up to 12 V yields gate high near 12 V (minus small sink effects), enough for IRF540N strong enhancement.
- Gate series \(100\Omega\) limits dI/dt and ringing.
- Gate pull-down \(10k\Omega\) guarantees OFF at startup/fault.

### 3.4 Heater Current and Power

Heater model \(R_H = 6\Omega\) on 12 V:

\[
I_H = \frac{12}{6}=2A,\quad P_H=VI=24W
\]

IRF540N conduction loss (typical \(R_{DS(on)} \approx 0.077\Omega\)):

\[
P_{MOSFET} \approx I^2R = (2)^2\cdot0.077 \approx 0.31W
\]

Thermally safe in simulation; for hardware use small heatsink margin.

---

## 4) Simulation Instructions (Proteus)

1. Open `/proteus/analog_temperature_controller.DSN`.
2. Confirm component values match BOM exactly.
3. For fan loads:
   - either use DC MOTOR elements
   - or resistive equivalents (`40 Ω` each) for deterministic electrical validation.
4. Set TH1 model to 10 kΩ @ 25°C, \(B=3950\) (or close manufacturer value).
5. Place virtual probes on: `VSENSE`, `VREF`, `COOL_CMD`, `HEAT_CMD`, `FAN1_GATE`, `HEAT_GATE`.
6. Sweep temperature:
   - At temperature above setpoint: `COOL_CMD` active, both fan gates high, heater gate low.
   - At temperature below setpoint: `HEAT_CMD` active, heater gate high, fan gates low.
7. Adjust RV1 for 20–26°C calibration endpoints.
8. Verify mutual exclusivity of outputs.

### Expected Waveforms / States

- `VSENSE` crosses `VREF` at setpoint.
- `COOL_CMD` and `HEAT_CMD` show complementary digital-like transitions.
- `FAN1_GATE` and `FAN2_GATE` track `COOL_CMD`.
- `HEAT_GATE` tracks `HEAT_CMD`.
- No gate floating during startup due to 10 kΩ pull-downs.

---

## 5) BOM (As Implemented)

### Resistors
- `R1` 10 kΩ 1% 0.25 W
- `R2, R3` 68 kΩ 1% 0.25 W
- `R4, R5` 10 kΩ 1% 0.25 W (LM393 pull-ups)
- `R6, R7, R8` 100 Ω 5% 0.25 W (gate damping)
- `R9, R10, R11` 10 kΩ 5% 0.25 W (gate pull-downs)

### Capacitors
- `C1, C2` 10 nF X7R
- `C3` 100 nF X7R

### Active
- `U1` LM393 dual comparator
- `TH1` NTC 10 kΩ @25°C (e.g., EPCOS B57164K0103K000)
- `RV1` 10 kΩ linear potentiometer
- `Q1, Q2, Q3` IRF540N
- `D1, D2, D3` 1N4007

### Power and Loads
- `+12V` regulated >=3 A
- `M1, M2` 12 V fans (or 40 Ω models)
- `RH1` 6 Ω heater model
