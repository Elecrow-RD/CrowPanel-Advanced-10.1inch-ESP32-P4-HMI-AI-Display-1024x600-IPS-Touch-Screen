# ESP32-P4 Display 10.1 inch V1.2 Product Hardware Driver Guide

| Item | Details |
|---|---|
| Document Version | V1.0 |
| Applicable Hardware | ESP32-P4 Display 10.1 inch V1.2 |
| Document Date | 2026-07-30 |
| Author | OpenAI Codex (compiled from the project schematics and validation code) |
| Schematic Baseline | `1.2/ESP32-P4 Display 10.1 inch V1.2.sch` and the PDF of the same name |
| Driver Baseline | `idf-code/Lesson01` through `Lesson17` |
| Software Platform | ESP-IDF, primarily targeting ESP32-P4; FreeRTOS |

## 1. Document Purpose and Evaluation Criteria

This document is intended for hardware maintenance, ESP-IDF driver porting, and onboarding handoffs. Cross-validation follows this order of precedence:

1. Constants and initialization code that are actually compiled in the validated BSP;
2. Default Kconfig values and component dependencies in the code;
3. Eagle `.sch` components and netlist;
4. Visual/textual annotations in the PDF schematic;
5. Comments and README files, used only as supplementary information.

When the code and schematic differ, this document records the code values as the “currently effective configuration” while retaining the schematic values in the discrepancy table. The code repository does not include a complete mass-production firmware project, the actual `sdkconfig`, test logs, or module firmware. Therefore, “validated” means that the repository provides a standalone Lesson example for the feature; it does not mean that all peripherals can operate simultaneously.

Status definitions:

- **Code-validated**: The repository contains complete initialization and usage examples.
- **Hardware-confirmed**: The hardware is present in the schematic, but the repository provides no direct driver, or it is managed internally by the chip/module firmware.
- **Optional external device**: Connected through an expansion connector; not an onboard component.
- **NC**: Explicitly marked as not populated in the schematic; must not be treated as a standard installed component.

## 2. System Architecture and Peripheral Overview

| Category | Component/Function | Schematic Designator | Interface | Current Status | Primary Code |
|---|---|---:|---|---|---|
| Main Controller | ESP32-P4NRW32 | U7 | Internal peripheral matrix | Code-validated | All Lessons |
| Boot Storage | W25Q128JVSIQ 128 Mbit NOR Flash | IC11 | Dedicated QSPI | Hardware-confirmed; managed by ROM/IDF | No standalone BSP |
| Wireless Coprocessor | ESP32-C6-MINI-1-N4 | IC1 | 6-line SDIO + EN | Code-validated | Lesson16, 17 |
| Display | 10.1-inch 1024×600, EK79007 timing controller | J21 | 2-lane MIPI-DSI/DPI | Code-validated | Lesson07, 09, 13-16 |
| Touch | GT911 (on the touch FPC/display module) | FPC2 | I2C + RST + INT | Code-validated | Lesson05, 06, 09, 16 |
| Backlight | MT9201 boost constant-current driver | U6 | GPIO31 PWM/EN, GPIO29 power gating | Partially code-validated | `bsp_illuminate` |
| Camera | SC2336 module | FPC1 | 2-lane MIPI-CSI + SCCB | Code-validated | Lesson13 |
| Memory Card | microSD | J5 | SDMMC 1-bit | Code-validated | Lesson08, 12 |
| Audio Output | I2S digital audio + ES8311/NS4168/NS4263B path | IC6/U3/U4 | I2S + I2C + GPIO | Partially code-validated | Lesson11, 12 |
| Audio Input | Onboard PDM microphone; additional ES7210 multichannel ADC | -/IC5 | PDM; I2S+I2C | PDM code-validated; ES7210 hardware-confirmed only | Lesson11 |
| USB Device | USB 2.0 OTG Type-C | J16 | ESP32-P4 USB D+/D- | Code-validated | Lesson06 |
| USB-to-Serial | CH340K + Type-C | U1/J1 | USB-UART0 | Hardware-confirmed; OS driver | Programming/logging port |
| UART Expansion | UART1, UART3/5 V input | J2/J10 | 3.3 V UART, level-shifted 5 V UART | UART1/2 code-validated | Lesson04, 14, 15 |
| I2C Expansion | HY2.0 4P, 7P Header | J13/J11 | 3.3 V I2C | Code-validated | Lesson05, 09, 10 |
| Environmental Sensor | DHT20 (external) | J13 | I2C | Code-validated | Lesson10 |
| LoRa | SX1262 (external) | J9/J11 | SPI + BUSY/IRQ/RST/NSS | Code-validated; Kconfig pin discrepancies | Lesson14 |
| 2.4 GHz | nRF24L01 (external) | J9/J11 | SPI + IRQ/CE/CS | Code-validated; Kconfig pin discrepancies | Lesson15 |
| GPIO Expansion | 2×12 Header | J7 | GPIO2-5, 25, 49-54, 3.3/5 V | Hardware-confirmed | User application |
| Buttons | P4 RESET, BOOT | K4/K3 | CHIP_PU, boot configuration | Hardware-confirmed | ROM boot logic |
| Battery Charging | TP4059 + battery connector | U2/J3 | Analog power, CHG/STD | Hardware-confirmed | Monitored by STC8 auxiliary controller |
| Power Auxiliary Controller | STC8H1K08-36I | U5 | ADC/GPIO/SPI/UART | Hardware-confirmed; no source code | None |
| Power Tree | RY3420, MT3406, MT3608, ME6211 series | U9/U10/U16/IC2/3/4/7/8 | DC-DC/LDO | Hardware-confirmed | ESP32-P4 internal LDOs are invoked |

## 3. Currently Effective ESP32-P4 Pin Assignment Table

| GPIO/Dedicated Pin | Current Function | Electrical/Multiplexing Notes | Code Basis |
|---:|---|---|---|
| 6/7/8 | External wireless SPI MOSI/MISO/SCLK | SPI3_HOST, 8 MHz; J9 through series resistors/filtering | Lesson14/15 `bsp_wireless.h` |
| 9/10 | SX1262 BUSY/NSS or nRF24 IRQ | Mutually exclusive between modules | Lesson14/15 |
| 12/13 | Camera SCCB SDA/SCL | I2C1, internal pull-ups, 100 kHz | Lesson13 |
| 14/15/16/17/18/19 | ESP32-C6 SDIO D3/D2/D1/D0/CLK/CMD | Dedicated onboard 6-line bus | Lesson16/17 + schematic |
| 20 | Microphone L/R selection | Schematic net `MIC_L/R`; not configured by the current PDM BSP | Schematic |
| 21/22/23 | I2S LRCLK/BCLK/DOUT | 16 kHz, 16 bit, stereo | Lesson11/12 |
| 24/26 | PDM CLK/DIN | 16 kHz mono, left channel | Lesson11 |
| 25 | GPIO expansion | J7-17 | Schematic |
| 27/28 | UART2 TX/RX; J9/J11 wireless control pins, used by the BSP as IRQ/RST or CE/CS | **The schematic intentionally multiplexes these signals; software must ensure mutual exclusion** | Schematic + Lesson14/15 |
| 29 | LCD backlight power MOS gate | Present in the schematic; not controlled by the current display BSP | Schematic |
| 30 | Audio amplifier control | Push-pull output; active-low in code | Lesson11/12 |
| 31 | LCD backlight PWM/EN | LEDC 30 kHz, 11 bit | Lesson07+ |
| 32 | ESP32-C6 EN | Main controller can reset/power down the C6; examples do not operate it directly | Schematic |
| 33/34 | UART3 RX/TX (5 V input expansion) | Level-shifted to J10 | Defined in Lesson04 but this pin group is not selected |
| 37/38 | UART0 TX/RX | CH340K programming/logging | Schematic |
| 39/43/44 | microSD D0/CLK/CMD | SDMMC slot 0, 1-bit, 10 MHz | Lesson08/12 |
| 40/42 | GT911 RST/INT | Configured active-low; current driver primarily uses polling | Lesson05/09 |
| 41 | LCD RESET | Connected in the schematic; EK79007 BSP uses `reset_gpio_num=-1` | Discrepancy; see Section 15 |
| 45/46 | System I2C SDA/SCL | I2C0, open-drain, internal pull-ups, 400 kHz | Lesson05/09/10/16 |
| 47/48 | UART1 TX/RX | J2 3.3 V; GPIO48 is also used by the LED example | Lesson02/04 + schematic |
| 49-54 | GPIO expansion | J7; 53/54 are the wireless Kconfig-recommended control pins | Schematic/Kconfig |
| USB DP/DM | USB2 D+/D- | Dedicated PHY to J16 | Lesson06 |
| Dedicated DSI/CSI Pins | MIPI D-PHY | Two data lanes plus one clock lane each | Lesson07/13 |

## 4. Main Controller, Clocks, and Boot Storage

### 4.1 ESP32-P4NRW32 (U7)

- All main-controller examples are based on ESP-IDF/FreeRTOS; the code uses the IDF driver layer rather than directly accessing chip registers.
- The external main crystal Y2 is 40 MHz; Y4 is 32.768 kHz.
- K4 pulls `CHIP_PU/P4_RST_EN` low to reset the device; K3 controls `SPI_BOOT`. The schematic labels high as SPI Boot and low as Download Boot.
- High-bandwidth display/camera buffers depend on SPIRAM. LVGL uses double buffering in SPIRAM, while the camera uses cache-aligned SPIRAM USERPTR buffers.
- The display/wireless examples acquire the ESP32-P4 internal LDO3 at 2500 mV and LDO4 at 3300 mV before initializing peripherals. This sequence must not be omitted during porting.

```c
esp_ldo_channel_config_t ldo3 = { .chan_id = 3, .voltage_mv = 2500 };
esp_ldo_channel_config_t ldo4 = { .chan_id = 4, .voltage_mv = 3300 };
ESP_ERROR_CHECK(esp_ldo_acquire_channel(&ldo3, &ldo3_handle));
ESP_ERROR_CHECK(esp_ldo_acquire_channel(&ldo4, &ldo4_handle));
```

### 4.2 W25Q128JVSIQ (IC11)

- Dedicated Flash bus: `FLASH_CS/CK/WP/HOLD/D/Q`; it does not use peripheral pins listed in the general-purpose GPIO table.
- The component model indicates a capacity of 128 Mbit (16 MiB). Actual usable partitions are determined by the project’s `partitions.csv` and flashing configuration.
- It is managed by the ROM bootloader and the ESP-IDF SPI flash layer. The application must not initialize this bus again.

## 5. Display, Touch, and Backlight

### 5.1 EK79007 MIPI-DSI Display

Currently effective parameters: 2 data lanes, 900 Mbps lane rate, 8-bit DBI commands and parameters; 1024×600, RGB565, 51 MHz DPI pixel clock; horizontal timing 160/70/160 and vertical timing 23/10/12 (back porch/pulse/front porch).

```c
esp_lcd_dsi_bus_config_t bus = {
    .bus_id = 0, .num_data_lanes = 2, .lane_bit_rate_mbps = 900,
};
esp_lcd_dpi_panel_config_t dpi = {
    .dpi_clock_freq_mhz = 51,
    .pixel_format = LCD_COLOR_PIXEL_FORMAT_RGB565,
    .video_timing = { .h_size=1024, .v_size=600,
        .hsync_back_porch=160, .hsync_pulse_width=70, .hsync_front_porch=160,
        .vsync_back_porch=23, .vsync_pulse_width=10, .vsync_front_porch=12 },
};
```

Dependencies: `esp_lcd_mipi_dsi`, `espressif/esp_lcd_ek79007 ^1.0.2`, `esp_lvgl_port ^2.6.0`, and LVGL 8.3.11. The LVGL task priority is `configMAX_PRIORITIES-4`, with a 16 KiB stack, a 5 ms timer, and a maximum sleep duration of 10 ms; full-frame double buffers reside in SPIRAM.

### 5.2 GT911 Touch

- I2C0: SDA=GPIO45, SCL=GPIO46; 7-bit address, 400 kHz; internal pull-ups enabled and a 7-cycle glitch filter.
- RST=GPIO40 and INT=GPIO42, both configured with active level 0; coordinates are 1024×600, with no XY swap or mirroring.
- The driver first attempts `ESP_LCD_TOUCH_IO_I2C_GT911_ADDRESS`, then tries `..._BACKUP` if the first attempt fails (common GT911 addresses are 0x5D/0x14; the final values are determined by the component header).
- The application polling path calls `esp_lcd_touch_read_data()`. Although INT is passed to the component, the examples do not register a user GPIO ISR.

Initialization sequence: acquire LDOs → `i2c_init()` → `touch_init()` → DSI/LVGL → register the LVGL touch input → turn on the backlight.

### 5.3 Backlight

- GPIO31 connects to MT9201 EN/backlight control, using LEDC low-speed timer0/channel0, the PLL divider clock, 30 kHz, and 11-bit resolution.
- `brightness=0` outputs duty 0; for nonzero values, duty=`brightness*18+200`. The caller must limit the value to 0-100, or the duty may exceed the 11-bit maximum of 2047.
- The schematic also includes GPIO29→Q11→`LCD_BK_VDD` power gating. The current BSP does not drive GPIO29 high; whether it is controlled by the power-on default or other firmware must be confirmed on physical hardware.
- The J21 panel also requires analog supplies such as 9.6 V AVDD, 18 V VGH, -6 V VGL, and 3.3 V VCOM. These are generated by U16/discrete circuitry and are not GPIO-driven.

## 6. SC2336 Camera

- FPC1 connector: 2-lane MIPI-CSI (D0/D1/CLK differential pairs), SCCB, XVCLK, RESET, and DOVDD 1.8 V/AVDD 2.8 V power supplies.
- SCCB uses I2C port 1, SDA=GPIO12 and SCL=GPIO13, with internal pull-ups at 100 kHz.
- The driver configures `reset_pin=-1` and `pwdn_pin=-1`, reuses the existing SCCB bus handle, and uses `ESP_VIDEO_MIPI_CSI_DEVICE_NAME` as the video node.
- RGB565 is enforced through V4L2 `VIDIOC_G_FMT/S_FMT`; the resolution is negotiated by the sensor driver and is not hard-coded in the BSP.
- At least two capture buffers are used with `V4L2_MEMORY_USERPTR`, SPIRAM, and cache alignment; H/V flip can be controlled through Kconfig.
- ISP parameter file `sc2336_custom.json`: 50 Hz anti-flicker, including AWB/AGC, noise reduction, gamma, sharpening, and a color matrix.
- Dependencies: ESP-IDF >=5.4.0, `esp_cam_sensor ^1.2.0`, `esp_sccb_intf ^0.0.5`, and `esp_video ^1.1.0`.

Note: The schematic includes a `CSI_RESET` control net, but the code explicitly uses -1. Retain the code configuration during migration unless the GPIO reset path is revalidated.

## 7. microSD

- GPIO43=CLK, GPIO44=CMD, and GPIO39=D0; SDMMC_HOST slot 0, 1-bit, 10 MHz, with internal pull-ups.
- J5 D1, D2, and CS are not used in the current 1-bit mode; the corresponding pads/pull-ups are also visible in the schematic.
- The FATFS mount point is `/sdcard`; automatic formatting is disabled, up to five files may be open, and the allocation unit is 16 KiB.

```c
sdmmc_host_t host = SDMMC_HOST_DEFAULT();
host.slot = SDMMC_HOST_SLOT_0;
host.max_freq_khz = 10000;
sdmmc_slot_config_t slot = SDMMC_SLOT_CONFIG_DEFAULT();
slot.clk = GPIO_NUM_43; slot.cmd = GPIO_NUM_44; slot.d0 = GPIO_NUM_39;
slot.width = 1; slot.flags |= SDMMC_SLOT_FLAG_INTERNAL_PULLUP;
```

The current BSP does not use card-detect CD for hot-plug detection. FATFS should be unmounted before inserting or removing the card to avoid file-system corruption.

## 8. Audio Input and Output

### 8.1 Currently Validated Playback Path

- GPIO21=LRCLK/WS, GPIO22=BCLK, and GPIO23=DOUT; I2S standard TX, 16 kHz, 16 bit, stereo, with an MCLK multiple of 256, but `mclk=I2S_GPIO_UNUSED`.
- GPIO30 controls the audio amplifier and is configured as a push-pull output with no pull-up or pull-down. In the code, `set_Audio_ctrl(true)` actually outputs a low level, so the control is active-low.
- The Lesson12 SD audio playback example uses the same GPIOs; the input file format/sample rate must match the actual I2S configuration.

### 8.2 Currently Validated Recording Path

- PDM RX: GPIO24=CLK and GPIO26=DIN; 16 kHz, 16 bit, mono, left slot.
- 8× downsampling, BCLK divider 8, 35.5 Hz high-pass, and amplify=1.
- The recording buffer is allocated at 32000 bytes/s. The example limits recordings to 60 seconds and, after recording, converts and plays them back through the playback path.

### 8.3 Audio Components Present in the Schematic but Not Fully Driven by the Examples

- ES7210 (IC5): multichannel audio ADC with I2C control and I2S/TDM output; the repository does not contain register initialization.
- ES8311 (IC6): codec with I2C control lines on the 3.3 V side, connected to I2S MCLK/SCLK/LRCK/SDOUT; the current playback BSP does not initialize its I2C registers.
- NS4168 (U3/U13) is marked NC in the schematic and must not be treated as a standard installed amplifier.
- NS4263B (U4) is the onboard analog amplifier path; the enable/disable and mute polarity should still be confirmed using waveforms from physical hardware.

## 9. USB and Serial Ports

### 9.1 USB2 Device Port J16

- Dedicated USB DP/DM connects directly to the ESP32-P4; both Type-C CC pins have 5.1 kΩ pull-down resistors, configuring the port as a device.
- Lesson06 uses a TinyUSB HID mouse: one HID interface, IN endpoint 0x81, 16-byte packets, and a 10 ms interval; it declares 100 mA and remote wakeup.
- Dependency: `espressif/esp_tinyusb ^1.1`. Touch is polled every 10 ms and converted into relative mouse movement.

### 9.2 USB-UART Programming Port J1/U1

- The CH340K connects to USB1 D+/D-, with UART0 connected to ESP32-P4 GPIO37 (TX)/GPIO38 (RX); DTR/RTS participate in automatic programming/reset.
- The application does not initialize the CH340K; the host PC uses the WCH CH340 driver. This port is used for both programming and logging, so the application should not remap UART0.

### 9.3 UART Expansion

- J2: 3.3 V UART1, GPIO47 TX and GPIO48 RX.
- J10: external 5 V UART input interface, connected through level shifting to GPIO34 TX and GPIO33 RX.
- Lesson04 actually initializes `UART_NUM_2` and maps it to GPIO47/48: 115200, 8N1, no flow control, and a 2048-byte RX buffer.
- Lesson14/15 optionally use `UART_NUM_1` at 115200 8N1 for transparent UART communication. GPIO27/28 in the header match the J9/J11 schematic connections, but the Kconfig defaults specify GPIO53/54 instead.

## 10. Onboard ESP32-C6 Wi-Fi

- The ESP32-P4 itself has no Wi-Fi; IC1 ESP32-C6-MINI-1-N4 serves as the wireless coprocessor.
- SDIO: P4 GPIO14/15/16/17/18/19 → C6 IO23/22/21/20/19/18 (D3/D2/D1/D0/CLK/CMD). GPIO32 controls C6 EN.
- The C6 has its own UART0 and BOOT test/programming network; do not confuse it with the P4 application UART.
- P4 software depends on `espressif/esp_hosted ~2.7.0` and `espressif/esp_wifi_remote ^0.16.1`; the application still calls the standard `esp_wifi_*` API.
- Lesson17 covers STA, SoftAP, and AP+STA; the channel, maximum number of connections, and authentication threshold are controlled by Kconfig. The Lesson16 weather application uses STA.

Porting must ensure all of the following: the C6 is flashed with slave firmware compatible with the esp-hosted version; the P4 SDIO pin/width configuration matches the board-level connections; and the GPIO32 EN timing is correct. The repository does not include the C6 slave firmware or the complete system `sdkconfig`; these are external dependencies for reproducing Wi-Fi operation.

## 11. I2C Environmental Sensor and Expansion Ports

System I2C0 (GPIO45/46) connects to J13/J11 and the onboard codec control interface through BSS138 level shifting/pull-ups. The code uniformly configures bus devices for 7-bit addressing at 400 kHz.The DHT20 is an external device at address 0x38:

- After initialization, check the calibration status bits: `status & 0x18 == 0x18`;
- Send the measurement command `AC 33 00`, wait at least 80 ms, and then poll the busy bit;
- Read 7 bytes. CRC-8 uses an initial value of 0xFF and polynomial 0x31;
- Humidity and temperature are both 20-bit values and must be converted to `%RH` and `°C`.


The touchscreen, DHT20, ES8311, and external devices share bandwidth on the same I2C0 bus. Address conflicts and the total pull-up resistance must be checked before connecting a new module.

## 12. External Wireless Modules

### 12.1 Shared SPI Bus

SPI3_HOST, GPIO8 SCLK, GPIO7 MISO, GPIO6 MOSI, mode 0, 8 MHz, DMA auto; CS is controlled in software by RadioLib through a GPIO, with `spics_io_num=-1`. J9 provides SPI and 3.3/5 V, while the control pins come from additional expansion GPIOs.

### 12.2 SX1262 (Lesson14)

Current valid header-file values: BUSY=GPIO9, IRQ/DIO1=GPIO27, NRST=GPIO28, and NSS=GPIO10; these values match the J9/J11 connections. LoRa parameters: 915 MHz, BW 125 kHz, SF7, CR 7, private sync word, 22 dBm, preamble 8, and TCXO voltage parameter 1.6. TX/RX completion uses GPIO ISR callbacks. RX uses continuous receive with boosted gain.

### 12.3 nRF24L01 (Lesson15)

Current valid header-file values: IRQ=GPIO9, CE=GPIO27, and CS=GPIO28; these values match the J9/J11 connections. Parameters: 2400 MHz, 250 kbps, address width 5, pipe address `01 02 11 12 FF`, and SPI at 8 MHz.

The SX1262 and nRF24 share the SPI bus and control pins, so only one may be selected at compile time. Product compliance must confirm whether 915 MHz complies with regulations in the deployment region.

## 13. Power, Charging, and Auxiliary Control

| Power Domain/Device | Function | Software Relationship |
|---|---|---|
| TP4059 U2 | Single-cell lithium battery charging from 5 V input, CHG/STD status | Monitored by STC8; no direct P4 control |
| STC8H1K08 U5 | Charging LEDs, VBAT ADC, power control | No STC8 firmware in the repository; maintenance gap |
| U9 RY3420 | 5 V→3.3 V DC-DC | Always on/hardware-enabled |
| U10 MT3406 | 5 V→approximately 1.2 V DC-DC | P4 DCDC power domain |
| U16 MT3608 | Front-end power supply for LCD high voltage | Hardware-controlled |
| IC2 ME6211 1.8 V | Camera DOVDD | Hardware-enabled |
| IC3 ME6211 2.8 V | Camera AVDD | Hardware-enabled |
| IC4 ME6211 1.8 V NC | Alternative LCD VDD | NC, not required |
| IC7/IC8 ME6211 3.3 V | Audio/auxiliary control power | Hardware-enabled |
| MT9201 U6 | Constant-current boost converter for LCD LEDs | GPIO31 EN/PWM |
| GPIO29/Q11 | Backlight input power gating | Currently omitted from the BSP; must be verified on actual hardware |

Type-C J1, J16, and the external 5 V input provide multiple power entry points, and the schematic includes diode/MOSFET isolation. During maintenance, never bypass the isolation and connect different power supplies in parallel. All expansion GPIOs use 3.3 V logic. Only J10, which is explicitly marked as having level conversion, can accept 5 V UART signals.

## 14. Expansion Connectors

| Connector | Key Signals | Voltage/Purpose |
|---|---|---|
| J7 2×12 | GPIO49, 2, 3, 4, 5, 50, 51, 52, 25, 53, 54 | Includes 3.3 V, 5 V, and GND; GPIOs are not 5 V tolerant |
| J9 7P | SPI SCLK/MISO/MOSI, GPIO27, 3.3 V, 5 V, GND | External SX1262/nRF24 |
| J11 7P | I2C SDA/SCL, GPIO28/9/10 | Wireless/I2C expansion |
| J13 HY2.0 | 3.3 V, SDA, SCL, GND | DHT20 and other I2C sensors |
| J2 HY2.0 | RXD1, TXD1, 3.3 V, GND | 3.3 V UART |
| J10 XH2.54 | UART_IN TX/RX, 5 V, GND | External UART with level conversion |
| J8 1×4 | STC8 power/programming-related signals | For auxiliary-control maintenance only |

## 15. Schematic and Code Differences/Conflicts

| ID | Item | Schematic/Kconfig | Current Working Code | Assessment and Handling |
|---|---|---|---|---|
| D01 | Wireless control pin configuration | Both schematic J9/J11 and the header file use GPIO27/28; Kconfig defaults to GPIO53/54 | GPIO27/28 hardcoded in the header file | Record 27/28 based on the effective code and schematic; the Kconfig values are currently ineffective. If changed to 53/54, the external wiring must also be changed and revalidated |
| D02 | Wireless configuration source | Kconfig defines `CONFIG_*GPIO*` | The header file does not reference Kconfig and instead defines macros directly | Changes in menuconfig do not take effect; migration requires standardizing on `CONFIG_...` |
| D03 | GPIO48 LED | The schematic shows GPIO48 as `RXD1` routed to J2; no independent onboard LED net was found | Lesson02 directly configures GPIO48 as push-pull output and describes high level as “LED on” | Record as operational according to the example, but it drives the UART RX net; do not use it simultaneously with UART1. Verify the silkscreen/board revision on actual hardware |
| D04 | LCD reset | J21 `LCD_RESET` connects to P4 GPIO41 | EK79007 uses `reset_gpio_num=-1` | Do not use GPIO reset according to the code; verify GPIO41 timing only if cold-start issues occur |
| D05 | Backlight power | GPIO29 controls Q11 `LCD_BK_VDD` | The BSP controls only GPIO31 PWM | May depend on a default pull-up or other firmware; GPIO29 must be tested and incorporated into production initialization |
| D06 | Camera reset | FPC1 has a `CSI_RESET` hardware net | `reset_pin=-1`, `pwdn_pin=-1` | Follow the code; do not independently drive shared pins such as GPIO46 based solely on the schematic |
| D07 | SD data width | J5 routes D0/D1/D2/CS | D0 only, 1-bit SDMMC | Not a conflict; follow the code for a more reliable 10 MHz 1-bit mode |
| D08 | Audio codec | The schematic includes ES7210/ES8311 | The examples use I2S/PDM directly without writing codec registers | Only demonstrates a simplified audio path; full codec functionality requires new driver validation |
| D09 | UART naming | Schematic signal names are UART1/UART3/UART2 | Lesson04 maps `UART_NUM_2` to GPIO47/48 | IDF UART controllers can be routed through the GPIO matrix; use GPIOs rather than variable names when determining connections during maintenance |
| D10 | GT911 address | The schematic does not specify the chip address | The component automatically tries the primary and alternate addresses | Follow the code; record the final address in the logs for maintenance |
| D11 | NS4168 | U3/U13 are marked NC | Audio descriptions may refer generically to an amplifier | Follow the BOM/actual hardware; do not create hard dependencies on NC devices |

The causes of these differences can be conclusively determined only after obtaining the hardware ECN, BOM, and test records. Current possibilities include unsynchronized board revisions, remnants from example migrations, or schematic labeling errors; this document does not treat possible causes as established facts.

## 16. Risks and Precautions

1. **GPIO27/28 resource conflict (High)**: The schematic multiplexes UART2 with the J9/J11 wireless control pins; they cannot be enabled simultaneously. If J7 GPIO53/54 are used instead, both the external wiring and BSP must be updated.
2. **GPIO48 conflict (High)**: The LED example shares a pin with UART1 RX; output mode may cause electrical contention with an external transmitter.
3. **5 V tolerance (High)**: ESP32-P4 GPIOs use 3.3 V logic. The presence of 5 V on J7/J9 does not mean the signal pins are 5 V tolerant.
4. **Backlight power sequencing (High)**: Complete DSI initialization and render the first frame before enabling GPIO29 power and GPIO31 PWM; use the reverse order during shutdown.
5. **Wi-Fi firmware coupling (High)**: The P4 `esp_hosted/esp_wifi_remote` version must match the C6 slave firmware.
6. **Power entry points (High)**: When USB, battery, and external 5 V supplies coexist, confirm that the isolation components are installed; never short the supplies together directly.
7. **MIPI signal integrity (High)**: DSI/CSI are controlled-impedance differential signals. Do not use jumper wires, long leads on ordinary probes, or arbitrary FPC substitutions.
8. **Memory pressure (Medium)**: A single 1024×600 RGB565 frame is approximately 1.17 MiB. Display double buffering plus camera double buffering requires sufficient SPIRAM.
9. **I2C pull-ups (Medium)**: Multiple onboard and external pull-ups connected in parallel may be too strong. At 400 kHz, verify the waveform and low-level sink current.
10. **SD hot-plugging (Medium)**: Card-detect handling is not currently implemented. Removing the card during writes may corrupt FATFS.
11. **Audio popping (Medium)**: GPIO30 is active low. Keep the amplifier disabled until the data is stable, and disable it only after the stream has stopped.
12. **LoRa regulations (Medium)**: 915 MHz and 22 dBm are not permitted in all regions. Production parameters must be adjusted for the target market.
13. **Security configuration (Medium)**: The Lessons contain example Wi-Fi SSIDs/passwords; they must not be included in production firmware.
14. **Missing auxiliary-control source code (Medium)**: The STC8 firmware is not in the repository, so the charging/power state machine cannot be fully audited or migrated.

## 17. Recommended System Initialization/Shutdown Sequence

Power-on initialization:

1. Confirm that the external 5 V/battery supply is stable, and keep the backlight and amplifier disabled;
2. Configure ESP32-P4 LDO3=2.5 V and LDO4=3.3 V;
3. Initialize the shared I2C0 bus, followed by the GT911 and other I2C devices;
4. Initialize MIPI-DSI, EK79007, and LVGL, and submit the first frame;
5. Confirm the GPIO29 backlight power-gating state, and then gradually increase GPIO31 from a low duty cycle;
6. Initialize the SD card, audio, and camera as needed; initialize camera SCCB before CSI/V4L2;
7. Start the ESP32-C6 and wait for hosted transport, then start `esp_wifi`;
8. Finally, enable the external wireless IRQ or UART after completing GPIO conflict checks.

Reverse the sequence for shutdown: stop application tasks/IRQs → stop Wi-Fi/wireless → stop the camera/release frame buffers → unmount the SD card → stop I2S and disable the amplifier → set backlight duty=0/disable power → release the display and LDOs.

## 18. Driver Migration Checklist

- [ ] Lock the hardware version to V1.2, along with the BOM and ECN; confirm all NC/DNP components.
- [ ] Ensure the target ESP-IDF meets the requirements of all components; standardizing on >=5.4.2 is recommended.
- [ ] Before copying the pin table, define the mutual-exclusion strategy for D01 and resolve D03 and D05.
- [ ] Enable PSRAM and verify DMA/cache alignment capabilities.
- [ ] First run individual tests for solid-color display output, touchscreen coordinates, SD read/write, audio loopback, and camera frame capture.
- [ ] Use a logic analyzer to verify I2C at 400/100 kHz, SPI at 8 MHz, SD at 10 MHz, and UART at 115200.
- [ ] Use an oscilloscope to verify GPIO30/31/29 polarity and power-on waveforms.
- [ ] Flash the matching version of the ESP32-C6 hosted firmware and test reconnection after disconnection.
- [ ] Then perform full-system concurrent stress testing: display + touchscreen + SD + audio + camera + Wi-Fi.
- [ ] Record the final `sdkconfig`, component lock file, C6 firmware hash, and test report.

## 19. Key Software Dependencies and Evidence Index

| Function | Software Layer/Component | Primary Evidence Files |
|---|---|---|
| GPIO/LEDC | ESP-IDF `driver/gpio.h`, `driver/ledc.h` | `Lesson02/.../bsp_extra.c`, `Lesson07/.../bsp_illuminate.c` |
| DSI/LVGL | ESP-IDF LCD + EK79007 + LVGL port | `Lesson07-Turn_on_the_screen/peripheral/bsp_illuminate/` |
| GT911/I2C | New IDF I2C master + esp_lcd_touch_gt911 | `Lesson05-Touchscreen/peripheral/` |
| SD | SDMMC + VFS FAT | `Lesson08-SD_Card_File_Reading/peripheral/bsp_sd/` |
| I2S/PDM | IDF I2S standard/PDM drivers | `Lesson11-Playback_After_Recording/peripheral/` |
| USB | TinyUSB/esp_tinyusb | `Lesson06-USB2.0/peripheral/bsp_usb/` |
| Camera | esp_video/V4L2 + esp_cam_sensor | `Lesson13-Camera_Real-Time/peripheral/bsp_camera/` |
| Wi-Fi | esp_hosted + esp_wifi_remote | `Lesson16.../idf_component.yml`, `Lesson17-Wi-Fi_function/` |
| SX1262/nRF24 | RadioLib 7.x + custom EspHal | `Lesson14.../bsp_wireless/`, `Lesson15.../bsp_wireless/` |
| Schematic Nets | Eagle 9.6.2 XML + PDF | `1.2/ESP32-P4 Display 10.1 inch V1.2.sch/.pdf` |

## 20. Maintenance Conclusions

The board’s verified primary path is ESP32-P4 + MIPI-DSI 1024×600 + GT911 + SDMMC + PDM/I2S + MIPI-CSI + ESP32-C6 hosted Wi-Fi. The schematic and code are generally consistent for the display, touchscreen, SD, camera data interface, primary audio GPIOs, and wireless J9/J11 connections. The main maintenance risks are GPIO27/28 function multiplexing and the Kconfig mismatch, GPIO48 LED/UART multiplexing, omission of backlight GPIO29 from the BSP, and missing source code for full codec/power auxiliary control. Before conducting concurrent multi-peripheral validation, any complete system integration must first resolve these discrepancies.