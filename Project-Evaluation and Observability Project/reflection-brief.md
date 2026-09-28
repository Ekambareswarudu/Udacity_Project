**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have shipped.

> System 2's `discrepancy-run.txt` is the clearest example. The extraction produced structured values, but the deterministic validator caught that the calculated total was `9642.17` while the stated total was `10892.17`, with a `-1250.0` delta. A trusting design could have accepted the structured extraction without checking the relationship between the fields.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> System 1 demonstrates this directly in its routing tests. The suite includes `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review`. The design therefore does not treat high model confidence as sufficient when an independent reviewer disagrees.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> For an insurance-claims intake workflow, I would use independent review with deterministic routing when the extracted fields affect a downstream decision. I would instrument field-level confidence, reviewer disagreement, validation failures, escalation counts, retry counts, and calibration sliced by document type and field. These measurements would show whether the extraction is degrading even when the overall success rate still looks acceptable.
