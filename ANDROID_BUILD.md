# Building mise for Termux/Android

## Current Status

Work in progress to build mise for aarch64-linux-android (Termux on Android).

## Changes Made

### 1. Replaced sys-info with os-release (COMPLETED)
- **File**: `Cargo.toml:145`
  - Removed: `sys-info = "0.9"`
  - Already has: `os-release = "0.1"` at line 108
- **File**: `src/env.rs:620-624`
  - Changed `linux_distro()` to use `os_release::OS_RELEASE.as_ref().ok().map(|release| release.id.clone())`
  - Reason: `sys-info` doesn't compile for Android targets

### 2. Fixed vfox sigstore-verification dependency (COMPLETED)
- **File**: `crates/vfox/Cargo.toml:47`
  - Changed: `sigstore-verification = { version = "0.1.7", default-features = false }`
  - Added to line 58: `rustls = [..., "sigstore-verification/rustls"]`
  - Reason: Default features pull in native-tls

### 3. Android NDK Setup (ALREADY CONFIGURED)
- **File**: `.cargo/config.toml:5-6, 17-19`
  - `ANDROID_NDK_ROOT = "/home/android-ndk-r27"`
  - `ANDROID_NDK = "/home/android-ndk-r27"`
  - Toolchain configuration for aarch64-linux-android
- **File**: `Cross.toml:46-63`
  - Android target configurations for cross-compilation
- **NDK Location**: `/home/android-ndk-r27` (NDK r27 installed)

## Current Problem

### OpenSSL/native-tls Issue
Building fails with: `error: failed to run custom build command for openssl-sys v0.9.111`

**Root Cause**: Many transitive dependencies pull in `reqwest` with **default features enabled**, which includes `default-tls` → `native-tls` → `openssl-sys`. OpenSSL doesn't cross-compile to Android easily.

**Dependencies pulling reqwest with defaults**:
- gix-transport (from gix)
- oauth2 (from sigstore → openidconnect)
- oci-client (from sigstore)
- self_update
- sigstore
- tough (from sigstore)
- ubi
- xx

Even when building with `--no-default-features --features rustls`, we get BOTH rustls AND native-tls simultaneously.

## Functionality Impact Analysis

### What Each Dependency Provides:

#### 1. **gix** (Git library) - IMPORTANT
**Used for:**
- `src/git.rs` - Core git operations (clone, update, fetch, etc.)
- `src/backend/spm.rs` - Swift Package Manager backend
- `src/task/task_file_providers/remote_task_git.rs` - Remote git task files
- Many other files for git repository operations

**Impact if removed:**
- ❌ Cannot install tools from git repositories
- ❌ Cannot use asdf plugins (most are git-based)
- ❌ Cannot use remote task files from git
- ❌ Swift Package Manager backend won't work
- ⚠️ **CRITICAL** - This breaks a core feature of mise

**Can it be made optional?** Yes, but would significantly limit functionality.

#### 2. **sigstore-verification** (Cosign signature verification)
**Used for:**
- `src/backend/aqua.rs:709-900` - Verify checksums with cosign for Aqua registry packages
- `crates/vfox/src/` - VFox plugin signature verification

**Impact if removed:**
- ⚠️ Cannot verify signatures on Aqua registry packages
- ⚠️ Security downgrade - no cryptographic verification
- ✅ Most functionality remains (checksums still work, just not cosign verification)
- Settings has `aqua.cosign = false` option to disable this

**Can it be made optional?** YES - Already has a runtime setting to disable.

#### 3. **self_update** (Auto-update functionality)
**Used for:**
- `src/cli/self_update.rs` - `mise self-update` command
- `src/cli/doctor/mod.rs` - Update check warnings
- `src/cli/version.rs` - Version information
- `src/config/mod.rs` - Update notifications

**Impact if removed:**
- ❌ Cannot run `mise self-update` command
- ✅ Can still update manually via package manager (preferred on Termux anyway)
- ✅ All other functionality remains
- ℹ️ Termux users typically use `pkg upgrade mise` instead

**Can it be made optional?** YES - Not critical for Termux/Android users.

## Recommended Solution for Android

### Create Android-Specific Build Profile

**Keep:**
- ✅ Core tool management
- ✅ Task execution
- ✅ Environment management
- ✅ HTTP-based tool downloads (GitHub releases, etc.)
- ✅ Aqua registry (without cosign verification)
- ✅ Most backends: cargo, npm, go, ubi, etc.

**Remove for Android:**
- ❌ `self_update` - Not needed (use Termux package manager)
- ❌ `sigstore-verification` - Security downgrade but avoids OpenSSL

**Make Conditional (use git command-line instead of gix library):**
- ⚠️ `gix` - Fall back to shelling out to `git` command (if available on system)
  - Termux has `git` package available
  - Would lose some advanced git features but keep basic clone/update

### Implementation Approach

**Option A: Minimal Changes (Recommended)**
1. Keep `gix` - it's too important
2. Remove `self_update` and `sigstore-verification` features for Android
3. Patch remaining reqwest dependencies to disable default features
4. Add Android-specific feature flag

**Option B: Shell Out to Git**
1. Create Android feature that replaces `gix` library calls with `git` CLI commands
2. Remove `self_update` and `sigstore-verification`
3. Simpler dependency tree, but less robust git handling

**Option C: Build OpenSSL for Android (Complex)**
1. Cross-compile OpenSSL for Android
2. Keep all features
3. Most work but full functionality

## Build Commands Tested

```bash
# Environment variables needed:
export ANDROID_NDK_ROOT=/home/android-ndk-r27
export ANDROID_NDK=/home/android-ndk-r27
export CMAKE_TOOLCHAIN_FILE=/home/android-ndk-r27/build/cmake/android.toolchain.cmake
export ANDROID_ABI=arm64-v8a
export ANDROID_PLATFORM=android-21
export CC=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang
export CXX=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang++
export AR=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar
export CC_aarch64_linux_android=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang
export CXX_aarch64_linux_android=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang++
export AR_aarch64_linux_android=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar
export CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang

# Add Rust target
rustup target add aarch64-linux-android

# Attempted build command:
cargo build --release --target aarch64-linux-android --no-default-features --features rustls
```

## Next Steps

### Recommended: Option A - Keep gix, remove self_update & sigstore

1. **Disable self_update for Android:**
   - Make `self_update` dependency optional
   - Guard self-update code with `#[cfg(not(target_os = "android"))]`

2. **Disable sigstore for Android:**
   - Make `sigstore-verification` optional
   - Set `aqua.cosign = false` by default on Android
   - Guard cosign code with feature flags

3. **Patch remaining reqwest users:**
   - This might still require work but with fewer dependencies

4. **Alternative: Try using only rustls crates:**
   - Fork/patch problem dependencies if needed
   - Use `[patch.crates-io]` to override dependency features

## Files Modified (Summary)
- `Cargo.toml` - removed sys-info duplicate
- `src/env.rs:620-624` - use os-release instead of sys-info
- `crates/vfox/Cargo.toml:47,58` - sigstore-verification with rustls

## Verification Commands

```bash
# Check what pulls in openssl
cargo tree --target aarch64-linux-android --no-default-features --features rustls -i openssl-sys

# Check reqwest features being used
cargo tree --target aarch64-linux-android --no-default-features --features rustls -i native-tls

# See all reqwest dependencies
cargo metadata --format-version=1 | jq -r '.packages[] | select(.dependencies[]? | .name == "reqwest") | {name: .name, reqwest_dep: (.dependencies[] | select(.name == "reqwest") | {default_features, features})}'
```

## Notes
- Android NDK r27 is installed at `/home/android-ndk-r27`
- Toolchain binaries at `/home/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/`
- API level 21 (Android 5.0) minimum target
- Cross.toml already configured for Android targets
- **Git is IMPORTANT** - Used by many backends and asdf plugins
- Self-update is NOT important for Termux (has package manager)
- Sigstore is nice-to-have security feature but not critical
