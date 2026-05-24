# Verilog Mini SRC CPU

This repository contains a Verilog implementation of a 32-bit Mini SRC (Simple RISC Computer) built for the ELEC 374 CPU design project. The project was developed through three phases: a single-bus datapath, a three-bus datapath, and a complete processor with an FSM-based control unit.

The final design supports register-register operations, immediate operations, memory access, conditional branches, jumps, subroutine linkage, I/O ports, HI/LO result movement, `nop`, and `halt`. Functional verification was done with Verilog testbenches and ModelSim-style waveform/transcript inspection. The project was completed through Phase 3; FPGA deployment was left as future work.

## Table of Contents

- [Repository Layout](#repository-layout)
- [Architecture Overview](#architecture-overview)
- [Instruction Format](#instruction-format)
- [Instruction Set](#instruction-set)
- [Addressing Modes](#addressing-modes)
- [Phase 1 - Single-Bus Datapath](#phase-1---single-bus-datapath)
- [Phase 2 - Three-Bus Datapath](#phase-2---three-bus-datapath)
- [Phase 3 - Control Unit](#phase-3---control-unit)
- [Performance Notes](#performance-notes)
- [Running Simulations](#running-simulations)
- [License](#license)

## Repository Layout

- `Phase 1/` - single-bus datapath components and unit/instruction testbenches.
- `Phase 2/` - three-bus datapath, memory, I/O ports, branch condition logic, and instruction-level testbenches.
- `Phase 3/` - integrated `processor`, `control_unit`, final three-bus `datapath`, and Phase 3 processor testbench.
- `374_Final_Report_Group_1.pdf` - project report with design discussion, instruction table, simulation notes, and appended code.
- `ELEC374_CPU.qpf` / `ELEC374_CPU.qsf` - Quartus project files.

## Architecture Overview

The Mini SRC is a 32-bit processor with sixteen 32-bit programmer-visible registers and dedicated internal CPU registers.

- `R0`-`R7` are general-purpose registers.
- `R8`-`R11` are argument registers.
- `R12` is the return address register used by `jal`.
- `R13`-`R14` are return value registers.
- `R15` is the stack pointer.
- `HI` stores the high-order multiplication result or division remainder.
- `LO` stores the low-order multiplication result or division quotient.
- `PC`, `IR`, `MAR`, and `MDR` support fetch and memory access.
- `InPort` and `OutPort` provide 32-bit external I/O.

The final datapath uses three 32-bit buses. Bus A and Bus B feed the ALU operands, while Bus C is the main writeback path. The memory subsystem is modeled as 2048 bytes of little-endian RAM, addressed as 512 32-bit words through `MAR[8:0]`.

## Instruction Format

All instructions are 32 bits wide. The implementation decodes fields using these bit positions:

| Field | Bits | Purpose |
| --- | --- | --- |
| `opcode` | `[31:27]` | Selects the instruction |
| `Ra` | `[26:23]` | Destination or primary source register |
| `Rb` | `[22:19]` | Source/base register or branch condition field |
| `Rc` | `[18:15]` | Second source register |
| `C` | `[18:0]` | 19-bit sign-extended immediate/offset |

The supported instruction families are:

- **R-format:** register-register ALU operations using `Ra`, `Rb`, and `Rc`.
- **I-format:** immediate, load/store, multiply/divide, and unary operations.
- **J-format:** jump, link, I/O, and HI/LO move operations.
- **B-format:** conditional branches using `Ra`, condition bits, and a sign-extended offset.
- **M-format:** miscellaneous instructions such as `nop` and `halt`.

## Instruction Set

| Instruction | Opcode | Format | Behavior |
| --- | --- | --- | --- |
| `add` | `00000` | R | `R[Ra] = R[Rb] + R[Rc]` |
| `sub` | `00001` | R | `R[Ra] = R[Rb] - R[Rc]` |
| `and` | `00010` | R | `R[Ra] = R[Rb] & R[Rc]` |
| `or` | `00011` | R | `R[Ra] = R[Rb] | R[Rc]` |
| `shr` | `00100` | R | Logical right shift of `R[Rb]` by `R[Rc]` |
| `shra` | `00101` | R | Arithmetic right shift of `R[Rb]` by `R[Rc]` |
| `shl` | `00110` | R | Logical left shift of `R[Rb]` by `R[Rc]` |
| `ror` | `00111` | R | Rotate right `R[Rb]` by `R[Rc]` |
| `rol` | `01000` | R | Rotate left `R[Rb]` by `R[Rc]` |
| `addi` | `01001` | I | `R[Ra] = R[Rb] + sign_extend(C)` |
| `andi` | `01010` | I | `R[Ra] = R[Rb] & sign_extend(C)` |
| `ori` | `01011` | I | `R[Ra] = R[Rb] | sign_extend(C)` |
| `div` | `01100` | I | `HI = R[Ra] % R[Rb]`, `LO = R[Ra] / R[Rb]` |
| `mul` | `01101` | I | `{HI, LO} = R[Ra] * R[Rb]` |
| `neg` | `01110` | I | `R[Ra] = -R[Rb]` |
| `not` | `01111` | I | `R[Ra] = ~R[Rb]` |
| `ld` | `10000` | I | `R[Ra] = M[R[Rb] + sign_extend(C)]` |
| `ldi` | `10001` | I | `R[Ra] = R[Rb] + sign_extend(C)` |
| `st` | `10010` | I | `M[R[Rb] + sign_extend(C)] = R[Ra]` |
| `jal` | `10011` | J | `R12 = PC`, then `PC = R[Ra]` |
| `jr` | `10100` | J | `PC = R[Ra]` |
| `br` / `brzr` / `brnz` / `brpl` / `brmi` | `10101` | B | If condition is true, `PC = PC + sign_extend(C)` |
| `in` | `10110` | J | `R[Ra] = InPort` |
| `out` | `10111` | J | `OutPort = R[Ra]` |
| `mfhi` | `11000` | J | `R[Ra] = HI` |
| `mflo` | `11001` | J | `R[Ra] = LO` |
| `nop` | `11010` | M | No operation |
| `halt` | `11011` | M | Stops the control unit run signal |

Branch condition bits are taken from the low two bits of the branch condition field:

| Condition bits | Meaning |
| --- | --- |
| `00` | Branch if zero |
| `01` | Branch if nonzero |
| `10` | Branch if positive or zero |
| `11` | Branch if negative |

The branch aliases map onto the same opcode: `brzr` uses `00`, `brnz` uses `01`, `brpl` uses `10`, and `brmi` uses `11`.

## Addressing Modes

- **Register addressing:** operands are read directly from registers, such as `add R2, R5, R6`.
- **Immediate addressing:** a sign-extended constant is embedded in the instruction, such as `addi R1, R1, 3`.
- **Direct addressing:** `Rb` can be selected as `R0` with `BAout`, producing an effective address from the constant alone.
- **Indexed addressing:** load/store addresses are formed as `R[Rb] + sign_extend(C)`.
- **Register indirect addressing:** an address can be taken from a register by using an offset of zero.
- **Relative addressing:** branch target addresses are formed from the current `PC` plus the sign-extended branch offset.

## Phase 1 - Single-Bus Datapath

Phase 1 builds the core datapath around one shared bus. Only one register or functional unit can drive the bus in a cycle, which keeps the wiring and control surface simple but requires extra staging cycles.

Key components include:

- A 32-to-1 bus multiplexer selected through priority encode logic.
- Sixteen 32-bit general registers and the internal `PC`, `IR`, `MAR`, `MDR`, `Y`, `Z`, `HI`, and `LO` registers.
- A Carry Lookahead Adder for addition and subtraction.
- Logical operations, shifts, rotates, negation, and bitwise NOT.
- Booth bit-pair recoding multiplication with a 64-bit result split into `HI` and `LO`.
- Non-restoring division with quotient in `LO` and remainder in `HI`.
- Unit and datapath testbenches for the ALU, bus, registers, arithmetic, logical, shift, rotate, multiply, divide, and instruction-fetch style flows.

The main tradeoff in this phase is cycle count. Because the ALU cannot receive two bus values at once, one operand must be staged through `Y`, and the result is staged through `Z` before writeback.

## Phase 2 - Three-Bus Datapath

Phase 2 redesigns the datapath around three buses:

- **Bus A:** first ALU operand.
- **Bus B:** second ALU operand.
- **Bus C:** writeback path.

This allows two registers to be read while a result is written back, reducing the number of micro-operations for arithmetic, logical, and address-computation instructions. The three-bus approach removes the final design's dependency on the `Y` register and makes load/store address calculation cleaner.

Phase 2 also adds or expands:

- `select_encode` logic for `Gra`, `Grb`, and `Grc` register-field decoding.
- `BAout` handling so base-register addressing can use zero when `R0` is selected.
- `ram`, `mar`, `mdr`, `in_port`, and `out_port` modules.
- `con_ff_logic` for branch conditions.
- Instruction-specific testbenches for `ld`, `ldi`, `st`, `br`, `jal`, `jr`, `in`, `out`, `mfhi`, `mflo`, and selected ALU operations.

## Phase 3 - Control Unit

Phase 3 integrates the datapath with an FSM-based control unit and wraps the system in `processor.v`. The control unit performs fetch, decode, execute, memory, and writeback sequencing by generating the datapath control signals for each instruction.

The common fetch sequence is:

1. Put `PC` on Bus C, load `MAR`, increment `PC`, and load `Z`.
2. Write the incremented value back to `PC` while reading memory into `MDR`.
3. Move `MDR` into `IR`.

After fetch and decode, the FSM follows an instruction-specific state sequence. Register-register ALU operations take two execute/writeback states after fetch, load/store instructions take four, multiply/divide take three, and one-cycle writeback operations such as `mfhi`, `mflo`, `in`, and `out` are shorter.

Phase 3 also extends select/encode control so Bus A, Bus B, Bus C, and the register writeback path can independently choose `Ra`, `Rb`, or `Rc`. This lets Bus C be used as a true source bus as well as a writeback bus, which avoids extra routing through `MDR` for operations such as `jr` and `jal`.

## Performance Notes

The report evaluates the final design using a 10 ns simulation clock, corresponding to 100 MHz in simulation. This is not a hardware timing guarantee, but it provides a stable basis for functional testing.

Average CPI from the Phase 3 FSM:

| Instruction group | Average CPI |
| --- | --- |
| Arithmetic/logical | 5.00 |
| Load/store | 6.33 |
| Branch/jump | 5.00 |
| I/O | 4.00 |
| Miscellaneous | 3.50 |
| Overall | 4.96 |

Compared with the original single-bus sequencing, the three-bus design reduces the overall average CPI from about 5.64 to 4.96, mostly by removing temporary operand staging and combining parallel register reads with writeback.

## Running Simulations

The repository is organized around Verilog source and testbench files. Typical simulation flow:

1. Open the project or source files in ModelSim/Questa or another Verilog simulator.
2. Compile the modules for the phase being tested.
3. Compile the matching files in that phase's `testbenches/` directory.
4. Run the desired testbench, such as `Phase 3/testbenches/processor_tb.v`, and inspect the transcript/waveforms.

The Phase 3 processor testbench initializes memory with a broad instruction program, runs until `halt`, and prints final register and memory values.

## License

This project is released under the MIT License. See `LICENSE` for details.
