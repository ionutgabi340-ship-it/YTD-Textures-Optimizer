# YTDToolio

Command-line tool for GTA V / FiveM texture dictionaries (`.ytd`). Unpack textures
to PNG, edit them with anything, pack them back.

Built for bulk texture optimisation — resizing clothing and map textures across
hundreds of files without opening a GUI once.

```
YTDToolio unpack shirt.ytd -d out/     # textures -> PNG
# resize the PNGs however you like
YTDToolio pack out/ -d shirt_small.ytd # PNG -> .ytd
```

---

## Why this repo exists

This is based on [kngrektor/ytdtool](https://github.com/kngrektor/ytdtool), which
has been unmaintained since 2021, ships no binaries, and contains a bug that makes
`pack` produce corrupted output. Both are fixed here.

**The bug:** `fuckdx/main.cpp` declared `encode()`'s source buffer as `uint8_t src[]`
but indexed it with pixel arithmetic:

```cpp
memcpy(&quad[4*y], &src[4*by*w + y*w + 4*bx], 4*sizeof(Pixel));
```

For a 4-byte-per-pixel buffer, the byte offset of row `4*by + y`, column `4*bx` is
`16*by*w + 4*y*w + 16*bx` — four times what was computed. Every block gathered its
pixels from the wrong place, so repacked dictionaries came out visibly smeared.

`decode()` used identical arithmetic but declared `Pixel dst[]`, which is correct.
That asymmetry is why `unpack` always produced good PNGs while `pack` did not.

The fix is a one-word change to the parameter type. `Pixel*` and `uint8_t*` are both
pointers, so the P/Invoke signature is unaffected.

---

## Install

Download the zip from [Releases](../../releases), extract anywhere, run.

Requires the [.NET 5 runtime](https://dotnet.microsoft.com/download/dotnet/5.0)
(Windows x64).

To call it from any folder, add the extracted directory to your PATH:

```powershell
$dir = "C:\path\to\extracted"
$old = [Environment]::GetEnvironmentVariable('Path','User')
if ($old -notlike "*$dir*") { [Environment]::SetEnvironmentVariable('Path',"$old;$dir",'User') }
```

Open a new terminal, then `YTDToolio` should print its usage.

---

## Usage

### List

```
YTDToolio list <file.ytd>
```

```
Loaded shirt.ytd:
  accs_diff_004_a_uni 1024x1024 in D3DFMT_DXT5 with 1 levels
lvl 0 is 1048576 bytes
```

Shows every texture in the dictionary with its dimensions, compression format and
mip count. Non-destructive — use it to audit a pack before changing anything.

### Unpack

```
YTDToolio unpack <file.ytd> -d <folder>
```

Writes one `.png` per texture, plus a small `.png.json` sidecar per texture:

```json
{"Format":894720068,"MipMapLevels":1}
```

`Format` is the D3D FourCC as an integer (`894720068` = `0x35545844` = `DXT5`).
**Keep the sidecars** — `pack` reads them to know how to re-encode. Dimensions are
read from the PNG itself, so resizing an image is all you need to do to change a
texture's size.

### Pack

```
YTDToolio pack <folder> -d <file.ytd>
```

Reads every PNG in the folder, re-encodes using the format from each sidecar, and
writes a new dictionary. Texture names inside the dictionary come from the PNG
filenames — don't rename them, `.ydd` meshes reference those names.

---

## Supported formats

| Format | BC | Status |
|---|---|---|
| DXT1 | BC1 | supported |
| DXT5 | BC3 | supported |
| ATI1 | BC4 | supported |
| ATI2 | BC5 | supported |
| DXT3 | BC2 | not supported |
| BC7 | BC7 | not supported |

DXT5 covers almost all clothing (it carries the alpha channel). BC7 textures will
fail — check with `list` first.

---

## Batch resizing

Typical workflow for halving every texture in a folder of dictionaries:

```powershell
# audit first
Get-ChildItem .\pack -Recurse -Filter *.ytd | ForEach-Object {
    YTDToolio list $_.FullName
}
```

Then unpack, resize the PNGs with whatever you like (Python/Pillow, ImageMagick,
texconv), and pack each folder back up.

A ready-made Python script for this is linked in the wiki — it walks a folder,
halves every texture with Lanczos resampling, optionally snaps to power-of-two
dimensions, and writes to a separate output folder so originals are never touched.

Two things worth knowing:

- **DXT is lossy.** Never run a resize pass over output from a previous pass —
  always start from the originals, or you compound compression artefacts.
- **Power of two matters.** Many packs ship at arbitrary sizes like 2000x1000.
  Snapping to 1024x512 costs ~5% of the pixels and samples better on the GPU.

---

## Building from source

Only needed if you want to modify it. The release binary is prebuilt.

Requires .NET SDK 8, Visual Studio Build Tools 2022 with the C++ workload, and
[premake5](https://premake.github.io/download) on PATH.

```powershell
git clone --recursive https://github.com/<you>/ytdtool
cd ytdtool

# msbuild only exists inside a Developer shell
Import-Module "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\Microsoft.VisualStudio.DevShell.dll"
Enter-VsDevShell -VsInstallPath "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools" -SkipAutomaticLocation

.\dev_win.bat
cd fuckdx
premake5 vs2022          # upstream generates vs2019, which needs the v142 toolset
msbuild build\FuckDX.sln /p:Configuration=Release /p:Platform=x64
cd ..
.\build_win.bat

Copy-Item fuckdx\build\bin\Release\FuckDX.dll ytdtoolio\bin\Release\net5.0\win-x64\publish\ -Force
```

Output lands in `ytdtoolio\bin\Release\net5.0\win-x64\publish\`.

---

## Troubleshooting

**Repacked textures look smeared or ghosted** — you have a `FuckDX.dll` built from
unpatched source. Rebuild, or use the release binary.

**`pack` succeeds but the texture is wrong** — `list` only reads metadata and will
report the expected size either way. Always verify visually in
[CodeWalker](https://github.com/dexyfex/CodeWalker).

**Alpha channel lost** — the PNG was saved without alpha, or the sidecar is missing
and it fell back to DXT1. DXT1 has no usable alpha; DXT5 does.

---

## Credits

- [kngrektor/ytdtool](https://github.com/kngrektor/ytdtool) — original implementation
- [rgbcx / bc7enc](https://github.com/richgel999/bc7enc) by Rich Geldreich — BC compression
- [gta-toolkit](https://github.com/Neodymium146/gta-toolkit) — RAGE resource format handling
- [CodeWalker](https://github.com/dexyfex/CodeWalker) by dexyfex — reference for GTA V formats

## License

See [LICENSE](LICENSE). Original copyright retained.
