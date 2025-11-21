# Production Agent Patterns - Code Examples
## Sanitized for Public Sharing

> **Note**: These examples are simplified and sanitized versions of production code.
> Vendor-specific names have been replaced with generic terms.

---

## 1. Multi-Provider Fallback Configuration

### Dynamic Config Builder

```python
from typing import List, Dict, Optional
from enum import Enum

class Provider(Enum):
    ANTHROPIC = "anthropic"
    BEDROCK = "bedrock"
    OPENAI = "openai"

def build_fallback_config(
    user_id: str,
    primary_provider: Provider,
    anthropic_key: str,
    bedrock_key: str,
    primary_model: str = "claude-4",
    fallback_model: str = "claude-3.7",
    guardrails: Optional[List[str]] = None
) -> Dict:
    """
    Build a multi-level fallback configuration with:
    1. Primary provider + primary model
    2. Secondary provider + primary model
    3. Primary provider + fallback model
    4. Secondary provider + fallback model
    """
    
    # Determine provider order
    if primary_provider == Provider.ANTHROPIC:
        first_key, second_key = anthropic_key, bedrock_key
        first_provider, second_provider = "anthropic", "bedrock"
    else:
        first_key, second_key = bedrock_key, anthropic_key
        first_provider, second_provider = "bedrock", "anthropic"
    
    # Convert model names for bedrock
    def to_bedrock_format(model: str) -> str:
        return f"us.anthropic.{model}-v1:0"
    
    # Build target configs
    def make_target(provider: str, key: str, model: str) -> Dict:
        model_id = model if provider == "anthropic" else to_bedrock_format(model)
        return {
            "virtual_key": key,
            "override_params": {
                "model": model_id,
            }
        }
    
    # Create 4-level fallback
    targets = [
        make_target(first_provider, first_key, primary_model),
        make_target(second_provider, second_key, primary_model),
        make_target(first_provider, first_key, fallback_model),
        make_target(second_provider, second_key, fallback_model),
    ]
    
    return {
        "name": f"{user_id[:8]}-multi-fallback",
        "config": {
            "output_guardrails": guardrails or [],
            "strategy": {
                "mode": "fallback",
                "on_status_codes": [400, 429, 500, 502, 503, 504]
            },
            "targets": targets
        }
    }
```

### Load Balancing + Fallbacks

```python
def build_loadbalanced_config(
    user_id: str,
    anthropic_key: str,
    bedrock_key: str,
    weight_anthropic: float = 0.5
) -> Dict:
    """
    Top-level load balancing with nested fallbacks per provider.
    """
    
    return {
        "name": f"{user_id[:8]}-lb-fallback",
        "config": {
            "strategy": {"mode": "loadbalance"},
            "targets": [
                {
                    "weight": weight_anthropic,
                    "strategy": {
                        "mode": "fallback",
                        "on_status_codes": [400, 429, 500, 502, 503, 504]
                    },
                    "targets": [
                        {
                            "virtual_key": anthropic_key,
                            "override_params": {"model": "claude-4"}
                        },
                        {
                            "virtual_key": anthropic_key,
                            "override_params": {"model": "claude-3.7"}
                        }
                    ]
                },
                {
                    "weight": 1 - weight_anthropic,
                    "strategy": {
                        "mode": "fallback",
                        "on_status_codes": [400, 429, 500, 502, 503, 504]
                    },
                    "targets": [
                        {
                            "virtual_key": bedrock_key,
                            "override_params": {"model": "us.anthropic.claude-4-v1:0"}
                        },
                        {
                            "virtual_key": bedrock_key,
                            "override_params": {"model": "us.anthropic.claude-3.7-v1:0"}
                        }
                    ]
                }
            ]
        }
    }
```

---

## 2. Sandboxed Code Execution (Notebook Pattern)

### MCP Tool Definition

```python
import asyncio
import sys
from io import StringIO
from contextlib import redirect_stdout, redirect_stderr
from IPython.core.interactiveshell import InteractiveShell

# Global shell instance for persistent state
shell: Optional[InteractiveShell] = None

async def initialize_notebook():
    """Initialize IPython shell with sandboxed environment"""
    global shell
    
    from IPython.terminal.interactiveshell import TerminalInteractiveShell
    import nest_asyncio
    
    # Allow nested async (needed for async code execution)
    nest_asyncio.apply()
    
    # Create shell
    shell = TerminalInteractiveShell.instance()
    
    # Set working directory
    import os
    if os.path.exists('/workspace'):
        os.chdir('/workspace')
    
    return shell

async def notebook_run_cell(
    code: str,
    description: str = "",
    timeout: int = 3600
) -> str:
    """
    Execute Python code in a Jupyter-like environment.
    
    Args:
        code: Python code to execute
        description: User-facing description of what this does
        timeout: Maximum execution time in seconds
    
    Returns:
        Combined stdout, stderr, and any error messages
    """
    global shell
    
    if shell is None:
        await initialize_notebook()
    
    # Capture output
    stdout_capture = StringIO()
    stderr_capture = StringIO()
    output = ""
    
    try:
        with redirect_stdout(stdout_capture), redirect_stderr(stderr_capture):
            # Check if code needs async execution
            if shell.should_run_async(code):
                # Run async code
                coro = shell.run_cell_async(code)
                await asyncio.wait_for(coro, timeout=timeout)
            else:
                # Run sync code
                shell.run_cell(code)
        
        # Get captured output
        stdout_output = stdout_capture.getvalue()
        stderr_output = stderr_capture.getvalue()
        
        if stdout_output:
            output += stdout_output
        if stderr_output:
            output += stderr_output
            
    except asyncio.TimeoutError:
        return f"Execution timed out after {timeout} seconds"
    
    except Exception as e:
        import traceback
        output += f"\nError: {str(e)}\n"
        output += traceback.format_exc()
    
    return output or "(no output)"
```

### Usage Example

```python
# Agent writes code to accomplish tasks
# Step 1: Scrape data
result1 = await notebook_run_cell(
    code="""
import requests
from bs4 import BeautifulSoup

url = 'https://example.com/data'
response = requests.get(url)
soup = BeautifulSoup(response.text, 'html.parser')
data = [item.text for item in soup.find_all('div', class_='data-item')]
print(f"Found {len(data)} items")
""",
    description="Scraping competitor data"
)

# Step 2: Analyze (state persists - 'data' variable still exists!)
result2 = await notebook_run_cell(
    code="""
import pandas as pd

df = pd.DataFrame({'value': data})
df['value_num'] = pd.to_numeric(df['value'], errors='coerce')
summary = df.describe()
print(summary)
""",
    description="Analyzing data"
)
```

---

## 3. Atomic Sandbox Creation

### Race Condition Prevention

```python
import asyncio
from enum import Enum
from typing import Dict, Any, Optional

class SandboxState(Enum):
    OFF = "off"
    STARTING = "starting"
    IDLE = "idle"
    ERROR = "error"

class SandboxManager:
    """Manages sandbox lifecycle with atomic operations"""
    
    def __init__(self, db):
        self.db = db
    
    async def get_or_create_sandbox(
        self,
        thread_id: str,
        _retry_count: int = 0
    ) -> Dict[str, Any]:
        """
        Get existing sandbox or create new one atomically.
        """
        if _retry_count > 3:
            raise Exception("Max retries exceeded")
        
        # Get current state
        thread_data = await self.db.get_thread(thread_id)
        current_status = thread_data.get("status", SandboxState.OFF.value)
        
        if current_status == SandboxState.OFF.value:
            # Try atomic creation
            return await self._atomic_create_sandbox(thread_id, thread_data)
        
        elif current_status == SandboxState.STARTING.value:
            # Another request is creating, wait for it
            return await self._wait_for_sandbox(thread_id, _retry_count)
        
        elif current_status == SandboxState.IDLE.value:
            # Sandbox exists, return info
            return await self._get_sandbox_info(thread_id, thread_data)
        
        else:
            # Error state, reset and retry
            await self.db.update_thread(thread_id, {"status": SandboxState.OFF.value})
            await asyncio.sleep(1)
            return await self.get_or_create_sandbox(thread_id, _retry_count + 1)
    
    async def _atomic_create_sandbox(
        self,
        thread_id: str,
        thread_data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Atomically create sandbox using database-level CAS.
        Only one request can successfully transition from 'off' to 'starting'.
        """
        # Database-level atomic operation (compare-and-swap)
        success = await self.db.atomic_update_thread_status(
            thread_id,
            expected_status=SandboxState.OFF.value,
            new_status=SandboxState.STARTING.value
        )
        
        if success:
            # This request won the race - create the sandbox
            try:
                return await self._create_new_sandbox(thread_id, thread_data)
            except Exception:
                # If creation fails, set error status
                await self.db.update_thread(
                    thread_id,
                    {"status": SandboxState.ERROR.value}
                )
                raise
        else:
            # Another request is already creating - wait for it
            await asyncio.sleep(1)
            fresh_data = await self.db.get_thread(thread_id)
            return await self.get_or_create_sandbox(thread_id, 0)
    
    async def _wait_for_sandbox(
        self,
        thread_id: str,
        retry_count: int
    ) -> Dict[str, Any]:
        """Wait for another request to finish creating sandbox"""
        max_wait = 120  # 2 minutes
        check_interval = 2
        elapsed = 0
        
        while elapsed < max_wait:
            await asyncio.sleep(check_interval)
            elapsed += check_interval
            
            thread_data = await self.db.get_thread(thread_id)
            status = thread_data.get("status")
            
            if status == SandboxState.IDLE.value:
                return await self._get_sandbox_info(thread_id, thread_data)
            elif status == SandboxState.ERROR.value:
                raise Exception("Sandbox creation failed")
        
        raise Exception("Timeout waiting for sandbox creation")
```

### Database Atomic Update

```python
async def atomic_update_thread_status(
    thread_id: str,
    expected_status: str,
    new_status: str
) -> bool:
    """
    Atomic compare-and-swap operation at database level.
    Returns True if update succeeded, False if status didn't match.
    """
    # PostgreSQL example using UPDATE with WHERE condition
    result = await db.execute("""
        UPDATE threads
        SET status = $1
        WHERE id = $2 AND status = $3
        RETURNING id
    """, new_status, thread_id, expected_status)
    
    return len(result) > 0
```

---

## 4. Redis Multi-tenancy with ACLs

```python
import secrets
import redis

class RedisMultiTenant:
    """Manage per-user Redis namespaces with ACL isolation"""
    
    def __init__(self, host: str, port: int, password: str):
        self.redis = redis.Redis(
            host=host,
            port=port,
            password=password,
            decode_responses=True,
            ssl=True
        )
    
    def provision_user(self, user_id: str) -> Dict[str, str]:
        """
        Create isolated Redis access for a user.
        
        Returns:
            dict with username, password, and connection info
        """
        username = f"user_{user_id}"
        
        try:
            # Generate cryptographically secure token
            token = self.redis.execute_command('ACL', 'GENTOKEN', username)
            
            # Create user with restricted permissions
            self.redis.execute_command(
                'ACL', 'SETUSER', username,
                'ON',                           # Enable user
                f'>{token}',                    # Set password
                f'~user:{user_id}:*',          # Key pattern 1
                f'~queue:{user_id}:*',         # Key pattern 2
                f'~data:{user_id}:*',          # Key pattern 3
                '+ping',                        # Allow ping
                '+brpop', '+blpop',            # Queue operations
                '+rpush', '+lpush',
                '+get', '+set', '+del',        # Basic operations
                '+exists', '+expire',
                '+hget', '+hset', '+hdel',     # Hash operations
                '-@dangerous',                  # Block dangerous commands
                '-flushdb', '-flushall',       # Block destructive ops
                '-acl'                          # Block ACL modifications
            )
            
            # Test the connection
            test_client = redis.Redis(
                host=self.redis.connection_pool.connection_kwargs['host'],
                port=self.redis.connection_pool.connection_kwargs['port'],
                username=username,
                password=token,
                decode_responses=True,
                ssl=True
            )
            test_client.ping()
            
            return {
                'user_id': user_id,
                'username': username,
                'password': token,
                'method': 'acl'
            }
            
        except Exception as e:
            # Clean up on failure
            try:
                self.redis.execute_command('ACL', 'DELUSER', username)
            except:
                pass
            raise
    
    def delete_user(self, user_id: str) -> bool:
        """Remove user's Redis access"""
        username = f"user_{user_id}"
        try:
            self.redis.execute_command('ACL', 'DELUSER', username)
            return True
        except Exception:
            return False
```

---

## 5. Progressive Caching Strategy

```python
from typing import List, Dict, Any

def count_tokens(text: str) -> int:
    """Estimate token count (simplified)"""
    return len(text) // 4

def apply_progressive_caching(
    messages: List[Dict[str, Any]],
    max_cache_blocks: int = 4
) -> List[Dict[str, Any]]:
    """
    Apply cache control markers to maximize cache hit rate.
    
    Strategy:
    1. Cache tools array (largest, changes least)
    2. Cache system prompt
    3. Cache large user content (>10K tokens)
    4. Cache recent tool results
    """
    
    candidates = []
    
    # Identify cacheable content
    for i, msg in enumerate(messages):
        role = msg.get('role')
        content = msg.get('content', '')
        
        if isinstance(content, str):
            token_count = count_tokens(content)
        else:
            # List content (multimodal)
            token_count = sum(
                count_tokens(item.get('text', ''))
                for item in content
                if item.get('type') == 'text'
            )
        
        # Add to candidates
        if role == 'system':
            candidates.append({
                'msg_idx': i,
                'type': 'system',
                'tokens': token_count,
                'priority': 1  # Highest
            })
        elif role == 'user' and token_count > 10000:
            candidates.append({
                'msg_idx': i,
                'type': 'large_user',
                'tokens': token_count,
                'priority': 2
            })
        elif role == 'tool':
            candidates.append({
                'msg_idx': i,
                'type': 'tool',
                'tokens': token_count,
                'priority': 3,
                'position': i  # For sorting by recency
            })
    
    # Reserve 1 slot for tools array (handled separately)
    available_slots = max_cache_blocks - 1
    
    # Select items to cache
    keep_items = []
    
    # 1. System prompts (usually 1)
    system_items = [c for c in candidates if c['type'] == 'system']
    keep_items.extend(system_items[:available_slots])
    available_slots -= len(system_items[:available_slots])
    
    # 2. Large user content (sorted by size)
    if available_slots > 0:
        large_user_items = sorted(
            [c for c in candidates if c['type'] == 'large_user'],
            key=lambda x: x['tokens'],
            reverse=True
        )
        keep_items.extend(large_user_items[:available_slots])
        available_slots -= len(large_user_items[:available_slots])
    
    # 3. Recent tool results
    if available_slots > 0:
        tool_items = sorted(
            [c for c in candidates if c['type'] == 'tool'],
            key=lambda x: x['position'],
            reverse=True  # Most recent first
        )
        keep_items.extend(tool_items[:available_slots])
    
    # Apply cache control markers
    for item in keep_items:
        msg_idx = item['msg_idx']
        content = messages[msg_idx]['content']
        
        if isinstance(content, list):
            # Add to last content block
            if content:
                content[-1]['cache_control'] = {'type': 'ephemeral'}
        else:
            # Convert to list format with cache control
            messages[msg_idx]['content'] = [
                {
                    'type': 'text',
                    'text': content,
                    'cache_control': {'type': 'ephemeral'}
                }
            ]
    
    return messages
```

### Canonicalization for Cache Consistency

```python
import json
from typing import Any, Dict

def canonicalize_for_cache(data: Dict[str, Any]) -> str:
    """
    Remove non-deterministic fields to improve cache hit rate.
    """
    # Fields that change every request
    non_deterministic_fields = {
        'timestamp',
        'execution_time_ms',
        'request_id',
        'trace_id',
        'created_at',
        'updated_at'
    }
    
    def clean_dict(d: Dict) -> Dict:
        if not isinstance(d, dict):
            return d
        
        return {
            k: clean_dict(v) if isinstance(v, dict) else v
            for k, v in d.items()
            if k not in non_deterministic_fields
        }
    
    cleaned = clean_dict(data)
    
    # Sort keys for consistency
    return json.dumps(cleaned, sort_keys=True)
```

---

## 6. Streaming with Graceful Degradation

```python
import asyncio
from typing import AsyncGenerator, Set

class StreamingOverloadedError(Exception):
    """Raised when streaming API is overloaded"""
    pass

class Agent:
    def __init__(self):
        self.streaming_failed_tools: Set[str] = set()
    
    async def run(
        self,
        messages: List[Dict],
        max_attempts: int = 3
    ) -> AsyncGenerator[str, None]:
        """
        Run agent with automatic fallback from streaming to non-streaming.
        """
        
        for attempt in range(max_attempts):
            try:
                # Check if streaming is disabled
                use_streaming = "__overloaded__" not in self.streaming_failed_tools
                
                if use_streaming:
                    # Try streaming
                    async for chunk in self._run_streaming(messages):
                        yield chunk
                    break  # Success
                    
                else:
                    # Use non-streaming
                    response = await self._run_non_streaming(messages)
                    yield response
                    break  # Success
                
            except StreamingOverloadedError as e:
                # Mark streaming as failed
                self.streaming_failed_tools.add("__overloaded__")
                
                # Notify user (only once)
                if attempt == 0:
                    yield json.dumps({
                        "type": "status",
                        "message": "Switching to non-streaming mode",
                        "reason": "api_overloaded"
                    })
                
                # Retry with exponential backoff
                if attempt < max_attempts - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
                else:
                    raise
            
            except Exception as e:
                # Other errors - retry or fail
                if attempt < max_attempts - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
                else:
                    raise
    
    async def _run_streaming(
        self,
        messages: List[Dict]
    ) -> AsyncGenerator[str, None]:
        """Run with streaming enabled"""
        stream = await self.client.create(
            messages=messages,
            stream=True,
            max_tokens=4096
        )
        
        async for chunk in stream:
            delta = chunk.choices[0].delta
            if delta.content:
                yield delta.content
    
    async def _run_non_streaming(
        self,
        messages: List[Dict]
    ) -> str:
        """Run without streaming"""
        response = await self.client.create(
            messages=messages,
            stream=False,
            max_tokens=4096
        )
        return response.choices[0].message.content
```

---

## 7. Circuit Breaker for Tool Calls

```python
import time
from typing import Dict, Set
from collections import defaultdict

class ToolCircuitBreaker:
    """
    Prevent infinite loops of failing tool calls.
    """
    
    def __init__(
        self,
        failure_threshold: int = 3,
        reset_timeout: int = 300  # 5 minutes
    ):
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        
        self.failure_counts: Dict[str, int] = defaultdict(int)
        self.last_failure_times: Dict[str, float] = {}
        self.blocked_tools: Set[str] = set()
    
    def is_tool_blocked(self, tool_name: str) -> bool:
        """Check if tool is currently blocked"""
        if tool_name not in self.blocked_tools:
            return False
        
        # Check if enough time has passed to reset
        last_failure = self.last_failure_times.get(tool_name, 0)
        if time.time() - last_failure > self.reset_timeout:
            self.reset_tool(tool_name)
            return False
        
        return True
    
    def record_failure(self, tool_name: str):
        """Record a tool failure"""
        self.failure_counts[tool_name] += 1
        self.last_failure_times[tool_name] = time.time()
        
        if self.failure_counts[tool_name] >= self.failure_threshold:
            self.blocked_tools.add(tool_name)
            print(f"⚠️  Tool '{tool_name}' blocked after {self.failure_threshold} failures")
    
    def record_success(self, tool_name: str):
        """Record a successful tool call (resets failure count)"""
        self.failure_counts[tool_name] = 0
        self.blocked_tools.discard(tool_name)
    
    def reset_tool(self, tool_name: str):
        """Manually reset a tool's circuit breaker"""
        self.failure_counts[tool_name] = 0
        self.blocked_tools.discard(tool_name)
        print(f"✓ Tool '{tool_name}' circuit breaker reset")
    
    def get_status(self) -> Dict:
        """Get current status of all circuit breakers"""
        return {
            "blocked_tools": list(self.blocked_tools),
            "failure_counts": dict(self.failure_counts),
            "last_failures": {
                tool: time.time() - ts
                for tool, ts in self.last_failure_times.items()
            }
        }

# Usage
circuit_breaker = ToolCircuitBreaker(failure_threshold=3, reset_timeout=300)

async def execute_tool(tool_name: str, tool_args: Dict) -> Any:
    """Execute tool with circuit breaker protection"""
    
    # Check if tool is blocked
    if circuit_breaker.is_tool_blocked(tool_name):
        raise Exception(f"Tool '{tool_name}' is temporarily blocked due to repeated failures")
    
    try:
        # Execute tool
        result = await actual_tool_execution(tool_name, tool_args)
        
        # Record success
        circuit_breaker.record_success(tool_name)
        
        return result
        
    except Exception as e:
        # Record failure
        circuit_breaker.record_failure(tool_name)
        raise
```

---

## 8. Tool Call Deduplication

```python
import hashlib
import json
from collections import deque
from typing import Dict, Tuple, Optional

class ToolDeduplicator:
    """Prevent agent from calling same tool with same args repeatedly"""
    
    def __init__(self, window_size: int = 10):
        self.recent_calls = deque(maxlen=window_size)
    
    def get_call_signature(self, tool_name: str, tool_args: Dict) -> str:
        """Generate unique signature for tool call"""
        # Sort args for consistency
        args_json = json.dumps(tool_args, sort_keys=True)
        
        # Create hash
        signature = f"{tool_name}:{hashlib.md5(args_json.encode()).hexdigest()}"
        return signature
    
    def should_execute(
        self,
        tool_name: str,
        tool_args: Dict
    ) -> Tuple[bool, Optional[str]]:
        """
        Check if tool call should be executed.
        
        Returns:
            (should_execute, reason)
        """
        signature = self.get_call_signature(tool_name, tool_args)
        
        if signature in self.recent_calls:
            return False, f"Duplicate call detected in last {len(self.recent_calls)} calls"
        
        # Add to recent calls
        self.recent_calls.append(signature)
        return True, None
    
    def clear(self):
        """Clear history (e.g., at start of new conversation)"""
        self.recent_calls.clear()

# Usage
deduplicator = ToolDeduplicator(window_size=10)

async def execute_tool_with_dedup(tool_name: str, tool_args: Dict) -> Any:
    """Execute tool with deduplication check"""
    
    should_execute, reason = deduplicator.should_execute(tool_name, tool_args)
    
    if not should_execute:
        return {
            "error": "Duplicate tool call",
            "reason": reason,
            "suggestion": "Try modifying your query or using different parameters"
        }
    
    # Execute tool
    return await actual_tool_execution(tool_name, tool_args)
```

---

## 9. Usage-Based Rate Limiting

```python
from datetime import datetime
from typing import Dict, Tuple
from decimal import Decimal

class UsageLimiter:
    """Prevent runaway costs with token-based limits"""
    
    def __init__(
        self,
        max_tokens_per_hour: int = 1_000_000,
        max_requests_per_minute: int = 100
    ):
        self.max_tokens_per_hour = max_tokens_per_hour
        self.max_requests_per_minute = max_requests_per_minute
        
        self.token_usage: Dict[Tuple[str, str], int] = {}
        self.request_counts: Dict[Tuple[str, str], int] = {}
    
    async def check_limit(
        self,
        user_id: str,
        estimated_tokens: int
    ) -> Tuple[bool, Optional[str]]:
        """
        Check if request is within limits.
        
        Returns:
            (allowed, reason_if_denied)
        """
        # Check token limit (hourly)
        hour_key = datetime.now().strftime("%Y-%m-%d-%H")
        token_key = (user_id, hour_key)
        current_usage = self.token_usage.get(token_key, 0)
        
        if current_usage + estimated_tokens > self.max_tokens_per_hour:
            return False, f"Hourly token limit reached: {current_usage:,} / {self.max_tokens_per_hour:,}"
        
        # Check request limit (per minute)
        minute_key = datetime.now().strftime("%Y-%m-%d-%H-%M")
        request_key = (user_id, minute_key)
        current_requests = self.request_counts.get(request_key, 0)
        
        if current_requests >= self.max_requests_per_minute:
            return False, f"Rate limit: {self.max_requests_per_minute} requests/minute"
        
        # Update usage
        self.token_usage[token_key] = current_usage + estimated_tokens
        self.request_counts[request_key] = current_requests + 1
        
        return True, None
    
    def get_usage(self, user_id: str) -> Dict:
        """Get current usage for user"""
        hour_key = datetime.now().strftime("%Y-%m-%d-%H")
        minute_key = datetime.now().strftime("%Y-%m-%d-%H-%M")
        
        return {
            "tokens_this_hour": self.token_usage.get((user_id, hour_key), 0),
            "requests_this_minute": self.request_counts.get((user_id, minute_key), 0),
            "limits": {
                "tokens_per_hour": self.max_tokens_per_hour,
                "requests_per_minute": self.max_requests_per_minute
            }
        }
```

---

## 10. Retry with Exponential Backoff and Jitter

```python
import asyncio
import random
from typing import Callable, Any, List

async def retry_with_backoff(
    func: Callable,
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    jitter: bool = True,
    retriable_exceptions: List[type] = None
) -> Any:
    """
    Retry function with exponential backoff and optional jitter.
    
    Args:
        func: Async function to retry
        max_attempts: Maximum number of attempts
        base_delay: Base delay in seconds
        max_delay: Maximum delay in seconds
        jitter: Whether to add random jitter to prevent thundering herd
        retriable_exceptions: List of exception types that should trigger retry
    """
    if retriable_exceptions is None:
        retriable_exceptions = [Exception]
    
    last_exception = None
    
    for attempt in range(max_attempts):
        try:
            return await func()
        
        except tuple(retriable_exceptions) as e:
            last_exception = e
            
            if attempt == max_attempts - 1:
                # Last attempt, don't retry
                raise
            
            # Calculate delay with exponential backoff
            delay = min(base_delay * (2 ** attempt), max_delay)
            
            # Add jitter (±25% of delay)
            if jitter:
                jitter_amount = random.uniform(-0.25, 0.25) * delay
                delay += jitter_amount
            
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay:.2f}s...")
            await asyncio.sleep(delay)
    
    # Should never reach here, but just in case
    raise last_exception

# Usage
async def make_api_call():
    """Example API call that might fail"""
    response = await http_client.post("/api/endpoint", json={...})
    return response.json()

# Retry with exponential backoff
result = await retry_with_backoff(
    make_api_call,
    max_attempts=3,
    base_delay=1.0,
    jitter=True,
    retriable_exceptions=[TimeoutError, ConnectionError]
)
```

---

## Notes

These examples demonstrate production-proven patterns for:
1. Multi-provider resilience
2. Secure code execution
3. Distributed system coordination
4. Cost optimization
5. Error handling and recovery

All code is simplified for clarity and has vendor-specific details removed.
For production use, add:
- Comprehensive error handling
- Detailed logging
- Metrics collection
- Security hardening
- Testing

