# Deep Agent Architecture - Speaker Notes

## Quick Reference for Delivery

**Total Time**: 10-12 minutes  
**Style**: Technical but accessible, high energy at hook/demo, conversational at end

---

## TIMING CHECKPOINTS

- **Min 0-1**: Hook + audience question (Slides 1-2)
- **Min 1-2**: Brainbase intro + metrics (Slide 3)
- **Min 2-5**: Playbooks concept (Slides 4-6)
- **Min 5-8**: Notebook execution (Slides 7-9)
- **Min 8-10**: Multi-agent orchestration + DEMO KICKOFF (Slides 10-12)
- **Min 10-11**: Architecture principles (Slide 13)
- **Min 11-12**: Demo results (Slide 14)
- **Min 12**: Takeaways + Q&A (Slide 15)

---

## AUDIENCE QUESTIONS (4 Total)

1. **Slide 1-2**: "Who built an agent that worked in demo but failed in production?"
2. **Slide 4**: "How many give agents the same prompt every time?"
3. **Slide 7**: "Who's seen their agent make the same call 50 times?"
4. **Slide 15**: "What's the most complex workflow you've tried to automate?"

---

## KEY TRANSITIONS

**After Slide 3 → Playbooks**:  
"Here's the thing nobody talks about: Your agent can have access to 2000 integrations, but if it doesn't know WHEN and HOW to use them, it's useless."

**After Slide 6 → Notebooks**:  
"What if your agent could write actual code instead of just calling predefined functions? Not as a workaround, but as the primary execution model?"

**After Slide 9 → Multi-Agent**:  
"Now here's where it gets really interesting. Once you have this execution environment, you can start composing agents."

**Slide 12 → DEMO**:  
"Enough theory. Let me show you this working. I'm going to kick off a workflow right now - we'll check results in a minute."

---

## CORE MESSAGES (Don't Forget These)

### Playbooks (Slides 4-6):
- Dynamic context loading via semantic search
- O(1) knowledge retrieval vs O(n) trial-and-error
- "This is how you scale from 100 to 10,000 requests/day"

### Notebooks (Slides 7-9):
- 50 tool calls = 5 min | 1 for loop = 30 sec
- Full programming environment = infinite composability
- Subagent with 1M context for deep analysis

### Multi-Agent (Slides 10-12):
- Specialized agents for specialized tasks
- File-based persistent planning (todo.md)
- Workflows can take hours/days and resume

### Why It Works (Slide 13):
- Persistence (file system backend)
- Composability (notebook + subagents)
- Guardrails (playbooks + validation)
- Reliability (99%+ completion rate)

---

## DEMO CHECKLIST

**Before starting**:
- [ ] Kafka interface open and ready
- [ ] Slack channels open in another tab
- [ ] Workflow trigger message ready to send

**During Slide 12**:
- [ ] Send trigger message
- [ ] Show playbook loading (quick glimpse)
- [ ] Show first 1-2 steps executing
- [ ] Continue to Slide 13 while it runs

**During Slide 14**:
- [ ] Switch to Slack tabs
- [ ] Show posted standup report in both channels
- [ ] Highlight: autonomous, correct data, proper formatting
- [ ] "This ran while I was talking. No supervision. No failures."

---

## ENERGY LEVELS

**HIGH**: Slides 1-2, 12, 14 (hook, demo kickoff, results)  
**MODERATE**: Slides 5-11 (technical explanations)  
**CONVERSATIONAL**: Slides 3, 13, 15 (intro, principles, takeaways)

---

## IF RUNNING LONG (Cut These)

- Slide 6 details (just mention "semantic search over playbooks")
- Slide 9 code example (just describe the pattern)
- Slide 13 (merge into Slide 14 quickly)

---

## IF RUNNING SHORT (Add These)

- Extra customer examples on Slide 3
- More detail on subagent use cases (Slide 9)
- Show execution logs during demo (Slide 14)
- Take more questions at end

---

## OPENING (MEMORIZE THIS)

"Raise your hand if you've built an AI agent that worked perfectly in your demo but completely fell apart when a real user touched it.

[pause for hands, make eye contact]

Yeah. Me too. That's because most of what we call 'agents' today are just chatbots with API access. Today I want to show you what production-grade agents actually look like."

---

## CLOSING (MEMORIZE THIS)

"So, three things: Add persistence. Enable code execution. Build orchestration. 

These aren't minor improvements - they're the difference between a demo and a production system.

[pause]

What questions do you have?"

---

## KEY STATS TO EMPHASIZE

- **2000+ integrations** (scale)
- **99.X% completion rate** (reliability)
- **50 tool calls vs 1 for loop** (efficiency)
- **1M token context** (capability)
- **XX% cost reduction** (optimization)

---

## BACKUP ANSWERS (If Demo Fails)

"Looks like the demo is taking longer than expected - that's the reality of production systems. Let me show you screenshots of a previous run..."

[Have backup screenshots ready]

---

## WHAT MAKES THIS FRONTIER

**Most agents today** (2024 patterns):
- ❌ No state persistence
- ❌ Only pre-defined tool calls
- ❌ Single monolithic agent
- ❌ No production guardrails

**Kafka** (late 2025):
- ✅ File-based persistent planning
- ✅ Full code execution environment
- ✅ Multi-agent orchestration
- ✅ Playbooks as harnesses

This is state-of-the-art, not chatbot wrappers.

---

## POST-TALK

**Stay for 10-15 minutes** to chat with interested engineers.

**Common follow-up questions**:
- "How do you handle security?" → Isolated sandboxes
- "What about cost?" → Caching reduces by ~40%
- "Can users create playbooks?" → Yes, natural language
- "How does this compare to LangChain?" → Framework vs platform

**Collect contacts** from people who want to learn more.

---

## REMEMBER

✓ Speak to the "why", not just the "what"  
✓ Use "you" language to connect with audience  
✓ Pause after questions for hands to go up  
✓ Make eye contact across the room  
✓ Vary your pace - slow for key points, faster for examples  
✓ Show enthusiasm - you've built something frontier  

**You've got this! 🚀**

