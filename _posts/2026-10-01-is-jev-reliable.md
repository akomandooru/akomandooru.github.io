---
layout: post
title: "Is Jev Reliable as a Confidence-Gated Decision Model?"
---
**If you only read one line:** Jev's confidence is useful as a cheap first stage in a cascade, but the cutoff and fallback need to be tuned per task, not reused as-is.

![As the choices grow, Jev's confidence falls with its accuracy, while the Laya base checkpoint's confidence rises as its accuracy falls]({{ '/assets/hero-confidence-vs-accuracy.png' | relative_url }})

Jev is a model from TypeSafe that picks one answer from a list and says how confident it is. Software can use that confidence to decide when to act on an answer by itself and when to hand the case to something slower, such as a larger model or a person.

**The question:** can Jev's confidence be trusted to make that call, and does it stay trustworthy as decisions get harder? I tested it against Laya's base English checkpoint (an open source alternative), Claude Haiku 4.5 (a small LLM), and GPT-5.6 Sol (a frontier LLM) on three classification tasks: sorting 500 support tickets into bug reports, billing questions, and feature requests; choosing an AI agent's next action from 5 to 40 options; and a small, harder ticket-sorting test.

**The short answer:** Jev's confidence was useful: its confidence scores were generally lower on answers that turned out to be wrong. But it was not plug-and-play. On the 500-ticket task, a Jev cascade sent only 3.4% of tickets to GPT-5.6 Sol, cost 5% as much as GPT-5.6 Sol alone, and scored higher against the test labels. On the next-action task, the same 0.6 cutoff did not transfer: at 40 options, the cascade was 71% accurate against GPT-5.6 Sol's 78%. Raising the cutoff to 0.9 restored 78% accuracy, but sent 65% of decisions to GPT-5.6 Sol and raised the cost. Jev can be an effective cheap first step, but its cutoff and fallback need to be tested for each task.

All three tasks were text classification, so test on your own task before relying on these results.

---

## 1. What I wanted to learn

Many applications need a fast, cheap "pick one" decision inside a larger system: routing a support ticket, choosing an agent's next action, deciding what needs a human. A common design is a [**confidence-gated cascade**](https://arxiv.org/abs/2605.06350), sometimes called [**confidence-based routing**](https://arxiv.org/html/2604.12262v1). A cheap model answers every case and reports how sure it is, and cases below a threshold go to a more expensive fallback model. Published systems report large savings with this design, for example [40 to 85% cost reduction](https://arxiv.org/html/2604.23577v1) on one six-task benchmark. Those results come from other systems and say nothing about Jev.

[Jev](https://docs.typesafe.ai) returns a typed answer, a probability for each option, and a confidence number, which TypeSafe describes as calibrated. The open source alternative I compared it with, [Laya](https://huggingface.co/convaiinnovations/laya), makes a similar calibration claim. Because both vendors make that claim, I measured it directly rather than taking it on trust. The question, stated precisely:

> **Can Jev's confidence be trusted enough to gate real decisions, and does that trust hold up as the decision gets harder?**

**What "trusted" means here.** A confidence score needs two separate properties:

1. **Ranking:** wrong answers should get lower scores than right answers. I measure this with AUROC: the probability that a randomly chosen right answer scored higher than a randomly chosen wrong one, with ties counting half. 0.5 means the score carries no information; 1.0 means it separates them perfectly.
2. **Scale:** a stated number should match reality. Answers scored 0.9 should be right about 90% of the time. I measure this with calibration error (ECE): the gap, in percentage points, between stated confidence and actual accuracy, averaged over 10 equal-width confidence bins and weighted by how many answers fall in each.

Ranking tells you whether a threshold can separate riskier answers from safer ones. Scale tells you whether the same numerical threshold is likely to mean the same thing. Choosing the threshold still depends on the task, the cost of an error, and how well the fallback handles the escalated cases.

A cascade has two steps: the cheap model's own answer, then the fallback for low-confidence cases. Section 4 measures the first step on three tasks. Section 5 measures a full cascade on two of them.

## 2. What Jev is

You send Jev a piece of text (the "state") and one or more typed questions. TypeSafe's documentation describes three question types: Choice (pick one option), Score (a level on a scale), and Noul (TypeSafe's name for a yes/no probability). Choice and Score return a probability for each option plus a separate confidence number meant for deciding whether to act. There is no generated text to parse.

A real request and response from this evaluation (trimmed; the response's `model`, `usage`, and per-answer `type` fields are omitted):

```json
{
  "model": "jev-latest",
  "state": "Subject: Required Assistance with Marketing Strategies\n\nBody: Our marketing agency is facing challenges ...",
  "questions": {
    "category": {
      "type": "choice",
      "instructions": "Classify this support ticket into exactly one category.",
      "criteria": {
        "bug_report": "Something is broken or behaving incorrectly",
        "billing": "Charges, invoices, refunds, subscriptions",
        "feature_request": "Asks for new functionality that doesn't exist yet"
      }
    }
  }
}
```

```json
{
  "answers": {
    "category": {
      "choice": "bug_report",
      "probabilities": { "billing": 0.01, "bug_report": 0.59, "feature_request": 0.40 },
      "confidence": 0.39
    }
  }
}
```

**Confidence is not the top probability.** Here the top option has probability 0.59, but confidence is 0.39. [TypeSafe's documentation](https://docs.typesafe.ai/confidence) says confidence is a statistic computed from the probability distribution, lower when the distribution is flatter, but does not publish the formula. On the main task the two numbers differed by more than 0.01 on 104 of 500 tickets, and confidence was never higher than the top probability. I gated every cascade on the `confidence` field, since that is the number TypeSafe documents for deciding whether to act. The same page suggests treating 0.5 as a floor for uncertain answers and raising the threshold with the risk of the action.

I called TypeSafe's API directly for every test. At the time of testing, Jev's published price was $0.042 per million input tokens ($42 per billion), with no separate output price. I used model ID `jev-latest`, which TypeSafe can update, so these results describe the model as it behaved in September 2026. No Jev calls failed in any setup.

## 3. What I did

I ran Jev on three evaluation tasks and compared it with other systems on the same items, labels, and scoring.

| Setup | Task | Data | Items | Where the correct answers come from | What it tests |
|---|---|---|---|---|---|
| **1** | Support ticket triage, 3 categories | [`Tobi-Bueck/customer-support-tickets`](https://huggingface.co/datasets/Tobi-Bueck/customer-support-tickets) | 500 tickets | Derived by my rule from the dataset's own `type` and `queue` fields; the dataset has no category field | Accuracy and confidence quality on a moderate task |
| **2** | Predict a person's next web action | [`osunlp/Mind2Web`](https://huggingface.co/datasets/osunlp/Mind2Web) | 100 next-action decisions, one from each of 100 recorded web tasks; each tested with 5, 10, 20, and 40 options | The person's recorded next action | Whether confidence stays useful as the number of choices grows |
| **3** | Support ticket triage, 8 overlapping categories | [`aws-samples/sample-nvidia-nemotron-cascade-workshop`](https://github.com/aws-samples/sample-nvidia-nemotron-cascade-workshop) (synthetic tickets) | 30 tickets | Labels generated by Claude Opus 4.7, not people | Confidence on a harder label set, exploratory only |

Each Setup 2 item captures one point in a recorded web task. The model sees the person's task and the actions already taken, then predicts the person's recorded next action. For example, the task is "Find top-rated outdoor activity events in Spain, that is happening on May 1 and book the tickets for two adults in morning group with an English guide" on Ryanair, and the steps taken so far are `[div] Events & Activities -> CLICK`, `[input] -> CLICK`, and `[button] Spain -> CLICK`. The model must pick what the person did next, `[button] All cities -> CLICK`, from a menu whose other options are actions from unrelated tasks, such as `[span] Newest -> CLICK`.

The charts compare four systems: the two decision models, one small LLM, and one frontier LLM.

| System | How it runs | Where its confidence comes from |
|---|---|---|
| **Jev** | Hosted API, called directly through TypeSafe | The API's `confidence` field |
| **Laya** | Open source (base English checkpoint of [`convaiinnovations/laya`](https://huggingface.co/convaiinnovations/laya), not fine-tuned), run locally | The model's own `confidence` field |
| **Claude Haiku 4.5** | Amazon Bedrock | A number the model writes with its answer. The prompt asked for a number well below 1.0 when the input could fit more than one option, and close to 1.0 only when certain. In Setup 3, I used the highest probability it returned as its confidence. |
| **GPT-5.6 Sol** | Amazon Bedrock | Setup 2: a number it writes, asked for the same way as Haiku. Setups 1 and 3: answer only, no confidence. Also the fallback in every cascade. |

Accuracy can be compared directly. Confidence cannot, because each kind of system produces it differently. Jev and Laya compute theirs from their own probabilities over the options. The LLMs state a number, because the API I used returned no token probabilities for them, and Anthropic does not offer them for Claude. Token probabilities would not settle this anyway. They measure how likely the model was to write a given answer, not how likely that answer is right, and they are often overconfident. So I compare how each system's confidence behaves, not its raw values: whether it ranks right answers above wrong ones, and whether its values match its accuracy.

## 4. Results: each model on its own

Every number in this section is a model's own answer, with no escalation. An accuracy followed by a range, such as 91.2% (88.4 to 93.4%), gives the 95% confidence interval (Wilson score method): the range the true accuracy likely falls in, given how many items were tested. Because every system answered the same items, I also checked whether one consistently got cases right that another missed. The p values provide supporting evidence, not proof.

### Setup 1: similar accuracy, but the labels matter

![Setup 1 accuracy comparison]({{ '/assets/setup1-accuracy.png' | relative_url }})

**Accuracy.** Jev and the Laya base checkpoint both scored 91.2% (88.4 to 93.4%), GPT-5.6 Sol scored 89.4%, and Claude Haiku 4.5 scored 88.6%. Jev was right where GPT-5.6 Sol was wrong on 14 tickets, and the reverse happened on 5 (p = 0.06), so this test does not establish a difference between them.

**Label sensitivity.** The dataset has no category field, so I created labels from its `queue` and `type` fields. Tickets in the "Billing and Payments" queue count as `billing`; all others use `type`. That rule is questionable because a queue says which team handled a ticket, not what the ticket is about. It labels "Report of Data Breach: Sensitive Medical Records Exposed" as `billing`, and on 20 of the 67 billing-queue tickets all four systems chose the answer implied by `type` instead.

The lighter bars show a second scoring on the 433 tickets outside the billing queue. Jev scored 97.5% (95.5 to 98.6%), GPT-5.6 Sol 96.1%, Haiku 95.4%, and the Laya base checkpoint 94.5%. Jev's lead over GPT-5.6 Sol was again not established (11 versus 5, p = 0.21). This subset is easier because it has no `billing` labels, and I did not manually check its `type` labels. It shows how much the label rule moves the result, not which labels are correct.

**Confidence.** Of the three systems that reported confidence in this setup, Jev's stayed closest to its accuracy. Its calibration error was 6.4 points on all 500 tickets and 2.1 outside the billing queue, compared with 43.3 and 46.9 for Laya and 19.7 and 26.6 for Haiku. Laya ranked right answers above wrong ones slightly better against the original labels (AUROC 0.81 versus Jev's 0.74), but its numerical scale ran low: answers it scored below 0.6 were still right 86% of the time. Jev returned confidence 1.0 on 289 of 500 tickets, including 10 labeled errors, so it could not distinguish risk among much of its high-confidence output. GPT-5.6 Sol was not asked for confidence in this setup.

### Setup 2: Jev's confidence stays useful as choices grow; the Laya base checkpoint's confidence carries almost no signal

![Setup 2 confidence versus accuracy]({{ '/assets/setup2-confidence-vs-accuracy.png' | relative_url }})

At one point in each of 100 recorded web tasks, the model chose the person's next action from 5, 10, 20, or 40 options. The smaller menus came from the larger ones, so only the number of wrong options changed. Jev's accuracy fell from 86% to 72%, its mean confidence fell with it from 86% to 72%, and it still ranked right answers above wrong ones at 40 options (AUROC 0.89 to 0.78). GPT-5.6 Sol was 3 to 8 points more accurate at every size, though no single gap was clear (p = 0.06 to 0.55), but it reported about 92% at every size, so its confidence did not show that the task had become harder. Haiku's accuracy fell further, from 82% to 59%, while its confidence fell less. The Laya base checkpoint's confidence carried almost no information (AUROC 0.49 to 0.56) and rose while its accuracy fell from 46% to 4%; at 20 options, a 0.6 gate would have accepted 45% of its answers, and 91% of those were wrong. Even for Jev, a fixed threshold did not hold: answers passing 0.6 were 93% right at 5 options and 80% at 40. The wrong options came from other tasks, which makes them easier to reject than those on a real web page, so the accuracies here are optimistic.

### Setup 3: 30 tickets, too few to rank

On 30 synthetic tickets with 8 overlapping categories, scored against labels that Claude Opus 4.7 generated (a frontier LLM grading other LLMs, which may favor LLM-style answers), GPT-5.6 Sol agreed with the labels on 29, Jev on 26, Haiku on 25, and Laya on 15. Jev's confidence ran high: its mean was 0.93 against 87% agreement, and one of its wrong answers scored 0.96. With model-made labels and 30 tickets, this test cannot rank the systems; the intervals for GPT-5.6 Sol, Jev, and Haiku overlap.

### What this means

Among the four systems shown, Jev's confidence was the most consistent at both ranking its answers and staying close to its accuracy on Setups 1 and 2. It was not the most accurate: GPT-5.6 Sol was 3 to 10 points ahead on Setups 2 and 3, though no gap between them was clear. The broader lesson is that accuracy and confidence quality have to be measured separately, and the labels need the same scrutiny as the models. One label rule I chose moved Jev's accuracy by 6.3 points, more than the 2.6-point spread between the four systems. Section 5 shows what this means for a full cascade, and why the threshold still has to be set for each task.

## 5. What a full cascade costs

A cascade pays off when the first model is cheaper, the gate sends the right cases, and the fallback performs better on the cases it receives. All three aligned on the ticket task, but the gate and fallback stopped working together as the agent task got harder. I replayed each cascade offline: wherever the cheap model's confidence was below 0.6, I used GPT-5.6 Sol's logged answer and added its logged cost and latency. I chose 0.6 before testing Jev and did not change it. Costs are API charges at published prices, without hardware or operations.

### Setup 1: ticket triage

![Setup 1 cascade comparison]({{ '/assets/setup1-cascade.png' | relative_url }})

Jev escalated 17 of 500 tickets (3.4%). Its cascade cost 5% as much as GPT-5.6 Sol alone ($0.056 versus $1.12 per 1,000) and 4% as much as a cascade with Claude Haiku 4.5 first, partly because Haiku's prompt was longer. Its median latency was 3.5 to 4 times lower than either. Against the test labels, it scored higher than both on the same tickets (p = 0.002 against GPT-5.6 Sol alone, p = 0.006 against the Haiku cascade). It worked because every ticket where GPT-5.6 Sol beat Jev was among the 17 that Jev escalated.

### Setup 2: predicting the next web action

![Setup 2 cascade comparison]({{ '/assets/setup2-cascade.png' | relative_url }})

At 5 options, the cascade was within 1 point of GPT-5.6 Sol; at 10, they tied. In both cases it cost under a quarter as much. As the options grew, Jev escalated more (16% to 31% of decisions) and the savings fell. At 40 options, the cascade was 7 points less accurate in this sample (71% versus 78%, p = 0.016 for that comparison) and cost 37% as much. All 7 decisions that GPT-5.6 Sol got right and the cascade got wrong were cases Jev had kept, with confidence of 0.6 or higher.

### The threshold has to be set per task

The same 0.6 threshold gave the best accuracy on Setup 1 and was too low for Setup 2 at 40 options. On Setup 1, raising it only lowered accuracy, because the extra escalations were mostly answers Jev already had right. On Setup 2 at 40 options, raising it to 0.9 brought the cascade level with GPT-5.6 Sol at 78%, but sent 65% of decisions to the fallback and cut the savings to 1.4 times. I looked at these other thresholds only after seeing the results, so they illustrate the trade-off rather than recommend a setting.

Choose the threshold on held-out items from your own task, with your own fallback, and choose it again when the task changes. If the fallback is a person, a higher threshold catches more errors at the cost of more reviews: on Setup 1, moving from 0.6 to 0.9 raised the share of tickets sent for review from 3.4% to 11% and caught 15 of Jev's 44 errors instead of 11.

## 6. When Jev fits

This evaluation supports a narrow claim: Jev's confidence is useful for routing single-choice text decisions among a moderate number of options, when a missed error is tolerable and there's a fallback to catch it. Within that scope, two things decide whether a cascade actually pays off, beyond Jev's confidence being informative on its own:

**The fallback has to be better specifically on the cases Jev escalates, not just better on average.** On the ticket task, GPT-5.6 Sol outperformed Jev on exactly the cases Jev flagged as uncertain, so the cascade beat both systems alone. On the next-action task at 40 options, several of Jev's errors scored above the 0.6 cutoff and were never escalated, so GPT-5.6 Sol never got the chance to fix them, and the cascade fell behind GPT-5.6 Sol alone. Before trusting a cascade, check what the fallback does specifically on the subset the gate sends it, not its overall accuracy.

**The threshold has to be set for your option count and task, and re-set when either changes.** Section 5 showed the same 0.6 cutoff producing the best accuracy on one task and a clearly worse one on another. There's no default that transfers.

Outside this scope, this evaluation gives no evidence either way: multi-label decisions, extraction or generation tasks, option counts beyond 40, near-identical options, or deployments with no labeled data to validate a threshold against.

## 7. What these results do not show

These results come from three text-classification tasks with 3 to 40 options, two of them ticket triage with text that looks generated rather than written by customers. I did not manually review the labels. Setup 2 uses actions recorded from human annotators, while Setup 1 uses a rule I wrote and Setup 3 uses labels from another model. The 0.6 threshold was set in advance and never validated on held-out data, and I did not compare against a classifier trained on the labels, which is often the strongest cheap option when labeled data exists. I tested only Laya's base checkpoint; its [model card](https://huggingface.co/convaiinnovations/laya) reports 0.766 accuracy for a fine-tuned checkpoint against 0.362 for the base checkpoint on the authors' own benchmark. TypeSafe has not published how Jev computes its confidence, so I observed only its behavior. Model versions and prices change, so recheck any number here before relying on it.

## 8. How to test this on your own task

1. **Measure accuracy and confidence separately, on your own items.** Use your own labels and option counts, with confidence intervals and paired tests for accuracy. For confidence, measure both ranking (AUROC) and scale (calibration error); a model can pass one and fail the other.
2. **Check your labels.** Review the items where every system agrees with each other and disagrees with the label. Here, that check exposed a bad label rule.
3. **Stress-test with more options.** Add wrong options and check that confidence falls with accuracy. This check showed that the Laya base checkpoint's confidence carried almost no signal.
4. **Choose the threshold on held-out data.** Split your labeled items, pick the threshold on one half for a target cost or accuracy, and report results on the other half.
5. **Replay the full cascade with your real fallback.** Count what the fallback fixed and what it broke, and report cost with median and tail latency. Count the cheap model's cost and latency on every item, including escalated ones.
6. **Recheck after changes, and before generalizing.** Re-run when the model version, label set, or number of options changes, since the right threshold moved with option count here. Test more than one task before drawing conclusions about any system.

---

## How to repeat this evaluation

<details>
<summary>Setup details, models and settings, and a spec prompt for building the harness</summary>

These numbers come from logged per-item results that aren't attached, so repeating this gets you comparable, not identical, numbers.

**Setup details:**
- **Setup 1:** from [`Tobi-Bueck/customer-support-tickets`](https://huggingface.co/datasets/Tobi-Bueck/customer-support-tickets), keep English rows with a non-empty `type` in nine support queues, shuffle with Python's `random.Random(42)`, and take 500. Labels: queue "Billing and Payments" becomes `billing`; otherwise type Incident or Problem becomes `bug_report`, and Request or Change becomes `feature_request`. For the second scoring, drop the 67 billing-queue tickets and keep the other 433, whose labels come only from `type`.
- **Setup 2:** from [`osunlp/Mind2Web`](https://huggingface.co/datasets/osunlp/Mind2Web), take one point from each of 100 different recorded web tasks, with at least one earlier action. Wrong options are drawn from other tasks. Build each item's 40-option menu first and take the smaller menus from the same draw, shuffling positions in each.
- **Setup 3:** the first 30 tickets of `data/synthetic-1k.json` in [`aws-samples/sample-nvidia-nemotron-cascade-workshop`](https://github.com/aws-samples/sample-nvidia-nemotron-cascade-workshop). The labels came from the repository's own labeling command (`npm run bakeoff -- --label --limit=30`, Claude Opus 4.7 at its default temperature), and aren't checked into the repository, so labels you generate will differ somewhat.

**Models and settings:** Jev `jev-latest`; Laya base checkpoint `convaiinnovations/laya`; Claude Haiku 4.5 `us.anthropic.claude-haiku-4-5-20251001-v1:0` and GPT-5.6 Sol `us.openai.gpt-5.6-sol` on Amazon Bedrock (us-east-1) with forced tool use. Escalation threshold 0.6. Prices as of September 2026, per million tokens: Jev $0.042 input (no separate output price); Claude Haiku 4.5 $1 input and $5 output; GPT-5.6 Sol $4 input and $20 output. Latency is client-side wall-clock time, with the 99th percentile taken as the nearest-rank value.

**Spec prompt you can hand to Kiro or Claude Code to build the harness:**

> Build an evaluation harness that tests whether a classification model's confidence score is good enough to gate a confidence-based cascade (escalate to a fallback model below a threshold).
>
> 1. For each candidate model and each labeled dataset, call the model once per item and log: item ID, prediction, gold label, the model's confidence, its full probability distribution if available, latency, input and output tokens, cost, and any error. Record where each model's confidence comes from (a native field, a top probability, or a self-reported number from a structured-output field). If the API returns both a probability distribution and a separate confidence field, log both.
> 2. Compute all statistics from the per-item logs: accuracy with 95% Wilson intervals and paired McNemar tests between models; AUROC of confidence for separating right from wrong answers (ties count half); expected calibration error with 10 equal-width bins; a reliability table; and the number of errors in each confidence band.
> 3. Stress test: for a fixed set of about 100 items, rebuild each item's menu at 5, 10, 20, and 40 options, where smaller menus are subsets of larger ones. At each size, report accuracy, mean confidence, calibration error, AUROC, and the share and accuracy of items passing a candidate threshold. Flag any model whose confidence rises, or stays flat, while accuracy falls.
> 4. Cascade replay: run the fallback model on the same items, including every menu size of the stress test. For a sweep of thresholds, replace answers below the threshold with the fallback's answer. Report escalation rate, accuracy, cost, and median and 99th percentile latency, counting the cheap model's cost and latency on every item. Report how many escalations the fallback fixed and how many it broke, and a paired McNemar test against each model alone.
> 5. Choose the operating threshold on a random half of the items and report every metric on the other half.
> 6. List the items where every model gives the same answer and disagrees with the label, for label review. If labels come from a rule, also score a subset chosen by a rule that does not look at model outputs, and report both.
> 7. Write a report that keeps accuracy, confidence ranking, confidence scale, cost, and latency in separate columns, and never combines them into one score.

</details>
