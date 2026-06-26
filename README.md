# bsp-mic-712-ox-16gb

Avocado OS board-support extension for the **Advantech MIC-712-OX** — a Jetson
Orin NX (P3767-0000, 16 GB) AI edge / PoE gateway on Advantech's P3768-class
carrier. It layers carrier provisioning + runtime support onto the generic
`jetson-orin-nx` target.

Derived from the extracted Advantech BSP `MIC-712_OrinNX_6.2.0_V1.0.3`
(L4T R36.4.3) and forensics on a factory Ubuntu unit.

> **Module-scoped.** This extension is pinned to the **Orin NX 16 GB** module
> (P3767-**0000**) via `CHECK_BOARDSKU=0000` — flashing it onto a different
> module fails fast at the tegraflash boardid check. The carrier also ships with
> the **8 GB** Orin NX (P3767-**0001**); support that with a sibling
> `bsp-mic-712-ox-8gb` differing only in `CHECK_BOARDSKU=0001`, the `-0001-`
> kernel + BPMP DTBs, and the SKU-0001 SDRAM BCTs — the carrier-side files
> (pinmux, GPIO BCT, runtime) are identical. This mirrors the MIC-733 AO5A1/AO6A1
> split.

## Carrier hardware

| Function | Detail |
|---|---|
| SOM | Jetson Orin NX P3767-0000 (16 GB), flashed in **super** mode |
| 8× GbE | I210 (`eth0`, mgmt) + 6× I350 behind an ASMedia ASM1806 PCIe switch (`eth1-6`, the PoE ports) + Realtek RTL8168 (`eth7`) |
| PoE | 8-port PoE/PoE+ via an **IP808AR PSE** on `i2c-1` @ `0x74` (no kernel driver; userspace-configured, ~53 V, AT mode, 150 W budget) |
| CAN | 2× `mttcan` (`c310000`/`c320000`) |
| GPIO | 3× TI `tca9554` expanders (`i2c` 0/1/7 @ `0x20`): camera ctrl, RS-232/422/485 enables + mode/term, CAN reset, IP808 SMBus select |
| Audio | `realtek,rt5640` codec (`i2c0` @ `0x1c`) |
| Sensors | `ti,ina3221` rail monitor (`i2c1` @ `0x40`); 2× PWM fan |
| Misc | onboard TPM (SPI), USB 3.x hub (Genesys), NVMe, MCU (Nuvoton M031) |

## Runtime bring-up & the `mic-712-ox` CLI

All carrier control lives in a single tool, `/usr/bin/mic-712-ox` (a clean
reimplementation of the factory `/opt/advantech` scripts, which are
proprietary). It uses libgpiod v2 + i2c-tools and reads `/etc/default/mic-712-ox`.

```
mic-712-ox poe init|status|on [PORT]|off [PORT]   # IP808 PSE (8 ports)
mic-712-ox gpio get <NAME> | set <NAME> <0|1>     # by-name GPIO (libgpiod v2)
mic-712-ox can up|down [iface] [bitrate]          # mttcan (default can0/125k)
mic-712-ox audio                                  # route rt5640 onto I2S4
mic-712-ox mpcie-reset                            # pulse mini-PCIe reset (PK.03)
mic-712-ox mcu enable|release                     # M031 MCU lines (advanced)
mic-712-ox fan-sync [--daemon]                    # mirror fan PWM
mic-712-ox carrier-init                           # boot bring-up (used by systemd)
```

Enabled-by-default services (in `enable_services`):

| Service | Does |
|---|---|
| `mic-712-ox-poe` | arms the 8 IP808 PSE ports (volatile — re-run every boot) |
| `mic-712-ox-carrier-init` | releases the CAN transceiver (`CAN_RST`/`CAN_TX_EN`) + brings CAN up |
| `mic-712-ox-fan-sync` | mirrors the primary fan PWM onto the secondary (thermals) |
| `mic-712-ox-audio` | rt5640 ALSA routing (best-effort; analog audio only) |

Opt-in (not enabled): `mic-712-ox-mcu.service` — see below.

### PoE

The IP808AR PSE has no kernel driver; `mic-712-ox poe init` configures it over
`/dev/i2c-1` (53 V, AT mode, 150 W) and arms all 8 ports. Per-port toggle:
`mic-712-ox poe off 3` / `on 3`. Disabling a port drops its 53 V rail (confirmed
on hardware — the rear-panel LEDs are PD-detection LEDs that light only when a
powered device draws current, so an armed-but-empty port shows no LED). Raw
register access if needed:

```sh
i2cset -y 1 0x74 0x00 0x40          # select page 1
i2cset -y 1 0x74 0x9<N> 0x00|0x01   # port N (0..7): regs 0x98..0x9f
```

### Ported from the factory `bsp_init`, and what was left out

Ported (via the CLI/services above): CAN transceiver release + bring-up, the
rt5640 audio routing, dual-fan sync, the mini-PCIe reset pulse, and PoE.

**No `devmem`.** The factory poked raw SoC registers; we don't. GPIO/MCU/
mini-PCIe lines are driven with `gpioset` (the MB1 pinmux BCT we ship already
muxes the pads), and tca9554 expander outputs latch in-chip so they persist
after the setter exits.

Deliberately **off by default**:

- **M031 MCU** (`mic-712-ox-mcu.service`, opt-in): the factory holds
  `MCU_PROG_EN` (PQ.05) and `MCU1_RST` (PP.06) **low** via a daemon. On an
  active-low part, RST-low reads as *held in reset*; the MCU's role isn't
  required for NICs/PoE/CAN/audio, so we don't touch it. Enable with
  `systemctl enable --now mic-712-ox-mcu.service` or `MIC712_ENABLE_MCU=1`.
- **USB micro-port eye-pattern tuning** (`devmem 0x03520094 w 0x6`): a signal-
  integrity tweak that belongs in the DTB pinmux, not a runtime poke. Not
  ported; add to the XUSB padctl node in the carrier DTB if micro-USB
  compliance matters.

## Provisioning overrides

`stone/carrier-bsp/` holds the files that differ from stock L4T (pulled from
the Advantech BSP, renamed `-adv`). `carrier.env` redirects the tegraflash
flow at them. The Advantech deltas vs. stock R36.4.3 are:

- **`ODMDATA=gbe-uphy-config-9,…`** (`CARRIER_ENV_ODMDATA`) — the UPHY lane map.
  The base `jetson-orin-nx` defaults to `gbe-uphy-config-8` (GBE UPHY → 4× MGBE);
  the MIC-712-OX routes the GBE UPHY to PCIe **C10** (`pcie@141e0000`) for the
  ASMedia switch + I350s, which is `config-9`. **Required** — without it MB1
  doesn't power C10's GBE UPHY and BL31/UEFI faults with a RAS UE at `p2u_gbe_1`
  (`0x…3f300c0`), powering off the core before boot.
- `tegra234-p3768-0000+p3767-0000-nv-super-adv.dtb` — kernel DTB with the
  carrier nodes (3× tca9554, rt5640, ina3221, 2× mttcan). The 4× MGBE nodes are
  `disabled` (the carrier uses PCIe NICs, not MGBE).
- `tegra234-mb1-bct-pinmux-p3767-hdmi-a03-adv.dtsi` — carrier pin routing;
  `#include`s the `-adv` GPIO BCT below.
- `tegra234-mb1-bct-gpio-p3767-hdmi-a03-adv.dtsi` — boot GPIO defaults.
- `tegra234-bpmp-3767-0000-3768-super.dtb` + `bpmp_t234-TE980M-A1_prod.bin` —
  the **super**-mode BPMP DTB + firmware (chip SKU D3 → TE980M). The base
  `jetson-orin-nx` MACHINE bakes the non-super defaults, so these are shipped
  here and selected via `carrier.env`.
- `tegra234-mb1-bct-padvoltage-p3767-hdmi-a03-adv.dtsi` — stock pad-voltage,
  shipped only so the pinmux/GPIO `#include` chain resolves within the overlay.

`CHECK_BOARDID=3767` / `CHECK_BOARDSKU=0000` pin the SOM so a wrong Orin NX/Nano
SKU fails fast at the tegraflash boardid check.

### Super vs. standard mode

This BSP mirrors the factory image, which flashes **super** mode (higher
MAXN-SUPER clocks; the MIC-712-OX has active cooling). To use the conservative
non-super profile instead, drop the super BPMP override and use the non-super
DTB: set `CARRIER_FV_BPFDTB_FILE="tegra234-bpmp-3767-0000-a02-3509-a02.dtb"`
and `CARRIER_ENV_DTBFILE` to a non-super carrier DTB.

## Status

**Validated on hardware** (flashed `jetson-orin-nx` + this extension, L4T R36.5.0
bootloader): all 8 NICs enumerate (igb I210 + 6× I350 behind C10/ASMedia, r8169),
PoE arms all 8 ports with working per-port control, CAN `can0` comes up at
125 kbps, the dual fans track, and the GPIO expanders/transceiver lines drive
correctly. First flash crashed in BL31 with a RAS UE at `p2u_gbe_1` — fixed by
`CARRIER_ENV_ODMDATA=gbe-uphy-config-9` (see above).

### Kernel support (both)

Verified on **both** avocado kernels — `CONFIG_IGB=m` and every carrier in-tree
module (`r8169`, `can`/`can-raw`/`can-dev`, `gpio-pca953x`, `at24`, `ina3221`,
`snd-soc-rt5640`/`rl6231`, `tcg-tis-spi`, `spidev`) are `=m` in the resolved
`.config` of **linux-yocto 6.6** *and* **linux-jammy-nvidia-tegra 5.15** (both
kernel bbappends consume the shared `avocado-extra.cfg`). `mttcan`/`nvpps` are
NVIDIA OOT modules built by `nvidia-kernel-oot` for both multiconfigs. The
per-kernel wifi blocks (5.15 → rtl8822ce, 6.6 → rtw88) and the `mic-712-ox` CLI
(pure userspace) are kernel-agnostic.

### Known limitation — analog audio on 6.6

The rt5640 **analog** audio (Tegra APE `nvidia,tegra186-audio-graph-card`) does
**not** register on the 6.6 kernel: the R36.4.3 audio-graph DT node fails ASoC
registration with `-EBUSY` ("parse error") — the one place the R36.4.3-DTB /
R36.5.0-kernel skew bites. `mic-712-ox-audio` detects the missing card and skips
cleanly. Expected to work on the 5.15 kernel (native DTB vintage) — verify if
analog audio is needed. HDMI audio (HDA card) works regardless.

### Open / deferred

- **L4T version skew.** The carrier DTB + MB1 BCT are R36.4.3 binaries under the
  R36.5.0 stack. `ODMDATA` is version-independent and the carrier I/O works, but
  regenerate the deltas against R36.5.0 sources to also recover analog audio.
- **Opt-in init** (not required for NICs/PoE/CAN) — the M031 MCU hold and the
  micro-USB eye-pattern tweak; see the bring-up section above.

## Using this extension

```yaml
extensions:
  avocado-bsp-mic-712-ox-16gb:
    source:
      type: package
      version: "*"

runtimes:
  my-runtime:
    extensions:
      - avocado-bsp-mic-712-ox-16gb
```

```sh
avocado install
avocado build
```
