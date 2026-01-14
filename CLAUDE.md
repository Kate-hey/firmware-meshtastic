# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Meshtastic firmware for LoRa mesh networking - long-range, low-power communication without internet/cellular infrastructure. Supports ESP32, nRF52, RP2040/RP2350, STM32WL, and Linux platforms.

## Build Commands

```bash
# Build specific hardware target
pio run -e tbeam

# Build and upload to device
pio run -e tbeam -t upload

# Build native/Linux version
pio run -e native

# Run unit tests
pio test -e native

# Format code before commits
trunk fmt

# Check code quality
trunk check

# Regenerate protobuf code
bin/regen-protos.sh
```

## Architecture

### Source Structure
- `src/mesh/` - Core mesh networking (Router, NodeDB, Channels, radio interfaces)
- `src/modules/` - Feature modules (Position, Telemetry, TextMessage, etc.)
- `src/graphics/` - Display drivers and UI
- `src/platform/` - Platform-specific code (ESP32, nRF52, etc.)
- `src/gps/` - GPS handling
- `src/mqtt/` - MQTT bridge for internet connectivity
- `variants/` - Hardware variant definitions with pin configs

### Module System

Modules inherit from `MeshModule` or `ProtobufModule<T>`:
- `handleReceivedProtobuf()` - Process incoming packets
- `allocReply()` - Generate response packets
- `runOnce()` - Periodic task execution (returns next run interval in ms)

Register new modules in `src/modules/Modules.cpp`.

### Hardware Variants

Each variant has:
- `variant.h` - Pin definitions (`LORA_CS`, `SX126X_DIO1`, etc.) and hardware capabilities (`HAS_GPS`, `HAS_SCREEN`)
- `platformio.ini` - Build configuration extending base configs like `esp32_base`

### Configuration Access

- `config.*` - Device configuration (LoRa, position, power)
- `moduleConfig.*` - Module-specific configuration
- `channels.*` - Channel configuration

### Default Values and Scaling

Use `Default` class helpers from `src/mesh/Default.h`:
- `Default::getConfiguredOrDefaultMs(configured, default)`
- `Default::getConfiguredOrMinimumValue(configured, min)`
- `Default::getConfiguredOrDefaultMsScaled(configured, default, numOnlineNodes)` - Scales based on network congestion

### Conditional Compilation

```cpp
#if !MESHTASTIC_EXCLUDE_GPS        // Feature exclusion
#ifdef ARCH_ESP32                   // Architecture-specific
#if defined(USE_SX1262)            // Radio-specific
#ifdef HAS_SCREEN                   // Hardware capability
```

## Coding Conventions

- Classes: `PascalCase`
- Functions/Methods: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Logging: `LOG_DEBUG`, `LOG_INFO`, `LOG_WARN`, `LOG_ERROR`
- Thread safety: Use `concurrency::Lock` for mutexes, `SPILock` for SPI access

## Protobuf

- Definitions in `protobufs/meshtastic/*.proto`
- Generated code in `src/mesh/generated/`
- Message types prefixed with `meshtastic_`
- Regenerate with `bin/regen-protos.sh`

## Traffic Management

The mesh has limited bandwidth. When modifying broadcast intervals:
- Respect minimum intervals on default/public channels
- Use `Default::getConfiguredOrMinimumValue()` to enforce minimums
- Consider `numOnlineNodes` scaling for congestion control
- Use `IF_ROUTER(routerVal, normalVal)` for role-based defaults
