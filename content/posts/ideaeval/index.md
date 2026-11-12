---
title: "IdeaEval: Impact-Oriented Evaluation of Scientific Hypothesis Generation"
date: 2026-10-16
author: ["Yanchen Huang", "Yuchen Niu", "Xiangkun Hu"]
description: "What makes a research idea worth pursuing—and can AI tell before the experiments? Lessons from building IdeaEval."
acknowledgment: "This work was conducted during an internship at Analemma AI. We gratefully acknowledge Hengquan Guo, Sheng Xia, Yuzhe Yang, Yuanbo Yang, Yiran Chen, and Tianxiang Sun for their valuable assistance and contributions."
math: true
tags: ["Auto Research", "Idea Evaluation", "LLM Agents"]
categories: ["Research"]
showToc: true
TocOpen: false
translation: /posts/ideaeval-zh
translationLabel: "中文"
---

> **TL;DR**
> Human researchers rely on research taste to judge whether an idea is worth pursuing before running full experiments. We tried to fit that ability for evaluating ideation systems: a judge agent that compares two proposals pairwise, calibrated against the research community's later feedback (citations). On proposal pairs reconstructed from real papers, the judge agreed with the later citation ranking **75–83%** of the time. But given only the two proposals' titles, it was still right **about 73%** of the time, and on 75 hand-labeled human–AI pairs its accuracy was just **49.3%**, close to chance. As far as our evidence goes, the judge is mostly judging topic heat, citeability, and how polished the packaging is, not whether an idea is sound and feasible given what the field already knows.

{{< resource-links >}}
https://github.com/02hyc/IdeaEval
https://huggingface.co/datasets/Yukino13/IdeaEval-Bench
https://huggingface.co/datasets/Yukino13/IdeaEval-Calibration
https://huggingface.co/datasets/Yukino13/IdeaEval-Corpus
{{< /resource-links >}}

Since March this year, we have made many attempts at evaluating research ideas. This post collects some of what we learned along the way, to leave a public trace of that work and perhaps a small bit of inspiration for whoever tries this next.

## The last step to high-quality auto research
It's November 2026, and auto research is everywhere. By September, ICLR 2027 had passed 60,000 abstract registrations [1]. A year earlier it had fewer than 20,000 valid submissions; ten years ago, the number was 490. As submission counts climb, peer review itself has been infiltrated by AI, and its quality is uneven.

We have to admit that agents can already turn a proposal into code and run a complete set of experiments. Early this year we built FARS [2], an end-to-end auto research system, and livestreamed its automated research process over the Spring Festival holiday. It produced a large number of papers with real code and run logs. But most of that output was mediocre, because the proposals the agents executed weren't good enough to begin with. Most agent-written proposals were trivial, a few existing components bolted together, and quite a few were barely feasible.

We think proposal quality is the key bottleneck of auto research. If an ideation system could come up with valuable, novel proposals on its own, it would close the last step toward high-quality auto research. So we turned our attention to agents' ideation ability.

![FARS pipeline: research direction, ideation, planning, experiment, writing, paper; all stages share one workspace](figures/01-fars.svg)

## Ways to evaluate a research idea
The question is how to measure the quality of the proposals an ideation system produces. In general, the best way to evaluate a proposal is to run it and let the results decide. AlphaEvolve [3] uses an LLM to drive an evolutionary search over programs, and an automatic evaluator scores every candidate and feeds the score back. Anthropic's Automated Alignment Researcher [4] studies weak-to-strong supervision: one fine-tuning experiment takes under two hours, and the agent submits its predictions to a remote API and gets a PGR (performance gap recovered) score back. These tasks share one property: they are outcome-gradable. The problem is defined in advance, there's a clear metric a machine can compute, feedback comes back in minutes or hours, and the system can try things at scale and hill-climb straight up the metric.

The auto research we care about is a more general setting. It deals with open research questions, and taking a proposal to experiments, from writing code and running experiments to analyzing results and iterating, usually takes weeks or months. An ideation process can't afford that time, and if a badly unreliable proposal goes straight into experiments, it also burns a lot of compute. So we need another way.

In ordinary research, human researchers evaluate an idea ahead of time with the research taste they've built up over their careers. Drawing on past experience and the current state of the field, they can often tell whether an idea is sound and valuable without running the full experiments, and decide from that whether to keep going. That is exactly how we'd want an ideation system to be evaluated. If the evaluator of an ideation system had this kind of research taste, it could give direct feedback on proposals at the ideation stage and filter out the ones not worth pursuing.

So we need to fit this research taste and use it as the standard for evaluating ideation systems. How do you quantify research taste? Plenty of work already models and benchmarks idea evaluation directly. Most of it doesn't target research taste as such, but each picks some signal as the ground truth for whether an idea is good, so each can be read as a different approximation of research taste. But each of those signals has its limits.

![Evaluation signals on an idea's timeline. When the idea is proposed, before any experiment: expert ratings [9][10] and novelty checks [14]. After experiments: experimental outcomes [15] and hypothesis verification [16][17]. At publication: matching the published paper [5–8] and acceptance or reviews [11–13]. Months to years later: citations and awards [18][19]. Earlier signals are noisy or partial; complete signals arrive only after the fact. Research taste means predicting, at the moment an idea is proposed, what the community will say later](figures/16-signal-timeline.svg)

### Published papers as ground truth

This line of work approximates research taste as "can you come up with the idea that actually got published," but what it really measures is the ability to reproduce a target paper, not the ability to judge whether an idea is good. The model gets the context that existed before the target paper came out and is scored on how close its idea lands to the real paper. IdeaBench [5] takes the abstracts of references cited by 2,374 biomedical papers as input and measures overlap with semantic similarity and LLM scoring. AI Idea Bench 2025 [6] uses the motivation and experimental plan of 3,495 top-venue AI papers as ground truth. ResearchBench [7] (and its predecessor MOOSE-Chem [8]) breaks scientific discovery into three steps: retrieving inspirations, composing hypotheses, and ranking hypotheses. This kind of ground truth is reliable and easy to scale, but for any research question there are far more ideas worth pursuing than the one that got published, and a proposal that goes a different way but is just as valuable gets marked wrong.

### Expert judgment as ground truth

This is the most direct approximation of research taste: ask people who have taste. But expert taste is noisy and hard to scale. Si et al. [9] had more than 100 NLP researchers blind-review ideas written by humans and by LLMs, and reviewers agreed with each other only 56.1% of the time, below official NeurIPS and ICLR reviewing. Anthropic's **TASTE** [10] had 10 AI safety researchers score, discuss, and revise their ratings of proposals generated by Claude Opus 4.6, ending up with only 92 pairs with a clear preference (that work is close to ours in spirit, but it wasn't out yet while we were working). **GraphEval** [11], **InnoEval** [12], and **ScholarEval** [13] fall back on acceptance decisions or review comments from venues like ICLR and NeurIPS as labels. These are cheaper to get, but an acceptance decision mixes in writing quality, how complete the experiments are, reviewer match, submission timing, and more, so there's still a sizable gap between it and the value of the idea itself.

### A single verifiable dimension as ground truth

This line of work reduces research taste to one aspect that can be checked objectively. The results are reliable, but taste shows up precisely in weighing many dimensions against each other. Literature-Grounded Novelty Assessment [14] judges novelty only, retrieving related papers and comparing facet by facet (purpose, mechanism, evaluation). Forecasting Research Success [15] judges experimental results only, predicting across more than ten thousand idea pairs which idea scores better on the same benchmark. TruthHypo [16] and HypoBench [17] judge only a hypothesis's truthfulness and explanatory power, against the later literature and against the true data-generating mechanism of synthetic data, respectively. But a novel idea can be pointless, and an idea that bumps a benchmark score can be just another engineering trick, which was exactly FARS's typical problem. And what these works evaluate is usually a single hypothesis or method, not a full proposal.

### Later community feedback as ground truth

This line of work approximates research taste as anticipating an idea's long-term impact, the signal closest to an idea's real value. **Proof of Time** [18] gives an agent a snapshot of the evidence before some date and asks it to predict what comes after: citations and awards for papers, shifts in researchers' directions, or where performance on a benchmark will land. 

**AI Can Learn Scientific Taste** [19] defines scientific taste directly as the ability to judge and propose high-impact ideas. From 2.1 million arXiv papers it builds about 720,000 pairs from the same field and period, labels each pair by citations to train a Scientific Judge, then uses that judge as a reward model to train a Scientific Thinker that proposes follow-up ideas. But this work evaluates a paper's title and abstract, which usually already contain the experimental results and conclusions. That makes it a judgment after the fact, while the whole point of research taste is to judge beforehand. 

Still, this work gave us an important insight: a human researcher's taste is built by watching, over and over, which ideas the community later adopts and which it forgets. So if we could align the ex-ante evaluation of a proposal at the ideation stage with the community feedback that comes later, we would be fitting research taste. That was the starting point of this work.

## Judging an idea against its time
To align the ex-ante evaluation of an idea (research taste) with the feedback that comes after (community feedback), we first lay out how an idea goes from being proposed to creating value.

An idea is always proposed for some research question at some point in time. Its value depends on the knowledge the field already has when it's proposed, and on how it shapes later work once it's been carried out; the same idea proposed at different times can be worth very different things. Let $K_t$ be the field's knowledge base at time $t$ and $H$ a proposal's core hypothesis. We define an idea's value as the expected gain to the future knowledge base from pursuing $H$ under $K_t$:

$$V(H, K_t) = \mathbb{E}[\Delta K \mid H, K_t]$$

Under this definition, research taste is the ability to estimate $V(H, K_t)$ before $H$ is carried out. $V$ has three properties:
1. **It is a prediction.** The true value of $V$ only shows up after the idea has been carried out and taken up by the community; an evaluator can only give an expectation under $K_t$.
2. **It has to be calibrated against community feedback.** Which ideas the community later adopted, built on, and kept comparing against: those are later observations of $V$.
3. **It suits relative comparison.** There's no reliable objective label for an idea's absolute value, and an absolute score is hard to calibrate directly; comparing two proposals on the same research question under the same $K_t$ is much easier. This also fits how an ideation system works: by repeatedly comparing and iterating, generate, evaluate, refine, under the same $K_t$, it should in principle converge on better ideas.

![Envisioned ideation loop: a Seeder generates proposals, a Filter removes invalid ones, a pairwise Ranker updates Elo scores in the candidate pool, an Evolver revises the top proposals, and the best one is delivered](figures/02-evolution.svg)

History is an evaluator that always shows up late: the community's verdict on an idea often arrives long after it was proposed. Our plan was to use that accumulated feedback to fit ex-ante evaluation: turn historical papers back into pre-experiment proposals, label them with the community feedback they later received, and build an idea evaluation benchmark from them. The benchmark would serve two purposes: calibrating the evaluator itself, and evaluating the proposals ideation systems produce. Building it meant answering three questions. What form should the evaluation take? How do we find proposals that share the same $K_t$? Which community signal should serve as the label?

## Turning history into a ruler
### Compare proposals only against their contemporaries
Because $V(H, K_t)$ suits relative comparison and the comparison has to happen under the same $K_t$, the basic unit of evaluation is a pair of proposals under the same $K_t$. We approximate that papers on the same research question published within the same short window share the same $K_t$, so the proposals reconstructed from them are comparable. And if an ideation system writes a proposal for the same research question at the start of the window, $t$, it too can only draw on $K_t$, so it can be compared directly with those human proposals.

In this setup, the evaluator is a pairwise ranker. It judges between two human proposals, and their later community feedback checks whether it was right: that's calibration. Once calibration is trustworthy, it compares an AI proposal with every human proposal in the same window, and the win rate measures the ideation system: that's ideation evaluation. So the goal of building the benchmark is to find enough papers that share the same $K_t$.

![K_t is the work on one research question published before cutoff t. Human proposals from the next 31 days build on the same K_t, and an AI proposal generated at t can only draw on K_t. A pairwise ranker compares two human proposals and is checked against later citations (calibration), then compares the AI proposal with every human proposal in the cohort (ideation)](figures/03-comparison.svg)

### Carving the literature into cohorts
Sharing the same $K_t$ takes two conditions: working on the same research question, and being published in the same period. The natural approach is to cluster the literature by research question first, then split each cluster into time windows.

**Data selection** Historical papers have to come out after the knowledge cutoff of the evaluator's base model; otherwise the model may have seen them during training. GPT-5.4 and Opus 4.6, the models we planned to use, both have cutoffs in August 2025 [20][21], so we used AI papers posted to arXiv between September 2025 and January 2026 (cs.LG, cs.AI, cs.CL, cs.CV, stat.ML). After removing surveys, pure benchmark papers, pure dataset papers, and anything else not built around a research hypothesis, we had 35,505 papers.

**Clustering** We cluster papers into a tree by the semantics of the research problem itself, and we let cluster size, not semantic rules, set the granularity of a research question. Both choices came from the failure of our first attempt. We first tagged every paper with labels such as topic, method family, and benchmarks used, and grouped papers by label overlap. But overlapping labels don't mean the same research question: SimpleTIR [22] and Inpainting-Guided Policy Optimization for Diffusion LLMs [23] have almost identical labels (RL, reasoning, math benchmarks), yet one studies multi-turn RL with tool calls and the other policy optimization for diffusion language models. The resulting clusters also varied wildly in granularity and overlapped with each other, and that kind of research-question granularity is very hard to pin down with natural-language rules.

For the implementation we followed TaxoAdapt [24], clustering level by level on each paper's title, research problem, task formulation, and benchmarks used, and splitting until every leaf held 10 to 50 papers; each leaf is one research question. The first two levels were deduplicated by hand, and each paper can belong to 1 to 3 nodes. The result is a 6-level tree with 1,799 leaves. After writing a research question for each leaf and checking every paper against it, we removed 4,357 papers that didn't fit.

**Splitting by time** Within each topic, we cut 31-day windows into cohorts: a group of papers on the same research question published in the same month, our approximation of "the same $K_t$." A topic here is a node whose subtree holds at most 150 papers, not a leaf, because leaves hold too few papers to still have enough left after a time split. Some papers had also been public before they reached arXiv, as journal papers or OpenReview submissions. We ran a date-leak check over every paper, corrected the actual publication dates, and removed the problematic ones.

![Left: the corpus of 35,505 papers splits level by level until every leaf holds 10 to 50 papers. Research-question granularity is set by size: the highest node whose subtree holds at most 150 papers is a topic, itself a broader research question, and each leaf is a finer one. Right: one topic on a timeline cut into 31-day windows; each window is a cohort. Each leaf alone is sparse within a window, while the topic as a whole gives enough papers](figures/04-taxonomy.svg)

### Rewinding papers into proposals
With cohorts in hand, the papers in them still have to be turned back into pre-experiment proposals. The conversion follows two principles: don't leak the results, and stay faithful to the paper's own research plan. We sort a paper's content into three kinds: research plan, prior knowledge, and experimental results. Results are removed entirely: every number for the proposed method in the main results table becomes TBD, and the paper's conclusions are rewritten as hypotheses to be tested. Which baselines to compare against, which ablations to run, and which datasets to evaluate on belong to the research plan and stay in full.

Every reconstructed proposal follows the same seven-section structure: Introduction (core insight, a falsifiable hypothesis, and possible failure modes), Proposed Approach, Related Work, Experiments, Success Criteria, Impact Statement, and References. The ideation baselines produce proposals in the same structure, so the two can be compared directly and the evaluator won't favor either side because of format.

![Paper to proposal: a paper's content is sorted into plan, prior knowledge, and results; results are removed, findings are rewritten as hypotheses, and the rest goes into a seven-section proposal template with results marked TBD](figures/06-paper-proposal.svg)

### Four-month citations predict the twelve-month ranking
We use citation counts within a cohort as the label for *which one is better*. The difficulty is that when we pulled the data, these papers had been out for only 4 to 8 months. A backtest shows that as long as the citation gap between two papers is large enough, the early citation ranking already agrees closely with the ranking 12 months out.

**Choosing the community signal** What we really want to measure is an idea's incremental contribution to the field's knowledge base. It can't be observed directly, only through indirect signals. Conference review scores are one option, but review quality varies, reviewers don't share the same standards, and there's little public data. Citations are noisy too, swayed by institutions and author reputation. But when we compare long-term citations (12 to 18 months after publication) on the same research question in the same period, a good part of the main confounders, such as how hot the field is and when a paper came out, cancels out. What remains mostly reflects whose method was adopted, whose problem framing was carried forward, and whose results kept being compared against, which is much closer to the value of the idea itself.

**Long-term citations are out of reach** To avoid contamination we could only use papers from September 2025 onward, and when we pulled citations in May 2026, they had been out for only 4 to 8 months. Waiting a year to run the evaluation isn't realistic, because by then new models' knowledge cutoffs would cover these papers: trustworthy labels need time, while a valid evaluation needs papers that are new. We also tried weighting early citations by their context to approximate long-term citations (for example Background 0.5, Comparison 1, Use and Extension 1.5), but the citation contexts from the Semantic Scholar API [25] had too many gaps, the correction didn't work well, and we dropped it.

**Backtesting on historical data** Instead we tested directly: can the raw early citation ranking agree with long-term citations under certain conditions? We took 29,135 AI papers posted to arXiv between January and May 2024, clustered them the same way as the benchmark, split them into monthly cohorts, paired papers within each cohort, and checked whether the citation ranking 4 to 8 months after publication matched the ranking at 12 months. The finding: when two papers' citation counts $c_a$ and $c_b$ meet both of the following conditions, the early ranking is already trustworthy:

$$|c_a - c_b| \ge 4 \quad \text{and} \quad |\log(1+c_a) - \log(1+c_b)| \ge 0.5$$

The absolute gap makes sure the difference isn't down to one or two chance citations; the log gap makes sure the difference is large relative to the base. Neither works alone: the log gap alone lets in pairs like 0 vs. 1 or 1 vs. 3, which differ by only one or two citations, and the absolute gap alone lets in pairs like 40 vs. 44, where the relative gap is tiny. At the 4-month window, requiring both gives the highest agreement, with coverage almost the same as the absolute gap alone:

| Filtering rule | Coverage | Agreement with 12-month ranking | Ranking flip rate |
|---|---|---|---|
| Absolute gap only, $d \ge 4$ | 37.9% | 96.9% | 2.5% |
| Log gap only, $\text{log\_gap} \ge 0.5$ | 63.2% | 93.1% | 5.4% |
| Both | 37.2% | 97.3% | 2.1% |

As the observation window grows, agreement rises from 97.3% to 99.4%, and coverage grows from 37.2% to 49.9% of pairs:

| Observation window | Pairs selected | Coverage | Agreement with 12-month ranking | Ranking flip rate |
|---|---|---|---|---|
| 4 months | 2,910 | 37.2% | 97.3% | 2.1% |
| 5 months | 4,439 | 42.3% | 97.8% | 1.6% |
| 6 months | 5,897 | 45.7% | 98.3% | 1.2% |
| 7 months | 7,248 | 47.9% | 99.0% | 0.7% |
| 8 months | 8,667 | 49.9% | 99.4% | 0.3% |

So within a cohort, for paper pairs that meet both conditions, citation count can serve as the label for *which one is better*.

### Two benchmarks from the same cohorts
These cohorts and the citation rule make up two benchmarks, one for each use of the pairwise ranker: the calibration benchmark is a set of citation-labeled human proposal pairs that tests the ranker's own reliability; the ideation benchmark is a set of cohorts that measures how many contemporary human proposals an AI proposal can beat.

|  | Calibration benchmark | Ideation benchmark |
|---|---|---|
| Purpose | Calibrate the ranker | Evaluate ideation systems |
| Unit | Paper pairs from the same topic, published ≤ 31 days apart | A 31-day cohort within a topic |
| Label | Order given by the citation rule | None needed; the ranker compares one by one |
| Size | 260 pairs covering 37 topics | 91 cohorts covering 40 topics |

**Calibration benchmark** Pairs of papers from the same topic published no more than 31 days apart, kept only if they meet the citation rule from the previous section; the more-cited paper wins. So that calibration results carry over to ideation evaluation, we kept its topic distribution close to the ideation benchmark: the first version had 260 pairs covering 37 of the ideation benchmark's 40 topics (the next chapter explains why we later adjusted it).

**Ideation benchmark** Starting from 70 active topics hand-picked from the clustering tree, we cut 31-day cohorts. To keep the human proposals' quality up, we removed zero-citation papers; to give each cohort enough proposals to compare against, we kept only cohorts with at least 10 papers. That left 91 cohorts covering 40 topics. Each cohort's cutoff $t$ is the day before the earliest public date of any paper in it. At evaluation time, the ideation system gets the cohort's research direction and $t$, writes one proposal, and the ranker compares it with every human proposal in the cohort; the win rate is the system's score.

## The judge was grading topics, not ideas
With a benchmark and citation labels in hand, the next step was to build the judge and calibrate it on historical data. The first round of calibration looked good, but the deeper we dug, the murkier the results became.

### A three-stage ranker with a non-compensatory rubric
As described above, the idea evaluator is a pairwise ranker: given two proposals and a cutoff date $t$, it decides which has the larger $V(H, K_t)$ under the same $K_t$. With GPT-5.4 and Opus 4.6 as base models, we split the evaluation into three stages:
1. **Literature Review Agent**: surveys the pre-cutoff literature on the research direction the two proposals share and writes a reusable literature review, an explicit representation of $K_t$;
2. **Parallel Analysis Agents**: starting from that review, run additional searches for each proposal separately and produce two analysis reports, each analyzing one proposal's incremental value under $K_t$;
3. **Judge Agent**: reads the literature review and both analysis reports and makes the final call according to the rubric.

![The Literature Review Agent builds a shared literature review from before t; two analysis agents examine proposals A and B against it; the Judge reads the review, both reports, and both proposals, and outputs which proposal is better](figures/07-ranker.svg)

The rubric was our main prompting lever for calibrating the judge. It breaks $V(H, K_t)$ into seven factors, asks the judge to compare the two proposals head-to-head on each, then combine them into A ≻ B or A ≺ B. Because $V$ is an expected gain, the factors cover both an idea's value if it succeeds and the probability that it gets realized:

1. **Demand**: does the research community care about this problem?
2. **Marginal Contribution**: how much does it add beyond the work the community has already absorbed?
3. **Realization Probability**: how likely is it that its core claims actually pan out?
4. **Informative-Negative Value**: if the hypothesis turns out to be wrong, is the failure still informative?
5. **Adoptability**: if it succeeds, can later work use it or build on it?
6. **Experimental Credibility**: can its experimental design produce evidence the community will find convincing?
7. **Cost or Risk**: are the resources it needs and the risks it carries under control?

These seven factors can't compensate for one another; they are stages an idea must pass through, in order, to create value: if any one is close to zero, $\Delta K$ is close to zero. Duplicating existing work means zero gain, an unreliable premise means the result can't be realized, and a weak experimental design means no credible evidence; a high score on one factor can't make up for a fatal flaw in another.

### The first calibration looked like a success
On the first calibration benchmark, the ranker agreed with the citation ranking about 75% of the time, well above random and the "always predict A ≺ B" baseline. We split the 260 pairs into a 60-pair dev set and a 200-pair test set (three runs on test):

| Setting | Data | Accuracy |
|---|---|---|
| Random | dev | 50.0% |
| Always predict A < B | dev | 55.0% |
| Ranker (GPT-5.4) | dev | 78.3% |
| Ranker (GPT-5.4) | test | 76.0% / 75.5% / 74.5% |

Bucketed by the citation gap between the two papers, all 15 pairs with a gap above 50 were judged correctly, as expected: the bigger the citation gap, the clearer the quality difference between the two proposals. Below a gap of 50, though, accuracy didn't rise steadily with the gap:

| Citation gap | 0–10 | 11–19 | 20–49 | 50+ |
|---|---|---|---|---|
| Pairs | 42 | 77 | 66 | 15 |
| Accuracy | 76.2% | 67.5% | 78.8% | 100% |

### Removing the literature review, or even the core idea, didn't hurt
Digging further, this 75% agreement depended neither on the three-stage structure we designed nor on the parts of a proposal that carry the idea.

**Without the literature review and analysis, accuracy holds** With only the Judge Agent reading the two proposals directly, accuracy matched the full three-stage ranker, consistently across GPT-5.4 and Opus 4.6:

| Configuration | Accuracy |
|---|---|
| Three-stage ranker + GPT-5.4 | 75.5% |
| Judge Agent only + GPT-5.4 | 76.0% |
| Judge Agent only + Opus 4.6 | 76.0% |

By our design, without a shared, trustworthy literature review the judge has no way to assess a proposal's gain under $K_t$, so accuracy should have dropped.

**With a better-balanced calibration set, accuracy went up** We first suspected the benchmark itself: most of the 260 pairs were dated September 2025, right after the two base models' August knowledge cutoffs, so the models may have already known the field well. We reselected pairs with the same filtering rule and built a 250-pair test set with a more balanced spread of dates, and accuracy rose above 80%.

**Without the core hypothesis and method, accuracy holds** We suspected that proposals reconstructed from real papers carry some quality signal unrelated to the idea. Ablating the proposals section by section, both models stayed around 79–83% across input combinations, and were even slightly higher with the core hypothesis and method removed:

| Input | GPT-5.4 | Opus 4.6 |
|---|---|---|
| Random | 50.8% | 50.8% |
| Core hypothesis + method + experiments | 79.2% | 82.0% |
| Core hypothesis + method | 78.8% | 79.6% |
| Full proposal | 82.4% | 80.4% |
| Full proposal minus core hypothesis and method | 83.2% | 80.8% |

We think part of the reason is redundancy across sections: delete the core hypothesis, and it still shows up indirectly in the Introduction and the experimental design. We also suspect that a high-quality proposal already writes in enough domain knowledge on its own, which may be why the literature review added nothing.

### Titles alone are enough for 73%
The most extreme ablation gave the judge only the two proposals' titles, and accuracy was still about 73% (72.8% for GPT-5.4 and 74.0% for Opus 4.6). Checking its reasoning, the judge really was judging the problem area and how likely the method was to spread. Take *Monet: Reasoning in Latent Visual Space Beyond Image and Language* [26] versus *CoCoVa: Chain of Continuous Vision-Language Thought for Latent Space Reasoning* [27]: from the titles alone, the judge decided the second one "names a more concrete and citeable research object", and it was right.

| Input | GPT-5.4 | Opus 4.6 |
|---|---|---|
| Random | 50.8% | 50.8% |
| Titles only | 72.8% | 74.0% |

We take this to mean that, under our rubric, the judge on this benchmark was mostly judging topic choice and spreadability: is the direction hot, is the contribution easy to cite, does the name have a nice ring to it. If a judge that reads only titles can solve most of the calibration benchmark, then what the benchmark measures is clearly different from the idea-evaluation ability we actually wanted to test.

### The judge falls for ambitious proposals
So where is the gap? The problem got worse on AI proposals: the judge favored proposals that looked ambitious and easy to adopt, without checking whether they held up.

Meanwhile, we built a set of ideation baselines. Given a cohort's research direction and cutoff date, a CLI agent such as Claude Code or Codex uses a set of literature search tools with a strict cutoff restriction to do a literature review, analyze gaps, propose a hypothesis, design experiments, and finally write a proposal. We produced proposals with five models on each of the ideation benchmark's 91 cohorts, and an Opus 4.6-based judge compared each AI proposal with every human proposal in the same cohort. Win rates were high across the board, but the ranking didn't match the agents' actual capabilities:

| Ideation system | Win rate vs. human proposals |
|---|---|
| GLM-5.1 + Claude Code | 78.8% |
| Opus 4.6 + Claude Code | 75.5% |
| DeepSeek-V4-Pro + Claude Code | 69.4% |
| GPT-5.4 + Codex | 54.6% |
| Kimi-K2.6 + Claude Code | 50.7% |

Take DeepSeek-V4-Pro as an example. On a multi-turn agentic RL cohort, it proposed a method for credit assignment in multi-turn interaction and beat all 13 human proposals in the cohort one by one. The judge's reasoning: the proposal tackles a "fundamental algorithmic problem," it's a "drop-in module" with "no auxiliary model," the broad GRPO/DAPO user base could adopt it directly, and so it would earn "durable method citation." But its core mechanism doesn't hold up. It treats the actions taken at the same step index across different rollouts as counterfactuals of one another and uses them to estimate a per-step advantage; yet by step $k$ of a multi-turn interaction, the observation histories of different rollouts have long since diverged. Those actions aren't different choices made in the same state, so they don't form a counterfactual.

{{< iframe src="figures/ideaeval/08-proposal.html" height="600px" title="Counterfactual Step Advantage · DeepSeek-V4-Pro proposal" >}}

The judge was won over by proposals like this because it didn't seriously check feasibility. And the calibration benchmark couldn't expose that weakness: its proposals were all rewritten from real papers, so their feasibility was guaranteed by construction. That is a distribution shift between calibration and real evaluation.

### The judge is at chance on human–AI pairs
To close that gap, we built human–AI pairs to recalibrate the judge. Its accuracy on them was just 49.3%, close to chance, and it systematically favored the AI proposal.

**Labeling** Human–AI pairs have no citations to serve as labels, so they had to be labeled by hand. We went through each AI proposal from the ideation baselines: had existing work already done it, did it contain hallucinations, was the method sound, did the experimental design have flaws? We marked how severe each problem was, attached evidence, and paired the proposal with human proposals from the matching cohort. The few AI proposals that really were good were compared honestly too. In the end we had 75 pairs: the AI proposal wins in 18 and the human proposal in 57, and most of the AI proposals in the second group were marked as having significant flaws.

**Results** We changed the judge to follow our labeling process: first identify a proposal's flaws and judge how severe they are, then make an overall call. Its accuracy was 83% on pairs where the AI proposal wins, but only 39% on pairs where the human proposal wins:

| Label | Pairs | Accuracy |
|---|---|---|
| AI proposal better | 18 | 83% |
| Human proposal better | 57 | 39% |
| Overall | 75 | 49.3% |

**Flaw detection** Across every pair it got wrong, the judge never once marked the AI proposal as having a fatal flaw; of the 15 pairs where annotators found a fatal flaw in the AI proposal, it caught only 8. In individual cases, the judge often read right where the flaw was without checking it; sometimes it even spotted the exact risk the annotators had flagged and still picked the AI proposal. It preferred the AI proposal's complete, polished experimental plan, and it would narrow an AI proposal's shaky, overly broad hypothesis down to a version that could still work, but never gave the human proposals the same benefit of the doubt.

### Nothing we tried helped
None of the later trials beat the original 49.3% result:
- **Reordering the rubric**: audit the proposal's core claims first, then judge soundness and feasibility, then check prior work, and only at the end weigh impact.
- **Forced verification**: require the judge to verify every formula and key signal in a proposal by hand, writing code to run a minimal example when needed. This did change its behavior: the judge started checking dimensions, verifying derivations line by line, and catching some of the labeled flaws, but accuracy didn't improve.
- **Switching base models**: including GPT-5.5, which is more capable but whose knowledge cutoff doesn't meet our requirements.

| Judge Agent | Overall accuracy | AI-better subset | Human-better subset |
|---|---|---|---|
| GPT-5.4 (original rubric) | 49.3% | 83% | 39% |
| GPT-5.4 (restructured rubric) | 45% | 89% | 32% |
| Opus 4.6 (forced verification) | ~43% | 100% | 32% |
| GPT-5.5 (forced verification) | 45% | 67% | 39% |

Our guess is that ideation systems package AI proposals to look novel, ambitious, and high-impact, while human proposals faithfully describe real papers, which are therefore often incremental, crowded, and engineering-heavy; and the judge's job is precisely to predict impact, novelty, and how likely a paper is to be cited, the very dimensions where AI proposals have the edge. Many factors still needed further analysis, but around this point some outside factors came up, and the exploration ended here.

## Afterword
We never managed to calibrate the judge agent to a usable level. The project's paper was almost done, missing only the evaluation numbers for the ideation systems, but a judge that is near chance on human–AI pairs can't produce results anyone could trust.

Looking back at the design itself, this evaluation setup does have real limitations:
1. **Citations are not a perfect signal of an idea's value** Citations also reflect topic heat, how well-known the authors are, their institutions, whether the code is open-sourced, whether the benchmark is widely used, and more. Labels for the human–AI pairs depend on the quality of hand annotation; annotators may differ, and we didn't have the people to cross-check.
2. **Turning papers into proposals brings hindsight with it** Even with every experimental result removed, a reconstructed proposal inherits the finished paper's structure, narrative, and experimental choices. It is more coherent and more conservative than a proposal actually written before the experiments, and its feasibility is guaranteed by construction. So the calibration benchmark can't test the judge's sense of feasibility, and human and AI proposals differ in style on top of everything else.
3. **Pairwise comparison amplifies the judge's systematic preferences** When two proposals are close in true value, which one the judge picks often comes down to its preferences, such as nicer prose or a method it knows better, and that is hard to get rid of.

Beyond that, we framed the question too broadly. When the judge got something wrong, we often couldn't tell whether the problem was in the labels, the paper-to-proposal conversion, the rubric, the retrieval, or the model's own ability. Any of them could have been the cause, and with all that noise mixed together we could neither run controlled analyses and adjustments nor see a clear causal picture, so many failure modes stayed hard to diagnose or verify.

We defined research taste as the ability to estimate $V(H, K_t)$ before running experiments, and believed it shows up in weighing many dimensions together. That's why we didn't take the route of using "a single verifiable dimension as ground truth" and instead tried to fit the overall judgment all at once. In hindsight, that trade-off may be exactly why those works get trustworthy results: cleaner validation signals and clearer failure modes.

This integrated research taste is necessary: it is the key constraint on auto research reaching real quality. We didn't get there this time, but it remains a fascinating ability and a direction worth exploring.

## References

1. Reviewer3. *ICLR Went From 490 Submissions to 62,000 in Ten Years*. 2026. <https://reviewer3.com/evidence/iclr-submissions-2027>
2. Qiong Tang, Tianxiang Sun, Xiangkun Hu, et al. *FARS: A Fully Automated Research System Deployed at Scale*. arXiv:2606.31651, 2026. <https://arxiv.org/abs/2606.31651>
3. Alexander Novikov, Ngân Vũ, Marvin Eisenberger, et al. *AlphaEvolve: A Coding Agent for Scientific and Algorithmic Discovery*. arXiv:2506.13131, 2025. <https://arxiv.org/abs/2506.13131>
4. Jiaxin Wen, Liang Qiu, Joe Benton, Jan Hendrik Kirchner, Jan Leike. *Automated Weak-to-Strong Researcher*. Anthropic Alignment Science Blog, 2026. <https://alignment.anthropic.com/2026/automated-w2s-researcher/>
5. Sikun Guo, Amir Hassan Shariatmadari, Guangzhi Xiong, et al. *IdeaBench: Benchmarking Large Language Models for Research Idea Generation*. KDD 2025. <https://arxiv.org/abs/2411.02429>
6. Yansheng Qiu, Haoquan Zhang, Zhaopan Xu, et al. *AI Idea Bench 2025: AI Research Idea Generation Benchmark*. arXiv:2504.14191, 2025. <https://arxiv.org/abs/2504.14191>
7. Yujie Liu, Zonglin Yang, Tong Xie, et al. *ResearchBench: Benchmarking LLMs in Scientific Discovery via Inspiration-Based Task Decomposition*. Findings of ACL 2026. <https://arxiv.org/abs/2503.21248>
8. Zonglin Yang, Wanhao Liu, Ben Gao, et al. *MOOSE-Chem: Large Language Models for Rediscovering Unseen Chemistry Scientific Hypotheses*. ICLR 2025. <https://arxiv.org/abs/2410.07076>
9. Chenglei Si, Diyi Yang, Tatsunori Hashimoto. *Can LLMs Generate Novel Research Ideas? A Large-Scale Human Study with 100+ NLP Researchers*. ICLR 2025. <https://arxiv.org/abs/2409.04109>
10. Hasan Baig, Hailey Joren, Joe Benton. *TASTE: Can AI Models Judge AI Safety Research Proposals?* Anthropic Alignment Science Blog, 2026. <https://alignment.anthropic.com/2026/taste/>
11. Tao Feng, Yihang Sun, Jiaxuan You. *GraphEval: A Lightweight Graph-Based LLM Framework for Idea Evaluation*. ICLR 2025. <https://arxiv.org/abs/2503.12600>
12. Shuofei Qiao, Yunxiang Wei, Xuehai Wang, et al. *InnoEval: On Research Idea Evaluation as a Knowledge-Grounded, Multi-Perspective Reasoning Problem*. ICML 2026. <https://arxiv.org/abs/2602.14367>
13. Hanane Nour Moussa, Patrick Queiroz Da Silva, Daniel Adu-Ampratwum, et al. *ScholarEval: Research Idea Evaluation Grounded in Literature*. arXiv:2510.16234, 2025. <https://arxiv.org/abs/2510.16234>
14. Simra Shahid, Marissa Radensky, Raymond Fok, et al. *Literature-Grounded Novelty Assessment of Scientific Ideas*. SDP Workshop 2025. <https://arxiv.org/abs/2506.22026>
15. Srujan P Mule, Aniketh Garikaparthi, Manasi Patwardhan. *Teaching Language Models to Forecast Research Success Through Comparative Idea Evaluation*. Findings of ACL 2026. <https://arxiv.org/abs/2605.21491>
16. Guangzhi Xiong, Eric Xie, Corey Williams, et al. *Toward Reliable Scientific Hypothesis Generation: Evaluating Truthfulness and Hallucination in Large Language Models*. IJCAI 2025. <https://arxiv.org/abs/2505.14599>
17. Haokun Liu, Sicong Huang, Jingyu Hu, Yangqiaoyu Zhou, Chenhao Tan. *HypoBench: Towards Systematic and Principled Benchmarking for Hypothesis Generation*. arXiv:2504.11524, 2025. <https://arxiv.org/abs/2504.11524>
18. Bingyang Ye, Shan Chen, Jingxuan Tu, et al. *Proof of Time: A Benchmark for Evaluating Scientific Idea Judgments*. arXiv:2601.07606, 2026. <https://arxiv.org/abs/2601.07606>
19. Jingqi Tong, Mingzhe Li, Hangcheng Li, et al. *AI Can Learn Scientific Taste*. arXiv:2603.14473, 2026. <https://arxiv.org/abs/2603.14473>
20. OpenAI. *GPT-5.4*. OpenAI API documentation. <https://developers.openai.com/api/docs/models/gpt-5.4>
21. Anthropic. *Claude Opus 4.6*. Claude API documentation. <https://platform.claude.com/docs/en/models/opus-4-6/overview>
22. Zhenghai Xue, Longtao Zheng, Qian Liu, et al. *SimpleTIR: End-to-End Reinforcement Learning for Multi-Turn Tool-Integrated Reasoning*. arXiv:2509.02479, 2025. <https://arxiv.org/abs/2509.02479>
23. Siyan Zhao, Mengchen Liu, Jing Huang, et al. *Inpainting-Guided Policy Optimization for Diffusion Large Language Models*. arXiv:2509.10396, 2025. <https://arxiv.org/abs/2509.10396>
24. Priyanka Kargupta, Nan Zhang, Yunyi Zhang, et al. *TaxoAdapt: Aligning LLM-Based Multidimensional Taxonomy Construction to Evolving Research Corpora*. ACL 2025. <https://arxiv.org/abs/2506.10737>
25. Semantic Scholar. *Academic Graph API*. <https://api.semanticscholar.org/api-docs>
26. Qixun Wang, Yang Shi, Yifei Wang, et al. *Monet: Reasoning in Latent Visual Space Beyond Images and Language*. arXiv:2511.21395, 2025. <https://arxiv.org/abs/2511.21395>
27. Jizheng Ma, Xiaofei Zhou, Yanlong Song, Han Yan. *CoCoVa: Chain of Continuous Vision-Language Thought for Latent Space Reasoning*. arXiv:2511.02360, 2025. <https://arxiv.org/abs/2511.02360>