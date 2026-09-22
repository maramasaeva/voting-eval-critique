# Taking an LLM-judge eval apart

A critique and partial rebuild of a benchmark that measures whether AI systems give
voters accurate election-logistics information: registration deadlines, ID rules,
postal-ballot return rules, across three national elections.

15 questions x 2 frontier models = 30 cached answers, graded by an LLM judge.

**The code and the full write-up are available on request.** Open an issue, or reach
me at [maramasaeva.com](https://maramasaeva.com) or [@rssmrm](https://x.com/rssmrm),
and I'll share the repo.

---

## What the first run said

```
gpt-5-mini   accuracy 0.933   high-stakes 0.875   28/30 correct
gpt-5.4      accuracy 0.933   high-stakes 0.875   28/30 correct
```

Two different frontier models, the same number twice. A benchmark built to compare
systems could not tell these two apart, and nothing in the output said so. The entire
result rested on 2 wrong answers out of 30.

## What was actually wrong

Not the models. The scoring rule.

**One reference, one bit.** Every question carries a single free-text reference value
and the judge returns a single verdict for the whole thing. One item in the set is
twelve sentences holding at least four separable facts. Another is forty-three
characters and one date. Both were worth exactly the same amount of signal.

A binary has nowhere to put a partially correct answer. So when both models described
the registration rule correctly and omitted one deadline, the judge had to log it as
`incorrect` and `suppressive` while its own reasoning called it a missing detail. That
single item is essentially the whole result.

**The judge was primed.** It was shown the expected failure mode before it read the
answer. Unevenly, too: 9 of 15 items carry one, 6 do not, so items were graded under
two different instructions.

**A failed run could report a number.** A judge endpoint that rejects `temperature=0`
fails both branches of the call path, every item errors, the errors land in the
accuracy denominator, and the file is written with `accuracy: 0.0` and no error count.
Exit code 0. Total instrument failure is persisted in the same shape as a finding.

**Controls were pooled into the headline.** 3 of 15 questions are easy-by-design
sanity checks. All 6 of their answers are correct. They are 20% of the denominator.

## What was rebuilt

A second scorer alongside the original, so the two can be diffed and re-run against
each other rather than one replacing the other.

- **Blinded judge.** The expected failure, the model name and the item id are no
  longer in the prompt.
- **Claim-level grading.** Each reference decomposes into atomic claims, authored into
  the dataset rather than by the judge at runtime, and the judge returns a status per
  claim: asserted correct, asserted wrong, or omitted, each with a required evidence
  quote. An omission stops being an argument and becomes "one of four correct, three
  omitted, zero wrong".
- **Deterministic source checking.** The official-source field was the only metric
  separating the two models, and it was being produced by the judge under suspicion.
  It is now an authored per-election domain allowlist, unit-testable with no API key.
- **Aggregation that cannot lie.** Unscored rows are excluded from every denominator
  and reported as counts, controls are separated from real items, every denominator is
  stated, and the process exits non-zero if anything failed to score.

## What this does not establish

The judge is still unvalidated. There are no human labels, no second annotator, no
agreement statistic. Every finding above was produced by the instrument I am
suspicious of, which is the honest limit on all of it.

The next step is hand-labelling all 30 items blind and reporting percent agreement
alongside Cohen's kappa. At 28 of 30 in a single category kappa is close to
meaningless on its own, so a real reliability number needs a category-balanced
validation set, which is also the only way to test the verdict categories that never
fired in this run.

---

Self-directed work. I write about this at
[mess in ecessity](https://messinecessity.substack.com), and more of my evaluation
work lives at [maramasaeva/evals](https://github.com/maramasaeva/evals).
