# ScatterSense-LoRAB: Turning the MYOSA Board into an Ambient FM Backscatter RFID Tag


---

## Acknowledgements

Based on our previous peer-reviewed papers "All-Digital CSS Ambient Backscatter with 17.6 uW Power Consumption and a MISO Receiver" (2026 IEEE International MTT Symposia) and "The New Era of Long-Range “Zero-Interception” Ambient Backscattering Systems: 130 m with 130 nA Front-End Consumption" (2022 Sensors Journal). Thanks to the FM-backscatter and LoRa research community, and to the open-source SDR ecosystem (GNU Radio, Airspy). This MYOSA build ports the original nRF52833 tag to the MYOSA board. Thanks to my supervisor Dr. Spyridon N. Daskalakis, who has made this work possible.

---

## Overview

LoRAB  is an ultra-low-power, battery-free wireless tag. Instead of generating its own radio signal, it **reflects existing ambient FM radio broadcasts** and modulates data onto them by switching an antenna between two impedance states (backscatter). The clever part: it produces **Chirp Spread Spectrum (CSS / LoRa-style) chirps entirely in the digital domain**, using the MYOSA board's **I2S peripheral** to drive an RF switch — no Digital-to-Analog Converter (DAC) and no Voltage-Controlled Oscillator (VCO).

**What it does:** Sends data wirelessly with no RF transmitter of its own.
**How it works:** MYOSA I2S outputs a sampled square-wave chirp → drives an ADG919 RF switch → switch toggles antenna impedance → ambient FM carrier is reflected with CSS modulation → SDR receiver decodes it.
**Who it's for:** Battery-free IoT / sensor nodes, RFID-style tags, long-lifetime deployments.
**Problem it solves:** Traditional radios burn milliwatts in the power amplifier. This tag skips the PA entirely and runs at **17.6 uW**, reaching **150 m**.

**Key features:**
* Fully-digital CSS chirp generation — no DAC, no VCO
* 17.6 uW average tag power (9.8 uA @ 1.8 V, 1% duty cycle)
* 150 m measured range
* Two-component tag: MYOSA board + RF switch
* MISO receiver using multiple ambient FM stations for robustness

---

## Demo / Examples

### Images

<p align="center">
  <img src="Figures/lorab-architecture.png" width="800"><br/>
  <i>LoRAB  system architecture: ambient FM stations excite the MYOSA tag, which backscatters I2S-generated CSS chirps to an Airspy SDR + GNU Radio receiver.</i>
</p>

<p align="center">
  <img src="Figures/tag-real-life.jpg" width="800"><br/>
  <i>Real-life tag deployment: the MYOSA-class board with RF switch and copper-wire dipole mounted at a window.</i>
</p>

<p align="center">
  <img src="Figures/digital-chirp.png" width="800"><br/>
  <i>Backscatter modulation principle. Left: two reflection-coefficient states on the Smith chart. Right: the ideal analog chirp vs the implemented digital square-wave chirp sampled by I2S.</i>
</p>

<p align="center">
  <img src="Figures/css-frame-spectrum.png" width="800"><br/>
  <i>Spectrogram of a received CSS frame showing preamble down-chirps, preamble up-chirps, and payload chirps.</i>
</p>

<p align="center">
  <img src="Figures/range-map.png" width="800"><br/>
  <i>Deployment geometry. Left: tag-to-receiver link of 150 m. Right: FM emitter (Black Hill transmitting station) 34.5 km from the tag.</i>
</p>

### Videos

<p align="center">
  <img src="Video/lorab-myosa-demo.gif" width="800">
</p>

A high-quality MP4 version with the sound of FM music is available at `Video/lorab-myosa-demo.mp4`.

---

## Features (Detailed)

### 1. All-Digital CSS Chirp Generation (no DAC / no VCO)

The MYOSA board samples an analog CSS chirp in the digital domain and stores the result as a lookup table. The **I2S peripheral** streams that bitstream out over its data line as a square-wave control signal. This toggles the RF switch between two impedance states, reflecting a CSS-modulated version of the ambient FM carrier. Prior CSS backscatter needed a DAC, a VCO, a switch mapper, and an SP8T switch — all removed here.

### 2. DMA-Driven, CPU-Asleep Operation

A Direct Memory Access (DMA) controller autonomously feeds lookup-table samples to the I2S peripheral, so the CPU stays mostly asleep during transmission. This is the main reason the tag hits **17.6 uW**. Deep-sleep current stays below 1 uA between packets.

### 3. Impedance-Modulated Backscatter Front-End

The I2S output drives the control pin of an **ADG919 SPDT switch** (VDD 1.8 V). The switch connects a handmade **1.5 m copper-wire dipole** (1.5 mm diameter) that reflects ambient FM. The two switch states give impedances Z0 ≈ 5.7 + j5.03 and Z1 ≈ 27.97 − j504.12, a reflection-coefficient distance of ΔΓ ≈ 1.785.

### 4. MISO Receiver Across Multiple FM Stations

The receiver captures a **6 MHz chunk** with an Airspy Mini SDR (~$100), isolates individual FM channels in GNU Radio, FM-stereo demodulates each, and feeds a custom correlation-based CSS decoder. Using 2–3 stations sharply lowers Bit Error Rate (BER) and protects against outages and interference.

### 5. Measured Performance

* **Range:** 150 m
* **Bit rate:** 273 bps
* **Power:** 17.6 uW (9.8 uA @ 1.8 V)
* **Best BER:** minimum at SF 7 / SF 8
* Outdoor SF 7, CR 4/8 packets decoded at SNR −10.83 dB

---

## Usage Instructions

**1. Flash the MYOSA board.** Load the CSS chirp lookup tables (upchirp + downchirp) into memory and configure the I2S peripheral + DMA to stream them to the data line.

```plaintext
# configure I2S sampling rate for chosen SF / bandwidth
# rate is programmable between 100 kHz and 16 MHz
```

**2. Wire the front-end.** I2S data pin → ADG919 control pin. ADG919 VDD = 1.8 V. Switch RF port → 1.5 m copper-wire dipole.

**3. Place the tag** near ambient FM coverage. FM broadcast band is 88–108 MHz.

**4. Run the receiver.** Airspy Mini + GNU Radio captures 6 MHz, filters per FM channel, FM-stereo demodulates, then runs the CSS decoder.

```python
# receiver pipeline (conceptual)
def decode_css(iq_6mhz):
    channels = split_fm_channels(iq_6mhz)       # frequency-isolating filters
    reals    = [fm_stereo_demod(c) for c in channels]
    packets  = css_correlate_decode(reals)      # preamble + payload chirps
    return packets
```

---

## Tech Stack

* **MYOSA board** — ultra-low-power MCU, I2S peripheral + DMA for chirp generation
* **ADG919 SPDT RF switch** — impedance-modulated backscatter front-end
* **Airspy Mini SDR** — 6 MHz capture receiver
* **GNU Radio** — channelization + FM-stereo demodulation
* **Python / MATLAB** — correlation-based CSS decoder

---

## Requirements / Installation

Receiver-side dependencies:



Install **Python**, **GNU Radio** and **Airspy** host tools (via your OS package manager). Firmware for the MYOSA board is built with the standard MYOSA / MCU toolchain.

---


## Appendix: Backscatter vs Active Bluetooth Front-End Power

A core reason this tag is battery-free is that the **backscatter front-end** does not synthesize or amplify RF. It only reflects an existing ambient carrier. Compare the RF front-end cost against an active BLE radio transmitting at +8 dBm:

| Front-end | Supply | Current | Power | Notes |
|---|---|---|---|---|
| ADG919 backscatter (RF-front-end) | 1.8 V | ~130 nA | ~0.23 uW | Passive reflection, no PA |
| BLE TX @ +8 dBm (nRF52-class radio) | ~3.0 V | ~45–50 mA | ~135–165 mW | Active PA dominates |

**Takeaway:** the backscatter front-end draws roughly **orders of magnitude less current** than an active BLE transmitter at +8 dBm (~130 nA vs ~45 mA ≈ ~350,000×). BLE must run a power amplifier to emit +8 dBm; backscatter skips the PA entirely by reusing the ambient FM carrier. The 130 nA figure is the RF-front-end only, which includes MCU I2S chirp synthesis.
