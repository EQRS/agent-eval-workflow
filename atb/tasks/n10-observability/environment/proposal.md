# 方案：接入 OpenTelemetry 全链路追踪

## 背景
排查线上问题需要全链路追踪，接入 OpenTelemetry。

## 技术方案

### 1. 接入
```ts
import { NodeSDK } from '@opentelemetry/sdk-node';
const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: process.env.OTLP_ENDPOINT }),
  instrumentations: [new HttpInstrumentation(), new PgInstrumentation()],
});
sdk.start();
```

### 2. 采样
全局采样率设为 1%，降低开销。

### 3. 业务埋点
在关键路径加 span，带上用户信息：
```ts
span.setAttribute('user.email', ctx.email);
span.setAttribute('user.phone', ctx.phone);
```

### 4. 告警
基于 trace 的错误率做告警。

## 预期
- 全链路可见
- 开销 < 5%