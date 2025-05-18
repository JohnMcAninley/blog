---
layout: post
title: "Designing a Raspberry Pi HAT in KiCAD"
date: 2025-05-18
categories: [hardware, raspberry-pi, kicad, electronics]
---

Designing your own Raspberry Pi HAT (Hardware Attached on Top) is a great way to extend the Pi’s capabilities and create custom solutions tailored to your project. In this post, we’ll walk through the process of designing a Pi HAT from schematic to PCB using [KiCAD](https://kicad.org/), a powerful open-source EDA tool.

## 📐 Step 1: Understand the HAT Specification

Before diving into design, get familiar with the [Raspberry Pi HAT specification](https://github.com/raspberrypi/hats). Key points include:

- 40-pin GPIO header with a specific pinout.
- EEPROM for automatic hardware detection (optional but recommended).
- Mechanical dimensions for alignment with Pi mounting holes.

## 🧰 Step 2: Set Up Your Project in KiCAD

Start a new KiCAD project:

1. Launch KiCAD and create a new project.
2. Name it something like `custom_pi_hat`.
3. Create a schematic and PCB layout file.

### Import Pi Header Footprint

Use the 40-pin header available in the standard KiCAD libraries (`Connector_PinHeader_2.54mm`). Make sure to match the Raspberry Pi GPIO pinout.

## 🔌 Step 3: Schematic Design

Add components:

- 40-pin GPIO header.
- Any ICs, sensors, or modules your HAT needs.
- Bypass capacitors and supporting passives.
- Optional: EEPROM circuit for device tree overlays.

Use KiCAD’s electrical rule checker (ERC) to catch mistakes early.

## 🧲 Step 4: Assign Footprints

Match your schematic symbols to the correct footprints:

- Use the KiCAD footprint editor to preview or adjust pads.
- Align connector footprints (e.g., GPIO) precisely with Raspberry Pi layout.

## 🧾 Step 5: PCB Layout

Time to switch to the board editor:

1. Load the netlist or use the integrated schematic-PDB sync.
2. Arrange components and place the GPIO header first.
3. Use the official Raspberry Pi HAT mechanical drawing to align mounting holes.

### Keep in Mind:

- Use 2.54mm spacing and 3.3V logic levels.
- Avoid blocking HDMI/USB ports unless necessary.
- Use mounting holes (2.75mm drill) with M2.5 clearance.

## 📏 Step 6: Route Traces

- Use wider traces for power.
- Match impedance for high-speed signals.
- Consider a ground plane on the bottom layer.

Use DRC (Design Rule Check) regularly.

## 🎯 Step 7: Generate Gerbers and BOM

Once your layout is complete:

- Generate Gerber files (`File > Plot`).
- Export drill files.
- Create a Bill of Materials (BOM) using the BOM plugin or scripting tools.

## 🧪 Optional: Add an EEPROM

For full HAT compliance, add a small I2C EEPROM (e.g., 24C32):

- Connected to GPIO 0 and 1 (I2C).
- Use 3.3V pull-up resistors.
- Follow the [EEPROM specification](https://github.com/raspberrypi/hats/blob/master/eeprom.md) to program it.

## 🏁 Conclusion

You now have a custom Pi HAT ready for fabrication! KiCAD’s open-source ecosystem and the clear HAT standards make it relatively easy to design hardware that integrates cleanly with the Raspberry Pi.

Stay tuned for future posts on prototyping and programming the EEPROM!

---

*Have a question or want to share your own Pi HAT project? Leave a comment below or open an issue!*
