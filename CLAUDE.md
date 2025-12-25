# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

fingerprint-ocv is a Linux userspace driver for FPC (Fingerprint Cards) USB fingerprint sensors (USB ID 10a5:9201). It implements the [fprintd D-Bus protocol](https://fprint.freedesktop.org/) for integration with desktop environments like GNOME.

## Build

```bash
git submodule init
git submodule update
cmake -S . -B build
cmake --build build
```

Dependencies (Ubuntu): `libusb-1.0-0-dev libevent-dev libdbus-1-dev libssl-dev libopencv-dev cmake pkg-config gcc g++`

## Architecture

```
src/
├── main.cpp           # Entry point, async loop, D-Bus setup
├── manager.cpp/hpp    # USB hotplug, device management
├── cvext.cpp/hpp      # OpenCV image processing (matching, merging)
└── drv_fpc/           # FPC9201 sensor driver
    ├── fpc9201.cpp/hpp    # USB communication, sensor commands
    ├── crypto.cpp/hpp     # TLS/encryption
    ├── fingerprint.cpp/hpp # Enrollment, template storage
    └── fpcbio.cpp/hpp     # BIO pipe streaming
```

**Key patterns:**
- C++17 with async/await via jinx library (git submodule)
- RAII resource management
- libevent-based async loop

## D-Bus Interface

Implements `net.reactivated.Fprint` on the system bus:
- `net.reactivated.Fprint.Manager`: Device enumeration
- `net.reactivated.Fprint.Device`: Fingerprint enrollment/verification

## Testing

Test executables in `src/`: `test_cvext.cpp`, `test_dbus_call.cpp`, `test_dbus_service.cpp`
