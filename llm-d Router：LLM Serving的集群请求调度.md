# llm-d Router：LLM Serving 的集群请求调度

> 本文是对[llm-d Router源码阅读笔记: 面向大规模 LLM Serving 的集群级智能请求调度层](https://zhuanlan.zhihu.com/p/2107219475208655682)和[gateway-api-inference-extension源码](https://github.com/kubernetes-sigs/gateway-api-inference-extension)两部分内容的总结。

---

## 一、要解决的问题：通用负载均衡为什么不适用于推理

Web 服务的负载均衡有两个隐含假设：每条请求处理成本相近；后端之间没有状态差别。推理服务在这两点上都不成立。

### 1.1 请求成本差异巨大

一条 50 token 的问答和一条带 10 万 token 文档的总结，在网关看来都是"一个 HTTP 请求"，占用的 GPU 时间却差几个数量级。按连接数做均衡，反映不出后端的实际负载。

### 1.2 成本只能预知一半

Prompt 长度写在请求体里，Prefill 计算量可以估算；但输出长度要到生成结束才确定，Decode 会占用 Pod 多久，调度时无从得知。"等待队列有 3 条请求"既可能是 3 条一句话问答，也可能是 3 篇要生成几千 token 的长文。

“成本只能预知一半”真正含义是**"需求无法预先声明"**，通用负载均衡以kube-scheduler 做对比，Pod 调度时声明了 `resources.requests`，推理请求推理请求的成本是"已知输入 + 随机输出"的叠加，无法预先告知调度器自己要运行多久。

补充一个容易被忽略的点：通用负载其实也不知道 Pod 会活多久，之所以没成为矛盾，是因为 Pod 生命周期长、每个 Pod 只调度一次；而推理是**每条请求都要调度一次、单条成本差几个数量级**，这才把"未知时长"放大成调度核心难题。

### 1.3 推理后端带着状态（KV Cache）

模型处理 Prompt 时为每个 token 算出中间结果存在显存里（KV Cache）。新请求的开头若与之前某条请求相同，这部分不用重算——前缀缓存命中。多轮对话、共用系统提示词、RAG 反复引用的文档都产生大量相同前缀。但前缀缓存只存在于算过它的那个 Pod 上：轮询会把同一会话的几轮对话分到不同 Pod，每个 Pod 都要重算完整历史，**首 Token 延迟（TTFT）上升**。此外显存容量本身限制 Pod 能接什么请求：KV Cache 快占满时分不到 KV 块只能排队；运行中的请求生成时也需不断分配新块，引擎只能抢占部分请求释放 KV 块后重算。因此一个 Pod 即使队列很短，只要显存接近占满，也已无法承接长请求。

### 1.4 共同根源与解法

判断请求该发往哪个 Pod，所需信息（包括各 Pod 的队列长度、显存占用、缓存了哪些前缀）**都在推理引擎内部**，通用负载均衡器无从获取。llm-d Router 要解决的正是：**把引擎内部状态引入调度决策**。

---

## 二、架构总览：llm-d Router 在 GAIE 体系中的位置

### 2.1 llm-d Router 是什么

- llm-d 是在 Kubernetes 上运行大模型推理的开源方案，Router 是其**流量入口**，位于客户端与模型 Pod 之间：推理请求先到 Router，再转给某个运行 vLLM 的 Pod。
- 对每条请求发挥两个作用：**准入控制**（现在能否放行）+ **路由**（从一组 vLLM Pod 中选一个）。
- Router 由两部分组成：**网关（Envoy）**负责收发 HTTP 流量；**EPP（Endpoint Picker）**是独立 Go 服务，准入与路由两项决策都由它完成。

### 2.2 GAIE：llm-d Router 建立在它之上

这套"网关 + EPP"的结构来自 Kubernetes 社区项目 **Gateway API Inference Extension（GAIE）**。GAIE 优化 K8s 自托管生成式模型，手段是**借用 Envoy 的 ext-proc（External Processing）协议，把任何同时支持 ext-proc 和 Gateway API 的网关升级为 Inference Gateway**。

术语定义：

- **Inference Gateway（IGW）**：耦合了 Endpoint Picker 的代理/负载均衡器；
- **Inference Scheduler**：可扩展的端点择优组件，基于模型服务的 Metrics and Capabilities 决策；
- **Metrics and Capabilities**：服务方提供的性能/可用性数据（前缀缓存状态、LoRA 适配器）；
- **EPP**：通过 ext-proc 协议通信的数据面组件，拦截推理请求并路由到最优副本。

**仓库现状必须注意**：完整 **EPP 及关联 API（InferenceObjective、InferenceModelRewrite）已迁往 `llm-d/llm-d-router`**；**BBR（Body Based Router）已迁往 `llm-d/llm-d-inference-payload-processor`**（即 llm-d IPP）；GAIE仓库仅保留 `InferencePool` API（`api/v1`）、轻量参考实现 **LWEPP**（`pkg/lwepp`，专供一致性测试）、**conformance 测试套件**（`conformance/`）与协议提案（`docs/proposals/`）。顶层结构：

```
api/v1/                  # InferencePool CRD（GA 核心）
apix/v1alpha1/           # InferencePoolImport（多集群，实验性）
pkg/lwepp/               # 轻量 EPP 参考实现（server/handlers/datastore/controller/metadata）
pkg/common/envoy/        # ext_proc 通用工具（metadata 提取/header 读取/body 分块）
conformance/             # 一致性测试 + 厂商通过报告（istio/agentgateway/nginx-gf/kgateway/ack 等）
docs/proposals/          # 003 模型服务协议 / 004 EPP 协议 / 0683 EPP 架构 / 0602 前缀缓存路由 等
cmd/lwepp/               # LWEPP 可执行入口
```

一句话：**GAIE 本体 = 一个 CRD + 一份 ext_proc 数据面协议 + 一份模型服务器指标协议 + 一套一致性测试；它把"智能"留在可替换的 EPP 插槽里（参考实现放轮询 LWEPP，生产放 llm-d EPP），把"转发"完全交给既有网关生态。** 

### 2.3 Envoy 与 EPP 的分工

llm-d 没有重新实现连接维持、TLS、超时、响应转发这些通用问题，直接用 Envoy 做网关，只在 EPP 里做推理调度：

- 对推理影响最大的是**流式响应**：`stream: true` 时 vLLM 随生成进度发 SSE，Envoy 收到一段转发一段。一条流可能持续几分钟，而 Envoy 路由默认超时 15 秒，官方 chart 因此把路由超时和流空闲超时设为 86400 秒；慢客户端时 Envoy 缓冲写满后暂停从 vLLM 读取，积压有上限。
- Envoy 与 EPP 经 **ext_proc** 协作（协议详见第四章）：每条请求两路通信——HTTP 承载用户请求与生成结果，gRPC 承载 Envoy 发给 EPP 的请求内容与 EPP 返回的选址指令。
- 改打分规则、加缓存感知策略只需修改 EPP，网关不动。**代价**：运行时两者仍是同步依赖——Envoy 转发每条请求前都要等 EPP 返回选址，**EPP 位于每条请求的必经路径上**。

### 2.4 一条请求的全程

以客服应用为例（每条请求带同一段 2000 token 系统提示词；集群有 A、B、C 三个 vLLM Pod，A 刚处理过这个应用）：

1. 客户端通过 OpenAI 兼容 HTTP 接口把请求发给 Envoy；
2. Envoy 通过 ext_proc 把请求头和请求体发给 EPP；
3. EPP 从请求体解析出模型名和 Prompt，在候选 Pod 里筛选、打分，选出一个（核心决策：A 有缓存但在排队 vs B 空闲但没缓存）；
4. EPP 把选中 Pod 的 IP:Port 写进请求头返回，Envoy 按该地址转发给 Pod 上的 vLLM；
5. vLLM 生成并以 SSE 流式返回，经 Envoy 传回客户端（响应同时流过 EPP 做记账，见 4.4）。

---

## 三、控制面：InferencePool API（源码解析）

### 3.1 定位：替代 Service 的 backend

`site-src/concepts/api-overview.md:24`：InferencePool 在 Gateway API 资源模型中是 **backend**——"把 HTTPRoute 中的 Service 替换成 InferencePool"；它有 Service 的形态（选 Pod + 指定端口），但额外提供路由扩展点与推理专属优化。

### 3.2 Spec 四字段（`api/v1/inferencepool_types.go:60-101`）

```go
type InferencePoolSpec struct {
    Selector          LabelSelector        // ① 池成员
    TargetPorts       []Port               // ② 1~8 个端口
    AppProtocol       AppProtocol          // ③ http | kubernetes.io/h2c
    EndpointPickerRef *EndpointPickerRef   // ④ EPP 参照物
}
```

1. **Selector**（`:61-70`）：只按标签、只限同命名空间。注释点明结构"刻意做得与 Service selector 兼容，因为某些实现可把它翻译成 Service 资源"——这是网关厂商低成本接入的机关。
2. **TargetPorts**（`:72-81`）：CEL 校验端口号唯一、1~8 个；**每个端口被 EPP 视为独立端点 `podIP:portNumber`**（`:73-74`）——天然支持一个 Pod 暴露多个服务端口（P/D 不同端口、gRPC/HTTP 并存）。
3. **AppProtocol**（`:83-94`）：默认 HTTP/1.1；`kubernetes.io/h2c`（明文 HTTP/2）典型用于 gRPC 类模型服务（提案 2162-grpc-support）。
4. **EndpointPickerRef**（`:129-174`）：默认 `Kind=Service` 且 CEL 规则强制 Service 必须带 port（`:128`）；**明确禁止 ExternalName Service**（`:142-147`，引用 CVE-2021-25740）。
   - **`FailureMode` 枚举 `FailOpen | FailClose`，API 层默认 `FailClose`**（`:168-189`）：EPP 无响应时，FailOpen = 网关按自己的方式挑端点转发（退化为普通 LB），FailClose = 直接丢请求。注意与 llm-d 的差异：笔记描述"llm-d 官方 chart 默认放行（FailOpen）"，那是产品层在 Envoy ext_proc 过滤器上配 `failure_mode_allow: true` 的选择；GAIE 规范默认更保守。
   - `endpointPickerRef` 现为**可选**（`:96-100`；v1.5.0 前必填），Status 备有 `EndpointPickerRefMissing` reason（`:311-320`）——部分实现允许不带 EPP 自行做负载均衡。

### 3.3 Status：Parents 模式（`:191-249`）

按 Gateway API 惯例组织：最多 32 个 parent（通常是 Gateway），每个 parent 各带 Conditions——`Accepted`（reason：`Accepted` / `NotSupportedByParent` / `HTTPRouteNotAccepted` / `EndpointPickerRefMissing`）、`ResolvedRefs`（`InvalidExtensionRef`，EPP 引用不可解析）、`Exported`（多集群导出：`Exported` / `NotRequested` / `NotSupported`）。

### 3.4 多集群：InferencePoolImport（`apix/v1alpha1`）

提案 1374 的落地：**由控制器生成（非用户手写）的集群本地镜像**，代表远端集群的 InferencePool，使本地流量可路由到远端模型服务器。

### 3.5 如何接入路由：HTTPRoute → InferencePool

GAIE 不修改 Gateway API 本体，允许 `backendRefs` 指向 `group: inference.networking.k8s.io, kind: InferencePool`；支持的网关控制器（从 `conformance/reports/` 各厂商目录可见：istio、agentgateway、nginx-gateway-fabric、kgateway、gke-gateway、ack-gateway、higress、envoy-ai-gateway、kubvernor）读取 `endpointPickerRef` 找到 EPP Service，生成 ext_proc 过滤器 + ORIGINAL_DST 集群配置；EPP 则 watch `selector` 选中的 Pod 作为候选列表。

**池化的本质价值**：让同一模型的所有业务流量共享一组 Pod，把"谁能用哪份资源"从部署时推迟到请求到达时——部署时只知道各业务可能需要多少，请求到了才知道此刻谁真的需要；固定分配要备各业务峰值之和，池化只需备总负载峰值（统计复用）。与传统无状态计算池的关键区别：推理池里各 Pod 带不同缓存/排队状态、处理同一请求代价不一样——选"有缓存但排队的 A"还是"空闲但重算的 B"是权衡，要做这个判断，EPP 必须实时掌握每个 Pod 的负载和缓存状态（第五章）。

---

## 四、数据面协议：ext_proc 如何衔接 Envoy 与 EPP

规范出处 `docs/proposals/004-endpoint-picker-protocol/README.md`（状态 Implemented，协议 v1.0.0 / 2025-07-29）。

### 4.1 协议六要点

1. **服务契约**：EPP **必须实现 Envoy `ExternalProcessor` gRPC 服务，且必须支持流式模式**——全双工通信支撑 token 级实时响应转发（SSE 长流的基础）。
2. **子集约束（数据面→EPP）**：网关可在 ext-proc 请求的 `filter_metadata` 里限定候选：外层键 `envoy.lb.subset_hint`、内层键 `x-gateway-destination-endpoint-subset`（值 `ip:port` 列表）。设了它，EPP **必须只从列表内选**；列表空或全不合格 → 必须返回 `ImmediateResponse` **503**（不许擅自扩大范围）。不设则从 InferencePool selector 全集选。
3. **选址结果（EPP→数据面）**：两条等价通道——HTTP 头 `x-gateway-destination-endpoint: <ip:port>`，或 ext-proc 响应 `dynamic_metadata` 的 `envoy.lb` 命名空间；**禁止两者填不同值**。头值支持**有序端点列表**（逗号分隔）：首个有效者为主，配 retry 顺延——即 fallback 语义（v0.4.0 引入）。换 InferencePool backend 重试时必须重新调用新 backend 的 EPP。错误语义：没有就绪端点→**503**；应丢弃的可 shed 请求（高负载下的低优先级）→**429**——这正是笔记第六章准入控制（429/503）的协议出口。
4. **served endpoint 回报（数据面→EPP，响应阶段）**：因为可能给了多个端点、网关按 retry 逐个尝试，Envoy 必须通过响应侧 `metadata_context` 回传 `envoy.lb` / `x-gateway-destination-endpoint-served`，EPP 据此知道**实际服务者**并准确记账。
5. **健康检查**：标准 gRPC Health Checking 三个服务——`liveness`（进程在即可）、`readiness`（**datastore 已同步完成 + 多副本时必须是选举出的 leader**）、`envoy.service.ext_proc.v3.ExternalProcessor`（同 readiness）。这给笔记的主备 Lease 选主、索引未同步不接流量提供了规范依据。
6. 命名空间选择原因：`envoy.lb` 是 Envoy subset load balancing 的预定义 host metadata 惯例，ORIGINAL_DST/subset 机制天然识别。

### 4.2 协议词表（`pkg/lwepp/metadata/consts.go`）

| 常量 | 键 | 方向与用途 |
|---|---|---|
| SubsetFilterNamespace/Key | `envoy.lb.subset_hint` / `x-gateway-destination-endpoint-subset` | 网关→EPP：限定候选 |
| DestinationEndpointKey | `x-gateway-destination-endpoint` | EPP→网关：选址结果 |
| DestinationEndpointServedKey | `x-gateway-destination-endpoint-served` | 网关→EPP（响应）：实际服务者 |
| FlowFairnessIDKey | `x-gateway-inference-fairness-id` | 请求：Flow Control 公平流标识 |
| ObjectiveKey | `x-gateway-inference-objective` | 请求：优先级目标 |
| ModelNameRewriteKey | `x-gateway-model-name-rewrite` | 请求：转发给模型服务时的模型名重写 |

注意最后三行：**llm-d 的 Flow Control / InferenceObjective / 模型名重写的线格式，已在本仓库的 API 组里定义**（ llm-d 用的头名 `x-llm-d-inference-fairness-id` 等是同一语义的实现侧变体；llm-d 还另有自己的扩展如 `x-llm-d-prefiller-host-port`）。

### 4.3 LWEPP：参考实现示范 ext_proc 状态机

`pkg/lwepp/README.md:3-10` 自述：最小实现，只做**协议正确性 + 轮询**选址，不含任何推理智能——但它是协议里最难的"ext_proc 流式状态机"的活文档。组件：`StreamingServer`（实现 `ExternalProcessor.Process`，每请求一条 gRPC 流，`handlers/server.go`）+ `Datastore` 接口（`PoolGet`/`PodList`，`:38-42`）+ `EndpointPicker` 接口（`:81-83`，`RoundRobinPicker` atomic 递增取模，`:86-102`）+ `controller/`（watch InferencePool 与 Pod 维护内存态）。

`Process()` 主循环（`server.go:126-340`）的四个工程细节：

1. **候选集构建**（`request.go:34-139`）：从 filter metadata 读 subset hint，**同时兼容逗号字符串与 JSON 数组两种形态**（`:54-56` 注释：声明式 HeaderToMetadata 填平字符串、编程式控制面填数组，不支持双形态会导致"静默 fail-open 路由到全部 Pod"）；区分"仅 IP（放行所有端口）"与"ip:port 精确匹配"（`:106-129`）；**设了子集就严格遵守，匹配不到留空候选→503**（`:130-133`），对应提交 `fix(lwepp): respond with 503 when no endpoint is eligible (#3084)`。
2. **body 处理与乱序防御**（`server.go:226-295`）：请求体上限 10MB（`:104`，超限 `ResourceExhausted`）；**headers 应答未发出前收到的 body 块先挂起**（`headersDeferred`，`:235-242`）——Envoy 把"先于 headers 应答的 body 应答"视为乱序、判为 spurious 并拆流；必须等 body `EndOfStream` 后才 `pickEndpoint`。这解释了 llm-d EPP 为什么"选址发生在读完整请求体之后"：前缀打分需要 Prompt 全文，且协议约束了应答顺序。
3. **选址响应三件套**（`server.go:182-215`）：`HeaderMutation` 写 `x-gateway-destination-endpoint` + **`ClearRouteCache: true`**——没有它，Envoy 会用请求进来时按 HTTPRoute 解析好的路由，写了新头也不会重走路由；清缓存后 ORIGINAL_DST 集群才按新地址建连 + `DynamicMetadata` 双通道同值（与协议"任选其一、不许冲突"一致）。
4. **失败语义**（`sendUnavailable`，`:106-124`）：用 `ImmediateResponse` 503 而非断 gRPC 流——注释解释：断流会让 fail-open 配置的网关把请求发到子集外的端点；**显式 503 才是带子集约束时的安全失败方式**。

### 4.4 llm-d EPP 在协议钩子上的实现

- **请求侧**：收到头与体后解析模型名/Prompt，进入决策（第六、八章）。防伪造：客户端自行设置的 `x-gateway-destination-endpoint` 会在所有过滤器执行前被删除（笔记 2.1）。
- **响应侧**：响应头体经 ext_proc 流过 EPP；结束时该 Pod 在途计数减一；若响应在首段数据之前就出错，**撤销这条请求在前缀索引里的登记**，避免重试被误判为缓存命中（笔记 2.1）。LWEPP 的 `response.go:30-79` 演示了同一钩子：提取 served endpoint 记结构化日志；生产 EPP 在此做记账。
- 测试通道（`pkg/lwepp/README.md:12-39`）：`test-epp-endpoint-selection` 头可限定候选到指定 IP（仅测试环境用）；`x-conformance-request-id` 关联 served-endpoint 观测日志供 conformance 测试轮询——这是"路由确实生效"的验证机关。

---

## 五、信息面：EPP 的两类知识

### 5.1 负载指标：提案 003 的标准化（`docs/proposals/003`，Partially implemented）

协议要求：模型服务器必须实现 OpenAI Completions/Chat。指标：**名称不必完全一致，但 Gauge 类型与语义必须符合**，源码表格直接给出四引擎映射：

| 标准指标 | vLLM | Triton TensorRT-LLM | SGLang |
|---|---|---|---|
| TotalQueuedRequests | `vllm:num_requests_waiting` | `nv_trt_llm_request_metrics{request_type=waiting}` | `sglang:num_queue_reqs` |
| TotalRunningRequests | `vllm:num_requests_running` | `{request_type=scheduled}` | `sglang:num_running_reqs` |
| KVCacheUtilization | `vllm:kv_cache_usage_perc` | `nv_trt_llm_kv_cache_block_metrics{fraction}` | `sglang:token_usage` |
| [可选] BlockSize | `vllm:cache_config_info{block_size}` | `{tokens_per}` | `cache_config_info{page_size}` |
| [可选] NumGPUBlocks | `vllm:cache_config_info{num_gpu_blocks}` | `{max}` | `cache_config_info{num_pages}` |

可选两个指标供 prefix cache scorer 使用，缺失时回退到 prefix 插件配置。LoRA 与前缀缓存：`:37-57` 要求支持 `model` 参数动态加载适配器并暴露 `vllm:lora_requests_info`（标签 `max_lora` / `running_lora_adapters` / `waiting_lora_adapters`）；`:59-64` 要求 SHOULD 支持前缀复用（如 vLLM APC），v0.4.0 起 EPP 支持前缀感知调度。`:9-10` 备注：可插拔架构（提案 0683）落地后此协议"按定义不可能再严格"——扩展方可自定义数据摄取，代价是引擎开箱移植性下降。

**实现侧**：EPP 每 50 毫秒抓取每个 Pod 的 `/metrics`，解析表中前三个值。

### 5.2 前缀缓存：两种准确度不同的方式

- **近似模式（默认）**：不向引擎查询，EPP 自维护前缀索引——按字符数估算 token 边界，把 Prompt 每 64 token 切块做**链式哈希**（每块哈希包含之前所有块内容，哈希相同 ⇒ 从开头到该块完全一致），登记在选中 Pod 名下按 LRU 淘汰；新请求算出哈希序列，查各 Pod 从第一块起连续命中多少块。零引擎配合。
  **根本落差**：登记的是"EPP 往哪个 Pod 派过哪些前缀"，不是"Pod 显存里此刻有什么"。显存淘汰、Pod 重启、多副本独立记账都会让索引与真实状态脱节。
- **精确模式**：vLLM 每存入/淘汰一个 KV 块经 **ZMQ** 发事件，EPP 为每个 Pod 维持订阅、同步块索引。准确性的代价：块事件频率是请求数的很多倍（CPU 开销）；EPP 必须用与引擎相同的 Token ID 与块大小重算哈希——不能再按字符估算，要引入真实分词开销。
- **选择**：单副本、显存充裕 → 近似足够；KV 占用长期接近 80%（淘汰频繁）或多副本未开同步 → 近似会系统性高估命中率，前缀长时每次误判 = 数千 token 重算，精确的开销就值得了。

---

## 六、调度决策：过滤、打分、挑选

### 6.1 EPP 分层架构（提案 0683，Implemented）

`:24-31` 定义 EPP 处理顺序：**Ext-Proc Server（不可扩展）→ Routing Layer → Flow Controller → Scheduling Layer**，纵向由 **Data Layer** 支撑所有层：

- **Ext-Proc Server**（`:70-75`）：协议定义过死，偏离会让 EPP 不可用或不稳定，禁止改动；
- **Data Layer**（`:48-52`）：消费 K8s 配置与模型服务器指标（提案 003），加工出资源稀缺度提示与 autoscaling 建议；
- **Routing Layer**（`:54-62`）：用户定义的按模型路由规则与请求增强（系统提示注入、RAG callout、请求校验、模型名→权重切分翻译）；API 保持极窄范围防"垃圾场化"；
- **Flow Controller**（`:64-66`）：消费资源稀缺数据，按 Priority/Fairness/Queueing 做入口准入——第八章的 llm-d Flow Control 即其实现；
- **Scheduling Layer**：基于池当前状态智能路由的负载均衡算法——下面的过滤/打分/挑选插件。

llm-d-router 的插件体系就是这套蓝图的落地：用哪些插件、权重多少，写在 EPP 的 `EndpointPickerConfig` 配置文件里。

### 6.2 过滤

默认由 `utilization-detector` 完成（自动加入每套调度配置，配置文件里看不到）：剔除指标缺失或超 200ms 未更新的 Pod、等待队列达到 5 的 Pod、KV Cache 占用达到 80% 的 Pod；若会剔除全部则保留全部候选。

### 6.3 打分与挑选（官方 chart 默认）

```yaml
schedulingProfiles:
- name: default
  plugins:
  - pluginRef: queue-scorer                # 等待队列越短分越高
    weight: 2
  - pluginRef: kv-cache-utilization-scorer # KV 占用越低分越高
    weight: 2
  - pluginRef: prefix-cache-scorer         # 前缀命中块越多分越高
    weight: 3
```

每个插件给 0~1 分，乘权重相加；未写 picker 时自动补 `max-score-picker`（取最高分，同分轮流）。默认队列分数按**请求条数**计算，不区分长短——"成本只能预知一半"在这里直接体现；llm-d 另提供按在途 token 数打分的 `token-load-scorer` 和按预测延迟打分的 `latency-scorer` 弥补。

### 6.4 算例与权衡本质

**算例**（新请求 = 2000 token 系统提示 + 200 token 问题）：A 队列 1 条、KV 60%、命中≈0.9；B 队列 0、KV 20%、无缓存；C 队列 4 条、KV 50%、无缓存。队列分按池内极值归一化：B=1、C=0、A=0.75。A 总分 0.75×2+0.4×2+0.9×3=**5.0**，B=1×2+0.8×2+0=**3.6**，C=1.0 → 发往 A。A 队列涨到 4：队列分 0、总分 3.5 < 3.6 → 改发 B（即使重算 2000 token）；涨到 5 → 过滤阶段直接出局，无论缓存命中多高。

**这个权衡的本质不是"TTFT vs TPOT"**。两项代价是：选 A 的**排队等待**与选 B 的 **Prefill 重算**——两者**都是 TTFT 的组成部分**（都发生在第一个 token 之前），Decode 阶段基本无差别。它考的是"**在两个都伤 TTFT 的坏选项里挑一个更小的**"。真正的 TTFT vs TPOT 冲突在另外两处：

1. **P/D 分离的动机**（第七章）：长 Prompt 的 Prefill 占满某步算力，同 Batch 正在 Decode 的请求集体等待、token 间隔拉长——牺牲一部分请求的 TTFT（Prefill 资源）还是牺牲另一部分请求的 TPOT（Decode 流畅度），解法是拆到不同 Pod；
2. **vLLM 引擎内部每步调度**（第八章）：`max_num_batched_tokens` 预算在"放行新请求降其 TTFT"与"保住运行中请求 TPOT/避免抢占重算"之间分配。

（路由例子里 TPOT 只有二阶影响：A 满载时 Batch 更大、每步略慢，但不是主轴。）

### 6.5 调权方法论

- 权重 2/2/3 表达"缓存命中足以抵消队列增到 3 条、不足以抵消 4 条"的换算，但**换算率不固定**：队列分按池内当前极值**相对归一化**——池内差距大时每多一条排队分数下降少，差距小时下降多。低负载的反直觉现象：池子都很空闲（队列 1/0/0）时，仅仅多 1 条排队就让该 Pod 分数从 1 骤降 0，微小负载差被放大、不必要地打断缓存亲和。
- 能确定的是方向：**前缀越长重算代价越高 → 缓存权重调高；输出越长每条排队请求占用 Pod 时间越长 → 队列权重调高**。
- **调完观察 TTFT P99，不是缓存命中率**：命中率是局部效率指标；若命中率升而 P99 变差，说明缓存权重给多了、请求向少数 Pod 集中——把共享推理池重新切成事实上的"静态粘性孤岛"，**吞噬池化带来的统计复用红利**。

### 6.6 正反馈与抑制

前缀打分 + 取最高分形成正反馈：命中缓存的 Pod 高分 → 收更多同前缀请求 → 前缀一直留在它显存里。近似索引还会放大它：热点 Pod 派发记录最多、淘汰最频繁，而索引看不到淘汰——**最容易高估命中的恰是最不该继续收请求的 Pod**。抑制机制：队列变长/显存升高使另两项分数下降，队列达 5 被过滤——与 CDN"**带负载上限的一致性哈希**"同类（优先发到已缓存节点、超上限再换）；区别在 llm-d 用加权打分代替哈希、负载上限由过滤阈值给出。另有 `prefix-cache-affinity-filter`：按概率把候选缩小到高命中 Pod，同时比较这些 Pod 与其余 Pod 的 TTFT，差距过大时不再缩小候选。

---

## 七、Prefill/Decode 分离（P/D）

### 7.1 动机

一次推理两阶段对硬件需求截然不同：Prefill 把整段 Prompt 一次算成 KV Cache，计算密集，Prompt 越长耗时越长；Decode 逐步生成，每步计算量小、受显存带宽限制，但一条请求要执行数百到上千步。同一 Pod 混部时，长 Prompt 的 Prefill 会占满某一步计算，同 Batch 正在 Decode 的请求在这一步集体等待——即 6.4 说的 TTFT/TPOT 冲突。分离后 Prefill Pod 与 Decode Pod 各自按瓶颈选硬件、配并行度、扩缩容。

### 7.2 选址：先 Decode 后 Prefill

配置里两套 scheduling profile（各带标签过滤）。**先选 Decode**：一轮生成结束后，该请求的 KV Cache 作为前缀缓存留在 Decode Pod 上，同会话下一轮以上一轮全部内容为前缀——**缓存亲和主要沉淀在 Decode 侧**。选定 Decode 后判断要不要远程 Prefill：检查 Prompt 在该 Decode Pod 上未命中的 token 数，少于阈值（`prefix-based-pd-decider` 的 `nonCachedTokens`）就本地补算——P/D 分离本质是**用网络通信换计算解耦**，只补几十个 token 时网络序列化/传输的固定开销超过计算耗时，本地顺带算完反而更快。

### 7.3 执行链路：Sidecar 接力 + NIXL 拉取

EPP 把 Decode Pod 地址作为转发目标返回 Envoy，同时注入头 `x-llm-d-prefiller-host-port`（Prefill Pod 地址）。Envoy 把请求发到 Decode Pod，接收方是 Pod 里的 **Sidecar**（独立代理容器，经 localhost 调用本地 Decode vLLM）：

1. Sidecar 把 `max_tokens` 改成 1、在 `kv_transfer_params` 标明只执行 Prefill，发给 Prefill Pod 的 vLLM；
2. Prefill 算完，在响应 `kv_transfer_params` 里返回自己的地址 + KV Cache 块编号；
3. Sidecar 把这组参数放进原始请求，发给本地 Decode vLLM；
4. Decode 按块编号用 **NIXL**（NVIDIA 推理数据传输库，通常走 RDMA + GPUDirect）**直接从 Prefill Pod 显存读取 KV Cache**，开始生成。

**控制面与数据面分离**：Sidecar 只转发轻量的 KV 位置元数据，动辄数 GB 的 KV 张量完全不经过 Sidecar 和 CPU——避免数据面代理成为吞吐瓶颈；无 RDMA 时 NIXL 回退 TCP，性能差距大，官方仅建议开发测试环境用。头里没有 Prefill 地址时 Sidecar 直接转本地 vLLM；Prefill 返回 5xx 时也回退此路径。协调逻辑全放 Decode 侧、让 Prefill 无状态的两层好处：Prefill Pod 可像无状态服务弹性扩缩；Decode Pod 作为请求生命周期的最终持有者，远程 Prefill 异常/超时时本地兜底，**单请求无缝降级**而不暴露错误给上游。

三者分工：**EPP 选出 Prefill 和 Decode 两个 Pod，Envoy 把请求送到 Decode Pod，Sidecar 让两阶段按正确顺序接力。**

GAIE 源码侧的呼应：InferencePool 的 TargetPorts"每端口独立端点"语义（3.2-②）支撑一 Pod 多服务端口；`x-llm-d-prefiller-host-port` 是 llm-d 在标准选址头之外的扩展。

---

## 八、两层队列：准入控制与延迟绑定

### 8.1 第一层：EPP 入口队列（Flow Control，默认关闭）

（笔记 2.6）饱和度计算无论如何都进行：每 Pod 取 max(等待队列/5, KV 占用/0.8)，池取均值。

- **关闭时**：不排队；饱和时优先级为负的请求直接拒绝（协议 429，见 4.1-③），其余照常转发、到 vLLM 里排队。
- **开启时**：饱和时新请求留在 EPP 内存队列——排队期间 ext_proc 调用不返回，Envoy 持有请求持续等待（chart 把 Envoy 等 EPP 回复的超时设为 1000 秒，就是为排队留时间）。**网关与 EPP 共同构成"请求缓冲池"，链路超时预算必须对齐**：上游客户端超时要长于排队上限，且客户端断开后 EPP 须及时感知取消排队，否则把无人认领的请求放行给 GPU 是浪费算力。
- 队列组织两个维度：**优先级**（InferenceObjective——llm-d 自定义 CRD，工作负载负责人用它声明一类流量的优先级并绑定池；请求经 `x-llm-d-inference-objective` 头声明归属）× **流**（通常对应租户，`x-llm-d-inference-fairness-id` 头标识，无此头归入默认流）。两个头应由可信上游网关/鉴权层写入并删除客户端自带值。取请求：先选"优先级最高且有排队"的一档，档内按公平策略选流，流内默认 FCFS。公平策略示例：同优先级下租户甲积压 100 条、乙只有 1 条，轮询下乙不必排在甲的 100 条之后。
- 出口：队列满或排队超时 → 客户端 429；池内无任何可用 Pod → 503。

### 8.2 第二层：vLLM 执行队列

请求到 vLLM 后默认 FCFS；引擎每执行一步做一次调度：先给运行中请求分配这步的 token 预算和新 KV 块，剩余预算/空闲块分给等待中的请求（步预算上限 `max_num_batched_tokens`）。若这步为释放 KV 块发生了抢占，等待中的新请求这步都不被调度。P/D 时 Decode 请求还要等 KV 传输完成才参与下一步调度，能否进 Batch 仍取决于预算。

### 8.3 为什么两层：延迟绑定

在哪层排队后果不同：请求一旦转发给某 Pod，只能在该 Pod 队列等待，即使别的 Pod 先空闲也无法处理；留在 EPP 的请求尚未绑定 Pod，可分配给**最先出现空闲容量的 Pod**，高优先级也能优先取出。这是池化思路"再往后推一步"：**选 Pod 的时机从请求到达时推迟到有容量的那一刻，决策所用信息也更新**。低负载时调度追求缓存亲和；高饱和时排队等待耗时远超重算前缀的代价，延迟绑定把"尽快找到空闲算力"置于亲和之上——契合不同负载区间的核心矛盾。

### 8.4 队列深度的权衡

开启 Flow Control 后，饱和阈值实际决定每个 Pod 引擎队列保持多深。EPP 要等下一次指标抓取（50ms 后）才能确认某 Pod 有空余，这段反馈延迟里引擎队列必须留够请求喂 GPU，否则 GPU 空转到下一批放行。太浅 GPU 出现空闲；太深请求过早绑定 Pod，延迟绑定好处消失。**与网络按带宽时延积（BDP）设缓冲区同理：缓冲区至少覆盖一个反馈周期的处理量。** 另一选择：`concurrency-detector` 直接用 EPP 自计在途数，不依赖指标抓取、反馈更快，但要预给每 Pod 设并发上限——每条请求 token 长度差异巨大（第一章），静态条数上限难在"防长请求打爆显存"与"让短请求吃满算力"之间两全。

---

## 九、设计取舍

把调度从网关拆出来，换来调度策略独立演进 + 插件化扩展（加一个打分插件不改核心代码、不发版网关），也带来两类代价：**位置**（EPP 在每条请求转发路径上）与**信息**（EPP 对引擎状态的了解全是间接的）。

### 9.1 EPP 位于请求路径上

- **性能**：每条请求多一次 gRPC 往返；请求体**完整**经过 EPP——为匹配前缀缓存必须吞吐并切块哈希完整 Prompt，打破了传统代理"控制面只看头不碰体"的轻量假设，**CPU 消耗不再只随 QPS 增长而是与 Token 吞吐量挂钩**。官方容量文档：低吞吐下把前缀匹配上限从 16384 token 调到 40 万 token，EPP CPU 用量翻倍以上。EPP 处理能力是入口容量上限之一，要像对待网关一样规划资源。
- **故障**：内存队列在进程正常退出时给排队请求返回可重试 503，崩溃时连接直接断开——客户端或上游网关需具备重试能力。EPP 不可用时的行为由失败策略决定（规范层默认 FailClose，见 3.2-④；llm-d chart 选择 Fail-open：Gateway 模式绕过 EPP 直发池内就绪 Pod，Standalone 模式 Envoy 切备用路由按最少请求数分配——服务还在，但缓存亲和、负载感知、排队全没了，退化成普通负载均衡）。主备模式下基于 K8s Lease 选主（对应 4.1-⑤ readiness 的 leader 要求），主副本故障后从租约过期到备副本接手约 10—30 秒，期间走放行路径。
- **切换后的隐性代价**：近似索引只在内存，备副本接手时是空的——索引重新积累前前缀打分对所有 Pod 为 0，请求只按负载分配，同一前缀分散到多个 Pod：各 Pod 命中率一起下降、TTFT 拉高，且同一份公共前缀在多 Pod 显存里**重复计算和存储，变相压缩集群有效 KV Cache 容量**（进程重启同理）。精确模式可订阅时从 vLLM 拉快照恢复索引，目前实验性。
- 要避免切换空窗就要多副本同时工作 → 引出下一节。

### 9.2 状态偏差的三个来源

- **时间：指标延迟**。负载数据最多滞后 50ms，决策持续进行——某 Pod 刚空闲，下一次抓取前 EPP 一直认为它负载最低；突发流量下这段时间到达的请求扎堆发给它。根源是取最高分的**确定性**：两次抓取之间所有请求读到同一份指标、算出同一个最高分。负载均衡领域早有结论（Mitzenmacher 2000《How Useful Is Old Information?》）：信息越陈旧，"总选看上去最闲的节点"越容易扎堆，适当引入随机更稳。llm-d 提供 `weighted-random-picker`（分数当概率抽 Pod，官方 chart 启用延迟预测时默认使用）——以少量缓存命中换更均匀分布。针对滞后本身：`inflight-load-producer` 让 EPP 内部计数（派出一请求对应 Pod 加一、响应结束减一），计数大于指标里等待+运行数时把差值补进队列长度——**仅单副本成立**（多副本各自只见自己派发的请求）。
  滞后成疾即**失败模式反转**：超 200ms 未更新的 Pod 被当作完全饱和（默认值）；EPP 整体崩溃时网关 Fail-open 放行，但 EPP 进程健康、仅因网络策略配置错误抓不到任何指标时，全池判饱和 + Flow Control 开启 → EPP 转 **Fail-closed**，队列不再取出、请求陆续超时，而模型服务其实完全空闲。对策：监控 `flow_control_stale_endpoints` 指标；或把 utilization-detector 的 `stalenessPolicy` 改 `ignore`（过期 Pod 不计饱和度，代价是抓取全断时完全失去背压）。
- **语义：抽象泄漏**。队列长度这一指标在 P/D 池里含义变了：Sidecar 已把请求交给 Decode vLLM、但 KV 还在从 Prefill Pod 传输——请求暂时进不了 Batch，vLLM 却把这段时间计入等待队列。这些请求等的是**网络传输**而非 Decode 负载：一个只跑少量序列的 Decode Pod 若恰有 5 条请求在等 KV，会被默认过滤规则当成饱和剔除，还拉高全池饱和度。修复：vLLM ≥ v0.32 提供按传输阶段（未开始/进行中/完成）统计的请求数，EPP 把"正在传输"从队列长度减掉；"未开始"和"已完成在等 Batch"仍算排队（真实积压）。**只减队列长度、不减 KV 占用**——这些请求在开始拉取前就已分配好接收 KV 的显存块，不算排队却实实在在占显存，KV 占用计入是对的。教训：调度器读的每个指标，语义由推理引擎定义；引擎演进出新执行阶段时监控接口的字段名和类型都不变、不报错，但数字背后的物理含义已偏移——**上线新部署形态时要逐个确认所读指标是否仍保持原义**。
- **副本：多副本状态**。EPP 不少状态只在本地内存：Flow Control 队列与公平统计每副本一份（优先级/公平只在各自分到的流量里成立，队列上限也按副本分别计算）；近似前缀索引每副本一份（命中率随副本数下降）。缓解：**Redis-syncer** 插件经 Redis 共享在途请求数与最近登记的前缀哈希；为不阻塞转发主路径只能做成最终一致（1—2 个同步周期延迟），也不同步排队状态——又回到"陈旧信息"范畴。**调度层面的经典三角权衡：微秒级转发延迟、全局严格的公平性与缓存状态、无状态水平扩展，三者难以兼得。** 副本数要与所用功能一起决定：依赖严格优先级/公平 → 单活跃副本 + 主备（代价是切换空窗）；追求吞吐、策略主要依赖后端指标 → 多副本同时工作（代价是命中率与公平性下降）。

---

## 十、GAIE 契约 ↔ llm-d 实现对照表

| 契约层 | GAIE 源码位置 | 规范内容 | llm-d-router 实现 |
|---|---|---|---|
| 控制面 API | `api/v1/inferencepool_types.go` | InferencePool：selector/targetPorts/EPP 引用/FailureMode（默认 FailClose） | EPP watch selector 得候选；chart 选 FailOpen + 备用路由 |
| 数据面协议 | `docs/proposals/004` + `pkg/lwepp/metadata/consts.go` | subset 约束、`x-gateway-destination-endpoint`、served 回报、503/429、gRPC 健康检查（readiness=同步完成+leader） | 同协议；扩展 `x-llm-d-prefiller-host-port`、`x-llm-d-inference-*` 头 |
| 引擎侧协议 | `docs/proposals/003` | 三 Gauge 指标语义（四引擎映射）+ LoRA + 前缀缓存 SHOULD | 50ms 抓 `/metrics`；精确模式 ZMQ 块事件 |
| EPP 架构 | `docs/proposals/0683` | extproc 锁死 + Routing/Flow/Scheduling/Data 四层 | 过滤-打分-挑选插件 + Flow Control（优先级×流） |
| ext_proc 状态机 | `pkg/lwepp/handlers/server.go` | 应答顺序/10MB 闸门/503 vs 断流/子集严格 503 | 轮询换成加权打分；机制相同 |
| 一致性测试 | `conformance/` | 各网关厂商通过报告 | llm-d Router 为 conformant 生产实现 |

---

## 十一、一句话总结

llm-d Router 的本质是：**在通用网关（Envoy）之外引入一个"看得见引擎内部状态"的决策服务（EPP）——规范上它严格实现 GAIE 定义的 InferencePool CRD 与 ext_proc 数据面协议（subset 约束、选址回报、503/429、健康检查），实现上用插件化的过滤-打分-挑选做缓存亲和与负载感知的统一权衡；再通过 Sidecar + NIXL 实现 P/D 分离的控制面/数据面解耦、通过 Flow Control 实现延迟绑定与多租户公平。** 而其全部代价集中在一点：**EPP 位于请求必经路径上，且它掌握的一切引擎状态都是间接的——时间（指标滞后与扎堆）、语义（抽象泄漏）、副本（状态分片）三个维度的偏差不可避免。** 理解了 GAIE 的"协议/实现分离"，也就理解了为什么 llm-d 的智能调度可以插进任何 conformant 网关，而各家网关（istio、agentgateway、NGINX Gateway Fabric、kgateway、GKE/ACK Gateway、Higress 等）只需按规范转发。
