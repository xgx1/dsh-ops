# Local patches in this fork (`xgx1/dsh-ops`)

Upstream `T-Auto/dsh-ops` 0.3.0 is Windows x64 only. This fork adds Linux x64 so the
`file` and `ptc` components run on the author's Arch Linux DSH deployment. Keep this
list current so `git merge upstream/main` can be re-applied knowingly.

| File | Change | Why |
| --- | --- | --- |
| `package.json`, `package-lock.json` | `os` gains `linux` | Otherwise the package manager refuses to install (npm `notsup Actual cpu: x64`). |
| `lib/ptc/runtime.js` | `PLATFORMS` gains `linux-x64`; refusal text updated | Hard gate in `resolveWorker`; also drives `bin/native-runtime.mjs stage`, which validates the ELF header and writes `runtime/linux-x64/provenance.json`. |
| `dsh-plugin.json` | `dsh-ops-ptc` `platforms` gains `linux` | Marketplace/descriptor honesty only; DSH itself does not read this field. |
| `native/src/paths.rs` | new `sync_dir()` | cap-std opens directories with `O_PATH` on Linux; `fsync` on such a descriptor is `EBADF`, which broke every `ops_batch` and `ops_replace` with `IO_ERROR Bad file descriptor (os error 9)`. |
| `native/src/store.rs`, `native/src/file.rs`, `native/src/lib.rs` | call `sync_dir()` with the host path instead of `dir.try_clone()?.into_std_file().sync_all()` | Root-cause fix for the above, at both durability sites. |
| `test/ptc.test.mjs` | "unsupported platform" assertion now uses `darwin`/`arm64` and the new message | The old fixture asserted that `linux-x64` is rejected. |

Not ported, on purpose: `dsh-ops/shell` (bundles Git-for-Windows bash and PowerShell 7
payloads; its default `deniedHostTools` would remove the host `bash` with nothing to
replace it) and `dsh-ops/background` (Windows FastCtx payload).

Build and stage the Linux worker from the checkout root:

```console
npm run build:ptc
```

`node test/run.mjs` on Linux leaves four Windows-shaped checks failing —
`binary.test.mjs` "managed copy and the vendored build still outrank both packages",
`permissions.test.mjs` "owned Modify-only workspace…" and "preparation hands the spawned
interpreter the scrubbed environment", and `shells.test.mjs` "the pin table names one
upstream release per shell". They exercise Windows ACL/executable-name behavior and fail
on Linux upstream as well; the `ptc` suite passes.
