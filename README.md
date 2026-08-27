# dev-journey

Hi, I'm **IcedSakana** — a backend engineer focused on Java-based distributed systems.

![Java](https://img.shields.io/badge/Java-ED8B00?style=flat&logo=java&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat&logo=spring-boot&logoColor=white)
![Dubbo](https://img.shields.io/badge/Dubbo-blue?style=flat)
![Nacos](https://img.shields.io/badge/Nacos-lightblue?style=flat)
![Maven](https://img.shields.io/badge/Maven-C71A36?style=flat&logo=apache-maven&logoColor=white)

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
- [@Resource vs @DubboReference in Dubbo Projects](spring/2026-08-26_resource-vs-reference.md)

## License

[MIT](LICENSE) © 2026 IcedSakana
