# RMDelegationTokenSecretManager 深度解析

## 一、核心原理

### 1.1 架构概述

```
+---------------------------------------------------------------------+
| RMDelegationTokenSecretManager                                      |
+---------------------------------------------------------------------+
| 继承: AbstractDelegationTokenSecretManager<RMDelegationTokenIdentifier>
| 实现: Recoverable (支持RM HA故障恢复)                                 |
+---------------------------------------------------------------------+
| 核心职责:                                                            |
| 1. 生成/验证 delegation token密码                                    |
| 2. 管理密钥生命周期 (密钥滚动)                                        |
| 3. 令牌续期与撤销                                                     |
| 4. 状态持久化 (ZooKeeper/LevelDB)                                    |
+---------------------------------------------------------------------+
```

### 1.2 密码生成机制

Delegation Token的密码基于HMAC-SHA256算法生成:

```java
// AbstractDelegationTokenSecretManager.createPassword()
byte[] password = createPassword(identifier.getBytes(), currentKey.getKey());
```

**生成过程**:

1. 令牌标识符序列化 → `identifier.getBytes()`
2. 使用当前Master Key的SecretKey进行HMAC-SHA256签名
3. 生成256位(32字节)密码

### 1.3 密钥滚动机制 (Key Rolling)

```
时间线: [Key1活跃期] ----→ [Key1过期但保留用于验证旧token] ----→ [Key1销毁]
                ↓
       [Key2成为新Key]
```

**密钥状态**:

| 状态 | 说明 |
|------|------|
| `currentKey` | 当前用于生成新token的主密钥 |
| `allKeys` | 缓存所有密钥(包含历史密钥用于验证) |
| `expiryDate` | 密钥过期时间 = 创建时间 + keyUpdateInterval + tokenMaxLifetime |

**滚动触发** (通过 `ExpiredTokenRemover` 线程):

```java
// 每 keyUpdateInterval 毫秒触发一次
if (lastMasterKeyUpdate + keyUpdateInterval < now) {
    rollMasterKey(); // 滚动到新密钥
}
```

### 1.4 令牌生命周期

```
创建 ──→ 续期 ──→ 续期 ──→ ... ──→ 过期/撤销
│        │        │
│        │        └──────── 每次续期: now + tokenRenewInterval
│        │
│        └─ 初始renewDate: now + tokenRenewInterval
│
└─ 最大生命周期: tokenMaxLifetime
```

**关键约束**:

- `tokenMaxLifetime`: 令牌最大生存期 (默认7天)
- `tokenRenewInterval`: 令牌续期间隔 (默认24小时)
- `keyUpdateInterval`: 密钥更新间隔 (默认1天)

### 1.5 线程安全设计

使用 `ReentrantReadWriteLock` 保护所有状态:

```java
private final ReentrantReadWriteLock apiLock = new ReentrantReadWriteLock(true);

// 读操作: retrievePassword, verifyToken
apiLock.readLock().lock();

// 写操作: createPassword, renewToken, cancelToken
apiLock.writeLock().lock();
```

### 1.6 故障恢复 (HA支持)

实现 `Recoverable` 接口，RM重启时恢复状态:

```java
public void recover(RMState rmState) {
    // 1. 恢复MasterKey
    for (DelegationKey dtKey : rmState.getRMDTSecretManagerState().getMasterKeyState()) {
        addKey(dtKey);
    }
    // 2. 恢复Token
    for (Map.Entry<RMDelegationTokenIdentifier, Long> entry : rmDelegationTokens.entrySet()) {
        addPersistedDelegationToken(entry.getKey(), entry.getValue());
    }
}
```

---

## 二、核心流程图

### 2.1 令牌创建流程

```
Client                                                    RM
  │                                                        │
  │           getDelegationToken()                        │
  │  ───────────────────────────────────────────────────→  │
  │                                                        │
  │  1. 创建RMDelegationTokenIdentifier                   │
  │  2. 生成序列号、设置issueDate/maxDate                  │
  │  3. 使用currentKey生成密码                             │
  │  4. 存储到currentTokens                               │
  │  5. 持久化到StateStore                                 │
  │                                                        │
  │  Token(RMDelegationTokenIdentifier, password)         │
  ←───────────────────────────────────────────────────────  │
```

### 2.2 令牌验证流程

```
Client                                                    RM
  │                                                        │
  │           RPC with Token                              │
  │  ──────────────────────────────────→                  │
  │                                                        │
  │  1. 解码Token Identifier                              │
  │  2. 从currentTokens查找                                │
  │  3. 验证renewDate未过期                                │
  │  4. 验证密码(使用masterKey重新计算比对)                 │
  │                                                        │
  │           允许/拒绝访问                                │
  ←─────────────────────────────────→                     │
```

### 2.3 令牌续期流程

```
Client                                                    RM
  │                                                        │
  │           renewDelegationToken()                      │
  │  ───────────────────────────────────────────────────→  │
  │                                                        │
  │  1. 验证token未过期 (maxDate)                          │
  │  2. 验证renewer匹配                                    │
  │  3. 验证密码正确                                       │
  │  4. 计算新续期时间 = min(maxDate, now + renewInterval) │
  │  5. 更新currentTokens + StateStore                    │
  │                                                        │
  │           newExpirationTime                           │
  ←───────────────────────────────────────────────────────  │
```

---

## 三、使用场景

### 3.1 典型场景: 长时间运行的任务

**场景描述**:

- 用户提交一个需要运行多天的MapReduce作业
- 作业期间Kerberos票据可能过期
- 用户不希望频繁重新登录

**解决方案**:

```java
// 1. 获取 delegation token
Token<RMDelegationTokenIdentifier> token = rm.getDelegationToken(ugi);

// 2. 将token写入到作业上下文
job.getCredentials().addToken(RMDelegationTokenIdentifier.KIND_NAME, token);

// 3. 作业运行期间，YARN自动使用token认证

// 4. 作业结束前可以手动续期
token.renew(conf);
```

### 3.2 代理用户场景 (Proxy User)

**场景描述**:

- 用户A (realUser) 提交作业，由用户B (owner) 的服务代理执行
- 需要以用户A的身份访问RM

```java
// 创建token时指定realUser
RMDelegationTokenIdentifier identifier = new RMDelegationTokenIdentifier(owner, renewer, realUser);
```

### 3.3 YARN RM HA场景

**场景描述**:

- RM发生故障切换到Standby
- 新的Active RM需要继续验证已有token

**解决方案**:

- StateStore (ZooKeeper/LevelDB) 持久化:
  - MasterKey集合
  - 所有DelegationToken状态
  - 序列号

---

## 四、关键配置参数

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `yarn.resourcemanager.delegation.key.update-interval` | 1天 | 密钥更新间隔 |
| `yarn.resourcemanager.delegation.token.max-lifetime` | 7天 | 令牌最大生命周期 |
| `yarn.resourcemanager.delegation.token.renew-interval` | 24小时 | 令牌续期间隔 |
| `yarn.resourcemanager.delegation.token.remover-scan-interval` | 1小时 | 过期令牌扫描间隔 |

---

## 五、安全特性总结

1. **密钥轮换**: 定期滚动主密钥，即使当前密钥泄露，历史token也无法被验证

2. **密码验证**: 使用HMAC-SHA256，密码和密钥不直接暴露

3. **权限校验**: 只有owner或指定的renewer可以续期/撤销

4. **过期机制**: 令牌和密钥都有过期时间

5. **持久化安全**: 状态存储在RM的StateStore中，支持加密

---

## 六、源码位置

- **RMDelegationTokenSecretManager**: `hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/src/main/java/org/apache/hadoop/yarn/server/resourcemanager/security/RMDelegationTokenSecretManager.java`

- **RMDelegationTokenIdentifier**: `hadoop-yarn-project/hadoop-yarn/hadoop-yarn-common/src/main/java/org/apache/hadoop/yarn/security/client/RMDelegationTokenIdentifier.java`

- **AbstractDelegationTokenSecretManager**: `hadoop-common-project/hadoop-common/src/main/java/org/apache/hadoop/security/token/delegation/AbstractDelegationTokenSecretManager.java`