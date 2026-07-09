# x86 Bootloader and Minimal Kernel

A from-scratch exploration of the x86 boot process, built in two stages: a raw 512-byte Master Boot Record (MBR) loaded directly by the BIOS, and a GRUB-loaded, Multiboot2-compliant kernel running in 32-bit protected mode.

## Overview

Every x86 machine boots the same way: the BIOS loads a fixed 512-byte sector, hands it control in 16-bit real mode, and trusts it completely. This project traces that entire chain — **BIOS → MBR → Boot Loader → Kernel** — by implementing both the most primitive form of booting (a hand-written MBR) and the approach used by real operating systems (GRUB + Multiboot2).

## Part 1 — Master Boot Record

A 512-byte x86 assembly program (`boot.asm`) that:
- Sets up segment registers (`DS`, `SS`) and a 4 KiB stack by hand, since nothing is initialized for you in real mode.
- Prints `Hello` to the screen using BIOS interrupt `0x10` (teletype output).
- Is padded to exactly 510 bytes and terminated with the mandatory boot signature `0xAA55`, without which the BIOS refuses to treat the sector as bootable.

**Build & run:**
```bash
nasm -f bin -o boot.bin boot.asm
ls -l boot.bin        # must be exactly 512 bytes
qemu-system-x86_64 -drive file=boot.bin,index=0,media=disk,format=raw
```

## Part 2 — GRUB and a Minimal Kernel

Since real-mode BIOS calls are unavailable once GRUB hands off in 32-bit protected mode, this stage writes directly to VGA memory instead.

- **`multiboot_header.asm`** — a Multiboot2 header GRUB scans for, containing a magic number, architecture field, length, and checksum that must sum to zero mod 2³².
- **`kernel.asm`** — a minimal 32-bit kernel that writes `OK` directly to the VGA text buffer at `0xB8000` and halts.
- **`linker.ld`** — links the header and kernel into an ELF binary, ensuring the Multiboot2 header lands in the first 8 KiB (GRUB only scans that far) and loading everything at `1 MiB`.
- **`grub.cfg`** — the GRUB menu entry pointing at `kernel.bin`.

**Build & run:**
```bash
nasm -f elf32 multiboot_header.asm -o multiboot_header.o
nasm -f elf32 kernel.asm -o kernel.o
ld -m elf_i386 -o kernel.bin -T linker.ld multiboot_header.o kernel.o
readelf -a kernel.bin   # verify entry point is 0x100020

mkdir -p cdrom/boot/grub
cp kernel.bin cdrom/boot/
cp grub.cfg   cdrom/boot/grub/
grub2-mkrescue -o cdrom.iso cdrom

qemu-system-x86_64 -cdrom cdrom.iso
```

## Prerequisites

```bash
sudo dnf install nasm qemu-system-x86 grub2-tools-extra xorriso mtools
```

## Key Concepts Demonstrated

| Concept | Where |
|---|---|
| Real-mode segment addressing (`segment × 16 + offset`) | `boot.asm` |
| BIOS interrupts for I/O | `boot.asm` (`int 0x10`) |
| Boot sector signature requirement | `boot.asm` |
| Multiboot2 header format & checksum | `multiboot_header.asm` |
| Linker script control over memory layout | `linker.ld` |
| Direct VGA text-buffer memory access | `kernel.asm` |
| Protected-mode kernel entry via GRUB | full Part 2 pipeline |

## Result

| | Part 1 — MBR | Part 2 — GRUB + Kernel |
|---|---|---|
| CPU mode | 16-bit real mode | 32-bit protected mode |
| Output method | BIOS interrupt `0x10` | Direct VGA memory write |
| Boot medium | Raw 512-byte binary | ISO 9660 image |
| Loaded by | BIOS | GRUB (Multiboot2) |
| Message | `Hello` | `OK` |

## Reference

Montelius, J. (2018). *These boots are made for walking.* KTH Royal Institute of Technology.
