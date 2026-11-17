---
title: "AI Moved the Bottleneck to Code Review"
description: "Here's what the evidence says actually helps."
date: 2026-11-15
---

Earlier this year I watched an experienced team of eight engineers hit a wall in their pull request queue. They maintained a Java, SQL and React monorepo, and until the summer their review throughput was predictable. Then agent adoption across the company took off. Background coding agents opened full, CI-passing PRs, more of them, and more often in parts of the stack the team didn't usually touch. Each diff was clean and green, and reviewing one properly meant first working out what it was for.

So the team triaged by rough signals: who wrote the change, and how risky it looked. Small diffs got a quick look; larger or less familiar ones waited for days. Code generation had sped up, but verification had become the bottleneck.[^story]

It wasn't just that team. At one AI-forward company studied this year, PRs grew **3.1×** while human reviewers grew **1.5×**. By the end, **AI bots reviewed 84% of PRs and humans 68%**, and the median human reviewer roughly doubled their silent approvals, while the number of PRs they commented on stayed flat.[^load] Across 655 firms, adopting coding agents raised pull requests opened by about a third and time from submission to merge by 56%.[^jellyfish]

Pointing an AI at the queue keeps it moving, but the hard part was knowing each change was right. The team I watched tried it, with mixed results, and that sent me to the evidence: more than 170 sources, 49 read in depth, from papers and benchmarks to production write-ups and security disclosures. The short answer: **AI review can work, but not the way most teams deploy it.** What helps is the system around the model: deterministic checks first, and a written record of why each change exists, so human reviewers can judge whether a change is right instead of reconstructing what it was for.

## The naive version fails in three ways

**1. It lacks the context behind the diff.** Even an agent that explores the repository can't recover the decisions behind a change: what it's for, what was agreed and why. Left without them, the AI reviewer guesses. In one benchmark setup, nearly half of false positives came from missing context.[^swr] Atlassian found that the team's written guidelines did more for its AI reviewer than any other input it tested.[^atlassian] In another benchmark, the PR description helped an AI reviewer more than the surrounding code did.[^contextcr]

**2. It uses an LLM for jobs a linter does better.** Style, formatting and known-bad patterns are deterministic problems. Paying a language model to guess at them adds noise and cost. Uber found that readability nits and stylistic issues "consistently received poor ratings" from its developers, and concluded that for simple or syntactic patterns, "linters are accurate, reliable, and cheap."[^ureview]

**3. It floods the PR thread.** Every comment costs the author time to read and answer, and many of these weren't worth it. In a study of 31,000 CodeRabbit comments, **56% of comments that received a developer reply were rejected**, mostly because they were false positives or argued against deliberate design choices.[^coderabbit]

## Where the evidence points: a layered design

None of the teams with the strongest published results (Uber, Google, ByteDance, Atlassian, Anthropic) states it this way, but read side by side, their designs converge on the same idea:[^deployments]

> **Deterministic where you can, agentic where rules run out, human where it's a judgement call.**

- **Deterministic checks** for anything you can state as a rule: linters, type checks, static analysis, dependency and layering rules.
- **Agentic review** for bugs a rule can't describe, with real repository context, a separate verification step, and strict filtering so only findings worth a human's time get through. Precision first.
- **Humans** for intent, design trade-offs, risk acceptance and teaching, brought in according to risk rather than on every change.

Anthropic's version requires zero, one or two human approvals depending on the change's risk, and an agent's approval never counts as a human's. Over the same period that merged PRs more than doubled, the share of PRs later reverted fell by more than 40%, though that covers the whole delivery pipeline, not the AI reviewer alone.[^anthropic]

Not every change needs human review. For docs and other low-risk changes an agent-first process may be perfectly fine. For security-sensitive or architecture-changing work it clearly isn't. Rachel Laycock of Thoughtworks calls this "review by exception": architectural alignment moves upstream into joint design and fitness functions, and human review is kept for the changes that need it, such as architecture, security boundaries and anything the team isn't confident about. The real decision is *which change gets which process*. Nobody has yet published whether routing like this holds up; that's one of the questions at the end.[^routing]

Here's my version.

Every change gets the same baseline:

- **Deterministic checks:** lint, type checks, tests, static analysis, security scans and architecture rules.
- **Agent review** with repository context and the linked ticket.
- **The agent's assumptions**, on agent-written PRs, listed in the PR description, with any it's unsure of flagged. Agents rarely flag uncertainty unless asked, so ask (more on why below). Where no human approves, the reviewing agent checks them.

| Change                                         | Adds to the baseline                                                                                                      | Human approval                                                               |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Low risk: docs; additive changes behind a flag that's off by default | Nothing                                                                                                                   | None if the person directing the agent is on the trusted list; one otherwise |
| Behaviour changes not behind a flag            | The reviewing agent checks the change against the ticket's acceptance criteria, and says so when there aren't usable ones | One person, checking intent and the assumptions, not just code; with no usable acceptance criteria, the intended behaviour gets written down before approval |
| Security-sensitive                             | A dedicated security review whose findings block the merge                                                                | Two people on a fixed list of paths; one otherwise                           |
| Large blast radius, unfamiliar critical code   | An AI summary of what depends on the changed code                                                                         | The code owner, plus an affected consumer owner where the impact warrants it |
| Architectural trade-offs, new boundaries       | An AI summary of which boundaries and dependencies the change touches                                                     | The design owner, before merge; ideally agreed before the code is written    |

Some rules apply to every row:

- **The starting tier comes from rules:** paths, code owners and what the diff touches. Security-sensitive paths are a fixed list. A new module or a new dependency between modules goes to the architecture row; a change to a public API or database schema counts as large blast radius; critical paths the team hasn't changed in months count as unfamiliar.
- **Rules won't catch everything, so people can escalate.** Anyone can move a change up a tier, never down: anything a person isn't confident about goes up, and so does any assumption the agent flagged as uncertain that touches behaviour, security, data or boundaries. The reviewer judges that, not the agent.
- **If several rows match, the change gets all of their checks and approvals.** One person can fill more than one role.
- **A broken architecture rule blocks the merge.** Humans step in only to approve an exception or change the rule.
- **Whoever directed the agent doesn't count as a reviewer.** So a low-risk change from a trusted person can merge without any independent human seeing it. The baseline checks and fast rollback are what make that acceptable, and the trusted list is one the team keeps, for people who know the code and know when to escalate.
- **An approval covers one commit.** New pushes need approval again.
- **No human approval only works where rollback is fast:** a revert, feature flags, staged release.
- **Turning a flag on is a behaviour change.** It goes through the behaviour row, even when it's a config switch rather than a PR.

## Review needs the why, not just the diff

I've seen this first-hand. In pipelines that already had solid linting and static analysis, I pointed an agent at pull requests and asked it to review them. It wrote plenty of comments, but few were worth acting on: talking it over with the team, I'd put it at about one in ten, and sorting those from the rest took time. We hadn't measured this formally, but the pattern was clear. Giving it the ticket wouldn't have helped much: the tickets were sparse or stale, and the real decisions had been made on calls, in Slack, or on wiki pages nobody linked. What was missing was the why.

Reviewers have always needed to understand what a change is meant to do before judging it: a Microsoft study of hundreds of developers found that "context and change understanding" is the key to any review.[^bacchelli] Human reviewers still spend much of their effort here. In a large study of GitHub review conversations, human feedback "focuses more frequently on understanding and clarifying developer intent": such questions made up 17–31% of human reviewers' comments, while over 95% of AI reviewers' comments were code improvements or defect reports.[^zhong] Agents make this harder. A developer who lacks context usually asks, or flags the doubt: a question on the ticket, a "not sure about this part" in the PR. Unless prompted to flag uncertainty, an agent can fill the gap with a plausible assumption and keep going, presenting the guess as if it were information it had been given. The reviewer inherits the guess, and a reviewer, human or AI, who sees only the diff can't catch a requirement nobody wrote down.

Other studies agree. In one benchmark, giving an AI reviewer the PR description improved it far more than giving it the surrounding code (+26 F1 points against +16).[^contextcr] On agent-written PRs, about a quarter of human review comments are instructions to the agent: people steering the agent during review, the most expensive point to do it.[^duma] Meta's research prototype addresses this directly. Its AI reviewer predicts what the developer intended and flags where the agent drifted: instructions it didn't follow, and changes nobody asked for.[^arctic]

Give review the why, then. **Improve the input:** a clear task description, acceptance criteria and team guidelines (the description is what lifted the benchmark above; guidelines are what mattered most at Atlassian), so the agent and its reviewers share one definition of "right". When a decision is made on a call or in Slack, put a two-line summary in the ticket or PR description, or it's invisible to every reviewer. That's what the agentic layer needs most. **Make assumptions visible:** ask the agent to list the assumptions it made in the PR description, and have the human reviewer check those against what the business actually needs. That's what "human where it's a judgement call" means in practice, and it's a better use of a human's attention than re-checking what the deterministic checks and the agentic reviewer already cover.

Getting the why right before review even starts is a bigger subject, for another time.

## Protecting your architecture: the layers in practice

As an engineer, I'm increasingly concerned about something more systemic than individual bugs: slow architectural drift. Duplicated capabilities, leaked boundaries and dependencies that shouldn't exist add up across a hundred agent-written PRs a day.

The research supports one of these jobs better than the other:

- **Checking compliance with explicit decisions looks promising, but not yet at the PR level.** An ICSA 2026 study had models judge whether codebases complied with their architecture decision records (ADRs). On 305 decisions checked by hand, the best model reached 91% accuracy (MCC 0.84, the fairer measure here). The misses clustered in infrastructure, intent and cross-module decisions.[^adr]
- **Evidence that AI can provide useful open-ended design review is much weaker.** At Meta, design made up 16% of a sample of an AI reviewer's comments, but only 3.3% of the AI comments engineers acted on, about the same share as for human comments (3.6%). Nobody has shown that AI design comments are useful more often than that. The authors suspect many were too subjective to act on, but they didn't measure it.[^arctic]

In practice this is the layered design at a smaller scale:

1. **Encode the invariants** as executable rules (import-linter, ArchUnit, fitness functions).
2. **Let the deterministic check find the violation.**
3. **Use the LLM to explain it** in context and flag what *might* matter.
4. **Keep real design discussion with humans, and move it upstream** into design sessions, before the code exists.

"Does this follow our architecture rules?" and "Is this a good design?" are different questions. AI is showing promise at the first, particularly when the rules are explicit. The second one is still your job.

## Where to start

This is my reading of the evidence, in the order I'd do it. It's well-motivated, not proven: the next section says how to tell whether it's working, and the last one lists the questions still open.

1. **Turn on deterministic checks first**, including architecture rules, before any AI reviewer.
2. **Give the AI reviewer context**: PR description, linked ticket, team guidelines, relevant design docs. Put them in front of it; don't just make them reachable. Include the decision summaries and the agent's list of assumptions from above.
3. **Filter hard.** Fewer, verified findings beat more findings. Check each finding before it's posted, switch off comment categories developers keep ignoring,[^ureview] and gate on whether a comment is likely to be acted on.[^atlassian] Then measure comments per PR and what share are worth acting on.
4. **Route by risk.** Decide which changes need zero, one or two humans, and never let an agent's approval, or the person who directed the agent, stand in for an independent reviewer.
5. **Lock down the AI reviewer.** It's an attack surface in its own right. Load its configuration only from your protected base branch, give the model that reads PR content no secrets and no write access, and let a separate, deterministic step post its findings.
6. **But don't neuter it: let it run the code.** Build it, run the tests, and write scripts that reproduce a suspected bug, all in a throwaway sandbox with no credentials and no open network. Turning "this might be wrong" into "here's a failing case" is one of the most promising approaches I've found, though it's untested on real PRs (see the open questions at the end).
7. **Make sure "no findings" means the AI reviewer actually ran.** One team found its reviewer still posted a confident, well-formatted review when its sandbox couldn't execute a single command.[^ionix] Fail loudly when the reviewer couldn't run; never let that look like a clean PR.

## How you'll know it's working

**"Accepted", "resolved", "led to a code change" and "made the software better" are four different things**, and figures for AI review rarely say which one they mean. Vendors report that 66–80% of their own tools' comments are "addressed".[^vendor] Studies that look across tools find that 33–36% led to a code change.[^independent] Both can be true. A resolved thread doesn't mean anything was fixed, and a later commit touching the flagged line doesn't mean the comment caused it.

Offline benchmarks don't settle it either. Atlassian's AI reviewer matched human reviewers' comments only **4%** of the time in offline tests, yet **39%** of its comments led to code changes once it was deployed.[^atlassian] Matching what a human would have said is a poor proxy for being useful.

Decide up front what you're measuring, and track these side by side:

- **Acceptance:** the share of comments that led to a code change, and comments per PR.
- **Outcomes:** reverts, escaped defects and incidents.
- **Flow:** time from opening a PR to merging it.
- **Human attention, where you routed it:** on the changes your routing sends to a person, did they engage with the intent and the agent's assumptions, through a question, a comment or an explicit sign-off? Silent approvals on low-risk changes are the design working. Silent approvals on behaviour changes or security-sensitive code mean the gate has become a rubber stamp. The company in the introduction shows the pattern to watch for: silent approvals rose while reviews with comments stayed flat.

## Back to the team of eight

For that team, I wouldn't start with a better reviewer. I'd start by making the intent part of every agent-written PR: what outcome the task is meant to achieve, its acceptance criteria, and the assumptions the agent made along the way. The time reviewers spent reconstructing the purpose of each diff could go on deciding whether it was right.

Written intent lets reviewers spend their attention on the decisions no rule can make.

## Over to you

If you run an AI reviewer in production, you can answer the open questions better than any benchmark can, and most of us are tuning these systems by gut feel. These are the ones I'd most like to see answered:

- **Did it reduce escaped defects?** Reverts, incidents and defects found after merge, compared with how you reviewed before.
- **Did routing by risk hold up?** If low-risk changes merge with no human approval, did reverts or incidents go up?
- **Does letting the reviewer run the code help?** In one benchmark study, AI reviewers often flagged correct code as buggy. Making them back each rejection with a fix, then running both versions against the existing tests plus generated ones, threw out more than half of those false alarms.[^overcorrection] Nobody has reported it on real PRs.
- **Does architecture context help?** Did feeding it ADRs, module maps or import rules change what it caught?

Write up what you find, even if it's a short post. I'd like to read it.

---

## Notes and sources

*Most figures here come from 2025–2026 studies, several of them preprints or vendor reports. Each footnote names the venue or says what kind of source it is (preprint, company blog, vendor data). The field moves fast; this reflects the evidence as of November 2026.*

[^story]: The data partly matches what I saw. At the AI-forward company in the next paragraph, median PR size rose from 35 to 61 lines as AI use grew (He, Vasilescu et al., below); across 655 firms, though, PR size didn't change significantly after agent adoption (Chen & Stratton, below). In open source, 67.5% of AI-co-authored PRs came from contributors who didn't own the code they changed, against about 61% of human-only PRs (Gao et al., ["On Autopilot?"](https://arxiv.org/abs/2601.13754), MSR 2026). And reviewers mostly stick to code they know: across those 655 firms, 82.8% of reviews were by someone who had already worked in that code base, which helps explain why unfamiliar changes wait.

[^load]: He, Vasilescu et al., ["AI Writes Faster Than Humans Can Review"](https://arxiv.org/abs/2607.01904) (2026, preprint). One company: 802 developers and about 196K PRs, January 2024 to April 2026. The authors call it an upper bound, not a typical case. Review coverage is descriptive: human review fell from 89% to 68% of PRs while AI-bot review rose from about 19% to 84%. Late in the period, the firm also routed some PRs through AI review plus auto-approval. Silent approvals roughly doubled per median human reviewer, while commented reviews held at about three a month. The share of PRs with substantive human feedback fell from about 39% to 21%. The authors read the flat revert rate as showing that reverts are a poor guardrail once the review gate itself changes, not as evidence that quality held.

[^jellyfish]: Chen & Stratton, ["Artificial Intelligence in the Firm: Bottlenecks in Software Production"](https://fion.ac/jellyfish.pdf) (Harvard job market paper, version of 14 Nov 2026; not peer-reviewed). The design is a staggered difference-in-differences on when firms adopted AI tools, using data from Jellyfish, an engineering analytics vendor: 720,542 workers at 655 firms, January 2021 to August 2026. The firms are Jellyfish clients that opted in to research use, mostly medium-to-large and established. Pull requests opened rose 35.7%, and time from submission to merge rose 56.1% (1.99 days on a 3.56-day baseline). Resolved Jira issues rose 13.7%, which wasn't statistically significant. Earlier versions and press coverage give different figures. Jellyfish reviewed the manuscript only for proprietary information.

[^contextcr]: Hu et al., ["ContextCRBench"](https://arxiv.org/abs/2511.07017), FSE 2026 (Industry). A triage task (does this diff hunk need further review?), not full review: average F1 for closed models went from 36 (diff only) to 62 with the PR description, and to 52 with surrounding code only. The textual context came from PRs linked to issues, which tend to have unusually good descriptions.

[^bacchelli]: Bacchelli & Bird, ["Expectations, Outcomes, and Challenges of Modern Code Review"](https://microsoft.com/en-us/research/?p=164195), ICSE 2013. Observations, interviews and surveys of developers and managers at Microsoft.

[^zhong]: Zhong, Noei, Zou & Adams, ["Human-AI Synergy in Agentic Code Review"](https://arxiv.org/abs/2603.15911) (2026, preprint). Comments classified by an LLM (κ 0.85 against manual labels); reviewers counted as AI or human by account.

[^swr]: Zeng et al., ["SWR-Bench"](https://arxiv.org/abs/2509.01494), FSE 2026. 1,000 PRs from 12 Python projects. The 48% comes from three authors classifying 100 false positives sampled from one configuration (PR-Review with Gemini-2.5-Pro).

[^atlassian]: Tantithamthavorn et al., ["RovoDev Code Reviewer"](https://arxiv.org/abs/2601.01129), ICSE-SEIP 2026. Atlassian staff evaluating their own tool over 12 months: about 2,000 repositories, 54,000+ comments. The 4% is an offline replay against 2,894 human comments, judged by an LLM. The 39% (38.7%) counts comments whose lines were changed in the next commit; the paper doesn't say how that was detected. The input finding comes from an ablation: removing written guidelines cost the most (5 points), and the PR description and Jira ticket each added 1–3.

[^coderabbit]: Lin et al., ["Is Agentic Code Review Helpful?"](https://arxiv.org/abs/2607.03316) (2026, preprint). Outcomes come from developers' replies, labelled mostly by an LLM (GPT-5.1) calibrated on a hand-coded sample. Only comments with a developer reply were counted, so comments nobody answered are left out.

[^vendor]: Each vendor's own definition: Greptile v5, 66% of comments "addressed" in an A/B test on over 1M PRs ([Greptile, Aug 2026](https://www.greptile.com/blog/greptile-v5)); Cursor Bugbot, 80% of bugs "resolved by merge time", method not stated ([Cursor, May 2026](https://cursor.com/blog/may-2026-bugbot-changes)).

[^independent]: Kodus "State of AI Code Review 2026": 33% of 180,739 suggestions led to a code change ([Kodus](https://kodus.io/data/); vendor telemetry, but across many customers' tools). The CodeRabbit study (Lin et al., above): 36%, labels mostly LLM-assigned.

[^deployments]: Uber, ["uReview"](https://www.uber.com/en-US/blog/ureview/) (2025, company blog); Google, ["AutoCommenter"](https://arxiv.org/abs/2405.13565) (AIware 2024); ByteDance, ["BitsAI-CR"](https://arxiv.org/abs/2501.15134) (FSE 2025 Companion); Atlassian and Anthropic as cited elsewhere in this post. The three-part framing is my synthesis, not theirs.

[^ureview]: Uber, ["uReview"](https://www.uber.com/en-US/blog/ureview/) (2025, company blog). Uber suppresses comment categories with "historically low developer value", such as `readability:naming`. It gives no separate figures for style comments.

[^anthropic]: Anthropic, ["How the Claude Code team does code review"](https://claude.dev/blog/solving-code-review-at-anthropic/) (Oct 2026, company blog). Observational and self-reported; the revert figure covers pre-PR checks, CI, merge queue and staged release together.

[^routing]: My synthesis. The closest precedents are Anthropic's risk tiers and Cloudflare's. Anthropic requires zero, one or two human approvals: docs, copy, internal-only features and additive changes behind a flag need none once automated review and security checks pass, but system prompts count as medium risk and a fixed set of paths always needs two people. The lighter paths apply only to authors on the repo's trusted list, each approval covers one specific commit, and the zero-approval tier sits on top of a merge queue, feature flags and staged release (see the Anthropic note above). One deliberate difference: at Anthropic, the person directing an agent counts as one of the required reviewers; in my version they don't.

    Cloudflare scales review depth by change size and always gives security-sensitive paths such as `auth/` and `crypto/` the full review ([Cloudflare, "Orchestrating AI Code Review at scale"](https://blog.cloudflare.com/ai-code-review), Apr 2026, updated Jul 2026, company blog; it reports cost and latency, not quality). Laycock's "review by exception" is from ["Maybe We Shouldn't Be Reviewing All This Code"](https://martinfowler.com/rachels-ramblings/code-review.html), martinfowler.com, Sep 2026, practitioner essay. The architecture row is my extension of her argument for agreeing design upstream; I know of no published deployment that gates on it. Neither company evaluates its routing.

[^adr]: Su et al., ["Evaluating LLMs for Detecting Architectural Decision Violations"](https://arxiv.org/abs/2602.07609), ICSA 2026 (Distinguished Paper). The 305 validated cases were sampled from 1,317 decisions in 980 ADR files. The best model was Marco-o1; most cases were the easy "compliant" class, so its MCC of 0.84 is a fairer measure than accuracy. It classified repository snapshots, not PRs. The error breakdown covers the 92 cases where at least one of four models was wrong. By type of decision, infrastructure and deployment made up 42%, principle or intent 26% and module interaction 17%. By cause, the largest share was the model misinterpreting the decision's logic (45%), ahead of missing context (28%).

[^arctic]: Maddila, Nagappan, Rigby, ["Code Critique" / ARCTIC](https://arxiv.org/abs/2607.29516) (Meta, 2026, preprint). The 16% comes from a random sample of all AI comments, and the 3.3% from a separate set of AI comments engineers acted on, so the two can't be combined into an ignore rate. Among comments that were acted on, design was 3.3% for AI and 3.6% for humans. Topics were tagged by an LLM. The authors suspect the design comments were too subjective or lacked codebase context, but didn't test that.

[^overcorrection]: Jin & Chen, ["Are LLMs Reliable Code Reviewers?"](https://arxiv.org/abs/2603.00539), [*Automated Software Engineering* (2026)](https://doi.org/10.1007/s10515-026-00638-5). Small benchmark functions (HumanEval, MBPP, QuixBugs), not PRs; the gain is only as good as the tests.

[^ionix]: IONIX engineering blog, ["We built an AI PR reviewer. The hard part was teaching it to say nothing"](https://www.ionix.io/blog/we-built-an-ai-pr-reviewer/) (Aug 2026).

[^duma]: Duma et al., ["These Aren't the Reviews You're Looking For"](https://arxiv.org/abs/2605.02273), EASE 2026. Comments classified by pattern matching; 25.9% of human comments on agent PRs were agent steering, vs 1.6% on human PRs.
