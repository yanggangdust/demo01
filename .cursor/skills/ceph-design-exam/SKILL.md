---
name: ceph-design-exam
description: >-
  Ceph 设计原理与实现（初级）考试考点复习资料。当用户询问 Ceph 架构、RADOS、
  MON/OSD/MGR/MDS、CRUSH、PG、Pool、副本/纠删码池、CephFS/RBD/RadosGW、
  读写流程、初级实现细节，或要求默写/抽问/对比考点时使用。
---

# Ceph 设计原理与实现（初级）考试考点（周四复习）

按网课目录组织。用户刷课时发截图，逐步填充各章考点。

## 学习目标（培训完成后要做到）

| # | 目标 | 要求 |
|---|------|------|
| 01 | **组件构成** | 列举 Ceph 软件服务的组件构成 |
| 02 | **逻辑概念** | 解释 Ceph 封装的逻辑概念 |
| 03 | **组件作用** | 陈述各组件的主要作用 |
| 04 | **协作关系** | 串起各组件的协作关系 |
| 05 | **存储协议** | 区分 Ceph 对外提供的上层应用协议类型 |
| 06 | **检索资料** | 从 Ceph 开源社区检索学习资料 |

## 一、Ceph 的架构

### 1.1 诞生

Ceph 发展时间线：

| 年份 | 事件 |
|------|------|
| **2004** | **Sage Weil** 在加州大学圣克鲁兹分校（UCSC）的**博士论文**项目中创立 Ceph；以 **LGPLv2** 开源 |
| **2006** | 开源项目持续推进（GitHub） |
| **2011** | **Inktank** 公司成立，通过 **Inktank Ceph Enterprise** 提供专业支持 |
| **2014** | **Red Hat** 收购 Inktank（2014 年 4 月），Ceph 并入 Red Hat |
| **2018** | **Linux Foundation** 在柏林 Ceph Day 正式宣布成立 **Ceph Foundation**，支持开源项目 |
| **2022** | Red Hat 将存储产品组合与团队转给 **IBM**（含 Red Hat Ceph Storage、ODF、Rook、NooBaa） |

易考点：
- 创始人 **Sage Weil**，源于 UCSC 博士论文；开源协议 **LGPLv2**。
- 关键公司脉络：Inktank(2011) → Red Hat(2014 收购) → IBM(2022 接手存储)。
- 治理：**Ceph Foundation**（隶属 Linux Foundation，2018 成立）。

### 1.2 去中心化的分布式架构

**分层架构（自上而下）**：

| 层 | 组件 | 说明 |
|----|------|------|
| **CLIENT** | Block / Object / File | 块存储、对象存储、文件系统三种访问 |
| **接口层** | **LIBRADOS**（含 **RBD**→块、**RADOS GW**→对象）；**CEPH FS**（→文件） | 应用接入接口 |
| **核心层** | **RADOS** | 抽象的对象存储集群，所有存储类型的基础 |
| **服务/管理层** | **MON**、**MDS**、**OSD** | MON 维护集群状态；MDS 元数据服务(主要给 CephFS)；OSD 主要数据承载 |
| **存储后端/内核层** | **FileStore**→XFS(内核)→HDD；**BlueStore**→Block Cache(内核)→SSD | 旧 FileStore 用 XFS；新 BlueStore 针 SSD 优化 |
| **物理层** | HDD / SSD | 实际磁盘 |

**关键定义**：
- Ceph 是**开源的分布式存储系统**，**支持对象存储、块存储和文件系统**。
- **RBD**：块存储接口
- **RGW（RADOS GW）**：对象存储网关
- **CEPH FS**：文件级存储接口
- **RADOS**：抽象的对象存储集群
- **MON**：集群状态维护
- **MDS**：元数据服务
- **OSD**：对象存储设备，主要的数据承载

易考点：
- 三层主线：Client → RADOS 接口(RBD/RGW/CephFS) → 物理 OSD。
- 组件职责：MON=状态、MDS=元数据(CephFS)、OSD=数据。
- 两种后端：FileStore(XFS/HDD) vs BlueStore(SSD 优化)。
- 去中心化：无单点元数据瓶颈，靠 RADOS + CRUSH。

#### 1.2.1 去中心化的 IO 视图

**核心原则**：Client 与 OSD **直接 I/O**，无需每次操作查中心化路由表。

**组件关系（配图）**：
- **OSD**（顶部集群）：去中心化逻辑磁盘集群。
- **Client**（底部）：通过 **Librados API** 与 RADOS 集群通信。
- **Monitor**（右侧，3 个组成集群，无单点故障）：管理少量元数据 **OSDMap**（布局视图）。
- **Object I/O**：Client ↔ OSD 粗箭头直连（数据面）。
- **Failure reporting / map distribution**：Monitor ↔ OSD/Client 细箭头（控制面）。

**客户端初始化与读写流程**：
1. Client 首次启动时从 Monitor 获取 **OSDMap**。
2. 用哈希函数 + **CRUSH 算法**计算目标 OSD 位置。
3. 直接与目标 OSD 通信读写数据。

**故障处理**：
1. 检测到 OSD 故障 → Monitor 在 OSDMap 中标记该 OSD 不可用。
2. 更新后的 OSDMap **增量（灰度）下发**给所有 Client 和其他 OSD。
3. 各组件基于新 OSDMap 达成新共识，继续提供服务。

易考点：
- 去中心化 IO = **Client 直连 OSD**，Monitor 只管 OSDMap（控制面），不参与数据 IO。
- 首次取 OSDMap → CRUSH 算位置 → 直连 OSD 读写。
- 故障：Monitor 改 OSDMap → 增量下发 → 全集群重共识。
- RADOS = OSD + Monitor；Monitor 无单点故障。

### 1.3 核心问题（分布式存储五大核心问题）

| # | 核心问题 | 机制 |
|---|----------|------|
| 1 | **数据可靠性** | 多副本或纠删码 |
| 2 | **数据自恢复** | 始终自动保持设定的副本数 |
| 3 | **数据高可用** | 副本存储到不同主机上 |
| 4 | **集群扩展性** | 扩容时最小化数据迁移 |
| 5 | **寻址算法与一致性算法** | 用户文件查找 + 副本间一致性 |

记忆：可靠性(副本/纠删码) → 自恢复(自动保副本数) → 高可用(副本跨主机) → 扩展性(最小迁移) → 寻址+一致性算法。这五大问题正是 Ceph 各组件/算法要解决的（CRUSH 寻址、副本/EC 可靠、自修复、横向扩展）。

### 1.4 软件定义存储 SDS

**SDS 核心特征**（六边形围绕 SDS）：支持 **块/文件/对象** 三种存储；**灵活性（多种方式访问）**；**可扩展性（不过度预留）**；**低成本（任意硬件）**。

**存储生态韦恩图**（按存储类型分类）：
| 存储类型 | 代表产品 |
|----------|----------|
| 文件存储 | BeeGFS、**Gluster**、Lustre |
| 对象存储 | **MinIO** |
| 块存储 | Sheepdog |
| **统一存储（三者交集）** | **Ceph**（同时提供块/文件/对象） |

**意义与定义**：
- **软硬件解耦**：SDS 最重要意义在于软件与硬件解耦。
- **高三性**：高可靠性、高可用性、高可扩展性。
- **商用硬件**：用通用商用硬件，而非专用专有硬件。
- **理论基础**：基于严谨的理论架构与系统软件工程实现。

**实际收益**：
- **成本**：避免被专用硬件锁定（通常高成本）。
- **资源池化**：把物理硬件池化为受管「池」，按需构建，避免浪费。
- **统一接口**：为块/文件/对象提供统一存储接口。
- **Ceph**：SDS 的**集大成者**。

**重点1**：以前通过**硬件**实现冗余，现在通过**软件**实现冗余。

易考点：
- SDS 关键 = **软硬件解耦** + 商用硬件 + 高三性 + 统一接口。
- Ceph = 统一存储（块+文件+对象）集大成者。
- 冗余从硬件实现 → 软件实现。
- 文件/对象/块代表产品：Gluster/MinIO/Sheepdog。

## 二、Ceph 的概念与算法

> 本章两大主题：**逻辑概念**（Ceph 封装的逻辑概念）与**数学原理**（CAP、CRUSH 等算法）。

### 2.1 如何设计一个存储系统

**引入问题**：如何设计一个存储系统来保存手机的照片和视频？

**四次迭代尝试（循环 1→4）**：

| # | 尝试 | 做法 | 问题 | 关键缺失 |
|---|------|------|------|----------|
| 1 | **简单 NAS** | 一台服务器 + 一块 1TB 硬盘 | 硬盘会坏 → 照片丢失 | **数据可靠性**无法保证 |
| 2 | **RAID 1** | 两块硬盘做 RAID 1 冗余 | 媒体数据持续增长 | 1TB 容量很快**写满** |
| 3 | **纵向扩容（多组 RAID 1）** | 再加 N 块硬盘、N 组 RAID 1 | 单机 CPU/内存/盘位有限 → **纵向扩容瓶颈**；服务器宕机 → 照片不可访问 | **数据高可用性**受损 |
| 4 | **双控/硬件冗余** | Host A 故障则盘交给 Host B 继续服务 | 软硬件强耦合；物理上限；仍有纵向扩容瓶颈 | **高可扩展性**无法保证 |

**演进主线**：单盘 → RAID（可靠性）→ 多盘（容量）→ 双控（可用性）。

**核心结论**：单机纵向扩容受限于 CPU/内存/盘位；要同时满足**可靠性、高可用性、高可扩展性**，必须走向**分布式（横向扩展）**——这正是 Ceph 等分布式存储的动机。

易考点：四个尝试分别缺失 可靠性 / 容量 / 高可用 / 可扩展性；纵向扩容瓶颈 → 引出分布式横向扩展。

#### 2.1.1 进阶探索（路径 5–8，引出 Ceph 核心机制）

**路径 5（多副本）**：每张照片存两份，买多台主机各存一份；一台故障另一台仍可访问。
- 问题 6：**照片如何寻址存储到不同主机上？**

**路径 6（哈希分片）**：两台主机互备组成**逻辑卷**，N 台组成 N/2 个逻辑卷；每个逻辑卷分配**哈希范围**，用**简单哈希算法**按照片名哈希分布。
- 问题 7：扩容后逻辑卷增多、哈希范围变化，旧照片按哈希**找不到**，需全局搜索，低效。
- 问题 8：扩容后旧节点已有大量数据、新节点为空 → **容量与负载都不均衡**。

**路径 7（全量重哈希均衡）**：对所有照片重新哈希，迁移数据使各逻辑卷均衡。
- 问题 9：**数据迁移量巨大**，挤占业务流量资源。
- 问题 10：单个大文件（如 1TB 视频）可能超过单盘 1TB 容量，**单盘存不下整个文件**（配图：跨 server1/server2 的 Striped Volume）。

**路径 8（文件切片）**：把大文件切成小块（如 **4MB**），分别存入不同逻辑卷。
- 问题 11：播放时如何按正确顺序拼回原文件？**由谁切割、由谁管理、元数据存哪里？**（配图：file → obj1/obj2/obj3）

**对应 Ceph 核心机制**（后续章节解答这些问题）：
- 寻址/均衡 → **CRUSH 算法**（一致性哈希思想，扩容少量迁移）。
- 大文件切片 → **对象化（4MB object）**。
- 元数据管理 → **RADOS / Monitor / MDS** 等。

易考点：路径 6 哈希分片的扩容问题（找不到+不均衡）→ CRUSH 解决；大文件 → 切 4MB 对象；元数据由谁管 → 引出 Ceph 组件。

### 2.2 逻辑概念

**核心：Ceph 数据映射四级层级** `File → Objects → PGs → OSDs`

| 层级 | 映射逻辑 | 说明 |
|------|----------|------|
| **File** | — | 用户文件，被切分成多段 |
| **Objects** | `(ino, ono) -> oid` | 文件按 inode号(ino)、对象序号(ono) 切成多个对象，每个分配唯一对象 ID(oid) |
| **PG（Placement Group）** | `hash(oid) & mask -> pgid` | 对 oid 哈希后与掩码相与得到 PG ID；PG 是逻辑分组，用于扩展元数据管理 |
| **OSD** | `CRUSH(pgid) -> (osd1, osd2)` | CRUSH 算法由 pgid 算出落到哪些物理 OSD（按故障域分组），多 OSD 即副本冗余 |

**关键要点**：
- **工作流**：File → Objects → PGs → OSDs。
- **解耦**：在 Object 与 OSD 之间引入 **PG**，使系统可扩展、可再均衡，**无需为每个对象维护中心化查表**。
- **CRUSH**：把逻辑 PG 映射到物理 OSD，按**故障域**（如机架/主机）放置，保证高可用。

易考点：
- 四级映射及对应公式：`(ino,ono)->oid`、`hash(oid)&mask->pgid`、`CRUSH(pgid)->(osd1,osd2)`。
- PG 的作用 = 扩展元数据管理 + 解耦对象与 OSD。
- CRUSH 按故障域选 OSD → 副本分布保证高可用。

#### 2.2.1 四大逻辑组件详解

**User 层面**：File（用户文件）——照片、视频等非结构化数据存入 Ceph RADOS。

**Ceph 层面四大组件**：

| 组件 | 要点 |
|------|------|
| **Rados Object** | 由 **Object Name + Binary Data + Metadata(k/v)** 组成；是集群**存储基本单元(粒度)**；数据语义由客户端接口决定（RGW=对象、CephFS=文件、RBD=块） |
| **Pool** | 硬件资源的逻辑表示与约束，存对象的**逻辑分区**；是 **PG 的集合**(1.1…1.N)；每个对象只属于一个 Pool；客户端通过打开某 Pool 的句柄定义 I/O 上下文 |
| **PG（Placement Group）** | **冗余策略与管理对象的基本单元**；是对象的集合（一个对象属于 Pool 中某个 PG）；映射到一组 OSD（如 `up=[1,2,3]`），靠多副本或纠删码提供冗余；PG 内对象物理分布在该组 OSD 上 |
| **OSD** | 存数据的**物理或逻辑设备**；通常一个 OSD 对应一个物理盘；可为 HDD、SSD 或混合 |

**层级关系**：Pool（逻辑分区）→ 包含多个 PG → PG 映射到一组 OSD → 对象落在 PG 内 → 物理存在 OSD 上。

易考点：
- Object 三部分 = name + data + metadata；语义由接口(RGW/CephFS/RBD)决定。
- Pool = 逻辑分区 + PG 集合；对象唯一属于一个 Pool；I/O 上下文按 Pool 句柄。
- PG = 冗余基本单元；`up=[1,2,3]` 表示映射到一组 OSD，靠副本/纠删码冗余。
- OSD = 物理盘（HDD/SSD）。

### 2.3 CAP 原理

**定理**：在跨地域网络连接的分布式数据存储系统中，**不可能同时**提供以下三者中的**两个以上**：一致性(C)、可用性(A)、分区容错性(P)。

**三要素定义**：
| 要素 | 含义 |
|------|------|
| **一致性 Consistency** | 访问所有节点得到**相同的数据结果** |
| **可用性 Availability** | 所有节点保持高可用（系统持续运行并响应请求） |
| **分区容错性 Partition Tolerance** | 网络不可靠、出现分区（通信中断）时系统仍能继续运行 |

**韦恩图组合**：CA、CP、AP；三者交集 = 「无状态、非持久、不可用」（持久分布式系统无法三者兼得）。

**取舍场景**（data1、data2，初始 number=1；写请求把 data1 更新为 number=2，data1 需同步给 data2）：

| 选择 | 行为 | 结果 |
|------|------|------|
| **CP**（一致+分区容错） | data1 必须复制给 data2；网络不可靠可能失败；若 data2 未及时收到更新，客户端读 data2 时**优先一致性**（返回错误或阻塞至同步完成），不返回旧数据 | 牺牲可用性 |
| **AP**（可用+分区容错） | data1、data2 都须在有限时间内响应；data2 可能没收到同步，**返回现有数据**（可能旧值 number=1），与 data1 不一致 | 牺牲一致性（最终一致） |
| **AC**（可用+一致） | 仅在网络极佳、完全可靠时可行；data1 立即可靠发给 data2。但真实网络有丢包，做到 AC 通常意味着**不容忍分区（丢 P）**，节点须在同一区域，违背跨地域分布式 | 实际放弃 P |

**核心结论**：真实分布式系统**P 必备**（网络必然不可靠），故实际选择常为 **CP vs AP**——要强一致就牺牲分区时可用性，要高可用就接受最终一致。

易考点：CAP 三选二；P 在分布式下必选 → 实际是 CP/AP 取舍；CP 牺牲可用、AP 牺牲一致（最终一致）；AC 需完美网络→单区域→非分布式。

### 2.4 OSDMap

**查看命令**：
- `ceph osd dump`：查看 OSDMap 当前状态（epoch、fsid、时间戳、flags、各 pool 配置、OSD 状态）。
- `ceph osd tree`：查看集群物理/逻辑层级（root → host → osd），含 ID、CLASS(ssd)、WEIGHT、TYPE、NAME、STATUS、REWEIGHT。

**关键概念**：
- 集群表（Cluster Map）主要由两部分组成：
  1. **集群拓扑 + CRUSH 规则**（用于数据定位/寻址计算）。
  2. **所有 OSD 的身份与状态信息**。
- **epoch 单调递增**；集群所有变更（OSD 上下线）按逻辑时间线**串行处理**。
- OSDMap 的强一致性与高可用由 **Paxos 分布式共识算法**保证。
- Ceph 通过 **POOL** 对外提供存储服务（而非直接经 OSD/PG），故 OSDMap 记录所有用户创建 pool 的详细信息。
- OSDMap 跟踪每个 OSD 状态；大集群中编码后 Map 会膨胀；除首次全量 OSDMap 外，后续更新以**增量 map（diff）**下发以优化传输。

**OSDMap 数据结构（五大部分）**：

| 部分 | 字段 |
|------|------|
| **Cluster（集群元数据）** | epoch、fsid、created/modified 时间戳、blacklist(客户端黑名单)、EC 模板、flags(noout/noscrub…) |
| **CRUSH** | CRUSH Map、CRUSH Rules、root/rack/host/osd 层级、CRUSH 调试参数、各层级 crush_weight |
| **POOL** | pools 列表、pool_name 列表、pool_max、type(副本/EC)、opts、pg_num/pgp_num、object_hash 算法、quota_max_bytes/objects |
| **PG** | pg_temp、primary_pg_temp、pg_upmap、pg_upmap_item |
| **OSD** | osd_state、osd_info、osd_weight、osd_addrs、osd_uuid、osd_xinfo、osd_primary_affinity、max_osd |

**利用率阈值**：`full_ratio 0.95`、`backfillfull_ratio 0.9`、`nearfull_ratio 0.85`。

易考点：
- OSDMap 两部分 = 拓扑+CRUSH规则 / OSD身份状态。
- epoch 单调递增、变更串行；一致性靠 **Paxos**。
- 大集群用**增量 map** 优化传输。
- 对外服务经 **POOL**；OSDMap 五大结构（Cluster/CRUSH/POOL/PG/OSD）。

### 2.5 Paxos 共识算法

**定义**：Paxos 解决多节点如何对一个提案达成共识；提案号需**全局单调递增**。

**三种角色**：**Proposer（提议者）**、**Acceptor（接受者）**、**Learner（学习者）**。
- 角色特点：更新交互多、效率低、**强一致性、无脑裂**。

**Quorum（多数派）原理**：任意两个「超过半数节点」的子集必有交集 → 提供容错，解决 2PC 的无限等待、脑裂、数据一致性问题。

**防活锁**：选出一个主 Proposer（Leader）避免活锁。

#### Basic Paxos 流程（两阶段）
1. Proposer 选新提案号 $n$。
2. Proposer 向所有节点广播 `Prepare(n)`。
3. Acceptor 响应：若 $n > \text{minProposal}$，则 $\text{minProposal}=n$，返回当前 `(acceptedProposal, acceptedValue)`。
4. Proposer 收到多数响应后：若有返回的 acceptedValue，则用最高 acceptedProposal 对应的 acceptedValue 替换当前 value。
5. Proposer 广播 `Accept(n, value)`。
6. Acceptor 响应：若 $n \ge \text{minProposal}$，则 $\text{acceptedProposal}=\text{minProposal}=n$、$\text{acceptedValue}=\text{value}$，返回 minProposal。

#### Multi Paxos vs Basic Paxos
- **核心改进**：Multi Paxos 增加 **Leader 选举（选主）**。
- **心跳**：节点周期性心跳判断网络中是否存在主提议节点。
- **选举**：无主时，节点用 Basic Paxos 两轮（Prepare/Accept）广播竞选；多数同意则成为 Leader。
- **主权限**：选出后**只有 Leader 能提议**（直到其故障触发重新选举）。
- **请求转发**：其他节点收到客户端请求会**转发给 Leader**。
- **效率**：Leader 确定后，后续提议**不必每次重复 Prepare 阶段**，视为同一提案 ID 下的一系列批准。

易考点：
- 三角色 Proposer/Acceptor/Learner；Quorum 多数派交集原理 → 解决 2PC 脑裂/无限等待。
- Basic Paxos 两阶段 Prepare/Accept；提案号单调递增。
- Multi Paxos = Basic + Leader 选举；选主后省 Prepare，效率提升。
- 特点：强一致、无脑裂，但交互多、效率低。

### 2.6 哈希散列算法（Object → PG，CRUSH 第一阶段）

**映射总流程**：`Object → [第一阶段] → PG → [第二阶段] → OSDs`。本节讲第一阶段（Object → PG）。

**哈希函数**：把输入映射为定长输出。好哈希 = ① 计算快 ② 冲突率低。

**ceph_stable_mod 流程**：
1. 对对象名用 `ceph_str_hash_rjenkins` 哈希 → **32 位整数**。
2. 再用 `ceph_stable_mod` 做**位掩码**运算：
```c
if ((hash & (2^n - 1)) < pg_num)
    return (hash & (2^n - 1));
else
    return (hash & (2^{n-1} - 1));
```
3. 输出：对象归入某 Pool 的某个 PG（如 Test-Pool 的 PG 1.1/1.2/1.3/1.4）。

**stable_mod 的作用与优势**：
- 用 32 位哈希与 pg_num 做位掩码。
- 保证**低位相同**的对象落入**同一 PG** → 为 **PG 分裂（扩 PG 数）**打基础。

**非 2 的幂的 PG 数处理（示例 pg_num=12, n=4, 2^4=16）**：
- PG 0~11 存在；若掩码结果落在 12~15（不存在的 PG），用第二逻辑 `hash & (2^{n-1}-1)` 重映射回 `[0, 2^{n-1}-1]`（即 0~7）。

**PG 分裂**：pg_num 增长（如 $2^4 \to 2^6$）时，因低位一致，1 个旧 PG 可分裂成 3 个新 PG。

易考点：
- 第一阶段 Object→PG 用 `ceph_str_hash_rjenkins` + `ceph_stable_mod`。
- stable_mod 公式（掩码 + 回退逻辑）；保证低位同→同 PG，为 PG 分裂打基础。
- 非 2 幂 pg_num 的回退映射；PG 分裂 $2^4\to2^6$ 一分三。

#### 2.6.1 PG 分裂二进制详解与重要提示

**重要提示**：创建存储池时 **pg_num 必须指定为 2 的幂**，否则各 PG 中对象数**不均衡**！

**非 2 幂处理（pg_num=12, n=4, 2^4=16）**：
- $[0, 2^{n-1}-1]=[0,7]$ 一定存在。
- 掩码结果落在 12~15（不存在的 PG）时，用 `else` 回退到 $[0,7]$。

**PG 分裂（$2^4 \to 2^6$）二进制表**：分裂时多看更高 2 位，1 个旧 PG 分成 4 个新 PG：

| 哈希高位 | 落点 |
|----------|------|
| `...00 X3X2X1X0` | 留在原索引 `X` |
| `...01 X3X2X1X0` | `1*16 + X` |
| `...10 X3X2X1X0` | `2*16 + X` |
| `...11 X3X2X1X0` | `3*16 + X` |

**结论**：分裂时只看新增的高位即可把对象从 1 个 PG 干净地重分布到多个新 PG，**无需全量迁移**。

易考点：建池 pg_num 须为 2 的幂（否则不均衡）；PG 分裂靠多看高位（00/01/10/11）实现一分多、不全量迁移。

### 2.7 一致性哈希算法 CRUSH（PG → OSD，第二阶段）

**映射总流程**：`Object →(第一阶段)→ PG →(第二阶段)→ OSDs`。本节讲第二阶段（PG → OSDs）。

**CRUSH 核心原则**：
- **按权重分布**：按各盘 **weight** 分布 PG，追求近似均匀的概率分布。
- **Cluster Map**：层级化集群地图，表示可用存储资源。
- **Placement Rules（放置规则）**：定义数据分布策略，指定：使用的集群拓扑、**故障域**（rack/host）、副本数、副本放置约束。

**算法对比（CRUSH 内部）**：
- **Straw**：把所有元素比作「稻草」，对给定输入为每个元素随机算一个长度，选最长者。
- **Unique**：执行效率最高，但抗结构变化能力最差。
- **Straw**：效率较低，但抗结构变化能力最强，保证集群扩缩容时**数据迁移最少**。

| 指标 | unique | list | tree | straw |
|------|--------|------|------|-------|
| 时间复杂度 | O(1) | O(N) | O(log N) | O(N) |
| 增加元素 | 差 | 最好 | 好 | 最好 |
| 删除元素 | 差 | 差 | 好 | 最好 |

- **注**：相同 pgid 和 map 下，一致性哈希**总输出相同 OSD 列表**。

**执行流程**：
1. 输入：PG 1.1
2. 哈希：`crush_hash32_rjenkins1_2(pgid, poolid)` → `pps`（32 位整数）
3. CRUSH（straw2）：`CRUSH(ruleid, pps, pool_size, osd_reweight)`
4. 选择：`take(root)` → 选 root → `emit` → 输出 `[1, 5, 9]`（OSD ID）

**Straw 伪代码**：
```python
max_x = -1; max_item = -1
for each item:
    x = hash(input, r)
    x = ln(x / 65536) / weight
    if x > max_x:
        max_x = x; max_item = item
```

**层级选择示例**（拓扑：root→rack1/2/3→各 2 host→各 2 OSD）：
1. `take(root)` → root
2. `select(3, rack)` → rack1、rack2、rack3
3. `select(1, host)` → host1、host3、host5
4. `select(1, osd)` → osd.1、osd.5、osd.9

→ 副本跨不同故障域（机架/主机）分布，保证高可用。

易考点：
- CRUSH 按权重分布；放置规则指定故障域/副本数。
- 算法对比：unique 最快但抗变差；straw 抗变最强、迁移最少（增删元素均最好）。
- 相同 pgid+map → 相同 OSD 列表（确定性）。
- straw 选最长；层级 take/select 逐层按故障域选副本。

### 2.8 分布式事务（2PC / 3PC）

**核心概念**：事务中所有参与者必须**要么都执行、要么都不执行**。若部分执行部分不执行 → **不一致** + **脑裂**。

#### 2PC（两阶段提交）
- **阶段1 Prepare**：协调者问参与者集群能否执行；参与者判断是否可执行但**不提交**；协调者收反馈（成功→提交；失败/超时→回滚）。
- **阶段2 Commit**：协调者发提交/回滚通知；参与者提交并返回结果。

**2PC 缺点**：
- **同步阻塞**：过程中参与者等待响应，无法做其他操作。
- **单点故障**：只有一个协调者，宕机则整个流程无法继续。
- **脑裂**：阶段2 若网络分区，部分参与者收到提交并执行、部分没收到 → 数据不一致。
- **太过保守**：协调者超时后会直接发起回滚/中止。

#### 3PC（三阶段提交）
- **关键区别**：3PC 允许参与者**超时操作**（2PC 不允许）。
- **阶段1 Can-Commit**：协调者询问能否提交；反馈为否或超时则中止。
- **阶段2 Pre-Commit**：协调者再询问；参与者执行但**不最终提交**。
- **阶段3 Do-Commit**：协调者发最终提交；参与者提交并返回。

**3PC 仍存在的问题**：
- **不一致未根治**：3PC 未完全解决数据不一致。
- **网络分区场景**：分区致协调者与参与者通信失败时：协调者可能发 abort，收到者回滚；但**收不到的参与者最终超时后会自行提交** → 不一致。

**核心矛盾**：3PC 试图解决 2PC 的阻塞/故障问题，但两者在**网络故障下都难以保证绝对一致**（这正是 Paxos 等共识算法的动机，见 2.5）。

易考点：
- 2PC 两阶段 Prepare/Commit；缺点=同步阻塞+单点故障+脑裂。
- 3PC 三阶段 Can-Commit/Pre-Commit/Do-Commit；引入超时，但仍有一致性问题。
- 2PC/3PC 不足 → 引出 Paxos（多数派共识，解决脑裂/无限等待）。

## 三、Ceph RADOS 子系统

> 第三章聚焦 RADOS 各子系统组件及其协作。

### 3.1 MON 子系统
> 待补充：Monitor 组件构成、作用、用 Paxos 维护 OSDMap 等。

### 3.2 MGR 子系统

**背景与动机**：MGR 引入前，OSD 容量、PG 状态等统计信息由 Monitor(Mon) 处理，给 Mon 带来沉重负担——而 Mon 的稳定与效率对整个集群可用性至关重要。MGR 的目标是**把非关键、统计密集的功能从 Mon 卸载**，提升性能与可扩展性。

**四大核心功能**：
1. **减轻 Mon 负担**：分担并扩展部分 Monitor 功能。
2. **监控 OSD 容量与 PG 状态**：采集 OSD 容量与 PG 状态统计；计算单个 pool 及整体集群容量；周期性上报给 Mon 以维护更新 **PGMap**。
3. **异常告警（异常检测）**：基于采集的集群信息与监控指标，异常时生成告警。
4. **Python 框架执行模块**：允许用户按需创建自定义监控模块；对外提供 Ceph 监控数据与性能指标（如 `iostat`）；处理集群数据**再均衡(balancer)**与管理告警信息。

**模块概览（ceph-mgr）**：
- **MgrStandby**：高可用模块。
- **PyModules**：StandbyPyModules / ActivePyModules。
- **内置模块**：Dashboard、Alerts、DiskPrediction、RESTful、Prometheus、Iostat、Crash、Ansible、Zabbix、Balancer。
- **系统组件**：**DaemonServer**（CLI 与上报）、**ClusterState**（维护集群状态）。

**高可用（HA）机制**：
- **主备模式（Active-Standby）**：多节点可同时运行多个 Mgr 进程，但**只有一个主 Mgr 处于 Active** 提供服务。
- **Standby**：备 Mgr 维持心跳，随时准备在主故障时接管。
- **Mon 的作用**：维护带版本号的 **MgrMap(epoch)**，负责指定哪个 Mgr 为主。
- **部署**：通常 MGR 实例与 Monitor 实例部署在同一节点。

**插件与通知框架**：
- MGR 的 Python 插件框架实现 **Notify 机制**。
- 用户可按需实现自定义功能插件进行集群管理。
- Notify 机制可对接外部监控系统与框架。

易考点：
- MGR 由来 = 卸载 Mon 的统计负担（OSD 容量/PG 状态 → PGMap）。
- 四大功能：减负、监控容量/PG、告警、Python 框架（balancer/iostat 等）。
- HA = Active-Standby；Mon 用 MgrMap 指定主；通常与 Mon 同节点。
- 内置模块：Dashboard/Prometheus/Balancer/Zabbix 等。

### 3.3 OSD 子系统（智能存储）

**核心角色**：OSD 是「磁盘守护者」。用 **OSDMap**、**CRUSH 算法**、**PG 状态机**自主完成数据分布；保证副本间强一致、自校验纠错、磁盘故障自恢复、自管理——这种自治是大规模智能存储的基础。

**信息上报**：OSD 周期性采集磁盘容量与 PG 状态，上报给 **Mgr**，由 Mgr 汇聚集群数据。

**网络双平面**：OSD 流量分两个平面——**Public 服务网络平面** 与 **Cluster 网络平面**。

**心跳机制**：
- **OSD-OSD 心跳**：对等 OSD 在两个网络平面维持心跳；网络路径阻塞则上报 MON；若**多个不同故障域的 OSD** 都报告某 OSD 异常，MON 在 OSDMap 中标记其为 **Down**。
- **OSD-MON 心跳**：OSD 与 MON 在 Public 平面维持心跳；连接丢失，MON 直接在 OSDMap 标记该 OSD 为 **Down**。

**承载内容**：OSD 承载 CRUSH 分配的 **PG** 及 PG 内存储的 **Rados Object**。

**PG 元数据**：
- **PGInfo**：记录 PG 集合历史（**PastIntervals**）、**Peering** epoch 逻辑时间点、**Recovery** 进度指针。
- **PGLog**：用单调递增逻辑时钟 **eversion（epoch + version）**。

**数据事务**：PG 把用户数据封装为 **Transaction**，通过多副本或纠删码等冗余策略，经网络发到多个 **ObjectStore** 节点并落盘。

**Peering 日志**：每次用户数据修改记为 `eversion PGLog`（记录哪个操作改了哪个对象）并更新 PGInfo；该元数据与用户数据打包进事务落盘，为 **Peering** 过程做准备。

**强一致性**：PG 状态机保证副本一致，尤其在 OSD 恢复或集群扩容时通过 **Peering** 算法保证。

**OSD 内部分层架构**：

| 层 | 组件 |
|----|------|
| **Msgr（通信层）** | Public Msgr、Cluster Msgr、Cluster/Public heartbeat Msgr、MgrClient、MonClient |
| **OSD（管理/控制层）** | handle/share OSDMap、Capacity/PG Stat、cls API、NetworkHeartbeat、ThreadHeartbeat、OSD Superblock、OSD Shard Queue、Scrub/Recovery Reservation |
| **PrimaryLogPG（PG 层）** | PG、PastIntervals、PGLog、Peering、PG History、PGTransaction、PG Info、PG Scrub |
| **PGBackend（一致性/后端层）** | ECBackend、ReplicatedBackend、ObjectTransaction |
| **ObjectStore（存储引擎层）** | FileStore、BlueStore、KStore、MemStore、RocksDB |

易考点：
- OSD=磁盘守护者；靠 OSDMap+CRUSH+PG 状态机自治（分布/强一致/自恢复）。
- 双网络平面（Public/Cluster）；两类心跳（OSD-OSD 多故障域举报→Down；OSD-MON 丢连→Down）。
- PG 元数据 PGInfo(PastIntervals/Peering epoch/Recovery 指针) + PGLog(eversion=epoch+version)。
- 事务封装 + 多副本/EC 落盘；PGLog 为 Peering 准备；Peering 保证恢复/扩容时一致。
- OSD 内部五层：Msgr/OSD/PrimaryLogPG/PGBackend/ObjectStore(BlueStore 等)。

#### 3.3.1 分布式副本间一致性问题

**问题背景**：
- **目标**：提升数据高可用、防单点故障。
- **方法**：数据多副本存在不同主机的磁盘上。
- **核心指标**：保证副本间**一致性**是评估存储系统的核心、开发时的重点。

**场景（试想一下）**：文件 3 副本存于 3 台主机（Host1/2/3），客户端向 3 台发写请求：
- Host1：写成功，返回成功。
- Host2：写成功，返回成功。
- Host3：**无响应**。两种原因：
  1. 请求未到达，Host3 宕机。
  2. 请求到达，但 Host3 断电（写前断电 / 写一半断电 / 写完未响应就断电）。

**Host3 恢复上线后必须解决的三个问题**：
1. **如何达成一致？**
2. **如何解决分歧？**
3. **如何解决写了一半？**

易考点：多副本为高可用→核心是一致性；Host3 无响应的两种原因（未到+宕机 / 到了+断电含写一半）；恢复后三问=达成一致/解决分歧/解决写一半（引出 Peering 与 PGLog）。

#### 3.3.2 强一致算法 PG Peering（PG 状态机）

**概念背景**：
- RADOS **不假设**不同 Map（OSD Map）间数据分布是连续的；仅当 Map 变更影响到当前 PG 时才建立一致性视图。
- **Peering 算法**：构建 PG 内容的一致视图，恢复正确的数据分布与副本。依赖 OSD 主动复制三样：
  - **PGLog**：操作记录。
  - **PG Content Info**：PG 应含哪些对象及版本的状态。
  - **PastIntervals**：该 PG 历史上 OSD 集合的变迁。
- **状态机复制**：保证多副本一致用有限状态机行为——若每个副本收到**完全相同的有序输入序列(log)**，回放后内部数据必然相同。

**保证一致的两件事**：
1. **严格有序 + 唯一「指挥官」**：所有消息严格有序；PG 对每次更新在 log 中分配**单调递增的 version**。为此同一时刻**只有一个「指挥官」能发指令**。
   - **Primary 选举**：`up` 或 `acting` 集合中**第一个健康在线的 OSD** 选为 **primary**，其余为副本。
2. **权威日志选择**：Peering 算法保证无论冗余级别内发生何种异常，都能选出**权威日志(authoritative log)** 并解决「日志分叉」，使所有副本日志严格一致。

**PastIntervals（epoch 区间示例）**：
| 区间 | up | acting | Primary |
|------|-----|--------|---------|
| Epoch 1–100 | [1,2,3] | [1,2,3] | OSD.1 |
| Epoch 101–200 | [1,2,4] | [1,2] | OSD.1 |
| Epoch 201–300 | [1,3,4] | [1] | OSD.1 |

- OSD.1 始终为主；其他 OSD(2/3/4) 在集合中进出变化。

**PG 状态机流程**（起点 `Initial → Reset → Started`，按 `is_primary` 分流）：
- **Primary**：→ `Primary` → `Peering`（执行 `GetInfo`/`GetLog`/`GetMissing`/`WaitUpThru`）；与副本交换消息（发 `MOSDPGQuery: INFO/LOG`，收 `MNotifyRec`/`MLogRec`）→ Peering 完成进入 `Active`/`Activating`；后续恢复态 `Recovering`/`WaitLocalRecoveryReserved` 等。
- **Replica**：→ `Stray`；响应主 `MNotifyRec`/`MLogRec`/`MInfoRec`；被主 `Activate` 后进入 `ReplicaActive`。

**小结**：Peering 从 `PastIntervals` 选 OSD；`acting`（实际参与）可能 ≠ `up`（CRUSH 认为应在的）；primary 会向 MON 上报这些差异。

易考点：
- Peering 依赖 PGLog / PG Info / PastIntervals；状态机复制=相同有序 log→相同数据。
- 一致两件事：严格有序+唯一指挥官（primary=up/acting 第一个健康 OSD）；权威日志解决分叉。
- up vs acting 可不同；primary 向 MON 上报差异。
- Primary 走 Peering(GetInfo/GetLog/...)→Active；Replica 走 Stray→ReplicaActive。

### 3.4 LibRADOS（librados 与 osdc）

**客户端接口**：上层应用访问 Ceph 的入口，支持**多事务原子操作**和扩展对象数据（**xattr/kv**）。

**模块组成**：客户端由 **Librados** 和 **Osdc** 组成；**Cls** 扩展模块基于它们扩展已有接口。

**Librados**：RADOS 对象存储的接口库，提供基本操作：建/删 pool，建/删/读/写对象。
- **Rados**：`connect`、`pool_create`、`pool_lookup`、`pool_list`、`cluster_fsid`、`shutdown`、`pool_delete`、`ioctx_create`、`get_pool_stats`、`cluster_stat`。
- **IoCtxImpl**：某个 pool 的上下文信息，**一个 pool 对应一个 IoCtxImpl 对象**；处理该 pool 内同步/异步对象操作（`create`/`write_full`/`read`/`remove`/`setxattr`/`rmxattr`/`append`/`write`/`stat`/`trunc`/`getxattr`/`watch/notify`）。

**OSDC 核心功能**：
- 封装操作数据。
- 拆分对象（Striper 负责条带）。
- 获取 **OSDMap**。
- 哈希计算对象所属 **PG**。
- 用 **CRUSH** 计算 PG 目标 **OSD 地址**。
- 发网络请求并处理超时。
- **Objecter**：`_calc_target`、`ObjectOperation`、`handle_osd_map`、`OSDSession`、`OSDOp`、`ObjectCacher`。

**上层应用（RGW / CephFS / RBD）**：三大存储协议都构建在 **librados + osdc** 之上；协议层无需关心冗余策略或底层对象逻辑，可靠/可用/可扩展由 RADOS 管理，简化开发。

易考点：
- 客户端 = Librados + Osdc（+ Cls 扩展）；接口支持原子事务 + xattr。
- 一个 pool = 一个 IoCtxImpl；Librados 管 pool/对象增删读写。
- OSDC 流程：封装→拆对象→取 OSDMap→哈希算 PG→CRUSH 算 OSD→发请求/超时处理。
- RGW/CephFS/RBD 都基于 librados+osdc，冗余与底层由 RADOS 管。

### 3.5 子系统间的协作关系

**整体架构（配图）**：
- **MON**：Paxos 状态机（mon.a/b/c，各维护 epoch、osdmap）。
- **MGR**：Active-Standby（mgr.a/b/c）。
- **OSD**：PG 状态机，按 CRUSH map 层级 `root → rack → host → osd` 组织。
- **客户端栈**：`osdc`、`librados`；接入协议 `rgw`(对象)/`rbd`(块)/`cephfs`(文件)；用户经客户端访问。

**① 状态上报与 Map 更新**：
1. **OSD** 周期性把自身容量和主 PG 状态上报给主 **MGR**。
2. **MGR** 汇总所有 OSD 的容量与 PG 信息，发给 **MON**。
3. **MON** 通过 **Paxos** 共识算法更新 **PGMap**。

**② 故障检测与传播**：
- a. **OSD-OSD** 与 **OSD-MON** 通过网络心跳互相监控。
- b. OSD 异常/故障时，**MON** 经 Paxos 更新 **OSDMap** 并标记该 OSD 为 **down**。
- c. MON 把更新后的 OSDMap 下发给**部分 OSD**。
- d. OSD 之间再通过心跳**互相传播** OSDMap。
- e. OSD 收到 OSDMap 后交给 **PG 状态机**。
- f. 若 CRUSH 算出的 OSD 因故障而变化，PG 进入 **Peering**；主与副本协商解决分歧、达成一致。
- g. PG Peering 完成后开始**数据恢复**。

**③ 客户端数据访问路径**：
1. 用户经客户端访问云存储。
2. 客户端从 **MON** 取 **OSDMap**；用哈希函数 + **CRUSH** 算出 **PGID** 和具体 **OSD**；经服务网络把请求发给 **Primary OSD**。
3. Primary OSD 从网络收到对象数据，封装为 **PG 事务**，再（按多副本或纠删码）转为 **Object 事务**处理落盘。

易考点：
- 状态上报链：OSD→MGR→MON(Paxos 更新 PGMap)。
- 故障链：心跳检测→MON 改 OSDMap(down)→增量下发→OSD 互传→PG 状态机→CRUSH 变则 Peering→恢复。
- 客户端访问：取 OSDMap→哈希算 PGID→CRUSH 算 OSD→发 Primary OSD→封装 PG 事务→副本/EC 落盘。
- 三大协议 rgw/rbd/cephfs 都经 librados+osdc。

## 四、Ceph 存储协议

### 4.1 对象存储协议 RGW（S3/Swift）

**架构栈（自上而下）**：
| 层 | 组件 |
|----|------|
| API | S3 兼容 API / Swift 兼容 API |
| 网关 | **radosgw** |
| 库 | **librados** |
| Ceph | OSDs / Monitors |

**对象云存储工作流**：客户端（Web 浏览器 / App / SDK）经 **HTTP** 访问 → Bucket 内含多个 Object。

**对象存储特点**：
- **效率高**：扁平化结构，不受复杂目录/文件夹系统性能下降影响。
- **访问方便**：支持 HTTP(S)、RESTful API 调用取数；新增 NFS、SMB 支持。
- **成本低**。
- **可扩展性高**：可扩展到数十/数百 EB，充分利用高密度存储。
- **适用场景**：静态或不常变文件（图片、视频、文档）。

**语义定义**：
- **Bucket（桶）**：对象存储的「桶」，提供**扁平命名空间**。
- **Object（对象）**：三部分 = **Key**(名称/ID) + **Data**(实际内容) + **Metadata**(元信息，如对象大小)。

**HTTP 接口与 API**：
- HTTP 接口：任何联网设备随时随地上传/下载；任何支持 HTTP 的客户端可访问。
- API 速览：
  | 接口 | 作用 |
  |------|------|
  | `PUT` | 上传对象 |
  | `GET` | 下载对象 |
  | `HEAD` | 获取对象元信息 |
  | `DELETE` | 删除对象 |
  | `MultiPartUpload` | 分块上传，优化大对象弱网环境 |
  | `ListPrefix` | 分页列举对象 |

易考点：
- RGW 栈：S3/Swift API → radosgw → librados → OSD/MON。
- 对象 = Key + Data + Metadata；Bucket = 扁平命名空间。
- 特点：扁平高效、HTTP(S)/RESTful（+NFS/SMB）、低成本、可扩 EB、适合静态文件。
- API：PUT/GET/HEAD/DELETE/MultiPartUpload/ListPrefix。

### 4.2 文件存储协议 CephFS（NAS）

**架构栈**：
- 顶层：CephFS 分布式文件系统。
- 客户端接入：
  - **Linux**：内核客户端(Kernel) 或 用户态客户端(FUSE)
  - **MacOS**：NFS-ganesha
  - **Windows**：SAMBA/CIFS
- 中间库：客户端经 **Libcephfs**（位于 **Librados** 之上）。
- 核心组件：**OSDs**、**MDSs**、**Monitors**。

**操作流程**：
- **元数据操作**（`open`/`mkdir`/`listdir`）：客户端直接与 **Active MDS** 交互。
- **数据操作**（`read`/`write`）：客户端直接与 **OSD** 交互，**绕过 MDS** 以获高性能。
- **MDS 冗余/扩展**：Active MDS 与 Journal 交互；Standby MDS 接收「元数据交换」；另一 Active MDS 做「元数据变更」并「Journal Flush」到存储。

**关键技术概念**：
- **设计哲学**：文件系统用层级树结构组织目录，符合人类思维，便于数据组织。
- **复杂度对比**：文件系统访问接口远多于对象存储；分布式文件系统的挑战是**保证 POSIX 语义正确性的同时最大化元数据可扩展性**。
- **Ceph MDS**：
  - 通过状态机实现**动态子树(dynamic subtrees)**。
  - 管理所有元数据（文件/目录属性、权限、位置）。
  - MDS 状态机维护目录结构，保证多 MDS 节点修改时的一致性。

**CephFS 客户端类型**：
- **原生客户端**：Linux 内核态(Kernel) + 用户态(FUSE)。
- **NFS 协议客户端**：Linux/Unix 类系统间文件共享。
- **SMB/CIFS 协议客户端**：Linux 与 Windows 间文件共享。

**POSIX 接口**：`open`/`read`/`seek`/`link`/`close`/`write`/`truncate`/`unlink`/`create`/`fallocate`/`flock`/`stat`/`opendir`/`mkdir`/`readdir`/`symlink`/`rmdir`/`fsync`/`readlink`。

**动态子树元数据分布**：把目录树不同部分分配给不同 MDS（如 MDS 0/1/3/4），通过**分区目录树到多个 active 服务器**来扩展元数据能力。

易考点：
- 架构栈：客户端(Kernel/FUSE/NFS-ganesha/SAMBA)→Libcephfs→Librados→OSD/MDS/MON。
- 元数据走 MDS、数据直走 OSD（绕 MDS 高性能）。
- MDS 用动态子树状态机管元数据、保多节点一致；动态子树分区目录树实现元数据扩展。
- 客户端类型：原生(Kernel/FUSE)、NFS、SMB/CIFS。
- 挑战：POSIX 语义正确 + 元数据可扩展。

### 4.3 块存储协议 RBD（iSCSI / 磁盘）

**块存储五大特点**：
1. **定长数据块**：以固定大小单元读写（通常 512B 或 4KB），高效精确。
2. **随机访问**：支持随机访问，可直接读写任意数据块，无需顺序访问；适合文件系统、数据库等负载。
3. **快照与克隆**：支持快照和克隆，用于备份、恢复、复制，提升可靠性与可用性。
4. **虚拟化支持**：块设备可虚拟化，映射为虚拟机或容器的存储。
5. **iSCSI 支持**：支持 iSCSI 标准协议，直接经网络访问 RBD。

**架构栈**：

| 栈 | 层级 |
|----|------|
| **Kubernetes 栈** | Kubernetes → **ceph-csi**(CSI) →（Kernel 模块 / rbd-nbd / librbd）→ RADOS 协议 → OSDs/Monitors |
| **OpenStack 栈** | OpenStack → **libvirt** → QEMU → **librbd** → **librados** → OSDs/Monitors |

**iSCSI 网络访问**：
- 集群网络（可选）→ OSD 层（OSD1/2/3/N）→ **RBD Image** → 公共网络 → **iSCSI GW RBD 模块（网关）** → **iSCSI Initiator**（各操作系统）。
- Initiator 经公共网络、通过 RBD Image 逻辑访问 OSD。

易考点：
- RBD 五特点：定长块(512B/4KB)、随机访问、快照克隆、虚拟化映射、iSCSI。
- K8s 走 ceph-csi（Kernel/librbd/rbd-nbd）；OpenStack 走 libvirt→QEMU→librbd→librados。
- iSCSI 经网关（iSCSI GW RBD 模块）让 Initiator 经公共网络访问 RBD Image→OSD。

## 五、Ceph 开源社区
> 待补充：预计涵盖社区检索学习资料、版本/文档获取等。

## 复习互动方式

1. **默写**：某一整块（如 CRUSH 流程 / 读写流程 / 三大接口）
2. **对比**：CephFS vs RBD vs RadosGW；副本池 vs 纠删码池
3. **抽问**：随机出简答/判断
4. **纠错**：用户口述，指出遗漏与易错点

回答时：先结论再展开；中文作答；紧扣本 skill，不擅自扩展成无关百科。

---

**进度说明：** 目前为骨架。用户发课程目录后，按章节替换上方内容；发各章截图后逐章填入考点。
