# @shxtmaker/dsh-remote-attachments

DeepSeek Harness（DSH）远程会话的**附件粘贴附加插件**：把 Windows Launcher 采集的普通文件与截图，
经带序号/ACK/背压的分块传输导入**当前可编辑会话**的待发送草稿。纯文本粘贴保持 Harness 原行为；
用户主动发送前不自动发送。

## 这个包做什么，不做什么

| | 归属 |
| --- | --- |
| 附件能力（本插件独占） | `attachment-paste-import` |
| 配对令牌、设备凭据、`/api/pair/*`、`/remote` 门控代理 | `@linxin666/dsh-remote-web-ui`（全家桶内，**本包不再装第二份**） |
| 会话草稿、上传端点与 receipt | DeepSeek Harness |

安装本插件不会替换、重新打包或修改 `@linxin666/dsh-web-all` 的聚合子插件清单。
禁用本插件只停止附件能力，配对、心跳与设备库保持原状。

## 环境要求

| 依赖 | 版本/要求 |
| --- | --- |
| Node.js | 22+ |
| pnpm | 11.x（`pnpm-lock.yaml` 固定 devDependencies） |
| DSH Harness | CLI `>= 0.1.5-rc.1`（见 `package.json` 的 `dsh.engines`） |
| 全家桶（仅夹具需要） | `@linxin666/dsh-web-all` `0.3.20`（含 `dsh-remote-web-ui` `0.3.20`） |
| 浏览器（仅夹具需要） | 固定版本 Chromium / Google Chrome（`DSH_ATTACH_CHROME` 可覆盖） |

## 安装

```bash
pnpm install --frozen-lockfile
pnpm run build && pnpm run test:pack      # 产出 pack/shxtmaker-dsh-remote-attachments-0.1.0.tgz
pnpm --dir <profile-dir> add file:<本仓库绝对路径>/pack/shxtmaker-dsh-remote-attachments-0.1.0.tgz
```

本包通过 `cordis.patch.yml` 挂载：host 半区以 **fetch 形状载体**安装 `__DSH_FILE_UPLOAD__`
（消费者读 `hook.fetch`），client 半区经 Cordis `ctx` 接入原生草稿/附件接口。
页面侧全局：`__DSH_ATTACHMENTS_BRIDGE__`、`__DSH_ATTACHMENTS_RECEIVER__`、
`__DSH_ATTACHMENTS_ADDON__`、`__DSH_ATTACHMENTS_STATUS__`（只读诊断，不暴露任意本机读取）。

**不要手改** `lib/client.js`：它由 `scripts/build-client-bundle.mjs` 生成。client-modules 的批次是各包
`client.js` 的**原始字节拼接**，入口顶层出现任何 `import`/`export` 都会让整批解析失败。

## 目录结构

```text
.
├── src/host.ts                    包的 `.` 入口（host 半区）
├── src/host/                      host 半区实现（含上传承载）
├── src/client/                    client 半区（桥、接收端、握手、草稿适配器、组合）
├── src/shared/                    两端共用：protocol/limits/capabilities + wire/ 冻结 codec
├── scripts/                       构建、清理、打包卫生扫描、线协议门禁
├── test/unit/                     单元测试与语料运行器
├── tests/fixtures/                真实 Harness/Chromium 夹具与各任务门禁
├── tests/interop/                 C# 生产协调器的对端驱动
├── docs/                          兼容/停用策略与 Linux 验证报告
├── schemas/remote-attachments/v1/ 线协议 v1 冻结语料（schema + golden + malicious + expected.json）
└── cordis.patch.yml               单一 Cordis 行：id=remote-attachments
```

## 命令

```bash
pnpm run typecheck      # tsc --noEmit
pnpm run build          # tsc（ESM）→ CJS → 经典包裹产物 lib/client.js
pnpm run test:unit      # node --test test/unit/**
pnpm run test:wire      # 线协议语料门禁（读 schemas/remote-attachments/v1，写 artifacts/）
pnpm run test:pack      # 生成真实 tarball 并逐条核对内容（P01–P08，含卫生扫描）
pnpm run verify         # typecheck + build + test:unit + test:pack
```

`test:pack` 判定的是**实际 tarball 内的字节**（不是工作树、不是 source link）：包身份、双面入口与类型声明、
`cordis.patch.yml` 的单行声明、"不含凭据/生产数据/本机绝对路径"、"不含测试与源码目录"。

**夹具类测试**（`fixture:*`）需要真实 `dsh` CLI、私有 `DSH_HOME`、固定全家桶版本
（`tests/fixtures/compatibility-lock.json`）与真实 Chromium。它们只在**私有**夹具目录
（`artifacts/fixture`，可用 `DSH_ATTACH_FIXTURE_ROOT` 覆盖）内运行，**绝不触碰** `$HOME/.dsh`；
`fixture:down` 只在双重确认（命令行含 profile 名 + `DSH_HOME` 指向夹具）后停进程。

`tests/interop/d18-interop-driver.mjs` 是 C# 生产协调器的对端驱动，需配合
[`shxtmaker/dsh-windows-launcher`](https://github.com/shxtmaker/dsh-windows-launcher) 的
`DshLauncher.Core.Tests`（`-trait interop=production`）运行，本仓库单独无法完成该用例。

## 状态

- **线协议 v1 已冻结**：11 类消息、`operationId==batchId`、`fileId` 绑定 target/documentEpoch/composerEpoch/
  sessionId/batchId、`seq` 每文件从 0、最多 2 块在途、256 KiB 块；36 个协议拒绝码与 19 个结果码分工不混用。
- **三态语义**：`transport{idle,buffering,buffered}` / `draft{none,staged,failed,partial}` /
  `upload{none,harness-owned}` —— 协议里**没有** `ready`/`uploaded`；`staged` 只表示对端草稿已接收字节，
  不保证 Harness 侧 receipt 仍有效。
- **能力与降级**：未知 hook ⇒ `capability-conflict` 且不覆盖不贴牌；远端不可用 ⇒ `unavailable`/`capability-disabled`
  且**不静默回退裸 `/api`**；停用/恢复与 `ReloadRequired` 可判定；配对、心跳、设备库不受影响。
- **Windows 实机路径未验证**（WPF/WebView2/Win32 端到端为 WindowsPending）；Linux 侧证据见
  [`docs/D24-LINUX-REPORT.md`](docs/D24-LINUX-REPORT.md)，兼容与停用策略见
  [`docs/COMPATIBILITY.md`](docs/COMPATIBILITY.md)。

## 来源与与上游副本的关系

本仓库从 [`shxtmaker/dsh-windows-launcher`](https://github.com/shxtmaker/dsh-windows-launcher) 的
**v2.1.0**（标签提交 `676f6a0`）提取：

- 插件本体 = 该仓库 `plugins/dsh-remote-attachments/`（85 个已跟踪文件）；
- `schemas/remote-attachments/v1/` = 该仓库同名目录（86 个文件，逐字节相同）——原仓库中该语料位于仓库根，
  被插件单测与门禁共同读取，独立仓库必须自带。

**为扁平化与可移植做的改动**：

1. `scripts/wire-contract-gates.mjs`：`repoRoot` 由 `resolve(projectRoot,'..','..')` 改为 `projectRoot`。
2. `test/unit/wire-corpus-support.mjs`：语料目录由 `'../../../..'` 改为 `'../..'`。
3. `tests/fixtures/*.mjs`（14 个）与 `tests/interop/d18-interop-driver.mjs`：`repoRoot` 由
   `resolve(pluginRoot,'../..')` 改为 `pluginRoot`。
4. `tests/fixtures/*.sh`（6 个）：`repo_root` 由 `$(cd "$plugin_root/../.." && pwd)` 改为
   **向上查找含 `schemas/remote-attachments/v1` 的最近祖先**（独立仓库与 launcher 仓库内两种布局都成立）。
5. `tests/fixtures/setup.sh`：tarball 由 `$repo_root/$tarball_rel` 改为 `$plugin_root/pack/$(basename …)`。
6. 排除 `tests/fixtures/.d20-probe-tmp.mjs`（遗留临时探针，含硬编码本机绝对路径）。
7. 本文件（`README.md`）已更新为独立项目说明；上游副本中的同名文件停留在 D03 阶段的描述。

> 上述第 1–5 项本身即布局无关，可反向合入上游仓库（那会改变已发布候选的源码树指纹，
> 需重跑 D10/D12/D18/D19–D23 相关门禁）。第 7 项会使本仓库构建的 tarball 与上游 tarball
> 在 **README 字节**上不同（文件数与其余内容相同）。

## 许可

`UNLICENSED`（见 `package.json`）。
