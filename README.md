# Autonomous Car with Collision Detection and Avoidance

A small-scale autonomous vehicle that uses computer vision and embedded systems to detect and avoid collisions in real time, without human intervention.

---

## Table of Contents

- [Motivation / Problem Statement](#motivation--problem-statement)
- [Features](#features)
- [System Overview & Architecture](#system-overview--architecture)
- [Collision Detection Approach & ML/CV Methods](#collision-detection-approach--mlcv-methods)
- [Hardware Bill of Materials (BOM)](#hardware-bill-of-materials-bom)
- [Wiring / Connection Overview](#wiring--connection-overview)
- [Setup & Installation](#setup--installation)
- [How to Run / Deploy](#how-to-run--deploy)
- [Dataset & Training Notes](#dataset--training-notes)
- [Results & Evaluation](#results--evaluation)
- [Repository Structure](#repository-structure)
- [References](#references)
- [License](#license)
- [Authors & Acknowledgements](#authors--acknowledgements)

---

## Motivation / Problem Statement

Autonomous vehicles must navigate safely without human input, which requires robust real-time collision detection and avoidance. Key challenges include:

- Detecting and accurately identifying dynamic obstacles (e.g., other vehicles, pedestrians) in real time under varying traffic and environmental conditions.
- Implementing algorithms that not only detect obstacles but also predict potential collisions and make quick, accurate driving decisions.
- Balancing detection speed and accuracy — especially critical for fast-moving vehicles.

This project demonstrates that an affordable, fully automated navigation system can be built using inexpensive microcontrollers (Raspberry Pi, Arduino UNO, ESP32-CAM) and machine learning–based image processing.

---

## Features

- **Real-time video capture** via ESP32-CAM (AI Thinker) streamed over Wi-Fi.
- **Object detection** using a Faster R-CNN model (ResNet-50 backbone) to identify pedestrians, vehicles, and traffic signs.
- **2D obstacle grid mapping**: detected bounding-box centers are projected onto a 10×10 environment grid.
- **A\* path planning** to compute the shortest collision-free route around detected obstacles.
- **Autonomous motor control**: Raspberry Pi sends movement commands (forward / left / right / stop) to Arduino UNO, which drives the L293D motor driver.
- **Short-range collision avoidance** via ultrasonic sensors as a secondary safety layer.
- **Fully automated** — no human input required once powered on.

---

## System Overview & Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Raspberry Pi 5                      │
│  (Master controller — image processing, path planning,  │
│   decision-making, serial commands to Arduino UNO)      │
└──────────────┬──────────────────────────┬───────────────┘
               │ Wi-Fi (JPEG stream)       │ Serial / GPIO
               ▼                           ▼
    ┌──────────────────┐        ┌───────────────────────┐
    │  ESP32-CAM       │        │      Arduino UNO       │
    │  (AI Thinker)    │        │  (Motor & sensor ctrl) │
    │  OV2640 camera   │        └───────────┬───────────┘
    │  Wi-Fi streaming │                    │
    └──────────────────┘           ┌────────▼────────┐
                                   │  L293D Motor    │
                                   │  Driver         │
                                   └──┬──────────┬───┘
                                      │          │
                              ┌───────▼─┐   ┌───▼──────┐
                              │  DC /   │   │ Ultrasonic│
                              │  Servo  │   │ Sensors   │
                              │  Motors │   └───────────┘
                              └─────────┘
```

### Hardware components

| Component | Role |
|-----------|------|
| **Raspberry Pi 5** | Master controller; runs Faster R-CNN inference, A\* planner, and sends drive commands |
| **ESP32-CAM (AI Thinker)** | Captures and streams real-time video over Wi-Fi; 2 MP OV2640 camera, up to UXGA (1600×1200) @ 15–30 FPS |
| **Arduino UNO** | Controls motor driver and ultrasonic sensors; receives commands from Raspberry Pi |
| **L293D Motor Driver** | Dual full-bridge driver; drives two DC motors (up to 36 V, 600 mA/channel) |
| **Ultrasonic Sensors** | Short-range obstacle detection via sound-wave reflection |
| **DC / Servo Motors** | Propulsion and steering |

### Software stack

| Layer | Technology |
|-------|-----------|
| Object detection | Faster R-CNN (Detectron2 + PyTorch) |
| Backbone | ResNet-50 |
| Path planning | A\* search algorithm |
| ESP32 firmware | Arduino IDE (C++) |
| Inference environment | Python 3.11, CUDA 12.4 |
| Dataset management | Roboflow |

---

## Collision Detection Approach & ML/CV Methods

### Faster R-CNN (primary detector)

Faster R-CNN (2015) merges region proposal and detection into a single end-to-end architecture:

1. **Region Proposal Network (RPN)** — slides a small convolutional network over the CNN feature map to generate candidate regions of interest (RoIs). Reduces proposal time from ~2 s to ~10 ms per image. Uses anchor boxes with multiple scales and aspect ratios, objectness scores, and Non-Maximum Suppression (NMS).
2. **RoI Pooling** — transforms variable-size RPN proposals into fixed-size feature maps for downstream layers.
3. **Feature extraction** — ResNet-50 backbone extracts hierarchical features from each RoI.
4. **Fully connected layers** — produce:
   - *Object classification*: softmax over N+1 classes (N object classes + background).
   - *Bounding box regression*: 4×N predicted offsets.
5. **Multi-task loss** — combines classification (cross-entropy) and regression (smooth-L1) losses.

### Obstacle mapping & path planning

- Each detected bounding box center (cx, cy) is mapped onto a 10×10 grid:

  ```
  gx = cx × (GRID_SIZE / frame_width)
  gy = cy × (GRID_SIZE / frame_height)
  ```

- Grid cells are marked `0` (free) or `1` (obstacle).
- **A\*** search finds the minimum-cost path using `f(n) = g(n) + h(n)`, where `h(n)` is the Manhattan distance to the goal.

### Ultrasonic sensors (secondary safety)

Measure round-trip time of ultrasonic pulses to detect nearby objects at short range, providing a hardware-level stop signal independent of the vision pipeline.

---

## Hardware Bill of Materials (BOM)

| # | Component | Specification |
|---|-----------|--------------|
| 1 | Raspberry Pi 5 | 2.4 GHz quad-core Cortex-A76, 4/8 GB LPDDR4X RAM, 40-pin GPIO |
| 2 | ESP32-CAM (AI Thinker) | ESP32 SoC, OV2640 2 MP camera, built-in PSRAM, 802.11 b/g/n Wi-Fi |
| 3 | Arduino UNO | ATmega328P, 14 digital I/O (6 PWM), 6 analog inputs, USB-B |
| 4 | L293D Motor Driver IC | 16-pin DIP, dual full-bridge, 4.5–36 V, 600 mA/channel |
| 5 | Ultrasonic sensor (×2) | HC-SR04 or equivalent |
| 6 | DC motors (×2–4) | Small-scale hobby motors |
| 7 | Servo motor(s) | For steering if applicable |
| 8 | Jumper wires | Male-to-male, male-to-female, female-to-female |
| 9 | Car chassis / cardboard | Base frame for all components |
| 10 | Steel rod | Mounting post for ultrasonic sensor |
| 11 | USB-C power supply | 5 V / 5 A for Raspberry Pi 5 |

> **Assumptions / To verify**: Exact ultrasonic sensor model, number of motors, and power bank/battery specifications are not explicitly stated in the report. The L293D handles logic supply from Arduino; motor power may require a separate battery pack.

---

## Wiring / Connection Overview

```
Raspberry Pi 5
  ├── USB / Serial ──────────────► Arduino UNO
  │                                    ├── Digital I/O ──► L293D (IN1–IN4 + EN pins)
  │                                    │                       ├── OUT1/OUT2 ──► Motor A
  │                                    │                       └── OUT3/OUT4 ──► Motor B
  │                                    └── Digital I/O ──► Ultrasonic Sensors (TRIG / ECHO)
  │
  └── Wi-Fi (LAN) ───────────────► ESP32-CAM
                                       └── OV2640 Camera (integrated)
```

ESP32-CAM streams JPEG frames to the Raspberry Pi over the local Wi-Fi network. The Raspberry Pi runs inference and sends directional commands (move forward / turn left / turn right / stop) to the Arduino UNO via USB serial. The Arduino UNO drives the L293D and monitors ultrasonic sensors.

> **Assumptions / To verify**: The exact GPIO pin mapping and serial baud rate used between Raspberry Pi and Arduino UNO are not included in the available files.

---

## Setup & Installation

### 1. ESP32-CAM (Arduino IDE)

1. Install the **Arduino IDE** (≥ 1.8 or 2.x).
2. Add ESP32 board support — go to **File → Preferences → Additional Boards Manager URLs** and add:
   ```
   https://dl.espressif.com/dl/package_esp32_index.json
   ```
3. Open **Boards Manager**, search for **esp32 by Espressif Systems**, and install.
4. Open `esp32_codee/esp32_code.ino`.
5. Select board **AI Thinker ESP32-CAM** and the correct COM port.
6. Update Wi-Fi credentials:
   ```cpp
   const char *ssid     = "<your-SSID>";
   const char *password = "<your-password>";
   ```
7. Ensure `CAMERA_MODEL_AI_THINKER` is the only un-commented camera model.
8. Select a partition scheme with **at least 3 MB APP space**.
9. Upload the sketch (requires IO0 pulled to GND during flashing on some boards).

### 2. Python / Raspberry Pi inference environment

Requirements (tested on Python 3.11, CUDA 12.4):

```bash
pip install torch torchvision torchaudio
pip install 'git+https://github.com/facebookresearch/detectron2.git'
pip install opencv-python numpy
```

To verify GPU availability:
```python
import torch
print("CUDA Available:", torch.cuda.is_available())
print("CUDA Version:", torch.version.cuda)
```

### 3. Dataset (Roboflow)

The notebook downloads the dataset automatically via the Roboflow public API:
```python
!curl -L "https://public.roboflow.com/ds/Wd21F3AObk?key=EnSFtk9CJD" > roboflow.zip
!unzip roboflow.zip
!rm roboflow.zip
```

---

## How to Run / Deploy

### Flash ESP32-CAM

1. Complete the Arduino IDE setup steps above.
2. Connect ESP32-CAM to the PC, hold IO0 to GND (reset to flash mode if required), upload.
3. Open Serial Monitor at **115200 baud** — copy the IP address printed after `Camera Ready! Use 'http://<IP>'`.

### Run Faster R-CNN inference (Jupyter Notebook)

```bash
jupyter notebook faster_rcnn.ipnyb
```

Execute cells in order:
1. Install dependencies (torch, detectron2).
2. Download Roboflow dataset.
3. Register COCO-format dataset with Detectron2.
4. Configure and run the `DefaultPredictor`.
5. Visualise detections using `Visualizer`.

The notebook maps detected bounding boxes to the 10×10 grid and computes an A\* path.

### Full autonomous pipeline

1. Power on the assembled car.
2. ESP32-CAM connects to Wi-Fi and begins streaming.
3. Raspberry Pi runs the inference script, receives frames from the ESP32-CAM stream, performs object detection, builds the obstacle grid, and plans a path.
4. Movement commands are sent over USB serial to Arduino UNO, which drives the motors accordingly.
5. Ultrasonic sensors provide an emergency stop if an obstacle is within the minimum safe distance.

---

## Dataset & Training Notes

- **Source**: Roboflow public dataset (vehicle / traffic scene images in COCO format).
- **API key used in notebook**: `EnSFtk9CJD` (public dataset key — stored in notebook output; do not treat as a secret).
- **Framework**: Facebook Detectron2 (`DefaultTrainer` / `DefaultPredictor`).
- **Backbone**: ResNet-50 pre-trained on COCO; fine-tuned on the Roboflow dataset.
- **Hardware**: Training performed on a GPU with CUDA 12.4 (single GPU).
- **Note**: The notebook file is named `faster_rcnn.ipnyb` (note the `.ipnyb` extension — this is a typo for `.ipynb`; it is still a valid Jupyter notebook).

> **Assumptions / To verify**: Specific training hyper-parameters (learning rate, number of iterations, batch size) and final mAP scores are not present in the extracted notebook output.

---

## Results & Evaluation

### Algorithm latency comparison

The following latency measurements were recorded from the comparisons report (`comparisions table_with different nn algorithms.pdf`) across multiple test images:

| Algorithm | Typical Latency Range |
|-----------|----------------------|
| **Faster R-CNN** | 9 – 14 s per image |
| **MobileNet** | 4 – 25 s per image |
| **YOLO** | 4.5 – 7 s per image |

### Key findings

- **Faster R-CNN** detects objects accurately but has the highest latency among the three tested models.
- **YOLO** has the lowest latency but is less precise in detection (missed detections observed).
- **MobileNet** shows variable latency; detection boxes appear but are sometimes delayed.
- For the autonomous car application, **Faster R-CNN was selected as the primary detector** due to its superior accuracy, with the understanding that latency improvements may be required for higher-speed scenarios.

### System results

- The Raspberry Pi successfully received object detections from the ESP32-CAM stream and issued directional commands (move left, right, forward, stop) based on A\* path planning output.
- Grid coordinates computed by A\* were verified against detected object positions (see Figs. 13–15 in `analog_cnn_final_report.pdf`).
- The system operated in a fully automated manner in controlled indoor test environments.

---

## Repository Structure

```
.
├── esp32_codee/
│   └── esp32_code.ino                          # Arduino sketch for ESP32-CAM
│                                               #   (Wi-Fi camera server, AI Thinker model)
│
├── faster_rcnn.ipnyb                           # Jupyter notebook: Faster R-CNN training &
│                                               #   inference pipeline using Detectron2
│
├── Autonomous_car_finall-report.pdf            # Project report with ESP32 setup guide,
│                                               #   code walkthrough, and component overview
│
├── analog_cnn_final_report.pdf                 # IEEE-style technical paper covering system
│                                               #   architecture, Faster R-CNN methodology,
│                                               #   A* path planning, and results
│
├── PPT_analog_cnn_review-1.pptx                # Presentation slides — Review 1
│
├── PPT_analog_cnn_review-3.pptx                # Presentation slides — Review 3
│
└── comparisions table_with different           # Latency comparison across Faster R-CNN,
    nn algorithms.pdf                           #   MobileNet, and YOLO on test images
```

---

## References

The following works are cited in the project reports included in this repository:

1. T. N. Nizar, *Human Detection and Avoidance Control Systems of an Autonomous Vehicle*, 2020.
2. A. Mukhtar, L. Xia, and T. B. Tang, *Vehicle Detection Techniques for Collision Avoidance Systems*, 2015.
3. D. Reichardt and J. Schick, *Collision Avoidance in Dynamic Environments Applied to Autonomous Vehicle Guidance on the Motorway*, 2002.
4. N. Shilpa, K. Veera Kishore, and S. Anitha, *Data Science Using Warning Systems and Vehicle Crash Detection*, 2022.
5. I. Sonata and Lukas, *Autonomous Car Using CNN Deep Learning Algorithm*, 2021.

---

## License

No license file found in this repository.

---

## Authors & Acknowledgements

### Team

| Name | Student ID | Institution |
|------|-----------|-------------|
| Bojja Dheeraj Chowdary | CB.AI.U4AIM24109 | Amrita Viswa Vidyapeetham, Coimbatore |
| Kumpatla Sai Charan | CB.AI.U4AIM24124 | Amrita Viswa Vidyapeetham, Coimbatore |
| **Lakkireddy Prem Siva Sai Kumar** | CB.AI.U4AIM24125 | Amrita Viswa Vidyapeetham, Coimbatore |
| Pippalla Chirudeep | CB.AI.U4AIM24137 | Amrita Viswa Vidyapeetham, Coimbatore |

*(Repository owner: **Premsivasai** — Lakkireddy Prem Siva Sai Kumar)*

### Faculty Supervisors

- **Dr. Amrutha V**
- **Dr. Snigdhatanu Acharya**

### Courses

- 24AIM113: Introduction to Neural Networks, CNN, and GNN
- 24AIM114: Analog System Design

### Third-party tools & frameworks

- [Detectron2](https://github.com/facebookresearch/detectron2) — Facebook AI Research
- [PyTorch](https://pytorch.org/)
- [Roboflow](https://roboflow.com/) — dataset hosting and export
- [Espressif Arduino Core for ESP32](https://github.com/espressif/arduino-esp32)
