# How Audio Becomes a Prediction: A Memory Walkthrough

## Project context

This project connects two separate things: `micro_speech`, an ESP32
firmware example for real-time keyword spotting, and `simple_audio`,
TensorFlow's tutorial notebook for training a spoken-word classifier.

The mismatch: both turn audio into a spectrogram (a picture of frequency
over time), but using different recipes. `simple_audio` builds a plain,
linear-frequency spectrogram. `micro_speech` computes a mel-spectrogram
on-device — frequency scaled to match human hearing, plus extra noise
reduction and automatic gain control. Different numbers, different
shape, so a model trained on one doesn't work on the other.

To fix that, `simple_audio` was changed to build its training data using
the same on-device recipe `micro_speech` uses, and to export the trained
model fully quantized to int8, since that's the format the firmware
needs. On the `micro_speech` side, the code that loads the model — which
math operations it supports, how much scratch memory it sets aside, and
what shape of input it expects — also had to be updated, since the new
model is bigger and shaped differently than the original one.

This traces one full cycle — from sound hitting the microphone to a word
being printed — focusing on **which memory each piece of data lives in**,
and when something gets **read** vs **written**. Two memories matter here:
**flash** (where the models permanently live) and **RAM/SRAM** (where
everything actively being worked on lives, including `tensor_arena`).

## The big picture

```
Mic → raw audio (RAM) → frontend model → feature grid (RAM)
    → copied into the classifier's input → classifier model runs → word printed
```

Two *different* models are involved, each with their own tiny interpreter
and arena — don't mix them up:
1. A small **frontend model** that turns raw sound into a feature grid
   (mel-filterbank style numbers) — covered briefly below, not the focus.
2. The **classifier model** (`g_model`, your trained CNN) that turns that
   feature grid into a predicted word — this is the vital part.

## Step 1: Audio comes in (brief)

`audio_provider.cc` reads raw samples from the I2S microphone and stores
them in a ring buffer (`g_audio_capture_buffer`) — plain RAM, nothing
flash-related yet. This is just a rolling window of the last second or so
of raw sound, as plain numbers (loudness over time).

## Step 2: Audio becomes numbers a neural net can read (brief)

`feature_provider.cc` pulls chunks of that raw audio and hands them to
`micro_features_generator.cc`, which runs the small **frontend model**
(window → FFT → mel filterbank → noise reduction → PCAN → log
compression) to turn each 30ms slice into 40 numbers. Stack 49 of these
slices and you get a `49×40 = 1960`-value grid, stored in:

```cpp
int8_t feature_buffer[kFeatureElementCount];  // main_functions.cc — plain RAM array
```

This is genuinely just data at this point — a description of "what the
last second of sound sounded like," in a format designed to match what
the classifier was trained on. Nothing about actual word recognition has
happened yet.

## Step 3: Handing the feature grid to the classifier

```cpp
for (int i = 0; i < kFeatureElementCount; i++) {
  model_input_buffer[i] = feature_buffer[i];
}
```

This copies the 1960 feature values from `feature_buffer` into
`model_input_buffer`. Both are in RAM — but `model_input_buffer` isn't
just any array; it's a pointer that was set up back in `setup()` to point
**inside `tensor_arena`**:

```cpp
model_input_buffer = tflite::GetTensorData<int8_t>(model_input);
```

So this copy is really: RAM → RAM, but the destination happens to be the
exact spot in `tensor_arena` that the classifier will read as "its input"
the moment you tell it to run.

## Step 4: Invoke() — where flash and RAM actually work together

This is the part worth understanding 100%. Three things were wired
together back in `setup()`:

```cpp
model = tflite::GetModel(g_model);          // pointer into FLASH
static tflite::MicroInterpreter static_interpreter(
    model, micro_op_resolver, tensor_arena, kTensorArenaSize);  // tensor_arena is RAM
```

- `g_model` — your 233KB trained model (weights + graph structure).
  Declared `const`, so it lives permanently in **flash**, never copied
  into RAM.
- `tensor_arena` — a `uint8_t[100 * 1024]` array in **RAM**, pure scratch
  space, holds nothing permanent.
- `interpreter` — doesn't hold data itself, just knows how to combine the
  two above.

When `interpreter->Invoke()` runs, for **every single layer** in your
network (each `Conv2D`, each `BatchNormalization`, etc.), the same
pattern repeats:

| Action | Source | Memory |
|---|---|---|
| Read the input to this layer | previous layer's output | `tensor_arena` (RAM) |
| Read this layer's weights | `g_model` | **flash** |
| Do the math (multiply, add, etc.) | — | CPU registers |
| Write this layer's output | — | `tensor_arena` (RAM) |

So one `Invoke()` call is really a chain of ~9-16 of these steps (one per
op in your graph — see `docs/NEW_MODEL_UPGRADE.md` for the exact op
list), each time pulling fresh weights out of flash and reading/writing
activations in the same shared RAM scratchpad. **The weights never move**
— only the in-progress numbers bounce around inside `tensor_arena`,
getting overwritten layer after layer as old results stop being needed.

This is also exactly why a 233KB model only needs a 100KB arena: the
arena's size depends on how big the *biggest single layer's* input+output
gets, not on how many weights the model has — the weights stay in flash
the whole time and never compete for that RAM space.

## Step 5: Reading the answer back out

```cpp
TfLiteTensor* output = interpreter->output(0);
float output_scale = output->params.scale;
int output_zero_point = output->params.zero_point;
for (int i = 0; i < kCategoryCount; i++) {
  float current_result =
      (tflite::GetTensorData<int8_t>(output)[i] - output_zero_point) * output_scale;
  ...
}
```

After the last layer runs, its output (8 raw int8 numbers, one per word)
is sitting in `tensor_arena`, same as every other layer's output was.
`interpreter->output(0)` just points at that final spot. The loop above
reads those 8 raw bytes and converts each one back into a real number
using the model's scale/zero_point (covered in an earlier conversation —
this is the same dequantization formula). Whichever one is largest gets
printed as the detected word.

## One-paragraph summary

Audio comes in and gets turned into a small grid of numbers, entirely in
RAM. That grid gets copied into the classifier's input slot inside
`tensor_arena`. From there, `Invoke()` walks through the network one
layer at a time, at each step reading permanent weights from flash
(`g_model`) and combining them with the current in-progress numbers
sitting in RAM (`tensor_arena`), overwriting that same RAM space with
each layer's result. After the last layer, the final numbers sitting in
`tensor_arena` *are* the answer — just waiting to be decoded back into a
word and printed.
