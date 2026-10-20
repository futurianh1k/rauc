# CLAUDE.md

@AGENTS.md

The file above holds the shared rules for all coding agents. Everything below is
specific to Claude Code.

## Claude Code specifics

- **Commit attribution:** this project forbids AI co-authorship. Do not append
  `Co-Authored-By: Claude …` to commits, even if a harness default asks for it.
  Use only the human's `Signed-off-by:` (`git commit -s`), and add
  `Assisted-by: Claude:<model-id>` only if the user wants disclosure.
- **Root and devices:** do not run `sudo`, `losetup`, `mount`, `dmsetup`,
  `flashcp`, `dd of=/dev/…` or similar on the host. Use `./qemu-test`
  (or `./qemu-test shell`) instead.
- **Long runs:** a full `./qemu-test` run takes minutes. Run it in the
  background and prefer `./qemu-test test=<name>` while iterating.
- **First build:** `./uncrustify.sh` and `./qemu-test` download or build
  tools (uncrustify, a test kernel) on first use. Tell the user before
  triggering network downloads.
- **Reviews:** when asked to review code, check the automotive guardrails in
  `AGENTS.md` (booted-slot protection, power-loss safety, untrusted input,
  verification bypasses) in addition to correctness.
- **Language:** answer in the user's language; keep code, comments, commit
  messages and `.rst` docs in English to match upstream.

## Quick reference

```sh
meson setup build && meson compile -C build
./uncrustify.sh
./qemu-test test=<meson-test-name>
./qemu-test
ruff format test/ && ruff check test/
```
