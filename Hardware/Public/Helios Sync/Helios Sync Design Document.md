<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: CC-BY-4.0
-->

# Helios Sync Design Document

*A Physical-Layer Mechanism for Global Clock Synchronization and Lightweight Control Signaling Based on Light Pulses*

---

> v0.2 (Independent)

---

## 1. Overview

Helios Sync is a physical-layer mechanism that utilizes **visible light pulses** as global synchronization and lightweight control signaling. Its core concept is: **in static, line-of-sight deployment environments, leveraging the constant speed of light and the measurable nature of geometric paths to transform the clock synchronization problem into an offline calibration problem**.

The system requires only a single master light source (which may be a dedicated LED lamp or integrated into the lighting system) to cover all DPU nodes via direct line-of-sight or mirror reflection. Each DPU integrates a miniature photosensitive sensor to detect light pulses and trigger local clock alignment or state response.

This solution combines ultra-low latency, zero wiring cost, high robustness, and multi-purpose scalability, making it particularly suitable for DPU interconnect architectures centered on "memory bus extension".

---

## 2. System Components

### 2.1 Light Emitter
- **Type**: High-brightness white LED or infrared LED (invisible to human eye)
- **Location**: Fixed installation
- **Output Capabilities**:
  - Pulse rise time < 10 ns
  - Supports frequency modulation (1 Hz – 10 kHz)
  - Programmable blinking sequences (information encoding)
- **Safety Constraints**:
  - Minimum pulse interval ≤ 1000 ms

> **Safety Design Principle**: The minimum time required for any physical change (such as replacing/adjusting mirrors) must span multiple pulse periods to effectively capture external condition changes and prevent physical security issues (such as break-in crimes).

### 2.2 Passive Mirrors
- **Material**: High-reflectance plane mirrors (aluminum-coated or dielectric film)
- **Purpose**: Bypass cabinet obstructions and extend coverage range
- **Key Characteristics**:
  - Passive, zero-delay, no power supply required
  - Position of each mirror recorded during deployment
- **Path Modeling**: Each optical path (direct or via N reflections) is treated as an independent propagation channel; total distance = Σ geometric length of each segment

### 2.3 DPU Opto-Sync Block
Integrated within the DPU ASIC package, comprising:
- Photodiode (PD) + Transimpedance Amplifier (TIA)
- Band-pass filter (configurable center frequency)
- Edge detector + Digital decoder
- Programmable delay compensation unit (Delay Line)
- State machine controller

---

## 3. Deployment and Calibration Process

### 3.1 Static Topology Calibration (One-time)
1. Fix the positions of all DPU cabinets and reflection mirrors.
2. Use a laser rangefinder to measure **the total length of each valid optical path from the light source to each DPU**, denoted as $L_i$ (unit: meters).
3. Calculate the theoretical propagation delay:
   $$
   \Delta t_i = \frac{L_i}{c} \quad (c = 3 \times 10^8  \text{m/s})
   $$
4. Write $\Delta t_i$ into the DPU's non-volatile memory (eFUSE/OTP).
5. The DPU automatically loads this value upon power-on to configure the internal delay line.

> **Result**: All DPUs, after receiving the same light pulse, achieve **local clock alignment error < 1 ns** (limited only by device jitter).

### 3.2 Dynamic Calibration (Optional)
- Emit a synchronization pulse every 10–100 ms to compensate for crystal oscillator drift.
- If long-term drift exceeds the limit (>10 ns), trigger an alarm.

---

## 4. Synchronization and Signaling Protocol

### 4.1 Basic Synchronization Mode
- **Pulse Characteristics**: 1 kHz square wave, duration 1 ms (i.e., one period)
- **Trigger Actions**:
  - Latch local clock phase
  - Reset PLL/CDR loop filter
  - Update global timestamp (for logs, snapshots, etc.)

### 4.2 Robustness Assurance: Frequency Verification
- Only when 3 consecutive cycles matching the preset frequency (e.g., 1 kHz ± 1%) are detected is the event considered a valid synchronization.
- This prevents false triggers from ambient light (sunlight, fluorescent lamps).

### 4.3 Multi-purpose Signaling Encoding
Control commands are transmitted by varying blinking frequency or sequence:

| Frequency / Mode        | Meaning               | Response Behavior |
|--------------------|--------------------|--------|
| 1 Hz (slow flash)       | System Sleep           | DPU enters low-power mode |
| 10 Hz              | Normal Operation           | Maintain current state |
| 100 Hz             | High Load Alert         | Preheat XPU, prefetch cache |
| 1 kHz (Standard Sync)  | Clock Alignment           | Execute synchronization action |
| 5 kHz + 10 ms Burst| Emergency Shutdown           | Safe memory flush, power-off |
| Specific Pseudo-random Sequence     | Firmware Update Trigger       | Enter bootloader |

> **Encoding Principle**: All frequencies are >200 Hz (imperceptible to human eye) to avoid interference with the working environment.

---

## 5. Small-Scale Temporary Deployment Support

For laboratories, demonstrations, and edge temporary clusters (≤8 nodes):

### 5.1 Mobile App Features
- Name: **Helios Mobile**
- Platform: iOS / Android
- Capabilities:
  - Screen-off (reduce ambient light interference)
  - Control phone flash to blink at specified frequency
  - Select preset modes (Sync / Sleep / Boot)
  - Display current DPU response status (via Bluetooth/WiFi feedback)

### 5.2 Usage Flow
1. User turns off room lights, positions phone ≥1 meter from DPU (avoid near-field saturation).
2. Open the app and select "Temporary Synchronization".
3. The app operates the phone flash to emit pulses, completing initialization alignment.
4. Subsequent control commands can be sent via the app.

> Zero hardware cost, rapid deployment, suitable for PoC verification.

---

## 6. Security and Reliability Design

| Risk | Mitigation Measure |
|------|--------|
| Accidental mirror movement | DPU monitors signal amplitude mutations and triggers alarms |
| Light source failure | Optional backup light source (e.g., second lamp); DPU degrades to local crystal oscillator mode after N missed syncs |
| Malicious light interference | Dual verification via frequency + sequence; only accepts pre-registered codes |
| Ambient light noise | Band-pass filtering + digital correlation detection; adaptive thresholding |

---

## 7. Performance Metrics

| Metric | Target Value |
|------|-------|
| Synchronization Accuracy (post-calibration) | < 1 ns RMS |
| Coverage Radius | ≥ 50 m (with mirrors) |
| Maximum Node Count | No theoretical limit (broadcast nature) |
| Power Consumption (DPU side) | < 200 μW |
| Deployment Calibration Time | < 5 minutes per cabinet |

---

## 8. Summary

Helios Sync transforms determinism from the physical world (constant speed of light, fixed positions) into simplicity of system design, achieving:

- Zero-cable global synchronization
- Natural broadcast, unlimited scalability
- Multi-purpose control signaling
- Ultimate low cost and high robustness

It is not merely a clock synchronization solution, but an engineering practice of the **"Physics-as-an-API"** concept, providing an elegant foundation for future decentralized, deterministic interconnect architectures.

---

**Appendix A**: Typical deployment schematic (omitted)  
**Appendix B**: Opto-Sync Block RTL interface definition (omitted)  
**Appendix C**: Helios Mobile APP UI prototype (omitted)