# Negative Viewer improvement plan

Date: 21 October 2026. Status: proposed for review; implementation has not started.

Make the app easier to start, frame, capture, adjust, and save while retaining its small, private, offline-capable design. This covers every improvement in the initial review: onboarding, control hierarchy, review before download, optional quick save, colour correction, manual crop, and a stable film guide.

## Proposed experience and defaults

1. Open to “View and capture film negatives”, a prominent **Start camera** button, and “Photos stay on your device. Nothing is uploaded.”
2. Show a short tip: “Place the negative against an evenly lit background. Keep the camera parallel to the film.” Camera permission is requested only after Start camera.
3. Start in **Negative** mode, with a visible **Negative / Normal** selector. Negative means an inverted preview; Normal means the original camera image.
4. Show the full camera frame with a centred **35 mm · 3:2** guide. Offer landscape/portrait orientation and **Full frame**. The guide defines the initial saved crop, rather than the browser window defining it.
5. Keep a prominent shutter button and labelled Capture action. Camera choice and mode remain easy to reach; theme, install, help, and Quick save move into a secondary settings panel.
6. Capture freezes the image and opens a review screen with **Retake**, **Crop**, **Adjust colour**, and **Save PNG**. Allow zooming in to check sharpness; the zoom is only for inspection and does not change the crop.
7. Save initiates one PNG download and returns to the camera. On export failure, preserve the review image and allow retry. Use wording such as “Download started”, because the app cannot confirm that a browser download reached the user's storage.
8. **Quick save** is off by default. When explicitly enabled, Capture bypasses review and saves the guide crop with the selected inversion mode. The live screen clearly says “Quick save on”. Per-image crop and colour edits do not silently carry into later captures.

These are proposed product choices for review. Keep the existing name, icons used for installation, local processing, PNG output, filenames, camera selection, conditional torch, and dark/light support. No framework, backend, account, analytics, or external image service is needed.

## Phase 0 — Documentation discovery and implementation boundaries

Completed for this plan: read the app source, README, manifest, service worker, and the linked browser documentation. Re-read the relevant sections before implementing each phase; line numbers below refer to the current source and will move.

Current implementation map:

| Concern | Starting point |
| --- | --- |
| Preview and toolbar | `index.html:224`, `index.html:229` |
| Theme and local settings | `index.html:287` |
| Inverted live preview | `index.html:315` |
| Device selection and stream setup | `index.html:327`, `index.html:391` |
| Metadata readiness | `index.html:371` |
| Torch and resolution display | `index.html:529`, `index.html:547` |
| Current window-dependent crop | `index.html:581` |
| Snapshot, inversion, download | `index.html:617` |
| Install and offline registration | `index.html:710`, `index.html:743` |
| Tooltips | `index.html:772` |
| Offline cache | `service-worker.js:1` |
| Existing correction/crop ideas | `README.md:33` |

Allowed browser API patterns:

- Copy the source/destination rectangle pattern from [MDN drawImage syntax](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage#syntax): `drawImage(image, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight)`.
- Copy the callback and null-result handling pattern from [MDN toBlob](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toBlob#examples). Retain object URLs for downloads and release them when safe.
- Copy the RGBA buffer, `getImageData(left, top, width, height)`, and `putImageData` patterns from [MDN pixel manipulation](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas). Use the same pixel transform for review and export.
- Copy the drag lifecycle from [MDN setPointerCapture](https://developer.mozilla.org/en-US/docs/Web/API/Element/setPointerCapture#examples), including cancellation handling. Limit `touch-action: none` to the crop interaction surface.
- Preserve the existing camera constraints and canvas snapshot inversion as the starting patterns. Require nonzero intrinsic dimensions and `readyState >= HTMLMediaElement.HAVE_CURRENT_DATA` before capture; see [MDN readyState values](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/readyState#value). A metadata timeout is not proof of a drawable frame.

Implementation boundaries: continue with plain HTML/CSS/JavaScript. Extract image geometry and pixel transforms into a small local `image-processing.js` file so the consequential image calculations can be tested independently. Keep UI and camera orchestration in `index.html`; do not introduce a general application architecture. Add the new script to the offline cache.

Verification: confirm every chosen method against its reference; record any compatibility gaps during implementation. Guard against assuming `toBlob` returns a Promise, relying on a canvas filter for export, or adding an undocumented camera capability as a requirement.

## Phase 1 — Welcome screen, clearer controls, and explicit app states

Depends on Phase 0.

Implement the welcome copy and lighting tip above. Replace toolbar emoji with consistent inline SVG icons, visible labels for primary actions, and a larger shutter control. Provide a small settings panel for secondary options. Retain a readable camera picker when multiple devices are available. Offer an explicit Stop camera action once running.

Represent idle, requesting camera, live, capturing, reviewing, saving, and recoverable error states. Show persistent permission/error guidance with Retry rather than relying only on disappearing toasts. Disable unavailable actions, prevent overlapping camera starts/switches, and enable Capture only when a current frame can be drawn. Keep the prior working stream when a camera switch fails where the device permits it.

Use native buttons, select/range controls, labelled groups, visible focus, at least 44 px touch targets, and reduced-motion support. Keep theme and install controls accessible. Handle unavailable local storage without preventing startup. Keep install guidance secondary to starting the camera.

References: `index.html:229`, `index.html:287`, `index.html:371`, `index.html:391`, `index.html:710`.

Verify: a first-time user can identify how to start and capture without tooltips; denied permission and no-camera cases have useful recovery; repeated clicks do not launch competing requests; keyboard focus follows the visible screen; narrow portrait and short landscape layouts keep Capture reachable.

Guards: no automatic camera permission prompt on page load; no active shutter before readiness; no icon-only primary action; no rewrite of working device-selection logic without a demonstrated need.

## Phase 2 — Stable film framing and shared image geometry

Depends on Phase 1; precedes review and manual crop.

Replace the window-shaped `object-fit: cover` capture model with a contained full sensor preview plus a centred film guide. Add 3:2, 2:3, and Full frame choices. Dim the region outside the guide and label it “Only the area inside the guide will be saved”. Fit the guide within the actual displayed image, excluding letterboxing.

Store the selected rectangle in source-image coordinates, not CSS pixels. Share one geometry calculation across the live guide, review starting crop, output dimensions, and PNG export. Preserve source pixels without upscaling. Window resizing, toolbar wrapping, and device rotation must not silently change an already selected source rectangle. Reset geometry deliberately when camera dimensions change.

References: `index.html:547`, `index.html:581`; MDN `drawImage` syntax in Phase 0.

Verify: use a numbered grid image to compare guide corners with exported pixels; test wide/tall cameras, both guide orientations, Full frame, letterboxing, and window rotation. Validate bounds and minimum dimensions. Check output size matches the HUD and no overlay is exported.

Guards: do not derive output aspect ratio from viewport shape, stretch the sensor image, or count letterbox pixels as photograph content.

## Phase 3 — Frozen review, explicit save, and quick save

Depends on Phase 2.

Separate taking a snapshot, rendering edits, encoding PNG, and initiating download. Capture an immutable full-resolution original into memory; initialise the review crop from the guide. Display its processed crop, with an inspection zoom and clear Retake/Save actions. Retake releases the snapshot and resumes the camera; stop camera tracks while hidden/backgrounded and recover through a visible resume action if required. Do not lose a review image when the camera stops.

Copy the `toBlob` export pattern linked in Phase 0 and retain the filename convention at `index.html:611`; allocate the filename when saving, not when entering review. Handle null blobs, exceptions, and slow callbacks. Ensure a timeout fallback and a late callback share one completion guard so they cannot trigger duplicate downloads. Avoid base64 fallback on the normal path. Release discarded canvases and object URLs. Keep review content available after an export error.

Add Quick save in settings with a visible live indicator. It uses the same snapshot/crop/inversion/export functions. Review edits apply to the current image only; quick save uses unadjusted tone/colour settings. Persist the Quick save preference when storage is available, and explain the bypass in its label.

References: `index.html:361`, `index.html:611`, `index.html:617`; MDN `toBlob` examples in Phase 0.

Verify: review does not download; Retake does not download; Save initiates one download; quick save initiates one download without review; double taps and a late encoder callback cannot produce duplicate files. Confirm the original frame remains unchanged, zoom does not alter export, and retries preserve edits.

Guards: do not claim confirmed filesystem success; do not mutate the original snapshot or silently discard it on failure; do not retain a history of full-resolution images.

## Phase 4 — Manual crop on the captured image

Depends on Phase 3.

Add a Crop view over the frozen full image. Start with the guide rectangle and offer drag-to-move, corner resizing, free crop, 3:2/2:3 ratio lock, Full frame, Reset, Apply, and Cancel. Apply updates the review; Cancel restores its previous crop. Use pointer capture for mouse and touch, with clear handles and an equivalent labelled numeric position/size interface for keyboard users.

Map interactions back into source coordinates through the shared geometry functions. Constrain the crop to valid image bounds, account for review zoom/letterboxing, and preserve it on resize. Invalidate any pending render when crop state changes.

References: Phase 2 geometry; MDN pointer capture example and `drawImage` source rectangles in Phase 0.

Verify: each handle, moving, ratio changes, cancellation, keyboard input, edge clamping, touch interruption, and rotation preserve the intended rectangle. Export a known pixel grid and check exact edges/dimensions. Verify Apply and Cancel are reversible.

Guards: no destructive recropping of the original; no page-wide suppression of touch gestures; no drag-only essential operation; no auto-crop claim.

## Phase 5 — Practical colour correction in review

Depends on Phases 3 and 4.

Add a collapsed Adjust colour panel with black-point and white-point pickers, Red/Green/Blue balance sliders, Brightness, Contrast, and Reset. Pickers operate on the inverted image before adjustment, or on the original in Normal mode. Explain them as choosing an area that should be black/white; sample a small neighbourhood to reduce single-pixel noise. Show picked colours and reject invalid/reversed per-channel ranges with a useful message. Users can always reset.

Define one deterministic pipeline: original snapshot → selected crop → optional inversion → per-channel black/white normalisation → RGB balance → brightness/contrast → clamp to displayable channel values → PNG. For channel value v (0–255), propose t = (v - B) / (W - B), then u = (t × gain - 0.5) × contrast + 0.5 + brightness, then output = round(255 × clamp(u, 0, 1)). Defaults B=0, W=255, gain=1, contrast=1, brightness=0 preserve the image. Proposed ranges: gain 0.5–2 per channel, contrast 0.5–2, brightness -0.5–0.5. Require W - B >= 1 in every channel; default endpoints apply until a picker is set. Record these choices beside the implementation. Do not present this as calibrated film restoration or guaranteed orange-mask removal.

Copy the pixel buffer patterns linked in Phase 0. Render interactive adjustments at a bounded preview resolution (proposed maximum long edge: 1280 px), coalesce rapid changes, and disregard stale results. Apply the identical transform once at native crop resolution when saving. Keep alpha intact, and provide a processing state during export. Add worker processing only if measurements show the simpler path blocks interaction unacceptably.

References: `index.html:315`, `index.html:617`, `README.md:38`; MDN pixel manipulation in Phase 0.

Verify: synthetic black/white/grey/colour patches demonstrate identity, inversion, clipping, invalid picker input, alpha preservation, and reset. Compare preview and decoded PNG transforms with a documented rounding tolerance. Try real colour and black-and-white negatives under even backlighting, checking clipping and casts. Confirm repeated slider changes always start from the original and do not accumulate degradation.

Guards: no full-resolution pixel processing on every live camera frame; no AI/cloud dependency; no promise that manual controls fully recover clipped exposure or handle every film stock. Real-negative evaluation is required before claiming improved scan quality.

## Phase 6 — Integration, offline support, and verification

Depends on all prior phases.

Update the README to describe the new flow, settings, full-frame/guide distinction, manual corrections, memory-only review, and browser-dependent download behaviour. Correct stale tooltip documentation while touching it: the current source creates one shared tooltip despite README claims about one per control. Maintain useful labels even if tooltips are removed.

Update the service-worker cache version and precache list for new local assets. Scope activation cleanup to this app's cache prefix: `service-worker.js:25` currently removes all other caches on the origin. Verify update behaviour without discarding an unsaved review image.

Add focused image-geometry/pixel-transform tests and meaningful capture-state/export tests, using a small local test harness rather than introducing a production framework. Use deterministic image fixtures and fake camera streams for automated checks; these do not substitute for physical-device verification.

Verification checklist:

- Run geometry, transform, and single-export tests; include denied camera permission and failed encoder paths.
- Verify the implementation uses the documented signatures; search for canvas filter export, unconditional `toDataURL`, duplicate download paths, and external image requests.
- Walk through welcome → camera → guide → capture → crop → colour adjustment → Save, then Retake and Quick save.
- Check desktop Chrome/Edge, Android Chrome, and iPhone Safari, including installed mode where available. Record unavailable device checks as pending rather than passed.
- Check keyboard-only use, focus restoration, a screen-reader pass on key controls/status, both themes, reduced motion, and small portrait/landscape screens.
- Check camera switching, unsupported torch, background/resume, camera disconnection, permission denial, and repeated captures on a physical phone.
- Check a warm offline launch and an upgrade from the previous cache version; unrelated caches must survive.
- Confirm the app makes no image uploads and references only its local assets; inspect PNG dimensions/crop and compare with review.
- Measure interaction and full-resolution export on a representative phone. Target a responsive slider preview, show progress during slow exports, and record capture/export latency and memory observations. No unbounded canvas accumulation after 20 captures/retakes.

References: `service-worker.js:1`, `service-worker.js:21`, `README.md:7`, and the Phase 0 API list.

Guards: no release based solely on desktop simulated-camera tests; no app-wide framework migration; no change to hosting or publication during this work without a separate instruction.

## Review checkpoints and scope

Review the welcome/live layout after Phase 1, guide/export parity after Phase 2, the capture flow after Phase 3, and representative before/after negative captures after Phase 5. Deliver each phase as a coherent change with its verification results.

The plan deliberately schedules the two highest-risk parts separately: coordinate correctness (guide/crop/export) and image correctness/performance (colour transforms). Both need evidence before release.

Deferred: automatic film-edge detection, AI restoration, perspective correction, dust removal, batch galleries, RAW capture, cloud storage, and hardware-specific exposure/focus controls. These were not part of the proposed first improvements. No implementation or deployment is authorised by this planning document itself.
