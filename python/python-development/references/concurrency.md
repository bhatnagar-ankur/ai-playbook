---
author: Ankur Bhatnagar
---

# Concurrency — Full Reference

asyncio, threading, multiprocessing, and concurrent.futures patterns for Python 3.11+.
For the overview see the **Python Conventions** section in `SKILL.md`.

---

## Table of Contents
1. [asyncio Basics](#asyncio-basics)
2. [Concurrent Tasks](#concurrent-tasks)
3. [Async Context Managers and Iterators](#async-context-managers-and-iterators)
4. [asyncio with httpx](#asyncio-with-httpx)
5. [ThreadPoolExecutor](#threadpoolexecutor)
6. [ProcessPoolExecutor](#processpoolexecutor)
7. [Queues and Pipelines](#queues-and-pipelines)
8. [Timeouts and Cancellation](#timeouts-and-cancellation)

---

## asyncio Basics

```python
from __future__ import annotations

import asyncio
import logging

logger = logging.getLogger(__name__)


async def fetch_order(order_id: str) -> dict[str, object]:
    """Simulate an async I/O operation."""
    await asyncio.sleep(0.1)   # Yields control to the event loop
    return {"id": order_id, "status": "pending"}


async def process_order(order_id: str) -> str:
    logger.info("Processing %s", order_id)
    order = await fetch_order(order_id)
    return str(order["id"])


# Entry point — run the event loop
def main() -> None:
    result = asyncio.run(process_order("ORD-001"))
    print(result)


if __name__ == "__main__":
    main()
```

---

## Concurrent Tasks

```python
import asyncio
from collections.abc import Sequence


async def fetch_all_orders(order_ids: Sequence[str]) -> list[dict[str, object]]:
    """Fetch all orders concurrently and return all results."""
    tasks = [fetch_order(oid) for oid in order_ids]

    # gather() — run all concurrently; raises on first exception by default
    results = await asyncio.gather(*tasks)
    return list(results)


async def fetch_all_orders_partial(
    order_ids: Sequence[str],
) -> list[dict[str, object] | BaseException]:
    """Fetch all; return successes and errors without aborting."""
    tasks = [fetch_order(oid) for oid in order_ids]

    # return_exceptions=True — collect errors as values instead of raising
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for i, result in enumerate(results):
        if isinstance(result, BaseException):
            logger.error("Failed to fetch order %s: %s", order_ids[i], result)

    return list(results)


async def process_with_semaphore(
    order_ids: Sequence[str],
    *,
    max_concurrent: int = 5,
) -> list[dict[str, object]]:
    """Limit concurrent I/O to avoid overwhelming downstream services."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def guarded_fetch(oid: str) -> dict[str, object]:
        async with semaphore:
            return await fetch_order(oid)

    return list(await asyncio.gather(*(guarded_fetch(oid) for oid in order_ids)))


async def process_as_completed(
    order_ids: Sequence[str],
) -> None:
    """Process results as they arrive, not waiting for all to finish."""
    tasks = [asyncio.create_task(fetch_order(oid), name=oid) for oid in order_ids]

    for coro in asyncio.as_completed(tasks):
        try:
            result = await coro
            print(f"Done: {result['id']}")
        except Exception as exc:
            logger.error("Task failed: %s", exc)
```

---

## Async Context Managers and Iterators

```python
from contextlib import asynccontextmanager


class AsyncOrderClient:
    def __init__(self, base_url: str) -> None:
        self._base_url = base_url
        self._session: httpx.AsyncClient | None = None

    async def __aenter__(self) -> AsyncOrderClient:
        self._session = httpx.AsyncClient(base_url=self._base_url)
        return self

    async def __aexit__(self, *_: object) -> None:
        if self._session:
            await self._session.aclose()

    async def get_order(self, order_id: str) -> dict[str, object]:
        assert self._session, "Client not open"
        response = await self._session.get(f"/orders/{order_id}")
        response.raise_for_status()
        return response.json()


# Usage
async def main() -> None:
    async with AsyncOrderClient("https://api.example.com") as client:
        order = await client.get_order("ORD-001")
        print(order)


# asynccontextmanager decorator
@asynccontextmanager
async def managed_transaction(db: AsyncSession):
    async with db.begin():
        try:
            yield db
        except Exception:
            await db.rollback()
            raise


# Async generator / iterator
async def stream_orders(page_size: int = 100):
    """Yield orders page by page — avoids loading everything into memory."""
    page = 1
    while True:
        orders = await fetch_orders_page(page, page_size)
        if not orders:
            break
        for order in orders:
            yield order
        page += 1


async def process_stream() -> None:
    async for order in stream_orders(page_size=50):
        await process_order(order["id"])
```

---

## asyncio with httpx

```python
import asyncio
import httpx
from collections.abc import Sequence


class OrderApiClient:
    def __init__(self, base_url: str, *, timeout: float = 10.0) -> None:
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout,
            headers={"Accept": "application/json"},
        )

    async def __aenter__(self) -> OrderApiClient:
        return self

    async def __aexit__(self, *_: object) -> None:
        await self._client.aclose()

    async def get_order(self, order_id: str) -> dict[str, object] | None:
        try:
            response = await self._client.get(f"/orders/{order_id}")
            if response.status_code == 404:
                return None
            response.raise_for_status()
            return response.json()
        except httpx.TimeoutException:
            logger.warning("Timeout fetching order %s", order_id)
            raise
        except httpx.HTTPStatusError as exc:
            logger.error("HTTP error %d for order %s", exc.response.status_code, order_id)
            raise

    async def fetch_many(
        self,
        order_ids: Sequence[str],
        *,
        max_concurrent: int = 10,
    ) -> list[dict[str, object] | None]:
        semaphore = asyncio.Semaphore(max_concurrent)

        async def guarded(oid: str) -> dict[str, object] | None:
            async with semaphore:
                return await self.get_order(oid)

        return list(
            await asyncio.gather(
                *(guarded(oid) for oid in order_ids),
                return_exceptions=False,
            )
        )
```

---

## ThreadPoolExecutor

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from functools import partial


# Run a blocking (sync) function in a thread pool without blocking the event loop
async def run_blocking(func, *args, **kwargs):
    loop = asyncio.get_running_loop()
    with ThreadPoolExecutor() as pool:
        return await loop.run_in_executor(
            pool,
            partial(func, *args, **kwargs),
        )


# Example: CPU-bound or blocking I/O in threads
def read_csv_file(path: str) -> list[dict[str, str]]:
    import csv
    with open(path) as f:
        return list(csv.DictReader(f))


async def process_csv(path: str) -> None:
    rows = await run_blocking(read_csv_file, path)
    print(f"Loaded {len(rows)} rows")


# Synchronous threading — use when you don't have asyncio
from concurrent.futures import ThreadPoolExecutor, as_completed


def process_orders_threaded(order_ids: list[str], max_workers: int = 4) -> list[str]:
    results: list[str] = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(process_order_sync, oid): oid for oid in order_ids}

        for future in as_completed(futures):
            oid = futures[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as exc:
                logger.error("Order %s failed: %s", oid, exc)

    return results
```

---

## ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor
import os


def cpu_intensive_task(data: list[float]) -> float:
    """Pure CPU work — no shared state."""
    return sum(x**2 for x in data)


def parallel_cpu_work(datasets: list[list[float]]) -> list[float]:
    """Distribute CPU-bound work across processes."""
    workers = os.cpu_count() or 1

    with ProcessPoolExecutor(max_workers=workers) as executor:
        # ProcessPoolExecutor uses pickle to pass data — keep payloads small
        return list(executor.map(cpu_intensive_task, datasets))


# Rules for ProcessPoolExecutor:
# - Functions and arguments must be picklable (top-level functions only, no lambdas)
# - No shared mutable state — each process has its own memory space
# - Overhead is high — only worth it for tasks taking > ~100ms each
# - For I/O-bound work, use ThreadPoolExecutor or asyncio instead
```

---

## Queues and Pipelines

```python
import asyncio


async def producer(queue: asyncio.Queue[str], order_ids: list[str]) -> None:
    for oid in order_ids:
        await queue.put(oid)
    # Signal consumers to stop
    for _ in range(NUM_CONSUMERS):
        await queue.put("")   # Sentinel value


async def consumer(
    queue: asyncio.Queue[str],
    results: list[str],
    worker_id: int,
) -> None:
    while True:
        order_id = await queue.get()
        if not order_id:   # Sentinel — exit
            queue.task_done()
            break
        try:
            result = await fetch_order(order_id)
            results.append(str(result["id"]))
        except Exception as exc:
            logger.error("Worker %d failed on %s: %s", worker_id, order_id, exc)
        finally:
            queue.task_done()


NUM_CONSUMERS = 4


async def run_pipeline(order_ids: list[str]) -> list[str]:
    queue:   asyncio.Queue[str] = asyncio.Queue(maxsize=20)
    results: list[str]          = []

    producer_task  = asyncio.create_task(producer(queue, order_ids))
    consumer_tasks = [
        asyncio.create_task(consumer(queue, results, i))
        for i in range(NUM_CONSUMERS)
    ]

    await asyncio.gather(producer_task, *consumer_tasks)
    await queue.join()

    return results
```

---

## Timeouts and Cancellation

```python
import asyncio


async def fetch_with_timeout(order_id: str, timeout: float = 5.0) -> dict[str, object]:
    try:
        async with asyncio.timeout(timeout):    # Python 3.11+
            return await fetch_order(order_id)
    except TimeoutError:
        logger.warning("Timeout after %.1fs fetching order %s", timeout, order_id)
        raise


async def cancellable_worker(stop_event: asyncio.Event) -> None:
    """Worker that shuts down cleanly when stop_event is set."""
    while not stop_event.is_set():
        try:
            await asyncio.wait_for(do_work(), timeout=30.0)
        except asyncio.CancelledError:
            logger.info("Worker cancelled — cleaning up")
            raise    # Always re-raise CancelledError
        except TimeoutError:
            logger.warning("Work timed out — will retry")


# Task cancellation
async def main() -> None:
    task = asyncio.create_task(long_running_work())
    await asyncio.sleep(2.0)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Task was cancelled cleanly")
```
