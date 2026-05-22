# NVIDIA 470.256.02 Driver — Kernel 7.0 Build Progress

## System Info
- **GPU**: NVIDIA GeForce GT 720 (GK208, Kepler)
- **Current driver**: nouveau (open-source, poor performance)
- **Target driver**: nvidia-470xx-dkms (470.256.02) — last proprietary driver supporting Kepler
- **Kernel**: 7.0.9-arch1-1
- **OS**: Arch Linux
- **AUR packages**: `nvidia-470xx-dkms`, `nvidia-470xx-utils`, `nvidia-470xx-settings`

## Key Problem
nvidia-470xx AUR packages have patches only up to kernel 6.19. Kernel 7.0 broke compatibility. The conftest system (which auto-detects kernel API) is also broken on 7.0 due to static_assert failures in kernel headers.

## Build Status: IN PROGRESS
- `nvidia.ko` (main driver) — **COMPILES SUCCESSFULLY** (only warnings)
- `nvidia-uvm.ko` (UVM module) — **FAILS** with `NV_WAIT_ON_BIT_LOCK_ARGUMENT_COUNT` undef error

## Source Code Changes (kernel source tree)

### kernel/Kbuild
- Replaced all `EXTRA_CFLAGS` with `ccflags-y` (kernel 7.0 removed EXTRA_CFLAGS)

### kernel/conftest.sh
- Added 8-arg `acpi_walk_namespace` test (before 7-arg test)
- Changed `wait_on_bit_lock` from `#error` to `#undef`
- Changed `radix_tree_replace_slot` from `#error` to `#undef`

### kernel/common/inc/nv-time.h
- Added `#include <linux/version.h>`
- Added `#define in_irq in_hardirq` for kernel >= 5.11

### kernel/common/inc/nv-linux.h
- Added `NV_ACPI_WALK_NAMESPACE_ARGUMENT_COUNT == 8` case
- Added `dma_is_direct` fallback for kernel >= 7.0: `if (get_dma_ops(dev) == NULL) is_direct = NV_TRUE;`
- Added `f_inode` fallback for kernel >= 3.9 when conftest fails
- Updated `NV_SET_CLOSE_ON_EXEC` macro for kernel 7.0 signature change: `set_close_on_exec(fd, 1)` instead of `set_close_on_exec(fd, fdt)`
- Changed `__set_close_on_exec` to `set_close_on_exec`

### kernel/nvidia/nv.c
- Added `#define del_timer_sync(timer) timer_delete_sync(timer)` for kernel >= 7.0

### kernel/nvidia/os-mlock.c
- Added `follow_pfnmap_start/end` fallback for kernel >= 7.0 (replaces removed `follow_pfn`/`unsafe_follow_pfn`)

### kernel/nvidia-uvm/uvm_linux.h
- Changed `wait_on_bit_lock` fallback to use 3-arg version (function still exists in 7.0)
- Changed `radix_tree_replace_slot` to use `#ifdef` instead of `#if` for undef-safe check
- Fixed `-Werror=undef` issues by wrapping in `#ifdef` guards

## Conftest Manual Fixes (compile-tests/*.h + types.h + functions.h)

The conftest compile tests are broken on kernel 7.0 (static_assert failures in kernel headers prevent detection). All results were manually corrected:

### Fixed to #define (were incorrectly #undef)
| Macro | Value | Notes |
|-------|-------|-------|
| `NV_FILE_HAS_INODE` | defined | `f_inode` exists since 3.9 |
| `NV_KUID_T_PRESENT` | defined | `kuid_t` exists since 3.5 |
| `NV_VM_FAULT_HAS_ADDRESS` | defined | `vm_fault.address` since 4.10 |
| `NV_VM_FAULT_T_IS_PRESENT` | defined | `vm_fault_t` since 3.10 |
| `NV_MM_HAS_MMAP_LOCK` | defined | `mmap_lock` since 5.8 |
| `NV_PROC_OPS_PRESENT` | defined | `proc_ops` since 5.6 |
| `NV_TIMESPEC64_PRESENT` | defined | `timespec64` since 4.18 |
| `NV_VM_AREA_STRUCT_HAS_CONST_VM_FLAGS` | defined | const `vm_flags` since 6.4 |
| `NV_VM_OPS_FAULT_REMOVED_VMA_ARG` | defined | vma arg removed in 4.17 |
| `NV_GET_USER_PAGES_HAS_ARGS_FLAGS` | defined | 4-arg with gup_flags |
| `NV_KERNEL_WRITE_HAS_POINTER_POS_ARG` | defined | `loff_t *` pos arg |
| `NV_KERNEL_READ_HAS_POINTER_POS_ARG` | defined | `loff_t *` pos arg |
| `NV_EFI_ENABLED_ARGUMENT_COUNT` | 1 | 1-arg version |
| `NV_FULL_NAME_HASH_ARGUMENT_COUNT` | 3 | `salt, name, length` |
| `NV_WAIT_ON_BIT_LOCK_ARGUMENT_COUNT` | 3 | 3-arg version (word, bit, mode) |
| `NV_ACPI_WALK_NAMESPACE_ARGUMENT_COUNT` | 7 | 7-arg version |
| `NV_GET_USER_PAGES_REMOTE_PRESENT` | defined | function exists |
| `NV_GET_USER_PAGES_REMOTE_HAS_ARGS_FLAGS_LOCKED` | defined | 6-arg version |
| `NV_PNV_NPU2_INIT_CONTEXT_PRESENT` | undef | PowerPC-only, not on x86 |

### Fixed to #undef (were incorrectly #define)
| Macro | Notes |
|-------|-------|
| `NV_ACPI_BUS_GET_DEVICE_PRESENT` | removed in 7.0 |
| `NV_PHYS_TO_DMA_PRESENT` | removed in 7.0 |
| `NV_DMA_IS_DIRECT_PRESENT` | removed in 7.0 |
| `NV_DMA_MAP_RESOURCE_PRESENT` | removed from dma_map_ops in 7.0 |
| `NV_SET_MEMORY_ARRAY_UC_PRESENT` | removed in 7.0 |
| `NV_JIFFIES_TO_TIMESPEC_PRESENT` | removed in 7.0 |
| `NV_ACQUIRE_CONSOLE_SEM_PRESENT` | removed in 7.0 |
| `NV_UNSAFE_FOLLOW_PFN_PRESENT` | removed in 7.0 |

## What's Left

### Immediate: Fix UVM compile error
The UVM module fails because `NV_WAIT_ON_BIT_LOCK_ARGUMENT_COUNT` is undefined in the conftest header chain that UVM uses. The conftest file at `conftest/compile-tests/wait_on_bit_lock.h` is correct but UVM might not be including it properly.

**Fix needed**: The `#if` checks in `uvm_linux.h` at lines 489/492 use `-Werror=undef`. Need to wrap in `#ifdef` guards OR ensure the conftest is included before `uvm_linux.h`.

### After UVM compiles
- Test `nvidia.ko` loads: `sudo insmod /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/nvidia.ko`
- Create DKMS package for auto-rebuild on kernel updates
- Remove nouveau: `sudo pacman -R nouveau`
- Install nvidia: `sudo pacman -S nvidia-470xx-dkms nvidia-470xx-utils nvidia-470xx-settings`
- Configure picom to use GLX backend properly

## Useful Commands
```bash
# Build nvidia kernel module
cd /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel
SYSSRC=/usr/lib/modules/7.0.9-arch1-1/build make modules

# Check for errors only
SYSSRC=/usr/lib/modules/7.0.9-arch1-1/build make modules 2>&1 | grep "error:"

# Check conftest results
cat /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/conftest/functions.h
cat /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/conftest/types.h

# Check if a function exists in kernel 7.0
grep -rn "function_name" /usr/lib/modules/7.0.9-arch1-1/build/include/ 2>/dev/null

# Clean and rebuild
cd /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel
SYSSRC=/usr/lib/modules/7.0.9-arch1-1/build make clean

# Test nvidia module loads (BEFORE removing nouveau)
sudo insmod /home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/nvidia.ko

# Remove nouveau (AFTER nvidia works)
sudo pacman -R nouveau

# Install from AUR
yay -S nvidia-470xx-dkms nvidia-470xx-utils nvidia-470xx-settings
```

## File Locations
- **Driver source**: `/home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/`
- **Kernel module source**: `/home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/`
- **Patches**: `/home/dev/gpu-setup/*.patch`
- **PKGBUILD**: `/home/dev/gpu-setup/PKGBUILD`
- **Conftest results**: `/home/dev/gpu-setup/build/NVIDIA-Linux-x86_64-470.256.02/kernel/conftest/`
- **Progress file**: `/home/dev/gpu-setup/PROGRESS.md`
- **Modified source files**:
  - `kernel/Kbuild` — EXTRA_CFLAGS → ccflags-y
  - `kernel/conftest.sh` — acpi_walk_namespace, wait_on_bit_lock, radix_tree_replace_slot
  - `kernel/common/inc/nv-linux.h` — acpi 8-arg, dma_is_direct, f_inode fallback, set_close_on_exec
  - `kernel/common/inc/nv-time.h` — in_irq → in_hardirq, added version.h include
  - `kernel/nvidia/nv.c` — del_timer_sync → timer_delete_sync
  - `kernel/nvidia/os-mlock.c` — follow_pfnmap_start/end fallback
  - `kernel/nvidia-uvm/uvm_linux.h` — wait_on_bit_lock, radix_tree_replace_slot fallbacks
