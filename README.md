# 🛡️ 보안 뉴스 자동화 & 취약점 탐지 시스템

> **n8n** 기반의 자동화 워크플로우로 매일 아침 보안 뉴스를 수집·분석하고,  
> 내 자산(CMDB)과 연관된 취약점을 즉시 탐지하여 경보를 발송하는 시스템입니다.

<br>

## 📋 목차

1. [프로젝트 개요](#-프로젝트-개요)
2. [프로젝트 배경](#-프로젝트-배경)
3. [전체 워크플로우](#️-전체-워크플로우)
4. [워크플로우 1 — 보안 뉴스레터 발송](#-워크플로우-1--보안-뉴스레터-발송)
5. [워크플로우 2 — AI 취약점 분석 & CMDB 매칭 경보](#-워크플로우-2--ai-취약점-분석--cmdb-매칭-경보)
6. [데이터베이스 스키마](#️-데이터베이스-스키마)
7. [기술 스택](#️-기술-스택)
8. [실행 방법](#-실행-방법)
9. [트러블슈팅](#-트러블슈팅)
10. [향후 개선 방향](#-향후-개선-방향)
11. [용어 사전](#-용어-사전-glossary)

<br>

---

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **도구** | n8n (자동화), PostgreSQL (DB), Groq AI (분석) |
| **실행 주기** | 매일 오전 자동 실행 (Schedule Trigger) |
| **뉴스 소스** | DailySec, TheHackerNews, BleepingComputer, KISA |
| **핵심 기능** | 뉴스 수집 → AI 분석 → CMDB 매칭 → 이메일 경보 |

<br>

## 💡 프로젝트 배경

보안 담당자로서 매일 아침 수십 개의 보안 뉴스를 직접 찾아보는 건 비효율적입니다.  
특히 **"이 취약점이 우리 시스템에도 해당되는가?"** 를 수동으로 판단하는 것은 놓치기 쉽고 위험합니다.

이 시스템은 다음 문제를 해결합니다:

- ✅ 매일 4개 소스의 보안 뉴스를 **자동 수집**하여 뉴스레터로 발송
- ✅ AI(Groq)가 각 뉴스를 **구조화된 데이터**로 분석 (CVE ID, 심각도, 대응방안 등)
- ✅ **CMDB(내부 자산 목록)** 와 자동 비교하여 우리 자산에 해당하는 위협 즉시 탐지
- ✅ 위협 탐지 시 **즉시 이메일 경보** 발송 → 빠른 대응 가능

> 예: `EFM-Networks ipTIME 취약점` 뉴스 수신 → CMDB에 ipTIME 장비 존재 → **자동 보안 경보 발송!**

<br>

---

## 🗺️ 전체 워크플로우

<img width="1106" height="400" alt="전체 워크플로우" src="https://github.com/user-attachments/assets/ade380f8-05e0-4fe0-ac57-f0f81ef0d48d" />



전체 파이프라인은 크게 **두 가지 흐름**으로 구성됩니다:

| 흐름 | 설명 |
|------|------|
| 🗞️ **뉴스레터 발송** | RSS 수집 → 필터링 → Merge → JavaScript 처리 → 이메일 발송 |
| 🔍 **취약점 탐지** | RSS 수집 → AI 분석 → DB 저장 → CMDB 매칭 → 경보 발송 |

<br>

---

## 📰 워크플로우 1 — 보안 뉴스레터 발송

<img width="677" height="384" alt="뉴스워크플로우" src="https://github.com/user-attachments/assets/b7cf7a02-32d2-48d1-9afd-ac4cb53f4157" />


### 동작 순서

```
Schedule Trigger
    │
    ├─► RSS Read1 (DailySec)   → Filter2 ──┐
    ├─► RSS Read2 (TheHackerNews)          ├─► Merge (append)
    ├─► RSS Read3 (BleepingComputer)       │       │
    └─► RSS Read4 (KISA) ─────────────────┘       │
                                                   ▼
                                        Code in JavaScript2
                                        (HTML 뉴스레터 생성)
                                                   │
                                                   ▼
                                           Send email2 📧
```

### 핵심 로직 (Code in JavaScript2)

소스별로 최대 7개씩 선별 후 랜덤 셔플하여 균형 잡힌 뉴스레터를 생성합니다.

```javascript
// 소스별 분류 및 색상 태그 지정
if (link.includes('boannews'))        boan.push({ ...data, tag: '[보안뉴스]',     color: '#d32f2f' });
else if (link.includes('dailysecu'))  daily.push({ ...data, tag: '[데일리시큐]',  color: '#1a73e8' });
else if (link.includes('krcert'))     kisa.push({ ...data, tag: '[KISA]',        color: '#2e7d32' });
else if (link.includes('thehackernews')) hackers.push({ ...data, tag: '[TheHackerNews]', color: '#f57c00' });
else if (link.includes('bleepingcomputer')) bleeping.push({ ...data, tag: '[BleepingComp]', color: '#455a64' });

// 각 소스에서 최대 7개 선택 후 랜덤 믹스
const finalSelection = [
    ...boan.slice(0, 7), ...daily.slice(0, 7),
    ...kisa.slice(0, 7), ...hackers.slice(0, 7), ...bleeping.slice(0, 7)
];
finalSelection.sort(() => 0.5 - Math.random());
```

### 실행 결과 — 이메일 수신 화면

<img width="799" height="619" alt="뉴스 메일" src="https://github.com/user-attachments/assets/ceac2d1c-c638-45fa-be84-bad21b4b0563" />

<img width="827" height="606" alt="뉴스" src="https://github.com/user-attachments/assets/5d837fef-02ee-4ec4-a609-832c6e209f2f" />

<br>

---

## 🔍 워크플로우 2 — AI 취약점 분석 & CMDB 매칭 경보

<img width="1232" height="277" alt="취약점 워크플로우" src="https://github.com/user-attachments/assets/e2fa7570-1a53-4eef-a220-3a2c77aa482f" />


### 동작 순서

```
Filter (보안 관련 키워드 포함 뉴스만 통과)
    │
    ▼
AI Agent (Groq) — 뉴스 분석 및 구조화
    │  - CVE ID 추출
    │  - 심각도(Severity) 판단
    │  - 영향받는 서비스명 추출
    │  - 대응방안 요약
    ▼
Code in JavaScript — AI 응답 JSON 파싱
    │
    ▼
Insert rows in a table — vulnerability_news 테이블에 저장
    │
    ▼
Execute a SQL query — CMDB 전체 자산 목록 조회
    │
    ▼
Code in JavaScript1 — CMDB vs 취약점 매칭 비교
    │
    ├─ 매칭 O ─► Insert rows in a table1 (security_alerts 저장)
    │                    │
    │                    ▼
    │               Send email 📧 (보안 경보 발송!)
    │
    └─ 매칭 X ─► (종료)
```

### AI Agent 프롬프트 (Groq 모델)

Groq AI가 뉴스 원문을 분석해 아래 형태의 JSON을 반환합니다:

```json
{
  "cve_id": "CVE-2026-24498",
  "target_service": "ipTIME",
  "severity": "HIGH",
  "summary": "EFM-Networks ipTIME 유무선공유기의 보안 기능이 우회되어 내부 네트워크 접근 가능",
  "solution": "최신 펌웨어 업데이트 설치 권장"
}
```

### CMDB 매칭 로직 (Code in JavaScript1)

```javascript
for (const asset of cmdbData) {
    const serviceName   = (assetJson.service_name || "").toLowerCase();
    const targetService = (news.target_service   || "").toLowerCase();

    // 양방향 포함 관계 확인 (부분 일치도 탐지)
    if (targetService.includes(serviceName) || serviceName.includes(targetService)) {
        alerts.push({
            json: {
                is_matched:   true,
                asset_id:     assetJson.id,
                news_id:      news.id,
                alert_level:  news.severity,
                asset_name:   assetJson.asset_name,
                service_info: `${assetJson.service_name} (v${assetJson.service_version})`,
                news_title:   news.title,
                cve_id:       news.cve_id,
            }
        });
    }
}
return alerts.length > 0 ? alerts : [{ json: { is_matched: false } }];
```

### 실행 결과 — 보안 경보 이메일

<img width="794" height="407" alt="보안알림" src="https://github.com/user-attachments/assets/41ada251-6cbe-49e4-9649-bcf816d8782a" />


<br>

---

## 🗄️ 데이터베이스 스키마

### ERD 구조

```
┌─────────────────────┐        ┌──────────────────────────┐
│        CMDB         │        │    vulnerability_news     │
├─────────────────────┤        ├──────────────────────────┤
│ id (PK)             │        │ id (PK)                  │
│ asset_name          │        │ title                    │
│ ip_address          │        │ summary                  │
│ os_info             │        │ cve_id                   │
│ service_name        │        │ target_service           │
│ service_version     │        │ severity                 │
│ last_scan_date      │        │ solution                 │
└──────────┬──────────┘        │ link                     │
           │                   │ created_at               │
           │           ┌───────┴──────────────────────────┐
           │           │                                  │
           └───────────►        security_alerts           │
                       ├──────────────────────────────────┤
                       │ id (PK)                          │
                       │ asset_id (FK → CMDB)             │
                       │ news_id  (FK → vulnerability_news)│
                       │ alert_level                      │
                       │ detected_at                      │
                       │ status (기본값: 'Open')           │
                       └──────────────────────────────────┘
```

### 테이블 상세

#### `CMDB` — 내부 자산 관리

```sql
CREATE TABLE IF NOT EXISTS CMDB (
    id               SERIAL PRIMARY KEY,
    asset_name       VARCHAR(100),
    ip_address       VARCHAR(45),
    os_info          VARCHAR(50),
    service_name     VARCHAR(50),    -- 매칭의 핵심 컬럼
    service_version  VARCHAR(20),
    last_scan_date   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `vulnerability_news` — AI 분석 취약점 정보

```sql
CREATE TABLE IF NOT EXISTS vulnerability_news (
    id              SERIAL PRIMARY KEY,
    title           TEXT,
    summary         TEXT,
    cve_id          VARCHAR(50),
    target_service  VARCHAR(50),     -- CMDB.service_name 과 비교
    severity        VARCHAR(20),
    solution        TEXT,
    link            TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `security_alerts` — 탐지된 위협 경보 로그

```sql
CREATE TABLE IF NOT EXISTS security_alerts (
    id          SERIAL PRIMARY KEY,
    asset_id    INTEGER REFERENCES CMDB(id),
    news_id     INTEGER REFERENCES vulnerability_news(id),
    alert_level VARCHAR(20),
    detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status      VARCHAR(20) DEFAULT 'Open'
);
```

| 테이블 | 역할 | 성격 |
|--------|------|------|
| `CMDB` | 내부 자산(서버, IP, 서비스 버전) 기준 정보 | 수동 관리 |
| `vulnerability_news` | AI가 분석한 외부 위협 정보 | 자동 적재 |
| `security_alerts` | CMDB × 뉴스 매칭 탐지 결과 | 자동 생성 |

<br>

---

## 📚 용어 사전 (Glossary)

> 이 프로젝트를 만들며 학습한 핵심 용어들을 정리했습니다.

| 용어 | 설명 |
|------|------|
| **n8n** | 노드 기반의 오픈소스 워크플로우 자동화 도구. 코드 없이도 API·DB·AI를 연결 가능 |
| **RSS** | Really Simple Syndication. 웹사이트의 최신 콘텐츠를 구독할 수 있는 XML 형식의 피드 |
| **Schedule Trigger** | 지정된 시간마다 워크플로우를 자동으로 실행시키는 n8n 노드 |
| **Merge (append)** | 여러 노드의 출력을 하나의 스트림으로 합치는 n8n 노드 |
| **CVE** | Common Vulnerabilities and Exposures. 공개된 보안 취약점에 부여되는 고유 식별 번호 (예: CVE-2026-24498) |
| **CMDB** | Configuration Management Database. 조직의 IT 자산(서버, 네트워크 장비 등) 구성 정보를 관리하는 DB |
| **KISA** | 한국인터넷진흥원. 국내 사이버 보안 위협·취약점 정보를 공식 발표하는 기관 |
| **Groq** | 고속 AI 추론 서비스. LLaMA 등 오픈소스 LLM을 빠르게 실행할 수 있는 플랫폼 |
| **AI Agent (n8n)** | LLM을 활용해 입력 데이터를 분석·판단하고 구조화된 출력을 생성하는 n8n 노드 |
| **Severity** | 취약점의 심각도 등급. CRITICAL / HIGH / MEDIUM / LOW 로 분류 |
| **Security Alert** | 탐지된 위협을 담당자에게 알리는 경보. 본 시스템에서는 DB 저장 + 이메일 발송으로 구현 |
| **Air-Gapped Network** | 인터넷과 물리적으로 분리된 폐쇄망. 고보안 환경에서 사용 |
| **RAT** | Remote Access Trojan. 공격자가 원격으로 시스템을 제어하기 위해 사용하는 악성 소프트웨어 |
| **Shodan** | 인터넷에 연결된 장비를 스캔하는 검색 엔진. IP 조회만으로 외부에 노출된 포트·서비스 확인 가능 |
| **NVD** | National Vulnerability Database. 미국 NIST가 운영하는 공식 CVE 데이터베이스. CVSS 점수 및 영향 제품 목록 제공 |
| **CVSS** | Common Vulnerability Scoring System. 취약점 심각도를 0~10점으로 수치화하는 국제 표준 |
| **Rate Limit** | API 서비스가 일정 시간 내 허용하는 최대 요청 횟수. 초과 시 429 에러 반환 |

<br>

---

## 🛠️ 기술 스택


<table>
  <tr>
    <th align="left">Category</th>
    <th align="left">Stack</th>
  </tr>

  <tr>
    <td><b>Automation</b></td>
    <td>
      <img src="https://img.shields.io/badge/n8n-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white"/>
    </td>
  </tr>

  <tr>
    <td><b>AI Analysis</b></td>
    <td>
      <img src="https://img.shields.io/badge/Groq-000000?style=for-the-badge&logo=groq&logoColor=white"/>
      <img src="https://img.shields.io/badge/LLaMA-4A4A4A?style=for-the-badge"/>
    </td>
  </tr>

  <tr>
    <td><b>Database</b></td>
    <td>
      <img src="https://img.shields.io/badge/PostgreSQL-336791?style=for-the-badge&logo=postgresql&logoColor=white"/>
    </td>
  </tr>

  <tr>
    <td><b>Notification</b></td>
    <td>
      <img src="https://img.shields.io/badge/SMTP-0078D4?style=for-the-badge&logo=gmail&logoColor=white"/>
      <img src="https://img.shields.io/badge/Email-6C757D?style=for-the-badge&logo=minutemailer&logoColor=white"/>
    </td>
  </tr>

  <tr>
    <td><b>Language</b></td>
    <td>
      <img src="https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black"/>
      <img src="https://img.shields.io/badge/n8n%20Code%20Node-FF6D5A?style=for-the-badge"/>
    </td>
  </tr>

</table>

<br>

---

## 🚀 실행 방법

1. **n8n 설치 및 실행**
   ```bash
   npx n8n
   ```

2. **PostgreSQL 테이블 생성** — 위 DDL 쿼리 순서대로 실행
   ```
   CMDB → vulnerability_news → security_alerts
   ```

3. **CMDB에 자산 등록** — 모니터링할 서비스 정보 입력

4. **n8n에서 워크플로우 Import** — `.json` 파일 업로드

5. **크리덴셜 설정**
   - PostgreSQL 연결 정보
   - Groq API Key
   - SMTP 이메일 계정

6. **Schedule Trigger 활성화** → 매일 자동 실행 🎉

<br>

---

## 🔧 트러블슈팅

> 프로젝트를 구축하며 실제로 맞닥뜨린 문제와 해결 과정을 기록했습니다.

<br>

### 🐳 인프라 이슈

#### Issue 1. 포트 충돌 (Port Already Allocated)

- **증상:** `docker-compose up` 실행 시 `Bind for 0.0.0.0:5678 failed: port is already allocated` 에러 발생
- **원인:** 과거에 실행했던 독립형 n8n 컨테이너가 동일한 포트(5678)를 점유하고 있어 신규 컨테이너가 구동 불가
- **해결:**
  ```bash
  docker ps                  # 충돌 컨테이너 ID 확인
  docker rm -f <container_id>  # 강제 종료 및 삭제
  docker-compose up -d       # 재실행
  ```
- **교훈:** 컨테이너 환경에서는 프로세스 종료뿐만 아니라 **포트 매핑 상태를 명확히 관리**해야 함

<br>

#### Issue 2. 암호화 키 불일치 (Encryption Key Mismatch)

- **증상:** n8n 컨테이너 로그에서 `Mismatching encryption keys` 에러와 함께 무한 재시작 발생
- **원인:** 기존 볼륨 데이터에 저장된 암호화 키와 `docker-compose.yml`의 `N8N_ENCRYPTION_KEY` 환경변수 값이 불일치 → 데이터 복호화 실패
- **해결:** 초기 개발 단계임을 고려하여 `n8n_data` 볼륨을 초기화하고 환경변수 키 값을 고정하여 데이터 일관성 확보
- **교훈:** n8n의 보안 메커니즘을 이해하고, **운영 환경 이관 시 암호화 키 관리**의 중요성을 학습함

<br>

### 🤖 AI Agent 이슈

#### Issue 3. `429 Too Many Requests` — Rate Limit 초과 (Gemini)

- **증상:** 수집된 수십 개의 뉴스를 AI가 한꺼번에 처리하려다 Gemini API 무료 티어 요청 제한 초과
- **해결 시도 1 (실패):** `Batch Size = 1`, `Delay = 5000ms` 설정 → 구글 서버의 일시적 차단으로 미해결

- **해결책 A — 데이터 전처리(Filtering) 도입**

  | 항목 | 전 | 후 |
  |------|----|----|
  | AI 처리 건수 | 50개 | 7개 |
  | 필터 기준 | 없음 | 제목에 `취약점` / `CVE` / `보안` 포함 여부 |
  | 효과 | - | 작업 부하 80% 감소, 토큰 비용 절감 |

- **해결책 B — AI 모델 교체 (Gemini → Groq)**

  속도와 할당량이 넉넉한 **Groq** 플랫폼으로 엔진 교체.  
  단, 초기 선택한 `llama3-8b-8192` 모델이 지원 종료(Decommissioned)되어 `Bad Request` 에러 재발.  
  → 최종적으로 **`llama-3.3-70b-versatile`** 모델로 업데이트하여 분석 성공.

<br>

### 🔗 데이터 파이프라인 이슈

#### Issue 4. AI 응답의 "덩어리" 현상 (String vs JSON)

- **증상:** AI 분석은 성공했는데 PostgreSQL 노드에서 데이터가 통째로 들어가거나 `null` 발생
- **원인:** AI의 결과값은 기본적으로 하나의 긴 **문자열(String)** 이므로 DB가 요구하는 칸(Field)별 데이터와 형식 불일치
- **해결:** Code 노드에서 `JSON.parse()`로 문자열을 개별 Object로 강제 변환

  ```javascript
  const outputText = $json.output || "";
  const jsonMatch = outputText.match(/\{[\s\S]*\}/);
  const aiResult = JSON.parse(jsonMatch[0]);
  ```

<br>

#### Issue 5. `Paired Item Data Unavailable` — 데이터 연결 고리 유실

- **증상:** Code 노드 이후 PostgreSQL 노드에서 `title`, `link` 등이 빨간 글씨로 변하며 에러 발생
- **원인:** n8n은 노드를 거칠 때마다 데이터의 '뿌리(Origin)'를 추적하는데, Code 노드에서 새 데이터를 생성하면서 이전 노드(RSS, Filter) 데이터와의 **연결 고리가 끊김**
- **해결:** Code 노드 안에서 이전 노드의 필수 값을 직접 불러와 AI 분석 결과와 **병합(Merge)하여 배출**

  ```javascript
  const originalData = $("Filter").first().json;
  return {
    ...aiResult,
    original_title: originalData["title"],
    link:           originalData["link"],
    pubDate:        originalData["pubDate"]
  };
  ```

<br>

#### Issue 6. 데이터 타입 불일치 (`invalid input syntax for type timestamp`)

- **증상:** DB 삽입 시 `invalid input syntax for type timestamp` 에러 발생
- **원인:** 매핑 시 변수 값이 아닌 `"pubDate"` 라는 단순 텍스트가 입력됨
- **해결:** Expression 모드(`{{ }}`)에서 정확한 노드 경로의 **값(Value)을 참조**하도록 수정

<br>

---

## 🚀 향후 개선 방향

> 현재 시스템의 한계를 인식하고, 다음 단계로 발전시킬 아이디어를 정리했습니다.

<br>

### 🔎 Shodan API 연동 — 자산의 "외부 노출 현황" 파악

**현재 한계:** CMDB는 내가 직접 입력한 자산 정보만 담고 있어, 해당 서비스가 **인터넷에 실제로 노출되어 있는지** 알 수 없습니다.

**Shodan이란?**  
인터넷에 연결된 장비를 스캔하는 검색 엔진입니다. IP 주소를 조회하면 해당 서버에서 어떤 포트가 열려 있고, 어떤 서비스가 외부에 노출되어 있는지 확인할 수 있습니다.

**도입 시 효과:**

| 현재 흐름 | 개선 후 흐름 |
|-----------|-------------|
| 취약점 발견 → CMDB 서비스명 매칭 → 경보 | 취약점 발견 → CMDB 매칭 → **Shodan으로 외부 노출 여부 확인** → 경보 |
| 모든 매칭 자산에 동일 경보 | **인터넷에 노출된 자산은 CRITICAL**, 내부망 자산은 HIGH 등 **위험도 차등 적용** |

> 예: ipTIME 취약점 탐지 → Shodan으로 해당 IP의 80/443 포트 오픈 여부 확인 → 외부 노출 시 즉시 CRITICAL 경보

<br>

### 📋 NVD API 연동 — CVE 공식 데이터로 정확도 향상

**현재 한계:** CVE 정보를 AI가 뉴스 본문에서 **추론**하기 때문에, 심각도(CVSS 점수)나 영향 범위가 부정확할 수 있습니다.

**NVD(National Vulnerability Database)란?**  
미국 NIST에서 운영하는 공식 취약점 데이터베이스입니다. 모든 CVE에 대해 **공식 CVSS 점수, 영향받는 제품 목록, 패치 링크** 등 정형화된 데이터를 제공합니다.

**도입 시 효과:**

```
AI가 CVE-2026-24498 추출
         │
         ▼
NVD API 조회 → 공식 CVSS Score: 9.1 (CRITICAL)
         │       영향받는 제품: EFM-Networks ipTIME <= 12.17.0
         │       공식 패치 링크 자동 첨부
         ▼
훨씬 정확하고 신뢰도 높은 경보 발송
```

| 항목 | AI 추론 (현재) | NVD API (개선 후) |
|------|---------------|------------------|
| 심각도 | AI가 판단 (오류 가능) | CVSS 공식 점수 |
| 영향 버전 | 뉴스 본문 기반 | 공식 CPE 목록 |
| 패치 정보 | 요약 텍스트 | 공식 참조 링크 |

<br>

---
