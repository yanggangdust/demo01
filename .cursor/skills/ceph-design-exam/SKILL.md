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

## 三、Ceph RADOS 子系统

### 3.1 （待补充）
> 待补充：RADOS 概述、组件协作、读写流程等。

### 3.2 OSDMap

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

### 3.3 Paxos 共识算法

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

## 四、Ceph 存储协议
> 待补充：预计涵盖 CephFS / RBD / RadosGW 三大接口及协议类型。

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
