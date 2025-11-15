# BEKernelDriver - Complete Technical Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Components](#components)
4. [Security Features](#security-features)
5. [Build Instructions](#build-instructions)
6. [Deployment Guide](#deployment-guide)
7. [Configuration](#configuration)
8. [API Reference](#api-reference)
9. [Troubleshooting](#troubleshooting)

---

## Project Overview

**BEKernelDriver** is a Windows kernel-mode driver with user-mode companion application designed for advanced memory operations and process manipulation on systems protected by BattlEye anti-cheat. The driver uses sophisticated techniques including:

- **CR3/DTB-based memory access** for physical memory read/write operations
- **ObCreateObject communication hijacking** for undetected kernel-usermode communication
- **Process hiding** via EPROCESS list unlinking
- **Driver trace cleanup** (PiDDB, MMU/MML, HashBucket)
- **Code cave hooking** for stealthy function interception

### Compatibility
- **Tested on**: Windows 10/11 (x64)
- **Anti-cheat systems**: BattlEye (Escape From Tarkov, DayZ, Rainbow Six Siege)
- **Note**: EAC compatibility requires CR3 buffering/recovery mechanism

### License
Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International
- ✅ Share and redistribute
- ❌ Commercial use
- ❌ Modifications (for distribution)
- ℹ️ Attribution required

---

## Architecture

The project consists of two main components:

### 1. Kernel-Mode Driver (KM)
Located in `/KM` directory, this is the core driver that operates at ring 0.

```
KM/
├── entry/              # Driver entry point and initialization
│   ├── main.cpp       # DriverEntry, initialization sequence
│   └── hook/          # Hook implementation
│       └── hook.hpp   # IoCreateObject hook via code caves
├── impl/              # Core implementations
│   ├── communication/ # IOCTL interface definitions
│   ├── invoked.h      # Request dispatcher
│   ├── imports.h      # Kernel API imports
│   ├── scanner.h      # Pattern scanning utilities
│   └── modules.h      # Module enumeration
├── kernel/            # Kernel utilities
│   ├── imports.h      # NT kernel function declarations
│   ├── struct.h       # Kernel structures (EPROCESS, etc.)
│   ├── log.h          # Debug logging
│   └── xor.h          # String obfuscation
├── processhyde/       # Process hiding functionality
│   ├── Hide.cpp       # EPROCESS unlinking
│   └── Offset.cpp     # Dynamic offset calculation
├── requests/          # Request handlers
│   ├── read_physical_memory.cpp
│   ├── write_physical_memory.cpp
│   ├── get_module_base.cpp
│   ├── signature_scanner.cpp
│   └── virtual_allocate.cpp
└── clean/             # Driver trace removal
    └── clean.hpp      # PiDDB/MMU/HashBucket cleanup
```

### 2. User-Mode Application (UM)
Located in `/UM` directory, communicates with the kernel driver.

```
UM/
├── entrypoint.cpp     # Main entry, driver initialization
├── impl/
│   └── driver/
│       ├── driver.cpp # Communication API implementation
│       └── driver.hpp # API interface definitions
├── Mapper.h           # Mapper integration stub
├── globals.h          # Global configuration
└── imports.h          # Windows API imports
```

---

## Components

### Kernel-Mode Driver Components

#### 1. Entry Point (`entry/main.cpp`)
**Purpose**: Driver initialization and setup

**Key Functions**:
- `DriverEntry()`: Main entry point called by Windows
- `OEPDriverEntry()`: Original entry point (post-hook)
- Hook initialization
- IOCTL interface setup
- Driver trace cleanup

**Initialization Sequence**:
```cpp
1. DriverEntry() called by Windows loader
2. initialize_hook() - Sets up communication hook
3. initialize_ioctl() - Creates device and symbolic link
4. CleanDriverSys() - Removes mapper traces
5. Driver operational
```

**Important Notes**:
- Must configure hook driver in line 89 (target module)
- CleanDriverSys removes mapper artifacts (lines 83-84)

#### 2. Hook Mechanism (`entry/hook/hook.hpp`)
**Purpose**: Establishes stealthy communication channel

**Technique**: Code cave hijacking in legitimate driver
1. Locates target kernel module (not PatchGuard protected)
2. Finds code cave via pattern scanning
3. Writes trampoline shellcode to cave
4. Redirects execution to custom handlers

**Functions**:
- `initialize_hook()`: Main hook setup
  - Gets ntoskrnl base
  - Finds IoCreateDriver export
  - Locates target module (line 89 - MUST BE CONFIGURED)
  - Finds code cave (line 96-98 - MUST BE CONFIGURED)
  - Writes trampoline shellcode

- `manual_mapped_entry()`: Hook handler
  - Creates device object
  - Sets up IRP handlers (CREATE, CLOSE, DEVICE_CONTROL)
  - Creates symbolic link for user-mode access

**Configuration Required**:
```cpp
// Line 89: Target module for code cave
const auto target_module = modules::get_kernel_module(_("TARGET_DRIVER.sys"));

// Lines 96-98: Signature and mask for code cave
globals::cave_base = modules::find_pattern(target_module,
    _("SIGNATURE_BYTES"),  // Your pattern
    _("MASK"));            // Matching mask
```

**Shellcode Structure**:
```assembly
; Trampoline (12 bytes)
mov rax, <address>  ; 0x48, 0xB8 + 8 byte address
jmp rax             ; 0xFF, 0xE0
```

#### 3. Communication Interface (`impl/communication/interface.h`)
**Purpose**: Defines IOCTL protocol between user/kernel mode

**Device Names** (MUST BE CHANGED):
```cpp
#define DEVICE_NAME L"\\Device\\MicroTech32_c"    // Line 2
#define DOS_NAME L"\\DosDevices\\MicroTech32_c"   // Line 3
```

**Request Codes** (`requests` enum):
- `invoke_base`: Get module base address
- `invoke_read`: Read physical memory
- `invoke_write`: Write physical memory
- `invoke_translate`: Translate virtual to physical address
- `invoke_dtb`: Get process DTB (Directory Table Base)
- `invoke_hideproc`: Hide process from EPROCESS list
- `invoke_protect_virtual`: Virtual memory allocation
- `invoke_read_kernel`: Read kernel memory

**Data Structures**:
```cpp
// Main request wrapper
typedef struct _invoke_data {
    uint32_t unique;      // Request validator
    requests code;        // Request type
    void* data;          // Pointer to specific request struct
} invoke_data;

// Read operation
typedef struct _read_invoke {
    uint32_t pid;        // Target process ID
    uintptr_t address;   // Virtual address
    uintptr_t dtb;       // Directory table base
    void* buffer;        // Output buffer
    size_t size;         // Bytes to read
} read_invoke;

// Write operation
typedef struct _write_invoke {
    uint32_t pid;
    uintptr_t address;
    void* buffer;
    size_t size;
} write_invoke;

// Module base
typedef struct _base_invoke {
    uint32_t pid;
    uintptr_t handle;    // Output: module base
    const char* name;    // Module name (NULL for main module)
    size_t size;
} base_invoke;

// DTB retrieval
typedef struct _dtb_invoke {
    uint32_t pid;
    uintptr_t dtb;       // Output: directory table base
} dtb_invoke;

// Address translation
typedef struct _translate_invoke {
    uintptr_t virtual_address;
    uintptr_t directory_base;
    void* physical_address;  // Output
} translate_invoke;
```

#### 4. Request Dispatcher (`impl/invoked.h`)
**Purpose**: Routes IOCTL requests to appropriate handlers

**Main Dispatcher**: `vortex::io_dispatch()`
- Validates request
- Routes to handler based on request code
- Returns result to user-mode

**Request Flow**:
```
User-mode → DeviceIoControl() → io_dispatch() → Handler → Response
```

#### 5. Memory Operations (`requests/`)

##### Read Physical Memory (`read_physical_memory.cpp`)
**Function**: `request::read_memory()`

**Process**:
1. Validates parameters (address, size, PID)
2. Looks up PEPROCESS by PID
3. Translates virtual address to physical using DTB
4. Reads from physical address via `MmCopyMemory()`
5. Handles page boundary conditions

**Key Features**:
- CR3/DTB-based translation
- Page-aware reading (respects 4KB boundaries)
- Physical memory access (bypasses memory protection)

**Translation**: `translate_linear()`
- Implements x64 4-level paging
- Handles 1GB/2MB large pages
- Returns physical address or 0 on failure

##### Write Physical Memory (`write_physical_memory.cpp`)
**Function**: `request::write_memory()`

**Process**:
1. Validates write parameters
2. Looks up target process
3. Uses `MmCopyVirtualMemory()` for write operation
4. Verifies bytes written

**Note**: Uses virtual memory copy (not physical like read)

##### Get Module Base (`get_module_base.cpp`)
**Function**: `request::get_module_base()`

**Process**:
1. If name is NULL: returns main module base via `PsGetProcessSectionBaseAddress()`
2. If name provided:
   - Attaches to process context
   - Enumerates PEB→Ldr→ModuleListLoadOrder
   - Compares module names (Unicode)
   - Returns DllBase of matching module

**Usage**:
- Get process base: `name = NULL`
- Get DLL base: `name = "module.dll"`

##### Address Translation
**Function**: `request::translate_address()`

Exposes `translate_linear()` to user-mode for manual address translation.

##### DTB Retrieval (`get_dtb()`)
**Function**: `request::get_dtb()`

Retrieves Directory Table Base (CR3) for a process:
- Reads EPROCESS+0x28 (x64 DTB offset)
- Fallback to EPROCESS+0x388 (x32 processes)

#### 6. Process Hiding (`processhyde/`)

##### Hide.cpp
**Function**: `HideProcess()`

**Technique**: EPROCESS doubly-linked list unlinking

**Process**:
1. `InitializeOffsets()`: Calculates dynamic EPROCESS offsets
   - Process name offset
   - PID offset  
   - List entry offset (PID + sizeof(HANDLE))

2. Traverses ActiveProcessLinks list
3. Finds target process by name comparison
4. Unlinks from list (A←→B←→C becomes A←→C)
5. Points target's links to itself

**Configuration Required**:
```cpp
// Line 34: Change to your user-mode executable name
const char target[] = "pcom5.exe";  // CHANGE THIS
```

**Visual Representation**:
```
Before: SystemProcess ↔ Process1 ↔ TARGET ↔ Process2
After:  SystemProcess ↔ Process1 ↔ Process2
        TARGET ↔ TARGET (isolated)
```

##### Offset.cpp
**Function**: `CalcProcessNameOffset()`, `CalcPIDOffset()`

Dynamically calculates EPROCESS structure offsets by:
1. Getting current process (guaranteed known)
2. Searching memory for known values
3. Calculating offset from EPROCESS base

**Why Dynamic**: EPROCESS structure changes between Windows versions

#### 7. Driver Trace Cleanup (`clean/clean.hpp`)

**Purpose**: Remove mapper artifacts to avoid detection

##### Functions:

**`clear::clearCache()`** - PiDDB Cache Cleanup
- Locates PiDDB (Driver Database) lock and table
- Searches for driver by name and timestamp
- Removes entry from AVL tree
- Prevents "known vulnerable driver" detection

**`clear::clearHashBucket()`** - CI.dll Hash Cleanup
- Locates g_KernelHashBucketList in ci.dll
- Finds driver entry by name
- Randomizes SHA1 hash (20 bytes)
- Prevents hash-based detection

**`clear::CleanMmu()`** - MMU/MML Cleanup
- Cleans MmUnloadedDrivers array
- Removes unload traces
- Shifts remaining entries
- Adjusts timestamps for consistency

**Invocation**:
```cpp
// In main.cpp DriverEntry
CleanDriverSys(UNICODE_STRING(RTL_CONSTANT_STRING(L"DriverKL.sys")), 0x63EF9904);
CleanDriverSys(UNICODE_STRING(RTL_CONSTANT_STRING(L"PdFwKrnl.sys")), 0x611AB60D);
```

**Parameters**:
- Driver name (Unicode string)
- TimeDateStamp (from PE header)

#### 8. Module Utilities (`impl/modules.h`)
Functions for module enumeration and manipulation:
- `get_kernel_module()`: Find loaded kernel module
- `get_kernel_export()`: Get exported function address
- `find_pattern()`: Signature scanning in module
- `write_address()`: Write to kernel memory with CR0 protection bypass
- `safe_copy()`: Safe kernel memory copy with exception handling
- `attach_process()`: Context switch to target process

#### 9. Utilities

##### String Obfuscation (`kernel/xor.h`)
Compile-time XOR encryption for strings:
```cpp
#define _(str) xor_string(str)
```
Prevents static string analysis.

##### Logging (`kernel/log.h`)
Debug logging via `DbgPrintEx()`:
```cpp
log(_("Message"));
print_dbg(_("Debug info"));
```

##### CRT Functions (`impl/crt.h`)
Kernel-mode standard library replacements:
- `kmemset()`, `kmemcpy()`, `kstrlen()`, etc.
- Required because kernel doesn't link CRT

---

### User-Mode Application Components

#### 1. Entry Point (`entrypoint.cpp`)
**Purpose**: Initialize driver communication and demonstrate usage

**Main Flow**:
```cpp
1. Initialize console
2. Get driver handle (initialize_handle())
3. Unlink process from EPROCESS list
4. Wait for F5 key
5. Get target process PID
6. Attach to process
7. Get image base
8. Resolve CR3/DTB
```

**Example**:
```cpp
const auto pid = request->get_process_pid(L"escapefromtarkov.exe");
request->attach(pid);
auto base_address = request->get_image_base(nullptr);
request->get_cr3(base_address);
```

#### 2. Driver Communication API (`impl/driver/driver.cpp`)

##### Class: `driver::communicate_t`

**Methods**:

**`initialize_handle()`**
- Opens handle to driver device
- Returns true if successful
```cpp
this->m_handle = CreateFileA(device_name, GENERIC_READ, 0, 0, 3, 0x00000080, 0);
```

**`get_process_pid(wstring name)`**
- Enumerates processes via `CreateToolhelp32Snapshot()`
- Returns PID of matching process

**`attach(int pid)`**
- Sets internal PID for subsequent operations

**`send_cmd(void* data, requests code)`**
- Sends IOCTL request to driver
- Uses NtDeviceIoControlFile (custom import)
- Returns operation result

**`get_image_base(const char* module_name)`**
- Gets main module base (if name == NULL)
- Gets specific DLL base (if name provided)
- Returns base address

**`read_virtual(address, buffer, size)`**
- Reads memory from attached process
- Uses DTB for translation
- Returns success/failure

**`write_virtual(address, buffer, size)`**
- Writes memory to attached process
- Returns success/failure

**`read<T>(address)`** (template)
- Type-safe memory read
- Returns value of type T
```cpp
auto value = request->read<int>(address);
```

**`write<T>(address, value)`** (template)
- Type-safe memory write

**`get_cr3(base_address)`** - CR3/DTB Resolution
- Critical for physical memory access
- Resolves target process DTB

**Algorithm**:
```cpp
1. Get current process DTB (known reference)
2. Get physical address of ntdll.dll in current process
3. Iterate through possible DTB values (0 to 0x50000000, step 0x1000)
4. For each DTB:
   a. Translate ntdll virtual address using candidate DTB
   b. Compare physical addresses
   c. If match, verify by reading MZ header from target
5. Save matching DTB
```

**Why This Works**:
- ntdll.dll loaded at same virtual address in all processes
- Physical address must match if DTB is correct
- MZ header verification ensures read works

**`translate_address(virtual, directory_base)`**
- Calls kernel's translate function
- Returns physical address

**`get_dtb(pid)`**
- Gets DTB for any process
- Uses kernel request

**`read_kernel(address, buffer, size, memory_type)`**
- Reads kernel memory
- Supports physical and virtual modes

**`virtual_protect(address, size, allocation_type)`**
- Allocates virtual memory in target process

**`unlinkprocess()`**
- Triggers process hiding in kernel

---

## Security Features

### 1. Anti-Detection Mechanisms

#### Code Obfuscation
- **XOR String Encryption**: All strings encrypted at compile-time
- **VMProtect Recommended**: Mentioned in setup (use commercial protector)
- **No Hardcoded Addresses**: Dynamic resolution

#### Communication Hiding
- **ObCreateObject Hook**: Not standard IOCTL
- **Custom Device Names**: Must be changed (not default)
- **No Driver Object**: Hook hijacks existing driver

#### Driver Trace Removal
- **PiDDB**: Removes from driver database
- **MMU/MML**: Cleans unload traces
- **HashBucket**: Randomizes CI.dll hashes

#### Process Hiding
- **EPROCESS Unlinking**: Process hidden from task manager
- **Dynamic Offsets**: Adapts to Windows version

### 2. Memory Access Security

#### CR3-Based Access
- Reads/writes at physical level
- Bypasses user-mode hooks
- Bypasses memory protection (PAGE_READONLY, etc.)

#### Validation
- Address range checking (< 0x7FFFFFFFFFFF)
- Process validation via PsLookupProcessByProcessId
- Size validation
- Page boundary handling

---

## Build Instructions

### Prerequisites

#### Software Requirements
1. **Visual Studio 2019/2022**
   - Desktop development with C++
   - Windows SDK 10.0.19041.0 or later

2. **Windows Driver Kit (WDK)**
   - Download from Microsoft
   - Must match SDK version
   - Includes:
     - Driver compiler (cl.exe for kernel)
     - Driver linker
     - Inf2Cat
     - SignTool

3. **Code Protector (Optional but Recommended)**
   - VMProtect
   - Themida
   - Or similar code mutation tool
   - Purpose: Prevent signature scanning

#### Hardware Requirements
- Windows 10/11 x64 (VM or physical)
- Test signing or disabled signature verification
- 4GB+ RAM recommended

### Build Steps

#### 1. Clone Repository
```bash
git clone https://github.com/TerraTR/BEKernelDriverUpdated
cd BEKernelDriverUpdated
```

#### 2. Open Solution
```
Open KrnlUpdated.sln in Visual Studio
```

#### 3. Configure Projects

**Solution contains 2 projects**:
- `driver-execute` (KM - Kernel driver)
- `um` (UM - User application)

#### 4. Build Kernel Driver

**Configuration**: Release x64

**Steps**:
1. Right-click `driver-execute` project
2. Select Configuration Manager
3. Set Active solution configuration: **Release**
4. Set Platform: **x64**
5. Build → Build driver-execute

**Output**: `x64/Release/driver-execute.sys`

**Common Build Errors**:
- **WDK not found**: Install Windows Driver Kit
- **SDK version mismatch**: Update target SDK in project properties
- **Spectre mitigation**: Disable or install Spectre-mitigated libraries

#### 5. Build User Application

**Configuration**: Release x64

**Steps**:
1. Right-click `um` project
2. Configuration Manager → Release x64
3. Build → Build um

**Output**: `x64/Release/um.exe`

#### 6. Post-Build (Optional)

**Apply Code Protection**:
```
1. VMProtect: Load um.exe
2. Add protection to all functions
3. Enable mutation
4. Save protected executable
```

**Sign Driver (Test Environment)**:
```cmd
REM Create test certificate (once)
makecert -r -pe -ss PrivateCertStore -n "CN=TestDriverCert" TestCert.cer

REM Sign driver
signtool sign /a /v /s PrivateCertStore /n TestDriverCert /t http://timestamp.digicert.com driver-execute.sys
```

---

## Deployment Guide

### Prerequisites

#### 1. Disable Driver Signature Enforcement

**Method 1: Test Signing (Development)**
```cmd
REM Run as Administrator
bcdedit /set testsigning on
REM Reboot required
shutdown /r /t 0
```

**Method 2: Disable Signature Verification (Testing Only)
```cmd
REM Run from Advanced Startup
bcdedit /set nointegritychecks on
```

⚠️ **WARNING**: These methods are for testing only. Do not use in production or on systems with sensitive data.

#### 2. Disable PatchGuard (Required)

PatchGuard (Kernel Patch Protection) will BSOD if it detects driver modifications.

**Recommended Solution**: EFIGuard
```
1. Download EFIGuard from GitHub
2. Configure UEFI bootloader
3. Adds DSEFix
4. Disables PatchGuard checks
```

**Alternative**: DSEFix, InfinityHook (research required)

### Deployment Steps

#### Step 1: Configure Driver

**File**: `KM/entry/main.cpp`

**Line 89-98**: Configure hook target
```cpp
// Find a target driver that:
// 1. Is loaded and persistent
// 2. Is NOT PatchGuard protected
// 3. Has executable code sections
// 4. Has identifiable patterns for code caves

const auto target_module = modules::get_kernel_module(_("diskperf.sys")); // Example

// Find code cave signature in target
// Use IDA/Ghidra to find unused code with pattern
globals::cave_base = modules::find_pattern(target_module,
    _("\x48\x89\x5C\x24\x08\x48\x89\x74\x24\x10"),  // Your signature
    _("xxxxxxxxxx"));                                 // Mask
```

**File**: `KM/impl/communication/interface.h`

**Lines 2-3**: Change device identifiers
```cpp
#define DEVICE_NAME L"\\Device\\YourUniqueName"
#define DOS_NAME L"\\DosDevices\\YourUniqueName"
```

**File**: `KM/processhyde/Hide.cpp`

**Line 34**: Set user-mode executable name
```cpp
const char target[] = "your_um_app.exe";  // Match UM executable name
```

#### Step 2: Prepare Mapper

**DO NOT USE**:
- KDMapper (detected)
- Manual mapping (detected)
- TDL/DSEFix direct load

**USE**:
- **PdFwKrnl Mapper** (by i32-Sudo) - Recommended
  - GitHub: github.com/i32-Sudo/PdFwKrnlMapper
  - Uses vulnerable driver exploit
  - Cleans traces
  
**Download mapper**:
```bash
git clone https://github.com/i32-Sudo/PdFwKrnlMapper
# Build according to mapper instructions
```

#### Step 3: Map Driver

**Using PdFwKrnl Mapper**:
```cmd
REM Run as Administrator
PdFwKrnlMapper.exe driver-execute.sys

Expected output:
[+] Vulnerable driver loaded
[+] Kernel shellcode allocated
[+] Driver image mapped
[+] Entry point called
[+] Driver initialized
[+] Vulnerable driver unloaded
[+] Success
```

**Verification**:
```cmd
REM Check if device exists (different methods)
REM Method 1: List devices (won't show due to hook)
REM Method 2: Try to open handle (from your app)
```

#### Step 4: Run User Application

```cmd
your_um_app.exe

Expected output:
Press F5 Once in menu...
[After F5 and target process running]
(No output if successful, console closes on error)
```

#### Step 5: Verify Operation

**Check Process Hiding**:
```cmd
REM Before: your_um_app.exe visible in Task Manager
REM After unlinkprocess(): NOT visible in Task Manager
REM But process still running (check with Process Hacker's kernel mode)
```

**Check Memory Access**:
```cpp
// In your code
auto value = request->read<int>(some_address);
// Should return correct value
```

### Troubleshooting Deployment

#### Driver Fails to Load
**Symptoms**: Mapper reports error, BSOD

**Solutions**:
1. Check PatchGuard is disabled
2. Verify test signing is enabled
3. Check target module exists and is loaded
4. Verify pattern matches in target module

**Debug**:
```cmd
REM Check loaded modules
driverquery

REM Check kernel log (WinDbg or DebugView)
# Look for DbgPrintEx messages from driver
```

#### Cannot Open Handle
**Symptoms**: `initialize_handle()` returns false

**Solutions**:
1. Verify device name matches (interface.h and driver.cpp)
2. Check driver loaded successfully
3. Ensure symbolic link created
4. Run as Administrator

**Debug**:
```cpp
// Add before CreateFileA
std::cout << "Opening: " << device_name << std::endl;
// Check exact string
```

#### Process Not Hidden
**Symptoms**: Process still visible after `unlinkprocess()`

**Solutions**:
1. Verify executable name matches (Hide.cpp line 34)
2. Check InitializeOffsets() succeeded
3. Verify HideProcess() called

**Debug**:
```cpp
// In Hide.cpp, uncomment lines 46-47
DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "Process String: %s\n", currentProcessName.Buffer);
// Check if your process name appears
```

#### Read/Write Fails
**Symptoms**: `read_virtual()` returns false, wrong data

**Solutions**:
1. Verify CR3 resolution succeeded (get_cr3)
2. Check target process is running
3. Validate address is user-mode (< 0x7FFFFFFFFFFF)
4. Check process didn't restart (CR3 changed)

**Debug**:
```cpp
// Add logging
std::cout << "DTB: 0x" << std::hex << request->dtb << std::endl;
std::cout << "Reading from: 0x" << address << std::endl;
```

#### BSOD During Operation
**Symptoms**: Blue screen, KERNEL_SECURITY_CHECK_FAILURE

**Common Causes**:
1. PatchGuard triggered (disable properly)
2. Bad memory access (invalid pointer)
3. CR3 invalidated (EAC games)
4. Stack corruption

**Debug**:
```
1. Boot in Safe Mode
2. Analyze dump with WinDbg
3. Check call stack for driver code
4. Review last operation before crash
```

---

## Configuration

### Required Configuration Changes

Before deploying, you **MUST** change these values:

#### 1. Device Names
**File**: `KM/impl/communication/interface.h`
```cpp
// Lines 2-3
#define DEVICE_NAME L"\\Device\\YourRandomName"
#define DOS_NAME L"\\DosDevices\\YourRandomName"

// Recommendations:
// - Use random string (not "Driver", "Kernel", etc.)
// - Mix letters and numbers
// - 8-16 characters
// - Examples: "MicroTech32_c", "SysAudio47_x", "WinUtil19_k"
```

#### 2. Hook Target Module
**File**: `KM/entry/hook/hook.hpp`
```cpp
// Line 89
const auto target_module = modules::get_kernel_module(_("YOUR_TARGET.sys"));

// Requirements:
// - Must be loaded kernel driver
// - NOT PatchGuard protected
// - Has executable code sections
// - Persistent (not unloaded)

// Candidates (research required):
// - diskperf.sys (disk performance)
// - Beep.sys (beep driver) - often used
// - null.sys (null device)
// - Various OEM drivers

// Lines 96-98: Code cave signature
globals::cave_base = modules::find_pattern(target_module,
    _("YOUR_SIGNATURE"),  // Byte pattern
    _("YOUR_MASK"));      // Mask (x = exact, . = wildcard)

// How to find:
// 1. Load target driver in IDA Pro/Ghidra
// 2. Find .text section
// 3. Look for:
//    - Unused code (unreferenced)
//    - Padding between functions (CC CC CC...)
//    - Error handling code (rarely executed)
// 4. Extract byte pattern and create mask
```

#### 3. Process Name
**File**: `KM/processhyde/Hide.cpp`
```cpp
// Line 34
const char target[] = "your_app.exe";  // EXACTLY your UM executable name
```

#### 4. Mapper Cleanup
**File**: `KM/entry/main.cpp`
```cpp
// Lines 83-84
CleanDriverSys(UNICODE_STRING(RTL_CONSTANT_STRING(L"YourMapper.sys")), TIMESTAMP);

// Get timestamp:
// 1. Open your mapper driver in PE viewer
// 2. Find TimeDateStamp in PE header
// 3. Convert to hex
// Or use: dumpbin /headers mapper.sys | findstr "time date"
```

### Optional Configuration

#### Target Process
**File**: `UM/entrypoint.cpp`
```cpp
// Line 24
const auto pid = request->get_process_pid(L"your_target.exe");

// Change to your target application
```

#### Logging
**Enable/Disable Debug Logging**:
```cpp
// Throughout code, comment/uncomment DbgPrintEx calls
// Example in main.cpp:
// DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, _("Message"));
```

#### Memory Pool Tag
**File**: `KM/clean/clean.hpp`
```cpp
// Line 8
#define BB_POOL_TAG 'Esk'  // Change to unique tag
// Format: 4 characters, reversed in memory
```

---

## API Reference

### User-Mode API

#### Class: `driver::communicate_t`

**Initialization**
```cpp
driver::communicate_t* driver = new driver::communicate_t();
bool success = driver->initialize_handle();
```

**Process Operations**
```cpp
// Get process ID by name
uint32_t pid = driver->get_process_pid(L"process.exe");

// Attach to process
bool result = driver->attach(pid);

// Hide process (unlink from EPROCESS list)
void driver->unlinkprocess();
```

**Module Operations**
```cpp
// Get main module base
uintptr_t base = driver->get_image_base(nullptr);

// Get DLL base
uintptr_t dll_base = driver->get_image_base("kernel32.dll");
```

**Memory Read**
```cpp
// Template read (type-safe)
int value = driver->read<int>(address);
float health = driver->read<float>(base + 0x1234);

// Buffer read
uint8_t buffer[256];
bool result = driver->read_virtual(address, buffer, sizeof(buffer));
```

**Memory Write**
```cpp
// Template write
driver->write<int>(address, 999);
driver->write<float>(base + 0x1234, 100.0f);

// Buffer write
uint8_t data[] = {0x90, 0x90, 0x90};  // NOP slide
bool result = driver->write_virtual(address, data, sizeof(data));
```

**DTB/CR3 Operations**
```cpp
// Resolve target process DTB (required for read/write)
bool success = driver->get_cr3(base_address);

// Get DTB for any process
uintptr_t dtb = driver->get_dtb(pid);

// Translate virtual to physical
uintptr_t physical = driver->translate_address(virtual_addr, dtb);
```

**Kernel Memory Access**
```cpp
// Read kernel memory
uint8_t kernel_data[128];
bool result = driver->read_kernel(
    kernel_address,
    kernel_data,
    sizeof(kernel_data),
    MM_COPY_MEMORY_VIRTUAL  // or MM_COPY_MEMORY_PHYSICAL
);
```

**Memory Allocation**
```cpp
// Allocate memory in target process
bool result = driver->virtual_protect(
    address,                  // Base address (0 for auto)
    size,                    // Size in bytes
    MEM_COMMIT | MEM_RESERVE // Allocation type
);
```

### Kernel-Mode Request Handlers

All kernel requests use the `invoke_data` wrapper structure sent via DeviceIoControl.

**Request Format**:
```cpp
invoke_data request;
request.unique = invoke_unique;  // Validation token
request.code = REQUEST_TYPE;     // See requests enum
request.data = &specific_data;   // Pointer to request-specific structure

// Send via NtDeviceIoControlFile or DeviceIoControl
```

**Example: Read Memory**
```cpp
// User-mode side
read_invoke read_data;
read_data.pid = target_pid;
read_data.dtb = process_dtb;
read_data.address = target_address;
read_data.buffer = output_buffer;
read_data.size = bytes_to_read;

invoke_data request;
request.unique = invoke_unique;
request.code = invoke_read;
request.data = &read_data;

// Send request
NtDeviceIoControlFile(handle, ...&request, sizeof(request)...);

// Kernel-mode handler
request::read_memory(&request);
// - Validates parameters
// - Translates virtual to physical
// - Reads from physical memory
// - Copies to user buffer
```

---

## Troubleshooting

### Build Issues

#### Error: Cannot open include file 'ntddk.h'
**Solution**: Install Windows Driver Kit (WDK)

#### Error: Spectre mitigation libraries required
**Solutions**:
1. Install Spectre-mitigated libraries via Visual Studio Installer
2. Or disable: Project Properties → C/C++ → Code Generation → Spectre Mitigation = Disabled

#### Error: Unresolved external symbol
**Solution**: Check WDK library paths in project settings

### Runtime Issues

#### BSOD: DRIVER_IRQL_NOT_LESS_OR_EQUAL
**Cause**: Accessing paged memory at elevated IRQL
**Solution**: Review memory access; use IRQL-safe functions

#### BSOD: KERNEL_SECURITY_CHECK_FAILURE
**Cause**: PatchGuard triggered
**Solution**: Ensure PatchGuard bypass (EFIGuard) is active

#### BSOD: PAGE_FAULT_IN_NONPAGED_AREA
**Cause**: Invalid memory access
**Solution**: Validate all pointers before dereferencing

#### Cannot find target module
**Cause**: Target driver not loaded or name mismatch
**Solution**: 
```cmd
driverquery | findstr "your_target"
# Verify exact name including .sys
```

#### Process not found
**Cause**: Process name mismatch or not running
**Solution**: Check exact process name (including .exe)

#### CR3 resolution fails
**Cause**: ntdll not loaded or DTB search range too small
**Solution**: 
- Ensure target process fully initialized
- Increase search range (line 238 in driver.cpp: 0x50000000)

#### Reads return zero/garbage
**Causes**:
1. Wrong DTB (CR3 changed)
2. Invalid address
3. Target process restarted

**Solutions**:
1. Re-run get_cr3() periodically
2. Validate addresses
3. Check process ID didn't change

### EAC Compatibility

**Issue**: Blue screen after 10-20 minutes in EAC games

**Cause**: EAC resets CR3 periodically, driver reads with stale DTB

**Solution** (requires implementation):
```cpp
// Add CR3 validation before each read
auto new_cr3 = get_cr3(base_address);
if (new_cr3 != cached_cr3) {
    cached_cr3 = new_cr3;
}

// Or catch bad reads
try {
    read_memory(...);
} catch {
    // Re-cache CR3
    cached_cr3 = get_cr3(base_address);
    retry();
}
```

### Detection Issues

**Issue**: Driver/process detected by anti-cheat

**Causes**:
1. Known mapper traces (PiDDB, MMU)
2. Signature matches (public code)
3. Driver object exposed
4. Process in EPROCESS list

**Solutions**:
1. Verify cleanup functions work (check with kernel debugger)
2. Use VMProtect/Themida on user-mode app
3. Ensure hook target is not monitored
4. Verify process hiding works
5. Change all identifiers (device names, pool tags, strings)
6. Modify code patterns before compiling

---

## Best Practices

### Security
1. ✅ Always change device names
2. ✅ Use code protector on user-mode executable
3. ✅ Change pool tags and identifiers
4. ✅ Test in isolated VM first
5. ❌ Never use on production system
6. ❌ Never distribute with default configurations

### Development
1. Use kernel debugger (WinDbg) for testing
2. Test each component separately
3. Log operations during development
4. Remove debug output in release
5. Validate all input from user-mode
6. Handle errors gracefully (no asserts in production)

### Deployment
1. Test mapper separately
2. Verify PatchGuard bypass
3. Check driver loads before running user app
4. Monitor system stability
5. Have restoration plan ready
6. Keep backups of working configurations

---

## Additional Resources

### Tools
- **IDA Pro/Ghidra**: Reverse engineering, finding code caves
- **WinDbg**: Kernel debugging
- **Process Hacker**: Process analysis, kernel object viewing
- **HxD**: Hex editor for pattern analysis
- **PE-bear**: PE file analysis, timestamp extraction
- **DebugView**: Debug output viewer

### Documentation
- Microsoft Driver Documentation: https://docs.microsoft.com/en-us/windows-hardware/drivers/
- Windows Internals Book (Russinovich): Deep driver development
- OSR Online: Driver development forum

### Related Projects
- **PdFwKrnl Mapper**: https://github.com/i32-Sudo/PdFwKrnlMapper
- **EFIGuard**: GitHub (search for PatchGuard bypass)
- **KDU**: Kernel Driver Utility (example exploits)

---

## Disclaimer

This software is provided for **educational and research purposes only**. 

⚠️ **WARNING**: 
- Using this software to cheat in online games violates Terms of Service
- May result in permanent bans
- May violate laws in your jurisdiction
- Can damage system stability
- Author assumes no liability for misuse

**Use responsibly and legally.**

---

**Version**: 1.0  
**Last Updated**: 2024  
**Author**: Based on work by bloodieys (Ezekiel Cerel)  
**Contact**: Discord - bloodieys
