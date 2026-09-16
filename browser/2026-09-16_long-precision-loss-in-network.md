---
date: 2026-09-16
title: "Long Precision Loss in Browser"
tags: [java, javascript, browser, precision]
---

# Long Precision Loss in Browser

## Background

I hit this while using an internal web tool. An 18-digit UID looked correct in F12 → Network, but the page showed a different number. Then I typed another UID into a form, and the request that actually reached the backend was already wrong.

Same UID, last few digits changed. Nothing threw.

## Problem

Two directions, same corruption.

**Viewing.** Backend returned the right value. Network → Response still had it. The page did not.

```json
{ "uid": 123456789012345678 }
```

```javascript
JSON.parse('{"uid":123456789012345678}').uid
// 123456789012345680
```

**Inputting.** I typed `123123123123123123`. The backend received `123123123123123120`.

```javascript
JSON.stringify({ uid: Number("123123123123123123") })
// {"uid":123123123123123120}
```

F12 → Network → Response shows raw HTTP text, so the first case still looks correct there. The value is already broken the moment JavaScript turns it into a `Number` — on the way in via `JSON.parse`, or on the way out via `JSON.stringify`.

## Cause

JavaScript `Number` is IEEE 754 double precision. Integers are only safe up to:

```
Number.MAX_SAFE_INTEGER = 9007199254740991  // 2^53 - 1
```

An 18-digit UID is far past that. Around this magnitude, the representable integers are 16 apart. The original number cannot be stored, so it snaps to the nearest value the mantissa can hold.

| Direction | Original | What JS actually uses |
|---|---|---|
| View | `123456789012345678` | `123456789012345680` |
| Input | `123123123123123123` | `123123123123123120` |

The view case: `...5678` is 2 away from `...5680` and 14 away from `...5664`, so it snaps to `...5680`. That is the number on the page.

The input case: `...3123` is 3 away from `...3120` and 13 away from `...3136`, so it snaps down to `...3120` before the request is even sent.

`type="number"` + `valueAsNumber`, `Number(uid)`, or a form schema typed as `number` all take the same path.

## Solution

Treat IDs as identifiers, not quantities. Send them as strings.

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer longToString() {
    return builder -> builder
        .serializerByType(Long.class, ToStringSerializer.instance)
        .serializerByType(Long.TYPE, ToStringSerializer.instance);
}
```

Or per field:

```java
@JsonSerialize(using = ToStringSerializer.class)
private Long uid;
```

JSON becomes:

```json
{ "uid": "123456789012345678" }
```

On the frontend, keep it a string all the way: input as text, request body as string, never `Number(uid)`. If the backend cannot change yet, parse with `json-bigint` / `lossless-json` instead of default `JSON.parse`.

## Reflection

- The loss is silent. Nothing throws. The last digits just change.
- Viewing and inputting look like two bugs. They are one: JSON number → JS `Number`.
- F12 Network → Response is the raw payload. Preview and `res.json()` are already parsed.
- A `long` UID is an identifier. The day it crosses the browser, it should already be a string.
