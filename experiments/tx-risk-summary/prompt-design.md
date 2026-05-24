# 交易风险摘要 Prompt — 设计与测试

> 实践目标：写一个 prompt，让 LLM 对任意交易输出结构化风险分析 JSON

## Prompt 设计

```markdown
你是一个 Web3 交易安全分析器。分析用户提供的交易信息，以 JSON 格式输出。

## 输出格式（严格遵守）

{
  "summary": "<一句话概括这笔交易在做什么>",
  "risk_level": "low | medium | high | critical",
  "risk_factors": ["<具体风险点>"],
  "uncertainties": ["<无法确定但需要关注的点>"],
  "requires_human_approval": true | false
}

## 风险判定规则

- **critical**：无限授权（approve max）、资金发往已知恶意地址、合约调用可疑函数
- **high**：大额转账到未知地址、approve 非零值、调用未验证合约
- **medium**：交互陌生合约、代币转账到新地址、gas 异常高
- **low**：普通 ETH 转账、已知协议交互、小额交易

## 分析原则

1. 不编造信息 — 只基于输入数据判断
2. 不确定的事放进 uncertainties，不要强行下结论
3. requires_human_approval = true 当 risk_level >= high 或 uncertainties 包含安全相关项
```

---

## 测试用例

### 场景 1：普通 ETH 转账

**输入数据：**
```json
{
  "from": "0xAbC...",
  "to": "0xDeF...",
  "value": "0.1 ETH",
  "method": "transfer",
  "gas_limit": "21000",
  "gas_price": "20 gwei"
}
```

**预期输出：**
```json
{
  "summary": "向 0xDeF... 转账 0.1 ETH 的普通交易",
  "risk_level": "low",
  "risk_factors": [],
  "uncertainties": ["接收地址身份未知"],
  "requires_human_approval": false
}
```

**为什么是 low**：标准 transfer，金额小，无合约交互。uncertainties 里有"地址未知"但不影响风险等级 — 这是信息缺失，不是危险信号。

---

### 场景 2：无限授权（approve max）

**输入数据：**
```json
{
  "from": "0xAbC...",
  "to": "0xUSDC_Contract...",
  "value": "0 ETH",
  "method": "approve(address,uint256)",
  "params": {
    "spender": "0xUnverifiedContract...",
    "amount": "115792089237316195423570985008687907853269984665640564039457584007913129639935"
  },
  "gas_limit": "50000"
}
```

**预期输出：**
```json
{
  "summary": "授权 0xUnverifiedContract... 无限额使用你的 USDC",
  "risk_level": "critical",
  "risk_factors": [
    "无限授权（approve max）",
    "被授权合约未验证",
    "授权后对方可随时转走你的全部 USDC"
  ],
  "uncertainties": [],
  "requires_human_approval": true
}
```

**为什么是 critical**：approve max + 未验证合约 = 你的 USDC 随时可能被盗。这是最常见的钓鱼/rug 模式。

---

### 场景 3：目标地址与预期不符

**输入数据：**
```json
{
  "from": "0xAbC...",
  "to": "0xSuspiciousContract...",
  "value": "2 ETH",
  "method": "swapExactETHForTokens",
  "params": {
    "amountOutMin": "0",
    "path": ["WETH", "UnknownToken"],
    "deadline": "9999999999"
  },
  "expected_destination": "Uniswap V3 Router"
}
```

**预期输出：**
```json
{
  "summary": "在未知合约上执行 ETH→UnknownToken 的 swap，非预期的 Uniswap 路由",
  "risk_level": "high",
  "risk_factors": [
    "目标合约非预期的 Uniswap Router",
    "amountOutMin 设为 0（无滑点保护）",
    "deadline 异常长（无时间保护）",
    "交易对包含 UnknownToken（非主流代币）"
  ],
  "uncertainties": ["UnknownToken 是否为 honeypot"],
  "requires_human_approval": true
}
```

**为什么是 high**：「你想去 Uniswap，但签的交易指向了别的合约」— 这是典型的 front-end hijack / 钓鱼签名。

---

## Prompt 设计要点总结

| 技巧 | 说明 |
|------|------|
| **角色设定** | "你是一个 Web3 交易安全分析器" — 给模型明确身份 |
| **结构化输出** | JSON schema 写死，减少格式漂移 |
| **规则先行** | 风险判定规则写在 prompt 里，不是让模型自己发挥 |
| **不确定性显式化** | uncertainties 字段强制模型承认"我不知道" |
| **人机协作** | requires_human_approval 定义何时必须人类介入 |

## 这个 prompt 的局限

1. **需要结构化输入**：LLM 不能自己去查链，需要先有 JSON 格式的交易数据
2. **规则覆盖不全**：只覆盖了最基础的 4 种情况，真实世界的攻击模式远多于这些
3. **没有地址标签**：不知道地址是 CEX / DEX / 个人钱包，这需要外部数据

> **这是 LLM 在 Agent 中的正确用法**：工具负责取数据，LLM 负责理解和解释。不是让 LLM 做所有事。
