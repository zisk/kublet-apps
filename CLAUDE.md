# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Community apps for the Kublet: an ESP32 (320 KB RAM, 4 MB flash) with a 240×240 ST7789 display and a single button on GPIO 19 (active LOW, internal pull-up). The manufacturer has shut down; `tools/KUBLET.md` is the reverse-engineered hardware/flash/OTA reference. The root `README.md` describes the defunct `krate` CLI — `./tools/dev` replaces it.

## Commands

All run from the repo root. Python tooling uses `uv` (`uv sync --extra dev` for pytest/ruff/pillow).

```bash
./tools/dev build <app>              # compile firmware (PlatformIO)
./tools/dev deploy <app> [device]    # build + WiFi OTA; --skip-build, --ip <addr>
./tools/dev init --name <device>     # one-time USB flash of dev firmware + WiFi creds + server_url into NVS
./tools/dev logs [app]               # serial logs over USB
./tools/dev devices                  # registered devices (tools/.devices.yml)
./tools/dev config <device> [key=value...]  # read/write the NVS `app` namespace over WiFi; --no-restart

./tools/emulate <app>                                    # run in SDL2 desktop emulator (brew install cmake sdl2 ffmpeg)
./tools/emulate <app> --screenshot out.png --after 3      # --after is app time, not wall time
./tools/emulate <app> --gif apps/<app>/assets/preview.gif --gif-duration 6
./tools/emulate <app> --button-at "2,5:800"              # scripted presses: seconds[:duration_ms]
./tools/emulate <app> --generate-reference               # write apps/<app>/assets/reference.png

./server/run                          # local data server on :8198 (PORT env overrides)

uv run pytest tools/tests                                    # all tests
uv run pytest tools/tests/test_emulate.py -k "hn"            # one app's emulator tests
uv run pytest tools/tests/test_build.py::<TestClass>::<test> # single test
uv run ruff check tools server
```

Never build with raw `pio run`: `./tools/dev build` writes the TFT_eSPI `User_Setup.h` pin config first, and firmware built without it crashes the device.

## Architecture

**Apps (`apps/<name>/`)** are standalone PlatformIO projects: `platformio.ini`, a single `src/main.cpp`, `README.md`, and `assets/`. Shared libraries live in `lib/` and are pulled in with `symlink://../../lib/KGFX` and `symlink://../../lib/OTAServer` (copy `platformio.ini` from a recent app such as `hn` or `notice`).

Every app must keep this boilerplate verbatim — OTAServer is how the next app gets deployed, and the multi-app bundler design (`docs/design-plans/`) regex-strips these exact lines:
- globals `OTAServer otaserver;` and `KGFX ui;`
- in `setup()`: `otaserver.connectWiFi(); // DO NOT EDIT.` then `otaserver.run(); // DO NOT EDIT`, then `ui.init();`
- in `loop()`: `if (WiFi.status() == WL_CONNECTED) { otaserver.handle(); // DO NOT EDIT }`
- `pinMode(BUTTON_PIN, INPUT_PULLUP);` with `#define BUTTON_PIN 19`

Drawing goes through KGFX (`lib/KGFX/kgfx.h`): `ui.t` is the raw `TFT_eSPI`, `ui.tft` is a TTF-text wrapper (Arial / Arial Bold fonts at various sizes); helpers include `drawTextCenter`, `drawCentered`, `drawProgressArc`, `truncateText`, charts. Use `TFT_eSprite` buffers for flicker-free redraws, and redirect text into them with `ui.tft.TTFdestination(&spr)`. Keep `loop()` non-blocking (millis()-based timers, edge-detected button), use fixed-size buffers, and ArduinoJson `DeserializationOption::Filter` to bound memory.

**Configuration** comes from NVS via `Preferences`: namespace `core` holds WiFi creds; namespace `app` holds `server_url` (written by `dev init` over USB) and any app-specific keys. Since OTAServer 1.2.0 the device serves `/config`, so `./tools/dev config <device> key=value` writes the `app` namespace over WiFi and restarts the device (apps read NVS in `setup()`). New app settings need no tooling change, but keys are limited to 15 characters, and `GET /config` only reports `server_url`.

**Server (`server/src/`)** is a stdlib HTTP server for data that can't come from a public API (macOS Music, Notification Center, Claude usage). Subpackages of `server/src/apps/` are auto-discovered: each `__init__.py` exports `ROUTES = {"/api/<x>": handler}` and optionally `register()` for background threads. Handlers take `(log, cached, query_params)` and return a dict (JSON), `{"_binary": path, "_content_type": ...}`, `{"_html": ...}`, or `None` (503). See `server/README.md`.

**Emulator (`tools/src/emulate/`)** compiles an app's `main.cpp` with CMake against mocks in `mock/` (Arduino, TFT_eSPI, WiFi, HTTPClient, Preferences, etc.); `main_wrapper.cpp` drives setup/loop with SDL. Screenshot mode runs a virtual clock (`millis()` advances 5 ms per `loop()` pass plus any `delay()`) and seeds `random()` with a constant, so screenshots are deterministic; GIF and interactive modes use the wall clock. If an app uses a library or API the mocks lack, extend the mock rather than `#ifdef`-ing the app. `HTTPClient` is served from `apps/<app>/assets/http_fixtures.json` (`[{"url": "*glob*", "file": "x.json", "status": 200}]`); a file named `name_1.json` auto-advances to `name_2.json`, `name_3.json`… on successive matching requests. `assets/notifications.json` feeds the notice app.

**Tests (`tools/tests/test_emulate.py`)** auto-discover every `apps/*/src/main.cpp` and require: it compiles in the emulator, renders a non-blank screenshot after 3 s, matches `assets/reference.png` within 5% if present, and has `http_fixtures.json` if the source uses `HTTPClient`. `KNOWN_BROKEN` / `_NEEDS_DATA` sets list exceptions. Other tests cover the `kublet_dev` CLI (build, config, devices).

## Adding an app

Follow the shape of recent apps (`hn`, `weather`, `notice`, `dvd`, `life`, `snake`): `src/main.cpp`, `platformio.ini`, `README.md` (sections: Preview, Features, Configuration, Dependencies, Build & Deploy, Button), `assets/preview.gif` generated by the emulator, HTTP fixtures if networked, and optionally `assets/reference.png`. Verify with `uv run pytest tools/tests/test_emulate.py -k <app>` and `./tools/dev build <app>`.
