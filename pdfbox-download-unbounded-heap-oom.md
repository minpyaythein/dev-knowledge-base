# Buffering PDF downloads in memory exhausts the container heap

Loading an object fully, re-buffering it through PDFBox, and never closing the `PDDocument` holds 3–4× the file size in heap and OOMs under concurrency.

## Why it matters

A download endpoint that buffers whole files works for years, then a handful of concurrent users exhaust the container heap and the task is SIGKILL'd. Because the kill leaves no `OutOfMemoryError` stack trace, the cause is easy to miss.

## How it works

Each request stacks several full copies of the file in heap, and none are released:

1. Reading the object fully into a `byte[]` — one full copy.
2. `Loader.loadPDF(data)` — `PDDocument` never closed, so its `COSDocument` and buffers stay un-GC'd.
3. `document.save(baos).toByteArray()` — a second copy of the decrypted PDF.

With no `-Xmx` set, the JVM defaults to 25% of container memory, so the ceiling is far lower than the container's total.

## Example

Close the document deterministically, spill large files to a temp scratch file, and stream the body instead of returning `byte[]`.

```java
try (PDDocument doc = Loader.loadPDF(
        data, "", null, null, IOUtils.createTempFileOnlyStreamCache())) {
    // ... process ...
}
// return an InputStreamResource over the source stream, not ResponseEntity<byte[]>
```

## Gotchas

- A SIGKILL'd container flushes no log; reconstruct the incident from the last log line plus request history, not from an exception.
- Cumulative concurrent load triggers it, not one heavy user — don't hunt a single bad actor.
- Set `-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError` so the heap uses the container memory and dies loudly.

## See also

- [spring-session-cleanup-deadlock-shedlock.md](spring-session-cleanup-deadlock-shedlock.md) — another JVM-service incident surfaced by running 2+ instances.
