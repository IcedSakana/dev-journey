---
date: 2026-08-26
title: "@Resource vs @Reference in Dubbo Projects"
tags: [java, spring-boot, dubbo, dependency-injection]
---

# @Resource vs @Reference in Dubbo Projects

## Background

In a typical Dubbo-based microservice project, the service provider exposes interfaces via Dubbo's RPC mechanism, while consumers inject those remote services into their Spring beans. Two annotations are commonly used for injection: Spring's `@Resource` and Dubbo's `@Reference` (or `@DubboReference` in newer versions). Confusing the two is a frequent source of bugs.

## Problem

Given a service interface published by a Dubbo provider:

```java
// Shared API module
public interface UserService {
    User findById(long id);
}
```

A developer on the consumer side writes:

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Resource           // ← Spring annotation, looks up a local bean
    private UserService userService;

    @Override
    public Order createOrder(long userId) {
        User user = userService.findById(userId);
        // ...
    }
}
```

At startup the application fails with:

```
NoSuchBeanDefinitionException: No qualifying bean of type 'com.example.api.UserService' available
```

The developer expected a remote Dubbo proxy but used the wrong annotation.

## Solution

### When to use `@Resource`

`@Resource` is a standard Java EE / Jakarta EE annotation (also supported by Spring). It performs **local Spring bean lookup** — by name first, then by type.

Use it when injecting a bean that lives in the **same** Spring application context:

```java
@Service
public class NotificationService {

    @Resource                           // local bean — correct
    private MailSender mailSender;
}
```

### When to use `@DubboReference` (`@Reference`)

`@DubboReference` (the modern form of the older `@Reference`) creates a **Dubbo proxy** for a remote service registered in the registry (e.g., Nacos, Zookeeper). The proxy handles serialization, load balancing, and fault tolerance transparently.

```java
import org.apache.dubbo.config.annotation.DubboReference;

@Service
public class OrderServiceImpl implements OrderService {

    @DubboReference(version = "1.0.0", timeout = 3000)
    private UserService userService;    // remote Dubbo proxy — correct

    @Override
    public Order createOrder(long userId) {
        User user = userService.findById(userId);
        // ...
    }
}
```

### Key configuration (consumer side)

```yaml
# application.yml
dubbo:
  application:
    name: order-service
  registry:
    address: nacos://127.0.0.1:8848
  consumer:
    timeout: 5000
    retries: 2
```

### Side-by-side comparison

| Feature | `@Resource` | `@DubboReference` |
|---|---|---|
| Origin | Java EE / Spring | Apache Dubbo |
| Lookup target | Local Spring bean | Remote Dubbo service |
| Proxy type | Direct bean reference | RPC proxy |
| Failure mode | `NoSuchBeanDefinitionException` | `RpcException` (network/timeout) |
| Typical use | Internal dependencies | Cross-service calls |

## Reflection

- Always distinguish between **in-process** (Spring bean) and **out-of-process** (Dubbo RPC) dependencies. The annotation you pick determines where the call goes.
- If you accidentally use `@Resource` on a Dubbo interface, Spring will fail at startup because there is no local bean — you get a fast, obvious failure. If you accidentally use `@DubboReference` on a local bean, Dubbo will try to find a provider in the registry, which may cause a slower, harder-to-diagnose timeout at runtime.
- In projects that mix Spring and Dubbo annotations heavily, a code review checklist item — "confirm `@Resource` vs `@DubboReference`" — is worth adding to avoid this class of bug entirely.
