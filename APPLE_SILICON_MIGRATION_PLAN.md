# MPlayer OSX Extended: Apple Silicon Migration Plan

This document outlines a comprehensive plan to update MPlayer OSX Extended to support **Apple Silicon (arm64) only**, removing all Intel (x86_64/i386) and PowerPC (ppc/ppc64) code paths.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Phase 1: Xcode Project Configuration](#3-phase-1-xcode-project-configuration)
4. [Phase 2: Build Scripts Updates](#4-phase-2-build-scripts-updates)
5. [Phase 3: Source Code Changes](#5-phase-3-source-code-changes)
6. [Phase 4: Binary Bundle System](#6-phase-4-binary-bundle-system)
7. [Phase 5: UI and Preferences](#7-phase-5-ui-and-preferences)
8. [Phase 6: Dependencies and Libraries](#8-phase-6-dependencies-and-libraries)
9. [Phase 7: Documentation](#9-phase-7-documentation)
10. [Phase 8: Testing and Validation](#10-phase-8-testing-and-validation)
11. [Summary Checklist](#11-summary-checklist)

---

## 1. Overview

### Current State
- The project supports: x86_64, i386, ppc, ppc64, ppcg3
- Universal binaries are created combining x86_64 + i386
- Runtime architecture detection uses `hw.optional.x86_64` sysctl
- No ARM64 support exists

### Target State
- Single architecture: arm64 (Apple Silicon)
- No universal binary creation needed
- Simplified build and runtime code
- Minimum deployment target: macOS 11.0 (Big Sur) - first macOS with Apple Silicon

---

## 2. Prerequisites

Before starting the migration:

1. **Development Environment**
   - Apple Silicon Mac (M1/M2/M3/M4)
   - Xcode 14+ (with arm64 SDK support)
   - macOS 11.0+ SDK

2. **Dependencies**
   - Rebuild all third-party libraries for arm64:
     - MPlayer/FFmpeg (core)
     - Fontconfig
     - Freetype
     - Any other bundled libraries
   - Verify Sparkle framework has arm64 support

3. **Testing**
   - Apple Silicon Mac for native testing
   - Various video formats for playback testing

---

## 3. Phase 1: Xcode Project Configuration

### File: `mposxext.xcodeproj/project.pbxproj`

#### 3.1 Update Deployment Target

| Setting | Current | New |
|---------|---------|-----|
| MACOSX_DEPLOYMENT_TARGET | 10.6/10.7 | 11.0 |

**Lines to modify:** ~1290, ~1321, ~1355, ~1393

```
MACOSX_DEPLOYMENT_TARGET = 11.0;
```

#### 3.2 Set Architecture to arm64 Only

Add/modify build settings:

```
ARCHS = arm64;
VALID_ARCHS = arm64;
EXCLUDED_ARCHS = i386 x86_64;
```

#### 3.3 Remove Build Settings No Longer Needed

- Remove `ONLY_ACTIVE_ARCH = YES` (or set explicitly for arm64)
- Remove any x86_64/i386 specific conditional build settings

#### 3.4 Update SDK Version

```
SDKROOT = macosx;  // (already correct, will use latest)
```

---

## 4. Phase 2: Build Scripts Updates

### 4.1 File: `extras/scripts/mposx_preparebuild`

**Current architecture handling (lines 76-100):**

Replace the entire architecture section with arm64-only:

```bash
# REMOVE these sections:
# - x86_64 flags (lines 76-80)
# - i386 flags (lines 83-87)
# - universal binary flags (lines 89-94)
# - i386 text relocation fix (lines 96-100)

# ADD arm64 configuration:
export CFLAGS="$CFLAGS -arch arm64"
export LDFLAGS="$LDFLAGS -arch arm64"
export CXXFLAGS="$CXXFLAGS -arch arm64"
```

**Update deployment target (line 58):**
```bash
# Change from:
export MACOSX_DEPLOYMENT_TARGET=10.6
# To:
export MACOSX_DEPLOYMENT_TARGET=11.0
```

**Update default target system (line 23):**
```bash
# Change from:
target_system="x86_64"
# To:
target_system="arm64"
```

### 4.2 File: `extras/scripts/mplayer/build`

**Update supported targets (line 48):**
```bash
# Change from:
targets=( "x86_64" "i386" "ppc" "ppcg3" "all" )
# To:
targets=( "arm64" )
```

**Remove/simplify sections:**
- Remove "all" target logic (lines 158-166)
- Remove PPC crosscompile configuration (lines 290-298)
- Remove universal binary merge logic (lines 371-375)
- Update binary naming to use `mplayer.arm64` or just `mplayer` (line 368)

**Add arm64-specific configuration if needed:**
```bash
# arm64-specific MPlayer configure options
if [ "$target_system" = "arm64" ]; then
    config_opts="$config_opts --enable-neon"  # NEON SIMD on ARM
fi
```

### 4.3 File: `extras/scripts/mposx_install`

**Complete rewrite needed:**

Current script expects `mplayer.i386` and `mplayer.x86_64` and creates universal binary.

**New approach:**
```bash
#!/bin/bash
# Simplified installation for arm64-only

# Check for arm64 binary
if [ ! -f "$dir/mplayer.arm64" ] && [ ! -f "$dir/mplayer" ]; then
    echo "mplayer arm64 binary not found at '$dir'."
    exit 1
fi

# Copy binary (no lipo needed)
if [ -f "$dir/mplayer.arm64" ]; then
    cp "$dir/mplayer.arm64" "$binary_container/Contents/MacOS/mplayer"
else
    cp "$dir/mplayer" "$binary_container/Contents/MacOS/mplayer"
fi

# Library collection and install_name_tool processing remains the same
# (otool -L and install_name_tool work identically on arm64)
```

**Remove:**
- lipo commands (lines 84, 162, 192)
- References to `.i386`, `.x86_64`, `.ub` suffixes
- Universal binary merge logic

### 4.4 File: `extras/scripts/ffmpeg/build`

**Update architecture handling (lines 323-338):**

Remove universal binary creation with lipo. Replace with arm64-only build:

```bash
# Remove the lipo-based universal binary creation
# Just build arm64 libraries directly
```

---

## 5. Phase 3: Source Code Changes

### 5.1 File: `source/core/MPlayerInterface.m`

#### Remove x86_64 host detection (lines 149-154)

```objc
// REMOVE:
int is64bit;
size_t len = sizeof(is64bit);
if (!sysctlbyname("hw.optional.x86_64",&is64bit,&len,NULL,0))
    is64bitHost = (BOOL)is64bit;

// REPLACE with (or just remove entirely as it's always true on AS):
is64bitHost = YES;  // Always 64-bit on Apple Silicon
```

#### Remove 32-bit binary forcing logic (lines 334-341)

```objc
// REMOVE entire block:
force32bitBinary = NO;
if (is64bitHost && [prefs boolForKey:MPEUse32bitBinaryon64bit]) {
    NSArray *arches = [pc objectForInfoKey:@"MPEBinaryArchs"
                                   ofBinary:[cPrefs objectForKey:MPESelectedBinary]];
    if ([arches containsObject:@"i386"])
        force32bitBinary = YES;
}

// REPLACE with:
// (Remove force32bitBinary variable entirely)
```

#### Remove arch command wrapper (lines 1245-1251)

```objc
// REMOVE:
if (force32bitBinary) {
    [myMplayerTask setLaunchPath:@"/usr/bin/arch"];
    [aParams insertObject:@"-i386" atIndex:0];
    [aParams insertObject:myPathToPlayer atIndex:1];
} else
    [myMplayerTask setLaunchPath:myPathToPlayer];

// REPLACE with:
[myMplayerTask setLaunchPath:myPathToPlayer];
```

#### Remove instance variable
- Remove `is64bitHost` ivar declaration
- Remove `force32bitBinary` ivar declaration

### 5.2 File: `source/core/BinaryBundle.h`

Keep `executableArchitectureStrings` method but update for arm64.

### 5.3 File: `source/core/BinaryBundle.m`

#### Update supported architectures string (line 35)

```objc
// Change from:
static NSString* const checkForArches = @"x86_64,i386,ppc64,ppc";
// To:
static NSString* const checkForArches = @"arm64";
```

#### Update architecture detection logic (lines 91-149)

The Mach-O parsing code should already work for arm64 since it uses standard headers. Verify:
- `MH_MAGIC_64` is used for arm64 (64-bit Mach-O)
- CPU_TYPE_ARM64 constant may need to be added

```objc
// Add if not present:
#ifndef CPU_TYPE_ARM64
#define CPU_TYPE_ARM64 0x0100000C
#endif

// Update architecture name mapping to include arm64
```

### 5.4 File: `source/controllers/PreferencesController2.m`

#### Simplify `binaryHasCompatibleArch:` method (lines 478-516)

```objc
- (BOOL) binaryHasCompatibleArch:(BinaryBundle *)bundle
{
    NSArray *binaryArches = [bundle executableArchitectureStrings];

    // On Apple Silicon, only arm64 binaries are compatible
    // (Rosetta 2 could run x86_64, but we're removing that support)
    return [binaryArches containsObject:@"arm64"];
}
```

#### Remove 64-bit capability check (lines 492-496)

```objc
// REMOVE entire block:
int is64bitCapable;
size_t len = sizeof(is64bitCapable);
if (sysctlbyname("hw.optional.x86_64",&is64bitCapable,&len,NULL,0))
    is64bitCapable = NO;
```

### 5.5 File: `source/core/Preferences.h`

#### Remove obsolete preference key (line 99)

```objc
// REMOVE:
#define MPEUse32bitBinaryon64bit @"MPEUse32bitBinaryon64bit"
```

### 5.6 File: `source/core/Preferences.m` (if exists)

Remove registration of `MPEUse32bitBinaryon64bit` default value.

---

## 6. Phase 4: Binary Bundle System

### 6.1 Update Binary Bundle Expectations

**Info.plist for binary bundles:**

The `MPEBinaryArchs` key should now only contain:
```xml
<key>MPEBinaryArchs</key>
<array>
    <string>arm64</string>
</array>
```

### 6.2 Remove Old Binary Bundles

Delete any existing binary bundles in `binaries/` that are Intel/PPC only:
- `binaries/mpextended.mpBinaries` (if Intel-only)

### 6.3 Create New arm64 Binary Bundle

Build new MPlayer binary for arm64 and create bundle structure:
```
NewBundle.mpBinaries/
├── Contents/
│   ├── Info.plist
│   ├── MacOS/
│   │   ├── mplayer
│   │   └── lib/
│   │       └── (arm64 dylibs)
│   └── Resources/
│       └── dsa_pub.pem
```

---

## 7. Phase 5: UI and Preferences

### 7.1 File: `resources/nibs/Preferences.xib`

#### Update architecture display text (line 1956)

```xml
<!-- Change from: -->
<string>ppc, i386, x86_64</string>
<!-- To: -->
<string>arm64</string>
```

#### Remove/update help text (line 1799)

Update the text about "different architectures" since there's only one now:
```
"Double click a binary to select it for playback. You can download and install additional ones with special features."
```

### 7.2 Remove 32-bit Binary Preference

If there's a UI checkbox for "Use 32-bit binary on 64-bit host":
- Remove the checkbox from Preferences.xib
- Remove associated outlet/action connections

---

## 8. Phase 6: Dependencies and Libraries

### 8.1 Rebuild All Dependencies for arm64

#### MPlayer/FFmpeg Core
```bash
# Configure for arm64
./configure --arch=arm64 --enable-neon ...
make
```

#### Fontconfig
```bash
./configure --host=aarch64-apple-darwin ...
```

#### Freetype
```bash
./configure --host=aarch64-apple-darwin ...
```

### 8.2 Verify Sparkle Framework

Ensure Sparkle.framework includes arm64:
```bash
lipo -info Sparkle.framework/Sparkle
# Should show: arm64
```

If using an old version, update to Sparkle 2.x which supports arm64.

### 8.3 Update Framework Search Paths

In Xcode project, verify framework paths point to arm64-compatible frameworks.

---

## 9. Phase 7: Documentation

### 9.1 Update README.md

Add Apple Silicon requirement:
```markdown
## Requirements
- macOS 11.0 (Big Sur) or later
- Apple Silicon Mac (M1/M2/M3/M4)

## Architecture
This version supports Apple Silicon (arm64) only.
Intel Macs are not supported.
```

### 9.2 Update Build Scripts README

**File: `extras/scripts/README`**

Update all references to architectures:
- Remove mentions of x86_64, i386, ppc
- Document arm64-only build process
- Remove universal binary documentation

### 9.3 Update Changelog

Document the architecture change in any changelog or release notes.

---

## 10. Phase 8: Testing and Validation

### 10.1 Build Verification

```bash
# Verify binary is arm64-only
file MPlayer\ OSX\ Extended.app/Contents/MacOS/MPlayer\ OSX\ Extended
# Expected: Mach-O 64-bit executable arm64

# Check for any Intel code
lipo -info MPlayer\ OSX\ Extended.app/Contents/MacOS/MPlayer\ OSX\ Extended
# Expected: Non-fat file... architecture: arm64

# Verify mplayer binary
file MPlayer\ OSX\ Extended.app/Contents/Resources/Binaries/*/Contents/MacOS/mplayer
# Expected: Mach-O 64-bit executable arm64
```

### 10.2 Runtime Testing

1. **Application Launch**
   - Verify app launches on Apple Silicon Mac
   - Check Activity Monitor shows "Apple" under Architecture column

2. **Playback Testing**
   - Test various video formats (H.264, HEVC, VP9, etc.)
   - Test audio formats
   - Test subtitle rendering

3. **Performance Testing**
   - Compare CPU usage to Rosetta 2 emulated version
   - Verify hardware acceleration works (VideoToolbox)

4. **Preference Testing**
   - Verify binary selection works
   - Confirm removed preferences don't cause crashes

### 10.3 Clean Build Test

```bash
# Clean and rebuild
xcodebuild clean
xcodebuild -configuration Release ARCHS=arm64
```

---

## 11. Summary Checklist

### Xcode Project
- [ ] Update MACOSX_DEPLOYMENT_TARGET to 11.0
- [ ] Set ARCHS = arm64
- [ ] Set VALID_ARCHS = arm64
- [ ] Add EXCLUDED_ARCHS = i386 x86_64
- [ ] Verify SDK settings

### Build Scripts
- [ ] Update `mposx_preparebuild` - remove x86/ppc flags, add arm64
- [ ] Update `mplayer/build` - arm64 target only
- [ ] Rewrite `mposx_install` - remove lipo/universal binary logic
- [ ] Update `ffmpeg/build` - arm64 only

### Source Code
- [ ] `MPlayerInterface.m` - Remove 64-bit detection, 32-bit forcing
- [ ] `BinaryBundle.m` - Update architecture list to arm64
- [ ] `PreferencesController2.m` - Simplify compatibility check
- [ ] `Preferences.h` - Remove MPEUse32bitBinaryon64bit

### UI/Resources
- [ ] Update Preferences.xib architecture display
- [ ] Remove 32-bit binary preference UI

### Dependencies
- [ ] Rebuild MPlayer/FFmpeg for arm64
- [ ] Rebuild Fontconfig for arm64
- [ ] Rebuild Freetype for arm64
- [ ] Verify/update Sparkle framework

### Documentation
- [ ] Update README.md with requirements
- [ ] Update build scripts README
- [ ] Update changelog

### Testing
- [ ] Build verification (file/lipo commands)
- [ ] Launch testing on Apple Silicon
- [ ] Playback testing
- [ ] Performance validation

---

## Appendix: Architecture Constants Reference

For reference when updating code:

| Architecture | CPU Type Constant | Mach-O Magic |
|--------------|-------------------|--------------|
| arm64 | CPU_TYPE_ARM64 (0x0100000C) | MH_MAGIC_64 |
| x86_64 | CPU_TYPE_X86_64 (0x01000007) | MH_MAGIC_64 |
| i386 | CPU_TYPE_I386 (0x00000007) | MH_MAGIC |
| ppc | CPU_TYPE_POWERPC (0x00000012) | MH_MAGIC |
| ppc64 | CPU_TYPE_POWERPC64 (0x01000012) | MH_MAGIC_64 |

---

*Plan created: 2026-02-22*
*Target: Apple Silicon (arm64) only*
*Minimum macOS: 11.0 (Big Sur)*
