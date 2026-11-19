# BITS firmware kit

A modular firmware kit for ESP32 boards (ESP-IDF 6.1). A product is a board choice plus a list
of feature switches; the kit turns that into a partition table, an sdkconfig and a boot sequence,
and builds it from three reusable layers. This repository is the parent that holds the three
layers side by side as git submodules.

Contents: [What this is](#1-what-this-is) | [Repository layout](#2-repository-layout) |
[Requirements](#3-requirements) | [Getting the code](#4-getting-the-code) |
[Profiles](#5-products-and-profiles) | [Build](#6-build-a-firmware-step-by-step) |
[Flashing](#7-flashing) | [Product notes](#8-product-notes) | [Editing and rules](#9-editing-code-and-the-rules) |
[Updating this repo](#10-updating-the-parent-repo-when-a-layer-changes) |
[Troubleshooting](#11-troubleshooting) | [Status and limits](#12-status-and-limits)

## 1. What this is

The kit extracts the shared parts of several ESP32 firmwares into layers, so that a new product is
a small folder (`product.cfg`, a few files of its own) and not a copy of a whole project.

```
        +-----------------------------------------------------------+
        |  fw-app       protocols, MQTT, scale stack, Modbus,       |
        |               VFD application (Python), templates,        |
        |               new_product.py                              |
        +-----------------------------+-----------------------------+
                                      | depends on
        +-----------------------------v-----------------------------+
        |  fw-logic     pure rules: dispense_core (adaptive only),  |
        |               VFD rules (Python). No hardware, no network |
        +-----------------------------+-----------------------------+
                                      | depends on
        +-----------------------------v-----------------------------+
        |  fw-platform  boards, Wi-Fi and Ethernet, provisioning,   |
        |               OTA, serial HAL, boot engine, MicroPython   |
        |               runtime, fw_gen.py                          |
        +-----------------------------------------------------------+
```

Dependency direction: app -> logic -> platform. A lower layer never depends on a higher one
(`fw-platform/tools/check_layers.py` checks it).

| Layer | Contains | Switches it owns (`fw_features.json`) |
|---|---|---|
| `fw-platform` | boards, networking, provisioning, OTA, serial HAL, the boot engine, MicroPython runtime, `fw_gen` | `SERIAL ETHERNET WIFI PROVISIONING OTA MDNS CONSOLE MICROPYTHON` |
| `fw-logic` | `dispense_core` (adaptive only), the VFD rules (Python) | `VFD_RULES` |
| `fw-app` | Modbus master and bridge, the scale stack, MQTT, the VFD application (Python), templates, `new_product` | `MODBUS MQTT SCALE_LINK SCALE_PUBLISH VFD_POLLING` |

A product states its shape in `product.cfg` (switches, board, flash size). `fw_gen.py` turns that
into `gen/` (partition table, sdkconfig defaults, boot table, MicroPython bindings) and refuses a
switch set that does not hold together, for example a switch whose layer is not pulled.

## 2. Repository layout

This repository contains no firmware source of its own: only `README.md`, `.gitignore`,
`.gitmodules` and the three submodule entries. `.gitmodules` lists the path and URL of each layer
(no branch is set); the exact commit of each layer is recorded in this repository's tree (see
`git submodule status`).

| Layer | Repo URL | Role | Main folders |
|---|---|---|---|
| `fw-platform` | https://github.com/Wendell-BitsDev/fw-platform.git | base layer, shared services | `components/`, `cmake/`, `python/`, `tools/` (`fw_gen.py`, `check_layers.py`, `check_image_fit.py`, `compare_names.py`, `strict_ratchet.ps1`), `tests/`, `docs/` |
| `fw-logic` | https://github.com/Wendell-BitsDev/fw-logic.git | pure rules, safety-critical code | `components/dispense_core`, `python/`, `tests/` (`host`, `python`, `qemu`), `tools/`, `docs/` |
| `fw-app` | https://github.com/Wendell-BitsDev/fw-app.git | application layer and product templates | `components/`, `python/`, `templates/` (`PROFILES.json`, one folder per profile), `tools/` (`new_product.py`, `run_all.ps1`), `tests/` (`qemu`, `qemu2`, `golden`, `python_company`), `docs/` |

Each layer has its own README, `VERSION`, `fw_features.json` and `docs/` (inventories, baselines,
notes). The layers are siblings on purpose: `new_product.py` and `run_all.ps1` find the other
layers as sibling folders of `fw-app`.

## 3. Requirements

- Windows with PowerShell. The kit is built and tested from native PowerShell, not Git Bash.
- ESP-IDF 6.1, with its PowerShell profile at
  `C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1`.
- git, and Python (used by `new_product.py` and the tools; the Python tests use `unittest`, no
  pytest).
- A board with 16 MB flash (every profile in `PROFILES.json` sets `flash_mb` to 16).
- At least 1.5 GB of free memory before a build (builds are heavy, the MicroPython one most of all).
- Internet access on the first build: two managed components are downloaded by the IDF component
  manager (`espressif/mqtt` and `espressif/mdns`, both declared with an unpinned or loose version).
- Access to the three layer repositories. They are PRIVATE: without access to all three
  (`fw-platform`, `fw-logic`, `fw-app`) the submodule clone fails.

## 4. Getting the code

```
git clone --recurse-submodules https://github.com/Wendell-BitsDev/BITS.git
```

If you cloned without `--recurse-submodules` (the layer folders are empty):

```
git submodule update --init --recursive
```

Check what you have:

```
git submodule status
```

Each line shows the commit of one layer; a leading `-` means the layer is not checked out yet, a
leading `+` means the layer is at a different commit than this repository records.

After a clone each submodule is on a detached HEAD: it sits on the exact commit this repository
records, not on a branch, so a commit made there would belong to no branch. Before editing a layer
switch to its branch:

```
cd fw-app
git switch main
```

## 5. Products and profiles

`python fw-app\tools\new_product.py --list` prints the profiles; they are defined in
`fw-app/templates/PROFILES.json` (one entry plus one template folder per profile).

| Profile | Board | Purpose | Protocol / transport | Switches | Status |
|---|---|---|---|---|---|
| `node` | `ESP32_LAN_RS485_V1_0_2` | network node: Wi-Fi, Ethernet, provisioning portal, OTA, mDNS (fw-platform alone) | none | `WIFI ETHERNET PROVISIONING OTA MDNS` | scaffold and build covered by `run_all.ps1` |
| `scale-flintec-rs232` | `RS232_V3` | scale weight to MQTT, Flintec indicator on RS-232 | Flintec FT-111 / FT210 continuous output, streamed, 9600 8N1, kg | `WIFI ETHERNET PROVISIONING MQTT SERIAL SCALE_LINK SCALE_PUBLISH` | matches the Flintec indicator in use; NOT verified on the bench |
| `scale-cas-rs485` | `ESP32_LAN_RS485_V1_0_2` | scale weight to MQTT, CAS CI-150A on RS-485 | CAS CI-150A polled (`ddRW`, device id 1, 9600 8N1) | same as above | matches the CAS CI-150A in use; NOT verified on the bench |
| `scale-rs232` | `RS232_V3` | variant: CAS on RS-232 | CAS CI-150A polled, 9600 8N1 | same as above | variant, no known matching hardware |
| `scale-rs485` | `ESP32_LAN_RS485_V1_0_2` | variant: Flintec on RS-485 | Flintec continuous output, 9600 8N1, kg | same as above | variant, no known matching hardware |
| `vfd-modbus` | `ESP32_LAN_RS485_V1_0_2` | VFD gateway: MicroPython application polls drives on both RS-485 buses, state over MQTT | Modbus RTU, read only (function codes 1 and 3), MQTT | `WIFI ETHERNET PROVISIONING OTA MQTT MODBUS MICROPYTHON VFD_RULES VFD_POLLING SERIAL` | NOT verified on the bench |

Common to the four scale profiles: weight is published to `cas/<device>/telemetry/weight`, OTA is
off, and only fw-platform and fw-app are pulled. `vfd-modbus` pulls all three layers and has two OTA
slots. `mp_node` (the platform's own MicroPython test product, `fw-platform/tests/products/mp_node`)
is a test product, not a profile.

The RS-485 board is `ESP32_LAN_RS485_V1_0_2` (display name `ESP32_lan_rs485_v1.0.2`).

Feature switches: see `fw-app/templates/FEATURES.md` and `python fw-app\tools\new_product.py --list-features`.

Measured image sizes (IDF 6.1, from `fw-platform/docs/MICROPYTHON.md`): `mp_node.bin` 1,214,112
bytes (16.5 percent of the 0x700000 slot `ota_0`); a scratch product with the Modbus bridge
1,248,592 bytes; the `vfd-modbus` template 1,376,192 bytes (18.7 percent of `ota_0`). No measured
size for the scale profiles is recorded in the layer docs.

## 6. Build a firmware step by step

Always native PowerShell with the ESP-IDF profile, never Git Bash. One heavy command at a time.

**1. Prepare the shell**

```
. C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1
$env:IDF_PY_BUILD_JOBS = 1
(Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory     # KB; need at least 1572864
```

**2. Create the product** (from the root of this clone; the destination must not
exist or must be empty, and must not be inside a layer working tree). Worked example, the Flintec
scale on RS-232:

```
python fw-app\tools\new_product.py --list
python fw-app\tools\new_product.py scale-flintec-rs232 C:\products\scale1
```

The default mode is `--layers link`: `modules/<layer>` in the product are directory junctions to
the layer folders next to `fw-app`. The scaffold runs `git init -b main` in the product and makes
no commit. Keep the product path short on Windows. `--name` sets the project name (default: the
destination folder name), which is also the name of the `.bin`.

**3. Build and check the fit**

```
cd C:\products\scale1
idf.py set-target esp32 build
python modules\fw-platform\tools\check_image_fit.py build gen\partitions.csv
```

Configure first runs `fw_gen.py generate`, so `gen/` always matches `product.cfg`. After editing
`product.cfg` or `sdkconfig.defaults`, run `idf.py set-target esp32` again. The first build needs
the internet for the two managed components. Expected end of the build output:

```
Project build complete. To flash, run:
```

`check_image_fit.py` takes `<build_dir> <partitions.csv>` and checks the image against the
partition table (an OTA profile must have two app slots).

**4. What the build leaves behind** (standard ESP-IDF layout; `scale1` is the project name):

```
C:\products\scale1\
    product.cfg  sdkconfig.defaults  main\  components\product_cfg\
    gen\                         generated: partitions.csv, sdkconfig.defaults.gen, ...
    modules\                     junctions (link mode) or submodules to the layers
    build\
        scale1.bin               the application image
        bootloader\bootloader.bin
        partition_table\partition-table.bin
        flash_args               the files and offsets to flash
        (ota_data_initial.bin    only on an OTA profile)
```

Do not commit `build/`, `build-work/` or `managed_components/` (this repo's `.gitignore` lists them).

**5. Collect the files to hand over** (everything named in `build\flash_args`, with the same
relative paths so that the offsets keep working):

```
$out = 'C:\products\scale1-release'
New-Item -ItemType Directory -Force $out | Out-Null
Set-Location C:\products\scale1\build
Copy-Item flash_args $out
foreach ($f in (Get-Content flash_args) -split '\s+' | Where-Object { $_ -match '\.bin$' }) {
    $dest = Join-Path $out $f
    New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
    Copy-Item $f $dest
}
```

(The snippet follows the usual `flash_args` format of offset and file pairs; it was written from
that format and not run against a real build here.)

## 7. Flashing

Always flash the bootloader, the partition table and the application together; the `flash_args`
file lists all of them with their offsets. From the `build` folder (or the collected folder), with
the ESP-IDF profile loaded and the board on its serial port:

```
python -m esptool --chip esp32 -p COM5 -b 460800 --before default-reset --after hard-reset write-flash "@flash_args"
```

Or let ESP-IDF do it from the product folder, and open the monitor:

```
idf.py -p COM5 flash monitor
```

Replace `COM5` with your port. These flashing commands are the standard ESP-IDF ones and were not
run for this README.

## 8. Product notes

**Scale, Flintec on RS-232 (`scale-flintec-rs232`)** - set the indicator to:

- continuous output format, CR and LF on, address 00;
- checksum output (CSUM) OFF (with it on, about 1 in 256 weight values cannot confirm the link:
  no weight is published, never a wrong one);
- blank tare (a net frame publishes the net weight);
- a unit that matches the firmware: `CONFIG_FLINTEC_UNIT_KG` (default) or `CONFIG_FLINTEC_UNIT_LB`
  in `sdkconfig.defaults`; the frame carries no unit, so the firmware cannot detect a mismatch.

The indicator must drive its TX line idle-high; the RX pin of channel 1 has no internal pull-up.
9600 baud, 8N1.

**Scale, CAS CI-150A on RS-485 (`scale-cas-rs485`)** - 9600 baud, 8N1, device id 1 (the
indicator's COMM.01), and the indicator in Command Type 1 (CMD x.4). The firmware polls with
`ddRW` requests. The MAX13487E transceiver switches direction by itself (no DE/RE pin). A/B
polarity, ground and termination are things to check on the bench.

**Weight message (all scale profiles)** - topic `cas/<device id>/telemetry/weight`, QoS 0, not
retained, one message per publish period (500 ms by default) while a fresh weight exists and MQTT
is connected. The field `weight_g` is signed integer grams. A weight older than 3000 ms, an
overload, an error frame or a down link publishes nothing (never a zero or a last value).

**`vfd-modbus`**

- Wi-Fi credentials are never committed: copy
  `components/product_cfg/include/wifi_primary.local.h.example` to `wifi_primary.local.h`
  (git-ignored) and put them there. The provisioning portal sets the network at run time.
- The calibration files `cal/devices.json` (drive list, empty by default) and `cal/uart.json` live
  on the `scripts` littlefs partition. The kit's build does not put them there. On a blank unit
  (empty `scripts` partition) the Python application starts with no drive and the image rolls back;
  put the `cal/` files on the unit first, then add the drives. A unit that already holds them keeps
  them across an OTA of this image.
- Drive writes are off by default: the Modbus link only sends function codes 1 and 3; the write
  functions raise `ValueError` unless the Kconfig `FW_MODBUS_ALLOW_WRITE` is on, and it is `n`.
- The state topic is `scada/vfd/<device name>/state`. The broker address used by the Python
  application comes from `app/mqtt_config.py`, not from the C placeholder in `product_cfg.c`.
- OTA keeps two slots; the Python application confirms the running image, otherwise the image is
  rolled back.

Each template folder under `fw-app/templates/<profile>/` has a README with the full details.

## 9. Editing code and the rules

**Link mode and submodule mode.** `new_product.py --layers link` (the default) makes
`modules/<layer>` a junction to the working tree of the layer on this machine; `modules/` is
git-ignored then. `--layers submodule` runs `git submodule add` for each layer (default URL: the
layer repos; the ref comes from `fw-app/templates/LAYER_TAGS.json`, currently `HEAD` for all three,
no layer is tagged yet) and is the mode for sharing a product.

**Never edit inside `modules/` of a product.** In link mode it is the real layer, uncommitted and
visible to every product. Change a layer in its own repo, with its own tests.

**Where your own logic goes:** the product's `main/` and `components/product_cfg` (key table,
broker and Wi-Fi defaults), `product.cfg`, `sdkconfig.defaults`.

**Safety-listed areas.** The stale-weight path (`scale_reader`, `weight_source`, `scale_link`,
`scale_publish`, the Flintec decoder) and the OTA mark-valid step need independent review and a
bench test before use. The `scale_link` outlier filter is ON; do not set `outlier_filter_off`
without an explicit per-product decision. `dispense_core` (fw-logic) and the existing scale stack
are not changed by product work.

**Strict warnings.** New C code compiles clean under a strict warning set in error mode (enforced
component: `scale_publish`, in `fw-app/tests/qemu2`). Existing code is held by a ratchet: the
warning counts may only fall. Exceptions are in `docs/DEVIATIONS.md` of each layer. This is build
hardening, not MISRA C and not a compliance claim (details: `fw-app/docs/STRICT_WARNINGS.md`).

**Tests.** From the folder that holds the three layers:

```
fw-app\tools\run_all.ps1 -ShowPaths     # print what it would do, create nothing
fw-app\tools\run_all.ps1 -Quick         # Python tests and template scaffolding only (no QEMU, no idf.py)
fw-app\tools\run_all.ps1 -TemplatesOnly # only the template steps (with -Quick: scaffold only)
fw-app\tools\run_all.ps1 -StrictOnly    # only the warning ratchet builds and checks
fw-app\tools\run_all.ps1                # everything (long)

python -m unittest discover -s fw-platform\tools\tests -p "test_*.py"
python -m unittest discover -s fw-platform\tools -p "test_*.py"
python -m unittest discover -s fw-app\tools -p "test_*.py"
python fw-app\tools\run_python_tests.py
```

QEMU suites, one per repo: `tools\run_qemu_tests.ps1` inside `fw-platform`, `fw-logic` and `fw-app`
(`fw-app` has two projects: `tests\qemu` is the full one, new suites go in `tests\qemu2`, run with
`-Project qemu2`). Each runner kills every `qemu-system-xtensa` process. A pass is
`TEST_SUITE_END` with `fail=0` and `ALL TESTS PASSED`. Names of the passing checks are compared with
the baselines in `<repo>\docs\baseline\*.txt`:

```
python fw-platform\tools\compare_names.py <baseline.txt> <log.txt>
```

Run ONE heavy command (idf.py, QEMU, `run_all.ps1`) at a time, with `IDF_PY_BUILD_JOBS=1` and
at least 1.5 GB free memory.

## 10. Updating the parent repo when a layer changes

1. Change code inside a layer (for example `fw-app`): `git switch main`, commit and push there first.
2. In this repository: `git add fw-app` (or the layer you changed), commit, push.
3. Everyone else runs: `git pull --recurse-submodules`

Order matters: if this repository is pushed before the layer, the pointer names a commit that
nobody else can fetch.

To move a pointer to a specific commit or branch:

```
cd fw-app
git fetch
git switch main        # or: git checkout <commit-or-tag>
git pull               # when on a branch
cd ..
git add fw-app
git commit -m "Update fw-app"
git push
```

`git status` in the parent shows `modified: fw-app (new commits)` while a layer is ahead of the
recorded pointer.

## 11. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `CMakeLists.txt not found in project directory` | `idf.py` was run outside the product folder, or in this parent repo. `cd` into the product (the folder `new_product.py` created) first. |
| `idf.py` is not recognised | the ESP-IDF 6.1 profile is not loaded in this PowerShell: run `. C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1`. Use native PowerShell, not Git Bash. |
| `fw-platform`, `fw-logic` or `fw-app` folders are empty | plain clone: run `git submodule update --init --recursive`. If it fails, you lack access to a private layer repo. |
| `fw_gen.py not found: the platform layer is not pulled` (product configure) | `modules/fw-platform` is missing: in submodule mode run `git submodule update --init`; in link mode scaffold again so the junctions exist. |
| build stops or the machine swaps | free memory below 1.5 GB: close other programs, keep `IDF_PY_BUILD_JOBS=1`, run one heavy command at a time. |
| `new_product.py` refuses the destination | it never writes into an existing non-empty folder, nor into a layer working tree. Choose a new path. |
| you want to delete a link-mode product | never `Remove-Item -Recurse`: Windows PowerShell 5.1 follows the junctions under `modules/` and deletes the layers' real, possibly uncommitted contents. Use `python fw-app\tools\new_product.py --remove <dest>` (it unlinks the junctions first and needs the `.fw_product` marker). |
| layer is on `HEAD detached` | normal after a clone; `git switch main` inside the layer before editing. |
| first build fails while downloading components | needs internet for `espressif/mqtt` and `espressif/mdns` on the first build. |

## 12. Status and limits

- Hardware bench tests are pending for the scale profiles (Flintec on RS-232, CAS CI-150A on
  RS-485), for the OTA mark-valid step and for the VFD gateway. The bench checklist is
  `fw-app/docs/BENCH_CHECKLIST.md` (sections: PC and board only, Flintec on RS-232, CAS CI-150A on
  RS-485, publisher and scale link, MicroPython runtime and OTA mark-valid, VFD gateway, decisions,
  not simulated). Simulator (QEMU) and host tests do not replace them.
- MISRA C is not adopted. Strict compiler warnings are a build-hardening measure, not a
  compliance claim.
- No safety certification of any kind is claimed.
- No licence has been chosen for this kit; all rights are reserved by default. The vendored
  MicroPython tree (`fw-platform/components/micropython_service/micropy`) and the TI driver headers
  inside it carry their own notices in the file headers and licence files.
- No layer is tagged yet; `LAYER_TAGS.json` points at `HEAD`.
- Never commit real credentials; the templates ship `*.local.h.example` files and `*.local.*` is
  git-ignored.
