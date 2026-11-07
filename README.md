# Motorola SM6375 Linux 6.1 build guide

Build LineageOS 24.0 for Moto G34/G45 (`fogos`) or Moto G82 (`rhodep`) with the shared Linux 6.1 port. All port repositories are public in [Motorola-Sedona61](https://github.com/Motorola-Sedona61).

The kernel starts from Motorola's `MMI-W1UOS36H.35-39-7` release, commit `e84e985ceaffdd728f2c71e0b551ad3ce13a05ea`, based on Linux 6.1.145. The kernel, retained display and graphics branches preserve their Motorola ancestry. Audio uses the retained APR/Q6 stack with the stock device firmware.

## Host setup

Use Ubuntu 24.04 x86_64, Python 3, Git and the Android `repo` launcher. Install the kernel build dependencies:

```sh
sudo apt update
sudo apt install bc bison build-essential clang-18 device-tree-compiler \
    dwarves flex git libelf-dev libssl-dev lld-18 llvm-18 openssl \
    python3 python3-venv rsync kmod
```

Install `repo` using the [Android instructions](https://source.android.com/docs/setup/download#installing-repo). The ROM additionally needs the packages listed in the [LineageOS build documentation](https://wiki.lineageos.org/devices/). Java and host tools used by the ROM are supplied in the Android checkout. Keep adequate free disk space for the full Android source and output trees.

## Build the kernel and every required module

```sh
mkdir sedona-kernel
cd sedona-kernel
repo init -u https://github.com/Motorola-Sedona61/kernel_manifest.git -b main
repo sync -c -j8
./build/setup.sh
./build.sh FOGOS
```

For Rhodep:

```sh
./build.sh RHODEP
```

Each command configures and builds the kernel, board device trees, retained audio, graphics, display, camera, video, Wi-Fi, data and Motorola board modules. It then produces:

```text
prebuilt/FOGOS/6.1/        # or prebuilt/RHODEP/6.1/
  Image
  dtb/
  dtbo.img
  modules/
  vendor_modules/
  modules.load.recovery
  modules.load.vendor
  modules.blocklist
  kernel-headers/
  kernel-uapi-headers.tar.gz
```

The Fogos and Rhodep driver sources share [motorola-kernel-modules](https://github.com/Motorola-Sedona61/motorola-kernel-modules/tree/sedona-6.1). The retained Fogos drivers are under `legacy/fogos`. Board load lists, DTBO ordering and camera options remain separate.

Jobs default to the host CPU count, capped at 192. Choose a smaller count or another LLVM installation when needed:

```sh
./build.sh FOGOS --jobs 32
./build.sh FOGOS --llvm /path/to/llvm/bin
./build.sh RHODEP --plan
```

Run kernel builds one at a time in a workspace. A build lock prevents simultaneous module builds. To capture a complete build log:

```sh
./build.sh FOGOS > fogos-build.log 2>&1
```

## Sync the ROM

The ROM manifest includes the public device, vendor and prebuilt repositories for both devices, the modified compatibility repositories, and the Spring4ever BSP forks. Unchanged Lineage projects use upstream sources. The local server-specific Soong/Ninja/RBE edits and the five excluded init/SELinux forks are not part of this manifest.

```sh
mkdir lineage
cd lineage
repo init -u https://github.com/LineageOS/android.git -b lineage-24.0 --git-lfs
mkdir -p .repo/local_manifests
curl -fL https://raw.githubusercontent.com/Motorola-Sedona61/local_manifests/main/sedona61.xml \
    -o .repo/local_manifests/sedona61.xml
repo sync -c -j8
```

Use a clean local-manifest directory, or remove earlier port manifests that declare the same project paths. Published vendor trees already contain the extracted device files and compatibility fixups, so extraction is not required for an ordinary build.

## Use the published prebuilts

The ROM manifest syncs the prebuilt sets into:

- `device/motorola/fogos-kernel/6.1`
- `device/motorola/rhodep-kernel/6.1`

No kernel compile is required when using those sets. To use your own kernel build instead, copy the entire matching `6.1` directory:

```sh
# Run inside the sedona-kernel workspace, after the build:
rsync -a --delete prebuilt/FOGOS/6.1/ \
    /path/to/lineage/device/motorola/fogos-kernel/6.1/
```

Alternatively, build and install the set in one command:

```sh
./build.sh FOGOS --android-tree /path/to/lineage
```

Use the Rhodep paths and `RHODEP` argument for Rhodep. Keep `Image`, DTs, modules and exported headers from the same generated set.

## Build the ROM

From the Lineage checkout:

```sh
source build/envsetup.sh
lunch lineage_fogos-cp2a-userdebug
m bacon -j32
```

For Rhodep:

```sh
source build/envsetup.sh
lunch lineage_rhodep-cp2a-userdebug
m bacon -j32
```

Outputs are in `out/target/product/fogos` or `out/target/product/rhodep`. Userdebug is the development configuration. Existing board debugging settings are kept in the device trees; review them when preparing a release. The source trees fall back to the public AVB test key when private signing keys are absent. Production signing keys are not published.

## Source layout

| Repository | Purpose |
|---|---|
| [kernel_manifest](https://github.com/Motorola-Sedona61/kernel_manifest) | Pinned kernel/module and helper-tool revisions |
| [kernel_build](https://github.com/Motorola-Sedona61/kernel_build) | `./build.sh FOGOS` / `RHODEP` builder and board profiles |
| [local_manifests](https://github.com/Motorola-Sedona61/local_manifests) | ROM projects, vendor trees and prebuilt sets |
| [kernel_motorola_sm6375-6.1](https://github.com/Motorola-Sedona61/kernel_motorola_sm6375-6.1) | Shared kernel and board DT sources |
| [android_device_motorola_fogos-kernel](https://github.com/Motorola-Sedona61/android_device_motorola_fogos-kernel) | Fogos Android prebuilt layout |
| [android_device_motorola_rhodep-kernel](https://github.com/Motorola-Sedona61/android_device_motorola_rhodep-kernel) | Rhodep Android prebuilt layout |

Successful compilation and staging establish that the build command completes. Generated sets still require device testing; they are not a claim that every hardware feature works.
