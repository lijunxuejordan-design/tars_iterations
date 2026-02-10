# CI/CD 端到端链路设计（多仓 + 上板 + 可视化验证）

本文给出**完整链路**与**集成顺序**，并提供各环节**工具建议**。不包含详细脚本/代码实现，便于按你们现有平台（GitHub Actions / GitLab CI / Jenkins / Buildkite 等）落地。

---

## 目标与约束

- **目标能力**
  - 多仓源码统一拉取与版本编译
  - 统一版本号与打标签（可追溯：源码 → 构建 → 产物 → 上板 → 测试 → 可视化）
  - 静态检查（含圈复杂度、指标 metrics、质量门禁）
  - 实物单板烧录/刷写（可重复、可回滚）
  - 单板环境测试执行（含性能指标：例如帧率、延迟、丢帧、CPU/GPU/内存等）
  - 可视化验证（例如 Unity 触发功能、录屏/截图/比对）
  - 整体集成性能（耗时）持续改进（缓存、并行、分层、可观测）

- **关键约束/前提**
  - 上板/Unity 环节通常需要**专用Runner（硬件实验室）**，与云端构建Runner隔离
  - 多仓依赖需要**确定性**（锁定commit或tag），否则难以复现
  - 产物与测试数据需要**制品库/对象存储**，保证可追溯与复用

---

## 仓库划分（4类）与“系统集成”的边界

你们的系统由 4 类仓库组成，建议用“**组件仓自治CI + 集成仓编排CI**”来做端到端集成：

1. **瑞芯微平台仓**：RK3588、1126B 及其单板软件（固件/系统镜像/驱动/中间件/业务进程等）
2. **NVIDIA/高算力平台仓**：Orin 或 THOR 相关软件（同样可能包含系统镜像/容器/业务进程/推理等）
3. **Unity 客户端仓**：部署在 Pad 或手机侧（Android/iOS），用于可视化交互与验证
4. **云端服务仓**：前端 + 后端（API、控制面、数据面、运维面板等）

**系统集成（System Integration）定义**：在同一个“版本坐标”下，把 1/2/3/4 的制品部署到同一套联调环境（云端 + 设备池 + Pad/手机），执行端到端用例与性能/可视化验证，并将证据与指标回传形成趋势。

---

## 总体链路（从提交到发布）

建议将流水线分为 3 条主链路（可复用同一套模板）：

1. **PR验证链路（快）**：强调反馈速度，尽量不跑上板/Unity，只做静态检查 + 单元/仿真（如果有）
2. **主干集成链路（全）**：主干合入后触发，多仓统一版本构建 + 制品发布 + 上板刷写 + 上板测试 + Unity可视化验证
3. **发布链路（可控）**：手动或按标签触发，固定版本号，产物签名/归档，生成Release Notes

---

## 端到端集成环境拓扑（推荐）

为避免“各仓各跑各的，最后联调不可复现”，建议明确一套固定拓扑与资源角色：

- **云端（Cloud Staging/QA）**
  - 部署云端服务仓（前端/后端/数据库/消息/缓存等）
  - 提供面向设备与Unity的统一API与观测（日志/指标/追踪）
- **设备池（Hardware Lab）**
  - RK3588 / 1126B 单板池（可刷写、可采集指标）
  - Orin / THOR 设备池（可刷写/可部署容器/可采集指标）
  - 统一调度（避免抢占/冲突），并支持远程电源/复位
- **Pad/手机池（Mobile Lab，可与设备池同机房）**
  - Android 设备（建议优先支持）
  - iOS 设备（如需要，需额外macOS构建与签名体系）
  - Unity App 安装/启动/自动化触发/录屏截图

> 备注：如果短期无法建设移动设备池，可先用“模拟器/仿真 + 少量真机”过渡，但最终可视化验证还是建议落到真机。

---

## 集成顺序（推荐）

### 阶段 0：触发与上下文解析（必做）

- **输入**：触发源（PR / push / tag / 手动）、目标环境（dev/stage/prod）、目标单板型号、Unity场景等
- **输出**：本次流水线“**版本坐标**”（见下文《统一版本号与标签策略》）、需要拉取的多仓清单与精确引用、将运行哪些测试集
- **工具建议**
  - CI平台：GitHub Actions / GitLab CI / Jenkins / Buildkite（任选）
  - 配置与参数：YAML + 可审计的 Pipeline-as-Code；敏感信息通过 CI Secret 管理

#### 阶段 0.1：影响分析与执行计划生成（核心：变更→组件→构建目标→测试集）

这一步的目标是让流水线“**足够快**”且“**足够安全**”：能按变更范围只跑必要的构建/测试，但一旦命中高风险规则就自动升级为全量，且每次选择都可解释、可复现。

建议把影响分析做成一个**独立的Plan生成器**（不强依赖CI平台），输出统一的 `plan.json`（或YAML）作为后续所有Job的输入；同时把“为什么选这些目标/测试”的理由写进Plan，便于审计与排障。

##### Step 1：拿到变更集（Change Set）

- **单仓 PR/MR（常见）**
  - 变更文件列表：`git diff --name-only origin/main...HEAD`
  - 也可用平台API拿“PR changed files”（更快，避免拉全历史）
- **多仓/系统集成（manifest场景）**
  - 集成仓内：对比 manifest 的变更（系统版本坐标变化）
  - 组件仓内：由 manifest 得到每个仓的 ref，再分别做 diff
  - 推荐基准：把“**上次通过的系统版本**”（last green system version）记录下来（例如在集成仓打tag或写入一份 `last_green.json`），用它做对比基线，确保“从稳定→当前”的差异可复现

输出：`changed_files[]`（可加 `repo` 字段，形成 `repo:path`）

##### Step 2：文件映射到“组件/子系统”（Change → Component）

最简单且非常有效：维护一份路径归属表，例如 `ownership/map.yaml`，做“路径前缀 → 组件”映射。

- **示例（思路）**
  - `kernel/` → `kernel`
  - `rkipc/` → `ipc`
  - `buildroot/` → `rootfs`
  - `unity/Assets/...` → `unity_client`
  - `cloud/backend/...` → `cloud_backend`
  - `cloud/frontend/...` → `cloud_frontend`

并在规则中加入“**升级为全量**”的开关（高风险判定）：

- 改到以下内容时，直接判定 `risk=high`，升级为**全量构建/全量测试**（至少对该仓全量；系统集成可按策略升级为全系统全量）：
  - `toolchain/`、`configs/`、`Kconfig`、公共头文件目录（如 `include/`）、基础镜像/基础容器（如 `docker/base/`）、CI/构建模板本身
  - 任何会影响“ABI/接口契约/部署拓扑”的变更（例如 proto/OpenAPI、设备-云协议、Unity通信协议）

输出：`changed_components[]` + `risk_level` + `reasons[]`

##### Step 3：计算依赖闭包（Component → Downstream Closure）

变更影响往往不是“只影响本组件”，需要把受影响范围扩展到下游。

- **轻量方案（推荐一期开局）**：手工维护组件依赖图（DAG）
  - 例如：`kernel → rootfs → image`，`cloud_backend → e2e_api_tests`
  - 影响传播：从变更组件出发做下游闭包，得到 `impacted_components[]`
- **重量方案（更自动、更精确）**：用构建系统的依赖查询
  - Bazel：`bazel query "rdeps(//..., <changed_targets>)"` 反推受影响目标（精度高）
  - CMake/Make：基于 `compile_commands.json` + `clang-scan-deps` / include分析推导头文件影响（工程量较大，建议二期）

输出：`impacted_components[]` + `closure_reasons[]`

##### Step 4：选择要跑的构建目标（Component → Build Targets）

建议区分“总是跑”和“按影响跑”：

- **总是跑（快速门禁）**
  - lint/静态检查（尽量diff-based）
  - 最小 smoke build（验证关键二进制/包能产出）
- **按影响跑（节省大量时间）**
  - 只构建受影响组件及其下游制品（镜像/容器/移动端包）
  - 若 `risk=high`：升级为全量构建（至少覆盖该仓；系统集成可升级到全矩阵或指定关键矩阵）

建议维护一份 `buildplan/map.yaml`：**组件 → 构建目标集合**，并把“目标矩阵”纳入选择（见上文 manifest 目标矩阵）。

输出：`build_targets[]`（例如 `rk3588.image`、`1126b.image`、`orin.container`、`cloud.backend_image`、`unity.android_apk`）

##### Step 5：选择要跑的测试（Build Targets → Test Suites）

先做测试分层，再做影响选择与经验回归加权。

- **测试分层（建议固定口径）**
  - `smoke`（分钟级）
  - `functional`（10–30分钟）
  - `performance`（长耗时/资源占用高）
  - `visual`（Unity可视化）
- **映射表（强烈推荐）**：维护 `testplan/map.yaml`：**组件 → 必跑测试集**
  - 例如：
    - `camera` → `fps/drop/latency`（上板性能）
    - `ota` → `A/B升级+回滚`（上板）
    - `cloud_backend` → `api_contract + integration`（云端）
    - `unity_client` → `visual_smoke + scene_trigger`（真机或模拟器）
- **经验回归（低成本增益）**
  - 基于历史失败统计：记录“哪个目录/组件的改动最容易打挂哪些测试”
  - 用简单的 `变更路径 × 失败计数` 加权；命中就额外跑对应测试（避免只靠静态映射漏测）
- **兜底策略**
  - `nightly`/每日：全量构建 + 全量测试（覆盖影响分析漏网）
  - 当 `risk=high` 或出现“新测试/新组件未纳入映射表”时：自动升级（至少跑全量 smoke + 关键E2E）

输出：`test_suites[]` + `test_selection_reasons[]`

##### Plan的标准化输出（建议）

为了让“选择可解释”，建议Plan里至少包含：

- 变更集：`changed_files`（带 repo）
- 组件映射：`changed_components`、`impacted_components`
- 风险：`risk_level`、`escalations[]`
- 构建：`build_targets`
- 测试：`test_suites`
- 理由：每项选择的 `reasons[]`
- 基线：`base_refs`（上次通过系统版本/各仓ref）

后续阶段（构建/刷写/测试/Unity）只消费Plan，不再各自“重复做判断”，从而保证一致性与可复现。

##### 影响分析的更优方案（从易到难的升级路线）

你们当前方案（输入变更文件列表 → `map.yaml` 规则映射组件 + 高风险升级全量 → 轻量DAG闭包 → 输出合同：`full_build / components / max_severity / unmatched_files`）已经是**性价比很高**的落地形态。进一步优化的关键方向是：**更少误伤（少跑）**、**更少漏测（该跑必跑）**、**可解释可治理（持续收敛）**。

下面给出从易到难的升级路线，均可在不改变总体架构的前提下逐步演进。

###### V1.5：规则增强（不引入重型依赖解析）

- **严重级别细化（max_severity分层）**
  - 把 `max_severity` 从 {low, high} 扩到 {low, medium, high, critical}
  - 建议把严重级别直接映射到“要额外跑的层级”：
    - `high`：全量构建 + `functional`
    - `critical`：在 `high` 基础上再加 **上板（刷写+关键性能）+ `visual`（Unity）**
- **高风险规则做细、做可解释**
  - 将 `toolchain/`、`configs/`、`Kconfig/defconfig`、`DTS`、`include/公共头文件`、协议/IDL（proto/OpenAPI等）、镜像/基础容器等拆分为不同严重级别
  - 在Plan里输出 `reasons[]`（命中哪条规则、由哪个文件触发），便于审计与排障
- **组件内“子域/二级粒度”**
  - 组件命名支持 `component:subdomain`（仍是规则驱动，但显著减少误伤）
  - 示例：`kernel_6_1:media`、`kernel_6_1:net`、`buildroot:package/<name>`、`cloud_backend:auth`
- **unmatched_files 的治理闭环**
  - 只要出现 `unmatched_files`：
    - 一期策略：默认提升为 `high/critical`（保守不漏测）
    - 治理策略：要求补齐一条映射（让 `unmatched_files` 趋近于 0）

###### V2：从“组件”升级到“构建目标 targets”（误伤最小化的关键）

核心升级是：Plan 不仅输出 `components`，还输出 `build_targets`（CI按targets生成构建矩阵，而不是按组件粗粒度展开）。

- **输出合同建议扩展**
  - 新增：`build_targets[]`（例如 `kernel_image`、`rootfs_rk3588`、`rootfs_1126b`、`orin_container`、`unity_android_apk`、`cloud_backend_image`）
  - CI：按 `build_targets` fan-out 并行构建；后续刷写/测试/Unity也按 targets 选择执行
- **按构建系统的落地方式**
  - **Bazel（最优）**：用 `bazel query "rdeps(//..., <changed_targets>)"` 精确反推受影响 targets（误伤最小）
  - **CMake/Ninja（工程量中等，建议二期）**：基于 `compile_commands.json` + 依赖扫描近似推导“源文件→目标”
  - **Buildroot/Yocto（可做到包/配方级）**
    - Buildroot：`package/<pkg>` 变更 → 重建相关包 + rootfs打包（不必全量toolchain）
    - Yocto：recipe/layer 级别归因 → 重建相关配方 + image重打包；命中 `layer.conf/local.conf` 等则升级高风险

###### V2+：测试选择更聪明（从“组件→必跑测试”升级到“证据驱动”）

在 `testplan/map.yaml（组件→必跑测试集）` 基础上增加两类“自动加测”：

- **历史失败加权（实现简单、收益大）**
  - 用历史数据统计：`变更路径/组件 × 测试集` 的失败次数/失败率
  - 命中热点回归就额外跑对应测试，快速提升“命中率”
- **覆盖率导向（精度高，但依赖体系完善）**
  - 对云端后端/部分客户端逻辑，可用覆盖率把测试映射到模块，做到“只跑覆盖到变更区域的测试”
- **flaky治理（减少噪声拖慢）**
  - 把不稳定测试隔离成单独层级；失败自动重跑确认；长期以修复/隔离为目标收敛

###### V3：依赖闭包来源自动化（减少手工DAG维护成本）

你们的轻量DAG非常实用；更进一步可以逐步“自动导出/校准”依赖闭包来源，降低手工维护成本：

- **从发布/打包拓扑导出**：镜像分层、容器 `FROM` 链、Helm chart依赖、OTA包组成关系
- **从接口契约导出**：proto/OpenAPI/设备-云协议/Unity通信协议变更 → 自动升级严重级别并选择E2E/visual
- **质量指标闭环（用nightly做真值近似）**
  - 统计“漏测率/误杀率”，用 nightly 全量结果回溯并校准规则与DAG

###### V4（可选，最强兜底）：语义级变更检测（接口/协议/ABI）

适合“云 + 板 + Unity”的强集成系统，用于显著降低跨端联动的漏测风险：

- **协议/接口契约变更检测**：对 proto/OpenAPI/schema 做diff，一旦变化直接标记 `critical` 并强制跑关键E2E + `visual`
- **ABI/API兼容性检查（C/C++关键SDK/中间件）**：对导出符号与兼容性做检查，作为 `high/critical` 触发器

### 阶段 1：多仓拉取与依赖锁定（编译前置）

核心目标是“**一次构建可复现**”。

- **多仓策略（选一种主打法）**
  - **A. Manifest/Repo管理（强烈推荐用于嵌入式/多组件）**
    - 用一个“集成仓（integration repo）”维护 manifest（列出各仓URL + 分支/commit/tag）
    - 主干集成只改 manifest；所有构建从 manifest 精确拉取
  - **B. Monorepo（如果未来可迁移）**
    - 简化依赖，但迁移成本高
  - **C. CI里动态拉取（仅在过渡期）**
    - PR里解析依赖仓分支，风险是复现性差，需要额外的锁定逻辑

- **工具建议**
  - 多仓管理：`repo`（Android/嵌入式常用）、`west`（Zephyr）、或自定义 manifest（YAML/JSON）
  - 源码拉取加速：Git 镜像/缓存（CI缓存 + 内网mirror）

### 阶段 2：统一版本号与打标签（编译与制品的“身份证”）

**版本号必须贯穿：源码 → 编译 → 制品 → 上板 → 测试 → Unity验证**。

- **推荐版本结构（示例）**
  - SemVer：`MAJOR.MINOR.PATCH`
  - 预发布/构建元数据：`MAJOR.MINOR.PATCH-rc.N+<shortsha>.<date>`
  - 内部构建号：CI自增 `build_id`（便于排序与回溯）

- **推荐策略**
  - PR验证：只生成临时版本（不打正式tag），例如 `0.0.0-pr.<PR>-<sha>`
  - 主干集成：生成“可用版本”（可选打 `nightly`/`snapshot` tag）
  - 发布：由 Release 流水线执行：
    - 更新版本号（或从变更日志推导）
    - 创建并推送 tag（必要时签名）
    - 生成 Release Notes

- **工具建议**
  - 版本生成：`GitVersion` / `semantic-release` / 自定义规则（以git tag与commit为源）
  - Tag/Release：`git tag` + 平台Release（GitHub Release / GitLab Release）
  - 变更日志：`Conventional Commits` + `release-please`/`semantic-release`（如团队可接受）

#### 2.1 针对“4类仓库”的版本坐标建议（强烈推荐）

建议引入**系统版本（System Version）**，它不是简单等于某一个仓的tag，而是一个“组合版本坐标”：

- **系统版本号**：例如 `sys-1.8.0+20260209.<build_id>`
- **系统版本内容**：manifest 里锁定 4 类仓库的精确引用（commit 或组件tag）
  - `rk_repo`: `<commit/tag>`（可进一步细分 rk3588 / 1126B 两条构建）
  - `orin_thor_repo`: `<commit/tag>`
  - `unity_repo`: `<commit/tag>`
  - `cloud_repo`: `<commit/tag>`

落地方式上：

- **每个组件仓**：仍保留各自的版本与tag节奏（便于独立发布/回滚）
- **集成仓**：用 manifest 形成“系统版本”，对外只认系统版本（便于联调、验收、回溯）

---

### 阶段 3：静态检查与质量门禁（圈复杂度、metrics）

静态检查建议拆为两层：**快速必跑** + **全量深检**。

- **检查维度**
  - 代码规范与格式：lint/format
  - 静态分析：潜在缺陷、未定义行为、线程/内存问题（视语言而定）
  - **圈复杂度**（Cyclomatic Complexity）：对关键模块/函数设置阈值
  - **指标 metrics**：可维护性指数、重复率、文件/函数规模、依赖层级
  - 安全与依赖：SCA、License、CVE（如需要）

- **质量门禁（Quality Gate）建议**
  - PR：只对变更范围做门禁（diff-based），避免历史债务阻塞
  - 主干：全量门禁 + 允许“例外清单”逐步收敛
  - 输出统一报告（HTML/JSON），并上传为制品

- **工具建议（按语言选用）**
  - 通用质量平台：**SonarQube / SonarCloud**（覆盖复杂度、重复率、质量门禁、趋势）
  - C/C++：
    - 静态分析：`clang-tidy`、`cppcheck`
    - 格式：`clang-format`
    - 编译器告警：`-Wall -Wextra -Werror`（按阶段逐步收敛）
  - Python：`ruff`、`mypy`、`radon`（复杂度/指标）
  - Java/Kotlin：`SpotBugs`、`Checkstyle`、`PMD`
  - JS/TS：`eslint`、`tsc`、`sonarjs`
  - 圈复杂度/metrics补充：
    - `lizard`（多语言复杂度/函数长度/参数等）
    - `radon`（Python复杂度/MI）
  - 覆盖率（如有单测）：`gcovr`/`llvm-cov`、`pytest-cov`、`jacoco`

### 阶段 4：版本编译与制品产出（多仓编译）

建议将构建产物分为：

- **构建输出**
  - 固件/镜像（bin/hex/elf/ota包等）
  - 符号与调试信息（可选拆包单独存储）
  - SBOM（如需要供应链合规）
  - 构建元数据（manifest快照、编译参数、工具链版本、CI build_id）

- **制品管理**
  - 每次主干集成构建都上传制品到制品库/对象存储
  - 产物命名包含：项目名 + 版本号 + 板型 + 构建号 + sha

- **工具建议**
  - 构建系统：CMake/Ninja、Bazel、Make、Gradle（按项目）
  - 依赖与包：Conan/vcpkg（C++）、pip/poetry、npm/pnpm、maven/gradle
  - 制品库：JFrog Artifactory / Nexus / GitHub Packages / S3兼容对象存储（MinIO）
  - SBOM：Syft + Grype（可选）

#### 4.1 组件仓“自治CI”建议（4类仓库各自先产制品）

把“系统集成”做稳，前提是每个仓先把自己的制品做成**可发布、可安装、可回滚**：

- **RK平台仓（rk3588 / 1126B）**
  - 产物：固件/系统镜像/OTA包/应用包 + 版本读回信息（build-id）
  - 工具链建议：交叉编译工具链容器化（Docker镜像固定版本）；如用 Yocto/Buildroot/自研SDK，则把 toolchain 与 layer/配置一并版本化
  - 静态检查：clang-tidy/cppcheck + lizard + SonarQube（如适用）
- **Orin/THOR平台仓**
  - 产物：镜像/容器镜像（强烈建议容器化交付业务服务）+ 配置包
  - 工具链建议：
    - Orin（Jetson类）：建议用固定的L4T/JetPack版本作为基线；构建环境用容器封装
    - THOR：按厂商SDK/工具链固定版本
  - 制品：除二进制外，强烈建议产出可部署的容器镜像（推到私有镜像仓库）
- **Unity仓（Pad/手机）**
  - 产物：Android `apk/aab`，iOS `ipa`（如需要）
  - 构建建议：
    - Android：Linux runner + Unity batchmode + Gradle
    - iOS：必须 macOS runner + 签名体系（证书/Provisioning Profile），建议用 Fastlane 管理发布到 TestFlight/企业分发
  - 证据：自动化运行日志、打包元数据（Unity版本、包名、git sha）
- **云端服务仓（前端+后端）**
  - 产物：后端容器镜像（或包）、前端静态资源包（或前端镜像）
  - 部署建议：Kubernetes + Helm/Kustomize；用 GitOps（Argo CD/Flux）做“声明式发布与回滚”
  - 质量：SAST + 依赖漏洞扫描（可选）+ API契约检查（OpenAPI/Proto等）

### 阶段 5：硬件单板烧录/刷写（真实环境）

这一步必须在“**硬件实验室Runner**”上运行，并具备可观测与可恢复能力。

- **设计要点**
  - Runner与单板绑定（或通过调度系统分配）
  - 刷写前检查：单板在线、串口/网络可达、存储空间、温度/电源状态（如有）
  - 刷写后验证：读回版本信息/校验和、启动日志关键字、健康检查端点
  - 失败策略：自动重试（有限次数）、自动电源循环（如果有PDU）、自动回滚到上一个稳定版本

- **工具建议**
  - 刷写工具：按芯片/平台选（例如 `openocd`、`dfu-util`、`fastboot`、厂商烧录工具CLI）
  - 设备控制：串口工具（`pyserial`生态）、远程电源PDU（APC等）或自研继电器控制
  - 实验室编排/设备农场：
    - 开源：LAVA（常用于板级自动刷机/测试）
    - 商业/平台：Device Farm类产品（如有预算）
  - 日志归档：串口日志、dmesg、应用日志统一上传到对象存储 + 关联本次版本坐标

#### 5.1 针对两类硬件平台的刷写落地建议

- **RK3588 / 1126B**
  - 典型方式：USB烧录（Maskrom/Loader）、分区镜像升级、或OTA（取决于你们固件体系）
  - 关键点：把“刷写介质与参数”标准化为设备能力描述（device capability），由调度层选择正确流程
- **Orin / THOR**
  - Orin（Jetson类）常见：SDK/flash脚本/网络刷写/OTA（依项目）
  - THOR：按厂商提供的flash/升级机制封装为统一接口

> 重要：不论哪种平台，CI侧只依赖“统一刷写接口 + 设备描述”，避免把平台细节散落在多个流水线里。

### 阶段 6：单板环境测试执行（性能指标：帧率等）

建议将测试分层，逐层加大成本：

- **测试分层**
  - Smoke（必跑，分钟级）：启动、基础功能、关键接口可用
  - Functional（中等，十分钟级）：功能用例集
  - Performance（成本高，受控触发）：帧率、延迟、吞吐、资源占用、稳定性（长稳）

- **指标建议（示例）**
  - FPS：平均/1% low/0.1% low，丢帧率，抖动（帧间隔方差）
  - 端到端延迟：P50/P95/P99
  - 资源：CPU/GPU/内存/温度/功耗（可选）
  - 稳定性：崩溃率、重启次数、错误码分布

- **工具建议**
  - 测试框架：pytest/Robot Framework/自研测试驱动（按语言与现有资产）
  - 远程执行：SSH + 受控命令白名单；或通过 LAVA job
  - 指标采集：
    - 应用内埋点 + 输出JSON
    - 系统级：`perf`、`top`/`pidstat`、`tegrastats`（NVIDIA平台）、厂商profiling工具
  - 结果存储与趋势：
    - 时序：Prometheus + Grafana
    - 报告：Allure（测试报告可视化）
  - 质量门禁：对关键指标设阈值（例如 FPS 不低于某值，P95延迟不高于某值），并支持“基线对比”

#### 6.1 端到端测试编排（跨“云 + 板 + Pad/手机”）

为了让 4 类仓库真正“集成验证”，建议至少具备一套 E2E 编排能力：

- **编排建议**
  - 先部署云端服务到 `staging`（拿到环境地址与访问凭据）
  - 再刷写 RK / Orin(THOR) 到目标系统版本（并注册到云端）
  - 再安装 Unity App 到 Pad/手机（配置指向同一 `staging`）
  - 最后启动端到端用例：Unity触发功能 → 云端下发/协同 → 设备侧执行 → 回传结果与指标

- **工具建议**
  - 编排层：集成仓提供“测试计划（test plan）”描述（YAML/JSON），由CI执行器解析
  - 移动端安装与控制：
    - Android：ADB（安装/启动/抓logcat/录屏）
    - iOS：需要macOS + 对应工具链（建议用 Fastlane 统一）
  - 云端联调：在CI里暴露每次部署的环境URL与版本坐标，便于回溯

### 阶段 7：可视化验证（Unity触发功能）

目标是把“人眼验证”的部分自动化到可重复、可回放。

- **推荐流程**
  - 在 Unity 侧提供**自动化入口**：场景加载、触发功能、采集结果（日志/截图/录屏/关键对象状态）
  - CI 在实验室Runner上启动 Unity（或Unity Player），执行自动化脚本
  - 输出“证据包”：日志 + 截图/视频 + 关键状态快照 + 版本坐标
  - 可选做视觉回归：与基线图片/视频关键帧做差异检测

- **工具建议**
  - Unity自动化：
    - Unity Test Framework（EditMode/PlayMode）
    - 命令行批处理（batchmode）运行测试/场景
  - 视觉回归：
    - 截图对比：SSIM/PSNR（算法层面），或现成的图像diff工具链
    - UI自动化：如需要可引入外部驱动（取决于运行环境）
  - 证据归档：对象存储 + 在CI页面贴出关键截图/链接

---

## 多仓版本编译：推荐的落地模型

### 集成仓（Integration Repo）+ Manifest（推荐）

- **做法**
  - 新建（或指定）一个“集成仓”只负责：
    - `manifest.yml`（列出各业务仓的精确引用：commit/tag）
    - 统一构建入口（构建参数矩阵：板型/特性开关/工具链版本）
    - 统一测试入口（上板任务定义、Unity任务定义）
  - 各业务仓独立开发，主干稳定后，通过PR更新 manifest 引用

- **收益**
  - 可复现（任何时刻manifest都能复现当时构建）
  - 变更隔离（业务仓不必关心其他仓的临时分支）
  - 易于做“集成门禁”（只有集成仓合入才触发全链路）

#### 适配你们“4类仓库”的 manifest 结构建议

manifest 里建议明确 4 类仓库与“目标矩阵”：

- **组件引用**
  - `rk_repo`：同时包含 `rk3588` 与 `1126B` 的构建目标（同仓或分仓均可）
  - `orin_thor_repo`：包含 `orin` 与 `thor` 的构建/部署目标
  - `unity_repo`：包含 `android` / `ios` 目标（如iOS暂不做，可先只定义android）
  - `cloud_repo`：包含 `frontend` / `backend` / `infra`（如有）
- **目标矩阵（示例维度）**
  - `hw_target`: `rk3588` | `1126b` | `orin` | `thor`
  - `mobile_target`: `android` | `ios`
  - `cloud_env`: `staging` | `perf` | `prod`

这样集成流水线可以做到：一次系统版本构建，按矩阵选择需要的构建与验证组合（例如只对 `rk3588+android+staging` 跑快集成，对全矩阵跑 nightly）。

---

## 工具链建议（按“平台能力矩阵”选型）

### CI编排层（任选其一）

- **GitHub Actions**：生态强，易接入Release/Packages，适合公网与开源/内网混合
- **GitLab CI**：自带制品与环境概念强，适合一体化
- **Jenkins**：高度可定制，适合复杂自建实验室与老系统整合
- **Buildkite**：适合自托管Agent，硬件实验室场景友好

### 制品与数据

- **制品库**：Artifactory / Nexus / GitHub Packages / GitLab Package Registry
- **对象存储**：S3/MinIO（存放日志、录屏、测试证据包）
- **指标与趋势**：Prometheus + Grafana（性能/稳定性趋势）
- **测试报告**：Allure（或CI原生JUnit报告）

### 代码质量/复杂度/metrics

- **首选统一平台**：SonarQube（或SonarCloud）
- **补充工具**：lizard（复杂度/函数指标），语言原生lint/format

### 上板与实验室

- **设备调度**：LAVA（推荐作为“上板刷写+测试”的中枢）
- **刷写**：openocd/dfu/fastboot/厂商CLI（按板型）
- **远程电源/复位**：PDU/继电器 + 标准化接口

### Unity可视化验证

- **Unity Test Framework + batchmode** 作为主入口
- **视觉回归**：截图diff/SSIM（按需求）

---

## 流水线性能（耗时）改进策略（第7点：持续提升）

### 1）度量先行：把“耗时分解”做出来

- 每个阶段输出：开始/结束时间、队列等待、重试次数、缓存命中率
- 形成趋势图：周/月维度，定位瓶颈（构建、拉取、上板占用、Unity运行等）

### 2）缓存与复用

- 源码：git mirror + shallow/partial clone（视情况）
- 依赖：包管理缓存（Conan/pip/npm/gradle）+ 远端缓存
- 构建：ccache/sccache（C/C++），Bazel remote cache（如用Bazel）
- 测试：只对受影响模块跑重测试（测试分层 + 变更影响分析）

### 3）并行与分层

- PR链路尽量并行：lint/静态分析/单测并行
- 主干链路分段并行：多板型构建矩阵并行；上板任务按设备池并行
- 将成本高的性能/长稳测试放到 nightly 或按需触发

### 4）硬件实验室吞吐提升

- 设备池化与自动调度（避免“人工抢板”）
- 刷写与测试脚本标准化，减少不可控依赖
- 失败快速判定（早停）：启动日志关键字/健康检查失败立即结束并释放设备

### 5）减少不必要的全量执行

- Diff-based质量门禁与测试选择
- 对历史债务采用“基线冻结 + 新增不退化”策略

---

## 交付物与可追溯性（建议的产物清单）

每次主干集成/发布建议最少产出：

- 构建制品：固件/镜像 + 校验和
- 构建元数据：manifest快照、工具链版本、编译参数、git sha、CI build_id
- 静态检查报告：复杂度/metrics/告警汇总
- 上板证据：刷写日志、启动日志、版本读回信息
- 测试报告：用例结果 + 性能指标（含原始数据）
- Unity证据：截图/录屏 + 自动化日志 + 可选差异报告

---

## 建议的实施路线图（从易到难）

1. **先打通 PR 快链路**：多仓拉取（最小化）+ 静态检查 + 基础构建
2. **引入 manifest 形成可复现构建**：主干构建统一版本与制品归档
3. **上板刷写自动化**：先单板单型号，再扩展设备池与调度
4. **上板测试分层**：smoke → functional → performance（引入阈值与趋势）
5. **Unity可视化验证自动化**：先“可运行+有证据”，再做视觉回归
6. **性能优化闭环**：基于阶段耗时与设备吞吐数据做缓存/并行/选择性执行

---

## 你们落地时需要提前准备的信息（不影响本文，但会影响实施）

- 板型与刷写方式（JTAG/USB DFU/fastboot/网络OTA等）
- 设备池规模与是否支持远程电源控制
- Unity运行环境（有无GPU、是否headless可用、目标平台Windows/Linux）
- 当前语言栈与构建系统（决定静态分析与缓存工具）
- 指标定义与门禁阈值（FPS/延迟/功耗等）

