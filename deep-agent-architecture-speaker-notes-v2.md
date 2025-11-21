# Speaker Notes - Design Principles for Production Agents

## Quick Reference for Delivery (Revised)

**Total Time**: 10-12 minutes  
**Framing**: Engineering lessons learned, not product pitch  
**Style**: Story-like but technically detailed

---

## CORE FRAMING

**You are teaching, not selling.**

Kafka = case study proving these principles work  
You = engineer sharing hard-earned lessons  
Audience = peers who can learn from your mistakes

---

## TIMING CHECKPOINTS

- **Min 0-2**: Hook + context (Slides 1-3)
- **Min 2-5**: Principle 1 - Executable context (Slides 4-6)
- **Min 5-8**: Principle 2 - Code execution (Slides 7-9)
- **Min 8-10**: Principle 3 - Multi-agent (Slides 10-12)
- **Min 10-11**: Demo kickoff + composition (Slides 13-14)
- **Min 11-12**: Demo results + takeaways (Slides 15-16)

---

## THREE PRINCIPLES (Your Core Message)

### 1. Dynamic Context Loading (Playbooks)
**Problem**: One prompt can't serve everyone's unique workflows  
**Solution**: User-created playbooks + semantic search  
**Cool detail**: Sentence transformers, cosine similarity thresholds

### 2. Code Execution Environment
**Problem**: Tool calls are too slow for bulk operations  
**Solution**: Full Python environment, not just tool schemas  
**Cool detail**: Firecracker VMs for isolation, composability

### 3. Multi-Agent Specialization + Persistence
**Problem**: One agent can't handle everything well  
**Solution**: Orchestrator + specialists + file-based state  
**Cool detail**: todo.md enables multi-day workflows

---

## KEY TRANSITIONS

**After Slide 3 → Principle 1**:  
"Let me tell you about the first problem we hit. Context."

**After Slide 6 → Principle 2**:  
"Playbooks solved the context problem. But they didn't solve the next problem: efficiency."

**After Slide 9 → Principle 3**:  
"Code execution solved efficiency. But complex workflows need more than one agent."

**Slide 13 → Demo**:  
"Let me show you these principles in action. This is a real workflow a customer uses every day."

---

## OPENING (MEMORIZE THIS)

"A year ago, we started building what we thought would be a simple agent. The plan was straightforward: good LLM, decent tools, some prompting. Ship it.

[pause]

Turns out, production changes everything. Today I want to share the hard lessons we learned - the architectural decisions that actually matter when users depend on your agent working every single day."

---

## CLOSING (MEMORIZE THIS)

"Three principles: Dynamic context loading. Code execution environments. Multi-agent specialization.

We spent a year learning these lessons. You don't have to. These are the architectural decisions that actually matter in production.

What questions do you have?"

---

## DEMO FLOW

**During Slide 13** (~10 min in):
1. Say: "Let me show you these principles in action."
2. Send trigger to Kafka: "Generate daily standup report"
3. Show playbook loading (quick glimpse)
4. Show first step or two executing
5. Say: "This is running now. While it finishes..."
6. Continue to Slide 14

**During Slide 15** (~11 min in):
1. Switch to Slack tabs
2. Show posted standup report in both channels
3. Point out: Autonomous, correct data, proper formatting
4. Say: "This ran while I was talking. These principles work in production."

---

## TECHNICAL DETAILS TO EMPHASIZE

When showing implementation details (the "cool" parts):

**Playbook Selection (Slide 6)**:
- Sentence transformers for embedding
- Learned thresholds: >0.85 auto-load, 0.70-0.85 confirm, <0.70 skip
- Why: O(1) retrieval, works with fuzzy matches, scales

**Code Execution (Slide 9)**:
- Firecracker microVMs (OS isolation)
- Network proxy, resource limits
- The unexpected benefit: infinite composability

**Multi-Agent (Slide 11-12)**:
- Main agent: broad, shallow (orchestration)
- Subagent: narrow, deep (1M context)
- File system as source of truth (not memory)

---

## AVOID THESE PHRASES

❌ "Kafka can..."  
❌ "Our platform features..."  
❌ "50% success → 95% success"  
❌ "10x faster" (unless you have real numbers)  
❌ "Demo vs production agents"

✅ "We learned..."  
✅ "The problem we faced..."  
✅ "What worked for us..."  
✅ "Send 50 emails took 3-4 minutes, now 30 seconds" (real timing)  
✅ "Early on, the agent guessed wrong half the time"

---

## ENERGY LEVELS

**CONVERSATIONAL** (Slides 1-3): Setting the stage, relatable  
**TEACHING** (Slides 4-12): Clear, detailed, story-like  
**PROOF** (Slides 13-15): Showing it works  
**ACTIONABLE** (Slide 16): What they can take away

---

## IF RUNNING LONG

Skip or compress:
- Slide 14 (composition) - merge into demo
- Some code examples - just describe pattern

## IF RUNNING SHORT

Add:
- More implementation details
- Additional real workflow examples
- Extended Q&A

---

## BACKUP IF DEMO FAILS

"Looks like this is taking longer than expected - let me show you a previous run..."

[Have screenshots ready]

Still emphasize: "This is a real workflow that runs every morning. The principles enable this level of autonomy."

---

## KEY MESSAGES TO HAMMER HOME

**On Playbooks**:
"We inverted the problem. Instead of putting all context in the system prompt, we let users define workflows and load them dynamically."

**On Code Execution**:
"We're artificially limiting agents by forcing them to use tool schemas. Give them a programming environment instead."

**On Multi-Agent**:
"File system as source of truth, not memory. That's what enables multi-day workflows."

**Overall**:
"These aren't features to bolt on later. They're architectural decisions from day one."

---

## WHAT MAKES THIS FRONTIER

Not what others have in 2025:
- Most agents: Static prompts for everyone
- Most agents: Tool calling only (no code execution)
- Most agents: Single monolithic agent
- Most agents: State lives in memory

What we learned works:
- Dynamic context loading per user
- Full programming environment
- Specialized agents + orchestration
- File-based persistent state

---

## POST-TALK

Stay 10-15 minutes for discussions.

Common questions you might get:
- "How do you handle X edge case?" → Be honest about trade-offs
- "What about cost?" → Real numbers if you have them, otherwise honest about what matters
- "Can you share code?" → Point to principles they can implement

**Remember**: You're teaching peers, not pitching. Be honest about what worked, what didn't, what you'd do differently.

---

## FINAL REMINDER

**You're not selling Kafka.**  
**You're teaching principles learned from building Kafka.**

The audience should leave thinking:
- "I should try dynamic context loading in my system"
- "Code execution makes sense for my use case"  
- "I need to rethink my agent architecture"

Not:
- "I should buy Kafka"
- "Kafka is better than what I have"

**You've got this. Teach them what you learned.** 🚀

