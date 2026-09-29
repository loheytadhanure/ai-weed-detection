# 🌱 AI Weed Detection System

An AI-powered precision agriculture system that detects weeds in real time using **YOLOv8**, captures live video through an **ESP32-CAM or laptop webcam**, maps detected weeds to spatial zones, and displays the results through a live web dashboard.

The system combines **Computer Vision, Edge IoT, Real-Time Video Processing, and Spatial Intelligence** to support targeted weed monitoring.

---

## 🚀 Overview

Manual weed scouting across large agricultural fields is labor-intensive and difficult to scale. Incorrect identification can lead to unnecessary herbicide usage or missed weeds.

This project automates weed detection using a real-time object detection pipeline.

The system:

- Captures live frames from an ESP32-CAM or laptop webcam
- Detects weeds using a custom-trained YOLOv8 model
- Calculates a confidence score for every detection
- Maps detected weeds to a **5×5 spatial grid**
- Monitors soil moisture using an ESP32 sensor
- Streams annotated video through a web dashboard
- Displays weed locations through a live heatmap
- Allows switching between camera sources in real time

---

## 🏗️ System Architecture

```text
                   ┌──────────────────────┐
                   │    Camera Sources    │
                   │                      │
                   │  Laptop Webcam       │
                   │        OR            │
                   │     ESP32-CAM        │
                   └──────────┬───────────┘
                              │
                              │ Live JPEG Frames
                              ▼
                   ┌──────────────────────┐
                   │    Flask Backend     │
                   │                      │
                   │  Camera Thread       │
                   │  YOLO Inference      │
                   │  Sensor Thread       │
                   └──────────┬───────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
             YOLOv8 Detection      Soil Moisture
                    │                   │
                    ▼                   │
              5×5 Grid Mapping          │
                    │                   │
                    └─────────┬─────────┘
                              ▼
                   ┌──────────────────────┐
                   │     REST APIs        │
                   │                      │
                   │ /video_feed          │
                   │ /api/prediction      │
                   │ /api/heatmap        │
                   │ /api/sensors        │
                   │ /api/status         │
                   │ /api/toggle_camera   │
                   └──────────┬───────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │   Web Dashboard      │
                   │                      │
                   │ Live Video            │
                   │ Weed Status           │
                   │ Confidence             │
                   │ 5×5 Heatmap            │
                   │ Soil Moisture           │
                   └──────────────────────┘
🧠 Machine Learning Pipeline
Dataset

The model was trained using CottonWeedDet3, a real-world cotton field weed dataset containing multiple weed species.

The original annotations were provided in VGG Image Annotator (VIA) JSON format.

Since YOLO requires normalized bounding-box annotations, a custom preprocessing pipeline was developed to convert the annotations into YOLO format.

VIA → YOLO Conversion

The conversion pipeline:

Reads the VIA JSON annotations
Extracts the image and bounding-box information
Converts top-left coordinates into center-based coordinates
Normalizes coordinates between 0 and 1
Generates one YOLO .txt annotation file per image
VIA JSON
   │
   ▼
Extract Bounding Boxes
   │
   ▼
Convert Coordinates
   │
   ▼
Normalize to 0–1
   │
   ▼
YOLO Annotation Files
🤖 YOLOv8 Model

The system uses YOLOv8 Nano with transfer learning from COCO pretrained weights.

Training Configuration
Parameter	Value
Model	YOLOv8 Nano
Parameters	~3.16M
Input Size	640 × 640
Epochs	50
Batch Size	16
Training Hardware	Google Colab T4 GPU
Dataset	CottonWeedDet3
Split	80% Training / 20% Validation

The trained model is approximately 6 MB, making the Nano architecture suitable for real-time inference on commodity hardware.

⚡ Real-Time Inference

Although the model was trained at 640px, inference is performed at 320px to reduce computational cost and improve real-time performance.

The detection pipeline uses two confidence thresholds:

YOLO NMS confidence
        ↓
      0.4
        ↓
Python-level filtering
        ↓
      0.5
        ↓
Final Weed Detection

This provides an additional filtering stage to reduce false positives.

🗺️ 5×5 Spatial Weed Mapping

One of the main features of the system is spatial mapping.

Instead of simply displaying bounding boxes, every detected weed is assigned to one of 25 zones.

┌────┬────┬────┬────┬────┐
│ 0,0│ 0,1│ 0,2│ 0,3│ 0,4│
├────┼────┼────┼────┼────┤
│ 1,0│ 1,1│ 1,2│ 1,3│ 1,4│
├────┼────┼────┼────┼────┤
│ 2,0│ 2,1│ 2,2│ 2,3│ 2,4│
├────┼────┼────┼────┼────┤
│ 3,0│ 3,1│ 3,2│ 3,3│ 3,4│
├────┼────┼────┼────┼────┤
│ 4,0│ 4,1│ 4,2│ 4,3│ 4,4│
└────┴────┴────┴────┴────┘

The center of each detected bounding box is mapped to a corresponding row and column.

The grid resets during every inference cycle, allowing the dashboard to represent the current detected state rather than accumulating historical detections.

📷 ESP32-CAM Integration

The system can use an ESP32-CAM with an OV2640 camera sensor as the physical edge camera.

The ESP32-CAM:

Captures frames at QVGA 320×240
Streams MJPEG over HTTP
Communicates with the Flask backend over Wi-Fi
Reads soil moisture data through an analog input

The camera resolution matches the inference resolution, avoiding unnecessary server-side resizing.

Hardware
ESP32-CAM
   │
   ├── OV2640 Camera
   │
   ├── Wi-Fi
   │
   └── Soil Moisture Sensor
            │
            ▼
       Flask Backend

The moisture sensor uses GPIO33 (ADC1) so that analog readings remain compatible with the ESP32 Wi-Fi operation.

🌐 Backend

The backend is implemented using Flask.

Three background threads handle the major real-time tasks:

Thread 1 → Camera Capture
Thread 2 → YOLO Inference
Thread 3 → Sensor Polling

The Flask main thread exposes the REST API and serves the dashboard.

API Endpoints
Endpoint	Purpose
/video_feed	Streams annotated MJPEG video
/api/prediction	Returns current weed status and confidence
/api/heatmap	Returns the 5×5 spatial grid
/api/sensors	Returns soil moisture information
/api/status	Returns system health
/api/toggle_camera	Switches camera source
🖥️ Web Dashboard

The frontend provides a real-time monitoring interface containing:

Live camera feed
Weed detection bounding boxes
Detection confidence
5×5 weed heatmap
Soil moisture gauge
System status
Camera source toggle

The dashboard communicates with the backend using REST APIs and periodically refreshes detection and sensor information.

⚙️ Real-Time Optimizations

Several optimizations were implemented to improve latency.

Latest Frame Extraction

When processing the ESP32 MJPEG stream, the system prioritizes the newest available frame instead of processing an old buffered frame.

This prevents network backlog from creating unnecessary detection latency.

DirectShow Camera Backend

For Windows laptop webcams, OpenCV uses the DirectShow backend to reduce camera buffering and latency.

Multi-Threaded Processing

Camera capture, YOLO inference, and sensor processing run independently using daemon threads.

             Flask Application
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
   Camera       YOLO Model     Sensor
   Thread       Thread         Thread
📊 Technology Stack
Machine Learning
Python
YOLOv8
Ultralytics
OpenCV
Google Colab
PyTorch
Backend
Flask
Python
REST APIs
Multithreading
Hardware / IoT
ESP32-CAM
OV2640 Camera
Soil Moisture Sensor
Arduino C++
Frontend
HTML
CSS
JavaScript
SVG
Communication
HTTP
MJPEG Streaming
Wi-Fi
📸 Hardware Prototype

The system was integrated with a physical ESP32-CAM and sensor setup.

ESP32-CAM Prototype

Hardware Assembly

Sensor and Camera Setup

Add the uploaded hardware images to an assets/ folder and rename them to match the paths above.

🔄 End-to-End Workflow
Camera / ESP32-CAM
        │
        ▼
Capture Frame
        │
        ▼
JPEG Frame Processing
        │
        ▼
YOLOv8 Inference
        │
        ▼
Confidence Filtering
        │
        ▼
Bounding Box Detection
        │
        ▼
5×5 Spatial Mapping
        │
        ├───────────────┐
        ▼               ▼
 Weed Status       Heatmap Zone
        │               │
        └───────┬───────┘
                ▼
          Flask REST API
                │
                ▼
         Web Dashboard
⚠️ Current Limitations

The current prototype has several known limitations:

ESP32-CAM supports a single active client connection
Soil moisture handling during streaming currently uses a fixed value in the prototype
Dashboard authentication is not implemented
Detection history is not persisted
Flask development server is used instead of a production WSGI server
The 5×5 grid represents screen-space zones rather than GPS-based field coordinates
🔮 Future Improvements

Potential production improvements include:

Train for more epochs with early stopping
Domain-specific augmentation for different lighting and weather conditions
ONNX Runtime inference for faster CPU execution
Upgrade from YOLOv8n to YOLOv8s when hardware permits
Replace API polling with WebSockets or Server-Sent Events
Use MQTT for asynchronous sensor communication
Add authentication to the dashboard
Persist detection history
GPS-based field mapping
Support multiple camera nodes
Integrate automated precision spraying using relay-controlled solenoid valves
🎯 Future Precision Agriculture Extension

The spatial grid provides a foundation for targeted weed treatment.

A future implementation could map detected weed zones to specific spray-control outputs:

YOLO Detection
      │
      ▼
5×5 Zone Mapping
      │
      ▼
Weed Zone Identified
      │
      ▼
ESP32 Control Signal
      │
      ▼
Relay
      │
      ▼
Solenoid Valve
      │
      ▼
Targeted Spraying

This would extend the system from weed detection to automated precision weed management.

👨‍💻 Project Focus

This project demonstrates the integration of:

Real-time Computer Vision
Object Detection
Transfer Learning
Edge IoT
Embedded Systems
REST API Development
Real-Time Video Streaming
Spatial Data Mapping
Sensor Integration
📌 Project Status

Prototype / Research Project

The current implementation demonstrates real-time weed detection and monitoring using YOLOv8 and ESP32-CAM, with documented paths toward a production-ready precision agriculture system.
