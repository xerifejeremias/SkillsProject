# Orchestration and operations checklist

How a pipeline is run matters as much as what it computes. A job that silently doesn't run or doesn't report its failure leaves stale outputs that look current. This applies to cron, Windows Task Scheduler, Airflow, Prefect, Dagster, Snowflake tasks and scheduled notebooks alike.

## Failures that nobody hears about
- **Alerting only around the main step:** an alert or status write that only covers the pipeline call itself misses failures before it (config errors, missing credentials, an import error) and after it (clean-up, copying outputs, writing the status file). Check every exit path of the job entry point. Does each one leave a record and send an alert?
- **Swallowed failures:** `except Exception: log` in a job runner, or an exit code of 0 after a failed step, so the scheduler reports success.
- **Status written last:** a "last run" or status file written at the end of the job is never updated when the job crashes, so a monitor reading it shows the previous successful run.
- **Alert channels that fail quietly:** an e-mail or webhook failure that is only logged is acceptable if the log file is always written and someone reads it. Check that the local record doesn't depend on the network call succeeding.

## Jobs that silently don't run
- **Scheduler defaults:**
  - **Windows Task Scheduler:** tasks created with `schtasks /Create` don't run missed occurrences by default (StartWhenAvailable off). By default they also don't start on battery power and stop when the laptop is unplugged. Changing these settings needs an XML task definition or PowerShell `Register-ScheduledTask`.
  - **cron:** missed runs are simply skipped while the machine is off.
  - **Airflow:** `catchup` and `start_date` decide whether missed intervals are backfilled.
  - **Snowflake:** tasks are created suspended.
  Check that the code's behavior matches what the docs and config comments promise.
- **Orphaned schedules:** removing a job from the config doesn't remove an already-installed scheduled task, cron entry or DAG. If the install or remove command only acts on jobs currently in the config, deleted jobs keep running.
- **Overlapping runs:** a long job still running when the next one starts, with both writing the same outputs. Is there a lock, or `max_active_runs=1`?
- **Environment differences:** the scheduler runs with a different working directory, user, PATH, environment variables (missing secrets) or timezone than the developer's shell. Relative paths and environment variables are the usual culprits.

## Clean-up and retention
- **Clean-up can crash the job:** deleting or moving files can raise (Windows file locks when a CSV is open in Excel, permissions, network drives). If clean-up runs before the status file or alert, a clean-up error hides the real job outcome.
- **What counts toward "keep N":** a retention rule that counts every folder (failed runs, manual reruns, several runs for the same data cutoff) keeps less useful history than it promises. Repeated failures can push out all the good runs. Check that the history the reports depend on (past forecasts for live accuracy, the backtests a selection references) is protected, or counted by what it represents.
- **Deleting what is still referenced:** a model registry, selection or report that points to a file the clean-up can delete.

## Consistency within a run
- **Reading a live source twice:** two steps of the same run (forecast and report, train and evaluate) each re-query a database or re-read a file that can change in between. They then see different data: new rows, a later cutoff, or a mid-load table. Read once per run and pass the data along, or snapshot it.
- **Mixed outputs:** a partial failure that leaves some outputs from the new run and some from the previous one, presented together as if they came from one run.

## Secrets and run records
- **Secrets in run records:** dumping the whole config or settings object (`dataclasses.asdict(cfg)`, `vars(args)`, the copied config files) into manifests, logs or run folders includes any password, token or connection string in it. Masking one field elsewhere doesn't help.
- **Incomplete masking:** masking only `user:password@` in URLs misses credentials in query strings (`?password=`, `odbc_connect=...PWD=...`, `token=`), passwords containing unencoded special characters, and secrets in exception messages that get logged or put into alerts.
- **Secrets in alerts:** an alert body that includes an error message or a config dump can send credentials to e-mail or chat.
