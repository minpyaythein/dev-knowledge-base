# Smoothing a bursty LLM token stream needs a reader thread, not a pull-based throttle

A generator that throttles by pulling from the upstream iterator cannot smooth network stalls — during a stall it has nothing to yield, and the burst that follows gets dumped as one slam. Decouple with a reader thread that drains the network at full speed into a queue, while the display side drips the buffer out on a fixed cadence.

## Why it matters

In production, SSE token streams often arrive in macro-bursts (measured on one deploy: ~95% of chunks back-to-back, the whole answer landing in ~7 clumps separated by 1–1.7 s stalls) even when the provider emits sub-word tokens smoothly — intermediaries batch them. Painting on arrival then slams hundreds of characters per paint. A naive `time.sleep`-based throttle in the same thread only reshapes the flood; it can't spread a burst across the stall that *follows* it, because it can't see the future gap.

## How it works

- A daemon **reader thread** consumes the network iterator flat-out into a `queue.Queue`, with a `None` end-of-stream sentinel. Exceptions are put on the queue too and re-raised on the consumer side, so caller error handling is unchanged.
- The consumer generator treats the queued text as a **reservoir**: every `interval` (e.g. 50 ms ≈ 20 paints/sec) it yields a slice sized to spread the current buffer over a target drain time (e.g. 1.5 s — bridging the observed stalls).
- Drain is **exponential and self-tuning**: quantum ≈ `len(buffer) * interval / drain_seconds`, floored (so the tail doesn't crawl) and capped (so a backlog catch-up never visibly slams). Speed the drain up ~3× once the stream has ended.
- First yield happens the moment the first token arrives — time-to-first-token is unchanged.

## Example

Streamlit pipeline: `st.write_stream(capture_answer(spinner_wrapper(pace(chain.stream(q)))))`. `pace()` wraps the **raw** network stream because its reader thread must only touch the iterator — anything needing the script thread's context (spinners, `st.session_state` writes) stays outside, consuming the paced output.

## Gotchas

- Measure before fixing: instrument chunk inter-arrival times in prod first. In dev the arrival was smooth and pacing is a near-no-op — the burstiness only existed on the deployed host.
- If the run is interrupted (a Stop button), the abandoned daemon reader just finishes draining the already-billed stream and exits — harmless, but know it happens.
