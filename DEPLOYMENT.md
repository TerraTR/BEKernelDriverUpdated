# BEKernelDriver - Deployment Guide

## Quick Start

This guide provides step-by-step instructions for deploying the BEKernelDriver in a testing environment.

⚠️ **WARNING**: This guide is for educational and testing purposes only. Follow all applicable laws and Terms of Service.

---

## Prerequisites Checklist

### Software
- [ ] Windows 10/11 x64 (VM or dedicated test machine recommended)
- [ ] Visual Studio 2019 or 2022 with C++ Desktop Development
- [ ] Windows SDK 10.0.19041.0 or newer
- [ ] Windows Driver Kit (WDK) matching SDK version
- [ ] PdFwKrnl Mapper or equivalent vulnerable driver mapper
- [ ] PatchGuard bypass (e.g., EFIGuard, DSEFix)
- [ ] Code protector (VMProtect/Themida) - optional but recommended

### System Configuration
- [ ] Test signing enabled OR signature verification disabled
- [ ] PatchGuard disabled
- [ ] Secure Boot disabled (if using UEFI bypass)
- [ ] Antivirus disabled (in test environment only)

### Knowledge Requirements
- [ ] Basic C++ programming
- [ ] Understanding of Windows kernel concepts
- [ ] Familiarity with debuggers (WinDbg/x64dbg)
- [ ] Understanding of anti-cheat systems

---

## Step-by-Step Deployment

### Step 1: System Preparation

#### 1.1 Enable Test Signing
```cmd
REM Open Administrator Command Prompt
bcdedit /set testsigning on

REM Reboot
shutdown /r /t 0
```

Verify test signing is enabled:
- Boot Windows
- Look for "Test Mode" watermark in bottom-right corner

#### 1.2 Install PatchGuard Bypass

**Option A: EFIGuard (Recommended)**
```
1. Download EFIGuard from GitHub
2. Run EFIGuard installer
3. Configure UEFI boot entry
4. Reboot and select EFIGuard entry
5. Windows will boot with PatchGuard disabled
```

**Option B: DSEFix**
```
1. Download DSEFix
2. Run DSEFix loader
3. Follow on-screen instructions
4. Reboot with F8 and select DSE disabled
```

**Verify PatchGuard is Disabled**:
```
No immediate way to verify, but:
- If PatchGuard is active, you'll get BSOD on driver load
- If no BSOD, PatchGuard is likely bypassed
```

#### 1.3 Disable Secure Boot (if applicable)
```
1. Reboot to UEFI firmware settings
2. Navigate to Security settings
3. Disable Secure Boot
4. Save and exit
```

---

### Step 2: Build the Driver

#### 2.1 Clone Repository
```bash
git clone https://github.com/TerraTR/BEKernelDriverUpdated
cd BEKernelDriverUpdated
```

#### 2.2 Configure Project Settings

Open `KrnlUpdated.sln` in Visual Studio.

**For driver-execute project**:
1. Right-click project → Properties
2. Configuration: Release, Platform: x64
3. Driver Settings → Target OS Version: Windows 10
4. C/C++ → General → Warning Level: Level 3
5. Linker → General → Enable Incremental Linking: No

#### 2.3 Make Required Configuration Changes

See [DOCUMENTATION.md - Configuration](DOCUMENTATION.md#configuration) section for detailed instructions.

**Minimum required changes**:

1. **Device Names** (`KM/impl/communication/interface.h`):
```cpp
#define DEVICE_NAME L"\\Device\\MyUniqueDevice"
#define DOS_NAME L"\\DosDevices\\MyUniqueDevice"
```

2. **Hook Target** (`KM/entry/hook/hook.hpp`):
```cpp
// Line 89
const auto target_module = modules::get_kernel_module(_("beep.sys"));

// Lines 96-98 - Find signature using IDA
globals::cave_base = modules::find_pattern(target_module,
    _("\x48\x89\x5C\x24\x08"),  // Example
    _("xxxxx"));
```

3. **Process Name** (`KM/processhyde/Hide.cpp`):
```cpp
const char target[] = "myapp.exe";  // Your UM executable name
```

4. **Mapper Cleanup** (`KM/entry/main.cpp`):
```cpp
CleanDriverSys(UNICODE_STRING(RTL_CONSTANT_STRING(L"PdFwKrnl.sys")), 0x611AB60D);
// Add your mapper's details
```

#### 2.4 Build Driver
```
1. Set solution configuration to Release x64
2. Right-click driver-execute project
3. Select Build
4. Output: x64/Release/driver-execute.sys
```

#### 2.5 Build User-Mode Application
```
1. Right-click um project
2. Select Build
3. Output: x64/Release/um.exe
```

#### 2.6 Post-Build Protection (Optional)

**VMProtect**:
```
1. Load um.exe in VMProtect
2. Add all functions to protection list
3. Enable: Code Virtualization, Mutation
4. Process and save protected file
```

---

### Step 3: Prepare Mapper

#### 3.1 Download PdFwKrnl Mapper
```bash
git clone https://github.com/i32-Sudo/PdFwKrnlMapper
cd PdFwKrnlMapper
# Follow mapper's build instructions
```

#### 3.2 Prepare Mapper Files

Ensure you have:
- PdFwKrnlMapper.exe (mapper executable)
- vulnerable_driver.sys (exploit driver)
- Your built driver-execute.sys

Place all in same directory:
```
Deployment/
├── PdFwKrnlMapper.exe
├── vulnerable_driver.sys
├── driver-execute.sys
└── um.exe
```

---

### Step 4: Deploy Driver

#### 4.1 Pre-Flight Checks

Run checklist:
- [ ] Test signing enabled (Test Mode visible)
- [ ] PatchGuard bypass active
- [ ] Antivirus disabled
- [ ] Running as Administrator
- [ ] No anti-cheat already running
- [ ] Mapper and driver in same folder

#### 4.2 Map Driver

```cmd
REM Open Administrator Command Prompt
cd C:\Path\To\Deployment

REM Run mapper
PdFwKrnlMapper.exe driver-execute.sys

REM Expected output:
[+] Loading vulnerable driver...
[+] Exploiting vulnerability...
[+] Mapping driver...
[+] Resolving imports...
[+] Calling entry point...
[+] Driver loaded successfully
[+] Cleaning up...
[+] Done
```

#### 4.3 Verify Driver Loaded

**Check for device**:
```cmd
REM Won't show in standard tools due to hook

REM Use Process Hacker (Kernel mode):
1. Open Process Hacker as Admin
2. Hacker → Handles
3. Filter for your device name
4. Should appear if device created
```

**Check debug output** (if enabled):
```
1. Download DebugView from Sysinternals
2. Run as Administrator
3. Enable Capture Kernel
4. Look for your DbgPrintEx messages
```

**Common Issues**:

| Symptom | Cause | Solution |
|---------|-------|----------|
| BSOD immediately | PatchGuard active | Verify bypass is working |
| "Failed to load" | Signature mismatch | Verify test signing enabled |
| No output | Driver loaded but hook failed | Check target module exists |
| Access denied | Not Administrator | Run CMD as Admin |

---

### Step 5: Run User-Mode Application

#### 5.1 Start Target Application

For testing with Escape from Tarkov:
```
1. Start Escape from Tarkov
2. Wait until fully loaded (in menu)
3. Verify process is running (Task Manager)
```

#### 5.2 Run UM Application

```cmd
cd C:\Path\To\Deployment
um.exe

REM Expected output:
Press F5 Once in menu...
```

#### 5.3 Initialize Connection

1. Press F5 when prompted
2. Application will:
   - Find target process
   - Attach to process
   - Resolve CR3/DTB
   - Begin operations

**Verification**:
```
- No crash = successful initialization
- Process should disappear from Task Manager (if hiding enabled)
- No error messages in console
```

#### 5.4 Troubleshooting UM

| Issue | Solution |
|-------|----------|
| "Cannot open handle" | Driver not loaded or device name mismatch |
| "Process not found" | Target not running or name mismatch |
| "CR3 resolution failed" | Process not fully initialized, try again |
| Immediate crash | Access violation, check addresses |

---

### Step 6: Verification

#### 6.1 Verify Process Hiding

```
Before: 
Task Manager → Details → your um.exe visible

After unlinkprocess():
Task Manager → Details → your um.exe NOT visible

BUT: Process Hacker kernel mode → still visible
```

#### 6.2 Verify Memory Access

Add test code to UM:
```cpp
// After successful initialization
auto base = request->get_image_base(nullptr);
auto mz_header = request->read<uint16_t>(base);

std::cout << "Base: 0x" << std::hex << base << "\n";
std::cout << "MZ Header: 0x" << std::hex << mz_header << "\n";

if (mz_header == 0x5A4D) {
    std::cout << "[+] Memory read successful!\n";
} else {
    std::cout << "[!] Memory read failed!\n";
}
```

#### 6.3 Monitor Stability

Run for 5-10 minutes:
- [ ] No BSOD
- [ ] No crashes
- [ ] Memory operations work consistently
- [ ] CR3 remains valid
- [ ] No unusual system behavior

---

### Step 7: Cleanup

#### 7.1 Stop User Application
```
Close console window or exit application
```

#### 7.2 Unload Driver (if possible)

**Note**: This driver has no unload routine (`DriverUnload = nullptr`)

**Force unload** (risky, may BSOD):
```
Not recommended - reboot instead
```

**Safe cleanup**:
```
1. Close all applications using driver
2. Reboot system
3. Driver will not persist across reboot (not installed as service)
```

#### 7.3 Remove Mapper Artifacts

```cmd
REM Delete temporary files
del vulnerable_driver.sys
del PdFwKrnlMapper.exe

REM Clean registry (if mapper creates entries)
REM Varies by mapper
```

---

## Production Deployment Considerations

⚠️ This section is for informational purposes only. Production deployment of such tools may violate laws and ToS.

### Distribution

**DO NOT**:
- Distribute with default configurations
- Include source code
- Use public file hosting
- Share on forums/Discord

**IF DISTRIBUTING** (educational context):
- Provide only compiled binaries
- Require users to configure themselves
- Include disclaimers and legal notices
- Verify license compliance (CC BY-NC-ND 4.0)

### Configuration

Users must configure:
1. Device names
2. Hook targets (unique to each user)
3. Process names
4. Pool tags

**Never include**:
- Pre-configured targets
- Ready-to-run packages
- Detailed cheat features
- Server-specific exploits

### Updates

With game/AC updates, you must:
1. Re-find signatures
2. Update offsets
3. Test thoroughly
4. Verify AC hasn't detected patterns

### Detection

Assume detection is possible:
- Public code is analyzed
- Signatures are collected
- Behaviors are profiled
- Network patterns detected

**Mitigation**:
- Unique modifications per user
- Code obfuscation
- Behavioral randomization
- Private use only

---

## Testing Environments

### Recommended Setup

**Option 1: Virtual Machine**
```
Host: Windows 10/11
VM: VMware Workstation / VirtualBox
Guest: Windows 10 x64
Config: 
- 4GB+ RAM
- 2+ CPU cores
- 60GB+ disk
- Network: NAT or Bridged
```

**Benefits**:
- Isolated from main system
- Easy snapshot/restore
- Multiple test configurations
- Safe experimentation

**Limitations**:
- Some games detect VMs
- Performance overhead
- May not support nested virtualization

**Option 2: Dedicated Test Machine**
```
Hardware: Spare PC or laptop
OS: Fresh Windows 10 x64 install
Network: Isolated or test network
Storage: SSD recommended
```

**Benefits**:
- Real hardware
- No VM detection
- Full performance
- Accurate testing

**Limitations**:
- Requires physical hardware
- Harder to restore
- Risk of hardware issues

**Option 3: Dual Boot**
```
Main OS: Your daily driver
Test OS: Separate Windows partition
Boot Manager: Windows Boot Manager
```

**Benefits**:
- Real hardware
- Isolated from main OS
- Easy to maintain

**Limitations**:
- Requires disk space
- Must reboot to switch
- Shared hardware

### Test Cases

#### Basic Functionality
- [ ] Driver loads without BSOD
- [ ] Device handle opens
- [ ] Process enumeration works
- [ ] Module enumeration works
- [ ] CR3 resolution succeeds
- [ ] Memory read returns correct values
- [ ] Memory write takes effect
- [ ] Process hiding works

#### Stability
- [ ] Runs for 10+ minutes without crash
- [ ] Multiple read/write cycles
- [ ] CR3 remains valid
- [ ] No memory leaks
- [ ] Clean shutdown

#### Edge Cases
- [ ] Target process not running
- [ ] Target process closes during operation
- [ ] Invalid memory addresses
- [ ] Large read/write operations
- [ ] Rapid read/write cycles

#### Security
- [ ] Process not visible in Task Manager
- [ ] Module not in PEB lists
- [ ] No debug flags set
- [ ] Driver traces cleaned
- [ ] No suspicious handles

---

## Troubleshooting Guide

### Driver Load Failures

#### BSOD: KERNEL_SECURITY_CHECK_FAILURE
```
Cause: PatchGuard detected modification
Solution: Verify PatchGuard bypass is active
Steps:
1. Reboot with bypass enabled
2. Check bypass status
3. Try again
```

#### BSOD: DRIVER_IRQL_NOT_LESS_OR_EQUAL
```
Cause: IRQL violation in driver code
Solution: Code issue, review driver
Steps:
1. Check kernel debugger output
2. Identify violating code
3. Fix and rebuild
```

#### Mapper Error: "Failed to exploit"
```
Cause: Vulnerable driver already patched
Solution: Use different exploit or mapper
Steps:
1. Check Windows update status
2. Try alternative mapper
3. Use different vulnerable driver
```

### User-Mode Issues

#### "Cannot open handle"
```
Cause: Device name mismatch or driver not loaded
Solution: 
1. Verify driver loaded (check debug output)
2. Check device name matches in:
   - interface.h (KM)
   - driver.cpp (UM)
3. Ensure exact match including case
```

#### "Process not found"
```
Cause: Target process not running or name mismatch
Solution:
1. Verify process running (Task Manager)
2. Check exact process name including .exe
3. Case-sensitive match required
4. Check for (32) suffix on 32-bit processes
```

#### "CR3 resolution failed"
```
Cause: Process not fully initialized or memory read failed
Solution:
1. Wait 5-10 seconds after process starts
2. Increase search range (edit driver.cpp line 238)
3. Verify ntdll.dll loaded in target
4. Check memory read works (test with base address)
```

### Runtime Issues

#### Reads return zeros or garbage
```
Cause: Invalid CR3 or bad address
Solution:
1. Re-run get_cr3()
2. Validate address is in user space (< 0x7FFFFFFFFFFF)
3. Check target process didn't restart
4. Verify process PID didn't change
```

#### Random crashes
```
Cause: Access violation or invalid pointer
Solution:
1. Validate all addresses before access
2. Check for NULL pointers
3. Verify process attachment
4. Add error handling
```

#### BSOD after some time
```
Cause: CR3 invalidated (common with EAC)
Solution:
1. Implement CR3 re-validation
2. Catch read errors
3. Re-cache CR3 on failure
4. Add retry logic
```

---

## Emergency Procedures

### If System Becomes Unstable

1. **Immediate Actions**:
   ```
   - Close all applications
   - Save any open work
   - Prepare for forced reboot
   ```

2. **Safe Mode Boot**:
   ```
   1. Force shutdown (hold power button)
   2. Power on and tap F8 repeatedly
   3. Select Safe Mode
   4. Remove driver files
   5. Reboot normally
   ```

3. **System Restore**:
   ```
   1. Boot to Safe Mode
   2. Control Panel → Recovery
   3. Open System Restore
   4. Select restore point before driver installation
   5. Follow wizard
   ```

### If Driver Won't Unload

```
Driver has no unload routine, so:

1. Reboot system (cleanest method)
2. Driver won't persist (not a service)
3. If BSOD on boot:
   a. Boot to Safe Mode
   b. Check for startup items
   c. Remove any auto-load mechanisms
```

### If Banned from Game

```
This is expected if detected:

1. Accept the ban
2. Don't attempt to evade
3. Review what was detected
4. Improve stealth for future
5. Use test accounts only
```

---

## Best Practices

### Before Deployment
- [ ] Test in VM first
- [ ] Verify all configurations
- [ ] Remove debug output
- [ ] Apply code protection
- [ ] Create system restore point

### During Operation
- [ ] Monitor system resources
- [ ] Watch for error messages
- [ ] Keep debugger attached (WinDbg)
- [ ] Log operations
- [ ] Be ready to force shutdown

### After Deployment
- [ ] Check system stability
- [ ] Review logs
- [ ] Clean temporary files
- [ ] Restore system settings
- [ ] Document any issues

### General Safety
- [ ] Use test accounts
- [ ] Never on production systems
- [ ] Isolated test environment
- [ ] No sensitive data on test machine
- [ ] Regular backups
- [ ] Have recovery plan ready

---

## Support

### Getting Help

1. **Documentation**:
   - Read DOCUMENTATION.md thoroughly
   - Check FIVEM_PORTING_GUIDE.md if applicable
   - Review code comments

2. **Community**:
   - UnknownCheats forums (educational)
   - GitHub issues (technical)
   - Discord: bloodieys (author)

3. **Self-Debugging**:
   - Use WinDbg for kernel debugging
   - Check DebugView for driver output
   - Use x64dbg for UM debugging
   - Enable logging and review

### Reporting Issues

When reporting problems, include:
- Windows version (build number)
- Driver configuration used
- Exact error messages
- Steps to reproduce
- WinDbg output (if BSOD)
- Whether test signing/PatchGuard bypass active

### What NOT to Report

- "How do I cheat in [game]?" - Not supported
- "Can you add [cheat feature]?" - DIY
- "Why was I banned?" - Expected behavior
- "Can you configure for me?" - Security risk

---

## Legal and Ethical Considerations

### Legal Disclaimer

This software is provided for **educational purposes only**.

**Using this software may**:
- Violate Terms of Service
- Result in permanent bans
- Violate laws in your jurisdiction
- Expose you to liability

**You are responsible for**:
- Complying with all laws
- Understanding risks
- Accepting consequences
- Not harming others

### Ethical Usage

**DO**:
- Learn about security mechanisms
- Test on your own servers/offline
- Respect others' experiences
- Use for education/research
- Report vulnerabilities responsibly

**DON'T**:
- Cheat in competitive games
- Grief other players
- Distribute maliciously
- Violate ToS knowingly
- Claim ignorance of rules

### Responsible Disclosure

If you discover security vulnerabilities:
1. Report to affected parties
2. Give time to fix (90 days standard)
3. Don't exploit in production
4. Document responsibly
5. Consider coordinated disclosure

---

## Version History

**v1.0** (2024)
- Initial release
- BattlEye support
- CR3-based memory access
- Process hiding
- Driver trace cleanup

**Tested Platforms**:
- Windows 10 21H2 (x64)
- Windows 11 22H2 (x64)

**Known Issues**:
- EAC requires CR3 buffering
- Some games detect public signatures
- Requires PatchGuard bypass

---

## Additional Resources

### Tools
- WinDbg: https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/
- Process Hacker: https://processhacker.sourceforge.io/
- DebugView: https://docs.microsoft.com/en-us/sysinternals/downloads/debugview
- x64dbg: https://x64dbg.com/
- IDA Pro / Ghidra: For reverse engineering

### Documentation
- Windows Driver Kit docs: https://docs.microsoft.com/en-us/windows-hardware/drivers/
- Windows Internals (book)
- OSR Online: https://www.osronline.com/

### Communities
- UnknownCheats: https://www.unknowncheats.me/ (educational)
- OSR Dev Forums: Driver development
- MSDN Forums: Windows API

---

**Author**: Based on work by bloodieys (Ezekiel Cerel)  
**License**: CC BY-NC-ND 4.0  
**Contact**: Discord - bloodieys  
**Last Updated**: 2024
