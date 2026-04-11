---
title: "The Anchoring Problem: Why My Brain Still Thinks Code Is Expensive"
date: 2026-04-09T16:50:00+00:00
tags:
  - opinion
  - personal-essay
  - economics
  - anchoring-bias
  - psychology
  - code-deflation
  - nostalgia
---

![Sketchnote diagram for: The Anchoring Problem: Why My Brain Still Thinks Code Is Expensive](/sketchnotes/articles/2026-04-09-anchoring-problem-brain-thinks-code-expensive.png)

# The Anchoring Problem: Why My Brain Still Thinks Code Is Expensive

Last year I was at a car boot sale in Saffron Walden. Trestle tables covered in somebody else's past. And there, between a chipped Le Creuset pot and a stack of Beano annuals, sat a Psion Series 5. Clamshell open, screen intact, that keyboard I once thought was the pinnacle of miniaturised engineering. I asked how much. "A pound," she said.

I picked it up. In 1997 this thing cost £500 — about £860 in today's money. I remember the reviews in *Personal Computer World*. I remember wanting one and not being able to justify the price. And here it was, in a cardboard box next to a pile of DVDs at 10p each, worth less than the petrol it took to drive it to the field.

I put it back down and walked away. But that moment stuck. Not because of the Psion. Because of what it made me realise about my own code.

## The price tag that won't unstick

I work with Codex CLI and multiple AI agents daily. I watch them produce in thirty seconds what would have taken me the better part of a day. And still — still — some part of my brain wants to carefully save the output. Archive it. Commit it to a repo I'll never revisit. Treat it like something *expensive*.

My brain is anchored to the old price of code.

In 1974, Amos Tversky and Daniel Kahneman published their landmark paper on heuristics and biases in *Science*.[^1] One of the three heuristics they described was **anchoring and adjustment**: people start from an initial value and adjust from it, but the adjustment is almost always insufficient. The anchor drags the final estimate toward itself, regardless of whether the anchor has any rational relevance.

Their most famous demonstration involved a rigged wheel of fortune. Participants who saw the wheel land on 10 estimated 25% of African countries were in the UN. Those who saw 65 estimated 45%. A completely arbitrary number reshaped their judgment. The anchor held.

My anchor is twenty years of writing code at human speed, at human cost. Every function represented hours. Every feature represented weeks. Every production system represented months of collective effort billed at £50-£200 per hour. That price is burned into my intuition. And my intuition is now wrong.

## The deflationary shock

The numbers are hard to argue with.

Before AI coding tools, a moderately complex feature took 800-1,200 hours of development time across a team. At blended rates, that's $50,000-$150,000 per feature. Developers — and I've seen the data on this, and it matches my experience — only write code for about 52 minutes per day on average.[^2] The rest is meetings, context-switching, code review, debugging, and staring at a screen trying to remember what you were doing before Slack interrupted you.

Now look at what's happened. I run three or four Codex CLI agents in parallel and watch them produce a complete feature — tests, documentation, the lot — in the time it takes me to make a coffee. Not a toy feature. A real one, with edge cases handled and error paths tested. The cost at current API pricing? Fractions of a penny. One dollar buys 14 million tokens — roughly 10.77 million words of output.[^3]

I won't cite the 2024 productivity studies here. I've written about this before: if your reference point is a 2024 Copilot study or an early-2025 benchmark, you're measuring a world that no longer exists.[^4] The capabilities have compounded. The shift from autocomplete to autonomous agents happened in months, not years. What matters is what I see on my screen today — and what I see is code that costs essentially nothing to produce.

Code has undergone the same deflationary shock that hit music, photography, encyclopaedias, and long-distance phone calls. The marginal cost of production has collapsed toward zero. But my brain — and, I suspect, yours — hasn't updated the price tag.

## A brief history of things becoming free

The Psion is a good starting image, but the pattern runs deeper. Walk through any charity shop in Britain and you'll see it repeated across shelf after shelf.

### DVDs

DVD sales peaked at $16.6 billion in 2006.[^6] A new release cost £15-£20. By 2007, Netflix had launched digital streaming, and the whole notion of owning a disc started to crack. By 2023, DVD purchases had shrunk 92% from the peak. Netflix mailed its final physical disc on 29 September 2023. By 2024, total physical disc sales had fallen to $959.6 million — a 94.2% decrease from the peak.[^7]

One seller took 30 DVDs in excellent condition to Half Price Books and was offered $4.00 total. Roughly 10 cents each. The objects still existed. The films on them hadn't changed. But the *cost of access* to those same films had dropped so far that the physical artefact lost essentially all its value.

### CDs to streaming

This one I lived through. I remember buying CDs at £12.99 from Our Price on the high street in the mid-1990s. By the early 2000s, new releases cost £16.99. Worldwide music revenue peaked in 1999 at $23.7 billion, with physical sales accounting for $22.3 billion of that.[^8]

Then Napster happened. Between 2001 and 2010, physical music sales declined by more than 60%, wiping out $14 billion in annual revenue. Digital sales grew from zero to about $4 billion in the same period — nowhere near enough to offset the loss. iTunes launched in 2003 with the $0.99-per-track model, itself a massive deflation from $12.99 per album.

Spotify launched in 2008. The industry bottomed out in 2014 at a 20-year low of $13.0 billion.[^9] Today, Spotify UK costs £9.99 per month for access to over 100 million songs — roughly the price of a single CD album. Music transitioned from something we *possess* into something we *access*. Researchers describe the shift as a desire to be "emancipated from ownership."[^10]

I still have a shelf of CDs in the spare room. I haven't played any of them in years. I can't bring myself to throw them away.

### Encyclopaedias

The 32-volume printed Encyclopaedia Britannica sold for $1,395-$1,500 per set.[^11] It was a status symbol. Embassies bought it. Upper-middle-class families displayed it. Then Wikipedia launched in 2001, and by March 2012, Britannica's president announced the end of print editions. Today, Britannica survives online at about $70 per year — a 95%+ price reduction. A $1,400 product was replaced by a free alternative.

The knowledge didn't become less valuable. It became so abundant that charging for access became untenable.

### Photography

A roll of Portra 400 plus developing plus scanning works out to about $1.17 per frame for 36 exposures.[^12] Every click of the shutter cost real money. You composed carefully. You didn't waste shots. Then the iPhone arrived in 2007, and the marginal cost of a photograph went from roughly a dollar to roughly zero. Kodak filed for bankruptcy in 2012. They saw the transition coming and couldn't adapt.[^13]

The parallel to code is almost too neat. Like film photography, code used to have a high per-unit cost. Each function, each feature, each line represented real time and money. Now, like digital photos, the marginal cost of generating code approaches zero. But developers still compose as if every keystroke costs a dollar.

## The four biases keeping your anchor stuck

The anchoring bias doesn't work alone. It's layered with at least three other cognitive biases that reinforce each other. Understanding the stack helps explain why the adjustment is so hard.

### The endowment effect

Richard Thaler coined this term in 1980 and eventually won the Nobel Prize in Economics partly for the work.[^14] The endowment effect is the observation that people assign greater value to things merely because they own them. In Kahneman, Knetsch, and Thaler's famous mug experiment at Cornell, students who were given a mug and offered the chance to sell it demanded roughly twice the price that non-owners were willing to pay.[^15] Ownership alone doubled the perceived value.

When a developer writes code by hand — spending hours debugging, refactoring, polishing — they develop ownership. Deleting that code triggers loss aversion. It *feels* like destroying something valuable, even when AI could regenerate equivalent functionality in seconds. The brain is anchored to the investment, not the replacement cost.

### The IKEA effect

Norton, Mochon, and Ariely identified this in 2011 at Harvard, Yale, and Duke.[^16] Labour alone is sufficient to induce greater liking for the fruits of that labour. Subjects who built furniture were willing to pay 63% more than those given pre-built furniture. Participants saw their "amateurish creations as similar in value to the creations of experts."

This maps directly onto code. The function you sweated over for three hours feels more *yours* — and therefore more valuable — than code an AI generated in thirty seconds, even when the output is functionally identical. The labour of writing creates attachment that has nothing to do with the quality of the result.

### The sunk cost fallacy

Businesses continue to invest in maintaining legacy systems rather than rebuilding because they've already spent so much on the existing codebase.[^17] "We've invested two years building this. A rewrite would be wasteful." The teams justify ongoing maintenance, even as the maintenance costs exceed what a rebuild would cost. The emotional anchor of past investment holds the decision in place, even after the economics have shifted underneath it.

In the AI era, this becomes especially irrational. When an agent can regenerate functionality in hours rather than months, the rational calculation has shifted dramatically. But the emotional anchor of "we spent two years building this" remains fixed.

### Loss aversion

Kahneman and Tversky showed that losing something feels about twice as painful as gaining the same thing feels good.[^18] Deleting code feels like a loss. Generating new code feels like a gain. The loss hurts more. So developers keep code, hoard repos, comment out instead of delete, and maintain systems long past their useful life. The asymmetry is baked into the wiring.

## The collector instinct

George V. Neville-Neil wrote about code hoarding in *Communications of the ACM* back in 2016, before AI coding tools existed.[^19] Even then, the patterns were clear: hidden features buried under obscure configuration options, code committed to repos that nobody believed in but couldn't bring themselves to remove, newer developers too afraid to delete anything in case it turned out to be important.

I recognise all of this in myself. I have repositories on GitHub that I haven't touched in years. Projects I'll never return to. Utility scripts for tools I no longer use. And every few months I look at them, think "I should clean these up," and then don't. Decision fatigue kicks in. It's easier to keep than to evaluate and delete.

The research on digital hoarding identifies five motivations: fear of losing information, sentimental attachment, decision fatigue, the effort of deletion, and zero storage cost removing the pressure to clean up.[^20] Researchers describe us as "a generation of digital hoarders" who hoard with the best of intentions, telling ourselves we'll look at it this weekend. The infinite list becomes a source of low-grade anxiety.

The AI era makes this worse, not better. When generating code is nearly free, the temptation is to generate more and discard nothing. The digital attic fills faster than ever.

## The emotional dimension

Here's where the argument gets uncomfortable. It's one thing to talk about cognitive biases in the abstract. It's another to acknowledge that people are genuinely hurting.

A developer wrote on Reddit in late 2025: "The impact was so strong that I still haven't recovered. It's not that AI writes better code than me. It's that it's replacing intellectual activity itself."[^21]

Another: "Am I falling behind? Should I be doing more? Is everything I've built going to be worthless?"[^22]

A 2026 study in *Frontiers in Psychology* identified seven themes in developer discourse about AI, including eroded identities, technostress, and what the researchers termed "expertise devaluation" — the deflation felt when skills developed over years become replicable by algorithms overnight.[^23]

On Hacker News, a commenter made a distinction I keep returning to: "LLMs are quite good at coding, but terrible at software engineering."[^24] Another observed: "Generating code is now cheap, but the cost of *owning* that code is not."

A software engineer reported that after four months of issuing hundreds of prompts per day, she'd "started to lose my ability to code."[^25] An Apple engineer said: "I believe that coding can be fun and fulfilling and engaging, and having the computer do it for you strips you of that."

30% of developers think AI will replace them entirely.[^26] The remaining 70% are split between cautious optimism and active denial.

I don't think dismissing these feelings is helpful. "I am someone who writes code" is a self-definition for millions of people. When that definition becomes unstable, the response isn't purely rational. The anchoring isn't only about price — it's about identity. My brain thinks code is expensive partly because accepting that it isn't means accepting that something I spent decades getting good at now costs less than a cup of coffee to replicate.

That's a hard update.

## How long does adjustment take?

The music industry offers the clearest timeline, because the data is comprehensive and the transition is complete.

The full cycle from CD peak to streaming recovery took approximately 20 years, from 1999 to 2019.[^27]

**Phase 1 — Denial and resistance (1999-2003):** About four years. The industry sued Napster, sued individual file-sharers, tried to maintain the $12.99 album model. Revenue was already declining, but labels insisted on the old economics.

**Phase 2 — Grudging digital acceptance (2003-2008):** About five years. iTunes launched at $0.99 per track — a massive price concession. Labels licensed reluctantly, with DRM restrictions. Physical sales in freefall, digital unable to compensate.

**Phase 3 — Streaming emergence (2008-2014):** About six years. Spotify launched, initially met with industry hostility. Revenue bottomed out in 2014 at $13.0 billion, down from $23.7 billion in 1999. A 15-year, 54% decline from peak.

**Phase 4 — Streaming acceptance and recovery (2014-2024):** About ten years. Labels finally embraced streaming. Revenue recovered to $17.7 billion in 2024, surpassing the nominal 1999 peak — but still roughly 46% below the 1999 peak in inflation-adjusted terms.

Research on individual consumer adjustment tells a different story. Studies of Spotify adoption identify three psychological periods: short run (weeks 0-1, initial exploration), medium run (weeks 2-24, behaviour adjustment), and long run (week 25 onwards, stable new behaviour).[^28] Individual consumers adjust in about six months. But the *industry* took 15 years to stop fighting the new economics.

Microsoft's research on AI coding tools suggests it takes about 11 weeks for developers to fully realise productivity gains.[^29] That's the individual adjustment period. The industry adjustment — the cultural, institutional, emotional adjustment — will take far longer.

Where are we in the software industry's version of this timeline? I'd argue we're somewhere in Phase 2. The tools are here. The early adopters have adjusted. But most of the industry is still licensing reluctantly, with restrictions. Still trying to maintain the old pricing model. Still anchored.

## What changes when you stop treating code as precious

Charity Majors at Honeycomb described software development bifurcating into two categories: disposable code and durable code.[^30] Disposable code is prototypes, experiments, scripts — "you generate some code to do a thing, you do the thing, you throw it away." Durable code is production systems needing reliability and deep expertise. The expensive part of software development, she argues, requires knowing which category you're in.

This framing only makes sense once you've unstuck the anchor. If code is expensive, there's no such thing as disposable code — everything is precious by default. If code is cheap, you can make the distinction freely.

When I catch myself adjusting to the new economics — when I manage to override the anchor — several things shift.

**Generate-test-discard becomes viable.** Instead of carefully writing a function, testing it, and iterating, I can generate five different implementations, run them all against the test suite, pick the best one, and throw the rest away. This was absurd at human-typing speed. It's natural at AI-generation speed.

**Architecture becomes more experimental.** If rebuilding a component costs hours instead of months, you can try three different architectural approaches and see which one holds up under load. Ivan Turkovic described this as "write code, run it, get the result, throw it away, generate it again next time maybe with better context or updated specs."[^31] The pressure around maintainability drops when regeneration is cheaper than maintenance.

**Technical debt changes shape.** As the team at Recursive AI put it: "If I can rebuild a complete system in days, why am I treating the code as sacred?"[^32] Technical debt disappears when rebuilding is faster than refactoring. Each version starts fresh with current best practices.

**The relationship shifts from possession to access.** This is the streaming-music parallel. Code transitions from something I *have* to something I can *get*. I don't need a shelf of CDs if I have Spotify. I don't need a repository of utility functions if I can generate them on demand.

Jeremy Rifkin described this broader dynamic in *The Zero Marginal Cost Society* in 2014, arguing that the drive toward near-zero marginal cost sits at the heart of capitalism's own contradictions.[^33] But there's a crucial difference with AI-generated code. Until now, zero marginal cost applied to the *reproduction* of content already created — copying a file, streaming a song. AI targets the *initial fixed cost of creation itself*.[^34] You're not copying existing code for free. You're generating new code for near-free. That's a structurally different kind of shift, and it's why the psychological adjustment may be harder than any previous transition.

## The subsidy problem: a price anchor built on sand

There's a complication I'd be dishonest to ignore. The current near-free economics of AI-generated code are partly artificial. Both Codex CLI and Claude Code are heavily subsidised right now — the token prices developers pay are significantly below the actual cost of inference. I'm very aware this could go away at any time.

This creates a uniquely disorienting triple bind. Your *old* anchor says code is expensive — hours of human labour per feature. Your *current* experience says code is nearly free — a few cents per generation. And your *scarcity awareness* whispers that this free period is temporary — the subsidy will pull back, and you don't know where the real price will settle.

It's the early Spotify problem again. Spotify ran at a loss for years, subsidised by venture capital, to build market share. The £9.99/month price felt impossibly cheap because it *was* — it was below cost. Consumers anchored to that subsidised price, and when Spotify eventually raised prices, the outrage was immediate even though the new price was still a fraction of what the equivalent music would have cost on CD.

The 491-comment billing thread on the Codex CLI GitHub repo already shows this anxiety in real time — developers reporting 5-10x increases in token consumption as usage scales, suspecting the subsidy is being quietly withdrawn.[^billing] The psychological whiplash of adjusting *down* to near-free code, only to be yanked *up* again when pricing normalises, may be worse than never having adjusted at all.

The lesson from every previous deflation cycle: anchor to *value delivered*, not to *current cost*. The price of tokens will fluctuate. The question that matters is whether the output — the working code, the passing tests, the shipped feature — is worth what you paid. That calculation is stable even when the price isn't.

[^billing]: "What happened to usage?" GitHub issue thread, Codex CLI repository, 2026. 491+ comments discussing billing transparency and token consumption increases.

## The craft question

There's a version of this argument that treats developer resistance as irrational — as pure cognitive bias to be overcome. I don't think that's entirely fair.

The experienced programmers I know who use AI tools most effectively are the ones with the strongest foundational skills and the clearest architectural vision.[^35] They use AI for the mundane work and apply human judgment to the structural decisions. The anxiety is real but so is the skill differential. Knowing *what* to build, *why* to build it, and *whether* the output is any good — that hasn't been deflated. Not yet.

The junior developer problem is genuine. If nobody hires junior devs because LLMs can do junior work faster and cheaper, how does anyone become an expert?[^24] That's not anchoring bias. That's a real structural question that the industry hasn't answered.

And the craft argument — that coding can be fun, fulfilling, and engaging — isn't irrational either. Film photographers still shoot film. Vinyl collectors still buy records. Some things have value beyond their marginal cost. The mistake is confusing personal craft with economic reality. You can enjoy hand-writing code the same way you enjoy hand-developing photographs. What you can't do is price it the same.

## Updating the anchor

I still have that shelf of CDs. I still have repos I'll never touch again. I still catch myself wanting to save AI-generated code as if it were scarce.

But I'm getting better at noticing when the anchor is speaking. The flinch when I'm about to delete a file I spent an afternoon on. The reluctance to regenerate something from scratch when I could patch the existing version. The collector instinct that says *keep it, you might need it*.

These feelings are real, but they're not rational. They're the ghost of a price tag that no longer exists.

The Psion 5 at the car boot sale was worth £500 once. The DVDs were worth £15 each once. The CD collection on my shelf was worth hundreds of pounds once. The code I wrote last year at human speed and human cost was expensive once.

None of those prices are coming back.

The question isn't whether the anchor will eventually update — it will, the same way it updated for music, for photography, for encyclopaedias, for long-distance phone calls. The question is whether I'll spend the next fifteen years in the denial phase, suing Napster and insisting albums should cost £12.99, or whether I'll adjust faster than the industry did last time.

I'm trying to adjust. Some days I manage it. I generate, test, discard, regenerate. I treat code as something I access rather than something I own. I delete without flinching.

Other days I find myself carefully saving a utility function that took Claude thirty seconds to write, filing it away in a repo marked "useful snippets," and pretending that's different from the box of 10p DVDs in my garage.

It isn't. But my brain hasn't finished updating the price tag.

---

## Citations

[^1]: Tversky, A., & Kahneman, D. (1974). "Judgment under Uncertainty: Heuristics and Biases." *Science*, 185(4157), 1124-1131. [doi:10.1126/science.185.4157.1124](https://www.science.org/doi/10.1126/science.185.4157.1124)

[^2]: Cleveroad. "Software Development Time Estimation." [cleveroad.com](https://www.cleveroad.com/blog/software-development-time-estimation/)

[^3]: OpenAI API pricing, April 2026. One dollar buys approximately 14 million tokens at current rates for GPT-5.3-Codex output.

[^4]: See "The Reference Point Problem" — a recurring theme in this article series. Studies based on 2024 Copilot-era data measure autocomplete productivity, not autonomous agent productivity. The capability gap between those eras is qualitative, not just quantitative.

[^6]: StatsSignificant. "The Rise, Fall, and (Slight) Rise of DVDs." [statsignificant.com](https://www.statsignificant.com/p/the-rise-fall-and-slight-rise-of-b06)

[^7]: Variety. "It's Official: The DVD Business Died Last Year." [variety.com](https://variety.com/vip/rip-dvd-business-2024-1236322977/)

[^8]: World Economic Forum. "Impact of Streaming on the Music Industry." [weforum.org](https://www.weforum.org/stories/2023/03/charted-the-impact-of-streaming-on-the-music-industry/)

[^9]: Digital Music News. "Despite Streaming, Revenues Down 50% from 1999 Peak." [digitalmusicnews.com](https://www.digitalmusicnews.com/2020/08/27/recorded-music-revenues-decrease/)

[^10]: Neuroscienceof.com. "How Digital Abundance Impacts the Psychology of Music." [neuroscienceof.com](https://www.neuroscienceof.com/human-nature-blog/psychology-digital-abundance-music-streaming-spotify-creativity)

[^11]: CSMonitor. "Encyclopaedia Britannica: After 244 Years in Print." [csmonitor.com](https://www.csmonitor.com/Business/Latest-News-Wires/2012/0314/Encyclopaedia-Britannica-After-244-years-in-print-only-digital-copies-sold)

[^12]: Dave Herring. "The Actual Cost of Film Photography." [dave.online](https://dave.online/writings/the-actual-cost-of-film-photography)

[^13]: Harvard Digital. "The Rise of the Smartphone Ecosystem and Kodak's Fall." [d3.harvard.edu](https://d3.harvard.edu/platform-digit/submission/the-rise-of-the-smartphone-ecosystem-and-kodaks-fall/)

[^14]: Wikipedia. "Endowment Effect." [en.wikipedia.org](https://en.wikipedia.org/wiki/Endowment_effect)

[^15]: Kahneman, D., Knetsch, J. L., & Thaler, R. H. (1990). "Experimental Tests of the Endowment Effect and the Coase Theorem." *Journal of Political Economy*, 98(6), 1325-1348.

[^16]: Norton, M. I., Mochon, D., & Ariely, D. (2011). "The IKEA Effect: When Labor Leads to Love." *Journal of Consumer Psychology*. [HBS Faculty Page](https://www.hbs.edu/faculty/Pages/item.aspx?num=41121)

[^17]: Canidium. "Legacy IT Systems and The Sunk Cost Fallacy." [canidium.com](https://www.canidium.com/blog/legacy-it-systems-sunk-cost-fallacy-overspending-underperforming-it-solutions)

[^18]: Kahneman, D. & Tversky, A. (1979). "Prospect Theory: An Analysis of Decision under Risk." *Econometrica*, 47(2), 263-291.

[^19]: Neville-Neil, G. V. (2016). "Code Hoarding." *Communications of the ACM*, February 2016. [cacm.acm.org](https://cacm.acm.org/opinion/code-hoarding/)

[^20]: Wikipedia. "Digital Hoarding." [en.wikipedia.org](https://en.wikipedia.org/wiki/Digital_hoarding); Taylor & Francis. "Digital Hoarding and Personal Use Digital Data." [tandfonline.com](https://www.tandfonline.com/doi/full/10.1080/07370024.2023.2293001)

[^21]: LAFFAZ. "I Stopped Writing Code After Using AI." [laffaz.com](https://laffaz.com/ai-replacing-developers-thinking-identity-crisis/)

[^22]: DEV Community. "To the Programmer Quietly Drowning in AI Anxiety." [dev.to](https://dev.to/kaniel_outis/to-the-programmer-quietly-drowning-in-ai-anxiety-42pm)

[^23]: Frontiers in Psychology. "Algorithmic Anxiety." [frontiersin.org](https://www.frontiersin.org/journals/psychology/articles/10.3389/fpsyg.2026.1745164/full)

[^24]: Hacker News. "Writing Code Is Cheap Now." [news.ycombinator.com](https://news.ycombinator.com/item?id=47125374)

[^25]: The New Stack. "AI Coding Tools Reckoning." [thenewstack.io](https://thenewstack.io/ai-coding-tools-reckoning/)

[^26]: DEV Community. "30% of Developers Think AI Will Replace Them." [dev.to](https://dev.to/ji_ai/30-of-developers-think-ai-will-replace-them-ghp)

[^27]: Visual Capitalist. "Visualizing 40 Years of Music Industry Sales." [visualcapitalist.com](https://www.visualcapitalist.com/music-industry-sales/)

[^28]: Neuroscienceof.com. "How Digital Abundance Impacts the Psychology of Music." [neuroscienceof.com](https://www.neuroscienceof.com/human-nature-blog/psychology-digital-abundance-music-streaming-spotify-creativity)

[^29]: GitHub. "Measuring Impact of Copilot." [resources.github.com](https://resources.github.com/learn/pathways/copilot/essentials/measuring-the-impact-of-github-copilot/)

[^30]: Honeycomb. "Disposable Code Is Here to Stay." [honeycomb.io](https://www.honeycomb.io/blog/disposable-code-is-here-to-stay)

[^31]: Ivan Turkovic. "AI-Generated Code and Maintainability." [ivanturkovic.com](https://www.ivanturkovic.com/2025/07/28/ai-generated-code-maintainability/)

[^32]: Recursive AI. "Code is Disposable." [recursiveai.net](https://recursiveai.net/articles/code-is-disposable/)

[^33]: Rifkin, J. (2014). *The Zero Marginal Cost Society*. Palgrave Macmillan. [us.macmillan.com](https://us.macmillan.com/books/9781137280114/thezeromarginalcostsociety/)

[^34]: Outpile. "Zero Marginal Cost Didn't Kill Capitalism." [outpile.com](https://www.outpile.com/en/articles/zero-marginal-cost-capitalism-digital-economy)

[^35]: Rob Bowley. "Is AI About to Expose How Mediocre Most Developers Are?" [blog.robbowley.net](https://blog.robbowley.net/2025/08/08/is-ai-about-to-purge-mediocre-developers/)

---

## Summary

- **Anchoring bias** causes developers to mentally price code at pre-AI rates, even as AI tools collapse the marginal cost of code generation toward zero.
- **Four reinforcing biases** — the endowment effect, the IKEA effect, the sunk cost fallacy, and loss aversion — keep the old price tag stuck.
- **Historical parallels** in DVDs (94% price collapse), CDs to streaming (20-year industry adjustment), encyclopaedias ($1,400 to free), film photography ($1/frame to zero), and PDAs (£500 to £1) all follow the same pattern: new technology makes production nearly free, but human psychology lags by years or decades.
- **The music industry adjustment took 20 years** from CD peak to streaming recovery. Individual consumers adjusted in about 6 months. The software industry is likely somewhere in early Phase 2 of a similar cycle.
- **Developer identity is entangled with the old economics.** The emotional resistance is real and not entirely irrational — craft, skill, and architectural judgment still matter — but the price of code production is not returning to pre-AI levels.
- **The practical shift** is from code-as-possession to code-as-access: generate-test-discard workflows, disposable vs. durable code distinctions, and treating regeneration as cheaper than maintenance.
- **The key question** is not whether the anchor will update, but how long the adjustment will take — and whether individuals can adjust faster than the industry did with music.
