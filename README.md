# 🔊 minimal-linux-audio

A minimal Linux system built from scratch that boots in QEMU and plays audio. No distro, no package manager — just a custom kernel, a hand-crafted initramfs, and BusyBox.

This project started as a challenge: *can you get audio working in a bare-bones Linux environment from scratch?* The answer is yes — but it takes patience.

---

## What This Is

- A **custom Linux kernel** (6.19.3) compiled with HDA Intel audio support
- A **minimal initramfs** (~24MB) containing BusyBox, ALSA userspace tools, and just enough libs to function
- Boots and plays WAV audio inside **QEMU** using a virtual Intel HDA sound card
- No systemd, no udev, no distro — just `init`, `sh`, and the bare essentials

---

## Requirements

- A Linux host (tested on Debian/Ubuntu with PipeWire)
- `qemu-system-x86_64`
- `gcc`, `make`, `bc`, `flex`, `bison`, `libelf-dev` (for kernel build)
- `cpio`, `gzip` (for initramfs)

```bash
sudo apt install qemu-system-x86 gcc make bc flex bison libelf-dev cpio gzip
```

---

## Project Structure

```
minimal-linux-audio/
├── README.md                   ← you are here
├── build.sh                    ← builds initramfs + launches QEMU
├── kernel.config               ← kernel config used to compile bzImage
└── fs/                         ← initramfs root filesystem
    ├── init                    ← init script (first thing the kernel runs)
    ├── bin/
    │   ├── busybox             ← provides sh, ls, cat, mount, echo, mknod, umount
    │   ├── cat
    │   ├── echo
    │   ├── ls
    │   ├── mknod
    │   ├── mount
    │   ├── sh
    │   └── umount
    ├── lib/
    │   ├── ld-musl-i386.so.1   ← musl dynamic linker
    │   └── libc.musl-x86.so.1  ← musl libc
    ├── usr/
    │   ├── bin/
    │   │   ├── aplay           ← WAV audio playback
    │   │   ├── arecord         ← audio recording
    │   │   ├── amixer          ← mixer/volume control
    │   │   ├── alsamixer       ← interactive mixer
    │   │   ├── speaker-test    ← audio test tone generator
    │   │   └── ...             ← other ALSA tools
    │   ├── lib/
    │   │   ├── libasound.so.2          ← symlink
    │   │   └── libasound.so.2.0.0      ← ALSA userspace library
    │   └── share/
    │       └── alsa/
    │           ├── alsa.conf   ← ⚠️ CRITICAL - without this, aplay fails
    │           ├── cards/      ← card-specific config files
    │           └── pcm/        ← PCM device definitions
    ├── etc/
    │   └── asound.conf         ← ALSA default device config
    ├── sbin/
    ├── var/
    └── dev/                    ← device nodes (created at boot by init)
```

---

## Building

### 1. Compile the Kernel

```bash
# Download kernel source from https://kernel.org (6.19.3)
tar xf linux-6.19.3.tar.xz
cd linux-6.19.3

cp ../kernel.config .config     # use the provided config
make olddefconfig               # apply it
make -j$(nproc)                 # compile (takes a while)
```

Key options enabled in `kernel.config`:
```
CONFIG_SND=y
CONFIG_SND_HDA_INTEL=y
CONFIG_SND_HDA_CODEC_REALTEK=y
CONFIG_INITRAMFS_SOURCE=""
```

### 2. Build the Initramfs

```bash
cd fs/
find . | cpio -o -H newc | gzip > ../init.cpio
```

Or just use the build script:

```bash
chmod +x build.sh
./build.sh initramfs
```

---

## Running

```bash
chmod +x build.sh
./build.sh        # auto-detects your audio backend and launches QEMU
```

Or manually:

```bash
qemu-system-x86_64 \
    -kernel linux-6.19.3/arch/x86/boot/bzImage \
    -initrd init.cpio \
    -audiodev pipewire,id=snd0 \
    -device intel-hda \
    -device hda-duplex,audiodev=snd0 \
    -append "console=ttyS0" \
    -nographic
```

Once booted at the `~ #` prompt:

```bash
# Play a WAV file
aplay -B 500000 /your_file.wav

# Test audio with a tone
speaker-test -t sine -f 440

# Check volume
amixer
```

> **Audio backend:** The build script auto-detects PipeWire / PulseAudio / ALSA. If launching manually, replace `pipewire` with `pa` (PulseAudio) or `alsa` accordingly.

---

## The Hard-Won Lessons

This took several hours of debugging. Here's what actually matters so you don't repeat the same pain:

### ⚠️ #1 — The most critical fix: ALSA config files

The single biggest blocker was missing `/usr/share/alsa/alsa.conf` inside the initramfs. Without it, `libasound` cannot resolve *any* PCM device names — not even `hw:0`. The error looks like this:

```
ALSA lib pcm.c:2722:(snd_pcm_open_noupdate) Unknown PCM hw:0
aplay: main:850: audio open error: No such file or directory
```

This is deeply confusing because `amixer` still works (it uses the control interface, not PCM), making it look like the hardware is fine but aplay is broken. The fix is simply:

```bash
mkdir -p fs/usr/share/alsa
cp -r /usr/share/alsa/alsa.conf fs/usr/share/alsa/
cp -r /usr/share/alsa/cards     fs/usr/share/alsa/
cp -r /usr/share/alsa/pcm       fs/usr/share/alsa/
```

### ⚠️ #2 — Match the QEMU audio backend to your host

| Host audio system | QEMU flag |
|-------------------|-----------|
| PipeWire | `-audiodev pipewire,id=snd0` |
| PulseAudio | `-audiodev pa,id=snd0` |
| Plain ALSA | `-audiodev alsa,id=snd0` |

Using the wrong backend means QEMU silently drops all audio with no error.

### ⚠️ #3 — Always wire the audiodev explicitly

```bash
-device hda-duplex,audiodev=snd0   ✅ correct
-device hda-duplex                  ❌ audio silently dropped
```

### ℹ️ #4 — Underruns are normal in QEMU

```
underrun!!! (at least 23.589 ms long)
```

This is QEMU struggling to feed audio data fast enough — not a real error. Use a larger buffer to reduce them:

```bash
aplay -B 500000 /your_file.wav
```

---

## Known Issues

- The initramfs is **32-bit (i386/musl)** — you cannot copy ALSA binaries directly from a 64-bit host, they must be cross-compiled or sourced from a 32-bit environment
- `aplay` crashes with an assertion error at the very end of a file — this is a PipeWire timing quirk, the audio plays fine
- No MP3 support — convert first with `ffmpeg -i input.mp3 output.wav` on your host

---

## Credits

- Inspired by this tutorial: [How to Build a Minimal Linux System](https://www.youtube.com/watch?v=vPU1j8aCD-w&t=640s)
- Linux kernel: [kernel.org](https://kernel.org)
- ALSA project: [alsa-project.org](https://www.alsa-project.org)

---

## License

MIT — do whatever you want with it.

---

*Built from scratch. Debugged with patience. It works.*
