# Zagotovka_M_BOOTLOADER

Dual-bank OTA bootloader for the **STM32F767ZI** (2 MB flash).

It is flashed into the microcontroller **once**, during the initial setup of the device, and is the companion of the main firmware project [**Zagotovka_M**](https://github.com/zagotovka/Zagotovka_M).

The bootloader makes safe over-the-air (OTA) firmware updates possible. It decides which of the two firmware banks (Bank A or Bank B) is started, tests a freshly uploaded firmware before trusting it, and switches back to the previous firmware if the new one does not prove to be healthy.

Language: **English** | [Русский](README_ru.md)

---

## How it fits together

Zagotovka_M uses a **Dual Bank** architecture: the application firmware is stored in two independent areas of flash, **Bank A** and **Bank B**. Only one of them is active at a time.

- The running application receives a new firmware file (`.bin`) over the web interface and writes it into the **inactive** bank. The image is verified by size and CRC.
- The application records the update in a settings record in flash (the target bank, the bank it came from, a "pending" flag, and the size and CRC32 of the new image) and reboots.
- On the next start the **bootloader** reads that record, picks the bank to run, verifies the new image and starts it.

In short: the bootloader provides **safe OTA updates of the main firmware and switching between the two memory banks**.

Because Bank A and Bank B are linked for different flash addresses, there are two separate builds of the application. Only the inactive bank is ever updated, so you upload the file for the **opposite** bank:

| Active bank | File to upload |
|---|---|
| Bank A | `Zagotovka_Bank_B.bin` |
| Bank B | `Zagotovka_Bank_A.bin` |

---

## Flash memory map

| Address range | Size | Purpose |
|---|---|---|
| `0x08000000` - `0x0803FFFF` | 256 KB | This bootloader |
| `0x08040000` | up to 768 KB | Application, **Bank A** |
| `0x08100000` | up to 768 KB | Application, **Bank B** |
| `0x081C0000` - `0x081FFFFF` | 256 KB | Settings area (94 wear-leveling slots) |

The device must run with the **Dual Bank** option byte enabled (`nDBANK = 0`). In Dual Bank mode the settings area consists of two 128 KB sectors (sectors 22 and 23); in Single Bank mode it is sector 11. A new STM32F767ZI leaves the factory in Single Bank mode. The bootloader cannot change this mode by itself: changing it destroys everything stored in the flash, including the bootloader. The mode is therefore set from the outside, **before** the first flashing, by the `make_first_flash.py` script (see "Switching to Dual Bank" below). If the chip is still in Single Bank mode, the bootloader prints an error and does not start the application.

---

## What the bootloader does at every start

1. Initializes the clock (HSI), USART3 (debug log, 115200 8N1) and the CRC unit.
2. Checks the flash mode. If the chip is in Single Bank mode, prints an error every second and does not start the application (see "Switching to Dual Bank" below).
3. Scans the 94 settings slots, validates each one (magic value and CRC) and takes the newest valid record. A damaged slot is reported and skipped.
4. Decides which bank to start:

| Situation | Result |
|---|---|
| No valid settings at all | Starts Bank A |
| No OTA pending | Starts the active bank from the settings |
| OTA pending, first start of the new image | Trial start of the new bank (its CRC32 is checked), attempt counter is incremented |
| OTA pending, the new image was confirmed (committed) | Starts the active bank |
| OTA pending, the application asked for a rollback | Goes back to the previous bank |
| OTA pending, trial attempts exhausted (3) | Automatic rollback to the previous bank |

5. Checks that the selected image is valid:
   - the initial stack pointer must point into SRAM (always checked);
   - during a trial start (OTA pending) the CRC32 of the image is also computed over the size stored in the settings and compared with the stored CRC32. This takes about 4 seconds at the HSI clock, so it is **not** done on normal starts.

   If the check fails, the bootloader restores the previous bank (when an OTA was pending) and tries the other bank. If the other bank is not valid either, it starts whichever bank has a valid stack pointer, even if its CRC did not match: a controller in a hard-to-reach place is better running than halted. If neither bank has a valid stack pointer, it halts.
6. Sets the vector table and jumps to the application.

### Update life cycle

```
running from Bank A (active)
  -> application uploads new image into Bank B, verifies size and CRC
  -> application writes settings: target = B, previous = A, pending = 1,
     size and CRC32 of the new image
  -> reboot
bootloader: pending, checks the CRC32 of Bank B, trial start (attempt 1 of 3)
  -> new firmware runs its health check and confirms itself (commit)
       -> settings: active = B, pending = 0     (update finished)
  -> or the CRC32 does not match, or it never confirms (crashes, hangs, resets)
       -> the bootloader rolls back to Bank A (immediately on a CRC mismatch,
          after 3 attempts otherwise)
```

A rollback can also be requested manually from the application.

---

## Settings storage

The settings are kept in the `HTTPSsettings` structure (domain, TLS key and certificates, Telegram token, OTA state, bank versions, and the size and CRC32 of the Bank A and Bank B images, and so on). Each write goes into the **next** of 94 slots, with an increasing version number and a CRC. When the ring wraps around, the whole settings area is erased and writing restarts from slot 0. This spreads the wear over the flash and keeps the previous valid record until the new one is written.

The bootloader verifies the erase (the whole area must read `0xFF`) and reads each slot back after programming.

**Important:** the `HTTPSsettings` structure is shared byte for byte between this bootloader and the main project (`Core/Inc/zagotovka.h` in Zagotovka_M). Change it only by **appending fields at the end**, and keep both copies identical. Never reorder fields or change their meaning.

The structure is currently 2780 bytes, so 94 slots occupy 261,320 of the 262,144 bytes of the settings area. There is room for fewer than 10 more bytes per slot: do not add new fields without reducing the number of slots, otherwise the compile-time check in `main.c` will fail.

---

## Switching to Dual Bank

The bootloader does **not** change the flash mode itself. Changing the `nDBANK` option byte changes how the flash is organised (the width of reading and the layout of the sectors), and everything already stored in the flash, including the bootloader itself, becomes unreadable. A program that runs from the flash cannot survive that: when the option is written, the processor stops. So the mode must be set from the outside, **before** the firmware is written:

1. Set `nDBANK = 0` with a programmer (STM32CubeProgrammer).
2. Erase the whole flash (after the change the old content is garbage).
3. Write the firmware.

The script `make_first_flash.py` does all three steps automatically (see "Installation" below).

At every start the bootloader only **checks** the mode. In Dual Bank mode it prints `[BOOT] Flash: Dual Bank (OK)` and goes on. In Single Bank mode it prints this line every second and does not start the application, so a wrongly prepared device is obvious at once:

```
[BOOT] *** ERROR: Flash is Single Bank. Run: STM32_Programmer_CLI -c port=SWD -ob nDBANK=0, then erase and reflash ***
```

Single Bank mode is not supported by OTA. If it is selected again (for example by `Switch_To_SingleBank()` of the application), the flash content is lost and the bootloader stops with this error on the next start: set Dual Bank again as described above and flash the firmware again.

---

## Direct flashing via ST-Link

A bank can be flashed directly with a programmer at any time, without erasing the settings area:

- Outside an OTA trial the bootloader checks only the stack pointer, so the new image is started even though the stored CRC32 of that bank is outdated.
- About 5 seconds after the start the application compares the stored CRC32 of both banks with the real flash contents and clears the outdated version, size and CRC32. The Firmware Update page then shows "flashed directly (not via OTA)".
- A damaged image cannot be told from a directly flashed one by its content. A bank damaged after its record was cleared is therefore started if its stack pointer is valid.
- If a bank is flashed while an OTA trial is pending (`pending=1`), the bootloader treats the CRC mismatch as a damaged image and rolls back to the previous bank. Confirm the firmware on the Firmware Update page first, or erase sectors 22 and 23.

---

## Debug log

The bootloader prints progress to **USART3** (115200, 8N1) with the prefix `[BOOT]`. Typical output:

```
[BOOT] === Bootloader started, build Oct 26 2026 11:07:25 ===
[BOOT] settings: struct=2780 slot=2780 OPTCR=0xDFFFAAFD
[BOOT] Flash: Dual Bank (OK)
[BOOT] Settings: bank=1 pending=0 state=0 retries=1 ver=93
[BOOT] No OTA -> active bank
[BOOT] Bank B: MSP check only (no OTA trial)
[BOOT] Target: BANK B
[BOOT] -> 0x08100000 (MSP=0x20080000)
```

The build date and time in the first line are filled in by the compiler, so they show which build of the bootloader is really in the chip. The second line shows the size of `HTTPSsettings` and the slot step; it must match the `[SET] struct=... slot=...` line printed by the application. `OPTCR` is the option byte register (bit `nDBANK` = 0 means Dual Bank).

If the chip is still in Single Bank mode, you will see `[BOOT] *** ERROR: Flash is Single Bank. Run: STM32_Programmer_CLI -c port=SWD -ob nDBANK=0, then erase and reflash ***` every second, and the application is not started.

During an update you will see `[BOOT] Testing BANK A` (or `BANK B`), `[BOOT] Bank B: CRC OK size=...` (or `CRC MISMATCH size=... exp=... got=...`), `[BOOT] Committed -> active bank`, `[BOOT] Max retries -> rollback to previous bank` and similar messages. A rejected image is reported as `Target bank image INVALID`, followed by `Falling back to the other bank`, `No bank passed CRC, booting ... by MSP only` or `BOTH banks invalid, halting`. Damaged or failed settings writes are reported as `Slot N: CRC bad` or `settings: ... FAILED`.

---

## Installation (once per device)

Before the firmware is written, the chip must be in **Dual Bank** mode (see "Switching to Dual Bank" above). The easiest way is the script `make_first_flash.py`: it switches the mode, erases the chip and flashes it in one run.

### Method 1: the script (recommended; Windows, macOS, Linux)

1. Install STM32CubeProgrammer (see below).
2. Connect the board to the computer with a USB cable (the ST-LINK connector).
3. Open the `scripts` folder of the Zagotovka_M project, open a terminal in it and run: `python make_first_flash.py` on Windows, `python3 make_first_flash.py` on macOS and Linux.
4. Wait for `DONE`. The board restarts by itself. In the debug log you must see `[BOOT] Flash: Dual Bank (OK)`.

### Method 2: drag and drop (only if the chip is already in Dual Bank)

This works with boards whose ST-LINK appears as a USB drive (NUCLEO-F767ZI: the drive is called `NODE_F767ZI`). Dragging a file onto the drive does **not** change the memory mode. A brand-new chip is in Single Bank mode, so prepare it once first, with STM32CubeProgrammer:

1. Connect the board, start STM32CubeProgrammer, choose `ST-LINK` and press **Connect**.
2. Open the **OB** tab, the group **User Configuration**, remove the tick at `nDBANK` and press **Apply**.
3. Open the **Erasing & Programming** tab and press **Full chip erase**.

Then:

1. Download `Zagotovka_FIRST_FLASH.bin`. It is one file that contains the bootloader and the firmware for Bank A.
2. Drag it onto the drive `NODE_F767ZI`. Wait until the copy window closes and the ST-LINK LED stops blinking. Do not unplug the cable before that.
3. Done. The device starts and can be opened in the web interface. All later updates are made over the air.

If the file is refused or the drive shows a `FAIL.TXT` file, unplug the cable, plug it in again and repeat the copy.

### Method 3: programmer by hand (for developers)

Run three separate commands, one after another. After the first one the chip restarts, so wait a few seconds before the second:

```
STM32_Programmer_CLI -c port=SWD -ob nDBANK=0
STM32_Programmer_CLI -c port=SWD -e all
STM32_Programmer_CLI -c port=SWD -w Zagotovka_FIRST_FLASH.bin 0x08000000 -v -rst
```

You can also flash the bootloader at `0x08000000` and the Bank A build at `0x08040000` separately. From then on, update the firmware over the air through the web interface of Zagotovka_M. The bootloader itself is changed only through the programmer.

### The script make_first_flash.py

The script lives in the main project: `scripts/make_first_flash.py` (Zagotovka_M). It needs no arguments: it takes `Bootloader/DO_NOT_TOUCH/Zagotovka_M_BOOTLOADER.bin` and the newest `Proshivka/Zagotovka_Bank_A_v*.bin`, puts the bootloader at the start, pads it with `0xFF` up to `0x08040000`, appends the Bank A image and writes `Proshivka/First_flash/Zagotovka_FIRST_FLASH_<version>.bin`. It refuses a bootloader larger than 256 KB, an application larger than 768 KB, and files built for the wrong address (for example the Bank B build). The settings area at `0x081C0000` is not part of the file. Then it reads the flash mode, switches it to Dual Bank if needed, erases the chip and flashes the file. If anything is wrong, it stops with the reason before the chip is touched.

Options: `--image-only` (only build the file), `--bootloader X.bin`, `--bank-a Y.bin`, `-o Z.bin` (use other files), `--cli PATH` (path to `STM32_Programmer_CLI` if it is not found automatically).

### STM32CubeProgrammer must be installed

The script works the same way on **Linux, Windows and macOS**, but only if the program **STM32CubeProgrammer** is installed on the computer. It is free, but you have to install it yourself: the script does not include it and does not download it.

Why it is needed:

- During installation it puts the drivers for the ST-LINK of the board into your system. Without them the computer does not see the board.
- The script talks to the chip through its command-line tool `STM32_Programmer_CLI`.
- Just in case, you can use it by hand to check the chip (see below).

**Where to get it.** Download it from the official site: <https://www.st.com/en/development-tools/stm32cubeprog.html>. Choose the installer for your operating system and run it. The script was tested with version 2.23.0.

**How to check the chip by hand.** Start STM32CubeProgrammer, choose `ST-LINK` in the list at the top right and press **Connect**. Open the **OB** tab (option bytes) and the group **User Configuration**: the box `nDBANK` must be **empty** (this is Dual Bank). If the box is ticked, the chip is still in the old mode: run the script again.

**Two more things that do not depend on the script:**

- **Windows.** The ST-LINK driver is needed (it is installed together with STM32CubeProgrammer or STM32CubeIDE). Type the command `python`, not `python3`.
- **Linux.** If the programmer says that it does not see the ST-LINK, but on another computer everything works, then your user has no right to use the USB device. This is fixed by the `udev` rules from the CubeProgrammer package (the folder `Drivers/rules` inside its installation folder): copy the `*.rules` files to `/etc/udev/rules.d/`, run `sudo udevadm control --reload-rules && sudo udevadm trigger`, and plug the board in again.

### Updating from an older version

When updating from an older bootloader or firmware that used a smaller `HTTPSsettings` structure, erase the settings area (sectors 22 and 23, `0x081C0000` - `0x081FFFFF`) once. The old records cannot be read with the new slot size. The settings (including the TLS key, certificate and Telegram token) are lost and have to be entered again.

If the settings area contains no valid record, the bootloader starts Bank A.

If the device already works in Dual Bank mode, nothing else has to be changed when the bootloader is replaced: flash the new bootloader and it only reports `Flash: Dual Bank (OK)`.

---

## Project layout

```
Core/Src/main.c              Boot logic, settings read/write, bank selection, Dual Bank check
scripts/make_first_flash.py   (in Zagotovka_M) Builds the first-flash file, sets Dual Bank, erases and flashes the chip
*.ld                         Linker scripts
fw_meta.h                    OTA states and retry limit (shared with the application)
```

Constants in `fw_meta.h`: `OTA_STATE_FIRST_BOOT`, `OTA_STATE_UNCOMMITTED`, `OTA_STATE_COMMITTED`, `OTA_STATE_AUTO_ROLLED_BACK`, and `OTA_BOOT_RETRY_MAX = 3`.
