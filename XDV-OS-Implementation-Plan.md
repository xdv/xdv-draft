> ARCHIVED DRAFT NOTICE
> Non-normative historical artifact. Do not treat this file as canonical specification.
> Canonical source: ../xdv-spec (see README.md, Corpus.md, and requirement index artifacts).
> New work and requirement changes must be authored in ../xdv-spec.
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
- âœ… **xdv-kernel** - 13 sectors implemented (boot, memory, cpu, drivers, kernel, dal, qdomain, phidomain, cds, umf, hypervisor, sdbm, odt)
- âœ… All 26 kernel source files compile with DPL v0.2 compiler
- âœ… CI pipeline configured for sector-by-sector builds

### Components to Build
- â³ xdv-boot (bootloader)
- â³ xdv-userspace (core utilities)
- â³ xdv-filesystem (file system)
- â³ xdv-shell (command shell)

---

## Architecture Overview

```
XDV Operating System
â”œâ”€â”€ xdv-boot/              # Bootloader (stage 1 & 2)
â”œâ”€â”€ xdv-kernel/            # Kernel (existing)
â”‚   â””â”€â”€ sector/
â”‚       â”œâ”€â”€ xdv_boot/     # Boot sector within kernel
â”‚       â”œâ”€â”€ xdv_memory/   # Memory management
â”‚       â”œâ”€â”€ xdv_cpu/     # CPU/interrupts
â”‚       â”œâ”€â”€ xdv_drivers/ # Device drivers
â”‚       â””â”€â”€ ...          # Other sectors
â”œâ”€â”€ xdv-userspace/        # User-space utilities
â”‚   â”œâ”€â”€ init/             # Init process
â”‚   â”œâ”€â”€ libc/             # C library
â”‚   â”œâ”€â”€ bin/              # System binaries
â”‚   â””â”€â”€ lib/              # Shared libraries
â”œâ”€â”€ xdv-filesystem/       # File system implementation
â”‚   â”œâ”€â”€ xdvfs/           # XDV native filesystem
â”‚   â”œâ”€â”€ fatfs/           # FAT filesystem driver
â”‚   â””â”€â”€ ext2/            # EXT2 filesystem driver
â””â”€â”€ xdv-shell/            # Command-line shell
    â”œâ”€â”€ shell/           # Interactive shell
    â””â”€â”€ commands/        # Built-in commands
```

---

## Phase 1: xdv-boot (Bootloader)

### Objectives
Create a bootloader that can load the XDV kernel from storage and transfer control to it.

### Boot Process Stages

```
Stage 1 (MBR)           Stage 2 (Bootloader)      Kernel Boot
â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€      â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€     â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
- Read stage 2         - Parse partition table   - Setup GDT
- Jump to stage 2      - Load kernel from disk   - Setup IDT
                       - Parse kernel headers    - Setup paging
                       - Jump to kernel entry   - Jump to kmain()
```

### Implementation Steps

#### 1.1 Stage 1 - MBR Bootloader
```
xdv-boot/
â”œâ”€â”€ stage1/
â”‚   â””â”€â”€ mbr.asm          # 512-byte MBR code
â”œâ”€â”€ stage2/
â”‚   â”œâ”€â”€ boot.asm        # Stage 2 assembly
â”‚   â””â”€â”€ boot.c          # C entry point (later)
â””â”€â”€ Makefile
```

**Requirements:**
- 512 bytes maximum
- Read boot sector from disk
- Jump to stage 2

#### 1.2 Stage 2 - Extended Bootloader
```
xdv-boot/
â”œâ”€â”€ stage2/
â”‚   â”œâ”€â”€ boot.asm        # Protected mode setup
â”‚   â”œâ”€â”€ disk.asm        # Disk I/O
â”‚   â”œâ”€â”€ gdt.asm         # GDT setup
â”‚   â”œâ”€â”€ paging.asm      # Enable paging
â”‚   â””â”€â”€ main.c          # Boot main
â”œâ”€â”€ disk/
â”‚   â””â”€â”€ ata.c           # ATA/IDE driver
â””â”€â”€ fat/
    â””â”€â”€ fat12.c         # FAT12/16 parsing
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
â”œâ”€â”€ State.toml
â”œâ”€â”€ src/
â”‚   â”œâ”€â”€ init/            # Init process (PID 1)
â”‚   â”œâ”€â”€ libc/           # C standard library
â”‚   â”‚   â”œâ”€â”€ string.c
â”‚   â”‚   â”œâ”€â”€ stdio.c
â”‚   â”‚   â”œâ”€â”€ stdlib.c
â”‚   â”‚   â””â”€â”€ unistd.c
â”‚   â”œâ”€â”€ bin/            # User binaries
â”‚   â”‚   â”œâ”€â”€ sh/         # Shell
â”‚   â”‚   â”œâ”€â”€ ls/         # List files
â”‚   â”‚   â”œâ”€â”€ cat/        # Concatenate files
â”‚   â”‚   â”œâ”€â”€ echo/       # Echo text
â”‚   â”‚   â”œâ”€â”€ mkdir/      # Make directory
â”‚   â”‚   â”œâ”€â”€ rm/         # Remove files
â”‚   â”‚   â””â”€â”€ ps/         # Process status
â”‚   â”œâ”€â”€ lib/            # Shared libraries
â”‚   â””â”€â”€ syscalls/       # System call wrappers
â””â”€â”€ include/            # Headers
    â””â”€â”€ sys/
        â””â”€â”€ syscalls.h
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
â”œâ”€â”€ State.toml
â”œâ”€â”€ src/
â”‚   â”œâ”€â”€ main.c           # Shell entry point
â”‚   â”œâ”€â”€ shell.c          # Shell core
â”‚   â”œâ”€â”€ parser.c         # Command parser
â”‚   â”œâ”€â”€ executor.c       # Command execution
â”‚   â””â”€â”€ builtin/
â”‚       â”œâ”€â”€ cd.c         # Change directory
â”‚       â”œâ”€â”€ pwd.c        # Print working directory
â”‚       â”œâ”€â”€ exit.c       # Exit shell
â”‚       â””â”€â”€ help.c       # Help command
â””â”€â”€ commands/            # External commands
    â”œâ”€â”€ ls.c
    â”œâ”€â”€ cat.c
    â”œâ”€â”€ echo.c
    â””â”€â”€ ...
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
â”œâ”€â”€ Makefile                    # Main build
â”œâ”€â”€ boot/
â”‚   â””â”€â”€ Makefile
â”œâ”€â”€ kernel/
â”‚   â””â”€â”€ Makefile (existing)
â”œâ”€â”€ userspace/
â”‚   â””â”€â”€ Makefile
â”œâ”€â”€ filesystem/
â”‚   â””â”€â”€ Makefile
â””â”€â”€ shell/
    â””â”€â”€ Makefile
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
â”œâ”€â”€ README.md
â”œâ”€â”€ LICENSE
â”œâ”€â”€ Makefile
â”œâ”€â”€ STATE.md
â”œâ”€â”€ .github/
â”‚   â””â”€â”€ workflows/
â”‚       â””â”€â”€ ci.yml
â”œâ”€â”€ xdv-boot/           # NEW: Bootloader
â”‚   â”œâ”€â”€ State.toml
â”‚   â”œâ”€â”€ stage1/
â”‚   â”œâ”€â”€ stage2/
â”‚   â””â”€â”€ Makefile
â”œâ”€â”€ xdv-kernel/        # EXISTING: Kernel
â”‚   â”œâ”€â”€ State.toml
â”‚   â”œâ”€â”€ sector/
â”‚   â””â”€â”€ ...
â”œâ”€â”€ xdv-userspace/     # NEW: User space
â”‚   â”œâ”€â”€ State.toml
â”‚   â”œâ”€â”€ src/
â”‚   â”‚   â”œâ”€â”€ init/
â”‚   â”‚   â”œâ”€â”€ libc/
â”‚   â”‚   â””â”€â”€ bin/
â”‚   â””â”€â”€ Makefile
â”œâ”€â”€ xdv-filesystem/    # NEW: Filesystem
â”‚   â”œâ”€â”€ State.toml
â”‚   â”œâ”€â”€ src/
â”‚   â”‚   â”œâ”€â”€ vfs/
â”‚   â”‚   â”œâ”€â”€ fat/
â”‚   â”‚   â””â”€â”€ xdvfs/
â”‚   â””â”€â”€ Makefile
â””â”€â”€ xdv-shell/         # NEW: Shell
    â”œâ”€â”€ State.toml
    â”œâ”€â”€ src/
    â”‚   â”œâ”€â”€ shell.c
    â”‚   â””â”€â”€ builtin/
    â””â”€â”€ Makefile
```

---

## Conclusion

This implementation plan provides a roadmap for building a complete XDV Operating System. The phased approach allows incremental development while maintaining a working system at each milestone. The existing xdv-kernel provides a solid foundation, and the additional components (bootloader, userspace, filesystem, shell) will create a fully functional operating system.

**Total Estimated Duration:** 10 weeks

