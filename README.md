# NVIDIA graphics for the Android Emulator on Linux

Use the NVIDIA GPU for Android Emulator graphics, while keeping CPU virtualization enabled through KVM. This guide covers hybrid Intel/NVIDIA laptops running Ubuntu or a closely related Linux distribution.

The setup in this repository was verified on one laptop. Your results depend on your CPU, GPU, memory, display server, driver, emulator version, and Android Virtual Device (AVD). Using the NVIDIA GPU can improve graphics rendering; it cannot remove CPU bottlenecks or guarantee that every emulator will run without lag.

## What this setup changes

- Selects NVIDIA as the system graphics profile with Ubuntu's PRIME tool, when that is the desired battery/performance tradeoff.
- Adds NVIDIA PRIME render-offload variables for GPU-accelerated apps in the logged-in session.
- Starts the Android Emulator with host GPU rendering (`-gpu host`).
- Uses KVM for hardware-accelerated x86_64 guest CPU emulation.
- Provides checks to confirm which GPU rendered the emulator.

The laptop used for verification had an Intel UHD 630, NVIDIA GeForce GTX 1650 Mobile with 4 GB VRAM, Intel Core i5-9300H (4 cores / 8 threads), and 16 GB installed RAM (about 14 GiB available to Linux). It ran Ubuntu 26.04, GNOME on Wayland, NVIDIA driver 595, and Android Emulator 37.1.11 with an Android 35 x86_64 Pixel 6 Pro AVD. The emulator successfully selected the GTX 1650 for Vulkan/OpenGL rendering, KVM was usable, and Android booted. Those results are a reference, not a performance promise for other machines.

## 1. Check your hardware and driver

```bash
lspci -nnk | rg -A3 -i 'vga|3d|display'
nvidia-smi
prime-select query
```

Confirm that the NVIDIA GPU appears, the kernel is using the NVIDIA driver, and `nvidia-smi` shows a driver and GPU. On Ubuntu, install or select a compatible proprietary driver through **Software & Updates → Additional Drivers**, or use Ubuntu's driver tools. Reboot after a driver change.

On Ubuntu systems that provide `prime-select`, choose the NVIDIA system profile with:

```bash
sudo prime-select nvidia
```

Then reboot or log out and back in. Check with `prime-select query`. The `nvidia` profile can use more battery and keep the NVIDIA GPU active. PRIME render offload is a better battery tradeoff when only selected applications need NVIDIA. Don't switch profiles if your system does not provide `prime-select`; use the distribution's graphics settings instead.

**On the verified laptop, `prime-select query` already returned `nvidia`.** That system-wide mode was already selected; the change made for application rendering was the user-session environment file and emulator launcher below.

## 2. Enable NVIDIA render offload for your user session

Create `~/.config/environment.d/90-nvidia.conf` with:

```ini
__NV_PRIME_RENDER_OFFLOAD=1
__GLX_VENDOR_LIBRARY_NAME=nvidia
__VK_LAYER_NV_optimus=NVIDIA_only
```

These variables request NVIDIA PRIME offload for OpenGL and Vulkan applications. They do not move CPU work to the GPU, and they don't mean every app will use NVIDIA. After creating or changing the file, log out and back in so newly started desktop applications inherit the environment.

You can also offload one application without changing the session. The included `scripts/nvidia-run` helper does this for a command:

```bash
./scripts/nvidia-run glxinfo -B
```

If `glxinfo` is installed, its renderer should mention NVIDIA. The session-wide file is optional when using the one-command helper.

## 3. Use NVIDIA rendering in the Android Emulator

In Android Studio's **Device Manager**, edit the AVD and set **Emulated Performance → Graphics** to **Hardware**. With a recent Android Emulator, the equivalent AVD configuration is:

```ini
hw.gpu.enabled=yes
hw.gpu.mode=host
```

Start the AVD from a terminal with NVIDIA offload and host rendering:

```bash
./scripts/android-emulator-nvidia -avd Pixel_6_Pro
```

Replace `Pixel_6_Pro` with your AVD name. The script finds the SDK using `ANDROID_SDK_ROOT`, `ANDROID_HOME`, or the common `~/Android/Sdk` location. To add it to your PATH, copy the scripts into `~/.local/bin` or run the script by its full path.

Host GPU rendering accelerates graphics. The emulator still needs a compatible driver, and host rendering can expose driver-specific bugs. If the emulator reports that the GPU cannot be used, inspect the NVIDIA driver and emulator logs before switching graphics modes.

## 4. Enable and check KVM CPU acceleration

Graphics acceleration and virtual CPU acceleration are separate. On Linux, Android Emulator uses KVM for x86/x86_64 guest CPU acceleration. Check it with the emulator included in your Android SDK:

```bash
"$ANDROID_SDK_ROOT/emulator/emulator" -accel-check
```

If `ANDROID_SDK_ROOT` is not set, substitute the path to your SDK's `emulator` executable. The check should report that KVM is installed and usable. Follow the [Android Emulator acceleration guide](https://developer.android.com/studio/run/emulator-acceleration) for KVM setup on your distribution and hardware. CPU acceleration requires virtualization support enabled in firmware and usable KVM access; an NVIDIA GPU does not replace it.

## 5. Confirm the emulator is using NVIDIA

Start the AVD with `-verbose` to inspect renderer selection:

```bash
./scripts/android-emulator-nvidia -avd Pixel_6_Pro -verbose
```

Look for the NVIDIA device in the Vulkan device list and a graphics adapter/renderer naming the NVIDIA GPU. While the AVD is running, check:

```bash
nvidia-smi
```

The emulator's `qemu-system-x86_64` process should appear when it is actively rendering. Low utilization while the emulator is idle is normal. Check `emulator -accel-check` separately to verify KVM.

## 6. Tune for a laptop

The NVIDIA GPU only speeds up supported graphics work. Godot, Android Studio builds, Gradle, and much of Android guest execution also depend on the CPU, memory, storage, and thermals.

- Use an x86_64 system image on an x86_64 Intel/AMD laptop to use KVM acceleration.
- Start at the AVD's native resolution and 100% resolution scale. A lower-resolution or smaller AVD can reduce graphics and memory load.
- Assign a reasonable amount of RAM. On a 16 GB laptop, 4–6 GB for one AVD is a practical starting point when Android Studio and an IDE are also open. Avoid assigning most of host memory to the emulator.
- Assign CPU cores conservatively. More virtual cores do not make a 4-core host CPU faster; leave room for Linux and Android Studio.
- Close other emulators and memory-heavy apps when diagnosing stutter. Keep the laptop plugged in and check its power/performance profile and temperatures.
- Let the emulator finish its first boot and shader compilation before judging responsiveness.

The verified AVD had four virtual CPU cores and 8 GB RAM. That configuration booted successfully, but it can leave less memory for other applications on a 16 GB laptop. It should not be copied blindly as an optimization.

## Troubleshooting

### `nvidia-smi` fails or the NVIDIA GPU is missing

The proprietary NVIDIA driver may not be installed, loaded, or compatible with the current kernel. Check `lspci -nnk`, your distribution's driver utility, and reboot after installing a driver.

### Emulator says the GPU cannot be used

Confirm `nvidia-smi` works, start the emulator with the included launcher, and inspect `-verbose` output for renderer and Vulkan-device selection. If an emulator or driver update introduced a rendering regression, try another supported emulator graphics backend/version or the software renderer for diagnosis; software rendering may be slower.

### KVM is unavailable

Confirm CPU virtualization is enabled in BIOS/UEFI, install your distribution's KVM packages, and follow the Android acceleration guide for device permissions. Log out/in or reboot after changing group membership. Do not troubleshoot KVM by changing NVIDIA settings: these are independent paths.

### It still lags

Check CPU load, available RAM, swap activity, GPU selection, emulator resolution, laptop power mode, and thermal throttling. A GTX 1650 cannot compensate for CPU saturation, memory pressure, or an AVD that is too large for the host.

## Included scripts

- `scripts/nvidia-run` — run one command with PRIME render-offload variables.
- `scripts/android-emulator-nvidia` — locate the Android SDK and launch an AVD with NVIDIA offload and `-gpu host`.

Both scripts are user-level helpers and do not install drivers, change system-wide PRIME mode, or require root.

## References

- [Android Emulator hardware acceleration](https://developer.android.com/studio/run/emulator-acceleration)
- [Android Emulator command-line options](https://developer.android.com/studio/run/emulator-commandline)
- [NVIDIA PRIME Render Offload](https://download.nvidia.com/XFree86/Linux-x86_64/435.21/README/primerenderoffload.html)
