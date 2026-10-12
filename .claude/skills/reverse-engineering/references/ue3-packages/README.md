# Unpacking fully-compressed UE3 `.upk` packages

Some UE3 games ship **fully compressed (LZO)** packages that refuse the usual tools:

- **UELib** parses the header but its package decompressor is a stub (`CompressedChunkBlock.Decompress`
  just returns the size), so `UnrealPackage.Deserialize` returns early and you get `Exports=0`.
- **UPKUtils `DecompressLZO.exe`** works for XCOM-class packages but rejects others
  (Dishonored build 801/30 -> "Bad package version!").

`unpack_upk.py` decodes them directly.

## Format (verified: Dishonored, build 801/30)

```
header @0:  tag 0x9E2A83C1 | FileVersion | LicenseeVersion | HeaderSize | ... tables
            CompressionFlags(4) | CompressedChunks count(4) | chunk table[]
chunk table: 16-byte records — (UncompressedOffset, UncompressedSize, CompressedOffset, CompressedSize)
each chunk @CompressedOffset:
    FCompressedChunkHeader { Tag(4)=0x9E2A83C1, ChunkSize(4)=0x20000,
                             summary { CompSize(4), UncompSize(4) },
                             N * { CompSize(4), UncompSize(4) } }
    then N LZO blocks (CompressionFlags 2 = LZO; 1 = ZLIB).
```

The `CompressionFlags`/chunk-table offset **varies per package** (it depends on earlier summary
fields), so the script auto-detects it by checking that chunks chain correctly.

## Use

```bash
uv run --python 3.12 --with minilzo python unpack_upk.py <in.upk> <out.upk>
```

Then, to read it with UELib, clear **only** the `CompressionFlags` dword (leave the chunk
count/table in place) so `UnrealPackage.Deserialize` no longer returns early:

```python
import struct
p = 'DishonoredGame.fixed.upk'; OFF = 0x79   # printed by the script
d = bytearray(open(p, 'rb').read()); struct.pack_into('<I', d, OFF, 0)
open(p, 'wb').write(d)
```

After that, `Exports` populates and you can dump classes/functions with UELib (see the
`dumper/` note in the Dishonored co-op project for a ready C# dumper).

## Notes

- Depends on the pypi `minilzo` package (pure-python LZO1X), installed by `uv`.
- Extracted assets/scripts are the user's own game data — do not redistribute.
