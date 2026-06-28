---
title: Concurrency — Threads vs Asyncio vs Multiprocessing
parent: Systems & Performance
nav_order: 1
---

# Concurrency: Threads vs Asyncio vs Multiprocessing
`Learned: 2026-06-28` · `Reviewed: —` · `Confidence: 🟢`

## ⚡ 5-minute regain

> **One question decides everything: is the task waiting (I/O-bound) or computing (CPU-bound)?**
>
> - **Waiting** (disk, network, DB) → use **threads** for moderate scale, **asyncio** for massive scale. A waiting worker does no work — the OS/hardware does the waiting and the GIL is released meanwhile — so you only need enough workers to keep the waits overlapping.
> - **Computing** (heavy pure-Python math) → use **multiprocessing**. The GIL serializes Python execution, so the only way to run Python on many cores at once is many *processes*.
>
> Threads cost a lot *per unit* but nothing in *effort* (normal blocking code). Asyncio costs nothing per unit but a lot in effort (async-native libraries, `await` everywhere, fragile). Multiprocessing buys real parallelism at the cost of process startup and data-copying overhead.

---

## The one decision that drives everything

Before picking a tool, classify the task:

- **I/O-bound** — most of the time is spent *waiting* for something external: reading files, network requests, database queries. The CPU is idle during the wait.
- **CPU-bound** — most of the time is spent *computing* in Python: tight loops, parsing, math that stays in Python bytecode.

A second idea that dissolves most confusion: **concurrency is the number of operations in flight, not the number of workers running them.** With threads those two numbers are forced equal (one OS thread per in-flight op). With asyncio they are decoupled (one tiny task per op, all multiplexed by a single thread). That decoupling is the whole reason one asyncio thread can outscale a thread pool.

---

## The GIL, precisely

The Global Interpreter Lock (GIL) allows only **one thread to execute Python bytecode at a time** inside a single process. This is the fact that makes threads useless for CPU-bound Python work — and the fact people *misread* into thinking threads are useless for everything.

The key nuance: **the GIL only governs running Python bytecode.** It is *released* in two situations:

1. **During blocking I/O.** When a thread waits on disk or network, it is parked by the OS, executing no bytecode — so the GIL is handed back to other threads.
2. **Inside many C extensions.** Libraries like Pillow (image codecs), NumPy (BLAS linear algebra), and OpenCV drop the GIL while doing heavy C-level work.

So threads overlap **one thread's wait (or C-level work) with another thread's Python work** — never two threads' Python work simultaneously. That is exactly enough to win on I/O-bound tasks, and exactly not enough to win on pure-Python CPU tasks.

> **Caveat:** Experimental free-threaded CPython builds (PEP 703) remove the GIL and change this picture. Default CPython still has the GIL — assume the GIL unless you deliberately installed a free-threaded build.

---

## Multithreading

### When to reach for it
I/O-bound work at **moderate** concurrency (dozens to a few hundred operations) — *and* when you want to keep using ordinary blocking libraries without rewriting anything.

### Mental model
One waiter per table. Each waiter takes an order and then stands idle while the customers eat. Fine for ten tables; wasteful and crowded at sixty, because you are paying sixty salaries for work that is mostly standing around.

### How it actually works
A thread that issues a blocking read does not perform the waiting itself — it hands the request to the OS kernel, which fetches the bytes and **puts the thread to sleep** until they arrive. Multiple sleeping threads mean multiple reads in flight at once, so the *waits stack on top of each other* instead of happening one after another. The OS schedules threads **preemptively**, so one badly-behaved thread cannot freeze the others. The price is that each thread is heavy: a stack of memory, a kernel scheduling entry, and a context-switch cost every time the OS swaps them. That weight is why threads top out in the hundreds-to-low-thousands.

### Worked example
```python
from concurrent.futures import ThreadPoolExecutor
import time

def fake_io(_):
    time.sleep(0.05)        # stands in for a blocking read (releases the GIL)
    return 1

with ThreadPoolExecutor(max_workers=16) as ex:
    results = list(ex.map(fake_io, range(200)))
# ~0.7s threaded vs ~10s sequential — the waits got overlapped, not sped up
```

### Gotcha
**More threads is not always faster.** Throughput rises, **plateaus** once the underlying resource saturates, then **degrades** as contention piles up. The degradation usually comes from the storage device hitting its IOPS/throughput ceiling, a connection/auth storm at startup, memory pressure, or pure context-switch overhead. Sweep the count (10, 20, 30, 40) and pick the floor of the curve.

---

## Asyncio

### When to reach for it
I/O-bound work at **massive** concurrency (thousands to tens of thousands of operations), especially **network** I/O — *and* when async-native libraries exist for your task.

### Mental model
One excellent waiter for the whole floor. He takes table 1's order and, instead of standing there, immediately moves to table 2, then table 3. When any table's food is ready or a hand goes up, he handles it. One waiter covers fifty tables because the tables spend almost all their time not needing him — the eating (the waiting) needs no waiter.

### How it actually works
Asyncio runs a single thread with one **event loop**. Each in-flight operation is a cheap coroutine/task (a few KB in user-space memory), not an OS thread. The loop hands the kernel a list of things it cares about via an OS readiness facility (`epoll` on Linux, `kqueue` on macOS, IOCP on Windows) and sleeps until **any** of them is ready. One thread is plenty precisely because the operations are ~99% waiting: at any instant almost everything is parked, so there is only a trickle of "ready now" events to service. Scheduling is **cooperative** — each task yields control at every `await` — which is why adding the ten-thousandth coroutine is nearly free, but also why one blocking call with no `await` freezes the entire loop.

### Worked example
```python
import asyncio

async def fake_io(_):
    await asyncio.sleep(0.05)   # yields to the loop while "waiting"
    return 1

async def main():
    tasks = [fake_io(i) for i in range(10_000)]
    return await asyncio.gather(*tasks)

asyncio.run(main())
# 10,000 concurrent "reads" on ONE thread — infeasible with 10,000 OS threads
```

### Gotchas
- Requires **async-native libraries**: `aiohttp`/`httpx` instead of `requests`, async DB drivers, etc. Many libraries have no async version.
- `async`/`await` **spreads** through the codebase — a normal function can't directly call an async one (the "colored function" problem).
- **One blocking call freezes everything** — there is no other thread and no OS preemption to save you.
- **No real benefit for local-file I/O.** OSes lack good cross-platform async file I/O, so libraries like `aiofiles` just run the reads in a thread pool under the hood — you'd get the same threads with more complexity. Asyncio's edge is sockets/network, not disk.

---

## Multiprocessing

### When to reach for it
**CPU-bound** work — heavy computation that stays in Python and would otherwise be serialized by the GIL.

### Mental model
Hire more whole workers, each in their own building. Every process gets its own Python interpreter and its own GIL, so they genuinely run on separate cores at the same time. The catch: getting work and results between buildings takes real effort (packing and shipping).

### How it actually works
`multiprocessing` (and `ProcessPoolExecutor`) spawn separate OS processes. Separate processes mean **separate GILs**, which means true parallel execution of Python code across cores — the one thing threads can't give you. The costs are process **startup time** and **inter-process communication**: arguments and return values are serialized (pickled) and copied between processes, so passing large objects back and forth can erase the gains. Best for chunky, independent, compute-heavy units of work. This is also exactly how PyTorch's `DataLoader(num_workers=N)` works — N independent worker processes, each typically pinned to one thread (`set_num_threads(1)`) to avoid oversubscription.

### Worked example
```python
from concurrent.futures import ProcessPoolExecutor

def cpu_work(n):
    total = 0
    for i in range(n):
        total += i * i      # pure-Python CPU work — GIL-bound in threads
    return total

if __name__ == "__main__":          # this guard is REQUIRED for multiprocessing
    with ProcessPoolExecutor(max_workers=8) as ex:
        results = list(ex.map(cpu_work, [5_000_000] * 8))
# Threads ≈ sequential here; processes ≈ several times faster
```

### Gotcha
Process startup and data-pickling overhead are not free. For small/cheap tasks the overhead can make multiprocessing *slower* than a plain loop. Use it when each unit of work is genuinely compute-heavy, and minimize the data shipped across the process boundary.

---

## Cheat sheet

| Workload | Concurrency | Tool | Why |
|----------|-------------|------|-----|
| I/O-bound | moderate (10s–100s) | **Threads** | overlaps waits; keeps normal blocking code; simple |
| I/O-bound | massive (1000s+), network | **Asyncio** | tiny tasks, one event loop, scales far past threads |
| CPU-bound | any | **Multiprocessing** | separate GILs = real parallelism across cores |

**Decision rule in one line:** I/O + moderate → threads · I/O + huge/network → asyncio · CPU-bound → multiprocessing.

---

## Case study: EXIF extraction from 2,500 images

A real run that ties it together. Extracting a few EXIF keys per image:

- **Sequential: ~40 min.** Almost all of that was idle waiting on disk reads.
- **16–30 threads: ~4 min.** The 10× gain *is* the waiting, overlapped. (If there were truly no waiting, threading could not have changed anything.)
- **45–60 threads: slower, stalled at startup.** Past ~30 the disk's IOPS saturated; extra threads only added contention and a connection/queue pile-up at the start.

Lessons applied:
- It's local-file I/O → **threads, not asyncio** (asyncio gives no edge here).
- 2,500 files is moderate → a small pool (8–16) is plenty; don't over-spawn.
- **Read only the header.** EXIF lives at the start of a JPEG, so use `Image.open(path).getexif()`, which reads metadata without decoding pixels. Cutting I/O volume beats tuning thread count.
- If using OpenCV, set `cv2.setNumThreads(0/1)` so its internal threading doesn't oversubscribe on top of your pool.

```python
from concurrent.futures import ThreadPoolExecutor
from PIL import Image
import pandas as pd

def get_exif(path):
    exif = Image.open(path).getexif()      # reads metadata, doesn't decode pixels
    return {"filename": path, "DateTime": exif.get(306), "Model": exif.get(272)}

with ThreadPoolExecutor(max_workers=16) as ex:
    rows = list(ex.map(get_exif, paths))

df = pd.DataFrame(rows)                      # build the DataFrame once, in the main thread
```

---

## Experiments worth running

Three tiny scripts that each isolate one lesson (all standard-library):

1. **Threads overlap waiting.** `time.sleep` in a `ThreadPoolExecutor` vs a sequential loop → ~10× drop. Proves the time was waiting, not computing.
2. **The GIL wall.** A pure-Python CPU loop run sequentially vs `ThreadPoolExecutor` vs `ProcessPoolExecutor` → threads ≈ sequential (sometimes worse), processes much faster. Proves the GIL serializes Python and only processes parallelize it.
3. **The saturation curve.** Sweep `max_workers` over [1, 2, 4, 8, 16, 32, 64, 128] on a fixed workload → time halves, then plateaus. Point the same sweep at the real files to also see it bend *back up*.

---

## Self-check (can I still answer these?)

<details markdown="block">
<summary>1. If the GIL lets only one thread run Python at a time, how did threads cut EXIF time from 40 min to 4 min?</summary>

Because the bottleneck was *waiting on disk*, not running Python. A waiting thread releases the GIL and is parked by the OS, so other threads issue their reads meanwhile — the waits overlap. The GIL never blocks parallel *waiting*, only parallel Python execution.
</details>

<details markdown="block">
<summary>2. Why did 60 threads run slower than 30?</summary>

The disk's throughput/IOPS saturated around 30 concurrent reads. Past that, extra threads can't get more data through; they only add context-switch overhead, memory pressure, and a startup connection/queue storm — so throughput plateaus then degrades.
</details>

<details markdown="block">
<summary>3. Why does asyncio give no real speedup for reading local files?</summary>

OSes lack good cross-platform async file I/O, so async file libraries (`aiofiles`) run the reads in a background thread pool anyway. You'd write more complex code to get the same threads. Asyncio's advantage is network/socket I/O, not disk.
</details>

<details markdown="block">
<summary>4. A function does heavy NumPy matrix math on 1,000 arrays and it's slow. Threads, asyncio, or multiprocessing?</summary>

Trick question — check where the time goes. Heavy NumPy linear algebra runs in BLAS C code that *releases the GIL*, so threads can actually parallelize it. If the heavy work were pure-Python loops instead, it'd be GIL-bound and you'd want multiprocessing. Asyncio is wrong either way (it's not I/O-bound).
</details>

<details markdown="block">
<summary>5. What single property of a task decides which tool to reach for first?</summary>

Whether it's I/O-bound (waiting) or CPU-bound (computing). Waiting → threads/asyncio. Computing → multiprocessing.
</details>

> If I can answer these without scrolling up, I'm back to working knowledge.

---

## Related pages
- Profiling Python — finding whether a task is I/O-bound or CPU-bound before choosing a tool.
- (PyTorch DataLoader internals — multiprocessing + per-worker single-threading in practice.)

## Go deeper (only when 5 min isn't enough)
- Run the three experiments above and watch the numbers.
- `concurrent.futures` docs — the unified `ThreadPoolExecutor` / `ProcessPoolExecutor` API.
- Python `asyncio` docs — event loop, tasks, `gather`.
