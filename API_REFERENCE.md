# BEKernelDriver - API Reference

Complete API reference for the BEKernelDriver user-mode and kernel-mode interfaces.

## Table of Contents
1. [User-Mode API](#user-mode-api)
2. [Kernel-Mode Request Handlers](#kernel-mode-request-handlers)
3. [Data Structures](#data-structures)
4. [Error Codes](#error-codes)
5. [Examples](#examples)

---

## User-Mode API

### Class: `driver::communicate_t`

Main interface for communicating with the kernel driver from user-mode.

#### Constructor

```cpp
driver::communicate_t()
```

Creates a new driver communication instance.

**Example**:
```cpp
auto driver = new driver::communicate_t();
```

---

### Initialization

#### `initialize_handle()`

Opens a handle to the kernel driver device.

```cpp
bool initialize_handle()
```

**Returns**: 
- `true` - Handle opened successfully
- `false` - Failed to open handle (driver not loaded or device name mismatch)

**Example**:
```cpp
if (!driver->initialize_handle()) {
    std::cerr << "Failed to initialize driver handle\n";
    return false;
}
```

**Failure Causes**:
- Driver not loaded
- Device name mismatch between KM and UM
- Insufficient permissions (not running as Administrator)

---

### Process Operations

#### `get_process_pid()`

Retrieves the process ID of a running process by name.

```cpp
std::uint32_t get_process_pid(const std::wstring& proc_name)
```

**Parameters**:
- `proc_name` - Process name including extension (e.g., L"notepad.exe")

**Returns**:
- Process ID (> 0) on success
- `0` on failure (process not found)

**Example**:
```cpp
auto pid = driver->get_process_pid(L"GTA5.exe");
if (pid == 0) {
    std::cerr << "Process not found\n";
    return false;
}
std::cout << "Found process (PID: " << pid << ")\n";
```

**Note**: Process name is case-insensitive and must include `.exe` extension.

---

#### `attach()`

Attaches to a target process for subsequent operations.

```cpp
bool attach(int pid)
```

**Parameters**:
- `pid` - Process ID to attach to

**Returns**:
- `true` - Attached successfully
- `false` - Invalid PID

**Example**:
```cpp
if (!driver->attach(pid)) {
    std::cerr << "Failed to attach to process\n";
    return false;
}
```

**Note**: This sets the internal PID used by read/write operations. Must be called before memory operations.

---

#### `unlinkprocess()`

Hides the current process from the EPROCESS list.

```cpp
void unlinkprocess()
```

**Parameters**: None

**Returns**: None (void)

**Example**:
```cpp
driver->unlinkprocess();
// Process now hidden from Task Manager
```

**Effect**: 
- Process removed from EPROCESS linked list
- No longer visible in Task Manager
- Still visible in kernel-mode tools (Process Hacker kernel mode)
- Cannot be re-linked without reboot

**Warning**: Irreversible until reboot or kernel-mode re-linking.

---

### Module Operations

#### `get_image_base()`

Retrieves the base address of a module in the target process.

```cpp
std::uintptr_t get_image_base(const char* module_name)
```

**Parameters**:
- `module_name` - Module name (e.g., "kernel32.dll"), or `nullptr` for main module

**Returns**:
- Base address on success
- `0` on failure (module not found)

**Example - Main Module**:
```cpp
auto base = driver->get_image_base(nullptr);
if (base == 0) {
    std::cerr << "Failed to get main module base\n";
    return false;
}
std::cout << "Base: 0x" << std::hex << base << "\n";
```

**Example - Specific DLL**:
```cpp
auto kernel32_base = driver->get_image_base("kernel32.dll");
if (kernel32_base) {
    std::cout << "kernel32.dll: 0x" << std::hex << kernel32_base << "\n";
}
```

**Note**: Module name is case-insensitive. Returns main module base if name is NULL.

---

### Memory Read Operations

#### `read_virtual()` (Raw)

Reads raw bytes from target process memory.

```cpp
bool read_virtual(
    const std::uintptr_t address,
    void* buffer,
    const std::size_t size
)
```

**Parameters**:
- `address` - Virtual address to read from
- `buffer` - Output buffer
- `size` - Number of bytes to read

**Returns**:
- `true` - Read successful
- `false` - Read failed (invalid address, process, or size)

**Example**:
```cpp
uint8_t buffer[256];
if (driver->read_virtual(address, buffer, sizeof(buffer))) {
    // Process buffer data
} else {
    std::cerr << "Read failed\n";
}
```

**Limitations**:
- Address must be in user space (< 0x7FFFFFFFFFFF)
- Process must be attached
- CR3 must be resolved
- Size limited by page boundaries (4KB)

---

#### `read<T>()` (Template)

Type-safe memory read. Template method for reading typed values.

```cpp
template <typename T>
T read(const std::uintptr_t address)
```

**Template Parameters**:
- `T` - Type to read (int, float, uintptr_t, custom structs, etc.)

**Parameters**:
- `address` - Virtual address to read from

**Returns**:
- Value of type T on success
- Default-constructed T on failure (0 for primitives)

**Example - Primitives**:
```cpp
// Read integer
int health = driver->read<int>(player_base + 0x100);

// Read float
float position_x = driver->read<float>(player_base + 0x200);

// Read pointer
uintptr_t weapon_ptr = driver->read<uintptr_t>(player_base + 0x300);
```

**Example - Structures**:
```cpp
struct Vector3 {
    float x, y, z;
};

Vector3 position = driver->read<Vector3>(player_base + 0x90);
std::cout << "Position: " << position.x << ", " 
          << position.y << ", " << position.z << "\n";
```

**Note**: Preferred over raw read_virtual for type safety and convenience.

---

### Memory Write Operations

#### `write_virtual()` (Raw)

Writes raw bytes to target process memory.

```cpp
bool write_virtual(
    const std::uintptr_t address,
    void* buffer,
    const std::size_t size
)
```

**Parameters**:
- `address` - Virtual address to write to
- `buffer` - Data to write
- `size` - Number of bytes to write

**Returns**:
- `true` - Write successful
- `false` - Write failed

**Example**:
```cpp
uint8_t patch[] = {0x90, 0x90, 0x90};  // NOP instructions
if (driver->write_virtual(address, patch, sizeof(patch))) {
    std::cout << "Patch applied\n";
} else {
    std::cerr << "Write failed\n";
}
```

---

#### `write<T>()` (Template)

Type-safe memory write. Template method for writing typed values.

```cpp
template <typename T>
bool write(const std::uintptr_t address, T value)
```

**Template Parameters**:
- `T` - Type to write

**Parameters**:
- `address` - Virtual address to write to
- `value` - Value to write

**Returns**:
- `true` - Write successful
- `false` - Write failed

**Example - Primitives**:
```cpp
// Write integer
driver->write<int>(player_base + 0x100, 999);  // Set health

// Write float
driver->write<float>(player_base + 0x200, 123.45f);  // Set position

// Write pointer
driver->write<uintptr_t>(player_base + 0x300, new_weapon_ptr);
```

**Example - Structures**:
```cpp
Vector3 new_position = {100.0f, 200.0f, 50.0f};
driver->write<Vector3>(player_base + 0x90, new_position);
```

---

### CR3/DTB Operations

#### `get_cr3()`

Resolves the CR3/DTB (Directory Table Base) for the target process.

```cpp
bool get_cr3(std::uintptr_t base_address)
```

**Parameters**:
- `base_address` - Main module base address of target process

**Returns**:
- `true` - CR3 resolved successfully
- `false` - Resolution failed

**Example**:
```cpp
auto base = driver->get_image_base(nullptr);
if (!driver->get_cr3(base)) {
    std::cerr << "CR3 resolution failed\n";
    return false;
}
// driver->dtb now contains the resolved CR3
std::cout << "CR3: 0x" << std::hex << driver->dtb << "\n";
```

**Algorithm**:
1. Gets current process DTB (known reference)
2. Translates ntdll.dll address using current DTB
3. Iterates possible DTB values (0 to 0x50000000, step 0x1000)
4. For each candidate:
   - Translates ntdll using candidate DTB
   - Compares physical addresses
   - Verifies by reading MZ header
5. Saves matching DTB to `driver->dtb`

**Important**: Must be called before read/write operations. CR3 may need to be re-resolved if target process restarts or after long periods (especially with EAC).

---

#### `get_dtb()`

Gets the DTB for any process by PID.

```cpp
std::uintptr_t get_dtb(std::uint32_t pid)
```

**Parameters**:
- `pid` - Process ID

**Returns**:
- DTB value on success
- `0` on failure

**Example**:
```cpp
auto current_dtb = driver->get_dtb(GetCurrentProcessId());
std::cout << "Current process DTB: 0x" << std::hex << current_dtb << "\n";

auto target_dtb = driver->get_dtb(target_pid);
std::cout << "Target process DTB: 0x" << std::hex << target_dtb << "\n";
```

**Note**: Reads directly from EPROCESS structure (offset 0x28 for x64 or 0x388 for x32).

---

#### `translate_address()`

Translates a virtual address to physical address using specified DTB.

```cpp
std::uintptr_t translate_address(
    std::uintptr_t virtual_address,
    std::uintptr_t directory_base
)
```

**Parameters**:
- `virtual_address` - Virtual address to translate
- `directory_base` - DTB to use for translation

**Returns**:
- Physical address on success
- `0` on failure (invalid address or DTB)

**Example**:
```cpp
auto physical = driver->translate_address(virtual_addr, driver->dtb);
if (physical) {
    std::cout << "Virtual 0x" << std::hex << virtual_addr 
              << " -> Physical 0x" << physical << "\n";
} else {
    std::cerr << "Translation failed\n";
}
```

**Use Cases**:
- Manual physical memory access
- Debugging page tables
- Validating address mappings
- Understanding memory layout

---

### Kernel Memory Operations

#### `read_kernel()`

Reads from kernel memory (physical or virtual).

```cpp
bool read_kernel(
    const uintptr_t address,
    void* buffer,
    const size_t size,
    const uint32_t memory_type
)
```

**Parameters**:
- `address` - Kernel address to read from
- `buffer` - Output buffer
- `size` - Number of bytes to read
- `memory_type` - `MM_COPY_MEMORY_PHYSICAL` or `MM_COPY_MEMORY_VIRTUAL`

**Returns**:
- `true` - Read successful
- `false` - Read failed

**Example - Read Kernel Virtual**:
```cpp
uint8_t kernel_data[64];
if (driver->read_kernel(
    kernel_addr,
    kernel_data,
    sizeof(kernel_data),
    MM_COPY_MEMORY_VIRTUAL
)) {
    // Process kernel data
}
```

**Example - Read Kernel Physical**:
```cpp
uint64_t phys_value;
driver->read_kernel(
    physical_addr,
    &phys_value,
    sizeof(phys_value),
    MM_COPY_MEMORY_PHYSICAL
);
```

**Warning**: Reading kernel memory incorrectly can cause BSOD. Validate addresses carefully.

---

### Memory Allocation

#### `virtual_protect()`

Allocates or changes protection of virtual memory in target process.

```cpp
bool virtual_protect(
    const std::uintptr_t address,
    const size_t size,
    const DWORD allocation_type
)
```

**Parameters**:
- `address` - Base address (0 for new allocation)
- `size` - Size in bytes
- `allocation_type` - Allocation flags (MEM_COMMIT, MEM_RESERVE, etc.)

**Returns**:
- `true` - Operation successful
- `false` - Operation failed

**Example - Allocate New Memory**:
```cpp
if (driver->virtual_protect(
    0,  // Let system choose address
    0x1000,  // 4KB
    MEM_COMMIT | MEM_RESERVE
)) {
    std::cout << "Memory allocated\n";
}
```

**Example - Change Protection**:
```cpp
// Make region executable
driver->virtual_protect(
    address,
    size,
    PAGE_EXECUTE_READWRITE
);
```

---

### Low-Level Communication

#### `send_cmd()`

Sends a raw command to the kernel driver.

```cpp
bool send_cmd(void* data, requests code)
```

**Parameters**:
- `data` - Pointer to request-specific data structure
- `code` - Request code (from `requests` enum)

**Returns**:
- `true` - Command sent and executed successfully
- `false` - Command failed

**Example**:
```cpp
read_invoke data = {0};
data.pid = target_pid;
data.dtb = driver->dtb;
data.address = target_address;
data.buffer = output_buffer;
data.size = bytes_to_read;

if (driver->send_cmd(&data, invoke_read)) {
    // Read successful
}
```

**Note**: This is a low-level function. Prefer using the high-level methods (read, write, etc.) instead.

---

## Kernel-Mode Request Handlers

### Request Dispatcher

All requests are dispatched through `vortex::io_dispatch()` in the kernel driver.

**Request Flow**:
```
User-mode → DeviceIoControl → io_dispatch → Handler → Response
```

### Request Codes

Defined in `requests` enum:

```cpp
enum _requests {
    invoke_start,           // Reserved
    invoke_base,           // Get module base
    invoke_read,           // Read memory
    invoke_write,          // Write memory
    invoke_success,        // Reserved
    invoke_unique,         // Validation token
    invoke_translate,      // Translate address
    invoke_read_kernel,    // Read kernel memory
    invoke_dtb,           // Get process DTB
    invoke_protect_virtual, // Allocate/protect memory
    invoke_hideproc        // Hide process
};
```

---

### Handler: `invoke_base`

**Purpose**: Get module base address

**Handler**: `request::get_module_base()`

**Input Structure**: `base_invoke`
```cpp
struct base_invoke {
    uint32_t pid;           // Process ID
    uintptr_t handle;       // [OUT] Module base
    const char* name;       // Module name (NULL for main)
    size_t size;           // [Reserved]
};
```

**Process**:
1. Validates PID
2. Looks up PEPROCESS
3. If name is NULL: Returns main module via `PsGetProcessSectionBaseAddress()`
4. If name provided: Enumerates PEB LDR list, finds matching module
5. Returns base address in `handle` field

---

### Handler: `invoke_read`

**Purpose**: Read process memory (CR3-based)

**Handler**: `request::read_memory()`

**Input Structure**: `read_invoke`
```cpp
struct read_invoke {
    uint32_t pid;          // Process ID
    uintptr_t address;     // Virtual address
    uintptr_t dtb;         // Directory table base
    void* buffer;          // Output buffer
    size_t size;           // Bytes to read
};
```

**Process**:
1. Validates parameters (address < 0x7FFFFFFFFFFF, size > 0)
2. Looks up PEPROCESS by PID
3. Translates virtual to physical using DTB
4. Reads from physical memory via `MmCopyMemory()`
5. Handles page boundary conditions
6. Copies data to user buffer

---

### Handler: `invoke_write`

**Purpose**: Write process memory

**Handler**: `request::write_memory()`

**Input Structure**: `write_invoke`
```cpp
struct write_invoke {
    uint32_t pid;          // Process ID
    uintptr_t address;     // Virtual address
    void* buffer;          // Data to write
    size_t size;           // Bytes to write
};
```

**Process**:
1. Validates parameters
2. Looks up PEPROCESS
3. Uses `MmCopyVirtualMemory()` to write
4. Verifies bytes written

**Note**: Uses virtual memory copy (not physical like read).

---

### Handler: `invoke_translate`

**Purpose**: Translate virtual address to physical

**Handler**: `request::translate_address()`

**Input Structure**: `translate_invoke`
```cpp
struct translate_invoke {
    uintptr_t virtual_address;    // Input
    uintptr_t directory_base;     // DTB to use
    void* physical_address;       // [OUT] Result
};
```

**Process**:
1. Validates inputs
2. Calls `translate_linear()` with DTB
3. Implements 4-level x64 paging
4. Handles 1GB/2MB large pages
5. Returns physical address

---

### Handler: `invoke_dtb`

**Purpose**: Get process DTB

**Handler**: `request::get_dtb()`

**Input Structure**: `dtb_invoke`
```cpp
struct dtb_invoke {
    uint32_t pid;          // Process ID
    uintptr_t dtb;         // [OUT] Directory table base
};
```

**Process**:
1. Looks up PEPROCESS by PID
2. Reads EPROCESS+0x28 (x64 DTB offset)
3. Fallback to EPROCESS+0x388 (x32 offset)
4. Returns DTB value

---

### Handler: `invoke_read_kernel`

**Purpose**: Read kernel memory

**Handler**: `request::mm_copy_kernel()`

**Input Structure**: `read_kernel_invoke`
```cpp
struct read_kernel_invoke {
    uintptr_t address;     // Kernel address
    void* buffer;          // Output buffer
    size_t size;           // Bytes to read
    uint32_t memory_type;  // PHYSICAL or VIRTUAL
};
```

**Process**:
1. Validates inputs
2. Routes to physical or virtual read based on `memory_type`
3. Uses `MmCopyMemory()` with appropriate flags
4. Returns data to user buffer

---

### Handler: `invoke_protect_virtual`

**Purpose**: Allocate or protect virtual memory

**Handler**: `request::virtual_allocate()`

**Input Structure**: `allocate_invoke`
```cpp
struct allocate_invoke {
    uintptr_t address;        // Base address
    SIZE_T dwSize;           // Size
    DWORD flAllocationType;  // Allocation type
};
```

**Process**:
1. Validates parameters
2. Calls kernel virtual memory functions
3. Allocates or modifies protection
4. Returns result

---

### Handler: `invoke_hideproc`

**Purpose**: Hide process from EPROCESS list

**Handler**: `UnlinkProcess()`

**Input Structure**: None (uses current process)

**Process**:
1. Calls `InitializeOffsets()` to get EPROCESS structure offsets
2. Calls `HideProcess()` to unlink from ActiveProcessLinks
3. Process becomes hidden from user-mode enumeration

---

## Data Structures

### Main Request Wrapper

```cpp
struct invoke_data {
    uint32_t unique;      // Must be invoke_unique for validation
    requests code;        // Request type (from enum)
    void* data;          // Pointer to specific request structure
};
```

**Usage**:
```cpp
invoke_data request;
request.unique = invoke_unique;
request.code = invoke_read;
request.data = &read_data;
```

---

### CR3 Manager (User Implementation)

Example implementation for managing CR3 stability:

```cpp
class CR3Manager {
private:
    driver::communicate_t* driver;
    uintptr_t cached_cr3;
    uintptr_t base_address;
    std::chrono::steady_clock::time_point last_validation;
    const int VALIDATION_INTERVAL_MS = 5000;

public:
    bool initialize();
    bool refresh_cr3();
    bool validate_and_refresh();
};
```

---

## Error Codes

### Driver Status Codes

Defined in kernel driver:

```cpp
namespace driver {
    enum status {
        successful_operation = 0,
        failed_sanity_check = 1,
        failed_intialization = 2
    };
}
```

### NTSTATUS Codes

Standard Windows NT status codes:
- `STATUS_SUCCESS` (0x00000000) - Operation succeeded
- `STATUS_UNSUCCESSFUL` (0xC0000001) - Generic failure
- `STATUS_ACCESS_DENIED` (0xC0000022) - Permission denied
- `STATUS_NOT_FOUND` (0xC0000225) - Object not found

### User-Mode Return Values

- `true` / non-zero - Success
- `false` / 0 - Failure

---

## Examples

### Complete Initialization

```cpp
#include "driver_api.h"

bool initialize_driver(uint32_t& pid, uintptr_t& base) {
    // Create driver instance
    auto driver = new driver::communicate_t();
    
    // Open handle
    if (!driver->initialize_handle()) {
        std::cerr << "Failed to open driver handle\n";
        return false;
    }
    std::cout << "[+] Driver handle opened\n";
    
    // Hide current process
    driver->unlinkprocess();
    std::cout << "[+] Process hidden\n";
    
    // Find target process
    pid = driver->get_process_pid(L"target.exe");
    if (pid == 0) {
        std::cerr << "Target process not found\n";
        return false;
    }
    std::cout << "[+] Found target (PID: " << pid << ")\n";
    
    // Attach
    if (!driver->attach(pid)) {
        std::cerr << "Failed to attach\n";
        return false;
    }
    
    // Get base address
    base = driver->get_image_base(nullptr);
    if (base == 0) {
        std::cerr << "Failed to get base address\n";
        return false;
    }
    std::cout << "[+] Base: 0x" << std::hex << base << "\n";
    
    // Resolve CR3
    if (!driver->get_cr3(base)) {
        std::cerr << "CR3 resolution failed\n";
        return false;
    }
    std::cout << "[+] CR3: 0x" << std::hex << driver->dtb << "\n";
    
    return true;
}
```

### Read/Write Operations

```cpp
void memory_operations_example(driver::communicate_t* driver, uintptr_t base) {
    // Read integer
    int health = driver->read<int>(base + 0x100);
    std::cout << "Health: " << health << "\n";
    
    // Write integer
    driver->write<int>(base + 0x100, 999);
    std::cout << "Health set to 999\n";
    
    // Read float
    float speed = driver->read<float>(base + 0x200);
    std::cout << "Speed: " << speed << "\n";
    
    // Read structure
    struct Vector3 { float x, y, z; };
    Vector3 pos = driver->read<Vector3>(base + 0x300);
    std::cout << "Position: (" << pos.x << ", " 
              << pos.y << ", " << pos.z << ")\n";
    
    // Write structure
    Vector3 new_pos = {100.0f, 200.0f, 50.0f};
    driver->write<Vector3>(base + 0x300, new_pos);
    
    // Read buffer
    uint8_t buffer[256];
    if (driver->read_virtual(base + 0x400, buffer, sizeof(buffer))) {
        std::cout << "Read 256 bytes successfully\n";
    }
    
    // Write buffer (patch)
    uint8_t patch[] = {0x90, 0x90, 0x90};  // NOPs
    driver->write_virtual(base + 0x500, patch, sizeof(patch));
}
```

### Module Enumeration

```cpp
void enumerate_modules(driver::communicate_t* driver) {
    const char* modules[] = {
        nullptr,           // Main module
        "kernel32.dll",
        "ntdll.dll",
        "user32.dll"
    };
    
    for (auto mod_name : modules) {
        auto base = driver->get_image_base(mod_name);
        if (base) {
            std::cout << (mod_name ? mod_name : "Main module") 
                     << ": 0x" << std::hex << base << "\n";
        } else {
            std::cout << (mod_name ? mod_name : "Main module") 
                     << ": NOT FOUND\n";
        }
    }
}
```

### CR3 Management

```cpp
class CR3Manager {
private:
    driver::communicate_t* driver;
    uintptr_t base_address;
    
public:
    CR3Manager(driver::communicate_t* drv, uintptr_t base) 
        : driver(drv), base_address(base) {}
    
    bool validate() {
        // Test if CR3 still valid
        auto test = driver->read<uint16_t>(base_address);
        if (test != 0x5A4D) {  // MZ header
            // CR3 invalid, refresh
            return driver->get_cr3(base_address);
        }
        return true;
    }
};

// Usage
CR3Manager cr3(driver, base);
while (true) {
    if (!cr3.validate()) {
        std::cerr << "CR3 validation failed\n";
        break;
    }
    
    // Perform operations
    auto value = driver->read<int>(some_address);
    
    Sleep(100);
}
```

### Error Handling

```cpp
bool safe_read(driver::communicate_t* driver, uintptr_t addr, int& out) {
    try {
        out = driver->read<int>(addr);
        
        // Validate result
        if (addr < 0x10000) {  // Suspicious low address
            std::cerr << "Warning: Reading from low address\n";
            return false;
        }
        
        return true;
    } catch (...) {
        std::cerr << "Exception during read\n";
        return false;
    }
}

// Usage
int value;
if (safe_read(driver, address, value)) {
    std::cout << "Value: " << value << "\n";
} else {
    std::cerr << "Read failed\n";
}
```

---

## Performance Considerations

### Optimization Tips

1. **Batch Operations**: Minimize kernel transitions
```cpp
// Bad: Multiple small reads
for (int i = 0; i < 100; i++) {
    int val = driver->read<int>(base + i * 4);
}

// Good: Single large read
uint8_t buffer[400];
driver->read_virtual(base, buffer, sizeof(buffer));
// Parse buffer in user-mode
```

2. **Cache Results**: Don't re-read static values
```cpp
// Cache module bases
static uintptr_t game_base = 0;
if (game_base == 0) {
    game_base = driver->get_image_base(nullptr);
}
```

3. **Validate Once**: Don't validate on every operation
```cpp
// Validate CR3 periodically, not every read
if (time_since_last_validation > 5000ms) {
    cr3_manager.validate();
}
```

4. **Use Templates**: Type-safe and compiler-optimized
```cpp
// Preferred
int value = driver->read<int>(addr);

// Over
int value;
driver->read_virtual(addr, &value, sizeof(int));
```

---

## Thread Safety

**Warning**: The driver communication API is **not thread-safe**.

**Guidelines**:
- Use from single thread only, OR
- Implement mutex/lock around driver operations
- Don't call send_cmd() concurrently

**Example - Thread-Safe Wrapper**:
```cpp
class ThreadSafeDriver {
private:
    driver::communicate_t* driver;
    std::mutex mtx;
    
public:
    template<typename T>
    T read(uintptr_t addr) {
        std::lock_guard<std::mutex> lock(mtx);
        return driver->read<T>(addr);
    }
    
    template<typename T>
    bool write(uintptr_t addr, T value) {
        std::lock_guard<std::mutex> lock(mtx);
        return driver->write<T>(addr, value);
    }
};
```

---

## Debugging

### Enable Debug Output

In kernel driver:
```cpp
// Uncomment DbgPrintEx calls
DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "Message: %d\n", value);
```

View with DebugView (run as Administrator with kernel capture enabled).

### User-Mode Debugging

```cpp
#ifdef _DEBUG
    #define DEBUG_LOG(msg) std::cout << "[DEBUG] " << msg << "\n"
#else
    #define DEBUG_LOG(msg)
#endif

// Usage
DEBUG_LOG("About to read from: 0x" << std::hex << address);
auto value = driver->read<int>(address);
DEBUG_LOG("Read value: " << value);
```

### Common Issues

| Symptom | Likely Cause | Debug Steps |
|---------|--------------|-------------|
| Read returns 0 | Invalid CR3 or address | Verify CR3, check address validity |
| BSOD on operation | Bad kernel pointer | Check kernel debugger, validate inputs |
| Handle open fails | Driver not loaded | Check DebugView, verify device name |
| Process not found | Wrong name or not running | List processes, verify exact name |

---

**Last Updated**: 2024  
**Version**: 1.0  
**See Also**: [DOCUMENTATION.md](DOCUMENTATION.md), [DEPLOYMENT.md](DEPLOYMENT.md)
