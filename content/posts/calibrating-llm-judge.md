---
title: "Calibrating an LLM judge against human labels"
date: 2026-11-26T22:55:00-08:00
draft: false
---

An LLM judge is a model that reads the output of another model and decides whether the output is good. Teams use LLM judges because people are too slow to grade every output, and exact string matching can't grade open-ended answers.

The problem is that a judge is also a model, and a judge makes mistakes. If you report "92% of answers passed" and the judge is wrong on one answer in five, nobody knows what the 92% means. Before you trust a judge, you need to compare its decisions with decisions made by people, and then change the judge until the two agree. I call that process calibration, and I describe the steps I follow below.

This gap shows up all the time in practice. A team builds a support or search assistant, writes a reasonable-looking judge prompt, and watches the pass rate climb past 90%. Then someone sits down to read fifty outputs by hand and finds that the judge is passing polite, fluent answers that point to the wrong setting, skip a required step, or quietly miss the user's actual question.

## Step 1. Make the judge answer one yes or no question

Start with a judge that checks one criterion and returns pass or fail. For example, "Does the answer resolve the user's problem?" is one criterion. A judge that returns a score from 1 to 5 for "overall quality" is much harder to calibrate, because two careful people will often disagree about whether an answer is a 3 or a 4, and you can't calibrate a judge against labels that people don't agree on.

If you care about several criteria, such as correctness and tone, build one judge for each criterion. You can then measure and fix each judge separately.

## Step 2. Collect human labels

You need a set of outputs that people have labeled with the same pass or fail question. A few points decide whether the labels are useful.

- **Use people who know the domain.** The labels define what "good" means, so the labelers should be the people whose judgment you want the judge to copy.
- **Have two people label the same items.** Measure how often they agree before you measure the judge. Human agreement is the ceiling, because a judge can't agree with people more than people agree with each other.
- **Ask for a written reason with each label.** The reasons are what you use later to improve the judge prompt.
- **Include enough failures.** If only 5% of your outputs fail, a random sample of 100 has about 5 failures, which is too few to measure anything. Add more failing outputs on purpose, and record that you did so.

Expect the rubric to change while people label. Shankar and coauthors describe this effect in "Who Validates the Validators?" [3]. People need criteria to grade outputs, and they also discover the criteria while they grade outputs. Plan for a second labeling pass after the rubric settles.

As a starting point, plan for 100 to 200 labeled outputs per judge, split roughly in half between the development set and the test set, with at least 30 to 50 human failures across the set so the failure-agreement rate is not just noise. With a clear binary rubric and short written reasons, two domain experts can usually label 100 items in two to four hours, spread across a couple of short sessions so fatigue does not creep in. You can always add more labels later, especially if you need tighter confidence intervals on the error rates.

## Step 3. Split the labeled set before you touch the prompt

Divide the labeled items into a development set and a test set. You use the development set to read disagreements and change the judge prompt. You use the test set once, at the end, to measure the final judge.

The split is necessary because you will edit the prompt many times while you look at the development set, and the prompt will start to fit those specific items. Agreement measured on the same items you tuned on is too high. The same rule applies when an automatic prompt optimizer does the editing for you.

## Step 4. Measure agreement with more than one number

The obvious measure is the share of items where the judge and the people give the same label. That share can look good while the judge is close to useless. The example below shows how.

**A worked example.** To see the arithmetic clearly, suppose people labeled 200 outputs and passed 180 of them, and we test an uncalibrated judge that tends to be overly lenient.

| | Judge says pass | Judge says fail | Total |
|---|---:|---:|---:|
| People say pass | 176 | 4 | 180 |
| People say fail | 14 | 6 | 20 |
| Total | 190 | 10 | 200 |

The judge agrees with people on 182 of 200 items, which is 91%. But the judge passes almost everything. Of the 20 outputs that people failed, the judge caught only 6, which is 30%. A judge that misses 70% of the failures does not do the job you built it for.

I report three numbers for every judge.

- **Agreement on passes.** Of the outputs people passed, the share the judge also passed. The standard name is the true positive rate. In the example it is 176 of 180, or 98%.
- **Agreement on failures.** Of the outputs people failed, the share the judge also failed. The standard name is the true negative rate. In the example it is 6 of 20, or 30%.
- **Cohen's kappa.** Kappa is agreement after you subtract the agreement that two labelers would reach by chance, given how often each one says pass [1].

Here is the kappa calculation for the example. The judge says pass 95% of the time and people say pass 90% of the time, so by chance alone they would agree on 0.95 × 0.90 + 0.05 × 0.10 = 0.86 of the items.

```
kappa = (observed − chance) / (1 − chance) = (0.91 − 0.86) / (1 − 0.86) = 0.36
```

Landis and Koch describe a kappa between 0.21 and 0.40 as "fair" agreement, and a kappa above 0.60 as "substantial" [2]. So the judge with 91% raw agreement has only fair agreement once you account for chance.

## Step 5. Read every disagreement

The numbers tell you how big the problem is, and the disagreements tell you what the problem is. Read each development item where the judge and the people differ, together with the judge's reasoning and the person's written reason. Each disagreement falls into one of three groups.

- **The judge is wrong.** The judge missed a rule, or applied a rule that isn't in the rubric. You fix these by changing the judge prompt.
- **The person is wrong.** Labelers make mistakes too. Fix the label and keep a count, because the count tells you how noisy your labels are.
- **The rubric is unclear.** Both readings are reasonable. Decide which one you want, write the decision into the rubric, and relabel the affected items.

The third group is where most of the real calibration work happens. Take the criterion from step 1: "Does the answer resolve the user's problem?" Suppose an answer gives the right fix, but uses an outdated menu name for one button or skips a caveat that only matters in a rare edge case. One reviewer passes it because a user would still figure it out; another fails it because someone following the steps literally would get stuck. Neither reviewer is wrong. Until you decide where to draw that line, write the decision into the rubric, and relabel the affected items, no prompt edit can make the judge agree with both of them.

## Step 6. Change the judge and measure again

These changes have helped me most, roughly in the order I try them.

1. Add the rubric decisions from step 5 to the prompt as explicit rules.
2. Add a few labeled examples from the development set, including examples of the failures the judge missed.
3. Ask the judge to write its reasoning before it gives the verdict. The reasoning also makes the disagreements easier to read.
4. Fix the settings that add noise. Set the temperature to 0, pin the exact model version, and require a fixed output format.

After each change, run the judge on the development set again and compare the three numbers. Watch for the known biases of LLM judges while you do so. Zheng and coauthors found that judges can favor the answer in a certain position, favor longer answers, and favor answers written by the same model as the judge [4].

### Using a prompt optimizer

Once you have clean labels, written reasons, and a clear metric, you do not have to make every prompt edit by hand. Prompt optimizers can search through prompt revisions much faster than a person can, and they can easily improve a judge prompt beyond what a few rounds of manual edits reach.

Two practical approaches work well for judge prompts.

- **Reflective prompt optimizers like GEPA.** Instead of looking only at whether the judge matched the label, GEPA reads the judge's reasoning and the human's written reason on the items the judge missed [7]. It uses that text feedback to diagnose why the judge failed, rewrite the prompt, and keep the strongest candidates across iterations. The written reasons you collected in step 2 do double duty here: they tell the optimizer why the judge was wrong, not just that it was wrong.
- **An autoresearch loop.** Following the pattern from Andrej Karpathy's `autoresearch` [8], you give a coding agent a single editable file (the judge prompt) and a read-only evaluation script that scores the judge on the development set. In each iteration, the agent inspects the current disagreements, edits the prompt, runs the evaluation, keeps the edit if the metric improves, and reverts it if not. Left to run for a few dozen iterations, a loop like this can uncover edge-case rules and phrasing improvements that are tedious to find by hand.

An optimizer works best when you pair it with three guardrails.

- **Clean the labels and rubric first.** An optimizer is great at fixing the first group of disagreements from step 5, where the judge missed a rule. It cannot fix noisy labels or an unsettled rubric. Do at least one pass reading disagreements yourself before you turn an optimizer loose, or it will learn to copy your labeling mistakes.
- **Optimize the right metric.** Never ask an optimizer to maximize raw agreement. As step 4 showed, an optimizer given raw agreement on an imbalanced set will quickly discover that passing almost everything scores well. Optimize Cohen's kappa, or a score that balances agreement on passes and agreement on failures.
- **Protect the test set.** The optimizer needs the same protection as a person does, which is a test set that it never sees. Because an optimizer can try dozens or hundreds of prompt edits in an hour, it overfits the development set much faster than a person does. Keep the test set out of reach of the optimizer and its evaluation script. If you have enough data, split the development set itself into a subset the optimizer inspects and a small validation subset used to decide whether to keep each edit, saving the test set for the single final measurement in step 7.

## Step 7. Measure once on the test set and decide

When the development numbers stop improving, run the judge on the test set once. Compare its kappa with the kappa between your two human labelers. If the judge agrees with people about as well as people agree with each other, the judge is as good as your labels can show. If the judge is far below that level, go back to step 5, and collect a new test set before the next final measurement.

As a practical rule of thumb, aim for a test-set kappa of at least 0.60, which is Landis and Koch's "substantial" threshold [2], and within about 0.05 to 0.10 of the kappa between human labelers. Check TPR and TNR separately as well, because the right floor depends on how you use the judge. For offline trend tracking where you correct the pass rate in step 8, 80% to 85% agreement on both passes and failures is often enough to be useful. If the judge is gating releases or catching safety and policy failures, push agreement on failures to 90% or above before trusting it on its own.

## Step 8. Correct the numbers you report

A calibrated judge still makes mistakes, but now you know its error rates, and you can use them. If the judge reports a pass rate on a large unlabeled set, you can estimate the pass rate people would have given with the correction below. The correction comes from Rogan and Gladen, who wrote it for medical tests with known error rates [5].

```
corrected pass rate = (judge pass rate + TNR − 1) / (TPR + TNR − 1)
```

In the example, the judge reports a 95% pass rate. The corrected rate is (0.95 + 0.30 − 1) / (0.98 + 0.30 − 1) = 0.90, which matches what people said. The correction is only as good as the error rates behind it. With 20 human failures, the 30% is a rough estimate, so the corrected number has a wide margin of error. Prediction-powered inference is a more complete method that gives confidence intervals for the same kind of estimate [6].

## Step 9. Check the judge again when something changes

Calibration describes one judge prompt, one judge model, and one kind of output. Measure again when any of them changes. A new version of the judge model can shift the verdicts. A new version of the product can produce outputs the rubric never covered. I keep a small stream of new human labels coming in so that I can compare them with the judge on a schedule and see the drift early.

## Summary

- Ask the judge one pass or fail question for each criterion.
- Collect labels from two people who know the domain, with reasons, and include enough failures.
- Split the labels into a development set and a test set before you edit the prompt.
- Report agreement on passes, agreement on failures, and kappa. Do not report raw agreement alone.
- Read the disagreements, and fix the prompt, the labels, or the rubric.
- Once the labels and rubric are clean, consider a prompt optimizer such as GEPA or an autoresearch loop to iterate on the judge prompt against kappa while keeping the test set hidden.
- Compare the judge with human agreement on the test set, and correct reported rates with the judge's error rates.
- Measure again when the judge model, the prompt, or the outputs change.

## References

1. Cohen, J. (1960). A coefficient of agreement for nominal scales. *Educational and Psychological Measurement*, 20(1), 37–46.
2. Landis, J. R., and Koch, G. G. (1977). The measurement of observer agreement for categorical data. *Biometrics*, 33(1), 159–174.
3. Shankar, S., et al. (2024). Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences. [arXiv:2404.12272](https://arxiv.org/abs/2404.12272)
4. Zheng, L., et al. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. [arXiv:2306.05685](https://arxiv.org/abs/2306.05685)
5. Rogan, W. J., and Gladen, B. (1978). Estimating prevalence from the results of a screening test. *American Journal of Epidemiology*, 107(1), 71–76.
6. Angelopoulos, A. N., et al. (2023). Prediction-powered inference. *Science*, 382(6671), 669–674. [arXiv:2301.09633](https://arxiv.org/abs/2301.09633)
7. Agrawal, L. A., et al. (2025). GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning. [arXiv:2507.19457](https://arxiv.org/abs/2507.19457)
8. Karpathy, A. (2026). autoresearch. [github.com/karpathy/autoresearch](https://github.com/karpathy/autoresearch)
