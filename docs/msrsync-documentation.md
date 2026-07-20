# msrsync — Multi-Stream rsync

> 将 rsync 并行化，最大化 TB 级数据传输的吞吐量

[English](#english) | [中文](#中文)

---

## English

### Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Quick Start](#quick-start)
4. [Options Reference](#options-reference)
5. [Architecture & How It Works](#architecture--how-it-works)
6. [Log Output & Audit](#log-output--audit)
7. [Safe --delete (Two-Phase)](#safe---delete-two-phase)
8. [Exit Codes](#exit-codes)
9. [Notes & Caveats](#notes--caveats)
10. [Makefile Targets](#makefile-targets)
11. [License](#license)

### Overview

**msrsync** parallelizes `rsync` to maximize throughput for large data transfers. It splits the file tree into buckets (of configurable size and file count), then runs multiple rsync processes concurrently — each handling its own bucket. No more single-threaded rsync bottleneck on multi-terabyte datasets.

Originally created by [Jean-Baptiste Denis](https://github.com/jbd/msrsync). This fork adds:

- **Enterprise-grade structured logging** with timestamped, level-tagged messages (`--log-file`)
- **Safe two-phase `--delete`** — all buckets transfer in parallel first (phase 1), then a single rsync process handles orphan removal using a merged manifest built during bucketing (phase 2). The manifest is read into memory and sorted for rsync efficiency before the delete pass. This eliminates the race condition where concurrent `--delete` workers would delete each other's files
- **Automatic failed-file and deleted-file tracking** — `_summarize_rsync_logs` scans per-bucket rsync logs after each phase, producing sampled summaries on the terminal and full detail files (`failed-files.txt`, `deleted-files.txt`) in the buckets directory (capped at 100,000 entries)
- **Python 2.6+ / 3.x cross-compatibility**, including macOS `fork` fix for Python 3.8+ `spawn` default
- **Improved progress reporting** with bandwidth, throughput, and queue-depth metrics
- **Case-insensitive size suffixes** — `-s 1k` and `-s 1K` are equivalent
- **Monitor worker crash protection** — `rsync_monitor_worker` catches unexpected exceptions and writes an error stats record to the queue, preventing the main process from deadlocking on `monitor_queue.get(timeout=300)`
- **Symlink-aware crawl optimization** — symbolic link directories are removed from `os.walk`'s `dirs` list in-place, avoiding unnecessary stat calls during directory traversal

### Requirements

| Component | Requirement |
|-----------|-------------|
| Python | 2.6+ or 3.x |
| rsync | **GNU rsync** (required — macOS `openrsync` lacks `--from0`) |
| scandir (optional) | `pip install scandir` for Python < 3.5 (improves crawl performance) |
| Root (optional) | Only for `--bench` / `--benchshm` (drops buffer caches via `/proc/sys/vm/drop_caches`) |

On macOS, install GNU rsync:
```bash
brew install rsync
```

### Quick Start

```bash
# Basic: 8 parallel rsync processes
./msrsync -p 8 /data/source/ /backup/dest/

# With progress display and custom rsync flags
./msrsync -p 16 -P --rsync "-avz --numeric-ids" /data/ /backup/

# With log file (recommended for production)
./msrsync -p 8 -P -L /var/log/msrsync.log --rsync "-a --numeric-ids" /data/ /backup/

# Mirror sync with safe --delete (two-phase)
./msrsync -p 8 --rsync "-a --delete-after" /data/ /backup/

# Dry-run: build buckets and plan, don't execute
./msrsync -p 4 -d --rsync "-a" /data/src1 /data/src2 /backup/

# Run self-tests
./msrsync --selftest

# Install system-wide
make install DESTDIR=/usr/local
```

### Options Reference

| Flag | Argument | Description | Default |
|------|----------|-------------|---------|
| `-p` | `--processes N` | Number of parallel rsync workers | `1` |
| `-f` | `--files N` | Maximum files per bucket | `1000` |
| `-s` | `--size N` | Maximum size per bucket (accepts K/M/G/T/P/E/Z/Y suffix, case-insensitive) | `1G` |
| `-b` | `--buckets DIR` | Directory to store bucket files | auto tempdir |
| `-k` | `--keep` | Keep bucket files after run | off |
| `-j` | `--show` | Print bucket directory path at start | off |
| `-P` | `--progress` | Show real-time progress line | off |
| | `--stats` | Print summary statistics at end | off |
| `-d` | `--dry-run` | Build buckets and plan; do not run rsync | off |
| `-v` | `--version` | Print version and exit | |
| `-L` | `--log-file PATH` | Write timestamped structured log to PATH | off |
| `-r` | `--rsync "OPTS"` | rsync options as a quoted string. **MUST be the last option before positional args.** | `-aS --numeric-ids` |
| `-t` | `--selftest` | Run embedded unit and functional tests | |
| `-e` | `--bench` | Run benchmarks (Linux, root recommended) | |
| `-g` | `--benchshm` | Run benchmarks in `/dev/shm` (or `$SHM`) | |

**Important:** `--rsync` options are always supplemented with `--from0 --files-from=... --quiet --verbose --stats --log-file=...`. Do not include these yourself — they are auto-added by msrsync. This means `--files-from` and `--from0` inside `--rsync` will conflict. Use msrsync's built-in bucketing instead.

**Bucket sizing strategies:**

- **Large files (videos, disk images):** Use large `--size` (e.g., `50G`) and small `--files` to avoid memory pressure.
- **Many small files (source code, logs):** Use moderate `--size` (e.g., `1G`) and larger `--files` (e.g., `10000`) for efficiency.
- **TB-scale small files:** Consider `--files 50000` and `--size 2G`. More processes help, but rsync's per-file overhead is the real bottleneck.

### Architecture & How It Works

```
Source Directory
      │
      ▼  crawl() — os.walk, yields (size, relative_path) tuples
      │
      ▼  buckets() — split into N buckets by size/files limit
      │
      ├── Bucket 1 ──► rsync --files-from=bucket1 ──┐
      ├── Bucket 2 ──► rsync --files-from=bucket2 ──┤──► Destination
      └── Bucket N ──► rsync --files-from=bucketN ──┘
      │
      ▼  (if --delete was specified)
 Phase 2: single rsync --delete using merged per-source manifest
```

**Process architecture (multi-process):**

```
┌─────────────┐
│  Main Proc   │  - Crawls source directory
│              │  - Builds buckets and per-source manifests
│              │  - Enqueues bucket jobs
└──────┬───────┘
       │
       ├──► messages_worker    (consumes G_MESSAGES_QUEUE → terminal + log file)
       ├──► monitor_worker     (consumes monitor_queue → progress + stats)
       ├──► rsync_worker 1..N  (consumes jobs_queue → runs rsync, reports to monitor)
```

Key design decisions:

1. **Three dedicated processes** keep I/O concerns separated: message formatting, progress monitoring, and rsync execution never block each other.
2. **SyncManager Queues** (multiprocessing) are used instead of threads to avoid GIL contention.
3. **`fork` on macOS** is forced because Python 3.8+ defaults to `spawn`, which would not inherit `G_MESSAGES_QUEUE`.
4. **`--from0` (null-delimited paths)** is used in all rsync invocations for robust handling of filenames with spaces and special characters.
5. **Two-phase `--delete`** (see below) prevents concurrent rsync processes from deleting files that other processes are simultaneously transferring.

### Log Output & Audit

msrsync provides multiple layers of logging, all designed for enterprise production environments with zero performance impact on data transfer.

#### 1. msrsync Structured Log (`--log-file`)

When `-L /path/to/log` is specified, all INFO and ERROR messages are written with ISO 8601 timestamps:

```
[2026-07-20T17:30:05] [INFO] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] [INFO] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
[2026-07-20T17:30:05] [INFO] delete mode: phase-1 transfers all data, phase-2 removes orphans
[2026-07-20T17:30:17] [INFO] crawl complete: 152340 entries, 856.3 G total, 87 buckets
[2026-07-20T17:45:32] [INFO] all rsync workers finished
[2026-07-20T17:45:32] [INFO] === FAILED FILE TRANSFERS: 3 errors ===
[2026-07-20T17:45:32] [INFO]   rsync: send_files failed to open "/data/src/broken_file.bin": Permission denied (13)
[2026-07-20T17:45:32] [INFO]   rsync: send_files failed to open "/data/src/corrupt.db": Input/output error (5)
[2026-07-20T17:45:32] [INFO]   Full list: /tmp/msrsync-abc123/failed-files.txt (3 entries)
[2026-07-20T17:45:40] [INFO] delete pass complete: 0 errors
[2026-07-20T17:45:40] [INFO] === DELETED FILES: 142 files ===
[2026-07-20T17:45:40] [INFO]   -- first 20 --
[2026-07-20T17:45:40] [INFO]   old_backup/2025/
[2026-07-20T17:45:40] [INFO]   temp/cache_001.dat
[2026-07-20T17:45:40] [INFO]   ... (20 samples shown)
[2026-07-20T17:45:40] [INFO]   -- last 20 --
[2026-07-20T17:45:40] [INFO]   ... (20 samples shown)
[2026-07-20T17:45:40] [INFO]   Full list: /tmp/msrsync-abc123/deleted-files.txt (142 entries)
```

**Log format:** `[ISO8601-timestamp] [LEVEL] message`

**Levels used:**
- `INFO` — Phase transitions, completion counts, summaries
- `ERROR` — Individual rsync failures, OS errors during crawl

**Performance characteristics:**
- Log writes happen in the dedicated `messages_worker` process (separate from rsync workers)
- Log file is opened with line buffering (`open(path, 'a', 1)`) — no performance impact on transfers
- Progress lines (`MSG_PROGRESS`) are **not** written to the log file to avoid noise
- Timestamps reflect event occurrence time, not print time

#### 2. Failed Files & Deleted Files Tracking

After the sync completes, msrsync **automatically scans all per-bucket rsync log files** to extract:

- **Failed transfers** — lines containing `rsync error`, `rsync: `, `failed:`, or `permission denied`
- **Deleted files** — lines containing `*deleting` (rsync's delete marker)

Results are presented as:

1. **Terminal/log summary** — first 20 and last 20 entries, or all if ≤40
2. **Detail file** — full list at `<buckets_dir>/failed-files.txt` and `<buckets_dir>/deleted-files.txt`

**Detail files are capped at 100,000 entries each** to prevent unbounded growth in TB-scale small-file scenarios. When the cap is hit, a `... truncated at 100000 entries (N total)` line is appended.

#### 3. Per-Bucket rsync Logs

Each rsync worker generates its own detailed log file controlled by rsync's `--log-file` and `--verbose --stats` options:

```
/tmp/msrsync-XXXXXX/0000/0000/tmpXXXXXX.log
/tmp/msrsync-XXXXXX/0000/0001/tmpXXXXXX.log
```

These contain rsync's native per-file transfer records. Use `-k` (keep) to preserve them, and `-j` (show) to print the buckets directory path.

#### 4. Progress Output (`-P`)

A live-updating line on the terminal:

```
[85230/152340 entries] [520.4 G/856.3 G transferred] [3520 entries/s] [3.1 G/s bw] [monq 12] [jq 5]
```

| Field | Meaning |
|-------|---------|
| `entries` | Files transferred / total |
| `transferred` | Data transferred / total |
| `entries/s` | File throughput |
| `bw` | Effective bandwidth |
| `monq` | Monitor queue depth (pending results to process) |
| `jq` | Remaining job buckets in queue |

Progress lines are NOT written to the log file.

#### 5. Stats Summary (`--stats`)

Printed at the end of the run:

```
Status: SUCCESS
Working directory: /home/user
Command line: ./msrsync -p 8 --stats src/ dest/
Total size: 856.3 G
Total entries: 152340
Buckets number: 87
Mean entries per bucket: 1751
Mean size per bucket: 9.8 G
Entries per second: 3520
Speed: 3.1 G/s
Rsync workers: 8
Total rsync's processes (87) cumulative runtime: 482.5s
Crawl time: 12.3s (1.3% of total runtime)
Total time: 965.2s
```

#### 6. Terminal Output

All non-progress messages are timestamped with `[ISO8601]` prefix and routed to stdout (INFO) or stderr (ERROR):

```
[2026-07-20T17:30:05] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
```

#### Log Output Summary Table

| Source | Content | Destination | Controlled By |
|--------|---------|-------------|---------------|
| msrsync structured log | Phase transitions, errors, summaries | `--log-file` path + terminal | `-L` |
| Failed files summary | Sampled + full detail file | Terminal/log + `failed-files.txt` | Automatic |
| Deleted files summary | Sampled + full detail file | Terminal/log + `deleted-files.txt` | Automatic (with `--delete`) |
| Per-bucket rsync log | Per-file transfer records | `<buckets_dir>/**/*.log` | Automatic (rsync built-in) |
| Progress line | Live throughput stats | Terminal only | `-P` |
| Stats summary | Cumulative statistics | Terminal + log file | `--stats` |

### Safe --delete (Two-Phase)

When `--delete` (or any variant: `--delete-after`, `--delete-before`, `--delete-during`, `--delete-delay`, `--delete-excluded`, `--del`) is passed inside `--rsync`, msrsync automatically intercepts it and performs a **two-phase** operation:

**Phase 1 — Parallel transfer (no delete):**
- The `--delete` flag is stripped from all bucket-level rsync commands
- All files are transferred in parallel by N workers
- During bucketing, a **manifest file** is built accumulating all source file paths

**Phase 2 — Single-process delete:**
- After all workers finish, msrsync sorts the manifest file
- A **single** rsync process is launched with `--delete` and `--files-from=<manifest>`
- This process identifies which files in the destination are not in the source and deletes them

**Why this matters:** Running `--delete` on concurrent rsync processes creates a race condition — one worker may delete files that another worker hasn't finished transferring yet. The two-phase approach eliminates this entirely.

**Performance note:** Phase 2 runs as a single rsync process. For very large directory trees (millions of files), this adds a second scan pass. Using `--files-from` with the pre-built manifest minimizes this cost since rsync only needs to stat each listed file rather than walk the entire tree.

### Exit Codes

| Code | Constant | Meaning |
|------|----------|---------|
| 0 | — | Success: all buckets transferred, no errors |
| 1 | — | Uncaught exception during run |
| 10 | `EPYTHON_VERSION` | Python < 2.6 |
| 11 | `EBUCKET_DIR_NOEXIST` | Specified bucket directory does not exist |
| 12 | `EBUCKET_DIR_PERMS` | Bucket directory is not writable |
| 13 | `EBUCKET_FILE_CREATE` | Cannot write bucket file |
| 14 | `EBIN_NOTFOUND` | rsync executable not found in `$PATH` |
| 15 | `ESRC_NOT_DIR` | Source is not a directory |
| 16 | `ESRC_NO_ACCESS` | No read/execute access to source directory |
| 17 | `EDEST_NO_ACCESS` | Destination directory not writable |
| 18 | `EDEST_NOT_DIR` | (reserved) |
| 19 | `ERSYNC_OPTIONS_CHECK` | rsync options validation failed |
| 20 | `ERSYNC_TOO_LONG` | rsync process timed out |
| 21 | `ERSYNC_JOB` | rsync process reported an error |
| 23 | `EDEST_IS_FILE` | Destination exists and is a regular file |
| 24 | `EDEST_CREATE` | Cannot create destination directory |
| 27 | `EMSRSYNC_INTERRUPTED` | Interrupted by Ctrl+C (SIGINT) |
| 28 | `EBUCKET_DIR_OSERROR` | OS error creating bucket temp directory |
| 97 | `EOPTION_PARSER` | Command-line parse error |

### Notes & Caveats

#### 1. GNU rsync Required
macOS ships with `openrsync`, which does **not** support `--from0` (null-delimited `--files-from`). msrsync relies on this for safe filename handling. Install GNU rsync:
```bash
brew install rsync
# Verify: /usr/local/bin/rsync --version | head -1
# Should show: "rsync  version 3.x.x ..."
```
Ensure `/usr/local/bin` appears before `/usr/bin` in `$PATH`.

#### 2. Local Directories Only
Source and destination must be local paths. Remote-shell syntax (`user@host:/path`) is **not supported** for msrsync's own checking logic. However, you can pass `-e ssh` inside `--rsync` to have rsync use SSH for actual transfers:
```bash
./msrsync -p 4 --rsync "-a -e ssh" /local/src/ user@remote:/backup/
```
In this case, source/destination pre-checks are performed locally — the remote path check is skipped and the remote rsync handles access validation.

#### 3. Source Must Be a Directory
Wildcards (`/data/*`) are not supported as source arguments. Use the parent directory and rely on rsync's native filtering instead:
```bash
# NOT supported:
./msrsync /data/* /backup/

# Use rsync filters inside --rsync instead:
./msrsync --rsync "-a --include='*.log' --exclude='*'" /data/ /backup/
```

#### 4. No Resume After Interruption
Interrupted runs must be restarted from scratch. To minimize re-work for very large transfers:
- Use smaller bucket sizes (`-s 500M -f 500`)
- Combine with `-k` to inspect per-bucket rsync logs after a failure
- The `-L` log file will show exactly which phase was running when interrupted

#### 5. --rsync Must Be Last
`--rsync` (or `-r`) must appear immediately before the source/destination arguments:
```bash
# Correct:
./msrsync -p 8 --rsync "-a --numeric-ids" src/ dest/

# Wrong — rsync options will be misinterpreted:
./msrsync --rsync "-a --numeric-ids" -p 8 src/ dest/
```

#### 6. --delete is Two-Phase
When `--delete` is detected in `--rsync`, msrsync uses a two-phase approach (see [Safe --delete](#safe---delete-two-phase)). Phase 2 runs as a single rsync process. For trees with millions of files, this adds a relatively small additional scan that benefits from the pre-built manifest.

#### 7. Bucket Directory Structure
Buckets are stored in a nested directory structure (`<buckets>/0000/0000/...`) to avoid filesystem performance degradation when handling thousands of bucket files in a single directory. This structure is created automatically and cleaned up unless `-k` is used.

#### 8. Python Version Compatibility
- Python 2.6, 2.7: Fully supported (required for RHEL 6 / CentOS 6)
- Python 3.3+: Fully supported
- Python 3.5+: `os.scandir` is used natively for faster directory traversal
- Python < 3.5: Install `scandir` via pip for the same optimization

#### 9. macOS Considerations
- Python 3.8+ on macOS defaults to `spawn` for multiprocessing. msrsync forces `fork` since `G_MESSAGES_QUEUE` must be inherited by workers.
- GNU rsync must be installed separately (see caveat #1).

#### 10. Memory Usage
Each rsync worker process has its own memory footprint (typically 10-50 MB for rsync itself). The main process holds the bucket queue in memory. With many buckets and many workers, the SyncManager queues also consume memory. For extreme cases (millions of files, hundreds of workers), monitor with `ps` or `top`.

#### 11. TB-Scale Small Files
When syncing directories with millions of tiny files:
- The crawl phase takes the most time (single-threaded `os.walk`)
- Use `--files 50000` or higher to reduce bucket count
- Failed/deleted detail files are capped at 100,000 entries each
- Per-bucket rsync logs grow proportionally — use `-k` only for debugging

### Makefile Targets

```
make test       Run embedded self-tests (same as ./msrsync --selftest)
make install    Install msrsync to $(DESTDIR)/bin (default: /usr/bin)
make lint       Run pylint static analysis
make cov        Generate coverage report (requires python-coverage)
make covhtml    Generate HTML coverage report
make bench      Run benchmarks (Linux only, root recommended for cache drops)
make benchshm   Run benchmarks in /dev/shm (Linux only, root recommended)
make clean      Remove .pyc files and __pycache__ directories
```

### License

GNU General Public License v3.0. See `LICENSE` file.

This project includes a copy of `options.py` from the [bup](https://github.com/bup/bup) project, which is BSD-licensed. Copyright 2010-2012 Avery Pennarun and contributors.

---

## 中文

### 目录

1. [概述](#概述)
2. [环境要求](#环境要求)
3. [快速开始](#快速开始)
4. [选项详解](#选项详解)
5. [架构与工作原理](#架构与工作原理)
6. [日志输出与审计](#日志输出与审计)
7. [安全的 --delete（两阶段删除）](#安全的---delete两阶段删除)
8. [退出码](#退出码)
9. [注意事项](#注意事项)
10. [Makefile 目标](#makefile-目标)
11. [许可证](#许可证)

### 概述

**msrsync** 通过并行化 `rsync` 来最大化大数据传输的吞吐量。它将文件树按可配置的大小和文件数拆分为多个桶（bucket），然后并发运行多个 rsync 进程——每个进程处理自己的桶。TB 级数据不再受单线程 rsync 的瓶颈限制。

原作者：[Jean-Baptiste Denis](https://github.com/jbd/msrsync)。本分支新增功能：

- **企业级结构化日志**，带时间戳和级别标签（`--log-file`）
- **安全的两阶段 `--delete`** — 所有桶先并行传输（阶段一），传输完成后由单个 rsync 进程使用分桶阶段构建的合并 manifest 文件统一删除孤儿文件（阶段二）。manifest 在执行 delete pass 前会被读入内存并排序以提升 rsync 效率。这消除了并发 `--delete` worker 互相删除对方文件的竞态条件
- **自动失败/删除文件追踪** — `_summarize_rsync_logs` 在每个阶段后扫描逐桶 rsync 日志，在终端输出采样摘要，并在桶目录中写入完整详情文件（`failed-files.txt`、`deleted-files.txt`，上限 10 万条）
- **Python 2.6+ / 3.x 跨版本兼容**，含 macOS Python 3.8+ 默认 `spawn` 的 `fork` 修复
- **改进的进度报告**，含带宽、吞吐量和队列深度指标
- **大小写不敏感的后缀** — `-s 1k` 与 `-s 1K` 等效
- **Monitor worker 崩溃保护** — `rsync_monitor_worker` 捕获意外异常并向队列写入错误统计记录，防止主进程在 `monitor_queue.get(timeout=300)` 上死锁
- **符号链接感知的遍历优化** — 符号链接目录被原地从 `os.walk` 的 `dirs` 列表中移除，避免目录遍历时的不必要 stat 调用

### 环境要求

| 组件 | 要求 |
|------|------|
| Python | 2.6+ 或 3.x |
| rsync | **GNU rsync**（必须 — macOS 自带的 `openrsync` 不支持 `--from0`） |
| scandir（可选） | Python < 3.5 时 `pip install scandir`（提升遍历性能） |
| Root（可选） | 仅 `--bench` / `--benchshm` 需要（通过 `/proc/sys/vm/drop_caches` 清除缓存） |

在 macOS 上安装 GNU rsync：
```bash
brew install rsync
```

### 快速开始

```bash
# 基本用法：8 个并行 rsync 进程
./msrsync -p 8 /data/source/ /backup/dest/

# 显示进度并自定义 rsync 参数
./msrsync -p 16 -P --rsync "-avz --numeric-ids" /data/ /backup/

# 带日志文件（生产环境推荐）
./msrsync -p 8 -P -L /var/log/msrsync.log --rsync "-a --numeric-ids" /data/ /backup/

# 镜像同步（安全的两阶段删除）
./msrsync -p 8 --rsync "-a --delete-after" /data/ /backup/

# 演练模式：仅分桶规划，不执行
./msrsync -p 4 -d --rsync "-a" /data/src1 /data/src2 /backup/

# 运行自测
./msrsync --selftest

# 系统级安装
make install DESTDIR=/usr/local
```

### 选项详解

| 短选项 | 长选项 | 说明 | 默认值 |
|--------|--------|------|--------|
| `-p` | `--processes N` | 并行 rsync 进程数 | `1` |
| `-f` | `--files N` | 每个桶最多文件数 | `1000` |
| `-s` | `--size N` | 每个桶最大大小（支持 K/M/G/T/P/E/Z/Y 后缀，大小写不敏感） | `1G` |
| `-b` | `--buckets DIR` | 桶文件存放目录 | 自动临时目录 |
| `-k` | `--keep` | 运行后保留桶文件 | 关闭 |
| `-j` | `--show` | 启动时打印桶目录路径 | 关闭 |
| `-P` | `--progress` | 显示实时进度 | 关闭 |
| | `--stats` | 运行结束后打印统计汇总 | 关闭 |
| `-d` | `--dry-run` | 仅分桶和规划，不执行 rsync | 关闭 |
| `-v` | `--version` | 打印版本并退出 | |
| `-L` | `--log-file PATH` | 将带时间戳的结构化日志写入 PATH | 关闭 |
| `-r` | `--rsync "OPTS"` | rsync 选项（引号包裹的字符串）。**必须作为最后一个选项，放在位置参数之前。** | `-aS --numeric-ids` |
| `-t` | `--selftest` | 运行内置单元测试和功能测试 | |
| `-e` | `--bench` | 运行基准测试（Linux，推荐 root） | |
| `-g` | `--benchshm` | 在 `/dev/shm`（或 `$SHM`）中运行基准测试 | |

**重要：** `--rsync` 的选项始终会被追加 `--from0 --files-from=... --quiet --verbose --stats --log-file=...`。不要自己传入这些选项——msrsync 会自动添加。这意味着在 `--rsync` 中再传入 `--files-from` 或 `--from0` 会产生冲突。请使用 msrsync 内置的分桶机制。

**桶大小策略：**

- **大文件（视频、磁盘镜像）：** 使用较大的 `--size`（如 `50G`）和较小的 `--files` 以避免内存压力。
- **大量小文件（源码、日志）：** 使用适中的 `--size`（如 `1G`）和较大的 `--files`（如 `10000`）以提高效率。
- **TB 级小文件：** 考虑 `--files 50000` 和 `--size 2G`。更多进程有帮助，但 rsync 的逐文件开销才是真正的瓶颈。

### 架构与工作原理

```
源目录
      │
      ▼  crawl() — os.walk，生成 (大小, 相对路径) 元组
      │
      ▼  buckets() — 按大小/文件数限制拆分为 N 个桶
      │
      ├── 桶1 ──► rsync --files-from=桶1 ──┐
      ├── 桶2 ──► rsync --files-from=桶2 ──┤──► 目标目录
      └── 桶N ──► rsync --files-from=桶N ──┘
      │
      ▼  (如果指定了 --delete)
 阶段二：单进程 rsync --delete，使用合并的逐源 manifest
```

**进程架构（多进程）：**

```
┌─────────────┐
│   主进程     │  - 遍历源目录
│              │  - 构建桶和逐源 manifest
│              │  - 将桶作业入队
└──────┬───────┘
       │
       ├──► messages_worker     (消费 G_MESSAGES_QUEUE → 终端 + 日志文件)
       ├──► monitor_worker      (消费 monitor_queue → 进度 + 统计)
       ├──► rsync_worker 1..N   (消费 jobs_queue → 运行 rsync，向 monitor 报告)
```

关键设计决策：

1. **三个独立进程** 将 I/O 关注点分离：消息格式化、进度监控和 rsync 执行互不阻塞。
2. **SyncManager Queue**（multiprocessing）代替线程，避免 GIL 争用。
3. **macOS 上强制 `fork`**，因为 Python 3.8+ 默认使用 `spawn`，后者不会继承 `G_MESSAGES_QUEUE`。
4. **所有 rsync 调用使用 `--from0`（null 分隔路径）**，安全处理含空格和特殊字符的文件名。
5. **两阶段 `--delete`**（见下文）防止并发 rsync 进程删除其他进程正在传输的文件。

### 日志输出与审计

msrsync 提供多层日志，全部为企业生产环境设计，对数据传输零性能影响。

#### 1. msrsync 结构化日志（`--log-file`）

指定 `-L /path/to/log` 后，所有 INFO 和 ERROR 消息以 ISO 8601 时间戳写入：

```
[2026-07-20T17:30:05] [INFO] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] [INFO] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
[2026-07-20T17:30:05] [INFO] delete mode: phase-1 transfers all data, phase-2 removes orphans
[2026-07-20T17:30:17] [INFO] crawl complete: 152340 entries, 856.3 G total, 87 buckets
[2026-07-20T17:45:32] [INFO] all rsync workers finished
[2026-07-20T17:45:32] [INFO] === FAILED FILE TRANSFERS: 3 errors ===
[2026-07-20T17:45:32] [INFO]   rsync: send_files failed to open "/data/src/broken_file.bin": Permission denied (13)
[2026-07-20T17:45:32] [INFO]   Full list: /tmp/msrsync-abc123/failed-files.txt (3 entries)
[2026-07-20T17:45:40] [INFO] delete pass complete: 0 errors
[2026-07-20T17:45:40] [INFO] === DELETED FILES: 142 files ===
[2026-07-20T17:45:40] [INFO]   -- first 20 --
[2026-07-20T17:45:40] [INFO]   old_backup/2025/
[2026-07-20T17:45:40] [INFO]   ... (显示 20 条采样)
[2026-07-20T17:45:40] [INFO]   -- last 20 --
[2026-07-20T17:45:40] [INFO]   ... (显示 20 条采样)
[2026-07-20T17:45:40] [INFO]   Full list: /tmp/msrsync-abc123/deleted-files.txt (142 entries)
```

**日志格式：** `[ISO8601时间戳] [级别] 消息`

**使用的级别：**
- `INFO` — 阶段转换、完成计数、摘要
- `ERROR` — 单个 rsync 失败、遍历期间的 OS 错误

**性能特征：**
- 日志写入在独立的 `messages_worker` 进程中完成（与 rsync worker 分离）
- 日志文件以行缓冲打开（`open(path, 'a', 1)`）——对传输吞吐零影响
- 进度行（`MSG_PROGRESS`）**不会**写入日志文件，避免噪音
- 时间戳反映事件发生时间，而非打印时间

#### 2. 失败文件与删除文件追踪

同步完成后，msrsync **自动扫描所有逐桶 rsync 日志文件**，提取：

- **传输失败** — 包含 `rsync error`、`rsync: `、`failed:` 或 `permission denied` 的行
- **已删除文件** — 包含 `*deleting`（rsync 的删除标记）的行

结果以两种形式呈现：

1. **终端/日志摘要** — 前 20 条和后 20 条（如果总数 ≤40 则显示全部）
2. **详情文件** — 完整列表在 `<buckets_dir>/failed-files.txt` 和 `<buckets_dir>/deleted-files.txt`

**详情文件上限为每条 100,000 个条目**，防止在 TB 级小文件场景下无限增长。达到上限时，末尾追加 `... truncated at 100000 entries (N total)` 行。

#### 3. 逐桶 rsync 日志

每个 rsync worker 生成独立的详细日志文件，由 rsync 的 `--log-file` 和 `--verbose --stats` 控制：

```
/tmp/msrsync-XXXXXX/0000/0000/tmpXXXXXX.log
/tmp/msrsync-XXXXXX/0000/0001/tmpXXXXXX.log
```

包含 rsync 原生的逐文件传输记录。使用 `-k`（保留）来保留它们，使用 `-j`（显示）打印桶目录路径。

#### 4. 进度输出（`-P`）

终端上的实时更新行：

```
[85230/152340 entries] [520.4 G/856.3 G transferred] [3520 entries/s] [3.1 G/s bw] [monq 12] [jq 5]
```

| 字段 | 含义 |
|------|------|
| `entries` | 已传输 / 总文件数 |
| `transferred` | 已传输 / 总数据量 |
| `entries/s` | 文件处理速率 |
| `bw` | 有效带宽 |
| `monq` | monitor 队列深度（待处理的结果数） |
| `jq` | 队列中剩余作业桶数 |

进度行不写入日志文件。

#### 5. 统计汇总（`--stats`）

运行结束时打印：

```
Status: SUCCESS
Working directory: /home/user
Command line: ./msrsync -p 8 --stats src/ dest/
Total size: 856.3 G
Total entries: 152340
Buckets number: 87
Mean entries per bucket: 1751
Mean size per bucket: 9.8 G
Entries per second: 3520
Speed: 3.1 G/s
Rsync workers: 8
Total rsync's processes (87) cumulative runtime: 482.5s
Crawl time: 12.3s (1.3% of total runtime)
Total time: 965.2s
```

#### 6. 终端输出

所有非进度消息都带 `[ISO8601]` 前缀，路由到 stdout（INFO）或 stderr（ERROR）：

```
[2026-07-20T17:30:05] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
```

#### 日志输出汇总表

| 来源 | 内容 | 输出目标 | 控制方式 |
|------|------|----------|----------|
| msrsync 结构化日志 | 阶段转换、错误、摘要 | `--log-file` 路径 + 终端 | `-L` |
| 失败文件摘要 | 采样 + 完整详情文件 | 终端/日志 + `failed-files.txt` | 自动 |
| 删除文件摘要 | 采样 + 完整详情文件 | 终端/日志 + `deleted-files.txt` | 自动（使用 `--delete` 时） |
| 逐桶 rsync 日志 | 逐文件传输记录 | `<buckets_dir>/**/*.log` | 自动（rsync 内置） |
| 进度行 | 实时吞吐统计 | 仅终端 | `-P` |
| 统计汇总 | 累计统计数据 | 终端 + 日志文件 | `--stats` |

### 安全的 --delete（两阶段删除）

当在 `--rsync` 中传入 `--delete`（或其变体：`--delete-after`、`--delete-before`、`--delete-during`、`--delete-delay`、`--delete-excluded`、`--del`）时，msrsync 自动拦截并执行**两阶段**操作：

**阶段一 — 并行传输（不含删除）：**
- 从所有桶级 rsync 命令中剥离 `--delete` 标志
- 所有文件由 N 个 worker 并行传输
- 在分桶过程中，构建 **manifest 文件**，累积所有源文件路径

**阶段二 — 单进程删除：**
- 所有 worker 完成后，msrsync 排序 manifest 文件
- 启动**单个** rsync 进程，携带 `--delete` 和 `--files-from=<manifest>`
- 该进程识别目标中存在但源中不存在的文件并删除

**为什么重要：** 在并发的 rsync 进程上运行 `--delete` 会产生竞态条件——一个 worker 可能删除另一个 worker 尚未传输完成的文件。两阶段方法完全消除了此问题。

**性能说明：** 阶段二作为单个 rsync 进程运行。对于非常大的目录树（数百万文件），会增加一次额外的扫描。使用预构建的 manifest 配合 `--files-from` 可最小化此开销，因为 rsync 只需 stat 每个列出的文件，而无需遍历整个树。

### 退出码

| 码 | 常量 | 含义 |
|----|------|------|
| 0 | — | 成功：所有桶已传输，无错误 |
| 1 | — | 运行期间未捕获的异常 |
| 10 | `EPYTHON_VERSION` | Python < 2.6 |
| 11 | `EBUCKET_DIR_NOEXIST` | 指定的桶目录不存在 |
| 12 | `EBUCKET_DIR_PERMS` | 桶目录不可写 |
| 13 | `EBUCKET_FILE_CREATE` | 无法写入桶文件 |
| 14 | `EBIN_NOTFOUND` | 在 `$PATH` 中找不到 rsync |
| 15 | `ESRC_NOT_DIR` | 源不是目录 |
| 16 | `ESRC_NO_ACCESS` | 无源目录读/执行权限 |
| 17 | `EDEST_NO_ACCESS` | 目标目录不可写 |
| 18 | `EDEST_NOT_DIR` | （保留） |
| 19 | `ERSYNC_OPTIONS_CHECK` | rsync 选项校验失败 |
| 20 | `ERSYNC_TOO_LONG` | rsync 进程超时 |
| 21 | `ERSYNC_JOB` | rsync 进程报告错误 |
| 23 | `EDEST_IS_FILE` | 目标已存在且为普通文件 |
| 24 | `EDEST_CREATE` | 无法创建目标目录 |
| 27 | `EMSRSYNC_INTERRUPTED` | 被 Ctrl+C 中断（SIGINT） |
| 28 | `EBUCKET_DIR_OSERROR` | 创建桶临时目录时 OS 错误 |
| 97 | `EOPTION_PARSER` | 命令行解析错误 |

### 注意事项

#### 1. 需要 GNU rsync
macOS 自带的 `openrsync` **不**支持 `--from0`（null 分隔的 `--files-from`）。msrsync 依赖此功能安全处理文件名。安装 GNU rsync：
```bash
brew install rsync
# 验证：/usr/local/bin/rsync --version | head -1
# 应显示："rsync  version 3.x.x ..."
```
确保 `/usr/local/bin` 在 `$PATH` 中排在 `/usr/bin` 之前。

#### 2. 仅支持本地目录
源和目标必须是本地路径。不支持远程 shell 语法（`user@host:/path`）用于 msrsync 自身的检查逻辑。但可以在 `--rsync` 中传入 `-e ssh` 使 rsync 使用 SSH 进行实际传输：
```bash
./msrsync -p 4 --rsync "-a -e ssh" /local/src/ user@remote:/backup/
```
此时源/目标预检在本地执行——远程路径检查被跳过，由远端 rsync 处理访问验证。

#### 3. 源必须是目录
不支持通配符（`/data/*`）作为源参数。请使用父目录并借助 rsync 的过滤功能：
```bash
# 不支持：
./msrsync /data/* /backup/

# 改用 rsync 过滤：
./msrsync --rsync "-a --include='*.log' --exclude='*'" /data/ /backup/
```

#### 4. 中断后不支持续传
中断后需从头开始。为最小化超大传输的重传代价：
- 使用较小的桶（`-s 500M -f 500`）
- 结合 `-k` 在失败后检查逐桶 rsync 日志
- `-L` 日志文件会精确显示中断时正在运行哪个阶段

#### 5. --rsync 必须放在最后
`--rsync`（或 `-r`）必须紧接在源/目标参数之前：
```bash
# 正确：
./msrsync -p 8 --rsync "-a --numeric-ids" src/ dest/

# 错误 — rsync 选项会被错误解析：
./msrsync --rsync "-a --numeric-ids" -p 8 src/ dest/
```

#### 6. --delete 是两阶段的
当在 `--rsync` 中检测到 `--delete` 时，msrsync 使用两阶段方法（参见[安全的 --delete](#安全的---delete两阶段删除)）。阶段二作为单个 rsync 进程运行。对于包含数百万文件的树，这会增加一次相对较小的额外扫描，但预构建的 manifest 可降低其开销。

#### 7. 桶目录结构
桶文件存储在嵌套目录结构中（`<buckets>/0000/0000/...`），以避免在单个目录中处理数千个桶文件时文件系统性能下降。此结构自动创建，除非使用 `-k`，否则运行后自动清理。

#### 8. Python 版本兼容性
- Python 2.6、2.7：完全支持（RHEL 6 / CentOS 6 必需）
- Python 3.3+：完全支持
- Python 3.5+：原生使用 `os.scandir` 加速目录遍历
- Python < 3.5：通过 pip 安装 `scandir` 获得同等优化

#### 9. macOS 注意事项
- macOS 上 Python 3.8+ 默认使用 `spawn` 进行 multiprocessing。msrsync 强制使用 `fork`，因为 `G_MESSAGES_QUEUE` 必须被 worker 继承。
- GNU rsync 必须单独安装（见注意事项 #1）。

#### 10. 内存使用
每个 rsync worker 进程有自身的内存占用（rsync 本身通常 10-50 MB）。主进程在内存中持有桶队列。桶和 worker 数量较多时，SyncManager 队列也会消耗内存。对于极端情况（数百万文件、数百 worker），使用 `ps` 或 `top` 监控。

#### 11. TB 级小文件场景
同步包含数百万小文件的目录时：
- 遍历阶段耗时最长（单线程 `os.walk`）
- 使用 `--files 50000` 或更高以减少桶数量
- 失败/删除详情文件上限为每条 100,000 个条目
- 逐桶 rsync 日志按比例增长——仅在调试时使用 `-k`

### Makefile 目标

```
make test       运行嵌入式自测（等同于 ./msrsync --selftest）
make install    安装 msrsync 到 $(DESTDIR)/bin（默认：/usr/bin）
make lint       运行 pylint 静态分析
make cov        生成覆盖率报告（需要 python-coverage）
make covhtml    生成 HTML 覆盖率报告
make bench      运行基准测试（仅 Linux，推荐 root 以清除缓存）
make benchshm   在 /dev/shm 中运行基准测试（仅 Linux，推荐 root）
make clean      删除 .pyc 文件和 __pycache__ 目录
```

### 许可证

GNU General Public License v3.0。详见 `LICENSE` 文件。

本项目包含一份来自 [bup](https://github.com/bup/bup) 项目的 `options.py` 副本，该副本采用 BSD 许可证。版权所有 2010-2012 Avery Pennarun 及贡献者。
