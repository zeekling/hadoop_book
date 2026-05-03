# Apache Ozone CLI 模块和安全模块详细分析报告

## 一、CLI 模块概述

Apache Ozone 项目包含四个主要的 CLI（命令行接口）模块，用于不同的管理、调试和修复操作。这些模块基于 **Picocli** 框架实现。

---

## 二、cli-admin 模块（Ozone 管理工具）

### 2.1 核心功能描述

**cli-admin** 模块是 Ozone 集群的核心管理工具，提供了针对 **Ozone Manager (OM)** 和 **Storage Container Manager (SCM)** 的管理员操作命令：

- 管理 OM 和 SCM 集群的元数据操作
- 节点管理（退役、重新加入、维护模式）
- 管道（Pipeline）管理
- 容器管理
- 复制管理器（Replication Manager）控制
- 安全模式（Safe Mode）控制
- 磁盘均衡器（Disk Balancer）操作
- 证书管理
- 升级相关的操作

### 2.2 核心类继承关系

```java
OzoneAdmin (主入口类)
├── 实现接口: ExtensibleParentCommand
├── 继承: Shell
└── 子命令:
    ├── OMAdmin
    │   ├── FinalizeUpgradeSubCommand
    │   ├── ListOpenFilesSubCommand
    │   ├── GetServiceRolesSubcommand
    │   ├── PrepareSubCommand
    │   ├── CancelPrepareSubCommand
    │   ├── TransferOmLeaderSubCommand
    │   └── SnapshotSubCommand
    └── ScmAdmin
        ├── GetScmRatisRolesSubcommand
        ├── FinalizeScmUpgradeSubcommand
        ├── TransferScmLeaderSubCommand
        ├── DecommissionScmSubcommand
        ├── RotateKeySubCommand
        └── DeletedBlocksTxnCommands
```

### 2.3 重要命令

```bash
# OM 管理
ozone admin om finalize-upgrade
ozone admin om get-service-roles
ozone admin om transfer-leader <hostname>
ozone admin om snapshot create <vol/bucket> <name>

# SCM 管理
ozone admin scm get-scm-roles
ozone admin scm finalize-upgrade
ozone admin scm transfer-scm-leader <hostname>

# 节点管理
ozone admin scm datanode decommission <hosts>
ozone admin scm datanode recommission <hosts>
ozone admin scm datanode list
ozone admin scm datanode status <host>

# 管道管理
ozone admin scm pipeline list
ozone admin scm pipeline activate <id>
ozone admin scm pipeline deactivate <id>

# 安全模式
ozone admin scm safemode enter
ozone admin scm safemode exit
```

---

## 三、cli-shell 模块（Ozone Shell 命令行）

### 3.1 核心功能描述

**cli-shell** 模块是 Ozone 面向用户的主要 Shell 工具，提供了对 Volume、Bucket、Key、Prefix、Snapshot、Token 和 Tenant 的完整操作能力。

### 3.2 核心类继承关系

```java
OzoneShell (主入口类)
├── 继承: Shell
└── 子命令:
    ├── VolumeCommands (volume)
    │   └── VolumeHandler
    ├── BucketCommands (bucket)
    │   └── BucketHandler
    ├── KeyCommands (key)
    │   └── KeyHandler
    ├── PrefixCommands (prefix)
    ├── SnapshotCommands (snapshot)
    ├── TenantUserCommands (tenant)
    └── TokenCommands (token)
```

### 3.3 重要命令

```bash
# 卷操作
ozone sh volume create <vol-name> --user=<owner>
ozone sh volume info <vol-name>
ozone sh volume list
ozone sh volume delete <vol-name>

# 桶操作
ozone sh bucket create <vol>/<bucket>
ozone sh bucket info <vol>/<bucket>
ozone sh bucket list <vol>
ozone sh bucket delete <vol>/<bucket>

# 键操作
ozone sh key put <vol>/<bucket>/<key> <local-file>
ozone sh key get <vol>/<bucket>/<key> <local-file>
ozone sh key delete <vol>/<bucket>/<key>
ozone sh key list <vol>/<bucket>

# 快照操作
ozone sh snapshot create <vol>/<bucket> <snapshot-name>
ozone sh snapshot delete <vol>/<bucket>/<snapshot-name>
ozone sh snapshot list <vol>/<bucket>

# 租户操作
ozone sh tenant create <tenant-id>
ozone sh tenant list
ozone sh tenant info <tenant-id>
ozone sh tenant delete <tenant-id>
```

---

## 四、cli-debug 模块（调试工具）

### 4.1 核心功能描述

**cli-debug** 模块是 Ozone 的调试和诊断工具，主要用于开发人员和运维人员排查问题：

- RocksDB 数据库解析
- 日志解析
- Kerberos 诊断
- Ratis 日志分析
- 容器数据验证

### 4.2 重要命令

```bash
# RocksDB 操作
ozone debug ldb scan --db=<path> --column_family=<table>
ozone debug ldb listtables --db=<path>

# Datanode 容器操作
ozone debug container list --db=<path>
ozone debug container info --db=<path> --container-id=<id>

# OM 调试
ozone debug om container-to-key-mapping --container-id=<id>

# Kerberos 诊断
ozone debug kerberos diagnose
```

---

## 五、cli-repair 模块（修复工具）

### 5.1 核心功能描述

**cli-repair** 模块是 Ozone 的高级修复工具，主要用于在紧急情况下修复损坏的数据或元数据：

- 快照链修复
- OM 数据库修复
- RocksDB 手动压缩
- FSO 修复
- 事务信息修复
- Datanode 容器 Schema 升级

### 5.2 重要命令

```bash
# 快照修复
ozone repair om snapshot repair-chain --om-db-dir=<path> --snapshot-name=<name>

# OM 数据库压缩
ozone repair om compact --om-db-dir=<path>

# RocksDB 手动压缩
ozone repair ldb compact --db-path=<path>

# Datanode 容器 Schema 升级
ozone repair datanode schema-upgrade --path=<container-path> --schema-version=V3
```

---

## 六、安全模块

### 6.1 multitenancy-ranger 模块（多租户 Ranger 集成）

**multitenancy-ranger** 模块实现了基于 Apache Ranger 的多租户访问控制：

```java
RangerClientMultiTenantAccessController
├── 实现接口: MultiTenantAccessController
├── RangerClient (Ranger SDK)
├── RangerPolicy
├── RangerRole
└── RangerService

// 主要方法
Policy createPolicy(Policy policy)
Policy getPolicy(String policyName)
Role createRole(Role role)
Role getRole(String roleName)
void deletePolicy(String policyName)
long getRangerServicePolicyVersion()
```

### 6.2 s3-secret-store 模块（S3 密钥安全存储）

**s3-secret-store** 模块基于 HashiCorp Vault 实现远程密钥存储：

```java
VaultS3SecretStore (实现类)
├── 实现接口: S3SecretStore
├── 认证方式支持: Kerberos、Token、AppRole
└── 主要方法:
    void storeSecret(String kerberosId, S3SecretValue secret)
    S3SecretValue getSecret(String kerberosID)
    void revokeSecret(String kerberosId)
```

---

## 七、模块依赖关系总结

| 模块 | 依赖模块 |
|------|---------|
| cli-admin | hdds-cli-common, hdds-client, ozone-cli-shell |
| cli-shell | hdds-cli-common, hdds-client, ozone-client |
| cli-debug | ozone-cli-admin, ozone-cli-shell, hdds-managed-rocksdb |
| cli-repair | ozone-cli-debug, ozone-cli-shell, hdds-managed-rocksdb |
| multitenancy-ranger | ozone-manager, hadoop-common |
| s3-secret-store | ozone-manager, vault-client |

---

## 八、总结

Apache Ozone 项目提供了完整的 CLI 工具链，满足不同用户角色的需求：

1. **cli-shell**：最终用户日常操作工具
2. **cli-admin**：管理员集群管理工具
3. **cli-debug**：开发/运维调试诊断工具
4. **cli-repair**：紧急修复工具

同时，安全模块提供了企业级的多租户访问控制和密钥管理功能。