> **Heart Rate & Body Temperature Monitoring For Remote Patients**
> 
> Department of Electrical & Electronic Engineering · Bangladesh University of Engineering & Technology (BUET)

![MATLAB](https://img.shields.io/badge/MATLAB-2016a-e16737?style=flat-square&logo=mathworks&logoColor=white)
![Arduino](https://img.shields.io/badge/Arduino-UNO-00979D?style=flat-square&logo=arduino&logoColor=white)
![RF](https://img.shields.io/badge/Wireless-RF_Module-6f42c1?style=flat-square)
![Sensors](https://img.shields.io/badge/Sensors-DHT11_%7C_IR_Pulse-39d353?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-39d353?style=flat-square)

A low-cost, portable wireless patient monitoring system that measures **heart rate (BPM)** and **body temperature (°C)** in real time using embedded sensors, RF transmission, and MATLAB signal processing — designed to assist remote patient care in hospital and home settings.

-----

## 📌 Abstract

Bangladeshis are experiencing heart attacks approximately 10 years earlier than typical sufferers in western countries. Around 40% of all cases occur in people under 50. Constant cardiovascular monitoring is critical yet often unaffordable. This project addresses that gap with a wireless, low-cost device that captures vital signs and transmits them to a doctor or caregiver anywhere in the hospital using RF technology.

-----

## ⚙️ System Pipeline

```
[Heart Rate Sensor]  ──┐
                        ├──► [Arduino UNO] ──► [RF Transmitter] ~~wireless~~► [RF Receiver] ──► [Arduino UNO] ──► [PC] ──► [MATLAB GUI]
[DHT11 Temp Sensor]  ──┘
```

### Step-by-step

1. **Sensing** — IR pulse sensor and DHT11 temperature sensor continuously read the patient’s vitals.
1. **Processing** — Arduino UNO microcontroller reads both sensor outputs and packages the data.
1. **Wireless Transmission** — Packaged data is sent via an RF transmitter module.
1. **Reception** — A second Arduino connected to the receiver RF module captures the incoming signal.
1. **Post-Processing** — Data is fed into MATLAB 2016a where EMD-based signal processing extracts the heart rate.
1. **Display** — Final BPM and temperature readings appear on a MATLAB GUI.

-----

## 🔬 Methodology

### Heart Rate Estimation — Empirical Mode Decomposition (EMD)

Raw PPG (photoplethysmography) signals are non-stationary and non-linear. EMD decomposes the signal into a set of **Intrinsic Mode Functions (IMFs)**:

```
y(n) = Σ(k=1 to N) [ s_k(n) + r_k(n) ]
```

- **1000 data points** collected at a **500 Hz** sampling rate
- PPG signal decomposed into IMFs and analysed in the spectral domain
- **Welch method** used for power spectral estimation
- IMFs with frequency peaks in the range **0.5–3 Hz** are selected and reconstructed
- Reconstructed signal is further processed to extract the final heart rate (BPM)

> **Complete Ensemble EMD (CEEMD)** was used over standard EMD to resolve “mode mixing” — the presence of very similar oscillations across different IMF modes.

### Temperature Measurement — DHT11 Sensor

The DHT11 communicates with the MCU via a single-wire protocol:

1. MCU pulls data line **LOW for ≥18 ms** (Start signal)
1. MCU pulls **HIGH for 20–40 µs**, then releases
1. DHT11 responds: **LOW 80 µs → HIGH 80 µs**
1. Sensor transmits **40 bits (5 bytes)** of data:

```
Data (40-bit) = [RH Integer] + [RH Decimal] + [Temp Integer] + [Temp Decimal] + [Checksum]

Checksum = Last 8 bits of (RH_Int + RH_Dec + Temp_Int + Temp_Dec)
```

Bit encoding:

- `0` → line HIGH for **26–28 µs** after 50 µs LOW
- `1` → line HIGH for **70 µs** after 50 µs LOW

-----

## 🩺 Sensors

### A. Heart Beat Sensor (IR-based, Finger-strap type)

|Property           |Detail                                                  |
|-------------------|--------------------------------------------------------|
|Type               |High-intensity IR reflectance sensor                    |
|Principle          |Blood-volume pulse changes IR reflectance in capillaries|
|Form factor        |Finger-strap clip                                       |
|Signal conditioning|Op-amp amplification (very low amplitude raw signal)    |

### B. Temperature Sensor — DHT11

|Property            |Specification                            |
|--------------------|-----------------------------------------|
|Supply voltage      |3–5.5 V                                  |
|Temperature range   |0–50 °C                                  |
|Temperature accuracy|±2 °C                                    |
|Humidity range      |20–95% RH                                |
|Humidity accuracy   |±5% RH                                   |
|Sampling rate       |Max 1 Hz (1 sample/second)               |
|Current draw        |2.5 mA max during conversion             |
|Body size           |15.5 mm × 12 mm × 5.5 mm                 |
|Interface           |Single-wire digital (4-pin, 0.1” spacing)|

-----

## 📡 RF Module

Wireless data transmission between the patient-side Arduino and the PC-side Arduino is handled by an RF module pair.

|Module         |Description                                                                                                      |
|---------------|-----------------------------------------------------------------------------------------------------------------|
|**Transmitter**|Small PCB subassembly; modulates and transmits data on a radio carrier wave alongside the patient-side Arduino   |
|**Receiver**   |Demodulates the received RF signal; superheterodyne type used for stability across voltage and temperature ranges|


> Superheterodyne receivers were chosen over super-regenerative for their **fixed crystal design**, providing better frequency stability and accuracy.

-----

## 🔧 Hardware — Arduino UNO

|Feature         |Specification                 |
|----------------|------------------------------|
|Microcontroller |ATmega328                     |
|Digital I/O pins|14 (6 PWM capable)            |
|Analog inputs   |6                             |
|Clock speed     |16 MHz ceramic resonator      |
|Interface       |USB, ICSP header              |
|Power           |USB or AC-DC adapter / battery|

-----

## 📊 Results

Signal processing pipeline output:

- **Raw PPG signal** decomposed into 8 IMFs + residual
- IMFs in the **0.5–3 Hz** cardiac frequency band isolated and reconstructed
- **Peak-picking algorithm** applied to detect heartbeat peaks in the reconstructed signal
- Results displayed live in MATLAB GUI

**Sample GUI output:**

```
┌──────────────────────────────────────────┐
│         ENTER PATIENT NUMBER:  1         │
│                                          │
│   MEASURE HEART RATE   MEASURE TEMP      │
│                                          │
│        83.28 BPM        38.0 °C          │
│       HEART RATE      TEMPERATURE        │
└──────────────────────────────────────────┘
```

-----

## 🌟 Significance

|Feature              |Benefit                                               |
|---------------------|------------------------------------------------------|
|📶 Wireless (RF)      |No line-of-sight required; works across hospital wards|
|💰 Low cost           |Affordable for developing-country healthcare settings |
|🔋 Low power          |Minimal battery drain for portable use                |
|📦 Portable           |Small form factor; wearable by patients               |
|🏃 Freedom of movement|Patient not tethered to bedside equipment             |
|🧑‍⚕️ Remote monitoring  |Doctor can view readings from anywhere in the hospital|

-----

## 🚀 Future Scope

- [ ] Add **ECG and blood pressure** monitoring
- [ ] Integrate **pulse oximeter** (SpO₂)
- [ ] Add **Galvanic Skin Resistance** (stress detection)
- [ ] **GPS integration** — automatically notify nearest hospital and dispatch ambulance in emergencies
- [ ] **Auto-call doctor** when vitals exceed threshold
- [ ] Improve RF anti-jamming and data integrity

-----

**Supervised by:**

- Dr. Md. Aynal Haque *(Professor, EEE, BUET)*
- Dr. Mohammed Imamul Hassan Bhuiyan *(Professor, EEE, BUET)*

-----


Report: <a href="https://drive.google.com/uc?export=download&id=1Vni5WGoBOu9bfa8YaN72HKMB3xVqpbAe" download>
        Download PDF
        </a>
