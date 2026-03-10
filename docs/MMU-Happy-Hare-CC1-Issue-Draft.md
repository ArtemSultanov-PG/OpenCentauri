# Feature Request: Happy-Hare Type B MMU Support for Elegoo Centauri Carbon 1 (CC1)

## Summary

This issue proposes adding native support for **Happy-Hare** Type B multi-material system to the **Elegoo Centauri Carbon 1 (CC1)** variant, which includes a dedicated "Multicolor" connector on the mainboard but lacks official AMS/MMU support in OpenCentauri firmware.

## Hardware Analysis

### CC1 Mainboard "Multicolor" Connector

The CC1 mainboard includes a dedicated **JST-XHB-5P** connector specifically designed for MMU/AMS integration:

| Pin | Marking | Function | Notes |
|-----|---------|----------|-------|
| 1 | 24V | +24V | Power for MMU stepper motors |
| 2 | GND | GND | Ground |
| 3 | 5V | +5V | Can be switched OFF to reset MMU MCU |
| 4 | DM | USB-DM | USB 2.0 data (D-) |
| 5 | DP | USB-DP | USB 2.0 data (D+) |

**Reference:** [`docs/hardware/CC1/mainboard.md`](docs/hardware/CC1/mainboard.md:111-121)

### CC2 CANVAS Module (Reference Implementation)

For comparison, the CC2 variant uses the **CANVAS** multimaterial module, which implements a **Type B Happy-Hare design**:

- **MCU:** GD32F303RCT6
- **Stepper Drivers:** 4x AT8833 (DRV8833 clone)
- **Motors:** 4x permanent magnet stepper motors
- **Filament Detection:** Hall effect sensors (similar to Flashforge IFS)
- **RFID Board:** I2C connection for filament information
- **Filament Multiplexer:** Mounted directly to extruder housing with 4mm OD metal tube

**Reference:** [`docs/hardware/CC2/CANVAS.md`](docs/hardware/CC2/CANVAS.md:1-67)

### Key Architectural Difference

| Aspect | CC1 (Target) | CC2 (CANVAS) |
|--------|--------------|--------------|
| **Connector** | JST-XHB-5P "Multicolor" (USB 2.0) | USB-C to toolhead board (Serial) |
| **Communication** | USB 2.0 protocol | Serial protocol |
| **MMU Location** | External (user-provided) | Integrated CANVAS module |
| **Multiplexer** | External (user-provided) | Integrated near toolhead |

## Technical Requirements

### 1. Happy-Hare Protocol Analysis

Happy-Hare Type B interface uses USB communication between the mainboard and MMU MCU. Based on the Happy-Hare documentation:

- **Protocol:** USB CDC-ACM (Communication Device Class - Abstract Control Model)
- **MCU:** Typically STM32 or GD32 series
- **Baud Rate:** Configurable (typically 115200 or 921600)
- **Command Structure:** Binary protocol with command/response packets

**Reference:** [Happy-Hare Wiki - Conceptual MMU](https://github.com/moggieuk/Happy-Hare/wiki/Conceptual-MMU)

### 2. Klipper Integration Points

The OpenCentauri firmware is based on Klipper. Integration requires:

#### A. Klipper MMU Module Configuration

```ini
[mcu mmu]
serial: /dev/ttyACM0  # USB CDC device from MMU

[extruder]
# Standard extruder configuration

[mmu]
# Happy-Hare specific configuration
```

#### B. SDCP Protocol Considerations

The Centauri uses the **Smart Device Control Protocol (SDCP)** v3.0.0 over WebSocket for motherboard communication. Multi-material operations may require:

- Extension of SDCP to include MMU status reporting
- New command codes for filament selection (Cmd: 132 "Stop Material Feeding" already exists)
- Status fields for MMU state and filament channel selection

**Reference:** [`docs/software/api.md`](docs/software/api.md:1-1096)

### 3. Firmware Integration Points

#### A. Mainboard Firmware (AllWinner R528-S3)

The CC1 mainboard runs a proprietary firmware based on AllWinner R528-S3 SoC. Integration requires:

1. **USB Device Mode Support:** Mainboard must enumerate as USB CDC device to MMU
2. **Serial Bridge:** USB-to-serial bridge for communication with MMU MCU
3. **Power Control:** GPIO control for 5V reset line (Pin 3 of Multicolor connector)

#### B. Klipper Configuration Files

OpenCentauri documentation references Klipper configuration:

**Reference:** [`docs/klipper-via-mainboard-replacement/mainsail-config.md`](docs/klipper-via-mainboard-replacement/mainsail-config.md:1-319)

Required additions:
- Happy-Hare module configuration
- Filament sensor integration
- MMU-specific G-code macros

## Implementation Plan

### Phase 1: Protocol Analysis and Testing

1. **Hardware Setup:**
   - Connect existing Happy-Hare MMU to CC1 "Multicolor" connector
   - Verify power delivery (24V, 5V)
   - Test USB communication

2. **Firmware Analysis:**
   - Analyze existing mainboard firmware for USB CDC support
   - Identify GPIO pins for 5V reset control
   - Document serial communication protocol

3. **Happy-Hare Compatibility Testing:**
   - Test Happy-Hare firmware with CC1 hardware
   - Identify protocol mismatches or missing features

### Phase 2: Klipper Integration

1. **Happy-Hare Module Integration:**
   - Add Happy-Hare configuration to Klipper setup
   - Configure MMU MCU communication parameters
   - Test filament loading/unloading cycles

2. **SDCP Extension (if needed):**
   - Define new command codes for MMU control
   - Add MMU status fields to status messages
   - Update API documentation

### Phase 3: Documentation and Testing

1. **User Documentation:**
   - Hardware installation guide
   - Klipper configuration examples
   - Troubleshooting guide

2. **Testing:**
   - Multi-material print tests
   - Filament change reliability testing
   - Long-term stability testing

## Known Challenges

### 1. Proprietary Firmware Limitations

The CC1 mainboard runs proprietary firmware. OpenCentauri provides "klipper-via-mainboard-replacement" which may not include MMU support:

**Reference:** [`docs/klipper-via-mainboard-replacement/index.md`](docs/klipper-via-mainboard-replacement/index.md:1-1005)

**Potential Solutions:**
- Use external Raspberry Pi with Klipper running full MMU support
- Modify mainboard firmware to expose USB CDC interface
- Implement serial bridge in mainboard firmware

### 2. USB Communication Protocol

The exact USB communication protocol between mainboard and MMU is undocumented. Happy-Hare expects specific CDC-ACM behavior that may not match the existing implementation.

**Potential Solutions:**
- Reverse engineer existing USB communication
- Use Happy-Hare's reference implementation as baseline
- Implement USB CDC-ACM driver in mainboard firmware

### 3. Power Management

The 5V reset line (Pin 3) requires GPIO control for MMU MCU reset. This GPIO may not be exposed in current firmware.

**Potential Solutions:**
- Identify unused GPIO pins on mainboard
- Add GPIO control to firmware
- Use external transistor circuit for reset control

## Comparison with Existing Solutions

### Flashforge IFS

Flashforge's IFS system uses a similar approach with:
- Hall effect filament sensors
- Independent stepper motors per channel
- USB communication to mainboard

**Relevance:** CANVAS module appears to use similar technology, suggesting Happy-Hare compatibility is feasible.

### Prusa MMU

Prusa's MMU uses:
- Serial communication (UART)
- Filament sensors per channel
- Integrated multiplexer

**Relevance:** Happy-Hare is designed to be compatible with Prusa MMU firmware, providing additional validation.

## Request for Community Input

1. **Do any community members have experience with Happy-Hare on CC1 hardware?**
2. **Is there existing work on CC1 MMU integration that could be referenced?**
3. **Are there developers familiar with AllWinner R528-S3 firmware who could assist with mainboard integration?**
4. **What is the preferred integration approach:**
   - Full Klipper on external Raspberry Pi (bypassing mainboard firmware)
   - Mainboard firmware modification
   - Hybrid approach

## References

- [Happy-Hare GitHub](https://github.com/moggieuk/Happy-Hare)
- [Happy-Hare Wiki - Conceptual MMU](https://github.com/moggieuk/Happy-Hare/wiki/Conceptual-MMU)
- [Happy-Hare Wiki - Type B Design](https://github.com/moggieuk/Happy-Hare/wiki/Conceptual-MMU#type-b-design)
- [OpenCentauri - CC1 Mainboard Documentation](docs/hardware/CC1/mainboard.md)
- [OpenCentauri - CC2 CANVAS Documentation](docs/hardware/CC2/CANVAS.md)
- [OpenCentauri - SDCP API Documentation](docs/software/api.md)
- [OpenCentauri - Klipper via Mainboard Replacement](docs/klipper-via-mainboard-replacement/index.md)

## Labels

- `enhancement`
- `multi-material`
- `happy-hare`
- `mmu`
- `cc1`
- `firmware`
