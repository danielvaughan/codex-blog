---
title: "Fast Books: What AI-Enabled Publishing Can Learn from Fast Fashion and Fast Software"
description: "In 1989, Zara opened its first store outside Spain. Its secret weapon was not design flair but supply-chain speed: a dress could move from sketch to shop."
date: 2026-05-08T00:00:00+00:00
last_modified_at: 2026-08-07T00:19:18+01:00
author: Daniel Vaughan
category: opinion
tags: [ai, publishing, fast-fashion, software-development, agentic-engineering]
parent: "Articles"
nav_order: 932
type: Technical Article
timestamp: 2026-05-08T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-08-fast-books-fast-software-fast-fashion-ai-publishing-parallels"
---
![Sketchnote diagram for: Fast Books: What AI-Enabled Publishing Can Learn from Fast Fashion and Fast Software](/sketchnotes/articles/2026-05-08-fast-books-fast-software-fast-fashion-ai-publishing-parallels.png)


# Fast Books: What AI-Enabled Publishing Can Learn from Fast Fashion and Fast Software

In 1989, Zara opened its first store outside Spain. Its secret weapon was not design flair but supply-chain speed: a dress could move from sketch to shop floor in two weeks, while competitors took six months. Three decades later, Shein compressed that cycle further — ten days from trend-spotted-on-TikTok to garment-in-warehouse, with 6,000 new designs added every single day.

The fashion industry calls this acceleration "fast fashion." The technology industry is living through its own version right now — twice over. **Fast software** ships AI-generated code at a pace that would have been inconceivable in 2023. And a quieter revolution is unfolding on platforms like Leanpub and Amazon KDP: **fast books**, where AI-assisted authoring tools let a single writer produce, update, and ship a technical book in weeks rather than years.

This article traces the structural parallels between all three, argues that fast books are a legitimate and valuable publishing model when done well, and warns that the same pathologies that gave us disposable clothing and mounting technical debt are already visible in publishing.

---

## The Pattern: Speed Democratises, Then Degrades

Every "fast" revolution follows the same arc:

| Phase | Fast Fashion | Fast Software | Fast Books |
|-------|-------------|---------------|------------|
| **1. Technology unlocks speed** | Sewing machine (1846), RFID inventory, real-time sales analytics | GitHub Copilot (2022), Claude Code, Codex CLI, vibe coding | ChatGPT (2022), Claude, agentic writing workflows, Leanpub in-progress publishing |
| **2. Incumbents lose their moat** | Couture houses lose exclusivity to H&M and Zara | Enterprise software vendors lose to solo devs shipping faster | Traditional publishers lose 18-month lead times to indie authors shipping in weeks |
| **3. Democratisation** | Trend-driven clothing accessible at every price point | Anyone can build and ship a SaaS product | Anyone can write and publish a technical book |
| **4. Volume explodes** | Shein lists 600,000 items at any given time | Average daily CI workflow runs up 59% year-on-year (CircleCI 2026 report) | 4 million books published in the US in 2025 — up 32% on 2024, with 3 million self-published |
| **5. Quality crisis** | Polyester garments worn once and landfilled; toxic dyes; 75-hour factory weeks | PR sizes up 150%, bug counts up 9%; 63% of devs spend more time debugging AI code than writing it manually would take | AI-generated mushroom guides list poisonous varieties as safe; Amazon limits self-publishing to 3 books per day |
| **6. Counter-movement** | Slow fashion: quality over quantity, ethical supply chains | Slow software: code review, testing, human oversight of AI output | Slow books? Or something better? |

The structural similarity is not coincidental. In each case, a new technology collapsed the marginal cost of production to near zero. When production is cheap, the constraint shifts from *can we make it?* to *should we make it?* — and the market takes time to answer that question honestly.

---

## Fast Fashion: The Cautionary Tale

Zara's genius was not cheapness but *responsive* cheapness. Its vertically integrated supply chain — design, prototyping, manufacturing, and logistics under one roof — meant it could read the market and react in days. Inditex (Zara's parent) controls much of its supply chain, using RFID, advanced ERP systems, and AI-driven trend forecasting to manage inventory with surgical precision.

Shein took the model further. Rather than owning factories, it orchestrated a network of thousands of micro-manufacturers, using machine learning to mine social media for trends and generate thousands of designs automatically. The result: 45 times more product choices than Zara in a comparable period, and 71 times more than H&M.

The environmental cost is staggering. Shein's transport emissions reached 8.52 million tonnes of CO2 in 2024. Polyester accounts for 76% of its fabrics, of which only 6% is recycled. Factory workers regularly work 75-hour weeks. Garments are worn an average of seven times before being discarded.

**The lesson for publishing:** speed and volume are not inherently virtuous. When the system optimises for throughput alone, quality, sustainability, and human welfare are the first casualties.

---

## Fast Software: The Current Crisis

The software industry is two years into its own fast-fashion moment. CircleCI's 2026 State of Software Delivery report, analysing 28 million CI workflows, tells the story in numbers:

- **59% increase** in daily workflow runs year-on-year
- **150% increase** in average PR size
- **9% rise in bug counts**
- A small group of top performers ship faster *and* more reliably; most teams fall behind as validation, integration, and recovery struggle to keep pace

The term of the moment is **vibe coding** — using AI agents to generate entire applications from natural-language prompts. Coined by Andrej Karpathy in February 2025, it has become both a productivity technique and a pejorative. A December 2025 analysis of 470 open-source GitHub pull requests found that code co-authored by generative AI contained approximately 1.7 times more major issues, with security vulnerabilities 2.74 times higher than human-written code.

The deeper problem is what developers call **comprehension debt**: the humans responsible for the code do not understand what it does. AI models hallucinate function names, reference deprecated APIs, and invent configuration options that never existed. A controlled experiment found that on complex, novel tasks, senior developers were 19% *slower* when using AI assistance — the tool's confident-but-wrong suggestions created more work than they saved.

Augment Code's memorable framing: vibe coding generates "tech debt at the speed of light."

**The parallel to fashion is exact.** Shein's garments *look* polished in the product photo; vibe-coded apps *look* functional in the demo. Both hide structural weakness — poor materials in one case, poor architecture in the other. Both reach a "spaghetti point" (around month three for software, around wash three for a Shein dress) where the hidden costs become visible.

---

## Fast Books: Where We Are Now

Publishing has arrived at the same inflection point. In 2025, approximately 4 million books were published in the United States — a 32% increase on 2024. At least 3 million were self-published. Amazon estimates that 10,000 to 40,000 AI-generated ebooks land on Kindle Direct Publishing every month.

The quality crisis is already here. AI-authored mushroom-picking guides listed poisonous species as safe to eat. Amazon responded by limiting self-publishing to three titles per day and requiring AI-use disclosure. The Authors Guild launched a Human Authored Certification programme. Hachette Book Group cancelled a contracted novel over suspected AI generation — the first major traditional publisher to do so publicly.

These are the publishing equivalents of Shein's toxic dyes and single-wear garments: high volume, zero curation, real harm.

**But fast books are not all slop.**

---

## The Case for Fast Books Done Well

There is a version of fast publishing that is neither cynical nor careless. It looks like this:

### 1. The Tool Accelerates the Expert, Not Replaces Them

When I write about Codex CLI, I use the tool itself to help me write. Claude Code researches changelog entries, cross-references documentation, drafts sections that I then rewrite, and checks facts against primary sources. The AI is a force multiplier for domain expertise I already possess — it does not manufacture expertise from nothing.

This is the Zara model, not the Shein model. Zara's speed comes from vertical integration and responsive design by skilled teams. Shein's speed comes from algorithmic generation at scale with minimal human oversight. The difference in output quality is visible to anyone who has worn both.

### 2. Continuous Publishing Matches the Subject Matter

A book about a command-line tool that ships new releases every week cannot afford an 18-month traditional publishing cycle. By the time a conventional publisher finished copyediting, half the chapters would be outdated. Leanpub's in-progress publishing model — where readers buy early and receive free updates — is structurally suited to fast-moving technical subjects. The book is never "finished" in the traditional sense; it is maintained, like software.

This mirrors the shift from waterfall to continuous delivery in software. The question is not "is the book done?" but "is the book correct as of today?"

### 3. AI Handles the Low-Value Work, Humans Handle the High-Value Work

The 32-chapter Codex CLI book exists because AI handles the tasks that would otherwise make a solo author give up: formatting, cross-referencing, consistency checking, citation verification, index generation. The editorial judgement — what to include, how to frame it, what the reader needs to understand — remains human.

In fast fashion terms: the sewing machine made garments affordable, but the best brands still employ designers. The sewing machine is not the problem. The decision to use it without a designer is.

---

## The Taxonomy of Speed

Not all fast production is equal. A useful framework:

| | **Responsive Speed** | **Extractive Speed** |
|---|---|---|
| **Fashion** | Zara: skilled designers + fast supply chain = affordable trend-responsive clothing | Shein: algorithmic generation + micro-factories = disposable clothing at planetary cost |
| **Software** | Agentic engineering: experienced devs + AI agents = faster shipping with maintained quality | Vibe coding: prompts in, code out, nobody reads what shipped |
| **Publishing** | AI-assisted authoring: domain expert + AI tools = rapid, maintained, accurate books | AI book spam: prompts in, 200-page PDF out, listed on Amazon before anyone reads it |

The distinction is not about speed itself but about **who is accountable for the output**. In responsive speed, a skilled human reviews, refines, and takes responsibility. In extractive speed, the system optimises for volume and nobody owns quality.

---

## The Slow Fashion Counter-Movement — and What It Means for Books

Slow fashion emerged as a direct response to fast fashion's excesses. Its principles: quality over quantity, transparent supply chains, ethical production, garments designed to last years rather than weeks.

The "slow software" equivalent is emerging too. CodeRabbit's thesis — "2025 was the year of AI speed; 2026 will be the year of AI quality" — captures the shift. Engineering teams are investing in AI-aware code review, automated testing of AI-generated output, and human oversight of agentic workflows.

For books, the counter-movement might be called **maintained publishing**: books that are written quickly but maintained carefully, updated continuously, fact-checked rigorously, and transparent about their use of AI tooling. The speed of initial production matters less than the ongoing commitment to accuracy.

A maintained book is not a fast book or a slow book. It is a *living* book — closer to well-maintained open-source software than to either a disposable Shein top or a couture gown that takes six months to produce.

---

## The Numbers Behind the Parallel

| Metric | Fast Fashion | Fast Software | Fast Books |
|--------|-------------|---------------|------------|
| **Volume increase** | Shein: 45x more choices than Zara | CI runs: +59% YoY | US publishing: +32% YoY (2025) |
| **Quality signal** | Garments worn avg. 7 times | Bug counts: +9% YoY | Amazon: 3/day publishing limit imposed |
| **Marginal cost** | ~\$0.50 per garment (Shein) | ~\$0 per line of AI code | ~\$0 per AI-generated chapter |
| **Time to market** | Shein: 10 days | Vibe-coded MVP: hours | AI book spam: same day |
| **Counter-movement** | Slow fashion brands | Code review + AI testing | Human Authored Certification |

---

## What This Means for Technical Authors

If you are writing a technical book in 2026, you face the same choice that fashion designers and software engineers face:

1. **Use AI to go faster and maintain quality.** This requires domain expertise, editorial judgement, and a commitment to accuracy that AI cannot provide on its own. The result is a book that ships quickly, stays current, and earns reader trust.

2. **Use AI to skip the hard parts.** This produces books that look complete but read as hollow — the publishing equivalent of a Shein dress that falls apart in the wash. Amazon's three-per-day limit and the Authors Guild certification exist because this path has already caused measurable harm.

3. **Reject AI entirely and ship slowly.** This is the couture option. It produces beautiful work, but for fast-moving technical subjects, the book may be outdated before it reaches readers. There is a place for this — just not in fields where the landscape changes weekly.

The sweet spot is option one: **responsive speed with human accountability**. Use the machines to handle the mechanical work. Keep the thinking for yourself.

---

## Conclusion: The Loom Is Not the Problem

The power loom did not cause fast fashion. The sewing machine did not cause disposable clothing. These tools democratised production. The pathology emerged when the industry optimised for volume over value, speed over sustainability, and margin over meaning.

AI writing tools are the power looms of publishing. They will produce both extraordinary books and enormous quantities of waste. The difference — as in fashion, as in software — comes down to whether a skilled human is accountable for the output.

Fast books are here. The question is whether we build the Zara model or the Shein model. The garment workers, the codebase, and the readers are all hoping we choose wisely.

---

*Daniel Vaughan is the author of [Codex CLI: Agentic Engineering from First Principles](https://leanpub.com/codex-cli) and [The AI Field Engineer's Handbook](https://leanpub.com/ai-fde-handbook), both written with AI assistance and maintained continuously.*
