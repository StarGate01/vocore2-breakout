# vocore2-breakout

KiCad breakout board for the VoCore2 module (MT7628AN SoC).

## J6 - JTAG Connector (J-Link EDU Mini)

J6 is a 2×5 1.27 mm pitch header wired to the standard 9-pin ARM Cortex debug
connector used by the J-Link EDU Mini.

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | `+3V3` | VTref - sense only, do not power board from here |
| 2 | `JTAG_TMS` | MT7628 `GPIO41` / `EPHY_LED2_N_JTMS` |
| 3 | `GND` | |
| 4 | `JTAG_CLK` | MT7628 `GPIO40` / `EPHY_LED3_N_JTCLK` |
| 5 | `GND` | |
| 6 | `JTAG_TDO` | MT7628 `GPIO43` / `EPHY_LED0_N_JTDO` |
| 7 | NC | KEY position - no pin on J-Link EDU Mini |
| 8 | `JTAG_TDI` | MT7628 `GPIO42` / `EPHY_LED1_N_JTDI` |
| 9 | `JTAG_TRST` | MT7628 `GPIO39` / `EPHY_LED4_N_JTRST_N` |
| 10 | `RESET` | MT7628 pin 138 / `PORST_N` (system reset, active-low) |

### J-Link EDU Mini reset pin specialities

The J-Link EDU Mini uses the **9-pin Cortex-M variant** of the connector, not the full 20-pin JTAG header:

- **Pin 7** is a physical key (no pin present). Not connected.
- **Pin 9** is repurposed by Segger as **nTRST** (JTAG TAP reset, active-low). In the standard Cortex-M spec pin 9 is NC; Segger uses it to support targets that need hardware TRST. Wire to `JTRST_N` (`GPIO39`).
- **Pin 10** is **nRESET** (system reset, active-low). Wire to `PORST_N`.

`WDT_RST_N` (`GPIO37` / MT7628 pin 137) appears adjacent to `PORST_N` and `JTRST_N` on the VoCore2 right-side connector and was considered for pin 9.

The vendor's JTAG how-to uses `WDT_RST_N` as "TRST" in their OpenOCD `sysfsgpio` config. This works because their setup is one VoCore2 bit-banging GPIOs to debug another - the host just needs two GPIO outputs to toggle, and driving `WDT_RST_N` low from an external GPIO triggers a full chip reset which also resets the JTAG TAP. OpenOCD's `sysfsgpio` driver doesn't care what hardware the signal is physically connected to; it just toggles the GPIO.

A J-Link is different. It is a hardware probe that follows the ARM Cortex debug connector spec strictly: pin 9 (`nTRST`) is driven by J-Link silicon to reset **only the JTAG TAP**, and the probe asserts and de-asserts it independently during normal debug sessions. If pin 9 connects to `WDT_RST_N`, every `nTRST` assertion by the J-Link triggers a full watchdog system reset, killing the debug session. `WDT_RST_N` is also a chip *output* when the watchdog fires, so the J-Link driving it creates a bus conflict. `JTRST_N` (`GPIO39`) resets only the TAP and has no output driver conflict.

### Pull-up resistors

All five JTAG signals (`TMS`, `TCK`, `TDI`, `TDO`, `TRST`) need pull-ups to 3V3. `RESET` (`PORST_N`) does not need external pull-ups from the breakout.

### OpenOCD configuration

Add `reset_config trst_and_srst` to the configuration so OpenOCD knows both hardware reset lines are present. Without this it auto-probes, which is version-dependent.

## JP1 - TXD1 JTAG mode selector

MT7628 `GPIO13` / `TXD1` (VoCore2 pad `PR01`) is sampled at power-on reset to determine whether the `EPHY_LED` pins (`GPIO39`–`GPIO43`) operate as Ethernet LEDs or as the JTAG interface.

| `TXD1` at boot | Mode |
|----------------|------|
| HIGH (3V3) | Normal - `EPHY_LED` pins as Ethernet LEDs, **JTAG disabled** |
| LOW (GND) | **JTAG enabled** - `EPHY_LED` pins as `TMS`/`TCK`/`TDI`/`TDO`/`TRST` |

`JP1` is a 3-pin jumper with 4.7 kΩ resistors on both options:

| `JP1` position | `TXD1` | JTAG |
|---------------|--------|------|
| 1-2 (default) | HIGH via `R1` (4K7 to `+3V3`) | Disabled |
| 2-3 | LOW via `R2` (4K7 to `GND`) | **Enabled** |

### VoCore2 module modification required

The stock VoCore2 module has **`R9` populated** (pull-up, 4.7 kΩ to 3V3) which holds `TXD1` high and disables JTAG. To use JTAG:

1. Remove `R9` from the VoCore2 module (eliminates the internal pull-up).
2. Set `JP` to position 2-3 on the breakout.

The breakout `JP` then has sole control over `TXD1` with no internal resistor fighting it. Leaving `R9` on the module while setting `JP` to 2-3 creates a voltage divider (~1.65 V, undefined logic level).

The original VoCore2 JTAG how-to describes moving `R9` to the `R6` footprint (a pull-down). With `R9` removed entirely and `JP` at 2-3, the effect is the same.

## References

- [J-Link EDU Mini KB](https://kb.segger.com/J-Link_EDU_Mini)
- [9-pin JTAG/SWD connector](https://kb.segger.com/9-pin_JTAG/SWD_connector)
- [VoCore2 pinout](https://vocore.io/v2.html)
- [VoCore2 JTAG how-to](https://vonger.cn/?p=14897)
