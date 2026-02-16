# Codex SSH 원격 운영 계획

- 작성일: 2026-02-15
- 목표: 홈서버에서 포트포워딩 없이, 외부 PC/모바일에서 Codex를 안전하게 원격 제어
- 범위: `codex exec` 배치 실행 + 모바일 SSH 대화형 사용 + 운영/보안 기준
- 대상 서버: Mac mini (macOS)

---

## 1. 요구사항 정리

1. 채팅 채널(텔레그램/디스코드 등) 없이 직접 제어
2. 포트포워딩 없이 외부 접속
3. 모바일에서도 명령/승인/결과 확인
4. 추가 과금 최소화(기존 ChatGPT Pro 사용)

---

## 2. 권장 아키텍처

1. 홈서버(macOS): `codex`, `cron`(또는 `launchd`), 블로그 API 서버 실행
2. 원격 접속: `Tailscale` 사설망으로 서버 접속
3. 제어 채널: `SSH`로 서버 접속 후 `codex` 또는 래퍼 스크립트 실행
4. 작업 모드:
   - 대화형: `ssh -t ... 'codex'`
   - 배치형: `ssh ... '/Users/codex/blog/scripts/codex-run.sh <prompt-file>'`

---

## 3. 구성 선택 기준

### 3.1 VPN

1. 기본 권장: `Tailscale`
2. 대안: `WireGuard` 직접 구성

권장 이유:
1. 포트포워딩 없이 연결 가능
2. 모바일 앱 지원이 쉬움
3. 초기 세팅 시간이 짧음

### 3.2 Codex 실행 방식

1. 대화형 운영(모바일 승인 포함): `codex` 인터랙티브
2. 자동 운영(크론/배치): `codex exec`
3. 무인 배치는 실행 옵션을 명시적으로 고정(예: `--full-auto`)해 환경별 동작 차이를 줄임

주의:
1. 크론은 무인 실행이므로 실시간 승인 불가
2. 무인 실행 정책(권한/작업범위/로그)을 사전 고정해야 함

---

## 4. 상세 구현 단계

## 4.1 서버 기본 준비

1. 서버 계정 준비: macOS 로컬 사용자(`codex`) 또는 전용 운영 계정
2. 작업 디렉토리 준비: 예) `/Users/codex/blog`
3. 필수 도구 설치(Homebrew): `node`, `tmux`, `jq`, `git`, `flock`, `coreutils`
4. SSH(Remote Login) 활성화: `sudo systemsetup -setremotelogin on`

## 4.2 Codex 설치 및 로그인

```bash
npm i -g @openai/codex
codex --version
codex
```

1. 최초 1회 로그인 완료
2. 로그인 후 테스트:

```bash
codex exec "한 단어로 답해: OK"
```

## 4.3 SSH 보안 설정

`/etc/ssh/sshd_config` 권장값:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers codex
```

1. 비밀번호 로그인 비활성화
2. keyboard-interactive 인증 비활성화
3. SSH 키 인증만 허용 + 운영 계정만 접근 허용
4. 변경 후 `sudo systemsetup -setremotelogin off && sudo systemsetup -setremotelogin on`

## 4.4 Tailscale 연결

1. 서버와 모바일/외부PC를 같은 Tailnet에 조인
2. 서버 접속은 Tailscale IP/호스트명 사용
3. 필요 시 ACL로 접근 단말 제한

## 4.5 래퍼 스크립트 배치

파일: `/Users/codex/blog/scripts/codex-run.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ $# -lt 1 ]; then
  echo "usage: $0 <prompt-file>"
  exit 1
fi

PROMPT_FILE="$1"
WORKDIR="/Users/codex/blog"
OUT_DIR="$WORKDIR/runs"
LOG_DIR="$WORKDIR/logs"
STAMP="$(date +%F-%H%M%S)"
OUT_FILE="$OUT_DIR/$STAMP.md"
LOG_FILE="$LOG_DIR/codex-run.log"
LOCK_FILE="/tmp/codex-run.lock"
PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"

mkdir -p "$OUT_DIR" "$LOG_DIR"

if [ ! -f "$PROMPT_FILE" ]; then
  echo "[ERROR] prompt file not found: $PROMPT_FILE" | tee -a "$LOG_FILE"
  exit 1
fi

if ! command -v codex >/dev/null 2>&1; then
  echo "[ERROR] codex not found in PATH=$PATH" | tee -a "$LOG_FILE"
  exit 127
fi

if ! command -v flock >/dev/null 2>&1; then
  echo "[ERROR] flock not found. Install with: brew install flock" | tee -a "$LOG_FILE"
  exit 127
fi

TIMEOUT_BIN="$(command -v timeout || command -v gtimeout || true)"
if [ -z "$TIMEOUT_BIN" ]; then
  echo "[ERROR] timeout/gtimeout not found. Install with: brew install coreutils" | tee -a "$LOG_FILE"
  exit 127
fi

if ! git -C "$WORKDIR" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  echo "[ERROR] WORKDIR is not a git repo: $WORKDIR" | tee -a "$LOG_FILE"
  exit 2
fi

flock -n "$LOCK_FILE" "$TIMEOUT_BIN" 15m \
  codex exec --full-auto -C "$WORKDIR" - -o "$OUT_FILE" < "$PROMPT_FILE" >> "$LOG_FILE" 2>&1

echo "$OUT_FILE"
```

권한:

```bash
chmod +x /Users/codex/blog/scripts/codex-run.sh
```

## 4.6 크론 작업 등록

예시(30분 주기):

```bash
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
*/30 * * * * /Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/latest.txt
```

운영 포인트:
1. `flock`으로 중복 실행 방지
2. `timeout` 또는 `gtimeout`으로 장시간 정지 방지
3. 출력 파일과 로그를 분리 저장
4. 크론 환경 변수(`PATH`)를 명시하고 절대경로를 사용
5. `codex exec` 옵션(`--full-auto`, `-C`)을 스크립트에서 고정

## 4.7 모바일/외부 사용

### 대화형 Codex

```bash
ssh -t codex@<tailscale-host> 'cd /Users/codex/blog && tmux new-session -A -s codex "codex"'
```

1. 모바일 네트워크 변경 시 세션 유실 대비 `tmux` 사용
2. 대화형에서는 승인/거부를 직접 처리 가능

### 원격 단발 실행

```bash
ssh codex@<tailscale-host> '/Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/today.txt'
```

---

## 5. 운영 정책

1. 무인 배치(`cron`)와 수동 대화형(`ssh -t`)을 분리
2. 무인 배치 프롬프트는 고정 템플릿 사용
3. 결과 업로드 전 유효성 체크(길이/금칙어/필수 섹션)
4. 실패 시 재시도 1~2회, 이후 알림/로그 확인
5. 월 1회 SSH 키/접근권한 점검

---

## 6. 비용 및 제한

1. 기본 구성은 추가 과금 없이 시작 가능(각 서비스 정책/한도 내):
   - SSH/OpenSSH
   - Tailscale 무료 플랜 정책/한도 내 사용
   - ChatGPT 플랜의 Codex 사용 한도 내 실행
2. 추가 과금이 생길 수 있는 경우:
   - Tailscale 무료 플랜 한도 초과 또는 정책 변경
   - ChatGPT 플랜 한도 초과 후 API 크레딧/유료 확장 사용
3. 배포 전 최신 가격/한도 정책을 공식 페이지에서 재확인

---

## 7. 점검 체크리스트

1. `codex exec`가 서버에서 성공하는가
2. 모바일에서 Tailscale 접속 후 SSH 로그인이 되는가
3. 래퍼 스크립트가 결과 파일을 생성하는가
4. 크론이 중복 실행 없이 동작하는가
5. `flock` 잠금 충돌 시 중복 실행이 차단되는가
6. `timeout`/`gtimeout` 발생 시 프로세스가 정리되고 로그가 남는가
7. 프롬프트 파일 누락 시 즉시 실패하고 에러 로그가 남는가
8. 로그인 만료/인증 오류 발생 시 복구 절차가 문서화되어 있는가
9. 실패 로그 확인 경로가 문서화되어 있는가
10. 블로그 API 업로드 전/후 검증 절차가 있는가
11. `cron -> codex -> /api/posts` 스모크 테스트가 1회 이상 성공했는가

---

## 8. 다음 단계(실행 순서)

1. 서버에 `codex` 로그인 완료
2. `Tailscale + SSH` 원격 접속 성공 확인
3. `codex-run.sh` 배치 및 단발 테스트
4. 크론 등록 후 24시간 모니터링
5. 결과를 `/api/posts`로 업로드하는 퍼블리셔 스크립트 추가
6. `cron -> codex -> /api/posts` E2E 스모크 테스트 수행
