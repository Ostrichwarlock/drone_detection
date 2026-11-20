# Drone detection and tracking

Finds and tracks up to 2 drones in 1920x1080 video and writes one result file per video:
`frame,id,left,top,width,height,confidence` (frames start at 1, ids are 1 or 2 and stay the same when a drone leaves and comes back).
Target: real time (50 fps or more) on a Raspberry Pi with the AI HAT+ (Hailo-8, 26 TOPS).

## How it works

1. **Detector**: YOLO11n ([SkyGuardAI/drone-detection-yolov11](https://huggingface.co/SkyGuardAI/drone-detection-yolov11), MIT), being fine-tuned on our data.
2. **Tracker** (`tracker.py`): Kalman filter + two-threshold matching, at most 2 ids, re-identifies a returning drone by appearance (colour/brightness histogram, shape) and position.
3. **Trajectory crops** (`stack.py`): where the tracker expects a drone, a zoomed crop is checked again, which finds small drones the full-frame pass misses.
4. **Extra input layers** (`layers.py`, for the multi-channel model B, not trained yet): short / long / background difference (camera-shake compensated), tracker map, previous frame, brightness shape.
5. **Motion detector** (`motion_live.py`): kept for the views; motion crops are off by default (they did not improve the score).

## Scores on the 3 practice videos (IoU 0.5)

| Setup | MOTA | IDF1 | Precision | Recall | ID switches |
|---|---|---|---|---|---|
| SkyGuard + tracker | 0.843 | 0.916 | 0.983 | 0.858 | 0 |
| + trajectory crops | 0.855 | 0.916 | 0.980 | 0.873 | 1 |
| + boxes clipped to the frame (default, team evaluate.py) | 0.862 | 0.920 | 0.987 | 0.873 | 1 |

## Setup

    python -m venv .venv && source .venv/bin/activate
    pip install -r requirements.txt

Put the pretrained weights at `../pretrained/drone-detection-yolov11/best.pt` (download from the Hugging Face link above).
Data is not in this repo: `../test_sample_clean/<video>/video.mp4 (+ gt/gt.txt)`, `../hackathon_dataset`, Anti-UAV-RGBT.

## Run

Demo with switchable views:

    python demo.py ../test_sample_clean/test_6_hard_PXL_20261113_154422245_hard/video.mp4 --show
    python demo.py 0 --show            # live camera

Keys: 1 camera, 2 difference, 3 motion mask, 4 four-way view, 5 short diff, 6 long diff, 7 background diff,
8 tracker map, 9 previous frame, b brightness shape, 0 all layers, t trajectory, m motion crops, p trajectory crops,
o helper boxes, space pause, q quit.

Score all practice videos:

    python run_video.py --weights <best.pt> --videos ../test_sample_clean --out preds/x --imgsz 1280 --conf 0.01
    python pipeline.py --videos ../test_sample_clean --dets preds/x --out preds/x_trk --weights <best.pt> --no-motion
    python eval_track.py --gt ../test_sample_clean --pred preds/x_trk

Pi-style timing on the Mac (same code, PyTorch instead of Hailo): `python pi_run.py <video.mp4>`.

## Raspberry Pi 5 + AI HAT+ (Hailo-8)

Status: the Pi code path (`pi_run.py --backend hailo`) is written but **not tested on a Pi yet**, and the 50 fps target is not measured yet.

**1. Export the model (on the Mac)** to ONNX:

    python -c "from ultralytics import YOLO; YOLO('runs/ft_hack/weights/best.pt').export(format='onnx', imgsz=1280, opset=11)"

**2. Compile ONNX to .hef (x86 Linux PC, not the Mac or the Pi).** Install the Hailo Dataflow Compiler
(free download from the Hailo Developer Zone, needs an account), then parse, quantize with ~500 calibration frames
from our videos, and compile for `hailo8`, with NMS included in the .hef. Copy the resulting `skyguard.hef` to the Pi.

**3. Set up the Pi** (Raspberry Pi OS 64-bit, AI HAT+ attached):

    sudo apt update && sudo apt install hailo-all     # Hailo driver + HailoRT + Python bindings, then reboot
    hailortcli fw-control identify                    # should list the Hailo-8
    git clone https://github.com/czarekkdev/drone_detection && cd drone_detection
    python -m venv --system-site-packages .venv && source .venv/bin/activate   # system site = hailo_platform
    pip install opencv-python numpy scipy

**4. Run on the Pi:**

    python pi_run.py video.mp4 --backend hailo --hef skyguard.hef                  # one video
    python pi_run.py data --backend hailo --hef skyguard.hef --out results         # folder of <video>/video.mp4
    python pi_run.py 0 --backend hailo --hef skyguard.hef                          # live camera
    python pi_run.py video.mp4 --backend hailo --hef skyguard.hef --realtime       # feed at the video's own fps

It prints end-to-end fps and ms per stage (read / model / tracker); the slowest stage is the fps limit.
Results go to `results/<video folder name>.txt`. The tracker takes under 1 ms per frame, so the model sets the speed.
If 1280 px is too slow on the Hailo, export at 960 or 640 and compare fps and scores.

## Training

    python prep_data.py --src ../hackathon_dataset --dst drone_data          # single class, hard links
    python convert_antiuav.py --help                                          # Anti-UAV-RGBT videos -> YOLO frames
    python finetune.py --weights ../pretrained/drone-detection-yolov11/best.pt --data drone_hack.yaml --epochs 10 --imgsz 640
    python finetune.py --weights ../pretrained/drone-detection-yolov11/best.pt --data drone_ft.yaml   # hackathon + Anti-UAV

`mlayer.py` / `train.py` are an earlier colour-vs-grayscale experiment, kept for reference.

## Views

![all layers, drone landed on grass](docs/layers6_hard_1000.jpg)
