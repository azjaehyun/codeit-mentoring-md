

# ECS Fargate 한눈에 보기 🎯

## 0. 큰 그림부터: 이게 다 뭐죠?

```mermaid
flowchart TB
    Q1["📄 Task Definition<br/>= 요리 레시피<br/>(어떻게 만들지 적어둔 종이)"]
    Q2["📦 Task<br/>= 완성된 요리<br/>(실제로 돌아가는 컨테이너)"]
    Q3["🍽️ Service<br/>= 식당 매니저<br/>(요리를 몇 개 유지할지 관리)"]
    Q4["🚪 ALB<br/>= 안내 데스크<br/>(손님을 빈 자리로 안내)"]

    Q1 -->|레시피대로 실행| Q2
    Q3 -->|개수 관리 & 자동 교체| Q2
    Q4 -->|트래픽 분배| Q2
```

> **한 문장 요약:** 레시피(Task Definition)를 보고 → 매니저(Service)가 요리(Task)를 만들고 → 안내데스크(ALB)가 손님을 나눠 보냅니다.

---

## 1. Task Definition (레시피) 📄

Task Definition은 컨테이너 애플리케이션을 실행하기 위한 청사진(JSON)입니다. Fargate에서는 몇 가지 필수 제약이 있는데, 핵심은 `networkMode`가 반드시 `awsvpc`여야 하고, CPU/Memory가 task 레벨에서 정해진 조합 중 하나로 지정되어야 한다는 점입니다.

### 레시피 구조 그림

```mermaid
flowchart TB
    subgraph TD["📄 Task Definition (레시피)"]
        direction TB
        A["family: my-web-app<br/>👉 레시피 이름"]
        B["cpu / memory<br/>👉 자원 크기"]
        C["networkMode: awsvpc ⚠️필수<br/>👉 네트워크 방식"]
        D["executionRoleArn<br/>👉 에이전트용 권한"]
        E["taskRoleArn<br/>👉 앱 코드용 권한"]
        F["containerDefinitions<br/>👉 컨테이너 목록"]
    end

    F --> G["image: ECR 주소<br/>portMappings: 8080<br/>healthCheck<br/>logConfiguration<br/>secrets"]
```

### 핵심 옵션 자세히 보기

**task 레벨 (최상위) 필드**

| 옵션 | 설명 | Fargate에서의 주의점 |
|------|------|---------------------|
| `family` | 태스크 정의의 논리적 이름. 같은 family 안에서 revision이 1, 2, 3...으로 증가 | 필수. 배포 시 항상 새 revision 생성 |
| `requiresCompatibilities` | `["FARGATE"]` 지정 시 등록 단계에서 Fargate 호환성 검증 | 명시 권장 |
| `networkMode` | 네트워크 모드 | **반드시 `awsvpc`** (ENI가 task마다 할당됨) |
| `cpu` / `memory` | task 전체에 할당할 리소스 | 아래 조합표 중 하나만 가능 (필수) |
| `executionRoleArn` | ECS 에이전트가 사용하는 역할 | ECR pull, 로그 전송, Secrets 조회용 |
| `taskRoleArn` | 컨테이너 안의 애플리케이션이 사용하는 역할 | S3/DynamoDB 등 앱이 호출하는 API용 |

**Fargate CPU/Memory 유효 조합 (Linux)** — 아무 숫자나 못 넣고, 아래 짝꿍만 가능합니다 ⚠️

| CPU | Memory 범위 |
|-----|-------------|
| 256 (.25 vCPU) | 512 MiB, 1 GB, 2 GB |
| 512 (.5 vCPU) | 1~4 GB |
| 1024 (1 vCPU) | 2~8 GB |
| 2048 (2 vCPU) | 4~16 GB (1GB 단위) |
| 4096 (4 vCPU) | 8~30 GB (1GB 단위) |

**컨테이너 레벨 핵심 포인트**

`portMappings`에서 awsvpc 모드일 때는 `containerPort`만 의미가 있습니다. `hostPort`는 무시되며 컨테이너 포트와 동일하게 자동 매핑됩니다.

`healthCheck`는 컨테이너 자체의 헬스 체크입니다. 이것은 ALB 타겟 그룹 헬스 체크와는 별개이며, 둘 다 통과해야 트래픽이 안정적으로 흐릅니다. `startPeriod`를 충분히 줘서(부팅 느린 앱은 60~120초) 초기 기동 중 불필요한 재시작을 막는 게 중요합니다.

`secrets`는 평문 `environment` 대신 민감 정보를 Secrets Manager / SSM Parameter Store에서 주입할 때 사용합니다. 이 기능을 쓰려면 **execution role**에 권한이 필요합니다.

`logConfiguration`에서 `awslogs-create-group: "true"`를 쓰면 로그 그룹을 자동 생성하는데, 이때 execution role에 `logs:CreateLogGroup` 권한이 있어야 합니다.

### 실제 레시피 예시 (JSON)

```json
{
  "family": "my-web-app",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::111122223333:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::111122223333:role/myAppTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "111122223333.dkr.ecr.ap-northeast-2.amazonaws.com/my-web-app:1.0.0",
      "essential": true,
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30, "timeout": 5, "retries": 3, "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-web-app",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
```


Spring Boot 환경 변수까지 추가하고, 각 항목에 주석을 모두 달아드리겠습니다.

> ⚠️ **먼저 알아둘 점:** 표준 JSON은 주석(`//`)을 지원하지 않습니다. 그래서 ECS에 실제로 등록할 때는 주석을 빼야 합니다. 아래는 **학습용 주석 버전(JSONC)**과 **실제 배포용(순수 JSON)** 두 가지로 드립니다.

---

## 1. 학습용 — 주석 달린 버전 (JSONC)

```jsonc
{
  // ── 태스크 정의의 논리적 이름. 같은 family로 등록할 때마다 revision(1,2,3...) 증가
  "family": "my-web-app",

  // ── Fargate 호환성 검증. 잘못된 옵션이 있으면 등록 단계에서 에러로 알려줌
  "requiresCompatibilities": ["FARGATE"],

  // ── Fargate는 반드시 awsvpc (task마다 ENI + 고유 사설 IP 할당)
  "networkMode": "awsvpc",

  // ── task 전체 CPU. 1024 = 1 vCPU (Fargate는 정해진 조합만 가능)
  "cpu": "1024",

  // ── task 전체 메모리. 2048 MiB = 2GB (cpu 1024와 짝이 맞아야 함)
  "memory": "2048",

  // ── 에이전트용 역할: ECR 이미지 pull, 로그 전송, Secrets 조회에 사용
  "executionRoleArn": "arn:aws:iam::111122223333:role/ecsTaskExecutionRole",

  // ── 앱 코드용 역할: 컨테이너 안 Spring Boot가 S3/DynamoDB 등 호출할 때 사용
  "taskRoleArn": "arn:aws:iam::111122223333:role/myAppTaskRole",

  "containerDefinitions": [
    {
      // ── 컨테이너 이름. GitHub Actions의 container-name, ALB 연결 시 이 이름을 참조
      "name": "web",

      // ── ECR 이미지 주소. 태그는 latest 대신 버전/commit SHA 권장(롤백 용이)
      "image": "111122223333.dkr.ecr.ap-northeast-2.amazonaws.com/my-web-app:1.0.0",

      // ── true: 이 컨테이너가 죽으면 task 전체 중단 (메인 앱은 보통 true)
      "essential": true,

      // ── awsvpc에서는 containerPort만 의미 있음. Spring Boot 기본 8080
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],

      // ── 평문 환경변수: 민감하지 않은 설정값만 (비밀번호 X → secrets 사용)
      "environment": [
        // 활성 프로파일. application-prod.yml을 읽게 됨
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },

        // 서버 포트(application.yml의 server.port와 일치). portMappings와 맞춤
        { "name": "SERVER_PORT", "value": "8080" },

        // JVM 옵션. Fargate 메모리(2GB)에 맞춰 힙 설정 + 컨테이너 인식 옵션
        { "name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75.0 -Duser.timezone=Asia/Seoul" },

        // DB 접속 정보 중 민감하지 않은 부분(호스트/포트/스키마)은 평문 OK
        { "name": "SPRING_DATASOURCE_URL", "value": "jdbc:postgresql://mydb.xxxx.ap-northeast-2.rds.amazonaws.com:5432/appdb" },
        { "name": "SPRING_DATASOURCE_USERNAME", "value": "appuser" },

        // 로그 레벨, 기타 앱 설정
        { "name": "LOGGING_LEVEL_ROOT", "value": "INFO" }
      ],

      // ── 민감 정보: Secrets Manager / SSM에서 안전하게 주입 (로그/콘솔에 노출 안 됨)
      //    ⚠️ 이 기능을 쓰려면 executionRole에 secretsmanager:GetSecretValue 권한 필요
      "secrets": [
        // DB 비밀번호 → Secrets Manager의 특정 키(:password::)를 주입
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:111122223333:secret:prod/db-AbCdEf:password::"
        },
        // JWT 시크릿 → SSM Parameter Store(SecureString)에서 주입
        {
          "name": "APP_JWT_SECRET",
          "valueFrom": "arn:aws:ssm:ap-northeast-2:111122223333:parameter/my-web-app/jwt-secret"
        }
      ],

      // ── 컨테이너 자체 헬스체크 (ALB 헬스체크와는 별개! 둘 다 통과해야 안정적)
      "healthCheck": {
        // Spring Boot Actuator의 health 엔드포인트 호출. 실패 시 exit 1
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,    // 30초마다 검사
        "timeout": 5,      // 5초 안에 응답 없으면 실패
        "retries": 3,      // 3번 연속 실패하면 unhealthy
        "startPeriod": 60  // 부팅 유예 60초(이 동안 실패는 카운트 안 함). 느리면 늘리기
      },

      // ── 로그를 CloudWatch Logs로 전송 (awslogs 드라이버)
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-web-app",        // 로그 그룹 이름
          "awslogs-region": "ap-northeast-2",         // 로그를 보낼 리전
          "awslogs-stream-prefix": "web",             // 스트림 이름 앞에 붙는 접두어
          "awslogs-create-group": "true"              // 그룹 없으면 자동 생성
          // ⚠️ create-group 쓰려면 executionRole에 logs:CreateLogGroup 권한 필요
        }
      }
    }
  ]
}
```

---

## 2. 실제 배포용 — 순수 JSON (그대로 등록 가능)

```json
{
  "family": "my-web-app",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::111122223333:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::111122223333:role/myAppTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "111122223333.dkr.ecr.ap-northeast-2.amazonaws.com/my-web-app:1.0.0",
      "essential": true,
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
        { "name": "SERVER_PORT", "value": "8080" },
        { "name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75.0 -Duser.timezone=Asia/Seoul" },
        { "name": "SPRING_DATASOURCE_URL", "value": "jdbc:postgresql://mydb.xxxx.ap-northeast-2.rds.amazonaws.com:5432/appdb" },
        { "name": "SPRING_DATASOURCE_USERNAME", "value": "appuser" },
        { "name": "LOGGING_LEVEL_ROOT", "value": "INFO" }
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:111122223333:secret:prod/db-AbCdEf:password::"
        },
        {
          "name": "APP_JWT_SECRET",
          "valueFrom": "arn:aws:ssm:ap-northeast-2:111122223333:parameter/my-web-app/jwt-secret"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-web-app",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "web",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
```

---

## 꼭 알아야 할 포인트 3가지

**environment vs secrets** — Spring Boot는 환경 변수를 `application.yml`보다 우선으로 읽습니다. 예를 들어 `SPRING_DATASOURCE_PASSWORD` 환경 변수는 자동으로 `spring.datasource.password` 설정에 매핑됩니다(점/대문자 ↔ 언더스코어 변환, Relaxed Binding). **비밀번호·토큰 같은 민감 정보는 절대 `environment`에 평문으로 넣지 말고 `secrets`로** 넣으세요. `secrets`로 넣으면 콘솔이나 로그에 값이 노출되지 않습니다.

**curl 이 컨테이너에 있어야 함** — 헬스체크 명령에 `curl`을 쓰는데, 슬림한 베이스 이미지(예: `eclipse-temurin:21-jre-alpine`)에는 curl이 없을 수 있습니다. 그럴 땐 Dockerfile에 `RUN apk add --no-cache curl`을 넣거나, curl 대신 다른 방법을 써야 합니다.

**권한 연결** — `secrets`를 쓰면 **executionRole**에 `secretsmanager:GetSecretValue`와 `ssm:GetParameters`(+ 커스텀 KMS 키면 `kms:Decrypt`) 권한이 있어야 하고, `awslogs-create-group: "true"`를 쓰면 `logs:CreateLogGroup` 권한이 필요합니다. 이 권한들이 없으면 task가 PROVISIONING에서 멈추거나 STOPPED로 떨어집니다.


---

## 2. 두 가지 역할(Role)의 차이 🔑

가장 헷갈리는 부분! **"누가 그 권한을 쓰느냐"**로 구분하세요.

```mermaid
flowchart LR
    subgraph TD["📄 Task Definition"]
      ER["🤖 executionRoleArn<br/><b>에이전트가 씀</b>"]
      TR["📱 taskRoleArn<br/><b>내 앱이 씀</b>"]
    end

    ER -->|이미지 가져오기| ECR[(ECR)]
    ER -->|로그 보내기| CW[(CloudWatch)]
    ER -->|비밀번호 꺼내기| SM[(Secrets Manager)]

    TR -->|앱이 직접 사용| S3[(S3)]
    TR -->|앱이 직접 사용| DDB[(DynamoDB)]
```

| 구분 | 🤖 Execution Role | 📱 Task Role |
|------|------------------|--------------|
| 누가 쓰나 | ECS/Fargate **에이전트** | 컨테이너 안 **내 앱 코드** |
| 언제 쓰나 | task를 **띄울 때** | 앱이 **돌아가는 중** |
| 예시 | ECR pull, 로그, Secrets | S3 업로드, DB 읽기 |

> 💡 **외우는 법:** "준비물 챙기는 건 에이전트(Execution), 실제로 일하는 건 내 앱(Task)"

**신뢰 정책(Trust Policy)** — 두 역할 모두 동일하게, `ecs-tasks.amazonaws.com`이 이 역할을 맡도록 허용합니다:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "ecs-tasks.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Execution Role에는 AWS 관리형 정책 `AmazonECSTaskExecutionRolePolicy`를 붙이면 ECR pull + 로그 전송이 기본 포함됩니다. ARN은 각각 task definition의 `executionRoleArn`, `taskRoleArn` 필드에 문자열로 그대로 넣습니다.

---

## 3. ALB 붙이기 🚪

### 전체 연결 그림

```mermaid
flowchart TB
    Client([🧑 손님]) --> ALB[🚪 ALB<br/>안내 데스크]
    ALB --> TG["🎯 Target Group<br/>⚠️ target-type: ip"]

    subgraph ECS["식당 (ECS Service)"]
      T1["📦 Task A<br/>IP: 10.0.1.21"]
      T2["📦 Task B<br/>IP: 10.0.2.34"]
    end

    TG -->|건강검진 후 안내| T1
    TG -->|건강검진 후 안내| T2
    ECS -.->|IP 자동 등록/해제| TG
```

> ⚠️ **학생들이 제일 많이 틀리는 부분:** Fargate는 task마다 고유 IP를 가지므로 타겟 그룹을 만들 때 **반드시 `target-type = ip`** 로 해야 합니다 (`instance` 아님!).

### 보안 그룹(방화벽) 연결 — 이것도 자주 막혀요 🔒

```mermaid
flowchart LR
    Net([🌐 인터넷]) -->|80/443 허용| SGA["🚪 ALB 보안그룹"]
    SGA -->|8080 허용<br/>소스 = ALB 보안그룹| SGT["📦 Task 보안그룹"]
```

> 💡 Task 보안그룹은 IP 대역(CIDR)이 아니라 **ALB 보안그룹을 소스로** 지정하는 것이 안전합니다.

---

## 4. ⭐ 새로 배포하면 IP가 어떻게 바뀌나요? (가장 중요!)

Fargate task는 배포할 때마다 **새 IP**로 교체됩니다. 하지만 손님이 보는 ALB 주소(DNS)는 그대로라서 **무중단**으로 바뀝니다. 그 비밀이 아래 순서입니다.

### 단계별 그림 (롤링 업데이트)

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 🧑‍💻 개발자
    participant ECS as 🍽️ ECS Service
    participant FG as ☁️ Fargate
    participant TG as 🎯 Target Group
    participant ALB as 🚪 ALB

    Dev->>ECS: 새 버전 배포!
    ECS->>FG: ① 새 Task(B) 먼저 실행
    FG-->>ECS: 새 IP 받음 (10.0.3.50)
    ECS->>TG: ② 새 IP 등록
    TG->>FG: ③ 건강검진 (/health)
    FG-->>TG: 200 OK → healthy ✅
    ALB->>FG: ④ 새 Task로 트래픽 보내기 시작
    ECS->>TG: ⑤ 기존 IP 해제 (draining 시작)
    Note over TG: ⏳ 진행 중 요청은<br/>끝까지 처리 후 종료
    ECS->>FG: ⑥ 기존 Task 종료 → 옛 IP 회수
    Note over Dev,ALB: 🎉 무중단 교체 완료!
```

### 한 줄씩 풀어보면

배포가 시작되면 ECS는 **기존 task를 유지한 채 새 task를 먼저 띄웁니다**(①). 이때 Fargate가 새 IP를 부여하는데, 이전 IP와 다릅니다.

새 task가 준비되면 ECS가 그 **새 IP를 타겟 그룹에 자동 등록**합니다(②). ALB는 바로 트래픽을 보내지 않고 먼저 **건강검진(헬스 체크)**을 합니다(③). 연속으로 성공해 `healthy`가 되어야 트래픽이 흐릅니다(④).

새 task가 정상이면 ECS는 **기존 IP를 해제(draining)**합니다(⑤). 이때 바로 죽이지 않고, **진행 중이던 요청은 끝까지 처리**한 뒤 종료합니다(⑥). 그래서 손님이 끊김을 느끼지 않습니다.

> 🎯 **핵심:** ALB 주소(고정) → 뒤의 IP만 살짝살짝 교체 → 손님은 변화를 모름!

### 무중단 배포 필수 설정 4가지

```mermaid
flowchart TB
    A["✅ minimumHealthyPercent: 100<br/>👉 항상 최소 개수 유지"]
    B["✅ maximumPercent: 200<br/>👉 새 task 먼저 띄울 여유"]
    C["✅ deregistration_delay<br/>👉 요청 처리 시간보다 길게"]
    D["✅ healthCheckGracePeriod<br/>👉 부팅 시간보다 길게"]
```

---

## 정리 카드 📌

```mermaid
mindmap
  root((ECS Fargate))
    Task Definition
      레시피 JSON
      networkMode awsvpc 필수
      CPU/Memory 짝꿍 정해짐
    역할 2가지
      Execution 에이전트용
      Task 내 앱용
    ALB
      target-type ip 필수
      보안그룹 SG 참조
    배포
      새 task 먼저
      헬스체크 통과 후 전환
      기존 task draining
      무중단
```
