## 1-1. 실습 요구사항 (로그 아카이빙)

- `~/logs/` 에서 mtime 기준 **30일 이상** 경과한 파일을 검색
- 대상 파일을 `~/archive/` 로 이동
- 이동된 파일을 `old_logs_2026.tar.gz` 으로 압축 후 원본 삭제

## 1-2. 실습 검증

### a. 이동 대상 확인

```bash
find ~/logs -type f -mtime +30
```

### b. archive 디렉토리로 이동

```bash
find ~/logs -type f -mtime +30 -exec mv {} ~/archive/ \;
```

### c. 압축 및 원본 삭제

```bash
tar -czf ~/archive/old_logs_2026.tar.gz ~/archive/*.log --remove-files
```

---

## 1-3. 문제 발생 지점

평문 로그는 **파일의 mtime** 기준으로만 판단하기 때문에 `touch` 로 조작되거나 파일 하나에 여러 날짜 로그가 섞여 있으면 정확하지 않다.

```
# 한 파일 안에 날짜가 섞인 경우
[2026-01-10 00:05:00] GET /api/users   ← 오래된 로그
[2026-03-19 00:05:00] GET /api/users   ← 최신 로그
```

로그가 JSON 형식으로 정제되어 있다면 `jq` 로 **로그 내용의 timestamp 값** 기준으로 더 정확하게 필터링할 수 있다.

```bash
# 30일 이전 로그만 추출
jq 'select(.timestamp < "2026-02-17")' ~/access.json > ~/archive/old_access.json
```

| 구분 | find -mtime (평문) | jq (JSON) |
|------|------------------|-----------|
| 기준 | 파일 수정시간 (mtime) | 로그 내용의 timestamp |
| 정확도 | touch 조작 시 오차 발생 | 내용 기준이라 정확 |
| 혼합 로그 | 파일 단위라 분리 불가 | 레코드 단위로 정확히 분리 |

---

## 2-1. 실습 요구사항

- `lab2/logs/app_recent.log`에서 각각 로그 레벨의 개수를 `awk`를 활용해 센다.
- `lab2/logs/access.json`에서 각각 로그 레벨의 개수를 `jq`를 활용해 센다.

---

## 2-2. 실습 검증

### a. awk의 `$<행 번호>` 변수를 활용하는 방법
> 로그 파일에서 로그 레벨이 몇 번째 행에 있는지 정확히 알 때 활용하는 방법

```bash
# !/bin/bash

echo "---전체 로그 레벨 개수($ 사용)---"
awk -F ' ' \
'{if($3 == "[INFO]") info_sum += 1; \
else if($3 == "[DEBUG]") debug_sum += 1; \
else if($3 == "[WARN]") warn_sum += 1; \
else if($3 == "[ERROR]") error_sum += 1; \
else if($3 == "[FATAL]") fatal_sum += 1; \
else unknown_sum += 1; \
} \
END{printf "INFO: %d\n", info_sum; \
printf "DEBUG: %d\n", debug_sum; \
printf "WARN: %d\n", warn_sum; \
printf "ERROR: %d\n", error_sum; \
printf "FATAL: %d\n", fatal_sum; \
printf "알 수 없는 로그: %d\n", unknown_sum;
print "" \
}' ~/lab2/logs/app_recent.log
```

### b. awk와 정규표현식을 활용한 방법
> 로그 파일에 로그 레벨이 몇 번째 행에 있는지 몰라도 사용 가능한 방법

```bash
# !/bin/bash

echo "---전체 로그 레벨 개수(정규표현식 사용)---"
awk \
'BEGIN{IGNORECASE = 1} \
{if($0 ~ /info/) info_sum += 1; \
else if($0 ~ /debug/) debug_sum += 1; \
else if($0 ~ /warn/) warn_sum += 1; \
else if($0 ~ /error/) error_sum += 1; \
else if($0 ~ /fatal/) fatal_sum += 1; \
else if($0 !~ /info|debug|warn|error|fatal/) unknown_sum += 1 \
} \
END{printf "INFO: %d\n", info_sum; \
printf "DEBUG: %d\n", debug_sum; \
printf "WARN: %d\n", warn_sum; \
printf "ERROR: %d\n", error_sum; \
printf "FATAL: %d\n", fatal_sum; \
printf "알 수 없는 로그: %d\n", unknown_sum;
print "" \
}' ~/lab2/logs/app_recent.log
```

### c. 실습 결과

![문제3풀이](Images\문제3-풀이.png)

---

## 2-3. 문제 발생 지점

awk로 작성된 스크립트는 불필요하게 양이 많고 복잡하다.\
로그가 JSON 형식으로 정제되어 있다면 `jq` 스크립트로 더욱 간단하게 로그 레벨 정보를 추출할 수 있다.

```bash
jq -n 'reduce inputs.level as $lvl ({}; .[$lvl] += 1)' access.json
```
### a. jq 스크립트 핵심 원리

1. **`n` (Null input 옵션)과 `inputs`**
    - `n`: `jq`가 파일을 한 번에 통째로 읽어들이지 않도록 제어
    - `inputs`: `.json` 파일의 객체들을 하나씩 차례대로 스트리밍하여 읽음. (수 기가바이트의 대용량 로그를 처리할 때 메모리 폭발을 막아주는 핵심 기능)
2. **`reduce ... as $lvl ({초기값}; 업데이트 로직)`**
    - 스트리밍되는 각 로그 객체의 `.level` 값을 추출하여 `$lvl` 이라는 임시 변수에 담고 반복문 실행.
3. **`{}` (초기 상태)**
    - 결과를 저장할 빈 JSON 객체(`{}`)를 초기값으로 생성
4. **`.[$lvl] += 1` (카운팅 로직)**
    - 방금 읽어온 레벨 이름(`$lvl`)을 키(Key)로 삼아 값을 1씩 증가
    - ex: 처음 `WARN`을 만나면 `{"WARN": 1}`이 되고, 다음에 또 `WARN`을 만나면 `{"WARN": 2}`로 누적

### b. 실행 결과

![문제3풀이jq](Images\문제3jq.png)
