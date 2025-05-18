---
title: Implementing a 9-Bit UART with RP2040 PIO for RS485 Compatibility
date: 2025-05-18
layout: post
author: John McAninley
---

While modern embedded systems often embrace Ethernet, CAN, or USB, many legacy networks still rely on **RS485** and **9-bit UART protocols** for robust, multi-drop serial communication. One such protocol is MDB (Multi-Drop Bus), but 9-bit UARTs also appear in custom industrial RS485 networks where the 9th bit is used to differentiate **address** and **data** frames.

My motivation for this project came from developing a new accessory for an **existing RS485 network that uses 9-bit data frames**. Ensuring **backward compatibility** was critical. However, unlike older microcontrollers like the ATmega32U4, the Raspberry Pi Pico’s RP2040 doesn’t natively support 9-bit UART. That’s where the RP2040’s **Programmable I/O (PIO)** subsystem becomes a game-changer.

This post walks through how I implemented a 9-bit UART transmitter and receiver using RP2040 PIO to support legacy 9-bit RS485 communication.

---

## UART Framing 101

**UART (Universal Asynchronous Receiver/Transmitter)** is a simple, widely-used serial protocol. A typical UART frame includes:

- **Start Bit (1 bit):** LOW to signal start of transmission.
- **Data Bits (5–9 bits):** Payload, usually 8 bits.
- **Parity Bit (optional):** Basic error detection (even/odd).
- **Stop Bits (1–2 bits):** HIGH to signal end of frame.

In **9-bit UART**, a 9th data bit is added to distinguish address frames from data frames in protocols like MDB. This is especially useful for RS485 multi-drop systems where each slave listens for an address frame before accepting data.

---

## Why Use PIO for UART?

RP2040’s **PIO subsystem** lets you define custom logic-level protocols using tiny state machines that run in parallel with the CPU. For non-standard UART configurations like 9-bit, PIO allows:

- Precise bit timing at arbitrary baud rates.
- Control over frame structure and logic levels.
- CPU offloading and deterministic execution.

---

## Transmitting 9-Bit UART Frames

```Assembly
.program uart_tx_9bit
.side_set 1

.pull_block
set x, 10
set pins, 0       side 0 [baud - 1]  ; Start bit

bitloop:
    out pins, 1   side 0 [baud - 1]
    jmp x--, bitloop

set pins, 1       side 1 [baud - 1]  ; Stop bit
```

The host code pushes a 9-bit value (1 address/data bit + 8-bit payload):

```c
Copy
Edit
uint16_t frame = (is_address << 8) | payload;
pio_sm_put_blocking(pio, sm_tx, frame);
```

## Receiving 9-Bit UART Frames
```Assembly
.program uart_rx_9bit
.wait 0 pin
set x, 9

bitloop:
    in pins, 1   [baud - 1]
    jmp x--, bitloop

push block
```
Host-side interpretation:
```c
uint16_t frame = pio_sm_get_blocking(pio, sm_rx);
bool is_address = (frame >> 8) & 1;
uint8_t data = frame & 0xFF;

if (is_address) {
    listening = (data == my_address);
} else if (listening) {
    process_data(data);
}
```
## Timing Considerations
To ensure proper UART timing, calculate bit cycles using the system clock:
```c
#define SYSTEM_CLOCK 125000000
#define BAUD 9600
#define CYCLES_PER_BIT (SYSTEM_CLOCK / BAUD)
```
Use this in PIO instruction delay slots like \[CYCLES_PER_BIT - 1\].

## Next Steps

### 1. Offload Address/Data Filtering to PIO

Currently, the host code separates address and data frames. This logic can be moved into the PIO program:

- Discard frames that don't match our address
- Trigger interrupts only for relevant frames

This reduces CPU load and increases system responsiveness.

---

### 2. Add Parity Bit Support in PIO

To support protocols requiring parity:

- **TX side:** Compute parity in software or add it in PIO.
- **RX side:** Validate parity inside the PIO state machine before pushing data.

This improves robustness for noisy or long cable runs.

---

### 3. Decouple Baud Rate from PIO Clock

Avoid tying PIO execution speed to the system clock frequency. Instead, use the PIO state machine’s clock divider:

```c
float clkdiv = (float) SYSTEM_CLOCK / (BAUD * bits_per_frame);
pio_sm_set_clkdiv(pio, sm, clkdiv);
```

This allows multiple PIO programs to run independently at different rates.

### 4. Create MicroPython Bindings
Make this work accessible in MicroPython:

- Wrap initialization and state machine loading
- Expose read() / write() for 9-bit UART frames
- Add address filtering and parity options

This enables prototyping and experimentation without writing C code.

## Conclusion
By leveraging the RP2040’s PIO subsystem, we can implement 9-bit UART protocols that would otherwise require dedicated UART hardware. This makes it possible to build modern accessories that remain compatible with legacy RS485 networks—a common requirement in industrial and vending applications.

As this project matures, it can become a flexible library for supporting a wide range of UART configurations, including MDB and other vendor-specific extensions.

## Resources
- [RP2040 Datasheet – PIO] (https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)
- [MDB Protocol Overview (NAMA)] (https://www.nayax.com/wp-content/uploads/2018/03/MDB-Specs.pdf)
