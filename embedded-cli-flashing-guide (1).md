# mcu-tool: Embedded CLI & Flashing Tool Guide

> **Portfolio note:** This is an illustrative documentation sample. `mcu-tool` is a designed CLI used to demonstrate help-text writing, error-message design, and technical evaluation of AI-generated output. Protocol details (ESP32 ROM loader, STM32 UART bootloader per ST AN3155) reflect real hardware behaviour.

`mcu-tool` is a command-line utility for detecting, flashing, and verifying firmware on ESP32 and STM32 microcontrollers over a serial connection.

## Contents

1. [CLI Help Text](#1-cli-help-text)
2. [Error Message Matrix](#2-error-message-matrix)
3. [Diff Analysis: Evaluating Two AI-Generated Outputs](#3-diff-analysis-evaluating-two-ai-generated-outputs)

---

## 1. CLI Help Text

### Writing principles

Every help screen in `mcu-tool` follows the same rules:

- **Lead with what the command does**, in one line, in the imperative.
- **Show the default for every optional flag**, so users never have to guess.
- **Put warnings where they apply**, next to the flag that causes the risk, not in a footer nobody reads.
- **End with examples**, ordered from most common to least common.

### Top-level help

```text
$ mcu-tool --help
Usage: mcu-tool <command> [options]

Detect, flash, and verify firmware on ESP32 and STM32 microcontrollers.

Commands:
  scan       List connected devices and detect chip type and baud rate
  flash      Write a firmware image to device flash
  verify     Compare device flash against a local image
  erase      Erase flash sectors or the entire chip
  info       Show chip ID, flash size, and protection status

Global options:
  -p, --port <PORT>      Serial port (e.g. /dev/ttyUSB0, COM5)
                         Default: auto-detect if exactly one device is found
  -v, --verbose          Print every protocol command and response
  -q, --quiet            Print errors only
      --version          Show version and exit
  -h, --help             Show this help and exit

Run 'mcu-tool <command> --help' for command-specific options.
```

### `flash` command help

```text
$ mcu-tool flash --help
Usage: mcu-tool flash [options] <IMAGE>

Write a firmware image (.bin or .hex) to device flash.

Arguments:
  <IMAGE>                   Path to the firmware file. For .hex files, the
                            load address is read from the file and
                            --address is ignored.

Options:
  -t, --target <CHIP>       Target chip family.
                            Values: esp32, esp32s3, esp32c3, stm32f1, stm32f4
                            Default: auto-detect via 'scan'
  -b, --baud <RATE>         Baud rate for data transfer.
                            Default: 115200
                            ESP32 targets support up to 921600 with most
                            USB-serial adapters. STM32 UART bootloader
                            auto-detects rates from 1200 to 115200.
  -a, --address <HEX>       Flash start address for .bin images.
                            Default: 0x10000 (ESP32 app partition)
                                     0x08000000 (STM32 main flash)
  -e, --erase <MODE>        Erase strategy before writing.
                            Values: sectors | all | none
                            Default: sectors (only sectors the image covers)
                            WARNING: 'all' erases NVS, Wi-Fi credentials, and
                            any calibration data stored in flash.
      --no-verify           Skip read-back verification after writing.
                            Default: verify enabled
      --reset <MODE>        Reset behaviour after flashing.
                            Values: hard | none
                            Default: hard (toggle RTS/DTR to reboot the chip)
      --dry-run             Validate the image and connection, then exit
                            without writing anything.
  -h, --help                Show this help and exit

Notes:
  ESP32 (original): bootloader is at 0x1000, not 0x0. Writing a bootloader
  image to the wrong offset leaves the chip unable to boot.
  ESP32-S3 / ESP32-C3: bootloader is at 0x0.

  STM32: the chip must be in system-bootloader mode (BOOT0 high during
  reset). If read-out protection (RDP) is active, flashing fails until it
  is removed, and removing RDP Level 1 mass-erases the device.

Examples:
  mcu-tool flash firmware.bin
  mcu-tool flash -p /dev/ttyUSB0 -b 921600 -t esp32 app.bin
  mcu-tool flash -t stm32f4 -a 0x08000000 build/motor_ctrl.bin
  mcu-tool flash --dry-run --erase all firmware.bin
```

### `scan` command help

```text
$ mcu-tool scan --help
Usage: mcu-tool scan [options]

List serial devices and identify the microcontroller on each one.

Options:
      --probe-baud          Try common baud rates (9600, 57600, 115200,
                            460800, 921600) on ports that don't answer at
                            the default rate.
                            Default: off
      --timeout <MS>        Wait time per probe attempt.
                            Default: 500
  -h, --help                Show this help and exit

Warning:
  Probing sends sync bytes to every port. Do not run 'scan' while another
  program (e.g. a serial monitor or ROS node) is using a port.
```

---

## 2. Error Message Matrix

### The format standard

Every `mcu-tool` error follows one structure:

```
Error [E<code>]: <what failed> on <where>. <evidence>. <next action>.
```

| Part | Purpose |
|---|---|
| **Code** | Stable and searchable. Users can paste it into an issue or a search engine. |
| **What failed** | The specific operation, not a generic category. |
| **Where** | The port, address, or file involved. |
| **Evidence** | The observed value versus the expected value, when one is available. |
| **Next action** | One concrete command or physical step the user can take. |

### Poor error vs. good error

| # | Situation | Poor error | Good error |
|---|---|---|---|
| E101 | Port does not exist | `Error: Could not open port` | `Error [E101]: Port /dev/ttyUSB0 not found. 0 serial devices detected. Check the USB cable, then run 'mcu-tool scan'.` |
| E102 | Baud rate mismatch | `Error: Serial communication failed` | `Error [E102]: Baud rate mismatch on /dev/ttyUSB0. Expected 115200, detected 9600. Run 'mcu-tool scan --probe-baud' to auto-detect.` |
| E103 | Linux permission denied | `Error: Permission denied` | `Error [E103]: No permission to open /dev/ttyUSB0. Current user is not in group 'dialout'. Run 'sudo usermod -aG dialout $USER', then log out and back in.` |
| E201 | ESP32 not in download mode | `Error: Failed to connect` | `Error [E201]: ESP32 on COM5 did not respond to sync after 7 attempts. Chip is likely running firmware, not the ROM bootloader. Hold BOOT, tap EN, release BOOT, then retry.` |
| E202 | STM32 bootloader no ACK | `Error: Timeout` | `Error [E202]: No ACK (0x79) to sync byte 0x7F on /dev/ttyUSB1. STM32 is not in system-bootloader mode. Set BOOT0 high, reset the board, then retry.` |
| E301 | Image too large | `Error: Write failed` | `Error [E301]: Image app.bin (1.6 MiB) exceeds app partition at 0x10000 (1.0 MiB). Reduce image size or update the partition table.` |
| E302 | Verify mismatch | `Error: Verification error` | `Error [E302]: Verify failed at 0x08004A00. Expected 0x3C, read 0xFF. Sector may not have been written. Re-run with '--erase sectors'.` |
| E401 | Read-out protection | `Error: Access denied` | `Error [E401]: Read-out protection (RDP Level 1) is active on STM32F411. Flash cannot be read or written. Removing it erases all flash: 'mcu-tool erase --unprotect'.` |

### Why the good versions work

- **They separate symptom from cause.** "Communication failed" is a symptom. "Baud rate mismatch" is a cause the user can act on.
- **They include numbers.** Expected versus detected values let the user check the diagnosis instead of trusting it blindly.
- **They name the fix.** Every good error ends with a command or a physical step, never "please try again".
- **They warn before destruction.** E401 tells the user that the fix erases flash *before* they run it.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Usage error (bad flag or missing argument) |
| `2` | Connection error (E1xx) |
| `3` | Device or bootloader error (E2xx) |
| `4` | Flash write or verify error (E3xx) |
| `5` | Protection or security error (E4xx) |

Distinct exit codes let CI pipelines and scripts react to the *type* of failure without parsing text.

---

## 3. Diff Analysis: Evaluating Two AI-Generated Outputs

### The task given to both models

> Write the verbose terminal output `mcu-tool` should print when flashing `motor_ctrl.bin` (48 KiB) to an STM32F411 over the UART system bootloader on `/dev/ttyUSB0`.

### Response A

```text
$ mcu-tool flash -v -t stm32f4 -p /dev/ttyUSB0 motor_ctrl.bin
[conn] Opening /dev/ttyUSB0 at 115200 baud, 8E1
[boot] TX 0x7F (sync)      RX 0x79 (ACK)
[boot] TX GET_ID 0x02      RX PID 0x0431 (STM32F411xC/E)
[boot] Bootloader version 3.1
[erase] Extended erase (0x44): sectors 0-2 (16 KiB each)
[write] 0x08000000 ████████████████████ 48.0 / 48.0 KiB
[verify] Read-back OK (49152 bytes)
[reset] Done. Set BOOT0 low before the next reset to run the new firmware.
```

### Response B

```text
$ mcu-tool flash -v -t stm32f4 -p /dev/ttyUSB0 motor_ctrl.bin
[conn] Opening /dev/ttyUSB0 at 115200 baud, 8N1
[boot] TX 0x7F (sync)      RX 0x79 (ACK)
[boot] TX GET_ID 0x02      RX PID 0x0431 (STM32F411xC/E)
[erase] Erase (0x43): sectors 0-2
[write] 0x08000000 ████████████████████ 48.0 / 48.0 KiB
[verify] Read-back OK (49152 bytes)
[reset] Device rebooted. Firmware running.
```

### The diff

```diff
- [conn] Opening /dev/ttyUSB0 at 115200 baud, 8E1
+ [conn] Opening /dev/ttyUSB0 at 115200 baud, 8N1

- [erase] Extended erase (0x44): sectors 0-2 (16 KiB each)
+ [erase] Erase (0x43): sectors 0-2

- [reset] Done. Set BOOT0 low before the next reset to run the new firmware.
+ [reset] Device rebooted. Firmware running.
```

### Findings

| # | Issue in Response B | Why it is wrong | Severity |
|---|---|---|---|
| 1 | Serial frame `8N1` | The STM32 USART bootloader requires **even parity** (8E1). The sync and GET_ID lines could still succeed by coincidence: 0x7F, 0x02/0xFD and the replies 0x79, 0x01, 0x04, 0x31 all have an odd number of 1-bits, so their even-parity bit is 1, which is exactly what an 8N1 stop bit looks like. The first byte with an even number of 1-bits breaks the link. Here that is the 0x00 bytes of the write address 0x08000000, which raise a parity error, so the 48 KiB write could not complete. | **Critical**: the transcript shows a write that could not succeed. |
| 2 | Erase command `0x43` | STM32F4 bootloaders implement **Extended Erase (0x44)**. The legacy Erase command (0x43) is not supported on these devices. | **Major**: wrong protocol command. |
| 3 | "Firmware running" after reset | If BOOT0 is still high, a reset re-enters the system bootloader, not the user firmware. The message claims an outcome the tool cannot guarantee. | **Minor**: misleading status message. |

Response A has no technical errors. Its one weakness is that it does not report elapsed time or transfer speed, which would help users compare baud-rate settings.

### Evaluation verdict

**Preferred: Response A.**

**Rationale:** B looks correct and polished, but it connects at 8N1 to a bootloader that requires 8E1. Its first lines could pass by coincidence of bit patterns, which makes the error easy to miss, but the write address bytes would fail parity, so the successful 48 KiB write it reports could not happen. B also uses the legacy erase opcode that F4 parts do not support. A gets both details right and adds an accurate BOOT0 reminder, which B replaces with an unverifiable claim.

### What this example shows

The dangerous error here is not a typo or a crash. It is a **plausible transcript that cannot happen on real hardware.** A reviewer who checks only formatting and flow would rate B equal to A. Even a reviewer who knows 8E1 is required might accept B's ACK lines, because those particular bytes happen to survive the wrong framing. Catching it requires working through the protocol byte by byte, which is why technical evaluation needs domain knowledge, not only a writing checklist.

---

### References

- STMicroelectronics, *AN3155: USART protocol used in the STM32 bootloader.*
- STMicroelectronics, *AN2606: STM32 microcontroller system memory boot mode.*
- Espressif Systems, *esptool.py documentation: Boot mode selection.*
