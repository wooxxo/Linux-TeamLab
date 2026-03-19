# Linux-TeamLab

## 👩🏻‍💻 Team DEV6

| <img src="https://github.com/Federico-15.png" width="120"/> |<img src="https://github.com/wooxxo.png" width="120"/> | <img src="https://github.com/HiLeeS.png" width="120"/> | <img src="https://github.com/ygreee0320.png" width="120"/> | <img src="https://github.com/cuterrabbit.png" width="120"/> | <img src="https://github.com/Zaixian5.png" width="120"/> | 
|:--:|:--:|:--:|:--:|:--:|:--:|
| [**류승환**](https://github.com/Federico-15) | [**우승연**](https://github.com/wooxxo) | [**이승준**](https://github.com/HiLeeS) | [**양규리**](https://github.com/ygreee0320) | [**이동욱**](https://github.com/Soooonnn) | [**사재헌**](https://github.com/Zaixian5) |

## 🛠️ 실습 환경 설정
### 1. 📁 로그 디렉토리 구조
    
    lab1/
    ├── logs/
    │   ├── project-a/
    │   │   └── app.log
    │   ├── project-b/
    │   │   └── app.log
    │   ├── access_2026-01-10.log
    │   ├── access_2026-01-25.log
    │   ├── access_2026-02-14.log
    │   ├── access_2026-02-17.log
    │   ├── access_2026-03-05.log
    │   ├── access_2026-03-15.log
    │   ├── access_2026-03-19.log
    │   ├── app_recent.log
    │   ├── webserver_login.log
    │   └── admin_audit.log
    └── archive/              ← 비어있음 (문제 2 이동 목적지)


```
lab2/
├── logs/
│   ├── project-a/
│   │   └── app.log
│   ├── project-b/
│   │   └── app.log
│   ├── access.json
│   ├── app_recent.log
│   ├── webserver_login.log
│   └── admin_audit.log
└── archive/              ← 비어있음 (문제 2 이동 목적지)
```

### 2. 📁 파일 목록 및 역할
**1. access_*.log**
* 웹 요청 로그 (핵심 데이터)  

* **포함 정보 :** `IP` `시간` `METHOD` `URL` `상태코드` `응답시간`
    ```
    45.33.32.156 - - [10/Jan/2024:02:31:05 +0900] "DELETE /index.html HTTP/1.1" 200 37897 1.584 "https://example.com" "python-requests/2.28.0"
    ```
> 대부분의 실습 문제에 사용 가능


 **2. webserver_login.log**
* 로그인·인증 관련 로그  

* **포함 정보 :** `IP` `유저명` `시간` `METHOD` `URL` `상태코드`
    ```
    185.220.101.5 - hacker02 [28/Mar/2024:01:22:11 +0900] "POST /api/login HTTP/1.1" 401 1285 0.968 "https://google.com" "-"
    ```

> 특정 IP / 유저의 로그인 실패 패턴 분석, blacklist 생성 문제에 최적

**3. admin_audit.log**
* 관리자 행동 감사 로그  

* **포함 정보 :** `USER` `ACTION` `TARGET` `STATUS` `DURATION`
    ```
    [2024-03-28 01:08:13] USER=alice ACTION=SEARCH TARGET=/api/users STATUS=DENIED DURATION=122ms
    ```

> 권한 분석, 정상 vs 비정상 행동 탐지 문제용


**4. app_recent.log / app.log**
* 시스템 내부 로그  

* **포함 정보 :** `날짜시간` `LEVEL(ERROR/WARN/INFO/DEBUG)` `모듈` `메시지`
    ```
    [2024-03-28 10:36:33] [ERROR] [scheduler] Database connection failed: timeout after 30s
    [2024-03-28 04:21:46] [WARN]  [scheduler] Disk usage at 80%
    ```

> 장애 분석, 에러 패턴 탐지, 레벨별 필터링 문제용

## ✅ [학습 내용 기반 문제](/solved1.md)

### 1. 
### 2. 
### 3. 
### 4. 
### 5. 


## ✅ [JSON/YAML 기반 문제](/solved2.md)

## 🛠️ 기술 스택

| 분류 | 도구 |
|------|------|
| 가상화 플랫폼 | VMware Workstation, VirtualBox |
| 운영체제 | Ubuntu 24.04 LTS |
| 터미널 | MobaXterm |