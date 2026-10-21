It's a web app that uses the device camera and can capture a direct image or an inverted image, which auto-downloads to the device. Can be used for digitising negatives with a smartphone, although probably not great quality.

Try it out at https://luigiman86.github.io/negativecamera/

I mean, I'm not a dev, I have no idea what I am doing, but I started with the the basic code from this gist: https://gist.github.com/rand0mC0d3r/442ab777ad0bf2c91b6e67f6681a27d0, thanks to [@rand0mC0d3r](https://gist.github.com/rand0mC0d3r), and rounded it out into a web app, because my friend needs to digitalise negatives. Can you help me do things like make it auto crop? AI colour correction? Let the pull requests flow!

## How it works

Everything happens on your device. There is no server, no account, and nothing is uploaded.

- **📹 Open WebCam** — starts the camera.
- **🌓 Toggle negative** — inverts the preview.
- **📷 Capture** — saves a PNG of the current frame, inverted if the toggle is on.
- **🔄 Switch camera** — front/back on a phone, cycles cameras on a desktop.
- **🔦 Torch** — appears only if your camera has one. Even light matters more than resolution when scanning a negative.
- **🌙 Dark/light** — follows your system setting until you override it.

Hover a button for its tooltip, or tap it on a phone for the same hint. The small badge in the top-left shows the camera resolution on the left, the size of the saved image in the middle, and which camera is currently in use.

## Notes on the implementation

A few things that are less obvious than they look:

- **The saved file matches the preview exactly.** The preview is a centre-crop of the camera frame (to fill the screen), and the capture applies that same crop at the camera's native resolution. You frame the shot, and you get that shot.
- **Inversion is applied to a static snapshot, not with a canvas filter.** The canvas `filter` property is not available in iOS Safari, so using it silently saved un-inverted negatives on iPhones. The capture now copies the frame into a snapshot canvas with a plain draw, then inverts it with a `difference` blend against white — that is exactly `invert(100%)`, works everywhere, and is pixel-exact. Doing it this way avoids a multi-megabyte `getImageData` round trip, which is slow on a 4K frame.
- **Each capture uses a fresh canvas.** Reusing one canvas means the PNG encoder can still be reading it while the next capture resizes it. There is also a `toDataURL` fallback in case a browser never calls the `toBlob` callback, so a capture is never silently lost.
- **Captures go through `toBlob`, not `toDataURL`.** A 4K frame as a base64 data URL is tens of megabytes of string, which can crash the tab on a phone.
- **Filenames are `negative-YYYYMMDD-HHMMSS-NNN.png`** — no colons, which are illegal in filenames on Windows, and a counter so two captures in the same second do not collide.
- **The service worker precaches the app shell**, so it opens offline once installed. It is registered as `./service-worker.js` so it works in a subdirectory as well as at a domain root.
- **Tooltips are drawn by the app, not the browser.** A single shared tooltip can only hold one piece of text, so it ends up showing whichever control was used last; and the native `title` attribute has the same problem because the browser keeps one tooltip slot per page and will redisplay a stale label. Each control now gets its own tooltip, positioned above it and kept inside the viewport.
- **The camera dropdown is allowed to be wide.** It is the only control that shows text, so it takes up to 460px rather than being squeezed to a fraction of the bar.

## Still to do

Ideas, in rough order of usefulness:

- [ ] **Auto crop** — detect the film edge automatically and crop to it.
- [ ] **Colour correction** — a per-channel curve to remove the orange base of the negative and neutralise the cyan cast that inversion leaves behind. A black/white point picker plus three sliders gets most of the way there without any machine learning.
- [ ] **Drag-to-crop** — a manual crop rectangle, as a fallback while auto-crop is being tuned.
- [ ] **Aperture priority** — a simple brightness/contrast preview so you can check the exposure before capturing.

## Local development

The camera needs a secure context, so `file://` will not work. Serve the folder over HTTP:

```sh
python -m http.server 8000
```

then open <http://localhost:8000>. On a phone, use your computer's LAN address and make sure the browser is on `https` or `localhost` — iOS will not grant camera access over plain `http` on a LAN address.
