# Delegating work between queue functions without loops

Keep the "needs delegation" decision in pure domain code, forward the original message verbatim from the trigger layer, and gate on a marker field so it can't loop.

## Why it matters

Re-enqueueing from deep inside a pipeline couples domain logic to messaging, and a converter that feeds its own input queue invites infinite convert-and-ingest loops.

## How it works

Pattern from a RAG ingest function that hands image-only Office files to a separate PDF-convert function:

1. The pipeline stays pure: it only returns `needs_conversion: true`, never touches queues.
2. The trigger layer that received the message forwards it verbatim (Base64, same event id) — no URL reconstruction, full traceability.
3. Delegation is gated: only when the extension needs converting AND `source_container` is unset, i.e. an unconverted original.
4. The converter sets `source_container` on re-enqueue and outputs `.pdf`, so the returning message fails the gate — no loop.
5. A successful forward counts as success for the original message, so no queue retry fires.

Splitting decision (step 1) from transmission (step 2) also lets failure policy differ per route: the bulk route re-raises for queue retry, the single-document route only logs.

## Example

```text
upload photo-only .docx
  → RAG queue → pipeline extracts no text → needs_conversion=true
  → trigger forwards original message to convert queue     (source_container unset → gate open)
  → converter: .docx → .pdf, re-enqueues with source_container=<original container>
  → RAG queue again → PDF extractor path; gate sees source_container set → stop
```

The marker field doubles as metadata: the ingest side uses it to record the original file, not the converted PDF.

## See also

- [keda-queue-scaler-aca-functions-requirements.md](keda-queue-scaler-aca-functions-requirements.md)
