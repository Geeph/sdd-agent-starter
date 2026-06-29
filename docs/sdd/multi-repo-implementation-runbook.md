# SDD Multi-repo 落地操作手册（方案 B）

> 本文是方案 B 的实施与运行手册，位于 scratch 目录，不承担 backlog 状态。真实任务状态只存在于对应 GitHub 仓库的 Issues。

## 1. 目标与边界

目标是让一个产品由产品控制仓库和多个独立实现仓库协作：

```text
sdd-platform
acme-product-control
acme-api
acme-web
acme-ios
acme-android
acme-api-clients
```

推荐完整形态共 7 个仓库：

- `sdd-platform` 是组织共享仓库，所有产品复用。
- 其余 6 个是当前产品仓库。
- 没有某个平台时可以省略对应仓库。
- 不使用独立 SDK 仓库时可省略 `acme-api-clients`，但会失去标准包升级和 Renovate 路径。

Multi-repo 方案适用于：

- 平台由独立团队负责。
- 权限、合规或外包要求隔离。
- 各平台发布周期独立。
- Backend 被多个产品复用。
- 单产品 monorepo 已影响 CI 或开发效率。

## 2. 命令状态说明

本文中的 `sdd ...` 是计划实现的 CLI 接口。CLI 尚未完成时，可以使用 GitHub Actions、`gh` 和人工 PR 执行等价操作，但产物、Gate 和 Source of truth 不变。

目标命令集合：

```text
sdd product init
sdd validate
sdd impact
sdd contract publish
sdd backlog compile
sdd backlog publish
sdd dispatch
sdd sync
sdd reconcile
```

## 3. Source of truth

| 内容 | 权威位置 |
|---|---|
| 产品需求、架构、设计 | `acme-product-control` |
| OpenAPI / events | `acme-product-control/contracts` |
| 项目拓扑 | `acme-product-control/projects.yaml` |
| 跨项目 plan | `acme-product-control/specs/<version>/plan.md` |
| Backend 代码和局部计划 | `acme-api` |
| Web 代码和局部计划 | `acme-web` |
| iOS 代码和局部计划 | `acme-ios` |
| Android 代码和局部计划 | `acme-android` |
| Generated SDK 源码和发布 | `acme-api-clients` |
| Backend 当前合同 revision | `acme-api/sdd.lock` |
| Web/iOS/Android 当前 SDK revision | 各仓库 package-manager lockfile + SDK 内嵌 metadata |
| 实现仓库当前产品 spec/design revision | 各实现仓库 `sdd.lock` |
| 任务状态 | 对应仓库 GitHub Issues |
| 跨仓库视图 | Organization GitHub Project |

Organization Project 只聚合 Issue/PR，不成为第二 backlog。禁止在 product-control 创建实现任务的镜像副本。

## 4. 仓库结构

### 4.1 `sdd-platform`

```text
sdd-platform/
├── cli/
├── factory/
├── backlog-compiler/
├── github-app/
├── schemas/
│   ├── projects.schema.json
│   ├── task.schema.json
│   ├── impact.schema.json
│   ├── contract-bundle.schema.json
│   ├── sdk-release-state.schema.json
│   └── sdd-lock.schema.json
├── templates/
│   ├── product-control/
│   ├── spring-boot/
│   ├── web/
│   ├── ios-tuist/
│   ├── android/
│   └── api-clients/
└── .github/workflows/
    ├── java.yml
    ├── web.yml
    ├── ios.yml
    ├── android.yml
    ├── sdk-generate.yml
    └── contract.yml
```

### 4.2 `acme-product-control`

```text
acme-product-control/
├── specs/
│   └── v1/
│       ├── spec.md
│       ├── architecture.md
│       ├── design.md
│       └── plan.md
├── contracts/
│   ├── openapi.yaml
│   └── events.yaml
├── design/tokens/
├── projects.yaml
├── template.lock
└── .github/workflows/
    ├── contract-gate.yml
    ├── publish-contract.yml
    └── impact.yml
```

### 4.3 实现仓库

每个仓库至少包含：

```text
AGENTS.md
specs/<version>/plan.md
sdd.lock
.github/workflows/ci.yml
.github/workflows/sdd-sync.yml
.github/ISSUE_TEMPLATE/
.github/pull_request_template.md
CODEOWNERS
```

Web、iOS、Android 还包含 Renovate 配置；Backend 包含 provider conformance workflow。

### 4.4 `acme-api-clients`

```text
acme-api-clients/
├── Package.swift
├── contract.lock
├── generated/
│   ├── typescript/
│   ├── kotlin/
│   └── swift/
├── generators/
└── .github/workflows/
    ├── generate.yml
    └── publish.yml
```

该仓库只保存 generated SDK、生成器配置和测试，不允许放产品业务逻辑。

SwiftPM 远程依赖要求 `Package.swift` 位于 Git 仓库根目录，因此根 manifest 的 target
必须指向仓库内的 `generated/swift`。Swift release 使用纯 SemVer Git tag，例如
`1.4.0`；不得使用 `api-client-swift-v1.4.0`。npm 和 Maven 仍从各自子目录发布，三种
语言共享同一个合同版本，因此不需要为 Swift 再拆一个仓库。

## 5. 一次性建设 `sdd-platform`

### 5.1 定义 schemas

必须先确定：

- `projects.yaml` schema。
- 平台 task schema。
- stable task ID 格式。
- contract bundle manifest。
- SDK release state manifest。
- `sdd.lock` schema：所有实现仓库存 product refs，只有 Backend 增加 contract refs；
  客户端 SDK revision 不得重复写入该文件。
- dispatch payload schema。

版本化规则：

```text
schema_version: 1
```

破坏 schema 兼容性时增加 major schema version，不在现有消费者下静默改变格式。

### 5.2 实现一个 Compiler

MVP 为单一进程：

```text
compiler
├── common strategy
├── backend strategy
├── web strategy
├── ios strategy
├── android strategy
└── dependency reconciliation pass
```

Compiler 输入：

```text
spec.md
architecture.md
design.md
plan.md
projects.yaml
contracts/*
```

Compiler 输出统一 task manifest，再由 MultiRepoPublisher 根据 `projects.yaml` 路由到目标仓库。

### 5.3 Stable ID 与 Issue upsert

Issue body 写入：

```html
<!-- sdd-task-id: android.auth.login-screen -->
<!-- sdd-source-revision: abc123 -->
```

Upsert：

```text
marker 不存在           -> create
未开始且内容变化        -> 生成 update diff
In Progress             -> 创建 Change Issue
Done                    -> 创建 Change/Migration Issue
相同 source revision    -> no-op
```

跨仓库依赖先使用 stable task ID 表达，publish 完成后解析成具体 Issue URL。不能将 Issue number 当全局稳定标识。

### 5.4 实现 GitHub App

GitHub App 安装在 product-control 和所有实现仓库。

建议最小权限：

```text
Metadata: read
Contents: read/write
Pull requests: read/write
Issues: read/write
Actions: read/write
Projects: read/write（只有需要更新 Project 时）
```

App 订阅：

```text
push
pull_request
release
workflow_run
```

职责：

- 只通过 GitHub 原生 webhook 接收入站事实，例如 product-control `release` 和可信
  `workflow_run`。
- 读取 `projects.yaml`。
- 做增量 impact。
- 作为唯一 fan-out 发送 `sdd_*` `repository_dispatch`。
- 创建 Change/Migration Issues。
- 创建标准包升级无法表达的同步 PR。
- 更新 Organization Project。
- 记录 delivery、失败、重试和 reconciliation 状态。

不要在各仓库存储长期 PAT。每次操作使用短期 installation token，并限制到目标仓库。

### 5.5 reusable workflows

集中维护：

```text
java.yml
web.yml
ios.yml
android.yml
sdk-generate.yml
contract.yml
```

调用仓库固定到 release tag 或 commit SHA，不引用可变 `main`。

### 5.6 模板生命周期

- 技术栈模板为一次性脚手架。
- 业务代码生成后不自动跟随模板更新。
- reusable workflows 独立版本化。
- 安全基线通过显式同步 PR 下发。
- 仓库规模扩大后，可用 Terraform/Pulumi 管理非敏感 repository/ruleset/environment 配置。
- Secrets 使用 OIDC、组织 secret 或外部 secret manager，避免进入 IaC state。

## 6. 创建产品仓库组

### 6.1 Dry-run

```bash
sdd product init acme \
  --mode multi-repo \
  --components backend,web,ios,android,api-clients \
  --dry-run
```

Dry-run 显示：

- 将创建的仓库。
- 每个仓库使用的模板 commit。
- GitHub App 安装范围。
- ruleset、labels、CODEOWNERS。
- Organization Project。
- reusable workflow refs。
- package namespace 和权限。
- environments 和缺少的 secrets。
- 不执行 GitHub 写操作。

### 6.2 创建仓库

```bash
sdd product init acme \
  --mode multi-repo \
  --components backend,web,ios,android,api-clients
```

Factory 创建：

```text
acme-product-control
acme-api
acme-web
acme-ios
acme-android
acme-api-clients
```

六个仓库逐一执行相同的 bootstrap，不得从“空仓库直接开初始化 PR”：

1. Factory 通过对应 GitHub Template 创建仓库。模板初始 commit 建立 `main`，并至少
   包含根骨架、基础 CI、`PR hygiene` 和接收事件所需的 `sdd-sync.yml`。这是建仓
   bootstrap，不是 agent 或开发者直接 push。
2. Factory 通过 GitHub API 配置 labels、teams、environments 和初始 ruleset。初始
   ruleset 禁止普通用户直接 push，并要求 PR/approval，但暂不配置 required checks。
3. 仓库创建后安装 SDD GitHub App，并把 installation 限制到这六个授权仓库。若 App
   尚处于 Phase 1 未部署状态，则先保留人工 `workflow_dispatch`，在 Phase 3 安装后
   才启用自动 fan-out。
4. Factory 为每个仓库创建 Bootstrap PR，写入产品名、owner、`CODEOWNERS`、固定的
   reusable workflow ref 和产品配置。
5. Bootstrap PR 产生真实 check context，CI 通过并人工批准后合并。
6. 对每个仓库确认 required check 名称真实存在，再把平台 CI、`PR hygiene` 及该仓库
   专属 Gate 加入 ruleset。
7. `sdd-sync.yml` 已进入默认分支且 App 已安装后，再执行一次签名 payload 的
   `repository_dispatch` smoke test。

GitHub Template API 只复制模板 Git 内容，不复制 labels、rulesets、branch protection、
environments、secrets 或 GitHub App installation；这些设置必须由 Factory 另行配置。
最终禁止任何 agent 或开发者直接 push `main`。

### 6.3 `projects.yaml`

```yaml
schema_version: 1
product: acme
repository_mode: multi-repo

repositories:
  product_control: org/acme-product-control
  api_clients: org/acme-api-clients

projects:
  - id: backend
    repository: org/acme-api
    template: spring-boot
    template_ref: v1.0.0
    owner: backend-team

  - id: web
    repository: org/acme-web
    template: web
    template_ref: v1.0.0
    owner: web-team
    sdk: npm

  - id: ios
    repository: org/acme-ios
    template: ios-tuist
    template_ref: v1.0.0
    owner: ios-team
    sdk: swiftpm

  - id: android
    repository: org/acme-android
    template: android
    template_ref: v1.0.0
    owner: android-team
    sdk: maven
```

### 6.4 配置 labels

所有代码仓库统一使用：

```text
track:code
track:contract
type:task
type:change
type:migration
status:blocked
contract-update
```

Product-control 增加：

```text
track:spec
track:design
type:epic
type:decision
```

### 6.5 配置 ruleset

每个仓库最终 ruleset：

- 禁止直接 push `main`。
- 要求至少一个人工 approval。
- 要求平台 CI。
- 要求 PR hygiene。
- 要求 review threads resolved。

Product-control 额外要求 Contract Gate。API 仓库额外要求 provider conformance。SDK 仓库要求三语言 SDK build/test。

required checks 只能在对应 Bootstrap PR 已成功上报这些 check context 后启用，顺序见
§6.2；不得在仓库刚由 Template 创建时一次性挂上尚不存在的 required checks。

### 6.6 Organization Project

建议字段：

```text
Product
Platform
Track
Version
Requirement ID
Contract Version
Status
Blocked By
```

Project 只读取真实 Issues/PRs，不创建重复 draft item 作为任务状态。

## 7. 日常产品流程

### 7.1 Intake 与 Spec Gate

新需求先进入 `acme-product-control` Intake Issue。

Commander 创建：

```text
specs/v1/spec.md
```

Spec 包含稳定 requirement IDs，例如：

```text
REQ-AUTH-001
REQ-PROFILE-002
```

人工批准 Spec PR 后记录批准 commit。

### 7.2 Architecture Gate

创建：

```text
specs/v1/architecture.md
projects.yaml
contracts/openapi.yaml
contracts/events.yaml
```

确认：

- 服务和平台边界。
- OpenAPI/event 合同。
- 安全、性能、可用性。
- 项目和 owner。
- 发布和兼容策略。
- 概念领域模型。

物理 DB schema 和 migrations 只属于 `acme-api`，不放入 product-control 作为跨仓库合同。

### 7.3 Design Gate

有 UI 时创建：

```text
specs/v1/design.md
design/tokens/
Figma link
```

稳定 screen IDs：

```text
SCR-LOGIN
SCR-DASHBOARD
```

Design Gate 检查用户流、边界状态、平台差异、accessibility、tokens 和 API 数据需求。

### 7.4 Plan Gate

创建跨项目：

```text
specs/v1/plan.md
```

各实现仓库仅保存自己的局部执行 plan 或产品控制仓库的固定 commit 引用，不复制完整 product spec。

## 8. Backlog 生成与发布

### 8.1 Dry-run

```bash
sdd backlog compile \
  --control-repo org/acme-product-control \
  --version v1 \
  --dry-run
```

报告包括：

- 每个目标仓库的 create/update/no-op。
- stable task ID。
- parent epic。
- requirement/screen/operation refs。
- 跨仓库依赖。
- 循环依赖。
- 找不到或无权限访问的仓库。

### 8.2 Publish

Plan Gate 通过：

```bash
sdd backlog publish \
  --control-repo org/acme-product-control \
  --version v1
```

发布规则：

```text
Epic/Design/Contract -> acme-product-control
Backend tasks        -> acme-api
Web tasks            -> acme-web
iOS tasks            -> acme-ios
Android tasks        -> acme-android
SDK generation       -> acme-api-clients
```

发布完成后：

- 解析 stable task ID 到真实 Issue URL。
- 回填 parent/dependency links。
- 加入 Organization Project。
- 保存 delivery audit，不创建第二份状态列表。

## 9. Contract-first 操作流程

### 9.1 Contract PR

在 product-control 修改：

```text
contracts/openapi.yaml
```

Contract Gate：

1. OpenAPI lint。
2. Breaking-change diff。
3. `$ref`、examples、operationId 完整性。
4. Impact report。
5. 调用 `sdd-platform` 中固定到 commit SHA 的唯一 `sdk-generate.yml`，从合同生成
   TS/Kotlin/Swift 候选产物。
6. 三语言 SDK 编译和最小 client tests。
7. 为候选源码包记录 `openapi_checksum`、generator commit、generator config checksum
   和各语言 artifact checksum，并生成 artifact attestation。

此阶段不要求 backend provider 已实现。Contract Gate 通过即可批准和合并合同；Backend
立即按合同实现，SDK 发布链路同时运行，SDK aggregate ready 后客户端即可基于 mock 与
Backend 并行开发。

Contract Gate 和 SDK 发布不得维护两套生成脚本或独立 generator 配置。后续发布只能
消费本 Gate 对相同 OpenAPI checksum 已验证并证明来源的候选产物；找不到匹配
attestation 时发布失败，不允许在发布 job 中无校验地重新生成一份。

### 9.2 发布合同版本

```bash
sdd contract publish \
  --control-repo org/acme-product-control \
  --version 1.4.0 \
  --dry-run

sdd contract publish \
  --control-repo org/acme-product-control \
  --version 1.4.0
```

发布 bundle：

```text
contract-v1.4.0/
├── manifest.json
├── spec.md
├── openapi.yaml
├── changed-requirements.json
├── impact.json
├── sdk-checksums.json
├── sdk-candidates/
│   ├── typescript.tar.gz
│   ├── kotlin.tar.gz
│   └── swift.tar.gz
└── checksums.txt
```

`manifest.json` 至少包含：

```json
{
  "contract_version": "1.4.0",
  "product_commit": "def456",
  "openapi_checksum": "sha256:...",
  "generator_commit": "012abc...",
  "generator_config_checksum": "sha256:...",
  "breaking": false
}
```

`sdk-checksums.json` 和 artifact attestation 将 Contract Gate 编译测试过的候选产物与
该合同版本绑定。`publish-contract` 必须从对应成功 Gate 下载这些产物并复核摘要，不能
自行调用另一套 generator。

### 9.3 发布 generated SDK

Product-control 发布 immutable GitHub Release 后，SDD App 接收原生 `release`
webhook，验证 bundle/attestation，再向 `acme-api-clients` 发送
`sdd_contract_published`。Product-control workflow 不直接向实现仓库 dispatch。

```text
下载并校验 contract bundle
  -> 更新 contract.lock
  -> 解包 Contract Gate 已验证的 TypeScript/Kotlin/Swift 候选产物
  -> 校验各语言 artifact checksum 与 attestation
  -> 生成或更新 generated-source PR
  -> PR 编译和测试、独立 review、人工批准并合并
  -> 发布 workflow 对合并后的源码重新计算摘要并与 attestation 比较
  -> 按语言发布并记录状态
  -> 全部必需语言成功后标记 aggregate ready
```

版本必须一致：

```text
contract bundle            1.4.0
@acme/api-client           1.4.0
com.acme:api-client        1.4.0
SwiftPM Git tag            1.4.0
```

每个 SDK 内嵌：

```text
contract version
product commit
OpenAPI checksum
generator version
generated artifact checksum
```

#### 部分发布语义

npm、Maven 和 Git tag 不具备跨 registry 事务，不能声称真正的 all-or-none。发布流程
必须为 `projects.yaml` 中实际需要的每种语言保存独立状态，例如：

```yaml
contract_version: 1.4.0
typescript:
  status: published
  checksum: sha256:...
kotlin:
  status: failed
swift:
  status: pending
aggregate: incomplete
```

实际 registry/package/tag 是发布事实的权威来源；状态记录用于审计和恢复，App 与
nightly reconciliation 必须重新查询外部事实后再更新状态。处理规则：

- 已发布且 checksum 相同：重跑为 no-op。
- 已发布但 checksum 不同：立即失败，禁止覆盖不可变版本。
- 某语言失败：保留其他语言成功状态，只重试失败或 pending 的语言。
- 所有必需语言发布成功：写入签名 `sdk-release-ready` manifest，并由 App 发送
  `sdd_sdk_release_ready`；ready manifest 作为 `acme-api-clients` GitHub Release asset
  保存，该原生 `release` webhook 由 App 验证后触发 fan-out。在此之前不向消费仓库
  fan-out 升级。
- 如果必须修改已部分发布版本的生成内容：废弃该聚合版本，发布新的合同 patch
  version，例如 `1.4.1`，不得复用或覆盖 `1.4.0`。

Renovate 只允许选择带签名 `sdk-release-ready` manifest 的版本。组织 Renovate preset
必须通过 custom datasource/version catalog 只暴露 ready versions；App 可在 ready 后
触发 Renovate run，但触发时机不能代替数据源过滤。普通 registry 中仅“已经出现”的
部分版本不能作为可升级信号。

### 9.4 Renovate 下发

Web、Android、iOS 模板必须包含 Renovate 配置或组织 preset：

- Web 启用 npm manager。
- Android 启用 Maven/Gradle manager。
- iOS 启用 SwiftPM manager。
- 只跟踪组织内 generated SDK。
- 使用 `contract-update` label。
- 同一 contract version 分组。
- 默认不 automerge。
- Major update 禁止 automerge，并要求 migration Issue。
- 数据源只暴露带签名 `sdk-release-ready` manifest 的聚合完成版本。

Renovate 创建 SDK bump PR 后，消费仓库 CI 检查：

```text
package-manager resolved SDK version
  == SDK metadata contract version

SDK metadata OpenAPI checksum
  == product-control immutable contract bundle checksum
```

客户端 `sdd.lock` 不再重复存储 SDK/contract version、commit 或 checksum，因此
Renovate 只需更新原生 dependency manifest 和 package-manager lockfile。版本、bundle
checksum、artifact checksum 或 attestation 任一不匹配时 CI 失败。

### 9.5 Backend Implemented Gate

Backend 收到 contract version 后：

1. 更新 `sdd.lock`。
2. 实现 API。
3. 启动真实 provider。
4. 使用固定 OpenAPI version 运行 schema/conformance tests。
5. MVP 可使用 Schemathesis。
6. 记录实现的 contract version。

通过后发送：

```text
sdd_backend_implemented
```

未通过 provider conformance 不能发送该事件，但不回滚已批准合同；应修正实现或创建新的 Contract PR。

### 9.6 Backend Deployed Gate

部署环境执行：

1. Smoke tests。
2. 固定合同的 provider conformance。
3. 验证部署实例报告的 contract version。
4. 验证兼容版本仍可用。

通过后发送：

```text
sdd_backend_deployed
```

路由规则：

```text
sdd_contract_published -> acme-api + acme-api-clients
sdd_sdk_release_ready  -> acme-web + acme-ios + acme-android 中实际存在的消费者
implemented/deployed   -> projects.yaml 中受影响的客户端
```

Backend 在 `sdd_contract_published` 后开始实现；客户端在 `sdd_sdk_release_ready` 后基于
已发布 SDK 和 mock 开始实现。两条线仍可并行，但客户端只有收到 deployed 事件后才能
启用真实能力或移除 feature flag。

## 10. `sdd.lock`

每个实现仓库的 `sdd.lock` 都固定产品规划 revision：

```yaml
schema_version: 1

product:
  repository: org/acme-product-control
  spec_version: v1
  spec_commit: abc123
  design_commit: bcd234
```

Backend 没有 generated SDK，因此额外固定它要实现的合同：

```yaml
schema_version: 1

product:
  repository: org/acme-product-control
  spec_version: v1
  spec_commit: abc123
  design_commit: bcd234

contract:
  version: 1.4.0
  commit: def456
  checksum: sha256:...
```

Web、iOS、Android 的当前 SDK revision 由各自原生 dependency manifest 和
package-manager lockfile 固定；合同 version、commit、checksum 从已解析 SDK 的内嵌
metadata 读取，并与 product-control immutable contract bundle 校验。客户端
`sdd.lock` 不重复保存这些字段，避免 Renovate 只升级依赖后与第二份版本状态死锁。

禁止：

```text
contract: main
contract: latest
unversioned SDK
客户端同时在 sdd.lock 和 package-manager lockfile 保存 SDK version
```

## 11. 跨仓库事件

### 11.1 事件类型

```text
sdd_spec_changed
sdd_design_changed
sdd_contract_published
sdd_sdk_release_ready
sdd_backend_implemented
sdd_backend_deployed
```

### 11.2 Payload

```json
{
  "schema_version": 1,
  "product": "acme",
  "source_repo": "org/acme-product-control",
  "source_commit": "def456",
  "contract_version": "1.4.0",
  "contract_checksum": "sha256:...",
  "breaking": false,
  "changed_requirements": ["REQ-AUTH-001"],
  "changed_operations": ["loginUser"]
}
```

上例是 `sdd_contract_published` payload。`sdd_sdk_release_ready` 还必须携带签名 ready
manifest 的 URL/checksum、各生态 package coordinates 和 artifact checksums。消费方
先验证 manifest，再允许 Renovate 或同步 workflow 处理该版本。

### 11.3 消费验证

接收 workflow 必须：

- 验证 event schema version。
- 验证 source repo 在 allowlist。
- 验证 commit、release 和 checksum。
- 拒绝 `main/latest`。
- 使用 concurrency key 防止同一版本重复处理。
- 保存 delivery ID 以保证幂等。

### 11.4 App 和 Renovate 的边界

```text
Renovate:
  只对 sdk-release-ready 版本创建非破坏性 SDK dependency bump

SDD GitHub App:
  接收原生 release/workflow_run webhook
  唯一发送 sdd_* repository_dispatch
  Spec/Design semantic changes
  Breaking contract
  SDK aggregate-ready fan-out
  Migration Issues
  Implemented/Deployed events
  Package bump 无法表达的同步 PR
```

不要让两套机器人同时修改同一个 dependency/version 文件。

禁止 product-control workflow、API workflow 或 SDK workflow 绕过 App 直接向其他仓库
发送 `repository_dispatch`。发送方只有 App；目标 workflow 依赖 delivery ID 保证幂等。

## 12. Spec / Design / Contract 变更

### 12.1 Impact

```bash
sdd impact \
  --control-repo org/acme-product-control \
  --base <approved-commit> \
  --head <candidate-commit>
```

报告：

- changed requirement IDs。
- changed screen IDs。
- changed operationIds。
- Breaking 与否。
- 受影响仓库。
- 受影响 Issues。
- 推荐事件和 migration tasks。

### 12.2 Issue 处理

```text
未开始            -> 生成 update diff
In Progress       -> 创建 Change Issue
Done              -> 创建 Change/Migration Issue
无语义影响        -> 不通知无关仓库
相同 revision     -> no-op
```

## 13. Breaking API 操作

流程：

```text
OpenAPI breaking PR
  -> impact report
  -> 新 major contract
  -> Migration Plan Gate
  -> 发布 v2 contract + SDK
  -> 各仓库 migration Issues
  -> backend v1/v2 并存
  -> clients 迁移
  -> 确认消费方全部完成
  -> 单独 PR 删除 v1
```

禁止：

- 覆盖已有 package version。
- 直接删除仍被消费的 operation。
- 自动合并 major SDK update。
- 在没有 migration Issue 时关闭旧版本。

## 14. Reconciliation

实时事件之外，每日运行：

```bash
sdd reconcile --product acme --check
```

检查：

- product-control 最新 approved spec/design/contract。
- 每个仓库 `sdd.lock` 中的 product refs，以及 Backend 的 contract refs。
- Web/iOS/Android package-manager lockfile 的 resolved SDK version。
- npm/Maven/Swift tag 的逐语言发布事实和 artifact checksum。
- 签名 `sdk-release-ready` manifest 是否只覆盖全部必需语言均成功的版本。
- SDK metadata 的 contract/OpenAPI/artifact checksum。
- partial/failed SDK publish 是否需要幂等重试或标记 abandoned。
- 遗漏或重复 delivery。
- 失败 Renovate PR。
- 缺失 Change/Migration Issues。
- Organization Project 中遗漏的 Issues。
- backend implemented/deployed contract version。

默认只报告，不自动覆盖代码或 Issue。修复操作需要显式 `--apply` 和 dry-run diff。

## 15. CI 与 Ruleset

### Product-control

Required：

```text
Spec/Docs validation
Contract Gate
PR hygiene
```

### API

Required：

```text
Java lint/typecheck/test/build
Provider conformance
PR hygiene
```

### API Clients

Required：

```text
TypeScript client build/test
Kotlin client build/test
Swift client build/test
SwiftPM root Package.swift validation
SDK metadata/artifact attestation validation
PR hygiene
```

API Clients release workflow 另外验证将创建的 Swift Git tag 是纯 SemVer，且 tag commit
与已合并 generated-source revision 一致。

### Web/iOS/Android

Required：

```text
平台 lint/typecheck/test/build
Resolved SDK↔metadata↔immutable contract bundle consistency
PR hygiene
```

## 16. 实现与 Review

每个实现任务：

```text
读取本仓库 Issue
  -> 读取固定 product spec/design refs
  -> Backend 从 sdd.lock、客户端从 resolved SDK metadata 读取固定 contract ref
  -> issue-<n>-<slug>
  -> 测试优先
  -> 实现
  -> 本地 checks
  -> PR
  -> 仓库 CI
  -> 独立模型 Review
  -> 人工批准
  -> merge
```

Agent 不应默认读取其他仓库全部内容，只读取 Issue 引用的固定文件和合同版本。

## 17. 发布流程

### Backend

```text
provider conformance
  -> backend release
  -> deploy staging
  -> deployed conformance
  -> deploy production
  -> sdd_backend_deployed
```

### Web/iOS/Android

```text
SDK contract version validated
  -> platform CI
  -> release artifact
  -> environment/store release
```

每个 release 记录：

- product spec commit。
- design commit。
- contract version/checksum（Backend 来自 `sdd.lock`，客户端来自 resolved SDK metadata）。
- SDK version。
- source commit。

## 18. 故障处理

### Dispatch 未触发

检查：

- App 是否安装到目标仓库。
- installation token 是否包含目标仓库。
- `sdd-sync.yml` 是否存在于默认分支。
- event type 是否匹配。
- product-control 是否只发布原生 release，而由 App 负责出站 dispatch。
- delivery ID 和 App logs。
- 运行 `sdd reconcile --check` 查漏。

### Renovate 没有创建 PR

检查：

- 对应 manager 是否启用。
- package registry 或 Git tag 是否可访问。
- package version 是否真的增加。
- 私有 registry 凭证。
- ignorePaths/packageRules 是否误过滤。
- major update 是否被策略挂起。

### SDK 与 contract 不一致

处理：

1. 不手工修改 generated SDK。
2. 客户端验证 package-manager lockfile 的 resolved version 和 SDK metadata。
3. 验证签名 `sdk-release-ready` manifest、输入 bundle checksum 和 artifact attestation。
4. 重新消费 Contract Gate 已验证的候选产物，不在发布 job 中独立重新生成。
5. 如果候选产物本身错误，发布新的合同 patch version，不覆盖旧 package/tag。

### SDK 只有部分语言发布成功

处理：

1. 查询 npm、Maven registry 和 Swift Git tag 的真实状态，不只相信 workflow 状态。
2. 已存在且 checksum 相同的语言标记为成功并 no-op。
3. 只重试 failed/pending 语言；禁止重发或覆盖成功版本。
4. 全部必需语言成功前保持 `aggregate: incomplete`，不发送
   `sdd_sdk_release_ready`，Renovate 不得选择该版本。
5. 若需要改变已发布内容，将该聚合版本标记 abandoned，并创建新的合同 patch
   version 完整发布。

### Compiler 重复创建跨仓库 Issue

检查：

- stable marker 是否存在。
- Compiler 是否读取 open 和 closed Issues。
- publish 是否使用 concurrency lock。
- delivery ID 是否持久化。
- task ID 是否错误包含标题、时间或数组下标。

### Backend 已部署但客户端不能启用

检查：

- deployed event 的 contract version。
- 客户端 package-manager lockfile、SDK metadata 和 ready manifest。
- feature flag 环境配置。
- production provider conformance。
- 是否仍被 migration Issue 阻塞。

## 19. 验收场景

系统落地完成前至少验证：

1. Factory dry-run 不产生 GitHub 写操作。
2. 六个产品仓库均由 Template 建立初始 `main`，再通过 Bootstrap PR 产生真实 check
   context 后启用 required checks。
3. `sdd-sync.yml` 进入默认分支且 App 安装后，dispatch smoke test 成功。
4. GitHub App 只能访问授权仓库，且是唯一 `sdd_*` dispatch 发送方。
5. 同一 backlog 发布两次不创建重复 Issues。
6. 跨仓库依赖可解析到真实 Issue URL。
7. Contract Gate 不依赖 backend 已实现。
8. Contract Gate 验证的 SDK artifact checksum 与最终发布内容完全一致。
9. SwiftPM 从 SDK 仓根 `Package.swift` 解析，并识别纯 SemVer `1.4.0` tag。
10. TS/Kotlin/Swift SDK 与 contract 使用相同版本和 checksum。
11. npm 成功、Maven 失败时 aggregate 保持 incomplete；重跑不重复覆盖 npm。
12. 全部必需语言成功后才产生签名 ready manifest 和客户端 fan-out。
13. Renovate 能分别创建 npm/Maven/SwiftPM bump PR，且无需修改客户端 `sdd.lock`。
14. Major SDK update 不会自动合并。
15. Provider 偏离 OpenAPI 时 implemented gate 失败。
16. 部署版本不匹配时 deployed gate 失败。
17. 丢失 dispatch 后 reconciliation 能发现。
18. In Progress/Done Issue 不会被自动覆盖。
19. 每个仓库可以独立 release。

## 20. 建议实施顺序

不要一次实现所有自动化：

### Phase 1：控制面和手工同步

1. 建立 `sdd-platform` schemas、Factory、Compiler。
2. 创建 product-control 和四个实现仓库。
3. 实现 stable-ID Issue publish。
4. Contract bundle 先使用 GitHub Release。
5. 跨仓库同步先人工触发 workflow_dispatch。

### Phase 2：Contract 与 SDK

1. 增加 api-clients 仓库。
2. 实现三语言 SDK codegen/build/test。
3. 将 Swift `Package.swift` 放在仓库根，并发布 npm、Maven、纯 SemVer Swift tag。
4. 实现逐语言 publish state、artifact attestation 和 aggregate ready manifest。
5. 下发只读取 ready versions 的 Renovate 配置。
6. 实现 resolved SDK↔metadata↔contract bundle consistency gate。

### Phase 3：事件自动化

1. 部署 GitHub App。
2. 实现 repository dispatch。
3. 实现 breaking impact 和 migration Issues。
4. 实现 implemented/deployed events。
5. 实现 delivery audit 和 retry。

### Phase 4：治理和对账

1. 实现 nightly reconciliation。
2. 配置 Organization Project automation。
3. 增加审计和指标。
4. 仓库规模增长后再引入 Terraform/Pulumi 管理非敏感配置。

## 21. 第一版完成定义

满足以下条件才算 Multi-repo MVP 可用：

- Product-control 能完成 Spec/Architecture/Design/Plan Gates。
- Compiler 能按平台幂等发布到不同仓库。
- Contract Gate 能发布固定 bundle 和带 attestation 的 SDK 候选产物，且不等待 backend 实现。
- Backend 能对固定合同执行 provider conformance。
- API Clients 能发布三语言 generated SDK，并在全部必需语言成功后生成 aggregate-ready manifest。
- 客户端能通过 package-manager lockfile 固定 SDK，并用内嵌 metadata 校验合同版本。
- GitHub Project 能聚合但不复制 backlog。
- 至少一个纵向功能完成 backend/web/iOS/Android 的跨仓库闭环。
- 自动化失败可通过审计记录和 reconciliation 恢复。

完整版本再增加：

- Renovate 自动升级。
- GitHub App fan-out。
- SDK 部分发布自动恢复。
- Implemented/deployed 事件。
- Breaking API migration 自动协调。

## 22. Hybrid 迁移

不要求一次性从 monorepo 拆成所有仓库。推荐逐步：

```text
阶段 1：product + all apps monorepo
阶段 2：共享 backend 拆出 acme-api
阶段 3：合同和 SDK 版本化
阶段 4：按真实团队/权限边界拆 Web 或移动端
```

每次拆分前必须：

- 固定 requirement/task/operation/screen IDs。
- 建立 contract version、Backend `sdd.lock` 和客户端 package-manager lockfile/SDK metadata 校验。
- 迁移真实 Issues，不复制任务状态。
- 验证新仓库 provider/client CI。
- 保留旧路径到新仓库的迁移说明和历史链接。
