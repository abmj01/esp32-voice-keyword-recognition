# I2S Build Error — Read This First

## The problem

Build fails with:
```
fatal error: driver/i2s.h: No such file or directory
HINT: The legacy I2S driver is removed...
```

This example (`audio_provider.cc`) uses the **legacy** I2S driver API
(`i2s_driver_install`, `i2s_set_pin`, `i2s_read`). That API has been fully
removed in newer ESP-IDF versions.

This isn't a bug in this repo specifically — even Espressif's own
`esp-tflite-micro` component (the one pulled in via `idf_component.yml`,
currently v1.3.7) still uses the legacy driver on its `master` branch. It
hasn't been ported to the new `i2s_std.h` API yet, so the example only
builds against older/mid ESP-IDF releases.

## The fix: use ESP-IDF v5.4

Install and use **ESP-IDF v5.4** for this project (via ESP-IDF Tools /
`install.ps1` + `export.ps1`, or the "ESP-IDF: New Project" installer if
you're on VS Code). v5.4 still ships `driver/i2s.h` (deprecated but
present), so the example builds unmodified.

Steps once v5.4 is installed and exported in your shell:
```
idf.py set-target esp32s3
idf.py build
```

Avoid ESP-IDF v5.5+ / master — that's where the legacy driver is gone
for good, and this example (as-is) won't compile there.

## Migrated to v5.4 — new error: broken toolchain

After installing v5.4 and running `idf.py build`, the I2S error was gone
but the build failed earlier, during compiler detection:
```
xtensa-esp32s3-elf-gcc.exe: fatal error: cannot execute 'cc1': CreateProcess: No such file or directory
```

This is unrelated to I2S — it means the `xtensa-esp-elf` toolchain
install is incomplete. `cc1.exe` (the actual C compiler backend that
`gcc.exe` shells out to) was missing from the toolchain's `libexec/`
folder entirely, most likely stripped by antivirus/Windows Defender
during install (a known false-positive target), or left over from an
interrupted install.

**Fix:**
1. Check your antivirus quarantine for `cc1.exe` / `xtensa-esp-elf` and
   restore it if found; add an exclusion for `C:\Espressif\tools` to
   prevent it happening again.
2. Delete the broken toolchain folder and reinstall it:
   ```
   Remove-Item -Recurse -Force C:\Espressif\tools\xtensa-esp-elf\esp-14.2.0_20241119
   ```
   then re-run the ESP-IDF Tools installer (or `install.ps1` from
   `C:\esp\v5.4\esp-idf`).
3. Reopen the IDF PowerShell environment and retry `idf.py build`.

## Other important info

- **Mic wiring (INMP441 → ESP32-S3)**: BCLK→GPIO6, WS→GPIO7, SD→GPIO9,
  L/R→GND. Already set in `main/audio_provider.cc`'s
  `CONFIG_IDF_TARGET_ESP32S3` pin block.
- **Long-term**: if you ever need to move past v5.4, `audio_provider.cc`
  will need a real migration to `driver/i2s_std.h` — the legacy calls
  don't have a drop-in replacement, so that's a code change, not just a
  version bump.
