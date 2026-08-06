## 1. Executor는 절대 기본값을 쓰지 않는다

`@EnableAsync`만 켜고 Executor 빈을 정의하지 않으면, Spring은 요청마다 새 스레드를 만드는
`SimpleAsyncTaskExecutor`를 기본으로 사용합니다. 풀링이 없다는 뜻이므로, 트래픽이 몰리면
스레드가 무한정 생성되다가 결국 OOM으로 서버가 죽습니다.

```java
@Bean("apiExecutor")
public ThreadPoolTaskExecutor apiExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(500); // Integer.MAX_VALUE 금지
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("api-async-");
    executor.initialize();
    return executor;
}
```

**핵심**: `queueCapacity`를 무한대로 두면 `maxPoolSize`는 절대 도달할 일이 없어 사실상 무의미해지고, 큐에 쌓인 작업만큼 메모리가 계속 증가합니다.

---

## 2. `@Async`는 프록시 기반이다 — 4가지 함정

Spring AOP 프록시를 거쳐야 비동기가 동작합니다. 아래 케이스는 전부 컴파일은 되지만 **조용히 동기로 실행**됩니다.

```java
@Service
public class BadExample {
    public void order() {
        this.sendMail();     // ❌ self-invocation, 프록시 안 거침
    }
    @Async public void sendMail() { }

    @Async private void internal() { }        // ❌ private 오버라이드 불가
    @Async public static void statics() { }   // ❌ static
}
```

**해결책**: 다른 빈으로 분리하거나 이벤트로 분리합니다.

```java
@Component
class OrderEventHandler {
    @Async("eventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        notificationService.send(event.orderId());
    }
}
```

---

## 3. 트랜잭션은 스레드를 넘지 못한다

`@Transactional`은 `ThreadLocal`에 커넥션을 보관합니다. 비동기 스레드는 부모의 커밋 여부와 무관하게
자기 커넥션으로 별도 조회를 하기 때문에, "방금 저장했는데 조회하면 없다"는 미스터리 버그가 생깁니다.

```java
// ❌ 위험한 패턴
@Transactional
public void order(OrderCommand cmd) {
    Order order = orderRepository.save(Order.from(cmd));
    notificationService.notifyAsync(order); // 커밋 전에 비동기 스레드가 조회 시도
}

// ✅ 안전한 패턴
@Transactional
public void order(OrderCommand cmd) {
    Order order = orderRepository.save(Order.from(cmd));
    publisher.publishEvent(new OrderCreatedEvent(order.getId())); // AFTER_COMMIT에서 처리
}
```

**규칙 4가지**
1. 비동기 실행은 `AFTER_COMMIT` 시점 이후로 미룬다.
2. 엔티티를 넘기지 않고 ID/DTO를 넘긴다 (지연 로딩 시 `LazyInitializationException` 방지).
3. 비동기 안에서 DB를 쓴다면 그 메서드에 `@Transactional`을 새로 선언한다.
4. 부모 트랜잭션이 롤백돼도 자식 비동기 작업은 롤백되지 않는다 → 보상 트랜잭션 설계 필요.

---

## 4. 예외는 조용히 사라진다

반환형이 `void`인 비동기 메서드는 예외가 발생해도 호출자가 알 방법이 없습니다.

```java
@Override
public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) ->
        log.error("[Async Error] method={}, params={}", method.getName(), params, ex);
}
```

결과가 필요한 작업은 `CompletableFuture`를 반환해서, `join()`/`get()` 시점에 `CompletionException`으로
예외를 받을 수 있게 만듭니다. void + 핸들러 미등록 조합은 "장애가 났는데 로그도 없는" 최악의 상황을 만듭니다.

---

## 5. ThreadLocal 컨텍스트(MDC, SecurityContext)는 자동으로 전파되지 않는다

스레드가 바뀌면 `traceId`, 로그인 사용자 정보가 사라집니다. 더 심각한 건, 스레드풀이 스레드를
재사용하기 때문에 `finally`에서 정리하지 않으면 **다른 사용자의 인증 정보가 다음 작업에 섞이는 보안 사고**로 이어집니다.

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> parentMdc = MDC.getCopyOfContextMap();
        SecurityContext parentCtx = SecurityContextHolder.getContext();
        return () -> {
            try {
                if (parentMdc != null) MDC.setContextMap(parentMdc);
                SecurityContextHolder.setContext(parentCtx);
                runnable.run();
            } finally {
                MDC.clear();                       // 필수: 반납 전 정리
                SecurityContextHolder.clearContext();
            }
        };
    }
}
```

Executor 빈에 `executor.setTaskDecorator(new MdcTaskDecorator())`로 등록해야 실제로 적용됩니다.

---

## 6. 모든 외부 호출에는 타임아웃을 건다

타임아웃이 없는 비동기 호출은 "느린 게 아니라 영원히 응답 없는" 상태로 스레드를 계속 점유시킵니다.

```java
CompletableFuture<Product> productF =
    CompletableFuture.supplyAsync(() -> productClient.find(id), executor)
                      .orTimeout(500, TimeUnit.MILLISECONDS);

CompletableFuture<List<Review>> reviewF =
    CompletableFuture.supplyAsync(() -> reviewClient.findByProduct(id), executor)
                      .completeOnTimeout(List.of(), 300, TimeUnit.MILLISECONDS)
                      .exceptionally(e -> List.of()); // 부가 정보는 실패 허용
```

핵심 API의 타임아웃은 `orTimeout` + 예외 전파, 있어도 없어도 되는 부가 정보는 `completeOnTimeout` + 기본값 반환으로 구분합니다.

---

## 7. `CompletableFuture`에는 항상 Executor를 명시한다

Executor를 넘기지 않으면 `ForkJoinPool.commonPool()`(코어 수 - 1개 스레드)을 씁니다. 여기서
블로킹 I/O를 하면 `parallelStream()` 등 애플리케이션 전체가 이 풀을 공유하다가 함께 멈출 수 있습니다.

```java
// ❌ commonPool 사용 (위험)
CompletableFuture.supplyAsync(() -> productClient.find(id));

// ✅ 전용 executor 명시
CompletableFuture.supplyAsync(() -> productClient.find(id), apiExecutor);
```

---

## 8. Graceful Shutdown을 반드시 설정한다

배포/재시작 시 진행 중인 비동기 작업이 강제 종료되면 데이터 유실이 발생합니다.

```java
executor.setWaitForTasksToCompleteOnShutdown(true);
executor.setAwaitTerminationSeconds(30);
```

---

## 9. `@Async`는 "유실 가능한" 비동기라는 것을 항상 인지한다

`@Async`는 애플리케이션 프로세스 메모리 안에서만 동작합니다. 큐에 대기 중이던 작업은 서버가
죽는 순간 그대로 사라집니다. 알림/로그처럼 유실돼도 괜찮은 작업은 `@Async`로 충분하지만,
결제 확정 후처리, 재고 차감처럼 유실되면 안 되는 작업은 반드시 Kafka나 Outbox 패턴 같은
외부 브로커 기반으로 옮겨야 합니다.

```
유실돼도 되는 작업 → @Async + 전용 풀
유실되면 안 되는 작업 → Kafka 발행 / Outbox 패턴
```

---

## 최종 체크리스트 (커밋 전 5초 점검)

- [ ] Executor 빈을 직접 만들었고, `queueCapacity`가 유한한가?
- [ ] self-invocation / private / static 함정에 걸리지 않았는가?
- [ ] 비동기 실행이 트랜잭션 커밋 이후로 보장되는가?
- [ ] void 비동기 메서드에 예외 핸들러가 연결되어 있는가?
- [ ] MDC/SecurityContext가 TaskDecorator로 전파·정리되는가?
- [ ] 외부 호출에 타임아웃이 걸려 있는가?
- [ ] `supplyAsync`에 executor를 직접 넘겼는가?
- [ ] 이 작업이 유실되면 안 되는 작업인지 판단했는가?
```
