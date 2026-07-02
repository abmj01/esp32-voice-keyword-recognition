# I2S Build Error — Read This First

**Update:** the project ended up migrating to ESP-IDF v6.0.2 with the new
`i2s_std` driver instead of staying on v5.4 — see "Final fix" at the
bottom. The sections below are kept as a record of the troubleshooting
path that led there.

## The problem

Build fails with:
```
fatal error: driver/i2s.h: No such file or directory
HINT: The legacy I2S driver is removed...
```

This example (`audio_provider.cc`) uses the **legacy** I2S driver API
(`i2s_driver_install`, `i2s_set_pin`, `i2s_read`). That API has been fully
removed in newer ESP-IDF versions.

This isn't a bug in this repo specifically, even Espressif's own
`esp-tflite-micro` component (the one pulled in via `idf_component.yml`,
currently v1.3.7) still uses the legacy driver on its `master` branch. It
hasn't been ported to the new `i2s_std.h` API yet, so the example only
builds against older/mid ESP-IDF releases.

## Previous fix: use ESP-IDF v5.4

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

## Migration to v5.4 — new error: broken toolchain

After installing v5.4 and running `idf.py build`, the I2S error was gone
but the build failed earlier, during compiler detection:
```
xtensa-esp32s3-elf-gcc.exe: fatal error: cannot execute 'cc1': CreateProcess: No such file or directory
```

This is unrelated to I2S — it means the `xtensa-esp-elf` toolchain
install is incomplete. `cc1.exe` (the actual C compiler backend that
`gcc.exe` shells out to) was missing from the toolchain's `libexec/`
folder entirely, most likely stripped by antivirus/Windows Defender
during install, or left over from an
interrupted install.

v5.4 was not used. 

## Final fix: migrated to ESP-IDF v6.0.2 + `driver/i2s_std.h`

Instead of staying pinned to v5.4, `audio_provider.cc` was rewritten
against the current (non-legacy) I2S driver, so the project now builds
on ESP-IDF v6.0.2. Summary of what changed:

- `driver/i2s.h` to `driver/i2s_std.h`; `i2s_port_t` to plain `int`
  (`I2S_NUM_0`/`I2S_NUM_1` are just `#define` constants in the new
  driver, not an enum type anymore).
- `i2s_driver_install`/`i2s_set_pin`/`i2s_read` to `i2s_new_channel` +
  `i2s_channel_init_std_mode` + `i2s_channel_enable` + `i2s_channel_read`.
- Removed the unused `esp_spi_flash.h` include (doesn't exist in v6.0.2;
  wasn't actually used in this file anyway).
- `main/CMakeLists.txt`: `PRIV_REQUIRES` now lists `esp_driver_i2s`
  instead of `driver` (the `driver` meta-component no longer pulls in
  I2S in v6.0.2).
- Mic slot mask is forced explicitly to `I2S_STD_SLOT_LEFT` — the
  `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG` macro defaults to `BOTH`
  regardless of mono/stereo on this chip family, so it must be set
  manually to match the INMP441's L/R→GND wiring.
- I2S read timeout bumped from 100ms to 1000ms — 100ms was too tight
  under normal scheduling jitter and caused reads to time out after
  only one DMA descriptor (960 of the requested 3200 bytes). The read
  still returns as soon as enough data is ready, so this only raises
  the worst-case wait, not steady-state latency.

## Other important info

- **Mic wiring (INMP441 → ESP32-S3)**: BCLK→GPIO6, WS→GPIO7, SD→GPIO9,
  L/R→GND. Set in `main/audio_provider.cc`'s `CONFIG_IDF_TARGET_ESP32S3`
  pin block.
