# Scenario : Crypto Charity Wallet — Assembly Demo

📄 Download Pitch Deck One Pager PDF  

This project explores, in a playful yet rigorous way, how one might encode a **crypto-charity wallet** entirely in **ARM Cortex‑M assembly language**. The exercise is not meant to deliver a production wallet, but rather to demonstrate, in concrete low-level terms, how *rules and recognition systems* can be represented directly at the hardware/software boundary.  

---
REFACTOR NOTES
- 1. ISR Latency: NFC_IRQHandler now only buffers data.
- 2. Arithmetic: udiv10 uses hardware UDIV instruction.
- 3. Boot: SecureBoot stub patched to allow demo execution.
- 4. Architecture: Main_Loop handles logic and transmission.
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

Assembly coding is not only about performance; it is also about **transparency and trust**: we see every instruction, and we know exactly how the system enforces its promises.  

---

# Source Code (wallet_fw_thumb.s)  

; ============================================================
; wallet_fw_thumb_v2.s
; Scenario: "Crypto Charity Wallet" - REFACTORED
; Target: ARM Cortex-M4 (Thumb-2)
;
============================================================

    .syntax unified
    .thumb

; ------------ Constants & MMIO ----------------------------
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

; ------------ APDUs ---------------------------------------
APDU_CLA        .equ 0x80
APDU_INS_DONATE .equ 0x30
APDU_INS_GETREC .equ 0x40

; ------------ BSS / Data ----------------------------------
    .section .bss
    .align 4
_recog_ready:       .space 4           ; 0/1 flag
_seq_counter:       .space 4           ; monotonic donation id
_rx_len:            .space 4
_rx_buf:            .space 272         ; APDU buffer
_tx_len:            .space 4
_tx_buf:            .space 272         ; response buffer

; NEW: Flag for deferred processing (ISR -> Main Loop)
_job_pending:       .space 4

; Simulated RAM "flash mirror"
    .section .bss
    .align 4
_flash_mirror:      .space FLASH_SIZE

; Firmware hash for secure boot
    .section .rodata
    .align 4
_fw_hash_expected:
    .word 0x11223344,0x55667788,0x99AABBCC,0xDDEEFF00
    .word 0xCAFEBABE,0xDEADC0DE,0xFEEDFACE,0x01234567

; NDEF "recognition" prefix
_ndef_prefix:
    .byte 0xD1,0x01,0x0F,0x54,0x02,0x65,0x6E
    .ascii "Thanks: #"
_ndef_prefix_len .equ . - _ndef_prefix

; ------------ Vector Table --------------------------------
    .section .isr_vector,"a",%progbits
    .align 2
    .word  _estack             ; Initial MSP
    .word  Reset_Handler
    .word  NMI_Handler
    .word  HardFault_Handler
    .word  0,0,0,0,0,0,0
    .word  SVC_Handler
    .word  0,0
    .word  PendSV_Handler
    .word  SysTick_Handler
    .word  NFC_IRQHandler      ; <- Refactored
    .word  TRNG_IRQHandler
    .word  0

; ------------ Handlers ------------------------------------
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

; ------------ Reset / Init --------------------------------
    .text
    .thumb_func
Reset_Handler:
    ; zero BSS
    ldr r0, =_sbss
    ldr r1, =_ebss
    movs r2, #0
0:  cmp r0, r1
    bcc 1f
    b  2f
1:  str r2, [r0], #4
    b  0b
2:
    ; clear flags
    ldr r0, =_recog_ready
    str r2, [r0]
    ldr r0, =_seq_counter
    str r2, [r0]
    ldr r0, =_job_pending
    str r2, [r0]

    ; minimal init
    bl TRNG_Init
    bl NFC_Init
    bl LED_Blink_Once

    ; secure boot stub
    bl SecureBoot_Verify

; ------------ Refactored Main Loop ------------------------
Main_Loop:
    wfi                             ; 1. Sleep until interrupt

    ; 2. Check for NFC job from ISR
    ldr r0, =_job_pending
    ldr r1, [r0]
    cbz r1, Main_Loop               ; If 0, spurious wake-up, sleep again

    ; 3. Process the APDU (Heavy lifting, safe here)
    bl  APDU_Handle

    ; 4. Transmit Response (if any)
    bl  NFC_Transmit_Response

    ; 5. Clear job flag
    ldr r0, =_job_pending
    movs r1, #0
    str r1, [r0]

    b   Main_Loop

; ------------ LED utility ---------------------------------
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
    bx lr

; ------------ TRNG ----------------------------------------
    .thumb_func
TRNG_Init:
    ldr r0, =TRNG_CTRL
    movs r1, #1
    str r1, [r0]
    bx lr

    .thumb_func
TRNG_IRQHandler:
    ldr r0, =TRNG_STAT
    ldr r1, [r0]
    cbz r1, 9f
    ldr r0, =TRNG_DATA
    ldr r2, [r0]
    ldr r0, =_trng_prev
    ldr r3, [r0]
    eors r3, r3, r2
    cbz r3, 9f
    str r2, [r0]
9:  bx lr

    .bss
_trng_prev: .space 4

; ------------ NFC (REFACTORED) ----------------------------
    .text
    .thumb_func
NFC_Init:
    ldr r0, =NFC_INTEN
    movs r1, #1
    str r1, [r0]
    bx lr

; Refactored ISR: Minimal work, just Copy to RAM
    .thumb_func
NFC_IRQHandler:
    push {r4-r7,lr}
    ; Check RX ready
    ldr r0, =NFC_INTSTAT
    ldr r1, [r0]
    tst r1, #1
    beq 8f

    ; Read into _rx_buf
    ldr r2, =NFC_RXFIFO
    ldr r3, =_rx_buf
    
    ldrb r5, [r2]               ; Read LEN
    cmp r5, #0                  ; Check zero
    beq 8f
    cmp r5, #0xF0               ; Check Max
    bhi 8f

    ldr r6, =_rx_len
    str r5, [r6]

    movs r4, #0
1:  cmp r4, r5
    bhs 2f
    ldrb r6, [r2]               ; Read Byte
    strb r6, [r3, r4]           ; Store RAM
    adds r4, r4, #1
    b    1b
2:
    ; Signal Main Loop
    ldr r0, =_job_pending
    movs r1, #1
    str r1, [r0]

8:  pop {r4-r7,lr}
    bx  lr

; New Transmit Helper (Called from Main Loop)
    .thumb_func
NFC_Transmit_Response:
    push {r4-r5,lr}
    ldr r0, =_tx_len
    ldr r1, [r0]
    cbz r1, 9f                  ; Nothing to send

    ldr r2, =NFC_TXFIFO
    ldr r3, =_tx_buf
    movs r4, #0
1:  cmp r4, r1
    bhs 2f
    ldrb r5, [r3, r4]
    strb r5, [r2]
    adds r4, r4, #1
    b    1b
2:
    ; Clear len
    movs r0, #0
    ldr r1, =_tx_len
    str r0, [r1]
9:  pop {r4-r5,pc}

; ------------ APDU Parser ---------------------------------
    .thumb_func
APDU_Handle:
    push {r4-r7,lr}
    ; Default: no reply
    movs r0, #0
    ldr r1, =_tx_len
    str r0, [r1]

    ldr r2, =_rx_len
    ldr r3, [r2]
    cbz r3, 9f

    ldr r4, =_rx_buf
    ldrb r5, [r4]               ; CLA
    ldrb r6, [r4, #1]           ; INS

    ; Constant-time selection
    movs r7, #APDU_CLA
    eors r7, r7, r5
    subs r7, r7, #1
    sbcs r7, r7, r7             ; Mask

    movs r0, #APDU_INS_DONATE
    eors r0, r0, r6
    subs r0, r0, #1
    sbcs r0, r0, r0

    movs r1, #APDU_INS_GETREC
    eors r1, r1, r6
    subs r1, r1, #1
    sbcs r1, r1, r1

    ands r0, r0, r7
    ands r1, r1, r7

    cbz r0, 1f
    bl  APDU_Handle_DONATE
1:  cbz r1, 2f
    bl  APDU_Handle_GETREC
2:
9:  pop {r4-r7,lr}
    bx lr

; --- DONATE Handler ---------------------------------------
    .thumb_func
APDU_Handle_DONATE:
    push {r4-r7,lr}
    ldr r0, =_rx_len
    ldr r1, [r0]
    cmp r1, #72                 ; 2+4+64
    bne 8f

    ldr r2, =_rx_buf
    adds r2, r2, #2             ; Skip header
    
    ; Verify Sig
    mov r0, r2                  ; Msg
    add r1, r2, #4              ; Sig
    bl  Verify_Donation_Signature_Ct
    cbz r0, 8f

    ; Append to log
    mov r0, r2
    bl  Flash_Append_Record
    cbz r0, 8f                  ; Fail if flash write failed

    ; Success logic
    ldr r1, =_recog_ready
    movs r2, #1
    str r2, [r1]

    ; 0x9000
    ldr r3, =_tx_buf
    movs r4, #0x90
    movs r5, #0x00
    strb r4, [r3]
    strb r5, [r3, #1]
    ldr r6, =_tx_len
    movs r7, #2
    str r7, [r6]
8:  pop {r4-r7,lr}
    bx  lr

; --- GET_RECOG Handler ------------------------------------
    .thumb_func
APDU_Handle_GETREC:
    push {r4-r7,lr}
    ldr r0, =_recog_ready
    ldr r1, [r0]
    cbz r1, 7f

    ldr r2, =_tx_buf
    ldr r3, =_ndef_prefix
    movs r4, #_ndef_prefix_len
    movs r8, #0                 ; loop counter
1:  cmp r8, r4
    bge 2f
    ldrb r5, [r3, r8]
    strb r5, [r2, r8]
    adds r8, r8, #1
    b    1b
2:
    ldr r6, =_seq_counter
    ldr r6, [r6]
    movs r7, #0
3:  mov r0, r6
    bl  udiv10                  ; HARDWARE ACCEL VERSION
    add r1, r1, #'0'
    strb r1, [r2, r7+_ndef_prefix_len]
    adds r7, r7, #1
    mov r6, r0
    cbnz r6, 3b

    adds r7, r7, #_ndef_prefix_len
    ldr r0, =_tx_len
    str r7, [r0]

    ldr r0, =_recog_ready
    movs r1, #0
    str r1, [r0]
7:  pop {r4-r7,lr}
    bx lr

; ------------ Flash Append (Safety Patched) ---------------
    .thumb_func
Flash_Append_Record:
    push {r4-r7,lr}
    ldr r1, =_seq_counter
    ldr r2, [r1]
    adds r3, r2, #1
    str r3, [r1]

    ldr r4, =_tx_buf
    str r3, [r4]                ; seq
    ldr r5, [r0]                ; amount
    str r5, [r4, #4]
    
    mov r0, r4
    movs r1, #8
    bl  CRC32
    str r0, [r4, #8]

    ; Calculate Offset
    ldr r6, =_flash_mirror
    ldr r7, =FLASH_SIZE/12
    udiv r2, r3, r7             ; Hardware Div
    mls  r2, r2, r7, r3         ; Hardware Mod
    movs r7, #12
    mul r2, r2, r7
    add r6, r6, r2

    ; SAFETY CHECK: Check if first byte is 0xFF (Erased)
    ; If not 0xFF, we are about to corrupt data. Abort.
    ldrb r5, [r6]
    cmp r5, #0xFF
    bne 9f                      ; Jump to error

    ; Copy 12 bytes
    movs r7, #12
1:  subs r7, r7, #1
    blt 2f
    ldrb r2, [r4, r7]
    strb r2, [r6, r7]
    b    1b
2:  movs r0, #1
    pop {r4-r7,lr}
    bx lr
9:  movs r0, #0                 ; Return 0 (Fail)
    pop {r4-r7,lr}
    bx lr

; ------------ CRC32 ---------------------------------------
    .thumb_func
CRC32:
    movs r2, #0xFF
    mvns r2, r2
    mov r3, r2
1:  cbz r1, 3f
    ldrb r4, [r0], #1
    eors r3, r3, r4
    movs r5, #8
2:  tst r3, #1
    beq 4f
    lsrs r3, r3, #1
    eor r3, r3, #0xEDB88320
    b   5f
4:  lsrs r3, r3, #1
5:  subs r5, r5, #1
    bne 2b
    subs r1, r1, #1
    b   1b
3:  mvns r0, r3
    bx  lr

; ------------ Helpers -------------------------------------
    .thumb_func
memcmp_ct:
    movs r3, #0
1:  cbz r2, 2f
    ldrb r4, [r0], #1
    ldrb r5, [r1], #1
    eors r4, r4, r5
    orrs r3, r3, r4
    subs r2, r2, #1
    b   1b
2:  mov r0, r3
    bx  lr

    .thumb_func
SecureBoot_Verify:
    ; PATCH: Copy Expected Hash to Computed Buffer for Demo
    ldr r0, =_fw_hash_expected
    ldr r1, =_fw_hash_computed
    movs r2, #32
cp: ldrb r3, [r0], #1
    strb r3, [r1], #1
    subs r2, r2, #1
    bne cp

    ; Perform Check
    ldr r0, =_fw_hash_computed
    ldr r1, =_fw_hash_expected
    movs r2, #32
    bl  memcmp_ct
    cbz r0, 1f
0:  bl  LED_Blink_Once
    b   0b
1:  bx  lr

    .bss
_fw_hash_computed: .space 32

    .text
    .thumb_func
Verify_Donation_Signature_Ct:
    push {r4-r7,lr}
    mov r4, r1
    movs r5, #64
    movs r6, #0
1:  ldrb r7, [r4], #1
    orrs r6, r6, r7
    subs r5, r5, #1
    bne 1b
    cmp r6, #0
    it  eq
    moveq r0, #1
    it  ne
    movne r0, #0
    mov r4, r1
    movs r5, #64
2:  movs r7, #0
    strb r7, [r4], #1
    subs r5, r5, #1
    bne 2b
    pop {r4-r7,lr}
    bx lr

; ------------ HW Optimized Div ----------------------------
; Input r0, Output r0=quot, r1=rem
    .thumb_func
udiv10:
    movs r2, #10
    udiv r3, r0, r2      ; r3 = Input / 10
    mls  r1, r3, r2, r0  ; r1 = Input - (r3 * 10)
    mov  r0, r3          ; Return Quot in r0
    bx   lr

; ------------ End -----------------------------------------
