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

## Supervised by

- Md. Shafiqul Islam *(Lecturer, EEE, BUET)*
- Zabir Ahmed *(Lecturer, EEE, BUET)*

-----

## Demonstration
<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; margin-bottom: 40px;">
  <iframe
    src="https://www.youtube.com/embed/EHEGDKUKCOA"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 0;"
    allowfullscreen
    loading="lazy"
    title="Laser Guided Robot">
  </iframe>
</div>

-----

Report: <a href="https://drive.google.com/uc?export=download&id=1OqSbmUEo-4TvIuF2snOefLXpir8Asz9i" download>
        Download PDF
        </a>




