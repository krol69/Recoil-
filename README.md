



## Overview

This is a sophisticated anti-recoil system designed for Rainbow Six Siege that utilizes specialized hardware components (DMA card and KMBox) to provide undetectable recoil compensation. The system operates externally to the game process, making it significantly harder to detect compared to traditional software-based solutions.

## ⚠️ Disclaimer

This project is for **educational and research purposes only**. Using this software in online multiplayer games violates the Terms of Service of most games and can result in permanent bans. The authors are not responsible for any consequences resulting from the use of this software.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Hardware Requirements](#hardware-requirements)
- [How It Works](#how-it-works)
- [Component Breakdown](#component-breakdown)
- [Build Instructions](#build-instructions)
- [Configuration & Usage](#configuration--usage)
- [Technical Details](#technical-details)
- [Project Structure](#project-structure)

## Architecture Overview

The system is built on a two-PC setup architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                         GAMING PC                            │
│  ┌────────────────────┐                                     │
│  │ Rainbow Six Siege  │                                     │
│  │   (Game Process)   │                                     │
│  └────────────────────┘                                     │
│           ↑                                                  │
│           │ Read Memory (DMA)                               │
│           │                                                  │
│  ┌────────────────────┐                                     │
│  │  DMA Card (PCIe)   │◄────────────────────┐              │
│  │  (Memory Reader)   │                      │              │
│  └────────────────────┘                      │              │
└──────────────────────────────────────────────┼──────────────┘
                                                │
                                                │ PCIe Cable
                                                │
┌───────────────────────────────────────────────┼──────────────┐
│                      CONTROL PC               │              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            Myself.exe (This Application)            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │    │
│  │  │ DMA Handler  │  │   Keyboard   │  │  KMBox    │ │    │
│  │  │   (Memory)   │  │   Manager    │  │  Control  │ │    │
│  │  └──────────────┘  └──────────────┘  └───────────┘ │    │
│  │  ┌─────────────────────────────────────────────────┐│    │
│  │  │       Recoil Compensation Engine                ││    │
│  │  └─────────────────────────────────────────────────┘│    │
│  │  ┌─────────────────────────────────────────────────┐│    │
│  │  │         ImGui-based User Interface              ││    │
│  │  └─────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────┘    │
│           │                                   ↓              │
│           │ Serial (COM Port)        Mouse Movement         │
│  ┌────────────────────┐               ┌──────────────┐      │
│  │   KMBox B Pro      │───────────────►     Mouse    │      │
│  │ (Hardware Emulator)│  USB HID      │   (to Gaming │      │
│  └────────────────────┘               │      PC)     │      │
└───────────────────────────────────────┴──────────────────────┘
```

## Hardware Requirements

### Essential Components

1. **DMA Card (FPGA-based PCIe DMA Device)**
   - Purpose: Direct Memory Access to read game memory from the gaming PC
   - Connects via PCIe cable between gaming PC and control PC
   - Uses `vmm.dll`, `leechcore.dll`, and `FTD3XX.dll` for communication
   - Operates at kernel level with FPGA algorithm 0

2. **KMBox B Pro (Hardware Input Emulator)**
   - Purpose: Inject mouse movements that appear as genuine USB HID input
   - Connects via USB-SERIAL CH340 interface (COM port)
   - Baud rate: 115200
   - Sends hardware-level mouse commands that are indistinguishable from real mouse input

3. **Two PCs**
   - Gaming PC: Runs Rainbow Six Siege
   - Control PC: Runs this software (Myself.exe)

### Software Dependencies

- Windows Operating System (Gaming PC and Control PC)
- Visual Studio 2019 or later (for building)
- DirectX 11 SDK
- setupapi.lib, d3d11.lib, d3dx11.lib, Comctl32.lib

## How It Works

### System Operation Flow

1. **Initialization Phase** (`entrypoint.cxx`)
   ```
   ┌─────────────────────────────────────────────────────────────┐
   │  1. Connect to KMBox (find USB-SERIAL CH340 port)           │
   │  2. Load DMA modules (vmm.dll, leechcore.dll, FTD3XX.dll)   │
   │  3. Initialize DMA handle (connect to gaming PC memory)     │
   │  4. Setup keyboard state reader (via DMA)                   │
   │  5. Start recoil compensation thread                        │
   │  6. Initialize DirectX 11 overlay GUI                       │
   └─────────────────────────────────────────────────────────────┘
   ```

2. **Keyboard Monitoring** (`communication.hxx` - `keyboard_manager`)
   - Reads keyboard state directly from Windows kernel memory via DMA
   - Monitors the `gafAsyncKeyState` kernel structure
   - Detects keypresses without any hooks or API calls
   - Tracks both left (VK_LBUTTON) and right (VK_RBUTTON) mouse buttons
   - Windows version detection (supports builds > 22000 and legacy versions)

3. **Recoil Compensation Loop** (`recoil.cxx` & `recoil.hxx`)
   ```cpp
   Main Loop (150 Hz):
   ├─ Check if enabled (toggle with F1 by default)
   ├─ Read keyboard state via DMA
   ├─ If both mouse buttons pressed (L+R click):
   │  ├─ Calculate recoil offset (horizontal, vertical)
   │  └─ Send move command to KMBox
   └─ Rate limit to 150 ticks per second
   ```

4. **Mouse Movement Injection** (`bpro.hxx` - `communication`)
   - Formats command: `km.move(x, y, beizer)\r\n`
   - Sends via serial to KMBox
   - KMBox generates hardware USB HID packets
   - Gaming PC receives as genuine mouse movement

5. **GUI Overlay** (`menu.hxx` & `initialization.hxx`)
   - DirectX 11 transparent overlay window
   - Always on top, click-through when not hovering
   - Real-time adjustment of recoil parameters
   - Visual feedback and status display

### Detection Avoidance

The system is designed to be undetectable through multiple layers:

1. **No Game Process Interaction**
   - Never opens handles to the game process
   - Never injects code or DLLs
   - Never uses Windows API to interact with the game

2. **Hardware-Level Operations**
   - DMA reads memory directly via PCIe (invisible to game)
   - KMBox generates authentic USB HID packets
   - No software hooks or kernel drivers on gaming PC

3. **External Processing**
   - All logic runs on separate PC
   - Game anti-cheat cannot scan control PC memory
   - Network traffic is minimal (only DMA and USB)

## Component Breakdown

### 1. DMA Handler (`device/dma/communication.hxx`)

**Purpose**: Interface with DMA hardware to read gaming PC memory

**Key Classes**:
- `dma_handler`: Manages DMA device initialization
  - Loads required DLLs (vmm.dll, leechcore.dll, FTD3XX.dll)
  - Initializes VMM (Virtual Machine Monitor) handle
  - Uses FPGA algorithm 0 for memory access

- `keyboard_manager`: Reads keyboard state from kernel memory
  - Locates `gafAsyncKeyState` in win32kbase.sys or win32ksgd.sys
  - Version-specific offsets for different Windows builds
  - Direct kernel memory reading with no user-mode API calls

**Technical Details**:
```cpp
// DMA initialization
LPCSTR args[] = {"-device", "fpga://algo=0", "-norefresh"};
global_handle = VMMDLL_Initialize(3, args);

// Read kernel memory
VMMDLL_MemReadEx(handle, PID | VMMDLL_PID_PROCESS_WITH_KERNELMEMORY, 
                 address, buffer, size, NULL, VMMDLL_FLAG_NOCACHE);
```

### 2. KMBox Communication (`device/kmbox/bpro.hxx`)

**Purpose**: Control hardware input emulator for mouse movements

**Key Classes**:
- `connection`: Manages serial port connection
  - Auto-detects "USB-SERIAL CH340" device
  - Opens COM port with 115200 baud rate
  - Handles port enumeration and configuration

- `communication`: Sends movement commands
  - Format: `km.move(x, y, bezier_smoothing)`
  - Optimized for minimal latency
  - Thread-safe command buffering

**Technical Details**:
```cpp
// KMBox command format
char cmd[24];
snprintf(cmd, 24, "km.move(%d, %d, %d)\r\n", x, y, beizer);
WriteFile(bpro_handle, cmd, strlen(cmd), &bytes_written, 0);
```

### 3. Recoil Engine (`recoil.hxx` & `recoil.cxx`)

**Purpose**: Core anti-recoil logic and timing

**Settings**:
- `enabled`: Master on/off switch
- `toggle_key`: Hotkey to toggle system (default: F1)
- `position`: Recoil compensation vector (horizontal, vertical)
  - Vertical: 0-10 pixels down (counteracts upward recoil)
  - Horizontal: -10 to +10 pixels (compensates spray pattern)
- `key_pressed`: Trigger condition (both mouse buttons)

**Timing System** (`tools/timing.hxx`):
- `limiter`: Precise 150 Hz rate limiting
- `performance_timer`: High-precision timing using QueryPerformanceCounter
- Ensures consistent recoil compensation regardless of system load

**Operation**:
```cpp
// Main loop at 150 Hz
while (true) {
    limiter.start();
    
    if (enabled && key_pressed) {
        // Apply recoil compensation
        communication_obj.move(position.first,   // horizontal
                              position.second,   // vertical  
                              1);                // bezier smoothing
    }
    
    limiter.end();  // Sleep to maintain 150 Hz
}
```

### 4. User Interface (`frontend/menu.hxx`)

**Purpose**: Real-time configuration and visual feedback

**Features**:
- **Transparent Overlay**: DirectX 11 window with alpha blending
- **Click-Through Mode**: Automatically becomes transparent when not hovered
- **Real-Time Adjustments**:
  - Enable/disable checkbox
  - Vertical recoil slider (0-10px)
  - Horizontal recoil slider (-10 to +10px)
  - Keybind configuration button
- **Visual Design**: Custom ImGui theme with outlined text
- **Always On Top**: WS_EX_TOPMOST window flag

**UI Components**:
```cpp
// Main window properties
const ImVec2 widget_child_size{650, 260};
Flags: ImGuiWindowFlags_NoDecoration
Style: WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOPMOST
```

## Build Instructions

### Prerequisites

1. **Visual Studio 2019 or later** with C++ desktop development workload
2. **Windows SDK** (included with Visual Studio)
3. **DirectX 11 SDK**
4. **DMA device drivers and libraries**:
   - vmm.dll
   - leechcore.dll  
   - FTD3XX.dll

### Building from Source

1. Clone the repository:
   ```bash
   git clone https://github.com/krol69/Recoil-.git
   cd Recoil-
   ```

2. Open solution:
   ```
   Mysëlf.sln
   ```

3. Configure project:
   - Target: Windows x64
   - Configuration: Release (recommended) or Debug

4. Build solution:
   - Press `Ctrl+Shift+B` or select `Build > Build Solution`

5. Output location:
   ```
   ./output/Mysëlf.exe
   ```

### Required DLLs

Ensure these DLLs are in the same directory as `Mysëlf.exe`:
- `vmm.dll` - Virtual Machine Monitor library
- `leechcore.dll` - DMA core library
- `FTD3XX.dll` - FTDI D3XX driver

## Configuration & Usage

### Initial Setup

1. **Hardware Setup**:
   ```
   Gaming PC ←─(PCIe Cable)─→ DMA Card ←→ Control PC
   Gaming PC ←─(USB Cable)──→ KMBox ←─(Serial)─→ Control PC
   ```

2. **Launch Application**:
   - Run `Mysëlf.exe` on control PC
   - Application will auto-detect KMBox and DMA device
   - Error messages appear if hardware not found

3. **Verify Connections**:
   - KMBox: Green status if "USB-SERIAL CH340" port detected
   - DMA: Green status if gaming PC memory accessible
   - Keyboard: Green status if kernel state reader initialized

### Usage

1. **Start Rainbow Six Siege** on gaming PC

2. **Configure Recoil Settings**:
   - Adjust **Vertical** slider (recoil compensation strength)
     - Higher values = more downward movement
     - Typical range: 3-7 for most weapons
   - Adjust **Horizontal** slider (spray pattern compensation)
     - 0 = center, negative = left, positive = right
   - Set **Toggle Keybind**:
     - Click "toggle keybind: F1" button
     - Press desired key (F1-F12 recommended)

3. **In-Game Activation**:
   - Press toggle key (default F1) to enable/disable
   - Hold both mouse buttons (L+R click) to apply compensation
   - System runs at 150 Hz for smooth, consistent recoil control

4. **Fine-Tuning**:
   - Test in training grounds
   - Adjust vertical/horizontal values per weapon
   - Different weapons have different recoil patterns

### Controls

| Control | Action |
|---------|--------|
| F1 (default) | Toggle recoil compensation on/off |
| L+R Mouse | Activate recoil compensation (both buttons must be pressed) |
| GUI Sliders | Real-time adjustment of compensation values |
| X Button | Close application |

### Recommended Settings by Weapon Type

| Weapon Type | Vertical | Horizontal | Notes |
|------------|----------|------------|-------|
| Low Recoil AR | 2-4 | 0 | Minimal compensation needed |
| High Recoil AR | 5-8 | -1 to 1 | Strong vertical pull |
| SMG | 3-5 | -2 to 2 | Fast fire rate, varied pattern |
| DMR | 1-3 | 0 | Semi-auto, low compensation |
| LMG | 6-10 | -3 to 3 | Strongest compensation needed |

## Technical Details

### Memory Reading Architecture

```cpp
// Kernel-level memory reading
VMMDLL_MemReadEx(
    handle,                                    // VMM handle
    pid | VMMDLL_PID_PROCESS_WITH_KERNELMEMORY, // PID with kernel flag
    gaf_async_key_state,                       // Kernel address
    buffer,                                    // Output buffer
    64,                                        // Size
    NULL,                                      // Optional flags
    VMMDLL_FLAG_NOCACHE                        // Bypass cache
);
```

### Keyboard State Detection

The system reads keyboard state from Windows kernel structures:

**Windows 11 (Build > 22000)**:
```
win32ksgd.sys + 0x3110 → g_session_global_slots
→ [deref] → [deref] → user_session_state
→ +0x3690 → gafAsyncKeyState
```

**Windows 10 and earlier**:
```
win32kbase.sys → Export Address Table (EAT)
→ Find "gafAsyncKeyState" export
→ Direct address to state array
```

### Rate Limiting System

```cpp
class limiter {
    // Target: 150 Hz (6.67ms per frame)
    std::chrono::nanoseconds target_frame_duration{1000000000 / 150};
    
    void start() {
        start_time = high_resolution_clock::now();
    }
    
    void end() {
        auto end_time = start_time + target_frame_duration;
        while (high_resolution_clock::now() < end_time) {
            // Busy wait for precision
        }
    }
};
```

### Multi-Threading Architecture

```cpp
Thread 1 (Main):        Initialize GUI → Render Loop
Thread 2 (Recoil):      Recoil Compensation Loop (150 Hz)
Thread 3 (Keyboard):    Keyboard State Monitor (1μs updates)
```

### Performance Characteristics

- **Latency**: < 7ms (150 Hz update rate)
- **Memory Reads**: Direct kernel memory access via DMA
- **CPU Usage**: < 5% on modern CPUs
- **Network**: No network traffic (local hardware only)

## Project Structure

```
Recoil-/
├── README.md                          # This file
├── Mysëlf.sln                         # Visual Studio solution
├── output/                            # Build output directory
│   ├── Mysëlf.exe                    # Compiled executable
│   ├── vmm.dll                       # DMA virtual memory manager
│   ├── leechcore.dll                 # DMA core library
│   └── FTD3XX.dll                    # FTDI driver
└── Mysëlf/                           # Main source directory
    ├── entrypoint.cxx                # Application entry point
    ├── recoil.hxx                    # Recoil system header
    ├── recoil.cxx                    # Recoil implementation
    ├── device/                       # Hardware interface
    │   ├── dma/                      # DMA card interface
    │   │   ├── communication.hxx     # DMA handlers & keyboard manager
    │   │   ├── pch.hxx              # Precompiled headers
    │   │   └── include/
    │   │       └── headers/
    │   │           └── vmmdll.hxx   # VMM API definitions
    │   └── kmbox/                    # KMBox interface
    │       └── bpro.hxx             # Serial communication & commands
    ├── frontend/                     # User interface
    │   ├── menu.hxx                 # Main GUI implementation
    │   ├── includes.hxx             # Common includes
    │   └── start/                   # GUI framework
    │       ├── initialization.hxx   # DirectX initialization
    │       ├── bytes.hxx           # Embedded resources
    │       └── imgui/              # ImGui library
    │           ├── imgui.h
    │           ├── imgui.cpp
    │           ├── imgui_draw.cpp
    │           ├── imgui_widgets.cpp
    │           ├── imgui_tables.cpp
    │           └── backend/        # DirectX 11 backend
    │               ├── ImGui_impl_dx11.h
    │               ├── ImGui_impl_dx11.cpp
    │               ├── ImGui_impl_win32.h
    │               └── ImGui_impl_win32.cpp
    └── tools/                       # Utility classes
        └── timing.hxx              # Timing & rate limiting

```

## Security & Privacy Considerations

### What This Software Does

- ✓ Reads game memory via DMA (external hardware)
- ✓ Monitors keyboard input via kernel memory reading
- ✓ Sends mouse commands via hardware emulator
- ✓ Runs entirely on separate PC from game

### What This Software Does NOT Do

- ✗ Inject code into game process
- ✗ Modify game files or memory
- ✗ Use Windows API hooks
- ✗ Install kernel drivers on gaming PC
- ✗ Communicate over network
- ✗ Collect or transmit user data

### Detection Risks

While designed to be undetectable, users should be aware:

1. **Hardware Signatures**: Some anti-cheats can detect DMA devices
2. **Behavioral Analysis**: Consistent perfect recoil may be flagged
3. **Kernel Protection**: Modern anti-cheats protect kernel memory access
4. **HWID Bans**: Hardware bans are permanent and bypass software changes

### Recommendations

- Use at your own risk
- Understand your game's Terms of Service
- Consider ethical implications
- Test in offline/private modes first
- Don't rely on "undetectable" claims

## Troubleshooting

### Common Issues

**Error: "kmbox port not found"**
- Ensure KMBox is connected via USB
- Install CH340 serial driver
- Check Device Manager for COM port

**Error: "failed to find modules"**
- Verify vmm.dll, leechcore.dll, FTD3XX.dll are in output/ directory
- Check DLL versions match (all from same DMA vendor)

**Error: "failed to connect to dma handle"**
- Verify DMA card is properly connected
- Check FPGA firmware is loaded
- Ensure gaming PC is powered on

**Error: "failed to connect to keyboard manager"**
- Gaming PC may be running unsupported Windows version
- Try running gaming PC in Safe Mode
- Check kernel memory protection settings

**Recoil compensation not working**
- Verify both mouse buttons are pressed (L+R)
- Check toggle key is enabled (default F1)
- Adjust sensitivity values higher
- Ensure Rainbow Six Siege is running

### Debug Mode

Enable debug logging by modifying `entrypoint.cxx`:
```cpp
// Add before initialization
#define DEBUG_MODE
```

Logs will show:
- Connection status for each component
- Memory read success/failure
- KMBox command transmission
- Frame timing information

## Legal Notice

This software is provided "as is" without warranty of any kind. The developers:

- Do not condone cheating in online games
- Are not responsible for account bans or hardware damage
- Provide this for educational/research purposes only
- Do not provide support for malicious use

By using this software, you acknowledge:
- You understand the risks involved
- You accept full responsibility for consequences
- You will not hold developers liable for any damages
- You will comply with all applicable laws and terms of service

## Credits & Acknowledgments

- **ImGui**: Dear ImGui library for GUI rendering
- **VMM/LeechCore**: Memory reading framework
- **KMBox**: Hardware input emulation
- **DirectX 11**: Microsoft graphics API

## Contributing

This is an educational project. Contributions that improve:
- Code quality and documentation
- Safety and error handling
- Performance and optimization

...are welcome. Contributions that:
- Bypass new anti-cheat measures
- Enable malicious use
- Violate any laws or ToS

...will not be accepted.

## Version History

- **Current**: Initial public release
  - DMA-based memory reading
  - KMBox hardware input
  - Windows 10/11 support
  - ImGui overlay interface

---

**Remember**: Use responsibly and at your own risk. This tool is for educational purposes to understand how anti-recoil systems work at a low level.
