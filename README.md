# pedcount

A browser-based pedestrian counter that runs on a phone camera in real time. Draw a gate line across a sidewalk and it counts people crossing it in each direction; draw a polygon and it counts people entering it. Timed counting periods, live rates, CSV export.

Everything runs locally in the browser. No video is uploaded, recorded, or sent anywhere — only counts and event timestamps leave the page, and only when you export them yourself.

**Live app:** https://YOUR-USERNAME.github.io/pedcount/

## Requirements

- A phone or laptop with a camera and a reasonably current browser (iOS Safari 15+, Chrome, Firefox, Edge)
- An internet connection on first load, to fetch the detection model (~5 MB, cached afterwards)
- The page must be served over **HTTPS**. Browsers refuse camera access otherwise, which is why opening the file directly from disk will not work

## Running it

Open the live URL above. On iOS, use the Share button → *Add to Home Screen* so it opens without the address bar; this frees screen space and prevents an accidental swipe from ending a count.

To host your own copy: put `index.html` on any HTTPS host. GitHub Pages, Netlify, or a university web directory all work. There is no build step and no dependencies to install — the file is self-contained apart from two CDN script tags.

## Using it

1. **Turn on camera.** Mount or brace the phone so it will not move. Any camera motion registers as pedestrian motion and inflates the count.
2. **Adjust the gate.** A vertical gate down the middle of the view is there by default — drag either endpoint to move it, or tap *Gate* and tap two new points to redraw it. People are counted crossing it in both directions, recorded separately as `L2R` (left to right on screen) and `R2L`.
3. **Or draw a zone.** Tap *Zone*, tap three or more corners, then tap *Close zone*. People are counted once on entry; live occupancy is shown next to the entry total.
4. **Set the period** under *Counting period* — 1 to 60 minutes, or open ended. The count stops itself when the period expires.
5. **Start count.** Leave the phone alone until it finishes.
6. **Zoom** with the −/+ controls at the top right if pedestrians are small in frame. This is a centre crop, and the crop is what the detector sees, so zooming genuinely increases the pixel height of each person in the model input rather than just magnifying the display.
7. **Download CSV** when done.

You can use a gate and a zone at the same time. They are counted independently.

## How it works

Frames are letterboxed to 640×640 and passed to **YOLOv8n** in ONNX format, running through ONNX Runtime Web. WebGPU is used where the browser provides it, falling back to WebAssembly. The raw `(1, 84, 8400)` output is decoded for class 0 (`person`) and reduced by non-maximum suppression at IoU 0.45. The backend actually in use is shown under Detection settings.

**COCO-SSD (MobileNetV2 lite)** remains selectable as a faster, markedly less accurate alternative — useful on older devices where YOLOv8n runs too slowly to track walking pace. You can also supply your own `.onnx` export through the file picker, provided it uses the YOLOv8 detection head with a 640×640 input.

Detections are linked across frames by a lightweight tracker: boxes are advanced by a per-track velocity estimate, then matched greedily to new detections by intersection-over-union above 0.2. Unmatched detections start new tracks; unmatched tracks coast on their velocity and are dropped after a configurable number of missed frames.

Each track is located by the **midpoint of the bottom edge of its box** — approximately where the person's feet meet the ground. This is the point that crosses the gate and enters the zone. Using the box centroid instead would place people some distance above the ground plane and would systematically shift crossing times.

- **Gate crossings** are detected by testing whether the segment from the track's previous foot position to its current one intersects the gate segment. Both endpoints are bounded, so someone walking along the gate's extension beyond its drawn ends is not counted. Direction is taken from the sign of horizontal movement across the screen — rightward is `L2R`, leftward is `R2L` — so the labels mean the same thing regardless of how the gate is angled. For a purely vertical walk across an angled gate, the gate normal is used instead.
- **Zone entries** are detected by a ray-casting point-in-polygon test, incremented on the transition from outside to inside. Exits are logged but not counted.

A track must survive a minimum number of consecutive frames before it is eligible to be counted, which suppresses single-frame false positives.

## Settings

| Setting | What it does |
|---|---|
| Confidence | Detection score threshold. Raise it if you see phantom detections on street furniture; lower it if distant pedestrians are missed |
| Smallest person | Ignores boxes below this fraction of frame height. The main defence against noise at the far end of the view |
| Frames before a track counts | Higher values reject flicker but delay counting of fast movers |
| Frames kept after a person is lost | How long a track coasts through an occlusion before being discarded. Raise it where people pass behind obstacles |
| Detector input | COCO-SSD only. YOLOv8n has a fixed 640 px input |
| Model | YOLOv8n or COCO-SSD, plus your own ONNX export |

Record the values you used. They are written into the CSV header.

## CSV output

Three blocks in one file.

**Header** — session start, duration, camera position and bearing, directional and total gate counts, zone entries, per-minute and extrapolated per-hour rates, and the detection settings in force.

**Bins** — one row per time bin:
`bin_start_s, bin_end_s, gate_crossings, zone_entries, gate_per_minute, camera_lat, camera_lon`

**Events** — one row per crossing or entry:
`event_time_iso, seconds_from_start, event, track_id, direction, x_norm, y_norm, camera_lat, camera_lon, camera_bearing_deg`

Two coordinate systems are recorded, and they mean different things.

`x_norm` and `y_norm` are **image** coordinates: the position of the person's feet within the video frame at the instant of the crossing, normalised to 0–1 with the origin at the top-left. They tell you where along the gate someone passed, not where they were on the earth. They can be converted to ground coordinates only if you rectify the view — for example by homography from four known points visible in the frame.

`camera_lat` and `camera_lon` are **WGS84** coordinates of the phone, taken from the device GPS when the count starts, and repeated on every row so each block loads directly into GIS without a join. `camera_bearing_deg` is the compass bearing the phone was facing, from the magnetometer, where available. Every count in a session shares one camera position — the observer's location, not the pedestrian's.

Accuracy is recorded in the header as `camera_gps_accuracy_m`. Phone GPS in a street canyon is commonly 10–30 m, which is coarser than the sidewalk you are counting; for anything requiring precise placement, snap the point to the surveyed segment afterwards rather than trusting the raw fix.

## Accuracy and limitations

**Validate before you rely on it.** Hand-count several minutes of the same view under the same conditions and compute an error rate for that specific camera position. Error is not a property of the software; it is a property of the software plus your angle, distance, lighting, and crowd density.

Known sources of error, roughly in order of importance:

- **Occlusion and groups.** People walking abreast or in single file merge into one detection, undercounting. This is the dominant error in busy conditions
- **Distance.** Detection degrades sharply below about 30 px of person height. Prefer a shorter, wider field of view over a long one
- **Camera motion.** Wind, a nudged tripod, or handheld drift all produce spurious crossings
- **Low light and glare.** Dusk, headlights, and strong backlight all reduce recall
- **Re-identification.** The tracker has no appearance model. Someone who stops on the gate, reverses, or is occluded across the line may be counted twice or not at all
- **Extrapolation.** The per-hour figure is a linear projection from the observed window. A 15-minute count at 4pm becomes an hourly rate only under an assumption about temporal stability that you must argue for separately

The counts are not a replacement for a validated fixed sensor. They are a fast field instrument, appropriate for reconnaissance, relative comparisons between locations counted under matched conditions, and screening before committing to instrumented counts.

## Privacy

Video frames are processed in memory and discarded. Nothing is written to disk, no frames are transmitted, and the exported CSV contains only counts, timestamps, and normalised positions — no imagery and no identifying information.

Filming in public space may still be subject to local law and to your institution's human subjects review, particularly where the view includes private property or where you intend to publish. Check before deploying, even though no imagery is retained.

## License and credits

Detection: [YOLOv8n](https://github.com/ultralytics/ultralytics) via [ONNX Runtime Web](https://onnxruntime.ai/). YOLOv8 is AGPL-3.0 — note that this carries obligations if you redistribute a modified version of this tool. Fallback model: [COCO-SSD](https://github.com/tensorflow/tfjs-models/tree/master/coco-ssd), TensorFlow.js, Apache 2.0.
