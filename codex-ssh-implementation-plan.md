# Codex SSH 원격 운영 구현 계획서

- 작성일: 2026-02-15
- 기준 문서: `codex-ssh-plan.md`
- 목표: 홈서버에서 포트포워딩 없이, 외부 PC/모바일에서 Codex를 안전하게 운영하고 자동 배치까지 검증 완료
- 대상 서버: Mac mini (macOS, Apple Silicon 기준)

---

## 1. 범위와 완료 기준

### 1.1 범위

1. 서버 세팅
2. 클라이언트(외부 PC, 모바일) 세팅
3. 대화형 운영(`ssh + tmux + codex`) 검증
4. 배치 운영(`cron` 또는 `launchd` + `codex-run.sh`) 검증
5. API 퍼블리셔(`/api/posts`) 연동 스모크 테스트
6. 보안/운영 점검 항목 수립

### 1.2 최종 완료 기준 (Definition of Done)

1. 외부 PC와 모바일에서 Tailscale 경유 SSH 접속 성공
2. 대화형 세션에서 `codex` 실행 및 세션 재접속 성공
3. 배치 스크립트 단발 실행 성공 + 출력 파일 생성 확인
4. 스케줄러(`cron` 또는 `launchd`) 1회 이상 정상 실행 로그 확인
5. cron 실행 결과를 API(`/api/posts`)로 1회 이상 업로드 성공
6. 실패 시나리오 3종 이상(프롬프트 누락, 잠금 충돌, 인증 오류) 대응 절차 문서화

---

## 2. 사전 준비

### 2.1 서버 전제

1. Mac mini 1대 (권장: macOS 14+)
2. 관리자 권한 계정 1개
3. Node.js LTS 설치 가능
4. Git 저장소로 사용할 작업 디렉토리 준비 가능 (`/Users/codex/blog`)

### 2.2 클라이언트 전제

1. 외부 PC 1대 (macOS/Windows/Linux)
2. 모바일 1대 (iOS/Android)
3. Tailscale 앱 설치 가능
4. SSH 클라이언트 사용 가능

---

## 3. 단계별 구현 계획

## 3.1 Phase A: 서버 기본 계정/디렉토리/패키지

### 작업

1. 운영 계정 확정 (`codex` 전용 macOS 사용자 또는 기존 사용자)
2. 작업 디렉토리 구성
3. Homebrew 및 필수 도구 설치 (`node`, `tmux`, `jq`, `git`, `flock`, `coreutils`)
4. SSH(Remote Login) 활성화

### 실행 예시

```bash
mkdir -p /Users/codex/blog/{scripts,prompts,logs,runs}
sudo xcode-select --install || true
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/opt/homebrew/bin/brew shellenv)"
brew install node tmux jq git flock coreutils
sudo systemsetup -setremotelogin on
```

### 검증

1. 사용자 확인: `id codex` 또는 운영 계정 확인
2. 디렉토리 권한 확인: `ls -ld /Users/codex/blog /Users/codex/blog/scripts`
3. 도구 확인: `ssh -V`, `tmux -V`, `git --version`, `flock -h`, `gtimeout --version`
4. Remote Login 상태 확인: `sudo systemsetup -getremotelogin`

### 합격 기준

1. `codex` 계정 존재
2. `/Users/codex/blog` 하위 디렉토리 생성
3. 필수 바이너리 실행 가능
4. Remote Login이 `On` 상태

---

## 3.2 Phase B: SSH 보안 설정

### 작업

1. `sshd_config` 하드닝 적용
2. 설정 문법 검사
3. 설정 반영(Remote Login 재시작)

### 권장 설정

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers codex
```

### 검증

1. 문법 검사: `sudo /usr/sbin/sshd -t`
2. 반영값 점검: `sudo /usr/sbin/sshd -T | rg 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin|allowusers'`
3. 로컬 루프백 로그인 테스트: `ssh -o PreferredAuthentications=publickey codex@localhost 'whoami'`
4. 반영 적용: `sudo systemsetup -setremotelogin off && sudo systemsetup -setremotelogin on`

### 합격 기준

1. `sshd -t` 오류 없음
2. 비밀번호/keyboard-interactive 인증 비활성화 확인
3. 공개키 로그인만 허용됨

---

## 3.3 Phase C: Codex 설치 및 인증

### 작업

1. Node.js LTS 설치
2. Codex CLI 설치
3. 1회 로그인

### 실행 예시

```bash
sudo npm i -g @openai/codex
codex --version
codex
```

### 검증

1. 버전 확인: `codex --version`
2. 간단 실행 테스트:

```bash
cd /Users/codex/blog
git init
codex exec --full-auto -C /Users/codex/blog "한 단어로 답해: OK"
```

### 합격 기준

1. `codex` 명령이 서버에서 정상 실행
2. `codex exec` 1회 성공

---

## 3.4 Phase D: Tailscale 서버 조인 및 접근 제어

### 작업

1. 서버(Mac mini)에 Tailscale 설치/로그인
2. 서버/클라이언트를 동일 Tailnet에 조인
3. ACL(또는 기기 승인) 정책 적용

### 검증

1. 서버 상태: `tailscale status`
2. 외부 PC에서 핑: `tailscale ping <tailscale-host>`
3. 모바일에서 서버 식별 가능 여부 확인

### 합격 기준

1. 서버가 Tailnet에서 `online`
2. 외부 PC/모바일 모두 서버 주소 확인 가능

---

## 3.5 Phase E: 배치 스크립트 배포

### 작업

1. `/Users/codex/blog/scripts/codex-run.sh` 배치
2. 실행 권한 부여
3. 사전 검증 로직 포함

### 필수 로직

1. 프롬프트 파일 존재 확인
2. `codex` 바이너리 존재 확인
3. `WORKDIR` Git repo 여부 확인
4. `flock` 중복 실행 방지
5. `timeout` 또는 `gtimeout` 장시간 실행 방지
6. `codex exec --full-auto -C /Users/codex/blog` 고정
7. macOS `PATH`(`/opt/homebrew/bin` 포함) 고정

### 검증

1. 정상 실행:

```bash
echo "한 문장으로 상태 출력" > /Users/codex/blog/prompts/smoke.txt
/Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/smoke.txt
```

2. 출력 확인: `ls -lt /Users/codex/blog/runs | head`
3. 로그 확인: `tail -n 50 /Users/codex/blog/logs/codex-run.log`
4. 누락 파일 실패 테스트:

```bash
/Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/not-exists.txt ; echo $?
```

### 합격 기준

1. 정상 요청에서 결과 파일 생성
2. 실패 요청에서 비정상 종료 코드 + 에러 로그 생성

---

## 3.6 Phase F: 스케줄러 등록 및 자동 실행 검증 (기본: cron)

### 작업

1. `codex` 사용자 crontab 등록
2. 실행 주기 설정 (예: 30분)

### 실행 예시

```bash
crontab -e
```

```cron
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
*/30 * * * * /Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/latest.txt
```

### 검증

1. 스케줄 등록 확인: `crontab -l`
2. 1회 실행 로그 확인:
   - `tail -n 100 /Users/codex/blog/logs/codex-run.log`
   - `ls -lt /Users/codex/blog/runs | head`
3. 잠금 충돌 테스트:
   - 동일 스크립트를 2회 거의 동시에 실행
   - 1개만 성공하는지 확인

### 합격 기준

1. 크론 트리거로 결과 파일 생성 확인
2. 중복 실행 차단 확인

---

## 3.7 Phase G: 클라이언트 세팅 (외부 PC + 모바일)

### 작업

1. 외부 PC/모바일에 Tailscale 설치 및 로그인
2. 외부 PC에 SSH 키 생성 및 서버 `authorized_keys` 등록
3. 모바일 SSH 앱(Termius/Prompt 등)에 동일 키 또는 전용 키 등록

### 검증

1. 외부 PC SSH 접속:

```bash
ssh codex@<tailscale-host> 'whoami && hostname'
```

2. 모바일 SSH 접속:
   - `whoami` 결과가 `codex`인지 확인
3. 네트워크 전환(와이파이 <-> LTE) 후 재접속 가능 확인

### 합격 기준

1. 외부 PC/모바일 모두 SSH 접속 성공
2. 인증 실패 없이 일관된 키 기반 로그인 가능

---

## 3.8 Phase H: 대화형 운영 검증

### 작업

1. `tmux` 기반 대화형 접속 경로 고정

### 실행 예시

```bash
ssh -t codex@<tailscale-host> 'cd /Users/codex/blog && tmux new-session -A -s codex "codex"'
```

### 검증

1. Codex 프롬프트 입력/응답 확인
2. 접속 종료 후 재접속 시 기존 세션 복원 확인
3. 모바일에서 승인/거부 입력 가능 확인

### 합격 기준

1. 세션 유실 없이 대화형 운영 가능
2. 사용자 승인 흐름이 정상 동작

---

## 3.9 Phase I: E2E 스모크 테스트 (cron -> codex -> API)

### 작업

1. 스케줄러 입력 프롬프트(`latest.txt`) 준비
2. 임시 1분 주기 cron으로 실제 트리거 발생 확인
3. 최신 결과 파일을 `/api/posts`로 업로드하는 스모크 호출 수행
4. API 응답/상태코드 확인

### 실행 예시

```bash
cat > /Users/codex/blog/prompts/latest.txt <<'EOF'
블로그 초안 1개를 한국어로 작성해줘. 제목/요약/본문 3섹션으로 출력.
EOF
```

```bash
crontab -l > /tmp/codex-crontab.bak 2>/dev/null || true
{
  cat /tmp/codex-crontab.bak
  echo 'PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin'
  echo '*/1 * * * * /Users/codex/blog/scripts/codex-run.sh /Users/codex/blog/prompts/latest.txt'
} | crontab -
```

```bash
BEFORE_FILE="$(ls -t /Users/codex/blog/runs/*.md 2>/dev/null | head -1 || true)"
sleep 70
AFTER_FILE="$(ls -t /Users/codex/blog/runs/*.md 2>/dev/null | head -1 || true)"
[ "$BEFORE_FILE" != "$AFTER_FILE" ] || { echo "cron did not generate new file"; exit 1; }
LATEST_FILE="$AFTER_FILE"
API_BASE_URL="http://127.0.0.1:3000"
STATUS_CODE="$(
  jq -n \
    --arg title "codex cron smoke $(date '+%F %T')" \
    --arg content "$(cat "$LATEST_FILE")" \
    '{title:$title, content:$content}' \
  | curl -sS -o /tmp/codex-api-smoke.json -w "%{http_code}" \
      -H "Content-Type: application/json" \
      -X POST "$API_BASE_URL/api/posts" \
      -d @-
)"
echo "status=$STATUS_CODE"
cat /tmp/codex-api-smoke.json
```

```bash
crontab /tmp/codex-crontab.bak
rm -f /tmp/codex-crontab.bak /tmp/codex-api-smoke.json
```

### 검증

1. 결과 파일 생성 확인: `ls -lt /Users/codex/blog/runs | head`
2. cron 트리거로 새 파일이 생성되었는지 확인(`BEFORE_FILE != AFTER_FILE`)
3. API HTTP 상태코드 확인: `200` 또는 `201`
4. API 응답 JSON에 식별자(`id` 또는 `slug`) 포함 확인
5. 필요 시 GET 조회로 저장 확인(프로젝트 API 스펙에 맞는 조회 엔드포인트 사용)

### 합격 기준

1. 실제 cron 트리거로 Codex 결과 파일 생성 경로가 1회 이상 성공
2. 생성 결과를 API에 POST했을 때 성공 코드(200/201) 반환
3. 업로드 데이터가 서버에 실제 반영됨

---

## 4. 통합 테스트 매트릭스

| ID | 시나리오 | 실행 주체 | 검증 명령/방법 | 기대 결과 |
|---|---|---|---|---|
| IT-01 | 외부 PC 원격 접속 | 외부 PC | `ssh codex@<host> 'whoami'` | `codex` 출력 |
| IT-02 | 모바일 원격 접속 | 모바일 | SSH 앱 접속 후 `whoami` | `codex` 출력 |
| IT-03 | 배치 단발 실행 | 서버 | `codex-run.sh smoke.txt` | `runs/*.md` 생성 |
| IT-04 | 스케줄러 자동 실행 | 서버 | 주기 후 로그 확인 | 새 결과 파일 1개 이상 |
| IT-05 | 프롬프트 누락 | 서버 | 존재하지 않는 파일로 실행 | 실패 코드 + 에러 로그 |
| IT-06 | 중복 실행 충돌 | 서버 | 스크립트 동시 2회 실행 | 1회만 실행 |
| IT-07 | 대화형 세션 복원 | 모바일/외부 PC | `tmux` 재접속 | 이전 세션 유지 |
| IT-08 | E2E 업로드 스모크 | 서버 | cron 실행 후 `/api/posts` POST | API 성공 + 저장 확인 |

---

## 5. 운영 절차 (Runbook)

### 5.1 일일 점검

1. 최근 실행 로그 확인: `tail -n 100 /Users/codex/blog/logs/codex-run.log`
2. 최신 결과 파일 타임스탬프 확인: `ls -lt /Users/codex/blog/runs | head`
3. 실패 건 있으면 즉시 원인 분류:
   - 인증/로그인
   - 네트워크(Tailscale)
   - 입력 데이터(프롬프트)
   - Codex 실행 정책/한도

### 5.2 주간 점검

1. 크론 정상 동작률 점검
2. 디스크 사용량 점검 (`runs`, `logs`)
3. `/api/posts` 업로드 성공률/실패 로그 점검
4. 실패 재발 방지 조치 반영

### 5.3 월간 점검

1. SSH 키/접근 계정 점검
2. Tailscale ACL/기기 승인 상태 점검
3. 서비스 정책/한도(요금 포함) 최신화 확인

---

## 6. 리스크와 대응

1. 인증 만료로 무인 실행 실패
   - 대응: 수동 `codex exec` 스모크 테스트를 주간 1회 수행
2. 크론 환경 차이로 바이너리 미탐지
   - 대응: `PATH` 명시 + 절대 경로 사용
3. 중복 실행으로 결과 충돌
   - 대응: `flock` 잠금 파일 유지, 실패 로그 모니터링
4. 모바일 네트워크 변경으로 대화형 세션 단절
   - 대응: `tmux` 세션 고정, 재접속 절차 문서화
5. 사용량/요금 정책 변경
   - 대응: 배포 전/월간 정책 재확인 항목 유지
6. Mac mini 절전 모드로 스케줄 작업 누락
   - 대응: 시스템 설정에서 자동 잠자기/디스크 잠자기 정책 점검
7. API 스키마 변경으로 퍼블리셔 실패
   - 대응: 배포 전 스모크 테스트(IT-08) 필수화 및 실패 시 즉시 롤백

---

## 7. 실행 순서 요약 (체크박스)

1. [ ] Phase A 완료
2. [ ] Phase B 완료
3. [ ] Phase C 완료
4. [ ] Phase D 완료
5. [ ] Phase E 완료
6. [ ] Phase F 완료
7. [ ] Phase G 완료
8. [ ] Phase H 완료
9. [ ] Phase I 완료
10. [ ] 통합 테스트 IT-01 ~ IT-08 통과
11. [ ] 운영 점검 루틴(일/주/월) 담당자 지정
