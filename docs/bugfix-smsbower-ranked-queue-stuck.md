# SMSBower ranked queue acquisition appears stuck

## Symptom

After reloading the extension, SMSBower phone acquisition could appear stuck for a long time before a number was acquired or an error was shown.

## Root Cause

The ranked SMSBower acquisition path expanded the Web price payload into many single-agent candidates, then tried every candidate sequentially. Each `getNumber` request can wait up to the default request timeout, and the path did not log the Web price fetch, candidate count, or current candidate progress. When the queue was large or many agents returned `NO_NUMBERS`, the side panel looked idle even though the background worker was still trying candidates.

In the signup flow, the outer Step 2 acquisition timeout uses `Promise.race`. When the timeout won, the underlying acquisition promise was not cancelled. If that late promise eventually acquired a number, it could still write the normal "已从 ... 获取号码" log after the run had already been marked timed out/stopped.

## Fix

The SMSBower ranked queue now:

- Logs Web price queue loading.
- Logs filtered candidate count.
- Logs each attempted country / agent / rank / price candidate.
- Caps each acquisition round to the first 12 sorted candidates.
- Logs when remaining candidates are skipped to avoid a long apparent hang.
- Ignores and best-effort cancels any activation that arrives after the signup acquisition timeout, so stale background work no longer emits a misleading success log.

## Verification

Ran:

```bash
node --test tests/phone-verification-flow.test.js tests/background-account-history-settings.test.js tests/sidepanel-phone-verification-settings.test.js
git diff --check
```

Result: all 95 targeted tests passed, and diff check passed.
