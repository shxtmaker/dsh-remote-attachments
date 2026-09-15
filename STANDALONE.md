# 独立仓库说明（来源与同步）

本文件记录本仓库与上游 [`shxtmaker/dsh-windows-launcher`](https://github.com/shxtmaker/dsh-windows-launcher)
的关系、抽取时所做的改动，以及如何保持同步。**包说明见 [README.md](README.md)**（与上游同名文件逐字节相同）。

## 来源

| 项 | 值 |
| --- | --- |
| 上游仓库 | GitHub `shxtmaker/dsh-windows-launcher`；Gitea `192.168.3.100:3300/lqy/dsh-windows-launcher` |
| 上游版本 | **v2.1.1**（2.1.0 的更正版）；抽取基线提交 `c2cae9b` |
| 抽取路径 | `plugins/dsh-remote-attachments/`（插件本体）+ `schemas/remote-attachments/v1/`（线协议冻结语料，86 文件） |
| 本仓库版本 | `@shxtmaker/dsh-remote-attachments@0.1.0`（标签 `v0.1.1` 为本仓库的发布序号） |

上游中该语料位于**仓库根**，被插件单测与门禁共同读取（也与 C# 生产 codec 共用同一份 `expected.json`），
因此独立仓库必须自带一份。

## 抽取时所做的改动

1. `scripts/wire-contract-gates.mjs`：`repoRoot` 由 `resolve(projectRoot,'..','..')` 改为 `projectRoot`。
2. `test/unit/wire-corpus-support.mjs`：语料目录由 `'../../../..'` 改为 `'../..'`。
3. `tests/fixtures/*.mjs`（14 个）与 `tests/interop/d18-interop-driver.mjs`：`repoRoot` 由
   `resolve(pluginRoot,'../..')` 改为 `pluginRoot`。
4. `tests/fixtures/*.sh`（6 个）：`repo_root` 由 `$(cd "$plugin_root/../.." && pwd)` 改为
   **向上查找含 `schemas/remote-attachments/v1` 的最近祖先**（独立仓库与上游工作区内两种布局都成立）。
5. `tests/fixtures/setup.sh`：tarball 由 `$repo_root/$tarball_rel` 改为 `$plugin_root/pack/$(basename …)`。
6. 排除 `tests/fixtures/.d20-probe-tmp.mjs`（遗留临时探针，含硬编码本机绝对路径；上游已在 `aecb3f3` 删除）。

上述 1–5 项本身即**布局无关**，可直接反向合入上游。

## 与上游的一致性

- **`README.md` 与上游逐字节相同**（自本仓库 `v0.1.1` 起）：因此两处构建出的插件 tarball 内容一致
  （`package.json` 的 `files` 只含 `lib/**`、`cordis.patch.yml`、`README.md`）。
- 其余差异仅为上面 1–5 项的路径解析 + 本文件（`STANDALONE.md` 不随包发布）。
- 同步方式：在独立仓库复制上游的 `src/**`、`scripts/**`、`test/**`、`tests/**`、`README.md`、`package.json`、
  `pnpm-lock.yaml`、`tsconfig.json`、`cordis.patch.yml` 与 `schemas/remote-attachments/v1/**`，
  再重新应用上面 1–5 项（或用 `git diff` 逐条比对）。

## 发布

| 本仓库标签 | 内容 | 对应上游 |
| --- | --- | --- |
| `v0.1.0` | 首次抽取（当时 README 为独立项目说明） | v2.1.0 |
| `v0.1.1` | README 与上游一致 + 来源说明移入本文件 | v2.1.1 |

发布资产：`shxtmaker-dsh-remote-attachments-0.1.0.tgz` + `SHA256SUMS.txt`，
GitHub 与 Gitea 同步；**Windows 实机路径仍未验证（WindowsPending）**。
