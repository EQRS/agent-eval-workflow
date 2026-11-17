# 方案：站点上多实例部署

## 背景
要来一波流量高峰，单台服务器扛不住，想上 3 台 web 实例做负载均衡。
现在 web 与 worker 在同一台机器上共享一个 SQLite 文件。

## 技术方案

### 1. 部署调整
```yaml
services:
  web:
    deploy:
      replicas: 3
    volumes:
      - ${DSH_REVIEW_DATA_DIR}:/app/.data
```

三台机器各自挂载自己的数据目录。

### 2. 数据库
保持 SQLite 不变，每台机器一个本地库文件。

### 3. 限流
现有的 RateLimiterSQLite 改为各实例独立计数，
每台机器按自己的额度限制。

### 4. 会话
会话仍然是数据库里的 user_sessions，各实例查询自己的库。

## 预期
- 三台机器分担流量
- 吞吐提升 3 倍
- 改造量小，不改业务代码
