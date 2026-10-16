# Local patches in this fork (`xgx1/dsh-ops`)

Upstream `T-Auto/dsh-ops` 0.3.0 is Windows x64 only. This fork adds Linux x64 so all four
components run on the author's Arch Linux DSH deployment. Keep this list current so
`git merge upstream/main` can be re-applied knowingly.

## Platform gates

| File | Change | Why |
| --- | --- | --- |
| `package.json`, `package-lock.json` | `os` gains `linux` | Otherwise the package manager refuses to install (npm `notsup Actual cpu: x64`). |
| `lib/ptc/runtime.js` | `PLATFORMS` gains `linux-x64`; refusal text updated | Hard gate in `resolveWorker`; also drives `bin/native-runtime.mjs stage`, which validates the ELF header and writes `runtime/linux-x64/provenance.json`. |
| `dsh-plugin.json` | `dsh-ops-ptc` `platforms` gains `linux` | Marketplace/descriptor honesty only; DSH itself does not read this field. |

## Linux runtime bug

| File | Change | Why |
| --- | --- | --- |
| `native/src/paths.rs` | new `sync_dir()` | cap-std opens directories with `O_PATH` on Linux; `fsync` on such a descriptor is `EBADF`, which broke every `ops_batch` and `ops_replace` with `IO_ERROR Bad file descriptor (os error 9)`. |
| `native/src/store.rs`, `native/src/file.rs`, `native/src/lib.rs` | call `sync_dir()` with the host path instead of `dir.try_clone()?.into_std_file().sync_all()` | Root-cause fix for the above, at both durability sites. |
| `lib/workspace-preparation.js` | the Windows interpreter path uses `path.win32.join` | The spec always names `powershell.exe`; a POSIX `path.join` produced `C:\Windows/System32/...` and made the spawn-spec assertion host-dependent. Same convention as the Windows resolver in `shells.js`. |

## Test portability (so `node test/run.mjs` is green on Linux)

| File | Change |
| --- | --- |
| `test/ptc.test.mjs` | The "unsupported platform" fixture now uses `darwin`/`arm64` and the new refusal text; the old one asserted that `linux-x64` is rejected. |
| `test/binary.test.mjs` | Seeds the managed copy under the fixture's platform (`FORGED.platform`), so the search-order check no longer mixes a Linux seed with a `fastctx.exe` lookup. |
| `test/permissions.test.mjs` | The three structured `permission.code === 'POLICY_DENIED'` assertions tolerate an absent `permission` object: Windows' sandbox reports a structured denial, a POSIX sandbox denies in the kernel (read-only filesystem). The effect assertions (non-zero exit, no file written) stay unconditional. |
| `test/mount.test.mjs` | The background scenario names `bashPath: /bin/bash` on non-Windows hosts. `allowSystemShellFallback` is retired by config update 1, so it must not be used as a fixture knob. |
| `test/shells.test.mjs` | The Windows-shaped `DSH_HOME` store-path assertion is Windows-only; `dshHome()` resolves a non-absolute value against the host cwd, so only a Windows host can assert the composed path. |

The five `provision-shells` / `ladder` checks self-skip: the pin table covers `win32-x64`,
and this fork does not add a Linux shell payload (see below).

## Building on Linux

```console
npm run build:ptc                     # native Rust worker -> runtime/linux-x64
node bin/dsh-ops.mjs build --force    # vendored FastCtx -> vendor/fastctx/target/release/fastctx
```

## Running on Linux

- `shell`: point `bashPath` at the host's bash (`/usr/bin/bash`). There is no Linux shell
  payload to provision, and `allowSystemShellFallback` is not an operator switch — config
  update 1 forces it back to `false`.
- `background`: each row resolves its **own** shell, so the background row needs the same
  `bashPath`; without it its bash rung is unresolved and it publishes no command tools.
  Command tools additionally require an authoritative `danger-full-access` session.
- `file` and `ptc` need no shell configuration.
