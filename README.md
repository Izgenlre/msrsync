# msrsync — Multi-Stream rsync

[English](#english) | [中文](#中文)

---

## English

### Overview

**msrsync** parallelizes `rsync` to maximize throughput for large data transfers. It splits the file tree into buckets (of configurable size and file count), then runs multiple rsync processes concurrently — each handling its own bucket. No more single-threaded rsync bottleneck on multi-terabyte datasets.

Originally by [Jean-Baptiste Denis](https://github.com/jbd/msrsync). This fork adds:

- **Enterprise-grade structured logging** with timestamped, level-tagged messages (`--log-file`)
- **Safe two-phase `--delete`** — all buckets transfer in parallel first, then a single rsync process handles orphan removal using a merged manifest, eliminating race conditions
- **Automatic failed/deleted file tracking** with sampled summaries and full detail files
- **Python 2.6+ / 3.x cross-compatibility**, including macOS `fork` fix for Python 3.8+
- **Case-insensitive size suffixes** — `-s 1k` and `-s 1K` are both accepted
- **Enhanced robustness** — monitor worker crash protection with timeout fallback, symlink-aware crawl optimization

### Requirements

- **Python** 2.6+ or 3.x
- **rsync** (GNU rsync required; macOS `openrsync` does not support `--from0`)

### Quick Start

```bash
# Basic: 8 parallel rsync processes
./msrsync -p 8 /data/source/ /backup/dest/

# With progress and custom rsync flags
./msrsync -p 16 -P --rsync "-avz --numeric-ids" /data/ /backup/

# Mirror sync with safe --delete (two-phase: transfer then single-process delete)
./msrsync -p 8 --rsync "-a --delete-after" /data/ /backup/

# Run self-tests
./msrsync --selftest

# Install
make install DESTDIR=/usr/local
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `-p, --processes N` | Number of parallel rsync workers | 1 |
| `-f, --files N` | Max files per bucket | 1000 |
| `-s, --size N` | Max size per bucket (K/M/G/T/P/E/Z/Y, case-insensitive) | 1G |
| `-b, --buckets DIR` | Bucket files directory | auto tmpdir |
| `-k, --keep` | Keep bucket files after run | off |
| `-j, --show` | Print bucket directory path | off |
| `-P, --progress` | Show real-time progress | off |
| `--stats` | Print summary statistics | off |
| `-d, --dry-run` | Bucket and plan, skip rsync | off |
| `-v, --version` | Print version and exit | |
| `-L, --log-file PATH` | Write timestamped log to PATH | off |
| `-r, --rsync "OPTS"` | rsync options (MUST be last) | `-aS --numeric-ids` |
| `-t, --selftest` | Run embedded tests | |
| `-e, --bench` | Run benchmarks | |
| `-g, --benchshm` | Benchmarks in /dev/shm | |

### How It Works

```
Source Directory
      │
      ▼
   crawl() ──► file list (size, path) tuples
      │
      ▼
  buckets() ──► split into N buckets (by size or count)
      │
      ├─► Bucket 1 ──► rsync --files-from=bucket1  ──┐
      ├─► Bucket 2 ──► rsync --files-from=bucket2  ──┤──► Destination
      └─► Bucket N ──► rsync --files-from=bucketN  ──┘
      │
      ▼ (if --delete)
  Phase 2: single rsync --delete using merged manifest
```

### Safe `--delete` (Two-Phase)

When `--delete` (or `--delete-after`, `--del`, etc.) is passed inside `--rsync`, msrsync automatically:

1. **Phase 1**: Strips `--delete` from bucket-level rsync commands. All files are transferred in parallel.
2. **Phase 2**: After all workers finish, runs a **single** rsync process with `--delete` using a merged manifest built during bucketing. This avoids the race condition where concurrent `--delete` workers delete each other's files.

Supported delete variants: `--delete`, `--delete-after`, `--delete-before`, `--delete-during`, `--delete-delay`, `--delete-excluded`, `--del`.

### Enterprise Logging & Audit

**`--log-file PATH`** — Writes a timestamped, structured log of the entire run:

```
[2026-07-20T17:30:05] [INFO] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] [INFO] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
[2026-07-20T17:30:17] [INFO] crawl complete: 152340 entries, 856.3 G total, 87 buckets
[2026-07-20T17:45:32] [INFO] all rsync workers finished
[2026-07-20T17:45:40] [INFO] delete pass complete: 0 errors
```

- `[timestamp]` — ISO 8601, reflects event time (not print time)
- `[LEVEL]` — INFO, ERROR, or DEBUG
- Log file is line-buffered; written from a dedicated worker process (zero impact on transfer throughput)
- Progress lines (`[PROGRESS]`) are NOT written to the log file

### Progress Output

When `-P` is used, msrsync displays a live progress line:

```
[85230/152340 entries] [520.4 G/856.3 G transferred] [3520 entries/s] [3.1 G/s bw] [monq 12] [jq 5]
```

| Field | Meaning |
|-------|---------|
| `entries` | Files transferred / total |
| `transferred` | Data transferred / total |
| `entries/s` | File throughput |
| `bw` | Bandwidth |
| `monq` | Monitor queue depth |
| `jq` | Remaining job buckets |

### Stats Summary

When `--stats` is used, a summary is printed at the end:

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

### Per-Bucket rsync Logs

Each rsync worker generates its own log file (controlled by rsync's `--log-file`):

```
/tmp/msrsync-XXXXXX/0000/0000/tmpXXXXXX.log
/tmp/msrsync-XXXXXX/0000/0001/tmpXXXXXX.log
```

These contain rsync's `--verbose --stats` output. Keep them with `-k`, show their location with `-j`.

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success (all buckets transferred, no errors) |
| 1 | Uncaught exception during run |
| 10 | Python < 2.6 |
| 11 | Bucket directory does not exist |
| 12 | Bucket directory not writable |
| 13 | Cannot write bucket file |
| 14 | rsync executable not found in PATH |
| 15 | Source is not a directory |
| 16 | No access to source directory |
| 17 | Destination not writable |
| 18 | Destination is not a directory |
| 19 | rsync options validation failed |
| 20 | rsync timeout |
| 21 | rsync job error |
| 24 | Cannot create destination directory |
| 27 | Interrupted (Ctrl+C) |
| 28 | OS error creating bucket directory |
| 97 | Command-line parse error |

### Caveats

1. **GNU rsync required.** macOS `openrsync` lacks `--from0`. Install GNU rsync via Homebrew: `brew install rsync`.
2. **Local directories only.** Remote-shell (`user@host:/path`) is not supported for source/destination checking, though rsync options can include `-e ssh`.
3. **Source directory only.** Wildcards like `/data/*` are not supported; use the parent directory.
4. **No resume.** Interrupted runs must be restarted from scratch. Consider smaller bucket sizes for very large transfers to minimize re-work.
5. **`--delete` is two-phase.** Phase 2 runs a single rsync process; for very large trees, this adds a second scan pass. The manifest approach with `--files-from` minimizes this cost.
6. **Root required for benchmarks.** `--bench` drops buffer caches via `/proc/sys/vm/drop_caches` (Linux only).

### Makefile Targets

```
make test       Run self-tests
make install    Install to $(DESTDIR)/bin
make lint       Run pylint
make cov        Coverage report
make bench      Run benchmarks (Linux, root)
make benchshm   Benchmarks in /dev/shm (Linux, root)
```

---

## 中文

### 概述

**msrsync** 通过并行化 `rsync` 来最大化大数据传输的吞吐量。它将文件树按可配置的大小和文件数拆分为多个桶（bucket），然后并发运行多个 rsync 进程——每个进程处理自己的桶。TB 级数据不再受单线程 rsync 的瓶颈限制。

原作者：[Jean-Baptiste Denis](https://github.com/jbd/msrsync)。本分支新增功能：

- **企业级结构化日志**，带时间戳和级别标签（`--log-file`）
- **安全的两阶段 `--delete`** — 所有桶先并行传输，传输完成后由单个 rsync 进程使用合并的 manifest 文件统一处理孤儿文件删除，杜绝竞态条件
- **自动失败/删除文件追踪**，提供采样摘要和完整详情文件
- **Python 2.6+ / 3.x 跨版本兼容**，含 macOS Python 3.8+ 的 `fork` 修复
- **大小写不敏感的后缀** — `-s 1k` 和 `-s 1K` 等效
- **增强健壮性** — monitor worker 崩溃保护与超时兜底、符号链接目录遍历优化

### 环境要求

- **Python** 2.6+ 或 3.x
- **rsync**（需要 GNU rsync；macOS 自带的 `openrsync` 不支持 `--from0`）

### 快速开始

```bash
# 基本用法：8 个并行 rsync 进程
./msrsync -p 8 /data/source/ /backup/dest/

# 显示进度并自定义 rsync 参数
./msrsync -p 16 -P --rsync "-avz --numeric-ids" /data/ /backup/

# 镜像同步（安全的两阶段 --delete：先传输再单进程删除）
./msrsync -p 8 --rsync "-a --delete-after" /data/ /backup/

# 运行自测
./msrsync --selftest

# 安装
make install DESTDIR=/usr/local
```

### 选项说明

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `-p, --processes N` | 并行 rsync 进程数 | 1 |
| `-f, --files N` | 每个桶最多文件数 | 1000 |
| `-s, --size N` | 每个桶最大大小（K/M/G/T/P/E/Z/Y，大小写不敏感） | 1G |
| `-b, --buckets DIR` | 桶文件存放目录 | 自动临时目录 |
| `-k, --keep` | 运行后保留桶文件 | 关闭 |
| `-j, --show` | 打印桶目录路径 | 关闭 |
| `-P, --progress` | 显示实时进度 | 关闭 |
| `--stats` | 打印汇总统计 | 关闭 |
| `-d, --dry-run` | 只分桶不执行 rsync | 关闭 |
| `-v, --version` | 打印版本并退出 | |
| `-L, --log-file PATH` | 将带时间戳的日志写入 PATH | 关闭 |
| `-r, --rsync "OPTS"` | rsync 选项（必须放在最后） | `-aS --numeric-ids` |
| `-t, --selftest` | 运行内置测试 | |
| `-e, --bench` | 运行基准测试 | |
| `-g, --benchshm` | 在 /dev/shm 中运行基准测试 | |

### 工作原理

```
源目录
      │
      ▼
   crawl() ──► 文件列表 (大小, 路径) 元组
      │
      ▼
  buckets() ──► 拆分为 N 个桶（按大小或文件数）
      │
      ├─► 桶1 ──► rsync --files-from=桶1 ──┐
      ├─► 桶2 ──► rsync --files-from=桶2 ──┤──► 目标目录
      └─► 桶N ──► rsync --files-from=桶N ──┘
      │
      ▼ (如果启用 --delete)
  Phase 2: 单进程 rsync --delete（使用合并的 manifest）
```

### 安全的 `--delete`（两阶段）

当在 `--rsync` 中传入 `--delete`（或其变体如 `--delete-after`、`--del` 等）时，msrsync 自动执行：

1. **阶段一**：从桶级 rsync 命令中剥离 `--delete`。所有文件并行传输。
2. **阶段二**：所有 worker 完成后，使用在分桶阶段构建的 manifest 文件，运行**单个** rsync 进程执行 `--delete`。这避免了并发 `--delete` worker 互相删除对方文件的竞态条件。

支持的 delete 变体：`--delete`、`--delete-after`、`--delete-before`、`--delete-during`、`--delete-delay`、`--delete-excluded`、`--del`。

### 企业级日志与审计

**`--log-file PATH`** — 将整个运行过程写入带时间戳的结构化日志：

```
[2026-07-20T17:30:05] [INFO] buckets directory: /tmp/msrsync-abc123
[2026-07-20T17:30:05] [INFO] starting with 8 rsync worker(s), files per bucket=1000, size per bucket=1073741824 bytes
[2026-07-20T17:30:17] [INFO] crawl complete: 152340 entries, 856.3 G total, 87 buckets
[2026-07-20T17:45:32] [INFO] all rsync workers finished
[2026-07-20T17:45:40] [INFO] delete pass complete: 0 errors
```

- `[时间戳]` — ISO 8601 格式，反映事件发生时间（而非打印时间）
- `[级别]` — INFO、ERROR 或 DEBUG
- 日志文件使用行缓冲；由独立的消息进程写入（对传输吞吐零影响）
- 进度行（`[PROGRESS]`）不会写入日志文件

### 进度输出

使用 `-P` 时，msrsync 显示实时进度行：

```
[85230/152340 entries] [520.4 G/856.3 G transferred] [3520 entries/s] [3.1 G/s bw] [monq 12] [jq 5]
```

| 字段 | 含义 |
|------|------|
| `entries` | 已传输 / 总文件数 |
| `transferred` | 已传输 / 总数据量 |
| `entries/s` | 文件处理速率 |
| `bw` | 带宽 |
| `monq` | monitor 队列深度（待处理结果数） |
| `jq` | 剩余作业桶数 |

### 统计汇总

使用 `--stats` 时，运行结束后打印汇总（格式见 English 部分）。

### 每个桶的 rsync 日志

每个 rsync worker 生成独立的日志文件（由 rsync 的 `--log-file` 控制）：

```
/tmp/msrsync-XXXXXX/0000/0000/tmpXXXXXX.log
/tmp/msrsync-XXXXXX/0000/0001/tmpXXXXXX.log
```

包含 rsync 的 `--verbose --stats` 输出。使用 `-k` 保留，`-j` 查看路径。

### 退出码

| 码 | 含义 |
|----|------|
| 0 | 成功 |
| 1 | 未捕获的异常 |
| 10 | Python < 2.6 |
| 11 | 桶目录不存在 |
| 12 | 桶目录不可写 |
| 13 | 无法写入桶文件 |
| 14 | PATH 中找不到 rsync |
| 15 | 源不是目录 |
| 16 | 无源目录访问权限 |
| 17 | 目标不可写 |
| 18 | 目标不是目录 |
| 19 | rsync 选项校验失败 |
| 20 | rsync 超时 |
| 21 | rsync 任务错误 |
| 24 | 无法创建目标目录 |
| 27 | 中断 (Ctrl+C) |
| 28 | 创建桶目录时 OS 错误 |
| 97 | 命令行解析错误 |

### 注意事项

1. **需要 GNU rsync。** macOS 的 `openrsync` 缺少 `--from0`。通过 Homebrew 安装：`brew install rsync`。
2. **仅支持本地目录。** 远程 shell（`user@host:/path`）不支持源/目标检查，但 rsync 选项可以包含 `-e ssh`。
3. **仅支持目录作为源。** 通配符如 `/data/*` 不支持；使用父目录。
4. **不支持断点续传。** 中断后需从头开始。对于超大传输，建议使用较小的桶以最小化重传。
5. **`--delete` 是两阶段的。** 阶段二运行单个 rsync 进程；对于超大目录树，会额外增加一次扫描。Manifest + `--files-from` 方案已最小化此开销。
6. **基准测试需要 root。** `--bench` 通过 `/proc/sys/vm/drop_caches` 清除缓存（仅 Linux）。

### Makefile 目标

```
make test       运行自测
make install    安装到 $(DESTDIR)/bin
make lint       运行 pylint
make cov        覆盖率报告
make bench      运行基准测试（Linux，root）
make benchshm   在 /dev/shm 中运行基准测试（Linux，root）
```
