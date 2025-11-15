# FiveM Porting Guide for BEKernelDriver

## Table of Contents
1. [Understanding FiveM Architecture](#understanding-fivem-architecture)
2. [Key Differences from Standard Games](#key-differences-from-standard-games)
3. [Required Modifications](#required-modifications)
4. [Implementation Steps](#implementation-steps)
5. [FiveM-Specific Challenges](#fivem-specific-challenges)
6. [Testing and Validation](#testing-and-validation)
7. [Detection Vectors](#detection-vectors)
8. [Best Practices](#best-practices)

---

## Understanding FiveM Architecture

### What is FiveM?

FiveM is a multiplayer modification framework for **Grand Theft Auto V (GTA V)** that allows custom multiplayer servers with custom game modes, scripts, and modifications. It's completely separate from GTA Online.

### FiveM Technical Architecture

```
FiveM Client
├── FiveM.exe (Launcher)
├── CitizenFX.ini (Configuration)
├── FiveM Application Data
│   ├── Cache
│   ├── Plugins
│   └── CitizenGame.dll (Core game wrapper)
└── GTA V Game Files (Rockstar)
    ├── GTA5.exe (Main game executable)
    ├── Game DLLs
    └── Rage Engine
```

### Key Components

1. **FiveM Client** (`FiveM.exe`)
   - Launcher and updater
   - Manages connection to servers
   - Handles authentication

2. **CitizenFX Core** (`CitizenGame.dll`, `CitizenFX Core.dll`)
   - Game framework wrapper
   - Hooks game functions
   - Manages native calls
   - Handles scripting (Lua/JavaScript)

3. **GTA V Process** (`GTA5.exe`)
   - Actual game executable
   - Rage Engine
   - Game logic and rendering

4. **Anti-Cheat**
   - **Server-Side**: Most FiveM servers use custom AC
   - **Client-Side**: Integrity checks, screenshot requests, resource scanning
   - **Rage Engine**: Built-in protection mechanisms

### Memory Layout

```
GTA5.exe Process Space:
├── 0x140000000 - GTA5.exe (main module)
├── 0x7FF... - CitizenGame.dll
├── 0x7FF... - Various game DLLs
├── 0x... - FiveM injected modules
└── Dynamic allocations
```

---

## Key Differences from Standard Games

### 1. Multi-Process Architecture

**Standard Game (e.g., Tarkov)**:
```
Single Process: EscapeFromTarkov.exe
```

**FiveM**:
```
Parent: FiveM.exe (launcher)
  └── Child: GTA5.exe (actual game)
```

**Impact on Driver**:
- Must target `GTA5.exe` not `FiveM.exe`
- Process name detection: `"GTA5.exe"` or `"GTA5.exe"`
- Parent-child relationship may complicate injection

### 2. Server-Side Anti-Cheat

**Standard Game**: Client-side AC (BattlEye DLL)
**FiveM**: Server authoritative + client verification

**Server-Side Checks**:
- Position validation
- Speed checks
- Weapon damage verification
- Health/armor bounds
- Event rate limiting
- Screenshot requests (can capture ESP/overlays)

**Impact**:
- Memory manipulation alone insufficient
- Must respect server validation
- Network packet inspection may detect anomalies

### 3. Dynamic Module Loading

**Standard Game**: Static module set
**FiveM**: Dynamically loads/unloads resources

**Impact**:
- Module base addresses may change
- Need to re-enumerate modules periodically
- Code caves may be invalidated

### 4. Scripting Environment

FiveM servers run Lua/JavaScript scripts that can:
- Request client screenshots
- Scan loaded modules
- Monitor performance anomalies
- Detect debug flags

**Impact**:
- Process hiding more critical
- Must prevent module enumeration
- Debug flags must be cleared

---

## Required Modifications

### 1. Process Target Changes

**File**: `UM/entrypoint.cpp`

**Before**:
```cpp
const auto pid = request->get_process_pid(L"escapefromtarkov.exe");
```

**After**:
```cpp
// FiveM launches GTA5.exe
const auto pid = request->get_process_pid(L"GTA5.exe");

// Validate it's actually FiveM (not GTA Online)
// Method 1: Check for CitizenGame.dll
auto citizen_base = request->get_image_base("CitizenGame.dll");
if (citizen_base == 0) {
    std::cout << "Not a FiveM process (CitizenGame.dll not found)\n";
    exit(1);
}
```

**File**: `KM/processhyde/Hide.cpp` (if using process hiding)

**Before**:
```cpp
const char target[] = "pcom5.exe";
```

**After**:
```cpp
const char target[] = "your_fivem_tool.exe";  // Your UM application name
```

### 2. Base Address Resolution

FiveM/GTA V uses ASLR heavily. Base addresses change on every launch.

**Enhanced Base Resolution**:
```cpp
// In UM code
bool initialize_game_base() {
    // Get main GTA5.exe base
    game_base = request->get_image_base(nullptr);
    if (!game_base) {
        std::cout << "Failed to get GTA5.exe base\n";
        return false;
    }
    
    std::cout << "GTA5.exe base: 0x" << std::hex << game_base << std::endl;
    
    // Verify MZ header
    auto mz = request->read<uint16_t>(game_base);
    if (mz != 0x5A4D) {  // "MZ"
        std::cout << "Invalid PE header\n";
        return false;
    }
    
    // Get CitizenGame.dll for FiveM-specific offsets
    citizen_base = request->get_image_base("CitizenGame.dll");
    std::cout << "CitizenGame.dll base: 0x" << std::hex << citizen_base << std::endl;
    
    return true;
}
```

### 3. Pattern Scanning

GTA V updates frequently. Hardcoded offsets break immediately.

**Implement Signature Scanning**:

Add to `UM` project:
```cpp
// signature_scanner.h
class SignatureScanner {
private:
    driver::communicate_t* driver;
    uintptr_t base;
    size_t size;

public:
    SignatureScanner(driver::communicate_t* drv, uintptr_t module_base, size_t module_size) 
        : driver(drv), base(module_base), size(module_size) {}
    
    uintptr_t find_pattern(const char* pattern, const char* mask) {
        size_t pattern_len = strlen(mask);
        uint8_t* buffer = new uint8_t[size];
        
        // Read entire module into buffer
        if (!driver->read_virtual(base, buffer, size)) {
            delete[] buffer;
            return 0;
        }
        
        // Search for pattern
        for (size_t i = 0; i < size - pattern_len; i++) {
            bool found = true;
            for (size_t j = 0; j < pattern_len; j++) {
                if (mask[j] == 'x' && buffer[i + j] != pattern[j]) {
                    found = false;
                    break;
                }
            }
            if (found) {
                delete[] buffer;
                return base + i;
            }
        }
        
        delete[] buffer;
        return 0;
    }
    
    // Find pattern with IDA-style signature
    uintptr_t find_pattern(const std::string& ida_pattern) {
        // Convert IDA pattern "48 8B 05 ? ? ? ?" to pattern and mask
        std::vector<uint8_t> pattern_bytes;
        std::string mask;
        
        std::istringstream stream(ida_pattern);
        std::string byte_str;
        
        while (stream >> byte_str) {
            if (byte_str == "?") {
                pattern_bytes.push_back(0);
                mask += '?';
            } else {
                pattern_bytes.push_back(std::stoi(byte_str, nullptr, 16));
                mask += 'x';
            }
        }
        
        return find_pattern(reinterpret_cast<const char*>(pattern_bytes.data()), mask.c_str());
    }
};
```

**Usage**:
```cpp
// Find player pointer
SignatureScanner scanner(request, game_base, module_size);
uintptr_t player_ptr_sig = scanner.find_pattern("48 8B 05 ? ? ? ? 45 ? ? ? ? 48 8B 48");
if (player_ptr_sig) {
    // Resolve RIP-relative address
    int32_t rip_offset = request->read<int32_t>(player_ptr_sig + 3);
    uintptr_t player_ptr_addr = player_ptr_sig + 7 + rip_offset;
    uintptr_t player_base = request->read<uintptr_t>(player_ptr_addr);
}
```

### 4. CR3 Stability

GTA V is more aggressive about CR3 changes than typical games.

**Add CR3 Validation**:
```cpp
class CR3Manager {
private:
    driver::communicate_t* driver;
    uintptr_t cached_cr3;
    uintptr_t base_address;
    std::chrono::steady_clock::time_point last_validation;
    const int VALIDATION_INTERVAL_MS = 5000;  // Re-validate every 5 seconds

public:
    CR3Manager(driver::communicate_t* drv, uintptr_t base) 
        : driver(drv), cached_cr3(0), base_address(base) {}
    
    bool initialize() {
        return refresh_cr3();
    }
    
    bool refresh_cr3() {
        if (!driver->get_cr3(base_address)) {
            return false;
        }
        cached_cr3 = driver->dtb;
        last_validation = std::chrono::steady_clock::now();
        return true;
    }
    
    bool validate_and_refresh() {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(now - last_validation);
        
        if (elapsed.count() > VALIDATION_INTERVAL_MS) {
            // Test if current CR3 still works
            uint16_t test = driver->read<uint16_t>(base_address);
            if (test != 0x5A4D) {  // MZ header should be consistent
                // CR3 changed, refresh
                return refresh_cr3();
            }
            last_validation = now;
        }
        return true;
    }
};
```

**Usage in Main Loop**:
```cpp
CR3Manager cr3_mgr(request, base_address);
if (!cr3_mgr.initialize()) {
    std::cout << "Failed to initialize CR3\n";
    return 1;
}

while (true) {
    // Before any read/write operations
    if (!cr3_mgr.validate_and_refresh()) {
        std::cout << "CR3 validation failed\n";
        Sleep(1000);
        continue;
    }
    
    // Perform operations
    auto value = request->read<int>(some_address);
    // ...
    
    Sleep(100);
}
```

### 5. Module Enumeration

FiveM servers may scan loaded modules.

**Add Module Hiding** (kernel-mode):

Create `KM/impl/module_hide.h`:
```cpp
#pragma once

// Hide DLL from PEB module list
NTSTATUS hide_module_from_peb(PEPROCESS process, const wchar_t* module_name) {
    PPEB peb = PsGetProcessPeb(process);
    if (!peb) return STATUS_UNSUCCESSFUL;
    
    PPEB_LDR_DATA ldr = (PPEB_LDR_DATA)peb->Ldr;
    if (!ldr) return STATUS_UNSUCCESSFUL;
    
    // Iterate through InLoadOrderModuleList
    for (PLIST_ENTRY entry = ldr->ModuleListLoadOrder.Flink;
         entry != &ldr->ModuleListLoadOrder;
         entry = entry->Flink) {
        
        PLDR_DATA_TABLE_ENTRY ldr_entry = CONTAINING_RECORD(
            entry, LDR_DATA_TABLE_ENTRY, InLoadOrderModuleList);
        
        if (wcsstr(ldr_entry->BaseDllName.Buffer, module_name)) {
            // Unlink from all three lists
            RemoveEntryList(&ldr_entry->InLoadOrderModuleList);
            RemoveEntryList(&ldr_entry->InMemoryOrderModuleList);
            RemoveEntryList(&ldr_entry->InInitializationOrderModuleList);
            return STATUS_SUCCESS;
        }
    }
    
    return STATUS_NOT_FOUND;
}
```

Add request handler for module hiding.

### 6. Screenshot Protection

FiveM servers can request screenshots to detect overlays/ESP.

**Mitigation Strategies**:

**A. Detect Screenshot Request**
```cpp
// Monitor for screenshot natives being called
// FiveM uses REQUEST_SCREEN_CAPTURE or similar

// Hook or monitor these game functions:
// - rage::grcTextureStore::Capture
// - Game screenshot functions

// When detected:
// 1. Disable overlays
// 2. Hide ESP elements
// 3. Wait for capture completion
// 4. Re-enable features
```

**B. Internal Rendering** (Advanced)
```cpp
// Instead of external overlay, render internally
// Hook game's DirectX/Vulkan present
// Draw ESP elements in game's rendering context
// Harder to detect via screenshots
```

---

## Implementation Steps

### Step 1: Environment Setup

1. **Install GTA V and FiveM**
```
1. Purchase/Install GTA V (Steam/Epic/Rockstar)
2. Download FiveM from fivem.net
3. Install and run once to initialize
4. Join a test server (ideally your own)
```

2. **Setup Development Environment**
```
1. Create local FiveM server (optional, for testing)
   - Download server files from fivem.net
   - Configure server.cfg
   - Start with minimal resources

2. Use single-player FiveM server for initial testing
   - Less risk of detection
   - No server-side validation
   - Easier debugging
```

3. **Prepare Driver**
```
1. Follow main DOCUMENTATION.md build steps
2. Apply FiveM-specific modifications (see above)
3. Build in Release mode
```

### Step 2: Initial Testing

**Test 1: Process Detection**
```cpp
// UM code
int main() {
    AllocConsole();
    // ...
    
    std::cout << "Waiting for GTA5.exe...\n";
    const auto pid = request->get_process_pid(L"GTA5.exe");
    if (pid == 0) {
        std::cout << "GTA5.exe not found!\n";
        system("pause");
        return 1;
    }
    std::cout << "Found GTA5.exe (PID: " << pid << ")\n";
    
    // Continue...
}
```

**Test 2: Module Enumeration**
```cpp
// Test key modules
struct Module {
    const char* name;
    uintptr_t base;
};

Module modules[] = {
    {"GTA5.exe", 0},
    {"CitizenGame.dll", 0},
    {"steam_api64.dll", 0},  // If Steam version
};

for (auto& mod : modules) {
    if (mod.name == "GTA5.exe") {
        mod.base = request->get_image_base(nullptr);
    } else {
        mod.base = request->get_image_base(mod.name);
    }
    
    std::cout << mod.name << ": ";
    if (mod.base) {
        std::cout << "0x" << std::hex << mod.base << "\n";
    } else {
        std::cout << "NOT FOUND\n";
    }
}
```

**Test 3: CR3 Resolution**
```cpp
auto result = request->get_cr3(game_base);
if (!result) {
    std::cout << "CR3 resolution failed!\n";
    return 1;
}
std::cout << "CR3 resolved: 0x" << std::hex << request->dtb << "\n";

// Verify with test read
auto mz_header = request->read<uint16_t>(game_base);
if (mz_header != 0x5A4D) {
    std::cout << "CR3 verification failed! (Read: 0x" << std::hex << mz_header << ")\n";
    return 1;
}
std::cout << "CR3 verified successfully\n";
```

### Step 3: Find Important Pointers

**Using Cheat Engine** (for initial research):
```
1. Attach Cheat Engine to GTA5.exe
2. Find values (health, position, etc.)
3. Pointer scan for stable pointers
4. Note base offsets from GTA5.exe
5. Create signatures for those offsets
```

**Example: Player Pointer**
```cpp
// In GTA V, player pointer is typically found via signature
SignatureScanner scanner(request, game_base, 0x6000000);  // Approximate size

// Example signature (game version dependent!)
// This is illustrative - actual signature will vary
uintptr_t player_sig = scanner.find_pattern("48 8B 05 ? ? ? ? 45 33 C9 48 8B 48 08");

if (player_sig) {
    // RIP-relative LEA: calculate actual address
    int32_t offset = request->read<int32_t>(player_sig + 3);
    uintptr_t player_ptr_address = player_sig + 7 + offset;
    
    // Read pointer
    uintptr_t player_base = request->read<uintptr_t>(player_ptr_address);
    std::cout << "Player base: 0x" << std::hex << player_base << "\n";
    
    // Common player offsets (example - verify with Cheat Engine)
    float health = request->read<float>(player_base + 0x280);
    std::cout << "Health: " << health << "\n";
}
```

### Step 4: Implement Core Features

**Example: God Mode**
```cpp
uintptr_t player = get_player_base();
if (player) {
    // GTA V health offset (example - game version specific)
    const uintptr_t HEALTH_OFFSET = 0x280;
    const uintptr_t MAX_HEALTH_OFFSET = 0x2A0;
    
    float max_health = request->read<float>(player + MAX_HEALTH_OFFSET);
    request->write<float>(player + HEALTH_OFFSET, max_health);
}
```

**Example: Teleport**
```cpp
struct Vector3 {
    float x, y, z;
};

void teleport(float x, float y, float z) {
    uintptr_t player = get_player_base();
    if (player) {
        const uintptr_t COORDS_OFFSET = 0x90;  // Example offset
        
        Vector3 pos = {x, y, z};
        request->write_virtual(player + COORDS_OFFSET, &pos, sizeof(Vector3));
    }
}
```

**Example: Money (Requires Server Compatibility)**
```cpp
// WARNING: Most FiveM servers validate money server-side
// This will only work on servers without proper validation

void set_money(int amount) {
    uintptr_t player = get_player_base();
    if (player) {
        const uintptr_t MONEY_OFFSET = 0x???;  // Find with CE
        request->write<int>(player + MONEY_OFFSET, amount);
    }
}
```

### Step 5: Server Validation Handling

**Understanding Server Validation**:
```
Client                          Server
  |                               |
  |-- Position Update ----------->|
  |                               |-- Validate: reasonable? --
  |                               |   - Speed check
  |                               |   - Distance check
  |                               |   - Physics check
  |<-- Reject/Correction ---------|
```

**Bypass Strategies**:

**A. Legitimate Value Manipulation**
```cpp
// Instead of instant teleport (detected):
void safe_teleport(float target_x, float target_y, float target_z) {
    const float MAX_DELTA = 5.0f;  // Max movement per frame
    
    while (true) {
        Vector3 current = get_position();
        Vector3 delta = {
            target_x - current.x,
            target_y - current.y,
            target_z - current.z
        };
        
        float distance = sqrt(delta.x*delta.x + delta.y*delta.y + delta.z*delta.z);
        if (distance < MAX_DELTA) {
            set_position(target_x, target_y, target_z);
            break;
        }
        
        // Move incrementally
        float ratio = MAX_DELTA / distance;
        set_position(
            current.x + delta.x * ratio,
            current.y + delta.y * ratio,
            current.z + delta.z * ratio
        );
        
        Sleep(10);  // Frame delay
    }
}
```

**B. Local-Only Features**
```cpp
// Features that don't require server sync:
// - ESP (visual only)
// - No recoil (local weapon handling)
// - Aimbot (local targeting)
// - Radar (visual overlay)
```

**C. Network Manipulation** (Advanced, requires packet analysis)
```cpp
// Intercept and modify network packets
// Requires additional kernel driver features
// Hook network stack or game's network functions
```

### Step 6: Anti-Detection

**Detection Vectors in FiveM**:

1. **Module Scanning**
```cpp
// Server requests: EnumerateLoadedModules()
// Mitigation: Hide UM executable from PEB
request->hide_module(L"your_tool.exe");
```

2. **Screenshot Requests**
```cpp
// Server can request screenshots
// Detection: Hook screenshot natives
// Mitigation: Hide overlays when screenshot detected

bool screenshot_in_progress = false;

void on_screenshot_request() {
    screenshot_in_progress = true;
    // Hide all visual elements
    disable_esp();
    disable_overlays();
}

void on_screenshot_complete() {
    screenshot_in_progress = false;
    // Re-enable
    enable_esp();
    enable_overlays();
}
```

3. **Performance Profiling**
```cpp
// Abnormal CPU/memory usage
// Mitigation: Optimize code, reduce polling frequency
Sleep(10);  // Between reads, not constant polling
```

4. **Debug Flags**
```cpp
// IsDebuggerPresent, PEB flags
// Mitigation: Clear debug flags

// In kernel driver, add function to clear PEB flags:
void clear_debug_flags(PEPROCESS process) {
    PPEB peb = PsGetProcessPeb(process);
    if (peb) {
        // Clear BeingDebugged flag
        *(BOOLEAN*)((ULONG_PTR)peb + 2) = FALSE;
        
        // Clear NtGlobalFlag
        *(ULONG*)((ULONG_PTR)peb + 0xBC) = 0;
        
        // Clear heap flags
        // ... (more extensive clearing)
    }
}
```

5. **Integrity Checks**
```cpp
// Server may hash game files
// Mitigation: Don't modify game files, only memory
// Use memory patches, not file patches
```

---

## FiveM-Specific Challenges

### Challenge 1: Server Authoritative Architecture

**Problem**: Server validates most game state
**Impact**: Memory manipulation may be overridden

**Solutions**:
- Focus on local-only features (ESP, aimbot)
- Use subtle value modifications (e.g., slight speed boost vs instant teleport)
- Analyze network protocol to send valid packets
- Target client-side predictions (movement, aiming)

### Challenge 2: Frequent Updates

**Problem**: GTA V and FiveM update regularly
**Impact**: Signatures and offsets break

**Solutions**:
- Use pattern scanning, not hardcoded offsets
- Maintain signature database for different game versions
- Implement version detection:
```cpp
std::string get_game_version() {
    // Read version from PE resources or version string
    uintptr_t version_ptr = scanner.find_pattern("48 8D 0D ? ? ? ? E8 ? ? ? ? 48 8D 0D ? ? ? ? E8");
    // Parse and return version string
}
```
- Automatic signature updating system

### Challenge 3: Script-Based Anti-Cheat

**Problem**: Server scripts can implement custom detection
**Impact**: Unpredictable detection methods

**Solutions**:
- Research specific server's AC implementation
- Join test servers with known AC
- Monitor FiveM forums for AC updates
- Stay under the radar (don't be blatant)

### Challenge 4: Resource System

**Problem**: FiveM loads/unloads resources dynamically
**Impact**: Module bases change, code caves invalidated

**Solutions**:
- Monitor resource events
- Re-scan when modules change
- Use stable base modules (GTA5.exe) for hooks
- Avoid hooking resource DLLs

---

## Testing and Validation

### Testing Phases

**Phase 1: Offline Testing**
```
1. Launch FiveM in single-player mode
2. Test driver attachment
3. Verify memory reads/writes
4. Test basic features (health, position)
5. No server connection = no detection risk
```

**Phase 2: Private Server Testing**
```
1. Setup private FiveM server
2. Minimal AC resources
3. Test features with server running
4. Monitor server logs for anomalies
5. Test server validation responses
```

**Phase 3: Public Server Testing**
```
1. Choose test server carefully (not your main)
2. Use burner account
3. Test one feature at a time
4. Monitor for kicks/bans
5. Refine based on detection
```

### Validation Checklist

- [ ] Driver loads successfully with FiveM/GTA5 running
- [ ] Process PID detected correctly
- [ ] Module bases resolved (GTA5.exe, CitizenGame.dll)
- [ ] CR3 resolution works and stays stable
- [ ] Memory reads return correct values
- [ ] Memory writes take effect
- [ ] No crashes during operation
- [ ] No BSOD from driver
- [ ] Process hiding works (if enabled)
- [ ] UM executable not detected by server
- [ ] Screenshots don't reveal overlays
- [ ] No server kicks for abnormal values
- [ ] Stable operation for extended periods (30+ min)

---

## Detection Vectors

### Client-Side Detection

1. **Module Enumeration**
   - EnumProcessModules
   - PEB LDR lists
   - **Mitigation**: Hide from PEB

2. **Memory Region Scanning**
   - VirtualQuery unusual regions
   - Scan for known signatures
   - **Mitigation**: Encrypt tool code

3. **Debug Flags**
   - PEB.BeingDebugged
   - NtGlobalFlag
   - Heap flags
   - **Mitigation**: Clear all debug flags

4. **Handle Detection**
   - Open handles to game process
   - Driver handles
   - **Mitigation**: Use driver (kernel-mode handles)

5. **Screenshot Analysis**
   - Detect overlays
   - Detect ESP boxes
   - **Mitigation**: Hide on screenshot event

### Server-Side Detection

1. **Statistical Analysis**
   - Abnormal accuracy
   - Impossible speeds
   - Rapid wealth gain
   - **Mitigation**: Humanize behavior

2. **Value Validation**
   - Health bounds
   - Position sanity
   - Weapon damage
   - **Mitigation**: Stay within valid ranges

3. **Packet Analysis**
   - Malformed packets
   - Impossible sequences
   - Rate violations
   - **Mitigation**: Send valid packets

4. **Behavioral Analysis**
   - Snap aiming
   - Perfect tracking
   - Inhuman reactions
   - **Mitigation**: Add randomness, delays

---

## Best Practices

### Development
1. Test offline first
2. Use private server for initial testing
3. One feature at a time
4. Log everything during development
5. Remove logs in production

### Features
1. Make features toggleable
2. Add hotkeys for quick disable
3. Implement "panic mode" (disable all)
4. Keep legitimate profiles for comparison

### Stealth
1. Don't be obvious (no flying, instant teleports)
2. Disable overlays on screenshots
3. Use reasonable values
4. Add randomization to avoid patterns
5. Don't use with public tools (detected faster)

### Safety
1. Use test accounts
2. Don't use on main account
3. Assume any server may detect
4. Have plausible deniability
5. Be prepared for bans

### Ethics
- Only use on servers where permitted
- Respect server rules
- Don't ruin other players' experience
- Consider running private server for testing
- Use for learning, not griefing

---

## Example: Complete FiveM Integration

Here's a minimal working example combining all concepts:

```cpp
// fivem_tool.cpp
#include "driver_api.h"
#include "signature_scanner.h"

class FiveMTool {
private:
    driver::communicate_t* driver;
    uint32_t game_pid;
    uintptr_t game_base;
    uintptr_t citizen_base;
    CR3Manager* cr3_manager;
    
    // Game pointers
    uintptr_t player_ptr_addr;
    
public:
    FiveMTool() : driver(nullptr), game_pid(0), game_base(0) {}
    
    bool initialize() {
        // Initialize driver
        driver = new driver::communicate_t();
        if (!driver->initialize_handle()) {
            std::cout << "[!] Failed to open driver handle\n";
            return false;
        }
        std::cout << "[+] Driver handle opened\n";
        
        // Find GTA5.exe
        game_pid = driver->get_process_pid(L"GTA5.exe");
        if (!game_pid) {
            std::cout << "[!] GTA5.exe not found\n";
            return false;
        }
        std::cout << "[+] Found GTA5.exe (PID: " << game_pid << ")\n";
        
        // Attach
        if (!driver->attach(game_pid)) {
            std::cout << "[!] Failed to attach\n";
            return false;
        }
        
        // Hide our process
        driver->unlinkprocess();
        std::cout << "[+] Process hidden\n";
        
        // Get game base
        game_base = driver->get_image_base(nullptr);
        if (!game_base) {
            std::cout << "[!] Failed to get game base\n";
            return false;
        }
        std::cout << "[+] Game base: 0x" << std::hex << game_base << "\n";
        
        // Verify it's FiveM
        citizen_base = driver->get_image_base("CitizenGame.dll");
        if (!citizen_base) {
            std::cout << "[!] Not FiveM (CitizenGame.dll not found)\n";
            return false;
        }
        std::cout << "[+] FiveM detected (CitizenGame.dll: 0x" << std::hex << citizen_base << ")\n";
        
        // Resolve CR3
        cr3_manager = new CR3Manager(driver, game_base);
        if (!cr3_manager->initialize()) {
            std::cout << "[!] CR3 resolution failed\n";
            return false;
        }
        std::cout << "[+] CR3 resolved: 0x" << std::hex << driver->dtb << "\n";
        
        // Find game pointers
        if (!find_signatures()) {
            std::cout << "[!] Failed to find signatures\n";
            return false;
        }
        
        return true;
    }
    
    bool find_signatures() {
        SignatureScanner scanner(driver, game_base, 0x6000000);
        
        // Find player pointer (example signature - game version dependent)
        uintptr_t player_sig = scanner.find_pattern("48 8B 05 ? ? ? ? 45 33 C9");
        if (!player_sig) {
            std::cout << "[!] Player signature not found\n";
            return false;
        }
        
        // Resolve RIP-relative address
        int32_t offset = driver->read<int32_t>(player_sig + 3);
        player_ptr_addr = player_sig + 7 + offset;
        std::cout << "[+] Player pointer: 0x" << std::hex << player_ptr_addr << "\n";
        
        return true;
    }
    
    uintptr_t get_player_base() {
        return driver->read<uintptr_t>(player_ptr_addr);
    }
    
    void god_mode(bool enable) {
        uintptr_t player = get_player_base();
        if (!player) return;
        
        // Example offsets - MUST BE UPDATED FOR CURRENT GAME VERSION
        const uintptr_t HEALTH_OFFSET = 0x280;
        const uintptr_t MAX_HEALTH_OFFSET = 0x2A0;
        
        if (enable) {
            float max_health = driver->read<float>(player + MAX_HEALTH_OFFSET);
            driver->write<float>(player + HEALTH_OFFSET, max_health);
        }
    }
    
    void run() {
        std::cout << "\n[+] FiveM Tool Running\n";
        std::cout << "[*] F1 - Toggle God Mode\n";
        std::cout << "[*] F2 - Exit\n\n";
        
        bool god_mode_enabled = false;
        
        while (true) {
            // Validate CR3 periodically
            cr3_manager->validate_and_refresh();
            
            // Check hotkeys
            if (GetAsyncKeyState(VK_F1) & 1) {
                god_mode_enabled = !god_mode_enabled;
                std::cout << "[*] God Mode: " << (god_mode_enabled ? "ON" : "OFF") << "\n";
            }
            
            if (GetAsyncKeyState(VK_F2) & 1) {
                std::cout << "[*] Exiting...\n";
                break;
            }
            
            // Apply features
            if (god_mode_enabled) {
                god_mode(true);
            }
            
            Sleep(100);  // Don't hammer the CPU
        }
    }
};

int main() {
    AllocConsole();
    FILE* fp;
    freopen_s(&fp, "CONOUT$", "w", stdout);
    
    std::cout << "=== FiveM Tool ===" << std::endl;
    
    FiveMTool tool;
    if (tool.initialize()) {
        tool.run();
    } else {
        std::cout << "\n[!] Initialization failed\n";
        system("pause");
    }
    
    return 0;
}
```

---

## Additional Resources

### FiveM Development
- FiveM Documentation: https://docs.fivem.net/
- FiveM Forums: https://forum.cfx.re/
- FiveM Native Reference: https://runtime.fivem.net/doc/reference.html

### GTA V Research
- GTA V Scripthook: http://www.dev-c.com/gtav/scripthookv/
- GTA V Native Database: http://www.dev-c.com/nativedb/
- UnknownCheats GTA V Section: https://www.unknowncheats.me/forum/grand-theft-auto-v/

### Tools
- Cheat Engine: Memory research
- IDA Pro/Ghidra: Reverse engineering
- x64dbg: Dynamic analysis
- Process Hacker: Process inspection
- Wireshark: Network analysis (for packet research)

---

## Disclaimer

This guide is for **educational purposes only**.

⚠️ **IMPORTANT WARNINGS**:
- Using external tools in FiveM violates Terms of Service
- Will result in bans from servers
- May violate laws in your jurisdiction
- Can damage server economies and player experience
- Author assumes no liability for misuse

**Ethical Usage**:
- Only use on servers where you have explicit permission
- Run your own test server for development
- Don't grief other players
- Respect server communities
- Use for learning about game security, not cheating

---

**Last Updated**: 2024  
**FiveM Version Compatibility**: Current release (check docs.fivem.net for latest)  
**GTA V Version**: May require updates with new game patches
