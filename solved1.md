# ✅ 실습 기반 문제

## 1. 로그 아카이빙
### 1-1. 실습 요구사항

- `~/logs/` 에서 mtime 기준 **30일 이상** 경과한 파일을 검색
- 대상 파일을 `~/archive/` 로 이동
- 이동된 파일을 `old_logs_2026.tar.gz` 으로 압축 후 원본 삭제

### 1-2. 실습 검증

#### a. 이동 대상 확인

```bash
find ~/logs -type f -mtime +30
```

#### b. archive 디렉토리로 이동

```bash
find ~/logs -type f -mtime +30 -exec mv {} ~/archive/ \;
```

#### c. 압축 및 원본 삭제

```bash
tar -czf ~/archive/old_logs_2026.tar.gz ~/archive/*.log --remove-files
```

### 1-3. 문제 발생 지점

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

## 2. 로그 레벨
### 2-1. 실습 요구사항

- `lab2/logs/app_recent.log`에서 각각 로그 레벨의 개수를 `awk`를 활용해 센다.
- `lab2/logs/access.json`에서 각각 로그 레벨의 개수를 `jq`를 활용해 센다.

### 2-2. 실습 검증

#### a. awk의 `$<행 번호>` 변수를 활용하는 방법
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

#### b. awk와 정규표현식을 활용한 방법
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

#### c. 실습 결과

![문제3풀이](Images\문제3-풀이.png)


### 2-3. 문제 발생 지점

awk로 작성된 스크립트는 불필요하게 양이 많고 복잡하다.\
로그가 JSON 형식으로 정제되어 있다면 `jq` 스크립트로 더욱 간단하게 로그 레벨 정보를 추출할 수 있다.

```bash
jq -n 'reduce inputs.level as $lvl ({}; .[$lvl] += 1)' access.json
```
#### a. jq 스크립트 핵심 원리

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

#### b. 실행 결과

![문제3풀이jq](Images\문제3jq.png)




## 3. 블랙리스트 만들기
### 1-1. 실습 요구사항
**시나리오**

> 당신은 보안팀 신입이다.
> 팀장이 웹서버 로그(`webserver_log.log`)를 던져주며 말했다.
>
> *"최근 서버에 이상한 로그인 시도가 많아. 로그 뒤져서 수상한 놈들 뽑아서 블랙리스트 만들어봐."*

**조건**

| 항목 | 값 |
|------|----|
| 사용 파일 | `webserver_log.log` |
| 최종 결과물 | `result.csv`, `blacklist.txt` |
| 로그인 요청 | `POST /api/login` |
| 실패 기준 | 상태코드 `401` |
| 블랙리스트 기준 | 실패 횟수 `10회 이상` |

**문제**

**Q1.** 로그 파일에서 로그인 실패한 줄만 출력하라

**Q2.** 유저명 / 실패횟수 / 마지막시도시간 / IP를 추출하여 `result.csv` 로 저장하라

**Q3.** `result.csv` 에서 실패 10회 이상인 유저와 IP를 `blacklist.txt` 로 추출하라

### 3-2. 실습 검증

**Q1 정답**
```
grep "POST /api/login" webserver_log.log | grep " 401 "
```

**Q2 정답**
```
echo "User,실패횟수,마지막시도,IP" > result.csv

grep "POST /api/login" webserver_log.log | grep " 401 " \
  | tr -d '[' \
  | awk '{
      count[$3]++
      last[$3] = $4
      if (ips[$3] !~ $1) {
        if (ips[$3] == "") ips[$3] = $1
        else ips[$3] = ips[$3] ";" $1
      }
    }
    END {
      for (user in count)
        print user","count[user]","last[user]","ips[user]
    }' >> result.csv
```

**Q3 정답**
```
awk -F',' 'NR>1 && $2 >= 10 {print $1, $4}' result.csv > blacklist.txt
```

**최종 결과**
```
# result.csv
User,실패횟수,마지막시도,IP
hacker01,17,28/Mar/2024:22:15:33,45.33.32.156
hacker02,23,28/Mar/2024:23:51:55,185.220.101.5

# blacklist.txt
hacker01 45.33.32.156
hacker02 185.220.101.5
```

## 4. 로그 포맷 변환
### 4-1. 실습 요구사항
`access_2024-01-10.log` 또는 `access_2026-01-10.log` 파일의 각 로그 행을 사람이 읽기 쉬운 형식으로 변형하여 출력한다.

- 대상 파일: `access_2024-01-10.log`, `access_2026-01-10.log`
- 사용 도구: `awk`
- 요구 조건:
  - 각 로그 행에서 시간, IP, METHOD, URL, 상태코드, 응답시간만 추출할 것
  - 큰따옴표(`"`)와 대괄호(`[]`)를 보기 좋게 정리할 것
  - 원본 로그 파일은 수정하지 않을 것
  - 결과는 표준 출력으로 확인할 것

출력 형식:
```text
[시간] IP → METHOD URL (상태코드, 응답시간s)
```

### 4-2. 실습 검증
```bash
awk '{
    time = substr($4,2) " " substr($5,1,length($5)-1)
    method = substr($6,2)
    url = $7
    status = $9
    response = $11

    printf "[%s] %s → %s %s (%s, %ss)\n", time, $1, method, url, status, response
}' access_2024-01-10.log
```

#### b. 실행 결과
![문제4풀이jq](Images\problem4.png)


## 5. 신입의 권한
### 5-1. 실습 요구사항

당신은 시스템 관리자(`root`)입니다.
개발 서버에는 `root`와 `ubuntu` 두 계정만 존재합니다.
`ubuntu`는 신입 개발자가 사용하는 일반 계정입니다.

>  **"신입 개발자(ubuntu)가 프로젝트 소스를 수정할 수 있고, 로그 파일까지 편집 가능한 현재 상태"**

#### 계정 현황

| 사용자 | 역할 | 비고 |
|--------|------|------|
| `root` | 시스템 관리자 | 모든 권한 보유 |
| `ubuntu` | 신입 개발자 | 읽기만 허용되어야 함 |


#### 조건

**[소유권 & 허가권]**

- `lab2/webserver/` — 소유자 `root`, 소유 그룹 `root`, 권한 `755`
  - `root`는 읽기/쓰기/실행, `ubuntu`는 읽기+실행만 (쓰기 불가)
- `lab2/archive/` — SGID를 설정하여 새로 생성되는 파일의 그룹 소유가 자동 유지되도록 한다
- `lab2/logs/` — 소유자 `root`, 소유 그룹 `root`, 권한 `700`
  - `root`만 읽기/쓰기/실행, `ubuntu`는 접근 자체가 불가능

**[위험 파일 탐지 — find]**

- `lab2/` 하위에서 권한이 `777`인 파일은 `750`으로, 디렉토리는 `755`로 일괄 수정하는 명령어를 작성한다
- `lab2/logs/`에서 other에게 쓰기 권한이 있는 파일을 찾아 쓰기 권한을 제거한다

---

### 5-2. 실습 검증

#### 1. 소유권 & 허가권 설정

**lab2/webserver/ — root가 읽기/쓰기, ubuntu는 읽기만**

```bash
sudo chown -R root:root lab2/webserver
sudo chmod -R 755 lab2/webserver
```

`755` = 소유자(root) rwx / 그룹(root) r-x / other(ubuntu) r-x
→ `ubuntu`는 other에 해당하므로 읽기+실행만 가능, 쓰기 불가

**lab2/archive/ — SGID 설정**

```bash
sudo chown root:root lab2/archive
sudo chmod 755 lab2/archive
sudo chmod g+s lab2/archive
```

SGID가 없으면 사용자가 파일을 만들 때 그 사용자의 기본 그룹이 소유 그룹이 된다.
SGID가 있으면 누가 만들든 그룹이 항상 디렉토리의 그룹(root)으로 유지된다.

**lab2/logs/ — root만 접근, ubuntu 완전 차단**

```bash
sudo chown -R root:root lab2/logs
sudo chmod -R 700 lab2/logs
```

`700` = 소유자(root) rwx / 그룹 --- / other(ubuntu) ---
→ `ubuntu`는 읽기, 쓰기, 실행 모두 불가능. `ls`조차 할 수 없다.

#### 2. 위험 파일 탐지 — find

**777 파일을 750으로 수정**

```bash
sudo find lab2/ -type f -perm 0777 -exec chmod 750 {} \;
```

- `-type f`는 일반 파일만, `-perm 0777`은 권한이 정확히 777인 것만 찾는다.
- `-exec chmod 750 {} \;`는 찾은 파일 하나하나에 `chmod 750`을 실행한다.

**777 디렉토리를 755로 수정**

```bash
sudo find lab2/ -type d -perm 0777 -exec chmod 755 {} \;
```

**로그 디렉토리에서 other 쓰기 권한 제거**

```bash
sudo find lab2/logs/ -type f -perm -o+w -exec chmod o-w {} \;
```

`-perm -o+w`에서 `-`는 "해당 비트가 포함된" 것을 의미한다.
other 쓰기 비트가 켜져 있는 파일만 찾아서 `chmod o-w`로 해당 비트만 제거한다.

<img width="652" height="127" alt="image" src="https://github.com/user-attachments/assets/5bdcfc95-41eb-4a36-8901-8e22886d9652" />
