# 目标
- 为公司专用在云内构建网络 -> VPC
- 目前 EC2 使用 default VPC -> 直接创建专用 VPC 并应用
- [v] 仅构建 VPC（最先要做的工作）
    -> [v] 将 EC2 连接到自定义 VPC 以构建基础设施
    -> web（public subnet）、was/db（private subnet）扩展 + NAT gateway、SSH 管理、IAM Role
    -> 源代码、Kubernetes、CI/CD
    -> 仪表盘（Grafana、Prometheus..）

# 组成
- AWS
  - VPC
  - CIDR
  - Public/Private Subnet
  - Internet Gateway
  - NAT Gateway
  - Route Table / Route Table Association
  - ..

# VPC（Virtual Private Cloud）
## 定义
- 在 AWS（云生态）内我（个人/企业）`直接构建的私有网络`
- AWS 内我自己（个人/企业）独占使用的独立（隔离）网络空间

## 现实世界比喻
- AWS：城市/国家（生态系统）
- VPC：多个公寓小区（云视角：个人/公司独占使用）
- subnet：栋（101栋、102栋..）
- EC2：各个房子（住户，101号、102号...）
- Internet Gateway：公寓大门（可与外部通信，流量可进可出）
  - Subnet 属于 public 形式（开放）
- Route table：指路牌（地址、路线、方向）
- Security Group：各住户门口的安保装置（出入控制 -> 端口/IP 约束（入站、出站））
- NACL（Network Access Control List）：公寓栋/小区的出入口控制（VPC 的安全层）

--------------
- NAT Gateway：
  - 为 Private Subnet 设计
    - 外部无法访问 Private Subnet
    - 若需要安装软件、获取源码等，内部需要从外部拉取资源（pull）
    - 这时需要 NAT Gateway

## 组成成员
### VPC
- Default VPC
  - 每个区域通常自动创建
  - 在创建 EC2 时可直接使用（公网默认设置）
  - 适合学习、测试
  - 不是专属 VPC
- [v] Custom VPC
  - 子项由自己配置与设计
  - 可用于生产环境
  - 是专属 VPC

### CIDR
- 在创建 VPC 时指定 IP 地址范围

```
# 设置
10.0.0.0/16
---
# 解释：前 16 位（2 字节）固定 / 后 2 字节可自定义
# 0 ~ 255 可用 => 256*256 组合 => 可用 IP 总数 65,536（实际保留若干地址）
10.0.0.10    -> 可用
10.0.1.10    -> 可用
10.0.255.254 -> 可用

10.1.0.10    -> 不可用
```

### 子网
- 将 VPC 划分为更小的网络

```
VPC
10.0.0.0/16
L Public   : 10.0.1.0/24
L Private  : 10.0.2.0/24

---
Public 网段
  - 前 3 个字节固定 => IP 范围理论上是 10.0.1.0 ~ 10.0.1.255：256 个
    - 2^(32-24) => 数量
  - AWS 保留 IP（5 个）-> 不能使用
    - 10.0.1.0 : 网络地址
    - 10.0.1.1 : AWS VPC 路由器地址
    - 10.0.1.2 : AWS DNS 服务器地址
    - 10.0.1.3 : AWS 未来预留地址
    - 10.0.1.255 : 广播地址
  - 最终：可用数量 256 - 5 = `251 个可用`
```

- 类型
  - 公共子网（Public Subnet）
    - 可直接与外部互联网通信的区域
    - 存在指向 Internet Gateway 的路由
    - 拥有可从外部访问的公网 IP（例如用于 web <-> was <-> db 的通信）
    - 常部署：Web 服务器、ALB（负载均衡器）、Bastion Host（用于访问私有子网）等

```
Internet <-> IGW <-> Route Table <-> Public Subnet <-> EC2
```

  - 私有子网（Private Subnet）
    - 与外部互联网隔离的安全区域，仅限内部通信
    - 外部无法直接访问（全部阻断）
    - 可通过 NAT Gateway 发起对外的通信（Pull）
    - 常部署：EKS 工作节点、数据库（RDS 或自建）、后端服务器（WAS）等

```
Internet                       X  Route Table <-> Private Subnet <-> EC2
Internet  <-(pull) NAT Gateway <-> Route Table <-> Private Subnet <-> EC2
```

### 网关
- 连接互联网的“门”

```
# 大体结构
互联网 <-> IGW <-> VPC
```
- 类型
  - IGW（Internet Gateway）
    - 将 VPC 与互联网连接，双向通道
    - 位于公共子网的资源（如 EC2）可以与互联网相互通信
  - NAT Gateway
    - 允许位于私有子网的资源从外部下载数据（库更新、Docker 镜像等）
    - 外部无法发起对私有网络的访问（单向）

### 路由表
- 导航／指路牌，用于决定数据包去向

```
目的地
10.0.0.0/16     local
0.0.0.0/0       IGW
```
- 路由表是一组规则，指导数据包如何找到目标 IP
- 示例：
  - 公共子网路由表：0.0.0.0/0 => 发送到 IGW
  - 私有子网路由表：0.0.0.0/0 => 发送到 NAT Gateway

### 安全防火墙
- 组成结构（简要流程）：

```
            [ Internet / External ]
                    │
                    ▼
        ┌──────────────────────────────┐
        │        Subnet Boundary       │
        │   ┌──────────────────────┐   │
        │   │ Network ACL (NACL)   │ ⬅️ 1차 검문 (Stateless, 서브넷 단위)
        │   └──────────┬───────────┘   │
        │              │               │
        │              ▼               │
        │   ┌──────────────────────┐   │
        │   │ Security Group (SG)  │ ⬅️ 2차 검문 (Stateful, 인스턴스/ENI 단위)
        │   └──────────┬───────────┘   │
        │              │               │
        │              ▼               │
        │   ┌──────────────────────┐   │
        │   │     EC2 Instance     │ ⬅️ 최종 도착지
        │   └──────────────────────┘   │
        └──────────────────────────────┘
```

- 安全组（Security Group）
  - 作用对象：EC2 实例、EKS 节点、Pod 等
  - 特性：只设置允许（allow）规则；默认拒绝所有流量，按需放行
  - 类型：入站（Inbound）、出站（Outbound）规则
- 网络 ACL（NACL）
  - 作用对象：应用于整个子网
  - 特性：可设置允许（allow）和拒绝（deny）规则（无状态）

# 网络拓扑
## 开发／运营／测试 等环境隔离策略

```
AWS
 ├── VPC-A (开发)
 │      ├── EC2
 │      ├── RDS
 │      └── ALB
 │
 ├── VPC-B (运营)
 │      ├── EC2
 │      └── RDS
 │
 └── VPC-C (测试)
```

## 常见基础拓扑（按功能划分）

```
                  Internet
                      │
              Internet Gateway
                      │
               ┌──────────────┐
               │     VPC      │
               │10.0.0.0/16   │
               └──────────────┘
                 │          │
        ┌────────┘          └────────┐
        │                            │
 Public Subnet                 Private Subnet
10.0.1.0/24                   10.0.2.0/24
        │                            │
      EC2(Web)                   EC2(DB, WAS)
        │                            │
    Public IP                    Private IP
```

## 包含网关的报文流动示意

```
+-----------------------------------------------------------------------------------+
| AWS Cloud (ap-northeast-2 : 서울 리전)                                              |
|                                                                                   |
|  [ VPC: 10.0.0.0/16 ] (가상 네트워크 )                                              |
|                                                                                   |
|   +---------------------------------------+ +----------------------------------+  |
|   | Public Subnet (10.0.1.0/24)           | | Private Subnet (10.0.101.0/24)   |  |
|   | (외부 공개 구역)                          | | (내부 보안 구역)                    |  |
|   |                                       | |                                  |  |
|   |  [ ALB / Bastion ]                    | |  [ EKS Worker Node / DB ]        |  |
|   +---------------------------------------+ +----------------------------------+  |
|                       |                                      |                    |
|                       | (직접 연결)                            | (NAT 통해 출구만 연결) |
|                       v                                      v                    |
|             [ Internet Gateway ]                    [ NAT Gateway ]               |
|                      |                                       |                    |
+----------------------+---------------------------------------+--------------------+
                       |                                       |
                       +------------------[ 인터넷  ]------------+
```