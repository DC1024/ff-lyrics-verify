# Fork 与 CI/CD 落地安排

> 面向「从零把两个仓库 fork 到 GitHub 并跑起来」的可执行手册。读完应当能直接照着做，不需要再查文档。

---

## 一、仓库拓扑：三个仓，各司其职

不要试图把两个仓库合成一个再开发。合并后的历史会同时污染上游同步和 bisect。

| 仓库 | 角色 | 生命周期 | 是否改代码 |
|---|---|---|---|
| `FluentFlyout`（fork） | **主开发仓**，最终交付物 | 长期维护 | ✅ 全部开发在这里 |
| `TaskbarLyrics`（fork） | **只读参考仓** | 长期保留 | ❌ 只摘取，不修改 |
| `ff-lyrics-verify` | **门槛验证仓**（本仓） | P0 结束即归档 | ❌ 冻结点 |

### 为什么 TaskbarLyrics 也要 fork 而不是直接引用

1. **上游可能删库或改协议**。已经摘过来的 58 个文件需要一个可追溯的出处。
2. **上游仍在修 bug**。P3 阶段若发现取词逻辑问题，可以直接 `git log` 比对，摘取单个 commit 而不是整段重抄。
3. **许可证举证**。GPL-3.0 分发时，MIT 部分的来源需要能指认到具体 commit。

但**不要**在 TaskbarLyrics fork 里提交任何改动——一旦分叉，摘取上游修复就会变成手工合并。

### fork 之后 verify 仓怎么办

P0 结束时，把 `.github/workflows/verify.yml` **复制**进 `FluentFlyout` fork（不是 submodule，不是 subtree），然后把 verify 仓设为 archived。理由：验证仓的源码快照会随主仓演进迅速过期，留着当 CI 源只会制造两份真相。

---

## 二、分支模型

```
upstream/main ──┐
                ├─► origin/main          （干净基线，只接受 upstream 合并）
                │        │
                │        ├─► feat/p1-lyrics-core       P1  Core 移植
                │        ├─► feat/p2-lyrics-render     P2  WPF 渲染
                │        ├─► feat/p3-lyrics-sources    P3  取词与缓存（可与 P1/P2 并行）
                │        ├─► feat/p4-integration      P4  集成联调
                │        └─► feat/p5-hardening        P5  测试与验收
```

| 规则 | 理由 |
|---|---|
| `main` **只接受** upstream 合并与 P 阶段分支的 PR，不直接提交 | 保证 `main` 永远等于「upstream + 已验收的增量」 |
| 每个 P 阶段一条长期分支，PR 拆成可独立回滚的小块 | FF 上游活跃，长分支必然冲突；小块 PR 才能 rebase |
| **每周一固定 rebase `main` 到 upstream** | 拖延一周 = 多一天的冲突解决成本 |
| PR 必须过 J1 + J2 才能合 | 见 §五 |

---

## 三、所需工具与权限

### 3.1 账号与权限

| 项 | 要求 | 说明 |
|---|---|---|
| GitHub 账号 | 免费账号即可 | 两个上游都是 public，fork 无门槛 |
| PAT scope | `repo` + **`workflow`** | ⚠️ **`workflow` 是最容易漏的一个**：没有它，`git push` 任何含 `.github/workflows/` 的提交都会被拒（`refusing to allow a Personal Access Token to create or update workflow`） |
| 仓库可见性 | **建议 public** | private 仓的 Windows runner 按 **2 倍** 计费；public 仓免费无限。本项目 CI 会频繁跑 `windows-latest`，private 会很快烧完额度 |
| Actions 开关 | fork 后**需手动启用** | fork 默认禁用 workflow，要进 Actions 页点一次确认 |

`gh` CLI 登录时把 scope 一次给全：

```bash
gh auth login --scopes "repo,workflow"
gh auth status          # 应看到 Token scopes 含 workflow
```

### 3.2 技术栈工具

| 工具 | 用途 | 备注 |
|---|---|---|
| .NET SDK 10.0.4xx | 构建 | 用 `global.json` 锁 `10.0.100` + `rollForward: latestFeature`，避免 runner 升级把构建打挂 |
| `dotnet build -warnaserror` | 零警告门槛 | **核心质量闸门**，不允许绕过 |
| `dotnet format` | 格式检查 | 移植代码先只报告不阻断 |
| `dotnet list package --vulnerable` | 依赖安全 | 仅报告，避免上游新增 CVE 阻断无关 PR |
| `actions/checkout@v4` | 取码 | |
| `actions/setup-dotnet@v4` | 装 SDK | 用 `10.0.x` 而非钉死补丁号 |
| `actions/cache@v4` | NuGet 缓存 | key 用 `hashFiles('**/*.csproj')`，还原从 ~60s 降到 ~10s |
| `actions/upload-artifact@v4` | 归档探针结果与漏洞报告 | |
| `actions/dependency-review-action` | PR 级依赖审查 | 拦「这个 PR 引入了有漏洞的包」 |
| Dependabot | 依赖升级 + 安全告警 | fork 中**默认关闭**，需手动开 |
| CodeQL | C# 静态扫描 | public 仓免费 |
| `gh` CLI | 批量建仓、开 PR、看 run 日志 | |

### 3.3 不需要的东西

| 不需要 | 原因 |
|---|---|
| 自建 runner | `windows-latest` 足够；ARM64 只做编译不做运行 |
| Azure Pipelines / AppVeyor | GitHub Actions 已覆盖，多一套配置就是多一份维护 |
| 代码签名证书 | 内测阶段用 portable zip，签名留到分发前 |

---

## 四、关键步骤

按顺序执行。每步都有可验证的完成标志。

### Step 0 — fork 两个上游（5 分钟）

```bash
# 网页操作更快：打开仓库页 → Fork → 保持同名
# 或用 gh：
gh repo fork unchihugo/FluentFlyout --clone=false
gh repo fork ANYNC/TaskbarLyrics  --clone=false
```

**完成标志**：`gh repo view <you>/FluentFlyout` 有输出。

### Step 1 — 启用 Actions 并关掉无关功能（5 分钟）

fork 后 Actions 是关闭状态，必须手动开一次，否则推送 workflow 后不会有任何运行。

同时关掉 Issues / Wiki / Discussions / Projects——bug 应该报给上游，不要在自己的 fork 里造第二个 issue 追踪体系。

**完成标志**：Actions 页面显示 "Workflows aren't being run on this forked repository" 的横幅消失。

### Step 2 — 本地克隆并接好 upstream（10 分钟）

```bash
git clone git@github.com:<you>/FluentFlyout.git
cd FluentFlyout
git remote add upstream https://github.com/unchihugo/FluentFlyout.git
git remote set-url --push upstream DISABLED   # 防止误推上游
git fetch upstream
```

`--push DISABLED` 这一步很重要：它把「手滑推到 upstream」变成一条必然失败的报错，而不是一个无法撤销的事故。

### Step 3 — 落地 CI（30 分钟）

把本仓的 `.github/workflows/verify.yml` 复制进 `FluentFlyout/.github/workflows/`，然后：

1. `git add` **必须用 `git add -f`**——`.gitignore` 若含 `.github` 相关规则会静默忽略。
2. 提交信息里不要出现反引号与 `$()`（bash 会做命令替换，把消息截断）。
3. 推送时确认 PAT 带 `workflow` scope，否则会被拒。

**完成标志**：Actions 里出现一次绿色运行。

### Step 4 — 配置分支保护（10 分钟）

`Settings → Rules → New ruleset`，对 `main` 设置：

| 规则 | 值 |
|---|---|
| Require a pull request before merging | ✅ |
| Required status checks | `J1 兼容性探针`、`J2 Core 移植 (win-x64)` |
| Require branches to be up to date | ✅ |
| Allow force pushes | ❌ |

**不要**把 ARM64 加成 required check——它在 PR 场景被跳过，会导致 PR 永远等不到状态。

### Step 5 — 开户 Dependabot（10 分钟）

`Settings → Code security → Dependabot`：开启 alerts 与 security updates。

加一个 `.github/dependabot.yml`，只对 NuGet 生效，**每周一次而不是每天**——FF 依赖十几个包，每天开 PR 会淹没真正需要看的变更。

### Step 6 — 建立 P0–P5 看板（15 分钟）

用 GitHub Projects（Board 视图），列：`Backlog / 本阶段 / 进行中 / 待验收 / 完成`。

把实施计划里的阶段任务直接拆成 Issue 贴进去，每个 Issue 标 `P0`–`P5` 标签。**验收项必须是可判定的**（例如「x64 + ARM64 双 RID 零警告」而不是「Core 移植完成」）。

### Step 7 — 打第一个真实 PR（P1 的第一个切片）

不要等整个 P1 做完才提 PR。第一个 PR 只做一件事：**把 TFM 改掉并让 J2 变绿**。

这个 PR 的价值不是代码，而是验证「CI 门槛真的能拦住人」。

---

## 五、流水线分层

| 层 | 触发 | 内容 | 时长目标 |
|---|---|---|---|
| **L1 快速门** | 每次 push | J1 探针 + J2 x64 编译 | < 5 min |
| **L2 完整门** | PR / merge to main | + J2 ARM64 + 依赖审查 + 格式 | < 12 min |
| **L3 发布门** | tag `v*` | 双 RID publish + 产物打包 | < 15 min |

L1 必须足够快，否则开发者会开始绕过它。**探针只在依赖或 TFM 变更时才可能失败，可以进一步用 `paths` 过滤到只在 `**/*.csproj` 变化时跑。**

### 关于 ARM64 的硬限制

`windows-latest` 是 x64 runner。`-r win-arm64` 只能证明**能编译**，不能证明**能运行**。

- 若要真机验证：GitHub 已提供 `windows-11-arm` runner（public 仓免费），可作为 L3 的可选 job。
- 在拿到 ARM64 设备前，把 ARM64 定位为「编译兼容性检查」，不要写成「ARM64 已支持」。

---

## 六、风险与注意事项

### 6.1 许可证（最高优先级，不要搞错方向）

| 事实 | 含义 |
|---|---|
| FluentFlyout 是 **GPL-3.0** | fork 必须继续用 GPL-3.0，**不得改成 MIT** |
| TaskbarLyrics 是 **MIT** | MIT → GPL-3.0 **允许**；反向不允许 |
| 合并后整体 | **GPL-3.0**，且必须保留 TaskbarLyrics 的 MIT 版权声明 |

**具体动作**：

1. 根目录保留 FF 原始 `LICENSE`（GPL-3.0），**一个字都不要改**。
2. 新增 `LICENSE-NOTICE.md` 记录每个来源文件的出处与原始许可证。
3. 每个从 TL 拷来的 `.cs` 文件头部加 SPDX 标识：`// SPDX-License-Identifier: MIT`（保留 MIT 声明，整体仍受 GPL-3.0 约束）。
4. 公开发布前做一次 `git log --diff-filter=A` 审计，确认没有来源不明的文件。

> 这是**发布阻断项**，不是「有空再弄」。GPL 违规会导致整个 fork 无法合法分发。

### 6.2 上游同步

| 风险 | 缓解 |
|---|---|
| FF 上游活跃，`main` 每周都有新提交 | 每周一固定 rebase；把改动集中在**新增文件**，对上游文件的修改控制在个位数 |
| 修改上游文件导致冲突频发 | 优先用**扩展点**（新增 partial class、注册新服务）而非改原文件 |
| 上游重构目录结构 | rebase 失败的当天就处理，不要堆积 |

**改动面积是可控性的唯一指标。** 如果某阶段发现需要改 20 个上游文件，说明方案需要重新审视。

### 6.3 不要把「合入上游」当成目标

现实判断：FluentFlyout 有 €2.99 的 Microsoft Store 版本，TaskbarLyrics 是独立产品定位。上游未必愿意接受一个把两者合并的大 PR。

**默认假设是「长期维护自己的 fork」**，而不是「做完就提 PR 合回去」。这意味着：

- 需要自己承担长期维护成本（跟上游同步、修 bug、发版）。
- 想合回上游的话，应该**先开 Issue 探口风**，而不是写完 5000 行再提 PR。
- **不要用 FluentFlyout 的名字发布**——商标与商店归属问题是独立的法务风险，与代码无关。

### 6.4 CI 特有的坑

| 坑 | 现象 | 处理 |
|---|---|---|
| `-warnaserror` × NU1903 | CI 首跑就红，且报错指向「包有漏洞」而非代码 | 根因是传递依赖带 CVE。**升级依赖消除，不要 `NoWarn` 屏蔽** |
| NuGet 审计随时间变红 | 某天突然失败，代码没改 | 审计设 `continue-on-error`，只用 Dependabot 定期升；否则上游新增 CVE 会阻断无关 PR |
| fork PR 拿不到 secrets | 来自 fork 的 PR 中 `secrets.*` 为空 | GitHub 安全设计，不要设计依赖 secret 的 PR 检查 |
| `global.json` 钉死补丁号 | runner 升级 SDK 后报 "SDK not found" | 用 `10.0.100` + `rollForward: latestFeature` |
| `dotnet run --no-build` 找不到产物 | 输出路径含 RID 子目录 | 用 `bin/Release/**/` 通配，或不要加 `--no-build` |
| Windows 路径分隔符 | 在 `run:` 里写 `/` 有时失败 | 用 `working-directory` 而不是拼接路径 |

### 6.5 时间与成本

| 项 | 估算 |
|---|---|
| 首次搭好全部 CI（Step 0–6） | 约 1.5 小时 |
| 每次 L1 流水线 | 3–5 分钟 |
| 每周 rebase 成本 | 0.5–2 小时（取决于改动面积） |
| Actions 额度 | public 仓免费；private 仓按 Windows 2 倍计费，本项目日均消耗约 10–15 分钟额度 |

---

## 七、一页速查

```
要做的：
  □ fork 两仓，verify 仓独立
  □ 手动启用 Actions（fork 默认关闭）
  □ PAT 带 repo + workflow
  □ 仓库设 public（省额度）
  □ main 加 ruleset，required check 只放 J1 + J2(x64)
  □ 每周一 rebase upstream
  □ 根目录 LICENSE 不动，新增 LICENSE-NOTICE.md
  □ 每个 TL 来源文件加 SPDX: MIT

不要做的：
  ✗ 在 TaskbarLyrics fork 里提交任何改动
  ✗ 把 GPL-3.0 改成 MIT
  ✗ 用 -warnaserror 之外的方式「消掉」NU1903
  ✗ 把 ARM64 加成 required check
  ✗ 用 FluentFlyout 的名字发布 fork
  ✗ 假设上游一定会接受合并 PR
```