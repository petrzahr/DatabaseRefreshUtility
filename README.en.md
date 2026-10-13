# Database Refresh Utility

[Česky](README.md) | **English**

This tool saves selected TEST database settings before a refresh and restores them afterwards. **The DBA performs the database refresh.**

Workflow: **Capture → DBA refresh → Preflight → Restore → Validate**.

## 1. Requirements

- Windows PowerShell 5.1, SQL*Plus, and Oracle 19c or later.
- A working SQL*Plus connection under the same Windows account that runs the tool.
- Permission to run PowerShell scripts under your server's policy. No extra modules or Python are needed.

## 2. Configure the connection

Copy `config/database.example.json` to `config/database.json`. Fill in the accounts and choose one connection method.

### Using a TNS alias

Your Oracle client must resolve `TESTDB`, for example through `tnsnames.ora` or `TNS_ADMIN`. **Omit `port` and `serviceName`.**

```json
{
  "host": "TESTDB",
  "sqlplusPath": "sqlplus.exe",
  "timeoutSeconds": 300,
  "schemaOrder": ["APP1"],
  "users": {
    "APP1": {"username": "APP1", "password": "REPLACE_PASSWORD"}
  }
}
```

### Using a server and service name

```json
{
  "host": "db-server.example.cz",
  "port": 1521,
  "serviceName": "testpdb.example.cz",
  "sqlplusPath": "sqlplus.exe",
  "schemaOrder": ["APP1"],
  "users": {
    "APP1": {"username": "APP1", "password": "REPLACE_PASSWORD"}
  }
}
```

`sqlplusPath` can be the full client executable path. Defaults are `sqlplus.exe`, port `1521`, and a `300` second timeout per SQL*Plus call. `host` also accepts `//server:1521/service` without separate port and service fields.

## 3. Configure operations for each schema

For account `APP1`, create `config/schemas/APP1.json`. The filename, entry in `schemaOrder`, and `username` must match. `schemaOrder` must list every configured schema exactly once, in restore order. Files named `*.example.json` are ignored.

A schema file contains a `steps` array. You can combine steps for different tables in one array. Keep passwords only in `database.json`.

### Selecting rows

| Field | Meaning |
| --- | --- |
| `type` | The operation to perform. |
| `table` | A table in the connected account's schema. |
| `key` | Columns used to select or identify rows. The primary key is never discovered automatically. |
| `match` | An array of conditions; each object contains exactly the columns listed in `key`. |
| `allRows` | Set to `true` to select all rows. |
| `columns` | Columns whose original values `restoreRows` restores. |
| `set` | Fixed values applied by `update`. |

Always use **either `match` or `"allRows": true`**. Conditions within one `match` object use AND; separate objects use OR. Selecting no rows is valid; an empty `match` array is an error.

**Uniqueness depends on the operation:**

- `restoreRows` and `insert`: each `key` combination must identify at most one selected row and cannot contain NULL. Duplicates outside `match` are allowed. Capture with `allRows` checks the entire table.
- `backupTable`, `update`, and `delete`: a condition may select multiple rows. `null` and `""` mean `IS NULL`.

Columns in `key` cannot appear in `columns` or `set`. Filters support equality, not custom SQL, LIKE, or ranges. Values are strings, numbers, or `null`; use `0`/`1` for flags, not `false`/`true`. Table and column names are converted to uppercase; quoted identifiers are unsupported.

### Operations at a glance

| Operation | Purpose | `key` with `match` | `key` with `allRows` |
| --- | --- | --- | --- |
| `backupTable` | Backup only; no automatic restore. | Required | Omit |
| `restoreRows` | Restore original values of the specified `columns`. | Required | Required |
| `replaceTable` | Replace the entire table contents. | Unsupported | Omit |
| `insert` | Add missing captured rows. | Required | Required |
| `update` | Apply `set` values to every selected row. | Required | Required |
| `delete` | Delete every selected row. | Required | Omit |

## 4. Configuration examples

Each example is a complete schema file. Replace table names, column names, and values with your own.

### backupTable — backup only

Creates CSV and INSERT SQL files. Preflight, Restore, and Validate skip this table.

**Entire table**

```json
{
  "steps": [
    {
      "type": "backupTable",
      "table": "AUDIT_LOG",
      "allRows": true
    }
  ]
}
```

**Selected groups of rows**

```json
{
  "steps": [
    {
      "type": "backupTable",
      "table": "CONFIG_TEST",
      "key": ["CONFIGTYPE"],
      "match": [
        {"CONFIGTYPE": "POA_CONFIG"},
        {"CONFIGTYPE": "BMI"}
      ]
    }
  ]
}
```

### restoreRows — original column values

Rows are selected during Capture. Restore uses `key` to restore each row’s own original values. New rows remain unchanged; missing rows are skipped with a warning.

**Selected rows**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"},
        {"CONFIG_KEY": "CALLBACK_URL"}
      ],
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

**All original rows**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "allRows": true,
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

**Composite key**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "TENANT_CONFIG",
      "key": ["TENANT_ID", "CONFIG_KEY"],
      "match": [
        {"TENANT_ID": 10, "CONFIG_KEY": "API_URL"},
        {"TENANT_ID": 20, "CONFIG_KEY": "API_URL"}
      ],
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

### replaceTable — entire contents

Restore first deletes all `replaceTable` tables in reverse configuration order, even if they are unrelated. It then inserts data in configuration order. List parents before children for related tables.

**One table**

```json
{
  "steps": [
    {
      "type": "replaceTable",
      "table": "LOOKUP",
      "allRows": true
    }
  ]
}
```

**Related tables**

```json
{
  "steps": [
    {
      "type": "replaceTable",
      "table": "USERS",
      "allRows": true
    },
    {
      "type": "replaceTable",
      "table": "USERGROUPS",
      "allRows": true
    }
  ]
}
```

### insert — add missing rows

Capture saves complete selected rows. Restore inserts missing rows and skips identical existing rows. An existing key with different values is a conflict; other rows remain unchanged.

**Selected rows**

```json
{
  "steps": [
    {
      "type": "insert",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"},
        {"CONFIG_KEY": "CALLBACK_URL"}
      ]
    }
  ]
}
```

**All original rows**

```json
{
  "steps": [
    {
      "type": "insert",
      "table": "TEST_USERS",
      "key": ["USER_ID"],
      "allRows": true
    }
  ]
}
```

### update — fixed values

Rows are selected after the refresh. All selected rows receive the same `set` values. One condition may select multiple rows.

**A specific condition**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"}
      ],
      "set": {"CONFIG_VALUE": "https://api-test.example.cz", "DESCRIPTION": "Test API"}
    }
  ]
}
```

**Multiple rows sharing a condition**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "INTEGRATIONS",
      "key": ["ENVIRONMENT"],
      "match": [
        {"ENVIRONMENT": "PROD"},
        {"ENVIRONMENT": "STAGING"}
      ],
      "set": {"ENABLED": 0}
    }
  ]
}
```

**All rows**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "NOTIFICATION_SETTINGS",
      "key": ["ID"],
      "allRows": true,
      "set": {"ENABLED": 0, "RECIPIENT_EMAIL": null}
    }
  ]
}
```

### delete — remove rows

Deletes rows after the refresh. Each run removes all rows currently matching the conditions. With `allRows`, it deletes the entire current contents, including rows added since the previous run.

**Selected groups of rows**

```json
{
  "steps": [
    {
      "type": "delete",
      "table": "TEMP_MESSAGES",
      "key": ["SOURCE", "STATUS"],
      "match": [
        {"SOURCE": "PROD", "STATUS": "PENDING"},
        {"SOURCE": "PROD", "STATUS": "ERROR"}
      ]
    }
  ]
}
```

**Entire table**

```json
{
  "steps": [
    {
      "type": "delete",
      "table": "TEMP_MESSAGES",
      "allRows": true
    }
  ]
}
```

## 5. Run Capture and Restore

**Stop application writes to the affected tables and keep them stopped until validation finishes.** Capture queries do not share one consistent database snapshot.

### Using the application window

From the project root, run `DatabaseRefreshUtility.bat`, or:

```powershell
powershell.exe -NoProfile -STA -File .\scripts\Start-DatabaseRefreshUtilityUI.ps1
```

1. Before the refresh, click **Capture** and wait for `CAPTURE SUCCESS`.
2. Keep the entire generated `snapshots/<time>/` directory. The UI automatically selects the new `snapshot.json`.
3. Have the DBA perform the refresh.
4. Run **Preflight**. If it succeeds, run **Restore**.
5. Restore repeats Preflight, restores the data, and validates the result. You can also run **Validate** separately.

Preflight only checks the database; it does not attempt writes. Some errors, such as CHECK constraint violations, may therefore appear only during Restore.

### Using scripts

```powershell
# Before the refresh:
powershell.exe -NoProfile -File .\scripts\Capture.ps1

# After the DBA refresh; replace <time> with the actual folder:
powershell.exe -NoProfile -File .\scripts\Preflight.ps1 -Snapshot ".\snapshots\<time>\snapshot.json"
powershell.exe -NoProfile -File .\scripts\Restore.ps1 -Snapshot ".\snapshots\<time>\snapshot.json"

# Optional repeated validation:
powershell.exe -NoProfile -File .\scripts\Validate.ps1 -Snapshot ".\snapshots\<time>\snapshot.json"
```

## 6. Output files and logs

| Location | Contents |
| --- | --- |
| `snapshots/<time>/snapshot.json` | Data for automatic restore. Keep the entire snapshot directory. |
| `snapshots/<time>/<SCHEMA>/` | CSV and manual INSERT SQL for each table. |
| `logs/<time>-<id>-<action>.log` | Automatic run log, for both UI and scripts. |
| `restore-plan.json` next to the snapshot | Retry plan; do not delete or edit it. |
| `recovery/` next to the snapshot | Skipped `restoreRows` rows and, when available, SQL for adding them manually. |

Logging starts before configuration is loaded and continues through the final result. Logs contain UTC timestamps, steps, row counts, SQL*Plus calls, warnings, errors, transaction confirmations, and a final `SUCCESS`/`FAILED`. The path is displayed as `LOG FILE`. Logs are written as the run progresses; they are not overwritten or automatically deleted. Forced termination may leave no final entry. If a log cannot be created, database work does not start.

Passwords, row values, and complete SQL statements are excluded from operational logs. Backups and recovery files do contain database data; protect them along with the configuration. **Save log** in the UI additionally saves the visible window output.

`backupTable` and `insert` export only the selected rows. Other operations back up the entire table during Capture. CSV preserves NULL using `__IS_NULL` columns. The snapshot and exports are checked using checksums before Capture reports success.

Manual SQL exports are not executed automatically and contain no COMMIT. Before using them, check target data, structure, and relationships; regular exports assume the inserted rows are not already present. Recovery SQL inserts only keys that are still missing. Identity columns may require preparing INSERT statements manually. A recovery-file generation error does not undo a committed restore.

## 7. Restore behavior

- Each schema has its own transaction and COMMIT. An error rolls back the current schema; earlier committed schemas remain committed. A failure in the separate post-restore validation cannot undo a COMMIT.
- Tables are locked during restore. A busy lock causes an error; TRUNCATE is never used.
- After fixing a failure, retry Restore with the same snapshot and plan. If the connection is lost during COMMIT, verify the outcome with Validate. Do not run concurrent restores of the same snapshot.
- UPDATE saves selected `key` values and the row count of each group. Retries require the same counts and validate all assigned values. Replacing rows with others having the same key values and count cannot be detected. Standalone UPDATE validation requires the saved plan.
- `backupTable`, `restoreRows`, `replaceTable`, and `insert` must be the only step for their table. Multiple UPDATE/DELETE steps are allowed, but UPDATE steps must use the same `key` list and must not overlap. DELETE must not remove rows needed to validate UPDATE.
- The DBA must handle enabled triggers. The tool does not disable triggers or constraints. For `replaceTable`, dependent child tables must also be replaced and listed after their parent; cycles and self-references are rejected.
- The environment must have no incoming foreign keys between schemas. The account may not see every such reference. Checks use `ALL_CONSTRAINTS` and `USER_CONSTRAINTS`; `SYS.DBA_CONSTRAINTS` access is not required.
- The connection defines the target. The tool checks the account but does not independently verify database identity. It cannot detect a target change hidden behind an unchanged TNS alias.

## 8. Data types and troubleshooting

Supported types: `CHAR`, `VARCHAR2`, `NCHAR`, `NVARCHAR2`, `NUMBER`, `DATE`, `TIMESTAMP`, and `TIMESTAMP WITH TIME ZONE`. Types such as `CLOB`, `BLOB`, `RAW`, and `TIMESTAMP WITH LOCAL TIME ZONE`, as well as oversized exported rows, are unsupported. A column outside `columns` can still block Capture because backups contain complete rows.

`insert` and `replaceTable` do not support identity or invisible columns. Virtual columns are not inserted. UPDATE cannot modify identity or virtual columns.

Use `2026-09-25 14:30:00` for dates, `2026-09-25 14:30:00.123456789` for timestamps, and append ` +02:00` for a time zone. The offset is preserved, not the region name. Write large exact numbers as JSON strings to avoid loss of precision. Configuration and exports use UTF-8.

| Problem | What to check |
| --- | --- |
| `ORA-12154` | The TNS alias must work with the client specified by `sqlplusPath`; check `tnsnames.ora` and `TNS_ADMIN`. |
| `selected keys must be unique and non-null` | For `restoreRows`/`insert`, check duplicates and NULL in the selected keys. With `allRows`, check the entire table. |
| `UPDATE group count changed` | A group's row count changed after the plan was created. Investigate the data and concurrent writes; do not edit the plan manually. |
| `layout changed` | The table structure after refresh differs from the snapshot. |
| A row is missing during `restoreRows` | Restore skips it; see `recovery/` for details. |

## 9. Older configurations and tests

The current snapshot format is **version 5**. Versions 1–4 are rejected. When upgrading, create a new Capture **before the refresh**. If the refresh has already happened, complete the old backup using the original tool version; do not manually convert the snapshot. Each new refresh requires a new Capture. Changing the connection or steps also requires a new Capture; changing a password does not.

Replace old `keyValues` with an array of `match` objects. Remove `maxRows`, `backupMaxRows`, `maxInsertRows`, `maxDeleteRows`, and `expectedRows`. Add `allRows: true` for whole tables. The old `credentials.json` is no longer read: accounts belong in `database.json`, and `tnsAlias` is now called `host`. Do not configure `expectedTarget`, `dbUniqueName`, or `conName`.

See [tests/README.md](tests/README.md) (Czech) for test instructions and Oracle test schema requirements.

```powershell
powershell.exe -NoProfile -File .\tests\Test-DatabaseRefreshUtility.ps1
powershell.exe -NoProfile -STA -File .\tests\Test-DatabaseRefreshUtilityUI.ps1
```
