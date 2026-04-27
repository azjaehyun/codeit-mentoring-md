# Sprint3 — 전체 공지용: 꼭 알고 넘어가야 할 사항 (코드·주석·상세 설명)

> 이 문서는 리뷰에서 반복적으로 지적된 항목을 **이름·개인 식별 없이** 정리한 것입니다.  
> 과제(디스코드잇) 구조: Spring Boot, JCF/File Repository, Service/DTO, `JavaApplication`·`DiscodeitApplication` 비교 등을 가정합니다.

---

## 1. `build.gradle` — Spring Boot 버전·스타터 이름

### 왜 중요한가

- **존재하지 않는 Boot 버전**이면 의존성 다운로드·해석이 실패합니다.  
- **공식이 아닌 artifact 이름**(`spring-boot-starter-webmvc` 등)은 저장소에 없어 **빌드가 아예 실패**할 수 있습니다.  
- 이 미션은 **Servlet + MVC(`DispatcherServlet`)** 흐름이 일반적이므로, 리액티브가 필요 없다면 **`spring-boot-starter-web`** 을 씁니다.

### 잘못된 예 (의미만 설명; 실제 프로젝트에 맞게 고칠 것)

```gradle
plugins {
    // BAD: 아직 공개되지 않았거나 잘못 기입한 버전이면 resolve 실패
    id 'org.springframework.boot' version '99.99.99'
}

dependencies {
    // BAD: "webflux" 는 Netty 기반 리액티브 스택. Servlet 기반 미션이면 보통 "web" 이 맞음
    implementation 'org.springframework.boot:spring-boot-starter-webflux'

    // BAD: 공식 Spring Boot 문서에 없는 이름은 Artifact 를 못 찾을 수 있음
    // implementation 'org.springframework.boot:spring-boot-starter-webmvc'  // ← 이런 이름은 쓰지 말 것
}
```

### 권장 예 (형태 참고; 버전은 팀/과제에 맞는 **실제 릴리스**로 고정)

```gradle
plugins {
    // OK: plugins.gradle.org / Spring 공식 release notes에서 확인한 버전
    id 'org.springframework.boot' version '3.4.0'   // 예시: 3.4.x 등 실제 존재하는 번호
    id 'io.spring.dependency-management' version '1.1.6'
    // ...
}

dependencies {
    // OK: 톰캣 내장, MVC + JSON 등 일반 REST 과제에 적합
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // OK: JUnit, MockMvc 등 테스트는 공식 "starter-test"
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### 정리

- **버전 번호**는 [Spring Boot releases](https://github.com/spring-projects/spring-boot/releases) 등에서 **확인한 숫자**만 사용한다.  
- **스타터 이름**은 [Spring Initializr](https://start.spring.io) 또는 공식 doc의 의존성명을 **복붙** 수준으로 맞춘다.  
- WebFlux는 **“리액티브 스택이 과제 요구”**가 아닌 이상 `web` 으로 통일하는 편이 안전하다.

---

## 2. `@Service` 누락 — Bean 이 안 만들어짐

### 왜 중요한가

- Spring 은(컴포넌트 스캔 대상이면) `@Service`, `@Component`, `@Configuration` 등 **스테레오타입**이 붙은 클래스를 Bean 으로 등록합니다.  
- **어노테이션이 없으면** `AuthService` 타입으로 주입·조회할 때 **`NoSuchBeanDefinitionException`** 이 납니다. “코드는 있는데 런타임에만 터지는” 전형적 원인입니다.

### 잘못된 예

```java
// BAD: @Service 가 없으면 BasicAuthService 는 Bean 이 아님
// (패키지가 스캔되어도 스테레오타입이 아니면 등록 대상이 아님)
@RequiredArgsConstructor
public class BasicAuthService implements AuthService {
    // ...
}
```

### 권장 예

```java
import org.springframework.stereotype.Service;

// OK: @Service = @Component + 서비스 계층 의미. Bean 으로 등록됨
@Service
@RequiredArgsConstructor
public class BasicAuthService implements AuthService {
    private final UserRepository userRepository;
    // ...
}
```

### 정리

- **인터페이스를 구현한 구현체만** 두고, **`@Service` 를 빼는 실수**는 “컴파일은 되는데 앱 뜨면 죽음”으로 이어진다.  
- 같은 패키지의 다른 `*Service` 들과 **동일한 스테레otype**을 맞출 것.

---

## 3. `@SpringBootApplication` 중복·`JavaApplication` 역할

### 왜 중요한가

- **`@SpringBootApplication`** 은 `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` 을 묶은 것이고, **부트 앱의 진입점은 보통 하나**가 명확합니다.  
- 과제는 **`JavaApplication`**: `new` 로 Repository/Service 를 직접 만들고 **수동 DI**  
  **`DiscodeitApplication`**: Spring 컨테이너·**자동 DI**  
  를 **나란히 비교**하라는 요구가 많습니다.  
- 둘 다 `@SpringBootApplication` 이면 “수동 DI 데모”가 섞이고, `JavaApplication` 이 Spring 에 의해 기동·스캔되면 **의도와 다르게 동작**할 수 있습니다.

### 잘못된 예

```java
// file: JavaApplication.java
@SpringBootApplication  // BAD: 수동 new 데모용 클래스에 붙이면 Spring 앱이 두 갈래가 됨
public class JavaApplication {
    public static void main(String[] args) {
        SpringApplication.run(JavaApplication.class, args);  // + 수동 new 와 섞이기 쉬움
    }
}
```

```java
// file: DiscodeitApplication.java
import static com.example.app.JavaApplication.*;  // BAD: 애플리케이션끼리 static import 결합

@SpringBootApplication
public class DiscodeitApplication {
    public static void main(String[] args) {
        // ...
        setupUser();  // 다른 main 쪽 테스트 헬퍼에 의존
    }
}
```

### 권장 방향 (개념)

```java
// JavaApplication.java — Spring 을 켜지 않음 (어노테이션에서 @SpringBootApplication 제거)
public class JavaApplication {
    public static void main(String[] args) {
        // OK: "직접 new" 로 저장소·서비스 조립
        UserRepository userRepository = new FileUserRepository(/* ... */);
        UserService userService = new BasicUserService(userRepository, /* ... */);

        // OK: 수동으로 만든 서비스로 시나리오 실행
        // runScenario(userService);
    }
}
```

```java
// DiscodeitApplication.java — Spring Boot 단 하나
@SpringBootApplication
public class DiscodeitApplication {
    public static void main(String[] args) {
        var ctx = SpringApplication.run(DiscodeitApplication.class, args);
        // OK: 이 클래스 안에 테스트용 메서드를 두거나, 별도 util 로 분리
    }
}
```

### 정리

- **`JavaApplication` 에는 `@SpringBootApplication` 을 달지 않는** 전략이 흔합니다.  
- `import static` 으로 **다른** `Application` 의 메서드를 끌어오는 것은 **결합**이 큼 → **같은 파일에 헬퍼** 두거나 **별도 `*TestUtils` 클래스**로 분리.

---

## 4. `application.yaml` / 프로퍼티·`@ConditionalOnProperty`

### 왜 중요한가

- `jcf` / `file` 전환, `file-directory` 경로는 **`application.yaml`에 두고** Bean 선택·경로 주입에 씁니다.  
- **파일이 없으면** 기본값·프로필·조건부 Bean 동작이 **의도와 다르게** 남을 수 있고, 과제 **“yaml 로 설정했다”** 항목도 불충분해질 수 있습니다.

### 권장 예 (필드명은 본인 프로젝트와 일치시킬 것)

```yaml
# src/main/resources/application.yaml
spring:
  application:
    name: discodeit

# OK: Bean 분기, File 경로에 공통으로 사용
discodeit:
  repository:
    type: jcf                 # jcf | file
    file-directory: .discodeit
```

### `File*Repository` 에서 — 하드코딩 vs 주입

```java
// BAD: 경로가 코드에 박혀 있으면 제출/운영/동료 PC 마다 바꾸려면 컴파일이 필요
public FileUserRepository() {
    this.basePath = Paths.get(System.getProperty("user.dir"), "file-data-map", "User");
}
```

```java
import org.springframework.beans.factory.annotation.Value;

// OK: yaml 의 discodeit.repository.file-directory 와 연결. 없으면 기본값 "data"
public FileUserRepository(
    @Value("${discodeit.repository.file-directory:data}") String fileDirectory
) {
    // user.dir + 설정값 + 엔터티명 등으로 경로 구성
    this.basePath = Paths.get(System.getProperty("user.dir"), fileDirectory, "User");
}
```

### 정리

- **리소스에 yaml/properties 없음** → 조건부 Bean·`@Value` **검증이 어려움**.  
- **같은 패턴**의 `File*Repository` 전부에 **일관되게** `@Value` 를 적용하는 것이 좋다.

---

## 5. `transient` 와 `password` (File + Serializable)

### 왜 중요한가

- `User` 가 **`Serializable`** 이고 `File*Repository` 가 **ObjectOutputStream** 등으로 직렬화한다면,  
  **`transient` 필드는 바이트 스트림에 안 실리고, 역직렬화 시 기본값(null)** 이 됩니다.  
- `password` 에 `transient` 를 달면 **“보안”이 아니라 “저장 안 함”** 이라, File 모드에서 **로그인·검증이 항상 실패**할 수 있습니다. (JCF만 쓰면 문제가 드러나지 않을 수 있음.)

### 잘못된 예

```java
public class User implements Serializable {
    // BAD: File 저장 후 읽기 시 password == null
    private transient String password;
}
```

### 권장 예 (과제 수준; 실제 서비스는 해시·암호화·별도 vault 가 별론)

```java
public class User implements Serializable {
    // OK: 직렬화에 포함. (실제 제품에선 평문 저장 자체를 피하는 것이 맞고, 여기서는 "유실" 버그 방지)
    private String password;
}
```

### 정리

- **`transient` 의미**를 정확히 이해한 뒤 쓸 것. “숨긴다”가 아니라 **직렬화에서 제외**다.  
- 비밀번호를 **아예 안 남기겠다**는 설계라면, 직렬화 대상에서 빼는 것이 **일관되게** User 전체·인증 흐름에 반영돼야 한다.

---

## 6. `Map<UUID, User>` + `findByUsername` — 키 타입 불일치

### 왜 중요한가

- 저장 구조가 `Map<UUID, User>` 이면 **키는 UUID** 입니다.  
- `map.get(username)` 처럼 **String 키**로 `get` 하면 **항상 null** 에 가깝고, **로그인·중복 닉네임 검사**가 전부 망가집니다.

### 잘못된 예

```java
public class JcfUserRepository implements UserRepository {
    private final Map<UUID, User> data = new HashMap<>();  // key = User 의 id (UUID)

    @Override
    public Optional<User> findByUsername(String username) {
        // BAD: UUID 키 맵에 String 을 넣고 조회 — 타입/의미가 맞지 않아 항상 empty 에 가깝다
        return Optional.ofNullable(data.get(username));
    }
}
```

### 권장 예 (과제·소규모 데이터 기준)

```java
@Override
public Optional<User> findByUsername(String username) {
    // OK: id 맵이면 전체 user 중 username 이 같은 것을 찾는다
    return data.values().stream()
        .filter(u -> u.getUsername().equals(username))
        .findFirst();
}
```

### 정리

- **“어떤 키로 O(1)에 찾는지”** 를 맵 설계에 맞출 것. username 조회를 많이 하면 **별도 `Map<String, UUID>`** 등을 고려(과제는 스트림으로도 충분한 경우가 많음).

---

## 7. `findById` vs `findByUserId` — id 의미 혼동

### 왜 중요한가

- `UserStatus` 는 **자신의 id(UUID)** 와 **userId(연관 User 의 id)** 가 다릅니다.  
- **`updateByUserId(userId, ...)`** 인데 `findById(userId)` 를 쓰면, **UserStatus 의 PK** 로 찾는 것이 되어 **의미가 어긋납니다**.

### 잘못된 예

```java
@Override
public UserStatus updateByUserId(UUID userId, UserStatusUpdateRequest request) {
    // BAD: findById 는 "UserStatus 엔터티의 id". 인자 userId 는 "User 의 id" 인 경우가 많음
    UserStatus st = userStatusRepository.findById(userId)
        .orElseThrow(() -> new NoSuchElementException("not found"));
    st.update(request.newLastActiveAt());
    return userStatusRepository.save(st);
}
```

### 권장 예

```java
@Override
public UserStatus updateByUserId(UUID userId, UserStatusUpdateRequest request) {
    // OK: "user 기준" 조회는 findByUserId
    UserStatus st = userStatusRepository.findByUserId(userId)
        .orElseThrow(() -> new NoSuchElementException("UserStatus for user " + userId + " not found"));
    st.update(request.newLastActiveAt());
    return userStatusRepository.save(st);
}
```

### 정리

- 메서드 이름·파라미터 이름과 **Repository 메서드 이름**이 같은 **id 의미**를 쓰는지 매칭할 것.

---

## 8. `existsById` 조건 **반전** — 가장 흔한 치명적 실수

### 왜 중요한가

- “**존재하지 않을 때** 예외” 를 던지려면 **`!existsById(id)`** 입니다.  
- **`existsById` 가 true 인데** “없다”고 예외를 던지면, **정상 케이스만 막고** 이상 케이스만 통과시키는 **역로직**이 됩니다.

### 잘못된 예 (ReadStatus 생성)

```java
public ReadStatus create(ReadStatusCreateRequest request) {
    UUID userId = request.userId();

    // BAD: "존재하는 사용자" 를 "없다"로 처리
    if (userRepository.existsById(userId)) {
        throw new NoSuchElementException("User id " + userId + " does not exist");
    }
    // ... 이후: 없는 userId 면 ReadStatus 를 만들 수 있게 됨 (데이터 훼손)
}
```

### 권장 예

```java
public ReadStatus create(ReadStatusCreateRequest request) {
    UUID userId = request.userId();

    // OK: 없을 때만 예외
    if (!userRepository.existsById(userId)) {
        throw new NoSuchElementException("User id " + userId + " does not exist");
    }
    // channel 존재, 중복 ReadStatus 체크 등
}
```

### 삭제에서의 반대 실수

```java
// BAD: "있는데" 못 지우고, "없는데" deleteById 를 호출
public void delete(UUID userStatusId) {
    if (userStatusRepository.existsById(userStatusId)) {
        throw new NoSuchElementException("not found");
    }
    userStatusRepository.deleteById(userStatusId);
}
```

```java
// OK: "없을 때" not found, "있을 때" 삭제
public void delete(UUID userStatusId) {
    if (!userStatusRepository.existsById(userStatusId)) {
        throw new NoSuchElementException("not found");
    }
    userStatusRepository.deleteById(userStatusId);
}
```

### 정리

- **예외 메시지**(`"does not exist"`, `"not found"`)를 읽고, **if 조건이 그 말과 같은 방향**인지 **반드시** 대조할 것.  
- 리뷰할 때 **한 줄씩 소리 내어 읽기**(`exists` 일 때 throw 가 맞냐?)를 권장한다.

---

## 9. JCF vs File — `update` 후 `save`

### 왜 중요한가

- JCF: 같은 객체 **참조**를 갱신하면 “메모리 안” 은 이미 반영됨.  
- File: **객체를 바꿔도** 디스크에 쓰는 건 보통 `save` 호출. **`save` 생략**이면 **재시작 시 변경이 사라짐**.

### 잘못된 예

```java
public Channel update(UpdateChannelRequest request) {
    Channel ch = channelRepository.findById(request.channelId())
        .orElseThrow(() -> new IllegalArgumentException("not found"));
    if (ch.getType() == ChannelType.PRIVATE) {
        throw new IllegalArgumentException("cannot update private");
    }
    ch.update(request.name(), request.description());
    // BAD: File 구현이면 여기서 파일에 flush 안 될 수 있음
    return ch;
}
```

### 권장 예

```java
public Channel update(UpdateChannelRequest request) {
    Channel ch = channelRepository.findById(request.channelId())
        .orElseThrow(() -> new IllegalArgumentException("not found"));
    if (ch.getType() == ChannelType.PRIVATE) {
        throw new IllegalArgumentException("cannot update private");
    }
    ch.update(request.name(), request.description());
    // OK: 변경분을 Repository 규약에 맞게 영속화
    return channelRepository.save(ch);
}
```

### 정리

- `Repository` 가 **“변경 감지” 자동 저장**이 아닌 **명시적 save** 모델이면, **update 경로마다 `save`** 를 점검할 것.

---

## 10. `User` 삭제·연관 데이터 (Binary, UserStatus)

### 왜 중요한가

- `User` 만 지우고 **프로필 `BinaryContent`·`UserStatus`** 를 남기면 **고아 데이터**·**디스크 누적**·**참조 불일치**가 생깁니다.

### 권장 흐름 (개념)

```java
public void delete(UUID userId) {
    // OK: 먼저 User 로드해서 profileId 등 확인
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new NoSuchElementException("User " + userId + " not found"));

    // OK: 프로필 파일/레코드 먼저
    if (user.getProfileId() != null) {
        binaryContentRepository.deleteById(user.getProfileId());
    }
    // OK: 부가 엔터티
    userStatusRepository.deleteByUserId(userId);
    // OK: 마지막 User
    userRepository.deleteById(userId);
}
```

### 정리

- **삭제 순서**는 “다른 쪽이 참조하는 것부터” / “부모·핵심 엔터티는 마지막” 등 **팀·데이터 모델**에 맞게 정하되, **누락이 없는지** 리스트업할 것.

---

## 11. `User` 업데이트 — DTO·엔터티 정합

### 왜 중요한가

- `UserUpdateRequest` 에 `email`, `password` 가 있는데, Service 나 Entity 에 **username·프로필만** 반영되면 **요구 스펙·모범과 불일치**입니다.

### 엔터티 (개념)

```java
// OK: null/blank, 동일 값이면 updatedAt 생략 등 팀 룰에 맞게
public void updateEmail(String newEmail) { /* ... */ }
public void updatePassword(String newPassword) { /* ... */ }
```

### 서비스 (개념)

```java
public UserResponse update(UUID id, UserUpdateRequest request) {
    User user = userRepository.findById(id).orElseThrow(/* ... */);

    if (request.username() != null) user.updateUsername(request.username());
    if (request.email() != null) user.updateEmail(request.email());
    if (request.password() != null) user.updatePassword(request.password());
    // profile 등

    return toResponse(userRepository.save(user));
}
```

### 정리

- **Request 필드** ↔ **Entity setter/update 메서드** ↔ **Service 분기** 가 **1:1**로 대응하는지 확인.

---

## 12. `toString` / 포맷터 — `null` `updatedAt`

### 왜 중요한가

- 엔터티 **생성 직후** `updatedAt` 이 **null** 인 설계가 많습니다.  
- `DateTimeFormatter#format(null)` → **`NullPointerException`**.

### 잘못된 예

```java
@Override
public String toString() {
    var f = DateTimeFormatter.ISO_INSTANT;
    // BAD: createdAt 만 있고 updatedAt 은 null 인 경우 NPE
    return "User{ createdAt=" + f.format(createdAt) + ", updatedAt=" + f.format(updatedAt) + "}";
}
```

### 권장 예

```java
@Override
public String toString() {
    var f = DateTimeFormatter.ISO_INSTANT;
    String u = (updatedAt == null) ? "N/A" : f.format(updatedAt);
    return "User{ createdAt=" + f.format(createdAt) + ", updatedAt=" + u + "}";
}
```

### 정리

- **로그·디버그용 toString** 도 null 안전을 기본으로 둔다.

---

## 13. `attachmentIds` null — NPE

### 왜 중요한가

- `MessageCreateRequest` 의 `attachmentIds()` 가 **null** 이고, 아래처럼 쓰면 **즉시 NPE** 입니다. 호출부가 `new MessageCreateRequest(..., null)` 를 쓰는 경우가 실제로 있습니다.

### 잘못된 예

```java
public Message create(MessageCreateRequest request) {
    // BAD: null 이면 forEach 전에 NPE
    request.attachmentIds().forEach(id -> { /* ... */ });
}
```

### 권장 예

```java
public Message create(MessageCreateRequest request) {
    // OK: null 이면 "첨부 없음" 으로 취급
    if (request.attachmentIds() != null) {
        request.attachmentIds().forEach(id -> { /* ... */ });
    }
    // 대안: record 팩토리/생성자에서 null 을 List.of() 로 정규화
}
```

### 정리

- **List 필드**는 **“빈 리스트”** 와 **“null”** 정책을 팀/과제에 맞게 **하나로 통일**하는 것이 안전하다.

---

## 14. DTO·중복 변환·`Instant` vs `LocalDateTime`

### DTO

- `find` / `findAll` 이 **Entity** 를 그대로 반환하면, **필드 추가·리네이밍**이 API 에 그대로 노출됩니다.  
- `User` + 온라인 여부(`UserStatus`) 처럼 **여러 엔터티를 합친 뷰**는 **UserDto** 에 담는 편이 일반적입니다.

### 중복 (채널 DTO 예)

```java
// OK: find / findAllByUserId 가 같은 구조의 ChannelDto 를 만든다면
private ChannelDto toDto(Channel channel) {
    Instant latest = /* messageRepository 로부터 channelId 기준 max */;
    // ...
    return new ChannelDto(/* ... */);
}
```

### 시간 타입

- 엔터티는 **`Instant` (UTC)** 인데, 응답에서만 `LocalDateTime` + `ZoneId.systemDefault()` 를 쓰면 **서버 타임존**에 따라 값이 달라 보일 수 있습니다.  
- **API 는 Instant 통일** 또는 **항상 UTC·명시 offset** 중 하나로 맞추는 것이 혼란이 적다.

### 정리

- **응답 DTO**와 **필드 의미(시간대)** 를 엔터티·문서·클라이언트 합의와 **일치**시킬 것.

---

## 15. 인터페이스 구현 — `@Override`·이중 조회

### `@Override` 누락

```java
// 권장: 인터페이스 메서드면 @Override (시그니처 바뀌면 컴파일러가 잡아줌)
@Override
public User create(UserCreateRequest request) { /* ... */ }
```

### 이중 조회 (같은 데이터를 두 번)

```java
// BAD: findById 로 이미 userStatus 를 얻었는데, 다시 findByUserId (동일 user 의 동일 status)
public UserStatus find(UUID id) {
    UserStatus s = repo.findById(id).orElseThrow();
    return repo.findByUserId(s.getUserId()).orElseThrow();  // 대개 불필요
}

// OK
public UserStatus find(UUID id) {
    return repo.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("not found: " + id));
}
```

### 정리

- **“한 번으로 충분한가?”** 를 File I/O·성능 관점에서도 점검.

---

## 16. 예외 타입·인증 `findAll`

### `RuntimeException` 남용

```java
// 권장: "없음" 은 NoSuchElementException 등 의미가 분명한 타입
return binaryContentRepository.findById(id)
    .orElseThrow(() -> new NoSuchElementException("BinaryContent " + id + " not found"));
```

### 인증

```java
// 동작은 하지만 비효율: findAll() 후 스트림 필터
// 개선: Optional<User> findByUsername(String) 이 있으면 O(n) 전체 스캔을 줄일 수 있음
return userRepository.findByUsername(name)
    .filter(u -> u.getPassword().equals(rawPassword))  // 실제는 해시 비교
    .orElseThrow(() -> new IllegalArgumentException("invalid credentials"));
```

### 정리

- **같은 계층·같은 도메인**에서 예외 종류를 **팀 룰**로 맞추면 상위에서 처리·로깅이 쉽다.

---

## 17. `DiscodeitApplication` 안의 `clearDataFiles`·`JavaApplication` 주석

### `clearDataFiles`

- `main` 시작 시 **`.discodeit` 전체 삭제** 같은 로직이 **기본 main**에 있으면, **실서버·실수 실행**에 데이터가 날아갈 수 있습니다.  
- **권장**: `@Profile("test")` + 테스트 전용 `ApplicationRunner` / 초기화 클래스로 옮기거나, **로컬 개발 플래그**로만 켜기.

### `JavaApplication` 이 전부 주석

- **IoC·DI·Bean** 비교를 **실행**으로 보여주려면, `new` + 수동 DI 코드가 **컴파일·실행 가능**해야 합니다.  
- 생성자 시그니처가 바뀌었다면, **주석이 아니라** 현재 `Basic*Service` / `File*Repository` 시그니처에 맞게 **고쳐서** 다시 돌릴 것.

### 정리

- **“교육용 데모 엔트리”** 는 제품 코드와 **같은 품질(컴파일·실행)** 을 갖는 것이 좋다.

---

## 18. 애플리케이션 테스트에서 `Repository` 직접 사용

- `UserStatusService` 를 두고도 테스트에서 `UserStatusRepository#deleteByUserId` 를 **직접** 부르면, **서비스 계층**을 건너뜁니다.  
- **권장**: “애플리케이션 수준 CRUD” 는 **Service API**만으로 시나리오를 작성해 **계층 경계**를 맞출 것 (과제/팀 룰에 따름).

---

## 19. Lombok `Repository` import 오탈·`@RequiredArgsConstructor` 통일

- Service 클래스에 **`import org.springframework.stereotype.Repository`** 가 남아 있고 **쓰지 않으면** IDE 경고. **삭제**.  
- 대부분의 `*Service` 가 `@RequiredArgsConstructor` + `private final` 인데, **한 곳만 수동 생성자**면 **스타일 통일**이 깨짐(기능은 동일).

---

