---
layout: post
title: "DIY PCB Panelization in KiCAD for JLCPCB: Why and How I Do It"
date: 2025-04-19
author: John McAninley
categories: [pcb, kicad, jlcpcb]
tags: [panelization, kicad, kikit, jlcpcb, fabrication-toolkit]
---

# DIY PCB Panelization in KiCAD for JLCPCB: Why and How I Do It

If you’re like me and use **JLCPCB** for PCB manufacturing (and sometimes assembly), you’ve probably run into their **automatic panelization option**. It's convenient—they'll panelize your design for free—but only with **v-cuts**, and you don’t get much control over the panel layout. For many use cases, that's totally fine. But for others, like mine, it’s a bit limiting.

For example, if you want to use **JLCPCB’s “economical assembly” service**, it makes a difference *how* the board is panelized. At the time of writing, JLCPCB charges **$25 for panelized boards using standard assembly**, versus **$8 for single-board economic assembly**. That might not be a dealbreaker for a commercial project or a decent production run, but for a hobbyist or someone ordering a bunch of small runs, it adds up fast.

Also? I like doing it myself. At first, I wanted to see if I could. Now that I’ve taken the time to learn the workflow, it’s no big deal—it’s fast, repeatable, and it gives me complete control over the result. That said, you might try it and decide it’s totally worth the $17 to let JLCPCB take care of it. But if you’re up for a little DIY, here’s how I approach it.

---

## Tool of Choice: KiKit

KiCAD doesn’t have built-in panelization support. Instead, I use a plug-in called [KiKit](https://github.com/yaqwsx/KiKit). It’s actively maintained, well documented, and has a **command-line interface (CLI)** and a **Python API** if you really want to get fancy.

Personally, I’ve never needed more than the CLI.

### Installing KiKit

Installation is straightforward. On **Windows**, you’ll want to run it from the **KiCAD Command Prompt** (not your regular terminal). On Linux/macOS, install it like any Python package:

```bash
pip install kikit
```

## Panelizing Your Board
The basic idea: you run KiKit on your finished board layout, and it outputs a new, panelized version according to your rules.

For example, a simple 3x2 grid with mouse bites might look like:
```bash
kikit panelize grid --rows 3 --cols 2 \
    --spacing 2mm \
    --tabs full \
    --mousebites 0.5mm 1mm \
    input.kicad_pcb output.kicad_pcb
```
### Workflow Tip

The main hiccup I ran into? KiCAD doesn’t like it when the file is open while KiKit modifies it. So you’ll need to **close your `.kicad_pcb` file before running KiKit**, then **reopen it** to view the output. Slightly annoying, but not a dealbreaker.

One feature I’d love to see: the ability to run KiKit panelization directly from within KiCAD—similar to how the **Fabrication Toolkit** works. Another would be having the panel update automatically when the source board changes. *Chef’s kiss.*

---

## JLCPCB-Specific Requirements

If you're sending panelized boards to **JLCPCB** for assembly, be aware of the following manufacturing requirements:

- **Fiducials**:  
  - Diameter: **1mm**  
  - Solder mask opening: **2mm**  
  - Position: **3.85mm from the edge**

- **Tooling holes**:  
  - Diameter: **2mm**  
  - Typically placed in each corner of the panel

- **Keep-out areas and spacing**:  
  - Ensure your design respects board-edge and clearance constraints defined by JLCPCB  
  - Make sure your overall panel size stays within their assembly limits

Also, **run a Design Rule Check (DRC)** on the panelized layout—*not just the original board*. It’s easy to introduce new clearance or mechanical issues during panelization that weren’t present in the single-board design.

---

## Exporting for Production

Once your panelized board passes all checks, it’s time to generate files for manufacturing.

If you haven’t already, I **highly recommend the [Fabrication Toolkit](https://github.com/INTI-CMNB/KiCad-Fabrication-Toolkit)** plug-in. It simplifies the process of creating all the necessary files for JLCPCB, including:

- Gerber files  
- Bill of Materials (BOM)  
- Component Placement List (CPL)

You can install it through KiCAD’s built-in **Plugin Manager**. Once installed, it’s just a few clicks to export everything you need.

---

## Final Thoughts

Panelizing your own boards adds a bit of up-front effort, but it can be well worth it—especially if you're doing:

- Small production runs  
- Projects with tight budgets  
- Iterative prototyping  
- Anything that benefits from full layout control

With **KiKit** and **Fabrication Toolkit**, the workflow is pretty smooth once you've done it once or twice. Even if you decide later that JLCPCB's automated panelization is worth the small cost, it's great to understand what's going on behind the scenes.

Happy panelizing!
