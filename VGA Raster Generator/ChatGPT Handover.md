# Project Handover: C64-Compatible VGA Raster Generator Using ATF1504AS CPLD

## 1. Project objective

I am designing a VGA raster timing generator as part of a larger project intended to display Commodore 64 (C64) resolutions on a VGA monitor.

The design must use **5 V digital logic** and an **18 MHz pixel clock**.

The current goal is to implement the raster timing generator in a **Microchip ATF1504AS CPLD**, replacing an earlier design based on discrete 74HC163 counters and TTL decoding logic.

The primary objectives are:

- Minimize hardware complexity and chip count.
- Generate accurate, stable VGA horizontal and vertical timing.
- Provide synchronous, glitch-free control outputs.
- Export a 9-bit C64-compatible raster line number.
- Leave CPLD capacity available for future video-system functions.
- Develop and test the system incrementally, beginning with the raster generator alone.

The CPLD will eventually form part of a larger C64-compatible video system involving VRAM, a DAC, CPU-controlled registers, and potentially VRAM access arbitration.

However, these additional functions are explicitly **outside the scope of the first implementation**.

## 2. Selected CPLD and development environment

**Target device: Microchip ATF1504AS**

Relevant characteristics:

- 64 macrocells.
- Native 5 V operation.
- Nonvolatile CPLD configuration.
- JTAG in-system programming.
- Currently listed by Microchip as an in-production device.
- Suitable for an 18 MHz raster generator.

The estimated initial macrocell requirement is approximately 25–35 macrocells, although this has not yet been confirmed by compiling and fitting the design.

Actual resource consumption and timing must be established from the ATF15xx fitter report.

### Development tools

The intended workflow is:

1. Develop a behavioural Verilog reference model.
2. Simulate and verify the raster generator.
3. Implement the circuit in CUPL.
4. Compile and fit it using **Microchip WinCUPL II**.
5. Inspect macrocell usage, product-term usage, pin assignments, and timing.
6. Generate a JEDEC programming file.
7. Program the ATF1504AS using an appropriate JTAG programmer.

I have access to a **Windows 11 PC**, so using WinCUPL II is not a problem.

Microchip's official WinCUPL page:

https://www.microchip.com/en-us/development-tool/wincupl

Microchip's ATF1504AS page:

https://www.microchip.com/en-us/product/ATF1504AS

The previous discussion identified WinCUPL II as the preferred development environment rather than pursuing a licensed Verilog synthesis flow.

**The next task is to write the first CUPL implementation of the raster generator.**

## 3. Raster counter architecture

The CPLD contains two logically separate counters:

| Counter | Width | Range | Purpose |
|---|---:|---|---|
| Horizontal H | 9 bits | `$000–$1FF` | Pixel position within scanline |
| Vertical V | 10 bits | `$000–$270` | Physical scanline number |

Hexadecimal values use the `$` prefix throughout this project.

The pixel clock is **18 MHz**.

### Horizontal operation

The horizontal counter increments on every rising edge of the 18 MHz clock.

It counts:

`$000, $001, ... $1FE, $1FF, $000`

Each scanline therefore contains 512 pixel clocks.

### Vertical operation

The vertical counter increments once per horizontal wrap.

It counts:

`$000, $001, ... $26F, $270, $000`

There are 625 physical scanlines per frame.

### Frame reset

The final valid raster position is:

- `V = $270`
- `H = $1FF`

The equivalent combined 19-bit counter value is:

`$4E1FF = 319999 decimal`

At the next rising edge of the 18 MHz clock, both counters synchronously return to zero.

The resulting sequence contains exactly:

`512 × 625 = 320,000 pixel clocks per frame`

The reset must be **synchronous**, not an asynchronous combinational clear.

### Timing frequencies

| Parameter | Value |
|---|---:|
| Pixel clock | 18 MHz |
| Pixel period | 55.556 ns |
| Pixel clocks per scanline | 512 |
| Scanline frequency | 35,156.25 Hz |
| Scanline duration | 28.444 µs |
| Scanlines per frame | 625 |
| Frame frequency | 56.25 Hz |
| Frame duration | 17.778 ms |

These timing parameters are intentional and should not be changed without discussion.

## 4. Required horizontal timing

The following are the exact required horizontal control intervals.

Ranges are specified as:

`START → END`

where START is inclusive and END is exclusive.

Wrapping ranges are permitted.

| Signal | Polarity | Active range |
|---|---|---|
| HSYNC | HIGH | `$1DC → $000` |
| HBLANK | Internal | `$1D0 → $040` |
| HBORDER_RAW | Internal | `$1A8 → $068` |

The border interval intentionally wraps through the horizontal blanking region.

This is a deliberate simplification.

For example:

- HBLANK is asserted from `$1D0` through `$1FF`, and from `$000` through `$03F`.
- HBORDER_RAW is asserted from `$1A8` through `$1FF`, and from `$000` through `$067`.
- HSYNC is HIGH from `$1DC` through `$1FF`.

HSYNC is a required external active-high signal.

HBLANK and HBORDER_RAW are internal conceptual signals. Their physical polarity may be chosen to minimize CPLD resources.

## 5. Required vertical timing

The following are the exact required vertical control intervals.

| Signal | Polarity | Active range |
|---|---|---|
| VSYNC | HIGH | `$26F → $000` |
| VBLANK | Internal | `$26E → $016` |
| VBORDER_RAW | Internal | `$20A → $07A` |

The vertical counter wraps from `$270` to `$000`.

Consequently:

- VSYNC is HIGH on physical scanlines `$26F` and `$270`.
- VBLANK is asserted on `$26E–$270` and `$000–$015`.
- VBORDER_RAW is asserted on `$20A–$270` and `$000–$079`.

As with horizontal timing, the raw border interval intentionally extends through the blanking interval.

## 6. Final BLANK and BORDER outputs

The CPLD must generate two external control signals:

- `BLANK`
- `BORDER`

The internal logical functions are:

```verilog
BLANK = HBLANK || VBLANK;

BORDER_RAW = HBORDER_RAW || VBORDER_RAW;

BORDER = BORDER_RAW && !BLANK;
```

**BLANK has absolute priority over BORDER.**

Therefore, whenever BLANK is asserted, BORDER must be deasserted.

This produces mutually exclusive external control signals, even though the internal raw border and blank intervals intentionally overlap.

The downstream DAC will ultimately behave as follows:

```text
if BLANK:
    RGB = 0V
else if BORDER:
    RGB = border colour
else:
    RGB = video pixel colour
```

The DAC should also implement blank priority defensively, although the CPLD already enforces it.

For CPLD version 1, use active-HIGH BLANK and BORDER outputs unless the device fitting process reveals a compelling reason to choose different polarities.

## 7. Registered outputs and timing alignment

The earlier discrete-logic design used one-count lookahead decoding and D flip-flops.

For the CPLD implementation, the preferred approach is simpler:

1. Calculate the next horizontal and vertical counter values.
2. Calculate the required output states corresponding to those next counter values.
3. On the same rising clock edge, update the counter registers and the control output registers.

Conceptually:

```verilog
always @(posedge clk_18mhz) begin
    h      <= h_next;
    v      <= v_next;

    hsync  <= hsync_next;
    vsync  <= vsync_next;
    blank  <= blank_next;
    border <= border_next;
end
```

This ensures that the control signals correspond to the counter position entered on that clock edge.

The external control signals should be **registered**, not combinationally driven from changing counter outputs.

This prevents combinational decoding glitches from appearing on VGA control outputs.

The implementation must avoid accidental one-clock offsets at transition boundaries.

### Reset behaviour

The external synchronous reset must initialize:

```text
H      = $000
V      = $000

HSYNC  = LOW
VSYNC  = LOW
BLANK  = HIGH
BORDER = LOW
```

These values correspond to the correct timing state at the beginning of a frame.

Reset polarity has not yet been fixed; it may be chosen during the CUPL implementation.

## 8. Exported C64-compatible raster number

The physical VGA raster contains 625 scanlines.

The eventual C64-compatible video system uses **two physical VGA scanlines per logical C64 raster line**.

The internal physical vertical counter is 10 bits wide:

`V[9:0]`

The exported logical raster number is 9 bits wide:

```verilog
assign raster[8:0] = v[9:1];
```

Thus:

| Physical VGA scanlines | Exported raster |
|---|---|
| 0, 1 | 0 |
| 2, 3 | 1 |
| 4, 5 | 2 |
| 6, 7 | 3 |
| ... | ... |
| 622, 623 | 311 |
| 624 | 312 |

This gives the expected 9-bit PAL-style C64 raster numbering range of 0–312.

The final logical raster value, 312, occupies only one physical scanline because the frame contains 625 lines.

This is intentional and follows from the specified frame length.

The exported raster bus will later support an external programmable raster-interrupt comparator.

## 9. Raster interrupts

The C64 supports programmable raster interrupts, where the interrupt occurs on a scanline selected by a CPU-writable register.

Therefore, raster IRQ generation will **not** be implemented inside the CPLD in version 1.

Instead:

- The CPLD exports `RASTER[8:0]`.
- External circuitry will compare that bus with a CPU-writable raster interrupt register.
- External logic will generate the CPU-compatible interrupt signal.

The CPU is expected to operate at approximately 1 MHz, with possible 2 MHz or 4 MHz turbo modes.

The interrupt circuitry must account for this separate clock domain.

For now, no `FRAME_IRQ` or `FRAME_TICK` output is required.

VSYNC remains a video synchronization output only.

## 10. VRAM and CPU interface — future development

The ultimate project is a C64-compatible video system.

The raster generator will eventually interact with:

- VRAM
- Pixel-data fetching logic
- Border-colour register
- RGB DAC
- CPU bus
- Programmable raster IRQ comparator
- VRAM access arbitration logic

These are not part of raster-generator version 1.

### Horizontal address export

We expect to export only some of the horizontal counter bits later.

VRAM data will probably be fetched byte-wise or word-wise rather than once per individual pixel.

Therefore, not all nine horizontal counter bits will be required externally.

The exact horizontal bus subset will be decided when the pixel-fetch architecture is designed.

### Border colour

The border-colour register will likely remain outside the raster CPLD because it needs to be writable by the CPU.

The CPLD only needs to produce the BORDER control signal.

### VRAM arbitration

Later versions may add signals controlling CPU versus video access to VRAM.

Whether this logic belongs inside the CPLD will depend on the final architecture, remaining macrocell capacity, and available I/O pins.

Do not add it to version 1.

## 11. CPLD version 1 interface

The agreed initial interface is:

| Signal | Direction | Width | Description |
|---|---|---:|---|
| `CLK_18MHZ` | Input | 1 | 18 MHz pixel clock |
| `RESET` | Input | 1 | Reset raster generator |
| `HSYNC` | Output | 1 | Active-high horizontal sync |
| `VSYNC` | Output | 1 | Active-high vertical sync |
| `BLANK` | Output | 1 | Active-high RGB blanking |
| `BORDER` | Output | 1 | Active-high border selection |
| `RASTER[8:0]` | Output | 9 | Logical C64 raster line, V[9:1] |

Total external signal count: **15 I/O signals**, including clock and reset.

No horizontal counter outputs need to be exported yet.

No frame interrupt output is required.

## 12. Previous design and verification

Before deciding to use a CPLD, a horizontal timing generator was built using discrete 74163 counters, Boolean decoders, XOR-feedback D flip-flops, and H. Neemann's Digital simulator.

The horizontal circuit was tested and its exported Verilog inspected.

The final horizontal timing equations were found to be logically correct.

This provided useful validation of the horizontal timing boundaries.

The old circuit used:

```text
HBLANK transitions:
$03F and $1CF

HBORDER_RAW transitions:
$067 and $1A7

HSYNC transitions:
$1DB and $1FF
```

These are one-count lookahead addresses.

The CPLD implementation does not need to reproduce the old discrete gate architecture.

It should instead implement the desired raster behaviour directly, with synchronous counters and registered control outputs.

The previous Digital circuit can serve as a conceptual reference, but the new CUPL implementation should be independently verified.

## 13. Required simulation and verification

Before programming the CPLD, verify the entire 320,000-clock frame.

Important horizontal transitions:

| Current H | Next H | Expected change |
|---|---|---|
| `$03F` | `$040` | HBLANK deasserts |
| `$067` | `$068` | HBORDER_RAW deasserts |
| `$1A7` | `$1A8` | HBORDER_RAW asserts |
| `$1CF` | `$1D0` | HBLANK asserts |
| `$1DB` | `$1DC` | HSYNC asserts |
| `$1FF` | `$000` | HSYNC deasserts |

Important vertical transitions occur at horizontal wrap:

| Current V | Next V | Expected change |
|---|---|---|
| `$015` | `$016` | VBLANK deasserts |
| `$079` | `$07A` | VBORDER_RAW deasserts |
| `$209` | `$20A` | VBORDER_RAW asserts |
| `$26D` | `$26E` | VBLANK asserts |
| `$26E` | `$26F` | VSYNC asserts |
| `$270` | `$000` | VSYNC deasserts; frame wraps |

Additional checks:

- Exactly 512 pixel clocks per scanline.
- Exactly 625 scanlines per frame.
- Exactly 320,000 clocks per frame.
- HSYNC and VSYNC active HIGH.
- BLANK and BORDER never simultaneously HIGH.
- BORDER correctly excluded during blanking.
- All output transitions aligned with the correct clock edges.
- Correct reset values.
- Correct exported `RASTER[8:0]`.
- Correct wrap from `$4E1FF` to `$00000`.

A behavioural Verilog testbench is desirable for exhaustive verification.

## 14. Resource and timing objectives

The selected ATF1504AS has 64 macrocells.

The bare raster generator is expected to require approximately:

- 19 macrocells for the H/V counters.
- 4 macrocells for registered HSYNC, VSYNC, BLANK, BORDER.
- Additional resources for counter rollover and Boolean decoding.
- Possible intermediate/helper macrocells depending on fitter optimization.

A preliminary total estimate was approximately 25–35 macrocells, but this is **not a measured result**.

The real usage must be obtained from the ATF15xx fitter.

The design must satisfy 18 MHz timing at 5 V with adequate margin.

Do not assume that a successful logical compile proves timing closure.

After fitting, inspect:

- Macrocell count.
- Product-term usage.
- Logic expansion/cascading.
- Pin assignments.
- Worst-case timing.
- Maximum supported clock frequency.

Any unused CPLD capacity should remain available for future video functions.

## 15. Immediate next steps

The project is ready to enter implementation.

**Step 1:** Install WinCUPL II on the available Windows 11 PC.

**Step 2:** Create a new project targeting the ATF1504AS, initially without committing to a specific package/pinout until the package choice is confirmed.

**Step 3:** Write the first complete CUPL source implementing:

- 9-bit horizontal counter.
- 10-bit vertical counter.
- 512-clock horizontal wrap.
- 625-line frame wrap.
- Registered HSYNC.
- Registered VSYNC.
- Registered BLANK.
- Registered BORDER with blank priority.
- Exported `RASTER[8:0] = V[9:1]`.
- Synchronous reset.

**Step 4:** Compile the CUPL source and resolve any tool/device-specific issues.

**Step 5:** Fit to the ATF1504AS and inspect resource usage and timing.

**Step 6:** Develop or maintain an independent Verilog reference model and verify the full raster timing.

**Step 7:** Once the CPLD raster generator is working, proceed incrementally to the pixel-fetch, VRAM, DAC, and CPU-interface stages.

## 16. Guidance for the next ChatGPT assistant

Please act as an experienced CPLD/programmable-logic designer familiar with Microchip ATF15xx devices, CUPL, WinCUPL II, synchronous counters, and video raster timing.

Preserve the agreed timing constants and output specifications.

Prioritize a **working, synthesizable, correctly timed ATF1504AS design** over premature manual Boolean minimization.

Be careful about CUPL-specific syntax, registered macrocell behaviour, clock polarity, reset behaviour, and the actual capabilities of the ATF1504AS fitter.

Do not assume generic Verilog constructs translate directly into CUPL.

Avoid adding unrequested VRAM, CPU, or interrupt functionality at this stage.

The next thing I need help with is:

**Writing the first WinCUPL II source file for the ATF1504AS raster generator, then compiling, fitting, and verifying it step by step on my Windows 11 PC.**

