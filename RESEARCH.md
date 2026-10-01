# Flight Simulator — Research: Pi-XPlane-FMC-CDU revival

Confidence tags: [Certain] means verified from source code or git history. [Likely] means strong inference. [Guessing] means gap-filling.

## 0. Project status (updated 2026-10-01)

**Phase:** **v2.2.0 is released and working end to end** on the owner's Pi 3: the panel's buttons drive the Zibo in X-Plane 12, and the Pi's own screen shows the live CDU (screen only, filling the display). **Remaining:** the sticky switches, then John's install.

| Item | Status |
|---|---|
| Code | [variable31/Pi-XPlane-FMC-CDU-Keys-Only](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only), `master` (PRs #1–#9 merged) |
| Latest release | [v2.2.0](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only/releases/tag/v2.2.0): `flight-simulator-keys` (armhf/arm64) plus `flight-simulator-display` (all) |
| User guide | [README step-by-step guide](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only#step-by-step-guide-start-here), Parts 1–6, written for John |
| Target | X-Plane 12 + **Zibo 737-800** (captain's CDU), Raspberry Pi 3 on Raspberry Pi OS (Trixie, arm64, booting to console) |
| CDU screen | Shown via the **WebFMC (free)** X-Plane plugin plus a Chromium kiosk on the Pi's HDMI→VGA screen |

### Releases
| Version | What changed |
|---|---|
| 2.0.0 | Keys-Only revival: kernel GPIO chardev, INI keymap, debounce, vendored UDP library, CI-built .debs |
| 2.0.1 | Keypad scan every 5 ms instead of 1 ms, because the service used about 14% of a core on the Pi 3. Guide lessons from the first hardware test. |
| 2.0.2 | **Zibo keymap is the default** (`laminar/B738/button/fmc1_*`, from the original author's `ZiboFMC.cpp`). The default-737 keymap ships as `keys-xplane-default.conf`. |
| 2.1.0 | Optional `flight-simulator-display` package: WAITING screen, then the WebFMC CDU in a kiosk, with automatic switch-over. The keys program publishes the X-Plane IP to `/run/flight-simulator/xplane-host`. |
| 2.1.1 | Detects the Pi desktop blocking the screen and prints the fix. The guide adds a "turn off the desktop" step. The log command becomes `journalctl -t display-launch`. |
| 2.2.0 | The Pi's screen shows **only the CDU screen**, without WebFMC's on-screen keys: the launcher opens WebFMC with `#screen=1,side=0`. New `display.conf` settings `screen_only`, `side`, `extra`. Guide: detailed X-Plane PC setup (Part 6 steps 1–7) and "Keep Aspect Ratio off" to fill the screen. |

### Proven on real hardware [Certain] (owner's Pi 3, from logs the owner pasted)
- **Install and upgrade** from the public release links, 2.0.0 → 2.2.0. Conffiles are kept, and the Zibo keymap replaced the unedited old one silently.
- **GPIO keypad:** 66/69 keys sent the correct command on the first pass. LEGS/PREV/SP failed because of **sticky switches** (PROG sticking down blocks its column), which is a hardware fix. PREV and SP worked on a retest.
- **CPU fix:** confirmed necessary on the Pi (13.3 s of CPU per 96 s before the fix).
- **Services** start at boot. The keys service runs as the unprivileged `flightsim` user.
- **CDU screen:**
  - Shows **WAITING FOR X-PLANE...** at boot on the HDMI→VGA screen.
  - The adapter passes the screen's resolutions, and the Pi auto-picks 1024x768@60, so no forced resolution is needed.
  - The Chromium `EGL ES 3.0` errors are harmless; the Pi 3 is GLES2-only and Chromium falls back.
- **Wi-Fi only**, Ethernet unplugged and Pi rebooted: it joins Wi-Fi, both services are active, the screen shows WAITING, and `ssh jobolami@fmc.local` works.
- **X-Plane found over Wi-Fi:** `found X-Plane "DESKTOP-U7CR3AR" at 10.0.0.87:49000`, so the beacon reaches the Pi through the router, and the host file holds `10.0.0.87`.
- **WebFMC (free) on X-Plane 12:** the phone test works, at both `http://10.0.0.87:9090` and `.../#screen=1,side=0` (screen only).
- **The Pi screen switches from WAITING to the live CDU** by itself, and with 2.2.0 shows the **screen only, no keys**.
- **The Zibo reacts to the panel:** LEGS changes the page on the Pi's screen and in X-Plane, which confirms the `laminar/B738/button/fmc1_*` names on the current Zibo.
- **Full-screen CDU:** after turning WebFMC's **Keep Aspect Ratio** off once, with a USB mouse on the Pi, the CDU fills the 1024x768 screen.

### Not yet verified (next steps, in order)
1. **Keep Aspect Ratio survives a reboot.** [Likely] It does, because the kiosk's Chromium profile lives in the service's StateDirectory; confirm with one `sudo reboot`.
2. **Hardware:** clean or replace the sticky switches (PROG first, then A, H, E, L), then re-run the 69-key dry run, including LEGS.
3. **John's install:**
   - README Part 6 steps 1–7 on his X-Plane PC (same home network as the Pi); the Pi is already set up.
   - Optionally enable Raspberry Pi Connect, with his consent, for remote help.

### Lessons learned on hardware (now in the guide)
- **Imager:** without **Services > Enable SSH**, `ssh` says "Connection refused". The fix is an empty `ssh` file on the boot partition. The hostname comes from Imager (`fmc`), so use `fmc.local`.
- **The prompt matters:** commands typed at `PS C:\` go to Windows ("Sudo is disabled on this machine").
- **The apt "Download is performed unsandboxed as root" notice is harmless.**
- **The Raspberry Pi desktop holds the screen,** so the kiosk can't start (`Failed to start a DRM session`). Boot to console with `systemctl set-default multi-user.target`.
- **The screen program logs under `journalctl -t display-launch`, not `-u`,** because the logind session moves it out of the unit's cgroup.
- **WebFMC's options go in the address after `#`** (`#screen=1,side=0`), which our config parser read as a comment, so the launcher now adds them itself.
- **Merge before you publish a release.** A release published before its PR was merged pointed at old code. CI's version guard refused to build it, but the empty release still became "latest" until it was deleted and republished.

### Known small follow-ups (no release on their own)
- On reinstall, postinst prints `warn: already a member of video/render/input`. Cosmetic; silence it in the next release.
- The display .deb uses zstd compression (built on Ubuntu). It works on Trixie; switch to `-Zxz` if any older dpkg complains.
- [Likely] If the Pi's network changes while running (Ethernet ↔ Wi-Fi), the beacon listener may stay on the old interface until the service restarts. Verified fine after a reboot; not tested live.

### Decisions made (resolving §6 below)
- Pi 3 first; Keys-Only before the screen.
- A generic INI keymap (`/etc/flight-simulator/keys.conf`), with zero dependencies.
- The Linux GPIO chardev uAPI v2 instead of libgpiod, because Bookworm and Trixie ship incompatible libgpiod APIs.
- **The Zibo 737 (captain CDU) is the default keymap**, because John flies the Zibo.
- **Two-key chords stay rejected,** with no rollover (the owner's choice). Sticky keys are fixed in hardware.
- **The Pi shows the CDU screen only** (WebFMC screen-only mode, captain's side), because the panel has real buttons.
- **CDU screen via WebFMC (free) plus a Pi kiosk,** instead of our own renderer. Chosen for the least maintenance: the vendor tracks Zibo/X-Plane changes, and the X-Plane 12 Web API is localhost-only anyway. Accepted risk: a closed-source plugin, but the keys never depend on it.
- Releases are published from the GitHub web UI **after** merging. The build session can't push tags, and the release job accepts an existing release.

## 1. What this is
- **Author:** Shahada Abubakar (GitHub `dotsha747`), Malaysia.
  - Blog: blog.shahada.abubakar.net/tag/737fmccdu
  - 3D model: Thingiverse 774564
  - PCB: EasyEDA "737FMCCDU_V2"
- **Hardware:** a Raspberry Pi with a 9×8 matrix keypad on GPIO and FMC annunciator LEDs. The full version also drives an HDMI screen.
- **License:** [Certain] every repo is GPL-3.0. libXPlane-UDP-Client is also LGPL-3.

| Repo | Commits | Last commit | Role |
|---|---|---|---|
| Pi-XPlane-FMC-CDU | 21 | 2019-02-10 | Full CDU: keypad, LEDs, and SDL2 screen that mirrors the FMC text (Zibo 737, X737FMC, XFMC) |
| Pi-XPlane-FMC-CDU-Keys-Only | 11 | 2019-01-27 | Keypad only. Sends X-Plane *commands*; no screen, no plugin needed |
| libXPlane-UDP-Client | – | 2019-02-10 | Author's library for X-Plane's native UDP protocol (BECN discovery, RREF datarefs, CMND commands) |
| libXPlane-ExtPlane-Client | – | 2018-02-19 | Author's TCP client for the ExtPlane plugin (used by the full CDU for screen text) |
| libsdl2-rpifb | – | 2018-01-02 | Build script for SDL2 on the Pi framebuffer (Stretch-era hack) |
| gitpkgtool | – | – | Author's .deb build/publish tooling |

## 2. Why it's dead [Certain]
The code didn't stop working. What died is the distribution channel.

1. **The apt repo is gone.** `repo.shahada.abubakar.net` no longer resolves in DNS. Both READMEs tell users to `apt-get install` from it, so every new install fails at step one.
2. **The dependency chain lived only in that repo.** `debian/control` requires these packages:
   - `wiringpi`
   - `libxplane-udp-client1`
   - `libxplane-extplane-client0`
   - `libsdl2-rpifb`
   - `libsdl2-rpifb-ttf`

   None of these are in Raspberry Pi OS today; they were custom builds.
3. **Platform rot.** [Likely]
   - Raspbian Stretch has been end-of-life since 2020.
   - The original wiringPi was deprecated by its author in 2019, and a community fork exists.
   - The Pi 5 moves GPIO onto the RP1 chip, so legacy wiringPi does not work there.
   - `apt-key` is deprecated.
   - The Stretch-era SDL2 framebuffer build is obsolete now that Bookworm uses KMS/DRM.

## 3. Architecture (from source)
```
[GPIO keypad matrix] --wiringPi--> KeypadScanner --> keyInfo[row][col] = "laminar/B738/button/fmc1_1L"
                                                           |
                               Keys-Only: XPlaneUDPClient.sendCommand()  (UDP "CMND" -> X-Plane :49000)
                               Full CDU:  ExtPlane TCP (:51000) for screen datarefs + commands
[LEDs] <--wiringPi-- LEDs.cpp <-- dataref subscriptions (EXEC, MSG, OFST...)
[HDMI] <--SDL2/TTF-- Screen.cpp <-- ZiboFMC / X737FMC / XfmcFMC text-line datarefs
```
- Keys-Only discovers the sim automatically: it listens for X-Plane's multicast **BECN** beacon, then sends **CMND** packets.
- Pin numbers in the code use wiringPi numbering. The code comments give the BCM equivalents, which makes the libgpiod migration mechanical.

## 4. X-Plane 12 rebuild path (the user's chosen target)
**Keys-Only is the fastest win.** [Likely] It needs about three changes:
1. Replace wiringPi with **libgpiod v2**, which ships with Raspberry Pi OS Bookworm and works on the Pi 5. Map pins with the BCM numbers already listed in the comments.
2. Vendor libXPlane-UDP-Client into the repo as a submodule or subdirectory, so there is no external apt dependency. X-Plane 12 still speaks BECN, CMND and RREF over UDP [Likely]; verify against XP12's `Instructions/X-Plane SPECS from Austin/Exchanging Data with X-Plane.rtfd`.
3. Build with CMake, and publish `.deb` packages for arm64 and armhf through **GitHub Releases** from a GitHub Actions cross-compile job. Drop the private apt repo entirely.

**Full CDU adds more work:**
- Check whether the ExtPlane plugin builds and loads on XP12 [Guessing]. The alternative is X-Plane 12's built-in Web API (REST/WebSocket on :8086 since 12.1.x [Likely]), which would remove the plugin.
- Replace libsdl2-rpifb with stock SDL2 on KMS/DRM, which is in Bookworm.
- The Zibo 737 dataref names for FMC lines may have changed since 2019. [Guessing] Needs verification against the current Zibo/LevelUp 737.

## 5. License obligations (GPL-3)
- The fork must stay GPL-3.
- Keep the original copyright notices.
- Ship the source (or an offer of source) alongside any binaries.
- Mark modified files.

## 6. Open decisions for the rebuild phase (resolved; see §0)
- Hardware target: Pi 4 or Pi 5, and 32- or 64-bit OS?
- Keep the 9×8 matrix and PCB as-is, or support a generic "N buttons → M commands" config file? A YAML key map would make it usable for any button box, not just an FMC.
- Keys-Only first (a weekend) or the full CDU (weeks)?
