# Spec: Waveshare 1.64 V2 board target

Status: draft for review · 2026-09-27 · Branch `feat/amoled-1.64-v2`

## What and why

Add a firmware target for the **Waveshare ESP32-S3-Touch-AMOLED-1.64 V2** so it
runs Taby the way the V1 does.

A V2 board flashed with the official 1.2.0 `amoled-1.64` bundle boots, answers
over USB, and passes `install.py verify`, but the screen stays black. The same
board showed Waveshare's clock demo before flashing, so the panel works.
The boot log reports `1.64 AMOLED board bring-up complete` and
`render_state=ambient_startup ... rendered=1`, so the firmware thinks it is
drawing.

Cause, from [Waveshare's 1.64 page](https://docs.waveshare.com/ESP32-S3-Touch-AMOLED-1.64):

| Signal | V1 | V2 |
| --- | --- | --- |
| LCD_CS | GPIO9 | GPIO46 |
| IMU_INT1 | GPIO46 | GPIO9 |
| LCD_TE | — | GPIO45 (new) |
| TP_INT | — | GPIO18 (new) |
| IMU_INT2 | — | GPIO17 (new) |

The firmware selects the panel on GPIO9 (`firmware/main/board_amoled_1_64.c:47`),
so on V2 the panel is never selected. Same FT3168 touch and QMI8658 IMU on both.
Waveshare does not publish the full V2 QSPI/I2C pin list; that the other
display pins are unchanged is **an inference**, which the hardware test below
confirms or refutes.

## Scope

In:

- New board key `amoled-1.64-v2`, revision `V2`, behaving exactly like V1:
  same resolution, animations, partitions, and features.
- LCD_CS on GPIO46. GPIO9 is not configured by the firmware (stays input), so it
  never drives against the IMU's INT1 output.
- Release packaging, CI build, installer, and docs for the new target.

Out (revisit when tearing or touch lag is seen on a V2 in use):

- LCD_TE (GPIO45) tear-free updates.
- Touch interrupt (GPIO18); touch stays polled as on V1.
- IMU interrupts (GPIO9, GPIO17); the IMU stays polled over I2C as on V1.
- V2 battery-charging behaviour; no firmware involvement.

## Design

### Why a separate board key

The tools resolve everything from one key with one revision
(`tools/common.py:7`, `firmware/boards.json`), and the running firmware reports
only `hardware_target`, not a revision. Reusing `amoled-1.64` would make
`identify` unable to tell V1 from V2, so an update could put V2 firmware on a
V1 (black screen) or the reverse. A distinct target string keeps
`identify`, `verify`, and bundle selection exact. The repo already uses
`amoled-1.64-v2` as its example unsupported target (`tests/test_device.py:59`).

### Changes

| File | Change |
| --- | --- |
| `firmware/boards.json` | Add `amoled-1.64-v2`: name, revision `V2`, esp32s3, 16 MB, 280×456, V1's `sdkconfig.defaults` and `partitions.csv`, `"assets": "amoled-1.64"`. |
| `firmware/CMakeLists.txt` | Accept `amoled-1.64-v2` in the target allow-list. |
| `firmware/main/CMakeLists.txt` | For V2, define `TABY_HARDWARE_AMOLED_1_64_V2=1` and use the `assets/amoled-1.64` pack. |
| `firmware/main/board_amoled_1_64.c` | In the 1.64 pin block, `PIN_NUM_LCD_CS` is `GPIO_NUM_46` when the V2 define is set, else `GPIO_NUM_9`. Log names the revision. |
| `firmware/main/taby_build_info.c` | V2 build reports `hardware_target` = `amoled-1.64-v2`. Shape and dimensions unchanged. |
| `tools/package_release.py` | Read the asset manifest from `BOARDS[board].get("assets", board)`. |
| `.github/workflows/build.yml` | Add `amoled-1.64-v2` to the firmware matrix. |
| `tests/test_device.py` | Replace the `amoled-1.64-v2` unsupported example with a genuinely unknown target; add V2 identify case. |
| `tests/test_install.py` | Cross-board bundle refusal between V1 and V2. |
| `README.md`, `INSTALL.md`, `firmware/README.md`, `firmware/targets/amoled-1.64-v2/README.md` | Board tables, V1/V2 identification, build command. |

No sdkconfig or partition copy: V2 points at V1's files so they cannot drift.
No new dependencies. The V1 and round code paths gain no new branches beyond
the `#if` on one pin define and one target string.

### Error states

- Wrong bundle for the selected board: `load_bundle` already raises
  `Release board does not match selected board`; covered by a new test.
- V2 firmware on a V1 board: black screen (CS on the V1's IMU_INT1 pin).
  Recoverable by flashing the V1 bundle. `identify` on that board reports
  `amoled-1.64-v2`, which INSTALL.md tells the user is compiled-in, not measured.

## Acceptance cases

Agreed 2026-09-27. Each is checked and recorded in the PR.

1. A V2 build flashed on the V2 board shows the startup face, and
   `confirmation` plays with correct colours, no cropping, no mirroring. *(Luis, on device)*
2. Touch works on V2. *(Luis, on device)*
3. Tilt/orientation works on V2. *(Luis, on device)*
4. `device.py identify` on the V2 board returns board `amoled-1.64-v2`, revision `V2`.
5. `install.py` refuses a V1 bundle with `--board amoled-1.64-v2`, and a V2
   bundle with `--board amoled-1.64`. *(unit test)*
6. V1 and round are unchanged: existing tests pass; the V1 build's compiled
   objects for `board_amoled_1_64.c` and `taby_build_info.c` disassemble
   identically before and after. (Firmware images are not byte-comparable:
   `CONFIG_APP_COMPILE_TIME_DATE=y` embeds the build time.)
7. The Taby desktop app connects to the V2 board. **Unverified:** the app may
   allow-list `hardware_target` values. *(Luis, app side)*

## Risks

- **Other V2 pin changes** not in Waveshare's table. Detected by case 1–3; if
  the screen stays black with CS on GPIO46, stop and get Waveshare's V2
  schematic or demo source before changing more pins.
- **App compatibility** (case 7). If the app rejects the new target, the
  firmware still works over USB commands; the app needs a matching update.
- **Upstream acceptance.** Work lands on the fork `lmunoz0806/firmware-taby`;
  a PR to `TRIIIS-LABS/firmware-taby` is a separate, explicit step.

## Rollout and rollback

1. Build `amoled-1.64-v2` locally with ESP-IDF v5.4.2, package, `inspect`.
2. Flash the V2 board, run cases 1–4, record results.
3. Run tests and the case-6 comparison; push the branch to the fork.
4. Upstream PR only on Luis's go-ahead.

Rollback: board — flash any other bundle or the Waveshare demo; the ESP32-S3
ROM bootloader is not writable. Code — revert the branch; no existing target
changes behaviour.
