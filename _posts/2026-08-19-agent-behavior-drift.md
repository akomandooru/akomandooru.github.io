---
layout: post
title: "Agent Behavior Drift: Detect Silent Quality Failure"
---
*What it is, why nothing in a normal agent stack catches it, and what to do about it.*

A deployed agent can get worse without producing a single operational signal. No
errors, no exceptions, no latency change, no failed health check. It keeps
responding fluently and confidently, dashboards stay green, and only the *content*
degrades: wrong tool calls, fabricated claims, stale answers, misrouted work.

Any one bad response can happen in a perfectly healthy agent. Drift is when the
*rate* rises and stays elevated.

Here is what that looks like in practice. I deployed a detector against the AWS Market
Trends Agent sample, running on real AgentCore infrastructure with real traffic and real
evaluators, then swapped its model for a much older one and watched a judge evaluator
that scores response tone: its mean fell from
0.68 to 0.33. Not because it started returning invalid scores. Every score it gave, in
both periods, was one the rubric accepts. What changed was the *mix*: good scores got
rarer, bad ones got common. No single response was provably broken. The trend was. That
is the entire problem, in one example: **no individual output is wrong enough to catch,
and the agent is measurably worse than it used to be.**

This is studied, not just anecdotal. Across 22 incidents in one production agent
runtime, each traced to root cause, the dominant recurring pattern was **a failure
whose error signal never reaches a human in actionable form**
([arXiv:2606.14589](https://arxiv.org/abs/2606.14589)).

---

## What it is

Two distinct things get called quality failure, and separating them decides
everything downstream:

| Shape | Question it answers | Example |
|---|---|---|
| **Per-response failure** | Is *this* output bad, on its own terms? | This response leaks PII; this response contradicts the tool output it just received |
| **Behavioral drift** | Has this agent gotten *worse than it used to be*? | Correctness scores have declined slightly every day for two weeks |

Per-response failures are absolute: one output violates a rule you can state in
advance. Drift is relative and statistical: no individual response is provably
wrong, but the trend is.

**This document covers drift.** Per-response failure is well-served territory:
[Amazon Bedrock Guardrails](https://aws.amazon.com/bedrock/guardrails/) for content
and safety policy, and deterministic code-based evaluators (see the
[Market Trends Agent sample](https://aws.amazon.com/blogs/machine-learning/build-custom-code-based-evaluators-in-amazon-bedrock-agentcore/)
for schema, numerical-accuracy, workflow, and PII checks) for contract enforcement.

Within drift, classical ML monitoring distinguishes **data drift** (production
inputs shift away from what the system was evaluated on) from **concept drift** (the
correct output for a given input changes; the world moved and the model is still
answering for the old world).

**Agent behavior drift is a superset of both**, because an agent is not a frozen
model. It is a composition of a foundation model, a system prompt, a tool set,
retrieval sources, memory, and orchestration logic. Its behavior shifts when any
component changes underneath it, usually with no coordinated release. Working
definition:

> A **sustained, statistically meaningful decline** in an agent's quality scores
> relative to a baseline established from that same agent's recent production
> behavior.

Two words carry the load. **Sustained**: a single low score is expected noise (under
a normal distribution roughly 1 in 40 samples falls below −2σ by chance), so drift is
a claim about a trend. **Relative**: measured against the agent's own history, not an
absolute bar.

---

## Why it matters, and why it stays invisible

**Degradation over time is the norm.** A study using datasets from four industries
(healthcare operations, transportation, finance, and weather) and four standard machine
learning models found temporal degradation in 91% of the model-and-dataset pairs tested
([Vela et al., *Temporal quality degradation in AI models*, Scientific Reports 2022](https://storage.ghost.io/c/44/95/449506ca-034e-480f-9725-fcde08ef1cc1/content/files/2025/04/Temporal-quality-degradation-inAI-models.pdf)).
Gartner's analysts describe the agent-specific version of the problem directly: model
updates shipped by a provider can shift behavior in ways that bypass conventional
technical detection, with commercial consequences arriving before anyone notices the
cause ([Gartner](https://www.gartner.com/en/articles/ai-model-updates-risk)). A 2026
StackGen study found AI now
accounts for roughly 1 in 10 production incidents, a 6x rise in three years, and
identifies silent quality failure as the fastest-growing category, from 1
customer-facing AI quality incident in 2025 to 89 in 2026 year-to-date among
AI-native companies
([BusinessWire](https://www.businesswire.com/news/home/20260804183296/en/AI-Now-Accounts-for-1-in-10-Incidents-a-6x-Rise-in-Three-Years-and-AI-Agents-Have-Destroyed-Live-Company-Systems-on-Their-Own-StackGen-Study-Finds)).
Failure costs at mid-market companies run $50K-$500K+ per incident through churn,
refunds, fines, and cleanup ([RaftLabs](https://www.raftlabs.com/blog/cost-of-ai-failure)).

### It is invisible for structural reasons, not because monitoring is bad

Four mechanisms compound:

1. **Fluency masks failure.** A broken traditional system emits a stack trace or a
   500. A wrong LLM emits grammatical, confident prose. The signature every
   monitoring tool is built to spot, output that *looks* broken, never appears.
2. **Confidence is uncorrelated with correctness.** There is no hesitation signal to
   alert on, and users routinely read confidence as correctness, which suppresses the
   human detection channel.
3. **No ground truth exists at inference time.** Classical monitoring compares
   predictions to labels. For an open-ended agent response the correct answer is not
   available at response time; if it were, you would not need the agent.
4. **Human feedback lags and misfires.** Users blame their own phrasing, retry, or
   quietly leave. Complaints arrive weeks after onset. Spot checks sample uniformly
   while early drift concentrates in a shifting slice of traffic.

The consequence is that **operational health and output quality are independent
axes**, and standard observability instruments only the first. AWS's own AgentCore
Evaluations material makes the point directly: dashboards can read green on latency
and error rate while the experience degrades because the agent has started choosing
wrong tools or giving less helpful answers
([AWS blog](https://aws.amazon.com/blogs/machine-learning/build-reliable-ai-agents-with-amazon-bedrock-agentcore-evaluations/)).

### None of the causes produce an error

| Cause | Example | Who notices first |
|---|---|---|
| Foundation model update | Provider updates the model behind the same model ID; tool-call fidelity or refusal behavior shifts | Nobody, initially |
| Prompt change | A "small" system prompt edit ships; the agent starts skipping a required step | Users |
| Tool/API change | Upstream schema changes; the agent misparses and weaves wrong data into fluent answers | Users, or an incident |
| Retrieval decay | Indexed documents go stale; the agent answers confidently from old facts | Users |
| Memory accumulation | Long-lived memory accretes contradictory entries that steer later behavior | Nobody, for a long time |
| Input population shift | Users start asking about a product line the agent was never evaluated on | Support teams |
| Dependency bump | An SDK upgrade changes orchestration defaults (temperature, retry, tool choice) | Nobody, initially |

### And the damage is not linear

Three effects shape the curve:

- **Compounding across steps.** Agent tasks are chains and reliability multiplies. At
  probability *p* per step an *n*-step task succeeds at roughly *pⁿ* (Lusser's law):
  95% per-step accuracy yields about 59% over 10 steps and about 35% over 20
  ([Mirantis](https://mirantis.com/blog/why-multi-step-ai-workflows-fail)). So a
  *small* per-step decline, the kind no per-response check flags, is amplified
  multiplicatively at task level. A 2-point per-step drop can take double digits off
  end-to-end success. Research on agentic workflows confirms reliability degrades as
  sequences lengthen, with minor errors compounding silently
  ([arXiv:2603.00532](https://arxiv.org/abs/2603.00532)).
- **Self-conditioning.** Models make more mistakes when their context already
  contains their own earlier mistakes, beyond what long-context limits alone explain
  ([arXiv:2509.09677](https://arxiv.org/html/2509.09677v1)).
- **Memory feedback.** For agents with persistent memory, today's degraded output is
  tomorrow's context, turning a slow decline into an accelerating one. This never
  shows up in test, because test environments do not carry weeks of accumulated state.

The practical shape is **a long shallow plateau of "slightly worse" followed by a
visible cliff.** During the plateau every interaction still looks acceptable and
every spot check passes. By the time it is user-visible, the decline has been running
for days or weeks. **The window where detection is valuable is precisely the window
where nothing looks wrong.**

### It is also becoming a compliance obligation

Under the EU AI Act, providers of high-risk systems must run post-market monitoring:
Article 72 mandates a systematic process to actively collect and analyze performance
data across a deployed system's lifetime, explicitly not a launch checkbox
([Article 72](https://artificialintelligenceact.eu/article/72/);
[Openlayer guide](https://www.openlayer.com/blog/eu-ai-act-post-market-monitoring-requirements)),
and Article 17 requires validation procedures before, during, *and after*
development ([Article 17](https://artificialintelligenceact.eu/article/17/)). For
agents in or near high-risk categories, documented drift detection is a compliance
line item. Similar expectations appear in medical device and financial model-risk
regimes.

When a silent failure becomes an incident, the questions are governance questions:
when did quality begin degrading, when could you reasonably have known, and what did
you do in between. Without quality telemetry the honest answers are "unknown,"
"unknown," and "nothing." There is a specifically agentic version of this too: if a
managed model update degrades your agent, the provider changed the component but you
own the customer impact, and "the model changed underneath us" is only a defense if
you can show you detected it and responded.

---

## What to do about it

*Two sources sit behind everything below, and they are labelled where it matters: a
controlled offline comparison of five detection methods against synthetic score
streams of known shape, and a detector deployed against eight real evaluators on a
live agent. Where they disagreed, the deployment won.*

### 1. Decide whether you need it

Two situations genuinely reduce the need, and both are narrower than they look.

First, when the stakes of sustained degradation are low *and* a strong human
detection channel exists: expert internal users who verify outputs before acting, in
a short feedback loop with the owning team. The humans are the detector. Note this is
not the same as "occasional wrong answers are tolerable": drift is never occasional,
it is a rising rate, so the question is whether degradation would be caught and would
matter.

Second, when the system is fully pinned *and* its ground truth is static (a
formatter, a classifier over a stable taxonomy). CI/CD regression testing then covers
most of the risk. This class is smaller than it appears: pinning freezes the system,
not the world, so any task measured against current reality still drifts, and full
pinning is only partly achievable anyway (snapshots get deprecated, serving stacks
change beneath frozen weights, third-party APIs change server-side).

Everyone else: any agent serving users or driving decisions, whose composition
changes on a cadence you do not fully control.

### 2. Rule out the three cheaper alternatives

Everything after this step is work. Before committing to it, rule out the three
things most teams reach for first, because none of them detect drift.

**A tighter evaluator threshold.** Recall the 0.68-to-0.33 example from the opening: a
threshold examines one score at a time, so a shift in the *mix* of an otherwise legal
distribution is invisible to it by construction. Set it high and it fires constantly on
healthy traffic; set it low and a large real shift walks underneath. It also has no
memory, so one unlucky response and a month of decline look identical. An absolute bar
cannot express a relative claim, wherever you put the bar.

**Scheduling the optimization loop.** Scheduling changes when it runs, not what it
compares. An A/B test measures both arms over the same window, and that concurrency is
what makes it clean *and* what makes it blind to change over time: if quality fell last
Tuesday it fell for both arms and the comparison reports nothing. Run it hourly and you
get "B beats A" forever, never "A is worse than A was last month." Worse, a loop that
promotes winners keeps re-tuning the prompt to compensate for whatever is degrading
upstream, so scores stay flat while the cause compounds. That is an automated cover-up.
A baseline needs a configuration that holds still.

The corollary matters later, when the alarm fires: **detection is the trigger,
optimization is the remediation, with a human in between.** From a real case, pointed
at a window where quality had dropped because a model had been accidentally downgraded,
a prompt recommendation returned a rewrite forbidding malformed nested tool calls. It
never mentioned the model and offered no explanation. Applying it would have "worked,"
leaving a fragile patch on the wrong model and no record that either fact existed.
Optimization compensates for a cause; it does not name one.

**Blocking responses inline.** This is a real and useful thing to build, and a
different job. An inline gate answers "is this one output bad," which catches an
isolated bad response a drift detector deliberately ignores. It cannot answer "has the
process shifted," because it has no memory across responses: nothing stops a
permanently worse mix from passing one at a time, each individually in range. And drift
statistics must never be used this way in reverse: one low score is expected noise, so
a per-response drift trigger ships a built-in false-positive rate (roughly 2.5% of
healthy traffic at −2σ) to real users. **Drift statistics act on the agent; guardrails
act on the response.**

### 3. Audit coverage before building anything

Detection is strictly downstream of scoring. **A detector cannot see anything its
evaluators cannot score.**

The useful split is not by failure mode. **Every failure mode has both a per-response
dimension and a rate dimension.** What differs is whether the check behind it produces
a fact or an opinion, because that decides what a single instance licenses you to do:

| Failure mode | The check is | One instance means | Rate worth watching |
|---|---|---|---|
| Tool-output contradiction | Deterministic contract check | A fact. Block it | Yes |
| Workflow violation | Session-level sequence check | A fact. Block it | Yes |
| Safety / PII regression | Scanner, regex, guardrail | A fact. Block it | Yes |
| Fabrication, hallucination | Judge-model grounding score | An opinion with an error rate | Yes, and it is the only trustworthy signal |
| Wrong tool selection | Tool-selection-accuracy evaluator | Usually a fact, if you have an expected trajectory | Yes |
| Stale knowledge | Grounded fact-check against a reference system | A fact, where a reference exists | Yes |
| Misrouting, misclassification | Downstream outcome tracking, judge | Often only knowable later | Yes |
| Degraded helpfulness | Judge helpfulness/completeness score | An opinion with an error rate | Yes, and it is the only trustworthy signal |

A deterministic check that fails has found something that is definitely wrong, so a
single instance is actionable and blocking it is correct. A judgment cannot carry that
weight: it has its own error rate, and healthy traffic legitimately produces low scores.
On the deployment behind this document a grounding judge returned its lowest label on
roughly a third of responses while the agent was perfectly healthy, because a third of
the questions had nothing to ground against. Blocking on that single score would have
rejected a third of good traffic. Only the *proportion* carried information.

That a rate matters even where blocking already works is the part worth spelling out,
because a successful block feels like the problem is handled. Take a PII scanner that
blocks 0.1% of responses on healthy traffic. A prompt edit quietly drops an instruction
about not repeating user-supplied identifiers, and it now blocks 4%. Every block worked.
No PII ever reached a user. And the agent is now attempting to leak PII forty times more
often than it used to, which the scanner cannot tell you, because from its point of view
it succeeded on every single response in both periods.

Blocking answers "did harm reach the user." The rate answers "did the system get worse."
Four reasons the second question still needs asking:

- **A block treats the symptom and says nothing about the cause.** It protected one
  response. Only the rate moving says something upstream changed and an investigation is
  due.
- **A blocked response is still a failed response.** The user got a fallback, an error,
  or a degraded answer. At 4% instead of 0.1% the agent is functionally 4% broken while
  the guardrail behaves perfectly. A guardrail converts a correctness failure into an
  availability failure, which is a better failure, not the absence of one.
- **Deterministic checks are a sample, not a census.** They catch what you wrote them to
  catch. A prompt edit loose enough to triple PII attempts has almost certainly loosened
  behaviors you have no check for, and those are now happening silently. A rising rate on
  the checks you do have is a canary for the ones you do not.
- **Only a rate licenses action on the agent.** Nobody rolls back a model over one
  blocked response. You roll back because the rate moved. That is the boundary rule read
  in the other direction: guardrails handle the response, and only the population-level
  view can tell you to handle the agent.

Two consequences follow, both measured rather than reasoned.

**Structural evaluators do not move when the model degrades.** Swapping a current model
for a much older one left schema validity, PII scanning, and the tool-contract check
completely flat, because a weaker model still emits well-formed tool calls and still
avoids leaking PII. What moved was tone and factual grounding. **If everything you
evaluate is deterministic, the most common cause of drift is invisible before any
detector enters the picture.**

One last nuance. Early drift rarely presents as one dramatic mode; it presents as a small
rate increase across several at once, which is why quality must be tracked per dimension
rather than collapsed into a composite. A composite averages the one dimension that moved
against seven that did not, and the movement disappears.

### 4. Verify the scores actually land

Generate traffic and confirm individual evaluation records exist before trusting
anything built on top of them. On the deployment behind this document, the evaluators
produced *nothing at all* on first setup and reported no error anywhere: a tracing
library was missing, and once that was fixed the spans were routed to a log group
evaluation does not read. **A silent evaluation pipeline and a healthy agent are
indistinguishable from the outside**, and a detector built on one will be reassuringly
quiet forever.

### 5. Measure each stream's shape, then choose the statistic

Call one evaluator's scores over time a **stream**. Eight evaluators is eight streams,
each with its own normal, and **they do not have the same shape.** Measured on healthy
traffic, five sat on a single value more than 90% of the time. Three had genuine
dispersion.

**That split is predictable from step 3's fact-versus-opinion distinction.** All five
deterministic checks landed in the pinned group and all three judges in the dispersed
group, which is not a coincidence: a deterministic check on a healthy agent passes nearly
every time, so its scores collapse onto one value, while a judgment has real spread
because judgment does. You can therefore anticipate which statistic a stream will need
before measuring it, though still measure, since this is a tendency rather than a law.

That shape inverts which statistic works:

| Shape | Method |
|---|---|
| Continuous, or 5+ levels in real use | EWMA, smoothing 0.2, limit 3σ |
| Coarse but genuinely dispersed | EWMA, smoothing 0.1, limit 4σ |
| **Concentrated on one value, sparse outliers** | **Per-sample z-score, −2σ** |

The last row is counterintuitive and it is the one that bites. When a stream sits on
one value almost always, its rare dips are isolated events rather than a distribution
tail. A smoothing statistic carries a single dip forward across many later samples;
those carried-forward samples then satisfy whatever run-length rule you set to confirm
drift (step 7), and the detector declares drift that never happened. A memoryless
check crosses once, the next sample is healthy, the run resets, nothing is declared.
**On this shape, memory is a liability.**

This is measured, not theoretical. A schema-validity stream sat at its top value for
98% of 53 healthy samples with one dip, during traffic known to be healthy. An EWMA
carried that dip across eight consecutive samples, cleared the run-length rule, and
latched a false drift. A memoryless z-score over identical data stayed silent.

So branch on **dispersion**, the fraction of samples on the most common value, not on
how many values the rubric can emit. A three-level rubric returning one value 98% of
the time is a constant with outliers. Process control reached this split long ago,
between charts for discrete pass/fail data and charts for continuous measurement, which
is also the distinction behind step 6's fix.

### 6. Fix the evaluators that cannot carry a signal

An evaluator written to answer "was this acceptable" returns a verdict, and a verdict
is close to the worst possible input to a control chart. The clearest case observed: a
price-accuracy evaluator computed a continuous percentage deviation between the quoted
figure and a reference, then threw the magnitude away and returned pass/fail against a
fixed internal threshold. That deviation would have been the most monitorable stream
on the agent. Thresholding it produced a stream sitting at its top value ~94% of the
time.

**Return the measurement, not the verdict.** A detector can always threshold a number;
it can never recover a number from a verdict.

This is not a new opinion, it is quality engineering's oldest distinction, applied to
evaluators. Statistical process control has always split its charts into two kinds. A
**variables chart** tracks a measurement: calipers give you 10.02mm, 9.98mm, 10.07mm,
values on a continuous scale. An **attribute chart** tracks a count or a proportion: a
go/no-go gauge gives you pass, pass, fail, pass. The standard guidance has always been
to prefer variables data wherever you can get it, because a measurement carries more
information per observation than a verdict does, and detects a real shift faster and
with fewer samples. The price-accuracy evaluator above had built the calipers, computed
the deviation, and then shipped the go/no-go gauge instead. Nothing forced that choice;
it discarded the more informative signal it already had.

Two related shapes to check for once you are looking at real measurements:

- **Pinned at the floor**: cannot fall, so nothing is ever detectable.
- **Pinned at the ceiling**: no measured variance, so the control limit comes entirely
  from whatever standard-deviation floor you set to avoid dividing by zero. The limit
  is then a convention you chose, not something learned from the agent.

Do this before building the detector, not after. It is cheaper than discovering the
detector can never fire.

### 7. Build the stateful layer

The evaluators stay as they are. What is missing is a component that reads their
output *as a sequence* and holds state. Five requirements, each preventing a specific
observed failure:

- **A learned baseline** (mean and variance from the stream's own recent healthy
  history) over a **bounded window**. Unbounded is not stationary even on healthy
  traffic: in the offline study, over 400 healthy samples a mean crept from 0.920 to
  0.958 while standard deviation shrank from 0.1033 to 0.0867, because every
  near-perfect score pulls the mean toward the ceiling and tightens the spread. Any
  σ-scaled threshold therefore drifts toward a hair trigger with nothing wrong.
- **A control limit derived from that variance**, so the alarm point moves with the
  agent instead of being a fixed line.
- **Warm-up**, because the baseline must be learned before it can be used. Publish the
  warming state explicitly: silence while learning is indistinguishable from silence
  while healthy, and only one is good news. Around 100 samples is defensible; it is a
  sample count, not a duration, so an agent handling 20 sessions a day needs over a
  week before any claim is possible.
- **A persistence rule** requiring N consecutive crossings before declaring anything.
  Five was the value at which every method tested reached zero false alarms in the
  offline study, and it is what the live deployment ran. Apply the same rule to every
  method you compare, or each one flatters itself. Without it, spurious alarms landing
  shortly after real drift begins get miscredited as fast detections.
- **Latching**, so one condition yields one alarm. On intermittent drift a smoothed
  statistic recovers and re-crosses whenever a run of healthy samples arrives; in the
  offline study one underlying condition produced nine separate alarm episodes across
  200 samples. Nine pages for one condition is how a monitoring system loses its
  audience.

Plus a rule for when the baseline **absorbs** a sample versus **freezes**: freeze
while alarming, so degraded scores cannot quietly become the new normal.

Two implementation notes that cost real time to learn.

**Read records, not metrics.** A pre-aggregated metric datapoint is a period average,
which silently changes the statistic you are monitoring by an amount that depends on
traffic volume. Individual evaluation records also carry the session identity a
scheduled job needs to count each score exactly once.

**Deduplicate on a per-response identifier.** Trace-level records within one session
can share a single session-start timestamp, so a key built from
evaluator + session + timestamp collapses distinct scores into one and computes the
baseline from a fraction of the traffic, with no error reported anywhere.

This all runs on a schedule, out of the request path, adds zero latency, and never
inspects or blocks a response.

### 8. Settle ownership before it fires

Silent quality failure sits in an organizational blind spot. SRE and platform teams own
monitoring, but their dashboards stay green by construction. ML and product teams own
quality, typically only pre-release. Post-deployment quality often has **no owner at
all**: no rotation is paged for a correctness decline, no SLO exists for grounding
scores.

Decide who receives the alert, who may freeze or roll back, and what the response SLA
is. A detector firing into an unowned channel is governance theater, and the state it
produces, an alarm history everyone learned to ignore, is harder to repair than having
had no detector.

---

### Limits of the approach

Those are the steps. Five limits are worth understanding before committing to them,
because none of them are fixable by better tuning.

**Baselines absorb slow decline.** Non-alarming samples fold into the baseline, so a
decline gradual enough stays inside the limit while dragging the limit down with it.
Freezing while alarming bounds this without eliminating it. Arbitrarily slow drift
against a self-learned baseline is unsolved.

**Detection needs traffic.** Every number here is per sample, not per hour. A
low-traffic agent takes proportionally longer to reach any verdict. That is a floor, not
a tuning problem.

**Memoryless streams are blind to rate.** The same property that stops a near-degenerate
stream from false-alarming also stops it confirming a failure rate that rises without
clustering. Observed twice, in both cases on the same stream: a check whose own failure
rate roughly quadrupled under a degraded model, and rose more than sixfold under a stale
data source, confirmed neither, because the failures were scattered rather than
consecutive.

**Diagnosis is manual, and the judge can drift too.** The detector says which stream
moved and by how much, never why. And semantic quality is scored by a model that nothing
here watches; attributing a decline to the system versus the evaluator is an open
research problem ([arXiv:2606.15474](https://arxiv.org/html/2606.15474v1)). Treat drift
on a judge-scored stream as "one of these two moved" and check the judge before
concluding anything about the agent.

**Naive detection is worse than none.** Repeated statistical testing across time and
features accumulates false positives into alarm fatigue, and corrections that suppress
false alarms reduce true sensitivity; the tradeoff is fundamental
([arXiv:2607.17336](https://arxiv.org/abs/2607.17336)). Treat your false-alarm rate as a
first-class requirement, and run in shadow mode to a non-paging channel for two to four
weeks before anyone gets paged.

---

## Evidence: what the experiment found

None of the guidance above is theoretical. It came from deploying a detector against the
AWS Market Trends Agent sample, an agent with eight evaluators running on real AgentCore
infrastructure, then inducing three separate causes of drift, with the streams predicted
to move written down in advance of each run, so the results could
contradict the prediction rather than be narrated around it.

**A model swap** (a current model replaced by a much older one through an environment
variable, same prompt, same tools, no redeploy) is the canonical cause, because there
is no deploy event to investigate. It confirmed drift on two streams: the professionalism
judge quoted in step 2, and a grounding judge that fell from 0.65 to 0.18 as most
responses stopped citing retrieved data at all. Five streams stayed correctly silent.

**A stale data source** was mostly missed, and instructively so. The grounding judge did
not move at all, because a frozen quote is still a *retrieved* value, and no evaluator on
the agent scored whether a quote was *fresh*. A whole class of degradation was invisible
for want of an evaluator, which is step 3's argument arriving as a result rather than a
recommendation.

**A dropped orchestration step** was the cleanest catch of the three. Removing the tools
that satisfied a required workflow contract makes the failure structural rather than
probabilistic, so the contract stream went from passing every session to failing every
session with nothing in between. Drift caused by a capability disappearing is far easier
to detect than drift caused by a model getting worse at something it can still do.

Two of the advance predictions were wrong, which is the useful part. The model downgrade
was expected to break multi-step tool sequencing and did not touch it; the dropped
profile step was expected to degrade personalization and left it flat. A model downgrade
degraded semantic judgment while leaving every mechanical check intact. An agent whose
evaluators were all mechanical would have detected none of it.

---

## In short

Nothing already in a normal agent stack compares the agent against its own past, and
that is the only comparison that catches a quality decline with no deploy event behind
it. Guardrails, evaluators, regression suites, and A/B tests all measure output against
a reference a human supplied. Drift detection needs a reference the agent supplied,
learned from its own recent behavior, which is why it requires new state rather than a
reconfiguration of something you already run.

Three things are worth carrying away even if you build none of it. **The evaluators are
the ceiling, not the detector**: an agent scored only on structure will not see a model
downgrade at all. **An absolute threshold cannot express a relative claim**, wherever
you put the bar. And **the window where detection is valuable is the window where
nothing looks wrong**, so waiting for the decline to become obvious means electing your
users as the detection mechanism.
