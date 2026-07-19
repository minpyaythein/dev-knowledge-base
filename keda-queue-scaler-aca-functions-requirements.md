# KEDA queue autoscaling for Azure Functions on Container Apps

KEDA queue scale rules only auto-generate with a system-assigned managed identity, a dedicated trigger connection name, and a fully-qualified `__queueServiceUri`.

## Why it matters

A misconfigured function still processes messages, so the setup looks healthy — it just never gets a scale rule and stays at minimum replicas.

## How it works

Three independent settings each silently break scale-rule generation on ACA (Azure Container Apps):

| Setting | Required | Why the alternative fails |
|---|---|---|
| Identity | System-assigned MI with `DefaultAzureCredential` | User-assigned MI: KEDA scaler auth is not auto-configured, so no scale rule is generated |
| Trigger connection name | A dedicated name, e.g. `ConversionQueueConnection` | Reusing host-reserved `AzureWebJobsStorage` makes ACA treat it as the host-management connection and skip scaler generation |
| Endpoint key | `<Conn>__queueServiceUri` (fully-qualified URI) | `__accountName` alone: KEDA cannot resolve the endpoint, autoscale never fires |

The identity also needs `Storage Queue Data Contributor` on the trigger queue — the scaler reads queue length with the same role the trigger uses to receive and delete messages.

## Example

```ini
ConversionQueueConnection__queueServiceUri = https://<storage-account>.queue.core.windows.net
ConversionQueueConnection__credential      = managedidentity
```

With the system-assigned MI holding `Storage Queue Data Contributor` on the queue, ACA generates the KEDA queue scaler automatically — no client ID setting needed.

## See also

- [queue-delegation-loop-prevention-pattern.md](queue-delegation-loop-prevention-pattern.md)
