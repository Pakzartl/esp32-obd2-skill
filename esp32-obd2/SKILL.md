---
name: iot-embedded
description: ESP32 IoT/embedded reference — hardware debugging, ESP-IDF workflow, CAN bus, BLE/NimBLE pairing, GATT services, Flutter BLE, OTA, ArtronShop ESP-OBD2, Honda ADV350 OBD2
triggers:
  - esp32
  - esp-idf
  - can bus
  - obd2
  - twai
  - iot embedded
  - flash firmware
  - ota update
  - serial port
  - ftdi
  - artronshop
  - adv350
  - ble
  - nimble
  - gatt
  - flutter ble
  - pairing
  - bonding
---

# IoT Embedded — ESP32 / CAN Bus / OBD2 Reference

## 1. macOS USB/Serial Debugging

### Find USB devices
```bash
ioreg -p IOUSB -w0 | grep -E '(\+-o)' | head -20
find /dev -name "cu.usb*" -maxdepth 1
```

### FTDI FT232 driver (macOS Apple Silicon)
- Install: `brew install --cask ftdi-vcp-driver`
- Run installer: `open /Applications/FTDIUSBSerialDextInstaller.app`
- Approve: System Settings → General → Login Items & Extensions → Driver Extensions → enable FTDI
- Apple built-in `AppleUSBFTDI` may work without third-party driver — disable FTDI dext if conflicting
- After enable/install: unplug/replug FT232
- Port appears as `/dev/cu.usbserial-*`

### ESP32-S3 native USB
- Shows as `USB JTAG/serial debug unit`
- Port: `/dev/cu.usbmodem*`
- No external driver needed

### Troubleshooting
- `ls` on `/dev/cu.*` may be filtered by rtk — use `find /dev -name "cu.usb*"` or `stat /dev/cu.usbserial-*`
- FT232 disappears from USB = overcurrent (board draws too much from FT232 3V3, max 50mA per FT232R datasheet)
- Fix: power board from separate source, FT232 for data only (TX/RX/GND, no 3V3)
- USB port disabled after overcurrent: unplug, try different Mac USB port

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

### Flash with OTA partitions (add ota_data)
```bash
# Same as above but add:
0xf000 build/ota_data_initial.bin \
0x20000 build/app.bin  # note offset 0x20000 not 0x10000
```

### Monitor serial (without miniterm)
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
1. Connect IO0 → GND
2. Press EN/reset (or power cycle)
3. Release IO0
4. Flash within a few seconds

### Backup flash
```bash
espflash read-flash --port /dev/cu.usbserial-110 --chip esp32 0x0 0x400000 backup.bin
```

## 3. ArtronShop ESP-OBD2 Board Reference

### Key specs
- MCU: ESP32-WROOM-32E (silicon: ESP32-D0WDQ6 rev v1.1, no PSRAM, 520KB SRAM, 4MB flash)
- CAN: SN65HVD230DR — **GPIO26 (TX), GPIO27 (RX)**
- Power: TPS54202 buck converter, **4.5-28V input**, 3.3V/2A output
- Feedback divider: R5=68kΩ (upper), R6=15kΩ (lower) → Vout = 0.596 × (1 + 68/15) = 3.30V
- CAUTION: schematic has stale annotation claiming R1=33.33k/R2=4.7k/Vout=4.82V — WRONG, ignore it
- Protection: TVS diode SMBJ30CA, Schottky SS210
- OBD2: J1962 male 16-pin (pin 6=CAN-H, pin 14=CAN-L, pin 16=+12V permanent battery, pin 4/5=GND)
- Size: 57 x 40 mm
- Price: ฿428

### Pin header (H3, 6-pin)
- Exposed: IO0, TX0, RX0, EN, GND, 3V3
- CAN: GPIO26 (TX), GPIO27 (RX) — internal to board, not on header

### LEDs
- LED4 green (power): NOT controllable (hardwired 3V3 → R10(330Ω) → LED → GND)
- LED5 cyan (IO5): controllable (IO5 → R9(330Ω) → LED → GND)
- LED2 cyan (IO26 / CAN TX): indicates CAN driver input activity (DI line to SN65HVD230)
- LED3 cyan (IO27 / CAN RX): indicates CAN receiver output activity (RO line from SN65HVD230)

### Buttons & switches
- SW1 = EN (reset) — tactile push button
- SW2 = BOOT (IO0) — tactile push button
- SW3 = power on/off — slide switch

### JP1 — CAN termination (120Ω)
- **Open JP1 when connecting to vehicle** (ECU has own termination)
- Bridge JP1 for bench testing (board is bus endpoint)
- Double termination = CAN bus errors

### Power from vehicle
- 12V from OBD2 pin 16 → TPS54202 → 3.3V
- Can power external peripherals via 3V3+GND header (TPS54202 supplies up to 2A total; ESP32 draws ~130mA typical with WiFi, leaving ~1.8A for external loads)
- OBD2 J1962 pin 16 = permanent battery +12V per standard. For Honda ADV350 6-pin connector, pin F = ignition-switched +12V

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

// Receive loop
twai_message_t msg;
if (twai_receive(&msg, pdMS_TO_TICKS(100)) == ESP_OK) {
    // msg.identifier, msg.data_length_code, msg.data[]
}
```

### TWAI errata fixes (add to sdkconfig)
```
CONFIG_TWAI_ERRATA_FIX_BUS_OFF_REC=y
CONFIG_TWAI_ERRATA_FIX_TX_INTR_LOST=y
CONFIG_TWAI_ERRATA_FIX_RX_FRAME_INVALID=y
CONFIG_TWAI_ERRATA_FIX_RX_FIFO_CORRUPT=y
CONFIG_TWAI_ERRATA_FIX_LISTEN_ONLY_DOM=y
```

### CAN frame serial output format
```
[elapsed_ms] #frame_count ID:0x{ID} [DLC] {hex bytes} (EXT/RTR flags)
```
Statistics logged every 10 seconds: total frames, elapsed, avg FPS.

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

### WiFi credentials via Kconfig (not hardcoded)
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

### OTA upload from Mac
```bash
curl -X POST -F "firmware=@build/app.bin" http://<ESP32-IP>/update
# or with mDNS:
curl -X POST -F "firmware=@build/app.bin" http://adv350.local/update
```

### Rollback pattern
```c
void app_main(void) {
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t state;
    if (esp_ota_get_state_partition(running, &state) == ESP_OK
        && state == ESP_OTA_IMG_PENDING_VERIFY) {
        // Run self-test, then:
        esp_ota_mark_app_valid_cancel_rollback();
    }
    // ... rest of app
}
```

### OTA implementation details
- HTTP multipart form upload — must skip boundary header (search for `\r\n\r\n`)
- Chunk size: 1024 bytes per `httpd_req_recv` call
- Uses `OTA_WITH_SEQUENTIAL_WRITES` flag
- Rollback validation: checks `esp_get_free_heap_size() > 50000` — minimal, not functional self-test
- Web server: `HTTPD_DEFAULT_CONFIG()` with `recv_wait_timeout=30`, `max_uri_handlers=4`
- Endpoints: `GET /` (HTML upload page), `POST /update` (firmware binary)

### mDNS setup
```bash
idf.py add-dependency "espressif/mdns^1.8.0"
```
```c
#include "mdns.h"
mdns_init();
mdns_hostname_set("adv350");  // → http://adv350.local
mdns_instance_name_set("ADV350 CAN Sniffer");
```

### WiFi gotchas
- `Expected CCMP frame` warnings on mobile hotspot are harmless — ignore
- DHCP IP may change on hotspot restart — use mDNS to avoid hardcoded IPs
- Must register disconnect handler that calls `esp_wifi_connect()` for auto-reconnect

## 6. Honda ADV350 OBD2 Reference

### Connector: 6-pin ISO 19689
```
┌─────────────┐
│  A   B   C  │
│  D   E   F  │
└─────────────┘
A = GND          B = CAN-H       C = SCS (service check)
D = K-Line       E = CAN-L       F = +12V (ignition switched)
```

### Protocol — CONFIRMED 2026-05-19
- CAN bus: ISO 11898, **500 kbps**, **29-bit extended CAN IDs only**
- **Standard OBD2 (11-bit 0x7DF) DOES NOT WORK** — ECU ignores completely
- **Honda UDS (29-bit)** is the correct protocol:
  - Request: `0x18DA10F1` (tester F1 → ECU 10)
  - Response: `0x18DAF110` (ECU 10 → tester F1)
  - Functional: `0x18DB33F1` (broadcast, also works)
- ECU: Keihin/Hitachi Astemo RH850 PGM-FI (shared with Forza 350, SH350)
- ECU **never broadcasts** — all data must be requested via UDS ReadDataByIdentifier (0x22)
- OBD compliance byte 0xF41C = 0x24 = ISO 15765-4 CAN + ISO 14230-4 KWP

### UDS Request Format
```c
// Single-frame ISO-TP: [PCI_len][SID][DID_high][DID_low][padding...]
// Example: Read RPM (DID 0xF40C)
twai_message_t msg = {
    .identifier = 0x18DA10F1,
    .extd = 1,
    .data_length_code = 8,
    .data = {0x03, 0x22, 0xF4, 0x0C, 0xAA, 0xAA, 0xAA, 0xAA},
};
// Response: 0x18DAF110 [05 62 F4 0C xx xx AA AA]
//   → RPM = (xx * 256 + xx) / 4
```

### Confirmed Working DIDs (0xF4xx = OBD2 PID + 0xF400)
| DID | PID | Parameter | Bytes | Formula | Unit |
|-----|-----|-----------|-------|---------|------|
| 0xF404 | 0x04 | Engine load | 1 | A*100/255 | % |
| 0xF405 | 0x05 | Coolant temp | 1 | A-40 | °C |
| 0xF406 | 0x06 | Short-term fuel trim | 2 | (A-128)*100/128 | % |
| 0xF40B | 0x0B | MAP | 1 | A | kPa |
| 0xF40C | 0x0C | RPM | 2 | (A*256+B)/4 | RPM |
| 0xF40D | 0x0D | Speed | 1 | A | km/h |
| 0xF40E | 0x0E | Ignition timing | 1 | (A/2)-64 | °BTDC |
| 0xF40F | 0x0F | Intake air temp | 1 | A-40 | °C |
| 0xF411 | 0x11 | Throttle | 1 | A*100/255 | % |

### NOT Available from ECU
Battery voltage, fuel rate, fuel level, injector PW, lambda/O2, gear position — all scanned ranges (D0, E0, 00-40, F4 second block) returned empty

### UDS Services
| Service | Name | Usage |
|---------|------|-------|
| 0x10 0x01 | Default session | Normal operation |
| 0x10 0x03 | Extended session | Same DIDs + F186 |
| 0x22 | ReadDataByIdentifier | Primary data access |
| 0x3E 0x00 | TesterPresent | Keep-alive every ~3s |

### Discovery Timeline
1. LISTEN_ONLY → 0 frames (ECU never broadcasts)
2. NORMAL + 0x7DF (11-bit) → TXerr:105, no ACK (ECU ignores standard OBD2)
3. 250kbps test → BUSerr:80k (confirmed 500kbps correct)
4. **0x18DA10F1 (29-bit) → ECU responds!** First Honda ADV350 CAN RE success
5. DID brute-force scan → 14 DIDs confirmed in F4xx + F1xx ranges

### Safety
- **READ-ONLY** — never write CAN frames to ECU until fully reverse-engineered and bench-tested
- Open JP1 (remove 120Ω termination) on ESP-OBD2 board
- Inline fuse 1-2A on 12V line
- TVS diode for voltage spikes

## 7. ESP-NOW (Peer-to-Peer between ESP32 boards)

- Works cross-chip: ESP32 ↔ ESP32-S3
- No WiFi router needed
- Can coexist with WiFi STA (same channel) and BLE
- Max payload: 250 bytes (v1.0)
- Range: ~150-200m outdoor, ~20-40m indoor
- Latency: <20ms
- Disable modem sleep when using with WiFi STA: `esp_wifi_set_ps(WIFI_PS_NONE)`

## 8. Thermal & Power Reference

### Operating temperature
| Component | Operating | Storage | Note |
|-----------|-----------|---------|------|
| ESP32-WROOM-32E | -40 to +85°C | -40 to +105°C | |
| ESP32-S3 N16R8 (Octal PSRAM) | -40 to **+65°C** | -40 to +105°C | PSRAM is the limiting component; ESP32-S3 silicon rated to +85°C |
| SN65HVD230 | -40 to +85°C | -40 to +150°C | |

### Power consumption
| Component | Average | Peak |
|-----------|---------|------|
| ESP32 (WiFi+BLE active) | 100-130 mA | 240 mA |
| SIM7600 (4G LTE) | 200-500 mA | 2000 mA |
| SIM7080G (CAT-M1) | 50-150 mA | 500 mA |
| NEO-M9N GPS | 31 mA | 50 mA |

### Motorcycle deployment checklist
- [ ] Inline fuse 1-2A on 12V input
- [ ] TVS diode (P6KE15A) for voltage spikes
- [ ] Open JP1 (CAN termination) on ESP-OBD2
- [ ] Verify Honda 6-pin connector pin F cuts power on ignition off
- [ ] IP65 waterproof enclosure
- [ ] Strain relief on OBD2 cable
- [ ] Metal enclosure or heatsink if running SIM module

## 9. BLE / NimBLE on ESP32 (ESP-IDF)

### RAM constraint — WiFi + BLE cannot coexist on ESP32-WROOM-32E
- WiFi STA ~60-80KB + NimBLE ~30-40KB = ~100-120KB combined
- ESP32-WROOM-32E has ~320KB usable DRAM (no PSRAM)
- Crash: `StoreProhibited` (NULL pointer) during BLE init when WiFi also initialized
- **Solution**: Run WiFi and BLE in separate modes (DEV=WiFi, PROD=BLE), never simultaneously

### NimBLE crash root causes (3 issues stacked)
1. **`CONFIG_BTDM_CTRL_MODEM_SLEEP=y` (default)** → controller crash during handshake → NULL pointer at `ble_hs_id_infer_auto`
2. **Max connections mismatch** — controller allows 3 but NimBLE host allows 1 → connection table inconsistency
3. **Bond storage + NVS persist** — allocation failure under memory pressure

### Critical sdkconfig for NimBLE on ESP32
```
CONFIG_BT_ENABLED=y
CONFIG_BTDM_CTRL_MODE_BLE_ONLY=y
CONFIG_BTDM_CTRL_BLE_MAX_CONN=1        # Must match NIMBLE_MAX_CONNECTIONS
CONFIG_BTDM_CTRL_MODEM_SLEEP=n          # CRITICAL — default y causes crash
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1      # Must match BTDM_CTRL_BLE_MAX_CONN
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=5120
CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_INTERNAL=y
```

### BLE disconnects after ~10s without GATT service
- No data exchange → BLE connection supervision timeout (4-20s)
- Fix: Add GATT service with at least one characteristic the central can interact with

### GATT registration order (must follow this sequence)
```c
ble_svc_gap_device_name_set("ADV350");
ble_svc_gap_init();
ble_svc_gatt_init();
ble_gatts_count_cfg(gatt_svcs);   // count first
ble_gatts_add_svcs(gatt_svcs);    // then add
```

### GATT service gotchas
- Notify-only characteristic with `access_cb = NULL` → `assert failed: ble_hs_event_start_stage2`
- **Every characteristic must have an `access_cb`**, even notify-only ones
- Fix: Make characteristic `READ | NOTIFY` with a real `access_cb`

### BLE UUID endianness
- `BLE_UUID128_INIT` stores bytes in **little-endian** order
- UUID string is big-endian
- Example: `BLE_UUID128_INIT(0xf0, 0xde, 0xbc, 0x9a, 0x78, 0x56, 0x34, 0x12, 0x78, 0x56, 0x34, 0x12, 0x78, 0x56, 0x34, 0x12)` = UUID `12345678-1234-5678-1234-56789abcdef0`
- Flutter app must use same byte-order — mismatch causes **silent** service discovery failure

### GATT CAN service UUIDs
```
Service:   12345678-1234-5678-1234-56789abcdef0
Frame chr: 12345678-1234-5678-1234-56789abcdef1  (READ|NOTIFY)
Ctrl chr:  12345678-1234-5678-1234-56789abcdef2  (READ|WRITE)
```

### CAN frame notification format
`[ID(4 bytes LE)][DLC(1 byte)][DATA(0-8 bytes)]` = max 13 bytes per notification

### Control characteristic protocol
- Write `0x00` = stop CAN live notify
- Write `0x01` = start CAN live notify
- Write `0x02` = ping (blinks LED 3 times: 30ms on, 200ms off)

### BLE advertising — NO service UUID in packet
- Advert includes: flags, complete name ("ADV350"), TX power level
- Service UUID is NOT in advertisement → **must scan by device name, not UUID filter**
- Params: `conn_mode=BLE_GAP_CONN_MODE_UND`, `disc_mode=BLE_GAP_DISC_MODE_GEN`, public address, forever duration

### GAP event handler must handle
- `CONNECT` — store conn_handle
- `DISCONNECT` — reset handle, turn off notify, re-advertise
- `SUBSCRIBE` — toggle live notify based on cur_notify
- `MTU` — log new MTU value
- `ENC_CHANGE` — log encrypted/authenticated/bonded state
- `REPEAT_PAIRING` — delete stale bond, return RETRY
- `PASSKEY_ACTION` — return `BLE_HS_ENOTSUP` (Just Works needs no passkey)

### ESP-IDF issue references for NimBLE crashes
- BTDM Modem Sleep crash: [#11624](https://github.com/espressif/esp-idf/issues/11624), [#15128](https://github.com/espressif/esp-idf/issues/15128)
- Max connections mismatch: [#13723](https://github.com/espressif/esp-idf/issues/13723)

## 10. BLE Pairing (Consumer Device Pattern)

### Android Settings pairing failure (status=13) — root causes
- `NVS_PERSIST=n` → bond keys lost after reboot → Android reconnect fails
- `MAX_BONDS=1` → re-pair slot full
- Key distribution missing `ID` key → Android uses RPA, ESP32 can't resolve
- No `REPEAT_PAIRING` handler → "Forget" + re-pair fails
- Passkey handler had 7-digit passkey (BLE max is **6 digits**)

### Consumer BLE pairing pattern (smartwatch/OBD2 dongle style)
- Use **Just Works** (no PIN) — same as most consumer devices
- Must have: NVS persist + bonds≥3 + key distribution with ID + repeat pairing handler
- `ble_store_config_init()` does NOT exist in ESP-IDF v5.5.3 — ESP-IDF handles store init automatically when `NVS_PERSIST=y`

### Android Settings disconnect after pairing is NORMAL
- Android Settings pairs + bonds then drops connection — expected behavior
- An app (nRF Connect, Flutter app) must hold the connection
- Same behavior as smartwatches — pair from Settings, then open companion app

### Key distribution fix (essential for Android RPA)
```c
// Before (broken — Android can't resolve RPA):
ble_hs_cfg.sm_our_key_dist = BLE_SM_PAIR_KEY_DIST_ENC;
ble_hs_cfg.sm_their_key_dist = BLE_SM_PAIR_KEY_DIST_ENC;

// After (working):
ble_hs_cfg.sm_our_key_dist = BLE_SM_PAIR_KEY_DIST_ENC | BLE_SM_PAIR_KEY_DIST_ID;
ble_hs_cfg.sm_their_key_dist = BLE_SM_PAIR_KEY_DIST_ENC | BLE_SM_PAIR_KEY_DIST_ID;
```

### Repeat pairing handler (essential for "Forget" + re-pair)
```c
case BLE_GAP_EVENT_REPEAT_PAIRING: {
    struct ble_gap_conn_desc desc;
    if (ble_gap_conn_find(event->repeat_pairing.conn_handle, &desc) == 0) {
        ble_store_util_delete_peer(&desc.peer_id_addr);
    }
    return BLE_GAP_REPEAT_PAIRING_RETRY;
}
```

### Working sdkconfig for BLE pairing
```
CONFIG_BT_NIMBLE_NVS_PERSIST=y
CONFIG_BT_NIMBLE_MAX_BONDS=3
CONFIG_BT_NIMBLE_SM_SC=y
CONFIG_BT_NIMBLE_SM_LEGACY=y
CONFIG_BT_NIMBLE_SECURITY_ENABLE=y
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=256
CONFIG_BT_NIMBLE_ACL_BUF_SIZE=260
CONFIG_BT_NIMBLE_ACL_BUF_COUNT=12
```

### POC security config (Just Works, no bonding — avoids MTU collision)
```c
ble_hs_cfg.sm_io_cap = BLE_SM_IO_CAP_NO_IO;
ble_hs_cfg.sm_bonding = 0;   // Every connection fresh — no bond cache bugs
ble_hs_cfg.sm_mitm = 0;
ble_hs_cfg.sm_sc = 0;
ble_att_set_preferred_mtu(256);
```

### Latest POC: security disabled entirely in sdkconfig
When bonding causes more problems than it solves (MTU collision, reconnect failures), disable at sdkconfig level too:
```
CONFIG_BT_NIMBLE_NVS_PERSIST=n
CONFIG_BT_NIMBLE_MAX_BONDS=0
CONFIG_BT_NIMBLE_SM_SC=n
CONFIG_BT_NIMBLE_SM_LEGACY=n
CONFIG_BT_NIMBLE_SECURITY_ENABLE=n
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=8192  # increased from 5120
```
This is the most reliable config for POC — zero bond-related issues.

### BLE reboot on disconnect (brute-force reliability)
NimBLE re-advertising after disconnect can be unreliable. Rebooting the ESP32 after disconnect is more reliable for POC:
```c
case BLE_GAP_EVENT_DISCONNECT:
    ESP_LOGI(TAG, "BLE: disconnected, reason=%d", event->disconnect.reason);
    xTaskCreate(ble_reboot_task, "ble_reboot", 2048, NULL, 1, NULL);
    break;
// ...
static void ble_reboot_task(void *arg) {
    vTaskDelay(pdMS_TO_TICKS(300));
    esp_restart();
}
```

### Add `ble_on_reset` callback for diagnostics
```c
ble_hs_cfg.reset_cb = ble_on_reset;
// ...
static void ble_on_reset(int reason) {
    ESP_LOGE(TAG, "BLE: host reset, reason=%d", reason);
}
```

### BTDM_CTRL_MODEM_SLEEP=n impact
- BLE controller doesn't sleep → +10-20mA continuous
- For vehicle-powered use case: negligible
- Benefit: faster BLE response, no sleep/wake timing bugs

## 11. Flutter BLE Integration

### Android BLE permissions (Android 12+)
- Manifest: `BLUETOOTH`, `BLUETOOTH_ADMIN`, `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`, `ACCESS_FINE_LOCATION`
- Runtime permission request needed before scanning
- Without permissions: scan returns zero results **silently** (no error)
- Call `FlutterBluePlus.turnOn()` before scan to prompt user if BT disabled

### Flutter BLE package
- `flutter_blue_plus: ^2.3.2`

### Scan by device name (not UUID — firmware doesn't advertise service UUID)
```dart
FlutterBluePlus.scanResults.listen((r) {
  for (final result in r) {
    if (result.device.platformName == 'ADV350') { ... }
  }
});
await FlutterBluePlus.startScan(timeout: Duration(seconds: 5));
```

### Flutter connection flow (full working sequence)
```dart
try { await device.removeBond(); } catch (_) {}     // clear stale bond
try { await device.clearGattCache(); } catch (_) {}  // clear stale GATT cache
await Future.delayed(Duration(milliseconds: 500));    // let Android BLE stack settle
await device.connect(autoConnect: false, timeout: Duration(seconds: 10));
try { await device.requestMtu(256); } catch (_) {}   // non-fatal if fails
final services = await device.discoverServices();
// find service by UUID string match, subscribe to frame chr notifications
```

### Auto-connect last device (SharedPreferences)
```dart
static const String _lastDeviceKey = 'last_ble_device_id';
// On connect success:
final prefs = await SharedPreferences.getInstance();
await prefs.setString(_lastDeviceKey, device.remoteId.str);
// On app launch — scan for last device:
final lastId = await prefs.getString(_lastDeviceKey);
// scan, find match by remoteId.str == lastId, auto-connect
// On intentional disconnect — clear:
await prefs.remove(_lastDeviceKey);
```
Requires `shared_preferences` package in pubspec.yaml.

### Auto-reconnect (3 attempts, re-scan each time)
```dart
for (int i = 0; i < 3; i++) {
  if (!_autoReconnect || _device != null) return;
  await Future.delayed(Duration(seconds: 3 + i * 2));  // 3s, 5s, 7s
  final results = await scan(timeout: Duration(seconds: 3));
  if (results.isEmpty || !_autoReconnect) continue;
  try { await connect(results.first.device); return; } catch (_) {}
}
```
Key: **re-scan before reconnect** — don't reuse stale `BluetoothDevice` object. Set `_autoReconnect = false` on intentional disconnect.

### MTU: skip negotiation entirely (POC)
- MTU request removed from connect flow — use default `mtu: 23`
- Connect with: `device.connect(mtu: 23, autoConnect: false, timeout: 10s)`
- CAN frames are max 13 bytes — default MTU (23 - 3 overhead = 20 payload) is sufficient
- Avoids ALL MTU collision/timeout issues (status=133, status=8, status=22)
- For production with larger payloads, negotiate MTU AFTER encryption completes

### Android GATT cache issue on reconnect
- After disconnect, Android uses cached GATT state that conflicts
- Fix: Call `device.removeBond()` + `device.clearGattCache()` + 500ms delay before reconnect

### CAN frame parsing in Dart
```dart
final canId = data[0] | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
final dlc = data[4];
final canData = data.length > 5 ? data.sublist(5) : <int>[];
```
Currently placeholder mapping — Honda CAN IDs not yet reverse-engineered.

## 12. GPIO5 / LED on ESP-OBD2

### GPIO5 is a strapping pin on ESP32
- Has internal pull-up during boot → LED connected to GPIO5 lights up during bootloader (~500ms)
- Software cannot control GPIO5 until `app_main()` runs
- Must call `gpio_reset_pin(GPIO_NUM_5)` + disable pull-up explicitly

### LED is active-LOW
- GPIO=0 → LED ON, GPIO=1 → LED OFF
- Blink code must end with `gpio_set_level(LED_GPIO, 1)` to turn off

### LEDC PWM and GPIO from BLE callback — use dedicated task
- BLE GAP/GATT callbacks run in NimBLE host task context
- LEDC driver calls and even `vTaskDelay` inside callbacks are unreliable
- **Solution**: Dedicated `led_task` polls a `volatile uint8_t pending_led_blink` flag
- BLE callback sets flag, LED task does the GPIO work in its own context
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

## 13. Dual Mode (DEV/PROD) via IO0 Button

### Mode switching
- SW2 (BOOT/IO0) is free after boot — usable as mode switch
- Pattern: Poll IO0 in FreeRTOS task, hold 5 seconds → save new mode to NVS → reboot
- NVS persists mode across reboots
- DEV mode: WiFi + OTA + CAN sniffer (no BLE)
- PROD mode: BLE + CAN sniffer (no WiFi)

### NVS erase resets mode to default
- After `esptool.py erase_flash`, NVS is cleared
- Mode reverts to default (DEV)
- Must re-switch to PROD mode after erase (IO0 hold 5s)

## 14. Toolchain Gotchas (macOS Apple Silicon)

### espup compilation fails with x86 Homebrew
- `liblzma` via Homebrew at `/usr/local` is x86_64, system is arm64
- `LIBRARY_PATH` and `RUSTFLAGS` both fail to fix architecture mismatch
- **Solution**: Download espup prebuilt binary from GitHub releases (aarch64-apple-darwin)

### cargo-generate liblzma fix
- Same liblzma issue as espup
- Fix: Set `LZMA_API_STATIC=1` to force build from source, or download prebuilt binary

### ESP-IDF template creates nested .git
- `cargo generate` creates its own `.git` directory inside generated project
- Must delete nested `.git` before committing to parent repo

### espflash monitor requires TTY
- `espflash monitor` fails in non-interactive terminals (Claude Code): `Failed to initialize input reader`
- Use Python serial reader workaround (see section 2)

### ESP32-S3 GPIO 48 LED on clone boards
- Official DevKitC-1 has WS2812 (NeoPixel) RGB LED on GPIO 48
- Clone boards may have plain green LED — GPIO toggle shows green only
- WS2812 requires RMT driver, not plain GPIO
- Use `TxChannelDriver` + `SimpleEncoder` (new RMT API), not `TxRmtDriver` (legacy)

## 15. TWAI / CAN in esp-idf-hal (Rust)

### Module naming
- Module is `can` not `twai` in esp-idf-hal
- API: `CAN<'d>` not `CanDriver`
- Frame construction: `Frame::new(id, flags, data)` — does not use `StandardId`

### Self-test mode
- Must call `start()` before `transmit()` — else `ESP_ERR_INVALID_STATE`
- Frames must have `SelfReception` flag set for loopback — without it, `receive()` blocks forever
- After 2-3 successful loopback frames, controller may enter bus-off (normal without transceiver)

### WiFi note
- ESP32 only supports **2.4 GHz WiFi**
- Mobile hotspot set to "Dual Band" or "5 GHz" → ESP32 won't find it
- Must set hotspot to **2.4 GHz only**

## 16. Complete Working sdkconfig.defaults (Latest v7)

Evolved through v1-v7 across 4 sessions. v7 disables security entirely for maximum POC reliability.

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

# BLE controller — ESP32 original chip
CONFIG_BT_ENABLED=y
CONFIG_BTDM_CTRL_MODE_BLE_ONLY=y
CONFIG_BTDM_CTRL_BLE_MAX_CONN=1
CONFIG_BTDM_CTRL_MODEM_SLEEP=n          # THE KEY FIX — default y causes crash

# NimBLE host
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1       # Must match BTDM_CTRL_BLE_MAX_CONN
CONFIG_BT_NIMBLE_ROLE_BROADCASTER=y
CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=y
CONFIG_BT_NIMBLE_ROLE_CENTRAL=n
CONFIG_BT_NIMBLE_ROLE_OBSERVER=n
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=8192  # increased from 5120 for stability
CONFIG_BT_NIMBLE_NVS_PERSIST=n              # v7: disabled — no bonding
CONFIG_BT_NIMBLE_MAX_BONDS=0               # v7: zero bonds
CONFIG_BT_NIMBLE_SM_SC=n                   # v7: disabled
CONFIG_BT_NIMBLE_SM_LEGACY=n               # v7: disabled
CONFIG_BT_NIMBLE_SECURITY_ENABLE=n         # v7: disabled entirely
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=256
CONFIG_BT_NIMBLE_ACL_BUF_SIZE=260
CONFIG_BT_NIMBLE_ACL_BUF_COUNT=12
CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_INTERNAL=y
```

### sdkconfig evolution summary
| Version | Change | Why |
|---------|--------|-----|
| v1 | Minimal CAN | First build |
| v2 | +OTA partitions, rollback | Dual OTA |
| v3 | +BLE (crashed) | Missing MODEM_SLEEP fix |
| v4 | BLE disabled | Fallback |
| v5 | +MODEM_SLEEP=n, MAX_CONN match | BLE working |
| v6 | +NVS_PERSIST=y, BONDS=3, SM_SC=y | Pairing fix |
| v7 | Security disabled, BONDS=0, stack 8192 | Max POC reliability |

## 17. Confirmed Wiring Configurations

### FT232 → ESP-OBD2 (data only, separate power)
```
FT232          ESP-OBD2
TX     ------> RXD (RX0 on header)
RX     <------ TXD (TX0 on header)
GND    ------> GND
3V3    ------> NOT CONNECTED (causes overcurrent!)
```

### ESP32-S3 power → ESP-OBD2
```
ESP32-S3 (USB to Mac)    ESP-OBD2
3V3     ---------------> 3V3
GND     ---------------> GND (shared with FT232 via Mac USB ground)
```
When both devices connected to same Mac via USB, GND shared through Mac's USB ground plane.

### Vehicle deployment
```
Honda ADV350 6-pin OBD
Pin A (GND)  ---> ESP-OBD2 GND (via OBD2 adapter)
Pin B (CAN-H) --> ESP-OBD2 CAN-H (via SN65HVD230)
Pin E (CAN-L) --> ESP-OBD2 CAN-L (via SN65HVD230)
Pin F (+12V)  --> ESP-OBD2 VIN (TPS54202 buck → 3.3V)
```
FT232 for serial monitoring: TX/RX/GND only, no 3V3 (vehicle powers board).

### Download mode procedure (ESP-OBD2, no auto-reset)
1. Disconnect power completely
2. Connect IO0 → GND with jumper wire
3. Reconnect power (board enters download mode)
4. Flash immediately: `idf.py -p /dev/cu.usbserial-110 flash`
5. Disconnect IO0 from GND, then reset

If board is crash-looping: must power off FIRST, then IO0→GND, then power on.

## 18. FreeRTOS Task Architecture

| Task | Stack | Priority | Core | Function |
|------|-------|----------|------|----------|
| main (app_main) | 8192 | 1 (default) | any | Init WiFi/BLE, start CAN, start mode switch |
| can_sniffer_task | 4096 | 5 | **Core 1** | TWAI receive loop + serial print + BLE notify |
| mode_switch_task | 2048 | 10 | any | Poll IO0 button every 200ms, 5s hold → switch |
| led_task | 2048 | 3 | **Core 1** | Poll `pending_led_blink` flag, GPIO blink |
| ble_reboot_task | 2048 | 1 | any | 300ms delay then `esp_restart()` on disconnect |
| nimble_host_task | 8192 | (NimBLE default) | Core 0 | BLE host processing |

**Core pinning strategy**: CAN sniffer + LED on Core 1, BLE/WiFi stack on Core 0. Use `xTaskCreatePinnedToCore(..., 1)` for real-time tasks.

## 19. Architecture Decisions & Pivots

### WiFi → BLE pivot
- Original: ESP32 → WiFi → Supabase direct
- Problem: No WiFi available on a moving motorcycle
- New: ESP32 → BLE → Flutter app (phone) → Cloud
- Rationale: Low power, no SIM cost, app buffers for cloud sync

### Supabase → Cloudflare D1 pivot
At 10 samples/sec, 2hr/day riding = 2.6GB/year:
- Supabase free tier: 500MB (overflows month 1 with raw CAN)
- Cloudflare D1 free tier: 5GB + 100K requests/day
- Decision: Cloudflare D1 + Workers + Pages

### ESP-OBD2 vs ESP32-S3 N16R8 usage strategy
- ESP-OBD2 (ESP32, 4MB, no PSRAM): CAN capture & reverse engineering phase
- ESP32-S3 N16R8 (16MB, 8MB PSRAM): Production permanent deployment
- Rationale: ESP-OBD2 has built-in CAN transceiver + OBD2 plug, perfect for prototyping

## 20. CLAUDE.md vs Actual Code (Discrepancies)

| CLAUDE.md says | Actual implementation |
|---|---|
| ESP32-S3 N16R8 (16MB + 8MB PSRAM) | ESP32-WROOM-32E (4MB, no PSRAM) |
| Rust + esp-idf-hal | C + ESP-IDF v5.5.3 |
| GPIO 4 TX, GPIO 5 RX | GPIO 26 TX, GPIO 27 RX |
| LittleFS offline storage | Not implemented; no LittleFS partition |
| Supabase cloud sync | Not implemented; SQLite local only |
| LE Secure Connections + bonding | Just Works, no bonding (POC) |
| tokio async runtime | FreeRTOS tasks (C) |
| 16MB partition with 4MB LittleFS | 4MB partition, dual OTA at 1.75MB each |

CLAUDE.md describes the final ESP32-S3 target architecture. Actual code runs on ESP-OBD2 board for prototyping phase. The Rust ESP32-S3 firmware exists in `firmware/` dir but is not the active firmware.

## 21. Rust ESP-IDF HAL Specifics (for ESP32-S3 migration)

### CAN module naming
- Module is `can` not `twai` in esp-idf-hal v0.46
- API: `CanDriver<'d>`, not `CAN`
- Frame: `Frame::new(id, extended_flag, data)` — 3 args, not 2

### Working CAN wrapper
```rust
use esp_idf_hal::can;
let config = can::config::Config::new()
    .timing(can::config::Timing::B500K)
    .mode(can::config::Mode::ListenOnly);
let mut driver = can::CanDriver::new(peripheral, tx_pin, rx_pin, &config)?;
driver.start()?;
let frame = driver.receive(BLOCK)?;
```

### Cargo.toml — git patches
Template generates `[patch.crates-io]` pointing to git repos. crates.io versions of ESP crates will conflict — don't mix.

### cargo-generate template values file
```toml
# /tmp/esp-template-values.toml — avoid interactive mode + bool type errors
[values]
mcu = "esp32s3"
std = true
ci = "none"
devcontainer = false
advanced = false
```
```bash
cargo generate esp-rs/esp-idf-template cargo --name firmware \
  --template-values-file /tmp/esp-template-values.toml --silent
```

### Tool versions confirmed working together
| Tool | Version |
|------|---------|
| rustc | 1.95.0 (stable-aarch64-apple-darwin) |
| espup | 0.17.1 (prebuilt binary) |
| espflash | 4.4.0 |
| cargo-generate | 0.23.8 (with LZMA_API_STATIC=1) |
| ESP-IDF | v5.5.3 |

## 22. Error Messages Quick Reference

| Error | Context | Solution |
|-------|---------|----------|
| `StoreProhibited` at `ble_hs_id_infer_auto` | NimBLE init | `CONFIG_BTDM_CTRL_MODEM_SLEEP=n` |
| `assert failed: ble_hs_event_start_stage2` | GATT init | Every characteristic needs non-NULL `access_cb` |
| `status=13` (BLE pairing) | Android pairing | NVS_PERSIST=y, MAX_BONDS=3, add ID to key dist |
| `status=133` (GATT_ERROR) | MTU on reconnect | Disable bonding or match ACL buffer to MTU |
| `status=8` / `status=22` | MTU timeout | Board not responding — check `ble_att_set_preferred_mtu()` |
| `ESP_ERR_INVALID_STATE` | TWAI transmit | Must call `twai_start()` before transmit |
| `Failed to initialize input reader` | espflash monitor | Needs TTY — use Python serial reader or real terminal |
| `Device not configured` (errno 6) | Serial read | FT232 disconnected — overcurrent, don't power board from FT232 3V3 |
| `_lzma_stream_decoder not found` | cargo install espup | x86 Homebrew on ARM Mac — use prebuilt binary or `LZMA_API_STATIC=1` |
| `E BOD: Brownout detector was triggered` | BLE init on bench power | `CONFIG_ESP_BROWNOUT_DET=n` in sdkconfig (bench only, re-enable on vehicle 12V) |
| TWAI TXerr:105, BUSerr:481, 0 frames | OBD2 11-bit on Honda | Honda ignores 11-bit — use 29-bit extended (0x18DA10F1) |
| TWAI BUSerr:80000+ | Wrong CAN bitrate | Confirm 500kbps. 250k on Honda = massive errors |

## 23. WiFi CAN Log Viewer (DEV Mode)

HTTP endpoints for remote CAN debugging without serial:
- `/log` — Live CAN frame viewer (dark theme, JS polling 200ms, auto-scroll)
- `/api/frames?since=N` — JSON: frames + total + fps + heap + TWAI status (state, tx_err, rx_err, bus_err)
- `/api/scan?range=xx&go=1` — DID brute-force scanner (pause sniffer+poll during scan)
- `/api/diag` — CAN diagnostic results (self-test, bitrate scan, protocol probe)

OTA flash via WiFi: `curl -X POST -F "firmware=@build/can_sniffer.bin" http://adv350.local/update`

## 24. Honda ADV350 CAN Debugging Methodology

### TWAI Status Interpretation
| Condition | Meaning |
|-----------|---------|
| Frames=0, all errors=0 | Bus completely dead — no CAN signal |
| TXerr high, RXerr=0, BUSerr>0 | TX not ACK'd — no other node on bus OR wrong protocol |
| BUSerr massive (80k+) | Wrong bitrate |
| TWAI:BUS_OFF | TX error counter ≥256 — controller shut down |

### Protocol Discovery Sequence (what worked for Honda ADV350)
1. LISTEN_ONLY → 0 frames = ECU doesn't broadcast
2. NORMAL + 0x7DF 11-bit → no ACK = ECU ignores standard OBD2
3. 250kbps test → BUSerr explodes = confirms 500kbps
4. NORMAL + no TX → 0 everything = bus truly silent without requests
5. **Multi-protocol probe → 29-bit extended 0x18DA10F1 = ECU responds!**

### Data Integrity Rule
- ALWAYS store `raw_ble_hex` (complete BLE packet) alongside decoded values
- NEVER discard raw bytes — schema changes can re-decode from raw
- NEVER fabricate/impute lost data and call it "recovered"
- Test data round-trip: save → read → verify ALL original bytes preserved

## 25. OTA via WiFi (DEV Mode, Honda Connected)

When board is on vehicle 12V + connected to phone hotspot:
1. Board auto-connects WiFi on boot (DEV mode)
2. OTA: `curl -X POST -F "firmware=@build/can_sniffer.bin" http://adv350.local/update`
3. Board reboots with new firmware — no need for download mode / FT232
4. Much faster iteration cycle when debugging on vehicle
