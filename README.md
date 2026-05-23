# 🧠 Snorkel-OS

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Latest Release](https://img.shields.io/github/v/release/Paullknya/Paull-kernel)](https://github.com/Paullknya/Paull-kernel/releases/latest)

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&pause=1000&color=00FF00&background=0D1117&width=500&lines=32-bit+x86+kernel+from+scratch;Protected+mode+%2B+syscalls;Rings+0%2F3+%2B+command+line;FAT32+and+multitasking+next)](https://git.io/typing-svg)

**paull-kernel(kernel the Snorkel-OS) 32‑bit x86 kernel from scratch**  
Protected mode, hardware interrupts, syscalls, rings 0/3, command line.

![Paull‑kernel shell](demo.gif)

## Build & Run

```bash
nasm -f bin boot.asm -o boot.bin
nasm -f bin kernel.asm -o kernel.bin
dd if=/dev/zero of=paull-kernel.img bs=512 count=2880
dd if=boot.bin of=paull-kernel.img bs=512 count=1 conv=notrunc
dd if=kernel.bin of=paull-kernel.img bs=512 seek=1 conv=notrunc
qemu-system-i386 -fda paull-kernel.img
```
Features

  - Protected mode (flat model, GDT, IDT)

  - Keyboard IRQ1 with scancode → ASCII

  - System calls int 0x80 (write, exit, getchar)

  - Command line with backspace, Enter

   - Magic sequences: hew → hello world, bye → goodbye cruel world, tu → Hello 231(tu)

   - Ring 3 switch on F1 with syscall demo

   - sudo reboot (password: admin)

Roadmap

   - FAT32 driver (read‑only)

  -  Round‑robin multitasking

  -  Paging (identity mapping)

  - 64‑bit long mode

Questions?

Join Discussions or open an Issue.

🔗 Author: Paullknya
💬 Community forum: paullknya.github.io
