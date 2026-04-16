# 歧义检测服务 v3 — 双Agent架构（模式二：真SubAgent）

## 一、架构概览

```
主 Agent（detectionChatClient）              子 Agent（subAgentClient）
┌──────────────────────────┐          ┌──────────────────────────┐
│ SKILL: ambiguity-detection│          │ SKILL: terminology-search │
│ 角色: 保险行业歧义检测专家  │          │ 角色: 术语检索专家         │
│                          │          │                          │
│ 工具: terminologySearch  │─调用──→ │ 工具:                     │
│   (子Agent包装函数)       │          │   keywordSearchAdam       │
│                          │←返回──  │   searchAcordAbbreviation  │
│                          │          │   browseAdamEntity        │
└──────────────────────────┘          └──────────────────────────┘
        │                                       │
        │ SKILL = system prompt                 │ SKILL = system prompt
        │ 每次调用强制携带                        │ 每次调用强制携带
        ▼                                       ▼
   Phase1+2 歧义检测                       ADaM/ACORD 参考数据检索
```

### 调用链路

```
DetectionService.detectBatch()
    │
    ├── chatClient.prompt().user(batchText).call()
    │     │
    │     ├── 主 Agent 处理 10 条数据
    │     │     ├── 第1条: Phase1 通过 → 不调子Agent
    │     │     ├── 第2条: Phase1 通过 → 不调子Agent
    │     │     ├── 第3条: Phase2 判定 A2
    │     │     │     └── 主 Agent 调用 terminologySearch(字段六维信息)
    │     │     │           │
    │     │     │           ├── Spring AI 执行 terminologySearch 函数
    │     │     │           │     └── subAgentClient.prompt().user(检索请求).call()
    │     │     │           │           │
    │     │     │           │           ├── 子 Agent 推理: "理赔系统，搜 claim 领域"
    │     │     │           │           ├── 子 Agent 调用 keywordSearchAdam(keyword, domain)
    │     │     │           │           │     └── Java 在内存 JSON 中匹配 → 返回结果
    │     │     │           │           ├── 子 Agent 调用 searchAcordAbbreviation(term)
    │     │     │           │           │     └── Java 在缩写表中查找 → 返回结果
    │     │     │           │           └── 子 Agent 返回结构化 JSON
    │     │     │           │
    │     │     │           └── 主 Agent 拿到子Agent结果 → 生成推荐命名
    │     │     │
    │     │     └── 第4-10条: 继续处理...
    │     │
    │     └── 返回完整 JSON 数组
    │
    ├── AmbiguityOrchestrator: Java 兜底增强（安全网）
    │     对 Agent 未调用子Agent 的 A2 项做 Java 确定性匹配
    │
    └── ExcelService: 写入结果 Excel
```

---

## 二、双Agent的职责分离

| 维度 | 主 Agent | 子 Agent |
|------|---------|---------|
| SKILL 文件 | ambiguity-detection/SKILL.md | terminology-search/SKILL.md |
| 角色 | 保险行业数据资产语义质量检测专家 | 保险行业数据标准术语检索专家 |
| 输入 | 批量字段数据（10条/批） | 单条字段的六维上下文 |
| 输出 | JSON 数组（检测结果 + 推荐命名） | JSON（ADaM/ACORD 匹配结果） |
| 工具 | `terminologySearch`（调用子Agent） | `keywordSearchAdam` / `searchAcordAbbreviation` / `browseAdamEntity` |
| 决策 | 什么时候需要检索（仅 A2） | 怎么检索（用哪个工具、搜什么关键词） |

---

## 三、文件变更清单

### 3.1 核心变更

| 文件 | 变更 |
|------|------|
| `config/AiConfig.java` | **重写**。两个 ChatClient Bean + 子Agent包装函数 |
| `skills/ambiguity-detection/SKILL.md` | A2 段落: 三个工具 → 一个 `terminologySearch` 子Agent调用 |
| `skills/terminology-search/SKILL.md` | **重写**为正式子Agent系统提示词（角色 + 工具 + 检索策略 + 输出格式） |

### 3.2 已有文件（上一轮创建，本轮不变）

| 文件 | 说明 |
|------|------|
| `config/ReferenceSearchFunctions.java` | 三个检索工具 Bean（现在由子Agent使用） |
| `service/ReferenceDataLoader.java` | 启动时加载参考数据 |
| `service/TerminologySearchService.java` | Java 确定性检索（Orchestrator 兜底用） |
| `model/AmbiguityItem.java` | A2 字段（recommendedEn/cn/referenceSource） |
| `model/AdamRecord.java` / `AcordAbbreviation.java` / `TerminologySearchResult.java` | 模型类 |
| `service/AmbiguityOrchestrator.java` | 含 Java 兜底增强 |
| `service/ExcelService.java` | 输出列含推荐命名 |
| `reference/adam/*.json` / `acord_abbreviations.json` | 示例参考数据 |

### 3.3 不变文件

| 文件 | 说明 |
|------|------|
| `DetectionService.java` | 不变。Spring AI 自动处理 function calling 多轮交互 |
| `AmbiguityController.java` | API 不变 |

---

## 四、LLM 调用成本分析

| 场景 | 主 Agent 调用 | 子 Agent 调用 | 总计 |
|------|-------------|-------------|------|
| 批次10条，0个A2 | 1次 | 0次 | 1次 |
| 批次10条，1个A2 | 1次（含1轮function call） | 1次（含2-3轮tool call） | ~2次 |
| 批次10条，3个A2 | 1次（含3轮function call） | 3次 | ~4次 |

相比模式一（单Agent + 3个工具）的额外成本：
- 每个 A2 多一次 LLM 调用（子Agent的推理）
- 但子Agent有独立的SKILL约束，检索质量更高
- 子Agent的prompt更短（只有检索指令），token消耗比主SKILL低

---

## 五、配置

```yaml
ambiguity:
  terminology-search:
    enabled: true   # false 时跳过 Java 兜底增强

  reference:
    adam-pattern: classpath:reference/adam/*.json
    acord-path: classpath:reference/acord_abbreviations.json

spring:
  ai:
    dashscope:
      chat:
        options:
          model: qwen-max
          temperature: 0.1
```

---

## 六、待办事项

- [ ] 替换 reference/ 下的示例数据为正式 ADaM/ACORD 数据
- [ ] 集成测试：验证主Agent→子Agent→工具的完整调用链
- [ ] 性能评估：测量子Agent引入的额外延迟
- [ ] 确认 cn_type / abbreviation_info 的编排层预处理逻辑
- [ ] （可选）为子Agent配置独立的 model/temperature（如用更快的小模型做检索）
