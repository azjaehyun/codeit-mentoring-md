# HR Bank — Critical 이슈 정리

## 1. 개요

본 문서는 [CODE_REVIEW.md](./CODE_REVIEW.md)에서 **Critical(최우선)** 로 분류된 항목만 추려 정리했습니다. Critical 이슈는 "나중에 고치면 되는 개선"이 아니라 **지금 당장 고쳐야 하는 버그**입니다. 잘못된 입력이 DB까지 들어가거나, 파일이 디스크에 영구 잔류하거나, API가 의도와 다른 HTTP 상태로 응답하는 등 **데이터·디스크 무결성과 API 신뢰도**에 직접 영향을 줍니다.

기능 추가나 UI 작업보다 **아래 4건을 먼저 수정**하는 것을 권장합니다.

| 구분 | Critical 건수 | 별도 섹션 |
|------|:-------------:|-----------|
| 버그·데이터 무결성 | 4건 | N+1 학습 Q&A 1절 |

---

## 2. Critical 이슈 목록

### C-1. Bean Validation 미적용 — **우선순위: Critical (1순위)**

**관련 파일**

| 파일 | 라인 |
|------|------|
| `build.gradle` | 21–50 |
| `src/main/java/com/codeit/hrbank/controller/DepartmentController.java` | 29–34 |
| `src/main/java/com/codeit/hrbank/dto/department/DepartmentCreateRequest.java` | 19–26 |
| `src/main/java/com/codeit/hrbank/controller/EmployeeController.java` | 28–40 |
| `src/main/java/com/codeit/hrbank/common/exception/GlobalExceptionHandler.java` | 10–34 |

#### 현상 (As-Is)

`build.gradle`에 `spring-boot-starter-validation` 의존성이 없습니다. Employee DTO에는 `@NotBlank`, `@Email` 등이 있고 Controller에 `@Valid`가 있지만, **스타터 없이는 검증이 동작하지 않을 수 있습니다**. Department는 Controller에 `@Valid` 자체가 없어 `DepartmentCreateRequest`의 검증 어노테이션이 무력화됩니다. 또한 `GlobalExceptionHandler`에 `MethodArgumentNotValidException` 핸들러가 없어, 검증이 동작하더라도 일관된 400 응답을 보장하기 어렵습니다.

**build.gradle (21–50행)**

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testCompileOnly 'org.projectlombok:lombok'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    testAnnotationProcessor 'org.projectlombok:lombok'

    // https://mvnrepository.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.9")

    // mapstruct
    implementation 'org.mapstruct:mapstruct:1.6.3'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.3'

    // QueryDSL JPA
    implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'

    // QueryDSL Annotation Processor
    annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

    // UUID v7
    implementation 'com.github.f4b6a3:uuid-creator:6.1.1'
}
```

**DepartmentController.java (29–34행)**

```java
  @PostMapping
  public ResponseEntity<DepartmentResponse> createDepartment(
      @RequestBody DepartmentCreateRequest request) {
    DepartmentResponse response = departmentService.createDepartment(request);
    return ResponseEntity.ok(response);
  }
```

**DepartmentCreateRequest.java (19–26행)**

```java
  @NotBlank(message = "부서 이름은 필수입니다.")
  @Size(max = 50)
  private String name;

  private String description;

  @NotNull(message = "설립일은 필수입니다.")
  private LocalDate establishedDate;
```

**GlobalExceptionHandler.java (전체)**

```java
package com.codeit.hrbank.common.exception;

import com.codeit.hrbank.dto.error.ErrorResponse;
import com.codeit.hrbank.dto.error.HrBankException;
import java.util.Map;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(HrBankException.class)
  public ResponseEntity<ErrorResponse> handleHrBankException(HrBankException ex) {
    ErrorCode code = ex.getErrorCode();
    return ResponseEntity.status(code.getStatus()).body(ErrorResponse.of(code, ex.getMessage()));
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<?> handleIllegalArgument(IllegalArgumentException e) {
    return ResponseEntity.badRequest().body(Map.of(
        "message", "IllegalArgumentException", 
        "detail", e.getMessage()
    ));
  }

  @ExceptionHandler(IllegalStateException.class)
  public ResponseEntity<?> handleIllegalState(IllegalStateException e) {
    return ResponseEntity.badRequest().body(Map.of(
        "message", "IllegalStateException",
        "details", e.getMessage() == null ? "" : e.getMessage()
    ));
  }
}
```

#### 수정 (To-Be)

**build.gradle — validation 스타터 추가**

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testCompileOnly 'org.projectlombok:lombok'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    testAnnotationProcessor 'org.projectlombok:lombok'

    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.9")

    implementation 'org.mapstruct:mapstruct:1.6.3'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.3'

    implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'

    annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

    implementation 'com.github.f4b6a3:uuid-creator:6.1.1'
}
```

**DepartmentController.java — `@Valid` 추가**

```java
  @PostMapping
  public ResponseEntity<DepartmentResponse> createDepartment(
      @Valid @RequestBody DepartmentCreateRequest request) {
    DepartmentResponse response = departmentService.createDepartment(request);
    return ResponseEntity.ok(response);
  }
```

**GlobalExceptionHandler.java — 검증 실패 핸들러 추가**

```java
package com.codeit.hrbank.common.exception;

import com.codeit.hrbank.dto.error.ErrorResponse;
import com.codeit.hrbank.dto.error.HrBankException;
import java.util.Map;
import java.util.stream.Collectors;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(HrBankException.class)
  public ResponseEntity<ErrorResponse> handleHrBankException(HrBankException ex) {
    ErrorCode code = ex.getErrorCode();
    return ResponseEntity.status(code.getStatus()).body(ErrorResponse.of(code, ex.getMessage()));
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
    String details = ex.getBindingResult().getFieldErrors().stream()
        .map(error -> error.getField() + ": " + error.getDefaultMessage())
        .collect(Collectors.joining(", "));

    return ResponseEntity.badRequest().body(Map.of(
        "message", "Validation failed",
        "detail", details
    ));
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<?> handleIllegalArgument(IllegalArgumentException e) {
    return ResponseEntity.badRequest().body(Map.of(
        "message", "IllegalArgumentException",
        "detail", e.getMessage()
    ));
  }

  @ExceptionHandler(IllegalStateException.class)
  public ResponseEntity<?> handleIllegalState(IllegalStateException e) {
    return ResponseEntity.badRequest().body(Map.of(
        "message", "IllegalStateException",
        "details", e.getMessage() == null ? "" : e.getMessage()
    ));
  }
}
```

#### 왜 고쳐야 하는지

빈 부서 이름이나 null 설립일 같은 잘못된 요청이 그대로 DB에 저장되면, 나중에 500 에러나 DB 제약 조건 위반으로 원인을 찾기 어렵습니다. `@Valid`와 validation 스타터, 그리고 전역 예외 핸들러를 함께 적용해야 Swagger 문서에 적힌 검증 규칙과 실제 API 동작이 일치합니다.

---

### C-2. 프로필 이미지 디스크 삭제 경로 오류 — **우선순위: Critical (2순위)**

**관련 파일**

| 파일 | 라인 |
|------|------|
| `src/main/java/com/codeit/hrbank/common/file/FileStorage.java` | 36–54, 126–139 |
| `src/main/java/com/codeit/hrbank/service/FileMetadataService.java` | 82–87 |
| `src/main/java/com/codeit/hrbank/service/EmployeeService.java` | 101–105, 117–123 |

#### 현상 (As-Is)

프로필 저장은 `uploads/images/`(`saveProfileImage`)인데, 삭제는 `deleteAllAttachments`에서 **`uploadDir`(uploads/)만** 조회합니다. `EmployeeService`의 수정·삭제는 `fileMetadataService.delete()`를 호출하므로 **DB 메타는 삭제되지만 이미지 파일은 디스크에 남을 수 있습니다**.

**FileStorage.java — saveProfileImage (36–54행)**

```java
  public FileMetadata saveProfileImage(MultipartFile file) {
    if (file == null || file.isEmpty()) {
      throw new IllegalArgumentException("빈 파일은 저장할 수 없습니다.");
    }

    Path imageDir = fileConfig.getImageDir();
    String storedName = buildStoredName(file.getOriginalFilename());
    Path dest = imageDir.resolve(storedName);

    transferFile(file, dest);

    return FileMetadata.builder()
        .originalName(file.getOriginalFilename())
        .storedName(storedName)
        .contentType(file.getContentType())
        .size(file.getSize())
        .fileType(FileTypeConst.PROFILE_IMAGE)
        .build();
  }
```

**FileStorage.java — deleteAllAttachments (126–139행)**

```java
  public void deleteAllAttachments(Collection<FileMetadata> files) {
    if (files == null || files.isEmpty()) return;

    Path uploadDir = fileConfig.getUploadDir();

    for (FileMetadata metadata : files) {
      if (metadata.getStoredName() == null) continue;
      try {
        Files.deleteIfExists(uploadDir.resolve(metadata.getStoredName()));
      } catch (IOException e) {
        throw new RuntimeException("첨부 파일 삭제 실패: " + metadata.getStoredName(), e);
      }
    }
  }
```

**FileMetadataService.java — delete (82–87행)**

```java
  @Transactional
  public void delete(Collection<FileMetadata> attachments) {
    if (attachments == null || attachments.isEmpty()) return;
    fileStorage.deleteAllAttachments(attachments);
    fileMetadataRepository.deleteAll(attachments);
  }
```

**EmployeeService.java — 프로필 교체·삭제 시 호출 (101–105, 117–123행)**

```java
        if (profile != null && !profile.isEmpty() && oldProfileImage != null) {
            employeeRepository.flush();
            fileMetadataService.delete(List.of(oldProfileImage));
        }
```

```java
        employee.removeProfileImage();
        employeeRepository.delete(employee);

        if (profileImage != null) {
            employeeRepository.flush();
            fileMetadataService.delete(List.of(profileImage));
        }
```

#### 수정 (To-Be)

**FileStorage.java — fileType별 저장 경로로 삭제**

```java
  public void deleteAllAttachments(Collection<FileMetadata> files) {
    if (files == null || files.isEmpty()) return;

    for (FileMetadata metadata : files) {
      if (metadata.getStoredName() == null) continue;
      Path targetDir = resolveStorageDir(metadata.getFileType());
      try {
        Files.deleteIfExists(targetDir.resolve(metadata.getStoredName()));
      } catch (IOException e) {
        throw new RuntimeException("파일 삭제 실패: " + metadata.getStoredName(), e);
      }
    }
  }

  private Path resolveStorageDir(String fileType) {
    if (FileTypeConst.PROFILE_IMAGE.equals(fileType)) {
      return fileConfig.getImageDir();
    }
    if (FileTypeConst.BACKUP.equals(fileType) || FileTypeConst.BACKUP_LOG.equals(fileType)) {
      return fileConfig.getBackupDir();
    }
    return fileConfig.getUploadDir();
  }
```

#### 왜 고쳐야 하는지

직원을 삭제하거나 프로필 사진을 교체했을 때 DB에서는 파일 정보가 사라지지만, 실제 이미지는 `uploads/images/`에 그대로 남습니다. 시간이 지나면 디스크가 불필요하게 차고, 개인정보(프로필 사진)가 의도치 않게 보관될 수 있습니다.

---

### C-3. `DataBackupService.parseStatus` catch 타입 오류 — **우선순위: Critical (3순위)**

**관련 파일**

| 파일 | 라인 |
|------|------|
| `src/main/java/com/codeit/hrbank/service/dataBackup/DataBackupService.java` | 265–272 |

#### 현상 (As-Is)

`BackupStatus.valueOf()` 실패 시 던지는 예외는 **`IllegalArgumentException`**인데, catch 블록은 **`HrBankException`**만 잡습니다. 잘못된 `status` 쿼리 파라미터가 `INVALID_STATUS`(400)로 정리되지 않고 500 또는 일관성 없는 응답이 됩니다.

**DataBackupService.java — parseStatus (265–272행)**

```java
  private BackupStatus parseStatus(String status) {
    if (status == null || status.isBlank()) return null;
    try {
      return BackupStatus.valueOf(status.toUpperCase());
    } catch (HrBankException e) {
      throw new HrBankException(ErrorCode.INVALID_STATUS);
    }
  }
```

#### 수정 (To-Be)

**DataBackupService.java — 올바른 예외 타입 catch**

```java
  private BackupStatus parseStatus(String status) {
    if (status == null || status.isBlank()) return null;
    try {
      return BackupStatus.valueOf(status.toUpperCase());
    } catch (IllegalArgumentException e) {
      throw new HrBankException(ErrorCode.INVALID_STATUS);
    }
  }
```

(`getLatestBackup` 149–152행의 `BackupStatus.valueOf(status.toUpperCase())`도 동일하게 `parseStatus`를 재사용하거나 같은 catch 패턴을 적용하는 것이 좋습니다.)

#### 왜 고쳐야 하는지

클라이언트가 `status=WRONG` 같은 잘못된 값을 보내면 "잘못된 입력(400)"이 아니라 "서버 오류(500)"로 보일 수 있습니다. API 사용자는 자신의 요청이 틀렸는지, 서버가 고장 났는지 구분할 수 없게 됩니다.

---

### C-4. `saveAttachFile`의 fileType 오타 — **우선순위: Critical (4순위)**

**관련 파일**

| 파일 | 라인 |
|------|------|
| `src/main/java/com/codeit/hrbank/common/file/FileStorage.java` | 63–81 |
| `src/main/java/com/codeit/hrbank/entity/file/FileTypeConst.java` | 8–12 |

#### 현상 (As-Is)

일반 첨부 저장 메서드가 `FileTypeConst.PROFILE_IMAGE`를 설정합니다. 다운로드 API는 type별 경로 분기를 쓰므로, 첨부 파일이 **잘못된 디렉터리**에서 조회되거나 타입 필터·통계가 틀어질 수 있습니다.

**FileStorage.java — saveAttachFile (63–81행)**

```java
  public FileMetadata saveAttachFile(MultipartFile file) {
    if (file == null || file.isEmpty()) {
      throw new IllegalArgumentException("빈 파일은 저장할 수 없습니다.");
    }

    Path uploadDir = fileConfig.getUploadDir();
    String storedName = buildStoredName(file.getOriginalFilename());
    Path dest = uploadDir.resolve(storedName);

    transferFile(file, dest);

    return FileMetadata.builder()
        .originalName(file.getOriginalFilename())
        .storedName(storedName)
        .contentType(file.getContentType())
        .size(file.getSize())
        .fileType(FileTypeConst.PROFILE_IMAGE)
        .build();
  }
```

**FileTypeConst.java (현재 — ATTACHMENT 상수 없음)**

```java
public final class FileTypeConst {

  private FileTypeConst() {
  }

  public static final String PROFILE_IMAGE = "PROFILE_IMAGE";

  public static final String BACKUP = "BACKUP";

  public static final String BACKUP_LOG = "BACKUP_LOG";
}
```

#### 수정 (To-Be)

**FileTypeConst.java — 첨부 전용 상수 추가**

```java
public final class FileTypeConst {

  private FileTypeConst() {
  }

  public static final String PROFILE_IMAGE = "PROFILE_IMAGE";

  public static final String ATTACHMENT = "ATTACHMENT";

  public static final String BACKUP = "BACKUP";

  public static final String BACKUP_LOG = "BACKUP_LOG";
}
```

**FileStorage.java — saveAttachFile**

```java
  public FileMetadata saveAttachFile(MultipartFile file) {
    if (file == null || file.isEmpty()) {
      throw new IllegalArgumentException("빈 파일은 저장할 수 없습니다.");
    }

    Path uploadDir = fileConfig.getUploadDir();
    String storedName = buildStoredName(file.getOriginalFilename());
    Path dest = uploadDir.resolve(storedName);

    transferFile(file, dest);

    return FileMetadata.builder()
        .originalName(file.getOriginalFilename())
        .storedName(storedName)
        .contentType(file.getContentType())
        .size(file.getSize())
        .fileType(FileTypeConst.ATTACHMENT)
        .build();
  }
```

(C-2의 `resolveStorageDir`에서도 `ATTACHMENT`는 `uploadDir`로 분기해야 저장·삭제·다운로드 경로가 일치합니다.)

#### 왜 고쳐야 하는지

첨부 파일은 `uploads/`에 저장되는데 DB에는 `PROFILE_IMAGE`로 기록되면, 다운로드 시 `uploads/images/`에서 파일을 찾으려 해 **404 또는 파일 없음** 오류가 납니다. 타입별 통계·필터링도 전부 틀어집니다.

---

## 3. N+1 문제 설명 (학생 Q&A)

> **질문:** "트러블 슈팅 N+1 제가 이해를 잘 못했는데"

### 한 줄 요약

**N+1**은 API를 **한 번** 호출했는데, DB에 SQL이 **1 + N번** 나가는 패턴입니다.  
첫 1번은 목록(부서)을 가져오고, 나머지 N번은 **목록 각 행마다** 추가 조회(COUNT)를 반복할 때 생깁니다.  
이 프로젝트에서는 `GET /api/departments` → `DepartmentService.getDepartments()`가 대표적인 예입니다.

---

### 비유: 카페 주문

손님 3명이 각자 음료를 주문했습니다.

- **좋은 방식:** 주문 3개를 한 번에 받고, 한 번에 3잔을 만든다.
- **N+1 방식:** 손님 명단(1번)을 받은 뒤, **한 명씩** "이 분 음료 뭐였죠?"를 다시 묻는다 → 1 + 3 = **4번** 대화.

DB도 똑같습니다. 부서 목록 1번 조회 후, 부서마다 "직원 몇 명?"을 **따로따로** 물으면 쿼리가 늘어납니다.

---

### 우리 프로젝트에서 어디?

| 파일 | 역할 |
|------|------|
| `DepartmentService.java` 86–137행 | 부서 목록 + **N+1 루프** |
| `DepartmentRepository.java` 18–24행 | 부서별 `COUNT` JPQL |

문제의 핵심은 아래 **127행 한 줄**입니다. `map` 안에서 부서마다 COUNT를 호출합니다.

```java
// DepartmentService.java — getDepartments()
departments = departmentRepository.findAll();          // ① 부서 목록 1번 조회

List<Department> paged = filtered.stream()
    .limit(request.getSize())                          // 예: size=3 → 3건만
    .collect(Collectors.toList());

return DepartmentSliceResponse.builder()
    .content(paged.stream()
        .map(d -> toResponse(d,
            (int) departmentRepository.countEmployeesByDepartmentId(d.getId())))  // ② 부서마다 COUNT
        .collect(Collectors.toList()))
    .build();
```

---

### 샘플 데이터 (미니)

아래는 이해용 **가상 데이터**입니다.

| 부서 | 직원 수 |
|------|:------:|
| 개발팀 | 12명 |
| 경영지원팀 | 4명 |
| 인사팀 | 3명 |

요청 예: `GET /api/departments?size=3` → 위 3개 부서가 한 페이지에 나옵니다.

---

### 무슨 일이 벌어지나?

`size=3`일 때 DB에는 이렇게 **4번** SQL이 나갑니다.

| 순서 | 무엇을 하나 | 쿼리 |
|:----:|-------------|------|
| ① | 부서 전체 목록 조회 | `SELECT ... FROM department` → **1번** |
| ② | 개발팀 직원 수 | `SELECT COUNT(*) ... WHERE department_id = ?` |
| ③ | 경영지원팀 직원 수 | `SELECT COUNT(*) ... WHERE department_id = ?` |
| ④ | 인사팀 직원 수 | `SELECT COUNT(*) ... WHERE department_id = ?` |

**합계: 1 + 3 = 4번**

공식은 간단합니다.

```
총 쿼리 수 = 목록 1번 + COUNT × (현재 페이지 행 수)
```

부서가 100개여도 화면에 3개만 보이면 COUNT는 3번이지만, **목록 조회는 여전히 100개 전부** 읽을 수 있다는 점도 같이 기억해 두세요 (`findAll`).

---

### 왜 문제?

- **느려집니다** — 화면에 3건만 보여도 DB 왕복이 4번입니다. 부서·페이지가 늘수록 더 느려집니다.
- **DB 부하가 커집니다** — 같은 형태의 `SELECT COUNT`가 연속으로 찍히면 N+1을 의심할 수 있습니다.
- **기능은 맞지만 성능이 틀립니다** — JSON 결과는 정확하지만, 트래픽이 늘면 실서비스에서 병목이 됩니다. (CODE_REVIEW.md H-6)

---

### 좋은 예 vs 나쁜 예

**좋은 예 — Employee 목록 (JOIN FETCH, 1번 쿼리)**

```java
// EmployeeRepositoryCustomImpl.java — 직원 + 부서를 한 번에
"SELECT e FROM Employee e JOIN FETCH e.department d WHERE 1=1"
```

직원 10명의 부서 **이름**이 필요할 때, JOIN FETCH로 **SELECT 1번**에 끝냅니다.

**나쁜 예 — Department 목록 (루프 COUNT, 1+N번 쿼리)**

```java
// DepartmentService.java 127행 — 부서마다 COUNT 반복
.map(d -> toResponse(d,
    (int) departmentRepository.countEmployeesByDepartmentId(d.getId())))
```

부서 **인원 수**는 JOIN FETCH로 해결할 수 없습니다. COUNT/GROUP BY 같은 **집계**가 필요합니다.

---

### 고치려면?

**방향:** 페이지에 보여줄 부서 id들을 모아서, COUNT를 **한 번에** 가져옵니다 (GROUP BY + `IN`).

```java
// To-Be 아이디어 (의사 코드)
List<UUID> pageIds = paged.stream().map(Department::getId).toList();

// COUNT 1번: WHERE department_id IN (...) GROUP BY department_id
Map<UUID, Long> counts = departmentRepository.countByDepartmentIds(pageIds);

// map 안에서는 DB 호출 없이 Map에서 꺼내 쓰기
.map(d -> toResponse(d, counts.getOrDefault(d.getId(), 0L).intValue()))
```

`size=3`이면 **1 + 1 = 2번** 쿼리로 줄일 수 있습니다. (목록 1번 + 배치 COUNT 1번)

---

### FAQ

**Q. N+1이면 버그인가요?**  
A. **기능 버그는 아닙니다.** 응답 JSON은 맞게 나옵니다. 다만 성능 문제라 고치는 게 좋습니다.

**Q. JOIN FETCH 쓰면 Department도 해결되나요?**  
A. **아닙니다.** JOIN FETCH는 연관 **엔티티**를 한 번에 가져올 때 씁니다. 필요한 건 직원 **수(COUNT)** 이므로 GROUP BY나 배치 COUNT가 맞습니다.

**Q. 로그에서 어떻게 알아보나요?**  
A. Hibernate SQL 로그에 **같은 형태의 `SELECT COUNT`가 연속**으로 찍히면 N+1을 의심하세요. `GET /api/departments` 한 번에 department SELECT 1번 + COUNT 3번 패턴이 보이면 이 코드입니다.

---

## 4. N+1 수정 가이드 (적용 완료)

> 프로젝트에 **이미 반영된** 수정입니다. 동일한 문제가 있으면 아래 두 파일만 보면 됩니다.  
> `DepartmentService`는 **삭제하지 않고**, Repository에 JPQL을 추가한 뒤 Service에서 **한 번만** 호출합니다.

### 수정 대상 파일

| 파일 | 하는 일 |
|------|---------|
| [`DepartmentRepository.java`](src/main/java/com/codeit/hrbank/repository/DepartmentRepository.java) | 여러 부서 id에 대한 COUNT를 **한 번** 조회하는 JPQL 추가 |
| [`DepartmentService.java`](src/main/java/com/codeit/hrbank/service/DepartmentService.java) | `getDepartments()`에서 루프 COUNT 제거, Map으로 조회 |

단건 API(`getDepartment`, `update`, `delete`)는 부서 1건이라 기존 `countEmployeesByDepartmentId` **그대로** 사용합니다.

---

### 4-1. `DepartmentRepository.java`

**위치:** `countEmployeesByDepartmentId` 메서드 **바로 아래**에 추가 (현재 26~34행)

#### As-Is (수정 전)

부서 id **하나**만 COUNT하는 메서드만 있었고, Service가 목록마다 이걸 반복 호출했습니다.

```java
@Query("""
       SELECT COUNT(e)
       FROM Employee e
       JOIN e.department d
       WHERE d.id = :departmentId
       """)
long countEmployeesByDepartmentId(@Param("departmentId") UUID departmentId);
// ← 목록 API에서는 이 메서드를 N번 호출 → N+1
```

#### To-Be (수정 후)

페이지에 나온 부서 id 목록을 `IN`으로 묶어 **GROUP BY** 한 번에 조회합니다.

```java
@Query("""
       SELECT d.id, COUNT(e)
       FROM Employee e
       JOIN e.department d
       WHERE d.id IN :departmentIds
       GROUP BY d.id
       """)
List<Object[]> countEmployeesGroupedByDepartmentIds(
    @Param("departmentIds") List<UUID> departmentIds);
```

- 반환: `[부서 UUID, 직원 수]` 쌍의 리스트
- 직원이 **0명**인 부서는 `GROUP BY` 결과에 없음 → Service에서 `0`으로 처리

---

### 4-2. `DepartmentService.java`

**위치:** `getDepartments()` 메서드 + private 헬퍼 추가

#### As-Is (수정 전) — 125~128행 부근

```java
return DepartmentSliceResponse.builder()
    .content(paged.stream()
        .map(d -> toResponse(d,
            (int) departmentRepository.countEmployeesByDepartmentId(d.getId())))
        .collect(Collectors.toList()))
```

`map` 안에서 DB 호출 → 페이지 행 수만큼 COUNT 쿼리 (**N+1**).

#### To-Be (수정 후)

**① import 추가**

```java
import java.util.Map;
```

**② `getDepartments()` — COUNT를 루프 밖에서 1번**

```java
Map<UUID, Long> employeeCountsByDepartmentId = fetchEmployeeCountsByDepartmentIds(paged);

return DepartmentSliceResponse.builder()
    .content(paged.stream()
        .map(d -> toResponse(d,
            employeeCountsByDepartmentId.getOrDefault(d.getId(), 0L).intValue()))
        .collect(Collectors.toList()))
```

**③ private 메서드 추가 (`toResponse` 위)**

```java
private Map<UUID, Long> fetchEmployeeCountsByDepartmentIds(List<Department> departments) {
  if (departments.isEmpty()) {
    return Map.of();
  }

  List<UUID> departmentIds = departments.stream().map(Department::getId).toList();
  return departmentRepository.countEmployeesGroupedByDepartmentIds(departmentIds).stream()
      .collect(Collectors.toMap(
          row -> (UUID) row[0],
          row -> (Long) row[1]
      ));
}
```

---

### 수정 후 쿼리 수 (`GET /api/departments?size=3`)

| 순서 | 내용 | 횟수 |
|:----:|------|:----:|
| ① | 부서 목록 (`findAll` / `searchDepartments`) | 1 |
| ② | `countEmployeesGroupedByDepartmentIds` (`IN` + `GROUP BY`) | 1 |
| **합계** | | **2** (이전: 1 + 3 = **4**) |

---

### 확인 방법

1. `spring.jpa.show-sql=true` (또는 Hibernate SQL 로그) 켜기
2. `GET /api/departments?size=3` 한 번 호출
3. **같은 형태의 `SELECT COUNT`가 1번만** 나오는지 확인 (이전에는 3번 연속)

---

### 아직 남은 이슈 (선택)

N+1(COUNT 반복)은 해결했지만, `findAll()`로 **전체 부서**를 메모리에 올린 뒤 정렬·페이징하는 구조는 그대로입니다.  
데이터가 많아지면 [CODE_REVIEW.md](./CODE_REVIEW.md) H-6처럼 DB keyset 페이징(QueryDSL 등)으로 옮기는 것을 권장합니다.

---

*본 문서는 [CODE_REVIEW.md](./CODE_REVIEW.md)의 Critical 4건과 N+1 학습 Q&A를 발췌했습니다. **섹션 4(N+1)** 는 프로젝트 소스에 적용된 수정 가이드입니다. Critical C-1~C-4는 아직 코드 미반영일 수 있습니다.*
