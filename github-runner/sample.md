
## 전체 플로우 개요

```
코드 push → GitHub Actions 트리거
  → 1) 이미지 빌드
  → 2) 이미지 태깅 (핵심)
  → 3) ECR 로그인 & push
  → 4) ECS Task Definition 업데이트
  → 5) ECS 서비스 배포
```

## 이미지 태그 전략 (가장 중요한 부분)

배포에서 가장 중요한 원칙은 **`latest` 태그만 쓰지 말 것**입니다. `latest`는 어떤 커밋이 배포됐는지 추적이 안 되고, 롤백도 어렵고, ECS가 이미지 변경을 감지하지 못하는 문제가 생깁니다.

실무에서 권장하는 방식은 **불변(immutable) 태그**를 쓰는 것입니다. 대표적으로 사용하는 태그 값들:

- **Git commit SHA** (`github.sha`): 가장 추적성이 좋음. 어떤 코드가 배포됐는지 1:1로 매칭됩니다. 보통 짧게 자른 short SHA(`a1b2c3d`)를 씁니다.
- **Git tag / 버전** (`v1.2.3`): 릴리스 관리에 유용
- **빌드 번호** (`github.run_number`): 순차적 식별
- **`latest`는 보조 태그로 병행**: 사람이 보기 편하라고 추가로 붙이되, 배포는 SHA 태그로 합니다.

가장 균형 잡힌 실무 표준은 **commit SHA를 메인 태그로 쓰고, `latest`를 함께 붙이는 것**입니다.

## 표준 YAML 파일

아래는 ECR + ECS(Fargate 또는 EC2)에 배포하는 표준 워크플로우입니다.

```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches: [ "main" ]   # main 브랜치 push 시 배포

env:
  AWS_REGION: ap-northeast-2          # 서울 리전
  ECR_REPOSITORY: my-app              # ECR 레포 이름
  ECS_CLUSTER: my-cluster             # ECS 클러스터 이름
  ECS_SERVICE: my-service             # ECS 서비스 이름
  ECS_TASK_DEFINITION: .aws/task-definition.json  # task def 템플릿 경로
  CONTAINER_NAME: my-app-container    # task def 안의 컨테이너 이름

permissions:
  id-token: write   # OIDC 인증용
  contents: read

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      # 1. 소스 체크아웃
      - name: Checkout
        uses: actions/checkout@v4

      # 2. AWS 인증 (OIDC 방식 권장 - 액세스 키 불필요)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      # 3. ECR 로그인
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # 4. 이미지 빌드 + 태깅 + push  ★ 핵심 ★
      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}   # commit SHA를 태그로 사용
        run: |
          # SHA 태그와 latest 태그를 동시에 부여
          docker build \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .

          # 두 태그 모두 push
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

          # 다음 스텝에서 쓸 수 있게 전체 이미지 URI를 output으로 전달
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      # 5. Task Definition에 새 이미지 주입
      - name: Fill in the new image ID in the task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      # 6. ECS 서비스에 배포
      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true   # 배포 안정화까지 대기
```

## 각 단계 상세 설명

**1단계 – 체크아웃**: 워크플로우 실행 러너에 소스 코드를 가져옵니다. task-definition.json도 리포에 있어야 하므로 필요합니다.

**2단계 – AWS 인증**: 예전에는 `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`를 GitHub Secrets에 넣었지만, 지금 표준은 **OIDC(role-to-assume)** 방식입니다. 장기 자격증명을 저장하지 않아도 되고 보안상 훨씬 안전합니다. IAM에 GitHub를 신뢰하는 OIDC provider와 role을 미리 만들어 둬야 합니다.

**3단계 – ECR 로그인**: `amazon-ecr-login` 액션이 docker가 ECR에 push할 수 있도록 인증 토큰을 발급하고, registry 주소를 output(`steps.login-ecr.outputs.registry`)으로 줍니다.

**4단계 – 빌드 & 태깅 (핵심)**: 여기가 이미지 태그 처리의 중심입니다.
- `IMAGE_TAG: ${{ github.sha }}`로 커밋 해시를 태그로 잡습니다.
- `docker build -t ...:$IMAGE_TAG -t ...:latest`로 동일 이미지에 두 개 태그를 동시에 붙입니다.
- 둘 다 push해서 ECR에 두 태그가 같은 이미지를 가리키게 합니다.
- 최종 이미지 URI(`...:$IMAGE_TAG`)를 `$GITHUB_OUTPUT`에 기록해서 다음 스텝(task def 렌더링)이 정확한 버전을 참조하도록 합니다. 배포에는 반드시 SHA 태그 URI를 쓰는 게 포인트입니다 (latest를 쓰면 ECS가 변경 감지를 못 함).

**5단계 – Task Definition 렌더링**: 리포에 있는 task-definition.json 템플릿에서 지정한 컨테이너의 `image` 필드를 방금 빌드한 새 이미지 URI로 치환한 새 파일을 만듭니다.

**6단계 – ECS 배포**: 치환된 task definition을 ECS에 새 리비전으로 등록하고, 지정한 서비스를 그 리비전으로 업데이트합니다. `wait-for-service-stability: true`를 주면 새 태스크가 정상적으로 안정화될 때까지 워크플로우가 기다리므로, 배포 실패 여부를 GitHub Actions에서 바로 알 수 있습니다.

## 추가로 챙기면 좋은 점들

**Short SHA 쓰기**: 전체 SHA가 길다면 short SHA를 별도로 만들어 쓸 수 있습니다.

```yaml
- name: Set short SHA
  id: vars
  run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
# 이후 ${{ steps.vars.outputs.sha_short }} 사용
```

**ECR 이미지 불변성(Tag Immutability) 활성화**: ECR 레포 설정에서 "Tag immutability"를 켜면 같은 태그를 덮어쓰지 못하게 강제됩니다. SHA 태그 전략과 잘 맞습니다. 단, 이 경우 `latest`를 매번 덮어쓰는 게 불가능해지므로 latest 병행 전략과는 충돌합니다 — 두 전략 중 하나를 선택해야 합니다.

**환경별 분리 (dev/staging/prod)**: 보통 브랜치나 환경별로 워크플로우/잡을 분리하고, `environment`와 GitHub Environments 기능으로 승인(approval) 게이트를 둡니다.

**롤백**: SHA 태그로 배포해 두면 롤백이 쉽습니다. 이전 SHA 이미지로 task definition 리비전을 다시 등록하거나, ECS 콘솔에서 이전 리비전으로 서비스를 업데이트하면 됩니다.




````markdown
# GitHub Actions로 ECS 배포하기 (방법 A: github.run_number 순차 태그)

## 1. 개요

GitHub Actions를 사용해 Docker 이미지를 ECR에 push하고 ECS 클러스터에 배포하는 표준 CI/CD 파이프라인입니다.
이미지 태그는 `github.run_number`를 사용하여 **1, 2, 3, 4, 5 ...** 형태로 빌드할 때마다 1씩 자동 증가합니다.

---

## 2. 전체 플로우

```
코드 push (main 브랜치)
      │
      ▼
GitHub Actions 트리거
      │
      ├─ 1) 소스 체크아웃
      ├─ 2) AWS 인증 (OIDC)
      ├─ 3) ECR 로그인
      ├─ 4) 이미지 빌드 + 태깅 (태그 = run_number)  ★ 핵심 ★
      ├─ 5) ECR에 push
      ├─ 6) Task Definition 이미지 주입
      └─ 7) ECS 서비스 배포
```

---

## 3. 이미지 태그 전략 (github.run_number)

### `github.run_number`란?

- 워크플로우가 실행될 때마다 **자동으로 1씩 증가**하는 정수 값입니다.
- 첫 실행은 `1`, 두 번째 실행은 `2`, 세 번째는 `3` ... 이런 식으로 올라갑니다.
- 워크플로우 단위로 카운트되며, 한번 올라간 숫자는 되돌아가지 않습니다.

### 장단점

| 구분 | 내용 |
|------|------|
| 장점 | 태그가 단순하고 사람이 읽기 쉬움(1, 2, 3...). 순차적이라 "몇 번째 배포인지" 직관적 |
| 단점 | 어떤 commit이 배포됐는지 태그만으로는 알 수 없음 (추적성은 SHA보다 떨어짐) |
| 보완 | `latest` 태그를 함께 붙이고, run_number와 commit SHA를 매핑한 로그를 남기면 좋음 |

> **참고**: 추적성을 더 원한다면 `v${{ github.run_number }}-${{ github.sha }}` 처럼 조합하는 방법도 있습니다.
> 본 문서는 요청대로 순수하게 `run_number`(1, 2, 3...)만 사용합니다.

---

## 4. 표준 워크플로우 YAML

`.github/workflows/deploy.yml` 파일로 저장합니다.

```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches: [ "main" ]   # main 브랜치 push 시 배포

env:
  AWS_REGION: ap-northeast-2                       # 서울 리전
  ECR_REPOSITORY: my-app                           # ECR 레포 이름
  ECS_CLUSTER: my-cluster                          # ECS 클러스터 이름
  ECS_SERVICE: my-service                          # ECS 서비스 이름
  ECS_TASK_DEFINITION: .aws/task-definition.json   # task def 템플릿 경로
  CONTAINER_NAME: my-app-container                 # task def 안의 컨테이너 이름

permissions:
  id-token: write   # OIDC 인증용
  contents: read

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      # 1. 소스 체크아웃
      - name: Checkout
        uses: actions/checkout@v4

      # 2. AWS 인증 (OIDC 방식 - 액세스 키 불필요)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      # 3. ECR 로그인
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # 4. 이미지 빌드 + 태깅 + push  ★ 핵심 ★
      #    태그 = github.run_number (1, 2, 3, 4, 5 ...)
      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.run_number }}   # ← 순차 증가 태그
        run: |
          # run_number 태그와 latest 태그를 동시에 부여
          docker build \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .

          # 두 태그 모두 push
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

          # 다음 스텝에서 쓸 수 있게 전체 이미지 URI를 output으로 전달
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      # 5. Task Definition에 새 이미지 주입
      - name: Fill in the new image ID in the task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      # 6. ECS 서비스에 배포
      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true   # 배포 안정화까지 대기
```

---

## 5. 각 단계 상세 설명

### 1단계 – 체크아웃
워크플로우 러너에 소스 코드를 가져옵니다. `task-definition.json`도 리포에 있어야 하므로 필요합니다.

### 2단계 – AWS 인증
**OIDC(role-to-assume)** 방식을 사용합니다. 장기 자격증명(Access Key)을 GitHub Secrets에 저장하지 않아도 되어 보안상 안전합니다.
IAM에 GitHub를 신뢰하는 OIDC provider와 role을 미리 생성해 둬야 합니다.

### 3단계 – ECR 로그인
`amazon-ecr-login` 액션이 docker가 ECR에 push할 수 있도록 인증하고, registry 주소를 output으로 제공합니다.

### 4단계 – 빌드 & 태깅 (핵심)
- `IMAGE_TAG: ${{ github.run_number }}` 로 **순차 증가 번호**를 태그로 사용합니다.
- 첫 배포 → `my-app:1`, 다음 배포 → `my-app:2`, 그 다음 → `my-app:3` ...
- 동일 이미지에 `run_number` 태그와 `latest` 태그를 동시에 붙여 둘 다 push합니다.
- 배포에는 **반드시 `run_number` 태그 URI**를 사용합니다 (latest를 쓰면 ECS가 변경 감지를 못 함).

### 5단계 – Task Definition 렌더링
리포의 `task-definition.json` 템플릿에서 지정한 컨테이너의 `image` 필드를 새 이미지 URI로 치환한 새 파일을 생성합니다.

### 6단계 – ECS 배포
치환된 task definition을 새 리비전으로 등록하고, 서비스를 해당 리비전으로 업데이트합니다.
`wait-for-service-stability: true` 옵션으로 새 태스크가 안정화될 때까지 대기하여 배포 성공/실패를 즉시 확인합니다.

---

## 6. Task Definition 템플릿 예시

`.aws/task-definition.json` (image 필드는 액션이 자동 치환하므로 비워두거나 placeholder만 넣어둡니다)

```json
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-app-container",
      "image": "PLACEHOLDER",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

## 7. 사전 준비 체크리스트

- [ ] ECR 레포지토리 생성 (`my-app`)
- [ ] ECS 클러스터 / 서비스 생성 (`my-cluster`, `my-service`)
- [ ] IAM OIDC provider 등록 (`token.actions.githubusercontent.com`)
- [ ] GitHub Actions용 IAM Role 생성 (ECR push, ECS 배포 권한 포함)
- [ ] `.aws/task-definition.json` 작성
- [ ] `ecsTaskExecutionRole` 생성

---

## 8. 배포 후 ECR 이미지 태그 예시

| 배포 순서 | run_number | 생성되는 태그 |
|-----------|-----------|--------------|
| 1번째 배포 | 1 | `my-app:1`, `my-app:latest` |
| 2번째 배포 | 2 | `my-app:2`, `my-app:latest` |
| 3번째 배포 | 3 | `my-app:3`, `my-app:latest` |
| 4번째 배포 | 4 | `my-app:4`, `my-app:latest` |
| 5번째 배포 | 5 | `my-app:5`, `my-app:latest` |

---

## 9. 참고 사항

- **롤백**: 이전 번호 이미지(예: `my-app:3`)로 task definition 리비전을 다시 등록하면 롤백됩니다.
- **ECR Tag Immutability**: run_number는 매번 새 숫자라 immutability를 켜도 충돌하지 않습니다. (단, `latest` 병행 시 latest는 덮어쓰므로 immutability와 충돌 — 둘 중 선택)
- **주의**: 워크플로우 파일 이름을 바꾸거나 새로 만들면 run_number가 1부터 다시 시작될 수 있습니다.
- 액션 버전(`@v4`, `@v2` 등)은 시간이 지나면 변경될 수 있으니 적용 전 최신 버전 확인을 권장합니다.
````
