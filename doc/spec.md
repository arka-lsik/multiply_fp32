# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

**The top-level module name must remain exactly `fmultiplier`**, matching the provided skeleton file exactly. The requirement that the output file be named `multiply_fp32.sv` refers only to the filename on disk — it does NOT mean the module identifier inside the file should be renamed. Do not change `module fmultiplier(...)` to any other name (e.g. `module multiply_fp32(...)`); doing so will break instantiation in the hidden testbench.

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.


**Note:** all rounding, shifting, and exponent-arithmetic registers used internally should be sized with at least one extra bit of headroom beyond their nominal IEEE-754 field width (e.g. mantissa-increment results, carry-detection bits, and any narrowed/biased exponent fields). Under-sized registers can silently truncate or wrap around during intermediate computation, producing incorrect results without any simulation or synthesis error.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
1. **Underflow alignment** toward exponent -126:
   - Computes shift amount `sh = (-126 - z_e)` when `z_e < -126`.
   - Shifts mantissa right and accumulates shifted-out bits into sticky.
2. **Normalize** if MSB missing:
   - Left-shifts mantissa while adjusting exponent, carrying guard into LSB.
   - If the shift amount required to align to exponent -126 meets or exceeds the mantissa width (i.e. the true product is too small to represent even as a denormal), the mantissa must become exactly zero and the sticky bit must be set to 1. The final packed result in Stage 7 must then be an exact zero (correct sign, zero exponent field, zero fraction) — this case occurs even when multiplying two ordinary normal numbers whose product underflows completely, not just with denormal inputs.
   - **Important:** steps 1, 2, and 3 in this stage are sequential operations that can all apply to the same result within one cycle — they are NOT mutually exclusive alternatives. Do not implement them as `if (underflow) ... else if (!normalized) ... else (round)`. A value that underflows still needs normalization/rounding logic applied to it afterward in the same cycle. Structure Stage 6 so that: first, if underflow-alignment applies, it updates the mantissa/guard/round/sticky/exponent; then, independently, check if the (possibly just-updated) mantissa needs normalization; then, independently, apply the RNE rounding check using the (possibly just-updated) guard/round/sticky bits. Use separate `if` blocks feeding into each other, not one big if/else-if/else chain.
3. **RNE rounding**:
   - If `G == 1` and `(R || S || LSB)` then increment mantissa.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.
     - Compute the incremented mantissa in a register at least 25 bits wide so the carry-out bit is directly observable (e.g. bit 24 of a 25-bit sum). Do not detect this overflow by comparing the mantissa to an all-1s pattern before incrementing, and do not check a bit index beyond the width of a 24-bit register — both approaches will silently fail to detect the carry.
    - **The only correct way to detect rounding overflow is:** compute `{1'b0, z_m} + 25'd1` (a 25-bit-wide sum) and check bit `[24]` of that sum — if it is 1, the mantissa overflowed. Any of the following are INCORRECT and must not be used: comparing `z_m == 24'hFFFFFF` before incrementing, using a reduction-AND check like `&z_m` before incrementing, or checking any bit index on a register narrower than 25 bits. The overflow check must always happen on the post-increment sum, not the pre-increment mantissa.

### Stage 7 — Pack
- For normal path:
  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
- Asserts `out_valid` for one cycle and clears `busy`.
  - **All overflow and underflow checks must be performed on the full-width signed exponent `z_e` (the 10-bit signed register), not on an already-narrowed 8-bit biased exponent.** Narrowing `z_e + 127` into an 8-bit field before checking its range can silently wrap around for extreme exponent values, causing genuine overflow/underflow cases to be misclassified as normal.
  - If `z_e` (checked in its full signed width) indicates overflow -> output INF.
  - If `z_e` (checked in its full signed width) indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
  - **At the exact boundary `z_e == -126`, the exponent field is NOT always 0.** You must check `z_m[23]` to decide: if `z_m[23] == 1`, the mantissa is normalized and this is actually the smallest representable **normal** number — pack it with biased exponent field = 1 (not 0) and fraction = `z_m[22:0]`. Only if `z_m[23] == 0` is this a true denormal or zero, in which case pack exponent field = 0 and fraction = `z_m[22:0]`. Do not unconditionally set the exponent field to 0 whenever `z_e == -126`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---