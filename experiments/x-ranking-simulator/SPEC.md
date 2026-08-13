# Simulator v1 specification

This specification defines an offline, explainable reproduction of the deterministic parts of the public weighted `RankingScorer` path.

## Inputs

A viewer context and one or more synthetic candidates. Each candidate provides model-head values such as favorite, reply, repost, share, quote, dwell time, follow-author, and negative-feedback heads, plus context such as in-network status, reply/repost status, mutual-follow status, and video duration.

## Weighted stage

For each candidate, multiply each supplied head by its public default weight from `public-defaults.json`, then split terms into positive and negative parts.

`combined = positive_part - negative_part`

For non-negative combined values, add the public `0.001` offset. For negative combined values, apply the same normalization formula used by `RankingScorer::offset_score` with the public positive/negative weight totals.

## Context-dependent adjustments

The implementation should mirror the public source behavior for bidirectional-follow reply/dwell weight adjustments, VQV duration/follower gating, quoted VQV duration gating when enabled, and additive `post_unexplored` gating.

## Author diversity

Sort candidates by the pre-diversity weighted score. For each author, count how many candidates from that author appeared earlier in this ordering. Apply:

`multiplier(k) = (1 - floor) * decay^k + floor`

Current public defaults are `decay = 0.5` and `floor = 0.25`.

## Out-of-network scaling

Apply the public OON multiplier after author diversity. The current default is `0.75`, or `0.5` for topic-scoped requests. By default, the same OON rescaling also applies to in-network replies and reposts.

## Output

Return each candidate with its weighted score, positive and negative parts, per-signal contribution breakdown, author-diversity multiplier, OON multiplier, and `final_score_before_vm_ranker`.

## Explicit non-goals

Do not estimate real Phoenix probabilities from post text in this layer. Do not fabricate viewer experiment overrides, `AuthorColdStart` experiment assignment/random target, VMRanker DPP reranking, visibility-filtering outcomes, unpublished rules, or production retrieval/blending state.

## Source anchors

- `home-mixer/scorers/ranking_scorer.rs`
- `home-mixer/params/param.rs`
- `home-mixer/params/config.rs`
- `home-mixer/util/candidates_util.rs`
- `home-mixer/scorers/author_cold_start.rs`
