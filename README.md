# postmarketOS on Google Pixel 4a (sunfish)

Run Linux on your Pixel 4a — Plasma Mobile, mainline kernel, real haptics.

![status](https://img.shields.io/badge/status-daily%20driver%20except%20cellular%2Faudio-orange)

---

## What you get

✅ Display + GPU (full Plasma Mobile)
✅ Touchscreen
✅ Charging + battery percentage
✅ Wi-Fi + Bluetooth
✅ Power button + volume keys
✅ **Haptics that feel like stock Android** (Pixel CLICK on every keystroke)
✅ USB-C tethering / USB-net to a computer

❌ Cellular data / calls / SMS (modem firmware loads but rmnet stays down)
❌ Audio (speakers, earpiece, mics — codec not wired)
❌ Sensors (auto-rotate, ambient light — ADSP firmware crash-loops)
❌ Cameras
❌ NFC, fingerprint

So it's a **Wi-Fi-only mini computer**, not a phone-phone. Plenty for browsing, terminal, code editor, calls over Signal/Matrix once audio lands, etc.

---

## Before you start

> ⚠️ **This wipes everything on your phone.** Back up first.

> ⚠️ **You will permanently unlock the bootloader.** Some banking / DRM apps will refuse to run if you ever go back to Android. Cannot be re-locked cleanly.

You need:
- A **Google Pixel 4a 4G** (codename `sunfish`). Not the 5G model — that's `bramble`, different SoC, won't work.
- A **Linux desktop or laptop** (Ubuntu / Fedora / Arch / Debian / etc.). macOS/Windows can work via WSL2 but is harder; not covered here.
- A **USB-C cable**. Data cable, not just power.
- About **45 min** of your time.

Estimated comfort level: you should be okay running shell commands and not freak out when something says "wiping userdata."

---

## Step 1 — Unlock your Pixel's bootloader

**On the Pixel 4a (stock Android):**
1. Settings → About phone → tap **Build number** seven times → developer mode enabled.
2. Settings → System → Developer options → enable **OEM unlocking**. (If the toggle is grayed out: connect to internet, wait an hour, try again. Google's policy.)
3. Power off the phone completely.
4. Hold **Power + Volume Down** until you see the white bootloader screen ("FASTBOOT MODE").

**On your computer:**
1. Install `android-tools` (provides `fastboot` and `adb`).

   ```sh
   # Arch / Manjaro
   sudo pacman -S android-tools

   # Ubuntu / Debian
   sudo apt install android-tools-adb android-tools-fastboot

   # Fedora
   sudo dnf install android-tools
   ```

2. Plug the phone into your computer. Verify:

   ```sh
   fastboot devices
   # should show: <some-serial>	fastboot
   ```

3. Unlock:

   ```sh
   fastboot flashing unlock
   ```

   The phone will show a warning screen. Press **Volume Up** to select "Unlock the bootloader", then **Power** to confirm. The phone wipes everything and reboots back into fastboot.

You stay in fastboot. Don't unplug yet.

---

## Step 2 — Install pmbootstrap on your computer

`pmbootstrap` is the tool that builds and flashes postmarketOS.

```sh
# Arch / Manjaro
sudo pacman -S pmbootstrap

# Ubuntu / Debian (newer pip; pmbootstrap isn't in the apt repos)
sudo apt install python3-pip python3-venv git
pipx install pmbootstrap

# Fedora
sudo dnf install pmbootstrap   # or: pipx install pmbootstrap
```

Verify:
```sh
pmbootstrap --version
# 3.9.0 or newer
```

---

## Step 3 — Clone this repo

```sh
cd ~
git clone https://github.com/miromraz/pmaports-sunfish.git
cd pmaports-sunfish
```

---

## Step 4 — Run pmbootstrap init

Pmbootstrap is going to ask you a bunch of questions. Here's what to answer:

```sh
pmbootstrap --aports=$PWD init
```

Important answers:
| Question | Answer |
|-|-|
| Work path | accept default (`~/.local/var/pmbootstrap`) |
| Channel | **edge** |
| Vendor | **google** |
| Device codename | **sunfish** |
| Username | whatever you want (just `user` is fine) |
| User interface | **plasma-mobile** |
| systemd | **never** (we use OpenRC; the haptic init script depends on it) |
| FDE | up to you — `none` is simpler for a first try |
| Hostname | whatever |
| Timezone | yours |

For everything else, accept defaults.

---

## Step 5 — Build the image

```sh
pmbootstrap install
```

This downloads/builds packages, prepares a system image. **Takes 30–90 minutes** on the first run (the kernel is the slow part). It'll ask you to set a password for `user`. **Use a numeric PIN** (e.g. `1234`) so you can type it on the lockscreen later — the on-screen keyboard only shows numbers there.

> ☕ Coffee break. The kernel takes 20–40 min on its own.

---

## Step 6 — Flash to the phone

Phone should still be in fastboot from step 1. If not, hold **Power + Volume Down** until you see the FASTBOOT MODE screen.

```sh
pmbootstrap flasher flash_kernel
pmbootstrap flasher flash_rootfs
```

The phone will reboot itself. **First boot takes 2-4 minutes** — be patient.

You'll see:
1. Google bootloader screen (1-2 seconds)
2. U-Boot Tauchgang screen (10-20 seconds, blue boot logo)
3. pmOS Plymouth splash
4. Plasma Mobile login screen

If the screen stays black for more than 5 minutes, **something went wrong** — see [Troubleshooting](#troubleshooting).

---

## Step 7 — First login

Tap the numeric PIN you set during `pmbootstrap install` on the on-screen keypad.

Once you're at the Plasma Mobile home:
- ✅ **Tap a key in any text field** — you should feel a crisp Pixel-style haptic CLICK. The kernel driver autoloads the haptic waveform library and configures the chip from boot; no userspace setup needed.
- Open System Settings → Wi-Fi to connect to Wi-Fi.
- You can SSH in from your computer at `user@172.16.42.1` (USB-net comes up automatically when phone is plugged in).

---

## Troubleshooting

### "fastboot: command not found"
Install `android-tools` — see Step 1.

### Phone stays at bootloader / black screen forever
Hold Power for 15 seconds to force-power-off, then Power + Volume Down to re-enter fastboot. Re-run Step 6.

### "No space left on device" during `pmbootstrap install`
The build needs ~15 GB free disk. Run `pmbootstrap zap` to clean caches, then retry.

### Haptics feel buzzy instead of crisp
The kernel driver should auto-load `drv2624.bin` and park the chip in
SEQ+CLICK mode on every boot. To verify it ran:
```sh
ssh user@172.16.42.1
sudo dmesg | grep drv2624
# expect: "drv2624.bin uploaded (25 byte RAM image)"
sudo i2cget -f -y 9 0x5a 0x07   # MODE; expect 0x01
sudo i2cget -f -y 9 0x5a 0x0F   # SEQ1; expect 0x01
```
If `MODE` is `0x02` (RTP) instead of `0x01`, the firmware file is
probably missing. Check `ls /lib/firmware/drv2624.bin` — it should be
45 bytes and ships with the `firmware-google-sunfish` package.

### Where's my Wi-Fi password / data going to come from?
You re-set everything from scratch — this is a clean install. Wi-Fi networks need to be re-added.

### I bricked it / can't boot to Android
Pixel 4a is hard to truly brick if the bootloader is still unlocked. You can always re-flash stock Android with Google's official tool: https://flash.android.com/. Pick "Pixel 4a (sunfish)", flash latest factory image, you're back to stock.

### How do I get cellular / audio / cameras working?
You don't. Yet. Those are listed as ❌ at the top of this README. Watch the [issues page](https://github.com/miromraz/pmaports-sunfish/issues) or contribute.

---

## What's inside this repo

```
device/community/
├── device-google-sunfish/      # ← Pixel 4a-specific stuff
│   ├── APKBUILD
│   └── deviceinfo              # codename: google-sunfish
├── firmware-google-sunfish/    # ← extracted proprietary blobs
│   ├── APKBUILD
│   ├── a615_zap.mbn            # GPU shader (14 KB)
│   └── drv2624.bin             # haptic ROM library (45 B)
├── device-qcom-sm7150/         # ← shared SoC-level package
└── linux-postmarketos-qcom-sm7150/   # ← our kernel APKBUILD
```

The kernel itself lives in a **separate repo**:
https://github.com/miromraz/linux (branch `sunfish-vibrator-only`)

This pmaports repo's `linux-postmarketos-qcom-sm7150` APKBUILD pulls the
kernel source from there at build time — you don't need to clone it
manually.

---

## How the haptic recipe works (technical)

The Pixel 4a uses a TI **DRV2624** LRA driver chip (i2c-9 address 0x5a,
enable on TLMM gpio 11). No mainline driver exists, so we wrote one
(see kernel branch).

The kernel driver alone gives you working but **buzzy** vibrations.
For stock-Pixel CLICK feel you need:

1. Six chip registers programmed in a specific order (CONTROL1,
   CONTROL2 with `LIB_LRA + 1ms tick`, DRIVE_TIME for 172 Hz LRA, sine
   wave shape, autocal, OL_LRA_PERIOD).
2. Factory autocal compensation values (`18 150 0` for sunfish — passed
   to the driver via the `ti,autocal-comp` DT property; originally
   from `/persist/haptics/drv2624.cal`).
3. Google's `drv2624.bin` RAM library (45 bytes, 4 effects: CLICK, TICK,
   DOUBLE_CLICK, HEAVY_CLICK) uploaded into the chip's 1 kB RAM via
   `request_firmware_nowait()`.
4. Chip parked in `MODE=RAM Waveform Sequencer` with `WAV_FRM_SEQ1=1` so
   feedbackd's `FF_RUMBLE` triggers play CLICK from ROM, not RTP.

All four happen **inside the kernel driver** (`drivers/input/misc/drv2624.c`)
at probe time. No userspace init script. The driver also re-runs the
full init on resume because the chip's RAM is volatile.

---

## Credits

- [postmarketOS](https://postmarketos.org/) — the base distribution
- [sm7150-mainline](https://github.com/sm7150-mainline) community —
  Danila Tikhonov, Casey Connolly, David Wronek, Jens Reidel, Connor
  Mitchell
- David Heidelberg + Petr Hodina — STMicro FTS5 v4 driver patches
- The [freedreno](https://freedreno.org/) team — Adreno DRM driver
- Google — for releasing the AOSP source that let us decode the haptic
  binary format and HAL behavior

## License

Code/configs: MIT, where not otherwise specified (matches postmarketOS
pmaports convention). Firmware blobs in `firmware-google-sunfish`:
proprietary (Qualcomm/Google) — see the package's `license="proprietary"`
declaration. The Adreno blob is binary-redistributed by many vendors and
the haptic binary is fetched verbatim from the open AOSP tree.

## Contributing

This is one person's hobby fork. Issues + PRs welcome on
https://github.com/miromraz/pmaports-sunfish — particularly for getting
audio / cellular / cameras / sensors working.
