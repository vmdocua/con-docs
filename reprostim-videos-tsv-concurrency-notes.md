# Introduction

## System Setup

We have a distributed dataset system with multiple boxes and workflows.

### Box A — `reproiner`

Box A (`reproiner`) captures video from the experiment video projector.

It automatically collects the recorded videos and stores them in the local `reprostim-reproiner` annexed dataset. The dataset is synchronized daily to the other nodes.

### Box B — `typhon`

Box B (`typhon`) contains the `dbic-reproflow` superdataset, which includes `reprostim-reproiner`, `dbic-QA`, `reprostim-birch`, and other subdatasets.

It is used for processing and analysis of the collected data.

Box B performs daily processing of the videos captured by Box A.

## `videos.tsv` as a File-Based Database

The root of the `reprostim-reproiner` dataset contains a `videos.tsv` file that acts as a simple file-backed database table for all collected videos and their metadata.

Each row in `videos.tsv` corresponds to a single recorded video.

The `video-audit` command is used to create and update this media database.

`video-audit` has several processing modes:

* **internal** — extracts information directly from the video file, such as duration, resolution, codec, etc.
* **external tools**

  * **`qr`** — performs QR-code-based analysis and generates `qrinfo` files. This processing is relatively slow.
  * **`nosignal`** — detects no-signal segments in the video and records their percentage. This processing is also relatively slow.

There are several scripts that run daily and process videos using `video-audit`. These scripts can run in parallel, for example:

* processing new videos from the previous day;
* recovering videos for which external processing previously failed;
* reprocessing failed external-tool operations.

The scripts can also be started manually.

In addition, processing is parallelized internally: a single script can spawn multiple `video-audit` processes that operate simultaneously.

## The Concurrency Problem

The main problem is how to keep `videos.tsv` consistent and correct when multiple processes are running in parallel and updating the same file.

The problem is similar to concurrent access to a database table:

* multiple processes may read the same data;
* multiple processes may modify different rows at the same time;
* multiple processes may modify the same row;
* a process may read an old version of the file and later overwrite changes made by another process;
* long-running processing should not unnecessarily block unrelated operations.

The challenge is therefore to provide some DB-like concurrency control while keeping `videos.tsv` as a plain file.

# RDBMS-Inspired Notes

When robustness, consistency, and concurrent access are important, an RDBMS provides well-established mechanisms for handling these problems.

In our case, however, we do not have an RDBMS. The BIDS/ReproStim metadata is stored in TSV files, so we need to implement some of the required concurrency mechanisms at the filesystem/application level.

In simplified terms, an RDBMS provides two important mechanisms:

* **locking/concurrency control** — prevents incompatible operations from interfering with each other;
* **transactions** — provide atomicity and consistency for groups of related changes.

At the database level, locks can exist at different granularities, depending on the DBMS and operation:

* database-level;
* metadata-level;
* table-level;
* page/block-level;
* row-level;
* sometimes finer-grained locking.

Transactions provide another layer of control. They define a unit of work and specify how changes become visible and what happens when an operation fails.

Typical transaction concepts include:

* transaction boundaries;
* commit and rollback;
* transaction isolation;
* atomic updates;
* distributed transactions when multiple databases or heterogeneous systems are involved.

Typical SQL isolation levels are:

* `READ UNCOMMITTED`
* `READ COMMITTED`
* `REPEATABLE READ`
* `SERIALIZABLE`

We do not need to reproduce the full RDBMS model for `videos.tsv`, but these concepts provide a useful way to think about the problem.

# File-Level Locking

For `videos.tsv`, we can use filesystem locking to prevent conflicting operations.

On Unix systems, `flock` provides advisory file locking. We use a dedicated lock file, for example:

```text
videos.tsv.lock
```

In Python, we use the platform-independent [`filelock`](https://pypi.org/project/filelock/) package and its `FileLock` mechanism.

The important property is that cooperating processes use the same lock, so they can coordinate access to `videos.tsv`.

## Exclusive Access

A global `videos.tsv` lock can be used for operations that need exclusive access to the entire table.

For example, a shell script can use `flock` to ensure that only one instance of a particular workflow is running at a time.

This is useful when the whole workflow should be serialized.

However, this can be too restrictive because different `video-audit` processes may be able to work on different videos concurrently.

# `video-audit` and the TSV Table Lock

There is another level of concurrency control inside `video-audit`.

Multiple `video-audit` processes can run in parallel and eventually need to update the same `videos.tsv` file.

The update operation can therefore be treated similarly to a database table operation:

1. acquire the TSV lock;
2. load the current version of `videos.tsv`;
3. identify the rows that need to be updated;
4. apply only the changes produced by this process;
5. write the updated table;
6. release the lock.

The important point is that the process should **not** keep an old copy of the table and write that copy back later.

Instead, when saving, it should load the latest version of `videos.tsv` again, because another process may have modified it since the initial read.

Conceptually:

```text
Process A                         Process B

read videos.tsv
process video A
                                  read videos.tsv
                                  process video B

acquire videos.tsv.lock
reload videos.tsv
merge changes for video A
write videos.tsv
release lock

                                  acquire videos.tsv.lock
                                  reload videos.tsv
                                  merge changes for video B
                                  write videos.tsv
                                  release lock
```

This prevents Process B from accidentally overwriting the changes made by Process A.

# Per-Video / Per-Operation Locks

For external processing modes such as `qr` and `nosignal`, there is another concurrency problem.

These operations can be relatively slow. Multiple jobs may independently select the same video for processing, for example when jobs are distributed using a round-robin or retry mechanism.

A global `videos.tsv` lock does not solve this problem efficiently.

Instead, we can use a separate lock associated with the specific video and processing operation.

For example:

```text
<video-path>.qr.lock
<video-path>.nosignal.lock
```

The lock therefore identifies both:

* the video;
* the processing operation.

This allows, for example:

```text
video-A + qr       -> locked
video-A + nosignal -> can run
video-B + qr       -> can run
```

while preventing two `qr` processes from processing the same video simultaneously.

This is conceptually similar to a more fine-grained database lock, such as a row-level lock.

It is not literally a database row lock, but the analogy is useful:

```text
videos.tsv
------------------------------------------------
video-A    ...    qr result
video-B    ...    qr result
video-C    ...    qr result
------------------------------------------------

video-A.qr.lock       -> protects QR processing of video-A
video-B.qr.lock       -> protects QR processing of video-B
```

# Separation of Processing and Metadata Updates

An important distinction is between:

1. **long-running video processing**, and
2. **short metadata update operations**.

For example, QR analysis may take a significant amount of time.

We do not want to hold the global `videos.tsv` lock for the entire duration of QR processing.

Instead:

```text
per-video lock
    |
    +-- long-running QR processing
    |
    +-- produce result
          |
          v
global videos.tsv lock
          |
          +-- reload current TSV
          +-- update relevant row/column
          +-- save TSV
          |
          v
       unlock
```

This keeps the expensive processing parallel while making the final metadata update short and serialized.

# Dirty Reads

We can also allow a "dirty" read mode where a process reads `videos.tsv` without acquiring the exclusive lock.

This can be useful for inspection, debugging, or non-critical reporting when the caller accepts that the data may change while it is being read.

Such access should be considered **best effort** and should not be used when a consistent snapshot is required.

# Stale Data and Lost Updates

The main failure scenario to avoid is the classic read-modify-write race.

For example:

```text
Initial:
video-A | status=old
video-B | status=old
```

Process A reads the file:

```text
A sees:
video-A | old
video-B | old
```

Process B reads the same file:

```text
B sees:
video-A | old
video-B | old
```

A changes `video-A` and writes the complete file:

```text
video-A | new-A
video-B | old
```

B changes `video-B` using its stale copy and writes the complete file:

```text
video-A | old       <-- A's update is lost
video-B | new-B
```

The solution is to acquire the global TSV lock and reload the latest file before writing:

```text
lock
  |
  +-- reload latest videos.tsv
  +-- apply only our changes
  +-- write complete file
  |
unlock
```

This provides a simple form of serialized read-modify-write behavior.

# Lock Recovery

We should also consider what happens when a process is killed or hangs during processing.

In particular:

* a process may be terminated;
* a process may crash;
* an external tool may hang;
* a machine may reboot;
* a lock may appear to remain after an abnormal termination.

The recovery strategy depends on the locking mechanism.

Before manually removing lock files, all related processing should be stopped or verified to be no longer running.

After confirming that no process is using the lock, stale lock files can be removed manually if necessary.

This should be treated as an administrative recovery procedure rather than a normal part of the workflow.

# Summary

The concurrency model for `videos.tsv` can therefore be viewed as a simplified, file-based version of database concurrency control:

| Level                   | Mechanism                           | Purpose                                        |
| ----------------------- | ----------------------------------- | ---------------------------------------------- |
| Workflow                | `flock` / global workflow lock      | Prevent duplicate workflow execution           |
| TSV table               | `videos.tsv.lock`                   | Serialize read-modify-write operations         |
| Video + operation       | `<video>.<operation>.lock`          | Prevent duplicate processing of the same video |
| Transaction-like update | Reload → modify → save              | Prevent lost updates                           |
| Dirty read              | Read without exclusive lock         | Allow non-critical inspection                  |
| Recovery                | Stop processes + remove stale locks | Recover from abnormal termination              |

The goal is not to turn TSV files into a full database, but to introduce the minimum concurrency-control mechanisms needed to make parallel ReproStim processing safe and predictable.
