---
layout: post
title: "Recreating the NYC Subway Countdown Clocks: A Deep Dive into My Development Process"
date: 2025-05-09
tags: [NYC Subway, Raspberry Pi, GTFS, Python, PySide, Transit Tech]
---

For many New Yorkers, the countdown clocks on subway platforms are more than just digital signage — they’re a small but critical reassurance that the train *is* coming. Inspired by this subtle piece of urban infrastructure, I set out to recreate my own version of the NYC subway countdown clocks: a live, self-contained display that shows upcoming train arrivals based on real-time [MTA GTFS data](https://new.mta.info/developers).

This post walks through my development process — from understanding the system’s architecture to building a working replica using a Raspberry Pi, real-time data, and a custom PySide interface.

---

## 📐 Inspiration and Goals

I wanted a physical device that looked and behaved like the real thing. That meant:

- A **high-contrast, wide-aspect display**
- **Real-time train arrival data** via the MTA’s public feeds
- A **minimalist, subway-authentic UI**
- A self-contained unit that could run 24/7 on a shelf or wall

![Real NYC Subway Countdown Clock](images/nyc-subway-clock-real.jpg)

---

## 🧰 Hardware Selection

I used a [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) for its balance of power and flexibility. For the display, I chose a [7.9" 400x1280 IPS screen](https://www.waveshare.com/wiki/7.9inch_HDMI_LCD) that matched the ultra-wide ratio of the subway signs. The display connects via HDMI and is powered by USB.

![Hardware Setup](images/hardware-setup.jpg)

---

## 🧠 Software Stack

The interface is written in Python using [PySide6](https://doc.qt.io/qtforpython/) for a native Qt GUI. This gave me smooth rendering and full control over layout and font rendering.

### Components:

- **MTA Feed Parser**: The MTA’s data comes in [GTFS-Realtime](https://developers.google.com/transit/gtfs-realtime/) format. I used [`gtfs-realtime-bindings`](https://pypi.org/project/gtfs-realtime-bindings/) to parse protobuf messages.
- **Trip Filtering Logic**: I filtered `TripUpdate` messages by route and stop ID, then calculated ETA from the `arrival.time` and `current system time`.
- **Custom Display Renderer**: The GUI dynamically updates every second with fresh data, mimicking the MTA’s “X MIN” format.

![Countdown Clock UI](images/ui-screenshot.jpg)

---

## 💡 Challenges

### GTFS Feed Interpretation

Stop IDs are cryptic (e.g., `R16N` for Union Square Northbound). I used the MTA’s static GTFS feed to map IDs to station names and platforms. See the [GTFS Static dataset](http://web.mta.info/developers/developer-data-terms.html#data) for stop locations and route metadata.

### Accurate Timekeeping

The Pi’s clock can drift if NTP is disabled or Wi-Fi is spotty. I ensured NTP was always enabled and added logic to fall back gracefully if the time source was ever lost.

### Visual Fidelity

I used a bold sans-serif font (similar to [Helvetica Neue Condensed Bold](https://fonts.adobe.com/fonts/helvetica-neue)) and precise padding/margins to match MTA signage. Font sizing and alignment required many iterations to “feel” right.

![Side-by-Side Comparison](images/side-by-side-comparison.jpg)

---

## 🖼️ The Final Result

The finished device displays the next two arrivals for a selected station and route in real time. It looks like something pulled straight off an L train platform, but runs silently in your kitchen or office.

![Final Product on Wall](images/final-mounted.jpg)

---

## 🔜 What’s Next?

I'm planning to expand the project with:

- A web-based or touchscreen config UI
- Multiple station views
- Automatic brightness based on time of day
- Optional voice announcements using [espeak-ng](https://github.com/espeak-ng/espeak-ng)

I’m also considering publishing the code as open-source once I finalize the setup script and documentation.

---

## 🧠 Learn More & Get Involved

If you’re into transit tech, data visualization, or embedded systems, I’d love to connect. Here are some helpful links:

- [MTA GTFS Developer Portal](https://new.mta.info/developers)
- [GTFS-Realtime Python Bindings](https://pypi.org/project/gtfs-realtime-bindings/)
- [PySide6 Docs](https://doc.qt.io/qtforpython/)
- [Waveshare HDMI Displays](https://www.waveshare.com/wiki/Main_Page)

---

## 📬 Let’s Talk

Have questions, want to build one yourself, or thinking about mounting this in your shop or studio? Drop a comment or [email me](mailto:your@email.com). I’d be happy to walk through the setup or collaborate.

---

