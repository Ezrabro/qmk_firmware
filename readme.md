# Ezrabro's Keychron QMK fork

A personal fork of [QMK Firmware](https://github.com/qmk/qmk_firmware) that keeps Keychron's
wireless stack working on top of a current QMK base.

The keyboard this exists for is a **Keychron K5 Version 2** (`k5x-h3-de`), built from
`keyboards/keychron/k5_version_2/` — three layouts (ANSI / ISO / JIS), each with an RGB and a
white-backlight variant, six builds in total.

This file replaces the stock QMK README. The upstream documentation still applies and lives at
[docs.qmk.fm](https://docs.qmk.fm); what follows is only about *this* repository.

---

## Where this comes from

| Remote | Repository | Role |
|---|---|---|
| `upstream` | https://github.com/qmk/qmk_firmware | Stock QMK. Source of everything that isn't Keychron. |
| `keychron` | https://github.com/Keychron/qmk_firmware | Keychron's own fork. Source of the wireless stack and all `keyboards/keychron/` boards. |
| `origin` | https://github.com/Ezrabro/qmk_firmware | This fork. |

The fork was originally cut from Keychron's fork, which was itself a snapshot of QMK. That
snapshot aged: `origin/master` is at `3ef9860ee1` (22 Jul 2026). On 29 Oct 2026 the working branch
was brought forward to **`180f394b48`** (16 Oct 2026, `upstream/master`) by merging 115 real
upstream commits — each one keeping its own hash, so `git log` still attributes them correctly and
`git blame` still works.

The ancestry:

```
qmk/qmk_firmware  ──►  Keychron/qmk_firmware  ──►  Ezrabro/qmk_firmware
                                                      ├── master       (pure upstream snapshot)
                                                      ├── Keychron     (the working branch)
                                                      └── openRGBTest  (personal customisations)
```

As of the last fetch, branch `Keychron` is **0 commits behind `upstream/master`** and 35 commits
ahead of it (all Keychron work, including the four repairs listed below).

---

## Branches

| Branch | What it is | State |
|---|---|---|
| **`Keychron`** | The branch you build from. Upstream QMK + Keychron wireless + the repairs below. | `6337740e03`, 120 ahead of `origin/Keychron`, **not pushed** |
| `master` | Untouched mirror of the upstream snapshot the fork was cut at. Kept for reference and diffing. | `180f394b48` |
| `openRGBTest` | `Keychron` + personal DIP-switch and OpenRGB work. | `e7985ccd52`, untouched during this work |
| `backup/Keychron-2026-10-29` | Safety branch taken before the sync. | `c2190644c9` |
| `backup/openRGBTest-2026-10-29` | Safety branch taken before the sync. | `e7985ccd52` |

Only `Keychron` was worked on. Nothing was pushed and nothing was flashed.

---

## What was done (29 Oct 2026)

### 1. Upstream sync

`git merge upstream/master` into `Keychron` — **115 upstream commits**, merge commit
`14b41fffef`. Merged rather than rebased, so the fork's history stays intact and each Keychron
commit keeps its identity.

Deliberately not done: re-adding the ~96 upstream keyboards that Keychron's fork removes. Those
deletions are load-bearing here — keeping them means `git status` stays clean and the tree stays
the shape Keychron intended.

### 2. Repairs — things that had rotted, or that upstream broke

The sync exposed code in this fork that no longer compiled against current QMK. Four fixes, each
its own commit:

| Commit | Problem | Fix |
|---|---|---|
| `4cc6cf386d` | The **legacy CKBT51 bluetooth stack** (`keyboards/keychron/bluetooth/`) had been broken by upstream API churn: `EECONFIG_RGB_MATRIX` / `EECONFIG_LED_MATRIX` are no longer public, ChibiOS 2.x GPIO macros were removed, and `debounce_free()` no longer exists. | Switched to `eeconfig_read_rgb_matrix()` / `eeconfig_read_led_matrix()`, replaced the GPIO macros, dropped the dead `debounce_free()` call, and hoisted `#include "hal.h"` above the `HAL_USE_RTC` guard. |
| `83de125d90` | `k0_max` declared ChibiOS HAL config version `8.0`; current ChibiOS wants `8.4`. | Bumped the version define. (Keychron made the same change independently in `e614a01715`.) |
| `ae39bfc6dc` | `lkbt51_send_system()` filtered out any usage outside `0x81–0x83`, which silently dropped the **release** report (`0x00`). A media/system key could stay held on the host forever. | The release is now forwarded, and `0x9B` (`SYSTEM_RESTART`) is also allowed through. |
| `6337740e03` | `encoder_cb_init()` installed PAL line callbacks unconditionally whenever `ENCODER_ENABLE` was on. Callbacks only exist when `PAL_USE_CALLBACKS == TRUE`. | Guarded all three call sites with `#if defined(ENCODER_ENABLE) && (PAL_USE_CALLBACKS == TRUE)`. All 30 encoder boards here already define `PAL_USE_CALLBACKS`, so no behaviour changes — the guard only protects builds that don't. |

One more repair, from the previous round and already on the branch before the sync:
`rgb_matrix_update_pwm_buffers()` had lost its prototype in `quantum/rgb_matrix/rgb_matrix.h`
when the fork was made.

### 3. Audit for things quietly dropped in the past

Swept `quantum/`, `tmk_core/`, `platforms/` and `drivers/` for declarations that exist in a `.c`
file but in no header — the classic way a fork loses a symbol and only finds out at link time on
some obscure board. Exactly one was missing, the RGB matrix prototype above. Also checked: no
stray conflict markers anywhere in the source, submodules unchanged (`a1e0a9a3ea` /
`02e126c578`), and the 96 non-Keychron board directories stayed deleted.

### 4. Build verification

All six `k5_version_2` variants plus five neighbouring boards that exercise the paths this work
touched — **11 builds, 11 PASS, 0 FAIL**:

```
keychron/k5_version_2/ansi/rgb:via      keychron/k5_version_2/iso/rgb:via      keychron/k5_version_2/jis/rgb:via
keychron/k5_version_2/ansi/white:via    keychron/k5_version_2/iso/white:via    keychron/k5_version_2/jis/white:via
keychron/k0_max:via                     keychron/k11_max/ansi_encoder/rgb:via
keychron/k3_max/ansi/rgb:via            keychron/q1_pro/ansi_knob:via
keychron/k8_pro/ansi/rgb:default        (legacy CKBT51 stack still links)
```

On top of that, a per-family sweep of the rest of `keyboards/keychron/` came back 27 PASS /
0 genuine failures.

---

## Hardware notes — why some things were left alone

- **MCU:** WB32F3G71, `wb32-dfu` bootloader (`k5_version_2/info.json`).
- **Radio:** an *external* **LKBT51** module over SPI (`LKBT51_*` pins in `config.h`,
  `LK_WIRELESS_ENABLE` via `common/wireless/wireless.mk`). That is why the modern
  `keyboards/keychron/common/wireless/` stack is the relevant one — not the CKBT51 stack in
  `keyboards/keychron/bluetooth/` that the older `*_pro` boards use.
- **Storage:** external I2C EEPROM, 2048 bytes (`EXTERNAL_EEPROM_BYTE_COUNT`). Not STM32 EFL
  wear levelling.
- **No encoder** on this board.

### Upstream / Keychron commits deliberately *not* ported

| Commit | Why not |
|---|---|
| `620468c58b` wireless raw HID | The `host_driver_t` / `wt_func_t` table in `common/wireless/` has no `send_raw_hid` slot and there is no `wireless_send_raw_hid()`. Porting this single commit means first porting the whole function-pointer plumbing — a structural change, not a fix. |
| `738e2c20f6` brightness cutoff | Upstream forces `RGB_MATRIX_BRIGHTNESS_TURN_OFF_VAL=48` from `keychron_common.mk` for every RGB board. **70 board configs in this tree set 32 explicitly** (q1_max, q10_max, q13_max, q14_max, q15_max, q1_pro, k10_pro, k8_pro, …). Forcing 48 would silently change all of them. `k5_version_2` already sets 48 itself, so nothing is lost by skipping. |
| `07bfc38a4b` `k0_max` source list | Already present: `k0_max/rules.mk` has `SRC += led_config.c ./common/matrix.c`. |
| `06056f1ef1` STM32L432 EFL ECCD fences | No Keychron board uses EFL wear levelling; `k5_version_2` uses the external EEPROM. Patching a driver nothing here compiles would be noise. |

Also not ported, by choice rather than by obstacle: adaptive NKRO, joystick/XInput over wireless,
Snap Click, `QK_CR_*` keycodes, the language module, HE/analog support, and USB report-interval
control. Several of those depend on Keychron Launcher; Snap Click is redundant with the QMK SOCD
module already in use.

---

## Building

Toolchain: **QMK MSYS** at `C:\QMK_MSYS`.

```sh
cd /e/qmk_firmware

# the six K5 Version 2 variants
qmk compile -kb keychron/k5_version_2/ansi/rgb   -km via
qmk compile -kb keychron/k5_version_2/ansi/white -km via
qmk compile -kb keychron/k5_version_2/iso/rgb    -km via
qmk compile -kb keychron/k5_version_2/iso/white  -km via
qmk compile -kb keychron/k5_version_2/jis/rgb    -km via
qmk compile -kb keychron/k5_version_2/jis/white  -km via
```

Flash with `qmk flash -kb keychron/k5_version_2/<layout>/<variant> -km via`, then put the board in
`wb32-dfu` mode (reset combo, or the bootloader key if your keymap has one).

### Building more than one keyboard at once

QMK writes a single global failure sentinel, `.build/error_occurred`, shared by every build in the
working directory. Two builds running in parallel will therefore report **false failures** — one
sees the other's sentinel. If you script a batch, give each target its own `BUILD_DIR`:

```sh
make keychron/k5_version_2/ansi/rgb:via BUILD_DIR=.build/k5_ansi_rgb
```

### Personal keymap

The real `azar` keymap lives outside the repo, in the QMK userspace overlay at
`C:\Users\Azar\qmk_userspace`, not in `keyboards/keychron/k5_version_2/*/keymaps/`.

---

## Not done / loose ends

- **Nothing has been pushed.** `Keychron` is 120 commits ahead of `origin/Keychron`.
- **Nothing has been flashed.** All results above are build-only.
- **`openRGBTest` was not touched** — still `e7985ccd52`, identical to its backup branch. Merging
  `Keychron` into it and re-applying the DIP-switch / OpenRGB work is the next job, and it is not
  started. That branch is 3 ahead and roughly 117 behind.
- **Scratch files** `_all.txt`, `_humans.txt` and `.tmp/` are untracked and not gitignored. They
  are build/sweep output and can be deleted at any time.

---

## Credit

All firmware code here is QMK's and Keychron's work, under their respective licences
(GPLv2 / GPLv3 / Modified BSD — see `license_*.md`). This fork adds the sync, the repairs
described above, and the personal customisations on `openRGBTest`.