<div align="center">

# Runtime Tools

[简体中文](README.md) | [English](README_EN.md)

</div>

The following files are required for extraction and packing:

| File | Description |
| --- | --- |
| `retoc.exe` | Version `0.1.5`, included with the Toolkit |
| `oo2core_9_win64.dll` | Must be obtained from an environment you legally own |

When `config.json` uses `"aesKey": "auto"`, you must also provide your own `aes-dumper.exe`. Alternatively, configure a legally obtained 32-byte hexadecimal key and omit the scanner.

Public AC8 Override Toolkit packages intentionally omit Oodle and `aes-dumper.exe`. See [`DISTRIBUTION.md`](../DISTRIBUTION.md) and [`THIRD_PARTY.md`](../THIRD_PARTY.md) for details.
