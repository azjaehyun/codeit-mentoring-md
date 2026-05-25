

> 자바 기준, 출제 빈도와 가성비 기준으로 선별한 필수 3유형

---

## 📌 1순위: 해시맵 (HashMap)

### 문제: 완주하지 못한 선수

마라톤에 참가한 선수 명단 `participant`와 완주한 선수 명단 `completion`이 주어질 때, 완주하지 못한 단 한 명의 선수 이름을 반환하세요.

**조건**
- 참가자는 1명 이상 100,000명 이하
- `completion`의 길이는 `participant`의 길이보다 1 작음
- 동명이인이 있을 수 있음

**입출력 예시**

| participant | completion | return |
|---|---|---|
| ["leo", "kiki", "eden"] | ["eden", "kiki"] | "leo" |
| ["mislav", "stanko", "mislav", "ana"] | ["stanko", "ana", "mislav"] | "mislav" |

### 정답 코드

```java
import java.util.HashMap;

public class Solution {
    public String solution(String[] participant, String[] completion) {
        HashMap<String, Integer> map = new HashMap<>();

        // 참가자 카운팅
        for (String p : participant) {
            map.put(p, map.getOrDefault(p, 0) + 1);
        }

        // 완주자만큼 차감
        for (String c : completion) {
            map.put(c, map.get(c) - 1);
        }

        // 카운트가 남아있는 사람이 미완주자
        for (String key : map.keySet()) {
            if (map.get(key) > 0) return key;
        }
        return "";
    }
}
```

### 핵심 포인트
- 동명이인이 있으므로 `Set`이 아닌 `Map`으로 **카운팅**해야 함
- `getOrDefault(key, 0)` 패턴은 카운팅 문제의 정석
- 시간복잡도 O(N)

---

## 📌 2순위: DFS/BFS (깊이/너비 우선 탐색)

### 문제: 타겟 넘버

음이 아닌 정수 배열 `numbers`가 있습니다. 각 숫자 앞에 `+` 또는 `-`를 붙여 모두 더한 결과가 `target`이 되는 경우의 수를 반환하세요.

**조건**
- `numbers`의 길이는 2 이상 20 이하
- 각 원소는 1 이상 50 이하
- `target`은 1 이상 1000 이하

**입출력 예시**

| numbers | target | return |
|---|---|---|
| [1, 1, 1, 1, 1] | 3 | 5 |
| [4, 1, 2, 1] | 4 | 2 |

`[1,1,1,1,1]`로 3 만들기: `+1+1+1+1-1`, `+1+1+1-1+1`, `+1+1-1+1+1`, `+1-1+1+1+1`, `-1+1+1+1+1` → 5가지

### 정답 코드

```java
public class Solution {
    int answer = 0;

    public int solution(int[] numbers, int target) {
        dfs(numbers, target, 0, 0);
        return answer;
    }

    private void dfs(int[] numbers, int target, int idx, int sum) {
        // 종료 조건: 모든 숫자를 다 사용한 경우
        if (idx == numbers.length) {
            if (sum == target) answer++;
            return;
        }

        // 현재 숫자를 더하는 경우
        dfs(numbers, target, idx + 1, sum + numbers[idx]);
        // 현재 숫자를 빼는 경우
        dfs(numbers, target, idx + 1, sum - numbers[idx]);
    }
}
```

### 핵심 포인트
- 모든 경우의 수를 탐색하는 **완전탐색 DFS**
- 종료 조건(`idx == numbers.length`) + 선택지(더하기/빼기) 분기 구조 암기
- 이 템플릿으로 미로찾기, 섬 개수, 조합 문제 등 응용 가능

### 💡 DFS 기본 템플릿

```java
private void dfs(파라미터, 현재상태) {
    // 1. 종료 조건
    if (종료조건) {
        결과 처리;
        return;
    }

    // 2. 가능한 모든 선택지 탐색
    for (선택지 : 후보들) {
        if (유효한 선택) {
            dfs(파라미터, 다음상태);
        }
    }
}
```

---

## 📌 3순위: 정렬 (Custom Comparator)

### 문제: 가장 큰 수

0 또는 양의 정수 배열 `numbers`가 주어질 때, 이어 붙여 만들 수 있는 가장 큰 수를 문자열로 반환하세요.

**조건**
- `numbers`의 길이는 1 이상 100,000 이하
- 각 원소는 0 이상 1,000 이하
- 결과가 너무 클 수 있으니 문자열로 반환

**입출력 예시**

| numbers | return |
|---|---|
| [6, 10, 2] | "6210" |
| [3, 30, 34, 5, 9] | "9534330" |
| [0, 0, 0] | "0" |

### 정답 코드

```java
import java.util.Arrays;

public class Solution {
    public String solution(int[] numbers) {
        // int → String 변환
        String[] strs = new String[numbers.length];
        for (int i = 0; i < numbers.length; i++) {
            strs[i] = String.valueOf(numbers[i]);
        }

        // 커스텀 정렬: (b+a) vs (a+b) 비교
        Arrays.sort(strs, (a, b) -> (b + a).compareTo(a + b));

        // 엣지케이스: 첫 글자가 0이면 전체가 0
        if (strs[0].equals("0")) return "0";

        StringBuilder sb = new StringBuilder();
        for (String s : strs) sb.append(s);
        return sb.toString();
    }
}
```

### 핵심 포인트
- 핵심은 `(b+a).compareTo(a+b)` **문자열 이어붙여 비교**
  - 예: `"3"`과 `"30"` 비교 시 `"303"` vs `"330"` → `"330"`이 크므로 3이 앞
- `[0, 0, 0]` 같은 엣지케이스 처리 필수 (모두 0이면 `"000"`이 아닌 `"0"` 반환)
- `StringBuilder`로 문자열 합치기 (성능)

---

## 🎯 시험 직전 체크리스트

| 유형 | 꼭 외울 패턴 |
|---|---|
| 해시맵 | `map.getOrDefault(key, 0) + 1` |
| DFS | 종료조건 → 선택지별 재귀 호출 |
| 정렬 | `Arrays.sort(arr, (a, b) -> ...)` |

## 💡 시험장 팁

1. 문제 읽고 **입출력 예시 손으로 직접 따라 그려보기**
2. 엣지케이스 먼저 떠올리기 (빈 배열, 0, 중복, 단일 원소)
3. 시간복잡도 체크 - N이 10만 이상이면 O(N²) 안 됨
4. 제출 전 예시 한 번 더 검증
