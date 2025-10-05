# EMG → Signal Processing → Mechanical Actuation (System Draft)

Below is a concrete, implementation‑oriented block diagram and the key engineering choices to get a working prototype that can scale.

---

## High‑Level Block Diagram

```
 ┌───────────────┐      ┌──────────────────┐     ┌──────────────────────┐     ┌────────────────────┐
 │  sEMG Sensors │──►──▶│  Analog Front‑End│──►──│  ADC (if external)   │──►──│  Microcontroller   │
 │  (3–8 ch)     │      │  (HP/LP/Notch,   │     │  or MCU SAR/DFSDM    │     │  + DSP / Features  │
 │  wet/dry Ag/AgCl)    │  Rectify/Buffer) │     │  (≥1 kS/s/ch, 12–24b) │     │  + ML inference    │
 └─────┬─────────┘      └───────┬──────────┘     └──────────┬───────────┘     └─────────┬──────────┘
       │                         │                             │                            │
       │                         │                             │                            │
       │                ┌────────▼─────────┐                   │                    ┌───────▼───────────┐
       │                │  Analog Isol./   │                   │                    │ Motor/Servo Ctrl │
       │                │  Clean Power     │                   │                    │ (PWM/CAN/RS‑485) │
       │                └────────┬─────────┘                   │                    └────────┬──────────┘
       │                         │                             │                            │
       │                   ┌─────▼─────────────────────────────▼────────────────────────────▼─────┐
       │                   │     Control Stack on MCU (closed‑loop, 50–200 Hz command rate)      │
       │                   │  • Windowing (100–200 ms)  • RMS/Envelope/AR features  • Classifier  │
       │                   │  • Motion Planner / Finger Mapper  • Safety/Watchdog                 │
       │                   └──────────────────────────────────────────────────────────────────────┘
       │                                                                                           
       │                         ┌───────────────────── Telemetry / Sync ─────────────────────┐
       │                         │ BLE 5 / USB‑C / UART  • Config • Model/version • Logs      │
       │                         └────────────────────────────────────────────────────────────┘
       │
 ┌──────▼────────┐                                         ┌───────────────┐           ┌──────────────┐
 │ IMU (hand)    │◄─feedback (motion tracking)─────────────│  Actuators    │◄──────────│  Motor Driver│
 │ Flex sensors  │                                         │ (tendons/etc.)│           │  (H‑bridge)  │
 └───────────────┘                                         └───────────────┘           └──────────────┘
```

**Mechanical/EE interfaces**
- **Sensors ↔ AFE:** shielded twisted pairs, right‑angle strain relief, snap connectors.
- **AFE ↔ MCU:** SPI (external ADC), or direct to MCU ADC/DFSDM.
- **MCU ↔ Actuation:** PWM (hobby servos), CAN/RS‑485 (brushless/industrial), or I²C for smart servos.
- **Feedback:** flex sensors in each finger and/or hand IMU (9‑DoF) to close the loop and reject EMG false positives.

---

##  Signal‑Processing Pipeline (tunable by sensor choice)

**Per channel:**
1. **Analog filtering:** HP 20–30 Hz (motion artifact), LP 450–500 Hz, Twin‑T 50/60 Hz notch; gain 500–2,000.
2. **Digitize:** ≥1 kS/s/ch (better 2 kS/s/ch) @ 12–24‑bit.
3. **Windowing:** 128–256 samples (64–128 ms) with 50% overlap.
4. **Features (compute‑light):** RMS/envelope, MAV, WL, ZC, SSC, AR(4), log energy, per‑channel normalization.
5. **Classifier:**
   - **On‑device:** TFLite‑Micro small MLP/CNN (multi‑label for finger groups) or linear SVM.
   - **Host model:** stream features to laptop/phone model; MCU runs a thin planner.
6. **Finger mapping:** decode to 5‑finger intents or continuous force levels (0–100%).
7. **Motion planner:** ramp/hold/decay profiles, rate limits, dead‑man switch.

**Why this split?** It lets us start simple (RMS + thresholds) and upgrade to learned models without changing the hardware.

---

##  Organizing 5 Fingers of Movement

- **Channel layout options:**
  - **8‑channel forearm placement** (FDS, FDP, EDC, FPL, lumbricals): better separation for thumb vs fingers.
  - **Armband (8 ch circumferential)** for fast don/doff, consistent geometry.
- **Decoding approaches:**
  - **Multi‑label classification** → {Thumb, Index, Middle, Ring, Little} active/inactive + strength class.
  - **Regression** → per‑finger continuous activation (0–1) for proportional control.
- **Feedback fusion:** combine EMG intent with **flex sensor angles** + **IMU** to enforce safe motion and reject artifacts.

---

## Microcontroller (recommended) & Reasoning

**Primary choice: _ESP32‑S3_ (Xtensa, 240 MHz, BLE 5, USB‑OTG)**
- Integrated BLE for phone/host sync, plenty of RAM for TFLite‑Micro, I²S/ADC for multi‑kS/s sampling.
- USB‑C DFU + logging; inexpensive and widely supported.

**Alternatives:**
- **nRF52840** (ultra‑low power BLE; lower CPU headroom, but great for wearables).
- **STM32F4/F7 or H7** (DFSDM + timers; best ADC determinism; add separate BLE module).
- **Teensy 4.1** (600 MHz headroom; hobby‑friendly, but add BLE + careful analog layout).

---

## EMG Sensor Options (board‑level)

1. **MyoWare 2.0 (Advancer Technologies)** – single‑ended sEMG, onboard gain/envelope; quick to prototype; fewer channels, envelope limits raw access.
2. **BioAmp EXG Pill v2 (Upside Down Labs)** – low‑noise INA‑based front‑end; exposes raw biopotential; stackable for multi‑ch.
3. **OpenBCI Ganglion (4‑ch BLE) / Cyton (8‑ch)** – higher resolution ADS1299‑class front‑ends, built‑in BLE/USB; bulkier but research‑grade.
4. **TI ADS129x AFE + Ag/AgCl electrodes (custom)** – best for scaling (24‑bit delta‑sigma, integrated PGAs, lead‑off detect); needs custom PCB.
5. **Delsys Trigno / OYMotion gForcePro (commercial)** – polished multi‑channel wearables; costly but robust.

**Recommendation:**
- **Phase 1 (fast demo):** 4–8× **BioAmp EXG Pill** + ESP32‑S3 internal ADC @ 2 kS/s/ch (or MCP3564 ext. ADC for 6–8 ch).
- **Phase 2 (upgrade):** **ADS1299 (8‑ch)** front‑end over SPI; move to STM32 if we need DFSDM and deterministic sampling.

---

## Interface(s)

- **Companion app (iOS/Android or desktop):**
  - Calibration wizard (per‑finger MVC capture), channel check, notch/HPF toggles.
  - Profile storage (per user), real‑time plots, start/stop, emergency stop.
  - Model management (upload model.tflite + JSON schema) and versioning.
- **On‑device:** tri‑color LED + buzzer; hardware **E‑stop** input; button for pairing/cal.

---

## Syncing ML Model ↔ Microcontroller

- **Data contract:**
  - `model.json`: input rate, window length, feature list/order, output semantics.
  - `model.tflite`: quantized INT8; checksum & version.
- **Transport:** BLE GATT service with characteristics: `CFG`, `MODEL`, `CMD`, `LOG`, `RAW`; or USB CDC for high‑rate.
- **Runtime:** double‑buffer features; run inference at 50–200 Hz; time‑stamp all packets; watchdog on missed frames.

---

##  Actuation Path

- **For tendon‑driven glove (recommended):**
  - **Servos (5–6×) or BLDCs** with planetary gearboxes.
  - MCU outputs **PWM**; driver/ESC supplies 6–12 V rail.
  - **Safety:** current limiting, soft‑limits from flex sensors, timeout if EMG silent, manual E‑stop.

---

##  Power Architecture & Options

- **Separate analog/digital domains:**
  - **Analog AFE:** isolated/clean 3.3 V (LDO from a pre‑reg buck; RC/π filters; star ground to avoid motor noise).
  - **Digital/MCU:** 3.3 V buck.
  - **Motors:** 2‑cell Li‑ion (7.4 V) or dedicated 12 V rail.
- **Battery choices:**
  - **Wearable pack:** 1S 2000–3000 mAh Li‑ion → buck (MCU) + boost (servos=6 V) OR 2S for 6–8 V servos.
  - **Charging:** BQ24074 (1S), fuel‑gauge MAX17048, protection DW01/FS8205.
- **Rough runtime calc (example):** MCU+AFE 120 mA + 5 servos avg 300 mA each (duty) → ~1.6 A avg. With 1S 3000 mAh → ~1.5–1.7 h. Scale pack to target runtime.

---

##  Possible negatives about this idea

- **Physiology:** electrode placement drift, sweat/skin impedance, inter‑user variability, crosstalk between muscles.
- **Signal:** motion artifacts <20 Hz, mains (50/60 Hz) hum, EMI from motors; need mechanical isolation and good cabling.
- **Latency:** windowing + inference + actuation must stay <100 ms for natural control.
- **Safety:** pinch forces; must enforce soft/angle limits and watchdogs.
- **Regulatory/safety (if clinical):** isolation, leakage currents, biocompatibility of electrodes.

---

## Bill‑of‑Materials (starter set)

- **Sensors:** 4–8× BioAmp EXG Pill (or MyoWare 2.0 if envelope‑only is OK), disposable **Ag/AgCl** electrodes, pre‑gelled; shielded leads.
- **MCU:** ESP32‑S3 module (e.g., ESP32‑S3‑WROOM‑1), USB‑C dev board.
- **ADC (optional ext.):** MCP3564 (24‑bit, 153.6 kS/s, 4‑ch) or ADS1299 (8‑ch, 24‑bit) for Phase 2.
- **Feedback:** 5× flex sensors (or hall‑based joint encoders), 1× 9‑DoF IMU (ICM‑20948).
- **Drivers/Actuators:** 5× high‑torque servos (or BLDC+ESC), 6–12 V supply, tendon routing.
- **Power:** 1S/2S Li‑ion pack, BQ24074 charger, buck (TPS62130), LDO (TPS7A20‑Q1) for AFE; e‑stop switch.

---

##  By Oct 6 (minimum functional demo)

1. **4‑ch prototype** with BioAmp EXG Pill → ESP32‑S3 internal ADC @ 2 kS/s/ch.
2. **Firmware** sampling + RMS/envelope + threshold classifier → PWM to 1–2 servos.
3. **Phone/desktop app**: live EMG plot, calibration (MVC), start/stop, E‑stop.
4. **Safety loop** with 1 flex sensor and timeout.

This path proves end‑to‑end detection → processing → actuation, then we iterate to 5‑finger control and the ADS1299 front‑end.

