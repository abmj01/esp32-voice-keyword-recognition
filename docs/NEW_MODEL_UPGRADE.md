# Upgrading to the new keyword-spotting model

## Background

The original `model.cc` was the stock TFLite Micro reference model: a tiny
4-class classifier (`silence`, `unknown`, `yes`, `no`). It's been replaced
with a model trained in `simple_audio.ipynb` on the mini Speech Commands
dataset (8 classes), using the same on-device feature pipeline
(`micro_features_generator.cc`) so training and inference see matching
input data. The trained Keras model — a deeper CNN (`Conv2D` ×3 +
`BatchNormalization` + `MaxPooling2D`, `Dense` ×2) than the original tiny
reference model — was quantized to a fully int8 `.tflite` file, converted
to a C array (`xxd -i -n g_model ...`), and pasted into `model.cc`.

The feature extraction pipeline (`audio_provider.cc`, `feature_provider.cc`,
`micro_features_generator.cc`) is unaffected by this change and needed no
edits — it still produces the same `49×40` int8 feature grid the new model
expects, since the frontend and window/stride settings didn't change.

Everything below is what actually changed to get the new model running —
including two real crashes hit along the way, since the fixes aren't
obvious from the error messages alone.

## Changes made

### 1. `model.cc`
`g_model[]` / `g_model_len` replaced with the new 233,544-byte model,
keeping the original variable names, `const` types, and
`DATA_ALIGN_ATTRIBUTE` alignment macro intact (required — `xxd -i` doesn't
add these, and the extern declarations in `model.h` require them to match
exactly).

### 2. `micro_model_settings.h` — category list

The new model outputs 8 classes, not 4, in the alphabetical order Keras
assigned during training (`label_names` in the notebook):

```cpp
constexpr int kCategoryCount = 8;
constexpr const char* kCategoryLabels[kCategoryCount] = {
    "down", "go", "left", "no", "right", "stop", "up", "yes",
};
```

### 3. `main_functions.cc` — tensor arena size

```cpp
constexpr int kTensorArenaSize = 100 * 1024;  // was 30 * 1024
```

30KB was sized for the old, much smaller model. Note this scales with the
model's *activation* sizes (intermediate layer outputs), not its file
size — a model can be 12x bigger on disk (more weights, which live in
flash, not the arena) while needing a similar or only moderately larger
arena. 100KB was a generous starting guess that turned out to be enough;
`interpreter->arena_used_bytes()` after a successful `AllocateTensors()`
call would give the exact number if you want to right-size it further.

### 4. `main_functions.cc` — op resolver (the first real blocker)

The resolver only registered the 4 ops the *old* model needed
(`DepthwiseConv2D`, `FullyConnected`, `Softmax`, `Reshape`). The new
model's Keras architecture compiles down to a different, larger op set —
confirmed by extracting the actual bytes out of `model.cc` and inspecting
the compiled graph directly rather than guessing from the Python code:

```
CONV_2D ×3, MUL ×3 + ADD ×3 (BatchNormalization, folded by the converter),
MAX_POOL_2D ×2, SHAPE/STRIDED_SLICE/PACK (Flatten's dynamic-shape plumbing),
RESHAPE, FULLY_CONNECTED ×2
```

Symptom: `Didn't find op for builtin opcode 'CONV_2D'` →
`AllocateTensors() failed` → `setup()` returns before `feature_provider`
is constructed → `loop()` dereferences it anyway → `LoadProhibited` crash
in `feature_provider.cc`. Fix: register the actual ops used, and bump the
resolver's capacity template argument to match:

```cpp
static tflite::MicroMutableOpResolver<9> micro_op_resolver;
micro_op_resolver.AddConv2D();
micro_op_resolver.AddMul();
micro_op_resolver.AddAdd();
micro_op_resolver.AddMaxPool2D();
micro_op_resolver.AddShape();
micro_op_resolver.AddStridedSlice();
micro_op_resolver.AddPack();
micro_op_resolver.AddReshape();
micro_op_resolver.AddFullyConnected();
```
(each call checked against `kTfLiteOk`, returning early on failure, as in
the original code.) `AddDepthwiseConv2D()` and `AddSoftmax()` were
dropped — this model doesn't use them (the last `Dense` layer has no
activation, and the firmware already does raw dequantize+argmax instead
of consuming a softmax output).

### 5. `main_functions.cc` — input tensor shape check (the second blocker)

The old model's Keras input was already flat, so its exported tensor was
2D: `[1, 1960]`. The new model declares `layers.Input(shape=(49, 40, 1))`
(needed for `Conv2D`), so its tensor is 4D: `[1, 49, 40, 1]` — same total
element count, different declared shape. The existing check assumed 2D
and rejected the new model with `Bad input tensor parameters in model`
(same crash chain as above). Fix:

```cpp
model_input = interpreter->input(0);
if ((model_input->dims->size != 4) || (model_input->dims->data[0] != 1) ||
    (model_input->dims->data[1] != kFeatureCount) ||
    (model_input->dims->data[2] != kFeatureSize) ||
    (model_input->dims->data[3] != 1) ||
    (model_input->type != kTfLiteInt8)) {
  MicroPrintf("Bad input tensor parameters in model");
  return;
}
```

The data-copying loop in `loop()` (`model_input_buffer[i] = feature_buffer[i]`)
needed no change — it just copies a flat 1960-byte buffer regardless of
how the tensor's shape metadata describes it.

## What did *not* need to change

- `kFeatureCount` / `kFeatureSize` (49 / 40) and the whole feature
  extraction pipeline — the frontend's output shape is unchanged.
- The `kTfLiteInt8` type check — the new model was quantized to int8
  specifically to satisfy it.
- The inference/argmax loop in `main_functions.cc::loop()` — it already
  iterates `kCategoryCount` dynamically.

## Current status

All five changes above are applied and confirmed working on-device
(build, flash, and monitor all succeed; the model runs and produces
predictions).
