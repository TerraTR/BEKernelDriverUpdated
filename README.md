# BEKernelDriver - Advanced Windows Kernel Driver

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-nd/4.0/)
[![Windows](https://img.shields.io/badge/Platform-Windows%2010%2F11-blue.svg)]()
[![Architecture](https://img.shields.io/badge/Architecture-x64-green.svg)]()

A sophisticated Windows kernel-mode driver with user-mode companion application for advanced memory operations on BattlEye-protected games. This is an updated version of the original BEKernelDriver with enhanced protections, cleaner code, and comprehensive documentation.

## 🎯 Features

- **CR3/DTB-Based Memory Access**: Physical memory read/write operations bypassing standard protections
- **ObCreateObject Hook**: Stealthy kernel-usermode communication via driver hijacking
- **Process Hiding**: EPROCESS list unlinking for complete process concealment
- **Driver Trace Cleanup**: Automatic removal of mapper artifacts (PiDDB, MMU/MML, HashBucket)
- **Anti-Detection**: String obfuscation, dynamic offset resolution, and minimal footprint
- **BattlEye Compatible**: Tested and working on Escape From Tarkov, DayZ, Rainbow Six Siege

## ⚠️ Important Notice

**This software is for educational and research purposes only.**

- ❌ Using this to cheat in online games violates Terms of Service
- ❌ May result in permanent bans
- ❌ May violate laws in your jurisdiction
- ✅ Use for learning about game security mechanisms
- ✅ Test only in authorized environments

The author assumes **no liability** for misuse. See [LICENSE](LICENSE) for full terms.

## 📚 Documentation

This project includes comprehensive documentation covering every aspect of development, deployment, and usage:

### Core Documentation
- **[DOCUMENTATION.md](DOCUMENTATION.md)** - Complete technical documentation
  - Architecture overview
  - Component descriptions
  - API reference
  - Build instructions
  - Deployment guide
  - Troubleshooting

- **[DEPLOYMENT.md](DEPLOYMENT.md)** - Step-by-step deployment guide
  - Prerequisites checklist
  - System preparation
  - Build process
  - Driver mapping
  - Verification steps
  - Emergency procedures

- **[FIVEM_PORTING_GUIDE.md](FIVEM_PORTING_GUIDE.md)** - Complete guide for porting to FiveM
  - FiveM architecture
  - Required modifications
  - Implementation steps
  - Testing procedures
  - Anti-detection techniques

## 🚀 Quick Start

### Prerequisites

1. **Software**:
   - Windows 10/11 x64
   - Visual Studio 2019/2022 with C++ Desktop Development
   - Windows Driver Kit (WDK)
   - [PdFwKrnl Mapper](https://github.com/i32-Sudo/PdFwKrnlMapper)
   - PatchGuard bypass (e.g., EFIGuard)

2. **System Configuration**:
   - Test signing enabled OR signature verification disabled
   - PatchGuard disabled
   - Running as Administrator

### Minimum Required Configuration

Before building, you **MUST** change these values:

1. **Device Names** (`KM/impl/communication/interface.h`):
```cpp
#define DEVICE_NAME L"\\Device\\YourUniqueName"
#define DOS_NAME L"\\DosDevices\\YourUniqueName"
```

2. **Hook Target** (`KM/entry/hook/hook.hpp`):
```cpp
const auto target_module = modules::get_kernel_module(_("target.sys"));
globals::cave_base = modules::find_pattern(target_module, _("SIGNATURE"), _("MASK"));
```

3. **Process Name** (`KM/processhyde/Hide.cpp`):
```cpp
const char target[] = "your_app.exe";
```

See [DOCUMENTATION.md - Configuration](DOCUMENTATION.md#configuration) for detailed instructions.

### Build Steps

1. **Clone the repository**:
```bash
git clone https://github.com/TerraTR/BEKernelDriverUpdated
cd BEKernelDriverUpdated
```

2. **Open solution**:
```
Open KrnlUpdated.sln in Visual Studio
```

3. **Configure and build**:
   - Set configuration to **Release x64**
   - Build `driver-execute` project (kernel driver)
   - Build `um` project (user-mode application)

4. **Deploy**:
   - Use PdFwKrnl Mapper to load driver
   - Run user-mode application
   - Follow prompts for initialization

See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed deployment instructions.

## 🏗️ Architecture

```
BEKernelDriverUpdated/
├── KM/                          # Kernel-Mode Driver
│   ├── entry/                  # Driver entry point
│   │   ├── main.cpp           # Initialization
│   │   └── hook/              # Hook implementation
│   ├── impl/                  # Core implementations
│   │   ├── communication/     # IOCTL interface
│   │   ├── invoked.h         # Request dispatcher
│   │   └── modules.h         # Module utilities
│   ├── kernel/               # Kernel utilities
│   ├── processhyde/          # Process hiding
│   ├── requests/             # Request handlers
│   └── clean/                # Trace cleanup
│
├── UM/                         # User-Mode Application
│   ├── entrypoint.cpp         # Main entry
│   ├── impl/driver/           # Driver communication API
│   └── Mapper.h              # Mapper integration
│
└── Documentation/
    ├── DOCUMENTATION.md       # Technical reference
    ├── DEPLOYMENT.md         # Deployment guide
    └── FIVEM_PORTING_GUIDE.md # FiveM porting
```

## 🔧 Technical Details

### Communication Method
- **Technique**: ObCreateObject Hook via code cave hijacking
- **Detection Risk**: Low (when properly configured)
- **Advantage**: Avoids standard IOCTL detection patterns

### Memory Access
- **Method**: CR3/DTB-based physical memory access
- **Translation**: 4-level x64 paging translation
- **Bypass**: Direct physical memory read/write
- **Compatibility**: BattlEye ✅, EAC ⚠️ (requires CR3 buffering)

### Stealth Features
1. **Driver Trace Cleanup**: Removes PiDDB, MMU/MML, and HashBucket entries
2. **Process Hiding**: Unlinks from EPROCESS list
3. **String Obfuscation**: XOR encryption at compile-time
4. **No Driver Object**: Hijacks existing legitimate driver

## 📖 Component Overview

### Kernel Driver (KM)
- **Entry Point**: Driver initialization and hook setup
- **Hook System**: Code cave hijacking in legitimate kernel module
- **Communication**: Custom IOCTL interface via hijacked driver
- **Memory Operations**: CR3-based physical memory read/write
- **Process Hiding**: EPROCESS list manipulation
- **Trace Cleanup**: PiDDB, MMU/MML, HashBucket removal

### User-Mode Application (UM)
- **Driver API**: High-level communication interface
- **CR3 Resolution**: Automatic DTB discovery
- **Process Management**: PID enumeration and attachment
- **Memory Access**: Type-safe read/write templates
- **Module Enumeration**: Base address resolution

## 🧪 Testing

### Tested Configurations
- ✅ Windows 10 21H2 (x64)
- ✅ Windows 11 22H2 (x64)
- ✅ BattlEye (Tarkov, DayZ, R6 Siege)
- ⚠️ EAC (requires CR3 buffering implementation)

### Testing Recommendations
1. Use isolated test environment (VM or dedicated machine)
2. Test with test accounts only
3. Start with offline/single-player modes
4. Monitor system stability
5. Have recovery plan ready

See [DEPLOYMENT.md - Testing Environments](DEPLOYMENT.md#testing-environments) for detailed testing procedures.

## 🛡️ Security Considerations

### Detection Vectors
- Signature scanning (public code)
- Module enumeration
- Driver trace analysis
- Behavioral heuristics
- Screenshot analysis (for overlays)

### Mitigation Strategies
1. **Change all identifiers** (device names, pool tags, strings)
2. **Use code protector** (VMProtect, Themida) on user-mode
3. **Unique hook targets** per deployment
4. **Remove debug output** in release builds
5. **Test signature detection** before deployment

### Best Practices
- ✅ Use unique configurations for each deployment
- ✅ Apply code mutation/obfuscation
- ✅ Test in isolated environments
- ✅ Monitor for detection patterns
- ❌ Never use default configurations
- ❌ Never distribute pre-configured builds

## 🔍 EAC Compatibility Note

**Current Status**: ⚠️ Requires additional implementation

The driver works with BattlEye but requires modifications for EAC compatibility:

**Issue**: EAC resets CR3/DTB every 10-20 minutes, causing BSOD if read fails.

**Solution** (requires implementation):
```cpp
// Add CR3 validation and recovery
1. Catch read failures
2. Re-cache CR3 before retrying
3. Implement buffering mechanism
4. Add error handling for stale DTB
```

See [DOCUMENTATION.md - EAC Compatibility](DOCUMENTATION.md#eac-compatibility) for detailed implementation guide.

## 📦 Dependencies

### Build Dependencies
- Visual Studio 2019/2022
- Windows SDK 10.0.19041.0+
- Windows Driver Kit (WDK)

### Runtime Dependencies
- Windows 10/11 x64
- PatchGuard bypass (EFIGuard recommended)
- Vulnerable driver mapper (PdFwKrnl recommended)

### Optional
- VMProtect or Themida (code protection)
- WinDbg (kernel debugging)
- Process Hacker (process analysis)
- IDA Pro/Ghidra (reverse engineering)

## 🤝 Contributing

This project is released under CC BY-NC-ND 4.0, which **does not permit derivative works for distribution**. However:

- ✅ Bug reports and issues welcome
- ✅ Documentation improvements appreciated
- ✅ Testing feedback valuable
- ✅ Security vulnerability reports encouraged
- ❌ Pull requests with code changes cannot be accepted

For security vulnerabilities, please contact privately via Discord.

## 📄 License

**Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International**

- ✅ Share and redistribute in any medium or format
- ✅ Attribution required
- ❌ Commercial use prohibited
- ❌ Derivative works cannot be distributed

See [LICENSE](LICENSE) for full legal text.

## 📞 Contact & Support

- **Author**: bloodieys (Ezekiel Cerel)
- **Discord**: bloodieys
- **Repository**: https://github.com/TerraTR/BEKernelDriverUpdated

### Getting Help
1. Read the [DOCUMENTATION.md](DOCUMENTATION.md) thoroughly
2. Check [DEPLOYMENT.md](DEPLOYMENT.md) for deployment issues
3. Review troubleshooting sections
4. Open GitHub issue with detailed information
5. Contact on Discord for sensitive matters

### What to Include in Bug Reports
- Windows version and build number
- Exact error messages
- Steps to reproduce
- Configuration used (with sensitive info removed)
- WinDbg output (if BSOD)

## 🙏 Acknowledgments

- **PdFwKrnl Mapper**: [i32-Sudo](https://github.com/i32-Sudo/PdFwKrnlMapper)
- **Community**: UnknownCheats, OSR Online, MSDN Forums
- **Tools**: Microsoft (WDK), Hex-Rays (IDA Pro), Sysinternals

## 📊 Project Status

- **Status**: Active
- **Version**: 1.0
- **Last Update**: 2024
- **Maintenance**: Updated as needed for compatibility

### Known Issues
- EAC requires CR3 buffering (not implemented)
- Public signatures may be detected (use code protection)
- Requires PatchGuard bypass (system modification)

### Future Plans
- EAC compatibility improvements
- Additional anti-detection techniques
- Extended documentation
- More testing on various configurations

---

**⚠️ Final Warning**: This project is for **educational purposes only**. Using it to cheat in online games is unethical, violates Terms of Service, and may be illegal. Always act responsibly and respect the communities you interact with.

**Remember**: The best way to win is to play fairly. Use this knowledge to understand security, not to undermine it.

---

**Made with** 🛡️ **for educational purposes**  
**Licensed under** [CC BY-NC-ND 4.0](LICENSE)
