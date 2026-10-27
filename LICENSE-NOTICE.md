// Copyright (c) 2026 anync
// SPDX-License-Identifier: MIT
//
// 本文件来自 TaskbarLyrics (https://github.com/ANYNC/TaskbarLyrics)，
// 原始许可为 MIT License。
//
// 本项目（ff-lyrics-verify）整体按 GPL-3.0-or-later 分发，
// 依据 MIT 与 GPL-3.0 的兼容性（MIT 为 GPL-compatible），
// 将 MIT 代码并入 GPL 作品是允许的，但必须：
//   1. 保留原始版权声明与许可声明（即本文件头）
//   2. 不删除原项目中的 LICENSE 文件副本
//
// 反向不成立：GPL-3.0 代码不得并入 MIT 项目后按 MIT 分发。

# 许可证归属说明

## 来源清单

| 目录 | 来源 | 许可证 | 处理方式 |
|---|---|---|---|
| `src/FluentFlyout.Lyrics/` | [ANYNC/TaskbarLyrics](https://github.com/ANYNC/TaskbarLyrics) `TaskbarLyrics.Core` | MIT | 保留版权头，整体按 GPL-3.0 分发 |
| `src/FluentFlyout.Lyrics.Probe/` | 本项目原创 | GPL-3.0-or-later | — |
| （后续）`FluentFlyout/` | [unchihugo/FluentFlyout](https://github.com/unchihugo/FluentFlyout) | GPL-3.0-or-later | 以 GPL-3.0 为主许可 |

## TaskbarLyrics 原始许可全文

```
MIT License

Copyright (c) 2026 anync

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 第三方依赖许可

| 包 | 版本 | 许可证 | 与 GPL-3.0 兼容 |
|---|---|---|---|
| Lyricify.Lyrics.Helper | 0.2.0 | Apache-2.0 | ✅ 是（Apache-2.0 与 GPL-3.0 兼容，但需注意其专利条款） |
| F23.StringSimilarity | 7.0.1 | MIT | ✅ 是 |
| Microsoft.EntityFrameworkCore.Sqlite | 10.0.0 | MIT | ✅ 是 |
| OpenCC | 0.1.0 | Apache-2.0 | ✅ 是 |
| Newtonsoft.Json | 13.0.3 | MIT | ✅ 是 |
| SharpZipLib | 1.4.2 | MIT | ✅ 是 |

## 待办

- [ ] 在正式并入 FluentFlyout 前，向上游 FF 作者开 issue 说明合并意图与许可安排
- [ ] 在每个从 TaskbarLyrics 拷贝的文件顶部补 `// SPDX-License-Identifier: MIT` 与来源注释
- [ ] 仓库根目录同时放置 `LICENSE`（GPL-3.0）与 `LICENSE-TaskbarLyrics`（MIT 原文）