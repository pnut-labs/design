> **Status:** notes, 2026-09-28. Copied from the development workspace
> (`NEXT-DEVICE.md`). Choices marked *Decided* were made by the maintainer;
> the rest are proposals. Hardware decisions will move into RFCs.

# Next device: hardware choices

Notes from choosing parts for a successor to the T-Deck Max (2026-09-27/28): what was decided, what is still open, and why. Nothing is ordered or designed yet; the T-Deck stays the development platform, and pnut-os carries over.

## Why a new device

What the T-Deck Max taught us (README, BACKLOG):

- **Kernel memory:** 151 KB of internal RAM for the kernel heap, 44 KB free normally, about 16 KB with Wi-Fi and mobile data up at once. PSRAM (8 MB) is plentiful: 0.9 MB used with everything running.
- **Standby with the modem on:** 48 mA, about 1.2 days on the ~1.4 Ah battery. The A7682E's own sleep gets 18–20 mA but misses commands, so it is off.
- **Mobile data:** 70–80 kbit/s, capped by the modem's 115200-baud UART (CMUX plus PPP).
- **Calls** fall back to GSM (no VoLTE), where the modem switches itself off.
- **The protected build** (memory protection between the kernel and native programs) is parked.
- **E-paper:** a 3.1" 240 × 320 panel (GDEQ031T10, UC8253); partial refresh ~0.7 s, full 1.0 s.

Compute is fine: the CPU sits idle and apps start in 0.15 s.

## Decided

| Part | Choice | Why |
|---|---|---|
| **Shape** | **a solid bar**, BlackBerry Classic style: screen, a navigation row (Call · Menu · 5-way key or trackpad · Back · End), then a 4-row keyboard about 64–66 mm wide | no hinge or slider; typing comfort worth a little height; the Minimal Phone shows what's missing without navigation keys. W/S/A/D stay as shortcuts. |
| **Modem** | **Quectel EG915U-EU**; fallback **LARA-R6801** (u-blox, now Trasna) | EG915U-EU: LTE B1/3/5/7/8/20/28 plus GSM, VoLTE, analog mic and earpiece plus PCM, USB with ECM/RNDIS, 1.4–1.6 mA asleep on LTE (paging cycle 1.28 s), 23.6 × 19.9 mm. LARA-R6801: VoLTE and CSFB, 3G/2G fallback, I2S, eDRX and PSM (0.9–1.9 mA), but USB serial ports only (PPP over USB), 26 × 24 mm. |
| **GNSS** | **wait for u-blox F11** (MAX-F11N dual-band L1+L5 expected Q4 2026); MAX-M10S stands in on dev kits | F11: 7 mW in its low-power mode, 1.0 m CEP with SBAS. Plan the MAX footprint and an L1+L5 antenna. The driver reads NMEA, so the swap is software-free. |
| **LoRa** | **Semtech LR2021**; fallback SX1262 | sub-GHz and 2.4 GHz LoRa, FLRC up to 2.6 Mbps, +22 dBm, RX 5.5 mA, backward compatible on air (Meshtastic and MeshCore both have LR2021 boards). Needs a NuttX driver (RadioLib as the reference), and lorad made chip-neutral. |
| **Wi-Fi** | 5 GHz minimum | 2.4 GHz only is a no-go. |
| **Bluetooth** | must work properly with AirPods Pro 3 | Apple lists Bluetooth 5.3 and the H2 chip, no LE Audio, so Classic audio: A2DP with AAC (SBC first), HFP for calls, AVRCP. |

## Wi-Fi and Bluetooth (proposed, not confirmed)

- **Chip:** **Murata Type 1MW** (CYW43455: Wi-Fi 5, 2.4 and 5 GHz, Bluetooth 5.0, 7.9 × 7.3 mm) for prototypes, because NuttX's `bcm43xxx` driver already covers the 43455 (SDIO), and `bt_uart_bcm4343x` loads the Bluetooth patch.
- **Aim for Murata Type 2FY** (CYW55513: Wi-Fi 6/6E tri-band, Bluetooth 5.4, SDIO 3.0, UART, PCM, I2S, same 7.9 × 7.3 mm; pinout match unverified). It needs a driver: port Infineon's WHD (licence to check) or extend `bcmf`.
- **Bluetooth audio software** is the larger risk: NuttX's own stack and NimBLE are LE only.
  - **First:** evaluate **openvela's Bluetooth framework** (NuttX-based; certified 5.4; A2DP source, AVRCP, HFP AG, LE Audio).
  - **Fallback:** **BTstack** (dual mode; needs a commercial licence if sold).
  - **Codecs:** SBC first. AAC later: an encoder library, its CPU load to measure, and a patent licence if sold.
- **Call audio to AirPods:** modem PCM ↔ MCU SAI ↔ Bluetooth PCM (SCO or mSBC), so the MCU can switch between the earpiece and Bluetooth.

## Open: the camera changes the MCU and the display

**The first choice, the STM32H7,** was agreed before the camera came up:
- mature NuttX port: USB OTG host, SDMMC, QSPI, Stop and Standby, tickless, RTC, MPU;
- 1–1.4 MB SRAM;
- USB full speed (12 Mbit/s) with the internal PHY;
- MIPI DSI only on the H747/H757.

**Then the goal became real photos and videos, not a scanner.** That needs an image processor, hardware video and a colour screen that updates instantly:

| Option | What it has | NuttX today |
|---|---|---|
| **STM32N6** (MCU) | Cortex-M55 at 800 MHz, 4.2 MB SRAM, MIPI CSI-2 (2 lanes) with an ISP (5 MP at 30 fps), H.264 **encoder** (1080p30), JPEG codec, NPU, LTDC (parallel RGB; no DSI found), xSPI and FMC for external RAM. **No H.264 decoder:** playback would be in software or MJPEG. | minimal: GPIO, serial, timers on the Nucleo-N657X0-Q |
| **STM32MP25** (MPU) | 2 × Cortex-A35 at 1.5 GHz plus a Cortex-M33, DDR4/LPDDR4, a camera ISP (CSI-2 or parallel), H.264 encode **and** decode, a GPU, DSI and LVDS | no port; Linux-class. pnut-os's daemons are mostly POSIX and could run on Linux. |
| STM32H747 + OV5640 (DVP, JPEG from the sensor) | cheapest route | mature H7; mediocre photos (an older 1/4" sensor) |

NuttX runs on these Cortex-A chips: A64 (PinePhone, with MIPI DSI), i.MX 8, i.MX 9, RK3399, RK3588, BCM2711, AM62x, A527, Zynq MPSoC.

**The display follows the camera:**
- **With a camera:** a colour IPS or AMOLED screen, 3.5–4". MIPI DSI needs an H747/H757 or an MPU. The N6 would take parallel RGB, and a 480 × 800 16-bit frame (~750 KB) fits its RAM.
- **Without a camera, the e-paper findings stand:**
  - **GDEY0397T81P:** 3.97", 800 × 480, 235 ppi, 4 greys, SSD1677, partial 0.3 s on the datasheet. Touch and front light are to be asked of Good Display. Portrait = the design at 2×.
  - **Alternatives:**
    - GDEY037T03 (3.7", 416 × 240, our UC8253 driver, touch and front light offered);
    - Sharp LS032B7DD02 memory LCD (3.16", 336 × 536, instant, microwatts; NuttX `memlcd`);
    - ST7305/ST7306 RLCD (4.2", 300 × 400);
    - direct-driven raw E Ink panels (EPDiy-style; a high-voltage power chip, parallel data, our own waveforms). Custom waveforms loaded into the SSD1677 come first.
- **Body size, estimated with the 3.97" panel:** about 70 × 150–155 mm. For reference: BlackBerry Classic 131 × 72.4 × 10.2 mm, Minimal Phone 142 × 78 × 8.6 mm.

**Still to decide:**
1. The MCU class: N6 (keep NuttX and WAMR, big port) or an MPU (MP25; Linux or a NuttX port).
2. The screen: IPS or AMOLED, and its interface.
3. The camera sensor.

## Risks to test on dev kits before any board

1. **VoLTE on your SIMs** (Orange, T-Mobile, and a Play or Plus SIM) with EG915U-EU and LARA-R6 evaluation boards, from a PC. Operators allow VoLTE only for devices they recognise, and module makers ship operator settings only for certified networks.
2. **The modem's USB network interface under NuttX** (CDC-ECM host). Plan AT commands and the ring line on a UART, and power USB only while data is on: USB up doubles or triples idle draw, and NuttX's host likely can't suspend the bus.
3. **Wi-Fi over SDIO on the chosen MCU** (1MW), and **A2DP to AirPods** from NuttX (openvela's stack, SBC).
4. **The LR2021 driver** on air against a MeshCore node.
5. **The camera pipeline** on the chosen MCU or MPU, once decided.

Dev kits so far: the chosen MCU's board, Murata 1MW, EG915U-EU and LARA-R6 evaluation boards (SparkFun LTE Stick), MAX-M10S, an LR2021 module (NiceRF LoRa2021 or RAK13700), later the 2FY.

## Sources

- Modems:
  - [EG915U specification](https://developer.quectel.com/en/wp-content/uploads/sites/2/2024/11/Quectel_EG915U_Series_LTE_Standard_Module_Specification_V1.3.pdf)
  - [EG915U hardware design](https://www.sigmaelectronica.net/wp-content/uploads/2023/03/eg915u.pdf)
  - [LARA-R6 datasheet](https://docs.sparkfun.com/SparkFun_LTE_Stick_LARA_R6/assets/component_documentation/LARA-R6-Datasheet.pdf)
  - [u-blox cellular to Trasna](https://www.u-blox.com/en/ir-news/u-blox-to-divest-cellular-business-to-trasna)
  - [EG800Q specification](https://www.quectel.com/content/uploads/2024/03/Quectel_EG800Q_Series_LTE_Standard_Specification_V1.6.pdf?wpId=117034)
- Radios:
  - [Murata 1MW datasheet](https://www.farnell.com/datasheets/2853020.pdf)
  - [Murata 2FY datasheet](https://pim.murata.com/asset/pim4/wifiBluetoothModule/TYPE2FY_PDF_WIFIBLUETOOTHMODULE?lastModifiedDatetime=20251028121238)
  - [Infineon WHD](https://github.com/Infineon/wifi-host-driver)
  - [openvela Bluetooth](https://github.com/open-vela/frameworks_bluetooth)
  - [BTstack](https://github.com/bluekitchen/btstack)
  - [AirPods Pro 3 specs](https://support.apple.com/en-us/125135)
- GNSS: [u-blox F11 (CNX)](https://www.cnx-software.com/2026/07/06/u-blox-f11-low-power-dual-band-gnss-chips-and-modules-consume-just-7mw-in-leap-mode/)
- LoRa:
  - [Semtech LR2021](https://www.semtech.com/products/wireless-rf/lora-plus/lr2021)
  - [Meshtastic LR2021 PR](https://github.com/meshtastic/firmware/pull/11819)
  - [MeshCore LR2021 PR](https://github.com/meshcore-dev/MeshCore/pull/3112)
- Displays:
  - [GDEY0397T81P](https://www.good-display.com/product/613.html)
  - [GDEY037T03](https://www.good-display.com/product/437.html)
  - [Sharp LS032B7DD02](https://datasheet.octopart.com/LS032B7DD02-Sharp-Microelectronics-datasheet-145014743.pdf)
  - [RLCD guide (Waveshare)](https://docs.waveshare.com/ESP32-Peripheral-Tutorials/Display/RLCD)
  - [EPDiy](https://github.com/vroland/epdiy)
- Processors:
  - [STM32N6](https://www.st.com/en/microcontrollers-microprocessors/stm32n6-series.html)
  - [STM32MP25 (CNX)](https://www.cnx-software.com/2023/05/16/stm32mp25-stmicro-stm32mp2-arm-cortex-a35-m33-mpu-family/)
  - [STM32H7 DSI host](https://www.st.com/content/ccc/resource/training/technical/product_training/group0/04/62/1b/8e/e0/bc/4a/cb/STM32H7-Peripheral-DSI_HOST_interface_DSIHOST/files/STM32H7-Peripheral-DSI_HOST_interface_DSIHOST.pdf/_jcr_content/translations/en.STM32H7-Peripheral-DSI_HOST_interface_DSIHOST.pdf)
  - [OV5640 on STM32H7](https://github.com/aaljo222/ov5640-stm32h7-capture-tutorial)
- Reference phones:
  - [BlackBerry Classic](https://en.wikipedia.org/wiki/BlackBerry_Classic)
  - [Minimal Phone review](https://goodereader.com/blog/reviews/full-review-of-the-minimal-phone-with-an-e-ink-screen)
