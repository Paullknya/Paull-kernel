# 🧠 Paull-Kernel-OS

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Latest Release](https://img.shields.io/github/v/release/Paullknya/AXonOS-kernel)](https://github.com/Paullknya/AXonOS-kernel/releases/latest)

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&pause=1000&color=00FF00&background=0D1117&width=500&lines=32-%D0%B1%D0%B8%D1%82%D0%BD%D0%B0%D1%8F+x86+%D0%9E%D0%A1+%D1%81+%D0%BD%D1%83%D0%BB%D1%8F;%D0%97%D0%B0%D1%89%D0%B8%D1%89%D1%91%D0%BD%D0%BD%D1%8B%D0%B9+%D1%80%D0%B5%D0%B6%D0%B8%D0%BC+%2B+%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%BD%D1%8B%D0%B5+%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D1%8B;%D0%9A%D0%BE%D0%BB%D1%8C%D1%86%D0%B0+0%2F3+%2B+%D0%BA%D0%BE%D0%BC%D0%B0%D0%BD%D0%B4%D0%BD%D0%B0%D1%8F+%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0;FAT32+%D0%B8+%D0%BC%D0%BD%D0%BE%D0%B3%D0%BE%D0%B7%D0%B0%D0%B4%D0%B0%D1%87%D0%BD%D0%BE%D1%81%D1%82%D1%8C+%D0%B4%D0%B0%D0%BB%D0%B5%D0%B5)](https://git.io/typing-svg)

**32-битная x86 операционная система с нуля**  
Защищённый режим, аппаратные прерывания, системные вызовы, кольца 0/3, командная строка.

![AxonOS shell](demo.gif)

## Сборка и запуск

```bash
nasm -f bin boot.asm -o boot.bin
nasm -f bin kernel.asm -o kernel.bin
dd if=/dev/zero of=axonos.img bs=512 count=2880
dd if=boot.bin of=axonos.img bs=512 count=1 conv=notrunc
dd if=kernel.bin of=axonos.img bs=512 seek=1 conv=notrunc
qemu-system-i386 -fda axonos.img
```
## Возможности

  - Защищённый режим (плоская модель, GDT, IDT)

 - Клавиатура IRQ1, преобразование скан-кода в ASCII

 - Системные вызовы int 0x80 (write, exit, getchar)

 - Командная строка с Backspace и Enter

 - Магические последовательности: hew → hello world, bye → goodbye cruel world, tu → Hello 231(tu)

 - Переключение в кольцо 3 по F1 с демонстрацией системного вызова

 - sudo reboot (пароль: admin)

 ## Планы

 - Драйвер FAT32 (только чтение)

 - Круговая многозадачность

 - Страничная память (identity mapping)

 - 64-битный длинный режим

## Вопросы?

Присоединяйтесь к обсуждениям или откройте Issue.

https://github.com/Paullknya/AXonOS-kernel/discussions
