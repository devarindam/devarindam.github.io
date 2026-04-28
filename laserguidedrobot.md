## 🤖 Laser Guided Robot

> **EEE 402 — Control System Sessional · BUET · Group 03 · Level 4 Term 1**

![Python](https://img.shields.io/badge/Python-2.7-3776AB?style=flat-square&logo=python&logoColor=white)
![Arduino](https://img.shields.io/badge/Arduino-UNO-00979D?style=flat-square&logo=arduino&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-2.78-5C3EE8?style=flat-square&logo=opencv&logoColor=white)
![Bluetooth](https://img.shields.io/badge/Bluetooth-HC--05-0082FC?style=flat-square&logo=bluetooth&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-39d353?style=flat-square)

A **closed-loop control system** that autonomously follows a laser pointer using real-time computer vision, Bluetooth serial communication, and differential motor control.

-----

## 📌 Objective

Build a practical closed-loop system in the form of a laser-following wheeled robot. The robot detects the position of a red laser pointer through a mounted smartphone camera, processes the video feed via computer vision on a PC, and steers itself to track the laser in real time — changing direction as the laser moves.

-----

## ⚙️ System Pipeline

```
📱 Android Camera  →  🖥️ PC (Python + OpenCV)  →  📡 Bluetooth (HC-05)  →  🔧 Arduino UNO  →  🚗 DC Motors
   IP Webcam app        Centroid detection           Serial command            L298 driver        100 RPM × 2
```

### Step-by-step

1. **Video Capture** — The Android app *IP Webcam* streams live video to a local IP (e.g. `http://192.168.0.105:8080/video`) over Wi-Fi.
1. **Image Processing** — Python reads the MJPEG stream, converts each frame to HSV, masks the red laser dot, and computes the centroid coordinates `(cx, cy)`.
1. **Command Generation** — Based on the centroid position within the 800×480 frame, a single character command is generated.
1. **Bluetooth Transmission** — The character is sent from PC to the HC-05 module on the Arduino via serial Bluetooth (`COM7`, 9600 baud).
1. **Motor Control** — Arduino receives the character and drives the L298 motor controller to steer the robot accordingly.

-----

## 🕹️ Motion Control Logic

Video resolution: **800 × 480 px**

|Char|Motion            |Pixel Condition                |Motor Action             |
|:--:|------------------|-------------------------------|-------------------------|
|`u` |Move Forward      |`cx < 400` AND `220 < cy < 260`|Both motors same speed   |
|`l` |Turn Left         |`cx < 400` AND `cy > 260`      |Slow left motor slightly |
|`r` |Turn Right        |`cx < 400` AND `cy < 220`      |Slow right motor slightly|
|`e` |Hard Left (pivot) |Laser exits left of frame      |Stop left motor entirely |
|`f` |Hard Right (pivot)|Laser exits right of frame     |Stop right motor entirely|
|`s` |Stop              |`cx >= 650`                    |Both motors off          |


> **Searching Mode:** If the laser disappears from view, the robot pauses, recalls the last known direction, rotates to search, and resumes tracking once the laser reappears.

-----

## 🐍 Image Processing (Python + OpenCV)

```python
import cv2, urllib2, numpy as np, time, serial

stream = urllib2.urlopen('http://192.168.0.105:8080/video')
bluetooth = serial.Serial("COM7", 9600)  # update COM port as needed

while True:
    # Decode MJPEG frame
    # ...frame extraction logic...

    # Convert BGR to HSV and isolate red laser dot
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    lower_red = np.array([160, 100, 100])
    upper_red = np.array([180, 255, 255])
    mask = cv2.inRange(hsv, lower_red, upper_red)

    # Find contours and compute centroid
    image, contours, hierarchy = cv2.findContours(
        mask, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE
    )
    if len(contours) > 0:
        M = cv2.moments(contours[pos])
        if M['m00'] != 0:
            cx = int(M['m10'] / M['m00'])
            cy = int(M['m01'] / M['m00'])

    # Send motion command via Bluetooth
    if abs(cy - 240) <= 20 and cx < 400:
        bluetooth.write(b'u')   # forward
    elif (cy - 240) > 20 and cx < 400:
        bluetooth.write(b'l')   # left
    elif (cy - 240) < -20 and cx < 400:
        bluetooth.write(b'r')   # right
    # ... additional conditions for 'e', 'f', 's'
```

> ⚠️ Update the IP address and COM port to match your local network and PC Bluetooth settings.

-----

## 🔧 Arduino Motor Control (C++)

```cpp
#include <SoftwareSerial.h>
SoftwareSerial BTserial(2, 3);

// Motor 1 pins
int ena = 5, outPin = 4, outPin2 = 6;
// Motor 2 pins
int enb = 9, outPin4 = 10, outPin3 = 11;

void loop() {
    if (BTserial.available()) {
        char bt = BTserial.read();

        if (bt == 'u') {                        // Forward
            analogWrite(ena, 120); analogWrite(enb, 120);
        } else if (bt == 's') {                 // Stop
            analogWrite(ena, 0);  analogWrite(enb, 0);
        } else if (bt == 'r') {                 // Right
            analogWrite(ena, 120); analogWrite(enb, 90);
        } else if (bt == 'l') {                 // Left
            analogWrite(ena, 90);  analogWrite(enb, 120);
        } else if (bt == 'e') {                 // Hard Left
            analogWrite(ena, 0);   analogWrite(enb, 120);
        } else if (bt == 'f') {                 // Hard Right
            analogWrite(ena, 120); analogWrite(enb, 0);
        }
    }
}
```

-----

## 🔩 Hardware Components

|Component       |Specification                     |
|----------------|----------------------------------|
|Microcontroller |Arduino UNO                       |
|Motor Driver    |L298N                             |
|Bluetooth Module|HC-05                             |
|Drive Motors    |DC, 100 RPM × 2                   |
|Camera          |Android Smartphone (IP Webcam app)|
|Network         |Wi-Fi Router                      |
|Chassis         |Cardboard + Castor Ball           |
|Power           |Nokia BL-5C battery               |

-----


## ⚠️ Limitations

1. **Communication chain fragility** — Any break in the Wi-Fi → PC → Bluetooth → Arduino chain halts the system completely.
1. **Speed compromise** — Image processing latency forced a reduced motor base speed to keep control loop timing in sync.
1. **PC dependency** — All processing runs on an external PC; the robot cannot operate standalone.

-----

## 🚀 Future Improvements

- [ ] Password-protect the Wi-Fi stream to prevent unauthorized control
- [ ] Port image processing on-board (e.g. Raspberry Pi) for a fully standalone robot
- [ ] Improve laser detection robustness under varying lighting conditions
- [ ] Add PID control for smoother, more accurate tracking

-----

### Demonstration
<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; margin-bottom: 40px;">
  <iframe
    src="https://www.youtube.com/embed/EHEGDKUKCOA"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 0;"
    allowfullscreen
    loading="lazy"
    title="Laser Guided Robot">
  </iframe>
</div>



Report: <a href="https://drive.google.com/uc?export=download&id=1OqSbmUEo-4TvIuF2snOefLXpir8Asz9i" download>
        Download PDF
        </a>

# 🏥 Automatic Wireless Health-Monitoring System

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

## 📁 Repository Structure

```
wireless-health-monitor/
├── matlab/
│   ├── emd_heart_rate.m        # EMD signal decomposition & BPM estimation
│   └── gui_display.m           # MATLAB GUI for live readout
├── arduino/
│   ├── transmitter/
│   │   └── tx_sensors.ino      # Read DHT11 + pulse sensor, transmit via RF
│   └── receiver/
│       └── rx_serial.ino       # Receive RF data, forward to PC via serial
├── report/
│   └── project_paper.pdf
└── README.md
```

-----

## 👥 Authors

|Name                      |Institution|
|--------------------------|-----------|
|Arindam Dev               |EEE, BUET  |
|A. N. M. Shahriyar Hossain|EEE, BUET  |
|Abdullah Al Nayeem        |EEE, BUET  |
|Md. Imtiaz Rashid         |EEE, BUET  |
|Purab Ranjan              |EEE, BUET  |

**Supervised by:**

- Dr. Md. Aynal Haque *(Professor, EEE, BUET)*
- Dr. Mohammed Imamul Hassan Bhuiyan *(Professor, EEE, BUET)*

-----

## 📚 References

1. “Wireless wearable pulse oximeter for health monitoring using Zigbee wireless network”, *ECIT-CON 2010*, pp. 575–579.
1. R. S. Khandpur, *Handbook of Bio-Medical Instrumentation*, 16th Ed., Tata McGraw Hill, 2003.
1. William Stalling, *Wireless Communication and Networks*, 4th Ed., Pearson, 2004.
1. Wang Q, Yang P, Zhang Y. “Artifact reduction based on EMD in photoplethysmography for pulse rate detection.” *Proc. IEEE Conf. Eng. Med. Biol.* 2010: 959–62.
1. Torres ME et al. “A Complete Ensemble EMD with Adaptive Noise.” *ICASSP 2011*, Prague.
1. Welch, P. D. “The use of fast Fourier transform for the estimation of power spectra.” *IEEE Trans. Audio Electroacoustics* 15.2 (1967): 70–73.
1. BRAVE Study — “Risk Factors for Heart Disease in Bangladesh.” Institute of Public Health, Cambridge.

-----

## 📄 License

This project was submitted as academic research at BUET. Feel free to reference or build upon it with attribution.

Report: <a href="https://drive.google.com/uc?export=download&id=1Vni5WGoBOu9bfa8YaN72HKMB3xVqpbAe" download>
        Download PDF
        </a>

## Design of a 4 Bit Comparator with Cascadable Comparator Cells

Report: <a href="https://drive.google.com/uc?export=download&id=1yAAAWbdVU-CnXUyG4t25b4Wo22TfN_kh" download>
        Download PDF
        </a>

