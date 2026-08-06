## 1. Executor는 절대 기본값을 쓰지 않는다

### 왜 위험한가
`@EnableAsync`만 켜고 Executor 빈을 별도로 정의하지 않으면, Spring은 기본적으로
`SimpleAsyncTaskExecutor`를 사용합니다. 이 이름을 잘 보면 "Simple"이라는 단어가 들어있는데,
이 Executor는 풀링(pooling)을 하지 않고 **작업이 들어올 때마다 매번 새 스레드를 만듭니다**.

### 무슨 일이 벌어지는가
평소에는 트래픽이 적어서 문제가 없다가, 갑자기 요청이 몰리면 `@Async` 메서드가 호출될 때마다
새로운 OS 스레드가 계속 생성됩니다. 스레드 하나당 기본 1MB 정도의 스택 메모리를 차지하기 때문에,
수천 개의 요청이 동시에 몰리면 수 GB의 메모리가 순식간에 소모되고 결국 `OutOfMemoryError`로
서버 전체가 죽습니다. "평소엔 잘 되던 코드가 갑자기 죽었다"는 장애의 전형적인 원인입니다.

### 어떻게 고치는가
직접 `ThreadPoolTaskExecutor`를 빈으로 등록해서, 스레드 개수와 대기 큐 크기에 명확한 한계선을 그어줍니다.

```java
@Bean("apiExecutor")
public ThreadPoolTaskExecutor apiExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);      // 평소에 유지할 스레드 수
    executor.setMaxPoolSize(50);       // 바쁠 때 늘어날 수 있는 최대 스레드 수
    executor.setQueueCapacity(500);    // 큐 크기 (절대 무한대로 두지 않기)
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("api-async-");
    executor.initialize();
    return executor;
}
```

`queueCapacity`를 기본값(사실상 무한대)으로 두면, 큐가 절대 다 차지 않기 때문에 `maxPoolSize`까지
스레드가 늘어날 일이 영원히 발생하지 않습니다. 대신 큐에 쌓인 작업 객체들이 메모리를 계속 잡아먹다가
결국 똑같이 OOM으로 죽습니다. "스레드 수를 늘려도 소용없다"는 착각이 여기서 나옵니다.

---

## 2. `@Async`는 프록시 기반이다 — 4가지 함정

### 왜 위험한가
`@Async`는 마법처럼 동작하는 게 아니라, Spring이 해당 빈을 감싸는 **프록시(Proxy) 객체**를 만들어서
"이 메서드가 호출되면 별도 스레드에 위임해라"라고 가로채는 방식으로 동작합니다. 즉 프록시를 거치지 않고
호출되는 경로는 `@Async`가 붙어 있어도 그냥 평범한 동기 메서드처럼 실행됩니다.

### 무슨 일이 벌어지는가

```java
@Service
public class BadExample {

    // ❌ 1. self-invocation: 같은 클래스 안에서 this로 호출
    public void order() {
        this.sendMail();   // 프록시를 거치지 않고 실제 객체(target)를 직접 호출
    }
    @Async
    public void sendMail() { }   // order()에서 호출하면 비동기 아님, 그냥 동기 실행됨

    // ❌ 2. private 메서드: 프록시가 오버라이드할 수 없음
    @Async
    private void internal() { }

    // ❌ 3. static / final: 프록시 상속 구조상 재정의 불가능
    @Async
    public static void statics() { }
}
```

여기서 무서운 점은 **컴파일 에러도 안 나고, 런타임 예외도 안 던진다**는 것입니다. 그냥 조용히 동기로
실행될 뿐입니다. 그래서 "@Async를 붙였는데 왜 응답이 느려요?"라는 질문이 나왔을 때 원인을 찾기가 매우 어렵습니다.

### 어떻게 고치는가
호출하는 쪽과 `@Async` 메서드를 **다른 빈으로 분리**하거나, 이벤트 기반으로 완전히 끊어냅니다.

```java
// ✅ 다른 빈으로 분리 (프록시를 거치도록)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final MailService mailService; // 별도 빈

    public void order() {
        mailService.sendMail();  // 외부에서 호출하므로 프록시를 거침 → 정상 비동기
    }
}

@Service
public class MailService {
    @Async
    public void sendMail() { }
}
```

---

## 3. 트랜잭션은 스레드를 넘지 못한다 (가장 헷갈리는 부분, 자세히 설명)

### 왜 위험한가
`@Transactional`은 내부적으로 `ThreadLocal`(정확히는 `TransactionSynchronizationManager`)에
"지금 이 스레드는 어떤 DB 커넥션을 쓰고 있고, 아직 커밋했는지 여부"를 저장합니다. `ThreadLocal`은
이름 그대로 **각 스레드마다 독립적으로 존재하는 저장소**이기 때문에, 다른 스레드는 그 정보를
전혀 들여다볼 수 없습니다. 그런데 `@Async`로 새로운 스레드를 만들면, 그 스레드는 원래 스레드의
트랜잭션 정보를 전혀 모르는 채로 시작됩니다.

### 무슨 일이 벌어지는가

```java
// ❌ 위험한 패턴
@Transactional
public void order(OrderCommand cmd) {
    Order order = orderRepository.save(Order.from(cmd)); // 아직 DB에 커밋 안 됨 (트랜잭션 중)
    notificationService.notifyAsync(order);               // 별도 스레드가 즉시 실행됨
}

@Async
public void notifyAsync(Order order) {
    Order found = orderRepository.findById(order.getId()); // 다른 스레드 = 다른 커넥션
    // 아직 커밋 전이라 DB에는 이 데이터가 안 보임 → found가 null이거나 예외 발생
}
```

시간 순서로 보면 다음과 같습니다.

1. `order()` 메서드 안에서 `save()`를 호출하면, 이 데이터는 "현재 트랜잭션 안에서만 보이는 상태"로
   DB에 잠정적으로 쓰입니다. 아직 `COMMIT`되지 않았기 때문에 다른 커넥션에서는 이 데이터가 보이지 않습니다.
2. `notifyAsync()`가 `@Async`로 호출되는 순간, 별도의 스레드가 **즉시** 실행을 시작합니다. 이 스레드는
   원래 스레드가 트랜잭션을 커밋했는지 여부를 전혀 신경 쓰지 않고 그냥 자기 할 일을 합니다.
3. 그 별도 스레드가 DB에서 방금 저장한 주문을 다시 조회하려고 하면, 아직 커밋이 안 됐기 때문에
   "데이터가 없다"는 결과를 받습니다.
4. 그 다음에야 `order()` 메서드가 끝나면서 `COMMIT`이 실행됩니다. 하지만 이미 늦었습니다.

이 버그의 가장 무서운 점은 **타이밍에 따라 발생 여부가 달라진다**는 것입니다. 커밋이 아주 빨리
일어나면 우연히 성공하기도 하고, 서버가 바빠서 커밋이 조금 늦어지면 실패하기도 합니다. 그래서
로컬 테스트에서는 멀쩍이 잘 되다가, 운영 환경에서 트래픽이 몰릴 때만 간헐적으로 터지는 "유령 버그"가 됩니다.

### 어떻게 고치는가 (이벤트 방식의 전체 동작 원리)

```java
// [1] 이벤트를 발행하는 쪽
@Component
@RequiredArgsConstructor
public class OrderFacade {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher publisher;

    @Transactional
    public void order(OrderCommand cmd) {
        Order order = orderRepository.save(Order.from(cmd));

        // 여기서 알림이 실행되는 게 아니라, "이런 이벤트가 발생했다"는 사실만 예약됩니다.
        publisher.publishEvent(new OrderCreatedEvent(order.getId()));

        // 메서드가 끝나면 @Transactional AOP가 COMMIT을 실행합니다.
    }
}

// [2] 이벤트 데이터 (평범한 객체, 특별한 의미 없음)
public record OrderCreatedEvent(Long orderId) { }

// [3] 이벤트를 받아서 실제로 처리하는 쪽 — 여기가 핵심
@Component
@RequiredArgsConstructor
public class OrderEventHandler {

    private final NotificationService notificationService;

    @Async("eventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        // 부모 트랜잭션이 완전히 커밋된 이후, 별도 스레드에서 실행됩니다.
        notificationService.send(event.orderId());
    }
}
```

`publishEvent()`가 호출되는 순간에는 `handle()`이 바로 실행되지 않습니다. Spring은 내부적으로
"이 이벤트를 구독하는 리스너 중에 `AFTER_COMMIT` 옵션이 붙은 게 있으니, 지금 실행하지 말고
트랜잭션 커밋이 성공하는 순간에 실행할 콜백 목록에 넣어두자"라고 처리합니다. 그래서 흐름은
다음과 같이 바뀝니다.

1. `save()`로 데이터를 씁니다 (아직 커밋 전).
2. `publishEvent()`는 "예약"만 하고 즉시 다음 줄로 넘어갑니다.
3. `order()` 메서드가 끝나면서 `COMMIT`이 실제로 일어납니다.
4. 커밋이 성공하는 그 순간, 미리 예약해 둔 `handle()`이 호출됩니다.
5. `handle()`에 `@Async`가 붙어 있으므로, 이 호출은 별도의 `eventExecutor` 스레드풀에서 실행됩니다.

여기서 두 어노테이션의 역할이 다르다는 걸 구분해야 합니다. `@TransactionalEventListener(AFTER_COMMIT)`는
**"언제" 실행할지**를 결정하고, `@Async`는 **"어느 스레드에서" 실행할지**를 결정합니다. 둘 중 하나만
쓰면 문제가 다시 생깁니다.

| 조합 | 결과 |
|---|---|
| `@TransactionalEventListener(AFTER_COMMIT)` 만 사용 | 커밋 후 실행되지만, `order()`를 호출한 스레드가 직접 처리 → 사용자가 알림 발송까지 기다려야 함 |
| `@Async` 만 사용 (`@EventListener`) | 다른 스레드에서 실행되지만 커밋 여부와 무관하게 즉시 실행 → 원래 버그 재발 가능 |
| 둘 다 사용 | 커밋 후, 별도 스레드에서 실행 → 실무 정석 조합 |

`phase` 옵션에는 이 외에도 `AFTER_ROLLBACK`(롤백됐을 때만), `AFTER_COMPLETION`(성공/실패 무관하게),
`BEFORE_COMMIT`(커밋 직전, 아직 같은 트랜잭션 안)이 있으며, 알림처럼 "성공했을 때만 처리하면 되는
작업"은 대부분 `AFTER_COMMIT`을 사용합니다.

**추가 규칙**
1. 비동기 메서드에는 엔티티를 넘기지 말고 ID나 DTO를 넘깁니다. 엔티티를 그대로 넘기면 다른 스레드에서
   지연 로딩(lazy loading)을 시도할 때 `LazyInitializationException`이 발생할 수 있습니다.
2. 비동기 작업 안에서 DB를 써야 한다면, 그 메서드에 **새로운 `@Transactional`을 별도로 선언**합니다.
3. 부모 트랜잭션이 롤백돼도 이미 실행된 비동기 작업은 자동으로 취소되지 않습니다. 실패를 감지해서
   되돌리는 보상 트랜잭션이나 재시도 로직을 별도로 설계해야 합니다.

---

## 4. 예외는 조용히 사라진다

### 왜 위험한가
반환형이 `void`인 `@Async` 메서드에서 예외가 발생하면, 그 예외를 던질 대상(호출자)이 이미 다른
스레드로 넘어가 있는 상황이라 아무도 그 예외를 받지 못합니다.

### 무슨 일이 벌어지는가

```java
@Async
public void job() {
    throw new RuntimeException("아무도 모른다"); // 로그도 안 남고 그냥 사라짐
}
```

호출한 쪽 코드는 정상적으로 다음 줄을 실행하고, 사용자는 정상 응답을 받습니다. 하지만 뒤에서
실행돼야 했던 작업(예: 알림 발송)은 실패했는데, 그 사실을 아무도 모릅니다. 나중에 "왜 알림이
안 갔지?"라는 문의가 들어올 때야 뒤늦게 원인을 찾게 됩니다.

### 어떻게 고치는가

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("[Async Error] method={}, params={}", method.getName(), params, ex);
    }
}
```

결과가 필요한 작업이라면 `void` 대신 `CompletableFuture<T>`를 반환하도록 만들면, 호출하는 쪽에서
`join()`이나 `get()`을 호출하는 시점에 `CompletionException`으로 감싸여서 예외를 전달받을 수 있습니다.

---

## 5. ThreadLocal 컨텍스트(MDC, SecurityContext)는 자동으로 전파되지 않는다

### 왜 위험한가
로그 추적용 `traceId`나 로그인한 사용자 정보(`SecurityContext`)는 모두 `ThreadLocal`에 저장됩니다.
`@Async`로 스레드가 바뀌면 이 정보들이 새 스레드에는 존재하지 않는 빈 상태로 시작됩니다.

### 무슨 일이 벌어지는가
로그를 찍어보면 원래 스레드에서는 `traceId=abc123`처럼 잘 찍히던 것이, 비동기 스레드로 넘어간
순간부터 `traceId=null`로 바뀝니다. 장애가 났을 때 "이 로그가 어떤 요청에서 나온 건지" 추적이
불가능해집니다. 더 심각한 경우, 스레드풀은 스레드를 재사용하기 때문에 **정리를 안 하면 이전에
그 스레드를 썼던 다른 사용자의 인증 정보가 남아있는 상태**로 다음 작업이 실행될 수 있습니다.

### 어떻게 고치는가

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> parentMdc = MDC.getCopyOfContextMap();      // 부모 스레드에서 복사
        SecurityContext parentCtx = SecurityContextHolder.getContext();
        return () -> {
            try {
                if (parentMdc != null) MDC.setContextMap(parentMdc);
                SecurityContextHolder.setContext(parentCtx);
                runnable.run();
            } finally {
                MDC.clear();                       // 스레드가 풀로 돌아가기 전에 반드시 정리
                SecurityContextHolder.clearContext();
            }
        };
    }
}
```

이 데코레이터를 Executor 빈에 등록해줘야 실제로 적용됩니다.

```java
executor.setTaskDecorator(new MdcTaskDecorator());
```

---

## 6. 모든 외부 호출에는 타임아웃을 건다

### 왜 위험한가
타임아웃이 없으면 "느린 응답"과 "영원히 응답 없는 상태"를 구분할 수 없습니다. 외부 API가
잠깐 멈추기만 해도, 그 요청을 처리하던 스레드는 무한정 대기 상태로 묶여버립니다.

### 무슨 일이 벌어지는가
외부 결제 API가 응답을 안 주면, 그 요청을 담당한 스레드는 풀에 반납되지 않고 계속 점유된 상태로
남습니다. 이런 요청이 몇 개만 쌓여도 스레드풀 전체가 소진되어, 정작 정상적으로 빨리 끝날 수 있는
다른 요청들도 순서를 기다리다가 함께 지연되거나 타임아웃됩니다.

### 어떻게 고치는가

```java
CompletableFuture<Product> productF =
    CompletableFuture.supplyAsync(() -> productClient.find(id), executor)
                      .orTimeout(500, TimeUnit.MILLISECONDS); // 핵심 정보: 실패하면 예외 전파

CompletableFuture<List<Review>> reviewF =
    CompletableFuture.supplyAsync(() -> reviewClient.findByProduct(id), executor)
                      .completeOnTimeout(List.of(), 300, TimeUnit.MILLISECONDS) // 부가 정보: 기본값 반환
                      .exceptionally(e -> {
                          log.warn("review fail", e);
                          return List.of();
                      });
```

응답이 없으면 전체 요청이 실패해도 되는 "핵심 정보"는 `orTimeout` + 예외 전파를 쓰고, 없어도
서비스가 돌아가는 "부가 정보"는 `completeOnTimeout` + 기본값을 쓰는 식으로 구분합니다.

---

## 7. `CompletableFuture`에는 항상 Executor를 명시한다

### 왜 위험한가
`supplyAsync()`를 호출할 때 두 번째 인자로 Executor를 넘기지 않으면, `ForkJoinPool.commonPool()`이라는
JVM 전역 공용 풀을 사용합니다. 이 풀은 기본적으로 CPU 코어 수 - 1개 정도의 스레드만 가지고 있고,
`parallelStream()` 같은 다른 병렬 연산도 이 풀을 공유해서 씁니다.

### 무슨 일이 벌어지는가

```java
// ❌ executor 없이 호출
CompletableFuture.supplyAsync(() -> productClient.find(id)); // commonPool 사용
```

여기서 만약 `productClient.find(id)`가 블로킹 I/O(네트워크 호출)라면, commonPool의 몇 안 되는
스레드가 이 대기 작업에 묶여버립니다. 애플리케이션 다른 곳에서 `parallelStream()`을 쓰는 코드가
있다면, 같은 풀을 공유하기 때문에 그 코드까지 함께 느려지거나 멈추는 현상이 나타납니다.

### 어떻게 고치는가

```java
CompletableFuture.supplyAsync(() -> productClient.find(id), apiExecutor); // 전용 풀 명시
```

I/O 작업용으로 별도로 만든 `ThreadPoolTaskExecutor`를 항상 명시적으로 전달합니다.

---

## 8. Graceful Shutdown을 반드시 설정한다

### 왜 위험한가
배포나 재시작 과정에서 애플리케이션이 강제로 종료되면, 그 순간 스레드풀에서 실행 중이거나
대기 중이던 작업들이 완료되지 못하고 그냥 사라집니다.

### 무슨 일이 벌어지는가
배포 직후에 "방금 주문한 알림톡이 안 왔어요"라는 문의가 간헐적으로 들어오는 경우, 십중팔구
배포 타이밍에 큐에 쌓여있던 비동기 작업이 그대로 유실된 것입니다. 배포는 매일 일어나는 일이기
때문에, 이 설정이 없으면 매 배포마다 조금씩 데이터가 새는 것과 같습니다.

### 어떻게 고치는가

```java
executor.setWaitForTasksToCompleteOnShutdown(true); // 종료 신호를 받아도 진행 중 작업은 완료시킴
executor.setAwaitTerminationSeconds(30);            // 최대 30초까지 기다려줌
```

---

## 9. `@Async`는 "유실 가능한" 비동기라는 것을 항상 인지한다

### 왜 위험한가
`@Async`로 처리되는 작업은 전부 애플리케이션 프로세스의 메모리 안에서만 존재합니다. 외부 브로커에
저장되는 게 아니라, 그냥 자바 객체로 큐에 들어있는 것뿐입니다.

### 무슨 일이 벌어지는가
서버가 예기치 않게 죽거나(OOM, 강제 종료 등), 위 8번처럼 배포 시점에 정리가 안 되면, 큐에
쌓여있던 작업은 디스크나 다른 서버에 복사본이 남아있지 않기 때문에 그냥 통째로 사라집니다.
알림 문자가 하나 안 가는 것은 큰 문제가 아닐 수 있지만, 만약 이 자리에 결제 후처리나 재고
차감 같은 작업이 있었다면 데이터 정합성이 깨지는 심각한 사고로 이어집니다.

### 어떻게 고치는가
작업의 성격에 따라 명확히 구분해서 설계합니다.

```
유실돼도 서비스에 큰 지장이 없는 작업 (알림, 로그, 통계 집계)
   → @Async + 전용 스레드풀로 충분

유실되면 안 되는 작업, 순서 보장이 필요한 작업, 나중에 재처리해야 하는 작업
   → Kafka 같은 외부 메시지 브로커 또는 Outbox 패턴 사용
```

작업을 만들기 전에 스스로에게 "지금 서버가 갑자기 죽으면, 이 작업이 사라져도 괜찮은가?"라고
질문해 보는 것이 `@Async`와 Kafka 중 무엇을 써야 할지 결정하는 가장 빠른 기준입니다.

---

## 최종 체크리스트 (커밋 전 5초 점검)

- [ ] Executor 빈을 직접 만들었고, `queueCapacity`가 유한한가?
- [ ] self-invocation / private / static 함정에 걸리지 않았는가?
- [ ] 이벤트 발행(`publishEvent`)과 실제 처리(`@TransactionalEventListener`)가 서로 다른 메서드에
      분리되어 있고, `phase = AFTER_COMMIT`이 지정되어 있는가?
- [ ] void 비동기 메서드에 예외 핸들러(`AsyncUncaughtExceptionHandler`)가 연결되어 있는가?
- [ ] MDC/SecurityContext가 TaskDecorator로 전파·정리되는가?
- [ ] 외부 호출에 타임아웃이 걸려 있는가?
- [ ] `supplyAsync`에 executor를 직접 넘겼는가?
- [ ] Graceful Shutdown 설정이 되어 있는가?
- [ ] 이 작업이 유실되면 안 되는 작업인지 판단했는가?
```
