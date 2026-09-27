# Waveshare 1.64 V2 Target Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an `amoled-1.64-v2` firmware target that runs Taby on the Waveshare ESP32-S3-Touch-AMOLED-1.64 V2 exactly as V1 does.

**Architecture:** A new `boards.json` key that reuses V1's sdkconfig, partitions and animation pack. One compile definition (`TABY_HARDWARE_AMOLED_1_64_V2`) moves LCD chip-select from GPIO9 to GPIO46 and makes the firmware report `amoled-1.64-v2`. The host tools learn one optional `"assets"` key so V2 can reuse V1's pack without an 11 MB copy.

**Tech Stack:** ESP-IDF v5.4.2 (C, CMake), Python 3.11+ host tools with `unittest`, esptool 4.11.0.

**Spec:** `docs/specs/amoled-1.64-v2.md` (approved 2026-09-27).

## Global Constraints

- ESP-IDF **v5.4.2** exactly (`tools/build.py` refuses anything else). Local copy: `/Users/cloudninja/Documents/code/esp-idf` (sibling repo; used read-only via `export.sh`, **needs Luis's OK before first use**).
- Host Python: the repo's `.venv` for tests/tools; the ESP-IDF Python env (after `export.sh`) for `build.py` and `package_release.py`.
- No new dependencies. No sdkconfig or partition copies for V2.
- V1 (`amoled-1.64`) and round (`round-1.32`) behaviour unchanged.
- Never `erase_flash`, `--force`, or eFuse/security changes.
- Commits: plain imperative subject, no `Co-Authored-By` or generated-with trailers. Branch `feat/amoled-1.64-v2`. No push without Luis's go-ahead.
- Supported-display docs change only after the hardware test passes (`firmware/README.md` "adding a board" step 5).

## Review Focus

1. **Board already running V1 firmware reports `amoled-1.64`.** Luis's V2 board currently does. An update that trusts `identify` would pick V1 again. Pinned by: flashing with explicit `--board amoled-1.64-v2` (Task 4) and `verify_info` rejecting a V1 reply for a V2 bundle (Task 2 test).
2. **V2 bundle ships a different animation pack version than the firmware expects.** `package_release.py` must read the V1 pack's version for V2. Pinned by `asset_dir` test (Task 2) and `install.py inspect` of the built V2 bundle (Task 3).
3. **V2 build silently falls through to V1 pins** (misspelt define, CMake branch not taken). Screen stays black, `identify` still says V1. Pinned by the `strings` check for `amoled-1.64-v2` in the built image (Task 3) and `identify` on device (Task 4).
4. **V1 build changes by accident** while editing shared files. Pinned by the case-6 object comparison (Task 3).
5. **Undocumented V2 pin differences beyond CS.** No test can catch it; the hardware test (Task 4) has an explicit stop rule.

---

### Task 1: Baseline and toolchain gate

**Files:** none in the repo. Scratch: `$SCRATCH/v1-before/`, `$SCRATCH/compare_objects.py`.

`$SCRATCH` = `/private/tmp/claude-501/-Users-cloudninja-Documents-code-firmware-taby/2eb5075e-187c-4757-b98a-61b8c59cb6b3/scratchpad`

**Interfaces:**
- Produces: `$SCRATCH/v1-before/{board_amoled_1_64.c.obj,taby_build_info.c.obj}` and `compare_objects.py BEFORE_DIR AFTER_DIR NAME SHIFT` (exit 0 = equivalent).

- [ ] **Step 1: Confirm the existing V1 build matches the current source**

Must run before any repo edit (Task 2 changes `boards.json`, which is part of the digest).

```sh
.venv/bin/python -c "
import sys, json; sys.path.insert(0, 'tools'); import common
stamp = json.load(open('firmware/build-amoled-1.64/taby-build.json'))
print(common.source_digest() == stamp['source_sha256'])"
```
Expected: `True` (checked 2026-09-27). If `False`, rebuild V1 first (Step 4 commands with `amoled-1.64`) before copying.

- [ ] **Step 2: Save the baseline objects**

```sh
mkdir -p "$SCRATCH/v1-before"
cp firmware/build-amoled-1.64/esp-idf/main/CMakeFiles/__idf_main.dir/board_amoled_1_64.c.obj \
   firmware/build-amoled-1.64/esp-idf/main/CMakeFiles/__idf_main.dir/taby_build_info.c.obj \
   "$SCRATCH/v1-before/"
```

- [ ] **Step 3: Write the comparison script**

Firmware images are not byte-comparable (`CONFIG_APP_COMPILE_TIME_DATE=y`), and `ESP_RETURN_ON_ERROR` embeds `__LINE__`, so adding lines above the functions shifts those constants. The script accepts only that shift.

`$SCRATCH/compare_objects.py`:

```python
"""Case 6: two builds of one object may differ only by source-line constants shifted by SHIFT."""
import re
import subprocess
import sys
from pathlib import Path

OBJDUMP = str(Path.home() / ".espressif/tools/xtensa-esp-elf/esp-14.2.0_20241119"
              "/xtensa-esp-elf/bin/xtensa-esp32s3-elf-objdump")
NUMBER = re.compile(r"-?\d+")
DATA = (".rodata", ".data", ".bss", ".dram")


def dump(obj, *args):
    return subprocess.check_output([OBJDUMP, *args, str(obj)], text=True).splitlines()[2:]


def data_sections(obj):
    names = re.findall(r"^\s*\d+\s+(\S+)", "\n".join(dump(obj, "-h")), re.M)
    return sorted(n for n in names if n.startswith(DATA))


def main(before_dir, after_dir, name, shift):
    before, after = Path(before_dir) / name, Path(after_dir) / name
    code_b = dump(before, "-d", "-r", "--no-show-raw-insn")
    code_a = dump(after, "-d", "-r", "--no-show-raw-insn")
    if len(code_b) != len(code_a):
        sys.exit(f"{name}: code differs in length ({len(code_b)} vs {len(code_a)} lines)")
    shifted = 0
    for b, a in zip(code_b, code_a):
        if b == a:
            continue
        if NUMBER.sub("#", b) != NUMBER.sub("#", a):
            sys.exit(f"{name}: instruction changed\n  before: {b}\n  after:  {a}")
        deltas = {int(y) - int(x) for x, y in zip(NUMBER.findall(b), NUMBER.findall(a)) if x != y}
        if deltas != {shift}:
            sys.exit(f"{name}: constant changed by {sorted(deltas)}, expected only +{shift}\n  {b}\n  {a}")
        shifted += 1
    sections = data_sections(before)
    if sections != data_sections(after):
        sys.exit(f"{name}: data sections differ")
    for section in sections:
        if dump(before, "-s", "-j", section) != dump(after, "-s", "-j", section):
            sys.exit(f"{name}: {section} contents differ")
    print(f"{name}: equivalent ({shifted} line constants shifted by +{shift}, {len(sections)} data sections identical)")


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4]))
```

- [ ] **Step 4: Self-test the script**

```sh
python3 "$SCRATCH/compare_objects.py" "$SCRATCH/v1-before" "$SCRATCH/v1-before" board_amoled_1_64.c.obj 0
```
Expected: `board_amoled_1_64.c.obj: equivalent (0 line constants shifted by +0, N data sections identical)`.

- [ ] **Step 5: Toolchain gate — STOP for Luis**

Ask Luis to OK sourcing `/Users/cloudninja/Documents/code/esp-idf/export.sh` (sibling directory; read-only use). Then confirm:

```sh
. /Users/cloudninja/Documents/code/esp-idf/export.sh >/dev/null && python "$IDF_PATH/tools/idf.py" --version
```
Expected: `ESP-IDF v5.4.2`.

No commit (nothing in the repo changed).

---

### Task 2: Host tools know the V2 target

**Files:**
- Modify: `firmware/boards.json`
- Modify: `hardware/catalog.json`
- Modify: `tools/common.py` (add `asset_dir`)
- Modify: `tools/check.py:4,37` (use `asset_dir`)
- Modify: `tools/package_release.py:11,61` (use `asset_dir`)
- Test: `tests/test_device.py:58-62`, `tests/test_install.py`

**Interfaces:**
- Produces: `BOARDS["amoled-1.64-v2"]` with keys `name, revision="V2", chip, flash_size_mb=16, width=280, height=456, sdkconfig, partitions, assets="amoled-1.64"`.
- Produces: `common.asset_dir(board: str) -> pathlib.Path` — `ROOT / "assets" / BOARDS[board].get("assets", board)`.

- [ ] **Step 1: Write the failing tests**

`tests/test_device.py` — in `test_unknown_target_and_conflicting_geometry_do_not_select_a_bundle`, replace the V2 example with a target that stays unknown:

```python
        for actual in ({"hardware_target": "amoled-9.99"},
                       {"hardware_target": "amoled-1.64", "display_width": 466}):
```

and add after `test_supported_firmware_metadata_selects_target_without_claiming_physical_detection`:

```python
    def test_v2_firmware_metadata_selects_the_v2_target(self):
        result = device.identify({"hardware_target": "amoled-1.64-v2",
                                  "display_width": 280, "display_height": 456})
        self.assertEqual(result["status"], "firmware_target")
        self.assertEqual((result["board"], result["revision"]), ("amoled-1.64-v2", "V2"))
        self.assertFalse(result["physical_revision_verified"])
```

`tests/test_install.py` — change the import line to

```python
from common import BOARDS, allowed_images, asset_dir, load_bundle
```

and add before `if __name__ == "__main__":`:

```python
def write_bundle(root, board):
    profile = BOARDS[board]
    manifest = {"schema": "taby-install-v1", "board": board, "revision": profile["revision"],
                "chip": profile["chip"], "flash_size_mb": profile["flash_size_mb"],
                "firmware_version": "test", "assets_version": "test", "images": []}
    for kind, (offset, _) in allowed_images(board).items():
        content = (kind * 2).encode()
        (root / f"{kind}.bin").write_bytes(content)
        manifest["images"].append({"kind": kind, "file": f"{kind}.bin", "offset": offset,
                                   "size": len(content), "sha256": hashlib.sha256(content).hexdigest()})
    (root / "manifest.json").write_text(json.dumps(manifest))
    return manifest


class RevisionTests(unittest.TestCase):
    """1.64 V1 and V2 share a screen size but not display wiring."""

    def setUp(self):
        temp = tempfile.TemporaryDirectory()
        self.addCleanup(temp.cleanup)
        self.root = Path(temp.name)

    def test_v1_and_v2_bundles_never_cross(self):
        for built, selected in (("amoled-1.64", "amoled-1.64-v2"), ("amoled-1.64-v2", "amoled-1.64")):
            with self.subTest(built=built, selected=selected):
                write_bundle(self.root, built)
                load_bundle(self.root, built)
                with self.assertRaisesRegex(ValueError, "Release board does not match selected board"):
                    load_bundle(self.root, selected)

    def test_v1_firmware_does_not_verify_a_v2_install(self):
        manifest = write_bundle(self.root, "amoled-1.64-v2")
        actual = {"hardware_target": "amoled-1.64", "firmware_version": "test", "assets_version": "test"}
        with self.assertRaises(ValueError):
            verify_info(actual, manifest)

    def test_v2_reuses_the_v1_layout_and_animation_pack(self):
        self.assertEqual(allowed_images("amoled-1.64-v2"), allowed_images("amoled-1.64"))
        self.assertEqual(asset_dir("amoled-1.64-v2"), asset_dir("amoled-1.64"))
        self.assertEqual(asset_dir("round-1.32").name, "round-1.32")
```

- [ ] **Step 2: Run to verify they fail**

Run: `.venv/bin/python -m unittest tests.test_install tests.test_device -v`
Expected: `ImportError: cannot import name 'asset_dir'` for `test_install`; `test_v2_firmware_metadata_selects_the_v2_target` FAIL (status `unsupported`).

- [ ] **Step 3: Implement**

`firmware/boards.json` — insert after the `amoled-1.64` entry:

```json
  "amoled-1.64-v2": {
    "name": "Waveshare ESP32-S3-Touch-AMOLED-1.64 (V2)",
    "revision": "V2",
    "chip": "esp32s3",
    "flash_size_mb": 16,
    "width": 280,
    "height": 456,
    "sdkconfig": "sdkconfig.defaults",
    "partitions": "partitions.csv",
    "assets": "amoled-1.64"
  },
```

`tools/common.py` — add after `BOARDS = ...`:

```python


def asset_dir(board):
    """A board that shares another board's artwork names that pack in boards.json."""
    return ROOT / "assets" / BOARDS[board].get("assets", board)
```

`tools/check.py` — import line becomes `from common import BOARDS, ROOT, asset_dir, inside, partitions, sha256`; in `check_assets`, `root = ROOT / "assets" / board` becomes `root = asset_dir(board)`.

`tools/package_release.py` — import line gains `asset_dir`; line 61 becomes:

```python
    assets = json.loads((asset_dir(board) / "manifest.json").read_text())
```

`hardware/catalog.json` — insert after the `amoled-1.64` target (no case model: V2 fit is untested):

```json
    {
      "board": "amoled-1.64-v2",
      "revision": "V2",
      "display": {
        "shape": "rectangle",
        "width_px": 280,
        "height_px": 456
      },
      "models": [],
      "status": "prototype"
    },
```

- [ ] **Step 4: Run tests and checks**

Run: `.venv/bin/python -m unittest discover -s tests -v`
Expected: all pass (baseline 51 run, 1 skipped for Pillow; now 55 run).

Run: `.venv/bin/python tools/check.py`
Expected: includes `amoled-1.64-v2: 84 animations, icons, catalog hashes and partition layout verified` and `hardware: board revisions and display geometry verified`.

- [ ] **Step 5: Commit**

```sh
git add firmware/boards.json hardware/catalog.json tools/common.py tools/check.py tools/package_release.py tests/test_device.py tests/test_install.py
git commit -m "Teach the host tools about the 1.64 V2 target"
```

---

### Task 3: Firmware builds for V2; V1 unchanged

**Files:**
- Modify: `firmware/CMakeLists.txt:13-16`
- Modify: `firmware/main/CMakeLists.txt:33-38`
- Modify: `firmware/main/board_amoled_1_64.c:46-59`
- Modify: `firmware/main/taby_build_info.c:25-31`
- Modify: `.github/workflows/build.yml:29`

**Interfaces:**
- Consumes: `BOARDS["amoled-1.64-v2"]` (Task 2), `compare_objects.py` and baseline (Task 1).
- Produces: `dist/taby-amoled-1.64-v2/` bundle; firmware reports `hardware_target` = `amoled-1.64-v2`.

- [ ] **Step 1: Allow the target**

`firmware/CMakeLists.txt` lines 13-16 become:

```cmake
set_property(CACHE TABY_HARDWARE_TARGET PROPERTY STRINGS "amoled-1.64" "amoled-1.64-v2" "round-1.32")
if(NOT TABY_HARDWARE_TARGET STREQUAL "amoled-1.64"
   AND NOT TABY_HARDWARE_TARGET STREQUAL "amoled-1.64-v2"
   AND NOT TABY_HARDWARE_TARGET STREQUAL "round-1.32")
    message(FATAL_ERROR "Unsupported TABY_HARDWARE_TARGET=${TABY_HARDWARE_TARGET}")
endif()
```

- [ ] **Step 2: Define V2 and reuse the V1 pack**

`firmware/main/CMakeLists.txt` lines 33-38 become:

```cmake
if(TABY_HARDWARE_TARGET STREQUAL "round-1.32")
    target_compile_definitions(${COMPONENT_LIB} PRIVATE TABY_HARDWARE_ROUND_1_32=1)
    set(TABY_AMOLED_PACK_DIR "${CMAKE_CURRENT_LIST_DIR}/../../assets/round-1.32")
elseif(TABY_HARDWARE_TARGET STREQUAL "amoled-1.64-v2")
    # V2 moves LCD chip-select to GPIO46; same panel and artwork as V1.
    target_compile_definitions(${COMPONENT_LIB} PRIVATE TABY_HARDWARE_AMOLED_1_64_V2=1)
    set(TABY_AMOLED_PACK_DIR "${CMAKE_CURRENT_LIST_DIR}/../../assets/amoled-1.64")
else()
    set(TABY_AMOLED_PACK_DIR "${CMAKE_CURRENT_LIST_DIR}/../../assets/amoled-1.64")
endif()
```

- [ ] **Step 3: Move chip-select on V2**

`firmware/main/board_amoled_1_64.c` — in the `#else` (1.64) block, replace line 47 `#define PIN_NUM_LCD_CS GPIO_NUM_9` with:

```c
#if TABY_HARDWARE_AMOLED_1_64_V2
/* V2 swapped LCD_CS and IMU_INT1; GPIO9 is left unconfigured so it never drives the IMU. */
#define PIN_NUM_LCD_CS GPIO_NUM_46
#else
#define PIN_NUM_LCD_CS GPIO_NUM_9
#endif
```

and replace line 59 `#define BOARD_MODEL_LOG "1.64 AMOLED"` with:

```c
#if TABY_HARDWARE_AMOLED_1_64_V2
#define BOARD_MODEL_LOG "1.64 V2 AMOLED"
#else
#define BOARD_MODEL_LOG "1.64 AMOLED"
#endif
```

All edits sit above the first function, so every V1 `__LINE__` constant shifts by the file's net line change (expected +9).

- [ ] **Step 4: Report the V2 target**

`firmware/main/taby_build_info.c` — `taby_hardware_target` becomes:

```c
const char *taby_hardware_target(void) {
#if defined(TABY_HARDWARE_ROUND_1_32) && TABY_HARDWARE_ROUND_1_32
    return "round-1.32";
#elif defined(TABY_HARDWARE_AMOLED_1_64_V2) && TABY_HARDWARE_AMOLED_1_64_V2
    return "amoled-1.64-v2";
#else
    return "amoled-1.64";
#endif
}
```

- [ ] **Step 5: Build V1 and compare with the baseline (case 6)**

```sh
. /Users/cloudninja/Documents/code/esp-idf/export.sh >/dev/null
python tools/build.py amoled-1.64
mkdir -p "$SCRATCH/v1-after"
cp firmware/build-amoled-1.64/esp-idf/main/CMakeFiles/__idf_main.dir/{board_amoled_1_64.c.obj,taby_build_info.c.obj} "$SCRATCH/v1-after/"
SHIFT=$(( $(wc -l < firmware/main/board_amoled_1_64.c) - $(git show main:firmware/main/board_amoled_1_64.c | wc -l) ))
python3 "$SCRATCH/compare_objects.py" "$SCRATCH/v1-before" "$SCRATCH/v1-after" board_amoled_1_64.c.obj "$SHIFT"
python3 "$SCRATCH/compare_objects.py" "$SCRATCH/v1-before" "$SCRATCH/v1-after" taby_build_info.c.obj 0
```
Expected: both print `equivalent`. Any other output: stop and inspect the diff; do not adjust the script to pass.

- [ ] **Step 6: Build V2 and confirm the target string**

```sh
python tools/build.py amoled-1.64-v2
grep -c "amoled-1.64-v2" firmware/build-amoled-1.64-v2/taby_firmware.bin
grep -c "1.64 V2 AMOLED" firmware/build-amoled-1.64-v2/taby_firmware.bin
```
Expected: build succeeds; both counts ≥ 1.

- [ ] **Step 7: Package and inspect V2**

```sh
python tools/package_release.py amoled-1.64-v2
.venv/bin/python tools/install.py inspect --board amoled-1.64-v2 --bundle dist/taby-amoled-1.64-v2
```
Expected: `board: amoled-1.64-v2`, `revision: V2`, `assets: 0.3.3`, offsets `0x0 0x8000 0x19000 0x40000 0x440000` (same as V1).

- [ ] **Step 8: CI matrix**

`.github/workflows/build.yml` line 29 becomes:

```yaml
        board: [amoled-1.64, amoled-1.64-v2, round-1.32]
```

- [ ] **Step 9: Commit**

```sh
git add firmware/CMakeLists.txt firmware/main/CMakeLists.txt firmware/main/board_amoled_1_64.c firmware/main/taby_build_info.c .github/workflows/build.yml
git commit -m "Build firmware for the 1.64 V2 with LCD chip-select on GPIO46"
```

---

### Task 4: Hardware test on Luis's V2 board — gated

**Files:** none. Results recorded for the PR in `$SCRATCH/v2-hardware-results.md`.

**Interfaces:**
- Consumes: `dist/taby-amoled-1.64-v2/` (Task 3).

- [ ] **Step 1: STOP — ask Luis to plug the V2 board in**, then:

```sh
.venv/bin/python tools/device.py ports
```
Expected: one `USB JTAG/serial debug unit` port (was `/dev/cu.usbmodem101`).

- [ ] **Step 2: Flash with the explicit V2 board**

The board currently runs V1 firmware and reports `amoled-1.64`; do not select the board from `identify`.

```sh
.venv/bin/python tools/install.py flash --board amoled-1.64-v2 --bundle dist/taby-amoled-1.64-v2 --port PORT --confirmed-board
```
Expected: ends with `"verified": true` and `"hardware_target": "amoled-1.64-v2"`.

- [ ] **Step 3: Identify and animate (cases 4 and 1)**

```sh
.venv/bin/python tools/device.py identify --port PORT
.venv/bin/python tools/device.py animation --port PORT --id confirmation
```
Expected: `status: firmware_target`, `board: amoled-1.64-v2`, `revision: V2`; animation `accepted: true`.

- [ ] **Step 4: Capture the boot log**

Reset and read ~8 s of serial output (same method as the 2026-09-27 diagnosis). Expected lines: `1.64 V2 AMOLED board bring-up complete`, touch and IMU init without errors.

- [ ] **Step 5: STOP — Luis checks the device**

Ask Luis, and record each answer:
1. Startup face visible; `confirmation` correct — colours, no cropping, not mirrored. (case 1)
2. Touch responds. (case 2)
3. Tilting the board changes orientation. (case 3)

**Stop rule:** if the screen is still black, stop. Do not change further pins; get Waveshare's V2 schematic or demo source first (spec, Risks). If touch or tilt fail but the screen works, record it and ask Luis whether to ship display-only or investigate.

---

### Task 5: Docs for the supported V2

Only after Task 4 cases 1–3 pass.

**Files:**
- Create: `firmware/targets/amoled-1.64-v2/README.md`
- Modify: `INSTALL.md:22-30,95-96,151,224`
- Modify: `README.md:45`
- Modify: `firmware/README.md:20-21,33-34,156-157`

- [ ] **Step 1: Target README**

`firmware/targets/amoled-1.64-v2/README.md`:

````markdown
# Taby 1.64 V2 firmware target

Waveshare ESP32-S3-Touch-AMOLED-1.64 **V2** (version silkscreen on the right
side of the PCB, near the pin headers). Same panel, touch, IMU, flash, and
artwork as V1; the difference that matters to firmware:

| Signal | V1 | V2 |
| --- | --- | --- |
| LCD_CS | GPIO9 | GPIO46 |
| IMU_INT1 | GPIO46 | GPIO9 |

This target reuses V1's `sdkconfig.defaults`, `partitions.csv`, and
`assets/amoled-1.64`. V2-only pins (LCD_TE GPIO45, TP_INT GPIO18,
IMU_INT2 GPIO17) are not used yet.

```bash
python tools/build.py amoled-1.64-v2
python tools/package_release.py amoled-1.64-v2
```

A V1 image on a V2 board, or the reverse, leaves the screen black.
Source: [Waveshare 1.64 documentation](https://docs.waveshare.com/ESP32-S3-Touch-AMOLED-1.64).
````

- [ ] **Step 2: INSTALL.md**

- Board table: add row `| \`amoled-1.64-v2\` | Waveshare ESP32-S3-Touch-AMOLED-1.64 **V2** | 16 MB |` after the V1 row.
- Line 28: replace `The 1.64 V2 is unsupported.` with `The 1.64 V1 and V2 need different bundles: V2 moved the display chip-select, so the other revision's image leaves the screen black.`
- Lines 95-96: `` `amoled-1.64` currently maps to V1; `amoled-1.64-v2` to V2; `round-1.32` maps to the original round board. ``
- Line 151: add `taby-amoled-1.64-v2.zip` to the download list.
- Line 224: if BOOT recovery was exercised on V2 in Task 4, change to `**1.64 V1 and V2:**`; otherwise add `- **1.64 V2:** same buttons as V1 (V2 adds an RC circuit to BOOT); not yet tested.`

- [ ] **Step 3: README.md and firmware/README.md**

- `README.md:45`: `<p><strong>Waveshare ESP32-S3-Touch-AMOLED-1.64</strong> (V1 or V2 — check the PCB marking)</p>`
- `firmware/README.md`: add `python tools/build.py amoled-1.64-v2` and `python tools/package_release.py amoled-1.64-v2` to the two command blocks; replace `Waveshare 1.64 V2 needs its own reviewed pin mapping and hardware tests.` with `The Waveshare 1.64 V2 is its own target, \`amoled-1.64-v2\`; see [its README](targets/amoled-1.64-v2/README.md).`

- [ ] **Step 4: Checks**

Run: `.venv/bin/python tools/check.py && .venv/bin/python -m unittest discover -s tests`
Expected: pass. Note `source_digest` now includes the new target README; the Task 3 build stamp goes stale, so rebuild V2 before packaging anything for release.

- [ ] **Step 5: Commit**

```sh
git add firmware/targets/amoled-1.64-v2/README.md INSTALL.md README.md firmware/README.md
git commit -m "Document the 1.64 V2 as a supported board"
```

---

### Task 6: Hand-off — gated

- [ ] **Step 1: Whole-branch review** (fresh reviewer, most capable model) against the spec and this plan.
- [ ] **Step 2: STOP — Luis decides** push of `feat/amoled-1.64-v2` to `origin` (`lmunoz0806/firmware-taby`). Copy-paste: `git push -u origin feat/amoled-1.64-v2`.
- [ ] **Step 3: Draft the upstream PR text** (results of cases 1–7, the case-6 comparison output, what is unverified: app compatibility, V2 case fit, BOOT recovery if untested). Opening it against `TRIIIS-LABS/firmware-taby` is Luis's separate go-ahead. Luis also decides whether `docs/specs/` travels upstream.
