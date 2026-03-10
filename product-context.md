# Product Context

> **Scope**: Define what you're building, who it's for, and what success looks like — before writing code
> **Prerequisites**: None — do this first
> **Deliverables**: Answers to 5 product questions, recorded in README.md and CLAUDE.md
> **Estimated effort**: XS

Answer these questions before writing CLAUDE.md or planning the backlog.

---

## Five Questions

### 1. What problem does this solve?

The pain, not the features. What can't users do today, or what's painful about how they do it?

- **Business**: "Small merchants can't accept payments without a $2,000 POS terminal and a 3-week setup process."
- **Personal**: "My reading list is scattered across Goodreads, Kindle, and Apple Books — I can't sort by actual reading time or create cross-service 'to read' lists."

### 2. Who is it for?

Specific persona — behaviors, goals, and context. "Everyone" means no one. "Me" is valid for personal projects, but be specific about what kind of "me."

- **Business**: "Independent coffee shop owners who process 50-200 transactions/day and have no IT staff."
- **Personal**: "Me — an avid reader who uses 3 book services, cares about reading stats, and wants curated lists based on my actual habits, not a recommendation algorithm."

### 3. Why build it? What's different?

What exists today? Why isn't it good enough? What's your angle?

- **Business**: "Square/Stripe exist but charge 2.9% + $0.30. We charge 1.5% flat. No hardware required — works on any phone."
- **Personal**: "Goodreads' recommendations are generic. Kindle's are limited to Amazon. No tool lets me combine data across services and define my own reading rules."

### 4. How will we know it's working?

Success criteria — measurable or experiential. Without this, you can't prioritize.

- **Business**: "100 merchants processing $10K/month within 6 months. Churn under 5%."
- **Personal**: "I use the reading lists it generates every week. Creating a new list takes under 30 seconds. I stop manually curating."

### 5. What does it look like when it's done?

Scope boundary. What's v1? What's explicitly out of scope? This prevents endless feature creep.

- **Business**: "v1: payment processing + daily settlement reports. NOT inventory, NOT employee scheduling, NOT loyalty programs."
- **Personal**: "v1: import Goodreads shelves + Kindle highlights, build reading lists from rules, push back to Goodreads. NOT an e-reader, NOT social features."

---

## Where to Record This

The same answers go in three places, each serving a different reader:

**README.md** (humans — new contributors, future you):
```markdown
## The Problem
[Answer to question 1]

## The Solution
[Answers to questions 3 + 5 — what you're building and how it's different]
```

**CLAUDE.md** (the AI agent — reads this every conversation):
```markdown
## What [Project] Does
**[One-line pitch]**

### User Problem
- [Pain points from question 1]
- [Who it's for from question 2]

### Solution
[How it works — concrete examples, not abstract descriptions]
```

**Backlog README** (planning — drives version priorities):
```markdown
## Key Technical Decisions
[Informed by questions 3-5 — what you're building, what you're not, success criteria]
```

---

## If You Don't Have Answers Yet

If you're using this guide with a coding agent and haven't provided product context:

The agent should prompt you with these five questions before proceeding. Quick, rough answers are fine.

Add this to your CLAUDE.md so the agent knows to ask:

```markdown
## Product Context
If the answers below are missing or incomplete, ASK the user before proceeding with implementation.
```
