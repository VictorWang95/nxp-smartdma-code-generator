---
name: smartdma-code-generator
description: Use when generating NXP SmartDMA code, SmartDMA C macro functions, firmware-array workflows, GPIO state machines, MCU peripheral register access routines, keyscan/camera/display/SPI/QDC SmartDMA routines, or when adapting SmartDMA code for a specific MCU package, pinout, base address, SDK driver, and toolchain.
---

# SmartDMA Code Generator

## Overview

Generate SmartDMA code only after the target MCU, official SmartDMA Cookbook, official pinout, SmartDMA base address, peripheral register map, parameter ABI, and output form are confirmed. Prefer readable C macro source first; produce firmware arrays only through a verified build/export path. SmartDMA can operate pins directly through its `GPO`/`GPD`/`GPI` registers when a pin is muxed to `SMARTDMA_PIOx`; it can also read/write MCU peripheral registers such as USB, timers, PWM, serial modules, display/camera interfaces, GPIO registers, and FIFOs when direct SmartDMA I/O is not the requested or available mechanism.

For a complete MCU software project, SmartDMA work is an end-to-end flow: generate/integrate code, add or update a Makefile, compile with an Arm GCC toolchain found on the computer, flash/debug with NXP LinkServer after the build passes, and use Saleae Logic capture for IO-output waveforms after the user confirms the hardware connection.

User-facing alias: `smartdma_code_generator`. The installed skill name uses `smartdma-code-generator` to satisfy skill naming rules.

## Hard Gates

Do not generate GPIO/pin-dependent or peripheral-register-dependent SmartDMA code until all of these are true:

- Project integration mode is confirmed: either an existing complete MCU software project is available and SmartDMA files should be added into that project, or a standalone SmartDMA code folder should be generated.
- For complete MCU software projects, linker/scatter files have been located and checked before adding SmartDMA code. SmartDMA code must be placed in the fixed RAM execution region required by the target MCU/project, and the project must compile after linker and source changes.
- For complete MCU software projects, linker MEMORY regions must be cross-checked against the SDK-generated linker/scatter file or official memory map for that exact MCU/core/package. Do not infer usable core-local SRAM from total SRAM size; verify that the reset vector's initial SP lands inside a valid RAM range.
- For complete MCU software projects, the source file that contains `main()` has been located and updated with SmartDMA initialization/configuration before the project is considered integrated.
- For complete MCU software projects, a Makefile build path, Arm GCC compiler path, and LinkServer flash/debug path must be planned. Do not attempt LinkServer flashing until the Makefile build passes.
- For IO-output SmartDMA functions in a complete MCU project, the generated pin assignment must be reported to the user and explicit hardware connection confirmation must be received before starting any Saleae Logic capture.
- Before writing or editing any SmartDMA code, search/open the latest official NXP `AN14650: SmartDMA Cookbook` from `www.nxp.com` or `docs.nxp.com`, then re-check and study the sections relevant to the requested code. Do this every time; do not rely only on cached memory, previous sessions, or the local summary.
- Target MCU part number and package are known, for example `MCXN947VDF`.
- NXP official datasheet has been found on `www.nxp.com` or `docs.nxp.com`.
- Datasheet pinout/pin mux/pin description section has been checked for SmartDMA-capable pins or the relevant GPIO/FlexIO/camera pins.
- For NXP docs pages rendered by the Zoomin web client, do not stop at the visible shell page. Fetch the official backend topic API and parse `topic_html`, for example `https://nxp-docs-be-prod.nxp.com/api/bundle/<bundle-id>/page/<topic-path>`, then extract the exact pin row and ALT entries.
- NXP official reference manual has been found on `www.nxp.com` or `docs.nxp.com`.
- Reference manual memory map or SmartDMA chapter has been checked for the SmartDMA control-register base address and register offsets.
- TrustZone/secure-state behavior has been checked before choosing peripheral base addresses. If LinkServer or the startup path reports the core is in secure mode, confirm whether the build must use `-mcmse` so SDK headers select secure aliases, or explicitly justify any non-secure alias use.
- If the official docs do not explicitly show SmartDMA pin function, peripheral register map, or base address, state the uncertainty and cross-check SDK headers, device header, `pin_mux.c`, and `fsl_smartdma*.c` before generating code.
- If the official SmartDMA Cookbook cannot be reached, stop before code generation unless the user explicitly approves using a named local copy. State the gap in the final answer.

Do not generate a firmware byte/word array by guessing instruction encoding. Generate or update firmware arrays only from compiled SmartDMA code, existing SDK arrays, disassembly/objcopy output, or a user-provided verified conversion flow.

## Workflow

1. Confirm whether there is a complete MCU software project. If yes, identify the project root and whether SmartDMA code should be integrated into that project. If no, generate a standalone SmartDMA code folder.
2. Confirm the task type: keyscan, GPIO state machine, MCU peripheral register access, memory copy/format conversion, FlexIO/LCD, camera capture, SPI/serial protocol, USB helper, QDC, or other.
3. Confirm output form: C macro source, firmware array workflow, or both.
4. Confirm MCU part number, package, board, SDK/toolchain, execution address, and whether code must run from RAM.
5. Re-check the latest official NXP SmartDMA Cookbook (`AN14650`) on `www.nxp.com` or `docs.nxp.com`. Record the document ID, source URL/domain, and publication/revision/last-updated date when visible. Read the architecture, timing/heartbeat, hold/event, branch, stack, interrupt, peripheral access, GPIO, and instruction-set sections that affect the requested code.
6. Search NXP official sources for datasheet and reference manual. Use only official NXP sources for pinout/base-address claims unless the user explicitly provides trusted local docs. When a `docs.nxp.com` datasheet topic loads through the web client and the table is not in the first HTML response, call the official backend topic endpoint `https://nxp-docs-be-prod.nxp.com/api/bundle/<bundle-id>/page/<topic-path>`, parse the returned JSON `topic_html`, and extract the exact row for the pin. Record the exact ALT number from that row.
7. If integrating into a complete MCU project, locate linker/scatter files (`*.ld`, `*.icf`, `*.sct`, IDE generated memory/linker scripts, or project-specific linker files). Inspect existing MEMORY/region definitions and section placement before writing code.
8. Cross-check linker RAM and flash regions with an SDK example linker/scatter file for the same MCU/core. After linking, inspect the vector table or binary header to confirm the initial SP and reset PC are valid; an out-of-range SP can leave the target in boot ROM or stop before `main()`.
9. Determine the SmartDMA fixed RAM execution region from the reference manual, local SDK/project conventions, and existing linker memory map. Add or update the SmartDMA code section placement, for example the target project's `for_smartdma_RAM` or equivalent section, so SmartDMA code lands in the required RAM region. Do not place SmartDMA instruction code in flash unless the target documentation explicitly allows it.
10. Decide secure/non-secure addressing before writing Arm-side SmartDMA register access. Prefer SDK device macros such as `SMARTDMA0_BASE` with the correct compiler/security flags over hardcoded addresses; verify `PORTx`, `SMARTDMA0`, and any peripheral bases resolve to the intended secure or non-secure alias.
11. Build a small design before writing code: project integration mode, linker section/region plan, pin allocation, whether pins are driven by SmartDMA direct I/O (`SMARTDMA_PIOx` with `GPO`/`GPD`/`GPI`) or by MCU peripheral register access, register/base-address assumptions, peripheral register-access table when peripherals are touched, parameter structure, SmartDMA register allocation, main loop, Arm/SmartDMA handshake, interrupt/event routing behavior, direct register boot path, and verification plan.
12. For C macro source, use the target project's `fsl_SMARTDMA_armclang.h` when present; if it is missing, copy the bundled fallback from `references/fsl_SMARTDMA_armclang.h` into the generated file set. Follow the instruction rules in `references/smartdma-instruction-reference.md`.
13. For reusable code patterns and firmware-array export guidance, read `references/smartdma-code-patterns.md`.
14. Generate or update exactly these SmartDMA code files: `pin_mux.c`, `pin_mux.h`, `app_smartdma.c`, `app_smartdma.h`, and `fsl_SMARTDMA_armclang.h`. For a complete MCU project, place/update them in the corresponding project paths, usually board/pin-mux files under `board/` and SmartDMA source/header/macro files under `source/` unless the project uses different established directories. For standalone output, create a folder containing those five files.
15. For a complete MCU project, locate the source file that contains `main()` or the established application initialization hook. Add the SmartDMA integration there: `#include "app_smartdma.h"`, SmartDMA pin init call, target SmartDMA clock/reset enable, static stack/parameter/result buffers, `SMARTDMA_SetExternalFlag(0U)` when used, generated parameter initialization, `SMARTDMA_AppInit()`/`SMARTDMA_AppBoot()` or generated equivalents, and INPUTMUX/NVIC/software-trigger setup when the requested routine uses events or interrupts.
16. For a complete MCU project, add a Makefile when one is missing, or update the existing Makefile without breaking the IDE project. Search the computer for an Arm GCC toolchain, starting with `arm-none-eabi-gcc` in `PATH` and then common installation directories. Record the compiler path and version.
17. For a complete MCU project, modify the linker/scatter file as needed, run the Makefile build with the discovered Arm GCC toolchain, and fix integration/linker/source errors until the project compiles. If the build toolchain or command is unavailable, state that compilation could not be verified and provide the exact command the user must run.
18. For a complete MCU project whose build passes, search for NXP LinkServer (`LinkServer.exe` or platform equivalent), then flash/debug the generated image with LinkServer. Do not flash when the build fails, the target image is missing, or LinkServer is unavailable; report the blocker and exact command that should be run. Treat `flash verify` as a Flash-content check that may halt the target; use `flash load`, `run`, reset, or an explicit debug resume before Saleae capture.
19. If the SmartDMA function drives IO output, report the pin assignment table and Saleae channel plan to the user, then wait for explicit user confirmation that the hardware is connected before starting Saleae Logic capture. After confirmation, capture the waveform, compare it with the expected timing or protocol, and iterate on code/build/flash/capture until the waveform is correct or a clear hardware/tool blocker is found. For Saleae timing statistics, exclude long idle tails caused by LinkServer timeouts, halted targets, or capture windows after program exit; use stable-cycle median/period measurements for pass/fail.
20. After code generation, include integration notes: generated file layout, required SmartDMA macro header source, linker/scatter changes and SmartDMA RAM section, main-function integration, Makefile/Arm GCC build result, LinkServer flash/debug result, Saleae capture result when IO output is used, parameter struct, stack allocation, direct `CTRL`/`ARM2SMARTDMA`/`BOOT` register setup, optional software trigger, NVIC setup when used, and pin mux setup. Do not use SDK `fsl_smartdma*` drivers unless explicitly requested.

## Required Output Checklist

Every generated SmartDMA answer must include:

- SmartDMA Cookbook source checked for this code-generation turn, including official NXP source and visible document date/revision when available.
- Project integration mode: complete MCU software project path and destination paths, or standalone SmartDMA folder path.
- Linker/scatter file path, SmartDMA RAM execution region, section name, and before/after section placement when integrating into a complete MCU project.
- Linker MEMORY validation evidence: SDK/reference source used for flash/SRAM ranges, reset vector initial SP/reset PC values, and confirmation that SmartDMA code plus stack/parameter buffers are in valid RAM.
- Main-function integration summary when integrating into a complete MCU project: source file path, include added, pin init call, clock/reset call, buffers/parameter block, SmartDMA init/boot calls, event routing, NVIC setup, and trigger policy.
- Makefile integration summary for complete MCU projects: Makefile path, generated image target, Arm GCC compiler path/version, build command, and build result.
- LinkServer flash/debug summary for complete MCU projects after a passing build: LinkServer path/version when available, target probe/MCU selection, image path, flash/debug command, and result.
- Saleae Logic verification summary for IO-output SmartDMA functions: pin assignment shown to the user, user hardware connection confirmation, Saleae channel mapping, sample rate, capture duration, measured waveform, and pass/fail comparison against expected timing.
- Confirmed MCU/package and source of truth for pinout and base address.
- Confirmed security state and address aliases: whether the image runs secure or non-secure, compiler flags such as `-mcmse`, and the resolved SmartDMA/PORT base addresses.
- Pin assignment table when any pin/GPIO/FlexIO/camera signal is involved.
- Direct SmartDMA I/O plan when a pin is muxed as `SMARTDMA_PIOx`: SmartDMA PIO index, `GPD` direction bit, `GPO` output bit handling, pin mux function, and whether any GPIO peripheral register access is intentionally avoided.
- SmartDMA base address and key register names/offsets when register access is involved.
- Peripheral register-access table when MCU peripherals are touched: module, base, offset, final address, access width, bit masks, FIFO/status/clear side effects, and source.
- Event routing table when a peripheral event triggers SmartDMA, for example ADC interrupt/status through INPUTMUX to a SmartDMA input slice.
- Parameter structure passed through `ARM2SMARTDMA`, including alignment and low-bit mask handling.
- SmartDMA internal register allocation table (`R0`-`R7`, `RA`, `SP`, `PC`, `GPO/GPD/GPI`, `CFS/CFM` as used).
- Generated C macro function or a firmware-array generation procedure.
- Arm-side direct register boot/integration snippet that does not depend on SDK `fsl_smartdma*` drivers unless requested.
- `SMARTDMA_SetExternalFlag()` helper when shared RAM ownership, Arm-side critical sections, or external-flag handshake is used.
- Generated file list with destination paths for exactly these SmartDMA code files: `pin_mux.c`, `pin_mux.h`, `app_smartdma.c`, `app_smartdma.h`, and `fsl_SMARTDMA_armclang.h`. The macro header is sourced from the target project when available or from bundled fallback `references/fsl_SMARTDMA_armclang.h` when absent.
- Verification evidence, including complete MCU project build result when integrated, map/listing evidence that SmartDMA code is in the intended RAM region, GPIO waveform, interrupt callback, shared RAM consistency, and build/export checks.

## Code Generation Rules

- Initialize `SP` from the parameter block even if the first revision does not use `E_PUSH`/`E_POP`.
- Clear low control bits from `ARM2SMARTDMA` parameter pointers with shift or mask logic before dereferencing.
- Prefer explicit register allocation comments near the SmartDMA function.
- For direct pin output/input on pins with a confirmed `SMARTDMA_PIOx` mux, use `GPO`, `GPD`, and `GPI` inside the SmartDMA program instead of writing GPIO peripheral registers. Configure the Arm-side pin mux to the `SMARTDMA_PIOx` function, set the `GPD` direction bit for outputs, and update `GPO` with bit operations such as `E_BSET_IMM`, `E_BCLR_IMM`, or `E_BTOG_IMM`.
- Do not infer ALT numbers from generated pin_signal order, generated comments, or slash-separated signal strings. The `pin_signal` order can omit ALT numbers and mix alternate-function signals with non-mux metadata. Use the official Pinout/Signal Multiplexing table row, then map the exact ALT number to `kPORT_MuxAltN`.
- Do not generate a GPIO peripheral register-access table for simple direct SmartDMA pin output just because the physical pad also has a GPIO name. Use GPIO register writes only when the requested behavior explicitly requires the GPIO peripheral path or no direct `SMARTDMA_PIOx` function exists.
- Use `E_COND_*` forms only after identifying the flag-setting instruction that feeds the condition.
- Treat writes to `PC` as branch operations and account for the two-instruction-ahead PC pipeline behavior.
- For GNU Arm GCC / GAS builds, do not generate `E_GOTO(label)` or `E_COND_GOTO(cond, label)` with symbolic C labels. The bundled macro header encodes those forms by left-shifting the label expression, and GAS rejects section-relative symbols with errors such as `invalid operands (*UND* and *ABS* sections) for '<<'`. Use a literal branch table of `E_DCD(label)` entries and branch with `E_LDR(PC, table_reg, offset)` / `E_COND_LDR(cond, PC, table_reg, offset)` instead, or use only target-verified numeric branch operands.
- Use `E_INT_TRIGGER()` only for unconditional interrupt output; it has no conditional form.
- Guard shared RAM updates with an external flag or an explicit Arm/SmartDMA ownership protocol when both sides can access the same buffer.
- Include `SMARTDMA_SetExternalFlag(uint8_t flag)` in generated templates that use external-flag ownership. Preserve the protected `CTRL` write key and update only the documented external-flag bit.
- For MCU peripheral access, use `E_PER_READ`/`E_PER_WRITE` for supported compile-time addresses or a parameter-table of absolute register addresses with `E_LDR`/`E_STR`/`E_LDRB`/`E_STRB`. Confirm access width and side effects from the reference manual before choosing byte or word operations.
- Do not hardcode SmartDMA or PORT control-register base addresses when SDK device headers provide `SMARTDMA0_BASE`, `PORTx`, or secure/non-secure aliases. If hardcoding is unavoidable, document why the selected alias is correct for the target security state.
- When adding `-mcmse` or otherwise changing TrustZone/security compilation, rebuild every file that includes SDK device headers and verify that linker placement, vector table values, and generated instruction sections are unchanged except for intentional address aliases.
- For ADC/PWM/timer event-driven templates, use a generic two-file structure: header-defined parameter ABI, source-defined peripheral register constants, `E_VECTORED_HOLD` event wait, ADC FIFO/result validation, threshold table, conditional timer/PWM register updates, `SMARTDMA2ARM` completion notification, and direct init/start style boot helpers. Do not include local project names, local source file names, or local project paths in generated templates.
- Avoid mixing byte and word stack operations across non-word-aligned stack boundaries.
- For complete MCU projects, do not stop after editing source files. Update the linker/scatter placement for the SmartDMA code section, run the project build, and treat linker placement or compile failures as integration issues to fix before completion.
- For complete MCU projects, do not leave SmartDMA initialization only in `app_smartdma.c`. The generated API must be called from `main()` or the project's established application initialization path, and the build must prove that the main-function integration is valid.
- For complete MCU projects, do not stop at a successful compile. Flash/debug with LinkServer after the Makefile build passes unless hardware/tool access is unavailable and reported.
- If LinkServer attach shows the PC in boot ROM or RAM values do not match the ELF symbols, check reset vector SP/PC, boot configuration, and whether the previous LinkServer command halted the target before debugging SmartDMA logic.
- For IO-output SmartDMA functions, do not start Saleae Logic capture before telling the user the exact MCU package pins/board pins/signals and receiving explicit hardware connection confirmation.
- When validating waveforms with Saleae around LinkServer sessions, start capture before run if needed, but compute frequency from stable repeated edges only; do not average across startup gaps or final low/high idle intervals.

## References

- Before any SmartDMA code generation, search/open the latest official NXP `AN14650: SmartDMA Cookbook` on `www.nxp.com` or `docs.nxp.com` and study the relevant sections again.
- Read `references/smartdma-instruction-reference.md` when selecting instructions or explaining opcode behavior.
- Read `references/smartdma-code-patterns.md` when producing C macro source, pinout/base-address research steps, Arm integration snippets, or firmware-array workflows.
- Use `references/fsl_SMARTDMA_armclang.h` only as the bundled fallback SmartDMA macro header when the target project does not provide one.
