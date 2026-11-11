# newbot Codex runtime

基于 OpenAI Codex main 的固定提交开发。精确源码、Rust 工具链及补丁清单见 `manifest.json`；私有源码副本位于 https://github.com/ohmydevelop/newbot-codex 。

原生修改负责两个边界：

- `plugins.<id>.local_root` 固定每轮插件资源目录，仍执行原生启用策略和钩子信任校验。
- `turn/settings/update.modelRoute` 原子发布后续模型请求使用的网关、模型及推理设置。已开始的请求持有旧快照，跨网关创建独立客户端并清除远端响应状态。`turn/settings/applied` 仅在真实请求使用该修订后通知客户端。

Bridge 继续负责鉴权、SQLite 配置、受限凭证注入与任务归属，不承接 Agent loop。原生保留管理策略、Guardian 约束、模型能力和历史兼容检查；缺少完整模型元数据、无法移交的压缩历史及容量不足会明确拒绝，不能以更换显示名称或重建线程替代。

## 构建与验证

在 newbot 根目录运行 `node scripts/build-native-codex.mjs`。脚本从固定上游提交检出独立源码，校验并应用补丁，使用锁文件构建；遇到不属于脚本的已有源码变更会停止，绝不重置个人工作目录。

构建输出 `.data/native/managed/bin/codex` 和包含源码摘要、二进制摘要的 `build.json`。使用该二进制重新生成协议，再运行两组真实内核测试：

```bash
CODEX_BIN="$PWD/.data/native/managed/bin/codex" npm run generate:codex
CODEX_BIN="$PWD/.data/native/managed/bin/codex" node --import tsx scripts/test-issue001-resources-native.mjs
CODEX_BIN="$PWD/.data/native/managed/bin/codex" node --import tsx scripts/test-issue001-live-native.mjs
```

测试使用独立 Codex Home、工作区及模拟 HTTP 网关；不会访问个人 Codex 配置。模拟结果证明协议和状态约束，不证明某个真实网关的兼容性。开发服务需在应用回归通过后再显式选择新二进制。

## 修改协议时

该上游版本将导出协议预编译到 `app-server-protocol/schema/precomputed`。只修改 Rust 类型后直接执行 `generate-ts` 会得到旧类型。先使用上游 `write_schema_fixtures_from_env` 测试入口分别生成 stable 与 experimental 导出，再重新构建二进制、导出 TypeScript 并验证 schema 一致性。生成文件和源代码必须一起提交。

原始 0.167.1 的探针补丁保存在 `patches/`，仅用于追溯；实际构建只使用 manifest 明确列出的 main 补丁。升级上游时更改固定提交，重新验证原生资源、请求路由和恢复，不自动追踪浮动分支。

## Source provenance

This repository starts from a source snapshot of OpenAI Codex commit `fe2f18520864ddbee86cdd4b103a9fefffe9c6a2` (2026-11-11). The preceding import commit contains the unmodified upstream source; the following commit contains the newbot changes. Upstream history is available at https://github.com/openai/codex. This preserves upstream LICENSE and NOTICE.

The manifest, patch set, integration probes and build wrapper referred to above live in the newbot client repository under `native/codex/` and `scripts/`. To build this source directly, run `CARGO_BUILD_JOBS=1 CARGO_PROFILE_DEV_DEBUG=0 cargo +1.95.0 build --locked -p codex-cli --profile dev-small` from `codex-rs/`.
