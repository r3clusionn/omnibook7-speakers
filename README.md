# omnibook7-speakers

A Linux kernel patch that makes the built-in speakers of the HP OmniBook 7 14-kh0xxx work. The laptop drives its speakers through two Cirrus Logic CS35L41 amplifiers, and its ACPI tables don't describe them the way the driver needs. Without the patch the amplifiers fail to probe and the speakers stay silent. The patch adds the laptop (subsystem ID 103C8F2E) to the driver's table of laptops whose amplifier settings are supplied by the kernel, as many HP models already are.

**Status:** in daily use on an HP OmniBook 7 14-kh0xxx (Ryzen AI 9 HX 475) with Linux 7.2.9, applied through Gentoo's user patches. It hasn't been sent upstream.

![Terminal output on a dark background, two dmesg excerpts. Without the patch: "Failed property cirrus,dev-index: -22", "error -EINVAL: Platform not supported", "probe with driver cs35l41-hda failed with error -22". With the patch: "Adding DSD properties for 103C8F2E", "Using extra _DSD properties, bypassing _DSD in ACPI", "Calibration applied: R0=10717", "Firmware Loaded - Type: spk-prot, Gain: 17", "CS35L41 Bound - SSID: 103C8F2E, BST: 1, VSPK: 1, CH: R, FW EN: 1, SPKID: 1"](docs/images/dmesg.png)

## Features

- **Speakers on the HP OmniBook 7 14-kh0xxx** (board 8F2E), through the snd-hda-scodec-cs35l41 driver.
- **Two lines in one file** (`sound/hda/codecs/side-codecs/cs35l41_hda_property.c`): the laptop's amplifier description and its entry in the match table.
- **Uses Cirrus' firmware** for this model from linux-firmware: speaker protection, tuning and calibration (`cirrus/cs35l41-dsp1-spk-prot-103c8f2e*`).

## How to install

Requires the kernel sources (a version with `sound/hda/codecs/side-codecs/`, as in 7.x) and linux-firmware with the `cirrus/cs35l41-dsp1-spk-prot-103c8f2e*` files.

**Gentoo** (user patches, applied whenever gentoo-sources is installed):

```sh
doas mkdir -p /etc/portage/patches/sys-kernel/gentoo-sources
doas cp cs35l41-103c8f2e.patch /etc/portage/patches/sys-kernel/gentoo-sources/
doas emerge -1 sys-kernel/gentoo-sources     # then build and install the kernel as usual
```

**Any kernel tree:**

```sh
cd /usr/src/linux
patch -p1 < /path/to/cs35l41-103c8f2e.patch
```

The kernel needs `CONFIG_SND_HDA_SCODEC_CS35L41_I2C` (module or built in), and the firmware files must be where the driver loads them (`/lib/firmware/cirrus/`).

## How to use

Nothing to do after booting the patched kernel: the speakers are an ordinary output of the laptop's HD-Audio device ("Ryzen HD Audio Controller Speaker"). Check it worked with:

```sh
dmesg | grep cs35l41     # "CS35L41 Bound - SSID: 103C8F2E" for both amplifiers
```

## How it works

- **The amplifiers** are on I2C and described in ACPI by a `CSC3551` device. The driver expects `_DSD` properties there telling it how many amplifiers there are, their channels, their GPIOs and boost settings. This laptop's firmware doesn't provide them, so the probe fails with `Failed property cirrus,dev-index` and "Platform not supported".
- **The driver has a fallback** for such laptops: a table of subsystem IDs with the properties written out (`cs35l41_prop_model_table`), matched through `generic_dsd_config`. The patch adds 103C8F2E with:
  - two amplifiers, left and right channel;
  - external boost (`EXTERNAL`; the logs report `BST: 1`), so there are no boost inductor or capacitor values;
  - the reset line at GPIO index 0, the speaker ID at GPIO 1, no chip-select GPIO.
- **Once bound,** the driver loads Cirrus' speaker-protection firmware for 103c8f2e, applies the factory calibration (R0), and attaches the amplifiers to the Realtek codec's speaker output.

## Measurements

None: this is a driver match. The kernel log shows the result (see Verification).

## Verification

- `dmesg` on the patched kernel shows, for both amplifiers (hda.0 and hda.1): DSD properties added for 103C8F2E, firmware loaded (spk-prot, gain 17), calibration applied, and "CS35L41 Bound". Without the patch both failed to probe with -EINVAL (`Platform not supported`).
- Sound plays from the speakers in daily use. Applications open the "Ryzen HD Audio Controller Speaker" device; Minecraft's log, for one, shows "OpenAL initialized on device Ryzen HD Audio Controller Speaker".

## Limits

- Tested on one laptop (14-kh0xxx, board 8F2E). Other OmniBook 7 variants with other subsystem IDs need their own entry.
- The settings were chosen to match the board and similar HP models, not taken from HP documentation.
- Not upstream yet. The same two lines would make a valid patch for the kernel's sound tree.

## License

GPL-2.0, as the Linux kernel file it changes (see `LICENSE`).
