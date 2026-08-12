

https://github.com/user-attachments/assets/fc710ce2-db8c-444a-97a1-47a4cc4f615a

# MG MIDI Controller — Full Architecture

```
iPad App (Safari / home-screen PWA)
        |
        |  WebSocket over Wi-Fi (ws://<mac-ip>:5004)
        |  {type:'noteOn'|'noteOff'|'cc'|'pitchBend', ...}
        v
Mac Companion — "MG MIDI Bridge" (Electron desktop app)
        |
        |  @julusian/midi -> CoreMIDI virtual port
        v
"MG Controller MIDI" (real CoreMIDI virtual device)
        |
        v
FL Studio / Ableton Live / Logic Pro (sees it as a normal MIDI input)
```

## Folder structure

```
MGMIDIController/
  ipad-app/
    index.html                 - your existing Sampler/pad-grid UI, extended with:
                                    - default port 5004 (was 17772)
                                    - hello handshake (sends project name on connect)
                                    - midiCC()/midiPitchBend() transport helpers
                                    - QR "Scan to Pair" flow (camera + jsQR)
  mac-companion/
    package.json                - Electron app manifest + electron-builder config
    main.js                     - main process: window, menu-bar tray, autostart
    preload.js                  - context-isolated IPC bridge
    src/
      midiBridge.js              - WebSocket server + CoreMIDI virtual port + Bonjour
    renderer/
      index.html / styles.css / renderer.js  - the status window UI
      assets/trayTemplate.png    - placeholder menu-bar icon (swap with MG branding)
  README.md                     - this file
```

## Running the Mac companion

```bash
cd mac-companion
npm install
npm start
```

This launches "MG MIDI Bridge" as a real desktop app: it creates the
`MG Controller MIDI` virtual CoreMIDI device and starts listening for the
iPad on port 5004 automatically — no button press required.

### Packaging as a standalone `.app`

```bash
npm run dist
```

Produces `mac-companion/dist/MG MIDI Bridge-*.dmg` / `.zip` via
electron-builder. This is unsigned (ad-hoc) by default, so the first launch
needs right-click → Open, same as any app distributed outside the Mac App
Store.

### Why `@julusian/midi` instead of the original `midi` package

Both wrap RtMidi and expose an identical API for creating a CoreMIDI virtual
port. `@julusian/midi` ships **prebuilt binaries** for common Electron
versions, so `npm install` usually doesn't need to compile anything. If it
ever falls back to a source build for your specific Electron version, you'll
need the Xcode Command Line Tools (`xcode-select --install`) — not the full
Xcode IDE — for `node-gyp` to compile against.

## Running the iPad app

Serve `ipad-app/index.html` from any local web server on your network (or
open it directly and "Add to Home Screen" for the full-screen PWA
experience, matching the existing `apple-mobile-web-app` meta tags already
in the file). Then: MIDI button → either scan the QR code shown in the Mac
app window, or type the Mac's IP manually.

**Important caveat on the QR scanner**: Safari requires a secure context
(HTTPS) for camera access (`getUserMedia`) on any host other than
`localhost`. If you're serving the iPad app over plain `http://` from a
local dev server, the "Scan to Pair" button will hit a permissions error —
manual IP entry (already built into the app) will still work fine. To get
camera scanning working, serve the app over HTTPS (a self-signed cert on
your local server, or a tool like `mkcert`/`ngrok`/Tailscale Serve).

## In your DAW

- **FL Studio**: Options → MIDI Settings → Input → enable "MG Controller MIDI"
- **Ableton Live**: Preferences → Link/MIDI → MIDI Ports → enable
  Track/Remote for "MG Controller MIDI"
- **Logic Pro**: detected automatically as a MIDI source, no manual step

## What's real here, and what isn't (read this before you ship)

I want to be direct about the tradeoffs in this architecture rather than
oversell it:

1. **This is genuinely a native desktop app**, not a web app pretending to
   be one — Electron produces a real installed macOS process with a Dock
   icon, menu-bar presence, and a real CoreMIDI device that any DAW
   recognizes exactly like a hardware controller.

2. **USB is not implemented in this transport.** The WebSocket architecture
   is IP-network-based (Wi-Fi only). A cable alone doesn't give a browser tab
   network connectivity to the Mac. Real USB MIDI needs either:
   - native CoreMIDI code on both ends (macOS auto-detects cabled iOS
     devices as MIDI sources for free — this is how the earlier
     Swift/CoreMIDI version of this project handled USB), or
   - `usbmuxd`/`iproxy`-style TCP-over-USB tunneling, which requires the
     iPad app to be an installed native or hybrid (e.g. Capacitor) app with
     a registered bundle ID, not a Safari tab.

   If USB is a hard requirement, the Swift/CoreMIDI native companion from
   earlier in this project is the correct tool for that specific job — it
   could run alongside or instead of this Electron bridge.

3. **"No manual IP typing" is solved via QR pairing, not true
   auto-discovery.** Browsers have no mDNS/Bonjour API, so a Safari-hosted
   web app cannot natively discover devices on the network the way a native
   app could. The Mac side does advertise itself over real Bonjour
   (`_mgmidibridge._tcp`) so a *future* native/Capacitor-wrapped version of
   the iPad app could browse for it automatically — but as a web app today,
   QR scanning is the closest thing to zero-config pairing that's honestly
   achievable.

4. **"Connected Device: iPad name" shows project name + IP, not the iPad's
   system name.** Safari doesn't expose the device's actual name
   ("Alex's iPad") to JavaScript for privacy reasons. There's no way around
   this short of a native app using `UIDevice.name`.

None of these are dealbreakers for the stated priority order (FL Studio
connection working > no Xcode friction > simple install) — they're just
the specific corners where "Electron + WebSocket" trades away some
capability a fully native app would have, in exchange for being much
easier to build, ship, and modify without Xcode.
