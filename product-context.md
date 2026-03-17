# Product Context

> **Scope**: Define what you're building, who it's for, and what success looks like — before writing code
> **Prerequisites**: None — do this first
> **Deliverables**: Answers to 5 product questions, proto-personas, core use cases — recorded in README.md, CLAUDE.md, and `docs/user-flows.md`
> **Estimated effort**: S

Answer these five questions before writing CLAUDE.md or planning the backlog. Every later decision — entities, pages, tests, cuts — traces back to these answers.

---

## Five Questions

### 1. What problem does this solve?

The pain, not the features. What can't users do today, or what's painful about how they do it?

- **Business**: "Small merchants can't accept payments without a $2,000 POS terminal and a 3-week setup process."
- **Personal**: "My reading list is scattered across Goodreads, Kindle, and Apple Books — I can't sort by actual reading time or create cross-service 'to read' lists."

**How to answer well**: Describe the user's world *without your product in it*. If your answer includes your feature set, you're describing the solution, not the problem.

**Anti-pattern**: "Users need a dashboard that shows..." — that's a feature. The problem is what makes them *want* a dashboard.

### 2. Who is it for?

Specific persona — behaviors, goals, and context. "Everyone" means no one. "Me" is valid for personal projects, but be specific about what kind of "me."

- **Business**: "Independent coffee shop owners who process 50-200 transactions/day and have no IT staff."
- **Personal**: "Me — an avid reader who uses 3 book services, cares about reading stats, and wants curated lists based on my actual habits, not a recommendation algorithm."

**How to answer well**: Describe behaviors, not demographics. "Checks the app every Monday for 5 minutes" tells you more than "35-year-old professional."

**Anti-pattern**: "Tech-savvy professionals who value efficiency" — too broad to constrain any decision. A useful persona tells you what to cut, not just what to build.

### 3. Why build it? What's different?

What exists today? Why isn't it good enough? What's your angle?

- **Business**: "Square/Stripe exist but charge 2.9% + $0.30. We charge 1.5% flat. No hardware required — works on any phone."
- **Personal**: "Goodreads' recommendations are generic. Kindle's are limited to Amazon. No tool lets me combine data across services and define my own reading rules."

**How to answer well**: Name the competitors (even if they're spreadsheets or manual processes). Explain what's missing. Your differentiator should be specific enough that someone could disagree with it.

**Anti-pattern**: "Nothing like this exists" — something always exists, even if it's a manual workaround. If you can't name what people use today, you don't understand the problem space.

### 4. How will we know it's working?

Success criteria — measurable or experiential. Without this, you can't prioritize.

- **Business**: "100 merchants processing $10K/month within 6 months. Churn under 5%."
- **Personal**: "I use the reading lists it generates every week. Creating a new list takes under 30 seconds. I stop manually curating."

**How to answer well**: Pick criteria you can actually check. "It feels good to use" isn't measurable. "I open it every Monday and finish my workflow in under 2 minutes" is. For personal projects, "I actually use it" is the ultimate test — most side projects die from disuse, not bugs.

**Anti-pattern**: Unmeasurable goals like "users are happy" or "the code is clean." Those are means, not ends.

### 5. What does it look like when it's done?

Scope boundary. What's v1? What's explicitly out of scope? This prevents endless feature creep.

- **Business**: "v1: payment processing + daily settlement reports. NOT inventory, NOT employee scheduling, NOT loyalty programs."
- **Personal**: "v1: import Goodreads shelves + Kindle highlights, build reading lists from rules, push back to Goodreads. NOT an e-reader, NOT social features."

**How to answer well**: The "NOT" list is more important than the feature list. Every feature you explicitly exclude is a decision you won't have to make again. When someone (including you at 2am) suggests adding social features, the answer is already written down.

**Anti-pattern**: An open-ended feature list with no boundaries. If everything is in scope, nothing gets finished.

---

## Proto-Personas

Turn your Q2 answer into structured personas. Proto-personas are hypothesis-based — no user research required, just deliberate thinking about behaviors.

### Template

Define 2-3 personas maximum:

| Field | What to write |
|---|---|
| **Name** | A memorable label (not a real name) — "The Weekly Curator", "The Data Hoarder" |
| **Context** | Their situation, tools they use, how often they engage |
| **Goal** | What they're trying to accomplish (in their words, not yours) |
| **Key behavior** | The specific action pattern — "builds a playlist every Sunday evening" |
| **Pain point** | What stops them or slows them down today |
| **Not this person** | What distinguishes them from adjacent personas who are NOT your user |

### Example

**The Weekly Curator** — Primary persona
- **Context**: Uses Spotify daily, Last.fm for tracking. Manages 15+ playlists, updates them weekly.
- **Goal**: "Keep my playlists fresh without spending an hour every week doing it manually."
- **Key behavior**: Opens the app Sunday evening, reviews what's changed, runs playlist rules, pushes to Spotify.
- **Pain point**: Cross-referencing Last.fm play counts with Spotify likes is manual — export CSV, open spreadsheet, filter, re-import.
- **Not this person**: Not a casual listener who uses Discover Weekly and is happy with it.

**The Passive Archivist** — Anti-persona (who you're NOT building for)
- Wants automatic everything. No rules, no configuration — just "make it work."
- Building for this person would mean replacing Spotify's algorithm, which is not the goal.

### Why anti-personas matter

Every feature request gets tested: "Does the Weekly Curator need this?" If the answer is "no, but the Passive Archivist would love it" — cut it.

### Alternative: Jobs to Be Done

If personas feel artificial, use JTBD format instead:

> "When I [situation], I want to [motivation], so I can [expected outcome]."

- "When I'm building my weekly playlist, I want to see which liked tracks I've been playing most this month, so I can keep the list fresh without manually checking play counts."
- "When I switch from Spotify to Apple Music, I want to bring my curated playlists with me, so I don't lose years of curation work."

JTBD focuses on the *job* rather than the *person*. More natural for solo projects where every persona is a variant of you.

---

## Use Cases & User Stories

Personas tell you who. Use cases tell you what they do. Write 5-10 core use cases that cover the 80% of what your primary persona will actually do with the app.

**Organize by workflow, not feature.** Group use cases by the user's real-world ritual — the sequence they actually follow. A monthly reconciliation app groups by "upload → review → settle up → budget" because that's the order people use it. This ordering carries through to IA (pages, CLI commands, or both follow workflow order), stories (import before review), and journeys.

### Use case format

| Field | Content |
|---|---|
| **Title** | Verb phrase — "Build a playlist from listening rules" |
| **Actor** | Which persona |
| **Precondition** | What must be true before this starts |
| **Flow** | Numbered steps from the user's perspective (not system internals) |
| **Postcondition** | What's true when it's done |
| **Acceptance criteria** | Given/When/Then — specific enough to implement, describes outcomes not mechanics |

### Example

**Build a "Current Obsessions" playlist**
- **Actor**: The Weekly Curator
- **Precondition**: Spotify likes imported, Last.fm play history imported
- **Flow**:
  1. Open workflow editor
  2. Define filter: liked tracks with 8+ plays in last 30 days
  3. Sort by play count descending, take top 20
  4. Preview the result
  5. Push to Spotify as "Current Obsessions" playlist
- **Postcondition**: Spotify playlist updated with 20 tracks matching the criteria
- **Acceptance criteria**:
  - Given a filter for liked tracks with 8+ plays in 30 days, when I preview, then only matching tracks appear
  - Given a previewed playlist, when I push to Spotify, then the Spotify playlist matches the preview exactly
  - Given play counts change over a week, when I re-run the workflow, then the result reflects current data

### User story shorthand

For backlog planning, compress use cases into stories:

> "As a [persona], I want [capability] so that [benefit]."

- "As the Weekly Curator, I want to filter my liked tracks by recent play count so that I can build playlists from what I'm actually listening to."
- "As the Weekly Curator, I want to push a generated playlist to Spotify so that I can listen to it on my phone without manual steps."

Each story needs Given/When/Then acceptance criteria — 2-4 statements that define "done" and double as test cases.

### How many to write

- **5-10 core use cases** covering the primary persona's main workflows
- Cover the 80% — the paths they'll use weekly. Edge cases emerge during development.

**User journeys** (written when building the interface — web, CLI, or both) validate that stories compose into coherent sessions. A journey walks through 5-8 concrete steps across multiple surfaces — proving that individual stories form a realistic session. This catches gaps a story-by-story review misses: "wait, how does the user get from import to review?"

---

## Product Anti-Patterns

Mistakes that no amount of clean architecture can fix:

**Building for the API, not the user** — "What can our APIs do?" leads to features nobody asked for. Start from "What is the user trying to accomplish?" If a feature doesn't trace to a use case, cut it.

**Features as problem statement** — "Users need a real-time dashboard" is a solution. The problem is "users don't know if their import is running or stuck." The fix might be a dashboard, a progress bar, or a notification.

**Scope creep through adjacency** — "While we're building playlist export, let's add collaborative playlists." Test every addition against your primary persona and scope boundary (question 5).

**Designing for the demo, not the daily** — AI-generated UIs optimize for the screenshot: happy path, perfect data. Real usage means empty states, long text, slow connections, and the 50th time opening the same screen.

---

## From Context to Code

Product answers are architectural inputs:

| Product answer | Drives... |
|---|---|
| **Personas** | Domain entities, use case signatures, API endpoint design |
| **Pain points** | Feature priority — fix the biggest pain first |
| **Use cases** | Backlog stories, acceptance criteria, E2E test scenarios |
| **Success criteria** | What to measure, what to test most thoroughly |
| **Scope boundary** | What NOT to build — the most valuable architectural constraint |
| **Anti-persona** | Feature requests to reject |

**The bridge**:
- Personas define the domain vocabulary → [Domain Modeling](domain-modeling.md)
- Use cases become backlog stories → [Backlog Planning](backlog-planning.md)
- Pain points drive visual identity decisions → [Design Identity](react-design-identity.md)
- Acceptance criteria become test cases → [Testing Strategy](testing-strategy.md)
- Behavioral spec stays connected to implementation → [Spec-Grounded Development](spec-enforcement.md)

---

## Where to Record This

The same answers go in multiple places, each serving a different reader:

**README.md** (humans — new contributors, future you):
```markdown
## The Problem
[Answer to question 1 — the pain, in the user's words]

## The Solution
[Answers to questions 3 + 5 — what you're building, how it's different, what's out of scope]
```

**CLAUDE.md** (the AI agent — reads this every conversation):
```markdown
## What [Project] Does
**[One-line pitch]**

### User Problem
- [Pain points from question 1]
- [Primary persona behavior from question 2]

### Primary Persona
[Name] — [one-line description of context and goal]

### Core Use Cases
- [Top 3-5 use cases as bullet points — what the user actually does]

### Solution
[How it works — concrete examples, not abstract descriptions]
```

**`docs/user-flows.md`** (behavioral spec — sits between domain knowledge and implementation):
```markdown
# User Flows

> See also: [design system](.claude/rules/web-design-system.md), [CLI patterns](.claude/rules/cli-patterns.md), [domain model](architecture/...)

## Persona
[Full proto-persona. Focus on details that drive design decisions:
"Tech comfort is high" → no onboarding tutorial.
"Primary question: How much do I owe?" → dashboard leads with settlement status.]

## Workflow Phases
[The real-world ritual the app supports. This ordering drives everything below.]
1. **[Phase]** — [what happens, frequency]
2. **[Phase]** — [what happens]

## Information Architecture
[Surfaces (pages, CLI commands, or both) ordered by workflow phase. Include ship version.]
| Surface | Purpose | Workflow Phase | Ships in |
|---|---|---|---|

## User Stories
[Grouped by workflow phase. GWT acceptance criteria. Version annotations on future stories.]
### [Phase 1] Stories
- **US-PREFIX-1: [Title]**
  - Given [precondition], when [action], then [outcome]

## User Journeys
[End-to-end sequences stitching stories into realistic sessions.
These validate composition — if a journey reveals a missing transition, add a story.]
### Journey 1: [Name]
| Step | Surface | Action | Result |
|---|---|---|---|
```

**Backlog README** (planning — drives version priorities):
```markdown
## Key Technical Decisions
[Informed by questions 3-5 — what you're building, what you're not, success criteria]
```

---

## If You Don't Have Answers Yet

If you're using this guide with a coding agent and haven't provided product context:

The agent should prompt you with the five questions and persona template before proceeding. Rough answers are fine — you can refine later.

Add this to your CLAUDE.md so the agent knows to ask:

```markdown
## Product Context
If the answers below are missing or incomplete, ASK the user before proceeding with implementation.
```
