---
title: Sandbox
weight: 930
description: >
  Documentation about the FiveM sandbox restrictions and allowed APIs.
---

FiveM implements a security sandbox for resources to ensure stability and prevent malicious code execution. This document describes the restrictions and allowed APIs within this sandbox environment.

## File System Operations

### Path Types
All file system operations support two types of paths:
1. Resource mount paths (recommended): `@resourceName/path`
2. Absolute paths: `/absolute/path/to/resource`

Resource mount paths are recommended as they're cleaner and more portable.

### Directory Listing
The `io.readdir()` function returns files and folders within a directory:

```lua
-- Using mount paths (recommended)
local files1 = io.readdir("@myResource/")
local files2 = io.readdir("@myResource/assets")

-- Using absolute paths
local files3 = io.readdir("/absolute/path/to/resource")
local files4 = io.readdir("/absolute/path/to/resource/assets")
```

### File Operations
```lua
-- Using mount paths (recommended)
local file1 = io.open("@myResource/data.json", "r")

-- Using absolute paths
local file2 = io.open("/absolute/path/to/resource/data.json", "r")

-- Available modes
"r"  -- Read
"w"  -- Write
"a"  -- Append
"+"  -- Create
```

#### Write Restrictions
Writing certain file types across resources is blocked to prevent code injection:
- `.lua` - Lua source files
- `.dll` - Dynamic Link Libraries
- `.ts` - TypeScript files
- `.js`, `.mjs`, `.cjs` - JavaScript files
- `.cs` - C# source files

Note: Writing these files is allowed within the same resource, but blocked across different resources.

The `SaveResourceFile` native function follows these same restrictions.

### System Path Restrictions
- Operations in the server main folder are blocked
- Operations outside resource folders are blocked
- Blocked operations return error code 13 and "Permission denied"

## OS Operations

### Time Functions
New time measurement functions are available:
```lua
local nano = os.nanotime()   -- Nanosecond precision
local micro = os.microtime() -- Microsecond precision
local delta = os.deltatime() -- Time difference
local tsc = os.rdtsc()       -- Read Time-Stamp Counter
local tscp = os.rdtscp()     -- Read Time-Stamp Counter and Processor ID
```

### Restricted Operations
```lua
-- Blocked with "Permission denied" (code 13)
os.execute("command")  -- All commands blocked
io.tmpfile()          -- Temporary files blocked
io.popen("whoami")    -- Only emulated 'ls' and 'dir' allowed

-- Limited functionality
os.getenv("os")       -- Returns "Windows" or "Linux"
os.getenv("PATH")     -- Returns nil
os.setlocale()        -- Returns current locale, cannot modify
```

### Allowed Operations
```lua
-- These operations remain available
load()                -- Not blocked
loadfile()           -- Not blocked
dofile()             -- Not blocked
```

## Debug Namespace Restrictions

The `debug` namespace is sandboxed on the server to prevent unauthorized access to the Lua environment. Please refer to the FiveM natives and APIs documentation for debugging capabilities.

## Manifest Modifications

### Unsafe Manifest Modifications
To allow resources to modify other resources' manifests, you'll need to use the `sv_unsafeManifestModifications` configuration. This should only be used when you trust the resources and their authors.

```lua
sv_unsafeManifestModifications = {
    "myresourceone",
    "myresourcetwo"
}
```

### Security Considerations
For manifest modifications to work:
1. Both resources must share the same author name in their manifests
2. Both resources must properly declare their dependencies
3. You must fully trust the resources and their authors

{{% alert theme="warning" %}}
Enabling manifest modifications can be dangerous if misused. Only enable this for resources you trust completely and understand their modifications.
{{% /alert %}}
