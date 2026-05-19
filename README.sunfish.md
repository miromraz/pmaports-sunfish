# postmarketOS on Google Pixel 4a (sunfish)

A mainline-Linux postmarketOS port for the Google Pixel 4a (codename
**sunfish**, Qualcomm SM7150). This is a fork of [postmarketOS
pmaports](https://gitlab.postmarketos.org/postmarketOS/pmaports) +
[sm7150-mainline/pmaports](https://github.com/sm7150-mainline/pmaports)
with the Pixel 4a-specific bringup work layered on top.

## What works

- Display + GPU (Adreno 618, with extracted `a615_zap.mbn`)
- Touchscreen (STMicro FTM5, via Heidelberg's stmfts5 v4 patches)
- Charging, fuel gauge, USB-PD type-c (PM6150 / qcom_smbx / qcom_qg)
- Wi-Fi (WCN3990) + Bluetooth
- **Haptics with Pixel-stock CLICK feel** (TI DRV2624 + Google's RAM library + factory calibration) — new mainline driver, see `device/community/firmware-google-sunfish/`
- Plasma Mobile UI
- Volume + power buttons
- USB-net over USB-C (`172.16.42.1`)

## What doesn't (yet)

- Audio (q6 stack starts but no codec/amp wired, ADSP crash-loops on CHRE)
- Sensors (accel/gyro/mag/ALS/baro/hall — blocked behind same ADSP crash)
- Cellular modem / IPA rmnet (firmware loads but rmnet stays DOWN)
- Camera (CAMSS has no SM7150 entry in mainline; IMX363/IMX355 sensor drivers don't exist mainline)
- NFC (NXP PN553 — DT node not wired)
- Fingerprint (no mainline driver for Goodix)

## How to use this fork

```sh
# Clone
git clone https://github.com/miromraz/pmaports-sunfish.git
cd pmaports-sunfish

# Tell pmbootstrap to use this aports tree
pmbootstrap init   # pick device "google-sunfish" when prompted
# OR point an existing pmbootstrap at it:
pmbootstrap --aports=$PWD install
```

The kernel source comes from
[`miromraz/linux`](https://github.com/miromraz/linux) branch
`sunfish-vibrator-only` — pulled automatically by the
`linux-postmarketos-qcom-sm7150` APKBUILD.

## Per-device packages

| Package | Purpose |
|-|-|
| `device-google-sunfish` | Pixel 4a-specific overlay. Depends on the two below + ships `/etc/local.d/drv2624-init.start` (the haptic init). |
| `device-qcom-sm7150` | Generic SoC-level package (shared with all SM7150 devices). |
| `firmware-google-sunfish` | Proprietary firmware blobs: `a615_zap.mbn` (GPU) and `drv2624.bin` (haptic waveform library). |
| `linux-postmarketos-qcom-sm7150` | Mainline kernel fork with stmfts5 v4 patches, DRV2624 driver, fuel-gauge fixes. |

## The haptic story (short version)

The Pixel 4a uses a TI **DRV2624** LRA driver chip on i2c-9 0x5a, enable pin
TLMM gpio 11. There is no mainline driver for it. We wrote one from scratch
in the kernel fork at `drivers/input/misc/drv2624.c` (+ binding YAML at
`Documentation/devicetree/bindings/input/ti,drv2624.yaml`).

For the chip to feel like stock Android — not buzzy resonant ringing — it
needs:
1. Six chip registers programmed from Google's downstream `drv2624.c`
   (CONTROL1, CONTROL2 with `LIB_LRA + 1ms ticks`, DRIVE_TIME for the
   172 Hz LRA, sine wave shape, autocal compensation, OL_LRA_PERIOD).
2. Factory calibration values from `/persist/haptics/drv2624.cal`
   (`autocal: 18 150 0`, `lra_period: 241`).
3. Google's `drv2624.bin` RAM library (45 bytes, 4 effects: CLICK, TICK,
   DOUBLE_CLICK, HEAVY_CLICK) uploaded into the chip's 1 kB RAM.
4. Chip parked in `MODE=RAM Waveform Sequencer` with `WAV_FRM_SEQ1=1`
   so feedbackd's `FF_RUMBLE` triggers play the ROM CLICK rather than RTP.

All four steps live in `/etc/local.d/drv2624-init.start` shipped by
`device-google-sunfish`.

## Firmware sources / redistribution note

The blobs in `device/community/firmware-google-sunfish/` are:
- `a615_zap.mbn` — extracted from `/vendor/firmware/a615_zap.elf` on a
  stock Pixel 4a Android 11 image. Qualcomm-signed; identical across all
  sunfish units.
- `drv2624.bin` — fetched verbatim from
  `android.googlesource.com/device/google/sunfish/+/refs/heads/android11-release/vibrator/drv2624/drv2624.bin`.
  Apache-2.0 surrounding tree.

These are vendor blobs included for replicability. Remove the package if
you'd rather extract them yourself from your phone's `vendor_a` partition.

## Credits

- [postmarketOS](https://postmarketos.org/) maintainers
- [sm7150-mainline](https://github.com/sm7150-mainline) community
  (Danila Tikhonov, Casey Connolly, David Wronek, Jens Reidel, Connor
  Mitchell)
- David Heidelberg + Petr Hodina — STMicro FTS5 v4 driver patches
- The freedreno team — Adreno DRM driver

## License

Code/configs: MIT, where not otherwise specified (matches postmarketOS
pmaports convention). Firmware blobs in `firmware-google-sunfish`:
proprietary (see the package's `license="proprietary"` declaration).
