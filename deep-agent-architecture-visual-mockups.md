# Deep Agent Architecture - Visual Mockups for Figma

## Detailed Visual Specifications for Each Slide

---

## SLIDE 1: Title Slide

**Layout**: Centered, clean

```
┌─────────────────────────────────────────────────┐
│                                                 │
│         Deep Agent Architecture                 │
│                                                 │
│    Production-Grade AI Employees                │
│                                                 │
│                                                 │
│         Abhinav Tumu                           │
│         Brainbase Labs (YC W24)                │
│         November 21, 2025                       │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Large headline (60pt): "Deep Agent Architecture"
- Subhead (36pt): "Production-Grade AI Employees"
- Your name (24pt)
- Company + date (18pt)
- Use Brainbase brand colors
- Optional: Subtle background pattern or gradient

---

## SLIDE 2: The Agent Reliability Gap

**Layout**: Split screen comparison

```
┌─────────────────────────────────────────────────┐
│                                                 │
│    "Who here has built an agent that worked    │
│     great in demos but failed in production?"  │
│                                                 │
├──────────────────┬──────────────────────────────┤
│  DEMO AGENTS     │  PRODUCTION AGENTS           │
│                  │                              │
│  ❌ Single LLM   │  ✅ Persistent planning      │
│     call         │                              │
│                  │  ✅ Multi-agent              │
│  ❌ No state     │     orchestration            │
│                  │                              │
│  ❌ Manual       │  ✅ Self-correcting          │
│     supervision  │                              │
│                  │  ✅ 99%+ completion          │
│  ❌ Fails on     │                              │
│     edge cases   │                              │
│                  │                              │
└──────────────────┴──────────────────────────────┘
```

**Design Elements**:
- Question at top in quotes (24pt)
- Left column: Reddish/orange background (problems)
- Right column: Blue/green background (solutions)
- Use checkmarks (✅) and X marks (❌)
- Icons: broken robot vs working robot

---

## SLIDE 3: Brainbase Labs + Kafka

**Layout**: Intro + metrics grid

```
┌─────────────────────────────────────────────────┐
│  Brainbase Labs (YC W24)                        │
│  Building the AI Employee Platform              │
│                                                 │
├─────────────┬─────────────┬─────────────────────┤
│             │             │                     │
│   2.5M      │   5,000     │     99.2%          │
│  requests/  │   hours     │  completion        │
│   month     │  autonomous │     rate           │
│             │             │                     │
├─────────────┴─────────────┴─────────────────────┤
│                                                 │
│  Customer Examples:                             │
│  • Grant research automation                    │
│  • Product management (ClickUp + Slack)         │
│  • AI research agent (self-improving)           │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Top section: Company name + tagline
- Metrics: LARGE numbers (72pt) with small units below
- Use icons for each metric (chart, clock, checkmark)
- Bottom: Simple bullet points with small logos if available
- Color scheme: Your brand colors

---

## SLIDE 4: The Problem

**Layout**: Flow diagram

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  "How many of you give your agents the same    │
│         prompt every time?"                     │
│                                                 │
│  Traditional Approach:                          │
│                                                 │
│  User Request → Generic Agent → Guess workflow │
│                           ↓                     │
│                      50% success                │
│                                                 │
│  ────────────────────────────────────────────  │
│                                                 │
│  Kafka with Playbooks:                          │
│                                                 │
│  User Request → Kafka + Playbook → Proven path │
│                           ↓                     │
│                      95% success                │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Question at top in quotes
- Two flow diagrams stacked vertically
- Top diagram: Reddish tint (problem)
- Bottom diagram: Blue/green tint (solution)
- Use arrows (→) to show flow
- Success rates in large, bold text

---

## SLIDE 5: Playbooks - Executable Workflow Templates

**Layout**: Concept + example side-by-side

```
┌─────────────────────────────────────────────────┐
│  Playbooks: Executable Workflow Templates       │
│                                                 │
├─────────────────────┬───────────────────────────┤
│ Core Concept:       │  ClickUp Daily Standup:   │
│                     │                           │
│ • SOPs as code      │  Trigger:                 │
│ • Dynamic loading   │  "Daily standup report"   │
│ • Semantic matching │                           │
│                     │  Playbook Loads:          │
│ Why it matters:     │  1. Slack channels        │
│ ✓ Domain knowledge  │  2. ClickUp workspace     │
│ ✓ Reusable          │  3. Data to extract       │
│ ✓ Self-documenting  │  4. Output format         │
│ ✓ Version control   │  5. Where to post         │
│                     │                           │
│                     │  → 8-step autonomous exec │
└─────────────────────┴───────────────────────────┘
```

**Design Elements**:
- Title at top
- Left: Bullet points with icons
- Right: Numbered list showing actual workflow
- Use code/monospace font for the right side
- Arrow at bottom showing result
- Box/container styling for the example

---

## SLIDE 6: Dynamic Context Loading

**Layout**: Vertical flow diagram

```
┌─────────────────────────────────────────────────┐
│     Dynamic Context Loading                     │
│                                                 │
│   User: "Update the sprint board"              │
│                ↓                                │
│         [Vector Search]                         │
│                ↓                                │
│   Matches: "ClickUp Sprint Management"         │
│          (0.94 similarity)                      │
│                ↓                                │
│         Loads Context:                          │
│         • ClickUp credentials                   │
│         • Board IDs                             │
│         • Update patterns                       │
│                ↓                                │
│      Executes with full context                │
│                                                 │
│  Key: O(1) retrieval vs O(n) trial-and-error  │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Center-aligned vertical flow
- Each box connected with arrows
- Vector search box with distinct styling (maybe with search icon)
- "Loads Context" section with indent/bullets
- Bottom: Key insight in highlighted box
- Use colors to show progression down the flow

---

## SLIDE 7: The Tool Call Trap

**Layout**: Performance comparison

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  "Who here has seen their agent make the same  │
│      API call 50 times in a loop?"             │
│                                                 │
├─────────────────────┬───────────────────────────┤
│ TRADITIONAL         │  KAFKA                    │
│                     │                           │
│ Send 50 emails      │  Send 50 emails           │
│ = 50 tool calls     │  = 1 notebook cell        │
│ = 50 LLM roundtrips │  = 1 for loop             │
│ = 5 minutes         │  = 30 seconds             │
│                     │                           │
│ Cost: $0.50         │  Cost: $0.01              │
│ Risk: HIGH          │  Risk: LOW                │
│                     │                           │
└─────────────────────┴───────────────────────────┘
```

**Design Elements**:
- Question at top
- Split screen: left (red/orange), right (green/blue)
- Equation-style layout (= signs align)
- Bold the time difference (5 min vs 30 sec)
- Cost and risk at bottom in larger text
- Use icons: broken tool vs working loop

---

## SLIDE 8: Notebook Execution Environment

**Layout**: Architecture + code example

```
┌─────────────────────────────────────────────────┐
│   Notebook Execution Environment                │
│                                                 │
│ Architecture:                                   │
│ • Every thread = Persistent Jupyter notebook    │
│ • Pre-imported: SearchV2, WebCrawler, Agent    │
│ • State persists across cells                   │
│ • Write actual code, not just tool calls        │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ from integrations import AppFactory         │ │
│ │ factory = AppFactory()                      │ │
│ │ gmail = factory.app("gmail")                │ │
│ │ send = gmail.action("gmail-send-email")     │ │
│ │                                             │ │
│ │ # Write a loop, not 50 tool calls           │ │
│ │ for person in recipients:                   │ │
│ │     send.configure({                        │ │
│ │         "to": person.email,                 │ │
│ │         "subject": f"Hi {person.name}"      │ │
│ │     })                                      │ │
│ │     send.run()                              │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Bullet points explaining architecture
- Code block with dark background (VS Code style)
- Syntax highlighting (Python colors)
- Comment in code should be obvious
- Monospace font for code (Fira Code or SF Mono)

---

## SLIDE 9: The 1M Token Context Window

**Layout**: Concept + code + diagram

```
┌─────────────────────────────────────────────────┐
│  The 1M Token Context Window                    │
│                                                 │
│ Subagent with 1M context (GPT-5)               │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ from agent import Agent                     │ │
│ │                                             │ │
│ │ subagent = Agent(model="gpt-5")             │ │
│ │ analysis = subagent.run([                   │ │
│ │   {"type": "text",                          │ │
│ │    "text": "Analyze this proposal"},        │ │
│ │   {"type": "image_url",                     │ │
│ │    "image_url": {"url": "file.pdf"}}        │ │
│ │ ])                                          │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│  Main Agent ────→ Subagent (1M context)        │
│  Orchestration    Deep reasoning               │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Subtitle explaining capability
- Code block (dark theme)
- Bottom: Simple diagram showing relationship
- Use arrow to show delegation
- Two boxes: Main agent and Subagent with labels

---

## SLIDE 10: Agents All the Way Down

**Layout**: Architecture tree diagram

```
┌─────────────────────────────────────────────────┐
│        Agents All the Way Down                  │
│                                                 │
│              User Request                       │
│                   ↓                             │
│        Main Kafka Agent                         │
│           (Orchestrator)                        │
│                   │                             │
│      ┌────────────┼─────────────┬──────────┐   │
│      │            │             │          │   │
│  Subagent    MeetingBot    VoiceAgent  Scheduled│
│  (GPT-5)     (Zoom/Meet)   (Phone)     Agent   │
│                                                 │
│  Deep        Recording    Outbound    Cron-like│
│  reasoning   Transcript    calls     execution │
│                                                 │
│  Key: Specialized agents for specialized tasks │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Tree diagram with boxes and connecting lines
- Main agent at top (larger box, blue)
- Four sub-agents below (smaller boxes, different colors each)
- Labels under each sub-agent explaining function
- Key principle at bottom in highlighted box
- Use icons for each agent type if possible

---

## SLIDE 11: Persistent Planning + Execution

**Layout**: Process flow + example file

```
┌─────────────────────────────────────────────────┐
│    Persistent Planning + Execution              │
│                                                 │
│ How it works:                                   │
│ 1. Complex request → 2. Create todo.md →       │
│ 3. Execute & update → 4. Pause/resume →        │
│ 5. Self-correct                                 │
│                                                 │
│ ┌───────────────────────────────────────────┐   │
│ │ # Grant Research Workflow                 │   │
│ │                                           │   │
│ │ [x] Search grants (SearchV2)              │   │
│ │     Completed: 2025-11-20 14:23           │   │
│ │                                           │   │
│ │ [x] Crawl websites (WebCrawler)           │   │
│ │     Completed: 2025-11-20 14:45           │   │
│ │                                           │   │
│ │ [in_progress] Extract requirements        │   │
│ │     Started: 2025-11-20 15:02             │   │
│ │                                           │   │
│ │ [ ] Draft application                     │   │
│ └───────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Process flow with numbered steps and arrows
- Example file in monospace font with light background
- Checkboxes: ✓ (done), ⋯ (in progress), ☐ (todo)
- Timestamps in lighter gray
- Make the file look like actual markdown
- Add subtle shadow/border to file container

---

## SLIDE 12: Real-World Orchestration

**Layout**: Numbered flow with integrations

```
┌─────────────────────────────────────────────────┐
│    Real-World: ClickUp Daily Standup            │
│                                                 │
│  1. Main agent: Load playbook                   │
│     ↓                                           │
│  2. Notebook: Fetch Slack message [Slack icon]  │
│     ↓                                           │
│  3. Notebook: Extract ClickUp data [ClickUp]    │
│     ↓                                           │
│  4. Subagent: Analyze comments                  │
│     ↓                                           │
│  5. Notebook: Format report (loop)              │
│     ↓                                           │
│  6. Notebook: Post to Slack channels [Slack]    │
│     ↓                                           │
│  7. Main agent: Confirm completion ✓            │
│                                                 │
│  [THIS IS WHERE YOU START THE DEMO]            │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title: Customer example name
- Vertical flow with numbered steps
- Integration icons/logos next to relevant steps
- Arrows connecting each step
- Bottom: Call-out box "Start demo here"
- Use different colors for different components:
  - Main agent: Blue
  - Notebook: Green
  - Subagent: Purple
  - Integrations: Their brand colors

---

## SLIDE 13: Why This Architecture Works

**Layout**: Four-quadrant grid

```
┌─────────────────────────────────────────────────┐
│      Why This Architecture Works                │
│                                                 │
├────────────────────┬────────────────────────────┤
│ 1. Persistence     │ 2. Composability           │
│                    │                            │
│ • File system      │ • Notebook + subagents     │
│ • Plans survive    │ • Mix code & LLM           │
│ • Resume workflows │ • Infinite tools           │
│                    │                            │
├────────────────────┼────────────────────────────┤
│ 3. Guardrails      │ 4. Reliability             │
│                    │                            │
│ • Playbooks        │ • Multi-provider           │
│ • Validation       │ • Caching                  │
│ • Error handling   │ • 99%+ completion          │
│                    │                            │
└────────────────────┴────────────────────────────┘
```

**Design Elements**:
- Title at top
- Four equal quadrants
- Number + principle name in bold
- 3 bullet points per quadrant
- Use distinct color for each quadrant
- Icons for each principle:
  - Persistence: Save/disk icon
  - Composability: Puzzle pieces
  - Guardrails: Shield
  - Reliability: Uptime/chart

---

## SLIDE 14: Demo Results

**Layout**: Screenshot + highlights

```
┌─────────────────────────────────────────────────┐
│           Demo Results                          │
│                                                 │
│  [SCREENSHOT OF SLACK CHANNELS WITH REPORT]    │
│                                                 │
│  ────────────────────────────────────────────  │
│                                                 │
│  ✓ Autonomous execution (no supervision)        │
│  ✓ Correct data extraction                      │
│  ✓ Proper formatting                            │
│  ✓ Posted to correct channels                   │
│  ✓ Completed in X seconds                       │
│                                                 │
│  "This ran while I was talking.                │
│   No supervision. No failures."                │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Large screenshot in center (actual Slack screenshot)
- Bottom: Checklist with green checkmarks
- Quote at very bottom in italic/highlighted
- Use arrows or callouts on screenshot if needed
- Make the time (X seconds) bold

---

## SLIDE 15: Takeaways

**Layout**: Three columns

```
┌─────────────────────────────────────────────────┐
│     Takeaways for Your Agents                   │
│                                                 │
├───────────────┬──────────────┬──────────────────┤
│               │              │                  │
│ 1. Add        │ 2. Enable    │ 3. Build         │
│ Persistence   │ Code         │ Orchestration    │
│               │ Execution    │                  │
│ • File-based  │ • Not just   │ • Specialized    │
│   planning    │   API calls  │   subagents      │
│ • State       │ • Let agents │ • Hierarchical   │
│   survives    │   write loops│   reasoning      │
│ • Execution   │ • Real       │ • Composable     │
│   logs        │   environment│   capabilities   │
│               │              │                  │
└───────────────┴──────────────┴──────────────────┘
│                                                 │
│  "What's the most complex workflow              │
│      you've tried to automate?"                 │
│                                                 │
│         [Your contact / website]                │
└─────────────────────────────────────────────────┘
```

**Design Elements**:
- Title at top
- Three equal columns
- Number + principle in large text
- 3 bullets per column
- Each column different color
- Bottom: Question in quotes
- Very bottom: Your contact info
- Icons for each principle:
  - Persistence: Floppy disk
  - Code: Code brackets <>
  - Orchestration: Network nodes

---

## GENERAL DESIGN GUIDELINES

### Colors:
- **Primary brand**: Use throughout
- **Success/Good**: Green (#10B981)
- **Problem/Bad**: Red/Orange (#EF4444)
- **Code blocks**: Dark (#1E1E1E) with VS Code colors
- **Backgrounds**: Light gray (#F9FAFB) or white

### Typography:
- **Headlines**: 48-60pt, bold
- **Subheads**: 32-40pt, medium
- **Body text**: 24-28pt, regular
- **Code**: 20-24pt, monospace (Fira Code or SF Mono)
- **Captions**: 18-20pt, light

### Spacing:
- Generous padding (60px minimum)
- Consistent margins
- White space between elements
- Readable from 20+ feet away

### Consistency:
- Use same icon style throughout
- Consistent arrow style
- Same code block styling
- Matching colors for similar concepts

### Accessibility:
- High contrast (4.5:1 minimum)
- Large text size
- Clear visual hierarchy
- Icons + text labels

---

## EXPORT SETTINGS

**For presentation**:
- 16:9 aspect ratio (1920x1080)
- Export as PDF for compatibility
- Backup: Export individual PNGs

**For sharing**:
- Export as PDF
- Include slide notes if needed

---

## ASSETS NEEDED

**Icons/Illustrations**:
- Checkmarks and X marks
- Integration logos (Slack, ClickUp, Gmail)
- Agent/robot illustrations
- Code/notebook icons
- Network/orchestration diagrams

**Screenshots**:
- Slack channels with standup report
- Optional: Kafka interface showing execution
- Optional: Todo.md file in action

**Logos**:
- Brainbase logo
- Customer logos (if allowed)
- YC logo

---

These mockups give you exact specifications for building each slide in Figma. Adjust colors to match your brand, but keep the layouts and information hierarchy as described for maximum impact.

