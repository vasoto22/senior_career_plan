# Logging Guidelines

## Overview

All logging in this project must go through the centralized logger at
`lib/core/utils/logger/logger.dart`. The logger provides:

- **Zero overhead in production** — all log calls are no-ops in release builds (`kReleaseMode` guard).
- **Automatic sensitive data sanitization** — messages and objects are passed through `SensitiveDataSanitizer` before being written to the console or DevTools.
- **Consistent formatting** — ANSI colors, aligned headers, and level symbols.

---

## Log Level Usage Guide

Use the narrowest level that accurately describes the event.

| Function | Level | When to use |
|---|---|---|
| `logDebugFinest(h, msg)` | debugFinest | Extremely verbose trace: loop iterations, micro-state transitions |
| `logDebugFiner(h, msg)` | debugFiner | Internal step details: intermediate values, sub-operation entry/exit |
| `logDebugFine(h, msg)` | debugFine | General development trace: cache hits, non-critical state changes |
| `logDebug(h, msg)` | debug | Default debug output: feature flow, incoming events |
| `logInfo(h, msg)` | info | Noteworthy non-error events: navigation, service initialization |
| `logSuccess(h, msg)` | success | Explicitly positive outcomes: login completed, data saved |
| `logWarning(h, msg)` | warning | Recoverable issues: retry triggered, fallback used, deprecated call |
| `logError(h, msg)` | error | Handled errors that degrade the experience: API failure, parse error |
| `logErrorObject(h, e, msg)` | error | Same as above, plus attaches the exception object for stack trace |
| `logFatal(h, e, msg)` | fatal | Unrecoverable errors: crash context before re-throw or Crashlytics report |

**Timing helpers** (`logTimerStart` / `logTimerStop`) and the build helper (`logBuild`)
use `debugFinest` by default; override via the `level:` parameter when needed.

---

## What NOT to Log

The following data categories are **strictly prohibited** in log messages,
even in debug builds. The sanitizer will mask most patterns automatically,
but the best protection is not logging sensitive data in the first place.

| Category | Examples |
|---|---|
| Auth tokens & credentials | JWT, Bearer tokens, API keys, passwords, session IDs |
| Personal data — identity | Full names, email addresses, ID numbers |
| Personal data — contact | Phone numbers, physical addresses |
| GPS / location | Latitude and longitude values, raw location objects |
| Full HTTP response bodies | API response payloads, server error bodies |
| Financial data | Card numbers, account numbers |

### Bad example
```dart
// ❌ Logs a full user object that contains email + phone
logDebug('Profile', 'User loaded: ${user.toJson()}');

// ❌ Logs a URL with an API key query param
logInfo('HTTP', 'Calling $url?apiKey=$key');
```

### Good example
```dart
// ✅ Log only non-sensitive identifiers
logDebug('Profile', 'User loaded: id=${user.id}');

// ✅ Use sanitizeUrl() explicitly when you must log a URL
logInfo('HTTP', 'Calling ${SensitiveDataSanitizer.sanitizeUrl(url)}');
```

---

## Sensitive Data Sanitizer

`SensitiveDataSanitizer` (at `lib/core/utils/logger/domain/sanitizer/sensitive_data_sanitizer.dart`)
is applied **automatically** inside `__log`, so console/DevTools output is always sanitized.
It is also applied to `url` and `page` fields sent to Firebase Crashlytics.

For complex objects passed to `logErrorObject` or `logFatal`, the sanitizer runs
on `object.toString()`. If this conversion would produce a large payload with sensitive
fields, **build a minimal custom string** before logging:

```dart
// ✅ Prefer targeted fields over dumping entire objects
logErrorObject(
  'Repo',
  AppException(statusCode: e.statusCode, errorCode: ErrorCodeEnum.genericError),
  'Failed to load vehicle list',
);
```

### Manual sanitization

When you need to sanitize explicitly (e.g. before passing to a third-party service):

```dart
final clean = SensitiveDataSanitizer.sanitize(rawMessage);
final cleanUrl = SensitiveDataSanitizer.sanitizeUrl(url);
final cleanObj = SensitiveDataSanitizer.sanitizeObject(responseBody);
```

---

## Prohibited Patterns

| Pattern | Why | Alternative |
|---|---|---|
| `print(...)` | Leaks in release builds; no level or context | `logDebug` / `logError` |
| `debugPrint(...)` | Not sanitized; no structured level | `logDebug` / `logError` |
| `dart.log(...)` directly | Bypasses sanitizer | `logInfo` / `logDebug` |
| Logging `user.toJson()` | Exposes PII | Log only `user.id` |
| Logging full `DioException.response` | Exposes HTTP body | Log `statusCode` + `errorCode` only |

`print()` is enforced as an **error** by the `avoid_print` analyzer rule.  
`debugPrint()` is enforced as a **warning** by the custom `avoid_debug_print` rule
(defined in `lib/core/packages/satrack_linters`).

> **Migration note**: existing `debugPrint()` calls in the codebase are tracked for
> removal in a follow-up task. Do not add new ones.

---

## Header Conventions

The first argument (`h`) to every log function is the header — a short, consistent
label that identifies the source of the log. Use PascalCase, max ~15 chars.

```dart
logInfo('VehicleMap', 'Stream connected');          // ✅
logError('VehicleCubit', 'Failed to load list');    // ✅
logDebug('plat/emitter', 'Socket reconnected');     // ✅
logDebug('this is way too long for a header', ...); // ❌
```

---

## Crashlytics Reporting

Only report to Crashlytics via `LogManagerService.reportError()` or
`LogManagerService.reportErrorHttp()`. Never call `FirebaseCrashlytics` directly
from feature code.

The `CrashlyticsServiceImpl` automatically:
- Sanitizes `url` and `page` fields via `SensitiveDataSanitizer.sanitizeUrl()`.
- Suppresses all reporting outside release mode (`kReleaseMode` guard).
- Sets structured custom keys (`statusCode`, `errorCode`, `method`, etc.).

---

## Quick Reference

```dart
// Trace / debug
logDebugFinest('Render',  'Building vehicle card');
logDebug('VehicleRepo',   'Cache hit for id=$id');

// Info / success
logInfo('Bootstrap',      'Firebase initialized');
logSuccess('Auth',        'Login completed');

// Warnings / errors
logWarning('Location',    'GPS accuracy below threshold');
logError('HTTP',          'Request failed: status=$statusCode');
logErrorObject('Cubit',   e, 'Unhandled exception in loadVehicles');

// Fatal (pre-crash context)
logFatal('Bootstrap',     e, 'Unrecoverable error during app startup');
```
