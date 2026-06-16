## Hi there 👋

# TokenLive：企业级高性能 LLM API 网关与治理平台

> 📖 **“在代码的脉络里，让治理永续，让生命长青。”** —— 纪念 TokenLive 的由来与精神传承。

---

## 🌟 项目使命与精神传承 (Our Mission)

在微服务与大模型交织的云原生时代，网关是所有流量的咽喉，也是决定 AI 应用生死存亡的“生命线”。**TokenLive** 并非是一个冰冷的流量转发器，TokenLive 的核心设计与治理理念，完全一脉相承自我们深耕多年的开源项目`joylive-agent`。我们继承了 JoyLive 强大的多活、全链路调度与无损治理能力，致力于在复杂的异构大模型路由下，为高并发的 Token 洪峰提供强韧的自愈与高可用底座。

---

## 🧱 核心技术背景与架构 (Architecture & Design)

TokenLive 深度践行了**“高内聚、低耦合、高性能”**的 Go 语言工程美学，采用了独创的 **Gin Shell + Engine Pipeline** “外壳+内核”分层架构。

```mermaid
graph TD
    User([下游客户端/业务系统]) -->|API Key 鉴权| GinShell[Gin Shell 外壳]
    
    subgraph GatewayEngine [Gateway Engine 内核 (零 Gin 依赖, 原生 net/http)]
        GinShell -->|HandleRequest| InboundChain{Inbound Filter 链}
        InboundChain -->|1. AuthZ 授权控制| LimitFilter
        LimitFilter -->|2. TPM/RPM 限流预扣| ValidateFilter[3. 参数与模型合法性校验]
        
        ValidateFilter --> FallbackInvoker[FallbackInvoker 模型级级联降级]
        
        subgraph ClusterInvoker [ClusterInvoker 单模型路由与重试]
            FallbackInvoker -->|获取可用端点| Discovery[Static/K8s 服务发现]
            Discovery --> RouterChain{Router Chain 动态过滤}
            RouterChain -->|熔断/模型/标签过滤| LoadBalancer[8 种高级负载均衡算法]
            LoadBalancer -->|挑选端点| ProviderInvoker[ProviderInvoker 适配器]
        end
        
        ProviderInvoker -->|调用失败 & 触发 error_rules| AttemptRetry[指数抖动退避重试]
        ProviderInvoker -->|SSE 增量流式拦截| OutboundChain{Outbound Filter 链}
        
        OutboundChain -->|1. Token 差额结算与退款| StickySave[2. 会话粘性保存]
        StickySave -->|3. 指标统计 & 审计日志| Done([最终成功响应])
    end

    subgraph StateLayer [状态与容灾层]
        LimitFilter & StickySave -.->|共享客户端| RedisState[Redis StateStore]
        OutboundChain -.->|持久化/结算失败| CompQueue[Compensation Queue 异常补偿队列]
        CompQueue -->|Redis Stream 异步消费重试| RedisState
    end
```

### 🛠️ 技术亮点
1. **Gin Shell + Engine Pipeline 骨架**：将外壳（路由、CORS、身份认证 AuthN）与网关内核（路由重试、模型熔断、流式计费）解耦。内核完全基于原生 Go `net/http` 开发，可零依赖作为 SDK 嵌入任何 Go 微服务中。
2. **高并发与内存池化**：贯穿请求生命周期的核心上下文 `GatewayContext` 采用 `sync.Pool` 进行对象池化复用，杜绝高并发下的频繁 GC，实现极低的时延波动。
3. **本地双轨二级缓存**：对 API Key 授权信息及治理策略采用 **30s 正向 LRU 缓存与 10s 负向缓存防穿透**，大幅降低 Redis 负载，即使在千级并发下也能提供微秒级的校验速度。
4. **高可靠异常补偿队列**：针对会话粘性、Token 结算等关键写操作，一旦 Redis 发生偶发故障，网关自动将任务推入 Redis Stream 驱动的**异常补偿队列 (Compensation Queue)**，通过后台消费组进行指数退避重试，保证财务与会话数据的最终一致性。

---

## 💎 产品特色与核心能力 (Product Features)

TokenLive 平台由两个子项目**双剑合璧**组成：**TokenLive Gateway (网关内核)** 与 **TokenLive Admin (管理控制台)**。

### 🛡️ TokenLive Gateway：坚如磐石的算力治理引擎

| 维度 | 功能特色 | 业务价值 |
| :--- | :--- | :--- |
| **多协议兼容与扩展** | 完全兼容 OpenAI 标准协议 (Chat, Embeddings, Models)；原生集成 OpenAI, Anthropic 等厂商；支持能力导向设计与 Endpoint 级别的 UpstreamModel 重写。 | 极简接入，业务代码无需修改即可随意切换大模型供应商，避免厂商锁定。 |
| **高可用流量治理** | 独创 Endpoint（单端点）与 Provider（服务商模型）**双层熔断隔离机制**；支持 Capability/Tag/CircuitBreaker 动态路由过滤链。 | 瞬间隔离故障端点或故障模型供应商，确保高并发下的系统自愈与业务高可用。 |
| **8 种高级负载均衡** | 内置 RoundRobin, Weighted, Random, LeastConnections, LeastLatency, Cost, Sticky ( Prompt Cache 亲和性) 等 8 种算法。 | 最大限度利用各上游节点的首字节延迟和 Prompt Cache 缓存，降低模型响应时延。 |
| **高精度流式结算** | 利用 `SSEInterceptWriter` 透明包装流式输出，增量解析首包时延 (TTFT) 与实际 Token 消耗。提供**“预扣费与差额退款结算”**。 | 完美应对大模型中途重试、级联降级、以及流式中断结算，避免 Token 漏扣或多扣。 |
| **配置无锁热更新** | 策略与机制解耦，通过 `atomic.Pointer` 整体原子替换内存 PolicyMatcher，策略随配随走。 | 策略热加载与变更无需重启网关，高并发请求零抖动。 |

### 📊 TokenLive Admin：精细化、多维度的控制面板

1. **基础资源管理**：
   * **供应商管理**：支持统一管理 OpenAI、Anthropic、Azure、DeepSeek、Qwen、自建 Ollama 等大模型接入点。
   * **模型与别名映射**：支持模型别名映射、多接入点的权重和优先级关联，支持在 Endpoint 级别精细重写上游模型名。
2. **可视化流量治理策略**：
   * 支持可视化的**路由策略**、**限流策略**、**熔断策略**、**故障注入**、**负载均衡选择**、**访问白名单权限**等配置。
   * 支持四级优先级策略覆盖算法：`用户+模型` > `模型+*` > `*+用户` > `全局兜底`，实现极细粒度的策略差异化下发。
3. **多空间（多租户）隔离**：
   * 提供多工作空间（租户级）的资源隔离，以组织不同部门、团队的供应商、模型和策略归属。
4. **精细的 RBAC 与审计**：
   * 基于 **Vue 3 + Vite + Ant Design Vue** 的高表现控制面板。
   * 采用 **GORM + Casbin** 的精细权限控制，内置 API Key 生命周期管理与操作审计日志，让每一次治理变更都有迹可循。

---

## 🚀 项目未来愿景 (Future Vision)

TokenLive 承载着微服务治理的经验积累和大模型时代的创新设计。在未来的路线图中，我们将持续推进以下方向：
- [ ] **更丰富的供应商集成**：深度接入 Google Gemini、DeepSeek-V3/R1、阿里 Qwen、Kimi 等国内外一流模型。
- [ ] **云原生服务发现**：全面打通 Kubernetes Discovery 动态服务发现，实现云原生大模型节点的自动上下线。
- [ ] **企业级多租户看板**：针对多团队 API Key 进行详细的多维度看板分析（Token 消费趋势、TTFT 监控、错误率分析、供应商成本核算等）。
- [ ] **大模型应用网关组件**：探索支持安全合规拦截（Prompt 敏感词过滤）、RAG 检索增强缓存、智能体（Agent）状态协同等高级 AI Gateway 特性。

**“开源不死，代码常青。”** 

无论技术如何快速迭代，TokenLive 将坚守使命，在大模型的繁荣生态中，做企业最坚实、最长青的算力治理底座。
