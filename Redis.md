- **Redis 8.0**（2025 年 5 月 GA）是当前主线版本：把原来的 Redis Stack 模块（JSON、TimeSeries、Search、概率数据结构等）全部并入了内核，社区版改名为 **Redis Open Source**；新增原生 **vector set（向量集合，beta）**、JSON、TimeSeries，以及 **Bloom / Cuckoo / Count-min sketch / Top-k / t-digest** 五种概率结构，还加入了 `HGETEX / HSETEX / HGETDEL` 等新命令。[cite:3e714445-1][cite:3e714445-2]
- **许可证变化**：Redis 7.4（2024 年）起从 BSD-3 改为 **RSALv2 + SSPLv1 双许可**（源码可用但非 OSI 开源）；Redis 8 额外提供了 **AGPLv3** 选项，同时把"Community Edition"更名为"Open Source"。[cite:3e714445-1][cite:b77f1c11-2]
- **"单线程"要说得精确**：Redis **执行命令仍是单线程**，但从 6.0 起**网络读写/解析已多线程**（`io-threads`）；Redis 8 重写了这套 I/O 线程实现，默认 `io-threads=1`，开到 8 可提升约 112% 吞吐。[cite:3e714445-1]

---

## 1、什么是Redis?有哪些有优缺点？

**答案：**

**Redis 是什么？**

Redis（**RE**mote **DI**ctionary **S**erver，远程字典服务器）是一个**基于内存、以键值对（Key-Value）为核心**的数据结构存储系统。它由 Salvatore Sanfilippo（antirez）于 2009 年开发，用 C 语言编写，常被同时当作：**数据库**、**缓存**、**消息中间件** 三种角色来用。

通俗理解：你平时操作数据库（MySQL）是把数据放在**硬盘**上，速度慢、要写 SQL；Redis 是把数据放在**内存**里，读写速度比硬盘快几个数量级，并且内置了一堆好用的数据结构（字符串、列表、哈希、集合、有序集合等），开箱即用。所以它特别适合做"**扛高并发、放热点数据**"的组件。

> 版本现状（避免旧认知）：目前主线是 **Redis 8.0**；请不要再按"Redis 只有 5 种数据类型""Redis 只能当缓存"这种 3.x/4.x 时代的印象来理解它。

**优点：**

1. **性能极快**：纯内存操作，单机读吞吐可达十万级 QPS，命令延迟通常是微秒~亚毫秒级（Redis 8 又把大量命令延迟降低了最高 87%）。[cite:3e714445-1]
2. **数据类型丰富**：String / List / Hash / Set / ZSet 五大基础类型，外加 Bitmap、HyperLogLog、Geo、Stream；Redis 8 更是原生内置 JSON、TimeSeries、向量集合、布隆过滤器等。
3. **支持持久化**：可以把内存数据存到磁盘（RDB/AOF/混合），重启后恢复，不只依赖内存。
4. **支持过期时间（TTL）**：天然适合做缓存。
5. **单命令原子性 + Lua 脚本**：适合做分布式锁、计数器等需要原子操作的场景。
6. **高可用与扩展性**：主从复制、Sentinel 哨兵、Cluster 集群，可水平扩展。
7. **生态极其成熟**：几乎所有语言都有官方/成熟客户端（C、C++、Java、Go、Python、Node 等），你作为 C++ 工程师可以用 hiredis、redis-plus-plus。
8. **功能丰富**：发布订阅（Pub/Sub）、事务（MULTI/EXEC/WATCH）、ACL 权限控制、客户端缓存、键空间通知等。

**缺点：**

1. **数据受内存容量限制**：数据全在内存，内存比硬盘贵，数据量巨大时成本高；超出物理内存必须靠集群分片。
2. **单线程命令执行**：虽然快，但**单个 CPU 密集命令**（如 `KEYS`、超大 `SORT`）会阻塞整个实例；**big key**（超大 value、超长 list）会拖慢一切。生产环境要刻意避免。
3. **持久化有数据丢失窗口**：RDB 快照间隔期间、AOF `everysec` 模式下最多丢 1 秒数据，做不到"零丢失"。
4. **弱一致/最终一致**：主从复制是**异步**的，主节点挂了、从节点顶上时可能丢掉一小部分写操作。
5. **事务语义与传统数据库不同**：没有回滚机制（详见第 10 题）。
6. **查询能力有限**：不是关系型数据库，没有 SQL、没有复杂关联查询；查询逻辑要在应用层拼。
7. **默认不安全**：默认无认证、无加密，生产必须手动配置 `requirepass` / ACL / TLS。
8. **许可证风险（新）**：从 7.4 起不再纯 BSD 开源，改为了 RSALv2+SSPLv1 双许可（8.0 增加 AGPLv3 选项）——**自研托管 Redis、改代码后再对外提供托管服务**等场景需要评估许可约束。[cite:b77f1c11-2][cite:3e714445-1]

---

## 2、为什么要用 Redis / 为什么要用缓存

**答案：**

**一句话**：因为"读多写少"的热点数据直接打到数据库，数据库扛不住；在数据库前面放一层内存缓存，绝大多数请求在缓存层就返回了。

**具体原因拆解：**

1. **数据库是慢设备，缓存是快设备**：数据库的数据在磁盘上（毫秒级），Redis 在内存里（微秒级）。访问速度差约 100~1000 倍。把热点数据放进 Redis，用户请求走"快车道"。
2. **扛高并发 / 削峰填谷**：秒杀、抢购、热点新闻这类流量瞬间暴涨的场景，如果全打 MySQL，连接数、磁盘 I/O 立刻被打爆；用 Redis 在前面挡流量，数据库只承受"少量真正该落库"的请求。
3. **降低数据库压力、保护后端**：数据库承载的压力小了，就不会因为 I/O 瓶颈拖垮整个系统。
4. **提升用户体验**：响应时间从几十毫秒降到几毫秒，页面更快。
5. **数据共享 / 解耦**：多个服务（比如订单、用户、推荐服务）可以共享同一份热点数据，不用各自查库。
6. **天然的多用途**：除了缓存，Redis 还常被用作：
   - 分布式锁（`SET NX EX`）
   - 计数器（`INCR` 点赞、浏览数）
   - 排行榜（ZSet）
   - 会话 Session 存储（Redis 8 对 Hash 的会话场景有专门优化）
   - 消息队列/流（List、Stream）
   - 限流（滑动窗口、令牌桶）

**必须知道的反面（新增成本）：**

引入缓存也带来三个经典问题，面试一定会追问：

- **缓存穿透**（见第 12 题）
- **缓存雪崩**（见第 13 题）
- **缓存与数据库双写一致性**：比如"先更新 DB 还是先更新缓存"，顺序错了会出现脏数据。常用的有 **Cache-Aside（旁路缓存）**：读时先查缓存，未命中查 DB 再回填；写时先写 DB 再删缓存（延迟双删）。这是标准做法，能接受短暂不一致。

**结论**：当系统"读远多于写、有热点数据、要扛高并发"时，用 Redis 缓存几乎是最主流、最经济的方案。

---

## 3、Redis为什么这么快

**答案：**

很多人背答案只背"单线程"，其实 Redis 快是**一整套机制**叠加的结果。分五层讲：

**1. 数据全在内存（最主要原因）**
内存访问是纳秒~微秒级，磁盘是毫秒级。这是数量级上的差距，一切其它优化都建立在这个基础上。

**2. 单线程执行命令，避免锁竞争和上下文切换**
命令执行由一个线程串行处理，天然**不需要加锁、没有线程切换、没有数据竞争**。注意精确说法：
- **执行命令**：单线程；
- **网络 I/O（读 socket、写 socket、解析协议）**：从 6.0 起改为**多线程**（`io-threads`），解决高并发下网络层成为瓶颈的问题；
- **后台任务**：RDB 快照、AOF 重写用 fork 子进程；惰性删除等用后台线程（bio threads），都不阻塞主线程。
- Redis 8 重写了 I/O 线程实现，默认 `io-threads=1`，设为 8 时吞吐最高提升约 112%。[cite:3e714445-1]

**3. 基于多路复用的非阻塞 I/O**
Redis 用 **epoll（Linux）/ kqueue（macOS）/ select** 这类 I/O 多路复用机制：一个线程就能同时监听成千上万个客户端连接，谁有数据就处理谁，不会因为某个慢客户端而卡住。

**4. 精心设计的底层数据结构（关键，别忽略）**
Redis 不是拿普通 C 字符串、链表凑合，而是为每种类型定制了紧凑高效的结构：
- **SDS（简单动态字符串）**：O(1) 取长度、二进制安全、避免频繁扩容；
- **Dict（哈希表）**：O(1) 读写；
- **SkipList（跳表）+ Dict**：实现 ZSet，支持 O(logN) 按分数查询；
- **Listpack / Quicklist / Intset**：对小数据用连续紧凑编码，省内存；
- **Stream 用 Rax（基数树）** 等。

**5. 其它工程细节**
- 命令集经过精心设计，大量操作是 O(1) 或 O(logN)；
- 使用 jemalloc 高效内存分配器；
- 客户端-服务端协议 RESP 极其轻量；
- 批量命令（Pipeline）、客户端缓存、命令的局部性优化（Redis 8 又有 30+ 项性能优化）。[cite:3e714445-2]

**要辩证看待"快"：**
单线程决定了它**怕 CPU 密集命令和 big key**。生产上禁止裸用 `KEYS *`（用 `SCAN`）、注意大 key，否则再快也会被阻塞。Redis 的快是"对常规命令很快"，不是"什么都能快"。

---

## 4、Redis有哪些数据类型

**答案：**

**第一梯队：五种基础类型（必须滚瓜烂熟）**

| 类型                   | 中文名           | 底层结构                   | 典型场景                        | 代表命令                            |
| ---------------------- | ---------------- | -------------------------- | ------------------------------- | ----------------------------------- |
| **String**             | 字符串           | int / embstr / raw（SDS）  | 缓存、计数器、分布式锁、Session | `SET GET INCR SETNX`                |
| **Hash**               | 哈希             | listpack / hashtable       | 存"对象"（用户信息、商品）      | `HSET HGET HGETALL HINCRBY`         |
| **List**               | 列表             | quicklist（listpack 组成） | 消息队列、最新评论、时间线      | `LPUSH RPUSH LPOP LRANGE`           |
| **Set**                | 集合（无序去重） | intset / hashtable         | 去重、共同好友、抽奖            | `SADD SMEMBERS SINTER SPOP`         |
| **ZSet（Sorted Set）** | 有序集合         | listpack / skiplist+dict   | 排行榜、延迟队列                | `ZADD ZRANGE ZINCRBY ZRANGEBYSCORE` |

> 技术细节：Redis 7.0 起用 **listpack** 全面取代了老的 ziplist 编码；小规模数据走紧凑编码省内存，规模变大后自动升级为 hashtable / skiplist。

**第二梯队：由基础类型派生的扩展类型**

| 类型            | 本质            | 用途                                            |
| --------------- | --------------- | ----------------------------------------------- |
| **Bitmap**      | String 的位操作 | 签到、在线状态、去重统计（亿级布尔标记）        |
| **HyperLogLog** | 概率结构        | 基数统计（UV），内存固定约 12KB，允许极小的误差 |
| **Geo**         | ZSet 封装       | 地理位置（附近的人、外卖配送）                  |
| **Stream**      | 消息流          | 可靠的流式消息队列（消费者组、ACK）             |
| **Bitfield**    | String 的位域   | 自定义位宽的多计数器                            |

**第三梯队：Redis 8 原生内置（旧认知里它们是独立模块）**

Redis 8 把原来 Redis Stack 的模块全部并入内核，新增 8 种数据结构：[cite:3e714445-1][cite:3e714445-2]
- **JSON**：直接存 JSON 文档，支持 JSONPath 局部读写（原子更新子字段）；
- **Time Series**：时间序列（物联网传感器、监控指标），带压缩和降采样；
- **Vector Set（向量集合，beta）**：高维向量相似度检索，面向 AI/RAG/推荐；
- **五种概率结构**：**Bloom Filter / Cuckoo Filter**（判存在）、**Count-min Sketch**（频次估计）、**Top-k**（最频繁 TopK）、**t-digest**（分位数估计）。

**选择建议**：日常开发 90% 场景用五种基础类型就够了；需要"对象缓存"优先 Hash（或 JSON），需要"排队"用 List/Stream，需要"去重"用 Set，需要"排序/排行"用 ZSet，需要"判断是否存在"用 Bloom Filter。

---

## 5、什么是Redis持久化？

**答案：**

**概念**：持久化就是**把内存中的数据保存到磁盘**。因为 Redis 本质是内存数据库，进程一退出、服务器一宕机，内存里的数据就全没了。有了持久化，Redis 重启后可以从磁盘把数据**恢复**回来。

**打个比方**：内存里的数据像写在白板上的字，断电（擦黑板）就没了；持久化相当于两种存档方式：
- **拍照存档（RDB）**：把当前黑板内容拍一张照片存起来；
- **流水账（AOF）**：把每次动笔写了什么，一行一行记到本子上。

重启时，把照片/账本拿出来，就能还原当时的黑板。

**Redis 提供三种持久化方式（Redis 4.0+）：**

1. **RDB（Redis Database）快照**：按时间间隔把内存数据整体保存为一份二进制文件（默认 `dump.rdb`）。
2. **AOF（Append Only File）追加文件**：把每一条**写命令**追加记录到日志文件（默认 `appendonly.aof`）。
3. **混合持久化（默认推荐）**：AOF 文件头部用 RDB 二进制格式存"已有数据"，后面再追加新写命令——**恢复快 + 数据安全**两头兼顾（`aof-use-rdb-preamble yes`）。

**也可以完全关闭持久化**：如果 Redis 只当"可丢失的纯缓存"，可以两种都关，省掉磁盘开销。

> 细节：Redis 7.0 把 AOF 改成了 **Multi-Part AOF**（manifest 清单 + base 基础文件 + incr 增量文件），解决了以前重写期间大文件占用磁盘的问题；加载时如果同时配了 RDB 和 AOF，**优先用 AOF**（因为它记录得更全）。

---

## 6、Redis 的持久化机制是什么？各自的优缺点？

**答案：**

### 机制一：RDB（快照）

**原理**：在满足条件时（时间间隔 + 改动次数，由 `save` 配置控制），Redis 通过 **fork 一个子进程**，利用操作系统 **Copy-On-Write（写时复制）**，让子进程把内存数据写入二进制快照 `dump.rdb`。`BGSAVE` 是后台保存（不阻塞主线程），`SAVE` 是前台保存（会阻塞，生产禁用）。

**触发方式**：
- 配置文件 `save 900 1 300 100 60 10000`（900 秒内有 1 次写 / 300 秒内 100 次 / 60 秒内 10000 次）；
- 手动 `SAVE` / `BGSAVE`；
- `SHUTDOWN` 正常关机时自动保存；
- 主从复制做全量同步时，主节点也会生成 RDB。

**优点**：
- 文件紧凑，是**二进制**，恢复加载快；
- 适合做备份、灾备、数据迁移；
- 对性能影响小：fork 子进程写盘，主线程几乎不阻塞（除非内存巨大导致 fork 本身变慢）。

**缺点**：
- **两次快照之间的数据会丢**：比如每 5 分钟存一次，宕机时最多丢 5 分钟数据；
- fork 时如果内存占用大，可能造成主线程**短暂卡顿**（几十~几百毫秒级）。

### 机制二：AOF（追加文件）

**原理**：每执行一条**写命令**，就把这条命令以 RESP 协议格式追加到 AOF 文件末尾。什么时候真正写入磁盘由 `appendfsync` 决定：
- `always`：每条写命令都 fsync 到磁盘（最安全，最慢）；
- `everysec`：每秒 fsync 一次（**默认**，最多丢 1 秒数据，性能好）；
- `no`：交给操作系统决定（最快，可能丢更多数据）。

AOF 文件会越来越大，所以有 **AOF 重写（`BGREWRITEAOF`）**：后台把当前数据集重新生成一份精简的 AOF，只保留最终状态，去掉历史冗余命令。Redis 7.0+ 用 Multi-Part AOF 管理重写过程。

**优点**：
- 数据**更安全**（`everysec` 最多丢 1 秒，`always` 不丢）；
- 是文本格式，**可读、可审计**（甚至可以手动修复）；
- 通过重写控制文件大小。

**缺点**：
- 文件比 RDB **大**；
- 启动加载比 RDB **慢**（要逐条回放命令）；
- 每条命令追加有额外写盘开销（`always` 模式下性能下降明显）。

### 机制三：混合持久化（4.0+，推荐）

**原理**：AOF 重写时，把已有数据用 RDB 二进制格式写进 AOF 文件头部，再继续追加新命令。配置 `aof-use-rdb-preamble yes`（默认开）。

**优点**：**既有 RDB 的恢复速度，又有 AOF 的数据安全**，是当前生产环境的默认首选。

**缺点**：AOF 文件头部是二进制 RDB 格式，不像纯文本 AOF 那样方便人读/人工修改。

### 怎么选（实用决策表）

| 你的需求                     | 推荐方案              |
| ---------------------------- | --------------------- |
| 能接受丢几分钟数据、要恢复快 | RDB 或混合            |
| 尽量不丢数据（秒级）         | AOF `everysec` 或混合 |
| 纯缓存、数据可丢             | 关闭持久化            |
| 既要安全又要快               | 混合持久化（RDB+AOF） |

---

## 7、Redis key的过期时间和永久有效分别怎么设置？

**答案：**

### 一、设置过期时间（TTL）

**方式 1：对已有 key 单独设置过期**

```bash
EXPIRE key 60        # 60 秒后过期
PEXPIRE key 60000    # 60000 毫秒后过期（=60 秒）
EXPIREAT key 1700000000    # 在指定 Unix 时间戳（秒）过期
PEXPIREAT key 1700000000123 # 在指定 Unix 时间戳（毫秒）过期
```

> 注意：`EXPIRE` 只能给**已存在的 key** 设置；key 不存在时返回 0。

**方式 2：设置值的同时带上过期时间（最常用）**

```bash
SETEX key 60 value          # 老式：设值 + 60 秒过期
SET key value EX 60         # 现代写法，等价
SET key value PX 60000      # 毫秒级
SET key value EXAT 1700000000   # 绝对时间（秒）
SET key value PXAT 1700000000000 # 绝对时间（毫秒）
SET key value EX 60 NX      # 过期 + 不存在才设置（分布式锁标准写法！）
SET key value EX 60 KEEPTTL # 覆盖值但保留原有 TTL
```

**查看剩余时间**

```bash
TTL key     # 返回剩余秒数
PTTL key    # 返回剩余毫秒数
# TTL 返回 -2：key 不存在；返回 -1：key 存在但没有过期时间（永久）
```

### 二、设置永久有效

**1. 新建的 key 默认就是永久的**：只要不设置过期时间，key 就一直有效。

**2. 把已有过期的 key 变成永久**：

```bash
PERSIST key     # 移除 key 的过期时间，让它永久有效
```

**3. 判断一个 key 是不是永久**：`TTL key` 返回 `-1` 就是永久。

### 三、补充：过期删除是怎么发生的？（常考）

Redis 不会盯着每个 key 到点就删，而是用两种策略配合：
- **惰性删除（Lazy）**：key 过期了，但不主动删；等到**被访问**的那一刻才检查并删除；
- **定期删除（Active）**：后台定时抽样检查一批过期 key 并删除（避免过期 key 长期占用内存）。

> 另外：Redis 7.4 引入了 **Hash 字段级过期**，Redis 8 补充了 `HSETEX / HGETEX / HGETDEL` 三个命令，可以给 Hash 里的**某个字段**单独设过期，非常适合会话/缓存场景。[cite:3e714445-2]

---

## 8、Redis事务的概念

**答案：**

**概念**：Redis 事务是一组命令的"**打包执行**"。用 `MULTI` 开始，把多条命令加入一个队列，最后用 `EXEC` 一次性、按顺序执行。核心特性是：**在 `MULTI` 和 `EXEC` 之间，这条事务里的命令不会被其他客户端的命令插队**——执行时整组命令是一个整体，中间不会混入别人的操作。

**打个比方**：你去便利店买东西，`MULTI` 是开始购物车，把商品（命令）一件件放进去（入队），`EXEC` 是去收银台一次性结账。结账过程中，其他顾客（其他客户端）不能插到你前面。

**但必须纠正一个常见误解**：Redis 事务**不是**传统数据库那样"要么全部成功、要么全部回滚"（ACID 里的原子性）。Redis 事务只保证"**一次性执行完、中间不被插队**"，但：
- **没有回滚机制**：如果执行到第 3 条命令时报运行时错误（比如对 String 用了 List 的命令），**前面和后面的命令照常执行**，不会撤销；
- 它更像一个"命令打包批量提交"。

**为什么还要用事务？**
- 批量执行，减少网络往返（性能）；
- 保证一组命令**不被并发插队**，满足很多业务（如"扣库存 + 记录订单"一起执行）；
- 配合 `WATCH`（乐观锁）实现 **CAS（Compare-And-Swap）** 式的并发控制：先监视 key，事务执行前如果 key 被改过，事务直接放弃。

**和 Lua 脚本的关系（现代实践）**：如果你需要"要么全执行、要么不执行"并且带复杂逻辑，现在更推荐 **Lua 脚本（`EVAL`）**。Redis 保证脚本整体原子执行，还能写 if/else 判断，比裸事务更强大。很多新项目用 Lua 代替了复杂事务。

---

## 9、Redis事务的三个阶段

**答案：**

Redis 事务的执行流程分 **3 个阶段**（这也是常考的背诵点）：

**阶段一：开启事务 —— `MULTI`**
```bash
MULTI
OK          # 进入事务状态，后续命令不会立即执行，而是入队
```

**阶段二：命令入队（Queuing）**
```bash
SET name "Alice"
QUEUED      # 只是加入队列，还没真正执行
INCR counter
QUEUED
```
此时命令只是**排队**，Redis 只做**语法检查**（命令是否存在、参数个数对不对），不真正执行。

**阶段三：执行/取消事务 —— `EXEC` / `DISCARD`**
```bash
EXEC
1) OK            # 依次返回每条命令的结果
2) (integer) 1
```
`EXEC` 把队列里所有命令**按顺序一次性执行完**，并返回一个结果数组。
如果不想执行了，用 `DISCARD` 清空队列、退出事务状态。

**补充：还有一个"前置"动作 `WATCH`（不算固定阶段，但常一起提）**
`WATCH key` 必须在 `MULTI` **之前**调用，用于监视 key。如果 `WATCH` 之后、`EXEC` 之前，这个 key 被**其他客户端**修改了，`EXEC` 会返回 `(nil)` 表示**放弃执行**（乐观锁 CAS 语义），你需要重试。

> 常见错误理解提醒：三个阶段是 **MULTI → 命令入队 → EXEC**。不是"MULTI → EXEC → 结果"，也不是"WATCH"算一个阶段（WATCH 是前置可选操作）。

---

## 10、Redis事务相关命令和事务的特征？

**答案：**

### 相关命令

| 命令                 | 作用                                           |
| -------------------- | ---------------------------------------------- |
| `MULTI`              | 开启事务，进入命令入队状态                     |
| `EXEC`               | 执行事务队列中的所有命令                       |
| `DISCARD`            | 取消事务，清空命令队列                         |
| `WATCH key [key...]` | 监视 key（乐观锁），key 被改动则 EXEC 放弃执行 |
| `UNWATCH`            | 取消所有监视                                   |

### 事务的特征（重点，面试爱考）

**1. 原子性执行（防插队）**
`EXEC` 期间，事务内的命令作为一个整体被执行，**其他客户端的命令不会插入到中间**。这是靠 Redis 单线程执行模型保证的。

**2. 队列化执行**
`MULTI` 之后的命令先进队列、不立即执行；`EXEC` 才真正执行，并按顺序返回结果数组。

**3. 没有回滚（最常考！）**
- **入队时**（编译期）发现错误（命令不存在、参数错误）→ 整个事务被拒绝，`EXEC` 报 `EXECABORT`，一条都不执行；
- **执行时**（运行时）出现错误（如对 String 类型执行 `LPUSH`）→ **该命令报错，其余命令照常执行，不撤销已执行的结果**。Redis 设计者明确说过"Redis 故意不做回滚"，因为错误通常来自编程 bug，回滚并不能修复逻辑错误。

**4. 隔离性依赖 WATCH（乐观锁）**
单纯 `MULTI/EXEC` 不保证"读-判断-写"的隔离；要防止并发覆盖，必须用 `WATCH` 做 CAS：
```bash
WATCH balance           # 监视余额
balance = GET balance   # 读
newBalance = balance - 100
MULTI
SET balance newBalance  # 写
EXEC
# 若 WATCH 之后 balance 被别的客户端改了，EXEC 返回 (nil)，需重试
```

**5. 不支持嵌套事务**
事务中再发 `MULTI` 会报错。

**6. 队列内每条命令独立返回结果**
`EXEC` 返回一个**结果数组**，顺序对应队列中的命令。

**7. 持久性取决于配置**
事务本身不保证数据落盘；是否持久取决于持久化配置（比如 AOF `always` 才最稳）。

**小结一句**：Redis 事务 = **MULTI 排队 + EXEC 一次性按序执行（防插队）+ WATCH 乐观锁做并发控制 + 但无回滚**。需要更强逻辑时改用 Lua 脚本。

---

## 11、将每种数据类型的命令多敲击一下，加深印象

（你分享的 DeepSeek 链接我已经完整读取，下面的命令清单基于该内容整理，并按 Redis 8 的现状做了补充修正：[cite:dca4d002-1]）

**准备：清空环境（方便反复练习）**
```bash
FLUSHALL
```

### 🟢 第一关：String —— 存 Token 和计数器

```bash
# 存一个字符串（用户 token）
SET user:1001:token "abc123xyz"
GET user:1001:token
STRLEN user:1001:token          # 字符串长度

# 追加内容（模拟 token 刷新）
APPEND user:1001:token "refresh"
GET user:1001:token

# 带过期时间的 token（5 秒后消失）
SETEX user:1002:token 5 "expired_token"
TTL user:1002:token
GET user:1002:token              # 等 5 秒后再敲，返回 nil

# 计数器（文章浏览量）
SET article:1001:views 0
INCR article:1001:views
INCRBY article:1001:views 10
DECR article:1001:views
GET article:1001:views

# 批量操作
MSET user:1 "Alice" user:2 "Bob"
MGET user:1 user:2

# 不存在才设置（分布式锁雏形）
SETNX lock:order "locked"
SETNX lock:order "locked_again"  # 返回 0，设置失败
GET lock:order
```

> 现代补充：分布式锁更标准的写法是 `SET lock:order "locked" NX EX 10`（原子地加锁 + 过期），比 `SETNX` + 单独 `EXPIRE` 两步更安全。

### 🟡 第二关：Hash —— 存"用户详情对象"

```bash
# 存一个对象（用户信息）
HSET user:1001 name "张三" age 25 city "北京"
HGET user:1001 name
HGET user:1001 age
HGETALL user:1001                # 一次拿全部字段和值
HKEYS user:1001                  # 只拿字段名
HVALS user:1001                  # 只拿值
HLEN user:1001                   # 字段数量

# 年龄加一岁
HINCRBY user:1001 age 1

# 追加字段（注意：用 HSET，不要再用 HMSET —— HMSET 在 4.0 起已被废弃）
HSET user:1001 sex "男" email "test@qq.com"

# 判断字段是否存在
HEXISTS user:1001 sex
HEXISTS user:1001 hobby

# 删除字段
HDEL user:1001 city
HGETALL user:1001
```

> 现代补充（Redis 8）：新命令 `HSETEX user:1001 age 60 26`（给字段设值并指定过期）、`HGETEX`、`HGETDEL` 可以按字段级操作。

### 🔵 第三关：List —— 消息队列 / 最新评论

```bash
# 左侧推入（最新放左边）
LPUSH comments:1001 "评论1: 太棒了"
LPUSH comments:1001 "评论2: 顶一下"
LPUSH comments:1001 "评论3: 沙发"

LRANGE comments:1001 0 -1        # 查看全部
RPUSH comments:1001 "评论4: 考古" # 右侧追加
LRANGE comments:1001 0 1         # 前两条

LPOP comments:1001               # 弹出左边第一条（处理最新消息）
RPOP comments:1001               # 弹出右边最后一条
LLEN comments:1001               # 列表长度
LINDEX comments:1001 0           # 按下标取值

# 在指定元素前插入
LINSERT comments:1001 BEFORE "评论2" "置顶消息"
LRANGE comments:1001 0 -1

# 修剪（只保留前 2 条）
LTRIM comments:1001 0 1
LRANGE comments:1001 0 -1
```

> 队列增强：需要"阻塞式"消费时用 `BLPOP` / `BRPOP`（带超时），需要可靠消费队列用 Stream。

### 🟠 第四关：Set —— 好友关系 / 抽奖池

```bash
# 两个用户的好友集合
SADD user:A:friends "B" "C" "D" "E"
SADD user:B:friends "C" "F" "G" "A"

SMEMBERS user:A:friends          # 全部成员
SCARD user:A:friends             # 好友数量
SISMEMBER user:A:friends "C"     # C 是不是 A 的好友

SINTER user:A:friends user:B:friends  # 共同好友（交集）
SDIFF  user:A:friends user:B:friends  # A 有而 B 没有（差集）
SUNION user:A:friends user:B:friends  # 并集

# 抽奖池
SADD lottery "user_001" "user_002" "user_003"
SPOP lottery 1                   # 随机弹出 1 个（删除）
SRANDMEMBER lottery              # 随机取一个（不删除）
SREM lottery "user_003"          # 删除指定成员
SMEMBERS lottery
```

### 🟣 第五关：ZSet —— 游戏排行榜

```bash
ZADD rank:game 100 "玩家_A"
ZADD rank:game 200 "玩家_B"
ZADD rank:game 150 "玩家_C"
ZADD rank:game 300 "玩家_D"

ZRANGE rank:game 0 -1 WITHSCORES          # 升序 + 分数
ZREVRANGE rank:game 0 -1 WITHSCORES       # 降序（从高到低）
ZREVRANGE rank:game 0 2 WITHSCORES        # 前三名（冠亚季军）

ZRANK rank:game "玩家_A"                  # 升序排名（0 是最后）
ZREVRANK rank:game "玩家_A"               # 降序排名（0 是第一名）
ZSCORE rank:game "玩家_B"                 # 查分数
ZINCRBY rank:game 50 "玩家_C"             # 加分（打赢了）

ZRANGEBYSCORE rank:game 100 250 WITHSCORES  # 按分数范围查
ZCOUNT rank:game 100 250                  # 统计分数区间人数
ZREM rank:game "玩家_D"                   # 删除成员
ZRANGE rank:game 0 -1
```

### ✍️ 终极综合：模拟一个"博客系统"（凭记忆盲敲一遍）

```bash
SET post:2026:title "Redis入门实战"        # 1. 标题（String）
HSET post:2026 author "小明" content "内容很多..." views 0  # 2. 详情（Hash）
HINCRBY post:2026 views 1                 # 3. 浏览量 +1（Hash 自增）
LPUSH latest_posts "2026"                 # 4. 最新文章列表（List）
SADD post:2026:tags "Redis" "数据库" "NoSQL"  # 5. 标签（Set）
ZADD post:score 9.5 "2026"                # 6. 评分（ZSet）
```

**敲命令小技巧**：
- 用**上下方向键**复用上一条命令，改几个字母即可；
- 开两个终端，一个敲 `MONITOR`，另一个操作，可以实时看到 Redis 收到的每一条命令；
- 命令记不住时敲 `HELP @string` / `COMMAND COUNT` 查看分类。

---

## 12、缓存穿透是什么？如何解决？

**答案：**

**概念**：**缓存穿透**是指查询一个"**缓存和数据库里都不存在**"的数据（比如用一个不存在的 id：`-1`、`abc`、`99999999`）。由于它不存在，缓存里永远不会有这个 key（也永远不会被写进缓存），于是**每次请求都绕过缓存、直接打到数据库**。如果攻击者批量构造这类请求（或恶意刷不存在的数据），数据库会被瞬间打爆。

**类比**：一个门卫（缓存）看名单放行，结果有人反复拿一张"名单上根本没有的人名"来问——门卫每次都只能去仓库（数据库）里翻一遍，累死了。

**为什么叫"穿透"**：因为请求"穿透"了缓存这一层，直接打到最底层的数据库。

**解决方案（从简单到复杂）：**

1. **接口层参数校验**：在最前面就把非法参数拦住（如 id 必须 >0、必须符合格式），从源头减少无效请求。
2. **缓存空值**（最简单有效，最常用）：查数据库返回空时，**也把这个 key 缓存起来**，value 记为占位符（如 `null`），并设置一个**较短的 TTL**（如 5 分钟）。这样下次同样的请求直接命中缓存，不再打库。
   ```bash
   # 伪逻辑：DB 查无结果时
   SET user:99999999 "" EX 300   # 空值缓存 300 秒
   ```
3. **布隆过滤器（Bloom Filter）**：在缓存前面再加一层"存在性判断"。把所有合法 key（如合法用户 id）预加载进布隆过滤器；请求来了先问布隆过滤器"这个 key 存在吗"，**不存在直接拒绝**，只有"可能存在"才继续查缓存/数据库。缺点是有极小的误判率、且不支持删除（可用 Cuckoo Filter 或 Redis 8 原生 Bloom Filter）。
4. **限流 / 熔断 / 降级**：对异常高频的访问做限流；数据库侧做连接池保护、降级兜底，防止被击穿。
5. **安全层面**：配合风控识别恶意 IP，做黑名单。

**再次提醒区分三个"缓存故障"**（面试必问三兄弟）：
- **缓存穿透**：查一个**不存在**的数据，绕过缓存打库（本题）；
- **缓存击穿**：一个**热点 key 恰好过期**，瞬间大量并发请求同时打库；
- **缓存雪崩**：**大量 key 同时过期** 或 **Redis 整体宕机**，导致流量全部打到数据库（下一题）。

| 故障 | 数据存在？   | 触发条件                        | 核心解法                         |
| ---- | ------------ | ------------------------------- | -------------------------------- |
| 穿透 | 不存在       | 请求非法 key                    | 空值缓存 / 布隆过滤器 / 参数校验 |
| 击穿 | 存在（热点） | 单个 key 过期瞬间               | 互斥锁 / 逻辑过期                |
| 雪崩 | 存在（大量） | 大量 key 同时过期 或 Redis 宕机 | TTL 随机化 / 高可用 / 多级缓存   |

---

## 13、缓存雪崩的概念？如何应对？

**答案：**

**概念**：**缓存雪崩**指大量请求在**同一时刻**涌向数据库，导致数据库被压垮，进而引发连锁反应、整个系统崩溃的场景。它通常由两种原因引发：

1. **大量 key 在同一时间集中过期**：比如一批缓存都设置了相同的 TTL（都是凌晨 0 点过期、或都设成 1 小时），到了那一刻，缓存集体失效，所有请求瞬间全部打到底层数据库；
2. **Redis 服务整体宕机/不可用**：缓存这层整个没了，全部流量直接打数据库。

数据库本身扛不住瞬间的全量流量 → 数据库崩溃 → 依赖它的服务也跟着挂 → **雪崩式连锁故障**。

**类比**：你给 1000 个闹钟都设了"同一分钟"响，到点全部一起响，把全家都吵醒了（数据库被吵爆）。

**应对方案（分层防御）：**

**① 针对"大量 key 集中过期"：**
- **TTL 随机化**（最常用）：给过期时间加一个随机偏移量，避免集中在同一时刻失效。比如 `EXPIRE key 3600 + random(0, 600)`；
- **热点 key 不过期 + 逻辑过期**：key 本身不设 TTL（永不过期），后台异步线程定期重建/续期，读到旧数据时返回旧值并触发异步刷新（保证极端情况下数据库也不会瞬间全量打到）；
- **双缓存 / 错峰**：同一份数据缓存两套（主缓存 TTL 短、备用缓存 TTL 长），主缓存失效时用备用缓存兜底；
- **缓存预热 + 定时续期**：上线前把热点数据预加载进缓存，并错峰刷新。

**② 针对"Redis 整体宕机"：**
- **高可用**：搭建**主从复制 + Sentinel 哨兵**（自动故障转移）或 **Redis Cluster**（分片 + 高可用），让 Redis 本身尽量不"全挂"（下一题就是动手配这个）；
- **多级缓存（本地 + 远端）**：在应用进程内加一层本地缓存（C++ 可用自己的 LRU，Java 用 Caffeine 等），Redis 挂了还有本地兜底；
- **限流 / 熔断 / 降级**：数据库前加限流（如 Sentinel/Hystrix 等框架、或自研令牌桶），超阈值直接熔断，返回降级数据（旧数据、默认值、友好提示），**宁可服务降级也不让数据库死**；
- **数据库侧保护**：连接池设置上限、读写分离、只读副本分担查询压力。

**③ 业务层面：**
- 提前做**容量评估和演练**（Redis 宕机预案）；
- 用**键空间通知 + 延迟消息**错峰重建缓存。

**一句话总结应对思路**：**让失效时间错开（随机 TTL）、让缓存不整体消失（高可用 + 多级缓存）、让数据库扛得住（限流熔断降级）**。

---

## 14、自己动手配置一下主从复制与哨兵模式、以及hiredis的封装

**答案：**

这是动手题，我按"**配起来 → 验证 → 封起来**"三步走给你一份可照着执行的完整方案（基于 Redis 6/7/8 通用语法）。

---

### 一、主从复制（Master-Slave / Master-Replica）

**原理速览**：从节点（Replica）启动后连上主节点（Master），经历两个阶段：
- **全量同步**：主节点 `BGSAVE` 生成 RDB 快照发给从节点，从节点加载；
- **增量同步**：同步期间产生的新写命令，通过主节点的**命令传播**持续发给从节点（有积压缓冲区 `repl_backlog` 兜底）；
- 之后是**持续复制**：主节点每笔写都会异步推给从节点，从节点每 1 秒发一次 `REPLCONF ACK` 心跳。
- Redis 8 还优化了复制：全量 + 增量**双流并行**传输，同步提速 18%、主节点复制期间峰值缓冲区内存降低 35%。[cite:3e714445-1]

**Step 1：准备 3 个实例（1 主 2 从），用三个配置文件：**

`redis-master.conf`（主节点，端口 6379）：
```conf
port 6379
daemonize yes
pidfile /var/run/redis-6379.pid
logfile /tmp/redis-6379.log
dir /tmp/redis-6379
appendonly yes
```

`redis-replica-1.conf`（从节点 1，端口 6380）：
```conf
port 6380
daemonize yes
pidfile /var/run/redis-6380.pid
logfile /tmp/redis-6380.log
dir /tmp/redis-6380
appendonly yes
# ↓↓↓ 关键：指定主节点（Redis 5.0+ 用 replicaof，旧版叫 slaveof）
replicaof 127.0.0.1 6379
replica-read-only yes   # 从节点默认只读，防止误写
```

`redis-replica-2.conf`（从节点 2，端口 6381）：
```conf
port 6381
daemonize yes
pidfile /var/run/redis-6381.pid
logfile /tmp/redis-6381.log
dir /tmp/redis-6381
appendonly yes
replicaof 127.0.0.1 6379
replica-read-only yes
```

**Step 2：依次启动并验证：**
```bash
redis-server redis-master.conf
redis-server redis-replica-1.conf
redis-server redis-replica-2.conf

# 在主节点写入，检查从节点是否同步
redis-cli -p 6379 SET hello "world"
redis-cli -p 6380 GET hello        # 应返回 "world"
redis-cli -p 6381 GET hello

# 查看复制状态
redis-cli -p 6380 info replication
# role:slave
# master_host:127.0.0.1
# master_link_status:up
```

**Step 3：命令行动态切换（不用改配置）**：
```bash
redis-cli -p 6381 REPLICAOF 127.0.0.1 6379   # 动态指定主节点
redis-cli -p 6381 REPLICAOF NO ONE           # 取消从属，自己当主
```

> 验证结论：主节点可写、从节点自动同步且只读（从节点写会报 `READONLY`）。

---

### 二、哨兵模式（Sentinel）—— 自动故障转移

**原理速览**：Sentinel 是"监控进程"，多个哨兵互相配合，实现主节点挂掉后的**自动切换**：
1. **主观下线（sdown）**：单个哨兵发现主节点心跳超时；
2. **客观下线（odown）**：达到**法定数量（quorum）**的哨兵都认为它挂了；
3. **选举 leader 哨兵**，leader 从所有从节点中选一个当**新主节点**（优先按优先级/同步进度等）；
4. 其他从节点改认新主，旧主恢复后自动降为从节点；
5. 哨兵会**重写配置文件**，并通过**发布订阅**通知客户端。

**Step 1：创建 3 个哨兵配置（哨兵本身也要奇数个，quorum=2）：**

`sentinel-26379.conf`（另外两个改成 26380、26381，内容相同）：
```conf
port 26379
daemonize yes
logfile /tmp/sentinel-26379.log
dir /tmp/sentinel-26379

# 监控名为 mymaster 的主节点，地址 127.0.0.1:6379，法定数 2
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000   # 5 秒没心跳判主观下线
sentinel failover-timeout mymaster 10000         # 故障转移超时
sentinel parallel-syncs mymaster 1               # 同时向几个从节点同步
```

**Step 2：启动 3 个哨兵：**
```bash
redis-sentinel sentinel-26379.conf
redis-sentinel sentinel-26380.conf
redis-sentinel sentinel-26381.conf

# 查看哨兵状态
redis-cli -p 26379 sentinel masters
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
```

**Step 3：演练故障转移（重点！亲手做一遍）**：
```bash
# 1. 模拟主节点宕机
redis-cli -p 6379 SHUTDOWN NOSAVE

# 2. 等几秒，查看哨兵是否自动选出了新主
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# 应返回 127.0.0.1 和某个从节点的端口（比如 6380 或 6381）

# 3. 重新拉起旧主，观察它自动变成从节点
redis-server redis-master.conf
redis-cli -p 6379 info replication   # 现在 role:slave
```

> 手写一遍"杀主 → 看切换 → 旧主回归变成从"，比看十遍文档都记得牢。

---

### 三、hiredis 的 C++ 封装（RAII + 线程安全）

hiredis 是 Redis 官方的 **C 语言客户端**（同步 + 异步），功能精简，非常适合嵌入式/C++ 项目。下面给一个现代 C++11/14 风格的 RAII 封装：

**RedisWrapper.h**
```cpp
#pragma once

#include <hiredis/hiredis.h>
#include <string>
#include <mutex>
#include <stdexcept>
#include <cstdarg>
#include <memory>

// ---- 1. RAII 管理 redisReply，杜绝内存泄漏 ----
class RedisReply {
public:
    explicit RedisReply(redisReply* r) : reply_(r) {
        if (!reply_) throw std::runtime_error("null redisReply");
    }
    ~RedisReply() { if (reply_) freeReplyObject(reply_); }

    RedisReply(const RedisReply&) = delete;
    RedisReply& operator=(const RedisReply&) = delete;
    RedisReply(RedisReply&& o) noexcept : reply_(o.reply_) { o.reply_ = nullptr; }
    RedisReply& operator=(RedisReply&& o) noexcept {
        if (this != &o) { freeReplyObject(reply_); reply_ = o.reply_; o.reply_ = nullptr; }
        return *this;
    }

    bool ok()     const { return reply_ && reply_->type != REDIS_REPLY_ERROR; }
    bool isNil()  const { return reply_ && reply_->type == REDIS_REPLY_NIL; }
    int  type()   const { return reply_ ? reply_->type : REDIS_REPLY_ERROR; }
    std::string str() const {
        return (reply_->str) ? std::string(reply_->str, reply_->len) : std::string();
    }
    long long integer() const { return reply_->integer; }
    size_t elements() const { return (reply_->type == REDIS_REPLY_ARRAY) ? reply_->elements : 0; }
    RedisReply element(size_t i) const { return RedisReply(reply_->element[i]); }

private:
    redisReply* reply_;
};

// ---- 2. 连接封装：自动管理 redisContext 生命周期 ----
class RedisClient {
public:
    RedisClient() = default;
    ~RedisClient() { disconnect(); }

    RedisClient(const RedisClient&) = delete;
    RedisClient& operator=(const RedisClient&) = delete;

    bool connect(const std::string& host, int port,
                 const std::string& password = "", int timeout_ms = 2000) {
        timeval tv{ timeout_ms / 1000, (timeout_ms % 1000) * 1000 };
        redisContext* c = redisConnectWithTimeout(host.c_str(), port, tv);
        if (!c || c->err) {
            last_error_ = c ? c->errstr : "connect failed";
            if (c) redisFree(c);
            return false;
        }
        ctx_.reset(c, redisFree);
        if (!password.empty()) {
            auto r = command("AUTH %s", password.c_str());
            if (!r.ok()) { last_error_ = r.str(); disconnect(); return false; }
        }
        return true;
    }

    void disconnect() { std::lock_guard<std::mutex> lk(mtx_); ctx_.reset(); }

    bool connected() const { return static_cast<bool>(ctx_); }

    // 变参命令，格式与 redisCommand 一致（%s %d %b...），由 hiredis 负责格式化
    // hiredis 的 redisContext 不是线程安全的，这里用互斥锁串行化，保证多线程可用
    RedisReply command(const char* fmt, ...) {
        std::lock_guard<std::mutex> lk(mtx_);
        if (!ctx_) throw std::runtime_error("redis not connected");

        va_list ap;
        va_start(ap, fmt);
        redisReply* r = static_cast<redisReply*>(redisvCommand(ctx_.get(), fmt, ap));
        va_end(ap);

        if (!r) throw std::runtime_error(ctx_->errstr ? ctx_->errstr : "command failed");
        return RedisReply(r);
    }

    const std::string& last_error() const { return last_error_; }

private:
    std::unique_ptr<redisContext, decltype(&redisFree)> ctx_{ nullptr, redisFree };
    std::mutex mtx_;          // hiredis context 非线程安全，加锁保护
    std::string last_error_;
};
```

**main.cpp 使用示例**
```cpp
#include <iostream>
#include "RedisWrapper.h"

int main() {
    RedisClient redis;
    if (!redis.connect("127.0.0.1", 6379, "your_password_optional")) {
        std::cerr << "连接失败: " << redis.last_error() << "\n";
        return 1;
    }

    // 写
    auto set = redis.command("SET user:1001:name %s", "Alice");
    std::cout << "SET => " << (set.ok() ? "OK" : "FAIL") << "\n";

    // 读
    auto get = redis.command("GET %s", "user:1001:name");
    if (!get.isNil()) std::cout << "GET => " << get.str() << "\n";

    // 计数器
    auto incr = redis.command("INCR %s", "counter:views");
    std::cout << "INCR => " << incr.integer() << "\n";

    // 带过期时间的分布式锁
    auto lock = redis.command("SET %s %s NX EX %d", "lock:order", "1", 10);
    std::cout << "加锁 => " << (lock.ok() ? "成功" : "失败/已被占用") << "\n";
    return 0;
}
```

**编译运行**
```bash
# 先装 hiredis：Ubuntu 上 apt install libhiredis-dev；或源码编译
g++ -std=c++17 main.cpp -lhiredis -o demo
./demo
```

**封装要点总结**（面试/代码评审都会问）：
1. **RAII 管理 `redisReply` 和 `redisContext`**：hiredis 是 C 库，`freeReplyObject` / `redisFree` 必须配对，用 RAII 保证异常安全；
2. **`redisContext` 非线程安全**：多线程共享一个连接必须加锁（我上面用了 `std::mutex`）；要么干脆"每线程一连接"；
3. **变参命令复用 hiredis 的格式化**（`redisvCommand`），天然支持二进制安全（`%b`）；
4. **进阶**（可扩展方向）：自动重连、连接池、Pipeline 批量、异步 API（`redisAsyncContext` + 事件循环）——嵌入式场景里异步事件循环很实用。

---

*"清单"**

- 能讲清 Redis 本质 + 8.0 新变化（许可、数据类型、I/O 线程）；✔
- 能说清"为什么快"五层原因，且知道"单线程"的精确边界；✔
- 5 种基础类型 + 扩展 + Redis 8 新结构各配一个场景；✔
- RDB/AOF/混合的触发、优缺点、怎么选；✔
- EXPIRE/PERSIST/TTL 三种状态（-1 永久、-2 不存在）；✔
- MULTI→入队→EXEC 三阶段，以及"无回滚 + WATCH 乐观锁"；✔
- 穿透/击穿/雪崩三兄弟的区别与解法；✔
- 亲手配一遍主从 + 哨兵，并用 C++ RAII 封装 hiredis。✔
