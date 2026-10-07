# nextpi-sandbox

Run [NextPi](https://wiki.specnext.dev/Pi:NextPi) — the Raspberry Pi Zero
accelerator distribution for the ZX Spectrum Next — on a desktop machine with
QEMU, so that ZX Spectrum Next emulators can talk to a real NextPi over the
Next's Pi UART: NextZXOS's own `.pisend`, the NextPi UI, and the speech, music
and tape-streaming players all work, with the Pi's sound on your speakers.

- **[jnext](https://github.com/jorgegv/jnext)** has this built in and does not
  need this project: `jnext --nextpi` (or **Settings > Preferences > Pi Zero**)
  downloads NextPi on first use, starts QEMU and wires the Pi to UART 1 itself.
- **CSpect** is supported through `bin/run-cspect`, which bridges its serial
  port to a NextPi this project starts.
- On its own, `bin/run-nextpi` boots NextPi with its console on your terminal
  or a TCP port.

## Requirements

- QEMU (`qemu-system-arm`, `qemu-img`)
- mtools (to copy the kernel out of the image without mounting it)
- curl, tar
- ~22 GB free disk (≈6 GB archive + ≈15 GB image; `--remove-archive` frees the archive)

On macOS:

```bash
brew install qemu mtools
```

## With jnext

Nothing from this project is needed — jnext prepares and runs NextPi itself
(proposed in [jorgegv/jnext#310](https://github.com/jorgegv/jnext/pull/310)):

```bash
jnext --nextpi
```

The first run offers to download NextPi (about 6 GB) into `~/.jnext/pizero`,
with a progress bar; later runs start at once. NextPi then takes about a minute
to boot. At the NextZXOS command line:

```
.pisend -q
.pisend -c nextpi-play_speech "Hello from the Spectrum Next"
```

To keep it on, tick **Start the Pi Zero** under **Settings > Preferences > Pi
Zero**, where the NextPi release (default `1_93D`, or `latest`), the directory,
the QEMU binary and the Pi's audio can also be changed. Typing on a Mac: `"` is
⌘+P and `_` is ⌘+0 in jnext (Symbol Shift is the key Qt calls Ctrl, which on
macOS is ⌘).

How jnext runs NextPi is what this project worked out: the same QEMU `raspi0`
machine, kernel command line and USB audio set-up described under
[How it works](#how-it-works).

## Quick start on its own

```bash
bin/setup-nextpi      # download the latest release from zx.xalior.com and prepare it
bin/run-nextpi        # boot; the Supervisor prompt "SUP>" appears after ~1 minute
```

Quit QEMU with **Ctrl-A then X**.

## bin/setup-nextpi

| Command | Effect |
|---|---|
| `bin/setup-nextpi` | install the latest release |
| `bin/setup-nextpi 1_93D` | install a specific release |
| `bin/setup-nextpi --list` | list releases on the mirror |
| `bin/setup-nextpi --remove-archive` | delete the `.tar.gz` after extracting |

The download resumes if interrupted and is MD5-verified. Rerunning is cheap: an
already-extracted release is not downloaded again.

## bin/run-nextpi

| Command | Effect |
|---|---|
| `bin/run-nextpi` | serial console on this terminal |
| `bin/run-nextpi --tcp 5577` | serial console on `127.0.0.1:5577` (for an emulator, or `nc 127.0.0.1 5577`) |
| `bin/run-nextpi --wav out.wav` | record the Pi's audio to a file instead of the speakers |
| `bin/run-nextpi --reset` | throw away all changes and boot the pristine image |

The release image is never modified; all writes go to `dist/overlay.qcow2`.

## bin/run-cspect

Runs [CSpect](https://mdf200.itch.io/cspect) with its Pi UART (`-com2`) wired to
the sandbox, so `.pisend` and other NextPi tools work from NextZXOS.

```bash
brew install socat                                  # plus mono from mono-project.com
bin/run-cspect --sd ~/next-images/cspect-next.img   # a NextZXOS SD card image
```

| Command | Effect |
|---|---|
| `bin/run-cspect --sd IMG` | boot CSpect from `IMG`, starting NextPi if it isn't running |
| `bin/run-cspect --sd IMG -fullscreen` | extra arguments are passed to CSpect |
| `bin/run-cspect --port 5578 ...` | use another TCP port for the Pi console (default 5577) |

`CSPECT_HOME` (default `~/bin/CSpect`) and `NEXT_SD` (default for `--sd`) can be
set in the environment. CSpect only opens serial devices, so `socat` makes a
pseudo-terminal bridged to `bin/run-nextpi --tcp`. The script discards the Pi's
boot output and waits for the `SUP>` prompt before launching CSpect; output
already waiting on the device would otherwise stop `.pisend -q` finding `SUP>`. A NextPi started by the script
stops when CSpect exits; to keep the Pi up across CSpect restarts (boot takes ~1
minute), run `bin/run-nextpi --tcp 5577` in another terminal first. Only one
client can attach to the console port, so close any `nc` session.

At the NextZXOS prompt (CSpect starts with `-basickeys`, so `.` types normally):

```
.pisend -q
.pisend -c nextpi-play_speech "Hello from the Spectrum Next"
```

## Trying out sound

At the `SUP>` prompt:

```
nextpi-play_speech "Hello from the Spectrum Next"
```

MIDI (the image ships no MIDI files; the console is the only way in, so send one
as base64). On the host:

```bash
base64 -i examples/twinkle.mid | pbcopy
```

Then at `SUP>` type `echo `, paste, and finish the line with
` | base64 -d > /ram/twinkle.mid`. Play it with:

```
nextpi-play_midiLQ /ram/twinkle.mid -g 1.5
```

FluidSynth takes ~30 s to load its soundfont from the emulated SD card before
the first note. `-g 1.5` raises its default gain (0.2), which is very quiet.
`/ram` is volatile; files there vanish on reboot. Pasting works for files of a
few KB.

## How it works

- **Machine**: QEMU `raspi0`, booting the release's own `kernel.img` and
  `bcm2708-rpi-zero.dtb`, which `setup-nextpi` copies out of the image's FAT boot
  partition into `dist/boot/`.
- **SD card**: a 16 GB qcow2 overlay over the raw image; QEMU requires
  power-of-two SD card sizes and the release image is not one.
- **Audio**: a QEMU `usb-audio` device on the Pi's USB port. On real hardware,
  NextPi plays to the I2S DAC as ALSA's default card; here the kernel option
  `snd_usb_audio.index=0` makes the USB card the default instead, so the
  `nextpi-play_*` commands work unchanged. The Pi USB driver's FIQ mode hangs
  device enumeration under QEMU and is disabled (`dwc_otg.fiq_enable=0 …`).
  On macOS the CoreAudio buffers are enlarged to avoid clicks.
- **Serial**: the PL011 UART (`ttyAMA0`, which the NextPi Supervisor listens on)
  is the QEMU serial port.

## Not emulated

- Networking (the raspi0 machine has none).
- I2S audio into the Next's mixer: the Pi's audio goes to the host speakers
  rather than through the Next (so NextREG 0xA2 enable/mute have no effect).
- Real-time speed: the emulated Pi is slower than a real Pi Zero.

## Layout

```
bin/setup-nextpi   download + prepare a release
bin/run-nextpi     boot it
bin/run-cspect     run CSpect connected to it
examples/          small test files (twinkle.mid)
dist/              (git-ignored) archives, extracted releases, boot files, overlay
```

Set `NEXTPI_HOME` to keep `dist/` somewhere else (e.g. an external disk).
