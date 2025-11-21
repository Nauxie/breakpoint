# Design Principles for Production Agents

## 10-12 Minute Talk - Engineering Lessons from Building Kafka

**Framing**: This is about the hard lessons we learned building a production agent system at Brainbase Labs. Kafka is the case study that proves these principles work. This is what we'd tell our past selves a year ago.

**Target Audience**: 50 SF engineers (20s-30s) building with AI daily  
**Style**: Story-like, technically detailed where it matters, no bullshit

---

## SLIDE 1: Title

```
Design Principles for Production Agents

Lessons from Building Kafka at Brainbase Labs

Abhinav Tumu
YC W24
November 21, 2025
```

**Delivery**: Straightforward intro, no theatrics.

---

## SLIDE 2: Hook - The Year in Review

**Opening** (conversational):

*"A year ago, we started building what we thought would be a simple agent. The plan was straightforward: good LLM, decent tools, some prompting. Ship it."*

[pause]

*"Turns out, production changes everything. Today I want to share the hard lessons we learned - the architectural decisions that actually matter when users depend on your agent working every single day."*

**Visual**: Timeline showing key inflection points
```
Month 1: "This works great!"
Month 3: First multi-day workflow failure
Month 6: Scaling crisis (100→1000 users)
Month 9: Architecture redesign
Month 12: Actually production-ready
```

**Key Point**: Not a sales pitch - these are real engineering lessons with scars to prove it.

---

## SLIDE 3: Context - What We Built

**Brainbase Labs** (YC W24)
Applied research lab focused on generalist agents

**Kafka: AI Employee Platform**
- Handles complex workflows (grant research, product management automation)
- Multi-hour/day autonomous execution
- 2000+ third-party integrations
- Real users, real stakes

**The Challenge**:
Users don't want a chatbot. They want an employee that **just works**.

**Visual**: Simple diagram of what Kafka does
- Input: Complex, ambiguous requests
- Output: Multi-day workflows that complete autonomously
- Not: Q&A bot or API wrapper

---

## PRINCIPLE 1: EXECUTABLE CONTEXT OVER STATIC PROMPTS (3 min)

### SLIDE 4: The Context Problem

**The Story**:

*"Early on, we had a massive system prompt. Every integration, every pattern, every edge case - all in one 10K token prompt. It was slow, expensive, and the agent still guessed wrong half the time."*

**The Core Issue**:
- Users have unique workflows (different tools, different processes)
- One prompt can't cover everyone's needs
- Can't keep adding to system prompt (doesn't scale)

**The Question We Asked**:
"What if context could be **loaded dynamically** based on what the user is actually trying to do?"

**Visual**: Simple before/after
```
Static Prompt (10K tokens)          Dynamic Context Loading
• All integrations listed           • Load only relevant workflows
• All patterns explained            • Inject user-specific patterns  
• Still needs clarification         • Zero clarification needed
```

**No made-up metrics** - just the architectural problem statement.

---

### SLIDE 5: Playbooks - How We Solved It

**Our Solution: Executable Playbooks**

Think of them as **SOPs (Standard Operating Procedures) as code**.

**How it works**:

1. User creates playbooks (natural language, like writing an SOP)
2. We embed playbooks into vector space
3. On each request, semantic search retrieves relevant playbook
4. Playbook content is injected as context

**Real Example** (customer workflow):

```
Trigger phrase: "Generate daily standup report"

Playbook content injected:
• Which Slack channels to check (#ls_salus, #ls_product)
• Which ClickUp workspace to query
• Exact data to extract (hours logged, estimates, due dates)
• Output format template
• Where to post results

Agent executes 8-step workflow with zero clarification.
```

**The Key Engineering Insight**:
- Playbooks are **user-created** (not us defining every workflow)
- Semantic search means approximate matches work
- Context is **personalized** without hard-coding every user

**Visual**: Flow diagram
```
Request → Vector Search → Match Playbook → Inject Context → Execute
```

---

### SLIDE 6: Implementation Detail - Playbook Selection

**The Interesting Part** (technical detail):

We use **sentence transformers** to embed playbooks:
- Embed user request: `embed("generate standup report")`
- Semantic search over playbook embeddings
- Cosine similarity for ranking

**Thresholds we learned**:
- Score > 0.85: Auto-load (high confidence)
- Score 0.70-0.85: Show user, ask confirmation
- Score < 0.70: Proceed without playbook

**Why This Works**:
- O(1) retrieval vs O(n) trial-and-error
- Works with fuzzy matches ("daily update" → "standup report")
- Scales: 10 playbooks or 1000 playbooks, same performance

**What We Learned**:
- Initially used keyword matching → terrible
- GPT-4 embeddings → too expensive
- Sentence transformers → sweet spot (fast, cheap, good enough)

**Visual**: Code snippet showing the selection logic
```python
# Simplified version
def select_playbook(user_request: str):
    request_embedding = embed(user_request)
    
    for playbook in playbook_library:
        similarity = cosine_similarity(
            request_embedding, 
            playbook.embedding
        )
        
        if similarity > 0.85:
            return playbook  # Auto-load
```

**Transition**: *"Playbooks solve the context problem. But they don't solve the next problem: efficiency."*

---

## PRINCIPLE 2: CODE EXECUTION OVER TOOL SCHEMAS (3 min)

### SLIDE 7: The Efficiency Problem

**The Story**:

*"We hit this wall where simple tasks were taking forever. Send 50 emails? That's 50 tool calls, 50 LLM roundtrips, 3-4 minutes. Users were frustrated."*

**The Core Issue**:
Traditional MCP pattern:
- 1 tool = 1 function = 1 API call
- For bulk operations, you call the tool N times
- Each call is an LLM roundtrip

**The Question We Asked**:
"What if the agent could write actual code instead of just calling pre-defined functions?"

**Visual**: Timeline showing execution
```
Traditional Tool Calling:
Call 1 → LLM → Call 2 → LLM → Call 3 → LLM → ... [50 times]
Time: 3-4 minutes

Code Execution:
Write loop → Execute → Done
Time: 30 seconds
```

**The Insight**: 
We're artificially limiting the agent by forcing it to use tool schemas. Give it a **full programming environment** instead.

---

### SLIDE 8: Notebook Execution Environment

**Our Solution: Persistent Jupyter-like Notebooks**

Every user thread gets:
- IPython shell (persistent state)
- Pre-imported helper classes (SearchV2, WebCrawler, AppFactory, etc.)
- Full Python environment
- Firecracker microVM for isolation

**Code Example** (show actual implementation pattern):

```python
# Traditional MCP (pseudocode):
for i in range(50):
    tool_call("send_email", {"to": emails[i], "subject": ...})
    # Wait for LLM, wait for response, repeat

# Our approach - agent writes this:
from integrations import AppFactory

factory = AppFactory()
gmail = factory.app("gmail")
send = gmail.action("gmail-send-email")

for person in recipients:
    send.configure({
        "to": person.email,
        "subject": f"Hi {person.name}",
        "body": template.render(person)
    })
    send.run()
    
print(f"Sent {len(recipients)} emails")
```

**What Changed**:
- Agent writes the loop, not us
- State persists (variables available across cells)
- Can compose tools in arbitrary ways
- Dramatically faster for bulk operations

---

### SLIDE 9: Why Code Execution + Implementation Details

**The Architectural Decision**:

We debated this for weeks:
- Security concerns (arbitrary code execution)
- Debugging complexity (code bugs vs LLM bugs)
- But the efficiency gain was undeniable

**How We Made It Safe**:
- **Firecracker microVMs**: OS-level isolation per user
- **Network proxy**: All external calls routed through proxy
- **Resource limits**: CPU, memory, disk quotas
- **Filesystem restrictions**: Can't access system files

**The Unexpected Benefit**:
Infinite composability. Agent can:
- Write loops, conditionals, error handling
- Use any Python library (requests, pandas, etc.)
- Combine tools in ways we never imagined

**Example** (real usage pattern):
```python
# Agent figures this out on its own:
results = []
for url in competitor_urls:
    content = await WebCrawler.crawl_simple(url)
    analysis = Agent(model="gpt-5").run(
        f"Extract pricing from:\n{content['markdown']}"
    )
    results.append(analysis)

# Loops + web crawling + subagent analysis
# All in one cell, composed by the agent
```

**The Lesson**: 
Don't constrain agents to pre-defined schemas. Give them primitives and let them compose.

**Transition**: *"Code execution solves efficiency. But complex workflows need more than one agent."*

---

## PRINCIPLE 3: MULTI-AGENT SPECIALIZATION (3 min)

### SLIDE 10: The Complexity Problem

**The Story**:

*"As workflows got more complex, we tried making one super-agent that did everything. Analysis, planning, execution, error recovery - all in one prompt. It was a mess."*

**The Core Issue**:
- One agent can't be great at everything
- Long prompts lead to lost instructions
- No clear separation of concerns

**The Question We Asked**:
"What if we had specialized agents for specialized tasks?"

**Visual**: Diagram showing the shift
```
BEFORE: Monolithic Agent
[Giant prompt with all instructions]
• Planning
• Execution  
• Deep analysis
• Error handling
→ Context overload

AFTER: Specialized Agents
Main Agent (orchestration)
├→ Subagent (deep reasoning, 1M context)
├→ MeetingBot (video meetings)
└→ ScheduledAgent (future execution)
→ Clear responsibilities
```

---

### SLIDE 11: Our Multi-Agent Architecture

**How We Structured It**:

**Main Agent** (Claude Sonnet 4.5):
- Orchestrates workflow
- Writes notebook cells
- Manages user communication
- Delegates to specialists

**Subagent** (GPT-5, 1M tokens):
- Deep analysis (long documents)
- Image understanding
- Complex reasoning tasks
- Called from notebook cells

**Delegation Pattern** (show code):
```python
# Main agent writes this in notebook:
from agent import Agent

subagent = Agent(model="gpt-5")
analysis = subagent.run([
    {"type": "text", "text": "Analyze this 500-page grant proposal"},
    {"type": "image_url", "image_url": {"url": "workspace/proposal.pdf"}}
])

# Main agent continues with analysis results
# Subagent handles the heavy lifting
```

**Why This Works**:
- Main agent: Broad, shallow context (orchestration)
- Subagent: Narrow, deep context (analysis)
- Each optimized for its job

---

### SLIDE 12: Persistence - The Missing Piece

**The Problem**:
Multi-hour or multi-day workflows can't live in memory.

**Our Solution: File-Based Planning**

We use `todo.md` files:
```markdown
# Grant Research Workflow

- [x] Search for relevant grants
      Completed: 2025-11-20 14:23
      
- [x] Crawl grant websites  
      Completed: 2025-11-20 14:45
      
- [in_progress] Extract requirements
      Started: 2025-11-20 15:02
      
- [ ] Draft application
- [ ] Schedule follow-up
```

**What This Enables**:
- Workflows can pause/resume (sandbox sleeps/wakes)
- User sees real-time progress
- Auditable execution trail
- Self-recovery on failure

**The Architectural Insight**:
In traditional systems, state lives in memory (lost on restart). We inverted this:
- **File system is source of truth**
- Memory is just a cache
- Enables multi-day autonomous workflows

**Real Example**: 
Grant research workflow runs for 3 days, pausing overnight when sandbox sleeps, resuming automatically in the morning.

---

### SLIDE 13: Putting It Together - Live Demo

**Real Customer Workflow**: ClickUp Daily Standup Automation

*"Let me show you these principles in action. This is a real workflow a customer uses every day."*

**What It Does**:
1. Checks Slack for daily standup message
2. Extracts ClickUp task link
3. Navigates ClickUp, pulls task data
4. Analyzes comments to see who's working on what
5. Formats report with hours logged, estimates, due dates
6. Posts to 2 Slack channels

**The Principles at Work**:
- **Playbook**: Auto-loads ClickUp standup context (channels, boards, format)
- **Code execution**: Loops through team members, composes report
- **Multi-agent**: Subagent analyzes natural language in task comments
- **Persistence**: If it fails mid-way, resumes from last step

*[Kick off the workflow here]*

*"This is running now. While it finishes, let me show you how these principles compose..."*

**Visual**: Execution log showing the steps

---

## SLIDE 14: How The Principles Compose

**The Magic is in the Composition**:

Each principle solves one problem:
- Playbooks → Context problem
- Code execution → Efficiency problem  
- Multi-agent + persistence → Complexity problem

**But together**, they enable something greater:

```
User Request
    ↓
[Playbook Selection: Semantic search loads workflow]
    ↓
Main Agent: Plans steps, creates todo.md
    ↓
[Code Execution: Writes notebook cells with loops]
    ↓
[Multi-Agent: Delegates analysis to subagent]
    ↓
[Persistence: todo.md tracks progress]
    ↓
Workflow Completes (hours/days later)
```

**What This Architecture Enables**:
- Arbitrary complexity (limited by time, not system)
- User-specific workflows (playbooks are personalized)
- Efficient execution (code, not tool calls)
- Reliable completion (persistence + recovery)

**The Meta-Lesson**:
You can't bolt these on after. They're architectural from day one.

---

## SLIDE 15: Demo Results

*[Switch to Slack channels]*

**Show the completed workflow**:
- Formatted standup report posted to both channels
- Correct data extracted (hours, estimates, due dates)
- Proper formatting, team member names

**Point Out**:
- This ran autonomously while I was talking
- No human intervention
- If it had failed at step 3, it would have resumed from step 3
- Customer runs this every morning at 9am (scheduled)

**The Proof**: These principles work in production.

---

## SLIDE 16: Takeaways - What You Can Apply

**Three Principles for Your Systems**:

**1. Dynamic Context Loading**
- Don't put everything in system prompt
- Let users define workflows (playbooks, SOPs, whatever)
- Use semantic search to inject relevant context

**2. Give Agents Programming Environments**
- Not just API calls - let them write code
- Enables composition and efficiency
- Sandbox carefully, but don't restrict unnecessarily

**3. Specialize Your Agents**
- One agent for orchestration
- Specialized agents for specialized tasks
- Use file system for state (not just memory)

**Final Thought**:

*"We spent a year learning these lessons. You don't have to. These are the architectural decisions that actually matter in production."*

*"What questions do you have?"*

---

## DELIVERY NOTES

### Energy & Pacing:

**Opening (Slides 1-3)**: Conversational, setting the stage (2 min)

**Principle 1 (Slides 4-6)**: Story-like, explaining the context problem (3 min)
- Start with the problem we faced
- Show how we solved it
- Implementation detail on playbook selection (the cool part)

**Principle 2 (Slides 7-9)**: Building energy, this is the "wow" moment (3 min)
- The efficiency problem was painful
- Code execution was controversial internally
- Show the actual code patterns

**Principle 3 (Slides 10-12)**: Bringing it together (2-3 min)
- Multi-agent is the culmination
- Persistence is the enabling factor
- Demo kickoff here

**Demo Results (Slides 13-15)**: Proof point (1-2 min)
- Show it working
- Emphasize the principles in action

**Closing (Slide 16)**: Clear, actionable (1 min)
- What they can take away
- Open for questions

---

## KEY IMPLEMENTATION DETAILS TO EMPHASIZE

When talking about the "cool" technical parts:

**Playbook Selection**:
- Sentence transformers for embedding
- Cosine similarity thresholds we learned through experimentation
- O(1) retrieval vs O(n) trial-and-error

**Code Execution Security**:
- Firecracker microVMs (OS-level isolation)
- Network proxy for all external calls
- Resource limits (CPU, memory, disk)

**Multi-Agent Delegation**:
- Main agent has broad context, delegates deep work
- Subagent with 1M token context for analysis
- Clean separation of concerns

**File-Based Persistence**:
- File system as source of truth
- Enables multi-day workflows
- Auditable execution trail

---

## NO MADE-UP METRICS

Instead of:
- ❌ "50% success → 95% success"
- ❌ "10x faster"
- ❌ "99% completion rate"

Use:
- ✅ "We hit this wall where..."
- ✅ "Early on, the agent still guessed wrong half the time"
- ✅ "Send 50 emails took 3-4 minutes, now 30 seconds"
- ✅ Real timing comparisons from actual usage

---

## FRAMING: LESSONS, NOT FEATURES

Throughout the talk, frame as:
- "We learned..."
- "The problem we faced..."
- "What worked..."
- "What we'd do differently..."

Not:
- "Kafka can..."
- "Our platform features..."
- "Why Kafka is better..."

Kafka is the **proof** that these principles work, not the product being sold.

---

This revision focuses on **teaching engineering principles** learned from building a production system, using Kafka as the case study. The audience leaves with actionable insights, not a sales pitch.

