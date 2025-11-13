# VisionScript OCR Suite

VisionScript OCR Suite is a Python-first playground for turning raw screenshots, photos, and clipboard captures into structured text with minimal ceremony.

## Overview

This repository wraps Apple’s Vision OCR pipeline behind a concise Python API. It is tailored for quick experiments, prototyping document automation, and exporting recognized text in formats that slot into downstream tooling.

### Highlights
- Works with images on disk, NumPy arrays, bytes buffers, or the system clipboard.
- Returns bounding boxes, rotation data, and confidence scores for every detected entity.
- Ships with a lightweight visualization recipe for annotating detections.
- Includes optional shell helpers for copy-to-clipboard OCR workflows.

## Installation

```
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

> [!NOTE]
> Apple Vision OCR requires macOS with the Vision framework available. Installing `pyobjc-framework-Vision` pulls in the native bridge.

## Quick Look

**Input sample**

<img src="assets/example1_input.png" width="800px" />

**Condensed output**

```python
{
    "image_width": 1672.0,
    "image_height": 622.0,
    "entities": [
        {"text": "Y Hacker News new | past | comments | ask | show | jobs | submit", "confidence": 1.0, "xmin": 41.8, "ymin": 552.889, "xmax": 1247.033, "ymax": 601.267},
        {"text": "1. A Starship will attempt a launch this weekend (faa.gov)", "confidence": 1.0, "xmin": 54.437, "ymin": 476.867, "xmax": 992.831, "ymax": 513.15},
        {"text": "32 points by LorenDB 1 hour ago | hide | 8 comments", "confidence": 1.0, "xmin": 121.836, "ymin": 443.0, "xmax": 764.713, "ymax": 469.092},
        # ... more entities ...
    ]
}
```

## Python Usage

```python
from pathlib import Path
import json
import numpy as np
from PIL import Image

from ocr import extract_text

image_path = Path("assets/example1_input.png")

# Directly from a file path
print(json.dumps(extract_text(image_path.as_posix()), indent=2))

# From an in-memory array
pixels = np.array(Image.open(image_path))
print(extract_text(pixels))

# From raw bytes
payload = image_path.read_bytes()
print(extract_text(payload))

# Straight from the clipboard
clipboard_result = extract_text("clipboard")
print(clipboard_result["entities"][0]["text"])
```

## Visualizing Detections

```python
from pathlib import Path
from typing import Iterable

import matplotlib.pyplot as plt
import matplotlib.patches as patches
import numpy as np
from PIL import Image

from ocr import extract_text

def draw_entities(axis, boxes: Iterable[dict]) -> None:
    for item in boxes:
        xmin, xmax = item["xmin"], item["xmax"]
        ymin, ymax = item["ymin"], item["ymax"]
        label = item["text"]

        if abs(item.get("rotation_degrees", 0.0)) < 5:
            axis.add_patch(
                plt.Rectangle(
                    (int(xmin), int(ymin)),
                    xmax - xmin,
                    ymax - ymin,
                    edgecolor="red",
                    facecolor=(0, 0, 0, 0.9),
                )
            )
        else:
            polygon = patches.Polygon(
                [
                    item["polygon"]["top_left"],
                    item["polygon"]["top_right"],
                    item["polygon"]["bottom_right"],
                    item["polygon"]["bottom_left"],
                ],
                closed=True,
                fill=False,
                edgecolor="red",
            )
            axis.add_patch(polygon)

        axis.annotate(
            f" {label}",
            (xmin, (ymin + ymax) / 2),
            color="white",
            fontsize=7,
            ha="left",
            va="center",
        )

target = Path("assets/example1_input.png")
image = np.array(Image.open(target))
result = extract_text(image, method="accurate", origin="top")

fig, ax = plt.subplots(figsize=(10, 4))
ax.imshow(image)
draw_entities(ax, result["entities"])
fig.set_tight_layout(True)
fig.savefig("assets/example1_annotated.png")
```

<img src="assets/example1_annotated.png" width="800px" />

## Command-Line Shortcuts

```bash
vision_copy() {
    python -c 'import json, ocr; print(json.dumps(ocr.extract_text("clipboard")))' |
        jq -r ".entities[].text" |
        tee >(pbcopy)
}

textshot() {
    local layout="${1:-}"
    [[ -n "$layout" ]] && layout="--psm $layout"
    screencapture -i -o /tmp/vision_ocr.png
    tesseract $layout /tmp/vision_ocr.png - | tee >(pbcopy)
}
```
