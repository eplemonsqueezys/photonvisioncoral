# PhotonVision + Coral USB Accelerator (Orange Pi 5)

Two patches:

| Patch | Applies to | What it does |
|---|---|---|
| `tflite_jni-coral.patch` | [PhotonVision/tflite_jni](https://github.com/PhotonVision/tflite_jni) @ `52e8133` | Adds a `CORAL` source that loads `libedgetpu.so.1` as a TFLite external delegate; accepts int8 models; adds a decoder for the standard Ultralytics single-output YOLO format |
| `photonvision-coral.patch` | [PhotonVision/photonvision](https://github.com/PhotonVision/photonvision) @ `4b8d9b5` | Detects a Coral on USB, adds a `CORAL` model family (`*_edgetpu.tflite`), loads tflite_jni when a Coral is present, lets the upload dialog accept Coral models alongside `.rknn` |

Files touched in PhotonVision: `NeuralNetworkModelManager.java`, `TFLiteModel.java`, new `common/hardware/EdgeTpu.java`, `RequestHandler.java`, `Main.java`, `ObjectDetectionCard.vue`, `SettingTypes.ts`, `build.gradle`.

## 1. Install the Edge TPU runtime on the Orange Pi

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/coral.gpg
echo "deb [signed-by=/etc/apt/keyrings/coral.gpg] https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
sudo apt-get update && sudo apt-get install -y libedgetpu1-std
```

If that package won't install on your Ubuntu version, use a community build from
https://github.com/feranick/libedgetpu/releases (arm64 `.deb`). Plug the Coral into a **USB 3** (blue) port, then replug it after installing.

## 2. Build and install the patched tflite_jni (on the Orange Pi)

```bash
sudo apt-get install -y build-essential cmake openjdk-21-jdk git
git clone https://github.com/PhotonVision/tflite_jni && cd tflite_jni
git checkout 52e8133
git apply /path/to/tflite_jni-coral.patch
git commit -am "Add Coral Edge TPU support" && git tag v2027.0.2-coral   # sets the published version

cmake -B cmake_build -S . -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=cmake_build -DOPENCV_ARCH=linuxarm64
cmake --build cmake_build --target install -- -j 8        # builds LiteRT; takes a while
./gradlew build publishToMavenLocal -x spotlessCheck -x test
```

## 3. Build the patched PhotonVision

```bash
git clone https://github.com/PhotonVision/photonvision && cd photonvision
git checkout 4b8d9b5
git apply /path/to/photonvision-coral.patch
./gradlew spotlessApply shadowJar      # build.gradle now defaults to tflite_jni v2027.0.2-coral from mavenLocal
```

Building on the Orange Pi is simplest. To build on your PC instead, copy `~/.m2/repository/org/photonvision/tflite_jni-*` from the Pi into the PC's `~/.m2`, then run `./gradlew shadowJar -PArchOverride=linuxarm64`.

Upload the jar under `photon-server/build/libs/` through **Settings → Offline Update**. Or copy it over the installed jar and run `sudo systemctl restart photonvision`.

## 4. Make a Coral model (on an x86-64 PC or Google Colab)

`edgetpu_compiler` only runs on x86-64 Linux, so do this on a PC or in Colab:

```bash
pip install ultralytics
yolo export model=best.pt format=edgetpu imgsz=320
# -> best_saved_model/best_full_integer_quant_edgetpu.tflite
```

Keep the `_edgetpu.tflite` ending. In PhotonVision, go to **Settings → Object Detection → Import**, choose the file, enter labels (comma separated), set width/height to **320**, and pick YOLOv8 or YOLO11.

## Notes

- The Coral is detected once at startup, so plug it in before PhotonVision starts.
- The log should show `Coral USB Accelerator detected, enabling CORAL backend`. If loading fails with "Failed to create Edge TPU delegate", libedgetpu isn't installed or can't see the device; check `lsusb` for `18d1:9302` or `1a6e:089a`.
- If the delegate segfaults, the libedgetpu build is probably incompatible with tflite_jni's LiteRT 2.1.5. Try a newer feranick build.
- RKNN `.rknn` models still work side by side. The Orange Pi 5's own NPU stays the default backend.
