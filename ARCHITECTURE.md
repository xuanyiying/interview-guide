# 架构设计文档

## 概述

InterviewGuide 采用现代化的微服务架构设计，基于Spring Boot生态系统，集成大语言模型和向量数据库，实现AI驱动的智能面试辅助平台。系统强调高可用性、可扩展性和AI集成的简洁性。

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    前端层 (React + TypeScript)                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  页面组件 │ 状态管理 │ API客户端 │ 组件库 │ 工具函数      │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────────────────┘
                      │ HTTP/WebSocket
┌─────────────────────┼───────────────────────────────────────────┐
│                    网关层 (Spring Boot)                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  控制器层 │ 异常处理 │ 参数校验 │ 跨域配置 │ 安全控制      │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────────┘
                      │ Service调用
┌─────────────────────┼───────────────────────────────────────────┐
│                   业务逻辑层                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 领域服务 │ AI集成 │ 异步处理 │ 业务规则 │ 数据转换         │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────────┘
                      │ Repository/DAO
┌─────────────────────┼───────────────────────────────────────────┐
│                   数据访问层                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  JPA/Hibernate │ VectorStore │ Redis客户端 │ 文件存储      │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
┌─────────┴─────┐ ┌───┴─────┐ ┌───┴─────┐
│ PostgreSQL    │ │ Redis    │ │ S3存储    │
│ + pgvector    │ │ Stream   │ │ (MinIO)   │
└───────────────┘ └─────────┘ └─────────┘
```

## 分层架构详解

### 1. 前端层

#### 技术栈
- **React 18.3**: 函数式组件 + Hooks
- **TypeScript 5.6**: 类型安全
- **Vite 5.4**: 快速构建工具
- **Tailwind CSS 4.1**: 原子化CSS框架
- **React Router 7.11**: 客户端路由
- **Framer Motion 12.23**: 动画库
- **Recharts 3.6**: 数据可视化
- **Lucide React 0.468**: 图标库

#### 架构特点
- **组件化**: 页面级组件 + 通用组件
- **状态管理**: React Hooks + Context API
- **API集成**: 自定义hooks封装API调用
- **类型安全**: TypeScript接口定义
- **响应式设计**: Tailwind CSS媒体查询

### 2. 后端层

#### 技术栈
- **Spring Boot 4.0**: 应用框架
- **Java 21**: 开发语言
- **Spring AI 2.0**: AI集成框架
- **PostgreSQL 14+**: 关系数据库
- **Redis 6+**: 缓存 + 消息队列
- **Apache Tika 2.9.2**: 文档解析
- **iText 8.0.5**: PDF导出
- **MapStruct 1.6.3**: 对象映射

#### 分层设计

##### 控制器层 (Controller)
```java
@RestController
@RequestMapping("/api/v1/resume")
public class ResumeController {
    // RESTful API接口
    // 参数校验 @Valid
    // 统一响应 @Result
}
```

##### 服务层 (Service)
```java
@Service
public class ResumeAnalysisService {
    // 业务逻辑封装
    // 事务管理 @Transactional
    // AI集成 ChatClient
}
```

##### 仓储层 (Repository)
```java
@Repository
public interface ResumeRepository extends JpaRepository<Resume, Long> {
    // 数据访问接口
    // 自定义查询方法
}
```

##### 配置层 (Config)
```java
@Configuration
public class AppConfig {
    // Bean配置
    // 外部服务集成
}
```

## AI集成架构

### Spring AI 组件集成

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   ChatClient    │    │ EmbeddingModel  │    │  VectorStore    │
│                 │    │                 │    │                 │
│ • 对话交互      │    │ • 文本向量化    │    │ • 向量存储      │
│ • 结构化输出    │    │ • 1024维向量    │    │ • 相似度检索    │
│ • Prompt模板    │    │ • DashScope集成 │    │ • pgvector      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   DashScope (阿里云)     │
                    │                         │
                    │ • qwen-plus模型         │
                    │ • OpenAI兼容接口        │
                    │ • 流式响应支持          │
                    └─────────────────────────┘
```

### AI处理流程

#### 同步处理
```
用户请求 → Controller → Service → ChatClient调用 → 返回结果
```

#### 异步AI处理
```
用户请求 → 任务创建 → 发送到Redis Stream → 立即返回
                                            ↓
                                     Consumer消费 → AI处理 → 结果存储
```

## 数据架构

### 数据存储设计

#### PostgreSQL (关系数据 + 向量数据)
```sql
-- 简历表
CREATE TABLE resume (
    id BIGSERIAL PRIMARY KEY,
    file_name VARCHAR(255),
    content_hash VARCHAR(64),
    status VARCHAR(20),
    analysis_result JSONB,
    created_at TIMESTAMP
);

-- 向量存储表 (Spring AI自动创建)
CREATE TABLE vector_store (
    id VARCHAR(255) PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding VECTOR(1024)
);
```

#### Redis (缓存 + 消息队列)
```redis
-- 缓存
interview:session:123 → 用户会话数据

-- 异步任务队列
STREAM interview:analysis → 任务消息
GROUP consumer-group → 消费者组
```

#### S3兼容存储 (文件存储)
```
bucket: interview-guide
├── resumes/
│   ├── hash1.pdf
│   └── hash2.docx
└── knowledge/
    ├── kb1/
    │   ├── doc1.pdf
    │   └── doc2.md
    └── kb2/
        └── doc3.txt
```

## 异步处理架构

### Redis Stream 设计

#### 生产者 (Producer)
```java
@Service
public class AsyncTaskProducer {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void sendAnalysisTask(ResumeAnalysisTask task) {
        redisTemplate.opsForStream().add("interview:analysis", task);
    }
}
```

#### 消费者 (Consumer)
```java
@Component
public class ResumeAnalysisConsumer {
    @StreamListener(target = "interview:analysis",
                    condition = "headers['type']=='resume_analysis'")
    public void handleAnalysis(Message<ResumeAnalysisTask> message) {
        // 执行AI分析
        // 更新数据库状态
    }
}
```

#### 状态流转
```
PENDING → PROCESSING → COMPLETED
    ↓         ↓            ↓
  失败重试   处理中       完成
    ↓         ↓            ↓
 FAILED     TIMEOUT      SUCCESS
```

## 部署架构

### Docker容器化部署

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Compose                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │   Frontend  │ │   Backend   │ │   MinIO     │           │
│  │   (React)   │ │ (SpringBoot)│ │  (S3存储)   │           │
│  │             │ │             │ │             │           │
│  │ Port: 80    │ │ Port: 8080  │ │ Port: 9000  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ PostgreSQL  │ │    Redis    │ │   PgAdmin   │           │
│  │ + pgvector  │ │             │ │ (可选管理)  │           │
│  │ Port: 5432  │ │ Port: 6379  │ │ Port: 5050  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

### 网络架构
```
Internet
    ↓
Load Balancer (Nginx)
    ↓
┌─────────────────┬─────────────────┐
│   Frontend      │   Backend API   │
│   (React SPA)   │   (Spring Boot) │
│                 │                 │
│ • 静态资源      │ • RESTful API   │
│ • 路由处理      │ • WebSocket      │
│ • CDN加速       │ • SSE流式响应   │
└─────────────────┴─────────────────┘
    ↓
┌─────────────────┬─────────────────┬─────────────────┐
│   PostgreSQL    │     Redis       │   MinIO S3      │
│   (数据存储)     │   (缓存/队列)    │   (文件存储)    │
└─────────────────┴─────────────────┴─────────────────┘
```

## 安全性设计

### API安全
- **JWT认证**: 用户身份验证
- **API密钥**: 外部服务访问控制
- **CORS配置**: 跨域资源共享
- **参数校验**: JSR-303 Bean Validation

### 数据安全
- **敏感信息加密**: API密钥、数据库密码
- **SQL注入防护**: JPA参数化查询
- **XSS防护**: 前端输入过滤
- **文件上传安全**: 类型校验、病毒扫描

## 可扩展性设计

### 水平扩展
- **无状态服务**: 后端服务可水平扩展
- **分布式缓存**: Redis集群支持
- **数据库分库分表**: 支持高并发场景

### 垂直扩展
- **模块化设计**: 新功能独立模块
- **SPI扩展**: 自定义AI模型、存储后端
- **配置化**: 动态调整系统参数

### 性能优化
- **缓存策略**: 多级缓存架构
- **异步处理**: 非阻塞IO操作
- **连接池**: 数据库、Redis连接复用
- **CDN加速**: 静态资源分发

## 监控和运维

### 应用监控
- **健康检查**: Spring Boot Actuator
- **指标收集**: JVM、业务指标
- **日志聚合**: 结构化日志输出

### AI监控
- **调用统计**: AI接口响应时间、成功率
- **Token使用**: API调用消耗统计
- **错误监控**: 异常情况告警

### 基础设施监控
- **容器监控**: Docker stats
- **数据库监控**: PostgreSQL性能指标
- **缓存监控**: Redis内存使用

## 总结

本架构设计充分考虑了AI应用的特殊需求，通过Spring AI框架简化了AI集成复杂度，同时保证了系统的可扩展性、高可用性和可维护性。采用异步处理架构确保了用户体验，同时支持复杂的AI处理流程。