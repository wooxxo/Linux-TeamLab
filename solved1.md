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
