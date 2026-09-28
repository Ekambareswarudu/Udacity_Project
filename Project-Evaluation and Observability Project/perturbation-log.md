# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
  Used the routing test perturbation for a missing required source field, represented by the `missing_source` routing/retry tests in `tests/test_us01_retry.py` and `tests/test_us04_routing.py`. The system is designed to stop immediately when a required source field is missing rather than retrying.
- **Command I ran:**
  `.venv/bin/pytest tests/test_us04_routing.py -v`
- **What I predicted:**
  A missing required source field should not be retried as if it were a recoverable model-format error; it should escalate to human review.
- **What actually happened (paste the key output line):**
  `tests/test_us04_routing.py::test_ac_04_02_integration_failure_routes_to_human_review PASSED`
- **How this differs from the unperturbed run:**
  The routing tests confirm that failure conditions such as integration failure and low confidence route to `human_review`, while an all-clear case routes to `auto_approve`. The live end-to-end API run was not available because the workspace had no API credentials, so no live routing JSON was produced.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
  Used the deliberately perturbed fixture `fixtures/documents/income_sum_mismatch.txt`, where the stated monthly income total does not match the sum of the extracted components.
- **Command I ran:**
  `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:**
  The extraction would still produce schema-valid JSON, but deterministic validation would detect the mathematical inconsistency.
- **What actually happened (paste the key output line):**
  `"consistent": false` with `"calculated": 9642.17`, `"stated": 10892.17`, and `"delta": -1250.0`.
- **How this differs from the unperturbed run:**
  The normal appraisal replay reported `"consistent": true` with no discrepancies, while the perturbed income fixture was accepted structurally but rejected by the consistency validator.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):**
  Ran the Meridian investigation with the logistics source deliberately made unavailable using `--simulate-timeout`.
- **Command I ran:**
  `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:**
  The system would preserve the remaining sources, explicitly report the unavailable logistics source, and mark logistics-dependent information as incomplete instead of treating it as zero or absent.
- **What actually happened (paste the key output line):**
  `Sources unavailable: logistics unavailable (timeout)`
- **How this differs from the unperturbed run:**
  In the normal run, `on_time_delivery_rate` was contested between 95.0% and 78.0% from two sources. With logistics unavailable, only the 95.0% supplier-audit value remained and the metric moved out of the contested section; `late_shipment_count` became incomplete because its logistics source timed out.
