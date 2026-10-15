# Porting to another SOF + ES8336 laptop

MateBookAudio was written for one machine (Huawei MateBook D15 2020, Comet Lake, board
BOHB-WAX9). The SOF IPC3 path and the ES8336 programming are generic; the values below are
board-specific. The easiest way to get them is to boot a Linux live USB on the target machine,
where `sof_es8336` already works, and copy what Linux does.

## 1. Collect the facts under Linux

```bash
lspci -nn | grep -i audio                       # DSP PCI ID (here 8086:02c8, Comet Lake-LP)
dmesg | grep -iE 'sof|es83|quirk'               # firmware, topology, sof_es8336 quirk mask
cat /sys/class/dmi/id/board_name                # board name (quirk table key)
cat /sys/kernel/debug/gpio                      # "speakers-enable" / "headphone-enable" lines
sudo i2cdump -y <bus> 0x10                      # ES8336 registers while audio is playing
```

The `sof_es8336` quirk mask tells you the SSP port, whether there is a DMIC, which GPIO
drives the amplifier and which mic input the headset uses (`SOC_ES8336_HEADSET_MIC1`).
This board uses quirk `0x1a0` = SSP0 | DMIC | headphone GPIO | headset on MIC1.

## 2. Values to change

| What | Where | MateBook D15 value |
|---|---|---|
| DSP PCI ID | `kext/MateBookAudio/Info.plist` → `IOPCIMatch` | `0x02c88086` |
| SOF firmware | `kext/MateBookAudio/Firmware/sof-*.ri` (embedded by `sof_firmware_embed.s`) | `sof-cml.ri` |
| Topology | pipelines are built by hand in `MateBookAudioDevice.cpp` (PIPE1 / PIPE2 / PIPE3), mirroring the `.tplg` Linux loads | `sof-cml-es8336-dmic2ch-ssp0.tplg` |
| SSP port and clocks | `SSP_GOLD_*` constants and `sspDaiConfig()` in `MateBookAudioDevice.cpp` | SSP0, MCLK 24 MHz, BCLK 2.4 MHz, 24-in-25 bits |
| Amplifier GPIO pads | `HUAWEI_*` defines in `ES8336Codec.hpp` (pin numbers, community, PADBAR) **and** `MB_SetAmp()` / padbar fallback in `plugin/MateBookAudioPlugin.c` | pins 40/41 (GPP_B15/B16), community 0, both high = speakers |
| Headset mic input | `ES8336Codec::setHeadsetMic()` → `ADC_PDN_LINSEL` | MIC1 (`0x00`, lin1-rin1) — MIC2 boards need `0x10` |
| DMIC channels | PIPE3 in `MateBookAudioDevice.cpp` | 2 channels |
| Codec I2C device | ACPI `ESSX8336` matched through VoodooI2C (`ES8336Codec.cpp`) | address from ACPI (usually 0x10) |

The headset mic on this board samples one bit early on SSP0 RX; the plugin compensates with a
1-bit shift (`MateBookAudioPlugin.c`, input path). Check the raw samples with
`tools/matebook_ctl` before keeping it on another board.

## 3. Bring-up order that worked here

1. Kext loads, DSP boots the firmware (`matebook_ctl status` shows the ROM status / FW ready).
2. Playback pipeline + SSP0 DAI_CONFIG accepted (no IPC errors in `log stream`).
3. Codec registers match the Linux dump while playing (`matebook_ctl calibrate spk`).
4. Amplifier GPIOs: hear the speakers, then the headphones.
5. Capture: DMIC first, then the headset mic.
6. Sleep / wake.

`docs/AUDIO.md` lists every bug hit along the way; most of them (S24 vs S32 formats, LEN_24,
`SYS_PDN`, DAC_SET1, triggering the host component only) will show up on any SOF + ES8336 port.

## 4. Speaker tuning

The output EQ in `plugin/MateBookDSP.h` is calibrated for the D15 speakers. On another laptop,
measure with `tools/measure/fr_measure.c` and redesign the EQ with `tools/measure/eqdesign.py`,
or start with the headphone (flat) profile.
