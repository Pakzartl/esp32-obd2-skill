---
name: esp32-obd2
description: "ESP32 IoT/embedded reference for CAN bus, OBD2, BLE/NimBLE, GATT services, Flutter BLE, OTA updates, and vehicle data logging. Use this skill whenever working with ESP32 + CAN/TWAI, OBD2 connectors, BLE pairing on ESP-IDF, NimBLE crashes, Flutter BLE integration, ESP-IDF firmware flashing, CAN bus sniffing, vehicle telemetry, motorcycle ECU reading, or embedded automotive projects — even if the user doesn't mention OBD2 explicitly."
license: MIT
metadata:
  author: Pakzartl
  version: "1.0.0"
  tags: esp32, obd2, can-bus, ble, nimble, flutter, esp-idf, automotive, iot, embedded
---

# ESP32 OBD2 — IoT Embedded Reference

Complete reference for building ESP32-based OBD2/CAN bus data loggers with BLE connectivity and mobile app integration. Covers the full stack from hardware wiring through firmware to Flutter app.

## 1. macOS USB/Serial Debugging

### Find USB devices
```bash
ioreg -p IOUSB -w0 | grep -E '(\+-o)' | head -20
find /dev -name "cu.usb*" -maxdepth 1
```

### FTDI FT232 driver (macOS Apple Silicon)
- Install: `brew install --cask ftdi-vcp-driver`
- Run installer: `open /Applications/FTDIUSBSerialDextInstaller.app`
- Approve: System Settings > General > Login Items & Extensions > Driver Extensions > enable FTDI
- Apple built-in `AppleUSBFTDI` may work without third-party driver — disable FTDI dext if conflicting
- After enable/install: unplug/replug FT232
- Port appears as `/dev/cu.usbserial-*`

### ESP32-S3 native USB
- Shows as `USB JTAG/serial debug unit`
- Port: `/dev/cu.usbmodem*`
- No external driver needed

### Troubleshooting
- FT232 disappears from USB = overcurrent (board draws too much from FT232 3V3, max 50mA per FT232R datasheet)
- Fix: power board from separate source, FT232 for data only (TX/RX/GND, no 3V3)
- USB port disabled after overcurrent: unplug, try different USB port

## 2. ESP-IDF Standalone Setup

### Install
```bash
mkdir -p ~/esp
git clone --depth 1 --branch v5.5.3 --recursive --shallow-submodules https://github.com/espressif/esp-idf.git ~/esp/esp-idf
~/esp/esp-idf/install.sh esp32
brew install cmake ninja  # if missing
```

### Build
```bash
export IDF_PATH=$HOME/esp/esp-idf
. $IDF_PATH/export.sh
idf.py set-target esp32
idf.py build
```

### Flash via FT232
```bash
python3 -m esptool --chip esp32 -p /dev/cu.usbserial-110 -b 460800 \
  --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 4MB --flash_freq 40m \
  0x1000 build/bootloader/bootloader.bin \
  0x8000 build/partition_table/partition-table.bin \
  0x10000 build/app.bin
```

### Flash with OTA partitions
```bash
# Same as above but add:
0xf000 build/ota_data_initial.bin \
0x20000 build/app.bin  # note offset 0x20000 not 0x10000
```

### Monitor serial
```python
# Use ESP-IDF's pyserial:
$HOME/.espressif/python_env/idf5.5_py3.13_env/bin/python3 -c "
import serial, time, sys
s = serial.Serial('/dev/cu.usbserial-110', 115200, timeout=0.5)
while True:
    try:
        data = s.read(512)
        if data:
            print(data.decode('utf-8', errors='replace'), end='')
            sys.stdout.flush()
    except:
        break
s.close()
"
```

### Download mode (no auto-reset)
1. Connect IO0 to GND
2. Press EN/reset (or power cycle)
3. Release IO0
4. Flash within a few seconds

## 3. ArtronShop ESP-OBD2 Board Reference

### Key specs
- MCU: ESP32-WROOM-32E (ESP32-D0WDQ6 rev v1.1, no PSRAM, 520KB SRAM, 4MB flash)
- CAN: SN65HVD230DR — **GPIO26 (TX), GPIO27 (RX)**
- Power: TPS54202 buck converter, **4.5-28V input**, 3.3V/2A output
- Protection: TVS diode SMBJ30CA, Schottky SS210
- OBD2: J1962 male 16-pin (pin 6=CAN-H, pin 14=CAN-L, pin 16=+12V permanent battery, pin 4/5=GND)
- Size: 57 x 40 mm

### Pin header (H3, 6-pin)
- Exposed: IO0, TX0, RX0, EN, GND, 3V3
- CAN: GPIO26 (TX), GPIO27 (RX) — internal to board, not on header

### LEDs
- LED4 green (power): NOT controllable (hardwired 3V3)
- LED5 cyan (IO5): controllable (IO5 > R9 330ohm > LED > GND)
- LED2 cyan (IO26 / CAN TX): CAN driver input activity
- LED3 cyan (IO27 / CAN RX): CAN receiver output activity

### Buttons & switches
- SW1 = EN (reset), SW2 = BOOT (IO0), SW3 = power on/off

### JP1 — CAN termination (120 ohm)
- **Open JP1 when connecting to vehicle** (ECU has own termination)
- Bridge JP1 for bench testing (board is bus endpoint)
- Double termination = CAN bus errors

### Schematic & docs
- Schematic: https://dl.artronshop.co.th/ESP-OBD2/Schematic.pdf
- Dimensions: https://dl.artronshop.co.th/ESP-OBD2/Dimension.pdf
- Example firmware: https://github.com/ArtronShop/ESP32-CAN-Web-Sniffer

## 4. CAN Bus Sniffer (ESP-IDF C)

### Minimal TWAI ListenOnly setup
```c
#include "driver/twai.h"

#define CAN_TX_GPIO  GPIO_NUM_26  // ArtronShop ESP-OBD2
#define CAN_RX_GPIO  GPIO_NUM_27

twai_general_config_t g_config = TWAI_GENERAL_CONFIG_DEFAULT(
    CAN_TX_GPIO, CAN_RX_GPIO, TWAI_MODE_LISTEN_ONLY);
twai_timing_config_t t_config = TWAI_TIMING_CONFIG_500KBITS();
twai_filter_config_t f_config = TWAI_FILTER_CONFIG_ACCEPT_ALL();

twai_driver_install(&g_config, &t_config, &f_config);
twai_start();

twai_message_t msg;
if (twai_receive(&msg, pdMS_TO_TICKS(100)) == ESP_OK) {
    // msg.identifier, msg.data_length_code, msg.data[]
}
```

### TWAI errata fixes (sdkconfig)
```
CONFIG_TWAI_ERRATA_FIX_BUS_OFF_REC=y
CONFIG_TWAI_ERRATA_FIX_TX_INTR_LOST=y
CONFIG_TWAI_ERRATA_FIX_RX_FRAME_INVALID=y
CONFIG_TWAI_ERRATA_FIX_RX_FIFO_CORRUPT=y
CONFIG_TWAI_ERRATA_FIX_LISTEN_ONLY_DOM=y
```

### GPIO mapping by board
| Board | CAN TX | CAN RX |
|-------|--------|--------|
| ArtronShop ESP-OBD2 | GPIO26 | GPIO27 |
| Custom ESP32-S3 + SN65HVD230 | GPIO4 | GPIO5 (or any free GPIO) |

## 5. OTA via WiFi (ESP-IDF)

### Partition table (4MB, dual OTA)
```csv
# Name,    Type, SubType,  Offset,   Size,
nvs,       data, nvs,      0x9000,   0x6000,
otadata,   data, ota,      0xf000,   0x2000,
phy_init,  data, phy,      0x11000,  0x1000,
ota_0,     app,  ota_0,    0x20000,  0x1C0000,
ota_1,     app,  ota_1,    0x1E0000, 0x1C0000,
```

### sdkconfig for OTA + rollback
```
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y
CONFIG_BOOTLOADER_WDT_ENABLE=y
CONFIG_BOOTLOADER_WDT_TIME_MS=9000
```

### WiFi credentials via Kconfig
Create `main/Kconfig.projbuild`:
```
menu "Project Configuration"
    config WIFI_SSID
        string "WiFi SSID"
        default "myssid"
    config WIFI_PASS
        string "WiFi Password"
        default "mypassword"
endmenu
```

### Rollback pattern
```c
void app_main(void) {
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t state;
    if (esp_ota_get_state_partition(running, &state) == ESP_OK
        && state == ESP_OTA_IMG_PENDING_VERIFY) {
        esp_ota_mark_app_valid_cancel_rollback();
    }
}
```

## 6. OBD2 Reference (Honda / Motorcycle)

### 6-pin connector (ISO 19689, Honda Euro 5)
```
+-------------+
|  A   B   C  |
|  D   E   F  |
+-------------+
A = GND          B = CAN-H       C = SCS (service check)
D = K-Line       E = CAN-L       F = +12V (ignition switched)
```

### Protocol
- CAN bus: ISO 11898, **500 kbps**; diagnostic: ISO 15765-4 (OBD-II over CAN)
- Euro 5 = CAN (not K-Line)

### Standard OBD2 PIDs
| PID | Data | Request |
|-----|------|---------|
| 0x0C | RPM = (A*256+B)/4 | 0x7DF: [02 01 0C 00 00 00 00 00] |
| 0x0D | Speed = A km/h | 0x7DF: [02 01 0D 00 00 00 00 00] |
| 0x11 | Throttle = A*100/255 % | 0x7DF: [02 01 11 00 00 00 00 00] |
| 0x05 | Coolant = A-40 C | 0x7DF: [02 01 05 00 00 00 00 00] |

### Safety
- **READ-ONLY** — never write CAN frames to ECU until fully reverse-engineered and bench-tested
- Open JP1 (remove 120 ohm termination) on ESP-OBD2 board
- Inline fuse 1-2A on 12V line

## 7. BLE / NimBLE on ESP32 (ESP-IDF)

### RAM constraint — WiFi + BLE cannot coexist on ESP32-WROOM-32E
- WiFi STA ~60-80KB + NimBLE ~30-40KB = ~100-120KB combined
- ESP32-WROOM-32E has ~320KB usable DRAM (no PSRAM)
- **Solution**: Run WiFi and BLE in separate modes, never simultaneously

### NimBLE crash root causes
1. **`CONFIG_BTDM_CTRL_MODEM_SLEEP=y` (default)** causes controller crash during handshake
2. **Max connections mismatch** — controller allows 3 but host allows 1
3. **Bond storage + NVS persist** — allocation failure under memory pressure

### Critical sdkconfig for NimBLE on ESP32
```
CONFIG_BT_ENABLED=y
CONFIG_BTDM_CTRL_MODE_BLE_ONLY=y
CONFIG_BTDM_CTRL_BLE_MAX_CONN=1
CONFIG_BTDM_CTRL_MODEM_SLEEP=n          # CRITICAL — default y causes crash
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1       # Must match BTDM_CTRL_BLE_MAX_CONN
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=5120
CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_INTERNAL=y
```

### BLE disconnects after ~10s without GATT service
- No data exchange triggers connection supervision timeout (4-20s)
- Fix: Add GATT service with at least one characteristic

### GATT registration order
```c
ble_svc_gap_device_name_set("DEVICE_NAME");
ble_svc_gap_init();
ble_svc_gatt_init();
ble_gatts_count_cfg(gatt_svcs);   // count first
ble_gatts_add_svcs(gatt_svcs);    // then add
```

### GATT gotchas
- Notify-only characteristic with `access_cb = NULL` causes assert failure
- **Every characteristic must have an `access_cb`**, even notify-only ones

### BLE UUID endianness
- `BLE_UUID128_INIT` stores bytes in **little-endian** order
- UUID string is big-endian
- Example: `BLE_UUID128_INIT(0xf0, 0xde, ...)` = UUID `12345678-1234-5678-1234-56789abcdef0`
- Flutter app must use same byte-order — mismatch causes **silent** service discovery failure

### CAN frame notification format
`[ID(4 bytes LE)][DLC(1 byte)][DATA(0-8 bytes)]` = max 13 bytes per notification

### GAP event handler must handle
- `CONNECT` — store conn_handle
- `DISCONNECT` — reset handle, turn off notify, re-advertise
- `SUBSCRIBE` — toggle live notify
- `MTU` — log new MTU value
- `ENC_CHANGE` — log state
- `REPEAT_PAIRING` — delete stale bond, return RETRY
- `PASSKEY_ACTION` — return `BLE_HS_ENOTSUP` for Just Works

## 8. BLE Pairing (Consumer Device Pattern)

### Android Settings pairing failure (status=13) — root causes
- `NVS_PERSIST=n` — bond keys lost after reboot
- `MAX_BONDS=1` — re-pair slot full
- Key distribution missing `ID` key — Android uses RPA, ESP32 can't resolve
- No `REPEAT_PAIRING` handler — "Forget" + re-pair fails

### Consumer BLE pairing pattern
- Use **Just Works** (no PIN) — same as most consumer devices
- Must have: NVS persist + bonds >= 3 + key distribution with ID + repeat pairing handler
- `ble_store_config_init()` does NOT exist in ESP-IDF v5.5.3

### Key distribution fix
```c
// Before (broken):
ble_hs_cfg.sm_our_key_dist = BLE_SM_PAIR_KEY_DIST_ENC;
ble_hs_cfg.sm_their_key_dist = BLE_SM_PAIR_KEY_DIST_ENC;

// After (working):
ble_hs_cfg.sm_our_key_dist = BLE_SM_PAIR_KEY_DIST_ENC | BLE_SM_PAIR_KEY_DIST_ID;
ble_hs_cfg.sm_their_key_dist = BLE_SM_PAIR_KEY_DIST_ENC | BLE_SM_PAIR_KEY_DIST_ID;
```

### Repeat pairing handler
```c
case BLE_GAP_EVENT_REPEAT_PAIRING: {
    struct ble_gap_conn_desc desc;
    if (ble_gap_conn_find(event->repeat_pairing.conn_handle, &desc) == 0) {
        ble_store_util_delete_peer(&desc.peer_id_addr);
    }
    return BLE_GAP_REPEAT_PAIRING_RETRY;
}
```

### POC security config (Just Works, no bonding)
```c
ble_hs_cfg.sm_io_cap = BLE_SM_IO_CAP_NO_IO;
ble_hs_cfg.sm_bonding = 0;
ble_hs_cfg.sm_mitm = 0;
ble_hs_cfg.sm_sc = 0;
ble_att_set_preferred_mtu(256);
```

### BLE reboot on disconnect (brute-force reliability for POC)
NimBLE re-advertising after disconnect can be unreliable. Rebooting is more reliable:
```c
case BLE_GAP_EVENT_DISCONNECT:
    xTaskCreate(ble_reboot_task, "ble_reboot", 2048, NULL, 1, NULL);
    break;

static void ble_reboot_task(void *arg) {
    vTaskDelay(pdMS_TO_TICKS(300));
    esp_restart();
}
```

## 9. Flutter BLE Integration

### Android BLE permissions (Android 12+)
- Manifest: `BLUETOOTH`, `BLUETOOTH_ADMIN`, `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`, `ACCESS_FINE_LOCATION`
- Without permissions: scan returns zero results **silently**

### Scan by device name (not UUID)
```dart
FlutterBluePlus.scanResults.listen((r) {
  for (final result in r) {
    if (result.device.platformName == 'DEVICE_NAME') { ... }
  }
});
await FlutterBluePlus.startScan(timeout: Duration(seconds: 5));
```

### Connection flow
```dart
try { await device.removeBond(); } catch (_) {}
try { await device.clearGattCache(); } catch (_) {}
await Future.delayed(Duration(milliseconds: 500));
await device.connect(autoConnect: false, timeout: Duration(seconds: 10));
final services = await device.discoverServices();
```

### Auto-reconnect (3 attempts, re-scan each time)
```dart
for (int i = 0; i < 3; i++) {
  await Future.delayed(Duration(seconds: 3 + i * 2));
  final results = await scan(timeout: Duration(seconds: 3));
  if (results.isEmpty) continue;
  try { await connect(results.first.device); return; } catch (_) {}
}
```
Key: **re-scan before reconnect** — don't reuse stale `BluetoothDevice` object.

### MTU: use default for small payloads
- CAN frames are max 13 bytes — default MTU (23 - 3 = 20 payload) is sufficient
- Avoids MTU collision/timeout issues (status=133, status=8, status=22)

### CAN frame parsing in Dart
```dart
final canId = data[0] | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
final dlc = data[4];
final canData = data.length > 5 ? data.sublist(5) : <int>[];
```

## 10. GPIO5 / LED on ESP32

### GPIO5 is a strapping pin
- Has internal pull-up during boot — LED lights up during bootloader (~500ms)
- Must call `gpio_reset_pin(GPIO_NUM_5)` + disable pull-up explicitly

### LED active-LOW pattern
- GPIO=0 = LED ON, GPIO=1 = LED OFF
- Always end blink with `gpio_set_level(LED_GPIO, 1)` to turn off

### Blink from BLE callback — use dedicated task
BLE callbacks run in NimBLE host context. GPIO work must happen in a separate task:
```c
static volatile uint8_t pending_led_blink = 0;
static void led_task(void *arg) {
    while (1) {
        if (pending_led_blink > 0) {
            uint8_t count = pending_led_blink;
            pending_led_blink = 0;
            for (int i = 0; i < count; i++) {
                gpio_set_level(LED_GPIO, 0);  // ON (active-low)
                vTaskDelay(pdMS_TO_TICKS(30));
                gpio_set_level(LED_GPIO, 1);  // OFF
                vTaskDelay(pdMS_TO_TICKS(200));
            }
        }
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

## 11. Dual Mode (DEV/PROD) via IO0 Button

- SW2 (BOOT/IO0) is free after boot — hold 5 seconds to switch mode
- NVS persists mode across reboots
- DEV mode: WiFi + OTA + CAN sniffer (no BLE)
- PROD mode: BLE + CAN sniffer (no WiFi)

## 12. Thermal & Power Reference

### Operating temperature
| Component | Operating | Note |
|-----------|-----------|------|
| ESP32-WROOM-32E | -40 to +85 C | |
| ESP32-S3 N16R8 (Octal PSRAM) | -40 to +65 C | PSRAM is limiting |
| SN65HVD230 | -40 to +85 C | |

### Motorcycle deployment checklist
- Inline fuse 1-2A on 12V input
- TVS diode (P6KE15A) for voltage spikes
- Open JP1 (CAN termination) on ESP-OBD2
- IP65 waterproof enclosure
- Strain relief on OBD2 cable

## 13. FreeRTOS Task Architecture

| Task | Stack | Priority | Core | Function |
|------|-------|----------|------|----------|
| main (app_main) | 8192 | 1 | any | Init WiFi/BLE, start CAN |
| can_sniffer_task | 4096 | 5 | Core 1 | TWAI receive + BLE notify |
| mode_switch_task | 2048 | 10 | any | Poll IO0 every 200ms |
| led_task | 2048 | 3 | Core 1 | GPIO blink |
| nimble_host_task | 8192 | default | Core 0 | BLE host |

**Core pinning**: CAN + LED on Core 1, BLE/WiFi on Core 0.

## 14. Complete sdkconfig.defaults

```
# Flash
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y

# Partition
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"

# Bootloader + rollback
CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y
CONFIG_BOOTLOADER_WDT_ENABLE=y
CONFIG_BOOTLOADER_WDT_TIME_MS=9000

# Main task
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_LOG_DEFAULT_LEVEL_INFO=y

# TWAI errata
CONFIG_TWAI_ERRATA_FIX_BUS_OFF_REC=y
CONFIG_TWAI_ERRATA_FIX_TX_INTR_LOST=y
CONFIG_TWAI_ERRATA_FIX_RX_FRAME_INVALID=y
CONFIG_TWAI_ERRATA_FIX_RX_FIFO_CORRUPT=y
CONFIG_TWAI_ERRATA_FIX_LISTEN_ONLY_DOM=y

# BLE controller
CONFIG_BT_ENABLED=y
CONFIG_BTDM_CTRL_MODE_BLE_ONLY=y
CONFIG_BTDM_CTRL_BLE_MAX_CONN=1
CONFIG_BTDM_CTRL_MODEM_SLEEP=n

# NimBLE host
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1
CONFIG_BT_NIMBLE_ROLE_BROADCASTER=y
CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=y
CONFIG_BT_NIMBLE_ROLE_CENTRAL=n
CONFIG_BT_NIMBLE_ROLE_OBSERVER=n
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=8192
CONFIG_BT_NIMBLE_NVS_PERSIST=n
CONFIG_BT_NIMBLE_MAX_BONDS=0
CONFIG_BT_NIMBLE_SM_SC=n
CONFIG_BT_NIMBLE_SM_LEGACY=n
CONFIG_BT_NIMBLE_SECURITY_ENABLE=n
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=256
CONFIG_BT_NIMBLE_ACL_BUF_SIZE=260
CONFIG_BT_NIMBLE_ACL_BUF_COUNT=12
CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_INTERNAL=y
```

## 15. Wiring Configurations

### FT232 to ESP-OBD2 (data only)
```
FT232          ESP-OBD2
TX     ------> RXD (RX0 on header)
RX     <------ TXD (TX0 on header)
GND    ------> GND
3V3    ------> NOT CONNECTED (causes overcurrent!)
```

### Vehicle deployment
```
Honda 6-pin OBD
Pin A (GND)  ---> ESP-OBD2 GND
Pin B (CAN-H) --> ESP-OBD2 CAN-H (SN65HVD230)
Pin E (CAN-L) --> ESP-OBD2 CAN-L (SN65HVD230)
Pin F (+12V)  --> ESP-OBD2 VIN (TPS54202 buck to 3.3V)
```

### Download mode (ESP-OBD2, no auto-reset)
1. Disconnect power completely
2. Connect IO0 to GND with jumper wire
3. Reconnect power (enters download mode)
4. Flash immediately
5. Disconnect IO0, then reset

## 16. Toolchain Gotchas (macOS Apple Silicon)

- **espup**: Download prebuilt binary from GitHub releases (aarch64-apple-darwin) — Homebrew liblzma is x86
- **cargo-generate**: Set `LZMA_API_STATIC=1` to force build from source
- **espflash monitor**: Needs TTY — use Python serial reader in non-interactive terminals
- **ESP32-S3 GPIO 48 LED**: Clone boards may have plain LED, not WS2812 — use RMT driver for NeoPixel

## 17. Error Messages Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| `StoreProhibited` at `ble_hs_id_infer_auto` | NimBLE + modem sleep | `CONFIG_BTDM_CTRL_MODEM_SLEEP=n` |
| `assert failed: ble_hs_event_start_stage2` | NULL access_cb | Every GATT chr needs access_cb |
| `status=13` (BLE pairing) | Missing bond/key config | NVS_PERSIST=y, BONDS=3, add ID key dist |
| `status=133` (GATT_ERROR) | MTU on reconnect | Disable bonding or match ACL to MTU |
| `ESP_ERR_INVALID_STATE` | TWAI transmit before start | Call `twai_start()` first |
| `Device not configured` (errno 6) | FT232 overcurrent | Don't power board from FT232 3V3 |
