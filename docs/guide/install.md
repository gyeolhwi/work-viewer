---
id: guide-install
title: 설치·실행과 다른 기기에서 보기
category: guide
summary: bun으로 뷰어를 systemd·launchd 서비스로 등록하고, 환경 변수와 Tailscale serve·SSH 터널로 아이패드·폰에서 여는 방법
keywords: [설치, install, install.sh, systemd, launchd, Tailscale, SSH 터널, WORK_ROOT, PORT, 서비스 재시작, 제거]
related_files:
  - scripts/install.sh
  - server.ts
last_updated: "2026-11-12"
---
# 설치·실행과 다른 기기에서 보기

## 필요한 것

- [Bun](https://bun.sh) — 서버 실행. 다른 의존성은 없다.
- (선택) herdr — 실행 중인 AI 에이전트 목록. 없으면 홈의 에이전트 부분만 비고 나머지 화면은 그대로 쓴다.
- (선택) [uv](https://docs.astral.sh/uv/) — `INDEX.md` 생성 도구(`skills/docs-tidy`)를 돌릴 때.
- (선택) git — Git 탭과 요약의 커밋 제목.

## 설치

뷰어는 작업공간 폴더(기본 `~/Work`) 하나를 본다. 작업공간 안의 `Projects/<이름>/` 폴더가 프로젝트 하나다.

```sh
mkdir -p ~/Work/Projects && cd ~/Work/Projects
git clone https://github.com/gyeolhwi/work-viewer.git
~/Work/Projects/work-viewer/scripts/install.sh --root ~/Work --port 7320 --skills
```

- Linux는 systemd 사용자 서비스 `work-viewer`, macOS는 launchd(`io.github.gyeolhwi.work-viewer`)로 등록하고 바로 띄운다.
- `--skills`는 `skills/work-setup`, `skills/docs-tidy`를 `~/.claude/skills`에 심볼릭 링크로 연결한다. 같은 이름의 스킬이 이미 있으면 건드리지 않는다.
- 설치가 끝나면 `http://127.0.0.1:7320`을 연다.

Claude Code에서 **"Work 세팅해줘"**(또는 `/work-setup`)라고 하면 작업공간 구조(README·INDEX·AGENTS·docs)를 조사 → 계획 승인 → 생성 순서로 만들어 준다.

## 다른 기기에서 보기

뷰어는 `127.0.0.1`에만 연다. 홈에 에이전트 대화의 지시와 답변 일부가 보이므로, 본인만 닿는 경로로 연다.

```sh
# Tailscale (권장): 같은 tailnet의 기기에서 https://<기기이름>.<tailnet>.ts.net:8443
tailscale serve --bg --https=8443 http://127.0.0.1:7320
tailscale serve --https=8443 off          # 끄기

# WSL이면 Windows 쪽 Tailscale로
"/mnt/c/Program Files/Tailscale/tailscale.exe" serve --bg --https=8443 http://127.0.0.1:7320

# Tailscale이 없으면 SSH 터널 후 http://127.0.0.1:7320
ssh -L 7320:127.0.0.1:7320 <서버>
```

아이패드·아이폰 Safari에서 "홈 화면에 추가"를 하면 앱처럼 열린다.

## 운영

```sh
systemctl --user status work-viewer        # 상태 (macOS: launchctl print gui/$(id -u)/io.github.gyeolhwi.work-viewer)
systemctl --user restart work-viewer       # server.ts를 고친 뒤 재시작 (public/는 새로고침만)
journalctl --user -u work-viewer -f        # 로그 (macOS: ~/.local/state/work-viewer/server.log)
scripts/install.sh --uninstall             # 서비스 제거. 기록과 코드는 남긴다
```

로그아웃해도 계속 돌게 하려면 Linux에서 한 번 `sudo loginctl enable-linger $USER`.

## 직접 실행과 환경 변수

```sh
WORK_ROOT=~/Work PORT=7320 bun server.ts
```

| 변수 | 기본값 | 뜻 |
|---|---|---|
| `WORK_ROOT` | `~/Work` | 보여 줄 작업공간 폴더 |
| `PORT` | `7320` | 포트 |
| `HOST` | `127.0.0.1` | 여는 주소. 바깥에 직접 열지 않는다 |
| `STATE_DIR` | `~/.local/state/work-viewer` | 바뀐 파일 기록(`activity.jsonl`, 30일치) 위치 |
