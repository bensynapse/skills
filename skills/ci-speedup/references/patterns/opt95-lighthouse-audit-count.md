---
pattern: OPT95
impact: MEDIUM
class: data-driven
detector: critical-path-log-leaf
risk: MEDIUM
affected_files: '.github/workflows/*.yml,.github/workflows/*.yaml,lighthouserc*,.lighthouserc*'
fix_strategy: lighthouse-audit-count
title_template: 'Lighthouse CI audits every URL N times per pass, one after another'
category: 11
---

# OPT95 — Lighthouse CI Audits Every URL N Times Per Pass, One After Another

**TL;DR**: A merge-gating job spends most of its wall time inside Lighthouse CI's collect phase. Every configured URL is audited `numberOfRuns` times, each run a complete cold page load plus the full audit (roughly 10 to 15 s on a hosted runner), the runs go one after another by design, and a job that runs one pass per device preset pays all of it again per pass. The audit count is the product passes x URLs x runs. The report names that product, measured from the log, and hands the owner the three dials: runs per URL, the URL list, and where the passes run. It credits no saving.

**Scope: Lighthouse CI only** (`@lhci/cli` via `lhci autorun` / `lhci collect`, and the GitHub action that drives the same collector). Other audit tools print other shapes and stay out of scope rather than being guessed at.

**Anti-pattern**: A `lighthouserc` config lists every page of the site, keeps or raises the default of 3 runs per URL (the Lighthouse variability guidance recommends a median of several runs, so 5 is common), and the workflow runs the collector once per device preset as consecutive steps. The budget check that was meant to take a minute now takes ten, and almost every audit re-audits a page the job already audited: with 4 URLs, 5 runs and 2 passes, 36 of 40 audits are repeats.

**Detection heuristic** (routed from the drilled long pole's captured job log by the `lighthouse-serial-audits` leaf detector in `blocking_path.py`, not by the static scan and not by the critical-path router; ARCHITECTURE §12.3). Everything below is read from the log, nothing inferred:

1. The log carries at least one COMPLETED collect pass: `Running Lighthouse N time(s) on <url>` lines (one per configured URL) closed by `Done running Lighthouse!`. A trailing block with no closing marker (a cancelled or crashed collect) is not counted; its audits never completed a gate.
2. Summing N over every URL of every completed pass gives at least 20 audits (lhci's default of 3 runs over a couple of URLs is a smoke audit and stays silent) AND at least two audits per distinct URL, so the count is a multiplier from repeat runs and/or repeat passes. One run over many URLs is a different lever (sharding the URL list) and stays silent here rather than being mislabelled.
3. The pole's measured dominant step is the Lighthouse step: category `scan` (the step categorizer bins `lighthouse` / `lhci` step names there) with a name carrying `lighthouse` or `lhci`. Otherwise the leaf is demoted to a labelled secondary observation (ARCHITECTURE §12.4b), never crowned as the pole's cause, exactly as the eslint leaf is demoted off a type-check-dominant pole.

The magnitude it reports is the share of audits that re-audit a URL the job already audited at least once (the repeat runs and repeat passes), validated across the sampled runs; it is config-driven, so it is stable run to run. The evidence is verbatim lhci output: the first `Running Lighthouse` line of each pass and lhci's own `Checking assertions against U URL(s), M total run(s)` total, which states the pass's audit count in the tool's words.

**Fix recipe** (the report hands off; it does not prescribe):

1. Read the config history first (`git log -p` on `lighthouserc*` and the workflow): why N runs, these URLs, these presets. A deliberate choice is a policy change for the owner, not a quick win.
2. Runs per URL. lhci's assertions aggregate across the runs of a URL (`aggregationMethod`, set per assertion: `median`, `optimistic`, `pessimistic` or `median-run`), so keep N at what those assertions need for a stable verdict, and compare the old and new medians side by side before lowering it. Lighthouse itself recommends a median of several runs over a single run; dropping to 1 trades speed for a flakier, or falsely green, budget.
3. The URL list: one representative page per template rather than every page, keeping every URL that carries a budget.
4. Where the passes run. The device presets (and URL shards) are independent of each other, so run them on separate runners (a matrix leg per preset) instead of one after another. Never run two Lighthouse passes at once on one runner: each `lhci collect` starts its own static server, and concurrent audits contend for the same cores, which skews the very scores the budget gates on. Lighthouse's own guidance is to scale across machines, not within one.

**Required-checks caveat**: a split into matrix legs creates new check names. If the Lighthouse job is a required status check, add the new jobs to branch protection as required checks (or the ruleset equivalent), or the split-out audits silently stop gating merges while everything stays green, until that admin-only re-gating step is done.

**Sizing: uncredited.** The skill measures the audit count and the repeat share, not the saving: how far N can drop is the owner's measurement-policy decision, a trimmed URL list is a coverage decision, and a runner split moves wall time off the job rather than removing it. Any addressable ceiling the report shows for the pole is that pole's measured wall, not this lever's saving.

**Risk: MEDIUM.** Fewer runs or fewer URLs weaken a performance budget's signal (a flakier gate, or a regression that slips through). Keep every assertion gating, and verify that a deliberately regressed page still fails the check after the change.
