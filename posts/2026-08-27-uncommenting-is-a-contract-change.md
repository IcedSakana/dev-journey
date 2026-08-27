---
title: "Uncommenting code is a contract change"
date: 2026-08-27
lang: en
status: published
kind: incident
topics: [api-compatibility, regression-testing, rpc, hotfix]
---

I watched a Java backend engineer walk through a two-step outage. The first failure was easy to moralize. The second one is the one that stuck, because I have made that shape of mistake myself.

## What happened

He restored a block of code that had been commented out.

That path sat on a live RPC surface. Once it was active again, other services started hitting it. Downstream was not compatible with the revived behavior and threw NPEs.

The test environment did not save them. Traffic there is thin, so the newly opened path barely ran. Production had the callers that mattered. That is when it showed up.

The fix was to tighten the path — narrow what the method would accept or return, close the hole that had just been opened. The change went out as an emergency, so it skipped a full client-side regression. The tightened contract was too tight. After that deploy, the app's main flow locked up.

Two production failures. Same interface. Opposite directions: first too open, then too closed.

## Why the first miss is unsurprising

Commented-out code on a published method is not dead code. It is a latent behavior change sitting on a contract other teams already depend on.

Uncommenting it does not look like an API change. The method name is the same. The signature is the same. Reviewers see "restoring previous logic" and treat it as cleanup. Callers do not get a new version, a new path, or a chance to opt in. They just start receiving a behavior they were not exercising yesterday.

Low-traffic test environments make this worse. They tend to cover the happy path of the services you remember to call. They do not cover the long tail of upstreams that only show up in production, or the downstreams that were written against the quiet version of this method.

An NPE at the downstream is the typical signature of this class of bug: someone assumed a field, a branch, or a return shape that the revived code no longer guarantees.

## Why the second miss is more dangerous

The hotfix had a reasonable goal: stop the bleeding. Tightening the path is the instinct that follows an unexpected NPE — add a guard, reject the bad case, collapse the logic so the dangerous branch cannot run.

That instinct has a cost. "Tighter" is also a contract change. If the client main flow depended on a case you just rejected, or on a return shape you just collapsed, you have not fixed the outage. You have moved it to the place users actually live.

Emergency process makes that likely. Backend tests and a smoke of the API can look green while the app is already stuck. The only test that would have caught it is the one that got dropped: a full client-side walk through the main flow, on a build that includes the hotfix.

I have been on that side of the keyboard. The first incident feels like the problem. The second one is caused by how we chose to fix it.

That is why I still try to keep the first problem out of production. A live fix is urgent by definition. The process will not be walked in full, the patch will be written fast, and mistakes concentrate. The repair is structurally more likely to go wrong than the change that caused it.

## Two rules I will not skip again

**1. A behavior change on a user-facing flow needs a client end-to-end pass, including hotfixes.**

Service-level tests are not a substitute. If the change can alter what the app does in the main path — empty states, blocked buttons, missing fields, a call that now fails closed — run that path on the client before the change is considered done. "We had to ship" is not a reason to skip it. It is the reason the skip is expensive.

**2. Do not mutate a live API. Add a new one. If you must change the old method, inventory every caller first.**

A published method is owned by its callers, not by the team that can edit the file. The safer move is a new service or a new method: new path, new version, old contract left alone until callers migrate.

Where I work now, CI/CD rejects production API signature changes. That is the right default. It still does not cover the failure in this story. The signature never changed. The behavior did — twice.

So the rule is broader than the linter. If you need to change an old method's logic, not just its signature, use whatever service governance you have — registry consumer lists, gateway access logs, RPC admin consoles, tracing — and confirm every upstream before you ship. If the tooling cannot name the callers, you do not know the blast radius.

## What I am actually guarding against

The tempting story is "don't uncomment old code" or "don't hotfix under pressure." Those are symptoms.

The real constraint is: **the contract is behavior, not the method signature.** Restoring a commented branch, tightening a guard, and renaming a parameter are the same class of change to everyone calling you. Test environments will not enumerate those callers for you. A skipped client regression will not either.

Ship a new API when you can. When you cannot, find the callers, then walk the main flow on the client as if this were a normal release — because for the people on the other side of the interface, it is.
