# Interview Guide - AI 智能面试官平台

## 项目一句话概述

基于 Spring Boot 4.1 + Java 21 + Spring AI 2.0 + React 18 的全栈 AI 面试平台。核心功能: 简历解析与分析、文字/语音模拟面试、RAG 知识库问答、面试日程安排。

---

## 技术栈速览

| 层级 | 技术 | 版本 |
|------|------|------|
| 后端框架 | Spring Boot | 4.1.0 |
| 语言 | Java | 21 |
| AI 集成 | Spring AI 2.0 (OpenAI 兼容模式) | 2.0.0 |
| 构建 | Gradle (wrapper) | 8.14 |
| 数据库 | PostgreSQL + pgvector | 16 |
| 缓存/消息 | Redis + Redisson | 7 / 4.0.0 |
| 对象存储 | MinIO / RustFS (S3 兼容) | - |
| 文档解析 | Apache Tika | 2.9.2 |
| PDF 导出 | iText 8 | 8.0.5 |
| 语音 | DashScope SDK (Qwen3 ASR/TTS) | 2.22.7 |
| 前端框架 | React + TypeScript | 18.3 / 5.6 |
| 前端构建 | Vite | 5.4 |
| 前端包管理 | pnpm | 10+ |
| 样式 | TailwindCSS | 4.1 |

---

## 项目结构

```
/workspace/interview-guide/
├── AGENTS.md                    # AI Agent 共享规则（长期规范）
├── CLAUDE.md                    # Claude Code 专属入口（引用 AGENTS.md）
├── .env.example / .env          # 环境变量（API Key、数据库密码等）
├── settings.gradle              # Gradle 项目配置
├── docker-compose.yml           # 生产环境全栈部署
├── docker-compose.dev.yml       # 本地开发依赖（PG + Redis + RustFS）
│
├── app/                         # Spring Boot 后端
│   ├── build.gradle             # 依赖管理
│   ├── Dockerfile               # 后端容器化
│   └── src/main/
│       ├── java/interview/guide/
│       │   ├── App.java         # @SpringBootApplication 主启动类
│       │   ├── common/          # 通用能力：AI调用、限流、异步、配置、异常
│       │   ├── infrastructure/  # 基础设施：文件存储、导出、Redis、MapStruct
│       │   └── modules/         # 业务模块(每个自含 MVC)
│       │       ├── resume/      # 简历管理
│       │       ├── interview/   # 文字面试
│       │       ├── voiceinterview/ # 语音面试 (WebSocket)
│       │       ├── knowledgebase/  # RAG 知识库
│       │       ├── schedule/    # 面试安排 (日历)
│       │       ├── auth/        # 认证模块
│       │       └── aiconfig/    # AI 模型配置管理
│       └── resources/
│           ├── application.yml  # 主配置
│           ├── prompts/         # StringTemplate Prompt 模板
│           └── skills/          # 面试 Skill 定义
│
├── frontend/                    # React 前端
│   ├── package.json             # pnpm 依赖与脚本
│   ├── vite.config.ts           # Vite 构建配置
│   ├── nginx.conf               # 生产环境 Nginx 配置
│   ├── Dockerfile               # 前端容器化
│   └── src/
│       ├── api/                 # Axios API 客户端
│       ├── components/          # 可复用组件
│       ├── pages/               # 页面组件
│       ├── types/               # TypeScript 类型定义
│       └── constants/routes.ts  # 路由常量
│
├── docker/                      # Docker 相关文件
│   └── postgres/init.sql        # PG 初始化脚本 (pgvector 扩展)
│
└── .claude/rules/               # Claude Code 目录规则
    ├── backend.md
    ├── frontend.md
    └── ai-and-async.md
```

---

## 架构原则

### 后端分层
```
Controller → Service → Repository (JPA)
```
- Controller: 路由 + 参数校验 + 委托 Service
- Service: 业务编排，`@Transactional` 仅放此层，范围最小
- Repository: 继承 `JpaRepository`，自定义查询用方法名或 `@Query`
- 响应统一使用 `Result<T>`，禁止直接返回 Entity

### 关键设计决策
1. **LLM 调用统一入口**: `LlmProviderRegistry.getChatClientOrDefault(provider)` - 所有聊天模型调用统一走此方法
2. **结构化输出**: `StructuredOutputInvoker` - 统一处理 JSON Schema 输出与重试
3. **异步任务**: Redis Stream + `AbstractStreamProducer`/`AbstractStreamConsumer` 模板
4. **限流**: `@RateLimit` 注解，可复用
5. **Prompt 管理**: `resources/prompts/*.st` StringTemplate 模板，不硬编码
6. **对象映射**: MapStruct (Entity ↔ DTO)
7. **依赖注入**: 构造器注入 + `@RequiredArgsConstructor`

### 基础设施
- **敏感信息**: 只放 `.env`，不提交 Git
- **配置**: `application.yml` + `@ConfigurationProperties`，不在 Service 里散落 `@Value`
- **异常处理**: `BusinessException(ErrorCode.XXX, "描述")`，全局处理器返回 HTTP 200 + `Result.error()`
- **事务**: LLM、S3、外部 HTTP 调用不放事务内
- **虚拟线程**: Java 21 虚拟线程启用(`spring.threads.virtual.enabled: true`)

---

## 环境要求与启动

### 前置依赖
- Java 21 (Gradle toolchain 可自动下载)
- Node.js 20+ + pnpm
- Docker + Docker Compose v2

### 快速启动

```bash
# 1. 启动开发依赖 (PostgreSQL + Redis + RustFS)
docker compose -f docker-compose.dev.yml up -d

# 2. 创建 RustFS bucket (首次运行，浏览器访问 http://localhost:9001)
#    使用 rustfsadmin / rustfsadmin 登录，创建名为 interview-guide 的 bucket

# 3. 配置 .env (从 .env.example 复制并填入 AI_BAILIAN_API_KEY)
cp .env.example .env

# 4. 启动后端 (端口 8080)
./gradlew :app:bootRun

# 5. 启动前端 (端口 5173)
cd frontend && pnpm install && pnpm run dev
```

### 全栈 Docker 部署

```bash
docker compose up -d
# 后端: 8080 / 前端: 80
```

### 常用命令

| 场景 | 命令 |
|------|------|
| 编译后端 | `./gradlew :app:compileJava` |
| 运行测试 | `./gradlew :app:test --no-daemon` |
| 启动后端 | `./gradlew :app:bootRun` |
| 启动前端 | `cd frontend && pnpm run dev` |
| 构建前端 | `cd frontend && pnpm run build` |
| 启动依赖 | `docker compose -f docker-compose.dev.yml up -d` |
| 查看 Swagger | http://localhost:8080/swagger-ui.html |

---

## 业务模块详解

### 1. 简历管理 (`modules/resume/`)
- 支持 PDF、DOCX、DOC、TXT 格式解析 (Apache Tika)
- Redis Stream 异步分析流程
- 失败自动重试 (最多3次) + 内容哈希重复检测
- PDF 报告导出 (iText 8)

### 2. 模拟面试 (`modules/interview/`)
- Skill 驱动出题 (10+ 面试方向: Java后端、阿里/字节/腾讯专项、前端、Python、算法等)
- 历史题目去重
- 智能追问 (可配置轮数)
- 统一评估架构 (分批评估 → 结构化输出 → 二次汇总 → 降级兜底)
- PDF 报告异步导出

### 3. 语音面试 (`modules/voiceinterview/`)
- WebSocket 实时双向通信
- Qwen3 ASR (语音识别) + TTS (语音合成)
- VAD 静音检测 + 多段 STT 合并
- 流式 TTS 推送 (chunked audio)
- 面试官 AI 语音回复 (限长 120 字)

### 4. RAG 知识库 (`modules/knowledgebase/`)
- 文档向量化 (pgvector, 维度 1024, COSINE 距离)
- Query 改写 + 自适应检索 (short/medium/long query)
- 对话记忆 (最近 120 条消息)

### 5. 面试安排 (`modules/schedule/`)
- 规则 + AI 双引擎邀请解析 (支持飞书/腾讯会议/Zoom)
- 日历视图 (日/周/月) + 拖拽调整
- 面试房间管理

### 6. AI 配置管理 (`modules/aiconfig/`)
- 运行时管理多个 LLM Provider (DashScope、Kimi、DeepSeek、GLM 等)
- API Key 加密存储
- 配置落盘 (`~/.interview-guide/llm-providers.yml`)

---

## 配置要点

### .env 必填项
```ini
AI_BAILIAN_API_KEY=sk-xxx           # 阿里云百炼 API Key (必填)
APP_AI_CONFIG_ENCRYPTION_KEY=xxx    # Provider Key 加密密钥 (必填，长随机串)
```

### AI Provider 配置
默认使用 DashScope (阿里云百炼)，模型 qwen3.5-flash。支持添加:
- Kimi (Moonshot)
- DeepSeek
- GLM (智谱)
- LM Studio (本地)
- 自定义 OpenAI 兼容 Provider

### 存储配置
开发环境使用 RustFS (S3 兼容，docker-compose.dev.yml)，生产环境使用 MinIO (docker-compose.yml)。
均通过 S3 SDK 访问，切换只需改环境变量。

---

## 关键文件路径速查

| 文件 | 路径 |
|------|------|
| 主配置 | `app/src/main/resources/application.yml` |
| 启动类 | `app/src/main/java/interview/guide/App.java` |
| AI Provider 配置类 | `app/src/main/java/interview/guide/common/config/LlmProviderProperties.java` |
| 存储配置类 | `app/src/main/java/interview/guide/common/config/StorageConfigProperties.java` |
| 全局异常处理 | `app/src/main/java/interview/guide/common/exception/GlobalExceptionHandler.java` |
| 统一响应体 | `app/src/main/java/interview/guide/common/result/Result.java` |
| Redis Stream 模板 | `app/src/main/java/interview/guide/common/redis/AbstractStreamProducer.java` |
| 限流注解 | `app/src/main/java/interview/guide/common/ratelimit/RateLimit.java` |
| 前端路由 | `frontend/src/constants/routes.ts` |
| 前端 API 客户端 | `frontend/src/api/` |
| 前端类型定义 | `frontend/src/types/` |

---

## 禁止事项 (来自 AGENTS.md)

- 不 `throw new RuntimeException()` — 用 `BusinessException`
- 不直接返回 Entity — 用 `Result<T>` 包装
- 不在 Service 散落 `@Value` — 用 `@ConfigurationProperties`
- 不在事务内调 LLM / S3 / 外部 HTTP
- 不同类内调 `@Transactional` 方法
- 不 `catch (Exception e) {}` 静默忽略
- 不循环调 DB — 批量查询/写入
- 不硬编码密钥/Token/密码
- 不使用 `Executors.newXxxThreadPool()` — 显式配置 `ThreadPoolExecutor`
