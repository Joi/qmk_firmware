# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QMK (Quantum Mechanical Keyboard) is an open-source keyboard firmware based on TMK. It powers thousands of keyboards with extensive customization capabilities through C firmware code and Python CLI tools.

## Essential Commands

### Building & Flashing
```bash
# Build firmware (make format)
make <keyboard>:<keymap>  # e.g., make planck/rev6:default
make <keyboard>:<keymap>:flash  # Build and flash

# Build firmware (QMK CLI - recommended)
qmk compile -kb <keyboard> -km <keymap>
qmk flash -kb <keyboard> -km <keymap>

# Clean build artifacts
make clean
qmk clean
```

### Testing & Validation
```bash
# Run tests
qmk pytest  # Python tests
qmk test-c  # C unit tests
make test:<testname>  # Specific test

# Linting & Formatting
qmk format-c  # Format C code
qmk format-python  # Format Python code
qmk lint -kb <keyboard>  # Lint keyboard files
```

### Development Tools
```bash
qmk doctor  # Check environment setup
qmk info -kb <keyboard>  # Show keyboard info
qmk list-keyboards  # List all keyboards
qmk new-keyboard  # Create new keyboard
qmk new-keymap  # Create new keymap
```

## Architecture & Structure

### Directory Layout
```
qmk_firmware/
├── keyboards/         # All keyboard definitions
│   └── <vendor>/
│       └── <keyboard>/
│           ├── info.json      # Data-driven config
│           ├── rules.mk       # Build rules
│           ├── config.h       # C configuration
│           └── keymaps/       # User keymaps
├── quantum/          # Core QMK code
│   ├── process_keycode/  # Keycode processors
│   ├── audio/        # Audio subsystem
│   ├── rgb_matrix/   # RGB effects
│   └── split_common/ # Split keyboard code
├── drivers/          # Hardware drivers
├── platforms/        # Platform-specific code (AVR, ARM, RP2040)
├── lib/              # External libraries
├── tests/            # Unit tests
└── util/             # Build utilities
```

### Core Architecture

QMK uses an **event-driven architecture** with a main loop that:
1. Scans the keyboard matrix for changes
2. Debounces switch states
3. Processes detected key events through feature modules
4. Sends USB HID reports

**Key Processing Pipeline:**
1. `matrix_scan()` - Read physical switches
2. `process_record()` - Process keycodes through feature chain
3. `host_keyboard_send()` - Send USB reports

**Feature Module Chain** (order matters):
- Each feature (e.g., tap dance, combos, layers) hooks into `process_record_*()`
- Features process keycodes in a specific order defined in `quantum/quantum.c`
- Early return from any processor stops the chain

### Configuration Hierarchy

1. **keyboard level** - `keyboards/<keyboard>/info.json` + `config.h`
2. **revision level** - `keyboards/<keyboard>/<rev>/info.json` + `config.h`  
3. **keymap level** - `keyboards/<keyboard>/keymaps/<keymap>/config.h`
4. **userspace** - `users/<username>/` (shared code across keymaps)

Data-driven configuration (`info.json`) is preferred over C macros when possible.

### Memory Constraints

**Critical for AVR controllers:**
- Flash: ~32KB (28KB usable after bootloader)
- RAM: 2KB
- EEPROM: 1KB

**Optimization strategies:**
- Use `PROGMEM` for constant data
- Minimize dynamic allocation
- Disable unused features in `rules.mk`
- Use `LTO_ENABLE = yes` for link-time optimization

### Split Keyboard Architecture

Split keyboards use a master/slave model:
- **Master side**: Runs full QMK, handles USB
- **Slave side**: Scans matrix, sends to master
- **Transport**: Serial (USART/SoftSerial) or I2C
- **State sync**: Automatic for LEDs, layers, mods

## Common Development Patterns

### Adding a New Feature
1. Check if feature already exists in `quantum/`
2. Add feature flag to `rules.mk`: `FEATURE_NAME_ENABLE = yes`
3. Implement in `process_record_user()` for keymap-specific
4. Or add to `quantum/process_keycode/` for global feature

### Custom Keycodes
```c
enum custom_keycodes {
    MY_CUSTOM = SAFE_RANGE,  // Start after QMK's keycodes
    // Add more here
};

bool process_record_user(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case MY_CUSTOM:
            if (record->event.pressed) {
                // On key press
            }
            return false;  // Skip further processing
    }
    return true;  // Continue processing
}
```

### Debugging
- Add `CONSOLE_ENABLE = yes` to `rules.mk`
- Use `dprintf()` for debug output
- View with `qmk console` or HID listen tool
- For matrix issues, enable `DEBUG_MATRIX_SCAN_RATE`

## Critical Considerations

1. **Matrix scanning consumes ~99% of CPU time** - Keep processing fast
2. **Avoid blocking operations** - No `delay()` in main loop
3. **Test on hardware** - Simulator doesn't catch all issues
4. **Check multiple platforms** - Code must work on AVR, ARM, RP2040
5. **Respect community standards** - Follow existing patterns and conventions