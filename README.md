# ff-lyrics-verify

FluentFlyout × TaskbarLyrics 合并前的**可行性验证仓**。目的只有一个：用可复现的 CI 证据回答「方案 B 能否开工」，不承载业务代码。

**当前结论：通过，0 个阻断项。** 完整证据与判定标准见 [`docs/verification-report.md`](docs/verification-report.md)。

## 这个仓库要证明什么

| 编号 | 待证命题 | 当前结论 |
|---|---|---|
| **V1** | `Lyricify.Lyrics.Helper`（netstandard2.1）能在 `net10.0-windows10.0.22000.0` 下加载 | PASS |
| **V2–V5** | QRC / KRC / YRC 逐字格式与 LRC 能被正确解析 | PASS |
| **V6** | KRC 解密路径（XOR + zlib）在 net10 下行为正确 | PASS（往返逐字节一致） |
| **V7** | QRC 解密路径（TripleDES + zlib）在 net10 下行为正确 | PASS（往返逐字节一致） |
| **V8** | 传递依赖（Newtonsoft.Json / SharpZipLib）能正确绑定 | PASS |
| **V9** | TLS 实际使用的网易/酷狗/LRCLIB 搜索器类型可加载 | PASS |
| **J2** | TaskbarLyrics.Core 全部 58 个源文件能在 FF 的 TFM 下编译 | PASS（x64 + ARM64，零警告） |

## 为什么这两条解密往返测试是关键

TaskbarLyrics 的公开测试样本（`TestData/Lyrics/*.qrc|*.krc|*.yrc`）全部是**明文 fixture**，走的是 Parser 而非 Decrypter。也就是说，仅跑 TL 现有测试**无法暴露**解密路径在 net10 上的问题。

本仓库的 V6/V7 按 `Decrypter.cs` 的逆过程自行构造加密数据，再交给库解密并比对原文：

- **KRC**：明文 → zlib 压缩 → XOR（16 字节密钥）→ base64（前置 4 字节头）→ 解密 → 比对
- **QRC**：明文 → zlib 压缩 → TripleDES 加密（8 字节块）→ hex → 解密 → 比对

这两条路径覆盖了 `TripleDESCrypt` 位运算、`InflaterInputStream` 解压、BOM 处理——即 net10 下最可能出问题的三处。

> 注意：构造密文必须用 `ZLibStream`（zlib 容器），不能用 `DeflateStream`（裸 deflate）。后者会让解密端抛 `Header checksum illegal`，看起来像库不兼容，实际是测试写错了。

## 本地运行

```bash
# 需要 .NET 10 SDK
dotnet restore src/FluentFlyout.Lyrics.Probe
dotnet build   src/FluentFlyout.Lyrics.Probe -c Release --no-restore -warnaserror
dotnet run     src/FluentFlyout.Lyrics.Probe -c Release --no-build
# 返回码 0 = 9/9 通过
```

## CI

`.github/workflows/verify.yml` 三个 job：

- **J1 兼容性探针** — 跑 9 项检查，`-warnaserror`，返回码即门槛，结果写入 Job Summary
- **J2 Core 移植** — x64 + ARM64 双 RID 编译 TL.Core，并输出依赖漏洞报告（仅报告不阻断）
- **J3 代码格式** — 非阻断，仅提示

## 目录

```
src/FluentFlyout.Lyrics/        TL.Core 原样移植体（58 文件 / 9,137 行，TFM 已改为 net10-windows）
src/FluentFlyout.Lyrics.Probe/  9 项探针
docs/verification-report.md     验证报告：目标、范围、判定标准、实测证据、结论
docs/fork-cicd-playbook.md      fork 与 CI/CD 落地手册：仓库拓扑、分支模型、权限、步骤、风险
```

## 依赖版本说明

EF Core 用 `10.0.14` 而非 `10.0.0`：`10.0.0` 会传递引入 `SQLitePCLRaw.lib.e_sqlite3 2.1.11`，命中 GHSA-2m69-gcr7-jv3q（高危），在 `-warnaserror` 下会升格为 `error NU1903` 直接阻断构建。升到 `10.0.14` 后传递依赖为 `2.1.13`，漏洞消失。

## 许可证

本仓同时包含 GPL-3.0（FluentFlyout 侧约定）与 MIT（TaskbarLyrics 侧代码）来源。合并后整体按 **GPL-3.0** 分发，MIT 部分的原始版权声明必须保留，见 `LICENSE-NOTICE.md`。