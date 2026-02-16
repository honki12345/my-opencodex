# Codex SSH 원격 운영 구현 계획서 (범용 Job 기반)

- 작성일: 2026-02-15
- 개정일: 2026-02-16
- 기준 문서: `codex-ssh-plan.md`
- 목표: Mac mini를 SSH로 제어해 프로젝트별 Job을 수동/자동으로 안정 운영하고, API 업로드를 포함한 다양한 후처리 작업까지 확장 가능하게 구축
- 대상 서버: Mac mini (macOS, Apple Silicon 기준)

---

## 1. 범위와 완료 기준

### 1.1 범위

1. 서버/클라이언트 원격 접속 기반 구축
2. 공통 Job 실행기(`codex-run-job.sh`) 구현
3. Job 정의 파일 카탈로그 작성
4. 대화형 운영(`ssh + tmux + codex`) 검증
5. 자동 운영(`launchd` 또는 `cron`) 검증
6. 샘플 Job 4종 이상 검증 (예: build/test/backup/api-upload)

### 1.2 최종 완료 기준 (Definition of Done)

1. 외부 PC + 모바일에서 Tailscale 경유 SSH 접속 성공
2. 대화형 Codex 세션 생성/재접속 성공
3. 공통 실행기로 서로 다른 Job 3종 이상 단발 실행 성공
4. 스케줄러로 Job 1종 이상 주기 실행 성공
5. API 업로드 Job 1종 성공(선택 Job) + 다른 Job과 충돌 없음
6. 실패 시나리오(프롬프트 누락, 잠금 충돌, 인증 오류, 타임아웃) 대응 절차 문서화

---

## 2. 표준 운영 구조

### 2.1 디렉토리

```text
/Users/codex/
  ops/
    scripts/
      codex-run-job.sh
    jobs/
      app-a-build.env
      app-a-test.env
      weekly-backup.env
      api-upload-posts.env
    logs/
    runs/
    state/
  projects/
    app-a/
    app-b/
```

### 2.2 Job 정의 포맷 (env 예시)

파일: `/Users/codex/ops/jobs/app-a-build.env`

```bash
JOB_ID="app-a-build"
PROJECT_DIR="/Users/codex/projects/app-a"
PROMPT_FILE="/Users/codex/ops/prompts/app-a-build.txt"
TIMEOUT="15m"
LOCK_FILE="/Users/codex/ops/state/app-a-build.lock"
OUTPUT_DIR="/Users/codex/ops/runs"
LOG_FILE="/Users/codex/ops/logs/app-a-build.log"
POST_HOOK=""
```

파일: `/Users/codex/ops/jobs/api-upload-posts.env`

```bash
JOB_ID="api-upload-posts"
PROJECT_DIR="/Users/codex/projects/content"
PROMPT_FILE="/Users/codex/ops/prompts/api-upload-posts.txt"
TIMEOUT="20m"
LOCK_FILE="/Users/codex/ops/state/api-upload-posts.lock"
OUTPUT_DIR="/Users/codex/ops/runs"
LOG_FILE="/Users/codex/ops/logs/api-upload-posts.log"
POST_HOOK="/Users/codex/ops/scripts/upload-post.sh"
```

---

## 3. 단계별 구현 계획

## 3.1 Phase A: 서버 기본 세팅

### 작업

1. 운영 계정 확정(`codex` 권장)
2. 기본 디렉토리 생성
3. 필수 도구 설치(`node`, `tmux`, `jq`, `git`, `flock`, `coreutils`)
4. SSH(Remote Login) 활성화

### 실행 예시

```bash
mkdir -p /Users/codex/ops/{scripts,jobs,prompts,logs,runs,state}
mkdir -p /Users/codex/projects
brew install node tmux jq git flock coreutils
sudo systemsetup -setremotelogin on
```

### 합격 기준

1. 필수 디렉토리/바이너리 확인 완료
2. Remote Login이 `On`

---

## 3.2 Phase B: SSH/Tailscale 보안 구성

### 작업

1. `sshd_config` 하드닝
2. Tailscale 조인 및 ACL/기기 승인 정책 적용

### 권장 SSH 설정

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers codex
```

### 검증

1. `sudo /usr/sbin/sshd -t`
2. `ssh -o PreferredAuthentications=publickey codex@localhost 'whoami'`
3. `tailscale status` / `tailscale ping <host>`

### 합격 기준

1. 공개키 로그인만 허용
2. 외부 PC/모바일에서 Tailnet 경유 접속 가능

---

## 3.3 Phase C: Codex 설치/인증

### 작업

1. Codex CLI 설치
2. 1회 로그인
3. 기본 smoke 실행

### 실행 예시

```bash
npm i -g @openai/codex
codex --version
codex
codex exec --full-auto -C /Users/codex/projects "한 단어로 답해: OK"
```

### 합격 기준

1. 서버에서 `codex` 실행 가능
2. `codex exec` 1회 성공

---

## 3.4 Phase D: 공통 Job 실행기 구현

### 작업

1. `/Users/codex/ops/scripts/codex-run-job.sh` 작성
2. 인자: `job_id` (필수), `override_prompt` (선택)
3. Job env 로드 -> 검증 -> 잠금 -> timeout -> `codex exec` 실행
4. 결과 파일 경로를 stdout으로 출력

### 필수 로직

1. Job 파일 존재 확인 (`/Users/codex/ops/jobs/<job_id>.env`)
2. `PROJECT_DIR`, `PROMPT_FILE`, `LOG_FILE`, `LOCK_FILE` 유효성 확인
3. `flock -n`으로 중복 실행 차단
4. `timeout/gtimeout`으로 실행 상한 강제
5. `codex exec --full-auto -C "$PROJECT_DIR" - -o "$OUT_FILE"`
6. `POST_HOOK`가 있으면 성공 시 실행

### 검증

1. 정상 Job 실행
2. Job 파일 누락/프롬프트 누락 실패 테스트
3. 동시 실행 충돌 테스트(1건만 성공)

### 합격 기준

1. Job 단위 실행/로그/출력 분리 동작
2. 실패 시 종료코드 + 로그 남김

---

## 3.5 Phase E: Job 카탈로그 초안 작성

최소 아래 4개 Job 작성:

1. `app-a-build`
2. `app-a-test`
3. `weekly-backup`
4. `api-upload-posts`

규칙:
1. 각 Job은 별도 `.env` 파일로 정의
2. 프롬프트/로그/출력 파일명은 `job_id` 기반으로 분리
3. API 업로드는 다른 Job과 동일한 표준 실행기로 처리

### 합격 기준

1. 4개 Job 정의 파일 생성 완료
2. 4개 중 3개 이상 단발 실행 성공

---

## 3.6 Phase F: 스케줄링 구성 (launchd 우선, cron 보조)

### 1안: launchd (권장)

1. Job별 plist 생성
2. `ProgramArguments`에 `codex-run-job.sh <job_id>` 지정
3. `StartInterval` 또는 `StartCalendarInterval` 사용

### 2안: cron (단순 주기)

```cron
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
*/30 * * * * /Users/codex/ops/scripts/codex-run-job.sh app-a-build
```

### 검증

1. 스케줄러가 실제로 Job을 트리거하는지 확인
2. 결과 파일(`runs`)과 로그(`logs`) 갱신 확인

### 합격 기준

1. 스케줄 Job 1건 이상 자동 실행 성공
2. 중복 실행/장기 실행 제어 정상 동작

---

## 3.7 Phase G: 대화형 운영 경로 검증

### 실행 예시

```bash
ssh -t codex@<tailscale-host> 'cd /Users/codex/projects/app-a && tmux new-session -A -s codex-app-a "codex"'
```

### 검증

1. 승인/거부 인터랙션 정상
2. 접속 종료 후 재접속 시 세션 복원

### 합격 기준

1. 모바일/외부 PC 모두 대화형 운영 가능

---

## 3.8 Phase H: 통합 스모크 테스트

### 시나리오

1. 외부 PC SSH 접속 -> `app-a-build` 실행
2. 모바일 SSH 접속 -> 대화형 세션 재접속
3. 스케줄러 자동 실행 -> `weekly-backup` 결과 생성 확인
4. `api-upload-posts` 실행 -> API 성공 코드 확인

### 합격 기준

1. 수동/자동/대화형/후처리(API) 4경로 모두 성공
2. 실패 로그와 복구 절차가 문서화

---

## 4. 테스트 매트릭스

| ID | 시나리오 | 검증 방법 | 기대 결과 |
|---|---|---|---|
| IT-01 | 외부 PC SSH 접속 | `ssh codex@<host> 'whoami'` | `codex` 출력 |
| IT-02 | 모바일 SSH 접속 | 모바일 SSH 앱에서 `whoami` | `codex` 출력 |
| IT-03 | Job 단발 실행 | `codex-run-job.sh app-a-build` | 결과 파일 생성 |
| IT-04 | Job 실패 처리 | 누락 프롬프트로 실행 | 실패코드 + 로그 |
| IT-05 | 중복 실행 차단 | 동일 Job 동시 2회 실행 | 1회만 실행 |
| IT-06 | 스케줄 자동 실행 | launchd/cron 실행 후 로그 확인 | 새 결과 파일 생성 |
| IT-07 | 대화형 세션 복원 | `tmux` 재접속 | 기존 세션 유지 |
| IT-08 | API 업로드 Job | `codex-run-job.sh api-upload-posts` | API 성공 코드 |

---

## 5. 운영 Runbook

### 5.1 일일 점검

1. `tail -n 100 /Users/codex/ops/logs/*.log`
2. `ls -lt /Users/codex/ops/runs | head`
3. 실패 Job 재실행 및 원인 분류

### 5.2 주간 점검

1. Job 성공률/실패율 집계
2. 디스크 사용량(`logs`, `runs`) 점검
3. 필요 없는 결과물 정리 정책 점검

### 5.3 월간 점검

1. SSH 키/접근 계정 점검
2. Tailscale ACL 점검
3. Codex/Tailscale 정책/요금 한도 재확인

---

## 6. 리스크와 대응

1. 인증 만료 -> 주 1회 수동 smoke 실행
2. 크론/launchd 환경 차이 -> 절대경로 + PATH 고정
3. 중복 실행 -> `flock` 잠금 강제
4. 장시간 점유 -> `timeout` + 필요 시 `nice`
5. 절전으로 스케줄 누락 -> macOS 전원 정책 점검
6. 특정 API 스키마 변경 -> API 업로드 Job만 분리 수정

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
9. [ ] IT-01 ~ IT-08 통과
10. [ ] 운영 점검 루틴 담당자 지정
