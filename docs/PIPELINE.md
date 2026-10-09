# Initial pipeline and validation plan

This is a proposed implementation, not a report of completed experiments. Hardware, endpoint capabilities, throughput, and accuracy remain unverified.

## Hypothesis

For a short, clear standing exchange, tracked limb motion can propose most attack attempts, and a video-understanding model can improve contact classification while preserving bounded end-to-end delay.

## 1. Video input and identity

- Begin with a continuous 30–60 second shot at normal speed, ideally with distinct clothing and both fighters' full bodies visible.
- Replay frames according to source timestamps. Do not expose future frames ahead of simulated live arrival.
- Keep a three-second rolling buffer at the original frame rate; start pose inference at 25–30 FPS with 640-pixel model input.
- Manually assign fighter A and B in the first frame and exclude the referee.
- Start with `yolo11n-pose` and ByteTrack; consider BoT-SORT for camera motion.
- Bind fighter identity to a maintained track mapping, not screen-left versus screen-right. Reassociate after cuts or ambiguous occlusion; suspend attribution while uncertain.

## 2. Candidate generation

Track wrist and ankle trajectories relative to the fighter's torso. Normalize distances and velocities by body scale. Estimate coarse head, torso, and leg regions from pose keypoints.

Propose an attack when a limb moves toward an opponent region, reaches a local distance minimum, then retracts or follows through. These rules detect candidates only: image-plane overlap does not prove physical contact, and wrist keypoints do not localize the glove surface.

Use per-limb states: ready -> approaching -> candidate -> retracting -> ready. Avoid a fixed one-second cooldown that could suppress combinations. Preserve multiple attacks within the same exchange.

## 3. Asynchronous verification

For a candidate at source time `t`, wait approximately 0.3 seconds and select footage from `t - 0.6` to `t + 0.3`. Keep a crop covering both fighters and the full attack trajectory.

Start with 12–20 source frames per clip, densely sampled around possible contact. Verify the endpoint's actual preprocessing and frame count: the documented Cosmos Reason2 NIM default is 4 FPS, which can omit brief contact. A higher upload frame rate alone does not guarantee higher model sampling.

Attach A/B identity markers and frame timestamps. Ask for compact structured output. Several nearby candidates may share a request, but each attack must retain its own event ID and outcome.

Outcome definitions:

| Outcome | Meaning | Heatmap behavior |
| --- | --- | --- |
| `landed` | Visible evidence supports direct contact with the reported body zone | Increment the defender's zone once |
| `blocked` | A defensive glove, forearm, or other guard intercepts the strike | Track separately |
| `missed` | Visible attack without contact | No increment |
| `unknown` | Occlusion, blur, identity uncertainty, or depth ambiguity prevents a decision | No increment; retain for review |
| `not_strike` | Feint or other motion incorrectly proposed as an attack | Discard from attack totals |

Illustrative event (not an observed result):

```json
{
  "event_id": "e017",
  "time_s": 12.43,
  "attacker": "A",
  "defender": "B",
  "limb": "right_hand",
  "outcome": "landed",
  "contact_zone": "head",
  "evidence_times_s": [12.40, 12.43, 12.47]
}
```

Allow null identity or contact zone where unknown. Validate outputs before use. Model self-reported confidence is not a calibrated probability. A defender's head movement alone is insufficient evidence of contact.

## 4. Scheduling and latency

Keep frame processing, model requests, and rendering asynchronous. Start with one active verification request and one waiting request. Merge overlapping candidate windows where possible without losing distinct attacks. Record timeouts and overload explicitly rather than silently dropping events.

Engineering targets, subject to measurement:

| Metric | Initial target |
| --- | --- |
| Pose / tracking throughput | Sustain 25–30 FPS |
| Complete per-frame processing | Average below approximately 33–40 ms at the selected input rate |
| Frame-to-overlay latency | P95 <= 200 ms |
| Contact-to-event-display latency | P95 <= 2 seconds |

Event latency includes post-contact observation, clip encoding, transfer, queuing, inference, and display. Throughput must be sufficient for sustained request arrival; low latency on isolated clips is not enough.

Warm models before steady-state measurements and report startup separately. Reduce unnecessary image area and output tokens before sacrificing contact-frame sampling. Use supported FP16 execution and consider TensorRT only if measured frame processing misses its budget.

Publish pending candidates separately from accepted events. Deduplicate heatmap updates by event ID; corrections must remove or replace the previous contribution. If the video-model endpoint is unavailable, keep final contact decisions pending and validate only tracking and candidate generation.

## 5. Evaluation

Manually label every attack attempt, including misses and blocks: source time, attacker, defender, contact region, outcome, and visibility. Prefer two independent annotations; mark disagreements as uncertain.

Use the first 10–15 seconds for threshold tuning and freeze parameters before evaluating the remaining continuous footage. Add another clip if the held-out portion contains too few events.

Compare geometry-only decisions against the same candidates with Cosmos verification. Report candidate recall separately so a verifier cannot conceal attacks the candidate generator never proposed.

Match predictions to annotations one-to-one within a predeclared time tolerance, initially +/- 0.2 seconds. Count duplicate predictions as extra outputs rather than matching them repeatedly to one annotation.

Proposed acceptance criteria:

- No A/B identity swaps on the held-out clip.
- Candidate recall >= 95% of annotated attack attempts.
- Joint landed-event precision >= 90%, requiring correct attacker, defender, contact zone, and landed outcome.
- Joint landed-event recall >= 70% over all visually decidable true landed events.
- A true landed event returned as unknown still reduces recall.
- Report a blocked/missed/landed confusion table, all unknown counts, and predictions made on human-uncertain footage separately.
- Report P50, P95, and maximum contact-to-display latency, including any dropped or timed-out events.
- Report raw counts, such as 9/10, alongside percentages. This clip is a feasibility check, not a general benchmark.

## 6. Viewer interface

Display the source video with stable A/B overlays, one received-strike heatmap per fighter, a timestamped event list, and evidence replay. Accumulate detected received strikes rather than inferred injury severity. Leave win-probability modeling outside the first milestone.

The first implementation check is a ten-second pose-and-tracking overlay. Verify identity and wrist/ankle tracking before building downstream contact statistics.
