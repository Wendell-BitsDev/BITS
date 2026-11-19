# BITS firmware kit

Parent repo that holds the three firmware layers as git submodules, side by side:

| Layer | Content |
|---|---|
| `fw-platform` | Base: platform services shared by every product. |
| `fw-logic` | Pure rules (no hardware or network code). |
| `fw-app` | Protocols, MQTT, profiles and product templates. |

Dependency direction: app -> logic -> platform. A lower layer never depends on a higher one.

## Clone

    git clone --recurse-submodules https://github.com/Wendell-BitsDev/BITS.git

If you cloned without `--recurse-submodules`:

    git submodule update --init --recursive

The three layer repos are PRIVATE. You need access to all three (fw-platform, fw-logic, fw-app) or the submodule clone fails.

## Build a firmware

Use the ESP-IDF 6.1 PowerShell profile (not Bash):

    . C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1

Use one build job and have at least 1.5 GB of free memory before a build:

    $env:IDF_PY_BUILD_JOBS = 1

List the available profiles, then create a product from one:

    python fw-app\tools\new_product.py --list
    python fw-app\tools\new_product.py <profile> <dest>

Then build inside `<dest>` with `idf.py build`. See the Device Firmware guide HOW_TO_BUILD_FIRMWARE.pdf if you have it.

Do not commit build output (`build/`, `build-work/`, `managed_components/`).

## Update

1. Change code inside a layer (for example `fw-app`), commit and push there first.
2. In this repo: `git add fw-app` (or the layer you changed), commit, push.
3. Everyone else runs: `git pull --recurse-submodules`

After a plain clone each submodule is on a detached HEAD. Before editing a layer run `git switch main` inside that layer.
