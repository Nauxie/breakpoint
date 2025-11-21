# Speaker Notes & Timing Guide
## Breakpoint Presentation - November 21, 2025

---

## Pre-Talk Checklist

- [ ] Test laptop/projector connection
- [ ] Have backup of slides on USB drive
- [ ] Water bottle ready
- [ ] Phone on silent
- [ ] Demo environment tested (if doing live demo)
- [ ] Backup slides ready (appendix)

---

## Timing Breakdown (Target: 30 minutes)

| Section | Slides | Time | Cumulative |
|---------|--------|------|------------|
| Opening + Hook | 1-3 | 3 min | 3 min |
| Part 1: Multi-Provider | 4-8 | 6 min | 9 min |
| Part 2: Sandboxing | 9-15 | 7 min | 16 min |
| Part 3: Cost Optimization | 16-21 | 6 min | 22 min |
| Part 4: Streaming & Resilience | 22-26 | 5 min | 27 min |
| Part 5: War Stories | 27-31 | 5 min | 32 min |
| Closing | 32-36 | 2 min | 34 min |

**Buffer**: 6 minutes for questions/transitions

---

## Detailed Speaker Notes

### Slide 1-2: Opening (2 min)

**Hook: The $10,000 Morning**

*"Good evening everyone. Before we talk about best practices, let me tell you about a very expensive mistake."*

**Key Points:**
- Set the stage: production systems, real customers, real money
- This happened to us, could happen to anyone
- Vulnerability: single point of failure

**Tone:** Self-deprecating, honest, relatable

**Transition:** *"This experience taught us that building agents for demos is very different from building agents for production. Tonight I'll share 5 lessons we learned the hard way."*

---

### Slide 3: Context Setting (1 min)

**About Kafka**

*"Quick context on what we're building..."*

**Key Points:**
- Scale matters (500K requests/day gives credibility)
- Multi-tenant (complexity)
- 24/7 operation (can't just restart things)
- SLA requirements (failures have consequences)

**Tone:** Factual, establishing credibility

**Avoid:** Sounding like a sales pitch. This is about engineering challenges, not product marketing.

---

## Part 1: Multi-Provider Architecture (6 min)

### Slide 4: The Problem (1 min)

*"Provider outages aren't hypothetical. Let me show you real data..."*

**Anecdotes to mention:**
- Anthropic 429 errors during Claude 3 launch week
- AWS Bedrock throttling during peak hours
- OpenAI occasional outages

**Audience connection:** *"Raise your hand if you've been rate limited in production."* (Expect many hands)

---

### Slide 5-6: Gateway Pattern (2 min)

**Slide 5: Architecture**

*"We implemented what I call a 4-level cascading fallback..."*

**Key emphasis:**
- Both provider AND model fallbacks needed
- Status codes trigger automatic switching
- <1 second failover time

**Visual walkthrough:**
- Point to each level
- Explain trigger conditions
- Mention it's automatic, no manual intervention

**Slide 6: Virtual Keys**

*"The key insight here is abstraction..."*

**Why this matters:**
- No credentials in code
- Easy rotation
- Per-user rate limits
- Usage attribution for billing

**Avoid:** Getting into implementation details of the gateway service (that's vendor-specific)

---

### Slide 7: Load Balancing (1.5 min)

*"Now here's where it gets interesting - we can COMBINE strategies..."*

**Key concept:** Nested strategies
- Top level: load balance (split traffic)
- Nested: fallback (within each provider)

**Benefit:** Traffic distribution + resilience

**Call out:** The config is declarative, version controlled, per-user

---

### Slide 8: Production Lessons (1.5 min)

*"Let me share what we learned the hard way..."*

**Story 1: Regional differences**
- Initially only used one region
- AWS region outage took us down
- Now: multi-region by default

**Story 2: Token counting**
- Different providers count differently
- Caused billing discrepancies
- Built normalization layer

**Story 3: 2-level wasn't enough**
- Started with 2-level
- Still saw 0.3% failures
- 4-level got us to 0.01%

**Tone:** Honest about mistakes, show iteration

---

## Part 2: Sandboxed Execution (7 min)

### Slide 9: The Tool Schema Problem (1 min)

*"Let's talk about a common pattern that doesn't scale..."*

**Set up the problem:**
- Show the traditional approach
- 60+ tools = 20K+ tokens
- Cost: $0.08 per request JUST for definitions
- LLM gets confused with too many options

**Audience question:** *"Anyone here using 10+ tools in their agents?"*

---

### Slide 10-11: Notebook Pattern (2 min)

**Slide 10: The Pattern**

*"We solved this with what I call the notebook MCP pattern..."*

**Key insight:** One tool, infinite capabilities

**Why it works:**
- LLM writes code instead of selecting from schemas
- Persistent state (like Jupyter)
- Maximum flexibility

**Contrast:** 60 tools → 1 tool

**Slide 11: Example**

*"Let me show you what this looks like in practice..."*

**Walk through the code:**
- Step 1: Scrape (agent writes scraping code)
- Step 2: Analyze (state persists - `data` variable still available)
- Composable: agent builds workflows

**Emphasize:** This is more powerful than pre-defined tools

---

### Slide 12-13: Sandboxing (2 min)

**Slide 12: Architecture**

*"Of course, you can't just let agents run arbitrary code..."*

**Security layers:**
1. OS-level: Firecracker VMs (mention lightweight)
2. Filesystem: namespace isolation
3. Network: all traffic through proxy
4. Data: Redis ACLs

**Key point:** Defense in depth

**Slide 13: Redis Multi-tenancy**

*"Let me zoom in on the data isolation..."*

**Walk through code:**
- GENTOKEN: cryptographically secure
- Namespace patterns: ~user:123:*
- Command whitelist: only safe operations
- Blocklist: no FLUSHALL, no dangerous commands

**Result:** User A cannot see User B's data, even with credentials

---

### Slide 14-15: Race Conditions (2 min)

**Slide 14: The Bug**

*"Here's a fun bug that cost us money..."*

**Tell the story:**
- Multiple requests arrive simultaneously
- All check "does VM exist?"
- Race condition: all say "no"
- All create VMs
- 💥 Crash and burn

**Why it happened:** Distributed systems, no atomic operations

**Slide 15: The Fix**

*"The fix is using database-level atomic operations..."*

**Key concept:** Compare-and-swap (CAS)
- Only ONE request can transition from "off" to "starting"
- Winners create VM
- Losers wait for winner

**Show code:** Point out the database-level atomicity

**Broader lesson:** In distributed systems, check-then-act is always a race condition

---

## Part 3: Cost Optimization (6 min)

### Slide 16: The Opportunity (1 min)

*"Let's talk about money. Specifically, how to save it..."*

**Set up the economics:**
- Show the pricing: $3.00 vs $0.30 (10x difference!)
- Break-even: just 2 requests
- Long conversations = huge savings opportunity

**Emphasize:** This isn't theoretical, this is real money

---

### Slide 17-18: Progressive Caching (2 min)

**Slide 17: Strategy**

*"Anthropic allows 4 cache breakpoints. Order matters..."*

**Walk through priority:**
1. Tools array (15K tokens, rarely changes)
2. System prompt (3K tokens, never changes)
3. Large user content (10K+ tokens, sometimes)
4. Recent tool results (varies, frequently referenced)

**Why this order:** Maximize hit rate × token count

**Slide 18: Implementation**

*"Here's how we implement this..."*

**Key algorithm:**
- Identify candidates
- Sort by priority
- Take top N (within limits)
- Add cache_control markers

**Note:** Anthropic limit is 4 breakpoints, we respect that

---

### Slide 19: Real Numbers (1.5 min)

*"Let me show you actual numbers from production..."*

**Walk through the math:**
- 100-message conversation
- Without: $0.27
- With: $0.12
- Savings: 56%

**Scale it up:**
- 500K conversations/month
- $18K/month savings
- **$216K/year**

**Tone:** Let the numbers speak. This is significant.

---

### Slide 20: Cache Miss Mystery (1 min)

*"Here's a fun debugging story..."*

**The mystery:** Expected 85%, got 23%

**The investigation:**
- Looked at cached content
- Found timestamps in tool results
- Every result slightly different
- Cache never hits!

**The fix:**
- Canonicalize: remove timestamps
- Remove execution times
- Sort keys for consistency

**Result:** 82% hit rate

**Lesson:** Non-deterministic data breaks caching

---

### Slide 21: Provider Differences (0.5 min)

*"Quick gotcha to watch out for..."*

**Show the difference:**
- Some providers include cache tokens in prompt_tokens
- Others separate them
- Need to detect and normalize

**Avoid:** Getting too deep into details. This is a "watch out for this" moment.

---

## Part 4: Streaming & Resilience (5 min)

### Slide 22-23: Streaming Challenges (1.5 min)

**Slide 22: Why Hard**

*"Streaming is great for UX but terrible for reliability..."*

**The tension:**
- Users want real-time responses
- But streaming APIs are less stable
- Can't easily retry
- Error recovery complex

**Slide 23: Graceful Degradation**

*"Our solution: automatic fallback to non-streaming..."*

**Walk through code:**
- Try streaming first
- Catch specific errors
- Mark streaming as failed
- Notify user (important!)
- Retry non-streaming
- Exponential backoff

**Key point:** User sees smooth experience, we handle complexity

---

### Slide 24: Circuit Breaker (1 min)

*"Circuit breakers aren't just for networks..."*

**Pattern:**
- Track failures per tool
- After 3 failures, block tool for 5 minutes
- Prevents infinite loops
- Auto-reset after timeout

**Why needed:** LLMs will keep trying failing tools

---

### Slide 25-26: The Proxy (2 min)

**Slide 25: Architecture**

*"Everything goes through a proxy layer..."*

**Benefits:**
- Unified observability
- Cost attribution
- Credential management
- Rate limiting

**Emphasize:** This is YOUR infrastructure, you control it

**Slide 26: Webhook Tracking**

*"Here's how we track actual usage..."*

**Show code:**
- Webhook fires after each request
- Extract metadata (API key, user, thread)
- Parse token counts
- Handle cache tokens correctly
- Store for billing

**Result:** Perfect attribution, no estimates

---

## Part 5: War Stories (5 min)

### Introduction to War Stories (0.5 min)

*"Alright, time for the fun part. Let me tell you about 5 spectacular failures..."*

**Tone:** Lighthearted, self-deprecating

---

### Slide 27: Story 1 - Infinite Loop (1 min)

*"Picture this: Monday morning, $547 in unexpected charges..."*

**Tell the story:**
- Agent got stuck in loop
- Same tool call, over and over
- 400+ calls in 15 minutes
- We only noticed because of billing alert

**The lesson:** AI doesn't naturally avoid redundancy

**The fix:** Deduplication logic

**Audience reaction:** Should get sympathetic laughs

---

### Slide 28: Story 2 - Zombie VMs (1 min)

*"Here's a subtle one that drove us crazy..."*

**The mystery:**
- Status: "running" ✓
- Health check: passing ✓
- User experience: infinite loading ✗

**The investigation:**
- VM running
- OS running
- Application crashed
- Nobody noticed!

**The lesson:** "Running" ≠ "Working"

**The fix:** Deep health checks with retries

---

### Slide 29: Story 3 - Rate Limit Cascade (1 min)

*"This one was spectacular..."*

**Play-by-play:**
- Show timeline
- Primary hits limit
- Everyone fails over
- Backup also hits limit
- 100% error rate
- Stampeding herd problem

**The fix:**
- Jitter in retry timing
- Per-provider circuit breakers
- Load balancing when both healthy

**Lesson:** Distributed systems need randomness

---

### Slide 30: Story 4 - Partial Streaming (0.75 min)

*"Users started reporting weird truncated messages..."*

**Show the bug:**
- User sees partial response
- Message ends mid-sentence
- Confusing experience

**Root cause:**
- Streaming content
- Tool call interrupts
- Stream closes
- Can't resume

**The fix:**
- Proper state management
- Send status updates
- Handle transitions

---

### Slide 31: Story 5 - Cost Anomaly (0.75 min)

*"And finally, the cost anomaly..."*

**The alert:** $3,247 vs expected $800

**The investigation:**
- One user's agent
- Analyzing 500-page document
- Page by page
- Each call successful (no circuit breaker!)

**The lesson:** Circuit breakers catch failures, not inefficiency

**The fix:** Usage-based limits

---

## Closing (2 min)

### Slide 32-33: Key Takeaways (1.5 min)

*"Let me leave you with the core principles..."*

**Go through each:**
1. Assume everything will fail
2. Sandbox religiously
3. Optimize for cost early

**Non-obvious lessons:**
- Race conditions are expensive
- "Running" ≠ "Working"
- Diversity has hidden costs
- Transparency matters

**Tone:** Authoritative but humble. These are lessons learned, not lectures.

---

### Slide 34: Metrics (0.5 min)

*"Here's the impact of these patterns..."*

**Quick numbers:**
- Uptime: 99.5% → 99.9%
- Costs: -60%
- Performance: stable

**Don't dwell:** Numbers speak for themselves

---

### Slide 35-36: Q&A (variable)

*"Thank you! I'd love to hear your questions..."*

**Prepare for common questions:**
1. "What gateway service do you use?" → Can discuss offline
2. "How do you handle model context limits?" → Have appendix slide
3. "What about cost with multiple providers?" → Address that the redundancy cost is offset by reliability and caching savings
4. "How do you test failover?" → Chaos engineering, we manually trigger failures

**If no questions:** *"Feel free to grab me after, happy to chat about specific implementations."*

---

## Potential Audience Questions & Answers

### Technical Questions

**Q: "What's your actual uptime?"**
A: Currently 99.9% over last 6 months. We measure this as "user requests successfully completed" not just "service available."

**Q: "How do you handle model context length limits?"**
A: We have a sliding window approach - keep first N and last M messages, summarize the middle. Also have logic to drop old tool results. The notebook pattern helps because code is compact.

**Q: "What about costs of running multiple providers?"**
A: Great question. The overhead is about 10-15% more than single provider, but we offset this with caching (60% savings) and avoiding failures (each failure costs support time + customer trust). Net positive.

**Q: "How do you test failover?"**
A: We do chaos engineering - randomly inject failures in staging. Also have integration tests that mock provider responses. We test each level of the fallback chain.

**Q: "What framework are you using?"**
A: Custom built on FastAPI for the main server, Claude/OpenAI SDKs for LLMs, asyncio for concurrency. The proxy layer is also FastAPI.

**Q: "How do you handle versioning of prompts?"**
A: System prompts are version controlled in git. Each thread has a prompt version tag. We can A/B test prompts and roll back if needed.

### Business Questions

**Q: "How do you explain cost overruns to customers?"**
A: Full transparency - we show them the token usage dashboard. Most overruns are from their own usage patterns. We set up alerts for them.

**Q: "Do you pass through provider costs or markup?"**
A: We have tiered pricing based on usage. For enterprise, we can do cost-plus. The key is transparency in token tracking.

**Q: "How do you handle support for multi-provider?"**
A: We abstract it - users don't need to know which provider was used. Our logs show it, but users just see "agent responded."

### Implementation Questions

**Q: "Can I see the code?"**
A: Some patterns are open source adjacent (MCP, etc). Happy to discuss architectures offline. We're considering open-sourcing some components.

**Q: "What's the learning curve for your team?"**
A: The concepts are harder than the implementation. Once you understand fallbacks, circuit breakers, etc., the code is straightforward async Python.

**Q: "How many engineers maintain this?"**
A: 2-3 engineers work on the core agent infrastructure. The beauty of these patterns is they're robust - we spend more time on features than firefighting.

### Gotcha Questions

**Q: "Isn't this over-engineered?"**
A: For a demo, yes. For production with SLAs and paying customers, no. Each pattern solves a real problem we hit. I showed you the failures - this is the minimum to be reliable.

**Q: "Why not just use [specific vendor]?"**
A: Some of these patterns work great with vendors. The key is understanding the patterns regardless of implementation. Don't blindly trust any single service.

**Q: "What about costs of all these VMs?"**
A: VMs are surprisingly cheap - $0.10-0.20 per hour per VM. We auto-stop idle VMs. The LLM costs dwarf infrastructure costs (10:1 ratio).

---

## Backup Material

### If Running Short on Time

**Skip:**
- Slide 21 (provider differences detail)
- Slide 26 (webhook code)
- One war story (probably #4)

**Prioritize:**
- Multi-provider architecture (core pattern)
- Notebook MCP (most unique)
- At least 2 war stories (most engaging)

### If Have Extra Time

**Dive deeper on:**
- Show live dashboard of token usage
- Walk through actual fallback logs
- More war stories (have 3-4 more ready)

**Appendix slides available:**
- Token optimization strategies
- Observability stack details
- Security considerations

---

## Post-Talk

**Follow-ups:**
- [ ] Share slides on Twitter/LinkedIn
- [ ] Write blog post with more details
- [ ] Collect feedback from attendees
- [ ] Add any great questions to FAQ

**Networking:**
- Be available for 20-30min after
- Exchange contact info with interested engineers
- Note down interesting challenges people mention

---

## Delivery Tips

### Pacing
- Speak slowly and clearly
- Pause after showing code (let people read)
- Watch the clock at Part breaks
- If ahead: add details
- If behind: skip backup material

### Energy
- Vary vocal tone
- Use hand gestures for architecture diagrams
- Make eye contact with different parts of room
- Show enthusiasm during war stories

### Code Slides
- Don't read code line by line
- Point out the key lines
- Explain the concept, not syntax
- Use phrases like "the key here is..."

### Engagement
- Ask "Has anyone seen this?" periodically
- React to audience reactions
- If someone nods, acknowledge it
- If confused faces, pause and clarify

### Common Mistakes to Avoid
- ❌ Rushing through code slides
- ❌ Going over time
- ❌ Assuming everyone knows terminology
- ❌ Being defensive about choices
- ❌ Forgetting to breathe!

---

## Emergency Backup Plans

**If projector fails:**
- Have laptop screen sharing as backup
- Can walk through architecture verbally
- War stories work without slides

**If demo environment is down:**
- Have screenshots ready
- Can show from local environment
- Focus more on concepts

**If running way over time:**
- Skip to war stories (most valuable)
- Quick summary of patterns
- Offer to share detailed slides

**If audience seems confused:**
- Slow down
- Ask what's unclear
- Use more analogies
- Offer to revisit after talk

---

## Success Metrics

**Good talk if:**
- At least 3 people ask follow-up questions
- Someone says "I had that exact problem"
- Get requests for slides/code
- Twitter mentions

**Great talk if:**
- People stick around to chat
- Get invited to speak elsewhere
- Someone implements a pattern
- Start a conversation in the community

---

Good luck! Remember: you've lived this, you know it well. Trust your experience.

