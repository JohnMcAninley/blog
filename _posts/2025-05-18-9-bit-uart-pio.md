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

```pio
.program uart_tx_9bit
.side_set 1

.pull_block
set x, 10
set pins, 0       side 0 [baud - 1]  ; Start bit

bitloop:
    out pins, 1   side 0 [baud - 1]
    jmp x--, bitloop

set pins, 1       side 1 [baud - 1]  ; Stop bit
