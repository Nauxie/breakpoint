# War Stories: Building Resilient AI Agents in Production
## Breakpoint - November 21, 2025

**Abhinav Tumu**  
Founding Engineer, Brainbase (YC W24)

---

## Slide 1: Title Slide

**War Stories: Building Resilient AI Agents in Production**

*What Actually Breaks When You Run Agents 24/7*

Abhinav Tumu | Brainbase | @abhinavtumu

---

## Slide 2: The $10,000 Morning

**6:47 AM - September 2024**

```
ERROR: Rate limit exceeded (429)
ERROR: Rate limit exceeded (429)
ERROR: Rate limit exceeded (429)
... 3,247 more errors
```

**What happened:**
- Primary LLM provider went down
- No fallback configured
- 200+ enterprise agents offline
- Customers woke up to failed workflows
- **Cost**: Support tickets, refunds, credibility

**Lesson**: Single points of failure are expensive.

---

## Slide 3: About Kafka

**The Context:**
- Conversational AI agents for enterprise workflows
- 24/7 autonomous operation
- Multi-tenant (1000+ concurrent users)
- Long-running tasks (hours to days)
- Every failure is customer-visible

**Scale:**
- ~500K LLM requests/day
- ~2M tool executions/day
- $50K+/month in infrastructure
- 99.5% uptime SLA

---

# Part 1: Multi-Provider Architecture
## Never Trust One Provider

---

## Slide 4: The Problem with Single Providers

**Reality Check:**
- Provider outages happen weekly
- Rate limits hit during peak hours
- Model availability varies by region
- Pricing changes without notice

**Our Requirements:**
- No single point of failure
- Automatic failover (<1s)
- Cost optimization
- Model flexibility

---

## Slide 5: The Gateway Pattern

```
User Request
    ↓
Gateway Config (per-user)
    ↓
Level 1: Provider A + Model X (primary)
    ↓ [429, 500-504 errors]
Level 2: Provider B + Model X (cross-provider fallback)
    ↓ [same errors]
Level 3: Provider A + Model Y (model downgrade)
    ↓ [same errors]
Level 4: Provider B + Model Y (last resort)
```

**Key Insight**: Need BOTH provider AND model fallbacks

---

## Slide 6: Architecture - Virtual Keys

```python
# Every user gets isolated credentials
user_config = {
    "anthropic_virtual_key": "vk_anth_user123",
    "bedrock_virtual_key": "vk_bed_user123",
    "openai_virtual_key": "vk_oai_user123"
}

# Gateway config references virtual keys
gateway_config = {
    "strategy": "fallback",
    "on_status_codes": [400, 429, 500, 502, 503, 504],
    "targets": [
        {"virtual_key": "vk_anth_user123", "model": "claude-4"},
        {"virtual_key": "vk_bed_user123", "model": "claude-4"},
        {"virtual_key": "vk_anth_user123", "model": "claude-3.7"},
        {"virtual_key": "vk_bed_user123", "model": "claude-3.7"}
    ]
}
```

**Benefits:**
- No API keys in code
- Per-user rate limits
- Easy key rotation
- Usage attribution

---

## Slide 7: Load Balancing + Fallbacks

```python
config = {
    "output_guardrails": ["content-filter-1"],
    "strategy": {"mode": "loadbalance"},  # Top level
    "targets": [
        {
            "weight": 0.5,  # 50% traffic
            "strategy": {"mode": "fallback"},  # Nested
            "targets": [
                {"provider": "anthropic", "model": "claude-4"},
                {"provider": "anthropic", "model": "claude-3.7"}
            ]
        },
        {
            "weight": 0.5,  # 50% traffic
            "strategy": {"mode": "fallback"},
            "targets": [
                {"provider": "bedrock", "model": "claude-4"},
                {"provider": "bedrock", "model": "claude-3.7"}
            ]
        }
    ]
}
```

**Result**: Traffic split + automatic failover

---

## Slide 8: Production Lessons - Providers

**What We Learned:**

1. **Different providers, different behaviors**
   - Token counting varies (cache tokens included or not?)
   - Error codes inconsistent
   - Latency profiles different

2. **Regional matters**
   - `us.anthropic.*` namespace for Bedrock
   - Cross-region redundancy critical
   - Latency vs reliability tradeoff

3. **Graceful degradation**
   - Model downgrade better than failure
   - Inform users about fallbacks
   - Track fallback frequency for alerting

**War Story**: Initially only had 2-level fallback → still saw 0.3% failure rate. 4-level fallback → 0.01%.

---

# Part 2: Sandboxed Execution
## Write Code, Not Tool Schemas

---

## Slide 9: The Tool Schema Problem

**Traditional Approach:**
```python
tools = [
    {"name": "web_search", "params": {...}, "description": "..."},
    {"name": "scrape_website", "params": {...}, "description": "..."},
    {"name": "send_email", "params": {...}, "description": "..."},
    # ... 60+ more tools
]
```

**Problems:**
- 20K+ tokens just for tool definitions
- Rigid schemas, hard to evolve
- LLM confused by too many options
- Every new capability = new tool

**Cost**: ~$0.08 per request just for tool definitions

---

## Slide 10: The Notebook MCP Pattern

**New Approach: Write & Execute**

```python
# Single tool, unlimited capabilities
@mcp.tool()
async def notebook_run_cell(
    code: str,           # Python code to execute
    description: str,    # What is this doing?
    timeout: int = 3600  # Max execution time
) -> str:
    """Execute Python in sandboxed Jupyter environment"""
    
    # Execute in persistent IPython shell
    result = shell.run_cell_async(code)
    
    # Capture stdout, stderr, exceptions
    return output
```

**Instead of**: 60+ pre-defined tools  
**Now**: 1 tool, agent writes code

---

## Slide 11: Why This Works

**Example - Multi-step Research:**

```python
# Step 1: Agent writes code to scrape
await notebook_run_cell(
    code="""
import requests
from bs4 import BeautifulSoup

response = requests.get('https://competitor.com/pricing')
soup = BeautifulSoup(response.text, 'html.parser')
prices = soup.find_all('div', class_='price')
data = [p.text for p in prices]
""",
    description="Scraping competitor pricing"
)

# Step 2: Analyze (state persists!)
await notebook_run_cell(
    code="""
import pandas as pd
df = pd.DataFrame({'price': data})
df['price_clean'] = df['price'].str.extract(r'(\d+\.?\d*)')[0].astype(float)
summary = df.describe()
print(summary)
""",
    description="Analyzing pricing data"
)
```

**Key**: Variables persist across cells, just like Jupyter

---

## Slide 12: Sandboxing Architecture

```
┌─────────────────────────────────────┐
│   Each Agent Thread                 │
├─────────────────────────────────────┤
│                                     │
│  ┌───────────────────────────┐     │
│  │ Dedicated Firecracker VM  │     │
│  ├───────────────────────────┤     │
│  │ - Isolated filesystem     │     │
│  │ - User-specific creds     │     │
│  │ - Redis ACL namespace     │     │
│  │ - Resource limits         │     │
│  │ - Network proxy           │     │
│  └───────────────────────────┘     │
│                                     │
│  Health Checks: 30s intervals       │
│  Auto-recovery: on failure          │
│  Timeout: 2 hours idle              │
└─────────────────────────────────────┘
```

**Isolation Layers:**
1. OS-level (VM)
2. Filesystem (namespace)
3. Network (proxy)
4. Data (Redis ACL)

---

## Slide 13: Redis Multi-tenancy

```python
# Each user gets their own Redis namespace with ACL
def provision_user(user_id):
    username = f"{user_id}"
    
    # Generate cryptographically secure token
    token = execute_command('ACL', 'GENTOKEN', username)
    
    # Create user with restricted access
    execute_command(
        'ACL', 'SETUSER', username,
        'ON',                          # Enable user
        f'>{token}',                   # Set password
        f'~user:{user_id}:*',         # Namespace pattern
        f'~queue:{user_id}:*',
        f'~data:{user_id}:*',
        '+brpop', '+rpush', '+get',    # Whitelist commands
        '-@dangerous',                 # Block dangerous
        '-flushdb', '-flushall'        # Block destructive
    )
    
    return {'username': username, 'password': token}
```

**Result**: User A cannot access User B's data, even with credentials

---

## Slide 14: The Race Condition Story

**The Bug:**
```
User sends message
    ↓
API receives 3 concurrent requests (browser, mobile, retry)
    ↓
All 3 check: "Does VM exist?" → No
    ↓
All 3 create VMs simultaneously
    ↓
💥 3 VMs for one thread, $$ wasted, crashes
```

**The Fix: Atomic Operations**
```python
async def atomic_create_sandbox(thread_id, current_status):
    # Database-level CAS operation
    updated = await db.atomic_update_thread_status(
        thread_id,
        expected_status="off",
        new_status="starting"
    )
    
    if updated:
        # This request won the race
        return await create_vm()
    else:
        # Another request is creating, wait for it
        await asyncio.sleep(1)
        return await get_existing_vm()
```

**Lesson**: Distributed systems need atomic primitives

---

## Slide 15: Production Metrics - Sandboxing

**Results After 6 Months:**

| Metric | Before | After |
|--------|--------|-------|
| Token usage (tool defs) | 22K tokens | 0.8K tokens (-96%) |
| Tool execution flexibility | 60 tools | Unlimited |
| Multi-tenant isolation incidents | 3/month | 0/month |
| VM creation race conditions | ~20/day | 0/day |
| Average sandbox startup time | 8.3s | 4.1s |

**Cost Impact**: Saved $12K/month on prompt tokens alone

---

# Part 3: Cost Optimization
## Caching at Scale

---

## Slide 16: The Caching Opportunity

**Problem:**
- Long agent conversations (50+ messages)
- System prompt repeated every request
- Tool definitions repeated
- Recent tool results referenced repeatedly

**Anthropic Claude Sonnet 4 Pricing:**
- Regular input: $3.00/M tokens
- Cache write (5min): $3.75/M tokens (+25%)
- Cache read: $0.30/M tokens (**-90%**)

**Break-even: 2 requests**

---

## Slide 17: Progressive Caching Strategy

**Anthropic allows 4 cache breakpoints:**

```python
# Priority order for cache slots:
Cache Slot 1: Tools array (rarely changes, ~15K tokens)
    ↓
Cache Slot 2: System prompt (~3K tokens)
    ↓
Cache Slot 3: Large user content (>10K tokens)
    ↓
Cache Slot 4: Most recent tool results
```

**Why this order?**
- Largest first (max savings)
- Least changing first (max hit rate)
- Most referenced first (max value)

---

## Slide 18: Implementation

```python
def apply_cache_controls(messages):
    candidates = []
    
    # Identify cacheable content
    for msg in messages:
        if msg['role'] == 'system':
            candidates.append({'type': 'system', 'tokens': 3000, ...})
        elif msg['role'] == 'user' and count_tokens(msg) > 10000:
            candidates.append({'type': 'large_user', 'tokens': 12000, ...})
        elif msg['role'] == 'tool':
            candidates.append({'type': 'tool', 'tokens': 5000, ...})
    
    # Sort: system first, then large user, then recent tools
    keep_items = []
    keep_items.extend([c for c in candidates if c['type'] == 'system'])
    keep_items.extend([c for c in candidates if c['type'] == 'large_user'])
    keep_items.extend(sorted(
        [c for c in candidates if c['type'] == 'tool'],
        key=lambda x: x['position'],
        reverse=True
    )[:remaining_slots])
    
    # Add cache_control markers
    for item in keep_items:
        item['cache_control'] = {'type': 'ephemeral'}
```

---

## Slide 19: Cache Performance - Real Numbers

**Case Study: 100-message conversation**

```
Without Caching:
- Request 1: 25K tokens × $3.00 = $0.075
- Request 2: 30K tokens × $3.00 = $0.090
- Request 3: 35K tokens × $3.00 = $0.105
Total: $0.270

With Progressive Caching:
- Request 1: 25K tokens × $3.00 = $0.075 (cache write)
- Request 2: 5K new + 20K cached = $0.015 + $0.006 = $0.021
- Request 3: 5K new + 25K cached = $0.015 + $0.0075 = $0.0225
Total: $0.1185

Savings: 56%
```

**Over 1 month:**
- 500K conversations
- Average 20 messages each
- Estimated savings: **$18,000/month**

---

## Slide 20: The Cache Miss Mystery

**War Story:**

Expected cache hit rate: 85%+  
Actual cache hit rate: 23%

**Investigation:**
```python
# Tool results included timestamps
{
    "tool": "web_search",
    "result": "Found 10 results",
    "timestamp": "2024-09-15T14:32:17.293Z",  # ❌ Changes every time!
    "execution_time_ms": 1247  # ❌ Also changes!
}
```

**Fix: Canonicalize cached content**
```python
def prepare_for_cache(tool_result):
    # Remove non-deterministic fields
    cleaned = {k: v for k, v in tool_result.items() 
               if k not in ['timestamp', 'execution_time_ms', 'request_id']}
    
    # Sort keys for consistency
    return json.dumps(cleaned, sort_keys=True)
```

**Result**: Cache hit rate → 82%

---

## Slide 21: Provider Differences - Token Counting

**The Gotcha:**

```python
# Anthropic Direct:
{
    "usage": {
        "prompt_tokens": 5000,              # Non-cached only
        "completion_tokens": 1000,
        "cache_read_input_tokens": 15000,   # Separate!
        "cache_creation_input_tokens": 0
    }
}

# Bedrock (Anthropic via AWS):
{
    "usage": {
        "prompt_tokens": 20000,              # INCLUDES cache tokens!
        "completion_tokens": 1000,
        "cache_read_input_tokens": 15000,
        "cache_creation_input_tokens": 0
    }
}
```

**Fix: Detect format**
```python
if prompt_tokens >= (cache_read + cache_creation):
    # Format 2: prompt includes cache
    non_cached = prompt_tokens - cache_read - cache_creation
else:
    # Format 1: prompt is separate
    non_cached = prompt_tokens
```

---

# Part 4: Streaming & Resilience
## Failing Gracefully

---

## Slide 22: Streaming Challenges

**Why Streaming is Hard:**
- Less stable than non-streaming APIs
- Can't easily retry (partial output sent)
- Tool calls interrupt streams
- Error recovery is complex

**Why We Need It:**
- Users expect real-time responses
- Better perceived performance
- Enables interruption mid-generation

**Goal**: Get streaming benefits, non-streaming reliability

---

## Slide 23: Graceful Degradation Pattern

```python
# Track tools that fail during streaming
streaming_failed_tools = set()

async def run_agent():
    max_attempts = 3
    
    for attempt in range(max_attempts):
        try:
            # Check if we should disable streaming
            use_streaming = "__overloaded__" not in streaming_failed_tools
            
            if use_streaming:
                stream = await client.create(..., stream=True)
                async for chunk in stream:
                    yield chunk
            else:
                # Fallback to non-streaming
                response = await client.create(..., stream=False)
                return response
                
        except StreamingOverloadedError as e:
            # Mark streaming as failed
            streaming_failed_tools.add("__overloaded__")
            
            # Notify user once
            await broadcast({
                "type": "streaming_recovered",
                "message": "Switching to non-streaming mode"
            })
            
            # Retry with exponential backoff
            if attempt < max_attempts - 1:
                await asyncio.sleep(2 ** attempt)
                continue
```

---

## Slide 24: Circuit Breaker for Tools

```python
class ToolCircuitBreaker:
    def __init__(self, failure_threshold=3, reset_timeout=300):
        self.failure_counts = {}
        self.blocked_tools = set()
        self.reset_timeout = reset_timeout
    
    def record_failure(self, tool_name: str):
        self.failure_counts[tool_name] = \
            self.failure_counts.get(tool_name, 0) + 1
        
        if self.failure_counts[tool_name] >= 3:
            self.blocked_tools.add(tool_name)
            log.warning(f"Tool {tool_name} blocked for {reset_timeout}s")
    
    def is_blocked(self, tool_name: str) -> bool:
        if tool_name not in self.blocked_tools:
            return False
        
        # Auto-reset after timeout
        if time.time() - self.last_failure[tool_name] > self.reset_timeout:
            self.reset_tool(tool_name)
            return False
        
        return True
```

**Prevents**: Infinite loops of failing tool calls

---

## Slide 25: The Transparent Proxy

```
Agent Code → Proxy Layer → Upstream Service
                ↓
            Middleware:
            - Extract API key from request
            - Validate permissions
            - Track token usage
            - Calculate costs
            - Log metadata
            - Handle errors
            - Return response
```

**All Services Through Proxy:**
- LLM providers (OpenAI, Anthropic, Bedrock)
- Web crawler APIs
- Search APIs
- Email/notification services

**Benefits:**
- Unified observability
- Cost attribution
- Rate limiting
- Credential rotation

---

## Slide 26: Webhook-based Usage Tracking

```python
@webhook_app.post("/llm-webhook")
async def track_usage(webhook_request):
    # Extract metadata
    vm_api_key = webhook_request.metadata.get("VM_API_KEY")
    thread = await db.get_thread_by_api_key(vm_api_key)
    
    # Parse response for token counts
    usage = webhook_request.response.get("usage", {})
    request_tokens = usage.get("prompt_tokens", 0)
    response_tokens = usage.get("completion_tokens", 0)
    cache_read = usage.get("cache_read_input_tokens", 0)
    cache_creation = usage.get("cache_creation_input_tokens", 0)
    
    # Calculate costs
    cost = await credit_tracker.track_llm_usage(
        service='anthropic',
        model=model,
        request_tokens=request_tokens,
        response_tokens=response_tokens,
        cache_read_tokens=cache_read,
        cache_creation_tokens=cache_creation,
        thread_id=thread['id'],
        user_id=thread['user_id']
    )
    
    # Store in database for billing
    await db.insert_usage_record(...)
```

---

# Part 5: War Stories
## The Best Teacher is Failure

---

## Slide 27: Story 1 - The Infinite Loop

**What Happened:**
- Agent kept calling `web_search("competitor pricing")`
- Same query, same results, every time
- 400+ calls in 15 minutes
- **Cost**: $547 in LLM fees

**Root Cause:**
```python
# Agent logic loop:
1. Search for "competitor pricing"
2. Get results
3. Think "I need more specific info"
4. Search for "competitor pricing" again
5. GOTO 1
```

**The Fix:**
```python
# Deduplicate tool calls
recent_tool_calls = deque(maxlen=10)

def should_execute_tool(tool_name, tool_args):
    call_signature = f"{tool_name}:{hash(json.dumps(tool_args))}"
    
    if call_signature in recent_tool_calls:
        return False, "Duplicate tool call detected"
    
    recent_tool_calls.append(call_signature)
    return True, None
```

**Lesson**: AI doesn't naturally avoid redundancy

---

## Slide 28: Story 2 - The Zombie VMs

**What Happened:**
- VMs showed status "running"
- Health checks passed
- But agents weren't responding
- Users saw infinite loading

**Investigation:**
```python
# Our health check:
def check_vm_health(sandbox_id):
    sandbox = daytona.get(sandbox_id)
    return sandbox.state == "running"  # ❌ Too shallow!
```

**The Problem:**
- VM was running
- OS was running
- But application crashed
- Port not listening

**The Fix:**
```python
# Deep health check
async def check_vm_health(endpoint):
    try:
        response = await httpx.get(
            f"{endpoint}/health",
            timeout=10
        )
        return response.status_code == 200
    except Exception:
        return False

# With retries
for attempt in range(3):
    if await check_vm_health(endpoint):
        return True
    await asyncio.sleep(2 ** attempt)

# Failed all checks → trigger recovery
await reinitialize_vm(sandbox_id)
```

**Lesson**: "Running" ≠ "Working"

---

## Slide 29: Story 3 - The Rate Limit Cascade

**Timeline:**
```
14:23:00 - Primary provider hits rate limit
14:23:01 - 50 requests failover to backup provider
14:23:02 - Backup provider also hits rate limit
14:23:03 - ALL REQUESTS FAILING
14:23:04 - Error rate: 100%
14:23:05 - Alerts firing
```

**Root Cause:**
- All users shared same fallback logic
- No jitter in retry timing
- Stampede effect

**The Fix:**
```python
import random

async def retry_with_backoff(attempt):
    # Exponential backoff
    base_delay = 2 ** attempt
    
    # Add jitter (±25%)
    jitter = random.uniform(-0.25, 0.25) * base_delay
    
    # Wait
    await asyncio.sleep(base_delay + jitter)

# Per-provider circuit breaker
provider_circuit_breakers = {
    'anthropic': CircuitBreaker(threshold=10, timeout=60),
    'bedrock': CircuitBreaker(threshold=10, timeout=60)
}

def get_provider():
    if provider_circuit_breakers['anthropic'].is_open():
        return 'bedrock'
    if provider_circuit_breakers['bedrock'].is_open():
        return 'anthropic'
    return random.choice(['anthropic', 'bedrock'])  # Load balance
```

**Lesson**: Distributed systems need jitter

---

## Slide 30: Story 4 - The Partial Streaming Bug

**User Report:**
```
Agent: "I'll search for that information..."
Agent: "According to the search results, the company was founded in"
[Message ends abruptly]
```

**What Happened:**
- Streaming content to user
- LLM decides to call a tool
- Stream interrupted
- Tool executes
- Results processed
- But streaming already closed!

**The Fix:**
```python
async def handle_streaming():
    current_text = ""
    
    async for chunk in stream:
        delta = chunk.choices[0].delta
        
        # Handle content
        if delta.content:
            current_text += delta.content
            yield {"type": "content", "text": delta.content}
        
        # Handle tool calls
        if delta.tool_calls:
            # Send "thinking" signal
            yield {"type": "status", "status": "calling_tool"}
            
            # Accumulate full tool call
            # ... tool execution ...
            
            # Resume streaming after tool
            yield {"type": "status", "status": "generating"}
```

**Lesson**: Streaming + tool calling need careful orchestration

---

## Slide 31: Story 5 - The Cost Anomaly

**Slack Alert:**
```
🚨 COST ALERT: Daily LLM spend $3,247 
(expected: $800)
```

**Investigation:**
- One user's agent running wild
- Analyzing same document 1000+ times
- Why wasn't circuit breaker triggered?

**Root Cause:**
```python
# Agent's internal loop:
for page in document.pages:  # 500 pages
    analysis = await analyze_page(page)  # Each call successful!
    summaries.append(analysis)

# ❌ No failures, so no circuit breaker
# ✅ But extremely inefficient pattern
```

**The Fix:**
```python
# Add usage-based limits
class UsageLimiter:
    def __init__(self, max_tokens_per_hour=1_000_000):
        self.token_usage = {}
        self.max_tokens_per_hour = max_tokens_per_hour
    
    async def check_limit(self, user_id, estimated_tokens):
        hour_key = datetime.now().strftime("%Y-%m-%d-%H")
        usage = self.token_usage.get((user_id, hour_key), 0)
        
        if usage + estimated_tokens > self.max_tokens_per_hour:
            raise UsageLimitExceeded(
                f"Hour limit reached: {usage:,} tokens"
            )
        
        self.token_usage[(user_id, hour_key)] = usage + estimated_tokens
```

**Lesson**: Circuit breakers catch failures, not inefficiency

---

# Closing: Key Takeaways

---

## Slide 32: Core Principles

**1. Assume Everything Will Fail**
- Multi-provider fallbacks (4+ levels)
- Circuit breakers for repeated failures
- Health checks that actually check health
- Graceful degradation over hard failures

**2. Sandbox Religiously**
- OS-level isolation (VMs)
- Data isolation (namespaced Redis ACLs)
- Network isolation (proxies)
- Atomic operations for state changes

**3. Optimize for Cost Early**
- Progressive caching strategies
- Token usage tracking per user/thread
- Usage-based limits and alerts
- Regular cost analysis

---

## Slide 33: The Non-Obvious Lessons

**Race Conditions Are Expensive**
- Atomic operations at the database level
- CAS (Compare-And-Swap) for state transitions
- Never trust "if exists" checks in distributed systems

**"Running" ≠ "Working"**
- Deep health checks (not just status)
- Multiple retry attempts before giving up
- Automatic recovery when possible

**Diversity Has Hidden Costs**
- Different token counting methods
- Inconsistent error codes
- Varying API behaviors
- Need normalization layers

**Users Deserve Transparency**
- Show when fallbacks happen
- Explain degraded modes
- Log everything for debugging

---

## Slide 34: Metrics That Matter

**Reliability:**
- Uptime: 99.5% → 99.9%
- Mean time to recovery: 5min → 30s
- Fallback success rate: 95%

**Cost:**
- Token costs: -60% (caching)
- Infrastructure: -15% (better resource utilization)
- Wasted compute: -85% (race condition fixes)

**Performance:**
- Average latency: P95 < 3s
- Streaming time-to-first-token: <500ms
- Cache hit rate: 82%

**The Bottom Line:**
- More reliable
- Less expensive
- Happier customers

---

## Slide 35: Questions?

**Topics we covered:**
1. Multi-provider fallback architecture
2. Sandboxed execution & notebook pattern
3. Cost optimization with caching
4. Streaming & error resilience
5. Production war stories

**Want to discuss:**
- Specific implementation details
- Other resilience patterns
- Your production challenges

**Connect:**
- GitHub: [mention if public]
- Email: [your email]
- X/Twitter: @abhinavtumu

---

## Slide 36: Thank You!

**Building resilient AI agents is about:**
- Expecting failure
- Building redundancy
- Monitoring everything
- Learning from incidents

**The journey continues...**

Questions? Let's chat!

---

# APPENDIX: Backup Slides

## Slide A1: Token Optimization Strategies

Beyond caching:

1. **Tool definition compression**
   - Notebook pattern: 96% reduction
   - From 22K → 0.8K tokens

2. **Message history pruning**
   - Keep first N + last M messages
   - Summarize middle sections
   - Preserve important context

3. **Dynamic system prompts**
   - Load only relevant sections
   - Context-aware prompt assembly

---

## Slide A2: Observability Stack

**Metrics:**
- Token usage per user/thread
- Cache hit rates
- Latency percentiles (P50, P95, P99)
- Error rates by provider
- Fallback frequency

**Logs:**
- Structured JSON logs
- Request/response correlation IDs
- Tool execution traces
- Cost attribution

**Alerts:**
- Cost anomalies (>2σ)
- Error rate spikes
- Provider health
- Quota utilization

---

## Slide A3: Security Considerations

**VM Isolation:**
- Firecracker microVMs
- Minimal attack surface
- No shared kernel

**Credential Management:**
- Virtual keys (no real API keys in VMs)
- Automatic rotation
- Scoped permissions

**Data Isolation:**
- Redis ACLs
- Filesystem namespacing
- Network policies

**Audit Trail:**
- All tool executions logged
- User actions tracked
- Cost attribution

