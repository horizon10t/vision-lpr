# Vision LPR

Self-hosted camera analytics in one container: **licence plate recognition**, **vehicle make/model and
direction of travel**, **face recognition** and **fire / smoke detection**, with a full web UI, MQTT,
Frigate integration and optional Google Coral acceleration.

![Docker](https://img.shields.io/badge/runs%20in-Docker-2496ED?logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/python-3.11-3776AB?logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

<p align="center">
  <img src="docs/screenshots/events.png" alt="Events page with known and blacklisted plates" width="900">
</p>

## Contents

- [Screenshots](#screenshots)
- [Features](#features)
- [Quick start](#quick-start)
- [Models](#models)
- [Configuring cameras](#configuring-cameras)
- [Direction of travel](#direction-of-travel)
- [Behaviour patterns](#behaviour-patterns)
- [Integrations](#integrations)
- [CPU or Google Coral](#cpu-or-google-coral)
- [Security](#security)
- [API](#api)
- [Development and tests](#development-and-tests)
- [Migrating from the original projects](#migrating-from-the-original-projects)
- [License and credits](#license-and-credits)

## Screenshots

| | |
|---|---|
| ![Live view](docs/screenshots/live-view.png)<br>**Live View**: go2rtc streams with PTZ, per-camera status and full screen | ![Events](docs/screenshots/events.png)<br>**Events**: filters, bulk review, known (green) and blacklisted (red) plate banners, IN / OUT tags |
| ![Event detail](docs/screenshots/event-detail.png)<br>**Event detail**: snapshot, confidence, vehicle make / model / colour, direction, Good / Bad feedback | ![Camera zones](docs/screenshots/camera-zones.png)<br>**Camera zones**: draw detect, exclude and IN / OUT direction regions on a live frame |
| ![Behaviour](docs/screenshots/behaviour.png)<br>**Behaviour**: arrival and departure patterns for a plate, with snapshots and a weekly heatmap | ![Search](docs/screenshots/search.png)<br>**Search**: by plate, vehicle, colour, date, event type, camera or a face image |
| ![Metrics](docs/screenshots/metrics.png)<br>**Metrics**: CPU, GPU, memory, per-process load, inference speed and Coral usage | ![Logs](docs/screenshots/logs.png)<br>**Logs**: filters, raw Docker output and export |

> Screenshots live in [`docs/screenshots`](docs/screenshots). Blur or crop real plates and anything
> that identifies your property before publishing them.

## Features

### Licence plate recognition (ANPR / LPR)
- YOLO plate detector with overlapping-tile search for small or distant plates, then OCR with
  [fast-plate-ocr](https://github.com/ankandrew/fast-plate-ocr) or PaddleOCR (RapidOCR), chosen
  automatically or set explicitly.
- Reads on a dedicated thread per camera, so a slow model elsewhere never makes a fast car miss.
  Faster polling while something moves in view, slower when the scene is still.
- **One event per vehicle pass**: plates are tracked across frames, confirmed over several reads (or one
  confident read), and the **sharpest read** within a configurable window is reported, with the
  majority-voted text. Parked vehicles are remembered and not reported again.
- Allowed characters, plate length and an optional regular expression for your region's plate format.
- **Known plates** and **blacklisted plates**, each with their own MQTT actions. Blacklist CSV import
  (including the NZ Police stolen-vehicle export layout). A plate cannot be on both lists.
- **Good / Bad review feedback** that the reader learns from: trusted plates correct near-miss reads,
  rejected reads are suppressed.
- Plate banners on every event are coloured **green for known** and **red for blacklisted** plates.

### Vehicles and direction of travel
- Vehicle make, model, generation, year and **colour** from an AutoVision classifier, merged into the
  plate event.
- **IN / OUT direction**: draw IN and OUT zones per camera; a vehicle that reaches the OUT zone and then
  the IN zone is reported **IN**, and the reverse **OUT** (see [Direction of travel](#direction-of-travel)).
- Direction tracking runs on its own fast thread and can use an **SSD MobileNet detector on a Google
  Coral** for several samples a second per camera.

### Behaviour
- Enter a plate to see **when it arrives and leaves**, and the routines that repeat: "Leaves every weekday
  at 8:05 am", "Arrives every fortnight on Sunday", with a confidence score, the next expected time,
  snapshots of the visits and a day-by-hour heatmap.

### Faces
- YuNet face detection and SFace recognition. Known faces from a few training images, unknown faces are
  still recorded, and a **face image search** across stored events.

### Fire, smoke and flame
- YOLO fire / smoke detector with a sustained-detection event lifecycle and cooldowns.
- Built-in filters for the usual false positives: **sun glare** (smooth bright regions), neutral white
  glare, and a multi-frame **flicker / motion gate**.
- Notifications by email (SMTP), webhook and [ntfy](https://ntfy.sh), each with its own message
  template and optional snapshot.

### Web UI
- Live View over go2rtc (MSE / WebRTC) with an MJPEG fallback, ONVIF **PTZ** controls and full screen.
- **Events** with camera, label and date filters, show-reviewed, bulk select (right click, Esc to
  cancel), mark reviewed and delete, and an Events / Search layout that adapts to phones.
- **Search** by plate, vehicle make, model or colour, time range, event type (plate, face, fire) and
  camera, or by an uploaded face image. Event types for disabled modules are hidden.
- **Camera settings**: per-module detect and exclude regions drawn on a live frame, cooldowns and
  ONVIF, all per camera.
- **Metrics**: CPU, memory, GPU, disk, network, per-process load, inference speed and Coral usage.
- **Logs**: filter by area, a raw **Docker** view of the container output, and **export** to a file.
- Light and dark themes, full-screen dialogs on mobile, login with **roles** (admin, operator, viewer)
  and optional **MFA** (email or authenticator app).

### Integrations
- **MQTT**: four topics (events, plates, vehicles, faces) plus per-plate and per-person custom messages.
- **Frigate**: optionally publish plate and face events to Frigate as manual events, with the plate in
  Frigate's licence plate field and the rest in the description.
- **go2rtc** bundled in the compose file.

## Quick start

On a Docker host:

```sh
git clone <this repository> vision-lpr
cd vision-lpr
cp config.example.yml config.yml
cp .env.example .env
docker compose up -d --build
```

Open `http://<host>:8080`. The initial config has no cameras, and the UI starts without model weights.
Add streams and cameras under **Settings / Config Editor** (or **Settings / Cameras**), restart, and
choose which detectors each camera runs. Model weights are not bundled: see [Models](#models).

`.env` controls the published ports and the models directory:

```env
WEB_PORT=8080
GO2RTC_PORT=1984
WEBRTC_PORT=8555
MODELS_DIR=./models
```

For WebRTC across subnets, set `go2rtc.webrtc.candidates` to the server address and the exposed
WebRTC port. MSE and MJPEG live options are always available.

## Models

Place weights in `models/` (or point `MODELS_DIR` at an existing directory). The container sees:

```text
/models/fire.pt                                   fire / smoke detector (or fire_smoke_yolov8.pt)
/models/plate-detector/yolov9-256-license-plates.onnx
/models/paddleocr/detection-small.onnx            PaddleOCR (RapidOCR) files, used when selected
/models/paddleocr/classification.onnx
/models/paddleocr/recognition.onnx
/models/paddleocr/en_dict.txt                     must match the recognition model
/models/yolov8n.pt                                vehicle detector
/models/autovision/car_classifier.tflite          vehicle make / model classifier
/models/face_detection_yunet_2023mar.onnx         face detector
/models/face_recognition_sface_2021dec.onnx       face recognizer
```

Flat layouts (`/models/fire.pt`, `/models/recognition.onnx`, ...) are accepted too. The default
`fast-plate-ocr` model downloads once into `/data/models` on first use. Model load failures are logged
and retried without stopping the other modules.

## Configuring cameras

```yaml
go2rtc:
  streams:
    driveway: rtsp://user:password@camera/stream1

cameras:
  - camera_name: driveway
    enabled: true
    fire_enabled: true
    anpr_enabled: true
    vehicle_enabled: true
    face_enabled: false
    anpr_event_cooldown_seconds: 20
```

Per camera you can also set detect regions (only look here), exclude regions (ignore here) for all
modules or for one module, a detection stream separate from the live stream, and ONVIF PTZ details.
Region coordinates are fractions of the picture (`x1,y1,x2,y2`, each 0 to 1), and the UI lets you
draw them on a live frame.

Useful plate settings under `anpr`:

| Setting | Meaning |
|---|---|
| `poll_interval_seconds`, `active_poll_interval_seconds` | read rate when the scene is still / moving |
| `confirmation_frames`, `instant_confirm_confidence` | reads needed to confirm a plate, and the single-read shortcut |
| `best_shot_seconds`, `best_shot_lost_seconds` | how long a confirmed plate keeps collecting reads, and how long it may go unseen before it is reported |
| `detector_tiling` | search overlapping tiles for small plates (more accurate, slower) |
| `ocr_backend` | `auto`, `fast_plate` or `paddle` |
| `plate_format` | optional regular expression the text must match |

## Direction of travel

1. In **Settings / Cameras**, switch on **Report direction (IN / OUT)** for the camera.
2. Pick **Direction: IN** or **Direction: OUT** in the region type dropdown and draw zones on a live
   frame. Draw IN where vehicles end up when they come in, and OUT at the other end of their route.
3. Save. Plate events on that camera now carry **Direction: IN / OUT**, and MQTT plate messages gain a
   `direction` field.

A vehicle is followed by its whole box. Its **wheel point** (bottom centre) decides which zone it is in,
and a vehicle that only clips a corner of a zone still counts. Direction is the order the vehicle first
shows up in each kind of zone, so any number of IN zones count as "IN" and any number of OUT zones as
"OUT". With only one kind of zone drawn, entering it gives that zone's direction and leaving it gives
the opposite.

Tips:
- Draw zones where the **wheels** pass, as bands across the road, not boxes around where the car body
  appears. Zones may touch each other.
- Direction needs several samples of a vehicle per pass. For fast cars, run the tracker on a Coral:

```yaml
vehicles:
  direction_detector: coral_ssd       # SSD MobileNet v2 (COCO) compiled for the Edge TPU
  direction_coral_model_path: /models/ssd_mobilenet_v2_coco_quant_postprocess_edgetpu.tflite
  direction_poll_interval_seconds: 0.2
```

The Edge TPU model is Google's precompiled `ssd_mobilenet_v2_coco_quant_postprocess_edgetpu.tflite`
(Apache-2.0, from the [google-coral/test_data](https://github.com/google-coral/test_data) repository).
Without a Coral it falls back to the CPU detector.

## Behaviour patterns

The **Behaviour** page needs no configuration beyond direction zones (or the per-camera "which way is
arriving" setting as a fallback). Enter a plate and a history range. Patterns need at least three weeks
of sightings and three matching days. Events recorded before direction was enabled show as "Seen"
rather than arriving or leaving, but still contribute to time-of-day patterns.

## Integrations

### MQTT

Set **Settings / Integrations / MQTT Topic Prefix**; four topics are derived from it. With the prefix
`vision/events`:

| Topic | Carries |
|---|---|
| `vision/events` | system and fire / smoke events (Frigate-style `type`, `before`, `after`) |
| `vision/plates` | plate reads: `plate`, `confidence`, `camera`, `time`, `direction` (when known), and for a known plate `known: true`, `name`, `notes`, `make`, `model` |
| `vision/vehicles` | `make`, `model`, `year`, `color`, `vehicle_type`, `confidence`, and the `plate` when it was read |
| `vision/faces` | `name` (or `unknown`), `known`, `confidence`, `camera`, `time` |

Plates, vehicles and faces are flat JSON objects, each with `id`, `camera`, `timestamp` and
`snapshot_url`. Any topic can be set explicitly with `mqtt.topic_events`, `topic_plates`,
`topic_vehicles` or `topic_faces`. Known and blacklisted plates, and known faces, can publish custom
messages (`{plate}`, `{name}`, `{camera}`, `{confidence}`, `{event_id}`, `{direction}`) to their own
topics. **Test MQTT** checks all four.

### Frigate

Set the address and admin credentials under **Settings / Integrations**, then switch on
`anpr.publish_to_frigate` and/or `faces.publish_to_frigate`. Each plate or face becomes a Frigate manual
event on the camera with the same name. The plate goes into Frigate's recognized licence plate field
(Frigate 0.16 or newer) and camera, direction, vehicle, and known / blacklisted status go into the event
description (Frigate 0.15 or newer). Frigate takes its own snapshot; the API has no way to upload ours.

### Notifications

Email, webhook and ntfy for fire and smoke events, each with a test button.

## CPU or Google Coral

The default build runs everything on the CPU. To add a Coral USB Accelerator:

```sh
docker compose -f docker-compose.yml -f docker-compose.coral.yml up -d --build
```

That builds the Edge TPU runtime into the image and passes the USB bus into the container. Turn on
**Settings / System / Hardware Acceleration / Google Coral** and press **Test Coral**, which checks the
runtime, the USB device and the delegate, and says which step failed. Through a VM, pass through both
USB ids: the Coral is `1a6e:089a` until first use, then `18d1:9302`.

A model uses the Coral only if a Coral build sits next to it, named `<model>_edgetpu.tflite`
(`fire.pt` and `fire_edgetpu.tflite`). Everything else, and everything when the Coral is off or
unusable, keeps running on the CPU, so enabling it cannot take a detector down. Create Coral builds with
`scripts/export-edgetpu.py` (x86-64 Linux with Google's `edgetpu_compiler`).

| Detector | Coral | Notes |
|---|---|---|
| Direction tracking (SSD MobileNet) | yes | precompiled model, see above |
| Fire / smoke (YOLO) | yes | export your `.pt` |
| Vehicle detector (YOLO) | yes | export `yolov8n.pt` |
| Plate detector | yes, with a YOLOv8 / 11 plate model | the bundled YOLOv9 ONNX cannot be exported |
| Vehicle make / model classifier | only if a compiled model exists | the bundled float16 model is not a Coral model |
| Plate OCR, faces | no | run on the CPU |

The Metrics page shows **Coral usage** (the share of time the Coral was busy, measured from this app's
own calls) once the Coral detector has run. NVIDIA CUDA and Apple Metal can be selected for the YOLO
models under the same Hardware Acceleration settings.

## Security

- The UI has a **login** with roles (admin, operator, viewer) and optional MFA. Turn it on under
  **Settings / Auth**, and still keep the UI and go2rtc on a trusted LAN, or behind an authenticated
  reverse proxy or VPN, before allowing remote access.
- `config.yml` holds credentials (camera URLs, MQTT, SMTP, password hashes, MFA secrets). It is in
  `.gitignore`: **never commit it**, and do not publish logs that contain stream URLs.
- Model weights, the `data/` directory and `.env` are also ignored by git.

## API

The web app is a FastAPI service; the UI uses these endpoints and so can scripts:

| Endpoint | Purpose |
|---|---|
| `GET /api/health` | application health (needs a login when auth is on) |
| `GET /api/events?event=license_plate` | stored events, with filters for `camera`, `event`, dates and `include_reviewed` |
| `POST /api/anpr/recognize` (also `/recognize`) | recognise a plate in an uploaded image (`file`) |
| `GET /api/anpr/status` | model readiness and per-stage timings (detect, OCR) |
| `GET/POST /api/known-plates`, `/api/blacklisted-plates`, `/api/known-faces` | manage the lists |
| `POST /api/search/face` | find faces matching an uploaded image |
| `GET/POST /api/camera-settings` | per-camera modules, regions and direction zones |
| `GET /api/metrics` | system and inference metrics history |
| `GET /api/logs?source=docker` | recent log lines (`source=docker` for the raw container output) |

## Development and tests

```sh
pip install -r requirements-dev.txt
python -m unittest discover -s tests -v
node tests/behaviour-engine.cjs        # the pattern engine behind the Behaviour page
```

The Python tests use controlled inference outputs to verify integration, event isolation, the plate
tracker, direction logic and the Frigate sync. Real model accuracy and RTSP performance should be checked
with your own weights and cameras.

For browser checks, run `python tests/serve_preview.py` in one terminal and
`node tests/browser-smoke.cjs` in another with Playwright and Chromium installed. This uses a
loopback-only test server with a disabled camera and temporary data. `node tests/ui-cleanup-smoke.cjs`
runs all-page checks in light and dark themes and desktop and touch layouts.

Project layout:

```text
app/
  main.py            camera loop, fire detection, startup
  web.py             FastAPI app and all API routes
  anpr/              plate detector, OCR, tracker and the per-camera reader
  vehicles/          vehicle detector, classifier, car tracker and the direction worker
  faces/             face detection and recognition
  direction.py       IN / OUT decision from zones
  frigate_sync.py    plate and face details sent to Frigate
  static/            the single-page web UI
scripts/             Edge TPU export, config merge
tests/               unit tests and browser smoke tests
demos/               sample streams for trying the app without cameras
```

Demo streams, with no real cameras needed:

```sh
docker compose -f docker-compose.yml -f docker-compose.demo.yml up -d --build
```

Configure plate streams with `rtsp://plate-demo:8554/car-plate-nz` (or `-uk`, `-us`). The fire demo
downloads sample assets at startup and needs internet.

## Migrating from the original projects

Vision LPR merges a fire / smoke addon and a standalone ANPR service. To import an existing Fire config
and the previous ANPR stream settings:

```sh
python scripts/merge-config.py --fire-config /path/to/fire/config.yml \
  --anpr-compose /path/to/anpr/docker-compose.yml --output config.yml
```

It refuses to overwrite an existing output, and ANPR cameras receive `anpr_` prefixes to prevent
collisions. Review camera names before enabling Frigate publishing: they must match Frigate's. For old
fire event history, stop the old service and copy its data directory to `./data` before starting.

## License and credits

Released under the MIT licence.

Built on [FastAPI](https://fastapi.tiangolo.com), [Ultralytics YOLO](https://github.com/ultralytics/ultralytics),
[ONNX Runtime](https://onnxruntime.ai), [RapidOCR](https://github.com/RapidAI/RapidOCR),
[fast-plate-ocr](https://github.com/ankandrew/fast-plate-ocr), [OpenCV](https://opencv.org)
(YuNet and SFace), [go2rtc](https://github.com/AlexxIT/go2rtc), and the Google Coral Edge TPU runtime
via [libedgetpu](https://github.com/feranick/libedgetpu). The Frigate integration and the SSD detector
choice follow [Frigate](https://frigate.video).
