<!-- obsidian --><ol>
<li><strong>惰性删除</strong>：访问 key 时才检查是否过期，过期则删除</li>
<li><strong>定期删除</strong>：定时抽查一批 key，删掉过期的</li>
<li><strong>持久化过滤</strong>：RDB/AOF 不会保存已过期的 key</li>
<li><strong>内存淘汰策略</strong>：内存满了时，按配置的策略（LRU/LFU/Random 等）主动腾空间</li>
</ol>
<p><strong>定期 + 惰性</strong> 管过期，<strong>淘汰策略</strong> 管内存上限，<strong>持久化</strong> 不备份死数据。<br>
maxmemory设置最大内存<br>
maxmemory_policy设置淘汰策略</p>

策略 | 淘汰范围 | 淘汰规则
-- | -- | --
volatile-lru | 只从设置了过期时间的键中选 | 最久未使用
volatile-lfu | 只从设置了过期时间的键中选 | 使用频次最低
volatile-ttl | 只从设置了过期时间的键中选 | TTL 最早到期
volatile-random | 只从设置了过期时间的键中选 | 随机
allkeys-lru | 从所有键中选 | 最久未使用
allkeys-lfu | 从所有键中选 | 使用频次最低
allkeys-random | 从所有键中选 | 随机
noeviction | 不淘汰 | 内存满时写入报错
<!-- obsidian -->
问题 | 场景 | 解决思路
-- | -- | --
雪崩 | 大量 key 同时过期或 Redis 宕机，请求全部压到 DB | 高可用（主从/集群）+ 限流降级 + 随机过期时间
穿透 | 查不存在的数据，缓存和 DB 都查不到，每次穿透到 DB | 缓存空值 + 布隆过滤器 + 参数校验
击穿 | 某个热点 key 过期，瞬间大量并发请求打到 DB | 热点 key 永不过期 + 互斥锁（只让一个请求查 DB 重建缓存）

<img width="780" height="209" alt="Image" src="https://github.com/user-attachments/assets/5f8e3964-a7d4-42ea-b2d0-21e7f21d15a7" />

### 缓存与数据库不一致

- **原因**：更新 DB 和删除缓存的先后顺序不当，或并发导致
- **解决**：
    - 采用 **Cache Aside Pattern**：先更新 DB，再删除缓存
    - 加上**延时双删**：更新前先删一次缓存，更新 DB 后再删一次（间隔几百毫秒）
    - 用 **Canal** 监听 MySQL binlog，异步删除缓存（最终一致性）
<!-- obsidian --><p>Redis 阻塞</p>
<ul>
<li><strong>原因</strong>：大 key 操作（如 <code>KEYS *</code>、<code>SMEMBERS</code>）、持久化 fork 阻塞、CPU 密集型操作</li>
<li><strong>解决</strong>：
<ul>
<li>用 <code>SCAN</code> 替代 <code>KEYS</code>，用 <code>SSCAN</code> 替代 <code>SMEMBERS</code></li>
<li>大 key 拆分（如大 hash 拆成多个小 hash）</li>
<li><code>bgsave</code>/<code>bgrewriteaof</code> 放在低峰期执行，调大 <code>min-slaves-to-write</code></li>
</ul>
</li>
</ul>
<h3 data-heading="内存爆满">内存爆满</h3>
<ul>
<li><strong>解决</strong>：
<ul>
<li>设置 <code>maxmemory</code> + 合理淘汰策略（如 <code>allkeys-lru</code>）</li>
<li>定期排查大 key，用 <code>redis-cli --bigkeys</code> 扫描</li>
<li>控制 key 的过期时间，避免集中过期</li>
<li>扩容（加节点、集群分片）</li>
</ul>
</li>
</ul>
<h3 data-heading="Redis 服务器挂了怎么办 + 如何防止数据丢失">Redis 服务器挂了怎么办 + 如何防止数据丢失</h3>
<h4 data-heading="1、挂了怎么办（应急）">1、挂了怎么办（应急）</h4>
<ol>
<li><strong>有从库</strong>：Sentinel 或 Cluster 会自动选主切换；手动则执行 <code>slaveof no one</code> 提升从库。</li>
<li><strong>无从库</strong>：重启 Redis，靠 AOF/RDB 恢复数据。</li>
<li><strong>AOF 损坏</strong>：用 <code>redis-check-aof --fix</code> 修复后再启动。</li>
<li><strong>服务先恢复</strong>：若数据暂时不重要，可先清空旧数据启动，避免长时间不可用。</li>
</ol>
<h4 data-heading="2、怎么防止丢数据（预防）">2、怎么防止丢数据（预防）</h4>

层面 | 措施
-- | --
持久化 | AOF 开启 + appendfsync everysec；RDB 定时快照做全量备份
刷盘安全 | 主机设 vm.overcommit_memory=1，防止 OOM Killer 杀进程
主从 | 一主多从，min-replicas-to-write 1，避免主库无备库时写入
高可用 | Sentinel / Cluster 自动故障转移，消除单点
备份 | 定期把 RDB/AOF 文件拷贝到异地或对象存储
监控 | 监控内存、连接数、主从延迟，提前告警

