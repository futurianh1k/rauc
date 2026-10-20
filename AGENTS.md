# AGENTS.md

Guidance for AI coding agents (and the engineers driving them) working on RAUC
in an automotive context: ECUs, infotainment/IVI head units, telematics units,
gateways and other embedded Linux targets that receive software updates over
the air (OTA) or in the workshop.

RAUC is a safe and secure update client for embedded Linux.
It installs signed *bundles* into redundant *slots* (A/B or A/B/recovery) and
uses the bootloader (*bootchooser*) to switch to, and fall back from, the
updated system.
In a vehicle a broken update can mean a bricked ECU, a tow truck or a recall,
so treat every change to the install, verification and boot-selection paths as
safety- and security-relevant.

## Project facts

| Item | Value |
| --- | --- |
| Language | C (GLib/GIO), Python for CLI tests |
| Build | Meson + Ninja |
| Key deps | GLib/GIO ≥ 2.64, OpenSSL ≥ 3.0 (CMS signatures), libcurl ≥ 7.81 (network), libnl-genl (streaming), D-Bus, json-glib; optional libfdisk (GPT), composefs |
| License | LGPL-2.1-or-later (see `COPYING`, `LICENSES/`) |
| Docs | Sphinx, reStructuredText in `docs/` |
| Upstream | https://github.com/rauc/rauc |

## Repository map

| Path | Contents |
| --- | --- |
| `src/main.c` | `rauc` CLI entry point and subcommands |
| `src/install.c` | Install orchestration: slot selection, handlers, hooks, bootchooser calls |
| `src/bundle.c`, `src/signature.c`, `src/crypt.c` | Bundle formats (plain, verity, crypt), CMS signing/verification, encryption |
| `src/manifest.c` | Manifest parsing and validation (untrusted input) |
| `src/config_file.c` | `system.conf` parsing |
| `src/slot.c`, `src/status_file.c`, `src/mark.c` | Slot model, slot status persistence, `rauc status mark-*` |
| `src/bootchooser.c`, `src/bootloaders/` | Boot selection: barebox, U-Boot, GRUB, EFI, custom backend |
| `src/update_handler.c`, `src/update_utils.c` | Per-image-type write logic (ext4, raw, UBI, NAND, eMMC boot partitions, …) |
| `src/emmc.c`, `src/gpt.c`, `src/mbr.c` | Atomic bootloader-partition updates |
| `src/nbd.c`, `src/network.c`, `src/polling.c` | HTTP(S) streaming, network access, polling (preview) |
| `src/service.c`, `src/*.xml` | D-Bus service `de.pengutronix.rauc.Installer` / `.Poller` |
| `src/artifacts*.c` | Artifact repositories (incl. composefs) |
| `src/event_log.c` | Structured event logging (journal) |
| `include/` | Public-to-the-project headers, one per module |
| `test/*.c` | GLib unit tests, one per module (`test/bundle.c` tests `src/bundle.c`) |
| `test/test_*.py` | pytest CLI tests |
| `fuzz/` | Fuzz targets |
| `contrib/` | Examples (CGI web UI, polling server, …) |
| `docs/` | User/integrator documentation; `docs/reference.rst` is the config/manifest reference |
| `CHANGES` | Release notes (rst) |

## Build and test

```sh
meson setup build                    # default options, tests enabled
meson compile -C build
./uncrustify.sh                      # mandatory C formatting (builds uncrustify on first run)
./qemu-test                          # full test suite inside a QEMU VM
./qemu-test test=install             # one glib test (meson test name)
./qemu-test test=pytest-install      # one pytest file
ruff format test/ && ruff check test/   # after touching Python tests
```

- **Never run tests with `sudo` on the host.** Many tests create loop
  devices, mount filesystems, use device-mapper or emulated NAND/eMMC.
  Use `./qemu-test`, which needs no host root.
- CI builds with `-Dwerror=true`. Keep the build warning-free.
- CI tests two configurations; check both if you touch `#if` feature code:
  - `-Dservice=true -Dgpt=enabled -Dcomposefs=enabled`
  - `-Dservice=false -Dgpt=disabled -Dcomposefs=disabled`
- Sanitizers: `meson setup -Db_sanitize=address,undefined build`, then
  `./qemu-test asan`.
- Fuzzers: `meson setup -Dfuzzing=true -Db_lundef=false -Db_sanitize=address build`,
  then `meson test -C build --suite rauc:fuzzing`.
- Cross builds (arm/v5, arm/v7, arm64, 386) run in CI. Do not assume 64-bit
  `long`, `size_t` or `off_t`, or little-endian layouts in on-disk formats.
- Interactive target simulation: `./qemu-test system` (RAUC service, D-Bus,
  dummy slots, GRUB mock bootloader; no reboot).

## Coding conventions

- Style is Linux-kernel-like with deviations. `./uncrustify.sh` is the source
  of truth. Tabs for indentation.
- Use GLib idioms consistently with surrounding code:
  - Errors via `GError **error` as the last parameter, set with `g_set_error()`
    using the module's quark (`R_INSTALL_ERROR`, `R_SLOT_ERROR`, …) and an
    enum code. Propagate with `g_propagate_prefixed_error()`.
  - Guard public functions with `g_return_val_if_fail(error == NULL || *error == NULL, …)`.
  - Prefer `g_autofree`, `g_autoptr()`, `g_auto(GStrv)` and
    `g_auto(filedesc) fd = …` (from `include/utils.h`) over manual cleanup
    and `goto out`.
  - Declare variables at first assignment (recent refactors move this way).
- Return `gboolean` for success/failure; on `FALSE`, `*error` must be set.
- New config keys go into `src/config_file.c`, `docs/reference.rst`, tests,
  and, if user-visible, `CHANGES`.
- Documentation (`.rst`) uses semantic linefeeds: one sentence per line.

## Commits and pull requests

- One conceptual change per commit.
- Subject prefix is the path or area: `src/install: …`, `test: …`,
  `docs: …`, `.github/workflows: …`. Imperative mood, lower case after the
  colon.
- Every commit needs `Signed-off-by:` with the human contributor's real name
  (`git commit -s`, DCO 1.1).
- **AI policy (from `docs/contributing.rst`):** AI tools must not be listed as
  author or co-author. Do **not** add `Co-Authored-By:` lines for an AI.
  Disclosure is optional via `Assisted-by: AGENT_NAME:MODEL_VERSION [TOOLS]`.
  The human contributor must understand and be able to defend every line.
- New features need docs and tests. Backward-incompatible changes must be
  described in the commit message and in `CHANGES`.

## Automotive engineering guardrails

These rules apply on top of upstream conventions whenever you change RAUC for,
or reason about it in, a vehicle program.

### Safety of the update path (fail-safe, never brick)

- The currently booted slot is sacred. Code must never write to the booted slot
  or to its active bootloader partition. Look for checks on `ST_BOOTED`
  and `slot->parent` before changing slot selection.
- Every operation that can be interrupted by power loss (ignition off, battery
  disconnect, crank undervoltage) must leave the ECU bootable.
  Examples: mark the target slot bad *before* writing, switch boot order only
  *after* a successful write and `fsync`, use the atomic eMMC
  boot-partition/GPT/MBR switching paths.
- Do not weaken fallback: boot attempts counters, `mark-good` /
  `mark-bad` semantics and bootchooser state transitions are the rollback
  mechanism. Changes there need tests for success, failure and interruption.
- Handle errors explicitly. No silent fallbacks, no ignored return values on
  write, sync, mount or bootloader calls.
- Watch resource use on constrained ECUs: avoid unbounded memory for large
  images, stream instead of buffering, and keep the D-Bus service responsive.

### Cybersecurity (ISO/SAE 21434, UNECE R155)

- Manifests, bundles, HTTP responses and D-Bus input are **untrusted**. Validate
  lengths, paths (`..`, `/` in names), digests and types before use. Recent
  fixes in `src/manifest.c` show the expected pattern.
- Never bypass or relax signature verification, keyring checks, certificate
  purpose or `check-crl` handling to make something "work". Do not add
  debug switches that skip verification in production builds.
- Prefer `verity` or `crypt` bundle formats; `plain` is legacy.
- Keys: never commit real private keys, PKCS#11 PINs or production
  certificates. Test keys live under `test/` and must stay obviously test-only.
- Do not log secrets, key material or full URLs with tokens.
- Treat new dependencies, network features and privilege changes (e.g.
  `streaming_user`) as attack-surface changes and call them out in the PR.

### Software update management (UNECE R156, ISO 24089)

- Keep traceability: the manifest's `compatible`, `version`, `build` and
  slot status (installed digest, timestamps, counts) are what an SUMS audit
  relies on. Do not drop or rename fields without migration.
- Keep `compatible` checks strict; installing a bundle for another ECU variant
  must fail.
- Preserve the event log (`src/event_log.c`) entries for install start,
  success, failure and boot-state changes.
- Version and downgrade policy belong in system integration (hooks, handlers,
  backend), not in ad-hoc patches to core logic.

### Functional safety (ISO 26262) note

RAUC itself is not developed or certified as an ASIL item. In a typical
architecture it runs on a QM partition (Linux) and must not interfere with
safety functions. If a program assigns safety requirements to the update path,
flag it to the human engineer rather than claiming compliance in code or docs.

## How agents should work here

1. Read the relevant module, its header in `include/`, and its test in
   `test/` before editing.
2. Make the smallest change that solves the problem; match the surrounding code.
3. Add or update tests (`test/<module>.c` or `test/test_<cmd>.py`).
4. Run `./uncrustify.sh`, build, and `./qemu-test` (or the targeted
   `test=` run). Report real results, including failures.
5. Update `docs/` and `CHANGES` for user-visible behavior.
6. In your summary, state the safety and security impact explicitly:
   which slots/boot paths are affected, behavior on power loss, and any change
   to verification or trust.

Do not without explicit human approval:

- change signature, keyring, encryption or certificate-handling logic;
- change on-disk formats (bundle, status file, verity/crypt metadata);
- change bootchooser state machines or bootloader backends;
- run anything as root on the host, or write to real block devices;
- push, open PRs, or change CI workflows.
