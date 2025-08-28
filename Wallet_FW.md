# Scenario 8: Crypto Charity Wallet — Assembly Demo

📄 [Download Pitch Deck One Pager PDF  

This project explores, in a playful yet rigorous way, how one might encode a **crypto-charity wallet** entirely in **ARM Cortex‑M assembly language**. The exercise is not meant to deliver a production wallet, but rather to demonstrate, in concrete low-level terms, how *rules and recognition systems* can be represented directly at the hardware/software boundary.  

---

## Concept  
- A user **donates** by sending a message (amount + digital signature).  
- The device logs the event and marks that a **“recognition token”** is available.  
- When queried, the device replies with a short payload:  

```
Thanks: #<donation number>
```  

Thus, each contribution is acknowledged with a unique, monotonically increasing identifier.  

---

## Why assembly?  
Assembly programming provides **total visibility and control** over execution:  
- One can directly manipulate **GPIO registers, DMA engines, and interrupt vectors**, without the indirection of libraries.  
- **Timing properties** become explicit; constant-time operations can be reasoned about instruction by instruction.  
- The result is compact, transparent firmware, where each instruction has a clear and measurable effect.  

For researchers, designers, and builders, this kind of exercise is valuable because it **exposes the mechanics** that are otherwise abstracted away. It reminds us that gamified systems are not only social constructs but also sequences of hardware instructions.  

---

## What the code demonstrates  
- **Interrupt-driven design**: the NFC peripheral signals when new data arrives, and the firmware responds immediately.  
- **Constant-time verification**: comparisons are written to avoid secret-dependent timing.  
- **Flash logging**: donation records are stored with sequence numbers and checksums, providing durability and integrity.  
- **Cryptographic placeholder**: a stub where digital signature verification would occur, showing how cryptography is integrated.  
- **Secure boot skeleton**: the firmware verifies its own integrity before executing.  

---

## How to build  
On a standard ARM embedded toolchain:  
```bash
arm-none-eabi-as -mcpu=cortex-m4 -mthumb wallet_fw_thumb.s -o wallet_fw_thumb.o
arm-none-eabi-ld -Ttext=0x08000000 wallet_fw_thumb.o -o wallet_fw_thumb.elf
arm-none-eabi-objcopy -O ihex wallet_fw_thumb.elf wallet_fw_thumb.hex
```  

The resulting firmware can be flashed to an ARM Cortex‑M board (e.g., STM32, nRF52) or stepped through in a simulator.  

---

## Why this matters  
Gamified systems often live at the intersection of **rules, enforcement, and recognition**. By recasting such a system in assembly language, we make the relationship between *abstract rules* and *physical enforcement* visible.  

Assembly coding is therefore not only about performance; it is also about **transparency and trust**: we see every instruction, and we know exactly how the system enforces its promises.  

---

# Source Code (wallet_fw_thumb.s)  

```asm
;
; wallet_fw_thumb.s
; Scenario 8 — "Crypto Charity Wallet" recognition firmware (fun demo)
; Target: ARM Cortex-M4 (Thumb-2), GNU as syntax (.syntax unified)
; Notes:
;  - This is a didactic, cut‑down firmware showing low‑level control flow,
;    constant‑time helpers, flash log append, and NFC APDU handling stubs.
;  - MMIO addresses are illustrative. Replace with your MCU's real map.
;  - ECDSA verify is stubbed but shows constant‑time patterns & zeroization.
;
; Build (example):
;   arm-none-eabi-as -mcpu=cortex-m4 -mthumb wallet_fw_thumb.s -o wallet_fw_thumb.o
;   arm-none-eabi-ld -Ttext=0x08000000 wallet_fw_thumb.o -o wallet_fw_thumb.elf
;   arm-none-eabi-objcopy -O ihex wallet_fw_thumb.elf wallet_fw_thumb.hex
;
; ============================================================
    .syntax unified
    .thumb

; ------------ Constants & MMIO (illustrative!) --------------
TRNG_BASE       .equ 0x40025000
TRNG_CTRL       .equ (TRNG_BASE + 0x00)
TRNG_STAT       .equ (TRNG_BASE + 0x04)
TRNG_DATA       .equ (TRNG_BASE + 0x08)

NFC_BASE        .equ 0x40026000
NFC_CTRL        .equ (NFC_BASE + 0x00)
NFC_STAT        .equ (NFC_BASE + 0x04)
NFC_RXFIFO      .equ (NFC_BASE + 0x08)
NFC_TXFIFO      .equ (NFC_BASE + 0x0C)
NFC_INTEN       .equ (NFC_BASE + 0x10)
NFC_INTSTAT     .equ (NFC_BASE + 0x14)

FLASH_BASE      .equ 0x08040000        ; log area (bank B)
FLASH_SIZE      .equ 0x00004000        ; 16 KiB demo log
FLASH_PAGE      .equ 0x00000400        ; 1 KiB page

GPIO_LED_BASE   .equ 0x40020000
GPIO_LED_OUT    .equ (GPIO_LED_BASE + 0x0C)

; ------------ APDUs (toy demo) ------------------------------
; CLA 0x80, INS 0x30 => DONATE (data: amount[4] || sig[64])
; CLA 0x80, INS 0x40 => GET_RECOG (returns NDEF-like blob)
APDU_CLA        .equ 0x80
APDU_INS_DONATE .equ 0x30
APDU_INS_GETREC .equ 0x40

; ------------ BSS / Data ------------------------------------
    .section .bss
    .align 4
_recog_ready:       .space 4           ; 0/1 flag
_seq_counter:       .space 4           ; monotonic donation id
_rx_len:            .space 4
_rx_buf:            .space 272         ; APDU buffer
_tx_len:            .space 4
_tx_buf:            .space 272         ; response buffer

; Simulated RAM "flash mirror" for demo (replace with real flash ops)
    .section .bss
    .align 4
_flash_mirror:      .space FLASH_SIZE

; Firmware hash (demo constant) for "secure boot" compare
    .section .rodata
    .align 4
_fw_hash_expected:
    .word 0x11223344,0x55667788,0x99AABBCC,0xDDEEFF00,
          0xCAFEBABE,0xDEADC0DE,0xFEEDFACE,0x01234567

; NDEF "recognition" prefix (toy)
_ndef_prefix:
    .byte 0xD1,0x01,0x0F,0x54,0x02,0x65,0x6E       ; Short record 'T', 'en'
    .ascii "Thanks: #"                            ; text payload starts here
_ndef_prefix_len .equ . - _ndef_prefix

; ------------ Vector Table ----------------------------------
    .section .isr_vector,"a",%progbits
    .align 2
    .word   _estack             ; Initial MSP from linker script
    .word   Reset_Handler
    .word   NMI_Handler
    .word   HardFault_Handler
    .word   0,0,0,0,0,0,0       ; Reserved
    .word   SVC_Handler
    .word   0,0
    .word   PendSV_Handler
    .word   SysTick_Handler
    .word   NFC_IRQHandler      ; <- NFC interrupts
    .word   TRNG_IRQHandler     ; <- TRNG data ready / health
    .word   0                   ; others omitted

; ------------ Handlers (weak defaults) ----------------------
    .thumb_func
NMI_Handler:         b .
    .thumb_func
HardFault_Handler:   b .
    .thumb_func
SVC_Handler:         bx lr
    .thumb_func
PendSV_Handler:      bx lr
    .thumb_func
SysTick_Handler:     bx lr

; ------------ Reset / Init ----------------------------------
    .text
    .thumb_func
Reset_Handler:
    ; zero BSS: assume symbols provided by linker (_sbss.._ebss)
    ldr r0, =_sbss
    ldr r1, =_ebss
    movs r2, #0
0:  cmp r0, r1
    bcc 1f
    b   2f
1:  str r2, [r0], #4
    b   0b
2:
    ; clear flags
    ldr r0, =_recog_ready
    str r2, [r0]
    ldr r0, =_seq_counter
    str r2, [r0]

    ; minimal init
    bl  TRNG_Init
    bl  NFC_Init
    bl  LED_Blink_Once

    ; secure boot stub (constant-time compare of hash)
    bl  SecureBoot_Verify

Main_Loop:
    wfi                                 ; wait for NFC/TRNG interrupts
    b   Main_Loop

; ------------ LED utility -----------------------------------
    .thumb_func
LED_Blink_Once:
    ldr r0, =GPIO_LED_OUT
    movs r1, #1
    str r1, [r0]
    movs r2, #0x40
1:  subs r2, r2, #1
    bne 1b
    movs r1, #0
    str r1, [r0]
    bx  lr

; ------------ TRNG ------------------------------------------
    .thumb_func
TRNG_Init:
    ; enable, simple health conf
    ldr r0, =TRNG_CTRL
    movs r1, #1
    str r1, [r0]
    ; enable interrupt (device specific — illustrative)
    bx  lr

    .thumb_func
TRNG_IRQHandler:
    ; read word to stir entropy pool (demo only)
    ldr r0, =TRNG_STAT
    ldr r1, [r0]
    cbz r1, 9f
    ldr r0, =TRNG_DATA
    ldr r2, [r0]
    ; cheap health: avoid stuck-at (compare to previous, stored in _trng_prev)
    ldr r0, =_trng_prev
    ldr r3, [r0]
    eors r3, r3, r2
    cbz r3, 9f                    ; identical consecutively? ignore
    str r2, [r0]
9:  bx lr

    .bss
_trng_prev: .space 4

; ------------ NFC -------------------------------------------
    .text
    .thumb_func
NFC_Init:
    ; enable RX interrupts
    ldr r0, =NFC_INTEN
    movs r1, #1
    str r1, [r0]
    bx  lr

    .thumb_func
NFC_IRQHandler:
    push {r4-r7,lr}
    ; check RX ready
    ldr  r0, =NFC_INTSTAT
    ldr  r1, [r0]
    tst  r1, #1
    beq  8f

    ; read APDU length (first byte) safely
    ldr  r2, =NFC_RXFIFO
    ldr  r3, =_rx_buf
    movs r4, #0

    ldrb r5, [r2]                 ; LEN
    cmp  r5, #0
    beq  7f
    cmp  r5, #0xF0                ; cap length
    bhi  7f

    str  r5, [r3, #-4]            ; store to _rx_len via r3-4 trick
    ldr  r6, =_rx_len
    str  r5, [r6]

    ; copy LEN bytes
1:  cmp  r4, r5
    bhs  2f
    ldrb r6, [r2]
    strb r6, [r3, r4]
    adds r4, r4, #1
    b    1b
2:
    ; parse & handle
    bl   APDU_Handle

    ; if tx_len>0, write reply
    ldr  r0, =_tx_len
    ldr  r1, [r0]
    cbz  r1, 8f
    ldr  r2, =NFC_TXFIFO
    ldr  r3, =_tx_buf
    movs r4, #0
3:  cmp  r4, r1
    bhs  4f
    ldrb r5, [r3, r4]
    strb r5, [r2]
    adds r4, r4, #1
    b    3b
4:  ; done
    movs r0, #0
    ldr  r2, =_tx_len
    str  r0, [r2]
8:  pop  {r4-r7,lr}
    bx   lr
7:  ; invalid length: drop
    pop  {r4-r7,lr}
    bx   lr

; ------------ APDU parser (constant-time core) --------------
; Buffer: _rx_buf[LEN], _rx_len; Response in _tx_buf/_tx_len
    .thumb_func
APDU_Handle:
    push {r4-r7,lr}
    ; default: no reply
    movs r0, #0
    ldr  r1, =_tx_len
    str  r0, [r1]

    ldr  r2, =_rx_len
    ldr  r3, [r2]
    cbz  r3, 9f

    ldr  r4, =_rx_buf
    ldrb r5, [r4]                 ; CLA
    ldrb r6, [r4, #1]             ; INS

    ; verify CLA == APDU_CLA (constant-time mask)
    movs r7, #APDU_CLA
    eors r7, r7, r5               ; r7=0 if equal
    subs r7, r7, #1
    sbcs r7, r7, r7               ; r7 = 0xFFFFFFFF if equal else 0
    ; branchless select based on INS
    ; if INS==DONATE -> handle donate
    movs r0, #APDU_INS_DONATE
    eors r0, r0, r6
    subs r0, r0, #1
    sbcs r0, r0, r0               ; r0=~0 if equal

    ; if INS==GETREC -> handle get recognition
    movs r1, #APDU_INS_GETREC
    eors r1, r1, r6
    subs r1, r1, #1
    sbcs r1, r1, r1               ; r1=~0 if equal

    ; Mask with CLA check (r7)
    ands r0, r0, r7
    ands r1, r1, r7

    ; Conditional calls
    cbz  r0, 1f
    bl   APDU_Handle_DONATE
1:  cbz  r1, 2f
    bl   APDU_Handle_GETREC
2:
9:  pop {r4-r7,lr}
    bx  lr

; --- DONATE: amount[4] || sig[64] (toy) ---------------------
; On success: append record & set recognition-ready.
    .thumb_func
APDU_Handle_DONATE:
    push {r4-r7,lr}
    ldr  r0, =_rx_len
    ldr  r1, [r0]
    cmp  r1, #70+2                ; CLA,INS + amount(4)+sig(64)
    bne  8f

    ldr  r2, =_rx_buf
    adds r2, r2, #2               ; skip CLA,INS
    ; r2 -> amount[0]
    ; Verify signature (stub but constant-time pattern)
    mov  r0, r2                   ; msg ptr
    add  r1, r2, #4               ; sig ptr
    bl   Verify_Donation_Signature_Ct
    cbz  r0, 8f                   ; 0 => fail

    ; Append to log
    mov  r0, r2                   ; amount ptr
    bl   Flash_Append_Record
    cbz  r0, 8f                   ; 0 => append fail

    ; Set recognition ready
    ldr  r1, =_recog_ready
    movs r2, #1
    str  r2, [r1]

    ; Reply: 0x9000
    ldr  r3, =_tx_buf
    movs r4, #0x90
    movs r5, #0x00
    strb r4, [r3]
    strb r5, [r3, #1]
    ldr  r6, =_tx_len
    movs r7, #2
    str  r7, [r6]
8:  pop  {r4-r7,lr}
    bx   lr

; --- GET_RECOG: return NDEF "Thanks: #<id>" -----------------
    .thumb_func
APDU_Handle_GETREC:
    push {r4-r7,lr}
    ldr  r0, =_recog_ready
    ldr  r1, [r0]
    cbz  r1, 7f

    ldr  r2, =_tx_buf
    ; copy prefix
    ldr  r3, =_ndef_prefix
    movs r4, #_ndef_prefix_len
1:  subs r4, r4, #1
    blt  2f
    ldrb r5, [r3, r4]
    strb r5, [r2, r4]
    b    1b
2:
    ; append decimal of _seq_counter
    ldr  r6, =_seq_counter
    ldr  r6, [r6]
    movs r7, #0
3:  ; convert to decimal (reverse) — quick loop
    mov  r0, r6
    movs r1, #10
    bl   udiv10                   ; r0=quot, r1=rem
    add  r1, r1, #'0'
    strb r1, [r2, r7+_ndef_prefix_len]
    adds r7, r7, #1
    mov  r6, r0
    cbnz r6, 3b

    ; total length
    adds r7, r7, #_ndef_prefix_len
    ldr  r0, =_tx_len
    str  r7, [r0]

    ; clear flag
    ldr  r0, =_recog_ready
    movs r1, #0
    str  r1, [r0]
7:  pop {r4-r7,lr}
    bx  lr

; ------------ Flash Append (log with seq + CRC32) -----------
; Input: r0 = amount ptr (4 bytes)
; Output: r0 = 1 on success, 0 on fail
    .thumb_func
Flash_Append_Record:
    push {r4-r7,lr}
    ; Build record in _tx_buf: [seq(4)|amount(4)|crc(4)]
    ldr  r1, =_seq_counter
    ldr  r2, [r1]
    adds r3, r2, #1
    str  r3, [r1]                 ; increment seq

    ldr  r4, =_tx_buf
    str  r3, [r4]                 ; seq
    ldr  r5, [r0]                 ; amount
    str  r5, [r4, #4]

    ; CRC32 over seq|amount (8B)
    mov  r0, r4                   ; data ptr
    movs r1, #8                   ; len
    bl   CRC32
    str  r0, [r4, #8]

    ; "Write to flash": copy into mirror at offset (seq % pages)*12
    ; (Demo only. Real code would erase/program pages and verify.)
    ldr  r6, =_flash_mirror
    ; offset = (seq % (FLASH_SIZE/12)) * 12
    ldr  r7, =FLASH_SIZE/12
    ; cheap modulo (seq < 2^32 OK):
    udiv r2, r3, r7               ; r2 = seq / N
    mul  r2, r2, r7
    subs r2, r3, r2               ; r2 = seq % N
    movs r7, #12
    mul  r2, r2, r7
    add  r6, r6, r2
    ; memcpy 12 bytes
    movs r7, #12
1:  subs r7, r7, #1
    blt  2f
    ldrb r2, [r4, r7]
    strb r2, [r6, r7]
    b    1b
2:  movs r0, #1
    pop {r4-r7,lr}
    bx  lr

; ------------ CRC32 (bitwise, poly 0xEDB88320) --------------
; r0=data ptr, r1=len -> r0=crc
    .thumb_func
CRC32:
    movs r2, #0xFF
    mvns r2, r2           ; r2 = 0xFFFFFFFF
    mov  r3, r2           ; crc
1:  cbz  r1, 3f
    ldrb r4, [r0], #1
    eors r3, r3, r4
    movs r5, #8
2:  tst  r3, #1
    beq  4f
    lsrs r3, r3, #1
    eor  r3, r3, #0xEDB88320
    b    5f
4:  lsrs r3, r3, #1
5:  subs r5, r5, #1
    bne  2b
    subs r1, r1, #1
    b    1b
3:  mvns r0, r3           ; ~crc
    bx   lr

; ------------ Constant-time memcmp (returns 0 if equal) -----
; r0=a, r1=b, r2=len -> r0=result
    .thumb_func
memcmp_ct:
    movs r3, #0
1:  cbz  r2, 2f
    ldrb r4, [r0], #1
    ldrb r5, [r1], #1
    eors r4, r4, r5
    orrs r3, r3, r4
    subs r2, r2, #1
    b    1b
2:  mov  r0, r3
    bx   lr

; ------------ Secure Boot Verify (stubbed) ------------------
    .thumb_func
SecureBoot_Verify:
    ; Compute hash (omitted); compare to _fw_hash_expected constant-time
    ; Demo: pretend we computed _fw_hash_computed in RAM
    ldr  r0, =_fw_hash_computed
    ldr  r1, =_fw_hash_expected
    movs r2, #32
    bl   memcmp_ct
    ; if result==0 ok, else blink LED fast (halt)
    cbz  r0, 1f
0:  bl   LED_Blink_Once
    b    0b
1:  bx   lr

    .bss
_fw_hash_computed: .space 32

; ------------ ECDSA Verify (stub, constant-time skeleton) ---
; r0=msg ptr (amount[4]), r1=sig ptr (64B) -> r0=1 ok, 0 fail
    .text
    .thumb_func
Verify_Donation_Signature_Ct:
    push {r4-r7,lr}
    ; Toy acceptance: signature equal to 64 zeroes + amount echoed?
    ; (Replace with real constant-time big-int ops)
    mov  r4, r1
    movs r5, #64
    movs r6, #0
1:  ldrb r7, [r4], #1
    orrs r6, r6, r7
    subs r5, r5, #1
    bne  1b
    ; r6==0 if all zeroes
    cmp  r6, #0
    it   eq
    moveq r0, #1
    it   ne
    movne r0, #0
    ; zeroize temp regs is implicit; zeroize sig (demo)
    mov  r4, r1
    movs r5, #64
2:  movs r7, #0
    strb r7, [r4], #1
    subs r5, r5, #1
    bne  2b
    pop {r4-r7,lr}
    bx  lr

; ------------ Unsigned div by 10 helper ---------------------
; Input r0=value -> r0=quot, r1=rem
    .thumb_func
udiv10:
    ; reciprocal multiply trick for speed is omitted; use library call if available.
    movs r2, #0
    movs r1, #0
1:  cmp  r0, #10
    blo  2f
    subs r0, r0, #10
    adds r2, r2, #1
    b    1b
2:  mov  r1, r0
    mov  r0, r2
    bx   lr

; ------------ End -------------------------------------------

```
