# SmartDMA Code Patterns

## Official Document Research Gate

For MCU-specific code, search official NXP sources before generating pin or base-address dependent code:

1. Search `www.nxp.com` / `docs.nxp.com` for the exact MCU part number plus `datasheet`.
2. Open the datasheet and inspect `Pinout`, `Pin muxing`, `Pin description`, and `Signal multiplexing`.
   - If `docs.nxp.com` renders only the web-client shell, fetch the official backend topic API directly: `https://nxp-docs-be-prod.nxp.com/api/bundle/<bundle-id>/page/<topic-path>`.
   - Parse the JSON `topic_html` field and extract the exact table row for the target pin.
   - Do not infer ALT numbers from generated pin_signal order, generated comments, or slash-separated signal strings. Use the official row's `ALTn - SIGNAL` entries.
3. Search `www.nxp.com` / `docs.nxp.com` for the exact MCU family plus `reference manual`.
4. Open the reference manual and inspect memory map, SmartDMA chapter, reset/clock/inputmux chapters, and interrupt table.
5. Cross-check local device files: device header, `fsl_device_registers.h`, `*_features.h`, generated `pin_mux.c`, linker/scatter files, and local SmartDMA macro headers. Use SDK SmartDMA driver files only as register-definition references, not as the generated code structure.

Record findings before code:

| Item | Value | Source |
|---|---|---|
| MCU/package |  | datasheet |
| SmartDMA base |  | reference manual / SDK header |
| IRQ name |  | reference manual / SDK header |
| Clock/reset IDs |  | reference manual / SDK header |
| Signal | Pin | exact ALT / mux function / source |

## Preferred Project Structure

Use a generic, self-contained SmartDMA project structure for generated examples. Generated templates must use generic file names and placeholders; do not embed local project names, local source file names, local project paths, or workspace-specific directories.

```text
<destination>/
  pin_mux.c                 // SmartDMA/GPIO pin mux verified from official pinout
  pin_mux.h                 // pin mux API and signal documentation
  app_smartdma.c            // SmartDMA macro function and direct register boot helpers
  app_smartdma.h            // parameter ABI, command/status values, API prototypes
  fsl_SMARTDMA_armclang.h   // required macro instruction header, target-provided or bundled fallback
```

Integration placement:

- If a complete MCU software project exists, add or update `pin_mux.c` and `pin_mux.h` in the project's established board/pin-mux directory, usually `board/`.
- Add or update `app_smartdma.c`, `app_smartdma.h`, and `fsl_SMARTDMA_armclang.h` in the project's established application source directory, usually `source/`.
- Locate the source file that contains `main()` and add the generated SmartDMA initialization/configuration calls there, or in the project's established application initialization hook if that is where board-level features are started.
- Locate the project linker/scatter files before adding code. Common files include GNU LD scripts (`*.ld`), IAR linker configs (`*.icf`), Arm scatter files (`*.sct`), and IDE-generated memory/linker files.
- Add or update the SmartDMA code section so the SmartDMA instruction function is linked into the fixed RAM region required by the target MCU/project. Use the section name already present in the project when available; otherwise define a clear section such as `for_smartdma_RAM` and place it in the confirmed SmartDMA executable RAM region.
- Add a Makefile when one is missing, or update the existing Makefile while preserving the IDE project files. Search for `arm-none-eabi-gcc` in `PATH` and common toolchain installation directories, then build with the discovered Arm GCC compiler.
- After source, linker, main-function, and Makefile changes, run the complete project build and fix compile/link errors. The integration is not complete until the project builds or the unavailable build toolchain is explicitly reported with the exact command the user should run.
- After a passing build, search for NXP LinkServer and use it to flash/debug the generated image. Do not flash a stale or failed-build image.
- If the function drives IO output, report the exact MCU/board pins and Saleae channel plan, wait for user hardware connection confirmation, then capture and verify the waveform with Saleae Logic.
- If the project uses different established directories, follow the existing project layout and report the chosen destination paths.
- If no complete MCU software project is available, create a standalone folder containing the five files listed above.
- Do not silently generate only a partial set. If one of the five files is intentionally omitted because the user explicitly requested a different integration style, state that deviation.

Generated code must be self-contained: define the SmartDMA register base, `ARM2SMARTDMA`, `SMARTDMA2ARM`, control-register type pointer, direct-I/O pin mapping when using `SMARTDMA_PIOx`, target peripheral register addresses when touching MCU peripherals, handshake helpers, boot helper, optional software trigger, and external-flag helper in the generated source/header pair. Keep the parameter ABI in the header and keep the SmartDMA macro function plus direct boot helpers in the source. Do not require `fsl_smartdma.h`, `fsl_smartdma.c`, `fsl_smartdma_mcxn.h`, `SMARTDMA_InitWithoutFirmware()`, `SMARTDMA_Boot_API()`, `SMARTDMA_InstallFirmware()`, or `SMARTDMA_Boot()` unless the user explicitly asks for SDK-driver integration.

Every generated SmartDMA file set must include or explicitly copy `fsl_SMARTDMA_armclang.h`; the SmartDMA source must include it directly:

```c
#include "fsl_SMARTDMA_armclang.h"
```

Header sourcing rule:

1. Prefer the target project's existing `fsl_SMARTDMA_armclang.h`.
2. If the target project does not contain this macro header, copy the bundled fallback from this skill: `references/fsl_SMARTDMA_armclang.h`.
3. Do not invent SmartDMA macro definitions from memory. If the target toolchain needs a different macro-header name, keep the generated source include and copied file name consistent and state the rename in integration notes.

## C Macro Function Skeleton

Use this shape for readable SmartDMA source:

```c
#include "fsl_SMARTDMA_armclang.h"
#include "<app>_smartdma.h"

#define SMARTDMA_CON_BASE 0x40033000U
#define ARM2SMARTDMA     (SMARTDMA_CON_BASE + 0x40U)
#define SMARTDMA2ARM     (SMARTDMA_CON_BASE + 0x44U)
#define SMARTDMA0        ((SMARTDMA_REG_Type *)SMARTDMA_CON_BASE)

typedef struct _smartdma_example_param
{
    uint32_t *smartdma_stack;
    uint32_t *p_gpio_reg;      /* Optional: only for GPIO peripheral-register access. */
    uint32_t *p_periph_reg;    /* Optional: only when touching MCU peripheral registers. */
    uint32_t *p_result;
    uint32_t *p_interval;
} smartdma_example_param_t;

void __attribute__((section("for_smartdma_RAM"))) Smartdma_example(void)
{
    /* Register allocation:
     * R0: temporary / input value
     * R1: output value
     * R2: interval
     * R5: optional peripheral register-address table pointer
     * R6: parameter pointer, then optional GPIO/direct-I/O pointer
     * R7: result buffer pointer
     * RA: scratch address
     */

    E_NOP;
    E_NOP;

    E_PER_READ(R6, ARM2SMARTDMA);
    E_LSR(R6, R6, 2);
    E_LSL(R6, R6, 2);

    E_LDR(SP, R6, 0);
    E_LDR(R5, R6, 2);
    E_LDR(R7, R6, 3);
    E_LDR(R2, R6, 4);
    E_LDR(R2, R2, 0);
    E_HEART_RYTHM(R2);
    E_LDR(R6, R6, 1);

    E_LABEL("loop");
    /* Body goes here. */
    E_INT_TRIGGER(1);
    E_GOSUB(loop);
}
```

Use the target project's existing section name when one already exists. Otherwise prefer a clear section such as `for_smartdma_RAM` and document the linker/scatter placement. The map/listing must show the SmartDMA function at the intended SmartDMA executable memory, for example RAMX on MCX/LPC parts.

## GNU Assembler Branch Table Pattern

Use this pattern when compiling SmartDMA C macro source with GNU Arm GCC / GAS and the code needs multiple symbolic branches. Avoid `E_GOTO(label)` and `E_COND_GOTO(cond, label)` with symbolic labels because the bundled macro header expands them as label expressions shifted left, for example `label << 9`; GAS cannot shift section-relative symbols and fails with `invalid operands (*UND* and *ABS* sections) for '<<'`.

Preferred pattern:

```c
#define BR_DONE      0
#define BR_ERROR     1
#define BR_LOOP      2
#define BR_WORDS     3
#define BR_BACK      ((BR_WORDS * 4) + 8)

#define SMARTDMA_BRANCH(offset)          E_LDR(PC, RA, offset)
#define SMARTDMA_COND_BRANCH(cond, off)  E_COND_LDR(cond, PC, RA, off)

void __attribute__((naked, section("for_smartdma_RAM"))) Smartdma_example(void)
{
    E_NOP;
    E_NOP;
    E_GOSUB(entry);
    E_DCD(done);
    E_DCD(error);
    E_DCD(loop);

    E_LABEL("entry");
    E_ADD_IMM(RA, PC, -BR_BACK);

    E_LABEL("loop");
    /* Set flags with an instruction such as E_SUBS/E_BTST_IMMS first. */
    SMARTDMA_COND_BRANCH(ZE, BR_DONE);
    SMARTDMA_COND_BRANCH(NZ, BR_ERROR);
    SMARTDMA_BRANCH(BR_LOOP);

    E_LABEL("error");
    /* Error cleanup. */

    E_LABEL("done");
    E_INT_TRIGGER(1);
    E_HOLD;
}
```

Rules:

- Initialize the table base register once after entering the executable body. The example uses `RA`.
- Keep offsets as word indexes because `E_LDR(..., offset8s)` uses word offsets for word loads in the existing macro header.
- Recompute `BR_BACK` when adding or removing branch table entries.
- Only use `E_COND_LDR(cond, PC, RA, offset)` after an instruction that intentionally sets the condition flags.
- Run the target Makefile after introducing branches; host source-contract tests do not prove SmartDMA macro expressions are accepted by GAS.

## Main Function Integration Pattern

When a complete MCU software project is available, generated SmartDMA files are not integrated until the project entry path calls them. Locate `main()` with project search, then add the target-specific initialization in the same place the project starts clocks, pins, peripherals, and application services.

Procedure:

1. Search the project for `main()` and identify the active source file used by the build configuration.
2. Add `#include "app_smartdma.h"` to that file.
3. Add static SmartDMA stack, parameter, result/debug buffers, direct-I/O timing/pin parameters, and any required peripheral register-address tables near the existing application globals.
4. In the initialization flow, after base board clocks are available and before the first SmartDMA trigger, call the SmartDMA pin mux function, enable the target SmartDMA clock/reset, initialize the external flag when used, fill the parameter block, then call the generated init and boot helpers.
5. Add INPUTMUX/event routing, NVIC enable/disable policy, and software-trigger setup in the same file when the SmartDMA routine depends on peripheral events or Arm interrupts.
6. Build the complete MCU project and inspect compiler/linker output plus the map/listing.

Generic sketch; replace clock IDs, buffer sizes, parameter fields, and event routing with target-verified values:

```c
#include "app_smartdma.h"

static uint32_t s_smartdma_stack[32];
static app_smartdma_param_t s_smartdma_param;
static uint32_t s_smartdma_result[8];

int main(void)
{
    BOARD_InitBootPins();
    BOARD_InitBootClocks();
    BOARD_InitDebugConsole();

    BOARD_InitSmartdmaPins();
    CLOCK_EnableClock(kCLOCK_Smartdma);

    SMARTDMA_SetExternalFlag(0U);
    s_smartdma_param.smartdma_stack = s_smartdma_stack;
    s_smartdma_param.p_result = s_smartdma_result;
    /* Fill direct-I/O timing values or peripheral register tables here. */

    SMARTDMA_AppInit(&s_smartdma_param);
    SMARTDMA_AppBoot(Smartdma_App);

    /* Configure INPUTMUX, NVIC, and software trigger when the routine uses them. */

    while (1)
    {
    }
}
```

## Linker Placement Pattern

SmartDMA instruction code runs from a fixed RAM execution region on supported MCU projects. In complete MCU software projects, always update linker/scatter placement together with the SmartDMA source files.

Procedure:

1. Locate all project linker/scatter inputs and generated linker files: `*.ld`, `*.icf`, `*.sct`, memory XML files, IDE managed linker fragments, and map/listing outputs.
2. Identify the RAM region that is valid for SmartDMA instruction fetch using the reference manual, project memory map, startup files, existing section names, and SDK/linker conventions.
3. Put the SmartDMA macro function section into that RAM region. Keep section names consistent between C attributes and linker placement.
4. Preserve existing reset vector, stack, heap, noncacheable, RAMX, and data-copy sections. Do not repurpose unrelated RAM regions without proving there is space and no overlap.
5. Build the complete project after modifying source and linker files.
6. Inspect the map/listing to confirm the SmartDMA function and related section are in the intended RAM region.

Examples of section naming only; adapt syntax to the target linker:

```c
void __attribute__((section("for_smartdma_RAM"))) Smartdma_example(void)
{
    /* SmartDMA instructions. */
}
```

```ld
/* GNU ld style sketch: use the project's actual RAM region name. */
.smartdma_code :
{
    KEEP(*(.for_smartdma_RAM))
    KEEP(*(for_smartdma_RAM))
} > SMARTDMA_RAM
```

```text
; IAR/Arm scatter style sketch: use the project's actual syntax and region name.
place in SMARTDMA_RAM { section for_smartdma_RAM };
```

## Makefile, Arm GCC, and LinkServer Flow

For complete MCU projects, add the build and flash path after SmartDMA code integration.

Procedure:

1. Detect the existing build layout: SDK make project, IDE-generated project, CMake project, or source-only project.
2. Add a Makefile if none exists. If a Makefile exists, update it narrowly for the generated SmartDMA sources, include paths, linker script, startup file, system file, libraries, and output image.
3. Search for Arm GCC in this order: `Get-Command arm-none-eabi-gcc`, project-local toolchain references, common NXP/MCUXpresso/GNU Arm Embedded install directories, then broader computer path search if needed.
4. Record `arm-none-eabi-gcc --version`, `arm-none-eabi-size`, and the exact `make` command.
5. Build from the project root. Treat missing includes, wrong startup/system files, linker section placement, and undefined SmartDMA symbols as integration issues to fix.
6. Inspect the generated ELF/AXF, map file, binary/hex file, and size output.
7. Search for LinkServer in `PATH` and common NXP installation directories. Use the detected image from the passing build for flash/debug.

Generic command shape; adapt executable names and target options to the installed tools and MCU:

```powershell
Get-Command arm-none-eabi-gcc
arm-none-eabi-gcc --version
make -C <project-root> clean all
Get-Command LinkServer.exe
LinkServer.exe <target-and-probe-options> <flash-or-debug-command> <image-path>
```

Never run LinkServer flashing when the Makefile build failed, the image path is not from the latest passing build, or the target/probe selection is uncertain.

## Saleae Logic IO Verification Gate

Use Saleae Logic only for SmartDMA functions that drive or toggle IO output signals. Before starting capture, stop and send the user the exact connection information:

| Signal | MCU package pin | Board connector/test point | Saleae channel | Expected waveform |
|---|---|---|---|---|
|  |  |  |  |  |

Wait for explicit user hardware connection confirmation. After confirmation:

1. Open or connect to Saleae Logic using the available local tool/API.
2. Set sample rate and capture duration high enough for the expected frequency, duty cycle, pulse width, or protocol timing.
3. Flash/debug the latest passing build if it has not already been loaded.
4. Capture the waveform and measure frequency, duty cycle, pulse count, latency, or protocol fields.
5. If the measured waveform is wrong, iterate through code changes, Makefile build, LinkServer flash/debug, and Saleae capture. Record each blocker separately as code, build, flash, probe, wiring, or measurement setup.

Do not ask the user to connect hardware after starting capture, and do not claim IO-output verification from firmware logs alone.

## GPIO Scan Pattern

Use this for matrix scan or state machine code:

1. Load GPIO register address from a table.
2. Write output state with `E_STRB` or `E_STR`.
3. Read input with `E_LDRB` or `E_LDR`.
4. Test pin bit with `E_BTST_IMMS`.
5. Conditionally accumulate bitmap with `E_COND_OR` or `E_COND_LSL_OR`.
6. Compare with previous shared result using `E_XORS`.
7. Write changed result and call `E_INT_TRIGGER`.

## MCU Peripheral Access Pattern

SmartDMA templates must support both GPIO pin operations and generic MCU peripheral register access. Use this for USB/LPSPI/LPI2C/I3C/CTIMER/PWM/ADC/FlexIO/display/camera style routines where SmartDMA reads status registers, writes control/data/FIFO registers, clears flags, or updates peripheral state machines.

Before writing code that touches a peripheral, record a register-access table:

| Module | Base | Register | Offset | Address | Width | Access | Bits/masks | Side effects | Source |
|---|---:|---|---:|---:|---|---|---|---|---|
|  |  |  |  |  | 8/16/32 | R/W/RMW/W1C |  | FIFO, W1C, read-clear, busy flag | RM + `PERI_<MODULE>.h` |

Use the reference manual as the source of truth for register semantics, then cross-check local device headers such as `PERI_USB.h`, `PERI_LPSPI.h`, `PERI_CTIMER.h`, `PERI_PWM.h`, `PERI_GPIO.h`, and `fsl_device_registers.h` for offsets and masks. Do not invent offsets or use SDK driver calls inside SmartDMA firmware. SDK driver files can be read only as explanatory references.

Two access styles are supported.

### Compile-Time Peripheral Address

Use `E_PER_READ` / `E_PER_WRITE` when the target `fsl_SMARTDMA_armclang.h` supports the target peripheral address form and the register address is a compile-time constant:

```c
#define PERIPH_CTRL_ADDR        (PERIPH_BASE + 0x00U)
#define PERIPH_STAT_ADDR        (PERIPH_BASE + 0x04U)
#define PERIPH_DATA_ADDR        (PERIPH_BASE + 0x08U)
#define PERIPH_ENABLE_BIT       0U
#define PERIPH_READY_BIT        3U
#define offset_wait_ready       0

/* Literal branch table for conditional PC loads. */
E_MOV(R4, PC);
E_NOP;
E_DCD(wait_ready);

/* Read-modify-write a control bit. */
E_PER_READ(R0, PERIPH_CTRL_ADDR);
E_BSET_IMM(R0, R0, PERIPH_ENABLE_BIT);
E_PER_WRITE(R0, PERIPH_CTRL_ADDR);

/* Poll a status bit before writing a data/FIFO register. */
E_LABEL("wait_ready");
E_PER_READ(R0, PERIPH_STAT_ADDR);
E_BTST_IMMS(R0, R0, PERIPH_READY_BIT);
E_COND_LDR(ZE, PC, R4, offset_wait_ready);
E_PER_WRITE(R1, PERIPH_DATA_ADDR);
```

For field masks wider than the immediate range, load constants from the parameter block or a literal table before the `E_AND`/`E_OR`/`E_XOR` operation. For write-one-to-clear flags, write only the documented flag mask; do not write back a whole status register unless the reference manual explicitly allows it.

### Runtime Register-Address Table

Use a parameter-table register map when register addresses are package/instance dependent, when the address form is not accepted by `E_PER_READ`/`E_PER_WRITE`, or when Arm must prepare absolute register addresses for SmartDMA at runtime. This is the preferred generic template because it also works for GPIO byte registers, peripheral FIFOs, status registers, and control registers.

```c
typedef struct _smartdma_periph_param
{
    uint32_t *smartdma_stack;
    uint32_t *p_gpio_reg;      /* Optional GPIO absolute register addresses. */
    uint32_t *p_periph_reg;    /* Absolute register addresses for the peripheral module. */
    uint32_t *p_result;
    uint32_t *p_interval;
} smartdma_periph_param_t;

enum
{
    PERIPH_REG_CTRL = 0,
    PERIPH_REG_STAT = 1,
    PERIPH_REG_DATA = 2,
    PERIPH_REG_FLAG = 3,
};

/* Arm side: fill with exact addresses from RM/device headers. */
static uint32_t s_periph_reg[] =
{
    PERIPH_BASE + PERIPH_CTRL_OFFSET,
    PERIPH_BASE + PERIPH_STAT_OFFSET,
    PERIPH_BASE + PERIPH_DATA_OFFSET,
    PERIPH_BASE + PERIPH_FLAG_OFFSET,
};

/* SmartDMA side: R5 is p_periph_reg, RA is scratch address. */
E_LDR(R5, R6, 2);

/* 32-bit read-modify-write control register. */
E_LDR(RA, R5, PERIPH_REG_CTRL);
E_LDR(R0, RA, 0);
E_BSET_IMM(R0, R0, PERIPH_ENABLE_BIT);
E_STR(RA, R0, 0);

/* 32-bit status poll. */
E_LDR(RA, R5, PERIPH_REG_STAT);
E_LDR(R0, RA, 0);
E_BTST_IMMS(R0, R0, PERIPH_READY_BIT);

/* 8-bit or 32-bit data/FIFO write, depending on the RM access width. */
E_LDR(RA, R5, PERIPH_REG_DATA);
E_STRB(RA, R1, 0); /* Use E_STR instead when the register requires 32-bit access. */

/* Write-one-to-clear flag. */
E_LDR(RA, R5, PERIPH_REG_FLAG);
E_LOAD_IMM(R0, PERIPH_DONE_MASK);
E_STR(RA, R0, 0);
```

When generating a concrete peripheral template, include the exact peripheral instance, base address, offsets, access widths, flag-clear behavior, FIFO behavior, clock/reset dependency, and whether Arm or SmartDMA owns each register at runtime. If both Arm and SmartDMA can touch the same peripheral registers, define an ownership/handshake rule just like shared RAM.

## ADC/PWM Event-Driven Peripheral Template

For ADC-threshold-to-PWM/timer-control tasks, use a generic two-file split: one header for the parameter ABI and public API, and one source file for the SmartDMA macro function, direct register constants, and boot/control helpers. Adapt names, register bases, offsets, and the macro header to the target MCU. Do not include reference project names, reference source file names, or reference paths in generated templates. Keep the generated macro header as `fsl_SMARTDMA_armclang.h`; translate legacy macro-header usage to the target SmartDMA macro header.

Use this structure:

- Header owns the parameter ABI, for example stack pointer, debug/result buffer, optional ADC result buffer, threshold table, and optional peripheral register table.
- Source owns all direct register constants: SmartDMA control base, `ARM2SMARTDMA`, `SMARTDMA2ARM`, ADC result/FIFO register, timer/PWM control register, timer/PWM match/reload register, and any event/interrupt routing register used by the SmartDMA trigger.
- SmartDMA function is placed in the target execution section, for example `for_pwm_engine`, `for_smartdma_RAM`, or the project linker section.
- Function startup reads `ARM2SMARTDMA`, clears low control bits, loads `SP`, loads result/debug and threshold pointers, then builds a literal branch table with label offsets.
- Event wait uses `E_VECTORED_HOLD(PC)` with documented CFS/CFM bit-slice routing when the peripheral interrupt/event is routed to SmartDMA.
- ADC/result handling polls or validates the documented FIFO/result valid bit, extracts command/channel ID if needed, extracts the conversion result, stores debug/result values only under a clear ownership rule, and compares against threshold entries from the parameter table.
- PWM/timer update performs conditional `E_COND_PER_READ`/`E_COND_PER_WRITE` or parameter-table `E_LDR`/`E_STR` operations to enable/disable the timer and update match/reload registers.
- Completion writes `SMARTDMA2ARM` or calls `E_INT_TRIGGER()` only after the peripheral update is complete.
- Arm-side `main.c` configures clocks, pins, ADC trigger, timer/PWM output, inputmux/event routing, SmartDMA clock, parameter block, external flag initial state, direct boot, and IRQ enable/disable policy.

Minimal event-flow skeleton:

```c
typedef struct _smartdma_adc_pwm_param
{
    void *smartdma_stack;
    uint32_t *p_debug;
    uint32_t *p_adc_result;
    uint32_t *p_threshold;
    uint32_t *p_periph_reg;
} smartdma_adc_pwm_param_t;

#define SMARTDMA_CON_BASE      0x40033000U
#define ARM2SMARTDMA           (SMARTDMA_CON_BASE + 0x40U)
#define SMARTDMA2ARM           (SMARTDMA_CON_BASE + 0x44U)
#define ADC_RESFIFO0_ADDR      (ADC0_BASE + 0x300U)
#define PWM_TCR_ADDR           (PWM_TIMER_BASE + 0x004U)
#define PWM_MATCH_ADDR         (PWM_TIMER_BASE + 0x07CU)

#define offset_start           0
#define offset_valid           1
#define offset_done            2
#define offset_cfs_value       3
#define offset_cfm_value       4

void __attribute__((section("for_pwm_engine"))) Smartdma_adc_pwm_engine(void)
{
    E_PER_READ(R6, ARM2SMARTDMA);
    E_LSR(R6, R6, 2);
    E_LSL(R6, R6, 2);
    E_LDR(SP, R6, 0);
    E_LDR(R7, R6, 1); /* debug/result buffer */

    E_ADD_IMM(R4, PC, -4);
    E_DCD(start);
    E_DCD(valid);
    E_DCD(done);
    E_DCD_VAL(BS7(0) | BS6(0) | BS5(0) | BS4(0) | BS3(0) | BS2(0) | BS1(0) | BS0(0));
    E_DCD_VAL(BS7(6) | BS6(6) | BS5(6) | BS4(6) | BS3(6) | BS2(6) | BS1(6) | BS0(BS_RISE) | (1U << 0));
    E_LDR(CFS, R4, offset_cfs_value);
    E_LDR(CFM, R4, offset_cfm_value);

E_LABEL("start");
    E_VECTORED_HOLD(PC);
    E_NOP;
    E_NOP;
    E_LDR(PC, R4, offset_valid);

E_LABEL("valid");
    E_BSET_IMM(CFM, CFM, 0);
    E_PER_READ(R1, ADC_RESFIFO0_ADDR);
    E_LSRS(R0, R1, 31);
    E_COND_LDR(ZE, PC, R4, offset_valid);

    E_LSL(R0, R1, 16);
    E_LSR(R0, R0, 16);
    /* Compare R0 with threshold table entries and conditionally update PWM/timer. */
    E_COND_PER_READ(AZ, R2, PWM_TCR_ADDR);
    E_COND_BSET_IMM(AZ, R2, R2, 0);
    E_COND_PER_WRITE(AZ, R2, PWM_TCR_ADDR);
    E_COND_PER_WRITE(AZ, R3, PWM_MATCH_ADDR);

E_LABEL("done");
    E_PER_WRITE(R0, SMARTDMA2ARM);
    E_GOSUB(start);
}
```

For each generated ADC/PWM instance, state how the event reaches SmartDMA, for example INPUTMUX route from ADC interrupt/status to a SmartDMA input slice, and whether the Arm ADC IRQ is disabled so the event is consumed by SmartDMA instead of the Arm core.

## Shared RAM Ownership Pattern

Use an external flag or equivalent protocol before SmartDMA writes shared result memory:

- Arm sets external flag before reading/updating shared RAM.
- SmartDMA checks `EX`; if set, branch back to loop or skip shared RAM writes.
- Arm clears external flag when done.
- SmartDMA writes result and triggers interrupt only when it owns the buffer.

Example branch shape:

```c
E_COND_LDR(EX, PC, R4, offset_start);
```

## Direct Register Boot Pattern

Use direct register access by default. Keep all SmartDMA boot/control helpers in the generated SmartDMA source file:

```c
void SMARTDMA_cfgHandshake(bool enable_handshake, bool enable_event)
{
    uint32_t handshake = enable_handshake ? 1U : 0U;
    uint32_t event = enable_event ? 1U : 0U;
    SMARTDMA0->ARM2SMARTDMA |= (handshake << SMARTDMA_HANDSHAKE_ENABLE) |
                                (event << SMARTDMA_HANDSHAKE_EVENT);
}

void SMARTDMA_AppInit(void *pParam)
{
    SMARTDMA0->CTRL |= 0xC0DE0000U | (1U << SMARTDMA_ENABLE_GPISYNCH);
    SMARTDMA0->ARM2SMARTDMA = (uint32_t)pParam;
    SMARTDMA_cfgHandshake(true, false);
}

void SMARTDMA_AppBoot(void *pProgram)
{
    SMARTDMA0->BOOT = (uint32_t)pProgram;
    SMARTDMA0->CTRL = 0xC0DE0011U |
                      (0U << SMARTDMA_MASK_RESP) |
                      (0U << SMARTDMA_ENABLE_AHBBUF);
}

void SMARTDMA_SetExternalFlag(uint8_t flag)
{
    uint32_t ctrl = SMARTDMA0->CTRL & 0x0000FFFFU;

    if (flag == 0U)
    {
        ctrl &= ~(1U << SMARTDMA_EXTERNAL_FLAG);
    }
    else
    {
        ctrl |= (1U << SMARTDMA_EXTERNAL_FLAG);
    }

    SMARTDMA0->CTRL = 0xC0DE0000U | ctrl;
}
```

Arm-side `main.c` should allocate the parameter block and stack, enable the SmartDMA clock/reset required by the target, configure pins, then call the direct helpers:

```c
static uint32_t s_smartdma_stack[32];
static smartdma_example_param_t s_param;

CLOCK_EnableClock(kCLOCK_Smartdma);
BOARD_InitSmartdmaPins();
SMARTDMA_SetExternalFlag(0U);
s_param.smartdma_stack = s_smartdma_stack;
s_param.p_result = result_buffer;
s_param.p_interval = &interval_cycles;
/* Fill either direct-I/O parameters such as PIO index/timing or peripheral register tables as required. */
SMARTDMA_AppInit(&s_param);
SMARTDMA_AppBoot(Smartdma_example);
```

When shared RAM or shared peripheral state is read by Arm while SmartDMA can write it, use `SMARTDMA_SetExternalFlag(1U)` before the Arm critical section and `SMARTDMA_SetExternalFlag(0U)` after the Arm critical section. The target reference manual and local macro header must confirm the external-flag bit name and protected `CTRL` write key; if the target uses different register/member names, adapt the helper without changing the ownership protocol.

## SmartDMA Direct I/O Pattern

Use this when the requested signal can be muxed to `SMARTDMA_PIOx` and the SmartDMA program should directly drive or sample a pin. Do not route this through `GPIOx->PDOR/PSOR/PCOR/PTOR` unless the user explicitly asks for GPIO peripheral register access or the pin lacks a usable `SMARTDMA_PIOx` mux.

Design checklist:

- Confirm the package pin, board connector/test point, and exact `SMARTDMA_PIOx` mux from the official pinout. Generated `pin_mux.c` comments are useful for board connector labels, but the official Pinout/Signal Multiplexing row owns the `ALTn` value.
- Arm side configures the pin mux to `SMARTDMA_PIOx`, enables the SmartDMA clock/reset, prepares the parameter block, and boots the SmartDMA code.
- SmartDMA side sets output direction with `GPD` for output pins, reads `GPI` for input pins, and writes/toggles `GPO` for output state.
- Parameter blocks for direct-I/O outputs usually need stack, timing values, optional result/debug buffer, and optional pin mask/PIO index. They do not need GPIO peripheral register-address tables.
- For waveform output, use `E_HEART_RYTHM`/`E_WAIT_FOR_BEAT` for periodic timing when appropriate, then update `GPO` with `E_BSET_IMM`, `E_BCLR_IMM`, or `E_BTOG_IMM`.

Minimal 50% duty-cycle output shape:

```c
typedef struct _smartdma_direct_io_param
{
    uint32_t *smartdma_stack;
    uint32_t half_period_beats;
    uint32_t pio_index;
    uint32_t *p_debug;
} smartdma_direct_io_param_t;

void __attribute__((section("for_smartdma_RAM"))) Smartdma_direct_pwm(void)
{
    /* Register allocation:
     * R0: half-period beats
     * R1: PIO index
     * R6: parameter pointer
     * GPD: SmartDMA pin direction
     * GPO: SmartDMA pin output state
     * SP: SmartDMA stack pointer
     */
    E_PER_READ(R6, ARM2SMARTDMA);
    E_LSR(R6, R6, 2);
    E_LSL(R6, R6, 2);

    E_LDR(SP, R6, 0);
    E_LDR(R0, R6, 1);
    E_HEART_RYTHM(R0);
    E_LDR(R1, R6, 2);
    E_BSET(GPD, GPD, R1);

    E_LABEL("loop");
    E_WAIT_FOR_BEAT;
    E_BTOG(GPO, GPO, R1);
    E_GOSUB(loop);
}
```

Concrete finding from MCXN947/FRDM-MCXN947 work: the NXP datasheet Pinout topic is available through bundle `MCXNP184M150F70` and topic path `topics/mcxnxxx_signal_multiplexing_and_pin_assignments.html`. If the visible page does not expose the table, fetch `https://nxp-docs-be-prod.nxp.com/api/bundle/MCXNP184M150F70/page/topics/mcxnxxx_signal_multiplexing_and_pin_assignments.html`, parse `topic_html`, and extract the `P3_19` row. That official row reports `P3_19`, ball `K17`, `ALT0 - P3_19`, `ALT2 - FC7_P6`, `ALT4 - CT2_MAT1`, `ALT5 - PWM1_X1`, `ALT6 - FLEXIO0_D27`, `ALT7 - SMARTDMA_PIO19`, and `ALT10 - SAI1_RX_FS`. For a 20 kHz, 50% duty-cycle SmartDMA PWM on that pin, configure `.mux = kPORT_MuxAlt7`, set `GPD` bit 19, and toggle `GPO` bit 19 every half period. Do not use a GPIO3 `PTOR` peripheral write for this direct-I/O case.

## Firmware Array Workflow

Use this only when the user needs firmware-array assets. Still do not rely on SDK SmartDMA boot APIs unless explicitly requested:

1. Generate or update C macro SmartDMA functions first.
2. Confirm linker/scatter placement for SmartDMA code at the required execution address.
3. Build with the target toolchain.
4. Extract the SmartDMA code section from ELF/AXF/map/listing using the project-supported tool (`objcopy`, IDE output, existing script, or SDK workflow).
5. Convert extracted bytes/words to a C array.
6. Verify the array entry table matches the API enum order.
7. Provide a direct-register load/boot path for the array, or only provide the array/export artifact if the user will integrate it manually.

Do not invent array bytes. If extraction tools or build artifacts are unavailable, provide the exact build/export procedure and list what the user must run.

## Verification Pattern

Provide task-specific checks:

- Build compiles with the target toolchain.
- Map/listing shows SmartDMA code at the intended RAM/execution address.
- Direct `CTRL` writes include `0xC0DE`; do not hide boot behavior behind SDK SmartDMA driver calls unless requested.
- `ARM2SMARTDMA` parameter pointer is aligned after low-bit mask removal.
- GPIO pins show expected waveform on logic analyzer.
- Interrupt callback fires only on expected events.
- Shared RAM result updates are coherent under Arm/SmartDMA contention.
