# 学习笔记
## 作业
**在work目录下哦😉**
## 微服务可用性设计
### 隔离
- **本质是对系统或资源进行分割, 保障某个服务出现故障后不会影响其它服务**
#### 服务隔离
- 动静分离
    - 由于cpu的cacheline产生的false sharing(伪共享)
    - 将数据库的表中,频繁更新与几乎不更新的字段拆表(隔离动静表)
    - cdn静态资源缓存加速、边缘计算
- 读写分离
    - 主从
    - Replicaset
    - CQRS

#### 轻重隔离
- 核心隔离
    - 按照业务的Level进行资源划分
    - 不同的level是否有地域区分或者备份多少
- 快慢隔离
    - 当有洪流数据, 服务无法及时处理, 会造成数据反压, 将数据利用label划分到不同的服务
- 热点隔离
    - 小表广播: 少量数据可以从remoteCache变成localCache
    - 对必要的服务页监控在线访问人数,达到top级别时将redis数据变更为localCache

#### 物理隔离
- 线程隔离
    - Fail Fast 当某一个业务无法请求时(线程池耗尽), 快速返回错误 做降级处理
    - 线程池隔离, 信号量隔离(不推荐)
- 进程隔离
    - docker, k8s, 弹性公有云
- 集群隔离
    -异地多活
- 机房隔离

#### Case
- 转码服务被超大视频攻击, 导致服务被大量阻塞 - 将一个转码服务划分为多个针对不同量级视频的转码服务
- 入口Nginx(SLB)故障, 影响全机房流量入口故障 - 使用多入口分流
- 缩略图服务被大图实时缩略耗尽cpu, 导致小图缩略被丢弃 - 将gif略缩服务单独隔离出来
- 数据库实例cgroup未隔离, 导致大SQL慢差距引起集体故障 - 使用cgroup将不同的数据库实现资源隔离
- INFO日志量过大, 导致ERROR日志采集异常 - 将INFO日志与ERROR日志分开

### 超时控制
- 本质上是fail fast, 不希望傻等
- 网络传递具有不确定性
- 客户端和服务端不一致的超时策略导致资源浪费
  - 依赖的微服务的超时策略发生变化时依赖者不知道, 导致依赖者出现超时 - 定义SLO, 配置默认值保护
- 默认值策略
- 高延迟服务导致client浪费资源等待, 使用超时传递: 进程间传递 + 跨进程传递
  - 进程内超时控制 - 在请求的每个阶段开始时检查是否有足够的剩余时间来处理请求, 以及继承他的超时策略
  - 服务间超时控制 - grpc天然支持, 依赖grpc的header的grpc-timeout
- 双峰分布
  - 监控超时不看mean, 看95th和99th, 设置合理超时, 拒绝超长请求, 当Server不可用时主动失败
- Case
  - SLB入口Nginx没配置超时导致连锁故障
  - 服务依赖的DB连接池漏配超时, 导致请求阻塞, 最终服务集体OOM
  - 下游服务发版耗时增加, 而上游服务配置超时过短, 导致上游请求失败
  
### 过载保护
#### [令牌桶](https://pkg.go.dev/golang.org/x/time/rate)
- 流入固定, 请求爆发
- 单机限流
#### 漏桶
- 流入任意, 超出固定, 流出固定
- 一样是单机
#### VS
- 漏桶和令牌桶都是设定指标, 但指标/阈值不好配置, 都是被动设置阈值
- 在增加及其或者减少机器时候阈值都得重新设置
- 人力运维成本过高
- 当流量高峰被限流时, 如果重新设置阈值, 早已过了时间
- **需要自适应的限流**

#### 利特尔法则
- $ L = \lambda W \ -> QPS * latency \:time $
- 计算系统的吞吐量, 作为限流的阈值进行流量控制, 得以达到系统保护
- 当服务器临近过载时, 主动抛弃一定量的负载, 目标是自
- 利用系统资源的信号量来估算系统吞吐 - 这里选择的是cpu, 如果选择内存的话, 由于内存的增长
  会频繁触发gc
- 可控延迟算法: tcp的bbr算法、vegas、CoDel - 以90%的cpu负载计算吞吐, 当高于吞吐抛弃、低于放行
- **如何计算接近峰值时的系统吞吐**
  - cpu的使用率: 使用一个独立的线程采样, 每隔250ms触发一次, 在计算均值时, 使用简单滑动平均除噪
  - inflight: QPS
  - Pass&RT: 针对最近五秒, 为每100ms采样窗口的成功请求数量(Pass), 计算出平均相应时间(RT)
  - 使用CPU的滑动均值(CPU>800), 过高则触发过载保护, (pass * rt) < inflight
  - 由于限流生效后, cpu会在临界值附近抖动, 所以使用一个冷却时间, 将限流时间持续1～2秒, 当
    冷却时间结束后再进行一次判断, 反复如此
  
### 限流
- 所有的可用性问题一定不是单点解决的, 一定是立体式防御, 一定是从客户端到服务端两个纬度去做方案
- 限流是指一段时间内, 定义某个客户或应用可以接收或处理多个请求的技术, 可以过滤掉产生峰值的客户和微服务
- 令牌桶、漏桶 针对当个节点, 无法分布式限流
- QPS 限流
  - 不同的请求可能需要数量不一的资源来处理
  - 静态QPS限流不准
  - ps.通过计算每qps的cpu成本来限流
- 给每个用户设置限制
  - 全局过载时针对异常控制, 比如异常账号操作
  - 一定程度的"超卖"配额
- 按照优先级丢弃
- 拒绝请求也需要成本

#### 分布式限流
- 使用redis限流
  - 单个大流量接口, 使用redis容易产生热点
  - pre-request模式对性能有一定影响, 高频的网络往返
  - 从获取单个quota升级成批量quota - 异步批量获取quota可以大幅度减少redis的请求频次
  - 但申请的配额需要手动设定静态值, 缺乏灵活 - 利用历史窗口的数据自动修改quota的请求数量
- 如何分配资源 - Max-Min Fairness
  - 多维度资源分配 - DRF  
- 单机限流 vs 动态限流 vs 全局限流
  - (实现简单、稳定可靠、性能高) vs (根据服务情况 动态限流、不用调整额度) vs (流量不均不会误触限流、
    机器数变动时无需调整、应用场景丰富 接口db等任何资源都可以使用)
  - ....
  - .....

#### 重要性
- 根据不同接口的不同重要性设定quota
  - 严重+
  - 严重
  - 可丢弃+
  - 可丢弃
- 全局配额不足时, 优先拒绝优先级低的

#### 熔断
- 熔断 - 客户端限流
- 为了限制操作的持续时间
- 当某个用户超过资源配额时, 后端任务会快速拒绝请求, 返回配额不足的错误, 
  但是拒绝回复仍会消耗一定资源, 有可能因为不断拒绝请求而导致过载
- max(0, (requests - K * accepts) / (requests + 1))

#### 客户端流控
- positive feedback
- 客户端需要限制请求频次
- 可以通过接口级别的error_details, 挂载到每个api返回的相应里
- 本地有默认值, 如果response有就用res的值

#### Gutter
- 基于熔断的gutter kafka, 用于接管自动修复系统运行过程中的负载
- 主集群 -> 抛弃流量 -> gutter集群 -> 接受不住 -> 主集群

#### Case Study
- 二级缓存穿透、大量回源导致的核心服务故障
- 异常客户端引起的服务故障(query of death)
  - 请求放大
  - 资源数放大
- 用户重试导致的大面积故障  

### 降级
- 丢弃不重要的请求, 提供一个降级的服务, 对某几个服务可进行空回复
  - 基于cpu、错误的降级回复, 回复一些mock值
  - 进入降级时, 不反悔一个复杂数据, 而是从一些缓存中捞取或者直接空回复
  - 降级一般在bff或者gate way层做, 防止缓存污染
  - 降级在意外流量或者意外负载时候触发
  - 降级在不定时演练, 保证功能可用
- 降级的本质: 提供有损服务
  - ui模块化, 非核心模块降级
    - BFF层聚合API, 模块降级
  - 页面上一次缓存副本
  - 默认值、热门推荐等。
  - 流量拦截+定期数据缓存(过期副本策略)
  - 页面降级、延迟服务、写/读降级、缓存降级(local cache)
  - 抛异常、返回约定协议、Mock数据、Fallback处理
#### Case Study
- 客户端解析协议失败, APP奔溃
- 客户端部分协议不兼容, 导致页面失败
- local cache 数据源缓存, 发版失效+依赖接口故障, 引起的白屏 - 备份remote cache避免白屏
- 没有playbook, 导致的MTTR上升

### 重试
- 当请求返回错误(例: 配额不足、超时、内部错误等), 对于backend部分节点过载的情况下, 
  倾向于立刻重试, 但是需要留意重试带来的流量放大
  - 限制重试次数(内网中一般不超过两次)和基于重试分布的策略(重试比例10%)
  - 随机化、指数型增长的重试周期: exponential ackoff + jitter
  - client侧记录重试次数直方图, 传递到server, 进行分布判定, 交由server判定拒绝
  - 只应该在失败这层重试, 当重试仍然失败, 全局约定错误码"过载, 无需重试", 避免级联重试
#### Case Study
- Nginx upstream retry过大, 导致服务雪崩
- 业务不幂等, 导致的重试, 数据重复 - (写请求不重试)
- 多层重试传递, 放大流量引起雪崩

### 负载均衡
- 某个服务的负载会完全均匀的分发给所有后端任务, 最忙和最不忙的节点永远消耗同样数量的CPU
  - 均衡的流量分发
  - 可靠的识别异常节点
  - scale-out, 增加同质节点扩容
  - 减少错误, 提高可用性 (N+2冗余)
- backend之间的load差异比较大
  - 每个请求的处理成本不同
  - 物理机环境的差异
    - 服务器很难强同质性
    - 存在内存资源争用(内存缓存、带宽、IO等)
  - 性能因素
    - FullGC
    - JVM JIT
  - JSQ(最闲轮训)
    - 缺乏服务端全局视图, 目标: 需要综合考虑 负载+可用性
  - the choice-of-2
    - 选择backend: CPU, client: health, inflight, latency 作为指标, 使用一个简单的线性方程进行打分
    - 对新启动的节点使用常量惩罚值(penalty), 以及使用探针方式最小化放量, 进行预热
    - 打分较低的节点, 避免进入"永久黑名单"而无法恢复, 使用统计衰减的方式, 让节点指标逐渐恢复到初始状态
    - 指标计算结合moving average, 使用时间衰减, 计算vt = v(t-1) * b + at * (1-b), b为若干次幂的倒数即: 
      Math.Exp((-span) / 600ms)
      
### 最佳实践
- 变更管理
  - 出问题先恢复可用代码
- 避免过载
  - 过载保护、流量调度等
- 依赖管理
  - 任何依赖都可能故障, 做chaos monkey testing, 注入故障测试
- 优雅降级
  - 有损服务, 避免核心链路依赖故障
- 重试退避
  - 退让算法, 冻结时间, API retry detail控制策略
- 超时控制
  - 进程内 + 服务间 超时控制
- 极限压测 + 故障演练
- 扩容 + 重启 + 消除有害流量