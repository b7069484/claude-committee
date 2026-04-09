# Committee - Expert Panel Reviews for Claude Code

A Claude Code plugin that assembles multi-perspective expert committees to review any deliverable. Each committee member brings a distinct analytical lens anchored to a real company's standards, producing independent reports followed by a synthesized analysis with consensus, debate, and ranked recommendations.

**v2** introduces an Executive Assistant that orchestrates the entire review. The default path is fast: compose → reports → synthesis → actionable next steps. Opt into Interactive mode for preliminary questions, agenda preview, and topic-based debates with checkpoints.

## What It Does

Type `/committee` in Claude Code and get a structured review from a panel of experts like:

- **Head of UI/UX at Apple** - evaluates design against Apple HIG standards
- **Head of Marketing/Business at McKinsey** - analyzes positioning and go-to-market strategy
- **Skeptic End User** - reacts as a jaded consumer who needs convincing
- **Head of AI at Meta (FAIR)** - evaluates AI architecture for practical viability
- ...and 22 more pre-built personas

Each member produces a full report with executive assessment, key issues (backed by evidence and real-world benchmarks), actionable suggestions, and a concrete counter-proposal. The synthesized report surfaces consensus points, debates tensions between disagreeing members, and ranks recommendations by impact.

## Quick Start

### Install

```bash
claude plugin marketplace add b7069484/claude-committee
claude plugin install committee
```

### Use

```bash
# Show available collectives
/committee

# Auto-suggest the best committee for what you're working on
/committee suggest

# Run a specific collective
/committee review sales-marketing-review

# Assemble a custom panel
/committee custom apple-uiux-head, skeptic-end-user, mckinsey-strategy-head --add-members 3
```

## What's Included

### 26 Expert Personas

**Domain Experts (13):** Apple UX, Google Learning, McKinsey Strategy, Meta Monetization, Instagram Algorithm, Disney Pixar 3D, TED Presentations, Anthropic Claude Code, Google News/Media, Remotion three.js, Meta AI (FAIR), OpenAI Product, Google DeepMind Research

**Engineering & Security (5):** Stripe Engineering, Netflix Platform, Vercel DX, CrowdStrike Threat Intelligence, Google Security (Project Zero)

**Adversarial (4):** Skeptic End User, Skeptic End Client, Devil's Advocate, Regulatory Compliance

**Generalists & Operators (4):** Scrappy Startup Founder, Data Scientist/Metrics Lead, Accessibility Advocate, VC Partner

### 7 Pre-Built Collectives

| Collective | Members | Best For |
|-----------|---------|----------|
| Tech Product Review | 10 | Architecture, DX, security, scalability |
| Creative Content Review | 10 | Visual design, 3D, presentations, creative strategy |
| Sales & Marketing Review | 10 | Landing pages, pitch decks, pricing, positioning |
| Financial / Economic Review | 8 | Business models, pricing, projections |
| Market / User Focus Group | 8 | User reactions, adoption barriers, objections |
| Education & Learning Review | 8 | Courses, curricula, learning tools |
| AI / ML Product Review | 10 | AI products, model architecture, safety |

Each collective has pinned favorites plus dynamic slots that are filled with context-specific experts generated for your particular deliverable.

## How It Works

1. Invoke `/committee review [collective]` or `/committee suggest`
2. Executive Assistant proposes committee with tiered report assignments and blind spot detection
3. Choose: Standard review (fast, recommended) or Interactive session (debates)
4. Committee members generate independent reports in parallel
5. Executive Assistant synthesizes: consensus, tensions, evidence, blind spots
6. Two-path Next Steps: implement everything, or pick unanimous items and review tensions
7. Implementation Bridge: chain directly into a plan and execution

### Dynamic Member Generation

Collectives have "open slots" that get filled with experts generated specifically for your deliverable. Reviewing a healthcare product? The system generates a healthcare-specific expert. Working on a developer tool? It generates someone from the developer tools space. If a dynamic member performs well, promote them to your favorites library with `/committee promote [name]`.

### Parallel Mode

Parallel execution is now the default — each committee member runs as an independent subagent and cannot see each other's work, producing genuinely independent perspectives. The `--parallel` flag is kept for backward compatibility but is a no-op. Use `--sequential` to opt out and run all members in the main context (v1 behavior).

```bash
/committee suggest --deep
/committee suggest --sequential  # opt out of parallel
```

## Commands

| Command | Action |
|---------|--------|
| `/committee` | Show available collectives |
| `/committee review [id]` | Run a review with a specific collective |
| `/committee suggest` | Auto-recommend collective for current context |
| `/committee list` | Show all collectives and favorites |
| `/committee custom [ids]` | Assemble ad-hoc from favorites |
| `/committee add "[desc]"` | Create a new favorite member |
| `/committee promote [name]` | Save a dynamic member to favorites |
| `/committee remove [name]` | Delete a favorite |

### Flags

| Flag | Effect |
|------|--------|
| `--parallel` | Now default. Kept for backward compatibility. |
| `--sequential` | Opt out of parallel (run in main context like v1) |
| `--interactive` | Select interactive mode (debates, preliminary questions) |
| `--standard` | Select standard mode (default fast path) |
| `--implement` | Chain into implementation planning after review |
| `--quick` | Shorter reports (~150 words/member) |
| `--deep` | Detailed reports (~600+ words/member) |
| `--focus "[topic]"` | Focus analysis on specific aspect |
| `--add-members N` | Add extra dynamic slots |

## What's New in v2

- **Executive Assistant** — a meta agent orchestrates the entire review with crisp, structured interaction
- **Two modes** — Standard (fast, default) and Interactive (debates, opt-in)
- **Model tiering** — Haiku for probes, Sonnet for reports, preserving your main context
- **Blind spot detection** — domain coverage mapping identifies missing perspectives
- **Tiered report depth** — full, focused, or flag-only based on deliverable relevance
- **Two-path Next Steps** — unanimous items ready to act + contentious tensions surfaced for your decision
- **Implementation Bridge** — chain approved recommendations directly into implementation planning with `--implement`
- **Session checkpoints** — reviews are resumable and auditable via SESSION.json
- **Failure handling** — sub-agent timeouts, quorum enforcement, graceful degradation

## Research Basis

This plugin's design is backed by peer-reviewed research:

- **Multi-Expert Prompting** (EMNLP 2024): +8.69% truthfulness over baselines
- **Multiagent Debate** (ICML 2024): significant reduction in hallucinations
- **Solo Performance Prompting** (NAACL 2024): strong results in GPT-4-class models
- **USC PRISM Study** (2026): medium-detail personas (~75 tokens) optimal for differentiation

The ~120-word persona format was empirically tested and captures ~85% of maximum review value at 50% of the prompt cost of rich personas.

## Creating Your Own Members

Each member is a simple markdown file:

```markdown
---
name: Head of [Role] at [Company]
id: company-role-head
tags: [domain1, domain2, domain3]
archetype: domain-expert
---

You are the Head of [Role] at [Company]. [Core philosophy.]

**Your analytical lens:**
- [What you obsess over]
- [What you scrutinize]
- [What you'd never tolerate]

**You evaluate against:**
- [Real product/company benchmark]
- [Another benchmark]

**Your output requirement:**
- Don't just critique - propose what [Company] would ship instead
- Cite specific comparable products when identifying issues
```

Save to `skills/committee/favorites/` and add an entry to `favorites/_index.md`.

## License

MIT
