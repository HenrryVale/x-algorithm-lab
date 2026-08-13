# X Algorithm Lab

Research workspace built on top of the upstream `xai-org/x-algorithm` repository.

## Branch strategy

- `main`: keep aligned with upstream. Avoid lab-only edits here.
- `research`: architecture notes, source mapping, signal analysis, and reproducible findings.
- `experiments`: local simulations and ranking experiments.
- `x-ranking-agent`: agent implementation for explainable post analysis.
- `skill-x-ranking`: reusable skill/prompt package for agent runtimes.

## Research goal

Build an explainable analyzer that maps a proposed post to the public ranking and visibility mechanisms in this repository. The tool should cite the exact source files and distinguish:

1. facts directly visible in source code;
2. defaults mirrored into the public repo;
3. assumptions or heuristic estimates made by the lab;
4. behavior that cannot be known from the public repository.

The analyzer must not present a deterministic "viral score" as if it were X's production score. Phoenix predicts viewer-specific action probabilities, configuration may vary by experiment, and some anti-abuse rules/prompts are intentionally unpublished.

## Current public request path

At a high level:

`viewer context -> candidate retrieval -> candidate hydration -> pre-scoring filters -> Phoenix action probabilities -> RankingScorer -> Top-K -> visibility filtering -> blending`

Candidate sources include Thunder for in-network posts and Phoenix/SimClusters for out-of-network retrieval.

## Ranking model

The public `RankingScorer` consumes Phoenix probabilities for positive, attention, author, and negative-feedback actions and combines them using configured weights.

Conceptually:

```text
score ~= sum(weight_i * P(action_i))
```

This is followed by additional adjustments such as author diversity and out-of-network handling, and then reranking.

### Public default weights observed in `home-mixer/params/param.rs`

These are repository defaults, not a guarantee that every production request uses them.

| Signal | Default weight |
|---|---:|
| Favorite | 0.5 |
| Reply | 5.0 |
| Retweet | 1.0 |
| Photo expand | 0.05 |
| Video open | 0.05 |
| Click | 0.4 |
| Open link | 0.2 |
| Profile click | 0.0 |
| Video quality view | 0.05 |
| Share | 2.0 |
| Share via DM | 5.0 |
| Share via copy link | 20.0 |
| Quote | 5.0 |
| Follow author | 4.0 |
| Post unexplored | 0.02 |
| Continuous dwell time | 0.004 |
| Not interested | -43.2 |
| Block author | -31.2 |
| Mute author | -58.8 |
| Report | -234.0 |
| Not dwelled | -0.02 |

There is also a public default bidirectional-follow reply boost of `15.0` for eligible candidates.

## Important configuration caveat

Many tunable values are read from a configuration/experimentation system. The upstream README states that defaults in files such as `home-mixer/params/param.rs` are periodically synchronized to primary production values, while experiments can override them for portions of traffic.

Therefore every lab result should include a confidence label:

- **Source-exact**: directly implemented in public code.
- **Default-derived**: based on public default parameters that may be overridden.
- **Heuristic**: an estimate produced by our analyzer.
- **Unknown**: requires private models/configuration/rules or live viewer state.

## Analyzer design

Proposed pipeline:

```text
Post + optional author/context
        |
        v
Static feature extraction
        |
        +--> visibility/risk checks
        |
        +--> engagement-affordance analysis
        |
        +--> source-backed ranking-signal map
        |
        v
Explainable report
  - likely positive signals
  - likely negative-feedback risks
  - uncertainty/confidence
  - exact source references
  - optional rewrite suggestions
```

The analyzer should optimize for useful, authentic content rather than spam, deceptive engagement bait, or attempts to evade safety/anti-abuse systems.

## Initial source map

- `README.md` — upstream architecture and transparency notes.
- `home-mixer/params/param.rs` — public ranking/config defaults.
- `home-mixer/scorers/ranking_scorer.rs` — scoring arithmetic and post-score adjustments.
- `home-mixer/scorers/phoenix_scorer.rs` — Phoenix score integration.
- `home-mixer/filters/` — pre/post scoring filtering.
- `visibility-filtering/` — visibility decisions.
- `phoenix/` — retrieval/ranking model training and serving code.
- `simclusters/` — out-of-network candidate retrieval.
- `vm-ranker/` — diversity-aware reranking service.

## Next research tasks

1. Trace every field in `PhoenixScores` to its model head and scorer usage.
2. Extract ranking defaults automatically into a machine-readable snapshot.
3. Document author-diversity, OON discount, new-author boost, and reranking math.
4. Build a small offline simulator that accepts synthetic action probabilities.
5. Add tests proving that source extraction does not silently drift after upstream updates.
6. Design the agent/skill with source citations, uncertainty labels, and prompt-injection-resistant handling of untrusted post text.
