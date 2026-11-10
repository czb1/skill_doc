---
name: fix-android-abi-mismatch
description: Diagnose and fix Android UnsatisfiedLinkError caused by NDK ARM ABI mismatch (symbol not found errors like __aeabi_idiv0). Trigger when user encounters dlopen failed errors referencing ARM ABI symbols, UnsatisfiedLinkError during System.loadLibrary, or questions about APP_ABI/APP_PLATFORM configuration in Android NDK projects. Also trigger when users see crash logs with cannot locate symbol errors in native libraries on Android devices, especially on older or custom ROM devices.

metadata:
  version: 1.0.0
---

# Fix Android NDK ABI Mismatch (UnsatisfiedLinkError)

## Overview

This skill helps diagnose and resolve `java.lang.UnsatisfiedLinkError` crashes in Android applications caused by ARM ABI (Application Binary Interface) mismatch between a compiled native library (.so) and the target device's system libraries. The most common symptom is an error like:

```
java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "__aeabi_idiv0" referenced by "/path/to/libxxx.so"...
```

This error occurs because the native library was compiled for an ARM ABI variant that is incompatible with the device's system runtime libraries.

## Root Cause

The `__aeabi_idiv0` symbol is an ARM EABI (Embedded Application Binary Interface) helper function for integer division by zero handling. It is part of the ARM compiler's runtime support library (`libgcc.a` or `compiler-rt`).

The root cause is one of:
1. **Soft-float ABI vs Hard-float ABI**: The `.so` was compiled for `armeabi` (soft-float, EABI) but the device/system expects `armeabi-v7a` (hard-float, EABIhf). The `__aeabi_idiv0` symbol exists in the soft-float runtime but may be absent in the hard-float runtime on some systems.
2. **NDK Version Mismatch**: The NDK version used to compile the library is incompatible with the Android system version running on the device. Newer NDK versions may reference runtime symbols not present in older system libraries.
3. **System Library Version**: The device runs a heavily customized Android system (common on set-top boxes, smart TVs, IoT devices) where system libraries may have been stripped of certain symbols.

## Step-by-Step Diagnosis

### Step 1: Confirm the Error Pattern

Check the crash log for the following indicators:
- Error type: `java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "__aeabi_idiv0"`
- The symbol name (e.g., `__aeabi_idiv0`, `__aeabi_uidiv`, `__aeabi_i2f`, etc.) — these are all ARM EABI runtime helper symbols
- The `.so` file path where the symbol is referenced

### Step 2: Identify the Device Platform

Ask for or determine the device's Android version and CPU architecture:
```bash
# Get Android version
adb shell getprop ro.build.version.release

# Get device ABI
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.cpu.abilist
```

Common ABIs and their compatibility:
| ABI | Architecture | Float Model | Notes |
|-----|-------------|-------------|-------|
| armeabi | ARMv5/ARMv6 | Soft-float | Deprecated, avoid use |
| armeabi-v7a | ARMv7 | Hard-float (VFP) | Standard for 32-bit ARM |
| arm64-v8a | ARMv8 64-bit | Hard-float | Standard for 64-bit ARM |
| x86 | x86 32-bit | N/A | For emulators and some devices |
| x86_64 | x86 64-bit | N/A | For emulators and some devices |

### Step 3: Examine the Build Configuration

Ask for or locate the NDK build configuration file (`Application.mk` or `CMakeLists.txt`):

For `Application.mk`:
```makefile
APP_ABI := armeabi-v7a          # <-- Check this value
APP_PLATFORM := android-31      # <-- Check this value
```

For `CMakeLists.txt` (CMake):
```cmake
set(ANDROID_ABI "armeabi-v7a")
set(ANDROID_PLATFORM "android-31")
```

**Check for these common misconfigurations:**
- `APP_ABI := armeabi` — **WRONG**: use `armeabi-v7a` or `arm64-v8a` instead
- `APP_ABI := all` — **RISKY**: includes deprecated `armeabi`; explicitly list target ABIs
- `APP_PLATFORM` is missing or too low — set it to match the target device's Android version

### Step 4: Check the Compiled .so File

If you have access to the compiled `.so` file, use these tools to inspect it:

```bash
# Check ELF header and ABI
arm-linux-gnueabi-readelf -a libxxx.so | grep -i abi

# Check needed shared libraries
arm-linux-gnueabi-readelf -d libxxx.so | grep NEEDED

# Search for the missing symbol
arm-linux-gnueabi-nm libxxx.so | grep idiv

# Alternative with llvm tools (if using NDK's toolchain)
<llvm-objdump -a libxxx.so
```

### Step 5: Check NDK Version

```bash
ndk-build --version
# or check the NDK installation directory version
```

NDK version compatibility guidelines:
- Android 12 (API 31/32): NDK r21 or higher
- Android 11 (API 30): NDK r19 or higher
- Android 10 (API 29): NDK r18 or higher
- Older versions: Corresponding NDK matching the API level

## Fixing the Issue

### Fix 1: Correct APP_ABI (Most Common)

Change `armeabi` to `armeabi-v7a` in `Application.mk`:

```makefile
# WRONG (soft-float, deprecated, causes __aeabi_idiv0 issues)
# APP_ABI := armeabi

# CORRECT (hard-float VFP, standard for 32-bit ARM devices)
APP_ABI := armeabi-v7a
```

For 64-bit devices:
```makefile
APP_ABI := arm64-v8a
```

For supporting both:
```makefile
APP_ABI := armeabi-v7a arm64-v8a
```

### Fix 2: Set Correct APP_PLATFORM

Set `APP_PLATFORM` to match the target device's Android API level:

```makefile
APP_PLATFORM := android-31    # For Android 12
```

Reference table:
| Android Version | API Level |
|-----------------|-----------|
| Android 4.4 | 19 |
| Android 5.0 | 21 |
| Android 6.0 | 23 |
| Android 7.0/7.1 | 24/25 |
| Android 8.0/8.1 | 26/27 |
| Android 9.0 | 28 |
| Android 10 | 29 |
| Android 11 | 30 |
| Android 12.0 | 31 |
| Android 12.1/12L | 32 |
| Android 13 | 33 |
| Android 14 | 34 |

### Fix 3: Update NDK Version

Ensure you are using a compatible NDK version. For Android 12 (API 31+), use NDK r21 or higher.

```bash
# Download the appropriate NDK from:
# https://developer.android.com/ndk/downloads

# Or use sdkmanager to install:
sdkmanager "ndk;25.2.9519653"
```

### Fix 4: Full Clean and Rebuild

After making configuration changes, always do a full clean rebuild:

```bash
ndk-build clean
ndk-build

# Or for Gradle-based projects:
./gradlew clean
./gradlew assembleDebug
```

### Fix 5: Check for Prebuilt Libraries

If the project includes prebuilt `.so` files (in `libs/` or `jniLibs/` directories), those also need to be updated:

1. Identify all prebuilt libraries in the project
2. Replace `armeabi` versions with `armeabi-v7a` versions
3. If they are third-party libraries, request the vendor to provide hard-float ABI versions
4. As a temporary workaround, you can delete the `armeabi` folder if `armeabi-v7a` versions exist

## Verification

After applying the fix, verify:

1. **Build succeeds** with no errors
2. **Check the compiled .so**:
   ```bash
   arm-linux-gnueabi-readelf -a libs/armeabi-v7a/libxxx.so | grep -i abi
   # Should show Tag_ABI_VFP_args: VFP registers (for armeabi-v7a)
   ```
3. **Install and test on the device** — the crash should no longer occur
4. **Test on multiple devices** if possible, especially older or customized devices

## Pitfalls to Avoid

1. **Don't use `armeabi` (deprecated)**: Google deprecated `armeabi` in NDK r16 and removed it in r17. Always use `armeabi-v7a` for 32-bit ARM.
2. **Don't use `APP_ABI := all` blindly**: This includes deprecated ABIs. Explicitly list only the ABIs you need.
3. **NDK version matters**: Using an NDK that's too old for the target API level can cause symbol lookup failures. Always check NDK compatibility.
4. **Prebuilt libraries can override your build**: If prebuilt libraries in `libs/` or `jniLibs/` are the wrong ABI, they will be packaged into the APK instead of your newly compiled ones.
5. **Custom ROMs and set-top boxes**: Devices running heavily customized Android (TVs, set-top boxes, IoT) may have non-standard system libraries. Test on the actual target device.

## Example

**Problem**: App crashes on an Android 12 set-top box with:
```
E AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "__aeabi_idiv0" referenced by "/system/lib/libjni_cpe.so"...
```

**Diagnosis**:
1. The `.so` was compiled with `armeabi` ABI (soft-float)
2. The device's system libraries don't include `__aeabi_idiv0`
3. Build config had `APP_ABI := armeabi-v7a` but a prebuilt library was still using `armeabi`

**Fix Applied**:
1. Replaced the prebuilt library with an `armeabi-v7a` version
2. Performed a full clean rebuild
3. Verified the new `.so` uses hard-float ABI (`Tag_ABI_VFP_args: VFP registers`)
4. App no longer crashes

## Common ARM EABI Symbols

| Symbol | Description |
|--------|-------------|
| `__aeabi_idiv0` | Signed integer division by zero handler |
| `__aeabi_uidiv` | Unsigned integer division (32-bit) |
| `__aeabi_idiv` | Signed integer division (32-bit) |
| `__aeabi_i2f` | Integer to float conversion |
| `__aeabi_f2iz` | Float to integer conversion (truncate toward zero) |
| `__aeabi_memcpy` | Memory copy (aligned) |
| `__aeabi_memset` | Memory set (aligned) |
| `__aeabi_cfcmple` | Float comparison (ordered) |

These symbols are part of the ARM EABI runtime library and are normally provided by the C/C++ runtime (`libgcc` or `compiler-rt`). When the ABI of the compiled code doesn't match the system's runtime, these symbols may not be found.

## See Also

For more detailed reference, see `references/abi-reference.md`.

To inspect a compiled `.so` file for ABI issues programmatically, use `scripts/check_abi.sh`.
