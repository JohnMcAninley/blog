---
title: Implementing a 9-Bit UART in PIO on the RP2040
date: 2025-05-18
layout: post
author: John McAninley
---

While modern embedded systems often embrace Ethernet or CAN for communication, many legacy and industrial systems still rely on RS485. A differential signal physical layer specification, RS485 allows multiple devices to be connected to the same bus. In order to reduce distribution to devices on the bus, systems using RS485 can utilize a serial protocol which includes a 9th bit to differentiate between address and data. Hardware support for this functionality allows each device to reduce its CPU load by filtering out irrelevant data until it reads its own address. One such protocol that uses this approach is MDB (Multi-Drop Bus), commonly used in vending machines. 

I took over a project to develop a new accessory for an existing system that also used such an approach. The system consists of distributed embedded devices communicating over an RS485 bus. The serial communication on the bus is 250kbps with 9-bit data, odd parity, and 1 stop bit (9odd1). 

While the existing devices were all based on the ATMega32U4, the project had already elected to use an RP2040 for this new accessory. It was necessary to select a chip other than the ATMega32U4 as at least 1 more UART was needed for this project. The decision to choose a non-AVR microcontroller was due to issues sourcing the ATMega32U4, especially during the pandemic. However, this choice failed to account for at least one difference and limitation of the RP2040: the ARM Primecell PL011 UART’s only support data sizes between 5 and 8 bits. That’s where the RP2040’s Programmable Input/Output (PIO) peripheral comes in handy.

---

## PIO Primer

The PIO (Programmable Input/Output) peripheral is a somewhat unique feature of the Raspberry Pi RP2040 microcontroller. Each of the two PIO blocks contains a 32-word instruction memory as well as four state machines to execute small, user-defined programs written in a simple assembly-like language. PIO excels at implementing custom, timing critical I/O handling with minimal CPU intervention. Ideal uses include custom protocols, PWM generation, precisely timed LED control, precise timing control, and more. This is particularly useful when requirements exceed the capabilities of the standard onboard peripherals.

---

## UART Basics
UART (Universal Asynchronous Receiver/Transmitter) refers to a common hardware peripheral and often to the serial protocol it implements as well. The asynchronous nature of the protocol means that there is no accompanying clock signal with the data, instead it is sent and received at a specific rate (baud rate) agreed upon by the transmitter and receiver. There is also a small amount of framing applied to the data.

![UART_Frame svg](https://github.com/user-attachments/assets/331cef55-f8f9-432a-ba48-182db0a039dc)

AmenophisIII, CC0, via Wikimedia Commons
- **Start Bit (1 bit):** LOW to signal start of transmission. The line is HIGH when IDLE.
- **Data Bits (5–9 bits):** Payload, commonly 8 bits.
- **Parity Bit (optional):** Basic error detection (even/odd).
- **Stop Bits (1–2 bits):** HIGH to signal end of frame.

UART data is most commonly sent least significant bit (LSB) first, so a 9th bit comes immediately before the parity or stop bit.

---

## Alternative Solutions
While I settled on implementing a 9-bit UART using PIO for this project, a number of other solutions were considered but decided against for various reasons. I am including them as one or more may be a better solution in different circumstances.

### 8-Bit
One option was to update the firmware on all of the system’s existing devices to use 8-bit data frames and add additional message framing to replace the 9th bit address flag that begins each message. However, this would require a far more involved installation process for the new accessory as each device would need to be updated.

This would also forfeit the benefits of the 9-bit protocol. Each device would need to use CPU time to process all bus traffic (including additional message framing), not just addresses and applicable data, a ~6x increase for this system. This would have a negative performance impact on the single-core ATMega32U4 based boards, some of which are doing time critical tasks such as motor control. Additionally, while the 9th-bit address flag is completely unique in framing the beginning of a message, a STX byte will likely collide with some value appearing in the data of a message. Therefore, any matching value would need to be escaped.

### UART IC
I conducted a brief search for a standalone UART IC that supported 9-bit data frames but I failed to find anything. Even if I had found a suitable candidate, this implementation would have still required writing some additional firmware to interact with the standalone UART and would have marginally increased the BOM and routing complexity.

### Parity Bit Stuffing
In theory it is possible to use the parity bit as a 9th data bit as it occurs in the same location in the bitstream. For example, if you want to send an address and the address has odd parity, you would set the UART mode to even parity so the parity bit is set. Receiving would require handling parity errors in addition to data since some valid data would appear to the UART to have incorrect parity. 

Besides the complexities of such an approach, the existing system was configured to use odd parity so I was unable to commandeer the parity bit.

### Alternative MCU
When I joined the project, boards using the RP2040 had already been designed and prototyped, so I continued on with the RP2040. However, I ended up later needing to redesign these boards, so in retrospect I would have switched the project back to an AVR microcontroller that met the project requirements. While the PIO UART implementation was quick and painless, choosing a similar microcontroller with the same architecture would have required less code to be ported and would have resulted in a cleaner codebase.

[AVR Microcontrollers Peripheral Guide](https://ww1.microchip.com/downloads/en/DeviceDoc/30010135E.pdf)

The above chart provides an overview of the capabilities of all AVR microcontrollers. I believe using the AtMega32U2, which is extremely similar to the existing AtMega32u4, but has 2 USART’s, would’ve been the most appropriate choice.

### Software UART
A 9-bit UART could be emulated in software. However, the system baud rate of 250kbps is above what is generally considered attainable with standard “bit-banging” software serial implementation. The RP2040, however, has a subsystem that falls between software and hardware that can implement a 9-bit UART: PIO. 

---


## Implementation

### Transmitter

```Assembly
.program uart_tx
.side_set 1 opt

; OUT pin 0 and side-set pin 0 are both mapped to UART TX pin.

    pull       side 1 [7]  ; Assert stop bit, or stall with line in idle state
    set x, 9   side 0 [7]  ; Preload bit counter, assert start bit for 8 clocks
bitloop:                   ; This loop will run 10 times (9 Odd 1 UART)
    out pins, 1            ; Shift 1 bit from OSR to the first OUT pin
    jmp x-- bitloop   [6]  ; Each loop iteration is 8 cycles.
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

---

## Next Steps

### 1. Offload Address/Data Filtering in PIO

Currently, the host code optionally filters out data frames. However, this filtering should occur in PIO to reduce CPU intervention. Toggling this setting must be communicated from the CPU to PIO somehow.

In theory, the PIO program could be aware of the device address and optionally filter out address frames as well. However, this is not behavior seen in hardware UART’s and is likely too complex for a PIO program. 

### 2. Add Parity Checking in PIO

The parity bit can continue to be calculated by the CPU on the TX side. However, the parity of received frames should be calculated and checked in PIO on the RX side to avoid the CPU having to deal with invalid data. If users want to be notified of parity errors and still check invalid data, this change wouldn’t make sense. 

### 3. Create MicroPython Bindings

MicroPython is another very popular way to program the RP2040. While the PIO program can remain the same, the host code must either be exposed via MicroPython bindings or reimplemented in pure MicroPython.

### 4. Oversample RX
Standard UART’s typically oversample the RX data at a rate of 16. This allows them to account for slight deviations of the transmitter and receiver clocks. The PIO implementation could be modified to oversample as well. 

## Conclusion
By leveraging the RP2040’s PIO subsystem, we can implement 9-bit UART protocols that would otherwise require dedicated UART hardware. This makes it possible to build modern accessories that remain compatible with legacy RS485 networks—a common requirement in industrial and vending applications.

As this project matures, it can become a flexible library for supporting a wide range of UART configurations, including MDB and other vendor-specific extensions.

## Resources
- [RP2040 Datasheet – PIO](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)
- [MDB Protocol Overview (NAMA)](https://www.nayax.com/wp-content/uploads/2018/03/MDB-Specs.pdf)
- Pico Examples: PIO UART [TX](https://github.com/raspberrypi/pico-examples/tree/master/pio/uart_tx) and [RX](https://github.com/raspberrypi/pico-examples/tree/master/pio/uart_rx)
