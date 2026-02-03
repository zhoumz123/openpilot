# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

openpilot is an open source driver assistance system that provides Adaptive Cruise Control (ACC) and Automated Lane Centering (ALC) for 300+ supported cars. It's developed by comma.ai as an operating system for robotics.

**Key properties:**
- **Language**: Python (primarily), with some C/C++ components
- **Build System**: SCons (Python-based build tool) + pyproject.toml
- **Architecture**: Distributed microservices with message passing
- **Hardware**: comma 3X device, TI CI platform, and PC/Webcam setups
- **Python**: Requires >= 3.12.3, < 3.13

## Build and Development Commands

### Initial Setup
```bash
# Setup development environment (managed)
./tools/op.sh setup

# Activate Python environment
source .venv/bin/activate

# Build the project
scons -u -j$(nproc)  # -u to update, -j for parallel jobs
```

### Build Variants
```bash
scons --minimal     # Minimal release build
scons --asan        # With AddressSanitizer
scons --ubsan       # With UndefinedBehaviorSanitizer
```

### Testing
```bash
# Run all tests
pytest --continue-on-collection-errors --durations=0 -n logical

# Run specific test categories
pytest -m "not slow"  # Skip slow tests
pytest -m "tici"      # Only run on TI CI hardware

# Run tests from specific directories
pytest common/ selfdrive/ system/

# Run single test file with verbose output
pytest -xvs path/to/test_file.py
```

### Linting
```bash
# Run all linters
./scripts/lint/lint.sh

# Run specific linters
./scripts/lint/lint.sh ruff ty codespell

# Fast lint (skip slow tests)
./scripts/lint/lint.sh --fast
```

## High-Level Architecture

### Microservices Architecture

openpilot uses a multi-process architecture where services communicate via ZeroMQ-based message passing (defined in `cereal/`).

**Core services managed by `system/manager/`:**
- **System Manager**: Central process manager, device registration, configuration
- **Camerad**: Camera capture and processing
- **Loggerd**: Data recording and compression
- **Sensord**: Sensor data collection
- **Athenad**: Cloud communication
- **Timed**: Time synchronization

**Self-drive stack (`selfdrive/`):**
- **Modeld**: Neural network inference (ONNX models)
- **Controlsd**: Control algorithms for steering/throttle/brake
- **Locationd**: GPS/location processing
- **Monitoringd**: Driver monitoring system
- **UI**: User interface (raylib-based)
- **Selfdrived**: State machine and safety logic

### Process Types (from `system/manager/process_config.py`)
- `DaemonProcess`: Background services
- `PythonProcess`: Python-based services
- `NativeProcess`: Compiled C/C++ services

Processes have conditional startup based on state (onroad/offroad, car type), and automatic restart on crashes (except critical processes).

### Communication Patterns

1. **Message Passing**: ZeroMQ via `cereal.messaging` (publish-subscribe model)
2. **Shared Memory**: For high-performance data (camera frames, model outputs)
3. **Cap'n Proto**: Serialization format for IPC and log files

## Key Directories

- `selfdrive/`: Core driving logic, controls, modeld, UI
- `system/`: System services (camerad, loggerd, manager)
- `common/`: Shared libraries (params, swaglog, hardware abstraction)
- `cereal/`: Message definitions (Cap'n Proto schemas) and messaging
- `msgq/`: Message queue implementation (git submodule)
- `opendbc/`: Car interface definitions (git submodule)
- `panda/`: Safety hardware interface (git submodule)
- `tools/`: Development tools and utilities
- `third_party/`: External dependencies

## Important Patterns

### Parameter System (`common/params/`)
- Persistent configuration via `Params` class
- Different parameter types with lifecycle flags
- Stored in `/data/params` on device
- Use for all persistent configuration

### Hardware Abstraction (`common/hardware/`)
- Platform detection (TI CI vs PC)
- Conditional compilation for different architectures
- Use abstraction layers for hardware-specific code

### Logging (`common/swaglog/`)
- Structured logging framework
- Multiple output targets (console, file, Sentry)
- Context-aware logging with device metadata

### Message Definitions (`cereal/`)
- All messages defined in Cap'n Proto schemas
- Type-safe message passing
- Versioned log format - never modify existing fields

## Git Submodules

Key submodules that must be initialized:
```bash
git submodule update --init --recursive
```

- `msgq/`: Message queue implementation
- `opendbc/`: Car interface definitions
- `panda/`: Safety hardware interface
- `rednose/`: Profiling and debugging
- `tinygrad/`: Neural network framework
- `teleoprtc/`: Teleoperation

## Contributing Guidelines

From `docs/CONTRIBUTING.md`:

**Priority order**: safety, stability, quality, features

**PR guidelines:**
- Keep PRs small and focused (< 500 lines preferred)
- Submit against `master` branch
- Include verification and benchmarks for changes
- Focus on safety-critical components

**Branches:**
- `master`: Development branch
- `release-mici/release-tizi`: Production releases
- `nightly`: Bleeding-edge development
- Car-specific branches for different hardware

## Safety Requirements

Critical safety constraints:
- **Driver monitoring must not be disabled**
- **Excessive actuation checks must be preserved**
- **Safety test suite must pass for any forks**
- **ISO26262 compliance** for safety-critical code

## Common Issues

### Build Issues
- Ensure Python >= 3.12.3, < 3.13
- Run `scons -u` to update build after pulling changes
- Check submodules are initialized

### Test Failures
- Some tests require specific hardware (marked with `tici` marker)
- Use `-m "not slow"` to skip long-running tests
- Some tests may require Docker environment

### Platform-Specific Code
- Check for `common/hardware/` utilities for platform detection
- Use conditional imports for platform-specific dependencies
- Test on both TI CI hardware and PC when possible
