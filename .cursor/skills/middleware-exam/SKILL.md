---
name: middleware-exam
description: >-
  通用中间件（基础）考试考点复习资料。当用户询问中间件种类与作用、Web中间件
  (Nginx/HAProxy/Tomcat)、缓存中间件(Redis/Memcached)、分布式中间件
  (Zookeeper/Kafka)，或要求默写/抽问/对比考点时使用。
---

# 通用中间件（基础）考试考点（周四复习）

按网课目录四章组织。用户刷课时发截图，逐步填充各章考点。

## 一、通用中间件种类与作用

> 本章目录：认识中间件 → 中间件架构 → 中间件种类。

### 1.1 认识中间件

#### 什么是中间件（MiddleWare）

**中间件概念**

- 一类**连接系统软件与应用软件**的软件
- 使**不同软件组件之间**能够通信
- 应用软件可借助中间件，在**不同技术架构**之间共享信息与资源

**广义中间件**

- 凡**不直接为终端客户提供业务价值**的软件，均可视为中间件
- 例：**Nginx**、**MySQL**

**广义中间件分类（四大类）**

| 类别 | 典型产品 |
|------|----------|
| **Web 中间件** | Nginx、Tomcat、HAProxy |
| **数据库和缓存中间件** | Redis、Memcached |
| **消息和分布式中间件** | Zookeeper & Kafka、MQ（消息队列） |
| **容器中间件** | Docker、k8s（Kubernetes） |

> 课件图示：上述四类均归入「中间件」范畴。

**狭义中间件**

- 特指位于**基础设施层软件**与**业务系统软件**之间的
- 具体软件、类库、框架

**易考点**

| 对比项 | 广义 | 狭义 |
|--------|------|------|
| 范围 | 很广，含 Nginx、MySQL 等 | 仅中间层专用组件 |
| 判断依据 | 是否直接面向终端业务价值 | 是否处于基础设施与业务系统之间 |

### 1.2 中间件架构

#### 中间件架构角色 — 狭义的中间件

典型后端分层架构（请求自上而下，数据自下而上）：

```
流量接入
  ↓
[代理服务器集群]  nginx / haproxy / lvs / keepalived   ← 绿色·通用组件
  ↓ 流量代理
[网关]  Gateway
  ↓
[业务工程]  消息中心、运营中心、鉴权中心、服务模块、开放平台、
             对象存储、音频项目、管理中心、在线编辑 …   ← 蓝色·各系统不同
  ↔ [mq 消息队列集群] / [kafka 消息队列集群]  （消息传递、异步处理）
  ↓
[云原生编排系统]  k8s 集群   ← 业务与队列运行其上的底座
  ↓
数据存储
  · 关系数据库集群 — pg、mysql、Rdb
  · 缓存数据库集群 — Zookeeper、etcd
  · 键值数据库集群 — Memcached、redis
```

**图例（课件）**

| 颜色 | 含义 |
|------|------|
| **绿色 — 通用组件** | 标准基础设施中间件（各系统基本一致） |
| **蓝色 — 业务工程** | 各业务/custom 逻辑（**不同系统间主要变化点**） |

**三层要点（课件原文）**

1. **流量入口**：前端**流量接入**，经 **Web 中间件**（Nginx、HAProxy、LVS 等）进入；一般作为**应用代理或流量转发器**。
2. **内部处理**：经转发到达后端应用；同时应用利用**消息队列**与其他系统或组件通信，并与**数据库**交换数据。
3. **可定制部分**：不同系统之间，主要差异在**业务工程（蓝色部分）**；中间件（绿色通用组件）大体一致。

**各层中间件速记**

| 层次 | 组件 |
|------|------|
| 流量接入/代理 | Nginx、HAProxy、LVS、Keepalived |
| 流量代理 | Gateway（网关） |
| 消息 | mq 消息队列集群、Kafka 消息队列集群 |
| 编排 | 云原生编排系统（k8s 集群） |
| 关系数据库集群 | pg（PostgreSQL）、MySQL、Rdb |
| 缓存数据库集群 | Zookeeper、etcd |
| 键值数据库集群 | Memcached、Redis |

#### 中间件架构角色 — OP 双中心

**建设目标**

- 实现**远程容灾**
- 实现**全流量高可用**

**架构特点（课件）**

| 特点 | 说明 |
|------|------|
| **双中心同担流量** | 两个中心均可同时承载业务流量 |
| **GSLB 全网负载** | 通过 **GSLB**（全局服务器负载均衡）做全网流量调度 |
| **容器化部署** | 应用全部容器化部署，按业务量**弹性伸缩**、**故障迁移** |
| **中间件集群化** | 中间件集群部署；**部分中间件两中心共享** |
| **统一 DevOps** | 两中心共用同一套 **CI/CD 工具** 与 **监控平台** |

**架构分层（双中心对称 + 中间共享）**

```
用户/客户端
  ↓
Local DNS → Cloud GTM（全局流量管理）
  ↓                    ↓
[中心 A]              [中心 B]          ← 对称双中心
  反向代理 & 负载均衡（HAProxy Master / Backup，两层）
  ↓
  K8S 集群
    · API 管理 — Swagger2
    · 注册中心 — Eureka
    · 配置中心 — Nacos
    · 分布式网关 — 服务接入层、SSO 统一认证、Zuul 服务网关
    · 服务集群 — 用户管理、日志服务、订单管理、营销、工单、产品管理 …
    · 链路监控 — APM / Skywalking
  ↓
[两中心共享/同步]
  · 消息队列
  · 数据库

[中间 DevOps 栈 — 两中心共用]
  Jenkins | Gerrit/Git | Sonar | ELK（日志中心）
  Kubernetes | Docker | Ansible | Maven
```

**OP 双中心 vs 狭义架构（易记）**

- 狭义架构：单中心分层（代理 → 网关 → 业务 → k8s → 存储）
- OP 双中心：在之上增加 **GSLB 双活**、**HAProxy 主备双层**、**K8S 微服务组件**（Eureka/Nacos/Zuul/SSO）、**统一 CI/CD 与监控**

#### 中间件简单架构角色 — 数据库集群 1

**MySQL 主从复制 + Keepalived 高可用架构**

```
APP（应用）
  ↓
VIP（虚拟 IP，如 192.168.205.193）   ← 应用唯一接入点
  ↓
keepalived master ←→ keepalived slave   （心跳/选主）
  ↓                      ↓
MySQL master  ←— repl —→  MySQL slave     （主从/双主复制）
  ↑                      ↑
keepalived 监控本机 MySQL 服务是否正常
```

**三条要点（课件）**

1. **MySQL 主从或双主复制**：数据库节点间通过 **repl** 同步数据。
2. **Keepalived 提供 VIP**：应用不直连各节点 IP，统一访问 **VIP** 进入 MySQL 集群。
3. **故障切换**：节点异常时，**Keepalived 负责将 VIP 漂移到其他正常节点**，实现高可用。

**易考点**

| 组件 | 作用 |
|------|------|
| **VIP** | 对应用暴露的单一入口，屏蔽后端节点变化 |
| **Keepalived** | 健康检查本机 MySQL + VIP 漂移/ failover |
| **MySQL repl** | 主从（或双主）数据复制，保证数据一致性 |

#### 中间件简单架构角色 — 数据库集群 2

**HAProxy + 三节点 MySQL（RDB）集群架构**

```
应用服务器
  ↓ 应用请求
VIP（虚拟 IP）   ← 两 HAProxy + Keepalived 提供应用入口
  ↓ 数据请求
主 HAProxy ←— 业务网 —→ 备 HAProxy
  ↓ 数据库访问              ↓
  └—→ RDB 节点1 ←— 数据同步（业务网）—→ RDB 节点2
              ↖—————————————↗
                   RDB 节点3
```

**三条要点（课件）**

1. **两 HAProxy + Keepalived** 提供应用入口（VIP 接入）。
2. **HAProxy 自身高可用**：主 HAProxy 宕机后，**备 HAProxy** 接管数据请求。
3. **三节点 RDB 可同时读写**：三个数据库节点经业务网**数据同步**，支持多活/多主写入。

**易考点**

| 组件 | 作用 |
|------|------|
| **VIP + Keepalived** | 应用统一入口；HAProxy 层 failover |
| **主/备 HAProxy** | 负载均衡 + 数据库访问路由；自身冗余 |
| **三节点 RDB** | 多节点同步，均可读写（区别于集群 1 的主从只读从库场景） |

**集群 1 vs 集群 2（对比速记）**

| | 集群 1 | 集群 2 |
|---|--------|--------|
| 入口高可用 | Keepalived + VIP | Keepalived + VIP + **HAProxy 主备** |
| 数据库 | **2 节点**主从/双主 repl | **3 节点** RDB，业务网数据同步 |
| 读写 | 主从为主 | **三节点均可同时读写** |

### 1.3 中间件种类

#### 常见中间件（课件四类）

> 与 **1.1 广义四类** 视角不同：此处将**分布式**与**消息**分开列举；Web 类补充 Keepalived、LVS。

| 类别 | 典型产品 |
|------|----------|
| **Web 中间件** | Nginx、Tomcat、HAProxy、Keepalived、LVS |
| **缓存中间件** | Redis、Memcached |
| **分布式中间件** | Zookeeper |
| **消息中间件** | Kafka、RabbitMQ、ActiveMQ |

**速记口诀**

- Web 五件套：Nginx / Tomcat / HAProxy / Keepalived / LVS
- 缓存双雄：Redis / Memcached
- 分布式：Zookeeper（协调、选主、配置）
- 消息三剑客：Kafka / RabbitMQ / ActiveMQ

**与 1.1 广义分类对照**

| 1.1 广义四类 | 1.3 常见中间件 |
|--------------|----------------|
| Web | Web（+ Keepalived、LVS） |
| 数据库和缓存 | 缓存（Redis、Memcached）；数据库另列 |
| 消息和分布式 | **拆分为** 分布式（ZK）+ 消息（Kafka 等） |
| 容器 | 本页未列（Docker、k8s 见架构章节） |

## 二、Web 中间件架构与运维

> 本章目录：**Nginx** ★、Tomcat、**HAProxy** ★、**Keepalived** ★（★ = 课件标注重点）

### 2.1 Nginx ★

#### Nginx 介绍

**定位**

- **轻量级 Web 服务器** + **反向代理服务器**
- **内存占用低**、**启动极快**
- **高并发能力强**，广泛用于互联网项目
- 工作在网络 **第七层（应用层）**，可对 HTTP 应用做**流量拆分策略**

**四个主要功能模块**

| # | 模块 | 说明 |
|---|------|------|
| 1 | **HTTP 服务器** | 可独立提供 HTTP 服务，作为**静态网页服务器** |
| 2 | **虚拟主机** | 在一台物理机上**虚拟多个网站**（如一台机器挂多个个人站点） |
| 3 | **反向代理** | 单台扛不住高流量时用**服务器集群**；Nginx 作**统一入口**分发请求 |
| 4 | **负载均衡** | 将负载**均匀分配**到多台服务器，避免单台过载、其他空闲 |

**易考点**

- Nginx = Web 服务器 + 反向代理 + 负载均衡 + 虚拟主机
- **七层**（HTTP）流量调度 vs LVS/HAProxy 也可做四层/七层（后续对比 HAProxy 时再记）

#### Nginx 作用

**三大作用（课件）**

1. **静态资源** — 提供 HTML、CSS、图片等静态文件服务
2. **反向代理** — 代客户端访问后端服务
3. **统一访问入口** — 所有流量经 Nginx 单一入口进入

**功能架构（左图）**

Nginx 内部三大模块：
- **静态资源**
- **API 服务** → 对接 **数据库/缓存服务** 与 **应用服务**
- **反向代理** → **缓存加速** + **负载均衡** → **应用服务**

后端关系：
- **应用服务** ↔ **数据库/缓存服务**

> 为后端数据库和缓存服务、应用服务提供对应功能。

**应用场景示例（右图 — Tomcat 集群）**

```
客户（Client）
  ↓
nginx（统一入口 + 负载均衡）
  ↓        ↓        ↓
Tomcat-1  Tomcat-2  Tomcat-3
  └────────┴────────┘
       ↓           ↓
   数据库      Redis（共享 Session 数据）
```

- Nginx 为后端 **Tomcat** 和**数据库**提供**统一访问入口**和**负载均衡**
- 多 Tomcat 节点通过 **Redis 共享 Session**，实现会话一致性

**易考点**

| 场景 | Nginx 角色 |
|------|------------|
| 静态页 | 直接由 Nginx 返回静态资源 |
| 动态 API | 反向代理 → 应用服务（Tomcat 等） |
| 高并发 | 负载均衡 + 缓存加速 |
| 多 Tomcat | 统一入口；Session 靠 Redis 共享 |

#### Nginx 编译安装（1.18.0 源码）

**安装前准备**

```bash
# 安装依赖
yum install gcc pcre-devel openssl-devel zlib-devel -y

# 创建 nginx 系统用户（禁止登录、不建家目录）
useradd -s /sbin/nologin nginx -M
```

**编译安装**

```bash
wget http://nginx.org/download/nginx-1.18.0.tar.gz
tar xf nginx-1.18.0.tar.gz
cd nginx-1.18.0

./configure --prefix=/usr/local/nginx \
  --user=nginx \
  --group=nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_realip_module \
  --with-http_stub_status_module \
  --with-http_gzip_static_module \
  --with-pcre \
  --with-stream \
  --with-stream_ssl_module \
  --with-stream_realip_module

make && make install
chown -R nginx:nginx /usr/local/nginx
```

**configure 常用模块速记**

| 参数 | 作用 |
|------|------|
| `--prefix` | 安装路径 `/usr/local/nginx` |
| `--user/--group` | 运行用户/组 `nginx` |
| `--with-http_ssl_module` | HTTPS |
| `--with-http_v2_module` | HTTP/2 |
| `--with-http_realip_module` | 获取客户端真实 IP |
| `--with-http_stub_status_module` | 状态监控页 |
| `--with-http_gzip_static_module` | 静态 gzip |
| `--with-stream` | 四层 TCP/UDP 代理 |
| `--with-stream_ssl_module` | Stream SSL |
| `--with-stream_realip_module` | Stream 真实 IP |

**易考点**

- 依赖：**gcc、pcre-devel、openssl-devel、zlib-devel**
- 运行用户：`useradd -s /sbin/nologin nginx -M`
- 安装后改属主：`chown -R nginx:nginx /usr/local/nginx`

#### Nginx 在线安装（yum）

**1. 安装前准备**

```bash
yum install epel-release -y
yum install gcc pcre-devel openssl-devel zlib-devel -y
useradd -s /sbin/nologin nginx -M
```

**2. 在线安装 Nginx 包**

```bash
yum install nginx
```

**3. 简单配置（测试页）**

```bash
echo "web1" > /usr/share/nginx/html/index.htm
```

**4. 启动服务**

```bash
systemctl start nginx
systemctl status nginx
```

**5. 访问验证**

- 浏览器访问：`http://192.168.205.164/index.htm`
- 页面显示 **web1** 即成功

**编译 vs 在线安装（对比）**

| | 编译安装 | 在线安装 |
|---|----------|----------|
| 方式 | 源码 wget + configure + make | `yum install nginx`（需 EPEL） |
| 路径 | `/usr/local/nginx` | 包默认路径（如 `/usr/share/nginx/html`） |
| 模块 | 自定义 configure 参数 | 发行版预编译模块 |
| 共同准备 | gcc/pcre/openssl/zlib 依赖 + nginx 用户 | 同上 + **epel-release** |

#### Nginx 基础运维

**查看服务和进程**

```bash
systemctl start nginx
systemctl status nginx
```

**status 输出要点（node164 示例）**

- 服务名：`nginx.service - The nginx HTTP and reverse proxy server`
- 单元文件：`/usr/lib/systemd/system/nginx.service`
- 状态：**active (running)**
- 进程结构：
  - `nginx: master process /usr/sbin/nginx`（主进程，如 PID 1892）
  - `nginx: worker process`（工作进程，如 PID 1893、1894）
- 启动前日志：`/etc/nginx/nginx.conf` **syntax is ok** / **test is successful**

**服务启停验证**

- 访问测试：`http://192.168.205.164/index.htm`

**查看网络连接**

```bash
netstat -anlp | grep nginx
```

| 状态 | 含义 |
|------|------|
| **LISTEN** `0.0.0.0:80` / `:::80` | master 进程监听 80 端口（IPv4/IPv6） |
| **ESTABLISHED** | worker 进程处理客户端连接 |

**现网标准化部署（课件）**

| 项 | 路径/说明 |
|----|-----------|
| 部署目录 | `/apps/svr/bcop_static/` |
| 日志路径 | `/apps/logs/nginx_80` |
| 管理脚本 | `/apps/sh/nginx_80.sh` |
| 脚本参数 | `start \| stop \| restart \| status` |

**易考点**

- systemd 管理：`systemctl start/status nginx`
- 进程模型：**1 master + 多 worker**；监听由 master，连接由 worker 处理
- 排查端口：`netstat -anlp | grep nginx`
- 生产环境常用**自定义脚本**而非仅 systemctl（路径标准化）

#### Nginx 运维操作（CLI 命令）

| 命令 | 说明 |
|------|------|
| `nginx` | 启动 nginx 服务 |
| `nginx -t` | 检查配置文件语法 |
| `nginx -T` | 检查配置并**打印完整配置**到标准输出 |
| `nginx -v` | 查看 nginx **版本** |
| `nginx -V` | 查看**详细版本 + 编译参数** |
| `nginx -s reload` | **重载配置**（不中断服务） |
| `nginx -s quit` | **优雅停止**（不影响已接受请求） |
| `nginx -s stop` | **立即停止**（中止请求） |
| `nginx -h` | 显示帮助 |

**终端示例（node164）**

```bash
which nginx          # /usr/sbin/nginx
nginx -v             # nginx version: nginx/1.20.1
nginx -t             # syntax is ok / test is successful
                     # 配置文件：/etc/nginx/nginx.conf
```

**易考点**

| 对比 | quit | stop | reload |
|------|------|------|--------|
| 行为 | 优雅退出 | 直接退出 | 热加载配置 |
| 进行中请求 | 处理完毕再停 | 中止 | 不中断服务 |
| 改配置后 | — | — | 先 `nginx -t` 再 `nginx -s reload` |

### 2.2 Tomcat
> 待补充

### 2.3 HAProxy ★
> 待补充

### 2.4 Keepalived ★
> 待补充

## 三、缓存中间件架构与运维

> 涵盖：Redis、Memcached。

### 3.1 Redis
> 待补充

### 3.2 Memcached
> 待补充

## 四、分布式中间件架构与运维

> 涵盖：Zookeeper、Kafka。

### 4.1 Zookeeper
> 待补充

### 4.2 Kafka
> 待补充

## 复习互动方式

1. **默写**：某一整块（如消息队列模型 / 缓存读写策略）
2. **对比**：常见中间件产品差异（如 Kafka vs RabbitMQ、Redis vs Memcached）
3. **抽问**：随机出简答/判断
4. **纠错**：用户口述，指出遗漏与易错点

回答时：先结论再展开；中文作答；紧扣本 skill，不擅自扩展成无关百科。考试时用户要求「只给答案」则只列要点。

---

**进度说明：** 第一章 ✅；2.1 Nginx（介绍 + 作用 + 安装 + 运维）✅；2.2–2.4 及第三、四章待截图补充。
