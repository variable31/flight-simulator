# Flight Simulator — Research: Pi-XPlane-FMC-CDU revival

Confidence tags: [Certain] means verified from source code or git history. [Likely] means strong inference. [Guessing] means gap-filling.

## 0. Project status (updated 2026-09-30)

**Phase:** Keys-Only **v2.0.0 is released**. It is waiting on its first test on real hardware.

| Item | Status |
|---|---|
| Rebuilt code | [variable31/Pi-XPlane-FMC-CDU-Keys-Only](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only), `master` (PRs #1, #2 merged) |
| Release | [v2.0.0](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only/releases/tag/v2.0.0): armhf + arm64 `.deb`, built on Debian Bookworm (glibc 2.36) |
| User guide | [README step-by-step guide](https://github.com/variable31/Pi-XPlane-FMC-CDU-Keys-Only#step-by-step-guide-start-here), written for a non-technical user (John) |
| Target | X-Plane 12, Raspberry Pi 3 (also Pi 4/5), Raspberry Pi OS Bookworm or newer |
| Full CDU (screen) | Not started. This repo is its home. |

**Verified [Certain]:**
- CI passes: unit tests (config parser, debounce and chords, UDP `CMND` bytes, `BECN` discovery) plus an end-to-end dry run.
- Both release `.deb`s were downloaded from the public `releases/latest` links and inspected: correct architecture, needs glibc ≤ 2.36, and `keys.conf` is a conffile.
- An x86 build of the same `v2.0.0` source was run through guide Part 2 (install) and the software side of Part 4 on a Linux container:
  - `apt install` created the `gpio` group and the `flightsim` user.
  - `--version` prints 2.0.0.
  - The systemd unit passes `systemd-analyze verify`.
  - `--dry-run` prints exactly the lines the guide shows.
  - An edited `keys.conf` survives a reinstall.
  - `apt remove` works cleanly.

**Not yet verified (next steps, in order):**
1. **Real Pi GPIO.** [Certain] The kernel GPIO code (`src/gpio_chardev.cpp`) has only been compiled; it has never run on hardware. Run guide Parts 2–4 on the owner's Pi 3 with the keypad wired. Also check the parts the container could not run: `raspi-config` (Part 3), the postinst service enable/start, and `systemctl stop`.
2. **X-Plane 12 end to end** (Part 5). [Likely] The default 737-800 responds to the `sim/FMS/*` commands in the default keymap, but this has not been confirmed. If the log shows `sent ...` and the sim doesn't react, fix the commands in `keys.conf`; the code does not need to change.
3. **Install at John's.** Only after steps 1 and 2 pass. Ask him for `journalctl -u flight-simulator-keys -n 50` if anything fails.

**Decisions made** (resolving §6 below):
- Pi 3 first.
- Keys-Only before the full CDU.
- A generic INI keymap (`/etc/flight-simulator/keys.conf`) rather than YAML, so there are zero dependencies.
- The Linux GPIO chardev uAPI v2 instead of libgpiod, because Bookworm and Trixie ship incompatible libgpiod APIs while the kernel ABI is stable.
- Releases are published from the GitHub web UI. The build session can push branches but not tags, so the release job accepts an existing release.

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
