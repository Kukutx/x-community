# Kukutx Connect

面向海外华人的本地社区、可信组织与直聘平台。产品以“可信企业 → 合规职位 → 隐私安全申请 → 招聘处理 → 受控沟通”为首发核心闭环；社区、商家、活动及后续变现保持清晰模块边界和 Feature Flag，而不是用功能数量代替成熟度。

> 当前状态：持续建设中，**不是 Release Candidate**。当前工作区是部分仓库快照；不会自动连接或修改任何真实云资源，也不会执行部署。准确进度见 `docs/implementation/status.md` 和 `docs/operations/launch-readiness-report.md`。

## 生产平台边界

生产服务统一采用 **GCP + Firebase**：

- Cloud Run：Next.js Web 与 ASP.NET Core API。
- Cloud Run Worker Pool：Outbox、通知、媒体处理和维护任务。
- Cloud Run Job：受控数据库迁移和权限协调。
- Cloud SQL for PostgreSQL 18：业务数据唯一真源。
- Memorystore for Redis：SignalR Backplane、短期缓存和协调，不作为业务真源。
- Google Cloud Storage：公开媒体、私有媒体和隔离区。
- Cloud Load Balancing、Cloud Armor、Cloud CDN、Artifact Registry、Secret Manager、Cloud Logging/Monitoring/Trace。
- Firebase Authentication、App Check、Cloud Messaging、Remote Config、Analytics、Crashlytics、Performance Monitoring 和 App Distribution。

明确不使用其他云应用托管、对象存储、数据库、缓存或函数平台；不使用 Firestore/Realtime Database 作为业务主库，也不把 Firebase Custom Claims 当作平台角色库。对应决定见：

- `docs/architecture/adrs/0001-crew-principles-and-deliberate-non-reuse.md`
- `docs/architecture/adrs/0002-gcp-firebase-single-cloud.md`

## 登录策略

默认启用：

```text
Google
Apple
手机号
```

默认禁用并禁止进入生产认证路径：

```text
邮箱密码
邮箱链接
匿名登录
```

可选扩展：

```text
微信
QQ
```

微信和 QQ 只能使用服务端授权码换票、Provider Subject HMAC 映射和 Firebase Custom Token；默认关闭。邮箱仅可作为联系资料，不能用于登录、自动合并账号、组织授权或账号恢复。

## 核心架构原则

1. PostgreSQL 是业务数据唯一事实源，ASP.NET API 是业务规则和授权事实源。
2. Firebase Token 只证明外部身份；平台角色、组织权限、账号状态和 Entitlement 存于 PostgreSQL。
3. 模块化单体与垂直切片优先，不为拆分而拆微服务。
4. 外部副作用通过 Transactional Outbox，在数据库事务提交后执行。
5. API、Worker 和 Migration 独立运行、扩缩与回滚。
6. 私有媒体不产生永久公开 URL；公开媒体仍通过私有 GCS Origin、Cloud CDN 和 Cloud Armor 提供。
7. API 不直接持有 GCS 对象权限；专用 Signer 与 Worker 使用最小权限身份。
8. 数据库 Runtime 使用 A/B 登录槽，二者只继承非登录最小权限角色；Migration 使用独立身份。
9. Cloud Run Secret 引用固定数字版本；镜像必须来自 Artifact Registry 并固定 Digest。
10. Dev、Staging、Production 使用独立 GCP/Firebase 项目、数据库、Bucket、Redis、App ID、域名和密钥。

## 当前可执行的离线门禁

```bash
python3 -m pip install -r tools/requirements-infra.txt
./tools/scripts/validate-gcp-firebase-only.sh
python3 -m unittest discover -s tools/tests -p 'test_*.py' -v
```

最近一次当前快照实测结果：

```text
Python 策略与变异测试：92 通过，0 失败
必需 GCP/Firebase 离线门禁：12/12 通过
Terraform 静态检查：0 错误，0 警告
仓库完整性：失败关闭（部分快照）
Terraform 原生门禁：失败关闭（缺精确 CLI 和 Provider Lockfile）
Release Eligible：false
```

机器证据：

- `dist/validation/gcp-firebase-architecture.json`
- `dist/validation/gcp-firebase-infra.json`
- `dist/validation/terraform-native.json`
- `dist/validation/gcp-firebase-evidence.sha256`

静态门禁不能替代 `terraform fmt/init/validate/plan`，也不能替代 .NET、pnpm、Web、Mobile、数据库和 E2E 验证。

## Terraform 设计

固定工具契约：

```text
Terraform 1.15.8
Google Provider 7.40.0
Random Provider 3.9.0
```

主要文件：

- `infra/terraform/versions.tf`
- `infra/terraform/main.tf`
- `infra/terraform/network.tf`
- `infra/terraform/database.tf`
- `infra/terraform/run.tf`
- `infra/terraform/storage.tf`
- `infra/terraform/iam.tf`
- `infra/terraform/monitoring.tf`
- `infra/terraform/policies/production-plan-policy.json`

真实自部署之前必须先生成完整、非 `-target` 的保存 Plan，并执行：

```bash
./tools/scripts/terraform-plan-review.sh <plan-or-plan-json> <environment>
```

Plan 门禁会拒绝非 GCP Provider、公共 Bucket/IAM、可变镜像、未固定 Secret、危险删除/替换、不完整 Plan，以及不安全的数据库 A/B 凭据轮换。

## 打包但不部署

统一入口：

```bash
./tools/scripts/package-release.sh --mode source-review
./tools/scripts/package-release.sh --mode verified-source
./tools/scripts/package-release.sh --mode release-candidate
```

- `source-review`：允许部分仓库，但包内明确标记 `NOT DEPLOYABLE`。
- `verified-source`：要求完整 Monorepo、干净 Git Commit 和全量验证证据。
- `release-candidate`：还要求 Staging、恢复、回滚、故障、负载、无障碍和移动生产构建证据。

当前实测：`source-review` 归档安全与 SHA-256 检查通过；`verified-source` 因仓库不完整正确拒绝并生成 0 个归档。三种模式都不会执行 Terraform Apply、Cloud Run Deploy、Firebase Deploy 或商店提交。

## 完整仓库恢复后的统一门禁

```text
pnpm format:check
pnpm lint
pnpm typecheck
pnpm test
pnpm test:integration
pnpm test:e2e
pnpm test:load
pnpm build
pnpm api:check-drift
pnpm db:verify
pnpm infra:validate
pnpm check
```

还必须完成 .NET Release Build、PostgreSQL/PostGIS 集成测试、OpenAPI Client 漂移检查、容器构建、真实 Dev/Staging Firebase 登录和 App Check、数据库迁移与 A/B 凭据轮换、备份恢复与回滚演练，才能生成最终自部署包。

## 文档入口

- `docs/implementation/status.md`
- `docs/implementation/next-tasks.md`
- `docs/operations/launch-readiness-report.md`
- `docs/security/authentication-and-account-linking.md`
- `docs/security/threat-model.md`
- `docs/operations/runbooks/terraform-plan-review.md`
- `docs/operations/runbooks/database-migration.md`
- `docs/operations/runbooks/database-audit.md`
- `docs/operations/runbooks/secret-rotation.md`
- `infra/database/security/runtime-role.sql`
- `infra/database/security/verify-runtime-role.sql`
