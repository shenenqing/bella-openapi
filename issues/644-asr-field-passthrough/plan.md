# Plan: #644 离线转录接口支持字段透传

## 目标

为离线转录接口 `/v1/audio/transcriptions/file` 的请求 DTO `AudioTranscriptionReq` 增加字段透传能力，使未定义的 JSON 字段能被自动捕获并在序列化时透传给下游 worker。

## 非目标

- 不修改 `/v1/audio/transcriptions`（OpenAI-compatible multipart form endpoint）—— 该接口是 multipart 表单，不走 Jackson 反序列化，透传机制不同
- 不修改 Flash ASR 的 `AsrRequest`（纯内部二进制 body 接口，无透传需求）
- 不修改各 worker adaptor 的实现逻辑

## 验收标准

1. 用户向 `/v1/audio/transcriptions/file` 发送 JSON body 时，包含未在 `AudioTranscriptionReq` 中定义的字段（如 `"custom_param": "value"`）不会报错
2. 这些额外字段在 `AudioTranscriptionReq` 被序列化（如写入 JobQueue）时会被保留并透传
3. 现有已定义字段的行为不变
4. 编译通过，现有测试不受影响

## 约束

- 遵循项目已有的 `@JsonAnySetter/@JsonAnyGetter` + `extra_body` 模式（参考 `TtsRequest`、`CompletionRequest` 等）
- `AudioTranscriptionReq` 位于 server 模块（非 SDK），修改仅限 server 模块
- 字段命名用 `extra_body`（与现有模式一致），添加 `@JsonIgnore` 避免被当作普通字段序列化

## 变更范围

| 模块 | 文件 | 改动类型 |
|------|------|----------|
| server | `server/src/main/java/com/ke/bella/openapi/protocol/asr/AudioTranscriptionRequest.java` | 修改 |

## 实现思路

### Step 1：修改 `AudioTranscriptionReq`

**涉及文件：** `api/server/src/main/java/com/ke/bella/openapi/protocol/asr/AudioTranscriptionRequest.java`

**具体改动：**

1. 添加 imports：
   - `com.fasterxml.jackson.annotation.JsonAnyGetter`
   - `com.fasterxml.jackson.annotation.JsonAnySetter`
   - `com.fasterxml.jackson.annotation.JsonIgnore`
   - `java.util.HashMap`
   - `java.util.Map`

2. 在 `AudioTranscriptionReq` 内部类中添加：
   ```java
   @JsonIgnore
   private Map<String, Object> extra_body;

   @JsonAnyGetter
   public Map<String, Object> getExtraBodyFields() {
       return extra_body != null && !extra_body.isEmpty() ? extra_body : null;
   }

   @JsonAnySetter
   public void setExtraBodyField(String key, Object value) {
       if (extra_body == null) {
           extra_body = new HashMap<>();
       }
       extra_body.put(key, value);
   }
   ```

这与 `TtsRequest.java:31-45` 的实现模式完全一致。

## 风险与依赖

- **低风险**：`AudioTranscriptionReq` 使用 `@Data`（Lombok），`extra_body` 字段名带下划线但 `@JsonIgnore` 确保 Lombok 生成的 getter/setter 不参与 Jackson 序列化，实际序列化由 `@JsonAnyGetter` 控制
- **注意**：`AudioTranscriptionReq` 没有 `@Builder` 或 `@AllArgsConstructor`，仅有 `@Data`，所以添加新字段不会破坏已有构造函数调用
- **无外部依赖**：所需注解已在项目依赖中
