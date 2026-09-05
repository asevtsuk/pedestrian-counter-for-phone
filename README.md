# pedcount

A browser-based pedestrian counter that runs on a phone camera in real time. Draw a gate line across a sidewalk and it counts people crossing it in each direction; draw a polygon and it counts people entering it. Timed counting periods, live rates, CSV export.

Everything runs locally in the browser. No video is uploaded, recorded, or sent anywhere — only counts and event timestamps leave the page, and only when you export them yourself.

**Live app:** https://asevtsuk.github.io/pdestrian-counter-for-phone/ 

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
5. **Start.** The button sits top-left over the camera view with the countdown beside it. Starting takes a GPS fix and a compass bearing automatically, and counting never waits on either.

   A coarse network fix is requested first, because it usually returns within a second or two and settles whether location works at all while the count is still young. A precise satellite fix follows with a 60-second window and replaces it, and a position watch keeps refining for the rest of the session. Because the phone is stationary during a count, any late fix is applied retrospectively to rows already recorded, and the banner clears itself when that happens.

   Warnings only interrupt during the first twenty seconds of a count. A failure discovered after that goes to the status line under the panel and is left to resolve itself in the background, so a long count is never interrupted partway through.

   If nothing ever arrives, the reason appears in a banner at the top of the screen — `permission denied` means a stored refusal to clear in Safari's settings, while `timed out` or `position unavailable` means the receiver is struggling, usually indoors or in a street canyon. Counting is unaffected either way; only the coordinate columns are left blank.
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

## Output

**Download ZIP** produces `session.csv` plus one JPEG per snapshot. **CSV only** gives the table without the images. The ZIP is written store-only — the JPEGs are already compressed — so any unzip tool opens it.

### Reference snapshots

The app saves a 640 px JPEG of the same cropped view the detector sees, so a session carries visual evidence of what was actually counted. It captures one at the start, then only on a meaningful change:

- **Position** — the phone moves further than the distance threshold (10 m by default) from where the last snapshot was taken, measured by equirectangular approximation against the live GPS watch
- **View** — the scene itself changes. Each frame is reduced to a 32×32 greyscale signature and compared to the signature at the last snapshot; when the mean absolute difference exceeds the sensitivity threshold on two consecutive checks, two and a half seconds apart, a new snapshot is taken. Requiring two consecutive checks stops a passing lorry or a group filling the frame from triggering one, while a genuine change of camera angle triggers immediately

Both thresholds are adjustable, there is a manual capture button, and the whole feature can be switched off. Sessions are capped at 40 snapshots.

Every gate crossing and zone entry records the `snapshot_id` in force when it happened, so counts join to the image showing the view they came from.

### CSV structure

Four blocks in one file.

**Header** — session start, duration, camera position and bearing, directional and total gate counts, zone entries, per-minute and extrapolated per-hour rates, and the detection settings in force.

**Bins** — one row per time bin:
`bin_start_s, bin_end_s, gate_crossings, zone_entries, gate_per_minute, camera_lat, camera_lon`

**Snapshots** — one row per reference photo:
`snapshot_id, file, time_iso, seconds_from_start, reason, lat, lon, gps_accuracy_m, bearing_deg, zoom, width_px, height_px`

`reason` records why it was taken: `session_start`, `moved_<n>m`, `view_changed`, or `manual`.

**Events** — one row per crossing or entry:
`event_time_iso, seconds_from_start, event, track_id, direction, x_norm, y_norm, camera_lat, camera_lon, camera_bearing_deg, snapshot_id`

`direction` is `L2R` or `R2L` — rightward or leftward across the screen.

Two coordinate systems are recorded, and they mean different things.

`x_norm` and `y_norm` are **image** coordinates: the position of the person's feet within the video frame at the instant of the crossing, normalised to 0–1 with the origin at the top-left. They tell you where along the gate someone passed, not where they were on the earth. They can be converted to ground coordinates only if you rectify the view — for example by homography from four known points visible in the frame.

`camera_lat` and `camera_lon` are **WGS84** coordinates of the phone. Event rows carry the live position at the moment of that crossing; the header records the fix at session start. `camera_bearing_deg` is the compass bearing the phone was facing, from the magnetometer, where available. Every count in a session shares one camera position — the observer's location, not the pedestrian's.

Rows recorded before the first fix are filled in with that fix once it arrives, on the assumption that a tripod-mounted phone has not moved during the count. Accuracy is recorded in the header as `camera_gps_accuracy_m`, and a coarse network fix will show a much larger figure than a satellite one. Phone GPS in a street canyon is commonly 10–30 m, which is coarser than the sidewalk you are counting; for anything requiring precise placement, snap the point to the surveyed segment afterwards rather than trusting the raw fix.

## Battery

Continuous neural inference is the heaviest thing a phone browser can do, and a long count will warm the device and drain it noticeably. Several things reduce that.

**Detection rate** is the dominant cost and the main control, under *Power*. The default caps inference at 6 per second. At walking pace that is a step of about 23 cm between frames — far finer than needed to catch a gate crossing — while an uncapped loop on a fast backend may run three or four times as often for no gain in accuracy. Drop to 3 per second for very long sessions; the step grows to about 47 cm, still comfortably below the point where the tracker starts losing people.

**Camera preview** can be dimmed or hidden entirely. The detector reads the camera stream directly rather than the display, so counting is completely unaffected, and on an OLED screen a dark display draws appreciably less. Useful once a tripod is set and you no longer need to watch the framing.

**GPS** manages itself. The receiver runs in high-accuracy mode only until a fix better than 25 m arrives, then drops to a low-power watch, on the reasoning that a tripod does not move. Snapshot displacement tests use the threshold or the fix's own error margin, whichever is larger, so a coarse fix cannot trigger spurious photos through noise alone.

**Backend** matters too: WebGPU is markedly more efficient per inference than the WebAssembly fallback. Check which one you have under *Detection*.

Beyond the app: lower screen brightness, close other tabs, and keep the phone out of direct sun. Thermal throttling will cut the detection rate on its own once the device gets hot, so shade is worth more than it sounds.

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

Video frames are processed in memory and discarded, and nothing is transmitted anywhere — all detection runs on the device.

Reference snapshots are the exception, and the one part of this worth thinking about before you deploy. They are photographs of public space that may contain recognisable people, they are written into the ZIP you download, and they change the review picture accordingly. Snapshots can be switched off entirely under *Snapshots*, in which case the export contains only counts, timestamps and coordinates. If you keep them, treat the ZIP as image data and store it as such.

Filming in public space may still be subject to local law and to your institution's human subjects review, particularly where the view includes private property or where you intend to publish. Check before deploying, even though no imagery is retained.

## License and credits

Detection: [YOLOv8n](https://github.com/ultralytics/ultralytics) via [ONNX Runtime Web](https://onnxruntime.ai/). YOLOv8 is AGPL-3.0 — note that this carries obligations if you redistribute a modified version of this tool. Fallback model: [COCO-SSD](https://github.com/tensorflow/tfjs-models/tree/master/coco-ssd), TensorFlow.js, Apache 2.0.
