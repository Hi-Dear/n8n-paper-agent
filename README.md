# n8n Paper Agent

Google Drive에 저장된 논문 PDF를 자동 감지하여 AI(Google Gemini)로 분석/요약하고, Notion DB에 정리하는 n8n 자동화 워크플로우입니다.

## 워크플로우 흐름

```
[Schedule Trigger] → [Google Drive 검색] → [중복 체크 (Notion 조회)]
    → [PDF 다운로드] → [텍스트 추출] → [Gemini 분석] → [Notion DB 저장]
```

### Step 1. 트리거 & 파일 탐색
- Schedule Trigger로 주기적 실행 (Polling 방식)
- Google Drive 지정 폴더의 하위 폴더까지 재귀적으로 PDF 파일 검색

### Step 2. 중복 체크
- Notion DB에서 Google Drive 파일 ID로 기존 처리 여부 확인
- 신규 파일만 다음 단계로 전달

### Step 3. AI 분석
- PDF에서 텍스트 추출 후 Google Gemini에 전달
- 논문 제목, 기존 문제, 해결 목표, 차별성, 연구 내용 요약, 결과, 적용 분야를 구조화된 형식으로 출력

### Step 4. Notion 저장
- 분석 결과를 Notion DB에 페이지로 생성
- Markdown → Notion Block 변환 후 본문 업로드

## 사용 서비스

| 서비스 | 용도 |
|--------|------|
| **n8n** | 워크플로우 자동화 엔진 |
| **Google Drive** | 논문 PDF 저장소 |
| **Google Gemini** | 논문 분석 및 요약 (LLM) |
| **Notion** | 분석 결과 저장 DB |

## 노드 구성

| 순서 | 노드 | 역할 |
|------|------|------|
| 1 | Schedule Trigger | 주기적 자동 실행 |
| 2 | Google Drive Search | 하위 폴더 PDF 검색 |
| 3 | Notion DB Query | 중복 파일 ID 체크 |
| 4 | IF (중복 필터) | 신규 파일만 통과 |
| 5 | Google Drive Download | PDF 바이너리 다운로드 |
| 6 | Extract from File | PDF 텍스트 추출 |
| 7 | Basic LLM Chain + Gemini | 논문 분석 및 구조화 요약 |
| 8 | HTTP Request (Notion API) | 제목 및 요약 본문 저장 |

## 설정 방법

### 1. n8n 인스턴스 준비
- [n8n 설치 가이드](https://docs.n8n.io/hosting/) 참고

### 2. 자격증명 설정 (n8n UI에서 직접 설정)

| 자격증명 | 설정 내용 |
|----------|-----------|
| **Google Drive OAuth2** | Google Cloud Console에서 OAuth2 Client ID 발급 → Drive API 활성화 |
| **Google Gemini API** | Google AI Studio에서 API Key 발급 |
| **Notion** | Notion Integration 생성 → 대상 DB에 연결 |

### 3. 워크플로우 가져오기
1. n8n UI에서 **Import from File** 선택
2. `Zotero PDF Fetcher.json` 파일 업로드
3. 각 노드에 자격증명 연결
4. Google Drive 폴더 ID, Notion DB ID를 본인 환경에 맞게 수정

## 파일 구조

```
├── CLAUDE.md                    # Claude Code 프로젝트 가이드
├── SOP.md                       # 워크플로우 설계 SOP 문서
├── Zotero PDF Fetcher.json      # n8n 워크플로우 JSON
└── README.md
```

## 주의사항

- Google Drive Trigger는 하위 폴더 변경을 감지할 수 없어 Polling(Schedule Trigger) 방식 사용
- Notion API는 초당 3요청 제한 — 대량 파일 동시 처리 시 Rate Limit 주의
- 자격증명(API Key 등)은 워크플로우 JSON에 포함되지 않으므로 n8n UI에서 직접 설정 필요