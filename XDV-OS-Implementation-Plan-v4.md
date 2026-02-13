# XDV Operating System Implementation Plan
## Using dustlib & dustlib_k - K-Domain Focus

**Version:** 4.0  
**Status:** Draft  
**Language:** DPL with dustlib & dustlib_k  

---

## 1. Executive Summary

This implementation plan defines building the XDV Operating System using **dustlib** and **dustlib_k** libraries. 

### Key Constraints
1. **K-Domain Only** - Runs on classical x86-64 hardware
2. **dustlib + dustlib_k** - Uses DPL standard libraries
3. **Full K Functionality** - Complete classical OS features
4. **Domain-Aware XDVFS** - Filesystem has Q/Φ type awareness (types defined, not functional)
5. **Boot to Shell** - Complete system from bootloader to interactive shell

### Libraries Used
```
dustlib      → Core types, Result, Option, String, Collections
dustlib_k   → Memory, Threading, Sync, I/O, Unsafe
```

---

## 2. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                          XDV Operating System                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                       Dependencies                                 │ │
│  │  ┌────────────────────┐  ┌─────────────────────────────────┐    │ │
│  │  │ dustlib            │  │ dustlib_k                        │    │ │
│  │  │ • Core types       │  │ • memory (alloc, free)          │    │ │
│  │  │ • Result, Option   │  │ • threading (spawn, join)       │    │ │
│  │  │ • String, Slice   │  │ • sync (mutex, condvar)        │    │ │
│  │  │ • Collections     │  │ • io (file, port, mmio)        │    │ │
│  │  │ • Formatting      │  │ • unsafe (raw memory)          │    │ │
│  │  └────────────────────┘  └─────────────────────────────────┘    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                         xdv-boot                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │
│  │  │ boot_mbr    │  │ boot_stage1  │  │ boot_stage2         │   │ │
│  │  │ (512 bytes) │  │ (DPL)        │  │ (DPL + dustlib_k)  │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                       xdv-kernel                                   │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────────┐  │ │
│  │  │boot    │ │memory  │ │cpu     │ │drivers │ │kernel       │  │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └─────────────┘  │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────────┐  │ │
│  │  │dal     │ │qdomain │ │phidomain│ │cds     │ │umf          │  │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └─────────────┘  │ │
│  │  ┌────────┐ ┌────────┐                                         │ │
│  │  │hypervisor│ │sdbm   │ ┌─────────────┐                       │ │
│  │  └────────┘ └────────┘ │odt          │                       │ │
│  │                        └─────────────┘                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                       xdv-xdvfs                                   │ │
│  │  K-Domain (Full):           Q/Φ-Domain (Awareness Only):       │ │
│  │  ┌──────────────┐           ┌─────────────────────────┐       │ │
│  │  │ K-File ops  │           │ Q-File types (stubbed)   │       │ │
│  │  │ • create    │           │ • QState shape           │       │ │
│  │  │ • read      │           │ • non-cloning marker     │       │ │
│  │  │ • write     │           │ Φ-File types (stubbed)   │       │ │
│  │  │ • delete    │           │ • PhiState shape         │       │ │
│  │  └──────────────┘           │ • coherence marker      │       │ │
│  │                             └─────────────────────────┘       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                      xdv-runtime                                  │ │
│  │  Uses dustlib_k:           Uses dustlib:                        │ │
│  │  ┌──────────────┐           ┌─────────────────────────┐       │ │
│  │  │ runtime_io   │           │ runtime_string          │       │ │
│  │  │ (dustlib_k::io)│         │ (dustlib::str)         │       │ │
│  │  └──────────────┘           └─────────────────────────┘       │ │
│  │  ┌──────────────┐           ┌─────────────────────────┐       │ │
│  │  │ runtime_mem │           │ runtime_process         │       │ │
│  │  │ (dustlib_k::memory)│   │ (dustlib_k::threading) │       │ │
│  │  └──────────────┘           └─────────────────────────┘       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                       xdv-shell                                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐   │ │
│  │  │ shell_main   │  │ shell_parser │  │ shell_commands     │   │ │
│  │  │ (dustlib_k)  │  │ (dustlib)   │  │ (dustlib + k)     │   │ │
│  │  └──────────────┘  └──────────────┘  └────────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Specifications

### 3.1 Dependencies

#### dustlib (Core Library)
```
sector/dustlib_core/src/
├── lib.ds           # Main library, exports all modules
├── prelude.ds       # Auto-imported definitions
├── result.ds        # Result<T, E> type
├── option.ds        # Option<T> type
├── str/             # String operations
├── slice/           # Slice type
├── primitive/       # Integer, bool methods
├── mem/             # Memory utilities
├── ops/             # Operator traits (Add, Sub, etc.)
├── fmt/             # Formatting (Display, Debug)
└── macros/          # Core macros

sector/dustlib_collections/src/
└── lib.ds           # Vec<T>, HashMap<K, V> (future)
```

#### dustlib_k (K-Regime Systems Library)
```
sector/dustlib_k/src/
├── lib.ds           # Main library, exports all modules
├── memory.ds        # alloc, free, zero_alloc
├── threading.ds     # spawn, join, thread ID
├── sync.ds          # Mutex, CondVar, Once
├── io.ds            # File, Port, MMIO operations
└── unsafe.ds        # Raw memory operations
```

---

### 3.2 xdv-boot (Bootloader)

Uses **dustlib_k::io** for disk access

```
xdv-boot/
├── State.toml
└── src/
    ├── boot_mbr.ds          # Stage 1: 512-byte MBR
    ├── boot_stage1.ds       # Stage 2: Basic init
    ├── boot_gdt.ds          # GDT setup
    ├── boot_idt.ds          # IDT setup
    ├── boot_paging.ds       # Enable paging
    ├── boot_disk.ds         # Disk I/O (dustlib_k::io)
    ├── boot_xdvfs_mount.ds  # Mount XDVFS
    └── boot_kernel_load.ds  # Load kernel
```

**Code Example:**
```dust
use dustlib_k::io;

// boot_disk.ds - Disk I/O using dustlib_k
forge BootDisk {
    
    const SECTOR_SIZE: K[UInt32] = 512;
    
    // Read sector using dustlib_k::io
    proc K::read_sector(
        drive: K[UInt8],
        lba: K[UInt64],
        buffer: K[Ptr[K[UInt8]]]
    ) -> K[Result[K[Unit], K[BootError]]] {
        let port = 0x1F0;  // ATA primary
        dustlib_k::io::io_write(port + 0, lba & 0xFF);
        dustlib_k::io::io_write(port + 1, (lba >> 8) & 0xFF);
        dustlib_k::io::io_write(port + 2, (lba >> 16) & 0xFF);
        dustlib_k::io::io_write(port + 3, (lba >> 24) & 0xFF);
        
        // Read data
        let i = 0;
        while i < SECTOR_SIZE {
            buffer[i] = dustlib_k::io::io_read(port) as K[UInt8];
            i = i + 1;
        }
        
        K[Ok(K[Unit])]
    }
}
```

---

### 3.3 xdv-kernel (Existing)

Already implemented 13 sectors using dustlib/dustlib_k patterns:
- xdv_boot, xdv_memory, xdv_cpu, xdv_drivers, xdv_kernel
- xdv_dal, xdv_qdomain, xdv_phidomain, xdv_cds
- xdv_umf, xdv_hypervisor, xdv_sdbm, xdv_odt

---

### 3.4 xdv-xdvfs (Domain-Aware File System)

**K-Domain:** Fully functional
**Q/Φ Domains:** Type definitions only (stubbed)

```
xdv-xdvfs/
├── State.toml
└── src/
    ├── xdvfs_superblock.ds   # Superblock (dustlib types)
    ├── xdvfs_inode.ds        # Inode management
    ├── xdvfs_block_alloc.ds  # Block allocation
    ├── xdvfs_k_file.ds      # K-File operations (FULL)
    ├── xdvfs_q_types.ds     # Q-File types (STUB - awareness only)
    ├── xdvfs_phi_types.ds   # Φ-File types (STUB - awareness only)
    ├── xdvfs_directory.ds   # Directory ops (dustlib)
    ├── xdvfs_mount.ds       # Mount operations
    └── xdvfs_errors.ds      # Error codes
```

**Code Example - K-File (Full):**
```dust
use dustlib::result::Result;
use dustlib::option::Option;

// xdvfs_k_file.ds - Full K-Domain file operations
forge KFileOps {
    
    // Create classical file
    proc K::k_create(
        parent: K[UInt64],
        name: K[Str],
        perms: K[UInt16]
    ) -> K[Result[K[UInt64], K[FsError]]] {
        // Full implementation using dustlib_k::memory
        let inode = allocate_inode()?;
        set_inode_type(inode, K[FILE_TYPE_REGULAR]);
        add_dir_entry(parent, name, inode)?;
        K[Ok(inode)]
    }
    
    // Read classical file
    proc K::k_read(
        inode: K[UInt64],
        offset: K[UInt64],
        count: K[Size]
    ) -> K[Result[K[Array[K[UInt8]]], K[FsError]]] {
        // Full implementation
        let data = read_blocks(inode, offset, count)?;
        K[Ok(data)]
    }
    
    // Write classical file
    proc K::k_write(
        inode: K[UInt64],
        offset: K[UInt64],
        data: K[Array[K[UInt8]]]
    ) -> K[Result[K[Size], K[FsError]]] {
        // Full implementation
        let written = write_blocks(inode, offset, data)?;
        update_mtime(inode);
        K[Ok(written)]
    }
}
```

**Code Example - Q/Φ Types (Stubbed):**
```dust
use dustlib::option::Option;

// xdvfs_q_types.ds - Q-Domain type awareness (stubbed)
// These types are defined but operations will fail gracefully
forge QTypes {
    
    // Q-File type marker
    const FILE_TYPE_QUANTUM: K[UInt16] = 2;
    
    // Quantum state placeholder (type only, not functional)
    shape QState {
        qubit_count: K[UInt32],
        coherence_ns: K[UInt64],
        fidelity: K[Float64],
        state_data: K[Array[K[UInt8], 16]]  // Placeholder
    }
    
    // Create quantum file stub - returns error
    proc K::q_create(
        parent: K[UInt64],
        name: K[Str],
        qubits: K[UInt32]
    ) -> K[Result[K[UInt64], K[FsError]]] {
        // Q-Domain not available on K-only hardware
        K[Err(K[FsError::DomainNotAvailable])]
    }
    
    // Check if Q-Domain is available
    proc K::q_available() -> K[Bool] {
        // Always false for K-only system
        return 0;
    }
}
```

```dust
// xdvfs_phi_types.ds - Φ-Domain type awareness (stubbed)
forge PhiTypes {
    
    // Φ-File type marker
    const FILE_TYPE_PHASE: K[UInt16] = 3;
    
    // Phase state placeholder (type only, not functional)
    shape PhiState {
        phase_value: K[Float64],
        coherence_ns: K[UInt64],
        admissibility: K[Float64]
    }
    
    // Create phase file stub - returns error
    proc K::phi_create(
        parent: K[UInt64],
        name: K[Str],
        phase_config: K[PhiConfig]
    ) -> K[Result[K[UInt64], K[FsError]]] {
        // Φ-Domain not available on K-only hardware
        K[Err(K[FsError::DomainNotAvailable])]
    }
    
    // Check if Φ-Domain is available
    proc K::phi_available() -> K[Bool] {
        // Always false for K-only system
        return 0;
    }
}
```

---

### 3.5 xdv-runtime (User Space Runtime)

Uses **dustlib** and **dustlib_k**

```
xdv-runtime/
├── State.toml
└── src/
    ├── runtime_io.ds         # I/O (wraps dustlib_k::io)
    ├── runtime_memory.ds     # Memory (wraps dustlib_k::memory)
    ├── runtime_string.ds     # String (wraps dustlib::str)
    ├── runtime_process.ds    # Process (wraps dustlib_k::threading)
    ├── runtime_scheduler.ds  # Scheduler
    ├── runtime_fs.ds         # FS interface (wraps xdvfs)
    ├── runtime_console.ds    # Console I/O
    └── runtime_init.ds       # Init process (PID 1)
```

**Code Example:**
```dust
use dustlib::result::Result;
use dustlib_k::io;
use dustlib_k::memory;

// runtime_console.ds - Console I/O using dustlib_k
forge Console {
    
    const VGA_ADDR: K[Ptr[K[UInt8]]] = 0xB8000 as K[Ptr[K[UInt8]]];
    const VGA_WIDTH: K[UInt32] = 80;
    const VGA_HEIGHT: K[UInt32] = 25;
    
    // Put character to console
    proc K::putchar(ch: K[UInt8]) -> K[Unit] {
        let pos = get_cursor_position();
        let row = pos / VGA_WIDTH;
        let col = pos % VGA_WIDTH;
        
        // Write to VGA memory
        VGA_ADDR[(row * VGA_WIDTH + col) * 2] = ch;
        VGA_ADDR[(row * VGA_WIDTH + col) * 2 + 1] = 0x07;  // light gray
        
        // Advance cursor
        if col < VGA_WIDTH - 1 {
            set_cursor_position(row, col + 1);
        } else if row < VGA_HEIGHT - 1 {
            set_cursor_position(row + 1, 0);
        } else {
            scroll_up();
            set_cursor_position(VGA_HEIGHT - 1, 0);
        }
        
        K[Unit]
    }
    
    // Put string to console
    proc K::puts(s: K[Str]) -> K[Unit] {
        let i = 0;
        while i < s.len() {
            putchar(s[i]);
            i = i + 1;
        }
        K[Unit]
    }
}
```

---

### 3.6 xdv-shell (Command Shell)

```
xdv-shell/
├── State.toml
└── src/
    ├── shell_main.ds        # Entry point
    ├── shell_loop.ds       # Main loop
    ├── shell_prompt.ds     # Prompt generation
    ├── shell_lexer.ds      # Tokenizer (dustlib::str)
    ├── shell_parser.ds     # Command parser (dustlib)
    ├── shell_executor.ds  # Command execution
    ├── shell_builtin/
    │   ├── builtin_cd.ds   # cd using xdvfs
    │   ├── builtin_ls.ds   # ls using xdvfs
    │   ├── builtin_cat.ds   # cat using xdvfs
    │   ├── builtin_mkdir.ds # mkdir using xdvfs
    │   ├── builtin_rm.ds    # rm using xdvfs
    │   ├── builtin_echo.ds  # echo
    │   ├── builtin_ps.ds    # process status
    │   ├── builtin_help.ds  # help
    │   └── builtin_exit.ds  # exit
    └── shell_completion.ds # Tab completion
```

**Code Example:**
```dust
use dustlib::result::Result;
use dustlib::str;

// shell_loop.ds - Main shell loop
forge ShellLoop {
    
    const PROMPT: K[Str] = "xdv$ ";
    const MAX_LINE: K[Size] = 256;
    
    // Main shell entry point
    proc K::main() -> K[Unit] {
        let running = 1;
        
        // Print welcome
        runtime_console::puts("XDV Shell v0.2\r\n");
        runtime_console::puts("Type 'help' for commands.\r\n\r\n");
        
        while running == 1 {
            // Print prompt
            runtime_console::puts(PROMPT);
            
            // Read line
            let line = runtime_console::readline(MAX_LINE);
            
            // Skip empty
            if dustlib::str::len(line) > 0 {
                // Parse command
                let cmd = shell_parser::parse(line)?;
                
                // Execute
                let result = shell_executor::execute(cmd);
                
                // Handle exit
                if cmd.name == "exit" {
                    runtime_console::puts("Goodbye!\r\n");
                    running = 0;
                }
            }
        }
        
        K[Unit]
    }
}
```

---

## 4. Implementation Phases

### Phase 1: xdv-boot (Weeks 1-2)

| Week | Deliverable | Dependencies Used |
|------|-------------|-------------------|
| 1 | MBR + Basic Boot | Minimal |
| 2 | Disk I/O + XDVFS Mount | dustlib_k::io |

**Milestone:** Bootloader can read from disk using dustlib_k

### Phase 2: xdv-xdvfs (Weeks 3-4)

| Week | Deliverable | Dependencies Used |
|------|-------------|-------------------|
| 3 | K-File Operations | dustlib, dustlib_k::memory |
| 4 | Directory + Mount | dustlib::Result, dustlib::Option |

**Milestone:** Full K-domain filesystem functional

### Phase 3: xdv-runtime (Weeks 5-6)

| Week | Deliverable | Dependencies Used |
|------|-------------|-------------------|
| 5 | Memory, String, IO | dustlib, dustlib_k |
| 6 | Process, Scheduler | dustlib_k::threading |

**Milestone:** DPL user space can run processes

### Phase 4: xdv-shell (Weeks 7-8)

| Week | Deliverable | Dependencies Used |
|------|-------------|-------------------|
| 7 | Shell Core + Parser | dustlib::str, dustlib::result |
| 8 | Built-in Commands | All libraries |

**Milestone:** Interactive shell works

### Phase 5: Integration (Weeks 9-10)

| Week | Deliverable |
|------|-------------|
| 9 | Boot to Shell |
| 10 | Testing + CI |

**Milestone:** Complete bootable system

---

## 5. File Organization

```
xdv-os/
├── README.md
├── LICENSE
├── State.toml
├── .github/
│   └── workflows/
│       └── ci.yml
├── xdv-boot/
│   ├── State.toml
│   └── src/
│       ├── boot_mbr.ds
│       ├── boot_stage1.ds
│       ├── boot_gdt.ds
│       ├── boot_idt.ds
│       ├── boot_paging.ds
│       ├── boot_disk.ds
│       ├── boot_xdvfs_mount.ds
│       └── boot_kernel_load.ds
├── xdv-kernel/           # EXISTING (13 sectors)
│   ├── State.toml
│   └── sector/
├── xdv-xdvfs/
│   ├── State.toml
│   └── src/
│       ├── xdvfs_superblock.ds
│       ├── xdvfs_inode.ds
│       ├── xdvfs_block_alloc.ds
│       ├── xdvfs_k_file.ds
│       ├── xdvfs_q_types.ds
│       ├── xdvfs_phi_types.ds
│       ├── xdvfs_directory.ds
│       ├── xdvfs_mount.ds
│       └── xdvfs_errors.ds
├── xdv-runtime/
│   ├── State.toml
│   └── src/
│       ├── runtime_io.ds
│       ├── runtime_memory.ds
│       ├── runtime_string.ds
│       ├── runtime_process.ds
│       ├── runtime_scheduler.ds
│       ├── runtime_fs.ds
│       ├── runtime_console.ds
│       └── runtime_init.ds
└── xdv-shell/
    ├── State.toml
    └── src/
        ├── shell_main.ds
        ├── shell_loop.ds
        ├── shell_prompt.ds
        ├── shell_lexer.ds
        ├── shell_parser.ds
        ├── shell_executor.ds
        ├── shell_builtin/
        │   ├── builtin_cd.ds
        │   ├── builtin_ls.ds
        │   ├── builtin_cat.ds
        │   ├── builtin_mkdir.ds
        │   ├── builtin_rm.ds
        │   ├── builtin_echo.ds
        │   ├── builtin_ps.ds
        │   ├── builtin_help.ds
        │   └── builtin_exit.ds
        └── shell_completion.ds
```

---

## 6. Library Usage Summary

| Component | dustlib | dustlib_k |
|-----------|---------|-----------|
| **xdv-boot** | - | io, memory |
| **xdv-kernel** | (existing) | (existing) |
| **xdv-xdvfs** | Result, Option, Str | memory, io |
| **xdv-runtime** | Result, Option, Str | io, memory, threading |
| **xdv-shell** | Result, Option, Str | io |

---

## 7. Summary

This implementation uses **dustlib** and **dustlib_k** for K-Domain only operation:

| Phase | Component | Duration | Status |
|-------|-----------|----------|--------|
| 1 | xdv-boot | 2 weeks | Boot from disk |
| 2 | xdv-xdvfs | 2 weeks | K-files work |
| 3 | xdv-runtime | 2 weeks | User space |
| 4 | xdv-shell | 2 weeks | CLI works |
| 5 | Integration | 2 weeks | Boot to shell |

**Total: 10 weeks**

**Key Points:**
- Full K-Domain (classical) functionality
- Q/Φ types defined but not functional (domain awareness only)
- Uses dustlib (core types) and dustlib_k (systems programming)
- All code in DPL with no external dependencies

---

*Document Version: 4.0*  
*Date: February 2026*  
*Uses: dustlib, dustlib_k*
