; ===========================================================================
; AxonOS Kernel (32-bit protected mode)
; Build: nasm -f bin kernel.asm -o kernel.bin
; Loader expects kernel at 0x10000, reads 32 sectors (16384 bytes)
; ===========================================================================

[bits 32]
[org 0x10000]

; ---------------------------------------------------------------------------
; Константы
; ---------------------------------------------------------------------------
VIDEO_MEM   equ 0xB8000
SCREEN_W    equ 80
SCREEN_H    equ 25
CURSOR_INIT equ 0          ; начальная позиция курсора (в символах)

; Селекторы (плоская модель, настроенные загрузчиком)
SEL_CODE    equ 0x08
SEL_DATA    equ 0x10
SEL_CODE3   equ 0x1B
SEL_DATA3   equ 0x23
SEL_TSS     equ 0x28

; ---------------------------------------------------------------------------
; Данные ядра
; ---------------------------------------------------------------------------
section .data
msg_welcome     db "AxonOS 32-bit ready", 0xA, 0
msg_unknown     db "Unknown command.", 0xA, 0
msg_sudo_usage  db "usage: sudo <command>", 0xA, 0
pass_prompt     db "[sudo] password: ", 0
pass_fail       db "Sorry, try again.", 0xA, 0
prompt_ring3    db "ring3> ", 0
reboot_str      db "reboot", 0
sudo_str        db "sudo", 0
admin_pass      db "admin", 0

; Последовательности
seq_hew         db "hew", 0
seq_bye         db "bye", 0
seq_tu          db "tu", 0
msg_hello       db "hello world", 0xA, 0
msg_bye         db "goodbye cruel world", 0xA, 0
msg_tu          db "Hello 231(tu)", 0xA, 0

; Буфер командной строки
cmd_buffer      times 128 db 0
cmd_len         dd 0

; Буфер пароля
pass_buf        times 32 db 0

; Состояния детектирования последовательностей
hew_pos         db 0
bye_pos         db 0
tu_pos          db 0
hew_ready       db 0
bye_ready       db 0
tu_ready        db 0

; Курсор (символьная позиция от 0 до 1999)
cursor_pos      dd CURSOR_INIT

; ---------------------------------------------------------------------------
; GDT (ядро само её перезагружает, чтобы иметь TSS и сегменты ring3)
; ---------------------------------------------------------------------------
section .data
gdt:
    dq 0                                         ; null descriptor
    dw 0xFFFF, 0x0000, 0x9A00, 0x00CF           ; 0x08 code ring0
    dw 0xFFFF, 0x0000, 0x9200, 0x00CF           ; 0x10 data ring0
    dw 0xFFFF, 0x0000, 0xFA00, 0x00CF           ; 0x1B code ring3
    dw 0xFFFF, 0x0000, 0xF200, 0x00CF           ; 0x23 data ring3
    dw 0x0000, 0x0000, 0x0000, 0x0000           ; 0x28 TSS (заполним позже)
gdt_end:

gdt_desc:
    dw gdt_end - gdt - 1
    dd gdt

; ---------------------------------------------------------------------------
; TSS
; ---------------------------------------------------------------------------
section .bss
tss:
    resb 104                ; минимальный размер TSS для 32-bit (104 байта)
tss_end:

; ---------------------------------------------------------------------------
; IDT
; ---------------------------------------------------------------------------
section .data
idt_start times 256*8 db 0
idt_end:

idt_desc:
    dw idt_end - idt_start - 1
    dd idt_start

; ---------------------------------------------------------------------------
; Таблица скан-кодов (set 1)
; ---------------------------------------------------------------------------
section .data
scancode_table:
    db 0,0,'1','2','3','4','5','6','7','8','9','0','-','=',0,0
    db 'q','w','e','r','t','y','u','i','o','p','[',']',0,0,'a','s'
    db 'd','f','g','h','j','k','l',';',"'",'`',0,'\\','z','x','c','v'
    db 'b','n','m',',','.','/',0,'*',0,' ',0,0,0,0,0,0
    times 128-($-scancode_table) db 0

; ---------------------------------------------------------------------------
; Стек ядра (отдельная секция, чтобы не пересекалось с кодом)
; ---------------------------------------------------------------------------
section .bss
kernel_stack_bottom:
    resb 16384              ; 16 КБ стека
kernel_stack_top:

; Стек для кольца 3
ring3_stack_bottom:
    resb 4096
ring3_stack_top:

; ---------------------------------------------------------------------------
; Код ядра
; ---------------------------------------------------------------------------
section .text

; ---------- Утилиты: вывод строки, newline, очистка экрана ----------
print_string:
    pusha
    mov edi, [cursor_pos]
    shl edi, 1
    add edi, VIDEO_MEM
.loop:
    lodsb
    test al, al
    jz .done
    cmp al, 0xA
    je .newline
    mov ah, 0x07
    stosw
    inc dword [cursor_pos]
    jmp .loop
.newline:
    call newline
    jmp .loop
.done:
    popa
    ret

newline:
    push eax ebx edx
    mov eax, [cursor_pos]
    mov ebx, SCREEN_W
    xor edx, edx
    div ebx
    mov eax, SCREEN_W
    sub eax, edx
    add [cursor_pos], eax
    pop edx ebx eax
    ret

cls:
    pusha
    mov edi, VIDEO_MEM
    mov ecx, SCREEN_W * SCREEN_H
    mov ax, 0x0720
    rep stosw
    mov dword [cursor_pos], 0
    popa
    ret

; ---------- Backspace (стирает один символ с экрана и в буфере) ----------
backspace:
    cmp dword [cmd_len], 0
    je .nothing
    dec dword [cmd_len]
    mov eax, [cursor_pos]
    test eax, eax
    jz .nothing
    dec dword [cursor_pos]
    mov edi, [cursor_pos]
    shl edi, 1
    add edi, VIDEO_MEM
    mov word [edi], 0x0720   ; пробел с атрибутом
.nothing:
    ret

; ---------- Стирание последовательности (hew/bye/tu) с экрана и буфера ----------
; Вход: al = количество символов для стирания (3 для hew/bye, 2 для tu)
erase_sequence:
    push ecx
    mov ecx, eax
.erase_loop:
    call backspace
    loop .erase_loop
    pop ecx
    ret

; ---------- Сравнение строк (ESI = str1, EDI = str2) -> CF=1 если равны ----------
; Возвращает флаг CF (чтобы удобно использовать jc/jnc)
strcmp:
    pusha
    cld
.loop:
    mov al, [esi]
    mov bl, [edi]
    cmp al, bl
    jne .not_equal
    test al, al
    jz .equal
    inc esi
    inc edi
    jmp .loop
.not_equal:
    popa
    clc
    ret
.equal:
    popa
    stc
    ret

; ---------- Преобразование скан-кода в ASCII ----------
; Вход: al = скан-код, Выход: al = ASCII (0 если непечатный)
scancode_to_ascii:
    mov ebx, scancode_table
    xlat
    ret

; ---------- Обновление состояний hew/bye/tu при вводе символа ----------
; Вход: al = ASCII-символ
update_sequences:
    pusha
    ; ---- hew ----
    mov bl, [hew_pos]
    cmp bl, 3
    je .skip_hew
    movzx ebx, bl
    mov dl, [seq_hew + ebx]
    cmp dl, al
    jne .reset_hew
    inc byte [hew_pos]
    cmp byte [hew_pos], 3
    jne .skip_hew
    mov byte [hew_ready], 1
    jmp .skip_hew
.reset_hew:
    mov byte [hew_pos], 0
    mov byte [hew_ready], 0
.skip_hew:
    ; ---- bye ----
    mov bl, [bye_pos]
    cmp bl, 3
    je .skip_bye
    movzx ebx, bl
    mov dl, [seq_bye + ebx]
    cmp dl, al
    jne .reset_bye
    inc byte [bye_pos]
    cmp byte [bye_pos], 3
    jne .skip_bye
    mov byte [bye_ready], 1
    jmp .skip_bye
.reset_bye:
    mov byte [bye_pos], 0
    mov byte [bye_ready], 0
.skip_bye:
    ; ---- tu ----
    mov bl, [tu_pos]
    cmp bl, 2
    je .skip_tu
    movzx ebx, bl
    mov dl, [seq_tu + ebx]
    cmp dl, al
    jne .reset_tu
    inc byte [tu_pos]
    cmp byte [tu_pos], 2
    jne .skip_tu
    mov byte [tu_ready], 1
    jmp .skip_tu
.reset_tu:
    mov byte [tu_pos], 0
    mov byte [tu_ready], 0
.skip_tu:
    popa
    ret

; ---------- Чтение пароля (без эха, звёздочки) ----------
; Выход: CF=1 если пароль верен, иначе CF=0
read_password:
    pusha
    mov esi, pass_prompt
    call print_string
    mov edi, pass_buf
    xor ecx, ecx
.read_loop:
    call getch
    cmp al, 0x0D          ; Enter
    je .check
    cmp al, 0x08          ; Backspace
    je .backspace
    cmp ecx, 31
    je .read_loop
    stosb
    inc ecx
    push eax
    mov al, '*'
    call putchar
    pop eax
    jmp .read_loop
.backspace:
    cmp ecx, 0
    je .read_loop
    dec ecx
    dec edi
    call backspace
    jmp .read_loop
.check:
    mov byte [edi], 0
    mov esi, pass_buf
    mov edi, admin_pass
    call strcmp
    jc .ok
    popa
    clc
    ret
.ok:
    popa
    stc
    ret

; ---------- Чтение одного символа с клавиатуры (блокирующее, без эха) ----------
getch:
    in al, 0x60
    test al, 0x80
    jnz getch
    call scancode_to_ascii
    test al, al
    jz getch
    ret

; ---------- Вывод одного символа (с учётом курсора) ----------
putchar:
    pusha
    cmp al, 0xA
    je .newline
    mov edi, [cursor_pos]
    shl edi, 1
    add edi, VIDEO_MEM
    mov ah, 0x07
    stosw
    inc dword [cursor_pos]
    jmp .done
.newline:
    call newline
.done:
    popa
    ret

; ---------- Перезагрузка ----------
reboot:
    mov al, 0xFE
    out 0x64, al
    jmp $

; ---------- Обработка командной строки ----------
process_command:
    pusha
    ; Завершаем строку нулём
    mov ebx, cmd_buffer
    mov ecx, [cmd_len]
    mov byte [ebx+ecx], 0

    ; Пропускаем пробелы в начале
    mov esi, cmd_buffer
    cmp byte [esi], ' '
    jne .check_sudo
    inc esi

.check_sudo:
    mov edi, sudo_str
    call strcmp
    jc .handle_sudo

    ; Обычные команды
    mov edi, reboot_str
    call strcmp
    jc .do_reboot

    ; Если команда не распознана
    mov esi, msg_unknown
    call print_string
    jmp .cleanup

.handle_sudo:
    ; пропускаем "sudo"
    add esi, 4
    cmp byte [esi], ' '
    jne .sudo_usage
    inc esi

    push esi
    call read_password
    jnc .wrong_pass
    pop esi
    ; теперь проверим подкоманду
    mov edi, reboot_str
    call strcmp
    jc .do_reboot
    ; иначе неизвестная команда
    mov esi, msg_unknown
    call print_string
    jmp .cleanup

.sudo_usage:
    mov esi, msg_sudo_usage
    call print_string
    jmp .cleanup

.wrong_pass:
    mov esi, pass_fail
    call print_string
    pop esi  ; убираем сохранённый esi
    jmp .cleanup

.do_reboot:
    call reboot

.cleanup:
    ; Очищаем буфер команд
    mov dword [cmd_len], 0
    mov edi, cmd_buffer
    mov ecx, 128
    xor al, al
    rep stosb
    ; Переводим строку
    call newline
    popa
    ret

; ---------- Кольцо 3 (демонстрация) ----------
ring3_entry:
    mov esi, prompt_ring3
    call print_string_ring3
    mov eax, 1        ; sys_exit
    int 0x80
    jmp $             ; никогда не сюда

print_string_ring3:
    pusha
    mov edi, [cursor_pos]
    shl edi, 1
    add edi, VIDEO_MEM
.l:
    lodsb
    test al, al
    jz .d
    mov ah, 0x07
    stosw
    inc dword [cursor_pos]
    jmp .l
.d:
    popa
    ret

enter_ring3:
    ; Устанавливаем esp0 в TSS для возврата из кольца 0 в кольцо 3
    mov eax, kernel_stack_top
    mov [tss + 4], eax
    mov dword [tss + 8], SEL_DATA   ; ss0

    push dword SEL_DATA3
    push dword ring3_stack_top
    pushfd
    pop eax
    or eax, 0x200      ; IF=1
    push eax
    push dword SEL_CODE3
    push dword ring3_entry
    iret

; ---------- Обработчик клавиатуры (IRQ1, вектор 33) ----------
keyboard_handler:
    pushad
    push ds
    push es

    mov ax, SEL_DATA
    mov ds, ax
    mov es, ax

    in al, 0x60
    test al, 0x80
    jnz .finish

    ; Проверка F1 (скан-код 0x3B)
    cmp al, 0x3B
    je .f1_pressed

    ; Преобразование в ASCII
    call scancode_to_ascii
    test al, al
    jz .finish

    ; Обработка Enter
    cmp al, 0x0D
    je .enter_pressed

    ; Обработка Backspace
    cmp al, 0x08
    je .backspace_pressed

    ; Обработка пробела с активацией hew/bye/tu
    cmp al, ' '
    jne .normal_char

    cmp byte [hew_ready], 1
    je .do_hew
    cmp byte [bye_ready], 1
    je .do_bye
    cmp byte [tu_ready], 1
    je .do_tu
    jmp .normal_char

.do_hew:
    mov eax, 3
    call erase_sequence
    mov esi, msg_hello
    call print_string
    mov byte [hew_ready], 0
    mov byte [hew_pos], 0
    jmp .finish

.do_bye:
    mov eax, 3
    call erase_sequence
    mov esi, msg_bye
    call print_string
    mov byte [bye_ready], 0
    mov byte [bye_pos], 0
    jmp .finish

.do_tu:
    mov eax, 2
    call erase_sequence
    mov esi, msg_tu
    call print_string
    mov byte [tu_ready], 0
    mov byte [tu_pos], 0
    jmp .finish

.normal_char:
    call update_sequences
    ; Добавляем в буфер команд
    mov ebx, [cmd_len]
    cmp ebx, 127
    jae .skip_buffer
    mov edi, cmd_buffer
    add edi, ebx
    stosb
    inc dword [cmd_len]
    ; Выводим на экран
    call putchar
    jmp .finish

.skip_buffer:
    ; буфер полон – игнорируем
    jmp .finish

.backspace_pressed:
    call backspace
    jmp .finish

.enter_pressed:
    call process_command
    jmp .finish

.f1_pressed:
    call enter_ring3
    jmp .finish

.finish:
    mov al, 0x20
    out 0x20, al
    pop es
    pop ds
    popad
    iret

; ---------- Системные вызовы (int 0x80) ----------
syscall_handler:
    pushad
    push ds
    push es

    mov ax, SEL_DATA
    mov ds, ax
    mov es, ax

    cmp eax, 0
    je .sys_write
    cmp eax, 1
    je .sys_exit
    cmp eax, 2
    je .sys_getchar

.sys_write:
    ; ebx = указатель на строку, ecx = длина
    mov esi, ebx
    mov ecx, [esp+8]   ; ? осторожнее, но в данном примере передаём через ecx
    ; Упростим: будем выводить через print_string, но для длины используем цикл
    ; Лучше сделать свою функцию putstr_n
    push ecx
    call print_string  ; выводит до нуля, игнорируя длину – не совсем правильно
    pop ecx
    jmp .done

.sys_exit:
    ; Возврат в ядро – просто выходим из системного вызова и попадаем в main_loop
    jmp .done

.sys_getchar:
    call getch
    mov [esp+16], eax   ; возвращаемое значение в eax после popad
    jmp .done

.done:
    pop es
    pop ds
    popad
    iret

; ---------- Инициализация IDT и PIC ----------
init_pic:
    mov al, 0x11
    out 0x20, al
    out 0xA0, al
    mov al, 0x20
    out 0x21, al
    mov al, 0x28
    out 0xA1, al
    mov al, 0x04
    out 0x21, al
    mov al, 0x02
    out 0xA1, al
    mov al, 0x01
    out 0x21, al
    out 0xA1, al
    mov al, 0xFD    ; разрешить IRQ1
    out 0x21, al
    mov al, 0xFF
    out 0xA1, al
    ret

set_gate:
    push edi
    mov edi, idt_start
    mov ecx, ebx
    shl ecx, 3
    add edi, ecx
    mov word [edi], ax
    mov word [edi+2], SEL_CODE
    mov byte [edi+4], 0
    mov byte [edi+5], 0x8E
    shr eax, 16
    mov word [edi+6], ax
    pop edi
    ret

init_idt:
    mov eax, keyboard_handler
    mov ebx, 33
    call set_gate
    mov eax, syscall_handler
    mov ebx, 0x80
    call set_gate
    lidt [idt_desc]
    ret

; ---------- Точка входа ----------
start:
    call cls
    mov esi, msg_welcome
    call print_string

    ; Перезагружаем GDT (с нашими сегментами и TSS)
    lgdt [gdt_desc]
    mov ax, SEL_DATA
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov esp, kernel_stack_top

    ; Инициализируем TSS
    mov eax, tss
    mov word [gdt + SEL_TSS + 0], (tss_end - tss - 1)
    mov word [gdt + SEL_TSS + 2], ax
    shr eax, 16
    mov byte [gdt + SEL_TSS + 4], al
    mov byte [gdt + SEL_TSS + 5], 0x89
    mov byte [gdt + SEL_TSS + 6], 0
    mov byte [gdt + SEL_TSS + 7], ah
    mov ax, SEL_TSS
    ltr ax

    call init_pic
    call init_idt
    sti

main_loop:
    hlt
    jmp main_loop

; ---------------------------------------------------------------------------
; Заполнение до размера 16 КБ (32 сектора), чтобы загрузчик прочитал всё ядро
; ---------------------------------------------------------------------------
times 512*32 - ($-$$) db 0
