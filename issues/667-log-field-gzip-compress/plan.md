# Plan: #667 日志大字段 gzip+base64 压缩存储，支持 trace 查询完整内容

## 目标

将 `ConsoleLogRepo` 的日志截断策略替换为 gzip+base64 压缩存储，使通过 trace ID 在 ES 中查询日志时能获取完整的 request/response 内容，提升生产排障效率。

## 非目标

- 不改变 `FULL_LOGGER` 的行为（仍保留完整明文日志写入本地文件）
- 不改变 Filebeat → ES 的采集链路
- 不处理历史已截断日志的回填
- 不改动 `/api/logs/trace` 工作流侧的解压逻辑（该路径依赖外部 workflow 服务）

## 验收标准

1. 单字段（request 或 response）序列化超过阈值时，该字段被 gzip+base64 压缩存储，对应 `*Compressed` 标记为 `true`
2. 未超阈值字段保持原样明文输出，完全向后兼容
3. 前端通过 `/api/logs` 接口查询 ES 日志时，压缩字段自动解压后返回完整内容
4. 新增 `CompressUtils` 工具类，含 compress/decompress 方法及单元测试
5. 配置项 `bella.log.compress-threshold-bytes` 可控，默认 10KB

## 约束

- 压缩后单条 JSON 仍需在 ES 单文档大小限制内（通常 100MB，压缩后远低于此值）
- 不能破坏现有 `EndpointProcessData` 的 Jackson 序列化兼容性
- `FULL_LOGGER` 路径必须继续输出未压缩明文
- 日志记录在高吞吐 Disruptor pipeline 中执行，压缩操作的性能影响需可控

## 变更范围

| 模块 | 文件 | 类型 |
|------|------|------|
| sdk (utils) | `api/sdk/src/main/java/com/ke/bella/openapi/utils/CompressUtils.java` | 新建 |
| server (log) | `api/server/src/main/java/com/ke/bella/openapi/db/log/ConsoleLogRepo.java` | 修改 |
| sdk (model) | `api/sdk/src/main/java/com/ke/bella/openapi/EndpointProcessData.java` | 修改 |
| server (config) | `api/server/src/main/resources/application.yml` | 修改 |
| server (config) | `api/server/src/main/resources/application-docker.yml` | 修改 |
| web (BFF) | `web/src/app/api/logs/route.ts` | 修改 |
| server (test) | `api/server/src/test/java/com/ke/bella/openapi/utils/CompressUtilsTest.java` | 新建 |

## 实现思路

### Step 1: 新建 `CompressUtils` 工具类

- **目标**：提供可复用的 gzip+base64 压缩/解压工具
- **涉及文件**：`api/sdk/src/main/java/com/ke/bella/openapi/utils/CompressUtils.java`（新建）
- **具体改动**：
  - `compress(String input) → String`：UTF-8 → gzip → Base64 encode
  - `decompress(String input) → String`：Base64 decode → gunzip → UTF-8
  - 使用 `java.util.zip.GZIPOutputStream` / `GZIPInputStream` + `java.util.Base64`
  - 参考项目中 `HuoshanStreamAsrCallback` 的已有 gzip 实现模式

### Step 2: 修改 `EndpointProcessData` 数据模型

- **目标**：添加压缩标记字段
- **涉及文件**：`api/sdk/src/main/java/com/ke/bella/openapi/EndpointProcessData.java`
- **具体改动**：
  - 添加 `private boolean requestCompressed = false` 字段
  - 添加 `private boolean responseCompressed = false` 字段
  - 添加对应的 getter/setter
  - 字段仅在序列化为 JSON 输出到日志时生效，标识该条日志的 request/response 是否为压缩态

### Step 3: 改造 `ConsoleLogRepo.record()` 逻辑

- **目标**：用压缩替代截断
- **涉及文件**：`api/server/src/main/java/com/ke/bella/openapi/db/log/ConsoleLogRepo.java`
- **具体改动**：
  - 新增配置 `@Value("${bella.log.compress-threshold-bytes:10240}")` 作为压缩阈值（默认 10KB）
  - 保留 `FULL_LOGGER` 先输出完整明文日志的逻辑不变
  - 替换原有"超过 max-size-bytes 则移除字段"逻辑为：
    1. 分别序列化 `request` 和 `response` 字段
    2. 如果单字段序列化大小超过 `compress-threshold-bytes`，调用 `CompressUtils.compress()` 压缩
    3. 将压缩后的 base64 字符串设为该字段的值
    4. 设置对应 `*Compressed = true` 标记
  - 可保留 `bella.log.max-size-bytes` 作为整体日志上限兜底（极端情况下压缩后仍超大时走原截断逻辑），但正常场景下压缩后不会触发

### Step 4: 添加配置项

- **目标**：增加压缩阈值配置
- **涉及文件**：
  - `api/server/src/main/resources/application.yml`
  - `api/server/src/main/resources/application-docker.yml`
- **具体改动**：
  - 在 `bella.log` 配置块下添加 `compress-threshold-bytes: 10240`

### Step 5: 前端 BFF 解压逻辑

- **目标**：ES 查询返回的压缩字段自动解压
- **涉及文件**：`web/src/app/api/logs/route.ts`
- **具体改动**：
  - 遍历 ES 查询结果，检测 `data_info_msg_requestCompressed` / `data_info_msg_responseCompressed` 字段
  - 当标记为 `true` 时，对 `data_info_msg_request` / `data_info_msg_response` 执行 Base64 decode → gunzip（Node.js 使用 `zlib.gunzipSync` + `Buffer.from(str, 'base64')`）
  - 将解压后的明文替换回结果中返回前端

### Step 6: 编写单元测试

- **目标**：验证压缩/解压正确性
- **涉及文件**：`api/server/src/test/java/com/ke/bella/openapi/utils/CompressUtilsTest.java`（新建）
- **具体改动**：
  - 测试 compress → decompress 往返一致性
  - 测试空字符串、超大文本（1MB+）、中文内容
  - 验证压缩率符合预期（JSON 文本 80-90%）

## 风险与依赖

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 高 QPS 下 gzip 压缩消耗 CPU | 影响 Disruptor pipeline 吞吐 | 仅对超阈值字段压缩；可设置较高阈值（如 10KB）减少触发频率 |
| ES 中字段名映射变化 | 新增 `*Compressed` 布尔字段需 ES mapping 兼容 | ES dynamic mapping 默认支持新字段自动识别 |
| 前端解压失败 | base64 损坏或格式异常导致页面报错 | BFF 侧 try-catch，解压失败时返回原始 base64 字符串并附带错误提示 |
| 历史日志不含 `*Compressed` 标记 | 旧日志字段为 null/不存在 | 解压逻辑默认 `false`，不影响旧日志展示 |
