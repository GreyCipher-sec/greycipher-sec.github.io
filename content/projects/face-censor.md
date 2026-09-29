+++
title = "Face Censor"
date = 2026-09-04T00:06:11+02:00
description = "A local command-line tool for automatically detecting and censoring faces in images, videos and live webcam feeds."

[taxonomies]
tags = ["python", "opencv", "computer-vision", "privacy", "cli", "automation"]
+++

**Face Censor** is a small command-line tool for automatically detecting and
censoring faces in images, videos, and live webcam feeds.

It was built around a simple problem: screenshots, recordings, dashcam
footage, and other material often contain people who were never meant to
become part of the final publication.

Manually removing every face is tedious. Doing it frame by frame in a video is worse.

Face Censor handles the process locally, from the terminal, without sending
frames to an external service.

## What it does

The tool detects faces and applies a configurable censorship effect to every
detected region.

Three effects are currently available:

- **Blur**: Gaussian blur applied to the detected region.
- **Pixelate**: reduces the region to a small pixel grid before scaling it back
up.
- **Blackbox**: completely covers the detected region with a solid black rectangle.

Each detection receives a small padding margin so that the censorship extends
beyond the exact bounding box returned by the detector.

The tool can operate on:

- static images
- video files
- live webcam feeds

Webcam mode also allows the censorship effect to be changed while the camera is
running.

## Detection

Face detection is handled primarily by OpenCV's DNN implementation using the
**ResNet-10 SSD face detector**.

For each frame, the detector:

1. Resizes the frame to `300×300`.
2. Creates a normalised input blob.
3. Runs the image through the SSD network.
4. Filters detections using a configurable confidence threshold.
5. Converts the resulting bounding boxes back to the original frame dimensions.
6. Applies the selected censorship effect.

The default confidence threshold is `0.5`.

When the DNN model is unavailable, Face Censor falls back to OpenCV's Haar
Cascade detector. This keeps the tool usable without the neural-network model,
although detection quality is significantly lower, particularly for angled faces
and difficult lighting conditions.

## Local by design

The entire processing pipeline runs locally.

There are no API calls involved in detecting or censoring faces, and the input
material does not need to leave the machine.

The DNN model is downloaded once during setup and can then be used completely offline.

This makes the tool particularly useful for footage or screenshots that should
not be uploaded to a third-party service simply to remove identifying information.

## Installation

Create a virtual environment and install the dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

The DNN models can then be downloaded with:

```bash
[greycipher@remnant ~]$ python3 main.py --download-models
Downloading deploy.prototxt...
Saved -> models/deploy.prototxt
Downloading res10_300x300_ssd_iter_140000.caffemodel...
Saved -> models/res10_300x300_ssd_iter_140000.caffemodel
Done.
```

If the models are not available, the program automatically falls back to Haar
Cascade detection.

## Usage

### Image

```bash
[greycipher@remnant ~]$ python3 main.py --input photo.jpg --effect blur
[INFO] Using OpenCV DNN face detector (ResNet SSD)
[OK] 3 face(s) censored -> photo_censored.jpg
```

An output path can also be specified:

```bash
[greycipher@remnant ~]$ python3 main.py -i photo.jpg -o result.jpg -e pixelate
```

Supported image formats include:

`jpg`, `jpeg`, `png`, `bmp`, `tiff`, `webp`

### Video

```bash
[greycipher@remnant ~]$ python3 main.py --input video.mp4 --effect blackbox
[INFO] Processing 1800 frames at 30.0 fps...
[OK] Done -> video_censored.mp4
```

The original resolution and frame rate are preserved.

The current implementation writes video using the `mp4v` codec. Audio is not
copied to the resulting file, so it can be restored afterwards with FFmpeg if
required.

```bash
ffmpeg -i video_censored.mp4 \
       -i original.mp4 \
       -c copy \
       -map 0:v:0 \
       -map 1:a:0 \
       final.mp4
```

### Webcam

```bash
[greycipher@remnant ~]$ python3 main.py --webcam --effect blur
```

The default camera is opened and faces are censored in real time.

The active effect can be changed without restarting the program:

| Key | Effect |
| --- | --- |
| `B` | Gaussian blur |
| `P` | Pixelation |
| `K` | Black box |
| `Q` | Quit |

## Detection tuning

The default confidence threshold is `0.5`.

If the detector produces too many false positives, increase it:

```bash
python3 main.py -i video.mp4 -e blur --confidence 0.7
```

If real faces are being missed, lower it:

```bash
python3 main.py -i video.mp4 -e blur --confidence 0.2
```

Lower thresholds make the detector more permissive, but also increase the
probability of false positives.

If faces are still being missed at lower thresholds, the detector itself
may be the limiting factor. The Haar Cascade fallback is considerably less
capable than the DNN detector, especially with partially occluded or non-frontal
faces.

## The core

The DNN detection logic is intentionally small:

```python
def detect_faces_dnn(net, frame, confidence_threshold=0.5):
    """Return list of (x, y, w, h) using DNN detector."""
    h, w = frame.shape[:2]

    blob = cv2.dnn.blobFromImage(
        cv2.resize(frame, (300, 300)),
        1.0,
        (300, 300),
        (104.0, 177.0, 123.0)
    )

    net.setInput(blob)
    detections = net.forward()

    faces = []

    for i in range(detections.shape[2]):
        confidence = detections[0, 0, i, 2]

        if confidence > confidence_threshold:
            box = detections[0, 0, i, 3:7] * np.array([w, h, w, h])

            x1, y1, x2, y2 = box.astype(int)

            x1, y1 = max(0, x1), max(0, y1)
            x2, y2 = min(w - 1, x2), min(h - 1, y2)

            if x2 > x1 and y2 > y1:
                faces.append((x1, y1, x2 - x1, y2 - y1))

    return faces
```

The detector returns bounding boxes. The censorship layer then expands each box
slightly and applies the selected effect.

Keeping detection and censorship separate also makes it possible to experiment
with different detectors without rewriting the rest of the processing pipeline.

## Limitations

Face Censor is deliberately small, and the underlying detector has limitations.

- Very small faces may not be detected.
- Extreme profile angles can cause missed detections.
- Heavy occlusion can reduce detection accuracy.
- Haar Cascade detection is substantially weaker than the DNN detector.
- Video audio is currently not preserved automatically.
- Detection is not a guarantee of anonymity.

That last point matters.

Automated face censorship should be treated as an aid, not as proof that footage
is completely anonymised. Faces outside the detector's capabilities can remain
untouched, and other identifying information, license plates, names, screens,
distinctive objects, voices, is outside the scope of the current implementation.

## CLI

| Option | Short | Default | Description |
| --- | --- | --- | --- |
| `--input` | `-i` | - | Input image or video |
| `--output` | `-o` | automatic | Output path |
| `--webcam` | - | false | Use the default webcam |
| `--effect` | `-e` | `blur` | `blur`, `pixelate`, or `blackbox` |
| `--confidence` | `-c` | `0.5` | DNN confidence threshold |
| `--download-models` | - | false | Download the DNN model files |

## Examples

Pixelate faces in a group photo:

```bash
python3 main.py -i group_photo.png -e pixelate
```

Black-box faces in a dashcam recording:

```bash
python3 main.py \
    -i dashcam.mp4 \
    -o dashcam_safe.mp4 \
    -e blackbox \
    -c 0.65
```

Start a webcam session with pixelation:

```bash
python3 main.py --webcam -e pixelate
```

Re-download the detection models:

```bash
rm -rf models/
python3 main.py --download-models
```

## What's next

There are a few obvious directions for future versions:

- preserve and remux audio automatically in video mode
- add manual regions for censoring arbitrary areas
- support batch processing
- add license-plate detection
- experiment with YOLO-based detection for better coverage of non-frontal faces
- improve tracking between video frames

The goal is not to turn Face Censor into a large computer-vision framework.

It is a small Unix-style tool that does one job, locally, and can be dropped into
a larger workflow.

## Source

The complete project is available on GitHub:

[github.com/Greycipher-sec/FaceCensor](https://github.com/GreyCipher-sec/FaceCensor)
