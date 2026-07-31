# 版本说明
## v1
- 基础设施工程（Infrastructure Engineering）：构建可运营的基础设施
```
Terraform -> 生产环境 AWS -> 三层架构 -> 多可用区（Multi-AZ）-> ALB（应用负载均衡）-> ASG（自动伸缩组）-> RDS

通过 SSM System Manager 连接 EC2（避免使用 pem）
```

## v2
- 平台工程（Platform Engineering）：构建容器平台 -> 基于 Kubernetes 进行整体重构
```
Terraform -> EKS -> 容器 -> Deployment -> Service -> Ingress
```

## v3
- DevOps：CI/CD
```
Github -> GitHub Actions / ArgoCD -> ECR -> EKS
```

## v4
- 可观测性（Observability）：运维与监控
```
Prometheus -> Grafana -> 指标（Metrics）-> 告警（例如 Slack）
```

# 基础设施工程：构建可运营的基础设施
- 特点
  - 使用 Terraform 构建整个生产/运营级基础设施
  - 高可用性
    - 当 Web1 出现故障或负载升高（例如 CPU 超过 50%）时，`ALB` 会将流量切换到 Web2
    - 流量分散用于故障应对
  - 可扩展性
    - 根据访问量增减，自动调整 EC2 实例数量（自动化管理）
    - `ASG (Auto Scaling Group)` 类似于 Kubernetes 的 replica
  - 不变基础设施（Immutable Infrastructure）
    - EC2 通过 `Launch Template` 进行配置：创建新 EC2、自动安装、自动启动服务
    - 当服务器配置变更时，更新 Launch Template 会触发 EC2 顺序替换，实现无停机改动（类似 Kubernetes 中替换容器）
  - 负载均衡
    - 高可用性的具体场景
    - 版本演进
      - v0（原始）：client -> web
      - v1（当前）：client -> ALB（绑定 EIP） -> web1 或 web2
      - v2（后续）：Ingress -> Service -> pod
  - 数据库分离运行
    - 数据库使用 AWS 专用 RDS 服务，创建大约需 10 分钟
  - 生产级架构
  - 架构设计便于以 Kubernetes 平滑替换与扩展
    - 各部分可逐步替换为容器化/编排方案

# 架构图
- 流量流向（API 请求、网页请求等）
```
Internet（外部网络，公网）
   │
Public ALB（EIP，域名，公共子网）
   │
WEB Auto Scaling Group（ASG，最少 2 台，定义扩展策略，最大 n 台）
├─ WEB EC2 / AZ-a（可用区 a -> 子网 a）
└─ WEB EC2 / AZ-c（可用区 c -> 子网 c）
   │
Internal ALB（私有子网，web <-> was，负责内部负载均衡）
   │
WAS Auto Scaling Group
├─ WAS EC2 / AZ-a
└─ WAS EC2 / AZ-c
   │
RDS MySQL Multi-AZ（高可用；若 DB 容量不足可启用自动扩容由 AWS RDS 处理，启用后生效）
├─ Primary
└─ Standby
```

- 网络布局（两个可用区，每区包含子网：1 个公共子网、2 个私有子网）
```
ap-northeast-2a               ap-northeast-2c
───────────────────────────────────────────────
Public-A                       Public-C
├─ Public ALB                  ├─ Public ALB
└─ NAT Gateway-A               └─ NAT Gateway-C

App-Private-A                  App-Private-C
├─ WEB ASG                     ├─ WEB ASG
├─ Internal ALB                ├─ Internal ALB
└─ WAS ASG                     └─ WAS ASG

DB-Private-A                   DB-Private-C
└─ RDS Primary/Standby         └─ RDS Primary/Standby
```
