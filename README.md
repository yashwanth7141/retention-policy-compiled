# Parquet Retention Agent

Parquet Retention Agent is a lightweight Python service that manages the retention and deletion of Parquet files based on configured retention dates.

The agent accepts retention information through CSV or Excel files, stores the retention metadata in a local SQLite database, and deletes eligible Parquet files when their retention date is reached.

---

## Features

- Import retention records from CSV files
- Import retention records from Excel (`.xlsx`) files
- Validate input files and required columns
- Validate individual input rows
- Prevent duplicate Parquet file paths
- Detect duplicate paths within the same input file
- Track import results:
  - Imported
  - Skipped
  - Failed
- Store retention metadata in SQLite
- Automatically archive successfully processed input files
- Delete Parquet files whose retention date has been reached
- Track deletion failures
- Track files that no longer exist
- Track execution history
- Provide database and execution statistics
- Prevent concurrent executions using a Linux file lock
- Support scheduled execution through Linux cron
- Provide application logging
- Support recovery from missed scheduled executions

---

## How It Works

The agent follows this workflow:

```text
CSV / Excel Input
       |
       v
Input Validation
       |
       v
Duplicate Detection
       |
       v
SQLite Database
       |
       v
Scheduled Retention Run
       |
       v
Find Eligible Files
       |
       v
Delete Parquet Files
       |
       v
Update Status / Execution History
```

---

A Parquet file becomes eligible for deletion when:

``retention_date <= current_date``

Only records with PENDING status are processed.

---

## Requirements

**Operating System**

Linux is required because the application uses fcntl for concurrent execution protection.

**Python**

Python 3.12 

**Python Packages**

The application uses following pip libraries:

* pandas
* SQLAlchemy
* openpyxl

---

**Installation**

Clone or copy the project to the target server.

Example:

```
cd /home/postgres
git clone  https://github.com/yashwanth7141/retention-policy-compiled.git
cd retention-policy-compiled
```

Create a Python virtual environment:

```
python3.12 -m venv ret_venv
```

Activate it:

```
source ret_venv/bin/activate
```

Install the required packages:

```
pip3.12 install -r requirements.txt
```

Verify the installation:

```
python3.12 main.pyc version
```

---

**Configuration**

The application uses its configuration file to define runtime locations such as:

* SQLite database location
* Lock file location
* Other application settings

Make sure the configured directories are writable by the user running the Retention Agent.

The application automatically creates required parent directories where applicable.

---

**Input File Format**

The Retention Agent accepts:

* CSV (.csv)
* Excel (.xlsx)

The minimum required columns are:

 ```
parquet_file_path
retention_date
```

Additional supported metadata columns include:

```
application_name
table_name
parquet_folder_name
```

**CSV Example**

```
application_name,table_name,parquet_folder_name,parquet_file_path,retention_date
APP1,TABLE1,FOLDER1,/data/parquet/file1.parquet,2026-12-01
APP2,TABLE2,FOLDER2,/data/parquet/file2.parquet,2027-01-01
```

**Retention Date Format**

The retention date must use:

``YYYY-MM-DD``

Example:

``2026-12-01``

Dates such as the following are invalid:

```
2026/12/01
12-01-2026
01/12/2026
```

---

## Importing Retention Records

Use:

```
python3.12 main.pyc import <file_path>
```

Example:

```
python3.12 main.pyc import tests/test_data/test_file.csv
```

Excel files are also supported:

```
python3.12 main.pyc import input/retention.xlsx
```

Import Result

The import command displays a summary similar to:

```
============================================================
Import Summary
==============

Input File     : retention.csv
Total Rows     : 100
Imported       : 98
Skipped        : 2
Failed         : 0
==================
```

---

**Duplicate Handling**

The same Parquet path cannot be imported more than once.

**Duplicate Already in Database**

If a Parquet path already exists in the database, the row is skipped.

Example log:

```
Skipping duplicate parquet path already present in database:
 /data/parquet/file1.parquet
```

**Duplicate Within the Same Input File**

If the same path occurs multiple times in one CSV/Excel file, only the first occurrence is imported.

Subsequent occurrences are skipped.

Example:

```
Skipping duplicate parquet path within input file:
 /data/parquet/file1.parquet
```

The remaining rows continue to be processed.

---

**Input File Archiving**

After a successful database commit, the input file is moved to the processed/ directory.

Example:

```
processed/retention_20260824_121417.csv
```

The original input file is archived only after the database transaction succeeds.

If archiving fails after a successful import, the import remains successful and the error is logged.

---

## Retention Processing

Run retention processing manually using:

```
python3.12 main.pyc run
```

The agent:

1. Retrieves all PENDING records.
2. Selects records where:
   ``retention_date <= today``
3. Checks whether the configured path exists.
4. Verifies that the path is a regular file.
5. Deletes the Parquet file.
6. Updates the database status.

Example:

```
============================================================
Retention Summary
=================

Eligible Files : 5
Deleted        : 4
Failed         : 0
File Not Found : 1
Skipped        : 0
==================
```

---

**Retention Statuses**

Retention records can have statuses such as:

```
PENDING
DELETED
FAILED
FILE_NOT_FOUND
SKIPPED
```

**PENDING**

The file is waiting for its retention date.

**DELETED**

The Parquet file was successfully deleted.

**FAILED**

The agent attempted to process the file but deletion failed.

For example:

* Permission denied
* Filesystem error
* Other deletion-related errors

**FILE_NOT_FOUND**

The configured path does not exist when retention processing runs.

---

**Handling Missed Retention Runs**

The retention process does not depend on processing a file on the exact retention date.

For example:

```
Retention Date: 2026-07-10Scheduled run:
2026-07-11 -> missed
2026-07-12 -> runs
```

The file is still eligible on July 12 because the condition is:

``retention_date <= current_date``

Therefore, a missed scheduled execution does not permanently skip a file.

---

## Statistics

Display database and latest execution statistics using:

```
python3.12 main.pyc stats
```

Example:

```
============================================================
Parquet Retention Agent Statistics
==================================

Database Summary
----------------

Total Records     : 100
Pending           : 90
Deleted           : 8
Failed            : 1
File Not Found    : 1

Latest Execution
----------------

Started At        : 2026-08-24 03:00:00
Completed At      : 2026-08-24 03:00:05
Processed         : 5
Deleted           : 4
Failed            : 0
File Not Found    : 1
```

---

**Concurrent Execution Protection**

The Retention Agent uses a Linux file lock to prevent multiple instances from running simultaneously.

The lock is acquired when a command requiring application execution starts.

If another instance is already running, the second instance exits with:

> Another instance of Retention Agent is already running.

This prevents situations such as two retention processes attempting to delete the same files simultaneously.

The lock is automatically released when the process finishes.

---

## Linux Cron Scheduling

Retention processing can be scheduled using Linux cron.

Example:

```
0 3 * * * /home/postgres/ret_venv/bin/python3.12 /home/postgres/parquet-retention-agent/main.pyc run >> /home/postgres/parquet-retention-agent/logs/cron.log 2>&1
```

This means:

```
0   3   *   *   *
|   |   |   |   |
|   |   |   |   +--- Day of week
|   |   |   +------- Month
|   |   +----------- Day of month
|   +--------------- Hour
+------------------- Minute
```

Therefore:

``Every day at 03:00 AM``

**Install the Cron Job**

Edit the crontab for the user running the agent:

```
crontab -e
```

Add:

```
0 3 * * * /home/postgres/ret_venv/bin/python3.12 /home/postgres/parquet-retention-agent/main.pyc run >> /home/postgres/parquet-retention-agent/logs/cron.log 2>&1
```

**View Cron Jobs**

```
crontab -l
```

**Remove or Modify the Schedule**

```
crontab -e
```

Then edit or remove the corresponding cron entry.

---

## Logging

Application logs contain information about:

* Import operations
* Duplicate records
* Validation failures
* Retention processing
* File deletion
* File-not-found conditions
* Permission failures
* Lock acquisition/release
* Database errors

Cron output can be redirected to:

```
logs/cron.log
```

---

## Database

The Retention Agent uses SQLite for local metadata storage.

The database stores retention information such as:

* Application name
* Table name
* Parquet folder name
* Parquet file path
* Retention date
* Status
* Retry count
* Error message
* Deletion timestamp

Execution history is also maintained to track retention runs.

The SQLite database is local to the Retention Agent installation.

---

**Directory Structure**

A typical project structure is:

```
parquet-retention-agent/
|
├── main.pyc
├── README.md
├── requirements.txt
|
├── config/
|   └── config.ini
|
├── data/
|   └── retention_agent.db
|
├── logs/
|
├── processed/
|
├── src/
|   ├── config.pyc
|   ├── constants.pyc
|   ├── enums.pyc
|   ├── logger.pyc
|   ├── lock.pyc
|   ├── statistics.pyc
|   ├── validator.pyc
|   |
|   ├── database/
|   ├── models/
|   ├── repository/
|   ├── services/
|   └── ...
|
```

Runtime directories such as data/, logs/, and processed/ should normally not be committed to source control.

---

## Production Deployment

Before deploying to a customer environment:

1. Install Python 3.12.
2. Create the virtual environment.
3. Install required Python packages.
4. Configure database and lock-file locations.
5. Verify filesystem permissions.
6. Import the customer's retention input file.
7. Verify imported records using:
   ``python3.12 main.pyc stats``
8. Perform a controlled retention run.
9. Configure the Linux cron job.
10. Verify cron execution and logs.

The Linux user running the agent must have sufficient permissions to delete the configured Parquet files.

---

## Troubleshooting

Retention Agent is already running

If you receive:

```
Retention Agent is already running.
```

another instance currently holds the application lock.

Check running processes before taking further action:

```
ps -ef | grep main.pyc
```

Do not manually remove the lock file while another instance is running.

---

**File Not Found**

If a retention record reports:

```
File Not Found
```

verify that the configured parquet_file_path exists and is accessible by the user running the Retention Agent.

---

**Permission Denied**

If deletion fails with:

```
PermissionError: [Errno 13] Permission denied
```

verify:

* File ownership
* Directory ownership
* Directory write permission
* User permissions
* NFS/filesystem permissions where applicable

---

**Invalid Retention Date**

Use:

``YYYY-MM-DD``

Example:

``2026-12-01``

---

**Missing Required Columns**

Ensure the input contains:

```
parquet_file_path
retention_date
```

---

**Command Reference**

| Command | Description |
|---|---|
| `python3.12 main.pyc version` | Display application version |
| `python3.12 main.pyc import <file>` | Import CSV/Excel retention records |
| `python3.12 main.pyc run` | Process eligible retention records |
| `python3.12 main.pyc stats` | Display database and execution statistics |
---

## Testing

The application has been tested for:

* Valid CSV import
* Excel import
* Duplicate records already in the database
* Duplicate records within the same input
* Re-importing the same file
* Mixed valid/invalid rows
* Invalid retention dates
* Missing required columns
* Input file archiving
* Lock acquisition and release
* Retention deletion
* File-not-found handling
* Directory paths
* Permission-denied deletion
* Database transaction rollback
* Concurrent execution protection
* Scheduler execution
* Logging and execution reporting
* End-to-end retention processing

---

Version

Parquet Retention Agent 1.0.0

---

License

Internal / Customer Deployment.

---
