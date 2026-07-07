# NXP SmartDMA Code Generator Skill

This repository packages an AI agent skill for generating and integrating NXP SmartDMA code. The skill is intended for both Codex and Claude Code style skill loaders.

## What Is SmartDMA?

SmartDMA is an NXP programmable DMA engine that can execute compact instruction streams to move data, react to events, access peripheral registers, and directly drive or sample SmartDMA-capable pins through SmartDMA I/O registers. It is useful for timing-sensitive helper logic such as GPIO state machines, keyscan, camera/display data movement, serial-style bit handling, QDC helpers, and peripheral register workflows that benefit from hardware-side execution.

SmartDMA code generation is device-specific. Correct output depends on the target MCU, package pinout, SmartDMA base address, register map, linker placement, toolchain, and whether the generated code must be integrated into a complete MCU firmware project or emitted as a standalone SmartDMA file set.

## What This Skill Does

The `smartdma-code-generator` skill guides an AI coding agent through NXP SmartDMA code generation with explicit research, integration, and verification gates. It helps produce:

- SmartDMA C macro source for readable instruction-level programs.
- Firmware-array generation workflows that avoid guessed opcode encodings.
- Pin mux and direct SmartDMA I/O plans for `SMARTDMA_PIOx` pins.
- Peripheral register access plans with base address, offset, width, side-effect, and source tracking.
- Complete project integration guidance for linker placement, `main()` startup calls, Makefile builds, LinkServer flashing, and Saleae Logic waveform verification when hardware output is involved.
- A bundled fallback `fsl_SMARTDMA_armclang.h` macro header for projects that do not already provide one.

The skill intentionally requires official NXP document checks before generating pin-dependent, peripheral-dependent, or base-address-dependent code. It also requires build and hardware verification steps when the task is a complete MCU software integration.

## Package Layout

```text
.
|-- SKILL.md
|-- agents/
|   `-- openai.yaml
`-- references/
    |-- fsl_SMARTDMA_armclang.h
    |-- smartdma-code-patterns.md
    `-- smartdma-instruction-reference.md
```

- `SKILL.md` is the main skill entry point for Codex and Claude Code compatible skill loaders.
- `agents/openai.yaml` provides Codex/OpenAI UI-facing skill metadata.
- `references/smartdma-instruction-reference.md` is a compact SmartDMA instruction and macro selection guide.
- `references/smartdma-code-patterns.md` contains generation patterns, integration rules, and firmware-array workflow guidance.
- `references/fsl_SMARTDMA_armclang.h` is the fallback SmartDMA macro header used only when a target project does not provide one.

## Installation

Clone or copy this repository into the skill directory used by your coding agent as `smartdma-code-generator`.

For Codex:

```powershell
git clone https://github.com/VictorWang95/nxp-smartdma-code-generator.git "$env:USERPROFILE\.codex\skills\smartdma-code-generator"
```

For Claude Code:

```powershell
git clone https://github.com/VictorWang95/nxp-smartdma-code-generator.git "$env:USERPROFILE\.claude\skills\smartdma-code-generator"
```

If the repository is already cloned elsewhere, copy or sync the repository contents into one of these locations:

```text
%USERPROFILE%\.codex\skills\smartdma-code-generator
%USERPROFILE%\.claude\skills\smartdma-code-generator
```

Restart the coding agent or reload skills so the `smartdma-code-generator` metadata is discovered.

## Example Requests

```text
Generate SmartDMA GPIO state-machine code for MCXN947VDF using direct SMARTDMA_PIO output.
```

```text
Integrate a SmartDMA keyscan routine into this MCUXpresso SDK project and build it with Arm GCC.
```

```text
Create a SmartDMA firmware-array workflow for this existing C macro source.
```

## Notes

This skill is designed for AI-assisted engineering work. It does not replace checking the official NXP SmartDMA Cookbook, datasheet, reference manual, SDK headers, linker scripts, or board hardware setup for the exact MCU and package being used.
