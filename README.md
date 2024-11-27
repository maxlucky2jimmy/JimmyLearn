以下是将上述内容转换为带有可点击目录的 Markdown 格式：

# 基于银行业务的分布式数据库系统案例

## 一、项目搭建
### 1. 创建数据库表及对应的实体类
[点击查看代码及说明](#创建数据库表及对应的实体类（以账户信息表为例）)
### 2. 创建数据库上下文类
[点击查看代码及说明](#创建数据库上下文类)
### 3. 项目配置
[点击查看配置详情](#项目配置)

## 二、组件间通信
### 1. 在.NET Core 应用程序与 MySQL 之间
[点击查看通信说明](#在NET-Core应用程序与-MySQL-之间)
### 2. 在数据存储实例节点与分片元数据服务（基于 ETCD）之间
[点击查看通信代码及说明](#在数据存储实例节点与分片元数据服务（基于-ETCD）之间)

## 三、银行业务逻辑规则与范围分片
### 1. 范围分片逻辑
[点击查看分片逻辑代码及说明](#范围分片逻辑)
### 2. 分片元数据服务控制数据流向
[点击查看数据流向控制代码及说明](#分片元数据服务控制数据流向)

## 四、搭建 ETCD 集群及实现分片元数据服务的可用性和数据一致性
### 1. 搭建 ETCD 集群（在 K8S 中）
[点击查看 ETCD 集群部署 YAML 文件及说明](#搭建-ETCD-集群（在-K8S-中）)
### 2. 实现分片元数据服务的可用性和数据一致性
[点击查看可用性和数据一致性说明](#实现分片元数据服务的可用性和数据一致性)

## 五、监控服务节点健康及动态添加删除节点
### 1. 监控服务节点健康
[点击查看监控代码及说明](#监控服务节点健康)
### 2. 动态添加删除节点
[点击查看动态添加删除节点操作说明](#动态添加删除节点)

## 六、部署步骤
### 1. 部署 MySQL 数据库 in K8S
[点击查看 MySQL 部署步骤详情](#部署-MySQL-数据库-in-K8S)
### 2. 部署 ETCD 集群 in K8S
[点击查看 ETCD 集群部署步骤详情](#部署-ETCD-集群-in-K8S)
### 3. 部署.NET Core 应用程序 in K8S
[点击查看.NET Core 应用程序部署步骤详情](#部署NET-Core应用程序-in-K8S)

## 七、注意事项和思考
### 1. 数据一致性和事务处理
[点击查看相关说明及思考点](#数据一致性和事务处理)
### 2. 数据迁移和平衡
[点击查看相关说明及思考点](#数据迁移和平衡)
### 3. ETCD 性能和容量规划
[点击查看相关说明及思考点](#ETCD-性能和容量规划)
### 4. 安全考虑
[点击查看相关说明及思考点](#安全考虑)
### 4. 监控和告警
[点击查看相关说明及思考点](#监控和告警)
### 6. 缓存策略
[点击查看相关说明及思考点](#缓存策略)

以下是详细的案例内容：

### 一、项目搭建

#### 1. 创建数据库表及对应的实体类（以账户信息表为例）

**数据库表结构（MySQL）**：

```sql
-- 创建账户信息表
CREATE TABLE account_info (
    -- 账户 ID，自增长主键
    account_id BIGINT NOT NULL AUTO_INCREMENT,
    -- 客户姓名
    customer_name VARCHAR(255),
    -- 账户余额
    balance DECIMAL(10, 2),
    -- 用于范围分片的字段，假设根据账户余额范围进行分片
    balance_range VARCHAR(20),
    PRIMARY KEY (account_id)
);
```

**对应的实体类（.NET Core）**：

```csharp
// 账户信息实体类
public class AccountInfo
{
    // 账户 ID
    public long AccountId { get; set; }
    // 客户姓名
    public string CustomerName { get; set; }
    // 账户余额
    public decimal Balance { get; set; }
    // 余额范围，用于分片标识
    public string BalanceRange { get; set; }
}
```

#### 2. 创建数据库上下文类

```csharp
// 银行数据库上下文类，继承自 DbContext
public class BankContext : DbContext
{
    // 构造函数，接收 DbContextOptions 用于配置数据库连接
    public BankContext(DbContextOptions<BankContext> options) : base(options)
    {
    }

    // DbSet 属性，用于与数据库中的 account_info 表进行交互
    public DbSet<AccountInfo> AccountInfos { get; set; }

    // 模型创建时的配置，这里设置 AccountId 为自增长
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<AccountInfo>().Property(e => e.AccountId).ValueGeneratedOnAdd();
    }
}
```

#### 3. 项目配置

在.NET Core 项目的 `appsettings.json` 文件中，配置数据库连接字符串以及 ETCD 相关信息（假设 ETCD 用于存储分片元数据等）。

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "BankDB": "Server=mysql-server;Database=bank_db;User Id=root;Password=your_password;TrustServerCertificate=True;"
  },
  "Etcd": {
    "Endpoints": [
      "http://etcd1:2379",
      "http://etcd2:2379",
      "http://etcd3:2379"
    ],
    "Username": "etcd_user",
    "Password": "etcd_password"
  }
}
```

这里假设 `mysql-server` 是 MySQL 服务在 K8S 中的服务名称（后续会详细说明如何在 K8S 中部署 MySQL），并且配置了 ETCD 集群的端点、用户名和密码信息。

### 二、组件间通信

#### 1. 在.NET Core 应用程序与 MySQL 之间

通过.NET Core EF（Entity Framework）实现与 MySQL 数据库的通信。在上述创建的数据库上下文类 `BankContext` 中，使用 `DbContextOptions` 配置数据库连接字符串，然后就可以在应用程序的各个业务逻辑层（如控制器、服务类等）通过注入 `BankContext` 来执行数据库操作，例如查询、插入、更新和删除账户信息等。

#### 2. 在数据存储实例节点与分片元数据服务（基于 ETCD）之间

数据存储实例节点（即运行 MySQL 数据库的 Pod）需要定时向 ETCD 集群发送心跳信息和分片相关信息（如当前节点负责的分片范围等）。可以使用 ETCD 的客户端库（如 `EtcdNet`）在.NET Core 应用程序中实现与 ETCD 的交互。

例如，以下是一个完整的代码用于向 ETCD 发送心跳信息（假设已经配置好 `EtcdClient`）：

```csharp
// 心跳服务类，用于向 ETCD 发送节点心跳信息
public class HeartbeatService
{
    // ETCD 客户端实例
    private readonly EtcdClient _etcdClient;
    // 当前节点 ID
    private readonly string _nodeId;

    // 构造函数，接收 ETCD 客户端和节点 ID
    public HeartbeatService(EtcdClient etcdClient, string nodeId)
    {
        _etcdClient = etcdClient;
        _nodeId = nodeId;
    }

    // 发送心跳信息的方法
    public async Task SendHeartbeat()
    {
        // 心跳信息存储在 ETCD 中的键，格式为 node/{nodeId}/heartbeat
        var key = $"node/{_nodeId}/heartbeat";
        // 心跳信息的值为当前时间的字符串表示
        var value = DateTime.UtcNow.ToString();

        // 使用 ETCD 客户端将心跳信息写入 ETCD
        await _etcdClient.PutAsync(key, value);
    }
}
```

在这个示例中，每个数据存储实例节点都有一个唯一的 `nodeId`，通过将当前时间作为心跳信息的值发送到 ETCD 中指定的键（`node/{nodeId}/heartbeat`）下，实现向分片元数据服务集群（ETCD）同步心跳信息。

### 三、银行业务逻辑规则与范围分片

#### 1. 范围分片逻辑

根据银行业务需求，假设我们按照账户余额的范围进行分片。例如，将账户余额划分为以下几个范围：

- 0 - 10000：由节点 1 负责存储。
- 10001 - 50000：由节点 2 负责存储。
- 50001 - 100000：由节点 3 负责存储。

在.NET Core 应用程序中，当插入或查询账户信息时，需要根据账户余额确定该账户应该存储在哪个节点上。以下是完整的代码用于实现这个逻辑：

```csharp
// 分片服务类，用于确定账户数据应存储在哪个节点
public class ShardingService
{
    // 存储分片范围信息的列表
    private readonly List<ShardRange> _shardRanges;

    // 构造函数，初始化分片范围列表
    public ShardingService()
    {
        _shardRanges = new List<ShardRange>()
        {
            // 定义每个分片范围及对应的节点 ID
            new ShardRange { MinBalance = 0, MaxBalance = 10000, NodeId = "node1" },
            new ShardRange { MinBalance = 10001, MaxBalance = 50000, NodeId = "node2" },
            new ShardRange { MinBalance = 50001, MaxBalance = 100000, NodeId = "node3" }
        };
    }

    // 根据账户余额获取对应的节点 ID
    public string GetNodeForAccount(decimal balance)
    {
        // 遍历分片范围列表
        foreach (var range in _shardRanges)
        {
            // 如果账户余额在当前分片范围内，则返回对应的节点 ID
            if (balance >= range.MinBalance && balance <= range.MaxBalance)
            {
                return range.NodeId;
            }
        }

        // 如果账户余额不在任何定义的分片范围内，则抛出异常
        throw new InvalidOperationException("Account balance out of sharding ranges.");
    }
}

// 分片范围类，用于存储分片范围的最小余额、最大余额和对应的节点 ID
public class ShardRange
{
    public decimal MinBalance = 0;
    public decimal MaxBalance = 0;
    public string NodeId = "";
}
```

这里定义了一个 `ShardingService` 类，其中包含了各个分片范围的信息以及一个方法 `GetNodeForAccount`，用于根据账户余额确定账户应该存储在哪个节点上。

#### 2. 分片元数据服务控制数据流向

分片元数据服务（基于 ETCD）存储了各个节点的分片范围信息以及节点的状态信息（通过心跳信息判断）。当应用程序需要插入账户信息时，首先通过 `ShardingService` 确定账户应该存储在哪个节点上（如上述代码所示），然后根据节点 ID 获取该节点对应的数据库连接信息（这部分可以在应用程序启动时从 ETCD 中读取并缓存各个节点的数据库连接信息），最后通过.NET Core EF 将数据插入到对应的节点数据库中。

例如，以下是一个完整的代码片段展示如何在插入账户信息时利用分片元数据服务：

```csharp
// 账户服务类，用于处理账户相关的业务逻辑，如插入账户信息
public class AccountService
{
    // 分片服务实例，用于确定账户所属节点
    private readonly ShardingService _shardingService;
    // 存储各个节点的数据库上下文实例的字典
    private readonly Dictionary<string, BankContext> _bankContexts;

    // 构造函数，接收分片服务实例
    public AccountService(ShardingService shardingService)
    {
        _shardingService = shardingService;
        _bankContexts = new Dictionary<string, BankContext>();
    }

    // 插入账户信息的方法
    public async Task InsertAccount(AccountInfo accountInfo)
    {
        // 根据账户余额确定账户所属节点 ID
        var nodeId = _shardingService.GetNodeForAccount(accountInfo.Balance);
        if (!_bankContexts.ContainsKey(nodeId))
        {
            // 从 ETCD 中读取节点的数据库连接信息并创建 BankContext
            var connectionString = await GetConnectionStringFromEtcd(nodeId);
            var options = new DbContextOptionsBuilder<BankContext>()
           .UseMySql(connectionString, ServerVersion.AutoDetect(connectionString))
           .Build();
            _bankContexts[nodeId] = new BankContext(options);
        }

        // 获取对应节点的数据库上下文实例
        var context = _bankContexts[nodeId];
        // 将账户信息添加到数据库上下文中
        context.AccountInfos.Add(accountInfo);
        // 保存更改到数据库
        await context.SaveChangesAsync();
    }

    // 从 ETCD 中获取节点数据库连接字符串的方法（此处简化实现，实际需根据 ETCD 存储结构获取）
    private async Task<string> GetConnectionStringFromEtcd(string nodeId)
    {
        // 这里假设从 ETCD 中读取节点的数据库连接字符串的具体实现
        // 例如，通过 ETCD 的键值查找，根据节点 ID 找到对应的连接字符串键值对并返回
        return "your_connection_string";
    }
}
```

在这个示例中，`AccountService` 在插入账户信息时，先通过 `ShardingService` 确定账户所属节点，然后从 ETCD 获取该节点的数据库连接信息（这里简化了从 ETCD 读取的过程），创建对应的 `BankContext`，最后将账户信息插入到该节点的数据库中。

### 四、搭建 ETCD 集群及实现分片元数据服务的可用性和数据一致性

#### 1. 搭建 ETCD 集群（在 K8S 中）

以下是一个完整的 ETCD 集群在 K8S 中的部署 YAML 文件示例（假设部署 3 个 ETCD 节点）：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd-cluster
  labels:
    app: etcd
spec:
  serviceName: etcd
  replicas: 3
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:latest
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 22379
          name: peer
        volumeMounts:
        - name: etcd-data
          mountPath: /var/etcd
        command:
        - etcd
        - --name=$(HOSTNAME)
        - --initial-advertise-peer-urls=http://$(HOSTNAME):22379
        - --listen-peer-urls=http://$(HOSTNAME):22379
        - --listen-client-urls=http://$(HOSTNAME):2379
        - --advertise-client-urls=http://$(HOSTNAME):2379
        - --initial-cluster=$(HOSTNAME)=http://$(HOSTNAME):22379
        - --initial-cluster-state=new
        - --data-dir=/var/etcd
      volumes:
      - name: etcd-data
        emptyDir: {}
```

This YAML file deploys a StatefulSet of 3 ETCD nodes in K8S. The ETCD nodes are configured with appropriate ports, volume mounts, and initial cluster settings.

#### 2. 实现分片元数据服务的可用性和数据一致性

ETCD 本身具有强一致性和高可用性的特性。在.NET Core 应用程序中，通过使用 ETCD 的客户端库（如 `EtcdNet`）来与 ETCD 集群进行交互，可以确保对分片元数据（如节点分片范围、节点状态等）的读写操作的一致性。

例如，当多个数据存储实例节点同时向 ETCD 发送心跳信息或更新分片范围信息时，ETCD 会根据其内部的一致性算法（如 Raft 算法）来保证数据的一致性。同时，ETCD 的多副本机制确保了在部分节点出现故障时，服务仍然可用。

### 五、监控服务节点健康及动态添加删除节点

#### 1. 监控服务节点健康

在.NET Core 应用程序中，可以通过定时查询 ETCD 中的心跳信息来监控各个数据存储实例节点的健康状况。以下是完整的代码片段用于实现这个功能：

```csharp
// 节点健康监控类，用于检查节点在 ETCD 中的心跳信息以判断健康状态
public class NodeHealthMonitor
{
    // ETCD 客户端实例
    private readonly EtcdClient _etcdClient;

    // 构造函数，接收 ETCD 客户端实例
    public NodeHealthMonitor(EtcdClient etcdClient)
    {
        _etcdClient = etcdClient;
    }

    // 检查节点是否健康的方法，根据节点 ID 查询其心跳信息并判断时间间隔
    public async Task<bool> IsNodeHealthy(string nodeId)
    {
        // 心跳信息在 ETCD 中的键，格式为 node/{nodeId}/heartbeat
        var key = $"node/{nodeId}/heartbeat";
