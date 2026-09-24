---
name: bidirectional-timeframe-investigation-framework
description: Dataframe-first SOC investigation skill that pivots backward and forward from a fixed anchor time with append-only evidence tables.
---

# Bidirectional Timeframe Investigation Framework

This skill defines a reusable investigation method for SOC and threat hunting workflows where analysis must move in both temporal directions from a known start point.

The approach is explicitly dataframe-first and append-only:

- Build durable evidence tables that can grow over time.
- Preserve extraction provenance and query history.
- Support iterative hypothesis testing without destroying prior results.
- Reuse the same schema across future investigations.

## When To Use

Use this skill when any of the following are true:

- You have an initial detection, alert, ticket, or observed event and need timeline context before and after it.
- You need defensible chronology: precursors before the alert and impacts after the alert.
- You want to avoid ad hoc, one-off searches and instead maintain expandable investigation dataframes.
- You anticipate repeated enrichment runs as new telemetry arrives.
- You want consistent case-to-case operations with reusable query templates and schema.

Typical investigation scenarios:

- Suspected account compromise where the initial login event is known but campaign start and blast radius are unknown.
- Endpoint malware alert where earlier process lineage and later lateral movement need structured extraction.
- Cloud IAM anomaly where precondition policy changes and post-event API behavior need alignment.
- Data exfil triage where an unusual transfer sits at T0 and both setup and follow-on activity must be collected.

## Capabilities

This skill provides a repeatable capability set:

### 1. Anchor-centered case initialization

- Define `case_id` and `anchor_time_utc` once per case version.
- Track anchor provenance (source, reason, timezone assumptions).
- Version anchor corrections instead of mutating history.

### 2. Bidirectional extraction protocol

- Query backward interval `[T0 - delta_back, T0)`.
- Query forward interval `(T0, T0 + delta_forward]`.
- Optionally include a center interval around T0 for anchor-adjacent events.

### 3. Expandable dataframe architecture

- Canonical append-only evidence tables.
- Explicit directionality field and signed time delta from anchor.
- Schema evolution via additive columns and `schema_version`.

### 4. Retry and widening logic

- Treat empty results as query/field mismatch until disproven.
- Pull raw events to discover true field names.
- Retry with field variants and broader constraints.
- Expand windows iteratively and preserve each run's metadata.

### 5. Reproducibility and auditability

- Record query text, source connector, extraction run ID, and exact time bounds.
- Keep `raw_event` plus normalized fields for forensics and replay.
- Allow deterministic dedup while retaining evidentiary integrity.

## Workflow

### 1) Initialize anchor and case context

Required fields in `df_case_anchor`:

- `case_id`
- `anchor_version`
- `anchor_time_utc`
- `anchor_source`
- `anchor_reason`
- `timezone_assumption`
- `initial_window_back`
- `initial_window_forward`
- `investigator`
- `created_at_utc`

Rules:

- Anchor is immutable within a version.
- If corrected, create a new `anchor_version` and retain the prior version.
- Every extracted event references `case_id` and `anchor_version`.

### 2) Perform initial narrow bidirectional pull

Start narrow for speed and signal quality:

- Backward pass: e.g. `T0-30m` to `T0`
- Forward pass: e.g. `T0` to `T0+30m`

Capture full raw + parsed data in dataframe form, not free text.

### 3) Normalize and append to master evidence

`df_events_master` required columns:

- `case_id`
- `anchor_version`
- `extraction_run_id`
- `extraction_time_utc`
- `source_system`
- `source_index_or_table`
- `source_type`
- `event_time_utc`
- `direction_relative_anchor`
- `seconds_from_anchor`
- `entity_primary`
- `entity_secondary`
- `event_type`
- `severity`
- `raw_event`
- `parsed_fields_json`
- `query_used`
- `window_start_utc`
- `window_end_utc`
- `window_label`
- `schema_version`

Directionality computation:

- `seconds_from_anchor = event_time_utc - anchor_time_utc`
- `< 0` => `backward`
- `= 0` => `anchor`
- `> 0` => `forward`

### 4) Build entity and hypothesis layers

`df_entities_master` suggested columns:

- `case_id`
- `extraction_run_id`
- `entity_id`
- `entity_type`
- `first_seen_utc`
- `last_seen_utc`
- `seen_in_backward`
- `seen_in_forward`
- `event_count`
- `related_entities_json`
- `latest_snapshot_flag`

`df_hypothesis_log` columns:

- `case_id`
- `hypothesis_id`
- `statement`
- `status`
- `tested_at_utc`
- `supporting_run_ids`
- `notes`

`df_query_registry` columns:

- `query_id`
- `connector`
- `query_template`
- `required_fields`
- `fallback_fields`
- `last_success_rate`
- `avg_runtime_ms`
- `notes`

### 5) Evaluate coverage and widen iteratively

For each run, assess:

- Backward/forward event counts
- Field completeness and parse quality
- Entity continuity across both directions
- Unexplained gaps around the anchor

If incomplete, widen windows and retry:

- Symmetric widening: `+/- 30m` -> `+/- 2h` -> `+/- 24h`
- Asymmetric widening when the hypothesis demands it (e.g. more backward for staging activity)

Never overwrite historical rows from prior runs.

### 6) Deliver pass outputs

Each extraction pass should output:

- New append batch for `df_events_master`
- New/updated append batch for `df_entities_master`
- Hypothesis updates in `df_hypothesis_log`
- Query performance update in `df_query_registry`
- Timeline sorted by `seconds_from_anchor` ascending

## Patterns

### Pattern A: Minimal bidirectional extraction template (SPL style)

```spl
search index=<idx> <seed_constraints>
| eval anchor_time=strptime("<ANCHOR_UTC>", "%Y-%m-%dT%H:%M:%SZ")
| eval event_epoch=_time
| eval seconds_from_anchor=event_epoch-anchor_time
| eval direction_relative_anchor=case(
    seconds_from_anchor<0, "backward",
    seconds_from_anchor>0, "forward",
    true(), "anchor"
  )
| where (seconds_from_anchor>=-<BACK_SECS> AND seconds_from_anchor<=<FWD_SECS>)
| eval case_id="<CASE_ID>", anchor_version="<ANCHOR_VERSION>", extraction_run_id="<RUN_ID>"
| eval extraction_time_utc=strftime(now(), "%Y-%m-%dT%H:%M:%SZ")
| eval window_start_utc=strftime(anchor_time-<BACK_SECS>, "%Y-%m-%dT%H:%M:%SZ")
| eval window_end_utc=strftime(anchor_time+<FWD_SECS>, "%Y-%m-%dT%H:%M:%SZ")
| eval window_label="bidirectional_initial"
| table case_id anchor_version extraction_run_id extraction_time_utc _time direction_relative_anchor seconds_from_anchor _raw
```

### Pattern B: Dataframe append and deterministic dedup

```python
# Pseudocode pattern for append-only evidence management
# Inputs: df_new_events, df_events_master_existing

df_new_events["dedup_key"] = (
    df_new_events["case_id"].astype(str) + "|" +
    df_new_events["source_system"].astype(str) + "|" +
    df_new_events["source_index_or_table"].astype(str) + "|" +
    df_new_events["event_time_utc"].astype(str) + "|" +
    hash_of_raw_event(df_new_events["raw_event"])
)

# Append first, then dedup by deterministic key while preserving earliest ingestion record
df_events_master = concat_rows(df_events_master_existing, df_new_events)
df_events_master = stable_drop_duplicates(df_events_master, subset=["dedup_key"], keep="first")
```

### Pattern C: Query fallback sequence when results are empty

1. Verify time bounds and anchor timezone.
2. Pull a raw sample:
   ```spl
   search index=<idx> | head 20 | table _raw
   ```
3. Confirm field existence:
   ```spl
   ... | fieldsummary | search field=<suspected_field>
   ```
4. Try field variants (OR strategy):
   - `user OR username OR userName`
   - `process_id OR pid OR ProcessId`
   - `src_ip OR source_ip OR ip`
5. Broaden constraints: remove strict sourcetype or exact-match filters.
6. Re-run and append with a new `extraction_run_id`.

## Troubleshooting

### Problem: Backward side returns data but forward side is sparse

Likely causes:

- Detection near data ingestion delay boundary
- Overly strict forward filters
- Wrong sourcetype for post-event telemetry

Actions:

- Extend the forward window first.
- Remove nonessential forward constraints.
- Pivot by entity from backward findings to drive forward search.

### Problem: Both sides are empty

Likely causes:

- Incorrect anchor timestamp or timezone assumption
- Wrong index/source/table
- Field mismatch in filters

Actions:

- Validate anchor timezone and convert to UTC explicitly.
- Run a broad raw sample around the expected period.
- Replace strict field predicates with keyword search in raw text.

### Problem: Results exist but cannot align on one timeline

Likely causes:

- Multiple event time fields with mixed semantics
- Parsing to string type instead of datetime

Actions:

- Normalize `event_time_utc` in one standard timezone.
- Compute `seconds_from_anchor` from normalized epoch only.
- Store the original timestamp field name for provenance.

### Problem: Schema drift across runs or cases

Likely causes:

- New source versions introducing renamed fields
- Connector-specific parsing differences

Actions:

- Use additive schema evolution with `schema_version`.
- Keep legacy columns and map to canonical fields.
- Record the source-specific parse method in each row.

## Guardrails and Non-Negotiables

- Always extract in both directions unless explicitly out of scope.
- Always preserve append-only evidence history.
- Never silently mutate anchor meaning.
- Never discard provenance fields from extracted rows.
- Never conclude no-data until field-variation and raw inspection retries are exhausted.

## Success Criteria

A run is successful when:

- Both backward and forward evidence were extracted relative to the anchor.
- Event rows include direction and signed time delta.
- Evidence tables remain appendable and reusable.
- New runs improve coverage without losing prior context.
- Timeline and entity pivots are reproducible from stored dataframes.

## Default Run Output Contract

For each execution cycle, produce:

- Short status summary
- Event counts by direction (`backward`, `anchor`, `forward`)
- Net-new entities discovered
- Net-new event types
- Coverage gaps
- Recommended next windows and hypothesis pivots
