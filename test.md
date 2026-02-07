## 前置：先把 GitHub Actions 配好（否则 Required checks 会一直红）

1) Settings → Actions → General → Workflow permissions：选 **Read and write permissions**

2) Settings → Secrets and variables → Actions → Secrets：至少配置

- `OPENAI_API_KEY`（Codex PR Review / PR Description / Issue Triage 会用到）
- `ANTHROPIC_API_KEY`（Claude PR Review / Issue workflows 会用到）

> `OPENAI_BASE_URL` / `ANTHROPIC_BASE_URL` 只有你用代理/兼容网关时才需要；不配会走默认。
>
> 注意：`openai/codex-action` 用的是 **Responses API**（最终会请求 `…/v1/responses`）。本仓库 workflow 允许你把 `OPENAI_BASE_URL` 填到 `…/v1` 或 `…/v1/responses`，会自动补全到正确的 `…/responses` 端点。

## 重要：`pull_request_target` 工作流从“默认分支”读取

本仓库里这些 workflow 用的是 `pull_request_target`（例如：PR Labels、Codex/Claude PR Review、Codex PR Description）。

GitHub 在 **2025-12-08** 起调整了行为：`pull_request_target` 事件会**始终使用默认分支（Default branch）**作为 workflow 来源与执行引用（与 PR 目标分支无关）。

所以：
- 你要改这些 workflow，必须把改动合进**默认分支**（你仓库当前默认分支是 `main`），否则 PR 上跑的还是默认分支里的旧 workflow。
- 如果你希望“以 `dev` 为准”来管理这些 workflow，最省事的做法是把默认分支改成 `dev`（Settings → Branches → Default branch）。

## 规则集 1：保护 dev（现在就能生效）

Ruleset Name

- 填：dev-protection（名字随意，能看懂就行）

Enforcement status

- 选：Active

Bypass list（怎么选）

- 推荐：只加 Repository administrators（或只加你自己），用于紧急情况下绕过规则
- 不推荐：加太多人/团队（会削弱门禁）

Bypass mode（Always / For pull requests only / Exempt）

- 推荐：**For pull requests only**（保留门禁，但允许管理员在 PR 合并时“点一下确认绕过”用于紧急解锁；同时阻止你在命令行直接 push 绕过门禁）
- 次选：Always（管理员连命令行 push 也能绕过；权限更大）
- 不推荐：Exempt（相当于完全不受 ruleset 约束）

Target branches（怎么设置）

（Ruleset 的 Target branches 不是“下拉选分支”，而是“按 pattern 选”）

- 点 `Add a target` / `添加目标`
- 选择：`Include` → `Branch name pattern`
  - Pattern 填：`dev`
- `Exclude`：留空

> 如果你在 UI 里真的只能选 “默认分支 / 全部分支”，选不到 pattern：
> - 方案 A（推荐）：把默认分支改成 `dev`，然后 ruleset 选“默认分支”
> - 方案 B：选“全部分支”，再用 Exclude pattern 把 `feature/*`、`ci/*` 等排除掉（避免把所有分支都锁死）
> - 方案 C：不用 Ruleset，改用经典的 Branch protection rule（Settings → Branches → Add branch protection rule，用 `dev` 作为 pattern）

### Branch rules（哪些勾，哪些不勾）
- Restrict creations：不勾（和工作流无关；一般不用）
- Restrict updates：不勾（⚠️会让“更新 dev 分支”的权限变得很严：通常只有 bypass actors 能更新，容易导致别人 PR 过了也合不进去）
- Restrict deletions：勾（防止误删 dev 分支）
- Require linear history：按需（开了就不能用 merge commit；只允许 squash/rebase 之类，和工作流无关）
- Require signed commits：按需（会拦截未签名提交/合并；和工作流无关，先不启用最省事）
- Require a pull request before merging：勾（✅让所有改动都走 PR，才能完整触发 PR 相关工作流：Codex/Claude/Labels/Description）
    - Required approvals：建议 dev=0（想强制人工评审就设 1；设 >0 会要求有人点 Approve 才能合并）
    - Dismiss stale pull request approvals when new commits are pushed：当 Required approvals > 0 时建议勾（新提交会让旧 Approve 失效）
    - Require review from specific teams：不勾（只有你有团队/想强制特定团队评审才需要）
    - Require review from Code Owners：按需（只有你有 `CODEOWNERS` 并想强制它生效才勾）
    - Require approval of the most recent reviewable push：多人协作建议勾；单人仓库建议不勾（否则你 push 了最新提交后，必须别人 Approve 才能合并，或你每次都要走 bypass）
    - Require conversation resolution before merging：dev 按需（会要求把 Review 对话都点 Resolve；更严格，也更容易因为“有未解决对话”而卡合并）
    - Allowed merge methods：建议只开 Squash（可选再开 Rebase；不建议开 Merge commit，除非你明确需要 merge commit）
- Require status checks to pass：勾（✅把 Actions 变成合并门禁）
    - 需要添加的 checks：
      - PR Checks / backend
      - PR Checks / frontend
      - Codex PR Review / pr-review
      - Claude PR Review / pr-review
      - Codex PR Description / pr-description（你想“PR 说明也必须生成”就把它也设为 Required）
- Require branches to be up to date before merging：dev 按需（勾上后 base 分支有新提交时，PR 必须先 update 才能合并；Required checks 会重跑，包含 Codex/Claude，耗时/成本更高但更安全）
- Do not require status checks on creation：建议勾（允许创建分支/仓库时不被 required checks 卡住；不放松合并门禁）
- Block force pushes：勾
- Automatically request Copilot code review：按需（不是必须；你已经有 Codex/Claude 自动审查评论了）

## 规则集 2：保护 main（等 .github/ 合进 main 之后再做）

先把 dev 合到 main（让 .github/workflows/* 进 main），不然你在 main ruleset 里要求 checks 会导致 永远过不了。



Enforcement status
- 合并前：可以先用 Evaluate（不拦截，只观察）
- 合并后：改成 Active

Bypass list

- 同 dev：只给 Repository administrators（或只给你自己）

Target branches

（同上，用 pattern 选分支）

- 点 `Add a target` / `添加目标`
- 选择：`Include` → `Branch name pattern`
  - Pattern 填：`main`

Branch rules（main 推荐更严格一点）

- 勾：Restrict deletions、Block force pushes、Require a pull request before merging
- 勾：Require status checks to pass + Require branches to be up to date（main 建议勾；确保 Required checks 是在最新 base 分支上跑的，更安全但会更频繁重跑 checks）
  - `Do not require status checks on creation`：建议勾（同 dev）
    - checks 同样选：
      - PR Checks / backend
      - PR Checks / frontend
      - Codex PR Review / pr-review
      - Claude PR Review / pr-review
      - （可选）Codex PR Description / pr-description
- `Require a pull request before merging` 里的子选项（main 建议更严格）：
    - Required approvals：建议 1（团队成熟可设 2）
    - Dismiss stale pull request approvals when new commits are pushed：建议勾
    - Require approval of the most recent reviewable push：多人协作建议勾（单人仓库可不勾）
    - Require conversation resolution before merging：建议勾
    - Allowed merge methods：建议只开 Squash（可选再开 Rebase）

## 你现在看到 “No required checks…” 怎么办

- 先创建一个 PR（目标分支是 dev），等所有 workflow 跑完；
- 去 PR 页面的 Checks 标签里看实际显示的检查名；
- 回到 Ruleset 的 Require status checks to pass → Add checks，用看到的名字把它们选进去（一般就是 PR Checks / backend、
PR Checks / frontend、Codex PR Review / pr-review、Claude PR Review / pr-review 等）。
测试 workflow 修复
## 测试权限修复

测试时间: 2026年02月 7日  4:47:35
测试目的: 验证 GitHub Actions workflow permissions 配置是否正确
## Workflow 测试 V3

测试时间: 2026-02-07 18:42:31

本次测试目的：
- ✅ 验证 PR Labels workflow（权限已修复）
- ✅ 验证 Codex PR Review workflow（权限已修复）
- ✅ 验证 Claude PR Review workflow
- ✅ 验证所有 workflows 正常运行

预期结果：
1. PR Labels 自动添加标签（不再有 403 错误）
2. Codex PR Review 成功调用代理服务并添加评论
3. Claude PR Review 成功运行
4. 所有 required checks 通过

---

**测试环境**：
- Base branch: dev (已包含权限修复)
- Workflows 版本: 最新
- Secrets: 已配置

**修复历史**：
- ✅ 修复 pull-requests: read → write
- ✅ 修复 codex-action responses endpoint
- ✅ 同步 main → dev

**开始测试！** 🚀
