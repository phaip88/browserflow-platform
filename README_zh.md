# BrowserFlow 自动化编排平台

[![CI](https://github.com/phaip88/browserflow-platform/actions/workflows/ci.yml/badge.svg)](https://github.com/phaip88/browserflow-platform/actions/workflows/ci.yml)
[![Docker Publish](https://github.com/phaip88/browserflow-platform/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/phaip88/browserflow-platform/actions/workflows/docker-publish.yml)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

[English](README.md) | [中文说明](README_zh.md)

**BrowserFlow Platform** 是一套面向生产环境设计的自托管、模块化浏览器自动化与流程编排平台。核心采用「控制平面与执行引擎进程级物理隔离」架构，从根源上杜绝了因复杂网页爬取、爬虫崩溃或浏览器内存泄漏导致 Web 管理面板不可用的问题。

---

## 核心技术特性

- **进程级运行隔离**：Playwright Chromium 浏览器实例仅在独立 Worker 进程（`apps/browser_worker`）内拉起，与 FastAPI 控制面板（`apps/api`）彻底解耦。
- **确定性 DAG 编译器**：对可视化拖拽的工作流进行严格的静态语法检验、成环检测与拓扑排序，生成不可变的确定性执行计划。
- **租约锁与心跳防死锁机制**：基于数据库行级排他锁的任务租约机制（`attempt_id` 与 `lease_token`），配合 Worker 定时心跳上报与陈旧节点自动回收（Stale-worker guard），实现分布式多节点任务消费与故障自愈。
- **企业级安全防护体系**：
  - **SSRF 私网请求防御**：内置 `BrowserRequestNetworkPolicy`，强行拦截对局域网内网段（RFC 1918、127.0.0.1、云平台元数据地址）的非法访问。
  - **目录穿越防护**：`SafePath` 严格限制文件与产物读写范围，杜绝越权读写。
  - **凭据硬件级隔离与脱敏**：敏感数据采用 AES-256-GCM 主密钥加密存储，在任务执行日志与前端展示时动态执行掩码脱敏。
  - **身份鉴权**：Argon2id 密码哈希、HttpOnly 会话 Cookie、防 CSRF 跨站请求伪造机制。
- **持久化定时任务调度**：支持 Crontab 表达式与单次触发任务，具备 IANA 时区解析、漏触发补偿（Misfire recovery）与触发去重能力。
- **插件化节点扩展（Node SDK）**：规范化的节点生命周期抽象，已内置浏览器自动化、控制流、数据处理与接口集成四大核心节点包。
- **原生双语支持**：内置中文（简体）与英文无缝切换，偏好配置持久化存储。

---

## 总体架构设计

```
                          ┌───────────────────────────┐
                          │   Web 前端界面 (@web)     │
                          │  Vite + React + React Flow │
                          └─────────────┬─────────────┘
                                        │ HTTP / WS
                                        ▼
                          ┌───────────────────────────┐
                          │    FastAPI 控制平面服务   │
                          │        (apps/api)         │
                          └─────────────┬─────────────┘
                                        │
           ┌────────────────────────────┼───────────────────────────┐
           │                            │                           │
           ▼                            ▼                           ▼
┌───────────────────────┐   ┌───────────────────────┐   ┌───────────────────────┐
│   PostgreSQL 数据库   │   │  浏览器工作节点池     │   │   持久化定时调度器    │
│  (系统唯一数据事实源) │   │ (Playwright Chromium) │   │   (apps/scheduler)    │
└───────────────────────┘   └───────────────────────┘   └───────────────────────┘
```

| 模块名称 | 采用技术 | 职责定位 |
| :--- | :--- | :--- |
| `apps/web` | Vite, React 18, React Flow, Tailwind CSS | 可视化流程图编排、执行回放诊断、凭据管理、双语界面 |
| `apps/api` | Python 3.12+, FastAPI, SQLAlchemy (异步) | 核心 RESTful API、流程编译器、用户认证与审计日志 |
| `apps/browser_worker` | Playwright Python, Chromium | 独立的无头/有头浏览器执行引擎，任务租约消费 |
| `apps/scheduler` | Python 3.12+, Croner | 定时 Cron 表达式计算与任务派发 |
| `packages/domain` | Python (DDD 纯领域模型) | 领域实体、执行状态机、枚举与核心业务规则 |
| `packages/flow_compiler`| Python (DAG 编译管线) | 流程校验、DAG 拓扑排序、确定性执行计划生成 |
| `packages/node_sdk` | Python (SDK 接口契约) | 统一的节点开发接口、输入输出校验与上下文传递 |

---

## 预置节点功能清单 (Release 1)

### 1. 浏览器自动化节点包 (`node_pack_browser`)
- `page.goto`：跳转目标网址，自动执行网络安全策略过滤。
- `page.click`：智能等待并点击 DOM 元素。
- `page.fill`：表单输入框文本录入。
- `page.screenshot`：整页或特定元素截图，保存为产物归档。
- `page.evaluate`：在目标网页上下文中执行自定义 JavaScript 代码。
- `page.wait_for_selector`：显式等待选择器出现或消失。
- `page.scrape_text`：提取选择器对应文本或正则捕获内容。
- `page.press` / `page.hover`：键盘按键触发与鼠标悬停交互。
- `page.reload`：重新加载当前页面。

### 2. 控制流逻辑节点包 (`node_pack_control`)
- `control.branch`：基于执行上下文变量的多条件分支跳转。
- `control.loop`：数组列表遍历与计次循环。
- `control.delay`：流程等待延时（支持优雅中断与取消）。

### 3. 数据转换节点包 (`node_pack_data`)
- `data.transform`：数据结构映射与字段提取。
- `data.extract_regex`：基于正则表达式的数据抽取。
- `data.json_parse`：JSON 字符串反序列化与字段校验。

### 4. 接口集成节点包 (`node_pack_integration`)
- `integration.http_request`：对外发起 HTTP/HTTPS 请求（支持请求头、认证与负载自定义）。
- `integration.webhook`：接收与分发 Webhook 触发信号。

---

## 生产部署指南（Docker Compose）

### 1. 环境准备
- 安装 Docker Engine 24+ 与 Docker Compose v2+
- 支持 Linux、macOS 或 Windows（WSL2）

### 2. 获取代码与初始化安全密钥
```bash
git clone https://github.com/phaip88/browserflow-platform.git
cd browserflow-platform

# 生成生产级随机安全密钥文件
mkdir -p secrets
python3 -c "import os, secrets; \
  open('secrets/postgres_password', 'w').write(secrets.token_hex(16)); \
  open('secrets/master.key', 'wb').write(os.urandom(32)); \
  open('secrets/session.secret', 'wb').write(os.urandom(48))"
```

### 3. 一键启动全部服务
```bash
docker compose -f docker-compose.production.yml up -d --build
```

### 4. 初始化数据库表结构与管理员账号
```bash
# 执行数据库版本迁移
docker compose -f docker-compose.production.yml exec api alembic upgrade head

# 创建首个系统管理员用户
docker compose -f docker-compose.production.yml exec api browserflow admin create
```

### 5. 访问控制面板
- **Web 前端管理面板**：[http://localhost:8080](http://localhost:8080)
- **API 接口交互文档**：[http://localhost:8000/docs](http://localhost:8000/docs)
- **健康检查探针**：`curl http://localhost:8000/health/live`

---

## 本地开发指南

### 1. 安装后端环境
```bash
# 创建并激活 Python 虚拟环境 (推荐 Python 3.12+)
python3 -m venv .venv
source .venv/bin/activate  # Windows 终端: .venv\Scripts\Activate.ps1
pip install -U pip
pip install -e ".[dev]"

# 安装 Playwright Chromium 驱动
python -m playwright install chromium

# 安装前端依赖 (推荐 pnpm 9+)
pnpm install
```

### 2. 启动本地数据库
```bash
docker run -d --name browserflow-pg -p 5432:5432 \
  -e POSTGRES_USER=browserflow \
  -e POSTGRES_PASSWORD=browserflow_dev \
  -e POSTGRES_DB=browserflow \
  postgres:16-alpine
```

### 3. 启动开发服务
```bash
# 终端 1：启动 API 后端服务
uvicorn browserflow.api.main:app --host 127.0.0.1 --port 8000 --reload

# 终端 2：启动浏览器执行 Worker
python -m browserflow.browser_worker

# 终端 3：启动定时调度守护进程
python -m browserflow.scheduler

# 终端 4：启动 Web 前端服务
pnpm --filter @browserflow/web dev
```

### 4. 执行自动化测试
```bash
# 运行后端全部测试套件（单元测试、节点契约、集成测试、安全性、稳定性）
pytest tests -q

# 运行前端测试与类型校验
pnpm --filter @browserflow/web test
pnpm --filter @browserflow/web typecheck
```

---

## 容器镜像发布机制

所有组件通过 GitHub Actions 自动化编译，并在推送到 `main` 分支时自动推送到 GitHub Packages (GHCR)：

- **API 与调度镜像**：`ghcr.io/phaip88/browserflow-platform-api:latest`
- **浏览器执行 Worker 镜像**：`ghcr.io/phaip88/browserflow-platform-worker:latest`
- **Web 前端镜像**：`ghcr.io/phaip88/browserflow-platform-web:latest`

---

## 安全合规声明

- **严禁明文回退**：严禁在任何配置文件中直接写入真实敏感信息，所有生产环境凭据必须通过环境变量或 Docker Secrets 注入。
- **漏洞反馈**：安全漏洞与通报流程请参阅 [SECURITY.md](SECURITY.md)。

---

## 开源协议

本项目基于 Apache 2.0 协议开源。详情请参阅 [LICENSE](LICENSE) 文件。
