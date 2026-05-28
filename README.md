# CEX 钱包系统

交易所钱包系统，提供安全的钱包管理和地址生成服务。这是一个 Web3 中心化交易所（CEX）托管钱包的完整实现，包含充值监控、提现处理、风控审核等核心功能。

## 系统架构

```
                    ┌─────────────┐
                    │   Signer    │  ← 签名机：地址生成、密钥管理
                    └──────┬──────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
    ▼                      ▼                      ▼
┌───────┐           ┌──────────┐           ┌──────────┐
│ Wallet │◄─────────│   Scan   │──────────►│  Risk    │
│ (API)  │          │ (扫描器)  │           │ Control  │
└───┬───┘           └────┬─────┘           └────┬─────┘
    │                    │                      │
    └────────────────────┼──────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  DB Gateway  │  ← 数据库安全网关（Ed25519签名验证）
                  └──────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │   SQLite DB   │
                  └──────────────┘
```

**核心流程**：
1. **充值流程**：用户链上转账 → Scan 扫描区块检测存款 → Risk Control 风控评估 → DB Gateway 写入 credit 记录
2. **提现流程**：用户发起提现 → Risk Control 风险评估 → Signer 签名交易 → 广播到链上 → Scan 确认 → 更新余额
3. **风控流程**：所有敏感操作（余额变更）必须经过 Risk Control 评估并签名，DB Gateway 验证双签名（业务签名 + 风控签名）后才执行

系统设计和实现思路，参考以下文章：

1. [交易所钱包系统的整体架构设计](https://learnblockchain.cn/article/20345)
2. [签名机与用户账户生成的方案](https://learnblockchain.cn/article/20693)
3. [用户充值](https://learnblockchain.cn/article/20925)
4. [用户提现](https://learnblockchain.cn/article/21061)
5. [风控设计](https://learnblockchain.cn/article/21233)

## 项目结构

```
cex-wallet/
├── wallet/             # 主模块：钱包管理 API (端口 3000)
│   ├── src/            # 源码
│   ├── mock/           # 测试脚本（部署Token、模拟转账、提现等）
│   └── tests/          # 测试用例
├── signer/             # 签名机：地址生成和密钥管理 (端口 3001)
│   └── src/
│       └── services/
│           └── signers/   # evmSigner, solanaSigner, btcSigner
├── scan/               # 区块链扫描器
│   ├── evm_scan/       # EVM链扫描 (端口 3002)
│   └── solana_scan/    # Solana链扫描
├── db_gateway/         # 数据库安全网关 (端口 3003)
├── risk_control/       # 风控模块 (端口 3004)
└── fund_rebalance/     # 资金调度模块
```

## 快速开始

### 环境要求

- **Node.js** >= 18.x
- **npm** >= 9.x
- **Foundry** (用于 anvil 本地以太坊测试网)
- **Solana CLI** (可选，用于 Solana 本地测试网)

### 1. 克隆项目

```bash
git clone https://github.com/1952154539/CEX-wallet.git
cd CEX-wallet
```

### 2. 安装依赖

```bash
# 安装所有模块的依赖
cd db_gateway && npm install && cd ..
cd risk_control && npm install && cd ..
cd signer && npm install && cd ..
cd wallet && npm install && cd ..
cd scan/evm_scan && npm install && cd ../..
cd scan/solana_scan && npm install && cd ../..
```

### 3. 生成密钥对并配置环境变量

#### 3.1 启动 DB Gateway

```bash
# 首先创建 .env 文件
cd db_gateway
cp .env.example .env
# 修改 .env 中的 WALLET_DB_PATH 为实际路径
# 例如：WALLET_DB_PATH=/home/user/CEX-wallet/db_gateway/wallet.db

# 启动服务
npm run dev
```

#### 3.2 生成三个密钥对（Wallet / Scan / Risk）

```bash
# 生成 Wallet 密钥对
curl -X POST http://localhost:3003/generate-keypair

# 生成 Scan 密钥对
curl -X POST http://localhost:3003/generate-keypair

# 生成 Risk 密钥对
curl -X POST http://localhost:3003/generate-keypair
```

#### 3.3 配置各模块 .env 文件

**db_gateway/.env**（配置三个公钥）：
```env
PORT=3003
WALLET_DB_PATH=/home/user/CEX-wallet/db_gateway/wallet.db
WALLET_PUBLIC_KEY=<wallet公钥>
SCAN_PUBLIC_KEY=<scan公钥>
RISK_PUBLIC_KEY=<risk公钥>
```

**risk_control/.env**（配置风控私钥）：
```env
PORT=3004
RISK_CONTROL_DB_PATH=/home/user/CEX-wallet/risk_control/risk_control.db
RISK_PRIVATE_KEY=<risk私钥>
WALLET_SERVICE_URL=http://localhost:3000
```

**signer/.env**（配置助记词和公钥）：
```env
PORT=3001
MNEMONIC=test test test test test test test test test test test junk
SIGNER_DEVICE=signer_device1
WALLET_PUBLIC_KEY=<wallet公钥>
RISK_PUBLIC_KEY=<risk公钥>
```

**wallet/.env**（配置钱包私钥和服务地址）：
```env
PORT=3000
SIGNER_BASE_URL=http://localhost:3001
RISK_CONTROL_URL=http://localhost:3004
WALLET_DB_PATH=/home/user/CEX-wallet/db_gateway/wallet.db
WALLET_PRIVATE_KEY=<wallet私钥>
ETH_RPC_URL=http://localhost:8545
```

**scan/evm_scan/.env**（配置扫描私钥）：
```env
ETH_RPC_URL=http://localhost:8545
WALLET_DB_PATH=/home/user/CEX-wallet/db_gateway/wallet.db
RISK_CONTROL_URL=http://localhost:3004
SCAN_PRIVATE_KEY=<scan私钥>
START_BLOCK=0
CONFIRMATION_BLOCKS=0
SCAN_INTERVAL=2
AUTO_START=true
```

### 4. 启动所有服务

**推荐启动顺序：db_gateway → risk_control → signer → wallet → scan**

```bash
# 终端1：启动 DB Gateway
cd db_gateway && npm run dev

# 终端2：启动 Risk Control
cd risk_control && npm run dev

# 终端3：启动 Signer（需要输入密码，默认 12345678）
cd signer && echo "12345678" | npm run dev

# 终端4：启动 Wallet
cd wallet && npm run dev

# 终端5：启动 Anvil 本地以太坊测试网
anvil --block-time 1

# 终端6：启动 EVM Scan
cd scan/evm_scan && npm run dev
```

### 5. 初始化测试数据

```bash
cd wallet

# 部署 ERC20 测试代币（OPS 和 USDT）
npm run deploy:erc20:tokens

# 初始化测试数据（创建用户、热钱包、代币配置、用户地址）
npm run mock:init
```

执行后会自动创建：
- 3个系统用户（hot_wallet1, hot_wallet2, multisig_wallet）
- 10个普通测试用户（test_user_1 ~ test_user_10）
- 1个EVM热钱包和1个Solana热钱包
- 每个用户分配EVM和Solana钱包地址
- ETH、OPS、USDT 代币配置（chain_id: 31337）

### 6. 模拟充值（存款）

```bash
cd wallet

# 执行模拟转账：向用户钱包地址转入 ETH、OPS、USDT
npm run mock:evm:transfer
```

转账完成后，Scan 服务会自动检测到存款交易，经风控评估后写入数据库。

**验证充值结果**：
```bash
# 查看用户1的总余额
curl http://localhost:3000/api/user/1/balance/total

# 查看用户1的ETH余额详情
curl http://localhost:3000/api/user/1/balance/token/ETH
```

### 7. 模拟提现（取款）

```bash
cd wallet

# 发起提现请求（ETH和OPS）
npm run mock:withdraw:evm

# 大额提现可能被风控拦截，需要人工审核
npm run mock:approveReview
```

**提现流程**：
1. 用户发起提现请求
2. 系统进行风险控制评估（小额自动批准，大额需要人工审核）
3. Signer 签名交易
4. 交易广播到链上
5. Scan 服务监控交易确认
6. 更新用户余额

**验证提现结果**：
```bash
# 查看提现记录详情
curl http://localhost:3000/api/withdraws/1

# 查看用户2的提现历史
curl http://localhost:3000/api/user/2/withdraws
```

## API 接口

各服务默认端口：
- **Wallet**: `http://localhost:3000`
- **Signer**: `http://localhost:3001`
- **DB Gateway**: `http://localhost:3003`
- **Risk Control**: `http://localhost:3004`

### 钱包管理

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/user/{user_id}/address?chain_type=evm` | GET | 获取/创建用户钱包地址 |
| `/api/user/{user_id}/balance/total` | GET | 获取用户所有代币余额总和 |
| `/api/user/{user_id}/balance/pending` | GET | 获取用户充值中的余额 |
| `/api/user/{user_id}/balance/token/{symbol}` | GET | 获取用户指定代币余额详情 |
| `/api/user/withdraw` | POST | 用户发起提现 |
| `/api/user/{user_id}/withdraws` | GET | 获取用户提现记录 |
| `/api/withdraws/{id}` | GET | 获取提现记录详情 |
| `/api/withdraws/pending` | GET | 获取待处理提现列表 |

### 风控管理

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/assess` | POST | 风控评估 |
| `/api/withdraw-risk-assessment` | POST | 提现风险评估 |
| `/api/assessment/:operation_id` | GET | 查询评估结果 |
| `/api/pending-reviews` | GET | 查看待审核列表 |
| `/api/manual-review` | POST | 人工审核 |

详细 API 文档见 [API_USAGE.md](API_USAGE.md)。

## 安全机制

### 双签名验证
DB Gateway 强制要求敏感操作（如余额变更）必须包含双签名：
1. **业务签名**（business_signature）：由 Wallet/Scan 模块使用其私钥生成
2. **风控签名**（risk_signature）：由 Risk Control 模块在风控评估通过后生成

### 防重放攻击
- 每个操作使用唯一的 operation_id（UUID）
- 时间戳验证（5分钟时间窗口）
- DB Gateway 验证签名后记录 operation_id，防止重复使用

### 风控规则
- **黑名单检测**：检查交易对手地址是否在黑名单中
- **大额交易审核**：超过阈值的提现需要人工审核
- **来自黑名单地址的充值**：自动冻结（frozen 状态）

### 密钥管理
- 私钥通过环境变量配置，不写入代码
- 助记词密码保护（signer 启动时需输入密码）
- 公钥公开配置，私钥严格保密

## 实测验证

本项目已完成以下功能验证：

### 充值测试
- ETH 转账到用户钱包地址：成功检测并入账
- ERC20（OPS/USDT）转账到用户钱包地址：成功检测并入账
- 黑名单地址转账：风控系统正确识别并冻结相关余额
- 用户余额查询：正确显示可用余额和冻结余额

### 提现测试
- ETH 小额提现（0.01 ETH）：自动批准，交易签名成功
- OPS 大额提现（20 OPS）：触发风控人工审核机制
- 人工审核通过后：系统正确回调并处理提现
- 提现后余额：正确扣减（含手续费）

### 测试数据示例

```
用户1（热钱包）余额：
  ETH:  20.0 (其中 0.25 冻结 - 来自黑名单地址)
  OPS:  1000.0
  USDT: 80300.0

用户2（普通用户）余额：
  ETH:  0.99 (提现 0.01 ETH 后)
  OPS:  880.0 (大额提现审核后)
  USDT: 900.0 (其中 300.0 冻结)
```

## 相关文档

- [API 使用说明](API_USAGE.md)
- [Signer 模块文档](signer/README.md)
- [Wallet 模块文档](wallet/README.md)
- [EVM Scan 模块文档](scan/evm_scan/README.md)
- [Solana Scan 模块文档](scan/solana_scan/README.md)
- [风控模块文档](risk_control/README.md)
- [数据库网关模块文档](db_gateway/README.md)

## 待办事项

参见 [Todo.md](Todo.md)，包括：
- 热钱包余额精确管理（含手续费）
- 大规模并发提现（EIP-7702 批量提现）
- ETH 合约内部转账支持
- 多链扩展（BTC）
- 资金归集
- 风控增强（单日限额、频率限制等）
- API 鉴权

## 许可证

MIT License
