# 业务流程文档

## 概述

InterviewGuide 是一个基于大语言模型的智能面试辅助平台，主要包含三个核心业务模块：简历管理、模拟面试和知识库管理。所有核心AI处理均采用异步方式，通过Redis Stream实现解耦和可扩展性。

## 核心业务逻辑

### 1. 简历管理模块

#### 业务流程
```
文件上传 → 内容解析 → 异步分析 → 结果存储 → PDF导出
```

#### 详细流程
1. **文件上传**
   - 支持格式：PDF、DOCX、DOC、TXT
   - 文件大小限制：< 10MB
   - 内容哈希校验：防止重复上传

2. **文档解析**
   - 使用Apache Tika提取文本内容
   - 文本清理和预处理
   - 提取关键信息（技能、经验等）

3. **异步AI分析**
   - 通过Redis Stream发送分析任务
   - Spring AI ChatClient生成分析报告
   - 状态流转：PENDING → PROCESSING → COMPLETED/FAILED
   - 失败重试机制（最多3次）

4. **结果存储**
   - 分析结果存储到PostgreSQL
   - 支持历史记录查询

5. **PDF导出**
   - 使用iText 8生成分析报告PDF
   - 内置中文字体支持

#### 核心服务
- `FileStorageService`: 文件存储
- `DocumentParseService`: 文档解析
- `ResumeAnalysisService`: 简历分析
- `PdfExportService`: PDF导出

### 2. 模拟面试模块

#### 业务流程
```
简历选择 → 问题生成 → 实时问答 → 答案评估 → 报告生成
```

#### 详细流程
1. **问题生成**
   - 基于简历内容生成个性化问题
   - 使用Spring AI ChatClient和结构化输出
   - 问题分类：项目经验、技术基础、系统设计等
   - 问题分布：确保各类型问题均衡

2. **实时问答**
   - WebSocket/HTTP长轮询支持
   - Redis缓存会话状态
   - 实时保存用户答案

3. **答案评估**
   - 异步评估用户答案
   - AI生成详细反馈和建议
   - 计算分数和改进建议
   - 生成参考答案

4. **报告生成**
   - 综合评估报告
   - 趋势分析和统计信息
   - PDF报告导出

#### 核心服务
- `InterviewQuestionService`: 问题生成
- `AnswerEvaluationService`: 答案评估
- `InterviewReportService`: 报告生成
- `InterviewSessionService`: 会话管理

### 3. 知识库管理模块

#### 业务流程
```
文档上传 → 内容分块 → 向量化和存储 → RAG检索 → 流式问答
```

#### 详细流程
1. **文档处理**
   - 支持格式：PDF、DOCX、MD、TXT等
   - 使用Apache Tika解析文档
   - TokenTextSplitter分块（500 tokens/块，重叠50 tokens）

2. **向量化和存储**
   - Spring AI EmbeddingModel生成向量
   - pgvector存储1024维向量
   - HNSW索引优化检索性能
   - 元数据支持多知识库隔离

3. **RAG检索增强**
   - 用户查询 → 向量相似度检索
   - 检索Top-K相关文档片段
   - 构建上下文提供给AI

4. **流式问答**
   - SSE（Server-Sent Events）流式响应
   - 实时显示AI思考过程
   - 支持多轮对话和会话管理

#### 核心服务
- `KnowledgeBaseService`: 知识库管理
- `KnowledgeBaseVectorService`: 向量处理
- `RagChatSessionService`: RAG对话
- `DocumentChunkService`: 文档分块

## 异步处理架构

### Redis Stream 异步流程
```
用户请求 → 立即返回taskId → 发送消息到Stream
                                      ↓
                               Consumer消费消息
                                      ↓
                            执行异步任务（分析/向量化）
                                      ↓
                        更新数据库状态和结果
                                      ↓
                     前端轮询获取最新状态
```

### 状态管理
- **AsyncTaskStatus**: PENDING → PROCESSING → COMPLETED / FAILED
- **重试机制**: 失败任务自动重试，最多3次
- **监控**: 通过Redis Stream实现任务监控

## 数据流图

### 简历分析数据流
```
上传文件 → 文件存储(S3) → 解析文本 → 生成向量 → AI分析 → 存储结果
```

### 面试数据流
```
选择简历 → 生成问题 → 用户答题 → 评估答案 → 生成报告 → 存储统计
```

### 知识库数据流
```
上传文档 → 解析内容 → 分块处理 → 生成向量 → 存储pgvector → RAG查询
```

## 核心技术组件

### AI集成
- **Spring AI 2.0**: 统一AI接口抽象
- **ChatClient**: 对话交互
- **EmbeddingModel**: 文本向量化
- **VectorStore**: 向量存储和检索

### 存储层
- **PostgreSQL + pgvector**: 关系数据 + 向量存储
- **Redis**: 缓存 + 异步消息队列
- **S3兼容存储**: 文件存储

### 异步处理
- **Redis Stream**: 异步任务队列
- **@StreamListener**: 消费异步任务
- **状态机**: 任务状态流转

## 异常处理

### 全局异常处理
- `GlobalExceptionHandler`: 统一异常响应
- 业务异常：`BusinessException`
- 参数校验：Spring Validation

### AI调用异常
- 网络超时：重试机制
- API限流：降级处理
- 模型错误：fallback到默认内容

## 性能优化

### 查询优化
- 向量检索：HNSW索引
- 缓存策略：Redis缓存热点数据
- 分页查询：前端虚拟滚动

### 异步优化
- 长时间任务异步化
- 流式响应减少内存占用
- 任务队列削峰填谷

## 扩展性设计

### 模块化架构
- 清晰的分层架构
- 依赖倒置原则
- SPI扩展点

### 配置化设计
- 外部化配置
- 动态prompt模板
- 可配置AI模型

### 监控和运维
- 异步任务监控
- AI调用指标
- 性能监控接口