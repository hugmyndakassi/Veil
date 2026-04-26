---
name: veil-update
description: >
  Synchronize Veil headers with the latest phnt (Process Hacker Native API) changes.
  Use this skill whenever the user wants to update Veil from upstream phnt, sync NT
  kernel headers, apply phnt diffs, convert PHNT macros to NTDDI macros, add
  __kernel_entry/Zw* function pairs, or generally keep Veil in step with
  winsiderss/systeminformer. Also use when the user mentions "update Veil", "sync
  phnt", "apply phnt changes", "VERSION_COMMIT", or asks to refresh Veil headers
  from upstream.
---

# Veil Update Skill

## Overview

Veil is a Windows native API header library that synchronizes with
[phnt](https://github.com/winsiderss/systeminformer/tree/master/phnt) and adapts
it to Veil's coding conventions.

## Prerequisites

| Component | Value |
|---|---|
| Build script | `BuildAllTargets.cmd` in `Veil.Test/` |
| Pre-build cleanup | Kill zombie MSBuild: `taskkill /f /im MSBuild.exe` |
| Current upstream | Read `VERSION_COMMIT` (root, hash only) |

## Update Workflow

### Phase 1 — Prepare

1. Read `VERSION_COMMIT` for the current upstream commit hash.
2. Sparse-clone or pull `winsiderss/systeminformer` into `.cache/systeminformer/`.
3. Run `git diff <old> <new> -- phnt/include/` to generate per-file diffs.
4. Identify which phnt files changed and map them to Veil headers (see File Mapping).

### Phase 2 — Apply

For each changed phnt file that has a Veil mapping:

1. Read the diff section and the latest phnt source.
2. Read the current Veil header.
3. Apply new additions (functions, structs, enums, macros, typedefs) that don't
   already exist in Veil.
4. **Do not** remove existing Veil content — only add or update.
5. Handle each change type as described below.

### Phase 3 — Convert

#### Macro Conversion

Replace all `PHNT_*` macros with Veil equivalents:

| phnt Macro | Veil Macro |
|---|---|
| `PHNT_VERSION` | `NTDDI_VERSION` |
| `PHNT_WINDOWS_OLDEST` | `NTDDI_WIN2K` |
| `PHNT_WINDOWS_XP` | `NTDDI_WINXP` |
| `PHNT_WINDOWS_SERVER_2003` | `NTDDI_WS03` |
| `PHNT_WINDOWS_VISTA` | `NTDDI_VISTA` |
| `PHNT_WINDOWS_7` | `NTDDI_WIN7` |
| `PHNT_WINDOWS_8` | `NTDDI_WIN8` |
| `PHNT_WINDOWS_8_1` | `NTDDI_WINBLUE` |
| `PHNT_WINDOWS_10` | `NTDDI_WIN10` |
| `PHNT_WINDOWS_10_TH2` | `NTDDI_WIN10_TH2` |
| `PHNT_WINDOWS_10_RS1` | `NTDDI_WIN10_RS1` |
| `PHNT_WINDOWS_10_RS2` | `NTDDI_WIN10_RS2` |
| `PHNT_WINDOWS_10_RS3` | `NTDDI_WIN10_RS3` |
| `PHNT_WINDOWS_10_RS4` | `NTDDI_WIN10_RS4` |
| `PHNT_WINDOWS_10_RS5` | `NTDDI_WIN10_RS5` |
| `PHNT_WINDOWS_10_19H1` | `NTDDI_WIN10_19H1` |
| `PHNT_WINDOWS_10_19H2` | `NTDDI_WIN10_19H1` |
| `PHNT_WINDOWS_10_20H1` | `NTDDI_WIN10_VB` |
| `PHNT_WINDOWS_10_20H2` | `NTDDI_WIN10_MN` |
| `PHNT_WINDOWS_10_21H1` | `NTDDI_WIN10_FE` |
| `PHNT_WINDOWS_10_21H2` | `NTDDI_WIN10_CO` |
| `PHNT_WINDOWS_10_22H2` | `NTDDI_WIN10_CO` |
| `PHNT_WINDOWS_11` | `NTDDI_WIN11` |
| `PHNT_WINDOWS_11_22H2` | `NTDDI_WIN11_NI` |
| `PHNT_WINDOWS_11_23H2` | `NTDDI_WIN11_NI` |
| `PHNT_WINDOWS_11_24H2` | `NTDDI_WIN11_GE` |
| `PHNT_WINDOWS_NEW` | `NTDDI_WIN11_BR` |

**Example:**
```cpp
// phnt:        #if (PHNT_VERSION >= PHNT_WINDOWS_10_RS1)
// Veil after:  #if (NTDDI_VERSION >= NTDDI_WIN10_RS1)
```

#### Nt Function Formatting

Add `__kernel_entry` before `NTSYSCALLAPI` for all `Nt*` syscall declarations.
Ensure the closing `);` has no leading spaces.

**Before (phnt):**
```cpp
NTSYSCALLAPI
NTSTATUS
NTAPI
NtFlushKey(
    _In_ HANDLE KeyHandle
    );
```

**After (Veil):**
```cpp
__kernel_entry NTSYSCALLAPI
NTSTATUS
NTAPI
NtFlushKey(
    _In_ HANDLE KeyHandle
);
```

#### Zw Function Pair

Immediately after each `Nt*` function, add the corresponding `Zw*` function:

```cpp
_IRQL_requires_max_(PASSIVE_LEVEL)
NTSYSAPI
NTSTATUS
NTAPI
ZwFlushKey(
    _In_ HANDLE KeyHandle
);
```

Use `_IRQL_requires_max_(PASSIVE_LEVEL)` and `NTSYSAPI` for Zw functions.

#### Other Conversions

- `static_assert` → `STATIC_ASSERT`
- Replace phnt-style `_Kernel_entry_` → Veil `__kernel_entry`
- Sync Zw parameter annotation fixes from upstream when applicable

### Phase 4 — Validate

After **each file update**, run the build:

```bash
taskkill /f /im MSBuild.exe
Veil.Test/BuildAllTargets.cmd
```

Fix compilation errors before proceeding to the next file.

### Phase 5 — Finalize

1. Update `VERSION_COMMIT` with the new upstream commit hash (hash only, no URL or date).
2. Report changes from `ntmisc.h` separately to the user.
3. Clean up `.cache/` directory.

## File Mapping

| phnt Source | Veil Header |
|---|---|
| `ntlpcapi.h` | `Veil/Veil.System.ALPC.h` |
| `ntbcd.h` | `Veil/Veil.System.BCD.h` |
| `ntregapi.h` | `Veil/Veil.System.ConfigurationManager.h` |
| `ntdbg.h` | `Veil/Veil.System.Debug.h` |
| `ntwmi.h` | `Veil/Veil.System.Etw.h` |
| `ntexapi.h` | `Veil/Veil.System.Executive.h` |
| `ntioapi.h` | `Veil/Veil.System.IOManager.h` |
| `ntkeapi.h` | `Veil/Veil.System.KernelCore.h` |
| `ntldr.h` | `Veil/Veil.System.Loader.h` |
| `ntmmapi.h` | `Veil/Veil.System.MemoryManager.h` |
| `ntnls.h` | `Veil/Veil.System.Nls.h` |
| `ntobapi.h` | `Veil/Veil.System.ObjectManager.h` |
| `ntpnpapi.h` | `Veil/Veil.System.PNP.h` |
| `ntpoapi.h` | `Veil/Veil.System.PowerManager.h` |
| `ntpfapi.h` | `Veil/Veil.System.Prefetcher.h` |
| `ntpsapi.h` | `Veil/Veil.System.Process.h` |
| `ntrtl.h` | `Veil/Veil.System.RuntimeLibrary.h` |
| `ntsam.h` | `Veil/Veil.System.SAM.h` |
| `ntseapi.h` | `Veil/Veil.System.Security.h` |
| `ntsmss.h` | `Veil/Veil.System.SMSS.h` |
| `ntsxs.h` | `Veil/Veil.System.SxS.h` |
| `nttp.h` | `Veil/Veil.System.ThreadPool.h` |
| `nttmapi.h` | `Veil/Veil.System.TransactionManager.h` |
| `ntmisc.h` | `Veil/Veil.System.AppPackage.h` (Package APIs only) |

**Unmapped** (skip): `nttypesafe.h`, `ntstrsafe.h`, `ntintsafe.h`, `ntd3dkmt.h`

## Code Conventions

### File Header

Every Veil header starts with:
```cpp
/*
 * PROJECT:   Veil
 * FILE:      Veil.System.<Module>.h
 * PURPOSE:   This file is part of Veil.
 *
 * LICENSE:   MIT License
 *
 * DEVELOPER: MeeSong (meesong@outlook.com)
 */

#pragma once

// Warnings which disabled for compiling
#if _MSC_VER >= 1200
#pragma warning(push)
#pragma warning(disable:4201)
#pragma warning(disable:4214)
#pragma warning(disable:4324)
#pragma warning(disable:4471)
#endif

VEIL_BEGIN()

// ... content ...

VEIL_END()

#if _MSC_VER >= 1200
#pragma warning(pop)
#endif
```

### Kernel/User Mode Guards

```cpp
#ifndef _KERNEL_MODE
// User-mode only definitions
#endif // !_KERNEL_MODE

#ifdef _KERNEL_MODE
// Kernel-mode only definitions
#endif // _KERNEL_MODE
```

### Enum Definition

```cpp
/**
 * Brief description.
 * \sa https://learn.microsoft.com/...
 */
typedef enum _ENUM_NAME
{
    EnumValue1, // ASSOCIATED_STRUCTURE_1
    EnumValue2, // ASSOCIATED_STRUCTURE_2
    MaxEnumValue
} ENUM_NAME;
```

### Structure Definition

```cpp
/**
 * Brief description.
 * \sa https://learn.microsoft.com/...
 */
typedef struct _STRUCTURE_NAME
{
    TYPE Field1;
    TYPE Field2;
    _Field_size_bytes_(Length) TYPE VariableField[1];
} STRUCTURE_NAME, *PSTRUCTURE_NAME;
```

## Verification Checklist

- [ ] All `PHNT_WINDOWS_*` macros converted to `NTDDI_*`
- [ ] All `Nt*` functions have `__kernel_entry` attribute
- [ ] All `Nt*` functions have corresponding `Zw*` pairs
- [ ] Closing parentheses aligned (no leading spaces before `);`)
- [ ] No compilation errors in `BuildAllTargets.cmd`
- [ ] Definition order respects dependencies
- [ ] `ntmisc.h` changes reported separately
- [ ] `VERSION_COMMIT` updated
- [ ] `.cache/` cleaned up

## Error Handling

| Issue | Cause | Solution |
|---|---|---|
| Undefined type | Missing dependency | Check definition order; move definition earlier |
| Redefinition | Duplicate in SDK/WDK | Wrap with `#ifndef _KERNEL_MODE` or version check |
| Missing SAL annotation | phnt inconsistency | Add `_In_`, `_Out_`, `_Inout_` as appropriate |
| Macro conflict | Name collision | Use `VEIL_` prefix or conditional compilation |

## Communication

- All interactions with the user: **Chinese**
- All documents/AI-readable content: **English**
