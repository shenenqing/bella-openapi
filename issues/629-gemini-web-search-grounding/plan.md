# Plan: #629 Gemini 适配层支持 Google Search grounding 工具

## 目标

在 Gemini 适配层完整支持 Google Search grounding 能力，覆盖请求注入、响应解析、grounding 元数据透传、按次计费四个方面。用户通过标准 `tools` 字段传入 `{"google_search": {}}` 即可启用，与 Google Gemini REST API 格式完全一致。

## 非目标

- 不改造独立的 Tavily Web Search 服务（`/v1/web/search`）
- 不修改 `GoogleAdaptor`（OpenAI 兼容 URL 路径，代理到 Google 的 OpenAI 兼容端点，该端点自身处理 grounding）
- 不在前端 Playground 添加 grounding 开关 UI
- 不改造 `responses` 计算器，而是扩展 `completion` 计算器复用相同模式

## 验收标准

1. 通过标准 `tools` 字段传入 `{"google_search": {}}` 时，Vertex 路径（`/v1/chat/completions`）能正确将 `google_search` 工具注入到 Gemini 请求
2. `tools` 中同时包含 `{"google_search": {}}` 和 `{"type": "function", ...}` 时，两者都能正确转换
3. Gemini 响应中的 `groundingMetadata` 能在 OpenAI 格式响应的 Choice 中以扩展字段 `grounding_metadata` 透传
4. 流式响应中最后一个 chunk 能正确携带 `grounding_metadata`
5. 原生 Gemini 路径（`/v1beta/models`）能正确序列化/反序列化 `google_search` 工具和 `groundingMetadata` 响应
6. `VertexConverter.convertUsage()` 能从 `groundingMetadata.webSearchQueries` 提取搜索次数，填入 `TokenUsage.tool_usage`
7. `completion` 计算器在 `CompletionPriceInfo.toolPrices` 配置后能正确按次计费
8. 单元测试覆盖：Tool DTO 序列化、GroundingMetadata 反序列化、VertexConverter 转换逻辑、grounding 计费计算

## 约束

- Java 8 运行时，不使用 Java 9+ 特性
- 保持对已有 `functionDeclarations` 和 `codeExecution` 工具的向后兼容
- 遵循现有 Lombok + Jackson 注解风格，`@JsonInclude(NON_NULL)` 避免序列化空值
- Gemini REST API 中请求 tool 名为 snake_case（`google_search`），响应字段为 camelCase（`groundingMetadata`），DTO 命名需分别对应
- `Message.Tool` 的 `_extra_body` 模式与 `ResponsesApiTool` 保持一致
- 计费扩展复用 `responses` 计算器已有的 `calculateToolCost()` 模式

## 变更范围

| 模块 | 文件 | 操作 |
|------|------|------|
| server/gemini DTO | `GroundingMetadata.java` | **新建**：grounding 响应元数据 DTO |
| server/gemini DTO | `Tool.java` | 修改：新增 `google_search` 字段（`@JsonProperty`） |
| server/gemini DTO | `Candidate.java` | 修改：新增 `groundingMetadata` 字段 |
| server/gemini DTO | `GeminiRequest.java` | 修改：新增 `toolConfig` 字段 |
| sdk/completion | `Message.java` | 修改：`Tool` 内部类新增 `@JsonAnySetter/@JsonAnyGetter` + `_extra_body` |
| server/protocol | `VertexConverter.java` | 修改：`convertTools()` 识别 `_extra_body` 中的 `google_search`；响应提取 groundingMetadata；usage 提取 tool_usage |
| sdk/completion | `CompletionResponse.java` | 修改：Choice 新增 `grounding_metadata`；TokenUsage 新增 `tool_usage` |
| sdk/completion | `StreamCompletionResponse.java` | 修改：Choice 新增 `grounding_metadata` |
| sdk/completion | `CompletionPriceInfo.java` | 修改：新增 `toolPrices` 字段 |
| server/cost | `CostCalculator.java` | 修改：提取 `calculateToolCost()` 为公共方法；`completion` 计算器增加 tool 按次计费 |
| server/gemini | `VertexAdaptor.java`（gemini 透传） | 修改：`parseSseResponse()` 解析 groundingMetadata |

## 实现思路

### Step 1：新建 `GroundingMetadata` DTO

- **涉及文件**：`api/server/src/main/java/com/ke/bella/openapi/protocol/completion/gemini/GroundingMetadata.java`（新建）
- **具体改动**：
  - 创建顶层类 `GroundingMetadata`，字段（全部 camelCase，与 Gemini 响应一致）：
    - `List<String> webSearchQueries` — 实际执行的搜索查询（列表长度 = 计费单位）
    - `SearchEntryPoint searchEntryPoint` — 搜索入口渲染内容
    - `List<GroundingChunk> groundingChunks` — 引用来源列表
    - `List<GroundingSupport> groundingSupports` — 文本段落与来源的映射
  - 内部类 `SearchEntryPoint`：`String renderedContent`
  - 内部类 `GroundingChunk`：`Web web`；内部类 `Web`：`String uri`, `String title`
  - 内部类 `GroundingSupport`：`Segment segment`, `List<Integer> groundingChunkIndices`, `List<Double> confidenceScores`
  - 内部类 `Segment`：`Integer startIndex`, `Integer endIndex`, `String text`
  - 全部使用 `@Data @Builder @NoArgsConstructor @AllArgsConstructor @JsonInclude(NON_NULL)` 注解

### Step 2：扩展 Gemini DTO（`Tool.java`、`Candidate.java`、`GeminiRequest.java`）

- **涉及文件**：
  - `api/server/.../gemini/Tool.java`
  - `api/server/.../gemini/Candidate.java`
  - `api/server/.../gemini/GeminiRequest.java`
- **具体改动**：
  - `Tool.java`：新增字段，因 Google REST API 中 tool 名为 snake_case，需用 `@JsonProperty` 注解：
    ```java
    @JsonProperty("google_search")
    private Map<String, Object> googleSearch;
    ```
    与 `codeExecution` 保持一致的 `Map<String, Object>` 类型。项目中 `Part.FileData` 已有 `mime_type`、`file_uri` 等 snake_case 先例
  - `Candidate.java`：新增 `private GroundingMetadata groundingMetadata;`（camelCase，与响应 JSON 一致，无需 `@JsonProperty`）
  - `GeminiRequest.java`：新增 `private Map<String, Object> toolConfig;`；在 `clearLargeData()` 中添加 `this.toolConfig = null;`

### Step 3：`Message.Tool` 新增 `@JsonAnySetter/@JsonAnyGetter`

- **涉及文件**：`api/sdk/src/main/java/com/ke/bella/openapi/protocol/completion/Message.java`
- **具体改动**：
  - 在 `Message.Tool` 类中新增，与 `ResponsesApiTool` 完全一致的模式：
    ```java
    @JsonIgnore
    private Map<String, Object> _extra_body;

    @JsonAnyGetter
    public Map<String, Object> getExtraBodyFields() {
        return _extra_body != null && !_extra_body.isEmpty() ? _extra_body : null;
    }

    @JsonAnySetter
    public void setExtraBodyField(String key, Object value) {
        if (_extra_body == null) {
            _extra_body = new HashMap<>();
        }
        _extra_body.put(key, value);
    }
    ```
  - 效果：用户发送 `{"google_search": {}}` 时，`google_search` 被原样捕获到 `_extra_body`，key 名为 `google_search`（snake_case）
  - 序列化时 `@JsonAnyGetter` 原样输出，对非 Vertex 适配器透明
  - 通用性：未来其他厂商的专属工具类型也能自动透传

### Step 4：修改 `VertexConverter.convertTools()` — 从 `_extra_body` 识别 `google_search`

- **涉及文件**：`api/server/.../completion/VertexConverter.java`
- **具体改动**：
  - 改造 `convertTools(List<Message.Tool>)` 方法，遍历时分两类处理：
    - 有 `type == "function"` 且 `function != null` → 走现有逻辑，收集 `FunctionDeclaration`
    - 有 `_extra_body` 且含 `"google_search"` key → 创建独立的 `Tool` 对象，仅设置 `googleSearch` 字段（值从 `_extra_body.get("google_search")` 取，转为 `Map<String, Object>`）
  - 最终返回的 `List<Tool>` 可包含多个对象：一个装 `functionDeclarations`（如果有 function 类型工具），一个装 `googleSearch`（如果有 google_search）
  - 在 `convertToVertexRequest()` 中，额外检查 `extra_body["toolConfig"]`，如果存在则设置到 `GeminiRequest.toolConfig`
  - 无需 key 名映射：用户发 `google_search`（snake_case），`Tool.java` 通过 `@JsonProperty("google_search")` 序列化为同名字段，与 Gemini API 完全一致
  - 用户调用示例：
    ```json
    {
      "tools": [
        {"google_search": {}},
        {"type": "function", "function": {"name": "get_weather", ...}}
      ]
    }
    ```

### Step 5：扩展 SDK 层 DTO（`CompletionResponse`、`StreamCompletionResponse`、`CompletionPriceInfo`）

- **涉及文件**：
  - `api/sdk/.../completion/CompletionResponse.java`
  - `api/sdk/.../completion/StreamCompletionResponse.java`
  - `api/sdk/.../completion/CompletionPriceInfo.java`
- **具体改动**：
  - `CompletionResponse.Choice`：新增 `private Object grounding_metadata;`
  - `StreamCompletionResponse.Choice`：新增 `private Object grounding_metadata;`
  - `CompletionResponse.TokenUsage`：新增 `private Map<String, Integer> tool_usage;`，语义与 `ResponsesApiResponse.Usage.tool_usage` 对齐
  - `CompletionPriceInfo`：新增 `private Map<String, BigDecimal> toolPrices;`，语义与 `ResponsesPriceInfo.toolPrices` 对齐。渠道配置示例：`{"web_search": 0.245}`（分/次）

### Step 6：修改 `VertexConverter` — 响应侧提取 groundingMetadata 和 tool_usage

- **涉及文件**：`api/server/.../completion/VertexConverter.java`
- **具体改动**：
  - `convertCandidate()`：检查 `candidate.getGroundingMetadata()`，非空时设置到 `Choice.grounding_metadata`
  - `convertStreamCandidate()`：同样处理（Gemini 流式在最后一个 chunk 携带 grounding 信息）
  - `convertUsage()`：新增参数 `List<Candidate> candidates`，从第一个 candidate 的 `groundingMetadata.webSearchQueries` 提取搜索次数，填入 `TokenUsage.tool_usage: {"web_search": N}`（N = `webSearchQueries.size()`）
  - 相应调整 `convertToOpenAIResponse()` 和 `convertGeminiToStreamResponse()` 中调用 `convertUsage()` 的地方，传入 candidates 参数

### Step 7：修改 `completion` 计算器 — 增加 tool 按次计费

- **涉及文件**：`api/server/.../cost/CostCalculator.java`
- **具体改动**：
  - 将 `responses` 计算器中的 `calculateToolCost(Map<String, BigDecimal> toolPrices, Map<String, Integer> toolUsage, List<ToolCostDetailItem> toolDetails)` 提取为 `CostCalculator` 内的公共静态方法
  - 在 `completion` 计算器的 `calculate()` 末尾增加：
    ```java
    if (price.getToolPrices() != null && usage.getTool_usage() != null) {
        totalCost = totalCost.add(calculateToolCost(
            price.getToolPrices(), usage.getTool_usage(), toolDetails));
    }
    ```
  - `CostDetails` 中携带 `toolDetails` 列表，与 `responses` 计算器输出格式一致
  - 现有渠道不配置 `toolPrices` 则不触发，完全向后兼容

### Step 8：修改 gemini 透传路径的日志解析

- **涉及文件**：`api/server/.../protocol/gemini/VertexAdaptor.java`
- **具体改动**：
  - `parseSseResponse()`：在累计 `part.getText()` 的同时，记录最后一次出现的 `candidate.getGroundingMetadata()`
  - 将 `webSearchQueries` 数量同步到日志中的 `CompletionResponse.TokenUsage.tool_usage`，确保透传路径计费准确
  - `parseResponseForLogging()`（非流式）：调用 `VertexConverter.convertToOpenAIResponse()` 已自动处理

## 风险与依赖

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| `GroundingMetadata` 在 server 模块，sdk 层 Choice 使用 `Object` 类型 | 类型安全性降低 | Jackson 序列化保证 JSON 输出正确；后续可将 DTO 下沉到 sdk 模块 |
| Gemini API grounding 响应结构可能随版本变化 | 字段不兼容 | `@JsonInclude(NON_NULL)` + Jackson 默认忽略未知字段提高容错性 |
| `Message.Tool` 新增 `@JsonAnySetter` 会捕获所有未知字段 | 非 Vertex 适配器收到意外字段 | 其他适配器序列化时 `@JsonAnyGetter` 原样输出，上游 API 自行忽略；VertexConverter 只读取已知 key |
| `calculateToolCost()` 需从 `responses` 计算器提取为公共方法 | 小范围重构 | 方法逻辑简单，提取为静态方法即可，不影响 responses 计算器行为 |
| Google grounding 定价可能调整 | 费率过时 | 配置在渠道的 `CompletionPriceInfo.toolPrices` 中，运营可随时更新 |
