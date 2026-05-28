# 从零构建 CEX 钱包系统：架构、风控与工程实践

> 本文基于 [cex-wallet](https://github.com/lbc-team/cex-wallet) 开源项目的源码阅读与完整实操验证，系统阐述中心化交易所钱包的架构设计、核心技术实现和风控体系。适合区块链后端开发者、交易所基础设施工程师阅读。

---

## 前言

中心化交易所（CEX）钱包系统不同于个人钱包——它需要同时管理数十万用户的地址，处理高频的充提币请求，并在安全性和效率之间找到平衡。一个合格的 CEX 钱包系统至少要解决三个核心问题：

1. **如何安全地生成和管理海量用户地址？** 不能每次充币都创建新私钥，但也不能让所有用户共用地址。
2. **如何可靠地检测链上充值？** 区块链可能发生重组（reorg），交易可能被替换，不能简单地"看到交易就加余额"。
3. **如何防止内部作恶和外部攻击？** 如果数据库被直接篡改，或者有人绕过风控发起提币，整个系统就会崩塌。

下面结合 cex-wallet 的实现，逐一拆解这些问题。

---

## 一、架构总览

cex-wallet 采用微服务架构，将钱包系统拆分为 6 个独立模块：

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Signer   │    │  Wallet   │    │  Scan    │
│ 签名机    │    │ 钱包API   │    │ 链上扫描  │
│ :3001    │    │ :3000    │    │ :3002    │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
                     │  业务签名 + 风控签名（Ed25519 双签）
                     ▼
              ┌──────────────┐
              │  DB Gateway   │  ← 数据库安全网关
              │  :3003        │    所有写操作必须经过双签验证
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐    ┌──────────────┐
              │   SQLite DB   │    │  Risk Control │
              │  wallet.db    │    │  风控服务 :3004│
              └──────────────┘    └──────────────┘
```

### 模块职责

| 模块 | 职责 | 关键能力 |
|------|------|----------|
| **Signer** | 密钥派生与交易签名 | 基于 BIP44 从助记词派生无限子地址；签名前需输入密码解密 |
| **Wallet** | 面向用户的 API | 创建地址、查询余额、发起提现 |
| **Scan** | 区块链扫描器 | 逐区块扫描，检测充值交易；支持重组处理 |
| **DB Gateway** | 数据库安全网关 | Ed25519 双签验证、防重放、敏感表强制风控 |
| **Risk Control** | 风控引擎 | 黑名单检测、大额预警、人工审核工作流 |

### 核心数据流：一次充值

```
用户向地址A转入 1 ETH
    │
    ▼
Scan 扫描到交易 ──► Risk Control 风控评估 ──► DB Gateway 写入 credits 表
                        │                           │
                        │ 评估结果：                 │ 验证：
                        │ - approve（正常入账）       │ - business_signature ✓
                        │ - freeze（黑名单地址）      │ - risk_signature ✓
                        │                           │ - operation_id 未使用 ✓
                        │                           │ - 时间戳在窗口内 ✓
                        └───────────────────────────┘
```

---

## 二、签名机：一个助记词管理无限地址

这是 CEX 钱包最核心的设计决策。

### 为什么不用"每用户一把私钥"？

如果给每个用户生成独立的私钥，交易所需要安全存储数百万个私钥，运维复杂度爆炸，且私钥泄露风险随规模线性增长。

### BIP44 分层确定性派生

cex-wallet 的方案是：**一个主助记词，通过 BIP44 派生路径生成无限子地址。**

```
助记词（12个单词）
  └── m/44'/60'/0'/0/0  →  地址0  (验证地址)
  └── m/44'/60'/0'/0/1  →  地址1  (热钱包)
  └── m/44'/60'/0'/0/2  →  地址2  (用户1)
  └── m/44'/60'/0'/0/3  →  地址3  (用户2)
  └── ...
  └── m/44'/60'/0'/0/N  →  地址N
```

关键设计：
- **索引自增**：数据库记录当前最大索引 `index_value`，每次创建新地址时 `index + 1`
- **验证地址**：首次启动时在 `m/44'/60'/0'/0/0` 创建验证地址并持久化；之后每次启动重新派生该地址，与数据库比对——密码错误则地址不匹配，服务拒绝启动
- **多链支持**：EVM 用 `m/44'/60'/0'/0/{i}`，Solana 用 `m/44'/501'/0'/0'`，BTC 用 `m/84'/1'/0'/0/{i}`

```typescript
// Signer 启动时的密码验证逻辑（简化为伪代码）
const testAddress = deriveAddress(mnemonic, password, "m/44'/60'/0'/0/0");
const storedAddress = db.getVerificationAddress();
if (testAddress !== storedAddress) {
  throw new Error("密码错误，服务启动失败");
}
```

### 这个方案的本质

**将"密钥管理"降维为"索引管理"**。安全性由助记词 + 密码保护，地址生成只需递增一个整数。即使数据库被拖走，没有助记词和密码也无法恢复私钥。

---

## 三、数据库安全网关：为什么不直接操作数据库

传统方案中，Wallet 和 Scan 服务直接连接数据库进行读写。这在安全上有一个致命缺陷：**任何模块的代码漏洞或内部人员都可能绕过业务逻辑直接篡改余额**。

### 双签名机制

DB Gateway 通过 Ed25519 非对称签名实现了"零信任"的数据库访问：

```
请求结构:
{
  "operation_id": "uuid",      // 唯一操作ID，防重放
  "operation_type": "sensitive", // read | write | sensitive
  "table": "credits",
  "action": "insert",
  "data": { ... },
  "business_signature": "abc...",  // 业务模块签名（Wallet 或 Scan）
  "risk_signature": "def...",      // 风控签名（Risk Control）
  "timestamp": 1640995200000
}
```

**验证流程**（`signature.ts` 三层中间件）：

1. **validateRequest**：校验字段完整性 → 敏感表强制 `operation_type: 'sensitive'` → 时间戳窗口（60秒）→ operation_id 去重
2. **verifyBusinessSignature**：使用 Wallet/Scan 公钥验证业务签名
3. **verifyRiskControlSignature**：如果是 sensitive 操作，使用 Risk 公钥验证风控签名

### 敏感表强制风控

```typescript
// sensitive-tables.ts
export const SENSITIVE_TABLES = [
  {
    table: 'credits',
    actions: ['insert', 'update', 'delete'],
    reason: '包含用户余额信息'
  },
  {
    table: 'withdraws',
    actions: ['insert', 'update'],
    reason: '提现操作需要风控'
  }
];
```

任何对 `credits` 表的写入操作，如果 `operation_type` 不是 `'sensitive'` 或缺少 `risk_signature`，DB Gateway 会直接拒绝。**这意味着 WALLET/SCAN 的私钥泄露不足以篡改余额——攻击者还需要同时控制风控模块的私钥**。

### 防重放攻击

操作 ID 被设计为一次性 nonce。Gateway 在验证通过后将 `operation_id` 写入 `used_operation_ids` 表，重复使用立即拒绝。配合 60 秒时间戳窗口，即使签名被截获，攻击窗口也极其有限。

---

## 四、区块链扫描：充值检测的工程难点

Scan 服务是充提币流程的"眼睛"，负责监听链上交易并将充值事件写入系统。

### 核心挑战

1. **不要漏掉交易**：区块重组（reorg）可能导致已确认的交易被回滚
2. **不要重复入账**：同一个交易被扫描两次会导致用户余额翻倍
3. **区分交易类型**：ETH 转账（native）、ERC20 转账（Transfer 事件）、合约内部转账（internal tx）

### cex-wallet 的实现

```typescript
// 批量优化区块分析（简化逻辑）
async function analyzeBlocks(fromBlock, toBlock) {
  // 1. 获取区块范围内的所有日志（ETH转账 + ERC20 Transfer事件）
  const logs = await client.getLogs({ fromBlock, toBlock });

  // 2. 匹配监控地址列表
  for (const log of logs) {
    if (monitoredAddresses.includes(log.address)) {
      // 3. 构建 deposit 记录
      // 4. 调用 Risk Control 进行风控评估
      // 5. 获取风控签名后，通过 DB Gateway 写入 credits 表
    }
  }

  // 6. 处理重组确认
  await processReorgConfirmation(fromBlock, toBlock);
}
```

### 确认策略

系统引入了分层确认状态：

```
confirmed (已确认) → safe (安全) → finalized (最终)
```

- **confirmed**：交易已被包含在区块中，但可能还在重组深度内
- **safe**：经过 `CONFIRMATION_BLOCKS` 个区块确认（默认 32），重组概率极低
- **finalized**：最终确认，余额正式计入可用

这种设计既保证了用户体验（confirmed 即可看到充值），又避免了重组导致的资金风险。

---

## 五、风控体系：不止是黑名单

cex-wallet 的风控设计体现了"纵深防御"的思想。

### 决策分级

| 风险级别 | 决策 | 行为 |
|----------|------|------|
| low | auto_approve | 自动批准，返回风控签名 |
| medium/high | manual_review | 创建审核记录，等待人工处理 |
| critical | deny | 直接拒绝（如黑名单地址） |

### 黑名单的精细化处理

一个容易被忽略的细节：**黑名单地址发起的充值不应该直接拒绝，而应该入账但冻结**。

原因很简单——如果直接拒绝，黑名单地址会发现自己被拉黑了，换一个地址继续作恶。而入账冻结则让攻击者无法转移资金，同时不知道已被风控标记。

实操验证中，用户2 的 USDT 余额显示 `frozen_balance: 300.0`，正是来自黑名单地址 `0x70997970...` 的转账被冻结。

### 大额提现的多层拦截

大额提现可能来自：
- 用户正常操作
- 账户被盗
- 内部作恶

cex-wallet 的风控对"大额"的定义不是一刀切——风控评估后还可以给出**建议金额**（`suggest_operation_data`），例如用户请求提现 20 OPS，风控建议降为 1 OPS 分批提现。审核员可以选择接受建议或批准原始金额。

---

## 六、业务数据流：一个提现请求的全生命周期

这是最值得理解的流程，因为它串联了所有模块。

```
1. 用户发起提现
   POST /api/user/withdraw { userId, to, amount, tokenSymbol, chainType }

2. Wallet 服务创建 withdraw 记录 (status: user_withdraw_request)
   └── DB Gateway 验证 business_signature + risk_signature → 写入 withdraws 表

3. Risk Control 提现风险评估
   └── 检查金额阈值 / 目标地址风险 → decision + risk_signature

4. Wallet 调用 Signer 签名交易
   └── Signer 使用助记词 + 密码解密私钥 → 构建并签名原始交易

5. 交易广播到链上
   └── status: pending

6. Scan 监控交易确认
   └── status: processing → confirmed

7. Credit 记录更新
   └── 用户余额扣减 (amount: -xxx)
   └── 热钱包余额扣减 (amount: -actual)  ← 实际转出金额 = 提现金额 - 手续费
```

### 手续费的账务处理

注意第 7 步中的两条 credit 记录：用户被扣了 0.01 ETH，但热钱包实际只转出了 0.0099 ETH。这 0.0001 ETH 的差额就是被"燃烧"的链上 Gas 费。当前版本尚未单独记录这部分费用，这也是 Todo.md 中提到的待改进点。

---

## 七、实操经验

在本地环境完整跑通 cex-wallet 后，有几个值得分享的点：

### 配置复杂度

启动 6 个微服务，需要生成 3 对 Ed25519 密钥（Wallet/Scan/Risk），配置 5 个 `.env` 文件，公钥私钥交叉引用。如果某一步出错——比如 Scan 的公钥配成了 Wallet 的——DB Gateway 会静默拒绝请求，错误日志藏在中间件里，排查起来不太直观。

**建议**：实际生产部署时，应该有一个初始化脚本自动完成密钥生成和配置分发。

### 冷热钱包分离

系统为热钱包（用户ID=1）分配了独立地址 `0xF4A4378A...`，所有用户的提现实际上是从这个热钱包地址转出。可以进一步扩展为冷热分离架构：
- **热钱包**：存放日常提现所需资金（如总量的 5%）
- **冷钱包**：大额资产离线存储
- **资金调度**：定期从冷钱包向热钱包归集

项目中的 `fund_rebalance` 模块正是为这个场景预留的。

### 重组的实际影响

Anvil 本地测试网 `block_time=1` 秒，`CONFIRMATION_BLOCKS=0`，所以不存在重组问题。但在以太坊主网，建议至少等 12 个区块确认（~2.5 分钟），币安智能链可以等 20+ 个区块。

---

## 八、架构的可改进方向

基于代码阅读和实践，我认为以下方向值得投入：

1. **EIP-7702 批量提现**：高峰期大量用户同时提现时，逐笔签名广播效率极低。EIP-7702 允许将多笔转账打包为一笔交易。
2. **ETH 内部转账检测**：目前 Scan 只检测 `eth_getLogs`，不支持合约内部转账（如通过 `call` 转入 ETH）。需集成 `debug_traceBlock` 或 `trace_block`。
3. **私钥的安全升级**：当前助记词密码解密后私钥在内存中明文存在。生产环境应使用 HSM（硬件安全模块）或至少使用 MPC（多方计算）方案。
4. **风控规则扩展**：目前仅实现了黑名单检测和大额预警。实际还需要单日限额、提现频率限制、新地址白名单等。
5. **API 鉴权**：当前所有 API 无需认证，生产环境必须加入 JWT 或 API Key 验证。

---

## 结语

CEX 钱包系统的本质是一个**分布式状态机**：链上状态（区块）是输入，用户余额是输出，中间的签名机、扫描器、风控、网关共同构成了一个带有多重校验的状态转换管道。

安全不在于某个环节的绝对可靠，而在于**每一层都假设前一层可能被攻破**——Scan 不知道 Wallet 是否正常，所以要求 Wallet 签名；DB Gateway 不信任任何调用方，所以要求双签；Risk Control 不假设交易都是善意的，所以拦截异常行为。

这种"零信任"的架构哲学，是 CEX 钱包设计的核心原则，也是这个开源项目最值得学习的地方。

---

*作者基于 [cex-wallet](https://github.com/lbc-team/cex-wallet) (Apache-2.0) 的源码分析和实操经验撰写。欢迎交流讨论。*
