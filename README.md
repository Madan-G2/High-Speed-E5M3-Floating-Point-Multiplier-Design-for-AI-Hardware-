# High-Speed-E5M3-Floating-Point-Multiplier-Design-for-AI-Hardware-

# E5M3 Floating Point Multiplier

A high-speed 9-bit floating point multiplier designed for AI accelerator applications, implemented in complementary CMOS technology.

## Overview

This project implements an E5M3 floating point multiplier targeting 2GHz operation. The design uses a 5-stage pipeline architecture with saturation logic instead of exception handling to maximize throughput for neural network inference workloads.

## Data Format

The 9-bit E5M3 floating point format:

| Field | Bits | Description |
|-------|------|-------------|
| Sign (S) | 1 | Sign-magnitude representation (0 = positive, 1 = negative) |
| Exponent (E) | 5 | Excess-15 (bias of 15) encoding |
| Mantissa (M) | 3 | Fractional part with implicit hidden 1 (effective 4-bit mantissa) |

**Bit Layout:** `[S][E4 E3 E2 E1 E0][M2 M1 M0]`

### Special Values

- **Zero:** Exponent = `00000`, Mantissa = `000` (sign bit ignored; supports ±0)
- **Maximum (Saturation Upper):** Sign = computed, Exponent = `11111`, Mantissa = `100`
- **Minimum (Saturation Lower):** Any negative exponent result → true zero (`00000 000`)

### Compatibility

The hardware accepts E5M2 format data by setting the low-order mantissa bit to zero, enabling compatibility with NVIDIA's 8-bit floating point standard.

## Architecture

### 5-Stage Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Stage 1   │───▶│   Stage 2   │───▶│   Stage 3   │───▶│   Stage 4   │───▶│   Stage 5   │
│ Denorm Det. │    │ Zero Handle │    │ Sign/Exp/   │    │ Final Add   │    │ Saturation  │
│             │    │             │    │ Mantissa PP │    │             │    │ & Output    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

**Stage 1 - Denormalization Detection:**
- Two 9-bit OR gates detect if operands are normalized
- Determines hidden bit value (1 for normalized, 0 for denormalized/zero)

**Stage 2 - Zero Operand Handling:**
- Multiplexer-based selection implements X × 0 = 0 identity
- Early zero detection simplifies downstream logic

**Stage 3 - Initial Multiplication:**
- Sign computation via XOR gate
- 7-bit carry-save exponent addition (Exp_A + Exp_B - 15 + 1)
- Mantissa partial product generation using AND gates

**Stage 4 - Final Addition:**
- Carry-skip adder for exponent path
- 4-bit ripple carry adder for mantissa path
- Generate-propagate logic for optimized carry calculation

**Stage 5 - Saturation and Output:**
- 3-input multiplexers select between normal result, upper saturation, or zero
- S0 (upper saturation) via 6-bit AND gate
- S1 (zero saturation) from exponent MSB

## Interface

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| A | Input | 9 | First operand (E5M3 format) |
| B | Input | 9 | Second operand (E5M3 format) |
| Z | Output | 9 | Product result (E5M3 format) |
| clk | Input | 1 | Clock (target: 2GHz) |
| rst_n | Input | 1 | Active-low reset |
| push_in | Input | 1 | Valid data on inputs |
| push_out | Output | 1 | Valid data on output |

## Design Constraints

- **Maximum Fanout:** 4 gates per driver
- **Clock Frequency Target:** 2GHz
- **Technology:** Complementary static CMOS
- **Pipeline Latency:** 5 clock cycles

## Key Components

### Basic Gates
- 2-input OR (NAND with inverted inputs)
- 2-input NOR (parallel PMOS, series NMOS)
- 2-input AND (NOR with inverted inputs)
- 2-input NAND (series PMOS, parallel NMOS)
- Pass Gate / Transmission Gate
- XOR Gate (pass-gate logic)

### Multi-Input Gates
- 3-input NOR/NAND
- 4-input OR/AND/NAND
- 6-input AND (saturation detection)
- 9-bit OR (3×3-NOR → 3-NAND structure)

### Arithmetic Units
- 1-bit Full Adder (factorized CMOS, no XOR gates)
- 4-bit Ripple Carry Adder
- Carry-Save Adder Tree

### Sequential Elements
- Mux-based Flip-Flop with active-low reset
- Provides both Q and Q' outputs

### Multiplexers
- 3-input MUX (pass-gate implementation)

## Buffer Strategy

Hierarchical buffer trees maintain timing and fanout constraints:

- **Clock Buffers:** Distribute clock to all flip-flops (fanout ≤ 4)
- **Inverted Clock Buffers:** Complementary clock for master-slave flip-flops
- **Reset Buffers:** Active-low reset distribution
- **Hold Time Buffers:** Inserted in short paths to prevent hold violations

## Floating Point Multiplication Algorithm

```
Result_Sign = Sign_A XOR Sign_B
Result_Exp  = Exp_A + Exp_B - Bias + Normalization_Adjustment
Result_Mant = (1.Mant_A) × (1.Mant_B)  // 4-bit × 4-bit = 8-bit product
```

### Normalization
- Mantissa product bits [7:5] (positions 2^5, 2^4, 2^3) selected for result
- Exponent adjusted based on mantissa overflow

### Saturation Logic
- **Overflow:** Result saturates to maximum representable value
- **Underflow:** Result saturates to true zero

## File Structure

```
project/
├── README.md                    # This file
├── doc/
│   ├── EE224proj_report.pdf    # Detailed design report
│   └── Proj224f25desc.pdf      # Project specification
├── src/
│   ├── gates/                  # Basic gate implementations
│   │   ├── or2.sch
│   │   ├── nor2.sch
│   │   ├── and2.sch
│   │   ├── nand2.sch
│   │   ├── xor2.sch
│   │   └── pass_gate.sch
│   ├── multi_input/            # Multi-input gates
│   │   ├── nor3.sch
│   │   ├── nand3.sch
│   │   ├── or4.sch
│   │   ├── and4.sch
│   │   ├── nand4.sch
│   │   ├── and6.sch
│   │   └── or9.sch
│   ├── arithmetic/             # Arithmetic units
│   │   ├── full_adder.sch
│   │   └── rca4.sch
│   ├── sequential/             # Sequential elements
│   │   └── dff_mux_rst.sch
│   ├── datapath/               # Datapath components
│   │   ├── mux3.sch
│   │   └── buffer.sch
│   └── top/                    # Top-level design
│       └── fp_mult_e5m3.sch
├── sim/
│   └── testbench.va            # VerilogA testbench
└── lib/
    └── gpdk045/                # 45nm PDK (if applicable)
```

## Simulation

A VerilogA testbench is provided for verification. The design must pass all test cases for full credit.

### Running Simulations

1. Open Cadence Virtuoso
2. Load the top-level schematic
3. Configure ADE with the provided testbench
4. Run transient simulation
5. Verify correct output for all test vectors

## Design Decisions

1. **Factorized Full Adders:** Avoids XOR gates for reduced critical path delay
2. **De Morgan Gate Implementations:** OR as NAND with inverted inputs, AND as NOR with inverted inputs
3. **Complementary Flip-Flop Outputs:** Q and Q' outputs enable direct feeding to inverted-input gates
4. **Pass-Gate Multiplexers:** Low-delay signal routing with rail-to-rail output
5. **Carry-Save Adder Tree:** Reduces critical path in mantissa multiplication
6. **Carry-Skip Adder:** Optimizes exponent addition path


## Course Information

**Course:** EE 224 - Digital Integrated Circuits  
**Term:** Fall 2025  
**Project:** E5M3 Floating Point Multiplier

## References

- IEEE Std 754-2019: IEEE Standard for Floating-Point Arithmetic
  - ISBN: 1-5044-5924-5
  - Available through university library
- NVIDIA E5M2 8-bit Floating Point Format Specification

