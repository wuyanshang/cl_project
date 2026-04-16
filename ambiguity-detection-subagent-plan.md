# 歧义检测服务 - Sub-Agent 编排方案（方案一：两阶段串行）

## 一、背景与目标

### 1.1 现状

当前 `ambiguity-detection-service` 的架构是**单 Agent 直线管道**：

```
Excel输入 → 批量拆分 → DetectionService（单个 ChatClient + 写死 SKILL.md） → 写Excel
```

- `AiConfig` 中只有一个 `detectionChatClient` Bean，system prompt 固定加载 `ambiguity-detection/SKILL.md`
- `DetectionService` 硬绑定这一个 ChatClient，所有数据无差别走同一个 skill
- `AmbiguityOrchestrator` 只做批次拆分和并发执行，无路由/分发/二次调用逻辑
- SKILL.md 中虽然描述了"A2 类型需调用 terminology-search Sub-Agent"，但实际上这个调用**完全依赖 LLM 在对话中模拟**，没有真正的独立 agent 实例

### 1.2 目标

将 `terminology-search` Sub-Agent **从 SKILL.md 中的文字描述变成真正独立运行的 agent**，具体来说：

1. 新增独立的 `terminologySearchChatClient`，拥有自己的 system prompt（`terminology-search/SKILL.md`）
2. 在编排层实现**条件触发**：仅当主 Agent 检测出 A2 类型歧义时，才调用 terminology-search
3. 将检索结果回注给主 Agent，生成带标准术语引用的最终 A2 结果
4. 整个流程对 API 调用方透明，输入输出格式不变

### 1.3 方案选型理由

| 方案 | 复杂度 | 适用场景 | 本次选择 |
|------|--------|----------|----------|
| 方案一：两阶段串行 | 低 | agent 数量少（2个），关系固定 | ✅ 本次采用 |
| 方案二：Function Calling | 中 | LLM 需自主决定调用时机 | 后续考虑 |
| 方案三：Graph 编排 | 高 | 3+ agent 复杂协作 | 后续考虑 |

---

## 二、整体架构

### 2.1 改造后的流程

```
Excel输入
    │
    ▼
AmbiguityOrchestrator（编排层）
    │
    ├── 阶段1: 批量拆分 → AmbiguityDetectionService（主Agent）
    │         system prompt = ambiguity-detection/SKILL.md
    │         返回初步检测结果（含 A1/A2 分类）
    │
    ├── 过滤: 提取所有 type=A2 的条目
    │
    ├── 阶段2: 对 A2 条目 → TerminologySearchService（Sub-Agent）
    │         system prompt = terminology-search/SKILL.md
    │         传入字段上下文（system/table/field 中英文名）
    │         返回 adam_results / acord_abbreviation_results / acord_naming_rule_results
    │
    ├── 阶段3: 将检索结果注入 → AmbiguityEnrichmentService（主Agent或轻量prompt）
    │         基于标准术语生成最终的 recommended_en/cn、reference_source、detail
    │
    └── 合并: 将 enriched A2 结果替换回原始结果列表
         │
         ▼
    写Excel输出
```

### 2.2 模块依赖关系

```
AmbiguityController
       │
       ▼
AmbiguityOrchestrator  ←──── 编排核心（新增阶段2/3逻辑）
       │
       ├── ExcelService              （不变）
       ├── AmbiguityDetectionService （原 DetectionService，改名以区分职责）
       ├── TerminologySearchService  （新增）
       └── AmbiguityEnrichmentService（新增）
```

---

## 三、资料目录准备

terminology-search Sub-Agent 需要检索 ADaM 和 ACORD 标准资料。需要在项目中准备资料目录。

### 3.1 资料目录结构

```
src/main/resources/
    ├── skills/
    │   ├── ambiguity-detection/
    │   │   └── SKILL.md              （已有，需微调）
    │   └── terminology-search/
    │       └── SKILL.md              （新增）
    └── reference/
        ├── adam/
        │   ├── claim.json
        │   ├── policy.json
        │   ├── party.json
        │   ├── agency.json
        │   └── ...
        ├── acord_abbreviations.json
        └── acord_naming_rules.md
```

### 3.2 资料文件说明

| 文件 | 用途 | 格式 |
|------|------|------|
| `adam/*.json` | ADaM 数据模型标准字段定义，按 Subject Area 拆分 | JSON 数组，每条含 entity/attribute_en/attribute_cn/attribute_definition |
| `acord_abbreviations.json` | ACORD 标准缩写对照表 | JSON 数组，每条含 word/abbr |
| `acord_naming_rules.md` | ACORD 命名规范原文 | Markdown 文本 |

> **注意**：如果资料文件较大，TerminologySearchService 可选择两种实现策略（见第五节）。

---

## 四、文件变更清单

### 4.1 新增文件

| 文件路径 | 说明 |
|----------|------|
| `src/main/resources/skills/terminology-search/SKILL.md` | Sub-Agent 的 system prompt |
| `src/main/resources/reference/adam/*.json` | ADaM 标准资料（需业务侧提供） |
| `src/main/resources/reference/acord_abbreviations.json` | ACORD 缩写对照表（需业务侧提供） |
| `src/main/resources/reference/acord_naming_rules.md` | ACORD 命名规则（需业务侧提供） |
| `src/main/java/.../service/TerminologySearchService.java` | Sub-Agent 调用服务 |
| `src/main/java/.../service/AmbiguityEnrichmentService.java` | A2 结果 enrichment 服务 |
| `src/main/java/.../model/TerminologySearchRequest.java` | 术语检索请求 DTO |
| `src/main/java/.../model/TerminologySearchResult.java` | 术语检索结果 DTO |

### 4.2 修改文件

| 文件路径 | 变更内容 |
|----------|----------|
| `AiConfig.java` | 新增 `terminologySearchChatClient` 和 `enrichmentChatClient` Bean |
| `DetectionService.java` | 改名为 `AmbiguityDetectionService`，职责不变 |
| `AmbiguityOrchestrator.java` | 新增阶段2/3编排逻辑，注入新服务 |
| `AmbiguityResult.java` | 新增 `recommended_en`、`recommended_cn`、`reference_source` 字段（如果尚未有） |
| `AmbiguityItem.java` | 确认 A2 专属字段已定义 |
| `application.yml` | 新增 terminology-search 相关配置 |
| `ambiguity-detection/SKILL.md` | 移除"自行模拟调用 Sub-Agent"的描述，改为"A2 结果将由编排层传递给 Sub-Agent" |

### 4.3 不变文件

| 文件路径 | 说明 |
|----------|------|
| `AmbiguityController.java` | API 接口不变，编排层内部改造对外透明 |
| `ExcelService.java` | Excel 读写逻辑不变（如需写入新字段则微调列映射） |
| `AssetField.java` | 输入模型不变 |
| `DetectionResponse.java` | 响应模型不变 |

---

## 五、详细编码方案

### 5.1 AiConfig.java — 新增 ChatClient Bean

```java
@Configuration
public class AiConfig {

    @Bean("detectionChatClient")
    public ChatClient detectionChatClient(
            ChatModel chatModel,
            @Value("classpath:skills/ambiguity-detection/SKILL.md") Resource skillResource
    ) throws IOException {
        String skillContent = loadResource(skillResource);
        return ChatClient.builder(chatModel)
                .defaultSystem(skillContent)
                .build();
    }

    @Bean("terminologySearchChatClient")
    public ChatClient terminologySearchChatClient(
            ChatModel chatModel,
            @Value("classpath:skills/terminology-search/SKILL.md") Resource skillResource
    ) throws IOException {
        String skillContent = loadResource(skillResource);
        return ChatClient.builder(chatModel)
                .defaultSystem(skillContent)
                .build();
    }

    @Bean("enrichmentChatClient")
    public ChatClient enrichmentChatClient(ChatModel chatModel) {
        String systemPrompt = """
            你是语义歧义检测的结果增强助手。你会收到：
            1. 一条被判定为 A2 歧义的字段数据（含字段上下文信息）
            2. terminology-search Sub-Agent 返回的标准术语检索结果
            
            你的任务是：
            - 基于检索结果，为该 A2 歧义生成最终的 detail（引用标准术语）
            - 按优先级生成 recommended_en 和 recommended_cn
            - 标注 reference_source
            
            优先级规则：
            1. ADaM 精确/模糊匹配 → 采用 ADaM 标准字段名，reference_source = "ADaM - {entity}实体"
            2. ACORD 缩写表 + 类词表组合 → 用 ACORD 标准缩写拼出推荐名，reference_source = "ACORD标准缩写/类词规则"
            3. 以上都无匹配 → AI 推理给出推荐，reference_source = "AI推理（无标准对照）"
            
            输出严格 JSON 格式：
            {
                "detail": "引用标准术语的歧义描述",
                "suggestion": "消歧建议",
                "recommended_en": "推荐英文名",
                "recommended_cn": "推荐中文名",
                "reference_source": "来源标注"
            }
            """;
        return ChatClient.builder(chatModel)
                .defaultSystem(systemPrompt)
                .build();
    }

    private String loadResource(Resource resource) throws IOException {
        return new String(resource.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    }
}
```

**设计说明**：

- `enrichmentChatClient` 使用内联 prompt 而非单独 SKILL.md，因为它的职责很轻量（纯合并/格式化），不需要独立 skill 文件
- 三个 ChatClient 共享同一个 `ChatModel`（同一个 qwen-max 实例），通过不同的 system prompt 区分角色
- 如后续 enrichment 逻辑变复杂，可随时提取为独立 SKILL.md

### 5.2 TerminologySearchService.java — 新增

```java
@Slf4j
@Service
public class TerminologySearchService {

    private final ChatClient chatClient;
    private final ObjectMapper objectMapper;

    @Value("${ambiguity.terminology-search.max-retries:1}")
    private int maxRetries;

    @Value("${ambiguity.terminology-search.retry-delay-ms:1000}")
    private long retryDelayMs;

    public TerminologySearchService(
            @Qualifier("terminologySearchChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
        this.objectMapper = new ObjectMapper()
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    /**
     * 对单条 A2 歧义字段调用 terminology-search Sub-Agent。
     * 传入字段的六维上下文信息，返回标准术语检索结果。
     */
    public TerminologySearchResult search(TerminologySearchRequest request) {
        String userMessage = formatSearchRequest(request);
        log.debug("术语检索请求: field_cn={}, field_en={}",
                request.getFieldCn(), request.getFieldEn());

        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            if (attempt > 0) {
                long delay = retryDelayMs * (1L << (attempt - 1));
                log.warn("术语检索重试（第{}次，等待{}ms）", attempt, delay);
                try { Thread.sleep(delay); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            try {
                String response = chatClient.prompt()
                        .user(userMessage)
                        .call()
                        .content();
                return parseResult(response);
            } catch (Exception e) {
                log.warn("术语检索第{}次尝试失败: {}", attempt + 1, e.getMessage());
            }
        }

        log.error("术语检索失败，返回空结果: field_cn={}", request.getFieldCn());
        return TerminologySearchResult.empty();
    }

    /**
     * 批量检索：对多条 A2 条目并发调用。
     */
    public List<TerminologySearchResult> searchBatch(List<TerminologySearchRequest> requests) {
        // 逐条调用（A2 条目通常远少于总数据量，无需并发池）
        // 如 A2 条目量大，可改为 CompletableFuture 并发
        return requests.stream()
                .map(this::search)
                .toList();
    }

    private String formatSearchRequest(TerminologySearchRequest req) {
        return String.format("""
                请根据以下字段上下文信息，检索相关的标准术语：
                
                ```json
                {
                  "system_en": "%s",
                  "system_cn": "%s",
                  "table_en": "%s",
                  "table_cn": "%s",
                  "field_en": "%s",
                  "field_cn": "%s"
                }
                ```
                """,
                nullToEmpty(req.getSystemEn()),
                nullToEmpty(req.getSystemCn()),
                nullToEmpty(req.getTableEn()),
                nullToEmpty(req.getTableCn()),
                nullToEmpty(req.getFieldEn()),
                nullToEmpty(req.getFieldCn()));
    }

    private TerminologySearchResult parseResult(String response) throws Exception {
        String json = extractJson(response);
        return objectMapper.readValue(json, TerminologySearchResult.class);
    }

    // extractJson / nullToEmpty 方法复用 DetectionService 中的逻辑
}
```

### 5.3 AmbiguityEnrichmentService.java — 新增

```java
@Slf4j
@Service
public class AmbiguityEnrichmentService {

    private final ChatClient chatClient;
    private final ObjectMapper objectMapper;

    public AmbiguityEnrichmentService(
            @Qualifier("enrichmentChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
        this.objectMapper = new ObjectMapper()
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    /**
     * 将术语检索结果与原始 A2 歧义信息合并，
     * 生成带标准术语引用的最终 A2 输出字段。
     */
    public AmbiguityItem enrich(
            AssetField field,
            AmbiguityItem originalA2Item,
            TerminologySearchResult searchResult) {

        String userMessage = String.format("""
                ## 原始字段信息
                - 系统英文名: %s
                - 系统中文名: %s
                - 表英文名: %s
                - 表中文名: %s
                - 字段英文名: %s
                - 字段中文名: %s
                
                ## 原始 A2 歧义判定
                - detail: %s
                - suggestion: %s
                
                ## 标准术语检索结果
                ```json
                %s
                ```
                
                请基于以上信息，生成最终的 A2 歧义输出。
                """,
                nullToEmpty(field.getSystemEn()),
                nullToEmpty(field.getSystemCn()),
                nullToEmpty(field.getTableEn()),
                nullToEmpty(field.getTableCn()),
                nullToEmpty(field.getFieldEn()),
                nullToEmpty(field.getFieldCn()),
                originalA2Item.getDetail(),
                originalA2Item.getSuggestion(),
                toJson(searchResult));

        try {
            String response = chatClient.prompt()
                    .user(userMessage)
                    .call()
                    .content();
            return mergeEnrichedResult(originalA2Item, response);
        } catch (Exception e) {
            log.warn("A2 enrichment 失败，保留原始结果: {}", e.getMessage());
            originalA2Item.setReferenceSource("AI推理（无标准对照）");
            return originalA2Item;
        }
    }

    private AmbiguityItem mergeEnrichedResult(
            AmbiguityItem original, String response) throws Exception {
        String json = extractJson(response);
        JsonNode enriched = objectMapper.readTree(json);

        if (enriched.has("detail"))
            original.setDetail(enriched.get("detail").asText());
        if (enriched.has("suggestion"))
            original.setSuggestion(enriched.get("suggestion").asText());
        if (enriched.has("recommended_en"))
            original.setRecommendedEn(enriched.get("recommended_en").asText());
        if (enriched.has("recommended_cn"))
            original.setRecommendedCn(enriched.get("recommended_cn").asText());
        if (enriched.has("reference_source"))
            original.setReferenceSource(enriched.get("reference_source").asText());

        return original;
    }
}
```

### 5.4 Model 层新增/修改

#### 5.4.1 TerminologySearchRequest.java — 新增

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TerminologySearchRequest {
    private String systemEn;
    private String systemCn;
    private String tableEn;
    private String tableCn;
    private String fieldEn;
    private String fieldCn;

    public static TerminologySearchRequest from(AssetField field) {
        return TerminologySearchRequest.builder()
                .systemEn(field.getSystemEn())
                .systemCn(field.getSystemCn())
                .tableEn(field.getTableEn())
                .tableCn(field.getTableCn())
                .fieldEn(field.getFieldEn())
                .fieldCn(field.getFieldCn())
                .build();
    }
}
```

#### 5.4.2 TerminologySearchResult.java — 新增

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TerminologySearchResult {

    private SearchContext searchContext;
    private List<AdamResult> adamResults;
    private List<AcordAbbreviation> acordAbbreviationResults;
    private List<AcordNamingRule> acordNamingRuleResults;

    public static TerminologySearchResult empty() {
        return new TerminologySearchResult(null, List.of(), List.of(), List.of());
    }

    @Data
    public static class SearchContext {
        private String detectedDomain;
        private String searchScope;
        private List<String> searchKeywords;
    }

    @Data
    public static class AdamResult {
        private String entity;
        private String attributeEn;
        private String attributeCn;
        private String attributeDefinition;
        private String matchType;   // exact_en / fuzzy_en / keyword_cn / entity_related
    }

    @Data
    public static class AcordAbbreviation {
        private String word;
        private String abbr;
    }

    @Data
    public static class AcordNamingRule {
        private String section;
        private String content;
    }
}
```

#### 5.4.3 AmbiguityItem.java — 修改（确认字段完整）

确保 `AmbiguityItem` 包含 A2 专属字段：

```java
@Data
public class AmbiguityItem {
    private String type;        // A1 / A2
    private String severity;    // HIGH / MEDIUM / LOW
    private String detail;
    private String suggestion;

    // A2 专属字段
    private String recommendedEn;
    private String recommendedCn;
    private String referenceSource;
}
```

### 5.5 AmbiguityOrchestrator.java — 核心改造

这是改动量最大的文件，新增阶段2和阶段3的编排逻辑。

```java
@Slf4j
@Service
public class AmbiguityOrchestrator {

    private final ExcelService excelService;
    private final AmbiguityDetectionService detectionService;        // 原 DetectionService
    private final TerminologySearchService terminologySearchService; // 新增
    private final AmbiguityEnrichmentService enrichmentService;      // 新增

    @Value("${ambiguity.default-batch-size:10}")
    private int defaultBatchSize;

    @Value("${ambiguity.max-concurrency:5}")
    private int maxConcurrency;

    @Value("${ambiguity.terminology-search.enabled:true}")
    private boolean terminologySearchEnabled;

    public AmbiguityOrchestrator(
            ExcelService excelService,
            AmbiguityDetectionService detectionService,
            TerminologySearchService terminologySearchService,
            AmbiguityEnrichmentService enrichmentService) {
        this.excelService = excelService;
        this.detectionService = detectionService;
        this.terminologySearchService = terminologySearchService;
        this.enrichmentService = enrichmentService;
    }

    public DetectionResponse process(String inputPath, int batchSize) throws IOException {
        if (batchSize <= 0) batchSize = defaultBatchSize;

        List<AssetField> allData = excelService.readExcel(inputPath);
        List<List<AssetField>> batches = splitBatches(allData, batchSize);
        log.info("共 {} 条数据，分为 {} 个批次", allData.size(), batches.size());

        // ===== 阶段1：主Agent 歧义检测 =====
        List<AmbiguityResult> allResults = executeParallelDetection(batches);

        // ===== 阶段2 + 3：A2 术语检索与结果增强 =====
        if (terminologySearchEnabled) {
            enrichA2Results(allData, allResults);
        }

        String outputPath = excelService.writeResultExcel(inputPath, allData, allResults);
        return buildResponse(allResults, outputPath);
    }

    /**
     * 阶段2+3：提取 A2 条目 → 调用 terminology-search → enrichment 合并
     */
    private void enrichA2Results(
            List<AssetField> allData, List<AmbiguityResult> allResults) {

        // 收集所有包含 A2 歧义的条目索引
        List<Integer> a2Indices = new ArrayList<>();
        for (int i = 0; i < allResults.size(); i++) {
            AmbiguityResult result = allResults.get(i);
            if (result.isHasAmbiguity() && result.getAmbiguities() != null) {
                boolean hasA2 = result.getAmbiguities().stream()
                        .anyMatch(item -> "A2".equals(item.getType()));
                if (hasA2) a2Indices.add(i);
            }
        }

        if (a2Indices.isEmpty()) {
            log.info("无 A2 类型歧义，跳过术语检索");
            return;
        }
        log.info("共 {} 条 A2 歧义，开始术语检索", a2Indices.size());

        // 阶段2：逐条调用 terminology-search
        for (int idx : a2Indices) {
            AssetField field = allData.get(idx);
            AmbiguityResult result = allResults.get(idx);

            TerminologySearchRequest searchReq = TerminologySearchRequest.from(field);
            TerminologySearchResult searchResult = terminologySearchService.search(searchReq);

            // 阶段3：对每个 A2 item 做 enrichment
            if (result.getAmbiguities() != null) {
                for (AmbiguityItem item : result.getAmbiguities()) {
                    if ("A2".equals(item.getType())) {
                        enrichmentService.enrich(field, item, searchResult);
                    }
                }
            }
        }

        log.info("A2 术语检索与结果增强完成");
    }

    // executeParallelDetection / buildResponse / splitBatches 方法不变
}
```

### 5.6 SKILL.md 调整

#### ambiguity-detection/SKILL.md 调整

移除主 Agent SKILL.md 中关于"自行调用 terminology-search Sub-Agent"的指令，改为通知 LLM：A2 的推荐命名将由编排层另行处理。

**修改前**（第128-158行）：

```
**⚠️ A2 判定后，必须执行标准术语检索：**
判定某条数据为 A2 后，在生成该条 A2 的 detail / suggestion / recommended 之前，
必须先调用 terminology-search Sub-Agent 进行标准术语检索。
...（调用方式和使用检索结果的详细说明）
```

**修改后**：

```
**⚠️ A2 判定说明：**
判定某条数据为 A2 后，只需输出 type、severity、detail、suggestion 四个基础字段。
recommended_en、recommended_cn、reference_source 将由编排层调用 terminology-search 
Sub-Agent 自动补充，主Agent 无需生成这三个字段。

detail 中请描述歧义点和可能的多种解释，不需要引用标准术语（标准术语将由编排层后续注入）。
```

这样做的好处：
- 主 Agent 的 system prompt 更短，减少 token 消耗
- 主 Agent 不需要"假装"调用 Sub-Agent，避免幻觉
- 职责分离更清晰

### 5.7 application.yml 新增配置

```yaml
ambiguity:
  default-batch-size: 10
  max-concurrency: 5
  max-retries: 2
  retry-delay-ms: 2000

  # 术语检索 Sub-Agent 配置
  terminology-search:
    enabled: true              # 是否启用术语检索（可通过配置关闭）
    max-retries: 1             # 术语检索每次最多重试次数
    retry-delay-ms: 1000       # 术语检索重试等待时间
```

---

## 六、数据流示例

### 6.1 完整流程走查

以输入数据 `field_cn=金额, field_en=amount, table_cn=t_common_data` 为例：

```
Step 1: AmbiguityDetectionService 检测
        → 输出: { type: "A2", severity: "HIGH", 
                  detail: "字段中文名'金额'指代不明，缺少业务限定",
                  suggestion: "需确认具体金额类型" }
        → 注意：此时没有 recommended_en/cn/reference_source

Step 2: Orchestrator 发现该条有 A2 → 调用 TerminologySearchService
        → 输入: { system_en/cn, table_en/cn, field_en: "amount", field_cn: "金额" }
        → 输出: { adam_results: [
                    { entity: "Claim", attribute_en: "ClaimAmt", attribute_cn: "理赔金额", ... },
                    { entity: "Claim", attribute_en: "PaidAmt", attribute_cn: "赔付金额", ... }
                  ],
                  acord_abbreviation_results: [{ word: "Amount", abbr: "Amt" }] }

Step 3: AmbiguityEnrichmentService 合并
        → 输出: { type: "A2", severity: "HIGH",
                  detail: "字段中文名'金额'指代不明，ADaM标准中Claim实体下区分了
                          ClaimAmt(理赔金额)、PaidAmt(赔付金额)、ReserveAmt(准备金)",
                  suggestion: "结合实际存储内容，确认具体金额类型并更新中英文名",
                  recommended_en: "ClaimAmt",
                  recommended_cn: "理赔金额",
                  reference_source: "ADaM - Claim实体" }
```

### 6.2 无 A2 时的短路

```
Step 1: AmbiguityDetectionService 检测
        → 所有条目均为 PASS 或 A1
Step 2: Orchestrator 发现无 A2 → 跳过术语检索
        → 日志: "无 A2 类型歧义，跳过术语检索"
Step 3: 直接写Excel
```

---

## 七、LLM 调用成本分析

| 阶段 | 调用次数 | Token 估算 | 说明 |
|------|----------|------------|------|
| 阶段1：主Agent | N / batchSize 次 | 高（SKILL.md 全文作 system prompt） | 与现有一致，无增量 |
| 阶段2：术语检索 | A2 条目数 × 1 次 | 中（SKILL.md 较短 + 检索指令） | 仅 A2 触发，通常 <20% 数据 |
| 阶段3：Enrichment | A2 条目数 × 1 次 | 低（prompt 很短 + 合并指令） | 可优化为批量调用 |

**优化空间**：阶段2和阶段3可合并为单次调用——将术语检索 + 推荐生成放在同一个 prompt 中，减少一轮 LLM 调用。但这会牺牲职责分离的清晰性，建议先按分离方案实现，确认可用后再考虑合并。

---

## 八、可选优化：本地检索替代 LLM

如果 ADaM/ACORD 资料是结构化 JSON，`TerminologySearchService` 可以**不走 LLM**，而是直接用 Java 代码做本地检索：

```java
@Service
public class TerminologySearchService {

    private List<AdamRecord> adamRecords;      // 启动时加载
    private List<AcordAbbreviation> acordAbbrs; // 启动时加载

    @PostConstruct
    public void init() {
        // 从 classpath:reference/ 加载所有 JSON 文件到内存
    }

    public TerminologySearchResult search(TerminologySearchRequest request) {
        // 1. 根据 system/table 关键词确定 domain
        // 2. 在 adamRecords 中做精确/模糊匹配
        // 3. 在 acordAbbrs 中做缩写匹配
        // 4. 组装返回结果
    }
}
```

**优势**：零 LLM 成本、零延迟、结果确定性 100%
**劣势**：模糊匹配能力不如 LLM（需要手写匹配逻辑）

**建议**：先用 LLM 方案跑通流程，收集实际检索 pattern，再考虑是否切到本地检索。

---

## 九、实施步骤

### 第一步：准备工作

1. 准备 ADaM / ACORD 资料文件，放入 `src/main/resources/reference/`
2. 将 `terminology-search/SKILL.md` 放入 `src/main/resources/skills/`

### 第二步：Model 层

3. 新增 `TerminologySearchRequest.java`
4. 新增 `TerminologySearchResult.java`
5. 确认 `AmbiguityItem.java` 包含 A2 专属字段

### 第三步：Service 层

6. 将 `DetectionService` 改名为 `AmbiguityDetectionService`（及相关引用）
7. 新增 `TerminologySearchService.java`
8. 新增 `AmbiguityEnrichmentService.java`

### 第四步：Config 层

9. 修改 `AiConfig.java`，新增两个 ChatClient Bean

### 第五步：编排层

10. 修改 `AmbiguityOrchestrator.java`，新增阶段2/3逻辑

### 第六步：Skill 调整

11. 修改 `ambiguity-detection/SKILL.md`，移除 Sub-Agent 自行调用描述
12. 修改 `application.yml`，新增配置项

### 第七步：测试验证

13. 单元测试：TerminologySearchService 解析逻辑
14. 集成测试：完整流程（含 A2 和不含 A2 两种 case）
15. 日志验证：确认阶段2/3的条件触发和短路跳过正常

---

## 十、风险与应对

| 风险 | 影响 | 应对措施 |
|------|------|----------|
| ADaM/ACORD 资料未及时准备 | 术语检索 Sub-Agent 无法返回有效结果 | 配置 `terminology-search.enabled=false` 关闭，退化为现有行为 |
| A2 条目过多导致延迟增加 | 整体处理时间变长 | 阶段2可改为并发调用；或将阶段2/3合并减少调用轮次 |
| LLM 术语检索结果不稳定 | enrichment 结果质量波动 | 后续切到本地 Java 检索（第八节方案） |
| 主Agent SKILL.md 修改后行为变化 | A2 判定逻辑可能偏移 | 对比修改前后的批量检测结果，确认一致性 |
