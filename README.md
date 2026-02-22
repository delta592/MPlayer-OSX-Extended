MPlayer OSX Extended
====================

## Requirements

- macOS 11.0 (Big Sur) or later
- Apple Silicon Mac (M1/M2/M3/M4)

## Architecture

This version supports **Apple Silicon (arm64) only**. Intel Macs are not supported.

Homepage:
http://www.mplayerosx.ch/

Issue tracker:
https://github.com/sttz/MPlayer-OSX-Extended/issues

Downloads:
http://code.google.com/p/mplayerosxext/downloads/list

Build Instructions
------------------

The project requires Xcode 14 or later with arm64 SDK support.
Make sure you've got the dependencies below and then you should be able to compile MPE directly with Xcode.

### Dependencies

All dependencies must be built for arm64 (Apple Silicon).

#### Sparkle

MPE requires [Sparkle](https://sparkle-project.org/) 2.x with arm64 support. The project historically used a [custom Sparkle fork](https://github.com/sttz/Sparkle) for binary bundle updates.

#### Fontconfig & Freetype

Build Fontconfig and Freetype for arm64:

```bash
source extras/scripts/mposx_preparebuild arm64
./configure --host=aarch64-apple-darwin --prefix=$BUILD_ROOT
make && make install
```

#### MPlayer Binary

See `extras/scripts/README` for detailed instructions on building the MPlayer binary for arm64.
