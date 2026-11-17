# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **项目已有 client_ip 解析模块，方案自取 x-forwarded-for**
  `src/server/platform/client-ip.mjs` 是权威实现：
  只有 `DSH_REVIEW_TRUST_PROXY=1` 时才读头，否则取 socket 地址。
  方案直接读 `x-forwarded-for`——**客户端可伪造**，
  伪造后限流被绕过。

- [ ] **enforceRateLimit 要求已登录用户**
  `src/server/platform/rate-limit.ts` 的 `enforceRateLimit`
  对空 userId 会抛 internal 错误。方案的键函数**绕过了这个约束**，
  但**没有改 `enforceRateLimit` 本身**——匿名路径仍会失败。

### L4 系统性
- [ ] **限流表是 key 主键，IP 会撑爆表**
  `rate_limiter_flexible` 的主键是 `key`。
  按 IP 限流会让 key 数量从「用户数」变成「IP 数」——
  NAT 后的公网 IP、爬虫 IP、IPv6 地址空间，
  **行数可能增长几个数量级**。方案没有容量估算。

- [ ] **匿名浏览是站点的正常流量**
  项目大量页面匿名可读（目录、评测详情）。
  给匿名加限流会**误伤正常用户**——尤其是共享 IP（学校、公司、运营商 NAT）。

- [ ] **NAT 后的用户会共享额度**
  一个公司 100 人共用一个出口 IP，所有人共享一份匿名额度。
  **一个爬虫就能让全公司无法访问**。

- [ ] **没有考虑限流失败时的降级**
  `rate_limiter_flexible` 读写失败（SQLITE_BUSY）时怎么办？
  项目现有实现是抛错，但在匿名路径上抛错 = **用户看到 500**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
