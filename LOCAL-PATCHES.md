# Local patches in this fork (`xgx1/dsh-ops`)

**There are none as of upstream 0.5.1.** This fork used to carry a Linux x64 port of 0.3.0 for the
file and PTC components; upstream implemented all of it itself, so `main` is now upstream `main`
plus this file. Re-add an entry here only for something upstream still gets wrong.

## Where the old patches went (0.3.0 fork → 0.5.1 upstream)

| Old patch | Upstream equivalent |
| --- | --- |
| `package.json` / `package-lock.json` `os` gains `linux` | `"os": ["win32", "linux", "darwin"]` |
| `lib/ptc/runtime.js` `PLATFORMS` gains `linux-x64` | `src/execution/runtime.ts`: `['win32-x64', 'linux-x64', 'darwin-arm64', 'darwin-x64']` |
| `dsh-plugin.json` `dsh-ops-ptc` `platforms` gains `linux` | `platforms: ["win32", "linux", "darwin"]`, `targets` names the four arch pairs |
| `native/src/paths.rs` `sync_dir()`: reopen the directory by host path, because cap-std's `O_PATH` descriptor fails `fsync` with `EBADF` on Linux | `native/crates/ops-files/src/lib.rs` `sync_directory(&Dir)`: tries `sync_all()`, on failure reopens the same handle through `/proc/self/fd/<n>` — same bug, tighter fix (no host-path reopen, so no TOCTOU window) |
| `native/src/store.rs`, `native/src/file.rs` durability call sites | `native/crates/ops-results/src/store.rs`, `native/crates/ops-files/src/file*.rs` |
| `lib/workspace-preparation.js`: the PowerShell interpreter path uses `path.win32.join` | Not needed: `test/permissions.test.mjs` now asserts only the host-relative tail off Windows |
| Five test-portability edits (`test/{ptc,binary,permissions,mount,shells}.test.mjs`) | Suites are written against all three platforms; the Linux/macOS gate sets `DSH_OPS_REQUIRE_RUNTIME=1`, so a missing worker fails instead of skipping |

## Building and running on this machine (Linux x64)

```console
npm ci
npm run build            # src/ -> lib/
npm run build:ptc        # native worker -> runtime/linux-x64 (cargo, ~30 s warm)
npm run build:bash-ipc   # Windows x64 only by design; refuses to run here
npm run validate:manifest
npm test                 # node test/run.mjs
npm run test:ptc         # node test/run.mjs ptc
```

- `lib/` and `runtime/` are gitignored: they are build output, never committed.
- `shell` and `background` on Linux run the host's own bash, so `bashPath` must point at it
  (`/usr/bin/bash`); the bundled PowerShell 7 and the Bash IPC proxies stay Windows x64 only.
- Merging upstream again: `git merge --no-ff --no-commit upstream/main`, and resolve every conflict
  to upstream.
