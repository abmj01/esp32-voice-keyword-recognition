# ESP32-S3 Keyword Recognition

A small ESP32-S3 board listens to a microphone and recognizes eight
spoken words: **down, go, left, no, right, stop, up, yes** entirely
on-device. No Wi-Fi, no cloud API, no internet connection: a trained
neural network runs directly on the microcontroller and prints the
detected word over serial in real time.

```
Detected     yes, score: 7.28
Detected      no, score: 6.85
Detected    stop, score: 8.35
```

This project connects two things: [Espressif's `micro_speech`](https://github.com/espressif/esp-tflite-micro/tree/master/examples/micro_speech)
example (the ESP32 firmware that runs a keyword-spotting model in
real time) and TensorFlow's [Simple audio recognition](https://www.tensorflow.org/tutorials/audio/simple_audio)
tutorial (how to train one). Neither works with the other as-is. See
[How they were linked](#how-they-were-linked) below for why, and what
had to change on both sides to make it work.

## Hardware

- **Microcontroller**: ESP32-S3 — this project used a [Freenove ESP32-S3-WROOM](https://github.com/Freenove/Freenove_ESP32_S3_WROOM_Board) board (N8R8: 8MB flash, 8MB PSRAM).
- **Microphone**: INMP441, a digital MEMS microphone using the I2S protocol.

| INMP441 pin | Function | ESP32-S3 pin |
|---|---|---|
| GND | Ground | Any ground pin |
| VCC | Power (3.3V) | 3V3 |
| SD (DOUT) | Data out | GPIO9 |
| WS | Word select (L/R) | GPIO7 |
| SCK (BCLK) | Bit clock | GPIO6 |
| L/R | Channel select | GND (selects left channel) |

## Software

- **ESP-IDF v6.0.2:** see [docs/I2S_DRIVER_NOTE.md](docs/I2S_DRIVER_NOTE.md) for why the version matters here: newer ESP-IDF removed the legacy I2S driver this example was originally written against.
- **[esp-tflite-micro](https://github.com/espressif/esp-tflite-micro):** Espressif's ESP-IDF port of Google's TensorFlow Lite Micro, pulled in as a component dependency.
- **TensorFlow / Keras** (Python) — for training and quantizing the model, in [`simple_audio.ipynb`](simple_audio.ipynb).

## How they were linked

`micro_speech` computes a **mel-spectrogram** on-device (frequency
scaled to match human hearing, plus noise reduction and automatic gain
control) before ever handing anything to its model. TensorFlow's
`simple_audio` tutorial trains on a **plain linear-frequency
spectrogram** instead. Different recipe, different numbers,
different shape. A model trained on one doesn't work on the other.

To close that gap:
- `simple_audio.ipynb` was changed to generate training features using
  TensorFlow's `audio_microfrontend` op, configured to match the exact
  window size, stride, and channel count the firmware uses. so that the
  training sees the same kind of data the device will. Full breakdown
  in [docs/SIMPLE_AUDIO_CHANGES.md](docs/SIMPLE_AUDIO_CHANGES.md).
- The notebook's CNN architecture was made deeper (three `Conv2D` +
  `BatchNormalization` blocks instead of two) and the trained model was
  quantized to a fully `int8` `.tflite` file, the firmware requires
  `int8` tensors specifically.
- On the firmware side, `micro_speech/main/main_functions.cc` and
  `micro_model_settings.h` had to be updated to match the new model:
  which math operations it uses, how much scratch RAM it needs, the
  output category labels, and its input tensor shape. Full breakdown,
  including two real crashes hit along the way and how they were
  diagnosed, in [docs/NEW_MODEL_UPGRADE.md](docs/NEW_MODEL_UPGRADE.md).

For a step-by-step trace of what happens in memory from the moment
audio comes in to the moment a word is printed — including exactly
where the model lives in flash vs. RAM — see
[docs/HOW_INFERENCE_WORKS.md](docs/HOW_INFERENCE_WORKS.md).

## Repository layout

```
micro_speech/       ESP-IDF firmware project — build and flash this
simple_audio.ipynb  Training notebook (adapted from TensorFlow's tutorial)
docs/                Technical write-ups of what changed and why
```

## Building and running

1. Install [ESP-IDF v6.0.2](https://docs.espressif.com/projects/esp-idf/en/v6.0.2/esp32s3/get-started/index.html) for the ESP32-S3.
2. Wire the INMP441 to the ESP32-S3 per the table above.
3. From an ESP-IDF terminal:
   ```
   cd micro_speech
   idf.py set-target esp32s3
   idf.py build
   idf.py -p <YOUR_PORT> flash monitor
   ```
4. Say one of the eight keywords near the microphone and watch the
   serial output.

To retrain or modify the model instead, open `simple_audio.ipynb` —
it downloads the mini Speech Commands dataset, trains the classifier,
and exports a quantized `.tflite` file ready to convert into
`micro_speech/main/model.cc`.

## Result

The quantized model reaches **84.50% accuracy** (703/832) on a
test set measured on the actual exported `.tflite` file, not just the
float training model, so quantization's real impact is accounted for.

## Credits

- [`micro_speech`](https://github.com/espressif/esp-tflite-micro/tree/master/examples/micro_speech) and [`esp-tflite-micro`](https://github.com/espressif/esp-tflite-micro) - Espressif Systems, built on Google's [TensorFlow Lite Micro](https://github.com/tensorflow/tflite-micro).
- [Simple audio recognition](https://www.tensorflow.org/tutorials/audio/simple_audio) tutorial and the mini [Speech Commands dataset](https://arxiv.org/abs/1804.03209) (Warden, 2018) - TensorFlow / Google.
