# dev-journey

Field notes from day-to-day engineering: Java, Spring Boot, Dubbo, distributed systems, and whatever else shows up at work.

Not every write-up is an incident. Most entries start as debugging, gotchas, design trade-offs, or performance work. Incidents are just one `kind`.

## Layout

```text
posts/                 # finished, shareable write-ups (including English posts)
notes/                 # working notes; incomplete is fine
  java/
  spring-boot/
  dubbo/
  distributed-systems/
  engineering/         # process, collaboration, conventions — not tied to a framework
templates/
  note.md
```

Topic lives in the folder. Genre lives in frontmatter. Look up “Dubbo timeout” by path; filter “was this an outage?” with `kind`.

## Frontmatter

```yaml
---
title: "Short, specific title"
date: 2026-08-27
lang: en                 # en | zh
status: draft            # draft | published
kind: debugging          # debugging | gotcha | design | performance | incident | deep-dive
topics: [dubbo, timeout]
---
```

| `kind` | Use when |
|---|---|
| `debugging` | Hunting a problem that was not an outage |
| `gotcha` | Framework or API behavior that did not match the mental model |
| `design` | Why this option won |
| `performance` | Latency, capacity, jitter |
| `incident` | Production impact with a real blast radius |
| `deep-dive` | How a mechanism actually works |

## Naming

`YYYY-MM-DD-short-slug.md`

Copy `templates/note.md` into `notes/<topic>/` or `posts/`. Promote a note to `posts/` when it is ready to share; keep `kind` and `topics`, set `status: published`.

## Posts

- [Uncommenting code is a contract change](posts/2026-08-27-uncommenting-is-a-contract-change.md) — restoring commented logic, then over-tightening the hotfix, both broke callers.
