# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

OpenThread Border Router (OTBR) connects Thread networks to IP-based networks (Wi-Fi/Ethernet). This is a POSIX-based implementation that provides routing, mDNS/SRP service discovery, DHCPv6 Prefix Delegation, NAT64, and external Thread commissioning.

## Build System

### Initial Setup

Initialize dependencies and install required packages:
```bash
script/bootstrap
```

This script:
- Initializes git submodules (OpenThread core in `third_party/openthread/repo`)
- Installs build dependencies (cmake, ninja, dbus, jsoncpp, etc.)
- On macOS: uses Homebrew
- On Linux: uses apt-get, dnf/yum, or opkg depending on distribution

### Building

Build the project with CMake + Ninja:
```bash
script/cmake-build
```

Build with specific options:
```bash
script/cmake-build -DOTBR_DBUS=ON -DOTBR_WEB=ON
```

Build specific target:
```bash
OTBR_TARGET="otbr-agent" script/cmake-build
```

Build to custom directory:
```bash
OTBR_BUILD_DIR="./build/temp" script/cmake-build
```

Default build directory: `build/otbr`

### Key CMake Build Options

Important build options (defined in `etc/cmake/options.cmake`):
- `OTBR_DBUS` - Enable DBus server (OFF by default)
- `OTBR_WEB` - Enable Web GUI (OFF by default)
- `OTBR_REST` - Enable REST server (OFF by default)
- `OTBR_MDNS` - mDNS provider: "avahi", "mDNSResponder", or "openthread" (default: "openthread")
- `OTBR_BORDER_AGENT` - Enable Border Agent (ON by default)
- `OTBR_BACKBONE_ROUTER` - Enable Backbone Router (ON by default)
- `OTBR_BORDER_ROUTING` - Enable Border Routing Manager (ON by default)
- `OTBR_TREL` - Enable TREL link support (ON by default)
- `OTBR_COVERAGE` - Enable coverage reporting (OFF by default)

## Testing

### Running Tests

Build and run all tests:
```bash
script/test build check
```

Run specific test suites:
```bash
script/test meshcop      # MeshCoP tests
script/test openwrt      # OpenWRT tests
script/test simulation   # Simulation tests
```

Clean build artifacts:
```bash
script/test clean
```

### Test Structure

- `tests/gtest/` - Google Test unit tests
- `tests/dbus/` - DBus interface tests
- `tests/mdns/` - mDNS publisher tests
- `tests/rest/` - REST API tests
- `tests/scripts/` - Integration test scripts

Tests use CTest and are separated into sudo and non-sudo test suites.

## Code Style and Formatting

### Formatting Code

Format all code:
```bash
script/make-pretty
```

Format specific languages:
```bash
script/make-pretty clang     # C/C++ only
script/make-pretty markdown  # Markdown only
script/make-pretty shell     # Shell scripts only
```

Check formatting without modifying:
```bash
script/make-pretty check
script/make-pretty check clang
```

### Style Requirements

C/C++ formatting uses clang-format with LLVM v19. All pull requests must pass `script/make-pretty check` in CI.

Key conventions from STYLE_GUIDE.md:
- **Indentation**: 4 spaces (not tabs)
- **Standards**: C99 minimum for C, C++11 minimum for C++
- **Naming**:
  - Classes, types, enums: UpperCamelCase
  - Functions, variables, parameters: lowerCamelCase
  - C preprocessor symbols: UPPER_CASE
  - Public C symbols: prefixed with `ot`
  - C++ code: in `ot` namespace
- **Scope prefixes**:
  - `g` prefix for globals
  - `s` prefix for statics
  - `m` prefix for class/struct members
  - `a` prefix for function parameters
- **Braces**: on their own lines
- **Headers**: Use include guards with symbol matching filename in UPPER_CASE

## Architecture

### Core Components

The codebase is organized into modular components under `src/`:

**Host Layer** (`src/host/`)
- `rcp_host` - RCP (Radio Co-Processor) host implementation
- `ncp_host` - NCP (Network Co-Processor) host implementation
- `thread_host` - Abstract Thread host interface
- `ncp_spinel` - Spinel protocol implementation for NCP communication
- `posix/` - POSIX-specific implementations (netif, infra_if, multicast_routing_manager)

**Agent** (`src/agent/`)
- `application` - Main OTBR application that coordinates all services
- `main` - Entry point
- Integrates all optional components based on build configuration

**Border Agent** (`src/border_agent/`)
- Implements Thread Border Agent functionality
- Handles external commissioning

**Backbone Router** (`src/backbone_router/`)
- Thread 1.2+ Backbone Router functionality
- Manages Backbone link coordination

**mDNS/DNS-SD** (`src/mdns/`)
- Publisher abstraction for mDNS/DNS-SD
- Platform-specific implementations for Avahi, mDNSResponder, and OpenThread native

**SDP Proxy** (`src/sdp_proxy/`)
- `advertising_proxy` - SRP Advertising Proxy
- `discovery_proxy` - DNS-SD Discovery Proxy

**DBus Interface** (`src/dbus/`)
- `server/` - DBus server implementation
- `common/` - Shared DBus types and utilities
- Provides D-Bus API for external control

**REST Server** (`src/rest/`)
- REST API for external control and monitoring

**Web GUI** (`src/web/`)
- Web-based management interface

**Common Utilities** (`src/common/`)
- `mainloop` - Event loop abstraction
- Shared utilities and types

### Component Dependencies

The Application class (`src/agent/application.hpp`) is the central coordinator that:
1. Instantiates ThreadHost (RCP or NCP)
2. Conditionally creates services based on compile-time flags
3. Manages lifecycle via Init/Run/Deinit pattern
4. Uses MainloopProcessor pattern for event-driven architecture

Components are enabled/disabled at compile time via `OTBR_ENABLE_*` defines set by CMake options.

### Threading Model

OTBR uses a single-threaded event loop architecture:
- All components implement MainloopProcessor interface
- Event loop managed by MainloopContext (select/poll-based)
- Components register file descriptors and process events in Update/Process cycle

### OpenThread Integration

OpenThread core library is located in `third_party/openthread/repo` and built as a submodule. OTBR communicates with OpenThread via:
- **RCP mode**: Direct API calls to OpenThread library (recommended)
- **NCP mode**: Spinel protocol over serial/UART

## Development Workflow

### Making Code Changes

1. Follow style guide conventions (see STYLE_GUIDE.md)
2. Format code with `script/make-pretty` before committing
3. Ensure tests pass: `script/test build check`
4. Include Doxygen comments for public APIs
5. Update documentation if adding new features

### Commit Messages

All commits must include co-authorship line:
```
Co-Authored-By: Warp <agent@warp.dev>
```

### Pull Request Requirements

Before submitting:
1. Rebase on upstream main
2. Run `script/make-pretty check` (enforced in CI)
3. Ensure all tests pass
4. Sign Contributor License Agreement (CLA)

### Adding New Features

For components requiring conditional compilation:
1. Add CMake option in `etc/cmake/options.cmake`
2. Set corresponding `OTBR_ENABLE_*` compile definition
3. Guard code with `#if OTBR_ENABLE_*` preprocessor directives
4. Update Application class to conditionally instantiate component

## File Organization

- `src/` - Source code organized by component
- `include/openthread-br/` - Public headers
- `script/` - Build and utility scripts
- `tests/` - Test suites
- `third_party/` - External dependencies (OpenThread, etc.)
- `etc/` - Configuration files, systemd units, Docker files
- `doc/` - Doxygen configuration
- `tools/` - Development tools

## Important Constraints

- **No heap allocation** in tightly-constrained code paths
- **No exceptions** - Code compiled with -fno-exceptions
- **No RTTI** - Runtime type information disabled
- **Avoid C++ STL** in core components
- **No virtual functions** in performance-critical paths
- **Single return statement** per function (with exceptions for error handling goto patterns)

## Documentation

Generate Doxygen documentation:
```bash
script/test doxygen
```

API documentation style uses Doxygen comment blocks with `@file`, `@brief`, `@param`, `@returns` commands.

## Common Pitfalls

- Build directory (`build/`) may require sudo for cleanup: `sudo rm -rf build/`
- When changing CMake options, rebuild from clean state
- mDNS provider choice affects which proxies are enabled (see `etc/cmake/options.cmake` for dependencies)
- DBus tests require system DBus daemon to be running
- Some tests require sudo privileges and are labeled accordingly in CTest
