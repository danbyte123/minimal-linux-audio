                                                                             
```
 ███╗   ███╗██╗      █████╗ 
 ████╗ ████║██║     ██╔══██╗
 ██╔████╔██║██║     ███████║
 ██║╚██╔╝██║██║     ██╔══██║
 ██║ ╚═╝ ██║███████╗██║  ██║
 ╚═╝     ╚═╝╚══════╝╚═╝  ╚═╝
   minimal · linux · audio
```
 

> A bare-bones Linux system built from scratch that boots in QEMU and plays audio.  
> No distro. No package manager. Just a custom kernel, a hand-crafted initramfs, and BusyBox.

---

## What This Is

- A **custom Linux kernel** (6.19.3) compiled with HDA Intel audio support
- A **minimal initramfs** (~24MB) containing BusyBox, ALSA userspace tools, and just enough libs to function
- Boots and plays WAV audio inside **QEMU** using a virtual Intel HDA sound card
- No systemd, no udev, no distro — just `init`, `sh`, and the bare essentials

---

## Requirements

### Tools I Used on Debian

```bash
# QEMU - to run the kernel
sudo apt install qemu-system-x86

# Compiler + essentials
sudo apt install gcc manpages-dev vim git make

# Kernel build dependencies
sudo apt install bzip2 libncurses-dev flex bison bc cpio libelf-dev libssl-dev syslinux dosfstools

# Debugging tools (optional but useful)
sudo apt install strace ltrace xtrace
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

Or use the build script:

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
aplay -B 500000 /your_file.wav   # play audio
speaker-test -t sine -f 440      # test with a tone
amixer                           # check volume
```

> **Audio backend:** replace `pipewire` with `pa` (PulseAudio) or `alsa` depending on your host.

---

## The Hard-Won Lessons

This took several hours of debugging. Here's what actually matters:

### ⚠️ #1 — Missing ALSA config files (the main culprit)

Without `/usr/share/alsa/alsa.conf` in the initramfs, `libasound` can't resolve any PCM device — not even `hw:0`. You'll see:

```
ALSA lib pcm.c:2722:(snd_pcm_open_noupdate) Unknown PCM hw:0
aplay: main:850: audio open error: No such file or directory
```

The confusing part: `amixer` still works because it uses the control interface, not PCM. Fix:

```bash
mkdir -p fs/usr/share/alsa
cp -r /usr/share/alsa/alsa.conf fs/usr/share/alsa/
cp -r /usr/share/alsa/cards     fs/usr/share/alsa/
cp -r /usr/share/alsa/pcm       fs/usr/share/alsa/
```

### ⚠️ #2 — Match QEMU audio backend to your host

| Host | QEMU flag |
|------|-----------|
| PipeWire | `-audiodev pipewire,id=snd0` |
| PulseAudio | `-audiodev pa,id=snd0` |
| Plain ALSA | `-audiodev alsa,id=snd0` |

### ⚠️ #3 — Always wire the audiodev explicitly

```bash
-device hda-duplex,audiodev=snd0   ✅
-device hda-duplex                  ❌ (audio silently dropped)
```

### ℹ️ #4 — Underruns are normal in QEMU

```
underrun!!! (at least 23.589 ms long)
```

Use `-B 500000` for a larger buffer to reduce them.

---

## Known Issues

- Initramfs is **32-bit (i386/musl)** — binaries can't be copied directly from a 64-bit host
- `aplay` crashes with an assertion at end of file — PipeWire timing quirk, audio plays fine
- No MP3 support — convert first: `ffmpeg -i input.mp3 output.wav`

---

## Credits

- Inspired by: [How to Build a Minimal Linux System](https://www.youtube.com/watch?v=vPU1j8aCD-w&t=640s)
- Linux kernel: [kernel.org](https://kernel.org)
- ALSA project: [alsa-project.org](https://www.alsa-project.org)

---

## License

MIT — do whatever you want with it.

---

*Built from scratch. Debugged with patience. It works.*
