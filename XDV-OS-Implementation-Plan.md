# XDV Operating System Implementation Plan

**Version:** 1.0  
**Status:** Draft  
**Target:** Complete XDV OS with Bootloader, Kernel, Filesystem, Shell  

---

## Executive Summary

This document outlines the implementation roadmap for building a complete XDV Operating System that can boot and provide a functional command-line environment. The XDV OS consists of multiple components:

1. **xdv-boot** - Bootloader/early initialization
2. **xdv-kernel** - Core kernel (already implemented in xdv-kernel repository)
3. **xdv-userspace** - User-level utilities and shell
4. **xdv-filesystem** - File system tools and management
5. **xdv-shell** - Command-line shell interface

---

## Current State

### Completed Components
- ✅ **xdv-kernel** - 13 sectors implemented (boot, memory, cpu, drivers, kernel, dal, qdomain, phidomain, cds, umf, hypervisor, sdbm, odt)
- ✅ All 26 kernel source files compile with DPL v0.2 compiler
- ✅ CI pipeline configured for sector-by-sector builds

### Components to Build
- ⏳ xdv-boot (bootloader)
- ⏳ xdv-userspace (core utilities)
- ⏳ xdv-filesystem (file system)
- ⏳ xdv-shell (command shell)

---

## Architecture Overview

```
XDV Operating System
├── xdv-boot/              # Bootloader (stage 1 & 2)
├── xdv-kernel/            # Kernel (existing)
│   └── sector/
│       ├── xdv_boot/     # Boot sector within kernel
│       ├── xdv_memory/   # Memory management
│       ├── xdv_cpu/     # CPU/interrupts
│       ├── xdv_drivers/ # Device drivers
│       └── ...          # Other sectors
├── xdv-userspace/        # User-space utilities
│   ├── init/             # Init process
│   ├── libc/             # C library
│   ├── bin/              # System binaries
│   └── lib/              # Shared libraries
├── xdv-filesystem/       # File system implementation
│   ├── xdvfs/           # XDV native filesystem
│   ├── fatfs/           # FAT filesystem driver
│   └── ext2/            # EXT2 filesystem driver
└── xdv-shell/            # Command-line shell
    ├── shell/           # Interactive shell
    └── commands/        # Built-in commands
```

---

## Phase 1: xdv-boot (Bootloader)

### Objectives
Create a bootloader that can load the XDV kernel from storage and transfer control to it.

### Boot Process Stages

```
Stage 1 (MBR)           Stage 2 (Bootloader)      Kernel Boot
─────────────────      ─────────────────────     ──────────────
- Read stage 2         - Parse partition table   - Setup GDT
- Jump to stage 2      - Load kernel from disk   - Setup IDT
                       - Parse kernel headers    - Setup paging
                       - Jump to kernel entry   - Jump to kmain()
```

### Implementation Steps

#### 1.1 Stage 1 - MBR Bootloader
```
xdv-boot/
├── stage1/
│   └── mbr.asm          # 512-byte MBR code
├── stage2/
│   ├── boot.asm        # Stage 2 assembly
│   └── boot.c          # C entry point (later)
└── Makefile
```

**Requirements:**
- 512 bytes maximum
- Read boot sector from disk
- Jump to stage 2

#### 1.2 Stage 2 - Extended Bootloader
```
xdv-boot/
├── stage2/
│   ├── boot.asm        # Protected mode setup
│   ├── disk.asm        # Disk I/O
│   ├── gdt.asm         # GDT setup
│   ├── paging.asm      # Enable paging
│   └── main.c          # Boot main
├── disk/
│   └── ata.c           # ATA/IDE driver
└── fat/
    └── fat12.c         # FAT12/16 parsing
```

**Requirements:**
- Load kernel from disk (FAT12/16)
- Setup protected mode
- Enable paging
- Jump to kernel entry point

#### 1.3 Kernel Loading
- Parse kernel ELF/PE headers
- Load kernel to memory (0x100000)
- Setup boot information structure
- Pass Multiboot2-compliant info

### Deliverables
- [ ] `xdv-boot/stage1/mbr.bin` - 512-byte bootable image
- [ ] `xdv-boot/stage2/boot.bin` - Extended bootloader
- [ ] Bootable disk image with kernel

---

## Phase 2: xdv-userspace (User Space)

### Objectives
Create the minimal user-space environment needed for system operation.

### Directory Structure
```
xdv-userspace/
├── State.toml
├── src/
│   ├── init/            # Init process (PID 1)
│   ├── libc/           # C standard library
│   │   ├── string.c
│   │   ├── stdio.c
│   │   ├── stdlib.c
│   │   └── unistd.c
│   ├── bin/            # User binaries
│   │   ├── sh/         # Shell
│   │   ├── ls/         # List files
│   │   ├── cat/        # Concatenate files
│   │   ├── echo/       # Echo text
│   │   ├── mkdir/      # Make directory
│   │   ├── rm/         # Remove files
│   │   └── ps/         # Process status
│   ├── lib/            # Shared libraries
│   └── syscalls/       # System call wrappers
└── include/            # Headers
    └── sys/
        └── syscalls.h
```

### Implementation Steps

#### 2.1 Init Process
```dust
// init.ds - First user-space process
forge Init {
    proc K::main() {
        // Mount filesystems
        mount_filesystems();
        
        // Start system services
        start_getty();
        
        // Wait for children
        while true {
            wait_for_child();
        }
    }
}
```

#### 2.2 C Library (libc)
Minimal implementation needed:
- `memcpy`, `memset`, `memcmp`
- `strlen`, `strcpy`, `strcmp`
- `printf`, `sprintf`
- `open`, `read`, `write`, `close`
- `fork`, `execve`, `waitpid`, `exit`

#### 2.3 System Calls
Define kernel-user ABI:
```dust
const SYS_READ: K[UInt64] = 0;
const SYS_WRITE: K[UInt64] = 1;
const SYS_OPEN: K[UInt64] = 2;
const SYS_CLOSE: K[UInt64] = 3;
const SYS_FORK: K[UInt64] = 57;
const SYS_EXECVE: K[UInt64] = 59;
const SYS_EXIT: K[UInt64] = 60;
```

### Deliverables
- [ ] Init process implementation
- [ ] Minimal libc implementation
- [ ] Basic system binaries (ls, cat, echo, mkdir, rm)
- [ ] System call interface

---

## Phase 3: xdv-filesystem (File System)

### Objectives
Implement file system support for reading/writing persistent data.

### Implementation Steps

#### 3.1 XDV Native Filesystem (xdvfs)
```dust
// XDV Filesystem structures
K XdvSuperblock = K[Struct {
    magic: K[UInt32],
    version: K[UInt32],
    block_size: K[UInt32],
    total_blocks: K[UInt64],
    inode_count: K[UInt64],
    free_blocks: K[UInt64]
}];

K XdvInode = K[Struct {
    mode: K[UInt16],
    size: K[UInt32],
    blocks: K[UInt16],
    direct: K[Array[K[UInt64], 12]>,
    indirect: K[UInt64],
    double_indirect: K[UInt64]
}];
```

#### 3.2 FAT Filesystem Driver
Support for FAT12/16/32 to read existing media:
```dust
forge FatDriver {
    proc K::fat_read_sector(drive: K[UInt8], lba: K[UInt64], buffer: K[Ptr[K[UInt8]]]) -> K[Result[K[Unit], K[Error]]];
    proc K::fat_parse_directory(entry: K[FatDirEntry]) -> K[Result[K[FileInfo], K[Error]]];
    proc K::fat_open_file(path: K[Str]) -> K[Result[K[FileHandle], K[Error]]];
}
```

#### 3.3 VFS Layer
Virtual File System abstraction:
```dust
forge Vfs {
    proc K::vfs_mount(device: K[Str], mount_point: K[Str], fs_type: K[FsType]) -> K[Result[K[Unit], K[Error]]];
    proc K::vfs_open(path: K[Str], flags: K[UInt32]) -> K[Result[K[FileHandle], K[Error]]>;
    proc K::vfs_read(fd: K[FileHandle], buf: K[Ptr[K[UInt8]]], count: K[Size]) -> K[Result[K[Size], K[Error]]];
    proc K::vfs_write(fd: K[FileHandle], buf: K[Ptr[K[UInt8]]], count: K[Size]) -> K[Result[K[Size], K[Error]]];
}
```

### Deliverables
- [ ] xdvfs filesystem implementation
- [ ] FAT filesystem driver
- [ ] VFS abstraction layer

---

## Phase 4: xdv-shell (Command Shell)

### Objectives
Implement an interactive command-line shell.

### Directory Structure
```
xdv-shell/
├── State.toml
├── src/
│   ├── main.c           # Shell entry point
│   ├── shell.c          # Shell core
│   ├── parser.c         # Command parser
│   ├── executor.c       # Command execution
│   └── builtin/
│       ├── cd.c         # Change directory
│       ├── pwd.c        # Print working directory
│       ├── exit.c       # Exit shell
│       └── help.c       # Help command
└── commands/            # External commands
    ├── ls.c
    ├── cat.c
    ├── echo.c
    └── ...
```

### Implementation Steps

#### 4.1 Shell Core
```c
// Shell main loop
void shell_loop(void) {
    while (running) {
        print_prompt();
        read_line();
        parse_command();
        execute_command();
    }
}
```

#### 4.2 Built-in Commands
| Command | Description |
|---------|-------------|
| `cd` | Change directory |
| `pwd` | Print working directory |
| `ls` | List directory contents |
| `cat` | Display file contents |
| `echo` | Print text |
| `mkdir` | Create directory |
| `rm` | Remove file/directory |
| `touch` | Create empty file |
| `cp` | Copy file |
| `mv` | Move/rename file |
| `help` | Show help |
| `exit` | Exit shell |

#### 4.3 Pipeline Support
```c
// Simple pipeline: cmd1 | cmd2
void execute_pipeline(char *cmd1, char *cmd2) {
    int pipefd[2];
    pipe(pipefd);
    
    if (fork() == 0) {
        // Child 1: cmd1
        dup2(pipefd[1], STDOUT);
        close(pipefd[0]);
        execute(cmd1);
    } else {
        // Child 2: cmd2
        dup2(pipefd[0], STDIN);
        close(pipefd[1]);
        execute(cmd2);
    }
}
```

### Deliverables
- [ ] Interactive shell
- [ ] Built-in commands
- [ ] Pipeline support
- [ ] Basic environment variables

---

## Phase 5: Integration & Testing

### Build System
```
xdv-os/
├── Makefile                    # Main build
├── boot/
│   └── Makefile
├── kernel/
│   └── Makefile (existing)
├── userspace/
│   └── Makefile
├── filesystem/
│   └── Makefile
└── shell/
    └── Makefile
```

### Testing Strategy

#### 5.1 Unit Tests
- Kernel sector tests (existing)
- Filesystem operation tests
- Shell command tests

#### 5.2 Integration Tests
- Boot test in QEMU
- Kernel panic handling
- System call verification

#### 5.3 CI/CD
```yaml
# .github/workflows/ci.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build bootloader
        run: make -C boot
      - name: Build kernel
        run: make -C kernel
      - name: Build userspace
        run: make -C userspace
      - name: Build shell
        run: make -C shell
      - name: Build disk image
        run: make disk.img
      - name: Test in QEMU
        run: qemu-system-x86_64 -kernel build/kernel.bin -hda disk.img
```

### Deliverables
- [ ] Complete build system
- [ ] Test suite
- [ ] CI pipeline

---

## Milestones

### Milestone 1: Bootloader (Week 1-2)
- [ ] Stage 1 MBR bootloader
- [ ] Stage 2 extended bootloader
- [ ] Load kernel from disk
- [ ] Boot to kernel prompt

### Milestone 2: Basic Userspace (Week 3-4)
- [ ] Init process
- [ ] Minimal libc
- [ ] Basic system calls work
- [ ] Simple shell prompt

### Milestone 3: File System (Week 5-6)
- [ ] VFS layer
- [ ] FAT driver
- [ ] xdvfs implementation
- [ ] File operations work

### Milestone 4: Shell & Utilities (Week 7-8)
- [ ] Full shell implementation
- [ ] Core utilities (ls, cat, mkdir, etc.)
- [ ] Pipeline support
- [ ] Environment variables

### Milestone 5: Integration (Week 9-10)
- [ ] All components build together
- [ ] Boot from disk image
- [ ] QEMU testing
- [ ] CI/CD complete

---

## Technical Requirements

### Compiler
- DPL v0.2 compiler (dust)
- Current compiler supports:
  - `forge` blocks with `proc` definitions
  - `const` declarations
  - `shape` type definitions
  - Basic expressions and binary operators

### Hardware Target
- x86-64 (Intel/AMD)
- QEMU for testing
- Minimum 512MB RAM
- Disk image for filesystem

### Build Tools
- Make
- NASM (assembler)
- QEMU (emulator)
- Git

---

## Appendix: File Organization

```
xdv-os/
├── README.md
├── LICENSE
├── Makefile
├── STATE.md
├── .github/
│   └── workflows/
│       └── ci.yml
├── xdv-boot/           # NEW: Bootloader
│   ├── State.toml
│   ├── stage1/
│   ├── stage2/
│   └── Makefile
├── xdv-kernel/        # EXISTING: Kernel
│   ├── State.toml
│   ├── sector/
│   └── ...
├── xdv-userspace/     # NEW: User space
│   ├── State.toml
│   ├── src/
│   │   ├── init/
│   │   ├── libc/
│   │   └── bin/
│   └── Makefile
├── xdv-filesystem/    # NEW: Filesystem
│   ├── State.toml
│   ├── src/
│   │   ├── vfs/
│   │   ├── fat/
│   │   └── xdvfs/
│   └── Makefile
└── xdv-shell/         # NEW: Shell
    ├── State.toml
    ├── src/
    │   ├── shell.c
    │   └── builtin/
    └── Makefile
```

---

## Conclusion

This implementation plan provides a roadmap for building a complete XDV Operating System. The phased approach allows incremental development while maintaining a working system at each milestone. The existing xdv-kernel provides a solid foundation, and the additional components (bootloader, userspace, filesystem, shell) will create a fully functional operating system.

**Total Estimated Duration:** 10 weeks
