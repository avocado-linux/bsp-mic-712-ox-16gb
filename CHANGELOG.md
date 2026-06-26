# Changelog

All notable changes to avocado-bsp-mic-712-ox-16gb are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0]

### Added
- Initial release: Board support for the Advantech MIC-712-OX (carrier extension on jetson-orin-nx).
- Carrier provisioning overrides in `stone/carrier-bsp/` (super-mode kernel DTB,
  super BPMP DTB + TE980M firmware, Advantech-modified MB1 pinmux + GPIO BCT)
  selected via `carrier.env`, pinned to the P3767-0000 SOM.
- `CARRIER_ENV_ODMDATA=gbe-uphy-config-9` — routes the GBE UPHY to PCIe C10
  (ASMedia switch + I350s) instead of the base `config-8` (4× MGBE). Fixes a
  boot-time BL31 RAS Uncorrectable Error at `p2u_gbe_1` (`0x…3f300c0`) seen on
  the first flash, where MB1 left C10's GBE UPHY unpowered.
- Runtime package set for the carrier I/O: Intel igb NICs (I210 + 6× I350),
  Realtek r8169, the mttcan CAN stack, the tca9554 GPIO expanders, the rt5640
  audio codec, the SPI TPM, plus i2c-tools/libgpiod-tools/alsa-utils/can-utils.
- `mic-712-ox` convenience CLI (poe / gpio / can / audio / mpcie-reset / mcu /
  fan-sync), a clean reimplementation of the factory bring-up scripts — no
  `devmem`; GPIO via libgpiod v2.
- Boot-time services: `mic-712-ox-poe` (IP808 PSE), `mic-712-ox-carrier-init`
  (CAN transceiver + interface), `mic-712-ox-fan-sync` (dual-fan PWM mirror),
  `mic-712-ox-audio` (rt5640 routing). `mic-712-ox-mcu` ships opt-in/disabled.
- CI via the shared `avocado-linux/actions` reusable workflows.

### Fixed (post-first-boot validation on hardware)
- CAN module load: `can_up` ran `modprobe can can_raw mttcan` (which loaded `can`
  with `can_raw`/`mttcan` as *parameters*). Now `modprobe -a`.
- CAN interface bring-up: the avocado base ships only BusyBox `ip`, which can't
  `ip link set type can bitrate`. Added `iproute2`; the CLI detects an iproute2
  `ip` and gives a clear message if absent.
- GPIO by name: the Advantech DTB bakes direction hints into line names
  (`"CAN_RST, IN"`), so exact-name lookups failed. The CLI now resolves by the
  name prefix (before the comma) to chip+offset — `CAN_RST`/`CAN_TX_EN` work.
- `gpio set` uses libgpiod v2 `--daemonize` (v2 gpioset holds until exit; killing
  it mid-I2C wedges the tca9554) with a `/proc`-based prior-holder drop.
- Audio: `amixer` is in the `alsa-utils-amixer` subpackage, not `alsa-utils`.
  `mic-712-ox audio` now detects a missing rt5640/APE card and skips cleanly
  (the analog audio-graph card doesn't register on the 6.6 kernel — see README).
- Verified `CONFIG_IGB=m` + all carrier in-tree modules build on **both** the
  linux-yocto 6.6 and linux-jammy-nvidia-tegra 5.15 kernels (shared cfg fragment).

### Requires
- Kernel config change in meta-avocado-nvidia: `CONFIG_IGB=m`
  (`recipes-kernel/linux/files/avocado-extra.cfg`) — the igb driver for the
  I210 + 6× I350 NICs is not in the stock tegra defconfig. Rebuild the kernel.
