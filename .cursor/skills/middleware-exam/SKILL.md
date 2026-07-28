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

#### Nginx 基础架构

**进程模型：master/worker 多进程**

```
MASTER PROCESS（主进程）
  ↓ 管理/生成
Child Processes（子进程）
  ├── CM — Cache Manager（缓存管理）
  ├── CL — Cache Loader（缓存加载）
  └── W  — Worker × N（工作进程，处理 HTTP 等网络流量）
```

**共享内存（Shared Memory）**

用于：**cache**、**session persistence**、**rate limits**、**session log**

**主进程（Master Process）**

- 主要功能：**与外界通信** + **管理内部其他进程**

**工作进程（Worker Process）**

- 由**主进程生成**
- 数量可在 **Nginx 配置文件**中指定（如 `worker_processes`）
- 正常情况下，**伴随主进程整个生命周期**存在
- **Worker 负责处理 HTTP 及其他网络流量**

**易考点**

| 进程 | 职责 |
|------|------|
| Master | 监听端口、管理 worker、读配置、不处理业务请求 |
| Worker | 实际处理客户端连接与请求 |
| CM / CL | 缓存管理与加载（可选子进程） |
| 共享内存 | 缓存、会话持久化、限流、会话日志 |

#### Nginx 练习实操

**题目**

> 在 CentOS 7 环境**在线安装** Nginx，访问首页内容为 **Hello Nginx**。

**完整步骤（参考答案）**

```bash
# 1. 安装前准备
yum install epel-release -y
yum install gcc pcre-devel openssl-devel zlib-devel -y
useradd -s /sbin/nologin nginx -M

# 2. 在线安装
yum install nginx

# 3. 启动服务
systemctl start nginx
systemctl status nginx    # 确认 active (running)

# 4. 配置首页并访问
echo "Hello Nginx" > /usr/share/nginx/html/index.htm
# 浏览器访问：http://192.168.205.164/index.htm
# 页面显示 Hello Nginx 即完成
```

**考点串联**

| 步骤 | 对应知识点 |
|------|------------|
| epel + 依赖 + 用户 | 在线安装前准备 |
| yum install nginx | 在线安装 vs 编译安装 |
| systemctl start | 基础运维 / systemd |
| echo > index.htm | 静态页默认目录 `/usr/share/nginx/html/` |

### 2.2 Tomcat

#### Tomcat 简介

**定义**

- **免费、开源**的 **Web 应用服务器**
- **轻量级应用服务器**
- 适用于**中小型系统**、**并发访问用户不多**的场景
- **开发和调试 JSP 程序的首选**

**三个要点**

1. Apache Tomcat 是一款 **应用（Java）服务器**
2. 是一个 **Servlet 容器**（JSP 也会被编译/翻译成 Servlet）
3. 能够**动态生成资源**并返回给客户端

**三大特点（课件金字塔）**

| 层次 | 特点 |
|------|------|
| 上 | **技术先进、性能稳定、开源免费** |
| 中 | 符合 **Java EE** 的 **JSP、Servlet 标准** 的 JSP 服务器 |
| 下 | **动态解析容器**，处理**动态请求**能力强 |

**易考点**

| 对比 | Nginx | Tomcat |
|------|-------|--------|
| 类型 | Web 服务器 / 反向代理 | **Java 应用服务器** |
| 静态/动态 | 擅长静态 + 反向代理 | 擅长 **JSP/Servlet 动态页面** |
| 关系 | 常作 Tomcat 前端的统一入口与负载均衡 | 后端应用容器 |

#### Tomcat 的架构基础

**核心组件**

- **连接器（Connector）**
- **容器（Container）**

**架构层次（课件）**

```
Tomcat
  └── Service（可多个）
        ├── Connector（连接器，可多个）
        └── Container（容器）
              请求 → Connector → ServletRequest → Container
              响应 ← Connector ← ServletResponse ← Container
```

**两大核心功能**

| 组件 | 方向 | 功能 |
|------|------|------|
| **Connector** | **对外** | 处理 **Socket 连接**；负责**网络字节流**与 **Request/Response 对象**的转化 |
| **Container** | **对内** | **加载并管理 Servlet**；处理具体的 **Request 请求** |

**请求处理流程**

1. 客户端 **请求** → **Connector** 接收
2. Connector 将字节流转为 **ServletRequest**，交给 **Container**
3. Container 调用 Servlet 处理，生成 **ServletResponse**
4. Connector 将 Response 转为字节流，返回 **响应** 给客户端

**易考点**

- Connector = **网络 I/O + 协议转换**（对外）
- Container = **Servlet 生命周期 + 业务请求处理**（对内）
- 一个 Service 内：**多个 Connector** 可共用一个 **Container**

#### Tomcat — 处理动态请求（Nginx + Tomcat 分工）

**核心思路**

- Nginx 处理**静态页面**效率**远高于** Tomcat
- **HTML、CSS、JS、图片**等静态资源 → **Nginx 直接处理**
- **动态请求**（如 JSP）→ **交给 Tomcat** 处理，提升系统吞吐量

**请求分流架构**

```
客户端
  ↓
Nginx 服务
  ├── 静态请求（html/css/js/图片）→ Nginx 直接返回
  └── 动态请求 → Tomcat 服务 → JSP
```

**Nginx 配置示例（课件）**

```nginx
# 静态资源 — nginx 处理
location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css)$ {
    root /webapps/myproject/code/static-resource;
    expires 3d;
}

# 其余动态请求 — 转发 tomcat
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

**配置要点**

| 指令 | 作用 |
|------|------|
| `location ~ .*\.(html\|htm\|…)$` | 正则匹配静态文件扩展名 |
| `root` | 静态文件根目录 |
| `expires 3d` | 浏览器缓存 3 天 |
| `proxy_pass http://127.0.0.1:8080` | 动态请求反向代理到 Tomcat（默认 8080） |

**易考点**

- **静动分离**：Nginx 静态 + Tomcat 动态，是经典生产架构
- Tomcat 默认端口 **8080**
- 与 **2.1 Nginx 作用**（Tomcat 集群 + Redis Session）可组合记忆

#### Tomcat 安装 — Linux 系统（8.0.26）

**1. 下载 Tomcat 8**

- 归档地址：`https://archive.apache.org/dist/tomcat/tomcat-8/`

```bash
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.0.26/bin/apache-tomcat-8.0.26.tar.gz
```

**2. 解压缩并启动**

```bash
tar -zxvf apache-tomcat-8.0.26.tar.gz -C /usr/local/
cp -rf apache-tomcat-8.0.26/conf/* apache-tomcat-8.5.39/conf/   # 课件原样（版本路径以实际为准）
cd /usr/local/apache-tomcat-8.0.26
./bin/startup.sh
```

**3. 查看 Tomcat 进程**

```bash
ps -ef | grep tomcat
```

**4. 访问 Tomcat 主页**

- 浏览器：`http://192.168.205.164:8080/`
- 成功标志：页面显示 **Apache Tomcat/8.0.26**，提示安装成功

**常用路径与命令**

| 项 | 说明 |
|----|------|
| 安装目录 | `/usr/local/apache-tomcat-8.0.26` |
| 启动脚本 | `./bin/startup.sh` |
| 停止脚本 | `./bin/shutdown.sh`（课件未列，运维常用） |
| 默认端口 | **8080** |
| 进程检查 | `ps -ef \| grep tomcat` |

**易考点**

- Tomcat **免编译**：下载 tar.gz → 解压 → `startup.sh` 即可
- 与 Nginx 不同：Tomcat 是 **Java 应用服务器**，需 JDK 环境（课件本页未强调，实操时注意）
- 验证：8080 端口 + 默认欢迎页

#### Tomcat 升级 — Linux 系统（8.0.26 → 8.5.39）

**背景**

- 日常运维中常因**安全漏洞**需升级 Tomcat 修复
- 课件示例：**8.0.26 升级到 8.5.39**

**1. 下载并解压新版本**

```bash
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.39/bin/apache-tomcat-8.5.39.tar.gz
tar -xf apache-tomcat-8.5.39.tar.gz
```

**2. 配置新 Tomcat（迁移旧版配置与应用）**

```bash
cp -rf apache-tomcat-8.0.26/conf/* apache-tomcat-8.5.39/conf/
cp -rf apache-tomcat-8.0.26/bin/* apache-tomcat-8.5.39/bin/
rm -rf apache-tomcat-8.5.39/webapps/*
cp -rf apache-tomcat-8.0.26/webapps/* apache-tomcat-8.5.39/webapps/
```

> 课件原图 `8.0.26 /bin/*` 应为 `8.0.26/bin/*`（路径空格为笔误）。

**3. 停止旧 Tomcat**

```bash
./apache-tomcat-8.0.26/bin/shutdown.sh
```

**4. 启动新 Tomcat**

```bash
./apache-tomcat-8.5.39/bin/startup.sh
```

**升级流程速记**

```
下载解压新版 → 拷贝 conf/bin/webapps → 停旧启新
```

| 迁移项 | 说明 |
|--------|------|
| `conf/*` | 保留原有配置（端口、用户等） |
| `bin/*` | 保留/custom 启动脚本等 |
| `webapps/*` | 先清空新版 webapps，再拷贝旧应用 |

**易考点**

- 升级核心：**配置与应用迁移** + **shutdown 旧版** + **startup 新版**
- 与安装页 `cp conf` 命令呼应：安装示例中的 cp 实为**升级场景**配置迁移
- 升级前建议备份；升级后验证 8080 与业务应用

#### 现网 Tomcat 运维操作（18080 + Manager Text）

> 生产环境示例：端口 **18080**；管理脚本标准化；通过 **Tomcat Manager Text** 接口管理应用。

**1. 启动、关闭、重启、查看进程**

```bash
/apps/sh/tomcat_18080.sh start|stop|status|restart
```

**2. 查看 Tomcat 下工程运行情况**

```bash
curl -u admin:admin密码 http://localhost:18080/manager/text/list
```

**3. 指定停止某工程包**

```bash
curl -u admin:admin密码 http://localhost:18080/manager/text/stop?path=/op-order-manage
```

**4. 指定重载（reload）某工程包**

```bash
curl -u admin:admin密码 http://localhost:18080/manager/text/reload?path=/op-order-manage
```

**5. 指定启动某工程包**

```bash
curl -u admin:admin密码 http://localhost:18080/manager/text/start?path=/op-order-manage
```

**6. 指定解部署（undeploy）某工程包**

```bash
curl -u admin:admin密码 http://localhost:18080/manager/text/undeploy?path=/op-order-manage
```

**7. 从目录部署 WAR 并指定工程名**

```bash
curl -u admin:admin密码 "http://localhost:18080/manager/text/deploy?path=/op-order-manage&war=file:/tmp/bcon20180807/op-order-manage-v2.war"
```

> 课件第 7 条 URL 末尾被截断；`war=file:` 路径以课件底部 `op-order-manage-v2.war` 为准。

**Manager Text 接口速记**

| 操作 | URL 路径 |
|------|----------|
| list | `/manager/text/list` |
| stop | `/manager/text/stop?path=/上下文` |
| reload | `/manager/text/reload?path=/上下文` |
| start | `/manager/text/start?path=/上下文` |
| undeploy | `/manager/text/undeploy?path=/上下文` |
| deploy | `/manager/text/deploy?path=/上下文&war=file:/路径/xxx.war` |

**易考点**

- 现网用**自定义脚本**（`/apps/sh/tomcat_18080.sh`）而非直接 startup.sh
- 应用管理走 **curl + Manager Text**，需 `-u admin:密码` 认证
- `path=` 为应用**上下文路径**（如 `/op-order-manage`）
- reload ≠ redeploy：reload 热重载；undeploy/deploy 为卸载/部署

#### Nginx 和 Tomcat 的区别

**1. 核心职能**

| | Nginx | Tomcat |
|---|-------|--------|
| 定位 | **静态内容服务** + **代理服务器** | **应用容器** |
| 行为 | 接收外部请求，**转发**给 Tomcat、Django 等后端 | 在容器内**运行 Java Web 应用** |

**2. 技术分类**

| 类型 | 代表 | 职责 |
|------|------|------|
| **HTTP Server** | Apache、Nginx | 严格意义上的 HTTP 服务器；向客户端提供服务器上存储的资源（HTML、图片等），经 HTTP **原样传输** |
| **Application Server** | Tomcat | **应用服务器**；**Servlet/JSP** 应用容器 |

**3. 注意（课件强调）**

> **Nginx 只做请求分发（代理/负载均衡），不做应用逻辑的实际处理。**

**对比速记（考试用）**

```
Nginx  = 静态 + 反向代理 + 分发请求
Tomcat = Servlet/JSP 容器 + 运行动态 Java 应用
组合   = Nginx 前置入口，Tomcat 后端处理动态逻辑（见 2.2 静动分离）
```

### 2.3 HAProxy ★

#### HAProxy — MySQL 多 AZ 高可用架构（课件示例）

**架构概览**

```
Region 1                          Region 2（中立协调）
┌──────── AZ1 ────────┐            ┌─────────────┐
│ Node: MySQL + Manager│←──────────→│   Manager   │
└──────── AZ2 ────────┘            │ （仅协调）   │
┌──────── AZ2 ────────┐            └─────────────┘
│ Node: MySQL + Manager│←──────────→      ↑
└─────────────────────┘                   │
         ↑           ↑                      │
    HAProxy(IP1)  HAProxy(IP2)  ← 用户轮询访问
```

**三条要点（课件）**

1. **两个 MySQL** 分别位于某资源池的 **AZ1** 和 **AZ2**。
2. 每个 MySQL 节点部署一个 **Manager** 进程；**Region2** 中立节点也部署一个 Manager，**仅作协调进程**（防脑裂/仲裁）。
3. 用户通过**轮询**两个 HAProxy 节点上 HAProxy 暴露的**服务端口**访问 MySQL。

**组件职责**

| 组件 | 职责 |
|------|------|
| **MySQL（AZ1/AZ2）** | 数据节点，跨 AZ 部署 |
| **Manager** | 管理/监控 MySQL；Region1 两节点 + Region2 协调节点组成管理层 |
| **HAProxy（IP1/IP2）** | **数据库访问入口**；双节点暴露端口，客户端轮询接入 |
| **Region2 Manager** | **不参与数据**，仅协调/仲裁 |

**与 1.2 数据库集群 2 关联**

- 1.2 **数据库集群 2**：HAProxy + 三节点 RDB，主备 HAProxy 高可用
- 本页：**多 AZ + Manager 协调 + 双 HAProxy 入口**，面向生产跨 AZ MySQL 场景

**易考点**

- HAProxy 在此架构中是 **MySQL 的统一访问入口/负载均衡**
- **双 HAProxy + 轮询** = 入口层高可用
- **三 Manager**（2 数据节点 + 1 中立协调）= 集群协调/选主

#### 安装和配置 HAProxy

**yum 在线安装**

```bash
yum install epel-release -y
yum install haproxy -y
systemctl start haproxy && systemctl status haproxy
```

**默认配置文件**

- `/etc/haproxy/haproxy.cfg`

**systemctl status 输出要点（node164 示例）**

- 服务名：`haproxy.service - HAProxy Load Balancer`
- 状态：**active (running)**
- 主进程：`haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid`
- worker：`/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg ...`

**现网 HAProxy 部署标准（课件）**

| 项 | 路径/命令 |
|----|-----------|
| 安装部署路径 | `/apps/svr/haproxy` |
| 启停管理 | `sudo /apps/sh/haproxy.sh start\|stop\|restart\|status` |
| 日志目录 | `/apps/logs/haproxy` |

**易考点**

| 对比 | yum 安装 | 现网标准 |
|------|----------|----------|
| 配置 | `/etc/haproxy/haproxy.cfg` | 自定义部署路径 |
| 管理 | `systemctl` | `/apps/sh/haproxy.sh` |
| 依赖 | 需 **epel-release** | 同左 + 路径/日志标准化 |

### 2.4 Keepalived ★

#### Keepalived 介绍

**定义**

- Keepalived 是**集群管理**中保证**集群高可用**的服务软件
- 用于**防止单点故障（SPOF）**

**实现基础**

- 以 **VRRP 协议**为实现基础
- **VRRP**（Virtual Router Redundancy Protocol）：用于解决**静态路由的高可用**

**易考点**

| 概念 | 说明 |
|------|------|
| **Keepalived** | 高可用软件；健康检查 + **VIP 漂移** |
| **VRRP** | 虚拟路由冗余协议；多节点选主/备，故障切换 |
| **典型场景** | 1.2 **MySQL + Keepalived + VIP**；Nginx/HAProxy 主备 |

**与课程其他组件关联**

- **1.2 数据库集群 1**：Keepalived 监控本机 MySQL，异常时 **VIP 迁移**
- **HAProxy/Nginx**：常与 Keepalived 组合实现入口层高可用（主备 + VIP）

#### Keepalived 组件作用举例 — MySQL 主从 + Keepalived 高可用

> 与 **1.2 数据库集群 1** 同一架构；Keepalived 的作用：**监控本机 MySQL + 管理 VIP 漂移**。

**Keepalived 在架构中的角色**

| 层级 | 组件 | 作用 |
|------|------|------|
| 应用 | APP | 只连接 **VIP**，不感知后端切换 |
| 高可用 | Keepalived master/slave | **监控本机 MySQL**；维护 **VIP 192.168.205.193**；故障时漂移 |
| 数据 | MySQL 120 ↔ 121 | **repl** 主从复制；可单向/双向，切换后互为主从 |

**架构图（实验 IP）**

```
APP → VIP 192.168.205.193
        ↓
keepalived master ←→ keepalived slave
        ↓                    ↓
MySQL（120）  ←—— repl ——→  MySQL（121）
```

**/etc/hosts**

```
192.168.205.120  node120.centos.com  node120
192.168.205.121  node121.centos.com  node121
```

**复制状态验证（演示环境 node121）**

```sql
-- 主库 node120：SHOW PROCESSLIST\G
-- User: repl | Host: 192.168.205.121:39814
-- Command: Binlog Dump GTID
-- State: Master has sent all binlog to slave; waiting for more updates

-- 从库 node121：SHOW SLAVE STATUS\G
-- Slave_IO_State: Waiting for master to send event
-- Master_Host: 192.168.205.120
-- Master_User: repl | Master_Port: 3306
-- Master_Log_File: mysql-bin.000004
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
```

**健康判断速记**

| 检查项 | 正常 |
|--------|------|
| `Slave_IO_Running` | **Yes** |
| `Slave_SQL_Running` | **Yes** |
| 主库 State | all binlog sent, waiting for updates |
| 复制协议 | **GTID** |
| VIP | **192.168.205.193**（应用唯一入口） |

#### Keepalived 主要概念（VRRP 术语）

| 概念 | 解释 |
|------|------|
| **VRRP 路由器** | 运行 **VRRP 协议**的设备 |
| **虚拟路由器** | VRRP 管理的抽象设备，又称 **VRRP 备份组**；作为共享 LAN 内主机**缺省网关**；含 **VRID** + 一组虚拟 IP |
| **虚拟 IP 地址（VIP）** | 虚拟路由器的 IP；一个虚拟路由器可有**一个或多个** VIP |
| **IP 地址拥有者** | 若某 VRRP 路由器将 VIP 作为**真实接口地址**，则为拥有者；正常时响应发往 VIP 的 ping、TCP 等 |
| **虚拟 MAC 地址** | 按 VRID 生成；格式 **`00-00-5E-00-01-{VRID}`**；应答 ARP 时用虚拟 MAC，非接口真实 MAC |
| **主 IP 地址** | 从接口真实 IP 中选出（通常**第一个**）；VRRP 广播报文以此作**源地址** |
| **Master 路由器** | **转发报文**、**应答 ARP** 的 VRRP 路由器；故障时 Backup **竞选**新 Master |
| **Backup 路由器** | Master 故障时，通过**竞选**成为新 Master |

**易考点速记**

- **VIP** = 对外服务 IP；**VRID** = 虚拟路由器 ID
- **虚拟 MAC**：`00-00-5E-00-01-{VRID}`
- **Master** 干活（转发+ARP）；**Backup** 待命竞选
- Keepalived 实现 VRRP → 健康检查 + VIP 在主备间切换

#### VRRP 机制（状态转换）

**VRRP 路由器三种状态**

1. **Initialize**（初始化）
2. **Master**
3. **Backup**

> 不同状态之间**可相互转换**。

**状态转换图（课件）**

```
                    Initialize
                   /          \
    startup且优先级=255    startup且优先级<255
                 /              \
            Master ←——————————→ Backup
                   \          /
        更高优先级VRRP报文    超时未收到Master hello
        或 shutdown          或 shutdown
```

| 当前状态 | 触发条件 | 下一状态 |
|----------|----------|----------|
| **Initialize** | 收到 startup，且**优先级 = 255** | **Master** |
| **Initialize** | 收到 startup，且**优先级 < 255** | **Backup** |
| **Master** | 收到 **shutdown** 消息 | **Initialize** |
| **Master** | 收到**更高优先级** VRRP 报文 | **Backup** |
| **Backup** | 收到 **shutdown** 消息 | **Initialize** |
| **Backup** | 规定超时内**未收到 Master 的 hello 报文** | **Master** |

**易考点**

- 优先级 **255** → 启动直接 **Master**；**< 255** → **Backup**
- Backup 升 Master：**Master hello 超时**（Master 宕机/网络故障）
- Master 降 Backup：收到**更高优先级** VRRP 报文
- shutdown → 回到 **Initialize**

#### VRRP 各状态说明

**1. Initialize（初始化）**

- VRRP **不可用**状态
- **不处理**任何 VRRP 报文
- 典型进入场景：**刚启动**；或**检测到故障**后

**2. Master**

| 行为 | 说明 |
|------|------|
| **发送 VRRP 通告报文** | 按 **Advertisement_Interval** 周期发送 |
| **应答 ARP** | 对 VIP 的 ARP 请求，用**自身 MAC** 应答 |
| **转发 IP 报文** | 目的 MAC 为**虚拟 MAC** 的报文 |
| **接收 IP 报文** | 目的 IP 为 **VIP** 的报文才接收；否则**丢弃** |

**3. Backup**

- **接收 Master 发送的 VRRP 通告报文**
- 据此判断 **Master 状态是否正常**
- 若超时未收到 hello → 竞选升为 **Master**（见上节状态转换）

**Master vs Backup 对比（易考点）**

| | Master | Backup |
|---|--------|--------|
| 发 VRRP 通告 | ✅ 周期性发送 | ❌ 只接收 |
| 应答 VIP 的 ARP | ✅ | ❌ |
| 转发/接收 VIP 流量 | ✅ | ❌（待命） |
| 监控 Master | — | ✅ 收通告判断 Master 健康 |

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

**进度说明：** 第一章 ✅；第二章 Web 中间件（2.1–2.4，Keepalived/VRRP 完整）✅；第三、四章待截图补充。
