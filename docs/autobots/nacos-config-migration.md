# Autobots 配置迁移 Nacos 实现方案

> 分支：`feat/nacos-config-center`（基于 `poc`）  
> 范围：Java `application.yml`（gateway / user / external）+ Python `autobots-ai/.env_*`  
> 不在本期：`autobots-frontend` 的 Vite 构建期 `.env.*`（构建时注入，非运行时配置中心）

---

## 1. 目标

| 目标 | 说明 |
|------|------|
| 环境配置外置 | poc / pre / prod 的环境差异配置迁入 Nacos，仓库只保留连接 Nacos 的最小引导配置 |
| 统一隔离模型 | POC 与 Pre 共用同一 Nacos 集群，靠 **namespace** 隔离；Prod 独立集群 + namespace |
| 密钥不进 Git | MySQL / Redis / SMTP / API Key 等敏感值只存在于 Nacos（或 K8s Secret → 环境变量），仓库模板用占位符 |
| 本地可离线 | `local` 仍可用本地 `application.yml` / `.env_local`，不强制依赖 Nacos |
| 热更新可选 | 非连接类配置支持 `@RefreshScope` / Python 监听；DB/Redis 地址变更仍需滚动重启 |

---

## 2. Nacos 拓扑（运维侧）

连接信息由部署环境变量注入，**不要写进仓库**：

| 环境 | `NACOS_SERVER_ADDR` | `NACOS_NAMESPACE` | 账号 |
|------|---------------------|-------------------|------|
| POC | `10.100.123.84:8848` | `poc` | `NACOS_USERNAME` / `NACOS_PASSWORD` |
| Pre | 同上 | `pre` | 同上 |
| Prod | `nacos-public-yz.100credit.cn:80` | `a2644ce7-54ef-43a7-9a56-1bf7ce3f7897` | `NACOS_USERNAME` / `NACOS_PASSWORD` |

说明：

- POC/Pre 的 namespace 若控制台创建的是「名称」而非 UUID，以控制台实际 **Namespace ID** 为准填入 `NACOS_NAMESPACE`。
- 账号密码仅通过部署 Secret / CI 变量注入。

---

## 3. DataId / Group 约定

**Group**：统一 `AUTOBOTS_GROUP`（避免与其他系统 `DEFAULT_GROUP` 混用）。

**DataId**（建议 YAML；Python 也可用 `properties` / `env` 文本）：

| DataId | 归属 | 内容 |
|--------|------|------|
| `autobots-common.yml` | 共享（可选） | JWT 公共项、公共超时、非敏感开关等 |
| `autobots-gateway.yml` | gateway | 路由、auth.skip-urls、Redis、JWT、环境相关 upstream |
| `autobots-user.yml` | user | datasource、Redis、lead-push、traffic-alert、proxy 等 |
| `autobots-external.yml` | external | datasource、Redis、external-interface.url、proxy 等 |
| `autobots-ai.yml` | Python | 扁平 YAML（KEY: VALUE），对应现有 `.env_poc` / `.env_pre` / `.env_prod` |

同一 DataId 在不同 namespace 下各一份（poc / pre / prod），**不要**用 DataId 后缀区分环境（避免和 namespace 双重维度）。

优先级（高 → 低）：

```
环境变量 / 系统属性
  > Nacos 私有配置（各服务 DataId）
  > Nacos 共享配置（autobots-common.yml，若启用）
  > 本地 application.yml 通用段（兜底 + 本地开发）
```

---

## 4. Java 实现方案

### 4.1 依赖

当前栈：Spring Boot `3.5.4` + Spring Cloud `2025.0.0`。

父 POM 增加 Spring Cloud Alibaba BOM（对齐 Boot 3.5）：

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-alibaba-dependencies</artifactId>
  <version>2025.0.0.0</version> <!-- 以官方 2025.0.x 最新稳定版为准 -->
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

在 `autobots-gateway` / `autobots-user` / `autobots-external` 引入：

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

本期默认 **只接 Config，不接 Discovery**（现有已是 K8s / 显式 URL 路由）。若后续要注册中心再加 `nacos-discovery`。

### 4.2 本地引导配置（保留在仓库）

各服务 `application.yml` 精简为：

1. **通用非敏感结构**（端口默认、MyBatis、连接池参数模板等，可用 `${...}`）
2. **Nacos 连接 + `spring.config.import`**
3. **`local` profile** 完整本地配置（不走 Nacos 或 `optional:nacos:`）

示例（gateway，其余服务同理，DataId 换名）：

```yaml
spring:
  application:
    name: autobots-gateway
  cloud:
    nacos:
      config:
        server-addr: ${NACOS_SERVER_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:}
        username: ${NACOS_USERNAME:}
        password: ${NACOS_PASSWORD:}
        group: AUTOBOTS_GROUP
        file-extension: yml
  config:
    import:
      - optional:nacos:autobots-common.yml?group=AUTOBOTS_GROUP&refreshEnabled=true
      - optional:nacos:${spring.application.name}.yml?group=AUTOBOTS_GROUP&refreshEnabled=true

---
spring:
  config:
    activate:
      on-profile: local
  cloud:
    nacos:
      config:
        enabled: false
# ... 保留现有 local 段 ...
```

要点：

- 使用 `spring.config.import`（SCA 2025.0.x 推荐；勿再依赖 bootstrap）。
- poc/pre/prod 用 `optional:nacos:` 便于本地无 Nacos 时降级；上线可改为强制 `nacos:` 以便配置缺失时快速失败。
- 从现有 `on-profile: poc|pre|prod` 段 **剪切** 到 Nacos 对应 namespace 的 DataId；本地文件删除这些 profile 段（或留空壳注释指向 Nacos）。

### 4.3 从现有 YAML 拆分

每个服务当前结构：`通用段` + `local|poc|pre|prod`。

迁移映射：

| 本地来源 | 目标 |
|----------|------|
| 通用段中真正跨环境不变的 | 继续留在 jar 内 `application.yml` |
| `on-profile: poc` | Nacos ns=`poc` → `autobots-{svc}.yml` |
| `on-profile: pre` | Nacos ns=`pre` → 同名 DataId |
| `on-profile: prod` | Prod Nacos ns → 同名 DataId |
| `on-profile: local` | 留在仓库 |

仓库增加可导入模板（无真实密钥）：

```
configs/nacos/
  README.md
  templates/
    autobots-gateway.yml
    autobots-user.yml
    autobots-external.yml
    autobots-ai.yml
  scripts/
    publish-nacos.sh   # 用环境变量登录并发布（不写死密码）
```

`PROJECT_NAVIGATOR.md` 已预留 `configs/nacos/`，与此对齐。

### 4.4 动态刷新

- 开关类、超时、skip-urls、业务 URL：加 `@RefreshScope` 或 `EnvironmentChangeEvent` 监听。
- `spring.datasource.*` / Redis 连接：变更后需重启，文档标明「连接类配置不热更」。
- 注意 SCA 2025.0.x 上 `spring.config.import` 路径曾有 refresh 问题（社区 issue #4331）；落地时用 POC 验证一次改配置是否生效，必要时钉到含修复的小版本。

---

## 5. Python（autobots-ai）实现方案

### 5.1 现状

`config/config.py`：`CONF_ENV` → `.env_{env}` → `load_dotenv` → `os.getenv`。

### 5.2 目标加载顺序

```
1. 进程环境变量（K8s / Docker 已注入的优先保留）
2. Nacos DataId: autobots-ai.yml（当 NACOS_SERVER_ADDR 存在且 CONF_ENV != local）
3. 本地 .env_{CONF_ENV} / .env_local（开发兜底）
4. 代码内默认值（逐步清空敏感默认值）
```

### 5.3 依赖与模块

- `requirements.txt` 增加：`nacos-sdk-python`、`PyYAML`。
- 新增 `config/nacos_loader.py`：
  - 读 `NACOS_SERVER_ADDR` / `NACOS_NAMESPACE` / `NACOS_USERNAME` / `NACOS_PASSWORD`
  - `get_config("autobots-ai.yml", "AUTOBOTS_GROUP")`
  - 解析扁平 YAML，**仅当 `os.environ` 中尚不存在该 key 时**写入（不覆盖已有进程环境变量）
- 修改 `config/config.py`：在 `load_dotenv` 前后接入 loader；`CONF_ENV=local` 跳过 Nacos。
- 可选：`add_config_watcher` 做热更新（多数常量在 import 时已固化，**一期可不做热更**，仅启动拉取）。

### 5.4 `.env_*` 文件处理

| 文件 | 迁移后 |
|------|--------|
| `.env_poc` / `.env_pre` / `.env_prod` | 内容上传到对应 namespace；仓库改为 `.env_*.example`（仅 key + 假值）或从 Git 删除并 gitignore |
| `.env_local.example` | 保留，本地开发继续用 |
| 已提交的真实密钥 | 迁移后建议在 Nacos 轮换，并清理 Git 历史（另开安全任务） |

---

## 6. 部署侧变更

各环境 Deployment / 启动脚本只需注入：

```bash
NACOS_SERVER_ADDR=...
NACOS_NAMESPACE=...          # poc | pre | prod-ns-id
NACOS_USERNAME=...
NACOS_PASSWORD=...
SPRING_PROFILES_ACTIVE=poc   # Java：仍可用于本地/兼容；迁完后可弱化，仅留 local
CONF_ENV=poc                 # Python
```

可选：

- 去掉 ConfigMap 里与 Nacos 重复的业务 YAML（`k8s/configmap.yaml` 中的 gateway/database 等），避免双源。
- `start-local-backend.sh`：local 不设 Nacos；若要联调 POC Nacos，文档说明导出上述变量。

---

## 7. 实施分期（建议在本分支按 commit 推进）

### Phase 0 — 约定与模板（本方案）

- [x] 创建分支 `feat/nacos-config-center`
- [x] 落地 `docs/nacos-config-migration.md` + `configs/nacos/{poc,pre,prod}/*`
- [ ] 运维在控制台创建 namespace / 账号权限 / Group，并按 `configs/nacos/README.md` 上传 12 条配置

### Phase 1 — Java 接入

- [x] 父 POM BOM + 三服务 nacos-config 依赖
- [x] `spring.config.import` + env 驱动连接
- [x] 从 `application.yml` 删除 poc/pre/prod 段（内容已导出到 `configs/nacos/`）
- [ ] 将配置发布到 Nacos 后做 POC 启动验证
- [ ] `mvn -pl autobots-gateway,autobots-user,autobots-external -am test`

### Phase 2 — Python 接入

- [x] `nacos_loader` + `config.py` 改造
- [x] `autobots-ai.yml` 导出到 `configs/nacos/{env}/`
- [ ] 发布到 Nacos 后对比现网启动冒烟
- [x] `test/test_nacos_loader.py`

### Phase 3 — 清理与文档

- [x] 仓库 `application.yml` 仅保留通用 + local
- [ ] `.env_poc|pre|prod` 改为 example 或移除（保留作回退，待 Nacos 稳定后再删）
- [ ] 更新 `CLAUDE.md` / `README` / `docs/local-dev.md` / `PROJECT_NAVIGATOR.md`
- [ ] Pre / Prod 配置发布与灰度

### Phase 4 — 加固

- [ ] 生产改为非 optional import（缺配置失败）
- [ ] 敏感默认值从代码清除
- [ ] 密钥轮换；评估 Git 历史清理
- [ ] （可选）热更新范围清单

---

## 8. 验证清单

**Java（每个服务）**

1. 未配 Nacos + `local`：与现网本地行为一致。
2. 配 POC Nacos：无本地 poc 段时，Redis/MySQL/路由与现 poc 一致。
3. 故意错误密码：启动失败信息清晰。
4. 改 Nacos 非连接项：确认是否刷新（记录实际行为）。

**Python**

1. `CONF_ENV=local`：只读本地文件。
2. `CONF_ENV=poc` + Nacos：关键变量与现 `.env_poc` 一致。
3. 进程已有 `DATABASE_URL` 时不被 Nacos 覆盖（若采用 setdefault 策略）。

**回归**

- 登录 / 网关鉴权 / 知识库调用 / 线索推送 / AI 对话各抽一条主路径。

---

## 9. 风险与注意

| 风险 | 缓解 |
|------|------|
| 启动强依赖 Nacos | 本地 optional；生产就绪后再改强制；评估 Nacos HA |
| 双源配置（文件 + Nacos） | Phase 3 删掉环境段；文档写明优先级 |
| 密钥已在 Git / 聊天中泄露 | Nacos 存新值并轮换；聊天中的密码勿再写入仓库 |
| YAML 与本地 .env 双格式 | Nacos 统一 YAML；本地开发仍可用 `.env_*` 回退 |
| Gateway WebFlux + Nacos | 仅 config starter，注意排除与 bootstrap 冲突；以 POC 实测为准 |
| 前端未迁 | 构建期 env 继续走 CI；若需运行时配置另开需求 |

---

## 10. 开发约定（本分支）

- 提交信息：Conventional Commits，例如 `feat(config): integrate nacos config for gateway`。
- 文档与模板中只写变量名，不写真实密码。
- 合并前至少完成 Phase 1 + Phase 2 在 POC 的冒烟；Phase 3 可同 PR 或 follow-up PR。
