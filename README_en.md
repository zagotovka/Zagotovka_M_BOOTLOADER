## Bootloader — Dual-Bank OTA Loader

The bootloader resides in the `Bootloader/` directory and occupies the first **128 KB** of Flash memory (`0x08000000`–`0x0801FFFF`). It is responsible for selecting and launching the main application, as well as providing a safe OTA update mechanism via bank switching.

### Flash Memory Map

| Region | Addresses | Size | Purpose |
|--------|-----------|------|---------|
| Bootloader | `0x08000000` — `0x0801FFFF` | 128 KB | Bootloader (this code) |
| Bank A | `0x08040000` — `0x080FFFFF` | 768 KB (sectors 5–7) | Main application |
| Bank B | `0x08100000` — `0x081BFFFF` | 768 KB (sectors 8–10) | Backup bank for OTA |
| Sector 11 | `0x081C0000` — `0x081FFFFF` | 256 KB | Settings storage (94 slots) |

### How It Works

1. On power-up, the bootloader reads settings from **Sector 11** (a ring buffer of 94 slots with CRC32 validation).
2. Based on the `ota_pending`, `ota_state`, `ota_active_bank`, and `ota_boot_retries` fields, it determines which bank to launch.
3. Executes `jump_to_app()` — full HAL deinitialization, NVIC/SysTick cleanup, interrupt vector table remapping (SCB->VTOR), and hands over control to the application.

### Bank Selection Logic

```
ota_pending == 0 (no update)?
  └─ Launch ota_active_bank (0 → Bank A, 1 → Bank B)

ota_pending == 1 (update in progress)?
  ├─ ota_state == 3 (COMMITTED)
  │   └─ Launch ota_active_bank — update confirmed
  │
  ├─ ota_state == 2 (ROLLBACK)
  │   └─ Roll back to Bank A, clear OTA flags
  │
  └─ ota_state == 0 or 1 (TESTING)
      ├─ boot_retries >= 3
      │   └─ Roll back to Bank A — app failed to start
      └─ boot_retries < 3
          ├─ Increment boot_retries
          └─ Launch Bank B for testing
```

### OTA States (ota_state)

| Value | Name | Description |
|-------|------|-------------|
| 0 | No data | No OTA data, use normal bank |
| 1 | First boot | First boot of a new image after OTA |
| 2 | Uncommitted | Rollback requested (by application command) |
| 3 | Committed | Update confirmed, rollback impossible |

### Protection Against Bricking

- **Auto-rollback**: If the new application fails to boot (hangs, crashes), after 3 failed reboots the bootloader automatically rolls back to the previous working bank (Bank A).
- **Manual rollback**: The application can request a rollback by writing `ota_state = 2` to settings. On the next reboot, the bootloader will switch to Bank A.
- **Confirmation**: After a successful boot, the application must call `mg_ota_commit()` to set `ota_state = 3`. Only then is the update considered complete.

### How to Flash the Bootloader

1. Flash `Bootloader/main.c` via STM32CubeIDE (configuration for NUCLEO-F767ZI).
2. Then flash the main application to **Bank A** (`0x08040000`).
3. The bootloader will automatically detect the bank and launch the application.

> **Important:** The bootloader is flashed separately from the main application. They use different linker scripts (`STM32F767_BOOT.ld` vs `STM32F767ZITX_FLASH.ld`).
