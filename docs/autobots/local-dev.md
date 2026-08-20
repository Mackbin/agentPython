# 本地启动 & 避坑指南

面向新机器 / 新同事：跑通 **autobots-frontend + autobots-ai + POC 后端** 的最短路径，附高频踩坑排查。

**默认拓扑（推荐）**：前端 `pnpm dev` 混合代理
- `/api/user`、`/api/external`、`/api/files` → POC（`https://autobots-poc.brgroup.com`）
- `/api/agents`、`/api/qa`、`/api/lead` → 本地 `autobots-ai`（`http://localhost:8085`）
- Java 三件套（gateway/user/external）**不需要在本机启动**，POC 已提供

如果确实要跑本地 Java 后端，见文末《方式二：本地 Java 全量后端》。

---

## 0. 前置依赖

必装：

| 组件 | 版本 | 用途 |
| --- | --- | --- |
| Node.js | 22.x | 前端 |
| pnpm | 9+ | 前端包管理 |
| Python | 3.11+ | autobots-ai |
| 公司 VPN / 内网 | — | POC MySQL、POC Redis、内网 RAG 模型 |

macOS 一次性检查：

```bash
node -v && pnpm -v && python3 --version
```

不装 Redis / MySQL：本地不需要。都走 POC。

---

## 1. 第一次拉代码后的初始化

### 1.1 前端

```bash
cd autobots-frontend
pnpm i

# 建本地混合代理配置（.env.local 已在 .gitignore）
cp .env.local.example .env.local
```

`.env.local` 默认值已是「混合代理」，无需改动：主接口指向 `localhost:8085`，用户/外部接口指向 POC。

### 1.2 autobots-ai

```bash
cd autobots-ai

# 建 venv（一次即可）
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 建本地 env 配置（.env_local 已在根 .gitignore）
cp .env_local.example .env_local
```

`.env_local` 内的数据库/Redis/MinIO/AgenticPlatform **全部指向 POC 对外域名**，不要改成 `-pre` 或内部 DNS（后面《避坑》细说）。

> `.env_local` 缺失时会怎样？`autobots-ai/config/config.py::resolve_env_file` 会回退到 `.env_pre`，而 `.env_pre` 里全是 K8s 内部 DNS（`http://autobots-user`、`http://autobots-gateway`），Mac 本地无法解析，前端会看到「AP permission manifest failed」502 类错误——这就是本仓库最常见的踩坑点。

---

## 2. 日常启动

```bash
cd autobots-frontend
pnpm dev
```

按提示选：
- **项目**：`App Manager`（勾 SDK 通常不需要——它只是把 `packages/web` 预构建，改 SDK 源码时 dev alias 已自动指向源码）
- **环境**：`dev`

脚本会：
1. 加载 `.env` + `.env.local`
2. 检测 `VITE_SERVER_DOMAIN` 是否指向 `localhost:8085`
3. 若指向，自动 `python start.py`（`CONF_ENV=local`）拉起 autobots-ai
4. 探活 `http://localhost:8085/health` 到就绪后启动 Vite

**首次是否重建 SDK**：只有改过 `packages/web` 源码时才选 `Yes`。默认可以直接 `No`（`app-manager` 在 dev 下走 SDK 源码 alias）。

启动后：
- 前端：<http://localhost:5174>
- AI：<http://localhost:8085/docs> （FastAPI Swagger）

---

## 3. 快速自检 4 条命令

一旦「访问后端有问题」，先按顺序敲：

```bash
# 1. AI 进程活着？
curl -s http://localhost:8085/health

# 2. AI 到底加载了哪份 env？（关键）
tail -n 50 autobots-ai/logs/autobots-ai.log | grep env_file

# 3. POC 通不通？
curl -s -o /dev/null -w "%{http_code}\n" https://autobots-poc.brgroup.com/api/auth/health

# 4. 前端代理请求转到了哪？
#    看 Vite 终端里的 “请求URL:” / “用户请求URL:” / “智能体请求URL:” 日志
```

`env_file` 必须是 `.env_local`。是 `.env_pre` 就代表 `.env_local` 没建 / 没被读到。

---

## 4. 高频踩坑

### 坑 1：前端看到 502 / 系统繁忙 / AP permission manifest failed
- **现象**：AI 日志出现 `AP permission manifest failed; using fallback manifest, status_code: 502`
- **根因**：`.env_local` 不存在 → 回退加载 `.env_pre` → `AGENTIC_PLATFORM_BASE_URL=https://autobots-ap-pre.brgroup.com`。你用 POC 账号登录拿到的 token 打 PRE 平台会拒
- **修复**：`cp autobots-ai/.env_local.example autobots-ai/.env_local`，重启 `pnpm dev`

### 坑 2：AI 起不来，日志找不到 host / DNS
- **现象**：`Cannot resolve autobots-user` / `autobots-gateway` / `rd-11-autobots-pre.brapp.com`
- **根因**：同坑 1，加载了 `.env_pre` 里的内部 K8s DNS
- **修复**：同坑 1

### 坑 3：数据库连不上
- **现象**：`OperationalError: (2003, "Can't connect to MySQL server on '10.100.123.109'")`
- **根因**：VPN 没连
- **修复**：连公司 VPN 或走 POC 跳板机；确认 `nc -zv 10.100.123.109 3306` 通

### 坑 4：Redis 连不上（登录后 session 丢）
- **现象**：AI 日志 `redis.exceptions.ConnectionError`
- **根因**：`REDIS_URL=redis://localhost:6379` 但本地没起 Redis
- **修复**：改成 POC `redis://rd-16-autobots-poc.brapp.com:6400`，密码 `16_rd-autobots-poc_awspwd`

### 坑 5：出网请求诡异 SSL / 挂起
- **现象**：只有本地会挂，POC/PRE 容器里正常
- **根因**：`AUTH_HTTP_TRUST_ENV=true` 让 `requests` 读了 mac 的 `http_proxy`/`https_proxy`
- **修复**：`.env_local` 保持 `AUTH_HTTP_TRUST_ENV=false`（模板里已默认）

### 坑 6：SDK 改了没生效
- **现象**：修改 `packages/web/src/**` 无反应
- **原因**：app-manager 在 dev 下走 SDK **源码 alias**，但 `VITE_APP_MANAGER_USE_SDK_DIST=true` 或存在 `packages/web/dist/vue/autobots-vue.es.js` 时会走 dist
- **修复**：删除 `packages/web/dist` 或 unset 变量后重启；只有需要把 Uা MD 打包版本装载到 iframe 时才必须选 SDK build

### 坑 7：Arco 提示预构建产物缺失
- 单独一次：`pnpm build:arco`
- 平时业务开发用不到，`packages/arco-design-vue/**/dist|es|lib` 已随代码提交

### 坑 8：前端启动崩溃提示 Windows 路径
- **现象**：`Cannot find module 'E:\产品项目\...\vite.js'`
- **根因**：`node_modules` 是别的机器上生成、含硬编码路径的软链
- **修复**：`rm -rf node_modules && pnpm i`

### 坑 9：8085 已被占用
- **现象**：`pnpm dev` 打印 `autobots-ai 已在 http://localhost:8085 运行`，但改了 `.env_local` 没生效
- **原因**：健康检查过后脚本会**复用**已存在的进程，不会重启
- **修复**：`lsof -ti:8085 | xargs kill`，再 `pnpm dev`

### 坑 10：POC 登录不上
- **现象**：captcha 校验 400 / 手机号发送次数超限
- **原因**：POC 数据库风控计数；同一号码 10 分钟 10 次上限
- **修复**：换测试号 / 等 10 分钟；或临时到 POC 的 Redis 里删 `login:failure:{phone}` key

### 坑 11：本地 Java 全量后端报 `Invalid token`
- **现象**：本地启动 gateway/user/external 后，前端或 curl 访问 `/api/external/**`、`/api/permission/**` 返回 `{"code":401,"msg":"Invalid token"}` 或 `{"detail":"...Invalid token..."}`。
- **根因**：本地服务混用了 POC/PRE。`user` 登录拿到一个环境的 token，但 `gateway`、`external` 或 `autobots-ai` 把同一个 token 发到另一个环境校验/转发。POC token 打 PRE AP、PRE token 打 POC AP 都会被拒。
- **判定 1：gateway 是否打错环境**
  ```bash
  tail -n 120 logs/autobots-gateway.log | grep 'validateUrl'
  ```
  本地全量联调应看到 `https://autobots-ap-poc.brgroup.com/api/auth/gateway/validate`。如果是 `autobots-ap-pre`，检查 `autobots-gateway/src/main/java/com/br/autobots/gateway/config/LocalExternalApiOverrideConfig.java` 和 `autobots-gateway/src/main/resources/application.yml` 的 local 段。
- **判定 2：gateway 已通过，但 external 仍返回 401**
  ```bash
  tail -n 160 logs/autobots-gateway.log | grep 'token.*HTTP'
  tail -n 160 logs/autobots-external.log | grep 'https://autobots-ap-'
  ```
  如果 gateway 日志里 token 校验 HTTP 状态码是 200，但 external 日志仍请求 `https://autobots-ap-pre.brgroup.com/api/v1/store/knowledge...`，说明 `autobots-external` local 段还在转发到 PRE。修复 `autobots-external/src/main/resources/application.yml` 的 local `external-interface.url` 为 `https://autobots-ap-poc.brgroup.com/api`。
- **判定 3：AI 权限接口 401**
  ```bash
  tail -n 80 logs/autobots-ai-console.log | grep permission
  grep AGENTIC_PLATFORM_BASE_URL autobots-ai/.env_local
  ```
  `AGENTIC_PLATFORM_BASE_URL` 必须是 `https://autobots-ap-poc.brgroup.com`。
- **修复后必须重建重启**
  ```bash
  mvn -pl autobots-gateway,autobots-user,autobots-external -am package -DskipTests
  # 重启 screen 里的 autobots-gateway-local / autobots-user-local / autobots-external-local
  ```
  重启后清掉浏览器 localhost 的旧 Cookie，重新登录。`token` 是 httpOnly Cookie，建议在 Chrome DevTools 的 Application -> Cookies 里删除 `localhost` 下的 `token`，或直接 Clear site data。
- **成功标志**
  ```bash
  curl -s -i http://localhost:8088/user/health
  curl -s -i http://localhost:8088/external/health
  ```
  登录后 `/api/external/knowledgebase/stats/global` 应返回 `code:0`，external 日志中的上游应是 `https://autobots-ap-poc.brgroup.com/api/v1/store/knowledge/stats/global`。

### 坑 12：`/api/permission/effective-manifest` 报 502 Bad Gateway
- **现象**：前端启动后路由守卫请求 `/api/permission/effective-manifest`，浏览器控制台报 `502 (Bad Gateway)`，Vue Router 启动失败。
- **根因**：本地 gateway / AI 服务仍在运行，但 `autobots-ai` 需要访问 AP 的 `/api/auth/me` 校验当前 Cookie。若本机到 `autobots-ap-poc.brgroup.com:443` 或 `autobots-ap-pre.brgroup.com:443` 连接超时，AI 会返回 502。
- **判定 1：本地服务是否挂了**
  ```bash
  lsof -nP -iTCP:5174 -iTCP:8088 -iTCP:8085 -sTCP:LISTEN
  curl -s -i --max-time 3 http://localhost:8085/health
  ```
  如果 5174、8088、8085 都在监听，且 AI health 返回 200，说明不是本地服务挂了。
- **判定 2：是否 AP 网络不可达**
  ```bash
  tail -n 120 logs/autobots-ai-console.log | grep -E 'permission/effective-manifest|auth/me|认证服务不可用|502'
  nc -vz -G 5 autobots-ap-poc.brgroup.com 443
  nc -vz -G 5 autobots-ap-pre.brgroup.com 443
  ```
  如果日志出现 `认证服务不可用`、`Connection to autobots-ap-poc.brgroup.com timed out`，且 `nc` 连接 443 超时，优先检查 VPN / 内网代理 / 公司网络。
- **恢复后验证**
  ```bash
  curl -s -i --connect-timeout 5 --max-time 8 https://autobots-ap-poc.brgroup.com/api/auth/me
  ```
  能快速返回 HTTP 响应后，刷新前端或重新登录。重启本地服务通常不能解决这类 502，除非网络恢复后进程状态异常。

---

## 5. 常用命令速查

```bash
# 停掉本地 AI
lsof -ti:8085 | xargs kill

# 手动只跑 AI（不启前端）
cd autobots-frontend && pnpm ai:local

# 仅刷新 SDK 到 app-manager 打包版
pnpm -F @autobots/sdk build

# 前端 lint（提交前）
pnpm -F app-manager lint

# 看 AI 加载的 env / 报错
tail -F autobots-ai/logs/autobots-ai.log

# 环境切回 POC 静态前端（不启 AI）
pnpm dev  # 选环境为 poc
```

---

## 6. 方式二：本地 Java 全量后端（不推荐日常使用）

如果需要在本地跑 gateway + user + external：

```bash
mvn clean package -DskipTests

# 三个终端分别启动
java -jar autobots-user/target/autobots-user.jar     --spring.profiles.active=local
java -jar autobots-external/target/autobots-external.jar --spring.profiles.active=local
java -jar autobots-gateway/target/autobots-gateway.jar   --spring.profiles.active=local
```

前端 `.env.local` 改成走本地 gateway：

```env
VITE_SERVER_DOMAIN=http://localhost:8088
VITE_SERVER_API_BASE=http://localhost:8088
VITE_SERVER_IS_LOCAL=true
# 删掉 VITE_USER_SERVER_DOMAIN / VITE_EXTERNAL_SERVER_DOMAIN，让所有 /api/* 都走 gateway
```

注意：本地 gateway 需要连本地 / POC 的 MySQL + Redis，端口、profile 里配。用户表要自己 `INSERT` 一条（`autobots-user/src/main/resources/application.yml` 的 `local` profile）。

---

## 7. 出问题时给别人看什么

排查请一次贴齐：

1. `curl http://localhost:8085/health` 输出
2. `tail -50 autobots-ai/logs/autobots-ai.log`
3. Vite 终端里最近的 `请求URL:` / `用户请求URL:` 行
4. 浏览器 Network 面板：失败接口的 `Request URL` + `Status` + `Response`

有这四样，本仓库 95% 的「本地跑不起来」都能一分钟定位。
