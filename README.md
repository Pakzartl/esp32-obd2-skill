# esp32-obd2-skill

Agent skill for ESP32 OBD2/CAN bus projects — covers firmware, BLE, Flutter app, OTA, and vehicle integration.

## Install

```bash
npx skills add Pakzartl/esp32-obd2-skill
```

## What's inside

Full-stack reference for building ESP32-based OBD2 data loggers:

- **Hardware**: ArtronShop ESP-OBD2 board, SN65HVD230 transceiver, wiring, pinouts
- **CAN bus**: TWAI driver setup, listen-only sniffer, errata fixes, frame format
- **BLE/NimBLE**: Pairing, GATT services, crash fixes, sdkconfig, UUID endianness
- **Flutter BLE**: Permissions, scan/connect, auto-reconnect, CAN frame parsing
- **OTA**: Dual partition, rollback, HTTP upload
- **OBD2**: Honda 6-pin connector, standard PIDs, safety guidelines
- **Deployment**: Thermal limits, power, fuses, waterproofing checklist

## Compatibility

- [Agent Skills](https://agentskills.io) open standard
- Works with: Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and 30+ other agents

## License

MIT
