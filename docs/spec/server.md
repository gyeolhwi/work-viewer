---
id: spec-server
title: 서버 구조·API·파일 감시와 에이전트 요약
category: spec
summary: Bun 서버가 작업공간을 감시해 SSE로 알리고, git·herdr·Claude Code 대화 기록을 읽어 API로 내주는 방식과 감추는 파일 규칙
keywords: [server.ts, API, SSE, 파일 감시, activity.jsonl, herdr, agent-summary, Claude 대화 기록, git log, 비밀 파일, 보안]
related_files:
  - server.ts
  - agent-summary.ts
  - public/app.js
last_updated: "2026-11-12"
---
# 서버 구조·API·파일 감시와 에이전트 요약

## 파일 구성

| 경로 | 무엇 |
|---|---|
| `server.ts` | 파일 감시·API·SSE·정적 파일 서버. Bun 내장 기능만 쓴다 |
| `agent-summary.ts` | herdr 창과 Claude Code 대화 기록을 짝지어 에이전트 요약을 만든다 |
| `public/` | 화면. 빌드 없이 HTML·CSS·JS 그대로. marked·DOMPurify·highlight.js와 Pretendard 글꼴은 CDN에서 받는다 |
| `scripts/install.sh` | 서비스 등록·스킬 연결·제거 |
| `skills/work-setup/` | 작업공간 구조를 세팅하고 뷰어를 설치하는 Claude Code 스킬 |
| `skills/docs-tidy/` | docs 머리말과 `INDEX.md`를 만드는 스킬과 검사 도구 |

## API

| 경로 | 내주는 것 |
|---|---|
| `GET /api/events` | SSE. 파일 변경(`activity`)과 에이전트 목록(`agents`)을 바로 보낸다 |
| `GET /api/projects` | `Projects/` 아래 프로젝트 목록과 종류(git·site·common·folder), git 브랜치·변경 수·최근 7일 커밋 제목 |
| `GET /api/activity?since=&limit=` | 바뀐 파일 기록 |
| `GET /api/agents` | herdr 에이전트 목록과 요약 |
| `GET /api/git` | 프로젝트별 브랜치·워크트리·최근 커밋 |
| `GET /api/git/graph?p=<프로젝트>&n=` | `git log --all --topo-order` 결과 (최대 1000개) |
| `GET /api/docs`, `/api/index?path=` | 루트와 프로젝트 `INDEX.md`의 문서 목록 |
| `GET /api/search?q=` | 파일 이름 찾기 |
| `GET /api/ls?path=`, `/api/stat?path=` | 폴더 목록, 파일 정보 |
| `GET /raw?path=[&download]` | 파일 내용. `content-security-policy: sandbox`로 내준다 |

## 감추는 것

- 작업공간 밖 경로는 열지 않는다(실제 경로로 바꿔 확인).
- `.env`, `.pem`, `.key`, `id_rsa` 같은 키 파일, 이름에 `credentials`가 든 파일은 목록에만 보이고 내용은 내주지 않는다.
- `.git`, `node_modules`, `__pycache__` 등은 감시와 목록에서 뺀다.

## 바뀐 파일 기록

작업공간을 재귀로 감시해 추가·수정·삭제를 `STATE_DIR/activity.jsonl`에 쌓고 30일이 지난 것은 지운다. 새 변경은 SSE로 바로 화면에 보낸다.

## 에이전트 요약

1. 3초마다 `herdr agent list`로 에이전트 창 목록을 받는다. herdr가 없으면 빈 목록이다.
2. Claude Code 창이면 `herdr pane process-info`로 창의 `claude` pid를 찾는다.
3. `~/.claude/sessions/<pid>.json`의 sessionId로 `~/.claude/projects/<cwd>/<sessionId>.jsonl` 대화 기록을 찾는다.
4. 기록 끝 2MB(지시가 없으면 20MB까지 통째로)를 읽어, 마지막으로 사람이 보낸 지시·마지막 도구 호출(지금 하는 일)·마지막 답변·답을 기다리는 질문·할 일 진행률을 뽑는다.
