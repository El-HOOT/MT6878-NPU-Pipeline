# MT6878 NPU Inference with LiteRT on Android 14

A verified, from-source method for running LiteRT NPU inference on the MediaTek Dimensity 7300 (MT6878), with no Play Store, no Google account, and no Android 15 requirement.

The MT6878 ships in millions of consumer devices. As far as I could find, this is the first documented and verified path to running LiteRT NPU inference on this chip on Android 14 without Play Store, a Google account, or upgrading to Android 15.

**Verified on:** CMF Phone 1 (MT6878) · Android 14 (SDK 34) · rooted · Ubuntu 24.04 host

```
NPU:  Non-zero: 978/1001  |  Top-1: 972  |  score: 0.0467224
CPU:  Non-zero: 978/1001  |  Top-1: 972  |  score: 0.0467224

NPU == CPU to 7 significant figures.
```

*(Low score is expected — the test harness uses a synthetic single-pixel input, not a real photo. What's being verified here is exact NPU/CPU agreement, not classification accuracy.)*

## Why this exists

Google's official LiteRT documentation assumes Play Store deployment and doesn't cover rooted, adb-only paths. The prebuilt MediaTek dispatch library shipped in the `v2.1.0rc1` runtime zip is compiled against Android 15 internals and fails silently on Android 14 with a misleading error. As far as I could find, no existing public writeup covers this device/OS/deployment combination — this guide documents the full working path, including every blocker hit along the way.

## What's in here

[`MT6878_NPU_LiteRT_Guide.md`](./MT6878_NPU_LiteRT_Guide.md) — the full guide, covering:

- Verifying NPU/NeuroPilot support on your device
- Building the LiteRT Docker dev environment
- Building `libLiteRtDispatch_MediaTek.so` from source (the critical fix for Android 14)
- AOT-compiling a model for MT6878
- Building and running the C++ inference binary on-device
- A troubleshooting table of every error encountered, with root cause and fix
- "Key Discoveries" — non-obvious findings from the debugging process
- Suggestions for next steps (custom models, fixing mixed-precision models like EmbeddingGemma)

## Should this work on your device?

Likely yes, with caveats, if you have:

- Any MT6878 (Dimensity 7300) device — root required
- Android 14 (SDK 34)
- A NeuroPilot runtime present on-device (check path in the guide — it was at `/system_ext/lib64/` on the CMF Phone 1, but may be under `/vendor/lib64/` on other devices)

Some paths and version numbers in the guide (e.g. Python version in venv paths) reflect the exact host setup used to verify this. They're noted inline where they might need adjusting for your environment.

## Disclaimer

This is a personal, independently verified record of a real debugging effort — not official MediaTek or Google documentation. Commands were tested on one specific device/host combination. Your mileage may vary on other MT6878 devices, and especially on other SoCs. Rooting your device voids warranty and carries its own risks; that part is on you.

## License

[MIT](./LICENSE) — use it, fork it, fix what's wrong with it.

## Contributing

If you get this working (or fail to) on a different MT6878 device, opening an issue or PR with your results would be genuinely useful — this is exactly the kind of fragmented, undocumented territory that benefits from more data points.
