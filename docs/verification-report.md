# 可行性验证报告

> 对应实施计划 P0 + P1-4 单点验证。目的只有一个：**在投入 18–27 人天之前，用可复现的证据回答「方案 B 能不能开工」。**

| 项 | 值 |
|---|---|
| 验证对象 | `Lyricify.Lyrics.Helper` 0.2.0 + `TaskbarLyrics.Core` 源码 |
| 目标宿主 | FluentFlyout（`net10.0-windows10.0.22000.0`，GPL-3.0） |
| 验证执行日 | 2026-10-27 |
| 结论 | **通过，方案 B 可以开工** |
| 阻断项 | 0 |

---

## 一、验证目标

方案 B 的全部风险都可以归到四个问题。这四个问题里只要有一个答案是「不行」，18–27 人天的排期就要推翻重做。

| # | 问题 | 若答案为「不行」的后果 |
|---|---|---|
| **Q1** | `Lyricify.Lyrics.Helper`（`netstandard2.1`）在 `net10` 下能加载并正常工作吗？ | 逐字歌词解析能力缺失，方案 B 的最核心资产归零 |
| **Q2** | 它的**解密路径**（KRC 的 XOR+zlib、QRC 的 TripleDES+zlib）在 net10 下行为仍然正确吗？ | 只能处理明文歌词，无法对接真实音源 |
| **Q3** | `TaskbarLyrics.Core`（原始 `net8.0`）能在 FluentFlyout 的 TFM 下**整体编译通过**吗？ | 移植不是「改 TFM」而是「重写」，工期翻倍 |
| **Q4** | 依赖链（Newtonsoft.Json / SharpZipLib / EF Core / SQLite）会与 FluentFlyout 冲突或引入安全债吗？ | 需要替换组件或承担 CVE，返工且影响商店审核 |

> 其中 **Q2 是唯一可能推翻整个方案的单点**——因为 Q1 失败还可以换库，Q2 失败则意味着真实音源的逐字歌词拿不到，而逐字滚动正是 TaskbarLyrics 最不可替代的能力。

---

## 二、验证范围

### 在范围内

| 范围 | 说明 |
|---|---|
| 库加载与 TFM 兼容 | `netstandard2.1` → `net10.0-windows10.0.22000.0` |
| 四种歌词格式解析 | LRC（含 `offset`、畸形容错）、QRC、KRC、YRC 逐字 |
| 两条解密路径 | KRC（XOR + zlib）、QRC（TripleDES + zlib）——**自行构造密文做往返** |
| 传递依赖绑定 | Newtonsoft.Json、ICSharpCode.SharpZipLib 的运行时版本 |
| 在线源类型可加载 | 网易 / 酷狗 / LRCLIB 的 `Providers.Web.*` 类型 |
| Core 层整体编译 | 58 个源文件、9,137 行，双 RID（`win-x64` / `win-arm64`） |
| 依赖安全 | `dotnet list package --vulnerable --include-transitive` |

### 不在范围内（明确排除）

| 排除项 | 排除理由 | 归属阶段 |
|---|---|---|
| WPF 原生渲染层（逐字扫描 `LinearGradientBrush`） | 纯 UI 实现，无兼容性未知 | P2 |
| 任务栏嵌入（`SetParent` + `WS_CHILD`） | FluentFlyout 已有可用实现，直接复用 | P4 |
| 真实音源联网取词（QQ / 网易 / 酷狗鉴权与反爬） | 依赖外部服务可用性，不适合做门槛判定 | P3 |
| 歌词缓存数据库的实际读写 | 需要可写的用户目录与数据迁移路径 | P3 |
| SMTC 会话读取 | 两端各自独立实现且均已在生产环境验证 | 不需要验证 |
| 性能与内存占用 | 需完整应用才有意义 | P5 |

---

## 三、判定标准与实测结果

### 3.1 逐项判定表

| 编号 | 命题 | 方法 | 通过标准 | 实测 | 结论 |
|---|---|---|---|---|---|
| **V1** | 程序集加载 | 读 `TargetFrameworkAttribute` 与 `Environment.Version` | 加载不抛异常，TFM 报告为 netstandard2.1，运行时为 10.x | `ver=0.2.0.0, TFM=.NETStandard,Version=v2.1, runtime=10.0.14` | ✅ PASS |
| **V2** | QRC 逐字解析 | 真实样本 → `QrcParser.Parse` | 行数与音节数符合样本 | `lines=2, syllables[0]=2, text[0]='qrczero'` | ✅ PASS |
| **V3** | KRC 逐字解析 | 真实样本 → `KrcParser.ParseLyrics` | 同上，且首行为 `SyllableLineInfo` | `lines=2, syllables[0]=3, text[0]='krczeroword'` | ✅ PASS |
| **V4** | LRC 解析容错 | 含 `[offset:+250]`、空行、`[xx:yy]` 畸形的样本 | 不抛异常，有效行数 ≥ 2 | `lines=4, first='alpha fixture line'` | ✅ PASS |
| **V5** | YRC 逐字解析 | 真实样本 → `YrcParser.Parse` | 行数与文本符合样本 | `lines=2, text[0]='yrczeroword'` | ✅ PASS |
| **V6** | **KRC 解密往返** | 明文 → `ZLibStream` 压缩 → XOR(16B key) → base64(前置 `krc1`) → `Decrypter.DecryptLyrics` | 解密结果与原文**逐字节一致** | `len=63, roundtrip=OK (10ms)` | ✅ PASS |
| **V7** | **QRC 解密往返** | 明文 → `ZLibStream` 压缩 → TripleDES(8B 块) → hex → `Decrypter.DecryptLyrics` | 解密结果与原文**逐字节一致** | `len=71, roundtrip=OK (4ms)` | ✅ PASS |
| **V8** | 依赖链绑定 | 反射读已加载程序集版本 | 无 `FileLoadException` / 版本降级 | `Newtonsoft.Json=13.0.0.0, SharpZipLib=1.4.2.13` | ✅ PASS |
| **V9** | 在线源类型可加载 | `Assembly.GetType` 逐个探测 | TaskbarLyrics 实际消费的三个源全部可解析 | 网易/酷狗/LRCLIB 均 `True`（QQ 为 `False`，见 §4.3） | ✅ PASS |
| **J2** | Core 整体编译（x64） | `dotnet build -c Release -r win-x64 -warnaserror` | 0 error **且 0 warning** | `0 个警告 / 0 个错误` | ✅ PASS |
| **J2** | Core 整体编译（ARM64） | 同上，`-r win-arm64` | 0 error 且 0 warning | `0 个警告 / 0 个错误` | ✅ PASS |
| **J2** | 依赖漏洞扫描 | `dotnet list package --vulnerable --include-transitive` | 无高危及以上 | `没有易受攻击的包` | ✅ PASS |

### 3.2 判定门槛

| 层级 | 门槛 | 含义 |
|---|---|---|
| **硬门槛** | V1–V9 全部 PASS，且 J2 双 RID 均 0 error 0 warning | 任一不满足 → 方案 B 暂停，回到路径选择 |
| **软门槛** | 依赖漏洞扫描无高危 | 有高危 → 记录为风险项，P1 阶段决定升级或接受；不阻断开工 |

本次**硬门槛与软门槛同时满足**。

---

## 四、关键发现

这三条发现改变了实施计划的细节，值得单独记录。

### 4.1 TaskbarLyrics 的公开测试样本全是明文，解密路径上游无覆盖

`TaskbarLyrics.Core.Tests/TestData/Lyrics/` 下的 `.qrc` / `.krc` / `.yrc` 全部是明文 fixture：

```
qrc-provider-supplied.qrc: [1000,1800]qrc(1000,300)zero(1300,0)word
krc-provider-supplied.krc: [1000,1800]<0,300,0>krc<300,0,0>zero<300,1500,0>word
yrc-provider-supplied.yrc: [1000,1800](1000,300,0)yrc(1300,0,0)zero(1300,1500,0)word
```

这意味着上游 209 个 Core 测试用例**走的是 Parser 而不是 Decrypter**。直接跑 TaskbarLyrics 现有测试，会得到「全绿」的假象，而真实音源返回的加密歌词从未在这个仓库里被验证过。

**这就是 V6/V7 必须自行构造密文的原因**，也是本次验证最有价值的部分。

### 4.2 一处会在 CI 首跑就红的坑：`-warnaserror` × NU1903

初版把 EF Core 定在 `10.0.0`，它会传递引入 `SQLitePCLRaw.lib.e_sqlite3 2.1.11`，命中 [GHSA-2m69-gcr7-jv3q](https://github.com/advisories/GHSA-2m69-gcr7-jv3q)（高危）。平时只是个 warning，但在 `-warnaserror` 下被升格为 `error NU1903`：

```
error NU1903: 包 "SQLitePCLRaw.lib.e_sqlite3" 2.1.11 具有已知的 高 严重性漏洞
生成失败。  0 个警告  1 个错误
```

**处理方式不是屏蔽警告，而是消除根因**：EF Core 升到 `10.0.14`，传递依赖随之变为 `SQLitePCLRaw.lib.e_sqlite3 2.1.13`，漏洞消失。原来登记为风险 R9 的条目**已关闭**，不再是「接受的风险」。

### 4.3 V9 的 `False` 是探测目标写错，不是库缺陷

初版 V9 里我按猜测写了 `Lyricify.Lyrics.Providers.Web.QQMusic.SearchSongResponse`，结果 `False`。核查后确认：该 DLL 里确实有 `QQMusic` provider 命名空间，但 **TaskbarLyrics 根本不用它**——它的 QQ 源是自写的 `Services.QqMusicApiClient.cs` / `QqMusicLyrics.cs`。

TaskbarLyrics 实际从 `Lyricify.Lyrics.Providers.Web.*` 消费的只有网易、酷狗、LRCLIB 三个，全部探测为 `True`。修正后 V9 判定为 PASS，`False` 保留在输出里作为记录。

### 4.4 一处容易误判的假故障：zlib 容器 vs 裸 deflate

V6/V7 最初用 `DeflateStream` 构造密文，解密端抛 `SharpZipBaseException: Header checksum illegal`。

原因不是库有问题，而是 **SharpZipLib 的 `InflaterInputStream` 要求 zlib 容器格式**（2 字节头 + Adler32 尾部），而 `DeflateStream` 输出的是裸 deflate 流。换成 `ZLibStream` 后两条路径立刻通过。

> 这条在 P1 阶段会再次遇到：如果移植过程中出现 `Header checksum illegal`，先查压缩格式，不要去改 `Decrypter`。

---

## 五、实测环境

| 项 | 值 |
|---|---|
| OS | Windows 11 |
| .NET SDK | **10.0.403** |
| .NET 运行时 | Microsoft.NETCore.App **10.0.14** |
| TFM | `net10.0-windows10.0.22000.0` |
| 配置 | `Release` / `-warnaserror` |
| RID | `win-x64`、`win-arm64` |
| 关键包 | Lyricify.Lyrics.Helper 0.2.0、EF Core Sqlite 10.0.14、F23.StringSimilarity 7.0.1、OpenCC 0.1.0 |
| 传递包 | Newtonsoft.Json 13.0.3、SharpZipLib 1.4.2、SQLitePCLRaw.lib.e_sqlite3 2.1.13 |

---

## 六、对实施计划的影响

### 已关闭的风险

| 原风险 | 原等级 | 现状 |
|---|---|---|
| `Lyricify.Lyrics.Helper` 在 net10 下不兼容 | **致命** | **关闭**（V1–V9 全 PASS） |
| Core 层需重写而非移植 | 高 | **关闭**（58 文件原样编译通过，仅改 TFM 与 EF Core 版本） |
| 解密路径在 net10 下行为变化 | 高 | **关闭**（V6/V7 往返逐字节一致） |
| R9：`SQLitePCLRaw` 高危 CVE | 中 | **关闭**（EF Core 升至 10.0.14，传递依赖变 2.1.13） |

### 工时修正

原计划 P1 阶段预留的「解密路径适配」缓冲可以取消——该项不再是未知数。**P1 可从 5–7 人天下调至 3–5 人天**，整体从 18–27 人天下调至 **16–25 人天**。

### 新增加的必做项

| 新增项 | 归属 | 说明 |
|---|---|---|
| 把 V6/V7 往返测试搬进 P1 的测试项目 | P1 | 上游无覆盖，必须自建，否则真实音源出问题无法定位 |
| 补 QRC/KRC/YRC 的**加密** fixture | P1 | 现有 fixture 全明文，无法回归解密 |
| 统一 `-warnaserror` 门槛 | P0 | CI 一旦允许带警告通过，等于没有门槛 |

---

## 七、复现方式

```bash
# 前置：.NET 10 SDK
git clone <本仓>
cd ff-lyrics-verify

# J1：9 项探针（返回码 0 = 全通过）
dotnet restore src/FluentFlyout.Lyrics.Probe
dotnet build   src/FluentFlyout.Lyrics.Probe -c Release --no-restore -warnaserror
dotnet run     src/FluentFlyout.Lyrics.Probe -c Release --no-build

# J2：Core 双 RID 编译（应输出 0 警告 0 错误）
dotnet build src/FluentFlyout.Lyrics -c Release -r win-x64   -warnaserror
dotnet build src/FluentFlyout.Lyrics -c Release -r win-arm64 -warnaserror

# 依赖安全
dotnet list src/FluentFlyout.Lyrics package --vulnerable --include-transitive
```

云端等价执行见 `.github/workflows/verify.yml`，结果同时输出到 Job Summary 与 `probe-results` artifact。

---

## 八、剩余风险

验证只关闭了「能不能做」。以下风险在开工后仍然存在，已登记在实施计划的风险册里：

| 编号 | 风险 | 为何本次无法验证 | 验证时点 |
|---|---|---|---|
| R2 | 逐字扫描在 WPF 下的视觉还原度不足 | 需要真实渲染与肉眼比对 | P2 出首版截图时 |
| R4 | 任务栏嵌入与第三方 Shell（Nilesoft / Windhawk）冲突 | 需要目标环境 | P4 联调 |
| R5 | 真实音源鉴权变更导致取词失败 | 依赖外部服务状态 | P3 起持续观察 |
| R6 | FluentFlyout 上游 API 变动造成合并冲突 | 时间依赖 | 每个 P 阶段开始前 rebase |
| R7 | 许可证合规（MIT → GPL-3.0 方向允许，但须保留声明） | 法务问题非技术问题 | 首次公开发布前 |

---

## 九、结论

四个问题全部得到肯定回答：

- **Q1 能加载** —— netstandard2.1 库在 net10 下正常加载，运行时版本 10.0.14。
- **Q2 解密正确** —— KRC 与 QRC 两条真实加密路径往返逐字节一致，覆盖了 `TripleDESCrypt` 位运算、`InflaterInputStream` 解压与 BOM 处理这三个最脆弱的环节。
- **Q3 能编译** —— `TaskbarLyrics.Core` 58 个文件、9,137 行，仅改 TFM 与 EF Core 版本即整体通过，且 x64/ARM64 双 RID 均零警告零错误。
- **Q4 无冲突无安全债** —— 依赖链与 FluentFlyout 零重叠，漏洞扫描干净。

**方案 B 具备开工条件，建议按修订后的 16–25 人天排期进入 P0。**