# Quality Report — Cleaning & Expectations

This document summarizes the new cleaning rules and expectations added by the Cleaning & Quality Owner.

## New Cleaning Rules (summary)
- doc_id alias mapping: common noisy doc_id values (e.g. `refund_policy_v4`, `policy_refund`) are mapped to canonical `policy_refund_v4`. Unknown post-map doc_ids are quarantined with reason `unknown_doc_id_after_map`.
- exported_at validation: only ISO date/time or `DD/MM/YYYY` accepted; malformed `exported_at` causes quarantine with `invalid_exported_at`.
- chunk_text cleaning: strip simple HTML tags, remove control/non-printable characters, collapse whitespace. Any chunk containing explicit markers like `ERROR`/`Traceback` is quarantined with `contains_error_marker`.

These rules are implemented in `transform/cleaning_rules.py` and produce both cleaned rows and quarantine rows consistently (quarantine rows include a `reason` field).

## New Expectations (summary)
- `no_html_tags_remaining` (severity: HALT): pipeline fails early if cleaned output still contains raw HTML markers.
- `exported_at_iso_or_empty` (severity: WARN): alerts when `exported_at` fields are malformed in cleaned output.

Expectations implemented in `quality/expectations.py`. HALT expectations will stop the pipeline (exit code 2) unless `--skip-validate` is used.

## How to reproduce / test impact
1. Run baseline pipeline:

```bash
python etl_pipeline.py run --run-id demo-clean-test
```

2. Inject a noisy row into `data/raw/policy_export_dirty.csv` (e.g. `doc_id=refund_policy_v4`, `exported_at=10/04/2026`, or `chunk_text=<div>test</div>`), then run again and observe `artifacts/quarantine/quarantine_<run-id>.csv` and pipeline logs showing `quarantine_records`.

## Evidence locations
- Code changes: `transform/cleaning_rules.py`, `quality/expectations.py`
- Quarantine artifacts: `artifacts/quarantine/*.csv`
- Run logs & manifests: `artifacts/logs/`, `artifacts/manifests/`

## Recommendations
- Add a small unit test harness for `clean_rows()` that asserts expected quarantine reasons for injected corrupt rows.
- Consider adding schema validation (pydantic / Great Expectations) for stronger guarantees in downstream stages.
