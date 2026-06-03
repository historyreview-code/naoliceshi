# 🛕 AI Temple — Stripe Payment Integration

AI Agent 可调用的 Stripe 支付服务。适合小额自动化收款场景。

## 目录

- [快速开始](#快速开始)
- [API 接口](#api-接口)
- [AI Agent 集成](#ai-agent-集成)
- [Stripe Webhook](#stripe-webhook-可选)
- [项目结构](#项目结构)
- [完整演示](#完整演示)

---

## 快速开始

### 1. 获取 Stripe API 密钥

1. 注册 [Stripe 账户](https://stripe.com)
2. 进入 Dashboard → **Developers → API keys**
3. 复制 `Secret key`（以 `sk_test_` 开头的是测试密钥）

### 2. 配置环境变量

```bash
cp .env.example .env
# 编辑 .env，填入你的 STRIPE_SECRET_KEY
```

### 3. 启动服务

```bash
npm start
# 或开发模式（文件变动自动重启）
npm run dev
```

服务运行在 `http://localhost:3000`

---

## API 接口

### 创建付款链接 — `POST /api/create-payment-link`

适合 AI Agent 调用，生成一个链接发给用户即可收款。

```json
// 请求
{ "amount": 5.00, "description": "AI 分析服务费" }

// 响应
{ "url": "https://buy.stripe.com/test_xxxx", "id": "plink_xxxx" }
```

### 创建结账页面 — `POST /api/create-checkout-session`

适合网页跳转，Stripe 托管完整结账流程。

### 查询支付状态 — `GET /api/payment-status/:sessionId`

确认用户是否已付款。

---

## AI Agent 集成

### OpenAI Function Calling

`src/openai-integration.js` 提供了标准的 OpenAI tools 格式定义，可以直接传给 Chat Completions API：

```javascript
// 在你的 AI Agent 代码中
import OpenAI from 'openai';
import { getOpenAITools, handleToolCall } from './src/openai-integration';

const openai = new OpenAI();

// 1. 把工具定义传给模型
const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: '你是 AI Temple 助手，可以生成付款链接来收取服务费用。' },
    { role: 'user', content: '我想做一个网站分析，帮我看看需要多少钱' },
  ],
  tools: getOpenAITools(),    // ← 注入支付工具
});

// 2. 处理模型发起的工具调用
const toolCall = response.choices[0]?.message?.tool_calls?.[0];
if (toolCall) {
  const args = JSON.parse(toolCall.function.arguments);
  const result = await handleToolCall(toolCall.function.name, args);
  // result 包含了付款链接，让 AI 发给用户
  console.log(result);
}
```

### 工具清单

| 工具名 | 触发方式 | 功能 |
|---|---|---|
| `generate_payment_link` | AI 自主调用 | 生成 Stripe 付款链接，发给用户 |
| `check_payment` | AI 自主调用 | 查询某笔支付的状态 |

### AI 定价流程

```
用户 → "我需要一份数据分析报告"
  │
AI Agent → "分析报告费用为 $5.00，正在生成付款链接..."
  │
AI Agent → 调用 generate_payment_link($5.00)
  │           ↓
  │         Stripe → 返回付款链接
  │
AI Agent → "请点击链接完成支付：pay.stripe.com/..."
  │
用户 → 点击链接 → 在 Stripe 页面付款
  │
Stripe Webhook → 通知 AI Temple "支付成功"
  │
AI Agent → 调用 check_payment(session_id) 确认
  │
AI Agent → "支付已确认！这是你的分析报告：[数据]"
```

---

## Stripe Webhook（可选）

用于自动接收支付结果通知，无需手动轮询。

1. Stripe Dashboard → **Developers → Webhooks** → **Add endpoint**
2. URL: `https://your-domain.com/webhook`
3. 选择事件：`checkout.session.completed`
4. 复制 `Signing secret`（`whsec_...`）填入 `.env` 的 `STRIPE_WEBHOOK_SECRET`

---

## 项目结构

```
ai-temple/
├── src/
│   ├── server.js              # Express HTTP 服务 + API 路由
│   ├── stripe.js               # Stripe SDK 封装核心
│   ├── agent-tools.js          # 通用 AI Agent 工具层
│   └── openai-integration.js   # OpenAI Function Calling 集成
├── demo.js                     # 演示脚本
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

---

## 完整演示

```bash
# 1. 先配置好 .env
cp .env.example .env

# 2. 运行演示脚本（无需启动服务器）
node demo.js

# 3. 或启动完整 HTTP 服务
npm start
```
