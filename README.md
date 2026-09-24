# Thermal Camera

A single-page WebUSB app for the RS PRO smart thermal camera. The camera is a rebadged **Seek Thermal Compact**: Windows reports it as `PIR206 Thermal Camera`, USB `289D:0010`, with a 206×156 sensor. The app shows the live feed with centre, min, max and cursor temperatures, and takes photos. Each photo is a PNG with the overlay and colour bar, and you can also save a per-pixel °C CSV.

## On an Android phone

The app is an installable web app (PWA). Chrome on Android supports WebUSB, and the phone needs no driver.

1. Open the hosted URL (for example `https://<github-user>.github.io/thermal-camera/`) in **Chrome**. WebUSB needs HTTPS.
2. In Chrome's ⋮ menu, choose **Add to Home screen** or **Install app**. After that it opens full-screen, and works offline too.
3. Plug the camera into the phone's USB-C port, using an OTG adapter if needed. Tap **Connect camera** and allow access to `PIR206 Thermal Camera`.

On a phone, tap the image to pin a spot. **Share** on a photo sends it to WhatsApp, email or Drive, and the screen stays on while the feed is live. iPhones can't use this app because no iOS browser has USB access.

### Hosting on GitHub Pages

1. Create a new **public** repository on github.com, for example `thermal-camera`.
2. Choose **Add file → Upload files** and drag in the *contents* of this folder: `index.html`, `manifest.webmanifest`, `sw.js`, `README.md` and the `icons` folder. Then commit.
3. In **Settings → Pages**, set Source to **Deploy from a branch**, Branch to `main`, and folder `/ (root)`, then save.
4. After a minute the app is live at `https://<github-user>.github.io/<repo>/`.

To publish an update, upload the changed files again. Also bump `CACHE` in `sw.js` (for example to `thermal-v2`) so installed copies clear their old cache.

## One-time driver setup (Windows)

Windows doesn't install a driver for the camera's USB interface (Device Manager shows `iAP Interface` with an error). The browser needs the generic WinUSB driver bound to it:

1. Run [Zadig](https://zadig.akeo.ie) and choose **Options → List All Devices**.
2. Select **iAP Interface** (USB ID `289D 0010 00`). Don't select `com.thermal.pir206.1`.
3. Set the driver to **WinUSB** and click **Install Driver**.
4. Unplug the camera and plug it back in.

To undo it: Device Manager → the device → Uninstall device → tick “delete the driver”.

## Run

```
python -m http.server 8206 --bind 127.0.0.1 --directory thermal-camera
```

Open http://localhost:8206 in **Chrome or Edge**, click **Connect camera** and pick `PIR206 Thermal Camera`. WebUSB needs a secure origin, and localhost counts as one.

Open http://localhost:8206/?demo to try the UI with a simulated camera and no hardware.

## Controls

- **Space**: take a photo. **F**: freeze or resume the feed.
- Hover over the image to read a temperature. Click to pin a spot, which also appears in photos.
- Palette, linear or equalised contrast, auto or locked range, noise reduction (temporal averaging), and rotate/mirror.
- **Auto-save to folder…** writes every photo (and the CSV, if ticked) straight to a folder you choose.

## Temperatures

The Compact outputs raw sensor counts, not temperatures. After the camera's own shutter correction, a pixel value of `0x4000` corresponds to the shutter temperature, which is roughly the camera's body temperature. The app converts with:

```
°C = shutter_temp + (counts − 0x4000) × °C_per_count     (defaults: 25 °C, 0.035)
```

Those defaults are estimates. For real readings, use **Temperature calibration**: pin a spot on something of known temperature and enter the value. One point sets the offset, and a second point at a different temperature sets the scale. Calibration is saved in the browser.

## Protocol

From [OpenThermal/libseek-thermal](https://github.com/OpenThermal/libseek-thermal):

- Vendor control transfers to interface 0 (`bmRequestType` 0x41/0xC1). The init sequence is in `SeekCompact.init()`.
- Frame: `START_GET_IMAGE_TRANSFER` (83) with `C0 7E 00 00` (32448 words), then bulk IN on EP 0x81 in 16224-byte reads, giving 208×156 uint16 LE.
- Word 10 is the frame type: `4` is the first frame and is used to find dead pixels, `1` is a shutter/flat-field frame, and `3` is an image. Usable area is x 0–206, y 1–154.
- Image = raw + 0x4000 − last shutter frame. Dead pixels are replaced with the mean of their 4 neighbours.
