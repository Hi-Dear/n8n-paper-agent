# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

n8n 워크플로우를 설계하고 MCP를 통해 n8n 인스턴스에 직접 배포하는 프로젝트.
워크플로우 로직은 `SOP.md`에 SOP 문서로 정의되어 있다.

## 필수 작업 순서

1. **`SOP.md`를 반드시 먼저 읽는다** — 워크플로우의 전체 흐름, 노드 구성, 트리거 전략, DB 스키마를 이해한 후 작업에 착수
2. **n8n MCP 도구로 노드 조사** — `search_nodes`, `get_node`로 실제 노드 스펙(파라미터, typeVersion 등)을 확인
3. **워크플로우 생성/수정** — `n8n_create_workflow` 또는 `n8n_update_*` 도구로 n8n에 직접 배포
4. **검증** — `n8n_validate_workflow`로 구조 검증 후 `n8n_autofix_workflow`로 자동 수정

## n8n MCP 연동

- n8n 인스턴스: `http://localhost:5678`
- MCP 서버: `n8n-mcp` (`.mcp.json`에 설정됨)
- 주요 MCP 도구:
  - `search_nodes` / `get_node` — 노드 검색 및 상세 스펙 조회
  - `validate_node` — 개별 노드 설정 검증
  - `n8n_create_workflow` / `n8n_update_full_workflow` / `n8n_update_partial_workflow` — 워크플로우 CRUD
  - `n8n_validate_workflow` / `n8n_autofix_workflow` — 워크플로우 검증 및 자동 수정
  - `search_templates` / `get_template` / `n8n_deploy_template` — 템플릿 활용

## n8n Skills 활용

노드 설정, 표현식 작성, 코드 노드 구현 시 관련 Skill을 적극 활용한다:

- `n8n-node-configuration` — 노드별 파라미터 설정 가이드
- `n8n-expression-syntax` — `{{ }}` 표현식 문법 및 `$json`, `$node` 변수
- `n8n-code-javascript` / `n8n-code-python` — Code 노드 작성
- `n8n-workflow-patterns` — 검증된 워크플로우 아키텍처 패턴
- `n8n-validation-expert` — 검증 오류 해석 및 수정
- `n8n-mcp-tools-expert` — MCP 도구 사용 가이드

## 워크플로우 제작 규칙

- 노드의 `typeVersion`은 반드시 `get_node`로 확인한 최신 버전 사용
- 표현식은 `={{ }}` 형식 (등호 prefix 필수)
- 워크플로우 생성 후 `n8n_validate_workflow` → `n8n_autofix_workflow` 순으로 검증
- 노드 간 연결(connections)의 key는 노드의 `name` 필드 (id 아님)
- 자격증명(credentials)은 노드에 포함하지 않음 — 사용자가 n8n UI에서 직접 설정
