# Scenario 8: Crypto Charity Wallet — Recognition Firmware (Fun Demo)

This is a **didactic ARM Cortex‑M4 (Thumb‑2) assembly** skeleton that implements the core mechanics of Scenario 8:

- Accept a **DONATE** APDU (amount + signature), verify in **constant‑time** (stubbed), append a donation record to a flash‑like log with **CRC32**, and set a **recognition‑ready** flag.
- Upon **GET_RECOG**, return a tiny **NDEF-like** payload: `Thanks: #<id>` where `<id>` is a monotonic sequence counter (record ID).

> ⚠️ This is **for fun / learning**. MMIO addresses and drivers are illustrative. Real hardware needs true flash drivers, NFC stack, TRNG health tests, and proper ECDSA.

## Files
- `wallet_fw_thumb.s` — fully commented assembly source.
- (No linker script is included; you’d normally provide `_estack`, `_sbss`, `_ebss` via a standard Cortex‑M linker script.)

## Build (example)
```bash
arm-none-eabi-as -mcpu=cortex-m4 -mthumb wallet_fw_thumb.s -o wallet_fw_thumb.o
arm-none-eabi-ld -Ttext=0x08000000 wallet_fw_thumb.o -o wallet_fw_thumb.elf
arm-none-eabi-objcopy -O ihex wallet_fw_thumb.elf wallet_fw_thumb.hex
```

## How it maps to the scenario
- **Low-level control:** direct **MMIO** for TRNG, NFC, GPIO; interrupt handlers for **NFC APDUs** and TRNG readiness.
- **Security primitives:**
  - `memcmp_ct` for constant‑time comparisons.
  - `Verify_Donation_Signature_Ct` shows the calling convention and zeroization pattern (replace with real ECDSA).
  - `SecureBoot_Verify` demonstrates constant‑time firmware hash checking (replace with real hashing).
- **Durability:** `Flash_Append_Record` maintains a compact log `[seq|amount|crc]` with simple wear‑leveling via modulo slotting (replace with real page erases and verify‑after‑write).

## Try it in a simulator
You can single‑step the control flow and validate that:
- `APDU_Handle_DONATE` sets `_recog_ready` and increments `_seq_counter`.
- `APDU_Handle_GETREC` builds a response in `_tx_buf` that contains `Thanks: #<id>`.

## Extending to real hardware
- Replace NFC stubs with your chip’s **ISO14443** or **NFC-A** peripheral driver.
- Implement true **ECDSA (secp256r1)** using constant‑time big‑int (Montgomery multiplication) or your MCU’s crypto accelerator.
- Swap the flash mirror for actual **erase/program** routines and verify via readback + CRC.
- Add **anti‑replay** (monotonic counters from RTC/secure element) and **tamper** inputs for lockout.

Have fun and hack safely!
