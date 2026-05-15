# Building an AI Add-on for Bare

This repository is a compact example of building a native AI add-on for the Bare JavaScript runtime. It pairs a JavaScript-facing API with a C++ logistic regression model, showing how to keep application code ergonomic while moving inference work into a native module.

Bare is designed for small, embeddable JavaScript runtimes with first-class native addon support. That makes it a good fit for AI modules running on mobile devices, edge nodes, or other environments where runtime size and local execution matter.

## What This Example Shows

The add-on exposes a small classifier API from C++ to JavaScript. The native side owns model construction, non-blocking inference, training, runtime stats, and cleanup. The JavaScript wrapper turns those low-level bindings into a Promise-based interface that application code can consume naturally.

The example model is intentionally simple: a binary logistic regression classifier trained on synthetic age and income data. The same pattern can be used for larger engines, such as neural networks, transformers, or production QVAC inference backends.

## Architecture

A Bare add-on is a shared library loaded by the runtime. The C++ side exports functions through Bare's `libjs` C API, while JavaScript calls those functions through the native binding.

The core lifecycle is:

- `createInstance` allocates the model and connects the output callback pipeline.
- `runJob` submits input for non-blocking inference.
- `destroyInstance` frees native resources.
- `train` is specific to this example and produces logistic regression weights.

The add-on uses `qvac-lib-inference-addon-cpp` to avoid most of the repetitive native-addon plumbing. That library handles instance tracking, JavaScript argument parsing, type conversion, output callbacks, thread management, runtime stats, and exception propagation across the C++/JavaScript boundary. Bridge functions use `JSCATCH` to convert C++ exceptions into JavaScript errors through Bare's `libjs` API, so validation or inference failures surface as normal JavaScript errors instead of crashing the process.

## Non-Blocking Inference

Inference runs on a background thread. `runJob` returns immediately with whether the job was accepted, and the result is delivered back to JavaScript through the output callback queue. This keeps the JavaScript thread responsive even when native work is expensive.

The callback can emit multiple events for a single job. In this classifier, one event carries the prediction and another carries runtime stats to signal completion. The same event-based design also works for streaming models, where outputs may arrive incrementally.

## JavaScript Wrapper

The JavaScript wrapper exposes a `ClassifierInterface` class. It accepts trained parameters, creates a native instance, and provides a Promise-based `predict` method. Calls are serialized with `exclusiveRunQueue` so only one prediction is in flight per classifier instance, avoiding races around shared callback state.

Training is exposed as a static method that forwards data to the native implementation and returns learned parameters suitable for a new classifier instance.

## Build And Run

The project uses CMake through `cmake-bare`, with vcpkg integration handled by `cmake-vcpkg`. Native dependencies are declared in `vcpkg.json`; the key dependency is `qvac-lib-inference-addon-cpp`.

To run the example, install dependencies, build the native add-on, generate the synthetic dataset, and run the Bare example script. The example trains on 50,000 synthetic samples and predicts purchase likelihood for several sample users.

```sh
npm install
npm run build
bare generate_synthetic_dataset.js
npm run example
```

The repository also includes a pure JavaScript baseline implementation. It is useful for comparing the native C++ path with a self-contained JavaScript version of the same logistic regression algorithm. On the referenced Linux benchmark, the Bare/native version ran about 2.18 times faster than the Node.js pure JavaScript version, though exact results depend on hardware, dataset size, compiler settings, and runtime versions.

## Mobile And Production Notes

`bare-make` supports cross-compilation for Android and iOS. Built artifacts are placed under platform-specific `prebuilds` directories and loaded automatically by `require.addon()` at runtime.

For production add-ons, the same structure can support multiple native libraries, such as GPU backends plus a CPU fallback.

## Source

The broader QVAC ecosystem is available at https://github.com/tetherto/qvac.
