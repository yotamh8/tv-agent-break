# TV agent: handoff notes

Context from a first research session, so work can continue in Claude Code on the owner's
Windows PC, which is on the same home network as the TVs.

The owner has two TVs:
- a **Hisense 32A4BG** on VIDAA (everything up to "Ground rules" below)
- a **Rowa Google TV** (see [Second TV: Rowa Google TV](#second-tv-rowa-google-tv) at the end)

## The goal

Control the owner's Hisense TV from a computer, and later from an AI agent. The model is the
"agentic TV" video below: you type a vague request ("play the show about the lawyer who never went
to law school"), Claude works out what you mean (*Suits*) and drives the TV to play it.

The owner will say exactly what they want built. Options discussed so far:
- a smart remote driven by Claude (like the video)
- voice commands
- automations (turn off at a set time, open YouTube in the morning, and so on)

## The TV

| | |
|---|---|
| Model | **Hisense 32A4BG** (A4 series, 32", HD, "LED Backlight TV") |
| OS | **VIDAA** (Hisense's own OS, **not** Android TV / Google TV) |
| Software version | `V0004.09.09E.P0930` |
| Extended version | `F.P093.X0005.W01.Y01.J01` |
| Region | Israel. Importer: ניופאן דיספליי (Nupan Display), support line *5060 |
| UI language | Hebrew |
| Network | Wi-Fi |

The serial number, device ID and Netflix ESN shown under "About" are deliberately left out of
this file. They are unique identifiers, and nothing here needs them.

Settings menu (Hebrew): תמונה (Picture), שמע (Sound), **חיבור** (Connection, where the network
and IP address are), ערוץ (Channel), נגישות (Accessibility), **מערכת** (System), **אודות** (About,
where the model and firmware are). Tip: switching the menu language to English (מערכת ← שפה)
makes it easier to follow online guides.

## Sources the owner shared

1. **A video** (short-form, about 30 s, "So many of you asked after my last TV video, does this
   work on a Samsung?"). A creator shows a Samsung Frame TV with a custom phone "remote". The
   phone has an "Ask for a show" chat box. He types "Play me that show about the guy who is a
   lawyer but he never went to university to be a lawyer", Claude answers "I think you mean
   Suits", and the TV opens and plays Suits. His code is behind a "Comment COUCH, I'll send you
   the Git link" offer, so we don't have it. On Samsung (Tizen), this kind of control normally
   goes through the TV's remote websocket API on port 8001/8002.
2. **https://github.com/Gathin23/cleanyourtv**: debloats **Android TV / Google TV** over ADB
   (`pm disable-user`) and installs a clean, Apple-TV-like launcher. Its README says plainly that
   **Samsung, LG, Roku, Vizio and Hisense VIDAA are not supported** (no ADB). So it does
   **not** apply to this TV. It's still useful as a reference for an agent-driven TV workflow
   (`tools/tv.py`, `PROMPT.md`), and `docs/devices/samsung-tizen.md` shows how a non-Android
   TV was scanned from the network.

## What is and isn't possible on this TV

| Goal | Status | Notes |
|---|---|---|
| **Remote control from the PC** (keys, volume, input source, opening apps, power state) | ✅ Most likely | VIDAA runs an MQTT broker on **port 36669**. Hisense's RemoteNOW / VIDAA phone app talks to it. Newer firmware needs PIN pairing and sometimes a client certificate. On *some* newer firmware, key commands are reportedly ignored even though the connection works, so test it. |
| **Installing your own apps** (web apps / PWAs, e.g. Jellyfin) | ⚠️ Probably blocked | The Vidaa-Edge and `camellan/vidaa-appstore` methods stopped working on firmware **08.5x** (`Hisense_installApp is not defined`). This TV runs **09.x**. People got around it by **downgrading the firmware, which risks bricking the TV. Don't do it.** |
| **Removing ads / replacing the home screen** | ❌ No | There's no known root or debloat for VIDAA. You can only switch tracking off in the settings. |

### Libraries to start from (check they're maintained before relying on one)

- **vidaa-control** (PyPI; GitHub tombabolewski): a Python library and CLI that connects over TLS to the
  TV's broker on 36669, with PIN pairing, about 48 keys, volume, source and apps.
  https://pypi.org/project/vidaa-control/
- **hisensetv**: an older `HisenseTv` class. Defaults to port 36669 and the TV broker's usual
  credentials. SSL can be turned off for some models. https://hisensetv.readthedocs.io/
- **pyvidaa / myhisense-tv** (warrenrees): a library plus an MQTT bridge. Notes that modern TVs need
  a client cert/key that you supply yourself. https://github.com/warrenrees/myhisense-tv
- Home Assistant integrations, to read the protocol details:
  https://github.com/warrenrees/ha_vidaatv, https://github.com/clockbrain/ha_hisense_tv,
  https://github.com/tiagofreire-pt/Home_Assistant_Hisense_TV
- Background on installing apps:
  https://forum.jellyfin.org/t-install-jellyfin-on-vidaaos-based-tv-hisense-sharp-toshiba-etc

## Setting up the Windows PC

1. Claude Desktop for Windows (claude.ai/download), **Code** tab.
2. Python 3 from python.org, with **"Add python.exe to PATH"** ticked.
3. Git from git-scm.com, then clone this repo and open it in Claude Code.

## Next steps, in order

1. **Get the TV's IP:** הגדרות ← חיבור ← רשת (Settings → Connection → Network) → IP address.
   A DHCP reservation for it on the router is a good idea later.
2. **Turn on network control on the TV** if those settings exist, under חיבור or מערכת: anything
   like Wake on LAN / Wake on Wireless Network / network standby / control from a mobile device.
3. **Check the official path works:** install Hisense **RemoteNOW** (or the "VIDAA" app) on a
   phone and control the TV with it. If it works, the network remote is enabled.
4. **Check the port from the PC** (PowerShell):
   `Test-NetConnection <TV_IP> -Port 36669`. We want `TcpTestSucceeded : True`.
5. **First real test:** pair with `vidaa-control` (the TV shows a PIN), then send a harmless key
   (volume up/down) and read the state back. Save the pairing credentials locally and keep them
   **out of git**.
6. Find out what works on this firmware: keys, launching apps (Netflix, YouTube, etc.), switching
   input source, power on/off, reading the current app.
7. Then build what the owner asks for, for example a small local web or phone UI with an
   "Ask for a show" box: Claude turns the request into a title and an app, and the script drives
   the TV (open the app → search → play).

## Ground rules

- Don't downgrade or flash firmware, and don't use factory or service menus.
- Never expose port 36669 (or anything on the TV) to the internet.
- Keep the TV's identifiers and any pairing secrets out of the repo.

---

## Second TV: Rowa Google TV

The owner bought a **Rowa** TV running **Google TV**. Unlike the Hisense, this one is Android, so
the **cleanyourtv** repo applies directly: https://github.com/Gathin23/cleanyourtv

| | |
|---|---|
| Brand | Rowa (sold in Israel) |
| OS | Google TV (Android TV) |
| Model | **Not known yet.** Ask the owner (sticker on the back, or Settings → System → About) |
| Android TV OS version | **Not known yet.** Same About screen |
| Tested list in cleanyourtv | **None.** No `examples/` folder mentions Rowa as of 2026-10-30. Rowa may build on another maker's board (TCL, Skyworth and others), so compare its packages with the closest lists in `examples/`. |

### What the owner wants (so far)

1. Debloat it **safely**: disable ads, tracking and preinstalled junk, with a full restore.
2. Maybe the clean cleanyourtv launcher (optional. Keeping the stock Google TV home is fine.)
3. Install other apps on it (see below).
4. Possibly the same agentic-remote idea as the Hisense, which is easier here because ADB gives full
   control: keys, launching apps, deep links, screenshots.

### Why it's safe, and how it's undone

- **Debug mode = Developer options.** Settings → System → About → press OK on
  **Android TV OS build** 7 times → back → Developer options → **USB debugging** on (and
  network/wireless debugging if shown). Accept "Allow USB debugging?" on the TV once.
- cleanyourtv only uses `pm disable-user`. It **never** uninstalls, roots or bypasses anything,
  and its tools refuse Android core packages.
- `tools/disable.sh` (`tools/disable.ps1` on Windows) **records the starting state first** in
  `~/.cleanyourtv/` on the PC: which packages were enabled, which component owned Home, and the
  animation scales.
- Restore on Windows:
  `powershell -ExecutionPolicy Bypass -File tools\restore.ps1 -TvHost <TV_IP> <list> -Stock com.google.android.apps.tv.launcherx`
- Fallbacks: re-enable apps under the TV's Settings → Apps; a **factory reset** restores everything
  too (disabled apps are still installed).

### How to do it

1. Clone cleanyourtv on the PC (`git clone https://github.com/Gathin23/cleanyourtv`), open it in
   Claude Code, paste its `PROMPT.md` and fill in the Rowa model. Use the repo's scripts, not
   hand-written ones. That's where the backup, the Home guard and the restore record live.
2. The launcher build needs a JDK 17–22 and Android SDK platform 34 + build-tools 34. The
   debloat alone needs only `adb` (Android platform-tools) and Python 3.
3. Answer the checklist: apps in use, the dock, and whether to keep the voice assistant,
   Chromecast and updates.

### Don't

- Don't disable the keyboard, the input/HDMI packages, the only Home app, or `setupwraith`
  (casting stops without it on Google TV, and it can own the saved Wi-Fi). Let
  `launcher/build-and-install.sh` do the Home handover.
- Don't leave USB/network debugging on afterwards. Anyone on the Wi-Fi could try to connect.
- After a firmware update some bloat may come back. Re-run the same list
  (see cleanyourtv `docs/firmware-updates.md`).

### Installing other apps (it's Android, so yes)

1. **Google Play Store** on the TV. The normal route. Only TV-compatible apps show up.
2. **Sideload an APK from the PC over ADB:** with debugging on, `adb connect <TV_IP>` then
   `adb install app.apk`. Use an APK built for Android TV (or at least one that works with a
   remote, since phone apps often need touch), and match the TV's CPU (usually `armeabi-v7a`
   or `arm64-v8a`; `adb shell getprop ro.product.cpu.abi` tells you).
3. **On the TV itself:** install a downloader app from the Play Store, then allow it under
   Settings → Apps → Security & restrictions → **Unknown sources** (turn this back off afterwards).

Safety: only take APKs from the developer's own site, GitHub releases or F-Droid. Avoid "modded"
or cracked APK sites, which are a common source of malware. Leave Google Play Protect on.
Sideloaded apps sometimes don't appear on the stock Google TV home. Open them from
Apps → "See all", or from the cleanyourtv launcher's grid, which lists every launchable app.
