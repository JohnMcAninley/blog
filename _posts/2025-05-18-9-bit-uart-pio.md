---
title: Implementing a 9-Bit UART in PIO on the RP2040
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

## Alternative Solutions
### 8-Bit
One option was to update the firmware on all existing devices on the system to use 8-bit data frames with an STX byte appended to the beginning of each frame. However, this would require a far more involved installation process for the new accessory as each device would need to be updated. 

This would also forfeit the benefits of the 9-bit protocol. Each device would need to use CPU time for ~6 times as much bus traffic including the additional message framing. This would have a negative performance impact on the single-core ATMega32U4 based boards, some of which are doing time critical tasks such as motor control. Additionally, while the 9th-bit address flag is completely unique in framing the beginning of a message, a STX byte will likely collide with some value appearing in the data of a message. Therefore, any matching value would need to be escaped.

### UART IC
I conducted a brief search for a standalone UART IC that supported 9-bit data frames but I failed to find anything. Even if I had found a suitable candidate, this implementation would have still required writing some additional firmware to interact with the standalone UART and would have marginally increased the BOM and routing complexity.

### Parity Bit Stuffing
In theory it is possible to use the parity bit as a 9th data bit as it occurs in the same location in the bitstream. For example, if you want to send an address and the address has odd parity, you would set the UART mode to even parity so the parity bit is set. 

Receiving would require handling parity errors as well since some valid data would appear to the UART to have incorrect parity. 

Besides the complexities of such an approach, the existing system was configured to use odd parity so I was unable to commandeer the parity bit.

### Alternative MCU
[AVR Microcontrollers Peripheral Guide](https://ww1.microchip.com/downloads/en/DeviceDoc/30010135E.pdf)

### Software UART
A 9-bit UART could be emulated in software. However, the system baud rate of 250kbps is above what is generally considered attainable with standard “bit-banging” software serial implementation. The RP2040, however, has a subsystem that falls between software and hardware that can implement a 9-bit UART: PIO. 


## Why Use PIO for UART?

RP2040’s **PIO subsystem** lets you define custom logic-level protocols using tiny state machines that run in parallel with the CPU. For non-standard UART configurations like 9-bit, PIO allows:

- Precise bit timing at arbitrary baud rates.
- Control over frame structure and logic levels.
- CPU offloading and deterministic execution.

---
![UART_Frame svg](https://github.com/user-attachments/assets/331cef55-f8f9-432a-ba48-182db0a039dc)
AmenophisIII, CC0, via Wikimedia Commons

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
