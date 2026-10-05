<div align="center">

# 运行时工具

[简体中文](README.md) | [English](README_EN.md)

</div>

提取和打包需要以下文件：

| 文件 | 说明 |
| --- | --- |
| `retoc.exe` | 版本 `0.1.5`，已随工具包提供 |
| `oo2core_9_win64.dll` | 必须从自己合法拥有的环境中取得 |

当 `config.json` 使用 `"aesKey": "auto"` 时，还需要自行提供 `aes-dumper.exe`。也可以配置合法取得的 32 字节十六进制密钥，从而无需扫描器。

公开发布的 AC8 Override Toolkit 会有意省略 Oodle 与 `aes-dumper.exe`。详情参见 [`DISTRIBUTION.md`](../DISTRIBUTION.md) 和 [`THIRD_PARTY.md`](../THIRD_PARTY.md)。
