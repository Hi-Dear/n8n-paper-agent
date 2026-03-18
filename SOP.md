# SOP: 논문 요약 정리 자동화 워크플로우

## 1. 워크플로우 개요

| 항목 | 내용 |
|------|------|
| **워크플로우 이름** | 논문 요약 정리 자동화 |
| **목적** | Google Drive에 저장된 논문 PDF를 자동 감지하여 AI로 요약하고, Notion DB에 정리 |
| **사용 서비스** | n8n, Google Drive, Google Gemini (2.5 Flash), Notion |
| **실행 방식** | Schedule Trigger (Polling) — 5분 간격 |

### 전체 흐름 요약

```
[Schedule Trigger] → [Google Drive 검색] → [중복 체크 (Notion 조회)]
    → [PDF 다운로드] → [Gemini 분석] → [Notion DB 저장]
```

---

## 2. 트리거 전략

### 2-1. 권장 방식: Schedule Trigger + Google Drive Search (Polling)

**방식**: Schedule Trigger로 5분마다 Google Drive API를 호출하여 대상 폴더(98_DB) 하위의 신규 PDF 파일을 검색한다.

**선정 근거**:
- Google Drive Trigger 노드는 **하위 폴더 변경을 감지할 수 없다** (`"Changes within subfolders won't trigger this node"`)
- 98_DB 폴더 구조가 `98_DB/분류폴더/논문.pdf` 형태이므로, 하위 폴더 재귀 탐색이 필수
- Google Drive API의 Search 기능은 상위 폴더 기준 하위 전체를 재귀적으로 탐색 가능

**중복 방지 전략**:
- Notion DB에서 Google Drive 파일 ID 존재 여부를 조회
- 이미 처리된 파일은 건너뛰어 중복 요약 방지

### 2-2. 대안 비교

| 방식 | 장점 | 단점 | 판정 |
|------|------|------|------|
| **Schedule + Search (Polling)** | 하위 폴더 재귀 탐색 가능, 안정적 | API 호출 비용 (5분마다), 최대 5분 지연 | **권장** |
| **Google Drive Trigger** | 실시간 감지, 간단한 설정 | 하위 폴더 변경 감지 불가 (치명적) | 부적합 |
| **Webhook (수동 트리거)** | 즉시 실행, 유연한 호출 | 자동화 불가, 외부 연동 필요 | 보조용 |

---

## 3. 처리 로직 (Step-by-step)

### Step 1: Schedule Trigger — 주기적 실행

- **역할**: 5분 간격으로 워크플로우를 자동 실행
- **설정**: Cron 표현식 또는 Interval 모드 (5분)

### Step 2: Google Drive Search — 신규 PDF 탐색

- **역할**: 98_DB 폴더 하위의 모든 PDF 파일을 재귀적으로 검색
- **검색 조건**: MIME type이 `application/pdf`이고, 상위 폴더가 98_DB인 파일
- **Google Drive API 쿼리 예시**:
  ```
  mimeType='application/pdf' and '<98_DB_FOLDER_ID>' in parents
  ```
  > 참고: 재귀 탐색이 필요한 경우 `trashed=false` 조건과 함께 폴더별 반복 탐색 또는 Drive 전체 검색 후 경로 필터링 적용

### Step 3: 중복 체크 — Notion DB 조회

- **역할**: 검색된 각 파일의 Google Drive 파일 ID가 이미 Notion DB에 존재하는지 확인
- **방법**: Notion API로 DB를 쿼리하여 파일 URL에 해당 파일 ID가 포함된 레코드가 있는지 필터링
- **결과**: 신규 파일만 다음 단계로 전달, 기존 파일은 스킵

### Step 4: PDF 다운로드 — Google Drive에서 바이너리 취득

- **역할**: 신규로 판별된 PDF 파일을 Google Drive에서 다운로드
- **출력**: 바이너리 데이터 (다음 단계의 Gemini 분석에 전달)

### Step 5: Gemini 분석 — 논문 요약 생성

- **역할**: 다운로드된 PDF를 Google Gemini에 전달하여 논문 제목 추출 및 요약 생성
- **노드 설정**:
  - Resource: `document`
  - Operation: `analyze`
  - Model: `gemini-2.5-flash`
- **프롬프트 예시**:
  ```
  이 논문 PDF를 분석하여 다음 정보를 추출해주세요:
  1. 논문 제목 (원문 그대로)
  2. 요약 (한국어, 300자 이내, 핵심 내용 중심)

  JSON 형식으로 응답:
  {
    "title": "논문 제목",
    "summary": "요약 내용"
  }
  ```

### Step 6: Notion DB 저장 — 결과 기록

- **역할**: Gemini 분석 결과를 Notion DB에 새 레코드로 저장
- **매핑**:
  - 1열 (파일): Google Drive 공유 링크 URL
  - 2열 (논문 제목): Gemini가 추출한 제목
  - 3열 (요약 내용): Gemini가 생성한 요약

---

## 4. 노드 구성표

| 순서 | 노드 이름 | 노드 타입 | 역할 |
|------|-----------|-----------|------|
| 1 | Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | 5분 간격 자동 실행 |
| 2 | Google Drive Search | `n8n-nodes-base.googleDrive` | 98_DB 하위 PDF 파일 검색 |
| 3 | Notion DB 조회 | `n8n-nodes-base.notion` | 중복 파일 ID 체크 |
| 4 | IF (중복 필터) | `n8n-nodes-base.if` | 신규 파일만 통과 |
| 5 | Google Drive Download | `n8n-nodes-base.googleDrive` | PDF 바이너리 다운로드 |
| 6 | Google Gemini | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | 논문 분석 및 요약 |
| 7 | Notion DB 생성 | `n8n-nodes-base.notion` | 결과를 Notion DB에 저장 |

---

## 5. Notion DB 스키마

| 열 번호 | 열 이름 | 속성 타입 | 내용 | 비고 |
|---------|---------|-----------|------|------|
| 1 | 파일 | Files & Media (URL) | Google Drive 공유 링크 | Notion API 제한으로 직접 업로드 불가, URL 방식 사용 |
| 2 | 논문 제목 | Title | AI가 추출한 원문 제목 | DB의 기본 Title 속성 활용 |
| 3 | 요약 내용 | Rich Text | AI가 생성한 한국어 요약 | 300자 이내 권장 |

### Notion 파일 저장 관련 참고사항

Notion API는 외부 파일의 직접 업로드가 제한적이다. 따라서 Google Drive의 공유 링크(URL)를 Files & Media 속성에 저장하는 방식을 사용한다.

- Google Drive 파일의 공유 설정이 "링크가 있는 모든 사용자" 또는 적절한 권한으로 설정되어야 열람 가능
- URL 형식: `https://drive.google.com/file/d/<FILE_ID>/view`

---

## 6. 주의사항 및 고려사항

### 인증 및 자격증명
- **Google Drive**: OAuth2 자격증명 필요 (Drive API 활성화)
- **Google Gemini**: API Key 자격증명 필요
- **Notion**: Internal Integration Token 필요 (대상 DB에 연결 필수)

### Google Drive Trigger 사용 불가 사유
- 공식 문서 명시: `"Changes within subfolders won't trigger this node"`
- 98_DB 하위 폴더 구조에서는 파일 생성을 감지할 수 없어 Polling 방식 필수

### API 호출량 관리
- 5분 간격 Polling 시 하루 약 288회 Google Drive API 호출 발생
- Google Drive API 일일 할당량(10억 쿼리) 대비 충분히 여유
- Notion API는 초당 3요청 제한이 있으므로, 대량 파일 동시 처리 시 Rate Limit 주의

### 에러 처리 권장사항
- Gemini 분석 실패 시 해당 파일을 건너뛰고 다음 실행에서 재시도되도록 설계
- Notion 저장 실패 시 Error Workflow로 알림 전송 고려
- Google Drive 권한 오류 시 자격증명 갱신 필요

### 확장 가능성
- Notion DB 열 추가: 저자, 발행연도, 키워드, 분류 태그 등
- 요약 언어 선택: 프롬프트 수정으로 영어/한국어 전환 가능
- 파일 형식 확장: PDF 외 DOCX 등 추가 지원 가능
