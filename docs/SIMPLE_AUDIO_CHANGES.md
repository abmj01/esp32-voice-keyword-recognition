# What changed from the original tutorial

This notebook started from TensorFlow's official
[Simple audio recognition](https://www.tensorflow.org/tutorials/audio/simple_audio)
tutorial. The dataset loading (mini Speech Commands, 8 words) and the
overall train → evaluate → export flow are unchanged. Everything below is
what was changed, and why — all driven by one goal: make the trained
model actually work with the `micro_speech` ESP32 firmware, not just
train well in Python.

## 1. Feature extraction: STFT → the firmware's real frontend

**Original**: a plain linear-frequency spectrogram via `tf.signal.stft`.
```python
def get_spectrogram(waveform):
  spectrogram = tf.signal.stft(waveform, frame_length=255, frame_step=128)
  return tf.abs(spectrogram)  # shape: (124, 129, 1)
```
**Now**: the same window → FFT → mel filterbank → noise reduction →
PCAN → log-compression pipeline the ESP32 actually runs on-device,
via TensorFlow's `audio_microfrontend` op, configured to match
`micro_speech/main/micro_model_settings.h` exactly (30ms window, 20ms
stride, 40 channels):
```python
features = frontend_op.audio_microfrontend(
    pcm16, sample_rate=16000, window_size=30, window_step=20,
    num_channels=40, enable_pcan=True, out_type=tf.float32,
)  # shape: (49, 40, 1)
```
**Why**: the original spectrogram has nothing to do with what the
firmware computes from the microphone. Training on one representation
and running inference on a completely different one would make the
model's learned weights meaningless on-device — this makes training
and inference see the same kind of data.

## 2. Model architecture: no resizing, deeper, batch-normalized

**Original**: downsampled the large 124×129 spectrogram to 16×16 first,
then 2 conv layers, 1 pooling step.
```python
layers.Resizing(16, 16),
norm_layer,
layers.Conv2D(16, 3, activation='relu'),
layers.Conv2D(32, 3, activation='relu'),
layers.MaxPooling2D(),
```
**Now**: the frontend's `49×40` grid is already small, so no resizing is
needed. Three conv blocks instead of two, each followed by
`BatchNormalization` (stabilizes training) and its own pooling step:
```python
norm_layer,
layers.Conv2D(16, 3, activation='relu'), layers.BatchNormalization(), layers.MaxPooling2D(),
layers.Conv2D(32, 3, activation='relu'), layers.BatchNormalization(), layers.MaxPooling2D(),
layers.Conv2D(64, 3, activation='relu'), layers.BatchNormalization(),
```
**Why**: removing `Resizing` follows directly from feature #1 — there's
nothing left to downsample. The extra depth + batch norm was a
deliberate accuracy improvement or the smaller "no-resize" model alone.

## 3. Quantization pipeline: added (not in the original at all)

The original tutorial stops at a float Keras model + a TF `SavedModel`
export for server/browser use. This notebook adds a full **int8
quantization** pass on top — `representative_data_gen` (using the same
frontend features as training, for accurate calibration) and a
`TFLiteConverter` step producing a `.tflite` file with `int8` input and
output tensors specifically, since the firmware requires `kTfLiteInt8`.

**Why**: the ESP32 has no Python runtime and very little RAM/flash — it
needs a quantized `.tflite` model, not a Keras/SavedModel file.

## 4. Evaluation: added a quantized-model check

The original only evaluates the float Keras model. This notebook adds a
second evaluation pass — `evaluate_quantized_model()` — that runs the
actual exported `.tflite` file through the same held-out test set, so
you can confirm quantization didn't quietly break accuracy before
flashing it to hardware.

## 5. Removed: cells specific to a different deployment target

The original tutorial's `ExportModel`/`tf.saved_model.save` section (an
end-to-end waveform-in, JS/Python-serving-style export), waveform/
spectrogram exploration plots, and a single-file playback demo were all
dropped — they relate to browser/server deployment, not an embedded
firmware, and aren't needed here.
