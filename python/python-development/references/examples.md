---
author: Ankur Bhatnagar
---

# Examples — Full Reference

Complete worked examples for general Python scripting and utilities.
Each example is self-contained and production-ready.

---

## Table of Contents
1. [CLI Tool — Order Processor](#example-1-cli-tool--order-processor)
2. [Async API Fetcher with Retry](#example-2-async-api-fetcher-with-retry)
3. [CSV-to-JSON ETL Pipeline](#example-3-csv-to-json-etl-pipeline)
4. [Config-Driven Script](#example-4-config-driven-script)

---

## Example 1: CLI Tool — Order Processor

```python
# src/order_processor/main.py

from __future__ import annotations

import logging
import sys
from pathlib import Path

import typer
from rich.console import Console
from rich.progress import Progress, SpinnerColumn, TextColumn

from order_processor.config import Config
from order_processor.models import Order, OrderStatus
from order_processor.processor import process_orders
from order_processor.readers import read_orders_csv
from order_processor.writers import write_orders_json

app     = typer.Typer(help="Process raw order exports into enriched JSON.")
console = Console()
logger  = logging.getLogger(__name__)


def configure_logging(verbose: bool) -> None:
    level = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(
        level   = level,
        format  = "%(asctime)s  %(levelname)-8s  %(name)s  %(message)s",
        stream  = sys.stdout,
    )
    logging.getLogger("httpx").setLevel(logging.WARNING)


@app.command()
def process(
    input_file: Path = typer.Argument(
        ...,
        exists=True,
        readable=True,
        file_okay=True,
        dir_okay=False,
        help="Path to the CSV export.",
    ),
    output_dir: Path = typer.Option(
        Path("./output"),
        "--output", "-o",
        help="Directory for the enriched JSON output.",
    ),
    status_filter: str | None = typer.Option(
        None,
        "--status", "-s",
        help="Filter to a specific status (pending, shipped, delivered).",
    ),
    dry_run: bool = typer.Option(
        False, "--dry-run",
        help="Validate input; do not write output files.",
    ),
    verbose: bool = typer.Option(False, "--verbose", "-v"),
) -> None:
    """Process orders from INPUT_FILE and write enriched JSON to OUTPUT_DIR."""
    configure_logging(verbose)
    config = Config.from_env()

    console.print(f"[bold]Reading:[/bold] {input_file}")

    try:
        raw_orders = read_orders_csv(input_file)
    except FileNotFoundError:
        console.print(f"[red]File not found:[/red] {input_file}", err=True)
        raise typer.Exit(code=1)

    console.print(f"Loaded [bold]{len(raw_orders)}[/bold] orders")

    if status_filter:
        try:
            filter_status = OrderStatus(status_filter.lower())
        except ValueError:
            console.print(f"[red]Invalid status:[/red] {status_filter}", err=True)
            raise typer.Exit(code=1)
        raw_orders = [o for o in raw_orders if o.status == filter_status]
        console.print(f"Filtered to [bold]{len(raw_orders)}[/bold] orders")

    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        console=console,
    ) as progress:
        task = progress.add_task("Processing...", total=None)
        try:
            processed = process_orders(raw_orders, config=config)
        except Exception as exc:
            logger.exception("Processing failed")
            console.print(f"[red]Error:[/red] {exc}", err=True)
            raise typer.Exit(code=1)
        progress.update(task, completed=True)

    if dry_run:
        console.print(f"[yellow]Dry run — not writing output.[/yellow]")
        console.print(f"Would write {len(processed)} orders.")
        return

    output_dir.mkdir(parents=True, exist_ok=True)
    output_file = output_dir / f"{input_file.stem}_enriched.json"
    write_orders_json(processed, output_file)
    console.print(f"[green]Done.[/green] Wrote {len(processed)} orders to {output_file}")


if __name__ == "__main__":
    app()
```

---

## Example 2: Async API Fetcher with Retry

```python
# src/order_processor/fetcher.py

from __future__ import annotations

import asyncio
import logging
from collections.abc import Sequence
from dataclasses import dataclass

import httpx

logger = logging.getLogger(__name__)

RETRY_STATUS_CODES = {429, 500, 502, 503, 504}


@dataclass(frozen=True)
class FetchResult:
    order_id: str
    data:     dict[str, object] | None
    error:    str | None = None

    @property
    def is_ok(self) -> bool:
        return self.error is None


class OrderApiFetcher:
    def __init__(
        self,
        base_url: str,
        api_key:  str,
        *,
        timeout:       float = 10.0,
        max_retries:   int   = 3,
        retry_delay:   float = 1.0,
        max_concurrent: int  = 10,
    ) -> None:
        self._base_url      = base_url
        self._api_key       = api_key
        self._timeout       = timeout
        self._max_retries   = max_retries
        self._retry_delay   = retry_delay
        self._semaphore     = asyncio.Semaphore(max_concurrent)
        self._client: httpx.AsyncClient | None = None

    async def __aenter__(self) -> OrderApiFetcher:
        self._client = httpx.AsyncClient(
            base_url = self._base_url,
            timeout  = self._timeout,
            headers  = {
                "Accept":        "application/json",
                "Authorization": f"Bearer {self._api_key}",
            },
        )
        return self

    async def __aexit__(self, *_: object) -> None:
        if self._client:
            await self._client.aclose()

    async def fetch_order(self, order_id: str) -> FetchResult:
        assert self._client, "Call inside async context manager"

        async with self._semaphore:
            for attempt in range(1, self._max_retries + 1):
                try:
                    response = await self._client.get(f"/orders/{order_id}")

                    if response.status_code == 404:
                        return FetchResult(order_id=order_id, data=None)

                    if response.status_code in RETRY_STATUS_CODES and attempt < self._max_retries:
                        wait = self._retry_delay * (2 ** (attempt - 1))
                        logger.warning(
                            "HTTP %d for order %s — retrying in %.1fs (attempt %d/%d)",
                            response.status_code, order_id, wait, attempt, self._max_retries,
                        )
                        await asyncio.sleep(wait)
                        continue

                    response.raise_for_status()
                    return FetchResult(order_id=order_id, data=response.json())

                except httpx.TimeoutException:
                    if attempt == self._max_retries:
                        return FetchResult(
                            order_id=order_id, data=None,
                            error=f"Timeout after {self._timeout}s",
                        )
                    await asyncio.sleep(self._retry_delay * attempt)

                except httpx.HTTPStatusError as exc:
                    return FetchResult(
                        order_id=order_id, data=None,
                        error=f"HTTP {exc.response.status_code}",
                    )

        return FetchResult(order_id=order_id, data=None, error="Max retries exceeded")

    async def fetch_many(
        self,
        order_ids: Sequence[str],
    ) -> list[FetchResult]:
        tasks = [self.fetch_order(oid) for oid in order_ids]
        return list(await asyncio.gather(*tasks))


# Sync wrapper for use in non-async code
def fetch_orders_sync(
    order_ids: list[str],
    base_url:  str,
    api_key:   str,
) -> list[FetchResult]:
    async def run() -> list[FetchResult]:
        async with OrderApiFetcher(base_url, api_key) as fetcher:
            return await fetcher.fetch_many(order_ids)

    return asyncio.run(run())
```

---

## Example 3: CSV-to-JSON ETL Pipeline

```python
# scripts/etl_orders.py

from __future__ import annotations

import csv
import json
import logging
import sys
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class RawOrderRow:
    order_id:    str
    customer_id: str
    status:      str
    amount:      str          # Raw string from CSV
    created_at:  str


@dataclass(frozen=True)
class EnrichedOrder:
    order_id:    str
    customer_id: str
    status:      str
    amount:      float
    created_at:  datetime
    is_high_value: bool


def parse_raw_row(row: dict[str, str], line_num: int) -> RawOrderRow | None:
    required = {"order_id", "customer_id", "status", "amount", "created_at"}
    missing  = required - set(row.keys())
    if missing:
        logger.warning("Line %d: missing columns %s — skipping", line_num, missing)
        return None
    return RawOrderRow(**{k: row[k].strip() for k in required})


def enrich(raw: RawOrderRow) -> EnrichedOrder | None:
    try:
        amount     = float(raw.amount)
        created_at = datetime.fromisoformat(raw.created_at)
    except ValueError as exc:
        logger.warning("Cannot parse row %s: %s — skipping", raw.order_id, exc)
        return None

    return EnrichedOrder(
        order_id     = raw.order_id,
        customer_id  = raw.customer_id,
        status       = raw.status.lower(),
        amount       = amount,
        created_at   = created_at,
        is_high_value = amount >= 1000.0,
    )


def run_etl(input_path: Path, output_path: Path) -> dict[str, int]:
    stats = {"read": 0, "parsed": 0, "enriched": 0, "skipped": 0}

    output_path.parent.mkdir(parents=True, exist_ok=True)

    with (
        input_path.open(encoding="utf-8", newline="") as src,
        output_path.open("w", encoding="utf-8") as dst,
    ):
        reader = csv.DictReader(src)

        for line_num, row in enumerate(reader, start=2):   # 1 = header
            stats["read"] += 1

            raw = parse_raw_row(row, line_num)
            if raw is None:
                stats["skipped"] += 1
                continue
            stats["parsed"] += 1

            enriched = enrich(raw)
            if enriched is None:
                stats["skipped"] += 1
                continue
            stats["enriched"] += 1

            # Write JSONL — one record per line
            dst.write(
                json.dumps(
                    {
                        "order_id":     enriched.order_id,
                        "customer_id":  enriched.customer_id,
                        "status":       enriched.status,
                        "amount":       enriched.amount,
                        "created_at":   enriched.created_at.isoformat(),
                        "is_high_value": enriched.is_high_value,
                    },
                    ensure_ascii=False,
                )
                + "\n"
            )

    return stats


def main() -> None:
    logging.basicConfig(
        level  = logging.INFO,
        format = "%(asctime)s  %(levelname)-8s  %(message)s",
        stream = sys.stdout,
    )

    if len(sys.argv) != 3:
        print("Usage: python etl_orders.py <input.csv> <output.jsonl>", file=sys.stderr)
        sys.exit(1)

    input_path  = Path(sys.argv[1])
    output_path = Path(sys.argv[2])

    logger.info("ETL: %s → %s", input_path, output_path)
    stats = run_etl(input_path, output_path)
    logger.info("Stats: %s", stats)


if __name__ == "__main__":
    main()
```

---

## Example 4: Config-Driven Script

```python
# src/order_processor/config.py

from __future__ import annotations

import os
from dataclasses import dataclass, field


class ConfigError(Exception):
    """Raised when required configuration is missing or invalid."""


@dataclass(frozen=True)
class DatabaseConfig:
    host:     str
    port:     int
    name:     str
    user:     str
    password: str = field(repr=False)

    @property
    def url(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"

    @classmethod
    def from_env(cls, prefix: str = "DB") -> DatabaseConfig:
        return cls(
            host     = _require(f"{prefix}_HOST"),
            port     = int(_get(f"{prefix}_PORT", "5432")),
            name     = _require(f"{prefix}_NAME"),
            user     = _require(f"{prefix}_USER"),
            password = _require(f"{prefix}_PASSWORD"),
        )


@dataclass(frozen=True)
class ApiConfig:
    base_url:    str
    api_key:     str = field(repr=False)
    timeout:     float = 10.0
    max_retries: int   = 3

    @classmethod
    def from_env(cls, prefix: str = "API") -> ApiConfig:
        return cls(
            base_url    = _require(f"{prefix}_BASE_URL"),
            api_key     = _require(f"{prefix}_KEY"),
            timeout     = float(_get(f"{prefix}_TIMEOUT", "10")),
            max_retries = int(_get(f"{prefix}_MAX_RETRIES", "3")),
        )


@dataclass(frozen=True)
class Config:
    db:        DatabaseConfig
    api:       ApiConfig
    log_level: str
    dry_run:   bool

    @classmethod
    def from_env(cls) -> Config:
        return cls(
            db        = DatabaseConfig.from_env(),
            api       = ApiConfig.from_env(),
            log_level = _get("LOG_LEVEL", "INFO").upper(),
            dry_run   = _get("DRY_RUN", "false").lower() == "true",
        )


def _require(key: str) -> str:
    value = os.getenv(key)
    if not value:
        raise ConfigError(f"Required environment variable '{key}' is not set.")
    return value


def _get(key: str, default: str) -> str:
    return os.getenv(key, default)


# Usage
# config = Config.from_env()
# print(config.api.base_url)   # "https://api.example.com"
# print(config.db.url)         # "postgresql://..."
```
