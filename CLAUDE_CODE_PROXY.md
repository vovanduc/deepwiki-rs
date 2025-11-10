# Claude Code Proxy Integration for Litho

> **"Hack" to use Claude Code subscription with Litho instead of paying for separate LLM API**

## 🎯 Problem Statement

Litho requires LLM API access (OpenAI, Anthropic, DeepSeek, etc.) which costs money. If you have a **Claude Code subscription**, you can leverage it to run Litho **without additional API costs**.

## 💡 Solution: MCP Proxy Server

Create a lightweight proxy server that sits between Litho and Claude Code:

```
┌──────────┐         HTTP API          ┌──────────────┐       Agent SDK        ┌──────────────┐
│  Litho   │ ────────────────────────> │ MCP Proxy    │ ────────────────────> │ Claude Code  │
│ (Rust)   │  Anthropic API format     │ (TypeScript) │  Uses subscription    │ Subscription │
└──────────┘                            └──────────────┘                        └──────────────┘
```

### Why This Works

- **Litho** expects Anthropic API-compatible endpoint
- **Proxy** translates between Anthropic API format ↔ Agent SDK
- **Agent SDK** uses Claude Code subscription (no API key needed!)
- **Result**: Litho works with your subscription instead of API credits

---

## 🏗️ Architecture Details

### Components

1. **Litho (Unchanged)**
   - Configuration points to `localhost:3000` instead of `api.anthropic.com`
   - Sends standard Anthropic API requests
   - No code modifications needed!

2. **MCP Proxy Server (New)**
   - Node.js/TypeScript application (~150 lines)
   - Exposes HTTP endpoint: `POST /v1/messages`
   - Uses `@anthropic-ai/claude-agent-sdk`
   - Translates request/response formats

3. **Claude Code (Existing)**
   - Your subscription provides the LLM access
   - Agent SDK automatically uses subscription
   - Subject to usage limits (40-80 hours/week for Pro)

---

## 📋 Implementation Guide

### Prerequisites

- **Node.js** 18+ with npm
- **Claude Code subscription** (Pro recommended for higher limits)
- **Litho** built and ready (`cargo build --release`)
- **Git** configured

### Step 1: Create Proxy Server Project

```bash
# Create project directory
mkdir litho-claude-code-proxy
cd litho-claude-code-proxy

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express @anthropic-ai/claude-agent-sdk
npm install --save-dev typescript @types/express @types/node tsx

# Initialize TypeScript
npx tsc --init
```

### Step 2: Create Proxy Server Code

Create `src/server.ts`:

```typescript
import express, { Request, Response } from 'express';
import { Agent } from '@anthropic-ai/claude-agent-sdk';

const app = express();
app.use(express.json({ limit: '50mb' }));

// Health check endpoint
app.get('/health', (req: Request, res: Response) => {
  res.json({
    status: 'ok',
    service: 'litho-claude-code-proxy',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

// Main endpoint - Anthropic API compatible
app.post('/v1/messages', async (req: Request, res: Response) => {
  const startTime = Date.now();

  try {
    const { messages, model, max_tokens, temperature, system } = req.body;

    console.log(`[${new Date().toISOString()}] REQUEST`);
    console.log(`  Model: ${model}`);
    console.log(`  Messages: ${messages?.length || 0}`);
    console.log(`  Max Tokens: ${max_tokens || 'default'}`);

    // Extract system prompt and user messages
    let systemPrompt = system || '';
    let conversationHistory: string[] = [];

    // Parse messages array (Anthropic API format)
    for (const msg of messages || []) {
      if (msg.role === 'system') {
        systemPrompt = msg.content;
      } else if (msg.role === 'user') {
        const content = Array.isArray(msg.content)
          ? msg.content.map(c => typeof c === 'string' ? c : c.text || '').join('\n')
          : msg.content;
        conversationHistory.push(`User: ${content}`);
      } else if (msg.role === 'assistant') {
        const content = Array.isArray(msg.content)
          ? msg.content.map(c => typeof c === 'string' ? c : c.text || '').join('\n')
          : msg.content;
        conversationHistory.push(`Assistant: ${content}`);
      }
    }

    // Combine into single prompt for Agent SDK
    const fullPrompt = conversationHistory.join('\n\n');

    console.log(`  System: ${systemPrompt.substring(0, 100)}...`);
    console.log(`  Prompt length: ${fullPrompt.length} chars`);

    // Initialize Claude Agent (uses subscription automatically)
    const agent = new Agent({
      systemPrompt: systemPrompt,
      maxTokens: max_tokens || 8192,
      temperature: temperature !== undefined ? temperature : 0.1
    });

    // Execute prompt via Agent SDK
    console.log(`[${new Date().toISOString()}] Calling Agent SDK...`);
    const response = await agent.prompt(fullPrompt);
    console.log(`[${new Date().toISOString()}] Agent SDK response received`);

    const duration = Date.now() - startTime;

    // Estimate token usage (rough approximation: 1 token ≈ 4 chars)
    const inputTokens = Math.ceil((systemPrompt.length + fullPrompt.length) / 4);
    const outputTokens = Math.ceil(response.length / 4);

    console.log(`  Response length: ${response.length} chars`);
    console.log(`  Est. tokens: ${inputTokens} in, ${outputTokens} out`);
    console.log(`  Duration: ${duration}ms`);

    // Return in Anthropic API format
    res.json({
      id: `msg_proxy_${Date.now()}`,
      type: 'message',
      role: 'assistant',
      content: [{
        type: 'text',
        text: response
      }],
      model: model || 'claude-sonnet-4-5-20250929',
      stop_reason: 'end_turn',
      stop_sequence: null,
      usage: {
        input_tokens: inputTokens,
        output_tokens: outputTokens
      }
    });

    console.log(`[${new Date().toISOString()}] SUCCESS (${duration}ms)\n`);

  } catch (error: any) {
    const duration = Date.now() - startTime;
    console.error(`[${new Date().toISOString()}] ERROR (${duration}ms):`, error.message);
    console.error(error.stack);

    res.status(500).json({
      type: 'error',
      error: {
        type: 'api_error',
        message: error.message || 'Internal server error'
      }
    });
  }
});

// 404 handler
app.use((req: Request, res: Response) => {
  res.status(404).json({
    type: 'error',
    error: {
      type: 'not_found',
      message: `Endpoint not found: ${req.method} ${req.path}`
    }
  });
});

const PORT = process.env.PORT || 3000;
const HOST = process.env.HOST || 'localhost';

app.listen(PORT, HOST as string, () => {
  console.log('╔════════════════════════════════════════════════════════════╗');
  console.log('║  Litho → Claude Code Proxy Server                          ║');
  console.log('╠════════════════════════════════════════════════════════════╣');
  console.log(`║  HTTP Server: http://${HOST}:${PORT}                     ║`);
  console.log(`║  Health Check: http://${HOST}:${PORT}/health            ║`);
  console.log(`║  Endpoint: POST /v1/messages                               ║`);
  console.log('╠════════════════════════════════════════════════════════════╣');
  console.log('║  ✓ Using Claude Code subscription (no API key required)   ║');
  console.log('║  ✓ Compatible with Anthropic API format                   ║');
  console.log('║  ✓ Ready to accept requests from Litho                    ║');
  console.log('╚════════════════════════════════════════════════════════════╝\n');
});
```

### Step 3: Update `package.json`

```json
{
  "name": "litho-claude-code-proxy",
  "version": "1.0.0",
  "description": "MCP proxy server to use Claude Code subscription with Litho",
  "main": "dist/server.js",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "curl http://localhost:3000/health"
  },
  "keywords": ["litho", "claude", "proxy", "mcp"],
  "author": "Duc Vo",
  "license": "MIT"
}
```

### Step 4: Configure TypeScript

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Step 5: Create `.gitignore`

```gitignore
node_modules/
dist/
.env
*.log
.DS_Store
```

---

## 🚀 Usage

### Start Proxy Server

```bash
cd litho-claude-code-proxy
npm run dev
```

You should see:
```
╔════════════════════════════════════════════════════════════╗
║  Litho → Claude Code Proxy Server                          ║
╠════════════════════════════════════════════════════════════╣
║  HTTP Server: http://localhost:3000                        ║
║  Health Check: http://localhost:3000/health                ║
║  Endpoint: POST /v1/messages                               ║
╠════════════════════════════════════════════════════════════╣
║  ✓ Using Claude Code subscription (no API key required)   ║
║  ✓ Compatible with Anthropic API format                   ║
║  ✓ Ready to accept requests from Litho                    ║
╚════════════════════════════════════════════════════════════╝
```

### Configure Litho

Create or edit `litho.toml` in your Litho project:

```toml
[llm]
provider = "anthropic"
api_base_url = "http://localhost:3000"
api_key = "dummy-not-used"  # Agent SDK doesn't need API key
model_efficient = "claude-sonnet-4-5-20250929"
model_powerful = "claude-sonnet-4-5-20250929"
max_tokens = 8192
temperature = 0.1
max_parallels = 3

[cache]
enabled = true
```

### Run Litho

Open a **new terminal** (keep proxy running in first terminal):

```bash
cd /path/to/deepwiki-rs

# Run Litho on your project
./target/release/deepwiki-rs -p ./src -o ./docs

# Or use cargo run
cargo run --release -- -p ./src
```

### Monitor Proxy Logs

Watch the proxy terminal for request logs:

```
[2025-01-10T10:30:15.123Z] REQUEST
  Model: claude-sonnet-4-5-20250929
  Messages: 2
  Max Tokens: 8192
  System: You are an expert software architect...
  Prompt length: 15234 chars
[2025-01-10T10:30:15.234Z] Calling Agent SDK...
[2025-01-10T10:30:22.456Z] Agent SDK response received
  Response length: 3456 chars
  Est. tokens: 3808 in, 864 out
  Duration: 7333ms
[2025-01-10T10:30:22.456Z] SUCCESS (7333ms)
```

---

## ✅ Testing

### Test 1: Health Check

```bash
curl http://localhost:3000/health
```

Expected:
```json
{
  "status": "ok",
  "service": "litho-claude-code-proxy",
  "version": "1.0.0",
  "timestamp": "2025-01-10T10:30:00.000Z"
}
```

### Test 2: Simple Message

```bash
curl -X POST http://localhost:3000/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5-20250929",
    "messages": [
      {"role": "user", "content": "What is 2+2? Answer in one word."}
    ],
    "max_tokens": 100
  }'
```

Expected:
```json
{
  "id": "msg_proxy_1704880200000",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "Four"}],
  "model": "claude-sonnet-4-5-20250929",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 15, "output_tokens": 1}
}
```

### Test 3: Run Litho on Small Project

```bash
# Test on Litho's own source code (small sample)
./target/release/deepwiki-rs -p ./src/cli.rs -o ./test-output
```

---

## ⚠️ Important Limitations

### 1. Claude Code Usage Limits

- **Pro Plan**: ~40-80 hours of Sonnet usage per week
- **Usage resets**: Every 5 hours with ~10-40 prompts per cycle
- **Large codebases**: Can consume quota quickly

**Monitor usage**:
- Check Claude Code app for usage stats
- Proxy logs show token estimates
- Consider caching (`cache.enabled = true` in `litho.toml`)

### 2. Authentication

**CRITICAL**: Do NOT set `ANTHROPIC_API_KEY` environment variable!

If `ANTHROPIC_API_KEY` is set, Agent SDK will use API credits instead of subscription.

```bash
# ❌ WRONG - This uses API credits
export ANTHROPIC_API_KEY="sk-ant-xxx"

# ✅ CORRECT - No environment variable
# Agent SDK automatically uses Claude Code subscription
```

### 3. Tool Calling / ReAct Mode

Current implementation **does not support** tool calling (ReAct mode).

To disable ReAct mode in Litho:

```bash
./target/release/deepwiki-rs -p ./src --disable-preset-tools
```

Or in `litho.toml`:

```toml
[llm]
disable_preset_tools = true
```

**Note**: Tool calling support can be added to the proxy but requires additional implementation (~200 more lines).

### 4. Performance

- **Network overhead**: Localhost communication adds ~10-50ms latency
- **Agent SDK overhead**: Additional ~100-200ms vs direct API
- **Acceptable for**: Documentation generation (not real-time apps)

---

## 🐛 Troubleshooting

### Issue 1: "Agent SDK not found"

```
Error: Cannot find module '@anthropic-ai/claude-agent-sdk'
```

**Solution**:
```bash
cd litho-claude-code-proxy
npm install @anthropic-ai/claude-agent-sdk
```

### Issue 2: "Connection refused"

```
Error: connect ECONNREFUSED 127.0.0.1:3000
```

**Solution**: Proxy server not running. Start it:
```bash
cd litho-claude-code-proxy
npm run dev
```

### Issue 3: "API error" from proxy

Check proxy logs for details. Common causes:
- Claude Code not logged in
- Subscription expired or limits exceeded
- Network connectivity issues

**Solution**: Restart Claude Code app or check subscription status.

### Issue 4: Slow responses

Agent SDK can be slower than direct API.

**Solutions**:
- Enable caching in `litho.toml`
- Reduce `max_tokens` to 4096 or less
- Use `--skip-preprocessing` if re-running

### Issue 5: High token usage

**Solutions**:
- Use smaller projects for testing
- Enable aggressive caching
- Skip stages: `--skip-preprocessing --skip-research`

---

## 📊 Cost Comparison

### Without Proxy (Traditional API)

| Project Size | Provider | Estimated Cost |
|--------------|----------|----------------|
| Small (100 files) | OpenAI | ~$0.50 |
| Small (100 files) | Gemini | ~$0.10 |
| Medium (1K files) | OpenAI | ~$2.00 |
| Medium (1K files) | DeepSeek | ~$0.30 |
| Large (10K files) | OpenAI | ~$10.00 |
| Large (10K files) | Gemini | ~$1.50 |

### With Proxy (Claude Code Subscription)

| Project Size | Cost |
|--------------|------|
| **Any size** | **$0** (uses subscription) |

**BUT**: Subject to usage limits (40-80 hours/week)

---

## 🔄 Future Enhancements

### Planned Features

1. **Tool Calling Support**
   - Enable ReAct mode
   - Map Litho tools to Agent SDK

2. **Request Caching**
   - Cache responses in proxy
   - Reduce subscription usage

3. **Usage Monitoring**
   - Track tokens consumed
   - Estimate remaining quota
   - Alert on limits

4. **Multi-Model Support**
   - Fallback to Ollama when limits exceeded
   - Load balancing between multiple subscriptions

5. **Docker Support**
   - Containerize proxy server
   - Easy deployment

---

## 🤝 Contributing

Found a bug or have an enhancement? Issues and PRs welcome!

### Development Setup

```bash
# Clone fork
git clone https://github.com/vovanduc/deepwiki-rs.git
cd deepwiki-rs

# Create proxy project
mkdir ../litho-claude-code-proxy
cd ../litho-claude-code-proxy

# Follow "Implementation Guide" above
```

---

## 📄 License

This proxy integration maintains the same license as Litho (MIT).

---

## 🙏 Credits

- **Litho (deepwiki-rs)**: [sopaco/deepwiki-rs](https://github.com/sopaco/deepwiki-rs)
- **Anthropic Agent SDK**: [@anthropic-ai/claude-agent-sdk](https://github.com/anthropics/anthropic-sdk-typescript)
- **Integration idea**: Community-driven "hack" to leverage existing subscriptions

---

## ❓ FAQ

### Q: Is this officially supported by Anthropic?

**A**: No, this is a community hack. Use at your own risk. Agent SDK is official but this specific use case is not documented by Anthropic.

### Q: Will this violate terms of service?

**A**: Agent SDK is designed for subscription use. However, review [Claude Code Terms](https://claude.ai/terms) to ensure compliance with your specific use case.

### Q: Can I use this in production?

**A**: Not recommended. This is best for:
- Personal projects
- Development/testing
- Small-scale documentation generation

For production, use official APIs with proper rate limits and SLAs.

### Q: What if I hit usage limits?

**A**: Proxy will return errors. Configure Litho with fallback:
1. Try proxy first
2. If fails, fallback to Ollama (free local)
3. Or wait for usage quota to reset (every 5 hours)

### Q: Can I run multiple instances?

**A**: Yes, but they share the same Claude Code subscription quota. Better to:
- Use caching aggressively
- Queue requests sequentially
- Monitor usage carefully

---

**🎉 Happy hacking! Use your Claude Code subscription to power Litho!**
