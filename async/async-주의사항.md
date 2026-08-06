
```markdown
# Spring 비동기 처리 핵심 6가지 (Bad vs Good)

> 아래 6가지만 확실히 이해하면 실무 비동기 코드에서 발생하는 사고의 대부분을 막을 수 있습니다.
> 각 항목은 "왜 위험한가 → ❌ 잘못된 코드 → ✅ 잘된 코드" 순서로 구성되어 있습니다.

---

## 1. Executor는 직접 만들어서 써야 한다

### 왜 위험한가
`@EnableAsync`만 켜고 스레드풀을 따로 정의하지 않으면, Spring은 작업이 들어올 때마다
새 스레드를 계속 만들어내는 방식을 기본으로 사용합니다. 트래픽이 몰리면 스레드가 무한정
생성되다가 메모리가 부족해져서 서버가 죽습니다.

### ❌ 잘못된 코드
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    // Executor 빈을 하나도 정의하지 않음
    // → 요청마다 새 스레드가 생성되는 기본 방식 사용
}
```

### ✅ 잘된 코드
```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("apiExecutor")
    public ThreadPoolTaskExecutor apiExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);      // 기본으로 유지할 스레드 수
        executor.setMaxPoolSize(50);       // 바쁠 때 늘어나는 최대 스레드 수
        executor.setQueueCapacity(500);    // 대기 큐 (반드시 유한한 숫자로!)
        executor.setThreadNamePrefix("api-async-");
        executor.initialize();
        return executor;
    }
}
```

**핵심 한 줄**: `@Async` 쓰기 전에 스레드풀 빈을 직접 만들고, `queueCapacity`에 반드시 숫자를 지정한다.

---

## 2. 같은 클래스 안에서 `@Async` 메서드를 호출하면 비동기가 아니다 (self-invocation)

### 왜 위험한가
`@Async`는 Spring이 프록시(대리인) 객체를 만들어서 호출을 가로채는 방식으로 동작합니다.
그런데 같은 클래스 안에서 `this.메서드()`로 호출하면 프록시를 거치지 않고 실제 객체를
직접 호출하게 되어, 컴파일도 되고 실행도 되지만 그냥 조용히 동기로 실행됩니다.

### ❌ 잘못된 코드
```java
@Service
public class OrderService {

    public void order() {
        this.sendMail(); // 같은 클래스 내부 호출 → 프록시를 거치지 않음
    }

    @Async
    public void sendMail() {
        // 비동기로 동작할 것처럼 보이지만 실제로는 동기 실행됨
    }
}
```

### ✅ 잘된 코드
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MailService mailService; // 완전히 다른 빈으로 분리

    public void order() {
        mailService.sendMail(); // 외부 빈 호출 → 프록시를 정상적으로 거침
    }
}

@Service
public class MailService {
    @Async
    public void sendMail() {
        // 이제 진짜로 별도 스레드에서 실행됨
    }
}
```

**핵심 한 줄**: `@Async` 메서드는 반드시 **다른 클래스(다른 빈)**에서 호출해야 한다.

---

## 3. 트랜잭션 커밋 전에 비동기 작업이 먼저 실행되면 데이터가 없다

### 왜 위험한가
`@Transactional`은 "지금 이 스레드가 어떤 DB 작업을 진행 중인지"를 해당 스레드에만
저장해둡니다. `@Async`로 새로운 스레드가 만들어지면 그 스레드는 원래 스레드의 트랜잭션이
아직 커밋 중인지 전혀 알지 못한 채로 즉시 실행을 시작합니다. 그래서 방금 저장한 데이터를
비동기 스레드가 조회하면 "아직 커밋 전이라 데이터가 없다"는 문제가 생깁니다.

### ❌ 잘못된 코드
```java
@Transactional
public void order(OrderCommand cmd) {
    Order order = orderRepository.save(Order.from(cmd)); // 아직 커밋 전
    notificationService.notifyAsync(order); // 비동기 스레드가 즉시 조회 시도
}

@Async
public void notifyAsync(Order order) {
    Order found = orderRepository.findById(order.getId());
    // found가 null일 수 있음! 아직 커밋되지 않았기 때문
}
```

### ✅ 잘된 코드
```java
// [1] 이벤트를 발행만 하는 쪽
@Component
@RequiredArgsConstructor
public class OrderFacade {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher publisher;

    @Transactional
    public void order(OrderCommand cmd) {
        Order order = orderRepository.save(Order.from(cmd));
        publisher.publishEvent(new OrderCreatedEvent(order.getId())); // 예약만 함
        // 메서드가 끝나면 여기서 COMMIT 발생
    }
}

// [2] 커밋된 이후에만 실행되는 쪽
@Component
@RequiredArgsConstructor
public class OrderEventHandler {

    private final NotificationService notificationService;

    @Async("eventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        // 부모 트랜잭션이 완전히 커밋된 다음, 별도 스레드에서 실행됨
        notificationService.send(event.orderId());
    }
}
```

**동작 순서**: ① 데이터 저장(아직 커밋 전) → ② `publishEvent`는 실행을 예약만 함 → ③ `order()` 종료 시 실제 COMMIT 발생 → ④ 커밋 성공 확인 후 `handle()` 실행 → ⑤ `@Async` 덕분에 이 실행은 별도 스레드에서 진행.

**핵심 한 줄**: 비동기로 DB 후처리가 필요하면 직접 호출하지 말고, `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` 조합의 이벤트로 분리한다.

---

## 4. `void` 비동기 메서드의 예외는 아무도 모른다

### 왜 위험한가
반환값이 `void`인 `@Async` 메서드에서 예외가 발생하면, 그 예외를 받아줄 호출자가 이미
다른 스레드로 넘어가 있어서 아무 로그도 남기지 않고 그냥 조용히 사라집니다.

### ❌ 잘못된 코드
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    // AsyncUncaughtExceptionHandler를 지정하지 않음
}

@Async
public void sendMail() {
    throw new RuntimeException("여기서 터졌는데 아무도 모른다");
}
```

### ✅ 잘된 코드
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("[Async 예외] method={}, params={}", method.getName(), params, ex);
    }
}
```

**핵심 한 줄**: `void` 반환 `@Async` 메서드를 쓴다면 `AsyncUncaughtExceptionHandler`는 필수로 등록한다.

---

## 5. 외부 호출에는 반드시 타임아웃을 건다

### 왜 위험한가
타임아웃이 없으면 외부 API가 응답을 안 줄 때, 그 요청을 처리하던 스레드가 영원히
대기 상태로 묶여버립니다. 이런 요청이 몇 개만 쌓여도 스레드풀 전체가 마비됩니다.

### ❌ 잘못된 코드
```java
CompletableFuture<Product> productF =
    CompletableFuture.supplyAsync(() -> productClient.find(id), executor);
    // 타임아웃 없음 → 외부 API가 멈추면 이 스레드도 영원히 멈춤
```

### ✅ 잘된 코드
```java
CompletableFuture<Product> productF =
    CompletableFuture.supplyAsync(() -> productClient.find(id), executor)
                      .orTimeout(500, TimeUnit.MILLISECONDS); // 500ms 안에 안 오면 실패 처리

CompletableFuture<List<Review>> reviewF =
    CompletableFuture.supplyAsync(() -> reviewClient.findByProduct(id), executor)
                      .completeOnTimeout(List.of(), 300, TimeUnit.MILLISECONDS); // 없어도 되는 정보는 기본값
```

**핵심 한 줄**: 없으면 안 되는 핵심 데이터는 `orTimeout`, 없어도 괜찮은 부가 데이터는 `completeOnTimeout`을 쓴다.

---

## 6. `@Async`로 처리한 작업은 서버가 죽으면 사라진다

### 왜 위험한가
`@Async`로 처리되는 작업은 애플리케이션 메모리 안에서만 존재합니다. 외부 저장소에
남는 것이 아니기 때문에, 서버가 갑자기 죽거나 재배포되면 처리 중이던 작업이 그대로 사라집니다.

### ❌ 잘못된 코드
```java
@Async
public void processPayment(Long orderId) {
    // 결제 후처리, 재고 차감처럼 유실되면 안 되는 중요한 작업을
    // 그냥 @Async로만 처리하고 있음
    inventoryService.decreaseStock(orderId);
}
```

### ✅ 잘된 코드
```java
// 유실돼도 괜찮은 작업(알림, 로그)은 @Async로 충분
@Async("eventExecutor")
public void sendNotification(Long orderId) {
    notificationService.send(orderId);
}

// 유실되면 안 되는 중요한 작업은 Kafka 같은 외부 브로커로 보냄
public void processPayment(Long orderId) {
    kafkaTemplate.send("payment-events", new PaymentEvent(orderId));
    // 서버가 죽어도 Kafka에 저장되어 있어 나중에 재처리 가능
}
```

**핵심 한 줄**: "이 작업이 사라져도 괜찮은가?"를 먼저 판단하고, 안 되면 `@Async`가 아니라 Kafka를 쓴다.

---

## 최종 체크리스트

- [ ] Executor를 직접 만들었는가? (`queueCapacity`는 무한대 금지)
- [ ] `@Async` 메서드를 같은 클래스 안에서 호출하고 있지 않은가?
- [ ] DB 후처리가 필요한 비동기 작업은 `@TransactionalEventListener(AFTER_COMMIT)`로 분리했는가?
- [ ] void 비동기 메서드에 예외 핸들러를 등록했는가?
- [ ] 외부 호출에 타임아웃을 걸었는가?
- [ ] 이 작업이 유실되면 안 되는 작업인지 판단했는가?
```
