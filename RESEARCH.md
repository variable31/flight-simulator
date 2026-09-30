# Flight Simulator — Research: Pi-XPlane-FMC-CDU revival

Confidence tags: [Certain] means verified from source code or git history. [Likely] means strong inference. [Guessing] means gap-filling.

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

## 6. Open decisions for the rebuild phase
- Hardware target: Pi 4 or Pi 5, and 32- or 64-bit OS?
- Keep the 9×8 matrix and PCB as-is, or support a generic "N buttons → M commands" config file? A YAML key map would make it usable for any button box, not just an FMC.
- Keys-Only first (a weekend) or the full CDU (weeks)?
