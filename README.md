# NOD8

NOD8 is a simple, regular 8-bit CPU designed to be understandable and buildable from discrete logic or 74xx parts. It keeps the datapath byte-wide throughout:

- 8-bit data and registers
- 8-bit program counter
- 8-bit address bus
- 256-byte unified address space
- fixed two-byte instructions
- no microcode and very little complex decoding

The design favors general primitives and software conventions over special-purpose hardware.

## Architectural registers

The three-bit destination/source register fields address eight architectural endpoints:

```text
000  A
001  B
010  C
011  D
100  E
101  F
110  FLAGS
111  PC
```

`A`–`D` are general-purpose registers. One may be used as a software-defined stack pointer, frame pointer, or other convention. There is no hardware stack increment/decrement logic.

## Instruction format

Primary byte:

```text
[ imm | cond.z | alu2 alu1 alu0 | dst2 dst1 dst0 ]
```

When `imm = 1`, the second byte is an 8-bit immediate:

```text
[ imm8 ]
```

When `imm = 0`, the second byte is the register-form operand:

```text
[ NW | LD | ST | spare | src2 src1 src0 ]
```

`NW` suppresses destination writeback. It supports register-form `CMP`, `TST`, harmless `MOV.NW` operations, and future extension markers.

`cond.z` predicates the instruction. A failed condition does not require a special shortened cycle: the instruction keeps its normal timing, but all architectural writes are suppressed.

`LD` routes memory[src] to B bus at t2
`ST` writes to memory[src] at t4

## ALU operations

```text
000  ADD
001  ADC
010  SUB
011  SBB
100  XOR
101  OR
110  AND
111  MOV
```

The bitmap is deliberately aligned with the hardware:

```text
alu2 = 0  arithmetic ripple-carry path
alu2 = 1  logic/output path
```

In arithmetic mode:

```text
alu1  selects add/subtract and B inversion
alu0  selects the carry source
```

In logic mode, `alu1:alu0` select the result source:

```text
00  XOR: carry-killed adder sum
01  OR:  dedicated OR circuit
10  AND: adder AND tap
11  MOV: direct B path
```

Thus the ALU is mainly an 8-bit ripple-carry adder, with a small OR circuit and a four-way output selection network around it.

## Datapath and timing

The two operand buses have fixed roles:

```text
BUS       → ALU B latch
BUS_2     → ALU A latch
ADDR      → memory address interface
```

Registers expose uniform output-enable classes:

```text
OE_BUS
OE_BUS2
OE_ADDR
```

PC drives `ADDR` during instruction fetch and can drive `BUS_2` when used as an ALU destination. MAR drives `ADDR` during memory access. The instruction latch is the intentional fixed-routing exception: it is connected directly to the decoder/fetch path.

A normal instruction follows one regular four-step schedule:

```text
1. Fetch IR_hi and advance PC
2. Fetch IR_lo and advance PC
3. Read operands and execute the ALU operation
4. Write back the result and capture flags
```

Condition failure and `NW` qualify the write enables rather than changing the timing schedule.

## Flags

The essential arithmetic flags are:

```text
CF  carry/borrow information
ZF  zero result
```

The eight-bit `FLAGS` register leaves room for system and user state such as `IE`, user flags, and potentially `HALT`. Flag storage uses master/slave latches so values commit cleanly at writeback. Halt behavior and its relationship to the clock and interrupt logic remain open design questions.

## Buses and backplane

The proposed 32-pin backplane exposes only signals that genuinely cross the CPU-board boundary:

| Pins | Signal |
| ---: | --- |
| 01–03 | `VCC`, `GND`, `CLK` |
| 04 | `RESET` |
| 05 | Reserved / possible `HALT_STATUS` |
| 06 | `IRQ` |
| 07 | `MEM_OE` |
| 08 | `MEM_WE` |
| 09–16 | `ADDR[7:0]` |
| 17–24 | `BUS[7:0]` |
| 25–32 | `BUS_2[7:0]` |

Opcode, operand fields, flags, ALU controls, condition logic, and sequencer details remain local to the CPU board.

## Memory and extensions

The 256-byte map is intentionally small but implementation-independent. A memory card may choose, for example:

```text
0xxxxxxx  ROM
1xxxxxxx  RAM and/or MMIO
```

A 16-byte DIP-switch ROM page naturally stores eight complete two-byte instructions: two 8-bit bytes per row. Larger ROM/RAM cards, external storage, copy/execute overlays, MMIO peripherals, and coprocessors can extend the system without changing the CPU core.

## Design philosophy

NOD8 is built around a small set of orthogonal primitives:

```text
regular byte-wide datapath
full-width immediates
universal predication
fixed two-byte instructions
software-defined stack and memory conventions
simple static decoders
specialized hardware kept external
```

The goal is not the smallest possible instruction set at any cost, but the smallest regular machine in which every remaining mechanism has a clear purpose.
