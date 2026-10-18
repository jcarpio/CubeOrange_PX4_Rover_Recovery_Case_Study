# CubePilot Cube Orange — Build C: Read-Only IOMCU Diagnostic Probe, PX4IO/UART Failure Analysis, and Thermal Measurements

**Technical report and repository integration guide**  
**Date:** 18 October 2026  
**Status:** Experimental diagnostic firmware; hardware fault unresolved  
**Target:** Original CubePilot **Cube Orange**, PX4 board ID **140** — **not Cube Orange+**  
**Firmware:** PX4 `release/1.17` (v1.17 development line) with local modifications  
**Branch:** `cubeorange-iomcu-diagnostics`  
**Build C source commit:** `2092c87c6aed28e0c45580f0b061c2994d2923e7`  
**Generated firmware:** `build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4`

> [!CAUTION]
> **DIAGNOSTIC USE ONLY — NOT A HARDWARE REPAIR.** The test Cube Orange has a persistent I/O microcontroller (IOMCU) communication fault and unexplained heating. Build C is **not validated for flight, motor operation, autonomous driving, or any safety-critical application**. Never connect motors, ESCs, vehicle batteries, or actuators during initial diagnostic tests. A disabled PX4 heater command does **not** guarantee that the physical heater is off. Disconnect USB if heating accelerates, the housing becomes unusually hot, or electrical abnormalities occur. The testing described below was performed on a stationary, USB-powered unit.

## 1. Executive summary

Build C was developed to diagnose an **unresponsive PX4IO/IOMCU** on a second-hand original CubePilot Cube Orange, while preserving the successfully booting **PX4 Differential Rover** environment established in Builds A and B.

Two earlier experiments isolated distinct effects:

- **Build A** included the PX4IO driver but excluded the unusually hot ICM20649 IMU. Rover and the remaining IMUs worked, but normal PX4IO firmware checking/initialization failed.
- **Build B** also omitted the PX4IO driver. The rover and two IMUs still functioned, and the automatic PX4IO startup errors disappeared; this **did not repair** the IOMCU or resolve heating (both remaining IMUs had previously reached ~67 °C).

Build C combines the desirable properties of both: it **compiles the PX4IO transport and driver into the firmware** yet **prevents their automatic boot-time startup, CRC checking and firmware update**. A new, deliberately constrained `px4io probe` command uses the existing serial transport to read one IOMCU **protocol-version register** without invoking the full PX4IO initialization/update sequence. Diagnostic performance counters then reveal how the request failed.

**Observed on the physical Cube Orange:**

```text
nsh> px4io status
INFO  [px4io] not running

nsh> px4io probe
ERROR [px4io] IOMCU communication failed: -1
px4io: txns: 1 events, 16us elapsed, 16.00us avg, min 16us max 16us   nanus rms
px4io: retries: 3 events
px4io: timeouts: 2 events
px4io: crcerrs: 0 events
arch_px4io_serial: DMA errors: 1 events
px4io: protoerrs: 0 events
px4io: uarterrs: 1 events
```

**Primary conclusion:** The flight-controller-side diagnostic command executes, but **no valid IOMCU protocol-version response is obtained**. The failed exchange involves **two timeout events and one DMA/UART error event**, with zero observed CRC or protocol-error events. It is **not possible** to determine from these counters alone whether the IOMCU is unpowered, held in reset, executing corrupt firmware, electrically damaged, or unable to communicate because of USART wiring, transceiver, DMA, or carrier-related faults. In particular, **a UART/DMA error is not proof that the IOMCU transmitted anything**.

Both active accelerometer instances reported `error_count: 0` throughout the captured measurements. Their temperature rose from approximately **34 °C** near 93 seconds of uptime to approximately **47.6 °C** at 385 seconds. A separate capture showed **53.39 °C / 52.92 °C** at approximately 10 seconds of **PX4 uptime** after the user had deliberately disconnected and reconnected USB. That timestamp reset was **not** an unexplained spontaneous reboot. The thermal source remains undetermined.

## 2. Project background and motivation

The longer-term goal is to use the Cube Orange as a MAVLink-capable autopilot for a two-motor, differential-drive agricultural rover, initially on a **Yahboom MicroROS V2**, eventually with a Raspberry Pi 5, ROS 2, RTK GNSS, LiDAR, cameras, and a tracked chassis for olive-grove operations.

The used board had shown failures with ArduPilot, including unsuccessful IOMCU firmware update / IO communication, making it important to separate main-flight-MCU functionality from the I/O coprocessor fault. A standard PX4 1.17 image booted, establishing that useful diagnostics could still be run on the STM32H7 flight controller. The ICM20649 IMU previously reported approximately **79–87.5 °C**, hotter than the others; removing that driver improved behavior in earlier tests without proving a hardware cause.

For full earlier context and reproducible Build A/B procedures, retain the companion files:

- `CubeOrange_PX4_Rover_Recovery_Case_Study.md`
- `README_CubeOrange_PX4_Rover_Release.md`

Build C **extends** these documents; it does not supersede their observations.

## 3. Environment, identities, and reproducibility

| Component | Confirmed information |
|---|---|
| Hardware | Original CubePilot Cube Orange, with carrier board |
| PX4 board ID | `140` |
| PX4 source branch | `release/1.17` |
| Diagnostic Git branch | `cubeorange-iomcu-diagnostics` |
| Previous rover configuration commit | `b377a2093b` |
| Build C diagnostic changes commit | `2092c87c6aed28e0c45580f0b061c2994d2923e7` |
| PX4 build target | `cubepilot_cubeorange_rover` |
| Build host | Ubuntu ARM64 virtual machine, user `yahboom` |
| Source checkout | `~/orange_cube_rover_firmware/px4-rover` |
| Python environment | `(px4-venv)` |
| Target OS shown in NSH | `NuttX-12.12.0` |
| Ground-control console | QGroundControl, MAVLink Console on macOS |
| Firmware artifact | `build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4` |
| Package `image_size` | **1,955,552 bytes** |
| Linker FLASH budget | **1,966,080 bytes** (1,920 KiB) |
| Remaining linker FLASH budget | **10,528 bytes** |
| FLASH occupancy | Approximately **99.46%** |

**Important distinction:** `image_size` is the uncompressed image size reported inside the PX4 package metadata; do not confuse it with the compressed `.px4` file size. The package was inspected with Python's `json` module, which reported `board_id: 140`, `image_size: 1955552`, and the exact full Git hash above. Reproducibility also depends on the same PX4 submodule revisions, compiler, configuration, and source tree state.

### 3.1 Source control sequence

1. `b377a2093b`: a Cube Orange Rover board configuration enabling differential-drive support and excluding ICM20649; earlier iteration had PX4IO disabled.
2. `2092c87c6a`: additional IOMCU probe implementation, the PX4IO serial diagnostic/error-handling changes, and the modified `rcS` boot gating that define **Build C**. PX4IO is enabled in the final board Kconfig, but deliberately **not started automatically**.

The final checked package explicitly identified `2092c87c6aed28e0c45580f0b061c2994d2923e7`; an intermediate package had temporarily reported the preceding commit before an incremental rebuild refreshed metadata.

## 4. Design requirements and limits

Build C was designed to:

1. **Retain** differential-rover capability and the two working IMU drivers.
2. **Exclude** the ICM20649 driver from this experimental image.
3. **Include** the PX4IO code and USART/DMA transport so the serial link can be diagnosed.
4. **Avoid automatic PX4IO initialization, IO firmware CRC check, firmware update, and normal startup** during boot.
5. **Avoid automatic PX4 heater startup** from the checked `rcS` path.
6. Offer a manually invoked **read-only application-level probe** of `PX4IO_PAGE_CONFIG / PX4IO_P_CONFIG_PROTOCOL_VERSION`.
7. Report communication failure categories through existing/augmented performance counters.
8. Cleanly release serial resources after the probe, including on partial initialization failure.

These are **software behavior objectives**, not guarantees about physical outputs: the IOMCU, its boot firmware, heater circuitry, and electrical rails can behave independently of the PX4 FMU application. The added command is read-only at the PX4IO register level; it still configures and exercises the FMU UART, interrupts, and DMA, and may trigger whatever behavior the remote hardware implements upon receiving a request.

## 5. Modified source files — Build C

**Six source files** were changed in commit `2092c87c6a`. The descriptions below document the behavior established by the development-session review; the authoritative, line-perfect code must be retrieved from Git using the commit and patch-export commands in Section 11.

### 5.1 `boards/cubepilot/cubeorange/rover.px4board`

The custom board variant derives from the Cube Orange default configuration and includes these significant selections:

```ini
CONFIG_MODULES_ROVER_DIFFERENTIAL=y
CONFIG_DRIVERS_PX4IO=y
# CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649 is not set
```

PX4IO is deliberately **compiled in**, unlike Build B. The absence of ICM20649 is an experimental isolation step, not a sensor repair. Other modules inherited from the base board configuration remain subject to what the final generated configuration enables.

### 5.2 `ROMFS/px4fmu_common/init.d/rcS`

The standard PX4IO startup block was guarded by a deliberately false condition, conceptually:

```sh
# In the PX4IO startup section of the experimental rcS:
if false
then
    # Existing px4io supported/checkcrc/update/start startup body
    # retained in source but not executed from this block
fi
```

The automatic `heater start` invocation in the relevant `rcS` path was removed/disabled. The generated ROMFS startup script was inspected after compilation and confirmed to contain `if false` at the PX4IO section (observed around generated `rcS` line 235); the known startup calls were within the disabled block. A search of the checked startup scripts identified no other matching automatic `px4io start`, `px4io update`, `px4io checkcrc`, or `heater start` invocations outside this guarded logic.

**Scope caveat:** this statement applies to the scripts reviewed during development. It is not a proof that no board-specific firmware, bootloader, or physical hardware behavior can activate the IOMCU or heater. Modification of the shared `rcS` also affects other PX4 builds made **from the same modified checkout**; preserve this experiment on its own branch and do not merge it into general PX4 source without board-specific conditionalization.

### 5.3 `src/drivers/px4io/px4io.cpp`

A new `px4io probe` verb was added to `PX4IO::custom_command(...)`, adjacent to the existing `supported` handling. The probe:

1. Rejects invocation if a normal PX4IO instance is already running.
2. Calls `get_interface()` to instantiate/initialize the architecture-specific PX4IO serial interface.
3. Performs a single logical register read for the IOMCU protocol version, using the PX4IO register address composed from `PX4IO_PAGE_CONFIG` and `PX4IO_P_CONFIG_PROTOCOL_VERSION`.
4. Prints the protocol version if exactly one register is returned; otherwise prints the failure code.
5. Calls `interface->ioctl(100, diagnostic_arg)` to report transport counters.
6. Deletes the interface on return; exits successfully only for a valid single-register read.

Representative implementation logic (illustrative; **not** a substitute for the exact Git diff):

```cpp
if (!strcmp(verb, "probe")) {
    if (is_running()) {
        PX4_ERR("PX4IO must be stopped");
        return 1;
    }

    device::Device *interface = get_interface();
    if (!interface) {
        PX4_ERR("IOMCU interface unavailable");
        return 1;
    }

    uint16_t protocol = 0;
    unsigned address = (PX4IO_PAGE_CONFIG << 8) | PX4IO_P_CONFIG_PROTOCOL_VERSION;
    int ret = interface->read(address, &protocol, 1);

    if (ret == 1) {
        PX4_INFO("IOMCU protocol version: %u", unsigned(protocol));
    } else {
        PX4_ERR("IOMCU communication failed: %d", ret);
    }

    unsigned diagnostic_arg = 0;
    interface->ioctl(100, diagnostic_arg);
    delete interface;
    return ret == 1 ? 0 : 1;
}
```

The actual output from the hardware was `IOMCU communication failed: -1`.

**Read-only qualification:** `PX4IO_serial::read()` can retry failed serial exchanges. The report's `retries: 3` should not be interpreted as three distinct operator invocations or as evidence of three valid remote responses.

### 5.4 `platforms/nuttx/src/px4/stm/stm32h7/px4io_serial/px4io_serial.cpp`

The STM32H7 `ArchPX4IOSerial::ioctl` gained an experimental `case 100` for printing communication/performance counters, including a DMA/UART-related diagnostic counter. The surrounding `switch (operation)` was corrected to prevent accidental fall-through after the nested ioctl handling.

Resource-lifecycle handling was hardened so that the serial destructor disables/relinquishes an attached IRQ and UART before releasing/stopping DMA resources. Successful IRQ attachment and UART initialization are tracked using state flags, avoiding unconditional cleanup of resources that may not have initialized. Semaphores are initialized earlier and their status is checked.

**Implementation detail:** the numeric ioctl operation `100` is a **local, experimental diagnostic convention**, not a public, stable PX4IO API. It must not be assumed compatible with future PX4 versions.

### 5.5 `platforms/nuttx/src/px4/stm/stm32_common/include/px4_arch/px4io_serial.h`

The relevant serial classes gained explicit initialization-state flags:

- `PX4IO_serial`: `_bus_sem_initialized{false}`.
- `ArchPX4IOSerial`: `_uart_initialized{false}`, `_irq_attached{false}`, `_completion_sem_initialized{false}`.

These distinguish valid cleanup paths following complete or partial startup. During development, a class-placement error for `_completion_sem_initialized` caused a compile error; moving it to the `ArchPX4IOSerial` class resolved the build. The final firmware compiled successfully.

### 5.6 `src/drivers/px4io/px4io_serial.cpp`

`PX4IO_serial::init()` checks the return of `px4_sem_init(&_bus_semaphore, 0, 1)` and sets `_bus_sem_initialized` only on success. The matching destructor conditionally destroys that semaphore. The architecture-specific completion semaphore follows an analogous guarded lifecycle in the STM32H7 implementation.

This reduced resource-lifecycle risk when using a short-lived `get_interface()`/`read()`/`delete` diagnostic path instead of the normal, long-running PX4IO driver.

## 6. Build and packaging evidence

The resulting build completed successfully, including:

```text
[18/20] Linking CXX executable cubepilot_cubeorange_rover.elf
[20/20] Creating .../cubepilot_cubeorange_rover.px4
```

Measured linker FLASH usage:

```text
FLASH: 1,955,552 / 1,966,080 bytes = 99.46%
Free: 10,528 bytes
```

This is **extremely close to the linker limit**. Extra drivers, debugging text, or instrumentation may make a future build overflow. Do not conflate a successful build with evidence that the hardware fault is fixed.

### 6.1 Rebuild in the documented checkout

```bash
cd ~/orange_cube_rover_firmware/px4-rover
# Activate the existing px4-venv first if it is not active.
git switch cubeorange-iomcu-diagnostics
make cubepilot_cubeorange_rover -j4
```

### 6.2 Verify the produced firmware metadata

```bash
python3 - <<'PY'
import json
from pathlib import Path
p = Path('build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4')
fw = json.loads(p.read_text())
for field in ('board_id', 'image_size', 'git_hash'):
    print(f'{field}: {fw.get(field)}')
PY
```

Output recorded from Build C:

```text
board_id: 140
image_size: 1955552
git_hash: 2092c87c6aed28e0c45580f0b061c2994d2923e7
```

The package was copied to the Mac Downloads folder and flashed through QGroundControl's custom firmware workflow. PX4 booted and NuttShell commands executed, demonstrating successful deployment of Build C, not safety or full hardware health.

## 7. Test protocol and observed outputs

### 7.1 Physical test configuration

- Cube Orange with the experimental Build C firmware.
- QGroundControl's **Analyze Tools → MAVLink Console** on macOS.
- USB supply for initial testing; no motors, ESCs, actuators or vehicle batteries intended to be connected.
- No normal PX4IO startup or firmware-update command intentionally invoked.
- Heating monitored through onboard accelerometer-reported temperature, **not** calibrated case temperature or a thermocouple on the IOMCU/heater.

### 7.2 Verify the PX4IO driver is inactive

```text
nsh> px4io status
INFO  [px4io] not running
```

This confirms the **normal PX4IO driver instance** was not running at the time of the check. It does **not** prove the separate physical IO processor was powered off, reset, or inactive.

### 7.3 Invoke the minimal diagnostic read

```text
nsh> px4io probe
ERROR [px4io] IOMCU communication failed: -1
px4io: txns: 1 events, 16us elapsed, 16.00us avg, min 16us max 16us   nanus rms
px4io: retries: 3 events
px4io: timeouts: 2 events
px4io: crcerrs: 0 events
arch_px4io_serial: DMA errors: 1 events
px4io: protoerrs: 0 events
px4io: uarterrs: 1 events
```

These counts were observed for the command's diagnostic interface; they should **not** be treated as persistent device-lifetime counters or precisely one-for-one mappings to electrical waveforms without checking the lower-level source.

| Reported statistic | Value | Supported interpretation |
|---|---:|---|
| Logical transaction performance event | 1 | Diagnostic read was attempted; does not mean successful communication |
| Retry events | 3 | Transport retried internally |
| Timeout events | 2 | Two timeout conditions occurred |
| CRC error events | 0 | No CRC error was counted; does not prove signal integrity |
| Protocol error events | 0 | No protocol-error event was counted; does not imply a valid packet |
| DMA error events | 1 | A low-level transport/DMA-related failure was counted |
| UART error events | 1 | A UART-level failure event was counted |
| `read()` result | `-1` | Protocol version not read successfully |

**Do not conclude that a received response existed.** Some implementation paths can classify UART/peripheral or malformed-transfer conditions as an error without a successfully decoded packet. The simultaneous DMA and UART event counts may reflect **one underlying fault reported by two counters**, not independent failures.

### 7.4 Verify accelerometer health

```sh
listener sensor_accel 1
```

Two instances were repeatedly published:

- Instance 0: `device_id: 3670050`, `Type: 0x38`, SPI:4.
- Instance 1: `device_id: 2621474`, `Type: 0x28`, SPI:4.

Both showed `error_count: 0` in all provided readings. Acceleration Z remained near gravitational acceleration when the controller was stationary. This establishes that these sensors were providing plausible data in the sampled moments; it is not a complete calibration, vibration, thermal-accuracy or long-term reliability qualification.

## 8. Thermal log — full measured series

The table reproduces the user-recorded temperature observations from the **same continuous-uptime capture**, using `sensor_accel.timestamp` converted from microseconds to seconds. This is an IMU's internal sensor temperature, not direct heater temperature.

| PX4 uptime (s) | IMU 0 (°C) | IMU 1 (°C) | Error count, both |
|---:|---:|---:|---:|
| 92.839 | 34.57650 | 34.08294 | 0 |
| 205.504 | 39.81518 | 39.68991 | 0 |
| 237.091 | 41.25459 | 40.79214 | 0 |
| 270.800 | 42.72338 | 42.46944 | 0 |
| 303.466 | 44.13831 | 44.09881 | 0 |
| 310.170 | 44.47858 | 44.43427 | 0 |
| 313.158 | 44.58140 | 44.29050 | 0 |
| 324.667 | 45.06365 | 44.62596 | 0 |
| 326.326 | 45.14688 | 44.96142 | 0 |
| 342.094 | 45.78580 | 45.63234 | 0 |
| 344.007 | 45.88862 | 45.53650 | 0 |
| 363.056 | 46.72338 | 46.35118 | 0 |
| 367.439 | 46.92411 | 46.68665 | 0 |
| 385.259 | 47.62179 | 47.54926 | 0 |

For the recorded ~292.4-second interval (92.8 s → 385.3 s):

- IMU 0 increased by approximately **13.05 °C**.
- IMU 1 increased by approximately **13.47 °C**.
- Average rise over the whole interval was roughly **2.7 °C per minute**. This is descriptive of the measured interval, **not a prediction** of the final equilibrium temperature.
- Their close tracking supports a **shared thermal environment** as a plausible explanation; it does not distinguish heater operation from MCU dissipation, neighboring power components, enclosure heat transfer, or other hardware sources.
- The series was **still rising** at its end; it does not establish thermal equilibrium.

### 8.1 A different session / reset timeline

A later capture showed:

```text
sensor_accel instance 0:
    timestamp: 10382371
    temperature: 53.38923
    error_count: 0

sensor_accel instance 1:
    timestamp: 10401127
    temperature: 52.91661
    error_count: 0
```

The timestamps indicate about **10.4 seconds of PX4 uptime**, substantially less than the preceding ~385 seconds. **The operator confirmed that USB had been disconnected manually**, explaining the reboot/reset rather than a demonstrated spontaneous controller reset. It is possible for the physical module to retain significant heat across a short unplug/replug cycle; a high temperature at 10 seconds of PX4 uptime is **not** proof that it heated from room temperature to 53 °C in ten seconds. The time powered off and starting physical temperature were not measured.

### 8.2 Ground station warning unrelated to the IO link diagnosis

During the later run, QGroundControl displayed:

```text
Invalid configuration for rate control: Neither feed forward nor feedback is setup
```

This identifies an incomplete **differential-rover rate-controller configuration**. It does **not** explain the PX4IO communication failure or demonstrate hardware damage. It also reinforces that the image is not ready to drive motors: rover control parameters have not been commissioned.

## 9. Interpretation, competing hypotheses, and what is not established

### 9.1 Confirmed observations

- PX4/NuttX boots and accepts shell commands on the main FMU.
- The `cubepilot_cubeorange_rover` image has board ID 140 and the expected Build C Git identity.
- `px4io status` reports no running PX4IO driver.
- `px4io probe` is callable and attempts the protocol-version read.
- The diagnostic read fails with the precise counters documented above.
- Two active accelerometer instances return plausible motion/gravity readings and no accumulated driver errors during the observed samples.
- Both active IMUs heat up together over the continuous measurement interval.
- A later PX4 uptime reset followed **intentional USB disconnection**, not an observed spontaneous reboot.

### 9.2 Plausible causes requiring electrical evidence

1. **IOMCU supply or reset fault:** absent/unstable rail, clock, or reset release.
2. **IOMCU firmware / bootloader corruption:** MCU powered and clocked but not responding as expected.
3. **FMU↔IOMCU UART path fault:** TX/RX wiring, connector, loading, electrical damage, wrong peripheral setup, framing error.
4. **IOMCU silicon or nearby electrical damage:** relevant given earlier use and unexplained thermals.
5. **Heater/control circuit anomaly:** the heater might be energized incorrectly, or heating could originate elsewhere.
6. **Carrier interface problem:** board seating and 80-pin connector issues remain possible, despite reseating and the module fitting firmly.

These possibilities are **not ranked as confirmed diagnoses** by the available evidence.

### 9.3 Misinterpretations to avoid

| Tempting statement | What the evidence actually supports |
|---|---|
| “PX4IO is disabled, therefore the IOMCU is off.” | Only the **normal PX4 FMU driver** was not running. |
| “No CRC errors means UART wiring is good.” | There may have been no valid frame on which to check CRC. |
| “One UART/DMA error proves the IOMCU transmitted a reply.” | It does not; low-level failures can be local or partial. |
| “The two IMUs are undamaged.” | They report data without errors in these snapshots, not a full hardware qualification. |
| “The heater is off because `heater start` was removed.” | Physical heater control has not been measured. |
| “53 °C at 10 s means an extreme 10-second warmup.” | The user unplugged/replugged USB; residual heat is possible. |
| “The rover controller warning caused the IOMCU fault.” | No causal evidence connects the parameter warning and UART failure. |
| “Build C fixed the Cube Orange.” | It did not; communication and thermal anomalies persist. |

## 10. Recommended next investigation — electrical, not more unattended heating

Further repetitive software-only probe attempts have diminishing value. Prefer an **unpowered inspection** followed by **short, supervised electrical measurements** by a suitably experienced person.

1. **Power down and cool fully.** Remove USB, battery/power module, telemetry accessories and every actuator connection before inspecting. Do not probe a live 80-pin connector blindly; nearby contacts are easy to short.
2. **Document the assembly.** Photograph both module and carrier, including connector alignment, contact condition, corrosion, bent pins and suspicious heating/discoloration. A tight mechanical fit alone does not establish contact integrity.
3. **Locate authoritative board schematics / pin maps.** Establish the IOMCU's MCU package, relevant supply pins, reset, ground, and UART TX/RX paths **before** energized measurement. No pin-by-pin voltage targets are asserted in this report without the matching board schematic.
4. **Measure power rails and reset using high-impedance equipment.** Check for absent rail, overcurrent, reset held low, or transient power sag. Protect against accidental shorts. Use bench current limiting and appropriate isolation if a laboratory power setup is available.
5. **Inspect USART traffic with a logic analyzer or oscilloscope.** Capture both FMU TX and IOMCU TX during one short probe. Verify signal polarity, idle level, transmission, any return activity, error conditions, and timing. The protocol/baud details must be checked against the exact Build C source; development review identified an STM32H7 USART6 pathway configured around **1.5 Mbps**, with approximately **10 ms** transaction timeout and internal retries.
6. **Investigate the heater independently.** Use documented PX4IO board/firmware mappings (the earlier project source review associated the IO heater control with PB14), and determine whether the physical heater control line and power path are active; do **not** infer hardware state solely from application `heater status`.
7. **Consider a recovery/repair decision.** If IOMCU power, reset, and electrical traffic are normal but there is still no response, firmware recovery may be worth considering only after verifying compatibility and available programming interfaces. If the device is electrically abnormal, prioritize repair/replacement over repeated flashing.

**Do not run a rover, mower, steering/drive actuator, or aircraft from this board until the IOMCU fault, output behavior, thermal stability, arming, failsafes, emergency stop and control mapping are independently verified.**

## 11. Adding the modified PX4 source to a repository

### 11.1 Recommended strategy: preserve the Git branch, not just a binary

**Yes, the modified PX4 source can be preserved and published.** The most reproducible record is an actual Git branch or fork carrying the two commits, plus the diagnostic report, the `.px4` artifact (preferably as a **GitHub Release asset**, not a large binary tracked in normal Git), and a patch/diff for external review.

Important: the working PX4 checkout is on the user's Ubuntu Builder VM, not inside this report. **This document contains a faithful overview and representative snippets but not an independently extracted copy of all six modified source files.** To publish the exact code, export it from the real checked-out commit. Do **not** reconstruct it from snippets in this report.

Suggested project organization:

```text
cubeorange-px4-diagnostics/                    # independent documentation repository, optional
├── README.md
├── docs/
│   ├── CubeOrange_PX4_Rover_Recovery_Case_Study.md
│   └── CubeOrange_Build_C_IOMCU_Diagnostics_Technical_Report.md
├── patches/
│   ├── 0001-rover-board-configuration.patch
│   └── 0002-iomcu-probe-and-serial-diagnostics.patch
└── logs/
    └── build-c-thermal-observations.csv

# The *actual PX4 source* should live in a separate fork/branch of the PX4 repo,
# or be reconstructed from the exported, exact commit patches.
```

**Option A — Best for active development: PX4 fork with diagnostic branch.** Keep the entire PX4 Git history and existing submodules. Commit only deliberate changes. Push the `cubeorange-iomcu-diagnostics` branch to a fork whose owner/repository you control. Add this report at `docs/` (or in a dedicated docs repository) and commit it separately.

```bash
cd ~/orange_cube_rover_firmware/px4-rover

git status --short
git branch --show-current
git log -3 --oneline

git show --stat --oneline 2092c87c6a
git show --format=fuller --no-patch 2092c87c6a
```

Verify that `git status` is clean **before** adding unrelated files or pushing. Local backup files such as `rcS.backup` or `rover.px4board.backup`, machine-local build output, generated caches, secrets, and temporary logs should **not** be committed by accident.

After configuring a personal GitHub fork as a remote, for example `myfork` (replace its URL with the **real** URL):

```bash
# Example only; use your actual fork URL and verify existing remotes first.
git remote -v
git remote add myfork git@github.com:YOUR_ACCOUNT/PX4-Autopilot.git

git push -u myfork cubeorange-iomcu-diagnostics
```

Do **not** run `git remote add` blindly if `myfork` already exists. Publishing requires the user's own repository credentials and explicit push action. This report has **not** pushed anything to GitHub.

### 11.2 Option B — Export exact source modifications as reviewable patches

For an independent repository, archive the precise changes using the real Git commits:

```bash
cd ~/orange_cube_rover_firmware/px4-rover
mkdir -p ~/cubeorange-build-c-export/patches

git format-patch -2 2092c87c6a \
  -o ~/cubeorange-build-c-export/patches

# Or produce a single, directly inspectable diff covering both commits:
git diff  b377a2093b^ 2092c87c6a \
  > ~/cubeorange-build-c-export/build-c-complete.diff

# Verify patches were actually written and contain the expected files:
ls -lh ~/cubeorange-build-c-export/patches

git diff --stat b377a2093b^ 2092c87c6a
```

**Why include two commits?** The earliest commit establishes the custom Rover target. The following commit implements Build C diagnostics. A patch containing only `2092c87c6a` assumes the preceding `rover.px4board` file already exists. `git format-patch -2` includes both commits, preserving the progression and allowing a reviewer to reconstruct them in order against the same PX4 base. Use `git apply --check` or `git am` on a clean, compatible PX4 checkout before claiming the patches can be replayed successfully; this has not been tested here.

### 11.3 Capture all six changed paths exactly

```bash
git show --pretty='' --name-only 2092c87c6a
```

Expected file list:

```text
ROMFS/px4fmu_common/init.d/rcS
boards/cubepilot/cubeorange/rover.px4board
platforms/nuttx/src/px4/stm/stm32_common/include/px4_arch/px4io_serial.h
platforms/nuttx/src/px4/stm/stm32h7/px4io_serial/px4io_serial.cpp
src/drivers/px4io/px4io.cpp
src/drivers/px4io/px4io_serial.cpp
```

The exact file order in Git output may differ. The base commit also adds the Rover board file; use the two-commit patch range if publishing a complete Build C replay package.

### 11.4 Store firmware and evidence without obscuring its provenance

```bash
cd ~/orange_cube_rover_firmware/px4-rover

sha256sum build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4

git rev-parse HEAD

git status --short
```

Save the SHA-256 hash in release notes **after obtaining the actual command output**; this report intentionally does not fabricate one. Upload the `.px4` image to a clearly labeled **experimental** GitHub Release only after deciding to publish it, with warnings about the faulty hardware and nearly-full flash. Do not claim the artifact was formally safety tested.

### 11.5 Licenses, upstream attribution and security

PX4 source is distributed under its upstream licensing terms. Preserve original copyright notices, LICENSE and attribution; distinguish local modifications from upstream PX4, and verify the licenses of any bundled submodules, board-related code, dependencies or separately supplied material before redistribution. Do not include personal login credentials, radio encryption keys, private repository tokens, host identifiers, or unrelated source files in public logs or commits.

## 12. Recommended README summary for GitHub

The following short text can be placed in the diagnostic fork's root README:

> **Cube Orange Build C — Experimental IOMCU Diagnostics (PX4 1.17)**
>
> This branch builds `cubepilot_cubeorange_rover` for an original CubePilot Cube Orange (board ID 140). It enables the PX4 differential-rover module, excludes the ICM20649 driver for experimental isolation, includes PX4IO without normal boot-time PX4IO startup or automatic update, and adds `px4io probe` to perform a limited IOMCU protocol-version read with UART/DMA diagnostic counters. On the tested board, the read **failed** with two timeouts and one DMA/UART error. Two other IMUs continued reporting data but warmed together, and the board's thermal behavior remains unresolved. **This software does not repair the hardware; it must not be used to drive motors or aircraft.** See `docs/CubeOrange_Build_C_IOMCU_Diagnostics_Technical_Report.md` for source paths, procedure, raw data and limitations.

## 13. Evidence inventory

| Evidence | Location / description | Confidence |
|---|---|---|
| Source identity | Firmware package `git_hash` = `2092c87c6aed28e0c45580f0b061c2994d2923e7` | Direct output |
| Build success and memory use | Local PX4 build output | Direct output from development session |
| Auto-start disabled | Reviewed generated `build/.../etc/init.d/rcS` | Development-time generated-script inspection |
| No running PX4IO instance | `px4io status` | Direct NSH output |
| Probe failure and counters | `px4io probe` | Direct NSH output |
| Temperature history | User's pasted `listener sensor_accel 1` log | Direct onboard sensor readings |
| Later high-temperature reading | Separate QGroundControl capture and pasted shell output | Direct onboard sensor readings |
| Cause of uptime reset | Operator explicitly confirmed disconnecting USB | Operator confirmation |
| Exact code patch contents | To be exported with `git format-patch` / `git show` | **Not attached to this report** |
| IOMCU power/reset/UART waveform | Not measured | Unknown |
| Heater line voltage/current | Not measured | Unknown |
| Vehicle control, actuator outputs and safety | Not tested | Unknown |

## 14. Final assessment

**Build C achieved its software diagnostic objective:** it booted as PX4 on an original Cube Orange, kept the normal PX4IO driver stopped, executed a narrowly scoped protocol-version read through the existing PX4IO serial interface, and exposed useful low-level failure counters. However, **the IOMCU still failed to communicate**, and the unit's thermal behavior remains unexplained. A subsequent timestamp reset was due to **intentional USB disconnection**, not evidence of spontaneous instability.

The most productive continuation is **electrical verification of the IOMCU supply/reset, USART6 signaling, carrier connections, and heater control path**, using documented board schematics and appropriate lab equipment. Maintain this diagnostic firmware on an isolated branch and publish the exact source commits or patches with this report, so the experiment is reproducible and its limitations remain visible.

---

*Prepared from the recorded PX4 development process, build/package metadata, QGroundControl/NSH outputs, user-supplied thermal log, and earlier Cube Orange recovery documentation. Code excerpts marked representative are explanatory; authoritative changes reside in the identified Git commit.*
