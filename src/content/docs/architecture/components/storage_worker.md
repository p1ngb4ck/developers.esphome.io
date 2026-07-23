---
title: "Storage Worker"
---

The storage worker is the asynchronous execution engine that sits on top of the
[storage interface](/architecture/components/storage/). Where the interface is deliberately synchronous — every driver
call blocks until it completes — the worker chunks large operations, keeps them off the main loop where the medium
allows it, and delivers a completion callback when they finish.

If your component moves files, streams an upload to storage, or writes an image to a raw device, this is the API you
call. If it only reads a small config file, the blocking helpers on the interface are simpler and enough.

## Why it exists

Written naively against the interface, "copy this 40 MB file from SD to USB" is a call that does not return for a
minute, takes the watchdog with it, and drops the API connection. Every consumer would otherwise write the same state
machine — open handles, track an offset, move one chunk per `loop()`, close on error, bail out when the medium
disappears. The worker writes it once.

It is also what makes the interface's drain contract true. `StorageRegistry::unregister_storage()` promises that no
consumer is still mid-call by the time it returns; the registry cannot enforce that on its own, because it does not
know who is mid-call. The worker does.

## Two engines, one API

| Condition | Engine |
| --- | --- |
| ESP32, driver opted in at codegen, storage reports `STORAGE_CAP_IO_TASK_SAFE` | Shared background FreeRTOS task |
| Anything else, including every non-ESP32 platform | Loop-sliced: one chunk per scheduler tick |

Callers cannot tell which one ran their request except by timing. Both conditions must hold: a driver declares
task safety at codegen with `request_storage_worker(task_safe=True)`, and the individual storage instance must report
`STORAGE_CAP_IO_TASK_SAFE` at runtime. A task-safe driver on a platform without FreeRTOS degrades to loop-sliced
rather than failing to compile.

The distinction matters at the driver level: a driver that owns its bus exclusively can pass `task_safe=True`, while
one that shares a bus with unrelated devices cannot — its safety would depend on how the user wired the node.

## Compilation and startup

`storage_worker.h` / `.cpp` are behind `USE_STORAGE_WORKER`, which codegen only defines when at least one path-based
driver calls `request_storage_worker()` in its own `to_code()`. A node with nothing but a `RawStorage` device does not
compile any of it.

Even when compiled in, the pools and the background task are created lazily on the first submit. A driver that links
the worker but never issues a transfer pays for a hotplug subscription and nothing else.

The component is a `PollingComponent` at `setup_priority::DATA` with a 5 ms default interval. The poller is armed by
the submit funnels and disarmed once every slot is free, so an idle worker schedules nothing.

## Submitting work

All entry points share the same shape and the same rules:

```cpp
TransferJob job = 0;
StorageError err = global_storage_worker->async_copy(src, "/sd/a.txt", dst, "/usb/a.txt",
                                                     [](StorageError result) {
                                                       ESP_LOGD(TAG, "copy: %s", error_to_string(result));
                                                     },
                                                     &job);
if (err != StorageError::OK) {
  // Callback will NOT fire — handle the failure here.
}
```

**The callback fires exactly once, always on the main loop, never reentrantly from inside your own submit call.** If
the submit returns anything other than `OK`, the request was not queued and the callback will not run at all — so you
never have to ask whether it already fired.

Two rejections to expect:

- `NOT_READY` — the request pool is full. This is backpressure, not a failure of the operation; retry later.
- `INVALID_ARGS` — a path exceeds `STORAGE_PATH_MAX`. Paths are copied into fixed buffers at submit time, because your
  pointers cannot be assumed to outlive submission.

### Available operations

| Call | Notes |
| --- | --- |
| `async_copy()`, `async_move()` | Single file; move uses the same-storage `rename()` fast path |
| `async_copy_tree()`, `async_move_tree()` | Whole directory tree, walked by the engine itself |
| `async_raw_read()` | Device range to a file |
| `async_raw_write()` | File to a device, optional erase first, optional verify passes |
| `async_raw_erase()` | Sliced erase, one geometry step per pass |
| `async_raw_verify_file()` | Read back and compare against a file, no write |

`overwrite=false` answers `ALREADY_EXISTS` for an occupied destination; `true` clears it first, recursively for trees —
inside the worker, never in your context. The worker also decides tree-versus-file itself by stat'ing the source, so
callers do not pre-classify.

Tree walking deliberately lives in the engine rather than in the caller. A caller walking a tree can only step forward
when its own `loop()` runs, which would make the transfer's speed depend on main loop scheduling; on task-safe media
the worker owns the whole operation start to finish, exactly as it does for a single file. The walk is iterative and
bounded to `STORAGE_MAX_RECURSION_DEPTH`, so it does not recurse on the task stack.

## Streaming

Use the stream API when the data is driven from outside — an HTTP upload arriving chunk by chunk, or a download being
pulled by a client — rather than by the worker reading both ends itself.

```cpp
StreamHandle handle{};
global_storage_worker->begin_write(storage, "/sd/upload.bin", &handle,
                                   [](StorageError err) { /* file is open */ });
// later, once per incoming chunk:
global_storage_worker->write_chunk(handle, data, len, [](StorageError err) { /* chunk written */ });
// when done, always:
global_storage_worker->end_write(handle, [](StorageError err) { /* slot released */ });
```

Three things to get right:

- **The buffer is not copied.** `data` must stay valid until the callback fires. Streams are large and frequent;
  copying would defeat the purpose.
- **Always call `end_write()` / `end_read()`**, including after an error, or the pool slot stays claimed.
- **A stream may sit idle indefinitely** between chunks — that is normal, since the caller drives it. It is subject to
  the idle timeout below.

`read_chunk()` fills `*bytes_read` before invoking the callback; zero means EOF, not an error.

Both `TransferJob` and `StreamHandle` carry a generation that is bumped whenever a slot is claimed, so a handle held
past its lifetime stops matching and the call is refused rather than writing into a stranger's file.

## Progress

`get_transfer_status(job, &out)` returns a snapshot for progress bars and job endpoints: state, result, bytes done and
total, the coarse phase of a raw job (erase, write, verify) with the current verify pass, a short label for the file in
flight, and per-file progress alongside the overall figures.

The per-file numbers are not redundant. A tree's total is unknown without walking it twice, so `bytes_total` is 0 for
tree operations, but the file currently in flight costs one `stat()` and can still be reported honestly.

A slot is recycled after its completion callback runs, so the finished snapshot is only observable until then.
**Capture the final result in the completion callback**; use polling only for progress.

## Hotplug and teardown

The worker subscribes to registry unregistration and drains synchronously: pending requests finish immediately,
loop-engine requests are drained in place inside the callback, and task requests are cancelled and waited on with a
bounded timeout. By the time the registry call returns, no data-plane call into the removed storage is in flight.

Drivers with removable media can additionally ask before acting:

```cpp
if (global_storage_worker->is_busy_with(this)) {
  // Defer the unmount; a job is still touching this storage.
}
```

## Timeouts

A request that stops moving would otherwise hold its storages — and everything blocked behind them — indefinitely.

| Guard | Timeout | Applies to |
| --- | --- | --- |
| Stall | 30 s | A running request whose progress counter has not advanced |
| Pending cap | 120 s | A request that was never picked up |
| Stream idle | 30 s | A stream whose external driver vanished mid-flight |
| Network ready | 20 s | Window in which `NOT_READY` from a network storage is read as "still connecting" |

A whole-chip erase is the deliberate exception: it busy-waits for tens of seconds without advancing any counter, so it
shields itself from the stall watchdog and stays bounded by the driver's own ready timeout instead.

## Buffers

One streaming buffer per engine, not per request — a buffer's content never outlives the chunk call that filled it,
and neither engine runs two requests at once. Each is allocated on first use and kept, because a repeated 16–64 kB
allocate/free cycle is exactly the heap fragmentation pattern to avoid.

The two engines size theirs differently because the execution context differs: the loop path stays in internal RAM and
small enough for its slice budget, while the task path stages a larger DMA-capable PSRAM chunk on variants that have
it. `dump_config()` reports the resolved policy at boot.

## Serialization

The interface requires that data-plane calls on one storage instance are externally serialized. With two engines that
is not automatic, so the worker holds back any request or stream that would touch a storage another engine is
currently driving. Consumers do not need to do anything about this; it occasionally costs a scheduler tick of latency
and never costs correctness.

## Configuration

The worker's tunables live on the same `storage:` block: task stack size and priority, request and stream pool depth,
and the engine tick interval. See the [user documentation](https://esphome.io/components/storage/) for the details.
Pool depths are sized exactly at codegen, so there is no per-request heap allocation at runtime.

## See also

- [Storage](/architecture/components/storage/) — the interface and registry this builds on
