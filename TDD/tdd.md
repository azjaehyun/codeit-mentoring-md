
## 목차

1. 꼭 알고 넘어가야 할 핵심 개념 리뷰
2. 테스트 코드 기본 문법 가이드 (JUnit 5 + Mockito + AssertJ)
3. Given-When-Then 작성 가이드
4. Spring MVC 패턴에서 REST API 테스트 방법
5. `@Mock` vs `@MockBean` 차이점
6. DB 데이터를 모킹(Mocking)으로 테스트하는 방법
7. 로직이 바뀌면 테스트도 바꿔야 하는데, 그래도 TDD가 중요한 이유

---

## 1. 꼭 알고 넘어가야 할 핵심 개념 리뷰

Spring 안정성 수업에서 학생들이 반드시 손에 익혀야 할 개념들만 추렸습니다.

### 1.1 TDD(Test-Driven Development)의 3단계 사이클

TDD는 "테스트를 먼저 짜고, 실패하는 걸 보고, 코드를 작성한다"는 개발 방법론입니다. 사이클은 항상 **RED → GREEN → REFACTOR** 순서입니다.

- **RED**: 아직 구현되지 않은 기능에 대한 테스트를 먼저 작성 → 실패하는 것을 확인
- **GREEN**: 테스트를 통과할 수 있는 **최소한의 코드**만 작성
- **REFACTOR**: 테스트가 통과하는 상태를 유지하며 코드를 깔끔하게 정리

### 1.2 테스트 피라미드

안정적인 서비스는 테스트가 다음 비율을 따릅니다.

| 단계 | 비율 | 특징 |
|------|------|------|
| Unit Test (단위 테스트) | 약 70% | 빠름, 외부 의존성 없음, Mock 사용 |
| Integration Test (통합 테스트) | 약 20% | Spring Context 일부 로드, DB/외부 시스템 일부 연동 |
| E2E Test (End-to-End) | 약 10% | 실제 환경과 가장 유사, 느리지만 신뢰도 높음 |

### 1.3 Layered Architecture (계층 구조)

Spring MVC는 보통 다음과 같이 계층을 나눠 책임을 분리합니다.

```
Client → Controller (요청/응답) → Service (비즈니스 로직) → Repository (DB 접근) → DB
```

각 계층을 독립적으로 테스트할 수 있어야 좋은 구조입니다.

### 1.4 핵심 어노테이션 정리

| 어노테이션 | 용도 |
|-----------|------|
| `@SpringBootTest` | 전체 Spring Context 로드 (통합 테스트) |
| `@WebMvcTest` | Controller 계층만 로드 (가볍고 빠름) |
| `@DataJpaTest` | JPA Repository 계층만 로드 |
| `@MockBean` | Spring Context의 빈을 Mock으로 교체 |
| `@Mock` | 순수 Mockito Mock 객체 생성 (Spring 없이) |
| `@InjectMocks` | `@Mock`으로 만든 객체를 대상 클래스에 주입 |
| `@ExtendWith(MockitoExtension.class)` | JUnit 5에서 Mockito 사용 활성화 |

---

## 2. 테스트 코드 기본 문법 가이드 (JUnit 5 + Mockito + AssertJ)

테스트 코드는 결국 세 가지 라이브러리 조합으로 짜입니다.
**JUnit 5(테스트 실행)** + **Mockito(가짜 객체)** + **AssertJ(검증)** 입니다. 각각의 핵심 문법을 정리합니다.

### 2.1 의존성 설정 (build.gradle)

```groovy
dependencies {
    // spring-boot-starter-test에 JUnit5, Mockito, AssertJ, MockMvc 등이 모두 포함됨
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()   // JUnit 5 사용 선언
}
```

### 2.2 JUnit 5 핵심 어노테이션

```java
class CalculatorTest {

    private Calculator calculator;

    @BeforeAll                                     // 클래스 전체에서 딱 1번 실행 (static)
    static void beforeAll() {
        System.out.println("테스트 시작 전 한 번만 실행");
    }

    @BeforeEach                                    // 각 @Test 실행 직전에 매번 실행
    void setUp() {
        calculator = new Calculator();             // 매 테스트마다 상태를 초기화 → 테스트 격리
    }

    @Test                                          // 이 메서드가 테스트 메서드임을 표시
    @DisplayName("1 + 2는 3이다")                  // 테스트 리포트에 표시될 사람이 읽기 쉬운 이름
    void add_oneAndTwo_returnsThree() {
        int result = calculator.add(1, 2);
        assertThat(result).isEqualTo(3);
    }

    @Test
    @Disabled("아직 구현 안 됨")                    // 일시적으로 테스트 비활성화
    void notImplementedYet() { }

    @AfterEach                                     // 각 @Test 종료 후 매번 실행 (리소스 정리)
    void tearDown() { }

    @AfterAll                                      // 클래스 전체 종료 후 1번 실행 (static)
    static void afterAll() { }
}
```

**왜 이렇게 하는가**
- `@BeforeEach`는 테스트 간 상태 공유를 막아 "어떤 순서로 돌려도 같은 결과"를 보장합니다. 테스트 격리는 신뢰성의 핵심입니다.
- `@DisplayName`은 한글로 의도를 적어두면 비즈니스 요구사항이 곧 테스트 이름이 되어 문서 역할을 합니다.

### 2.3 파라미터화 테스트 (반복되는 케이스를 한 번에)

```java
@ParameterizedTest                                 // 여러 입력값으로 같은 테스트를 반복 실행
@ValueSource(ints = {1, 3, 5, 7, 9})              // 각 값이 차례로 number에 주입됨
@DisplayName("홀수 판별 테스트")
void isOdd_returnsTrue(int number) {
    assertThat(NumberUtils.isOdd(number)).isTrue();
}

@ParameterizedTest
@CsvSource({                                       // 여러 인자를 동시에 주입 (CSV 형식)
        "1, 2, 3",
        "5, 5, 10",
        "-1, 1, 0"
})
@DisplayName("덧셈 검증")
void add(int a, int b, int expected) {
    assertThat(calculator.add(a, b)).isEqualTo(expected);
}
```

**왜 이렇게 하는가**: 같은 로직을 여러 케이스로 검증할 때 코드 중복을 없애고, 누락된 케이스를 한눈에 발견할 수 있습니다.

### 2.4 AssertJ 단언(Assertion) 문법

AssertJ는 `assertThat(actual).isXxx(expected)` 형태로 메서드 체이닝이 가능해 가독성이 매우 좋습니다.

```java
// 기본 타입 검증
assertThat(result).isEqualTo(3);                   // 같다
assertThat(result).isNotEqualTo(0);                // 다르다
assertThat(result).isGreaterThan(0);               // 크다
assertThat(result).isBetween(1, 10);               // 범위 안에 있다

// 문자열 검증
assertThat(name).isEqualTo("홍길동");
assertThat(name).startsWith("홍").endsWith("동").contains("길");  // 체이닝
assertThat(name).isNotBlank();

// boolean / null 검증
assertThat(active).isTrue();
assertThat(user).isNotNull();
assertThat(deletedAt).isNull();

// 컬렉션 검증
assertThat(users).hasSize(3);                                    // 크기
assertThat(users).contains(user1, user2);                        // 포함
assertThat(users).extracting("name").containsExactly("A", "B");  // 특정 필드만 추출해 검증
assertThat(users).allMatch(u -> u.getAge() >= 18);               // 모두 조건 만족

// 객체 필드 검증
assertThat(user)
        .extracting("name", "email")
        .containsExactly("홍길동", "hong@test.com");

// 예외 검증
assertThatThrownBy(() -> userService.getUserById(999L))
        .isInstanceOf(UserNotFoundException.class)
        .hasMessageContaining("사용자를 찾을 수 없습니다");

// 예외가 발생하지 않아야 할 때
assertThatCode(() -> userService.getUserById(1L))
        .doesNotThrowAnyException();
```

**왜 AssertJ인가**: JUnit 기본 `assertEquals`보다 IDE 자동완성이 잘 되고, 실패 메시지가 더 친절하며, 체이닝으로 의도가 명확해집니다.

### 2.5 Mockito 핵심 문법

```java
@ExtendWith(MockitoExtension.class)                // JUnit5에서 Mockito 어노테이션 활성화
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void mockitoSyntaxExamples() {
        User mockUser = new User(1L, "홍길동", "hong@test.com");

        // 1. Stubbing: "이렇게 호출되면 이걸 반환해라" 라고 시나리오 설정
        given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));
        // 같은 의미의 다른 문법 (when().thenReturn())
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        // 2. 예외를 던지도록 설정
        given(userRepository.findById(999L))
                .willThrow(new RuntimeException("DB 에러"));

        // 3. 호출마다 다른 값 반환 (순차적)
        given(userRepository.count())
                .willReturn(1L)                    // 첫 번째 호출
                .willReturn(2L);                   // 두 번째 호출

        // 4. 인자 매처(ArgumentMatcher) 사용 - 어떤 값이 들어와도 매칭
        given(userRepository.findById(anyLong())).willReturn(Optional.of(mockUser));
        given(userRepository.save(any(User.class))).willReturn(mockUser);

        // 5. Verify: 메서드가 호출됐는지 검증
        verify(userRepository).findById(1L);              // 1번 호출됐는지 (기본값)
        verify(userRepository, times(2)).findById(1L);    // 정확히 2번
        verify(userRepository, never()).delete(any());    // 한 번도 호출되지 않음
        verify(userRepository, atLeastOnce()).findById(1L);

        // 6. ArgumentCaptor: 호출 시 어떤 인자가 전달됐는지 잡아내기
        ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
        verify(userRepository).save(captor.capture());
        User capturedUser = captor.getValue();
        assertThat(capturedUser.getName()).isEqualTo("홍길동");
    }
}
```

**Mockito 사용 시 주의사항**
- `final` 클래스나 `static` 메서드는 기본적으로 모킹 불가합니다 (mockito-inline 추가하면 가능).
- 스터빙하지 않은 메서드는 기본값(0, null, 빈 리스트)을 반환합니다.
- `verify`를 과하게 쓰면 리팩토링 시 테스트가 자주 깨집니다. **상태 검증(assertThat)이 우선, 행위 검증(verify)은 꼭 필요할 때만** 사용하세요.

### 2.6 테스트 메서드 네이밍 컨벤션

가독성과 의도 전달을 위해 일관된 이름 규칙을 따르세요.

```java
// 패턴 1: 메서드명_상황_기대결과
void getUserById_existingId_returnsUser()
void getUserById_nonExistingId_throwsException()
void createUser_duplicateEmail_throwsException()

// 패턴 2: should_기대결과_when_상황
void should_returnUser_when_idExists()
void should_throwException_when_emailDuplicated()
```

여기에 `@DisplayName`으로 한글 설명을 덧붙이면 가장 좋습니다.

```java
@Test
@DisplayName("이메일이 중복되면 DuplicateEmailException이 발생한다")
void createUser_duplicateEmail_throwsException() { ... }
```

---

## 3. Given-When-Then 작성 가이드

테스트 코드는 **"준비 → 실행 → 검증"** 의 흐름을 가집니다. 이걸 명시적으로 주석으로 구분해 적는 것이 Given-When-Then 패턴이며, BDD(Behavior-Driven Development)에서 유래했습니다.

### 3.1 각 단계의 역할

| 단계 | 역할 | 들어가야 할 것 |
|------|------|---------------|
| **Given** | 테스트의 사전 조건과 입력 데이터 준비 | 테스트 데이터 생성, Mock 스터빙, 초기 상태 설정 |
| **When** | 실제로 테스트하려는 동작 1번 실행 | 테스트 대상 메서드 호출 (보통 1줄) |
| **Then** | 결과 검증 | assertThat, verify, 예외 검증 |

### 3.2 좋은 예시 vs 나쁜 예시

**나쁜 예시 - 구획이 섞여 있음**

```java
@Test
void createUser_test() {
    UserCreateRequest request = new UserCreateRequest("홍길동", "hong@test.com");
    given(userRepository.existsByEmail("hong@test.com")).willReturn(false);
    UserResponse result = userService.createUser(request);
    User savedUser = User.builder().id(1L).name("홍길동").build();   // 이건 given에 있어야 함
    given(userRepository.save(any())).willReturn(savedUser);          // 이미 when 다음인데 given 등장
    assertThat(result.getName()).isEqualTo("홍길동");
}
```

**좋은 예시 - 명확한 3단 구조**

```java
@Test
@DisplayName("사용자 생성 - 정상 케이스")
void createUser_success() {
    // given: 사전 조건과 입력값을 모두 준비
    UserCreateRequest request = new UserCreateRequest("홍길동", "hong@test.com");
    User savedUser = User.builder().id(1L).name("홍길동").email("hong@test.com").build();

    given(userRepository.existsByEmail("hong@test.com")).willReturn(false);
    given(userRepository.save(any(User.class))).willReturn(savedUser);

    // when: 테스트 대상 메서드 호출 (딱 한 줄)
    UserResponse result = userService.createUser(request);

    // then: 결과 검증
    assertThat(result.getId()).isEqualTo(1L);
    assertThat(result.getName()).isEqualTo("홍길동");
    verify(userRepository).save(any(User.class));
}
```

### 3.3 Given-When-Then 작성 7가지 규칙

**규칙 1. 주석 `// given`, `// when`, `// then` 을 반드시 적습니다.**
가독성이 비약적으로 올라가고, 다른 사람이 테스트를 읽을 때 "지금 보고 있는 줄이 어느 단계인지" 즉시 파악할 수 있습니다.

**규칙 2. When은 가능한 한 1줄로 작성합니다.**
테스트하려는 동작이 무엇인지 명확해집니다. When이 길어지면 "테스트 대상이 너무 많거나, 도우미 메서드가 필요하다"는 신호입니다.

```java
// 나쁨 - 무엇을 테스트하는지 모호함
// when
userService.validateUser(request);
User user = userService.findOrCreate(request);
UserResponse response = userService.toResponse(user);

// 좋음 - 한 줄로 의도가 명확
// when
UserResponse response = userService.createUser(request);
```

**규칙 3. Given은 "이 테스트에 꼭 필요한 것"만 넣습니다.**
관련 없는 데이터를 잔뜩 만들면 테스트 의도가 흐려집니다. 도메인 필드가 많다면 빌더 패턴이나 픽스처(Fixture)를 활용하세요.

```java
// 픽스처를 사용해 핵심 차이만 강조하는 예시
User adminUser = UserFixture.aUser().withRole(Role.ADMIN).build();
User normalUser = UserFixture.aUser().withRole(Role.USER).build();
```

**규칙 4. Then은 검증 이외의 코드가 없어야 합니다.**
Then 단계에서 새로운 데이터 가공이나 비즈니스 로직을 수행하면 테스트가 무엇을 검증하는지 흐려집니다.

**규칙 5. 한 테스트에는 하나의 시나리오만 검증합니다 (One assertion concept per test).**
검증문은 여러 개여도 되지만 모두 "같은 시나리오의 다른 측면"이어야 합니다. 여러 시나리오를 한 테스트에 넣으면 실패 원인 파악이 어렵습니다.

```java
// 나쁨 - 두 시나리오가 섞여 있음
@Test
void userTest() {
    // 성공 케이스
    UserResponse ok = userService.createUser(validRequest);
    assertThat(ok).isNotNull();

    // 실패 케이스
    assertThatThrownBy(() -> userService.createUser(invalidRequest))
            .isInstanceOf(IllegalArgumentException.class);
}

// 좋음 - 시나리오별로 분리
@Test void createUser_validRequest_returnsUser() { ... }
@Test void createUser_invalidEmail_throwsException() { ... }
```

**규칙 6. Given-When-Then 사이에 빈 줄을 넣어 시각적으로 구분합니다.**

**규칙 7. AAA(Arrange-Act-Assert) 패턴과 동일합니다.**
일부 팀은 `// given/when/then` 대신 `// arrange/act/assert`를 씁니다. 의미는 동일하니 팀 컨벤션을 따르면 됩니다.

### 3.4 복잡한 시나리오에서의 Given-When-Then

상황이 복잡하다면 단계 안에서 더 세분화할 수 있습니다.

```java
@Test
@DisplayName("주문 생성 - 재고 부족 시 예외 발생")
void createOrder_outOfStock_throwsException() {
    // given - 사용자 준비
    User buyer = UserFixture.aUser().build();
    given(userRepository.findById(1L)).willReturn(Optional.of(buyer));

    // given - 상품 준비 (재고 0개)
    Product product = ProductFixture.aProduct().withStock(0).build();
    given(productRepository.findById(10L)).willReturn(Optional.of(product));

    // given - 주문 요청
    OrderRequest request = new OrderRequest(1L, 10L, 1);

    // when & then
    assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(OutOfStockException.class)
            .hasMessageContaining("재고가 부족합니다");

    // then - 주문이 저장되지 않았음을 보장
    verify(orderRepository, never()).save(any());
}
```

### 3.5 When과 Then을 합쳐도 되는 경우

예외 검증처럼 "실행과 검증이 자연스럽게 한 표현으로 묶이는 경우" 는 `// when & then` 으로 표기해도 좋습니다.

```java
// when & then
assertThatThrownBy(() -> userService.getUserById(999L))
        .isInstanceOf(UserNotFoundException.class);
```

---

## 4. Spring MVC 패턴에서 REST API 테스트 방법

Spring MVC에서 REST API 테스트는 보통 `MockMvc`를 사용합니다. 실제 톰캣 서버를 띄우지 않고 가짜 요청을 보내 Controller를 테스트할 수 있습니다.

### 4.1 테스트 대상 코드 (Controller)

```java
// UserController.java
@RestController                                    // 이 클래스가 REST API Controller임을 Spring에 알림
@RequestMapping("/api/users")                      // 이 컨트롤러의 모든 엔드포인트는 /api/users 로 시작
@RequiredArgsConstructor                           // Lombok: final 필드에 대해 생성자 자동 생성
public class UserController {

    private final UserService userService;         // Service 계층 의존성

    @GetMapping("/{id}")                           // GET /api/users/{id}
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUserById(id);
        return ResponseEntity.ok(response);                    // HTTP 200
    }

    @PostMapping                                   // POST /api/users
    public ResponseEntity<UserResponse> createUser(@RequestBody @Valid UserCreateRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);  // HTTP 201
    }
}
```

**왜 이렇게 작성하는가**
- Controller는 "요청 파싱 → Service 호출 → 응답 포장"의 역할만 합니다. 책임이 명확해서 테스트하기 쉽습니다.
- `@Valid`로 입력값 검증을 어노테이션 기반으로 표준화하면 비즈니스 로직이 오염되지 않습니다.

### 4.2 Controller 테스트 코드

```java
// UserControllerTest.java
@WebMvcTest(UserController.class)                  // UserController만 로드 (빠른 테스트)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;                       // 가짜 HTTP 요청 도구

    @MockBean                                      // Spring Context의 UserService를 Mock으로 교체
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;             // 객체 ↔ JSON 변환

    @Test
    @DisplayName("GET /api/users/{id} - 사용자 조회 성공")
    void getUser_success() throws Exception {
        // given
        Long userId = 1L;
        UserResponse mockResponse = new UserResponse(1L, "홍길동", "hong@test.com");
        given(userService.getUserById(userId)).willReturn(mockResponse);

        // when & then
        mockMvc.perform(get("/api/users/{id}", userId)
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())                  // 200 검증
                .andExpect(jsonPath("$.id").value(1L))       // 응답 JSON 필드 검증
                .andExpect(jsonPath("$.name").value("홍길동"))
                .andExpect(jsonPath("$.email").value("hong@test.com"))
                .andDo(print());                             // 디버깅용 출력

        verify(userService, times(1)).getUserById(userId);
    }

    @Test
    @DisplayName("POST /api/users - 사용자 생성 성공")
    void createUser_success() throws Exception {
        // given
        UserCreateRequest request = new UserCreateRequest("홍길동", "hong@test.com");
        UserResponse mockResponse = new UserResponse(1L, "홍길동", "hong@test.com");
        given(userService.createUser(any(UserCreateRequest.class))).willReturn(mockResponse);

        // when & then
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))   // 객체→JSON
                .andExpect(status().isCreated())             // 201 검증
                .andExpect(jsonPath("$.id").value(1L))
                .andExpect(jsonPath("$.name").value("홍길동"));
    }

    @Test
    @DisplayName("POST /api/users - 이메일 형식이 잘못되면 400 반환")
    void createUser_invalidEmail_returns400() throws Exception {
        // given
        UserCreateRequest invalidRequest = new UserCreateRequest("홍길동", "not-an-email");

        // when & then
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest());         // @Valid 검증 실패
    }
}
```

### 4.3 MockMvc 주요 문법 정리

```java
// HTTP 메서드
mockMvc.perform(get("/api/users/1"));
mockMvc.perform(post("/api/users").content(jsonBody));
mockMvc.perform(put("/api/users/1").content(jsonBody));
mockMvc.perform(delete("/api/users/1"));

// 요청 파라미터/헤더
mockMvc.perform(get("/api/users")
        .param("page", "0")
        .param("size", "10")
        .header("Authorization", "Bearer token"));

// 응답 상태 검증
.andExpect(status().isOk())             // 200
.andExpect(status().isCreated())        // 201
.andExpect(status().isBadRequest())     // 400
.andExpect(status().isNotFound())       // 404

// 응답 본문 검증 (JsonPath)
.andExpect(jsonPath("$.name").value("홍길동"))
.andExpect(jsonPath("$.users[0].id").value(1))
.andExpect(jsonPath("$.users").isArray())
.andExpect(jsonPath("$.users.length()").value(3))

// 헤더 검증
.andExpect(header().string("Location", "/api/users/1"))
```

**왜 이렇게 작성하는가**
- `@WebMvcTest`는 Controller 관련 빈만 로드해서 `@SpringBootTest`보다 훨씬 빠릅니다.
- `MockMvc`는 실제 서버 없이 HTTP 요청을 시뮬레이션해서, CI/CD에서 별도 인프라 없이 테스트 가능합니다.
- `jsonPath`로 응답 본문을 정밀하게 검증하면 API 스펙 변경을 즉시 감지할 수 있습니다.

---

## 5. `@Mock` vs `@MockBean` 차이점

> 학생들이 가장 헷갈려하는 부분입니다. 핵심은 **"Spring Context를 띄우느냐 안 띄우느냐"** 입니다.

### 5.1 비교표

| 구분 | `@Mock` | `@MockBean` |
|------|---------|-------------|
| 소속 라이브러리 | Mockito | Spring Boot Test |
| Spring Context 필요 | ❌ 필요 없음 | ✅ 필요함 |
| 사용 환경 | 순수 단위 테스트 | Spring 통합 테스트 (`@WebMvcTest`, `@SpringBootTest`) |
| 동작 방식 | 그냥 Mock 객체를 변수에 넣음 | Spring Context의 실제 Bean을 Mock으로 교체 |
| 주입 방식 | `@InjectMocks`로 수동 주입 | Spring이 자동으로 의존성 주입 |
| 속도 | 매우 빠름 | 상대적으로 느림 |

### 5.2 `@Mock` 사용 예시 - Service 단위 테스트

```java
@ExtendWith(MockitoExtension.class)                // Mockito 어노테이션 활성화
class UserServiceTest {

    @Mock                                           // 가짜 Repository (Spring과 무관)
    private UserRepository userRepository;

    @InjectMocks                                    // 위 @Mock들을 이 객체에 주입
    private UserService userService;

    @Test
    void getUserById_success() {
        // given
        User mockUser = new User(1L, "홍길동", "hong@test.com");
        given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));

        // when
        UserResponse result = userService.getUserById(1L);

        // then
        assertThat(result.getName()).isEqualTo("홍길동");
        verify(userRepository).findById(1L);
    }
}
```

### 5.3 `@MockBean` 사용 예시 - Controller 테스트

```java
@WebMvcTest(UserController.class)                  // Spring Context 일부 로드
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean                                       // Spring Context의 UserService 빈을 Mock으로 교체
    private UserService userService;
    // ...
}
```

### 5.4 한 줄 정리

- **Spring을 띄우지 않는 순수 단위 테스트 → `@Mock` + `@InjectMocks`**
- **Spring을 띄우는 통합/슬라이스 테스트 → `@MockBean`**

> 참고로 질문하신 `@mocking`이라는 어노테이션은 실제로 존재하지 않습니다. 아마 `@MockBean`을 의미하신 것으로 보고 정리했습니다. Spring Boot 3.4부터는 `@MockBean`이 deprecated되고 `@MockitoBean`이 권장됩니다.

---

## 6. DB 데이터를 모킹(Mocking)으로 테스트하는 방법

DB 모킹의 핵심 아이디어는 **"Repository를 Mock으로 만들어서 진짜 DB에 안 가게 한다"** 입니다.

### 6.1 테스트 대상 - Service 계층

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public UserResponse getUserById(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다: " + id));
        return UserResponse.from(user);
    }

    @Transactional
    public UserResponse createUser(UserCreateRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException("이미 존재하는 이메일입니다: " + request.getEmail());
        }
        User newUser = User.builder()
                .name(request.getName())
                .email(request.getEmail())
                .build();
        User savedUser = userRepository.save(newUser);
        return UserResponse.from(savedUser);
    }
}
```

### 6.2 Repository를 모킹한 Service 테스트

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("ID로 사용자 조회 - 성공")
    void getUserById_success() {
        // given
        Long userId = 1L;
        User mockUser = User.builder()
                .id(userId).name("홍길동").email("hong@test.com").build();
        given(userRepository.findById(userId)).willReturn(Optional.of(mockUser));

        // when
        UserResponse result = userService.getUserById(userId);

        // then
        assertThat(result.getId()).isEqualTo(userId);
        assertThat(result.getName()).isEqualTo("홍길동");
        verify(userRepository).findById(userId);
    }

    @Test
    @DisplayName("ID로 사용자 조회 - 존재하지 않으면 예외 발생")
    void getUserById_notFound_throwsException() {
        // given
        given(userRepository.findById(999L)).willReturn(Optional.empty());

        // when & then
        assertThatThrownBy(() -> userService.getUserById(999L))
                .isInstanceOf(UserNotFoundException.class)
                .hasMessageContaining("사용자를 찾을 수 없습니다");
    }

    @Test
    @DisplayName("사용자 생성 - 이메일 중복 시 예외 발생")
    void createUser_duplicateEmail_throwsException() {
        // given
        UserCreateRequest request = new UserCreateRequest("홍길동", "hong@test.com");
        given(userRepository.existsByEmail("hong@test.com")).willReturn(true);

        // when & then
        assertThatThrownBy(() -> userService.createUser(request))
                .isInstanceOf(DuplicateEmailException.class);

        // then - save가 절대 호출되지 않아야 함
        verify(userRepository, never()).save(any(User.class));
    }

    @Test
    @DisplayName("사용자 생성 - 성공")
    void createUser_success() {
        // given
        UserCreateRequest request = new UserCreateRequest("홍길동", "hong@test.com");
        User savedUser = User.builder().id(1L).name("홍길동").email("hong@test.com").build();

        given(userRepository.existsByEmail("hong@test.com")).willReturn(false);
        given(userRepository.save(any(User.class))).willReturn(savedUser);

        // when
        UserResponse result = userService.createUser(request);

        // then
        assertThat(result.getId()).isEqualTo(1L);
        verify(userRepository).existsByEmail("hong@test.com");
        verify(userRepository).save(any(User.class));
    }
}
```

### 6.3 DB 모킹의 3가지 핵심 패턴

**패턴 1. 데이터 조회 모킹**
```java
given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));     // 존재
given(userRepository.findById(1L)).willReturn(Optional.empty());          // 없음
```

**패턴 2. 데이터 저장 모킹**
```java
given(userRepository.save(any(User.class))).willReturn(savedUserWithId);
```

**패턴 3. 예외 상황 모킹**
```java
given(userRepository.findById(1L))
        .willThrow(new DataAccessException("DB 연결 실패") {});
```

### 6.4 모킹 대신 실제 DB로 테스트가 필요할 때 - `@DataJpaTest`

순수 모킹은 빠르지만 "내 JPQL이 진짜 동작하는가?"는 검증할 수 없습니다. 이때는 인메모리 DB(H2)를 사용합니다.

```java
@DataJpaTest                                       // JPA 빈만 로드, 인메모리 DB 자동 설정
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("이메일 존재 여부 확인")
    void existsByEmail_works() {
        // given
        userRepository.save(User.builder().name("홍길동").email("hong@test.com").build());

        // when
        boolean exists = userRepository.existsByEmail("hong@test.com");
        boolean notExists = userRepository.existsByEmail("none@test.com");

        // then
        assertThat(exists).isTrue();
        assertThat(notExists).isFalse();
    }
}
```

**왜 DB 모킹을 하는가**
- 실제 DB를 띄울 필요가 없어 테스트가 수 밀리초 안에 끝납니다.
- DB 상태에 따라 테스트가 깨지지 않습니다 (테스트 격리).
- 비즈니스 로직 자체에 집중할 수 있습니다.
- 단, 쿼리 자체의 검증은 `@DataJpaTest`나 Testcontainers로 별도 수행해야 합니다. 모킹과 통합 테스트는 **상호 보완 관계** 입니다.

---

## 7. 로직이 바뀌면 테스트도 바꿔야 하는데, 그래도 TDD가 중요한 이유

학생들이 자주 하는 불만이 있습니다.

> "코드 한 번 바꾸면 테스트 코드 10개를 같이 고쳐야 해요. 차라리 안 쓰는 게 빠르지 않나요?"

이 의문에 정직하게 답변하자면 **"맞습니다. 단기적으로는 더 느립니다."** 그럼에도 불구하고 TDD는 다음과 같은 이유로 중요합니다.

### 7.1 변경의 두려움을 없애줍니다 (Confidence to Change)

테스트가 없는 코드는 시간이 지날수록 "건드리기 무서운 코드"가 됩니다. 잘 돌아가니까 그냥 둡니다. 이러면 기술 부채가 쌓이고, 결국 시스템이 굳어버립니다. 잘 만든 테스트가 있으면 "수정해도 기존 동작이 깨지지 않는다"는 확신을 가지고 리팩토링과 기능 추가를 할 수 있습니다.

### 7.2 테스트가 바뀌어야 한다는 것 자체가 "스펙이 바뀌었다"는 신호입니다

로직이 바뀌었는데 테스트를 안 고쳐도 된다면, 그 테스트는 처음부터 무의미했거나 너무 약했다는 뜻입니다. 테스트가 같이 바뀐다는 건 "요구사항 변화가 코드에 정확히 반영됐다"는 증거이며, 이 자체가 **살아있는 문서(Living Documentation)** 역할을 합니다. 새로 합류한 개발자에게 테스트는 최고의 사용 설명서가 됩니다.

### 7.3 설계가 좋아집니다

TDD를 하다 보면 "이 클래스를 테스트하기가 너무 어렵네?"라는 순간이 옵니다. 그건 보통 **설계가 잘못됐다는 신호** 입니다. 의존성이 너무 많거나, 책임이 한 클래스에 몰려 있거나, 결합도가 높다는 뜻입니다. 테스트하기 쉬운 코드는 자연스럽게 SOLID 원칙을 따르게 됩니다.

### 7.4 버그 발견 비용을 극적으로 줄입니다

버그는 늦게 발견될수록 수정 비용이 기하급수적으로 커집니다. 개발 단계에서 잡으면 분 단위, QA에서 잡으면 시간 단위, 운영에서 잡으면 일 단위, 고객이 발견하면 신뢰도까지 잃습니다. TDD는 가장 빠른 시점인 "코드를 작성하는 그 순간"에 버그를 잡아냅니다.

### 7.5 회귀 버그(Regression)를 막아줍니다

기능 A를 고쳤는데 엉뚱한 기능 B가 깨지는 경험, 한 번쯤 해보셨을 겁니다. 테스트가 충분하면 CI에서 즉시 이런 회귀를 잡아냅니다. 운영에 나가기 전에 막을 수 있다는 것만으로도 TDD는 충분히 비용을 정당화합니다.

### 7.6 "테스트도 같이 바꿔야 한다"는 부담을 줄이는 실전 팁

- **행위(behavior)를 테스트하지, 구현(implementation)을 테스트하지 마세요.** "어떻게"가 아니라 "무엇을" 검증해야 내부 구현이 바뀌어도 테스트가 살아남습니다.
- **Mock 검증을 과하게 하지 마세요.** `verify`를 모든 호출에 걸면 리팩토링이 어려워집니다. 정말 중요한 상호작용만 검증하세요.
- **테스트 데이터 빌더 패턴**(예: `UserFixture.aUser().withName("홍길동").build()`)을 사용해 중복을 줄이세요. 도메인 모델이 바뀌어도 한 곳만 고치면 됩니다.
- **Given-When-Then 구조**를 일관되게 유지하세요. 가독성이 곧 유지보수성입니다.

### 7.7 마지막 정리

> TDD는 **"지금 조금 느려지고, 평생 빨라지는"** 투자입니다.
>
> 테스트를 같이 고치는 비용은, 운영 환경에서 새벽 3시에 장애 대응하는 비용에 비하면 무시할 수 있을 만큼 작습니다. 안정적인 서비스는 천재 개발자가 만드는 것이 아니라, **꾸준한 테스트 문화** 가 만듭니다.

---

## 부록: 추천 학습 순서

1. JUnit 5 기본 문법 (`@Test`, `@BeforeEach`, AssertJ)
2. Mockito 기본 (`@Mock`, `given().willReturn()`, `verify`)
3. Given-When-Then 작성 연습
4. Spring 단위 테스트 (`@WebMvcTest`, `@DataJpaTest`)
5. Spring 통합 테스트 (`@SpringBootTest`)
6. 테스트 더블 종류 구분 (Dummy / Stub / Mock / Spy / Fake)
7. Testcontainers로 실제 DB 통합 테스트
8. 테스트 커버리지 측정 (JaCoCo)

---
