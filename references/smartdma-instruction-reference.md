# SmartDMA Instruction Reference

Use this as a compact selection guide. For exact macro spellings, inspect the project's `fsl_SMARTDMA_armclang.h`.

## Registers

| Register | Use |
|---|---|
| `R0`-`R7` | General-purpose registers. |
| `GPO` | SmartDMA GPIO output, or general-purpose if not mapped to pins. |
| `GPD` | SmartDMA GPIO direction, 1 output and 0 input. |
| `GPI` | SmartDMA GPIO input. |
| `CFS` | Bit-slice source configuration and low-bit status. |
| `CFM` | Bit-slice event configuration and result routing. |
| `SP` | Stack pointer; initialize before use. |
| `PC` | Program counter; normally two instructions ahead in the pipeline. |
| `RA` | Return address register. |

## Conditions

| Condition | Meaning |
|---|---|
| `EU` | Execute unconditionally. |
| `ZE`, `NZ` | Zero / non-zero. |
| `PO`, `NE` | Positive / negative. |
| `AZ`, `ZB` | Above zero / zero or below. |
| `CA`, `NC`, `CZ` | Carry set / carry not set / carry set and zero. |
| `SPO`/`UNS`, `SNE`/`NZS` | Algorithmic or scheduled-branch control conditions. |
| `BS`, `NBS` | Bit-slice satisfied / not satisfied. |
| `EX`, `NEX` | External flag set / clear. |

## Macro Naming

| Fragment | Meaning |
|---|---|
| `E_` | Unconditional macro. |
| `E_COND_` | Conditional macro; first argument is condition. |
| `S` | Set flags, except in `LDRB*` where `S` means signed byte. |
| `N` | Invert result. |
| `F`, `FEND`, `FBIT` | Byte-order or bit-order flip. |
| `_IMM` | Immediate operand. |
| `LSL`, `LSR`, `ASR`, `ROR` | Logical left, logical right, arithmetic right, rotate right. |
| `PRE`, `POST` | Pre/post pointer update. |
| `B` | Byte access. |
| `REG` | Register offset. |
| `LV`, `NRA`, `ACC` | Vectored hold large-vector, no-RA-update, accelerated variants. |

## Instruction Groups

| Group | Use |
|---|---|
| `E_MOV`, `E_MVN`, `E_LOAD_IMM`, `E_LOAD_SIMM` | Move register data or load 11-bit signed immediates, optionally shifted or inverted. |
| `E_ADD`, `E_SUB` | Add/subtract register or 12-bit signed immediate operands. Supports conditional, flag, invert, flip, and postshift variants. |
| `E_ADC`, `E_SBC` | Add/subtract with carry or borrow. Use for multiword arithmetic or carry-chain algorithms. |
| `E_AND`, `E_OR`, `E_XOR`, `E_ANDOR` | Boolean operations. `E_ANDOR` is useful for simultaneous clear/set bitfield updates. |
| `E_LSL`, `E_LSR`, `E_ASR`, `E_ROR` | Immediate preshift/prerotate combined with ALU operation. |
| `E_FEND`, `E_FBIT` | Byte-order or bit-order flip before shift and ALU. Useful for endian, display, and serial protocols. |
| `E_RLSL`, `E_RLSR`, `E_RASR`, `E_RROR` | Register-controlled shift/rotate using low 8 bits of the shift register. |
| `E_BTST`, `E_BSET`, `E_BCLR`, `E_BTOG` | Bit test, set, clear, toggle using register bit select or 5-bit immediate. |
| `E_MODIFY_GPO_BYTE` | Single-cycle modify of low byte of `GPO` using AND, OR, XOR masks. |
| `E_TIGHT_LOOP` | Hardware-assisted repeated block execution. The count is extra repetitions after first execution. |
| `E_HOLD` | Halt until Boolean or bit-slice pattern matches; vectored forms can branch by event. |
| `E_NOP` | No operation; useful for alignment and breakpoint-safe slots. |
| `E_HEART_RYTHM`, `E_SYNCH_ALL_TO_BEAT`, `E_WAIT_FOR_BEAT` | Heartbeat timing setup, global beat synchronization, and wait-for-beat delay. |
| `E_INT_TRIGGER` | Unconditional interrupt output to Arm/common interrupt and selected channels. |
| `E_GOTO`, `E_GOSUB` | Branch and subroutine branch. Consider toolchain limits and PC pipeline behavior. |
| `E_LDR`, `E_STR` | Memory/peripheral load/store with signed immediate offset; word/byte and pre/post variants. |
| `E_LDR_REG`, `E_STR_REG` | Load/store with register offset, useful for dynamic indexing. |
| `E_PER_READ`, `E_PER_WRITE` | Peripheral-region access with immediate offset from `0x40000000`. |
| `E_PUSH`, `E_POP` | Stack push/pop. SmartDMA stack grows upward. |

## High-Risk Rules

- Never assume `E_LDR` data is available with no latency if the next instruction consumes the destination.
- Do not use `SPO/SNE` with load/store/peripheral access instructions.
- Do not post-increment `PC`.
- Do not use `CFS` as byte-read destination.
- Do not set flags when the destination is `PC`; use the scheduled branch semantics deliberately.
- In GNU Arm GCC / GAS projects, avoid `E_GOTO(label)` and `E_COND_GOTO(cond,label)` with symbolic labels unless that exact macro/header/toolchain combination has already built. Prefer an `E_DCD(label)` literal branch table plus `E_LDR(PC, table_reg, offset)` / `E_COND_LDR(cond, PC, table_reg, offset)`.
