# CLAUDE.md

Project instructions for the champion winrate decomposition project.

## What this project is

A champion's raw winrate is a mixture of three things: how strong the champion is,
who plays it, and what situations it gets played in. This project separates them
so we can say *why* a champion sits at the winrate it does.

The target output for any champion looks like:

> Olaf top has a 53% winrate. Standardized to the average top laner's matchup
> distribution it is 50.8%. Standardized to the average top laner's mastery and
> rank distribution it is 52.4%. Standardized to both it is 50.2%. Conclusion:
> most of Olaf's surplus is matchup selection, not champion strength.

The gap between raw and standardized winrate *is* the result. We are not estimating
a causal effect of champion choice. We are attributing an observed gap to
composition versus returns.

## Scope discipline

Three analyses that share one data layer. Each is publishable alone. Build them in
this order and do not start the next until the previous one ships.

1. **Matchup selection.** How much of a champion's winrate comes from which
   opponents it faces. Cheapest, most self-contained, highest confidence.
2. **Playerbase composition.** How much comes from who plays it: rank, role
   primacy, account maturity, champion mastery, autofill.
3. **Mastery curve decomposition.** How performance changes with games played on
   the champion, net of composition. Hardest, most caveated, do last.

Do not build a general framework first. Build analysis 1 end to end, then extract
shared pieces.

Scope every run to one role, one patch range, one rank band unless there is a
specific reason to widen. Widening multiplies cells and buys little.

## Methodological commitments

These are decisions already made. Do not silently revisit them. If the data argues
against one, say so explicitly rather than quietly switching approach.

- **Decomposition by reweighting (Oaxaca-Blinder style).** Define a reference
  distribution (the average laner in the role) and reweight each champion's games
  to it, one factor at a time and then jointly. Report raw, each single-factor
  standardized rate, and the joint standardized rate.
- **No ANOVA, no variance-share framing.** Binary outcome and unbalanced design
  break it mechanically, and "share of variance explained" answers the wrong
  question anyway.
- **No raw cell means.** Champion x opponent x mastery bin x rank is tens of
  thousands of cells, most nearly empty. Use a hierarchical logistic model with
  partial pooling toward champion and matchup main effects so thin cells shrink
  instead of producing fake 62% winrates.
- **Prefer simple, well-specified GLMs.** A logistic model with sensible
  interactions and shrinkage is the default. Anything fancier needs to beat it on
  held-out data first.
- **Pre-pick vs post-pick.** Only variables fixed at or before champion select are
  legitimate controls (rank, mastery, role primacy, side, opponent, team comp,
  premade status, patch). Post-pick variables (gold, CS, KDA, items, game length)
  are mediators and are never used as controls. No kitchen-sink models.
- **Player fixed effects for selection** where the panel allows it.
- **Within-player difference-in-differences** (champion performance vs the same
  player's off-champion baseline) is the preferred mastery outcome.
- **Balanced panels** for anything over time, to avoid attrition bias from players
  who quit a champion after losing.
- **Rank is a dimension, not a pooled control.** Report by rank band and check
  interactions before pooling.

## Factor inventory

Things that can move a champion's raw winrate without the champion being stronger:

- **Matchup selection.** Counterpick champions appear disproportionately in
  favorable matchups.
- **Team composition** beyond the lane opponent (champions picked into specific
  comps, e.g. engage-dependent picks).
- **Playerbase skill.** Mean rank, off-champion winrate, account age.
- **Champion mastery.** Games on champion and games in role.
- **Role primacy / autofill.** Share of games played off the player's main role.
- **Concentration.** How games are distributed across players (one-tricks vs broad
  casual pool). Measure with mean share of a player's games or a Gini over
  games-per-player. Playrate alone does not capture this.
- **Duo and premade status.**
- **Build and rune divergence** from the common setup (descriptive only; it is
  post-pick, never a control).
- **Ban truncation.** High ban rate leaves a selected set of games where nobody
  prioritized the ban.
- **Patch drift.** Strength changes within any window long enough to accumulate
  data.

## Known traps

- **Counterpick without pick order.** Soloq match data has pickTurn for bans, not
  picks. Measure matchup selection without it: compare the champion's observed
  matchup distribution to the product of the role's marginals. Excess mass in
  favorable matchups is selection, whoever caused it. Side (blue first pick, red
  last pick) is a coarse secondary proxy only.
- **The blind-pick recursion.** "Strong counterpick, weak blind" is not cleanly
  identified, because players who blind a counterpick champion are themselves a
  selected group. Report the blind-pick component as a bound or range and say
  which way the bias runs.
- **Draft learning read as gameplay learning.** Part of a counterpick champion's
  mastery curve is learning when to pick it. Separate this by reweighting each
  mastery bin to a fixed matchup distribution. If the curve flattens, that part
  was draft timing. Plot expected matchup favorability against champion games to
  isolate it.
- **Composition masquerading as learning.** A steep early curve often reflects
  who plays a champion at low games (autofilled, low-effort, new accounts), not
  how fast it is learned. Yuumi is the reference case.
- **Matchmaking absorbs mastery.** For high-concentration champions, MMR adjusts
  to the player's skill on that champion, flattening the observed curve. These
  champions are genuinely hard to measure and also lack an off-champion baseline
  for diff-in-diff. Flag them rather than trusting the fitted curve.
- **Role assignment.** Positional data is inferred and imperfect. Check the
  failure rate before trusting anything conditioned on role.
- **Thin bins.** Many published "steep mastery curve" claims are noise in bins
  with a few hundred games.

## Diagnostics to run before modeling

For each champion, by games-played bin, plot the population rather than the
outcome: mean player rank, mean off-champion winrate, share of games outside
primary role, share of low-total-games accounts. If early bins sit far below late
bins on these, composition is doing the work, and that can be shown descriptively
without a model.

Run these first. They often answer the question.

## Curve fitting

Do not compare mastery curves visually. Fit a common form per champion, e.g.

    y = a + b * (1 - exp(-c * x))

and compare parameters: `a` is the floor, `b` is total available gain, `c` is how
fast it is realized. Plot champions in (a, b, c) space so claims about steepness
become specific and falsifiable.

## Code conventions

- Python. Prefer a columnar dataframe library (polars) over row-wise iteration.
- Separate layers: raw pull, cleaned match table, per-analysis feature build,
  model, figures. Modeling code never reaches back into raw pulls.
- Cache API pulls to disk. Never re-pull for a rerun. Respect Riot rate limits.
- Every aggregate carries its sample size. Every figure showing a rate shows n.
- Report uncertainty intervals on every standardized winrate.
- Seed everything.
- Keep the reference distribution definition in one place. Every analysis uses it,
  and drift between them is silent and fatal.

## Writing conventions

Output goes to machineloling.com. Analytical content in casual,
personality-driven prose.

- No em dashes. Plain punctuation.
- Conclusion first, then evidence, then caveats. Do not open with methodology.
- Caveats are specific or absent. No generic hedging.
- If a number is weakly identified, say which direction the bias runs.
