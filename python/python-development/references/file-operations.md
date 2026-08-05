---
author: Ankur Bhatnagar
---

# File Operations — Full Reference

pathlib, CSV, JSON, text, binary, streaming, and temporary file patterns for Python 3.11+.

---

## Table of Contents
1. [pathlib Fundamentals](#pathlib-fundamentals)
2. [Reading and Writing Text Files](#reading-and-writing-text-files)
3. [JSON](#json)
4. [CSV](#csv)
5. [Binary Files](#binary-files)
6. [Streaming Large Files](#streaming-large-files)
7. [Temporary Files and Directories](#temporary-files-and-directories)
8. [Directory Operations](#directory-operations)
9. [File Watching](#file-watching)

---

## pathlib Fundamentals

```python
from pathlib import Path


# Construction
home      = Path.home()
cwd       = Path.cwd()
data_dir  = Path("/tmp/data")
rel_path  = Path("reports/2025")

# Joining (use / operator)
output    = data_dir / "output" / "orders.json"
config    = home / ".config" / "app" / "settings.toml"

# Parts
p = Path("/home/user/reports/orders.csv")
p.name       # "orders.csv"
p.stem       # "orders"
p.suffix     # ".csv"
p.suffixes   # [".csv"]
p.parent     # Path("/home/user/reports")
p.parents[1] # Path("/home/user")
p.parts      # ('/', 'home', 'user', 'reports', 'orders.csv')

# Checks
p.exists()
p.is_file()
p.is_dir()
p.is_symlink()

# Stats
stat = p.stat()
stat.st_size     # bytes
stat.st_mtime    # modification time (float)

# Resolve to absolute path (follows symlinks)
resolved = p.resolve()

# Relative path
p.relative_to("/home/user")  # Path("reports/orders.csv")

# Pattern matching
list(data_dir.glob("**/*.csv"))       # All CSVs recursively
list(data_dir.glob("orders_*.json"))  # Glob in current dir only
list(data_dir.rglob("*.log"))         # Recursive glob shorthand
```

---

## Reading and Writing Text Files

```python
from pathlib import Path


path = Path("data/orders.txt")

# Always specify encoding — never rely on platform default
text = path.read_text(encoding="utf-8")

# Write atomically: write to temp, rename
tmp  = path.with_suffix(".tmp")
tmp.write_text(content, encoding="utf-8")
tmp.replace(path)   # Atomic on POSIX; near-atomic on Windows

# Read line by line (memory-efficient)
def iter_lines(path: Path) -> list[str]:
    with path.open(encoding="utf-8") as f:
        return [line.rstrip("\n") for line in f]

# Append to a file
with path.open("a", encoding="utf-8") as f:
    f.write("new line\n")

# Read with explicit error handling
def safe_read(path: Path, default: str = "") -> str:
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        return default
    except PermissionError as exc:
        raise RuntimeError(f"Cannot read {path}: permission denied") from exc
```

---

## JSON

```python
import json
from pathlib import Path
from typing import Any


def read_json(path: Path) -> Any:
    with path.open(encoding="utf-8") as f:
        return json.load(f)


def write_json(data: Any, path: Path, *, indent: int = 2) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    # Write atomically
    tmp = path.with_suffix(".tmp")
    with tmp.open("w", encoding="utf-8") as f:
        json.dump(data, f, indent=indent, ensure_ascii=False, default=str)
    tmp.replace(path)


# Streaming JSON Lines (JSONL) — one JSON object per line
def iter_jsonl(path: Path):
    with path.open(encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                yield json.loads(line)


def write_jsonl(records: list[dict[str, Any]], path: Path) -> None:
    with path.open("w", encoding="utf-8") as f:
        for record in records:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")


# Custom JSON encoder for non-serialisable types
from datetime import datetime, date
from decimal import Decimal
from enum import Enum
from uuid import UUID


class AppJsonEncoder(json.JSONEncoder):
    def default(self, obj: object) -> object:
        match obj:
            case datetime() | date():
                return obj.isoformat()
            case Decimal():
                return float(obj)
            case Enum():
                return obj.value
            case UUID():
                return str(obj)
            case _:
                return super().default(obj)


# Usage
json.dumps(data, cls=AppJsonEncoder, indent=2)
```

---

## CSV

```python
import csv
from dataclasses import dataclass
from pathlib import Path


@dataclass
class OrderRow:
    order_id:    str
    customer_id: str
    status:      str
    amount:      float


FIELDNAMES = ["order_id", "customer_id", "status", "amount"]


def read_csv(path: Path) -> list[OrderRow]:
    with path.open(encoding="utf-8", newline="") as f:
        reader = csv.DictReader(f)
        return [
            OrderRow(
                order_id    = row["order_id"],
                customer_id = row["customer_id"],
                status      = row["status"],
                amount      = float(row["amount"]),
            )
            for row in reader
        ]


def write_csv(rows: list[OrderRow], path: Path) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("w", encoding="utf-8", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=FIELDNAMES, extrasaction="ignore")
        writer.writeheader()
        for row in rows:
            writer.writerow({
                "order_id":    row.order_id,
                "customer_id": row.customer_id,
                "status":      row.status,
                "amount":      row.amount,
            })


# Stream-process a large CSV without loading into memory
def process_large_csv(path: Path) -> int:
    count = 0
    with path.open(encoding="utf-8", newline="") as f:
        for row in csv.DictReader(f):
            process_row(row)
            count += 1
    return count
```

---

## Binary Files

```python
import hashlib
from pathlib import Path


def read_bytes(path: Path) -> bytes:
    return path.read_bytes()


def write_bytes(data: bytes, path: Path) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(".tmp")
    tmp.write_bytes(data)
    tmp.replace(path)


def sha256_file(path: Path) -> str:
    """Compute SHA-256 of a file without loading it entirely into memory."""
    hasher = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(65_536), b""):
            hasher.update(chunk)
    return hasher.hexdigest()


def copy_file_chunked(src: Path, dst: Path, chunk_size: int = 65_536) -> None:
    dst.parent.mkdir(parents=True, exist_ok=True)
    with src.open("rb") as fsrc, dst.open("wb") as fdst:
        for chunk in iter(lambda: fsrc.read(chunk_size), b""):
            fdst.write(chunk)
```

---

## Streaming Large Files

```python
from __future__ import annotations

from collections.abc import Iterator
from pathlib import Path
import csv


def stream_lines(path: Path, encoding: str = "utf-8") -> Iterator[str]:
    """Yield lines one at a time — O(1) memory regardless of file size."""
    with path.open(encoding=encoding) as f:
        for line in f:
            yield line.rstrip("\n")


def stream_csv_rows(
    path: Path,
    *,
    batch_size: int = 1000,
) -> Iterator[list[dict[str, str]]]:
    """Yield batches of CSV rows for bulk processing."""
    with path.open(encoding="utf-8", newline="") as f:
        reader = csv.DictReader(f)
        batch: list[dict[str, str]] = []
        for row in reader:
            batch.append(dict(row))
            if len(batch) >= batch_size:
                yield batch
                batch = []
        if batch:
            yield batch


def count_lines(path: Path) -> int:
    """Count lines without loading the file."""
    with path.open("rb") as f:
        return sum(1 for _ in f)
```

---

## Temporary Files and Directories

```python
import tempfile
from pathlib import Path


# Temporary file — deleted on context manager exit
def process_with_temp_file(data: bytes) -> str:
    with tempfile.NamedTemporaryFile(
        suffix=".bin",
        delete=True,
        dir=Path("/tmp"),
    ) as tmp:
        tmp.write(data)
        tmp.flush()
        return compute_hash(Path(tmp.name))


# Temporary directory — entire directory deleted on exit
def extract_and_process(archive_path: Path) -> list[str]:
    with tempfile.TemporaryDirectory(prefix="extract_") as tmpdir:
        workdir = Path(tmpdir)
        extract_archive(archive_path, workdir)
        return [
            process_file(f)
            for f in workdir.rglob("*.json")
        ]


# Explicit control (for when you need the path after the function returns)
def write_report(data: object) -> Path:
    fd, tmp_path = tempfile.mkstemp(suffix=".json", prefix="report_")
    try:
        path = Path(tmp_path)
        with path.open("w", encoding="utf-8") as f:
            import json
            json.dump(data, f, indent=2)
        return path
    except Exception:
        Path(tmp_path).unlink(missing_ok=True)
        raise
    finally:
        import os
        os.close(fd)
```

---

## Directory Operations

```python
from pathlib import Path
import shutil


def ensure_dir(path: Path) -> Path:
    """Create directory and parents; no-op if already exists."""
    path.mkdir(parents=True, exist_ok=True)
    return path


def list_files(directory: Path, pattern: str = "*") -> list[Path]:
    return sorted(directory.glob(pattern))


def list_files_recursive(directory: Path, suffix: str = ".json") -> list[Path]:
    return sorted(directory.rglob(f"*{suffix}"))


def copy_tree(src: Path, dst: Path) -> None:
    """Copy directory tree, overwriting if dst exists."""
    if dst.exists():
        shutil.rmtree(dst)
    shutil.copytree(src, dst)


def move_file(src: Path, dst: Path) -> None:
    dst.parent.mkdir(parents=True, exist_ok=True)
    src.rename(dst)


def delete_old_files(directory: Path, *, older_than_days: int) -> int:
    """Delete files older than N days; return count deleted."""
    import time
    cutoff = time.time() - (older_than_days * 86_400)
    count  = 0
    for path in directory.rglob("*"):
        if path.is_file() and path.stat().st_mtime < cutoff:
            path.unlink()
            count += 1
    return count


def disk_usage(path: Path) -> int:
    """Total size of all files under path in bytes."""
    return sum(f.stat().st_size for f in path.rglob("*") if f.is_file())
```

---

## File Watching

```python
# Use watchdog for production file watching
# pip install watchdog

from pathlib import Path
from watchdog.events import FileSystemEvent, FileSystemEventHandler
from watchdog.observers import Observer
import time


class OrderFileHandler(FileSystemEventHandler):
    def on_created(self, event: FileSystemEvent) -> None:
        if event.is_directory:
            return
        path = Path(str(event.src_path))
        if path.suffix == ".json":
            print(f"New file: {path}")
            process_new_order_file(path)

    def on_modified(self, event: FileSystemEvent) -> None:
        if not event.is_directory:
            print(f"Modified: {event.src_path}")


def watch_directory(directory: Path) -> None:
    handler  = OrderFileHandler()
    observer = Observer()
    observer.schedule(handler, str(directory), recursive=False)
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```
