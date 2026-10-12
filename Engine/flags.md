# Flag atlases (`flagfiles*.tga`)

How the game builds, caches and loads the flag atlases, and why they cost so much memory.

Source: research of October 2026. The memory saving of DDS flag caches was first found by hand
(convert the caches to DDS keeping the `.tga` names, set them
read-only). Marks: **[C]** read in the code or the files, **[I]** inferred.

## Summary

- Flags are drawn from one atlas per government type: `gfx/flags/flagfiles.tga`,
  `flagfiles_<government>.tga` (+ `.txt` index), built from the individual `gfx/flags/TAG[_gov].tga` and
  cached in the user folder (`Documents/Paradox Interactive/Victoria II/<mod>/gfx/flags/`).
- Vanilla: 5 atlases of 2048×1024. TGC: **11 atlases of 4096×2048**, A8R8G8B8, 32 MB each. **[C]** (file
  headers)

## `FlagAtlas_BuildOrLoad` (0x007E3900) **[C]**

| site | call | arguments |
|---|---|---|
| `0x007E55CC` | `D3DXCreateTextureFromFileInMemoryEx` | individual flags: size `-2` (non-power-of-2), mips from file, Format UNKNOWN, Pool MANAGED |
| `0x007E5ADA` | `D3DXCreateTexture` | the atlas being built: A8R8G8B8 (`0x15`), 1 mip, MANAGED |
| `0x007E629C` | `D3DXCreateTexture` | a copy for saving: A8R8G8B8, MANAGED |
| `0x007E62CC` | `D3DXLoadSurfaceFromSurface` | atlas -> copy |
| `0x007E6367` | `D3DXSaveTextureToFileInMemory` | `D3DXIFF_TGA` (2); the map cache (`FUN_00803210`, `0x00803A52`) saves `D3DXIFF_DDS` (4) |
| `0x007E6492` | `D3DXCreateTextureFromFileInMemoryEx` | the cache file read through PhysFS (after a rebuild and on every later start): Format UNKNOWN, Pool MANAGED |

All D3DX calls are `CALL [import]` (`FF 15`).

- **Why it costs RAM:** `D3DPOOL_MANAGED` keeps a system-memory copy of every texture inside the process,
  so each TGC atlas pins 32 MB of the address space for the whole session (~350 MB for 11). **[I]** (D3D9
  behaviour)
- **Why DDS caches named `.tga` work:** the cache load passes Format UNKNOWN, so D3DX takes the format
  from the file *content*, not the extension; a DDS/DXT file named `.tga` loads as a compressed texture.
- **Flags have no alpha:** every 4×4 block of the TGC atlases is fully opaque (checked on DXT5
  conversions of all 11 atlases), so DXT1 (0.5 byte/pixel, 4 MB per TGC atlas) loses nothing over DXT5.
  **[C]**
