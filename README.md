# CubePilot Cube Orange: PX4 1.17 Differential Rover Recovery, ICM20649 Isolation, PX4IO Failure, and Thermal Investigation

**Technical case study / reproducible build guide — October 17, 2026**  
**Status: experimental workaround, not a hardware repair**  
**Target: original CubePilot Cube Orange (PX4 board ID 140), not Cube Orange+**

> [!CAUTION]
> The Cube Orange described here has an **unresolved PX4IO/IOMCU communication failure** and **unresolved abnormal heating**. Both builds described below are **experimental, for stationary diagnostics only**. They are **not validated for flight, autonomous driving, motor control, or safety-critical use**. Disabling a driver does not repair a damaged sensor, MCU, heater, power circuit, or carrier board. Disconnect actuators during all initial tests. Do not leave a unit that heats abnormally powered unattended.

## 1. Executive summary

A second-hand original Cube Orange initially appeared unusable for ArduPilot because it consistently reported failure to update the IO firmware, while its inertial sensors appeared unreliable or unavailable. The owner wanted to reuse the board as the autopilot of a two-wheel differential-drive agricultural rover, eventually integrated with a Yahboom MicroROS V2, ROS 2, RTK GNSS, LiDAR, and a Raspberry Pi 5.

A standard **PX4 v1.17.0** firmware booted on the board, providing a useful diagnostic environment. Three gyro/accelerometer instances initially appeared. The instance driven by **ICM20649** (device ID **2424842**) reported around **79–87.5 °C**, notably hotter than the other IMUs. Stopping its driver with `icm20649 stop` removed that sensor from the active sensor topics, preserved attitude estimation using the other two sensors, and coincided with a marked reduction in perceived heating. This was promising but did **not** identify the original heat source.

To enable differential-drive vehicle support, two custom PX4 builds were created:

1. **Build A — Rover + ICM20649 omitted, PX4IO retained.** Successfully compiled, flashed, recognized as `Generic Rover Differential` (airframe **50000**), and ran `rover_differential`; both remaining IMUs and both EKF2 instances operated. However, PX4IO initialization and bootloader communication still failed.
2. **Build B — Rover + ICM20649 omitted + PX4IO omitted.** Successfully compiled and flashed; the previous PX4IO update errors disappeared from boot logs because the driver and its update attempts were not running. Rover, two IMUs, EKF2, and the AUX PWM software driver continued to operate. **However, both remaining IMUs reached approximately 67.3 °C shortly after startup.** The thermal problem is therefore **not fixed**.

Source inspection also established that the Cube Orange heater is controlled via the IO processor on the documented PX4 hardware path: the IO firmware defines **PB14** as the heater control output and its initialization sets it low/off. Without a functioning IOMCU, we do not know whether that initialization occurs or whether some other circuit is responsible for heating. The case presently ends at **hardware diagnosis required**, not at a complete recovery.

## 2. Motivation and application

The intended end use is a **differential-drive agricultural rover** for an organic olive grove. The development platform is a **Yahboom MicroROS V2** two-motor robot, with planned integration of:

- Cube Orange as the PX4 autopilot and MAVLink endpoint;
- Raspberry Pi 5 / ROS 2 for perception, local planning, and obstacle avoidance;
- LiDAR, camera, and ultimately RTK GNSS for field coverage;
- RFD868x telemetry to a Mac running QGroundControl;
- a future tracked chassis and mowing / vegetation-control tools.

A differential rover needs the PX4 **`rover_differential`** application and its airframe/startup definitions. The standard PX4 firmware previously installed on this Cube Orange lacked the active rover module, so simply changing an airframe in QGroundControl was not sufficient.

The design goal was to **preserve the available aircraft controllers** (multicopter, fixed-wing, and VTOL) in the custom build, in case the hardware was later repaired or sold. Their presence in a binary must **not** be interpreted as certification of the damaged board for aircraft use.

## 3. Hardware and software environment

| Item | Observed configuration |
|---|---|
| Flight controller | Original CubePilot **Cube Orange**, with carrier |
| PX4 board ID | **140** |
| Bootloader | Version **5**, identified by QGroundControl in earlier testing |
| MCU | STM32H7 family, rev. V per PX4 boot log |
| PX4 source branch | `release/1.17` |
| PX4 Git identity | `v1.17.0-13-gf65ddce539` |
| Operating system on target | NuttX 12.12.0 |
| Build host | Ubuntu **22.04 ARM64**, Builder virtual machine |
| GCC toolchain | `arm-none-eabi-gcc` **10.3.1** |
| Python | Python **3.10**, dedicated virtual environment |
| Ground station | QGroundControl on macOS |
| Telemetry | RFD868x, connected via TELEM1 in previous tests, 57,600 baud |
| GNSS reported at boot | u-blox **NEO-M8N**, firmware SPG 3.01; this is not an RTK receiver |
| Source directory | `/home/yahboom/orange_cube_rover_firmware/px4-rover` |
| Virtual environment | `/home/yahboom/orange_cube_rover_firmware/px4-venv` |

The two available flash budgets in the build logs used a **1920 KiB linker FLASH region = 1,966,080 bytes**. The `.px4` distribution file's size is not identical to the linker FLASH usage.

## 4. Initial symptoms and observations

### 4.1 ArduPilot symptoms

Earlier attempts using ArduRover, including versions **4.6.3 / 4.7.2** during troubleshooting, repeatedly yielded errors similar to:

```text
Config Error: Failed to update IO firmware
IOMCU: 0 0 0
INS_ACC_ID=0
INS_GYR_ID=0
```

Attempts to reflash, reboot while pressing the safety button, and reseat the Cube on its 80-pin connector did not demonstrably resolve this. The Cube Orange had also appeared defective in imagery supplied by the seller. This is not proof of the failure's precise origin.

### 4.2 Standard PX4 booted and made sensor debugging possible

The official PX4 v1.17.0 firmware started successfully, unlike the problematic ArduPilot initialization. It exposed three IMU instances:

| Instance | Device ID | Observed behavior |
|---|---:|---|
| IMU 0 | `3670050` | Responsive; previously around 58–68 °C |
| IMU 1 | `2621474` | Responsive; previously around 58–68 °C |
| IMU 2 / ICM20649 | `2424842` | Repeatedly around **79–87.5 °C**; one transient gyro `error_count: 10` |

The identity of IMU 2 was additionally supported by:

```text
icm20649 status
```

which showed an active driver on SPI bus 1. After:

```text
icm20649 stop
icm20649 status
```

PX4 reported that the driver was no longer running. Both `listener sensor_gyro` and `listener sensor_accel` then returned **two**, rather than three, instances. `listener vehicle_attitude` continued to update when the board was manually moved. The case owner also noticed less heating after stopping that driver.

**Important limitation:** `icm20649 stop` is not persistent after reboot, and stopping sensor reads does **not** prove that a sensor or nearby heater is physically unpowered. The observed temperature reduction was a correlation, not a complete electrical diagnosis.

The earlier PX4 configuration also showed:

```text
heater status
INFO [heater] not running

px4io status
INFO [px4io] not running
```

and software status for six PWM AUX channels, without confirming physical electrical outputs.

## 5. Building a custom rover variant

### 5.1 Clone the PX4 sources

```bash
cd ~/orange_cube_rover_firmware
git clone --branch release/1.17 --recurse-submodules \
    https://github.com/PX4/PX4-Autopilot.git px4-rover
cd px4-rover

git branch --show-current
git describe --tags --always
```

The observed output was:

```text
release/1.17
v1.17.0-13-gf65ddce539
```

For reproducibility, a later checkout on a moving `release/1.17` branch might produce a different commit. Use the exact commit `f65ddce539` if reproducing this specific experiment, with appropriate submodules, rather than assuming HEAD is unchanged.

### 5.2 Available Cube Orange board variants and rover modules

```bash
find boards/cubepilot/cubeorange -name '*.px4board'
find src/modules -maxdepth 2 -iname '*rover*'
```

Observed board variants:

```text
bootloader.px4board
console.px4board
default.px4board
test.px4board
```

Observed rover modules:

```text
src/modules/rover_differential
src/modules/rover_ackermann
src/modules/rover_mecanum
```

There was no pre-existing `rover.px4board` for this board, so we created a separate variant rather than editing `default.px4board`.

### 5.3 Build A configuration: Rover + no ICM20649

```bash
cd ~/orange_cube_rover_firmware/px4-rover
cp boards/cubepilot/cubeorange/default.px4board \
   boards/cubepilot/cubeorange/rover.px4board
```

In `boards/cubepilot/cubeorange/rover.px4board`, change:

```ini
CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649=y
```

to:

```ini
# CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649 is not set
```

and add:

```ini
CONFIG_MODULES_ROVER_DIFFERENTIAL=y
```

Build A intentionally retained:

```ini
CONFIG_DRIVERS_PX4IO=y
CONFIG_DRIVERS_PWM_OUT=y
CONFIG_MODULES_EKF2=y
CONFIG_MODULES_MAVLINK=y
```

It also retained aircraft-specific applications including MC, FW, and VTOL modules. The resulting binary was therefore close to the Cube Orange's flash limit.

### 5.4 Python build dependencies under ARM64 Ubuntu Builder

We used a project-specific Python virtual environment to avoid changing the Yahboom ROS 2 installation:

```bash
python3 -m venv ~/orange_cube_rover_firmware/px4-venv
source ~/orange_cube_rover_firmware/px4-venv/bin/activate
python -m pip install --upgrade pip
```

Packages missing in the actual build attempts included:

| Import or generator failure | Package used |
|---|---|
| `No module named 'menuconfig'` | `kconfiglib` |
| `No module named 'em'` | `empy==3.3.4` |
| `No module named 'yaml'` | `PyYAML` |
| `No module named 'jsonschema'` | `jsonschema` |
| `No module named 'genmsg'` | `pyros-genmsg` |
| `No module named 'jinja2'` | `jinja2` |

Example installation:

```bash
python -m pip install \
  kconfiglib 'empy==3.3.4' PyYAML jsonschema pyros-genmsg jinja2
```

The log also warned about missing `lxml`; XML validation of some MAVLink definitions was skipped, but this warning did not prevent the build from completing. For other environments, use the actual dependency manifests included with the checked-out PX4 revision rather than assuming the list above is exhaustive.

**Virtual environment pitfall:** `pip install --user ...` fails inside a normal virtual environment because user site-packages are inaccessible. Use `python -m pip install ...` **without `--user`** while the virtual environment is activated.

### 5.5 Build and inspect the artifact

```bash
cd ~/orange_cube_rover_firmware/px4-rover
source ~/orange_cube_rover_firmware/px4-venv/bin/activate
make cubepilot_cubeorange_rover
```

A successful run ended with:

```text
[1258/1260] Linking CXX executable cubepilot_cubeorange_rover.elf
FLASH: 1954992 B / 1920 KB = 99.44%
[1260/1260] Creating .../cubepilot_cubeorange_rover.px4
```

The artifact was located at:

```text
build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4
```

File size displayed by `ls -lh`: approximately **1.8 MB**.

The generated Kconfig was checked with:

```bash
grep -E 'CONFIG_MODULES_ROVER_DIFFERENTIAL|CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649' \
  build/cubepilot_cubeorange_rover/boardconfig
```

Observed:

```ini
# CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649 is not set
CONFIG_MODULES_ROVER_DIFFERENTIAL=y
```

We also inspected the JSON metadata of the `.px4` package:

```bash
python3 - <<'PY'
import json
from pathlib import Path
p = Path('build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4')
d = json.loads(p.read_text())
for key in ('board_id', 'description', 'git_identity'):
    print(f'{key}: {d.get(key)}')
PY
```

Output:

```text
board_id: 140
description: Firmware for the CubePilot CubeOrange board
git_identity: v1.17.0-13-gf65ddce539
```

**Board ID 140 is a necessary compatibility check, not proof of safe hardware operation.** Do not flash this on Orange+ or another board merely because their names are similar.

## 6. Ensuring the differential rover actually starts

The PX4 source already contained:

```text
ROMFS/px4fmu_common/init.d/airframes/50000_generic_rover_differential
ROMFS/px4fmu_common/init.d/rc.rover_differential_defaults
ROMFS/px4fmu_common/init.d/rc.rover_differential_apps
ROMFS/px4fmu_common/init.d/rc.vehicle_setup
```

The module startup was located with:

```bash
grep -R -n 'rover_differential' ROMFS/px4fmu_common/init.d
```

Relevant relationships:

```text
Airframe 50000
  -> rc.rover_differential_defaults
       -> set VEHICLE_TYPE rover_differential
  -> rc.vehicle_setup
       -> rc.rover_differential_apps
            -> rover_differential start
```

The generated build tree also contained:

```text
build/cubepilot_cubeorange_rover/etc/init.d/airframes/50000_generic_rover_differential
```

After flashing Build A with QGroundControl, the vehicle was recognized as a Rover. The runtime console reported:

```text
rover_differential status
INFO [rover_differential] running
```

The actual boot log contained:

```text
Loading airframe: /etc/init.d/airframes/50000_generic_rover_differential
```

This establishes that Rover functionality was successfully built **and** started, not merely present as a file on the host.

## 7. Build A — observations after flashing

### 7.1 ICM20649 no longer available

```text
nsh> icm20649 status
nsh: icm20649: command not found
```

Two gyro and two accelerometer instances remained.

Representative Build A measurements:

| Approximate PX4 uptime | IMU 0 temperature | IMU 1 temperature | Error counts |
|---:|---:|---:|---|
| 122 s | 38.43 °C | 38.30 °C | 0 / 0 |
| 257 s | 44.68 °C | 44.63 °C | 0 / 0 |
| 282 s | 45.76 °C | 45.25 °C | 0 / 0 |
| 347 s | 48.49 °C | 48.27 °C | 0 / 0 |

The increase showed that the unit was still warming over that observation window. It did not establish a steady-state temperature.

### 7.2 Attitude estimation remained available

One `listener vehicle_attitude` result showed:

```text
Roll: 0.3 deg, Pitch: -2.1 deg, Yaw: -5.3 deg
```

The two EKF2 instances reported their respective IMU IDs and `healthy`:

```text
0: ACC: 3670050, GYRO: 3670050, MAG: 592905, healthy
1: ACC: 2621474, GYRO: 2621474, MAG: 592905, healthy
```

The EKF reported `attitude: 1` but `local position: 0, global position: 0` in this stationary test. GNSS / heading conditions were insufficient for navigation. A healthy EKF estimate **does not certify sensor hardware integrity or thermal safety**.

### 7.3 PX4IO failures remained

Build A boot logs clearly showed:

```text
WARN  [px4io] check CRC failed: -1, CRC: 3283766933
WARN  [px4io] IO refused to reboot
WARN  [px4io] reboot failed - -1, still attempting upgrade
[PX4IO] using firmware from /etc/extras/cubepilot_io-v2_default.bin
[PX4IO] bootloader not responding
ERROR [px4io] error updating PX4IO - check that bootloader mode is enabled
ERROR [px4io] Failed to communicate with IO, abort.
ERROR [px4io] Task start failed (-1)
PX4IO start failed
```

This independently reproduced the broad class of communication/update failure previously seen under ArduPilot. Causes **not determined**: IO MCU itself, IO power/reset/clock, firmware/bootloader, module-to-carrier contacts, or another board-level fault.

### 7.4 AUX PWM software driver remained available

```text
pwm_out status
INFO [mixer_module] Param prefix: PWM_AUX
```

It listed **six AUX channels**, initially with `func: 0` and `value: 1000`, at **400 Hz** on two timer groups. These are **software status observations only**. No oscilloscope, logic analyzer, voltmeter, ESC, servo, or motor test confirmed valid output pulses at physical connector pins.

## 8. Build B — remove the PX4IO driver to avoid failing initialization

### 8.1 Why this second build was made

The original PX4 startup script, `ROMFS/px4fmu_common/init.d/rcS`, contained a PX4IO section around line 389:

```sh
if px4io supported
then
    if [ -f $IOFW ]
    then
        if ! px4io checkcrc ${IOFW}
        then
            # Attempt PX4IO firmware update.
            ...
        fi
        if ! px4io start
        then
            echo "PX4IO start failed"
            set STARTUP_TUNE 2
        fi
    fi
fi
```

This explains the repeated CRC, reboot, bootloader, and update messages on the defective unit. Instead of modifying the shared `rcS` script, we tested a board-specific Kconfig variant that omitted the PX4IO driver entirely.

### 8.2 Build B config difference

Keep the Build A Rover and IMU configuration, and change:

```ini
CONFIG_DRIVERS_PX4IO=y
```

to:

```ini
# CONFIG_DRIVERS_PX4IO is not set
```

The resulting generated `boardconfig` reported:

```ini
# CONFIG_DRIVERS_IMU_INVENSENSE_ICM20649 is not set
CONFIG_DRIVERS_PCA9685_PWM_OUT=y
CONFIG_DRIVERS_PWM_OUT=y
# CONFIG_DRIVERS_PX4IO is not set
# CONFIG_MODULES_PX4IOFIRMWARE is not set
CONFIG_MODULES_ROVER_DIFFERENTIAL=y
```

**Notably:** the Cube Orange board configuration still referenced `CONFIG_BOARD_IO="cubepilot_io-v2_default"`; the observed build system nonetheless generated Build B successfully. Removing the driver does not repair or physically disable the IO MCU or the heater circuitry.

### 8.3 Build B result

```text
[1235/1237] Linking CXX executable cubepilot_cubeorange_rover.elf
FLASH: 1937484 B / 1920 KB = 98.55%
[1237/1237] Creating .../cubepilot_cubeorange_rover.px4
```

Build B saved **17,508 bytes** of linked FLASH relative to Build A. It still used most of the flash budget.

The embedded package metadata again showed:

```text
board_id: 140
description: Firmware for the CubePilot CubeOrange board
git_identity: v1.17.0-13-gf65ddce539
```

After flashing Build B:

```text
nsh> rover_differential status
INFO [rover_differential] running

nsh> px4io status
nsh: px4io: command not found
```

The log loaded airframe 50000. Importantly, **there were no PX4IO firmware update or bootloader errors in the observed Build B startup log**. This is expected because the PX4IO command is absent; it is **not** evidence that the IOMCU has recovered.

The new boot log still reported other warnings, including major compass inconsistency (roughly 121–148 degrees), excessive GNSS horizontal position error, and a MAVLink `stat failed read` message. These are not solved by the PX4IO exclusion. The case also contained two `bsondump` attempts failing to open `/fs/mtd_caldata` during startup; their significance has not been established.

### 8.4 Critical thermal outcome: Build B is not a heat fix

At PX4 uptime around **147.6 s**, the Build B gyros reported:

| Device ID | Temperature | Errors |
|---:|---:|---:|
| `3670050` | **67.35 °C** | 0 |
| `2621474` | **67.29 °C** | 0 |

Both EKF2 instances continued to say `healthy`, and `pwm_out status` still showed six AUX channels. Nevertheless, **the two remaining sensors were hot**. Temperature comparisons between Build A and Build B are suggestive, not controlled: room temperature, USB supply, uptime history, handling, and other variables were not formally matched.

**Do not publish this as a solved overheating defect.** The verified result is a successful **software isolation of two failing subsystems**, followed by discovery that significant heating persists.

## 9. Heater architecture: what the actual PX4 source shows

The investigation turned to the Cube Orange heater because multiple surviving IMUs warmed similarly even with the ICM20649 driver absent.

### 9.1 The heater is controlled through PX4IO on this hardware

In:

```text
boards/cubepilot/cubeorange/src/board_config.h
```

we found:

```c
#define PX4IO_HEATER_ENABLED
```

In:

```text
src/drivers/heater/heater.cpp
```

we found:

```cpp
#if defined(BOARD_USES_PX4IO_VERSION) and defined(PX4IO_HEATER_ENABLED)
// Heater on some boards is on IO MCU
// Use ioctl calls to IO driver to turn heater on/off
# define HEATER_PX4IO
#else
// Use direct calls to turn GPIO pin on/off
...
#endif
```

Thus, in the intended PX4 configuration, the flight management MCU commands the heater via the IO driver. The calls use:

```cpp
px4_ioctl(_io_fd, PX4IO_HEATER_CONTROL, HEATER_MODE_OFF);
px4_ioctl(_io_fd, PX4IO_HEATER_CONTROL, HEATER_MODE_ON);
px4_ioctl(_io_fd, PX4IO_HEATER_CONTROL, HEATER_MODE_DISABLED);
```

If the IO communication path is unavailable, commands cannot be assumed to reach the heater hardware.

### 9.2 IO firmware GPIO definition and startup behavior

In:

```text
boards/cubepilot/io-v2/src/board_config.h
```

PX4 defines:

```c
#define GPIO_HEATER_OUTPUT /* PB14 */ \
  (GPIO_OUTPUT|GPIO_CNF_OUTPP|GPIO_MODE_50MHz|GPIO_OUTPUT_CLEAR|GPIO_PORTB|GPIO_PIN14)
#define HEATER_OUTPUT_EN(on_true) stm32_gpiowrite(GPIO_HEATER_OUTPUT, (on_true))
```

`GPIO_OUTPUT_CLEAR` specifies the low initial output configuration **when firmware configures that pin**.

In:

```text
boards/cubepilot/io-v2/src/init.c
```

IO initialization contains:

```c
stm32_configgpio(GPIO_HEATER_OUTPUT);
...
/* disable heater */
HEATER_OUTPUT_EN(false);
```

**Consequence:** if this IO firmware boots and completes initialization, its intended state is heater OFF. We have **not** established that the defective IO MCU reaches this initialization routine.

### 9.3 Main PX4 driver's heater control registers

In:

```text
src/drivers/px4io/px4io.cpp
```

we found:

```cpp
case PX4IO_HEATER_CONTROL:
    if (arg == (unsigned long)HEATER_MODE_DISABLED) {
        io_reg_set(PX4IO_PAGE_SETUP, PX4IO_P_SETUP_THERMAL, PX4IO_THERMAL_IGNORE);
    } else if (arg == 1) {
        io_reg_set(PX4IO_PAGE_SETUP, PX4IO_P_SETUP_THERMAL, PX4IO_THERMAL_FULL);
    } else {
        io_reg_set(PX4IO_PAGE_SETUP, PX4IO_P_SETUP_THERMAL, PX4IO_THERMAL_OFF);
    }
    break;
```

Constants found in:

```text
src/modules/px4iofirmware/protocol.h
```

were:

```c
#define PX4IO_THERMAL_IGNORE UINT16_MAX
#define PX4IO_THERMAL_OFF    0
#define PX4IO_THERMAL_FULL   10000
```

This documents command intent, but cannot demonstrate the state of a physically malfunctioning IO processor.

### 9.4 IO firmware's default and control loop

The setup register defaults to:

```c
[PX4IO_P_SETUP_THERMAL] = PX4IO_THERMAL_IGNORE,
```

in `src/modules/px4iofirmware/registers.c`.

The IO firmware's main loop, in `src/modules/px4iofirmware/px4io.cpp`, contains:

```cpp
#if defined(HEATER_OUTPUT_EN)
    if (r_page_setup[PX4IO_P_SETUP_THERMAL] != PX4IO_THERMAL_IGNORE) {
        if (r_page_setup[PX4IO_P_SETUP_THERMAL] < PX4IO_THERMAL_FULL) {
            /* switch resistive heater off */
            HEATER_OUTPUT_EN(false);
        } else {
            /* switch resistive heater hard on */
            HEATER_OUTPUT_EN(true);
        }
    }
#endif
```

Meaning:

| Register state | Behavior of a functioning IO loop |
|---|---|
| `IGNORE` / 65535 | Leave the current GPIO state unchanged |
| `OFF` / 0 | Drive heater output OFF |
| `FULL` / 10000 | Drive heater output ON |

An important nuance: `IGNORE` **does not mean turn the heater on**. With normal initialization, the output would already have been cleared/off. It only means the loop refrains from changing it.

### 9.5 What is and is not established

**Established from observed source:**

- IO firmware has a defined heater GPIO on PB14.
- IO initialization explicitly configures this pin off.
- IO's register defaults to `IGNORE`, which does not override the initial state.
- A correctly operating IO implementation can be instructed to turn the heater on or off.
- The FMU-to-IO communication path is failing on this physical board.

**Not established:**

- That the heater is physically energized.
- That the GPIO is high while the IO MCU is nonfunctional.
- That the IO processor never initializes at all.
- That disabling the PX4IO driver changes the electrical power supplied to the heater.
- That heater malfunction, rather than a regulator/sensor/power fault, is causing the observed temperatures.
- That 67 °C is solely a heater symptom; the measurement is an **internal IMU sensor temperature**, not a calibrated measurement of the whole board.

**Current hypothesis, not proven:** a common thermal source may be affecting both sensors, with the IO/heater subsystem a candidate because of the concurrent IO failure. Alternative causes include a damaged power regulator, leakage/current draw, local component damage, or other thermal coupling.

## 10. Reproduction and validation commands

**On Ubuntu Builder:**

```bash
cd ~/orange_cube_rover_firmware/px4-rover
source ~/orange_cube_rover_firmware/px4-venv/bin/activate

git describe --tags --always

grep -E 'ICM20649|PX4IO|PWM_OUT|ROVER_DIFFERENTIAL' \
  build/cubepilot_cubeorange_rover/boardconfig

find build/cubepilot_cubeorange_rover/etc/init.d/airframes \
  -name '50000_generic_rover_differential'

make cubepilot_cubeorange_rover
```

**At the target's QGroundControl MAVLink Console (NSH), no actuators connected:**

```text
dmesg
rover_differential status
icm20649 status
px4io status
listener sensor_gyro
listener sensor_accel
listener vehicle_attitude
ekf2 status
pwm_out status
heater status
```

Expected from **Build A**: rover running, `icm20649` absent, `px4io` command present but runtime initialization failing on this defective hardware.

Expected from **Build B**: rover running, both `icm20649` and `px4io` commands absent, no PX4IO upgrade attempt in boot log, both remaining IMUs reported, AUX PWM software driver active. **Thermal safety remains unconfirmed.**

**Console quirk:** PX4's NSH `dmesg` does not behave like a full Linux shell pipeline in this setup. `dmesg | grep ...` printed the `dmesg` usage information rather than filtering the output. Run `dmesg` by itself and inspect the resulting text. Paste commands individually at a clean `nsh>` prompt to avoid accidental input corruption.

## 11. How to preserve and publish BOTH firmware images

Two separate `.px4` packages were built at different times with the **same default output filename**. That output path was overwritten when Build B was compiled. Therefore, **do not assume both packages still exist** just because both were successfully flashed.

Recommended filenames:

```text
CubeOrange_PX4_1.17_Rover_NoICM20649_WithPX4IO.px4
CubeOrange_PX4_1.17_Rover_NoICM20649_NoPX4IO.px4
```

### 11.1 Save Build B (current on-disk build)

```bash
cd ~/orange_cube_rover_firmware/px4-rover
mkdir -p ~/orange_cube_rover_firmware/releases
cp build/cubepilot_cubeorange_rover/cubepilot_cubeorange_rover.px4 \
   ~/orange_cube_rover_firmware/releases/CubeOrange_PX4_1.17_Rover_NoICM20649_NoPX4IO.px4
```

### 11.2 Recover Build A if saved separately

If the **first** `.px4` was saved in macOS Downloads, as described during testing, copy it back into the `releases` folder and name it distinctly. **Verify its actual configuration** rather than relying on filenames alone: inspect `board_id` and preserved build/config evidence, and do not relabel an unknown binary as Build A.

If no copy of Build A remains, reproduce it from source:

1. Preserve the current Build B `.px4` first.
2. Copy the current `rover.px4board` configuration to a safe file.
3. In `rover.px4board`, restore **only** `CONFIG_DRIVERS_PX4IO=y`; leave `ICM20649` disabled and Rover Differential enabled.
4. Rebuild using `make cubepilot_cubeorange_rover`.
5. Save and rename the resulting `.px4` as Build A.
6. Restore Build B's `.px4board` configuration afterward if it is to remain the active working version.

Do not make any of these changes while flashing or operating the hardware.

### 11.3 Check package metadata and checksums before release

```bash
python3 - <<'PY'
import json
from pathlib import Path
for p in sorted(Path.home().joinpath('orange_cube_rover_firmware/releases').glob('*.px4')):
    info = json.loads(p.read_text())
    print(p.name, '| board_id=', info.get('board_id'), '| git=', info.get('git_identity'))
PY

sha256sum ~/orange_cube_rover_firmware/releases/*.px4
```

Include actual SHA-256 values in release notes after computing them. Do **not** invent checksums or imply that both published images have been independently tested if one was recreated later.

### 11.4 GitHub repository structure

A practical layout:

```text
cubeorange-px4-rover-investigation/
├── README.md
├── docs/
│   └── CubeOrange_PX4_Rover_Recovery_Case_Study.md
├── configs/
│   ├── rover_build_A.px4board
│   └── rover_build_B.px4board
├── logs/
│   ├── build_A_boot.txt
│   ├── build_B_boot.txt
│   └── thermal_observations.csv
└── releases/                  # optional local staging;
    ├── ...WithPX4IO.px4       # prefer GitHub Release assets
    └── ...NoPX4IO.px4
```

For sharing downloadable firmware, **GitHub Releases assets** are convenient; they avoid committing generated binary builds to Git history. The repository should include the exact source commit, Kconfig differences, build instructions, hardware ID, measured thermal data, and the warnings prominently displayed at the top of this document. Use a license compatible with applicable PX4 upstream licensing and preserve relevant notices if distributing modified binaries/source.

Suggested GitHub Release wording:

> **EXPERIMENTAL — NOT FOR FLIGHT OR ACTUATOR USE.** These PX4 1.17 images target original Cube Orange board ID 140 and were built to investigate an unresponsive PX4IO/IOMCU and an overheating sensor path. The `NoPX4IO` variant suppresses the IO update attempt but does not repair the IO controller. Remaining IMUs have reached approximately 67 °C. No physical AUX PWM output, full thermal stability, flight, or motor-control validation has been completed. Use solely for isolated bench diagnostics at your own risk.

**Security/privacy review before posting logs:** remove any personal paths or accounts you do not wish to publish, serial numbers/GUIDs if sensitive, telemetry credentials, RTK/NTRIP credentials, tokens, Wi-Fi passwords, and identifying field coordinates.

## 12. Next diagnostic steps — safely and with proper equipment

### 12.1 Do not confuse software isolation with repair

Build B suppresses IOMCU startup/update messages by **removing the driver**. This makes the boot log cleaner but reduces hardware functionality and diagnostic visibility. It does not prove the IOMCU is healthy, does not physically switch off its heater, and does not restore MAIN outputs.

### 12.2 Confirm the thermal source before extended tests

Recommended follow-up for a qualified electronics technician:

1. With the board **unpowered**, inspect the module-to-carrier interface, connectors, visible damage, and signs of corrosion or liquid ingress.
2. Verify the carrier and module power rails, IO processor power/reset/clock, and relevant current draw using appropriate tools and CubePilot board documentation. Do not probe fine-pitch pins casually or inject power into the 80-pin connector.
3. If safely possible, use a thermal camera or carefully deployed contact probe to distinguish a warm IMU from a hot regulator or heater region. Do not infer board surface temperature from IMU telemetry alone.
4. Investigate IO MCU startup/bootloader communication and associated circuitry without automatically forcing firmware updates or bypassing safety systems.
5. Determine the heater control circuit's **electrical fail-safe state** when PB14 is undriven; the PX4 source alone cannot answer this.
6. Repeat short, supervised thermal measurements under controlled supply, ambient temperature, and fixed time intervals **only after electrical checks**.
7. Diagnose compass inconsistency and GNSS positional error separately before any attempt at navigation.
8. Validate AUX signals with a suitable logic analyzer/oscilloscope and a safe test load, subject to the board being electrically and thermally cleared. `pwm_out status` by itself is insufficient.

An experienced technician may find that the PX4IO failure and heating have separate causes. No hardware-level repair is documented in this case.

### 12.3 No release for flight or motors yet

Neither the presence of MC/FW/VTOL controllers nor successful `EKF2 healthy` output makes either binary flight-safe. The original board's IOMCU and heat issue make flight use inappropriate until both faults are diagnosed and repaired, all relevant outputs and redundancies are validated, and a standard supported firmware is restored and qualified as necessary.

## 13. Outcome at publication date

**What worked:**

- Built PX4 1.17 for original Cube Orange board ID 140 on Ubuntu 22.04 ARM64.
- Activated `Generic Rover Differential` (airframe 50000) and confirmed `rover_differential running`.
- Excluded the hot ICM20649 driver and preserved two reporting IMUs and two healthy EKF2 instances.
- Retained telemetry and a running AUX PWM software driver.
- Reproduced the PX4IO firmware/bootloader communication fault under Build A.
- Eliminated PX4IO **startup attempts and related error messages** under Build B by excluding the PX4IO driver.
- Identified the source-level PX4IO-controlled heater implementation and its intended OFF startup state.

**What did not work / remains unknown:**

- Neither firmware repaired the IOMCU, its bootloader, or MAIN outputs.
- Neither firmware proved the physical AUX pins are working.
- The actual thermal source is **unknown**, and IMUs reached **about 67 °C even in Build B**.
- GNSS precision and compass consistency were insufficient for autonomous missions in these tests.
- No electrical validation, rover driving test, endurance test, or flight qualification has occurred.

**Bottom line:** this is a **useful reproducible PX4 software workaround and diagnostic case study**, not a complete Cube Orange repair. It should help another developer distinguish an unavailable PX4 Rover module, a suspect IMU driver, an unresponsive IOMCU, a missing heater-control path, and a genuinely unresolved hardware temperature problem.

---

### Source and reproducibility notes

This report is based on terminal logs and source excerpts observed during the investigation of a single Cube Orange, using PX4 commit `f65ddce539`. Some conditions, such as ambient temperature, exact electrical supply, physical heater status, and manufacturing revision, were not measured and must not be assumed. Important paths were inspected directly in the checked-out source tree; readers should confirm them against the specific PX4 commit they use.

PX4 source: <https://github.com/PX4/PX4-Autopilot>  
PX4 documentation: <https://docs.px4.io/>  
QGroundControl: <https://qgroundcontrol.com/>
