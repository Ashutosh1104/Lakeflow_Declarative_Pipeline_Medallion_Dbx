# Pipeline operations

Use this guide when investigating failed updates or changing raw-table
protections. A recovery command is specific to the error; there is no single
query that fixes every pipeline failure.

## Troubleshoot a failed update

1. Open the failed update's event log. Identify the error and the dataset that
   failed.
2. Apply a fix for that specific cause. If the cause is expected to recur,
   update the pipeline definition so future runs handle it automatically.
3. Run a full refresh for the affected streaming tables when the changed
   definition or source state must be rebuilt. A refresh without fixing the
   cause may reproduce the same failure.

For example, `DELTA_SOURCE_IGNORE_DELETE` is a source-change issue, not a date
parsing issue. Check the event log and source table history before choosing a
recovery.

## Date values rescued by Auto Loader

If the raw `admission_date` column is typed as `DATE` but an incoming file
contains dates such as `7/20/2024`, Auto Loader may leave the typed value null
and store its original text in `_rescued_data`. To recover those values, edit
the `daily_patients` query in the SQL notebook used by the pipeline. Replace
the existing `admission_date` expression in its `SELECT` with:

```sql
COALESCE(
  CAST(admission_date AS DATE),
  CAST(try_to_timestamp(
    get_json_object(_rescued_data, '$.admission_date'),
    'M/d/yyyy'
  ) AS DATE)
) AS admission_date,
```

This is a `SELECT` expression, not a standalone command. It keeps dates already
parsed by Auto Loader and falls back to parsing the original rescued text.
Invalid values remain null and are dropped by the Bronze `required_fields`
expectation.

Save the notebook and run **Full refresh selection**, including
`daily_patients` and `processed_patient_data`; Gold views recompute downstream.
If the vendor confirms that this date format is permanent, leave the parsing
logic in the pipeline. If formats vary regularly, plan a durable ingestion
schema change to read dates as strings and parse supported formats in Bronze.
Do not reset Auto Loader schemas or checkpoints without planning how existing
raw data will be retained.

## Keep raw tables append-only

Raw tables are intended to preserve ingestion history. Delta's append-only
property blocks changes to existing rows while allowing Auto Loader appends
and additive schema evolution:

```sql
ALTER TABLE dlt_dbx.default.raw_patients_daily
  SET TBLPROPERTIES ('delta.appendOnly' = 'true');
ALTER TABLE dlt_dbx.default.raw_diagnosis_map
  SET TBLPROPERTIES ('delta.appendOnly' = 'true');
```

Operations such as `DELETE`, `UPDATE`, and `MERGE` that modify existing rows
are rejected. Append-only does not prevent a table from being dropped and
recreated, and it does not undo changes made before it was enabled. Reapply
the property after a reset; use permissions as an additional way to restrict
other users' access.
