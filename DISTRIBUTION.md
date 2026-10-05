# Distribution notice

Two release forms are intentionally separated:

- `LocalFull` may contain the locally obtained `oo2core_9_win64.dll` and
  `aes-dumper.exe` used for validation. It is for the current developer's local
  use and must not be uploaded or redistributed as-is.
- The public package omits both files. Each developer must provide a legally
  obtained Oodle runtime locally. They may either configure a legally obtained
  AES key in `config.json` or place their own `aes-dumper.exe` in `tools` and
  keep `aesKey` set to `auto`.

The public source repository also excludes game assets, AES keys, Oodle,
aes-dumper binaries, extracted workspaces, mappings generated from unsupported
game versions, and compiled project binaries.
