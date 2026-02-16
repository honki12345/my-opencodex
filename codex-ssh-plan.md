# Codex SSH 원격 운영 계획 (범용 Job Orchestrator)

- 작성일: 2026-02-15
- 개정일: 2026-02-16
- 목표: Mac mini를 SSH로 원격 제어해 프로젝트별 작업을 안전하게 실행하고, 필요 시 스케줄링(cron/launchd)까지 운영
- 범위: 대화형 운영(`codex`) + 무인 배치(`codex exec`) + Job 카탈로그 기반 자동화
- 대상 서버: Mac mini (macOS)

---

## 1. 요구사항 정리

1. 채팅 채널 없이 SSH만으로 직접 제어
2. 포트포워딩 없이 외부에서 접속
3. 프로젝트별로 서로 다른 작업(Job)을 실행
4. 필요 시 크론/launchd로 주기 실행
5. 로그/결과/실패 원인을 추적 가능하게 유지

---

## 2. 핵심 설계 원칙

1. `/api/posts` 같은 특정 기능은 "여러 Job 중 하나"로만 취급
2. 실행 경로는 Job 단위로 표준화 (`job_id` 기반)
3. 대화형 운영과 무인 배치를 분리
4. 무인 배치는 권한/경로/타임아웃/중복실행 차단을 기본값으로 강제
5. 프로젝트 단위 격리(디렉토리/로그/실행 계정) 유지

---

## 3. 권장 아키텍처

1. 접속 계층: `Tailscale + SSH`
2. 실행 계층:
   - 대화형: `ssh -t ... 'codex'`
   - 배치형: `codex exec --full-auto`
3. Job 계층:
   - Job 정의 파일(`jobs/*.env` 또는 `jobs/*.toml`)
   - 공통 실행기(`codex-run-job.sh`)가 Job 정의를 읽어 실행
4. 스케줄 계층:
   - macOS 기본 권장: `launchd`
   - 단순 주기 작업: `cron` 보조
5. 저장 계층:
   - 결과물: `runs/`
   - 실행 로그: `logs/`
   - 잠금/상태: `state/`

---

## 4. Job 모델 (표준)

각 Job은 최소 아래 필드를 가진다.

1. `JOB_ID`: 작업 식별자 (예: `app-a-build`, `weekly-backup`, `api-upload-posts`)
2. `PROJECT_DIR`: 작업 대상 프로젝트 루트
3. `PROMPT_FILE`: Codex 입력 프롬프트 파일
4. `MODE`: `interactive` 또는 `batch`
5. `TIMEOUT`: 최대 실행 시간 (예: `15m`)
6. `LOCK_FILE`: 중복 실행 방지 잠금 파일
7. `SUCCESS_CHECK`: 성공 판정 규칙 (예: 출력 파일 생성, 종료코드 0)
8. `POST_HOOK` (선택): 후처리 명령 (예: API 업로드)

참고:
1. `API 업로드`는 `POST_HOOK` 또는 독립 Job(`api-upload-posts`)으로 구성한다.
2. 빌드/테스트/문서생성/백업/배포도 동일한 Job 모델로 처리한다.

---

## 5. 표준 디렉토리 구조 (권장)

```text
/Users/codex/
  ops/
    scripts/
      codex-run-job.sh
    jobs/
      app-a-build.env
      app-a-test.env
      api-upload-posts.env
    logs/
    runs/
    state/
  projects/
    app-a/
    app-b/
```

운영 규칙:
1. Job 정의는 `ops/jobs`에만 저장
2. 결과와 로그는 `job_id` 기준으로 분리 저장
3. 프로젝트 코드는 `projects/` 하위로만 접근

---

## 6. 실행 방식

### 6.1 대화형 (수동 승인)

```bash
ssh -t codex@<tailscale-host> 'cd /Users/codex/projects/app-a && tmux new-session -A -s codex-app-a "codex"'
```

1. 모바일/외부 PC에서 승인/거부를 직접 처리
2. `tmux`로 네트워크 전환 시 세션 유지

### 6.2 배치 단발 실행

```bash
ssh codex@<tailscale-host> '/Users/codex/ops/scripts/codex-run-job.sh app-a-build'
```

### 6.3 스케줄 실행

1. `launchd` 또는 `cron`에서 `codex-run-job.sh <job_id>` 호출
2. Job별 주기/타임아웃/재시도 정책을 분리 관리

---

## 7. 보안/권한 정책

1. SSH 키 인증만 허용 (`PasswordAuthentication no`)
2. 운영 계정 최소권한 원칙 적용 (`AllowUsers codex`)
3. Job 실행 허용 경로를 `/Users/codex/projects/*`로 제한
4. 민감정보는 파일 권한 `600`으로 관리하고 로그에 출력 금지
5. 무인 Job은 `--full-auto` 사용 시 허용 범위(파일/명령)를 문서화

---

## 8. 안정성/리소스 정책

1. `flock`으로 중복 실행 차단
2. `timeout`/`gtimeout`으로 장시간 걸림 방지
3. 필요 시 `nice`로 저우선순위 실행
4. Mac 절전 정책 점검 (자동 잠자기 시 스케줄 누락 가능)
5. 로그 로테이션 및 보존기간 정책 적용

---

## 9. 비용/제한

1. 기본 구성은 추가 과금 없이 시작 가능 (서비스 정책/한도 내)
2. 변동 가능 항목:
   - Tailscale 플랜 정책
   - Codex 사용량 한도/요금 정책
3. 운영 전 월 1회 정책 재확인

---

## 10. 점검 체크리스트

1. 외부 PC/모바일 SSH 접속 성공
2. 대화형 Codex 세션 재접속 성공
3. Job 실행기(`codex-run-job.sh`)로 Job 단발 실행 성공
4. `cron` 또는 `launchd`에서 스케줄 Job 1회 이상 성공
5. 중복 실행/입력 누락/인증 오류 시 실패 로그가 남음
6. Job별 결과물 경로와 로그 경로가 분리됨
7. `API 업로드` Job이 포함되어도 다른 Job과 충돌 없이 동작

---

## 11. 다음 단계 (실행 순서)

1. 서버 계정/디렉토리/SSH/Tailscale 기본 세팅
2. 공통 Job 실행기 배치
3. Job 카탈로그 초안 작성 (빌드/테스트/백업/API 업로드)
4. 대화형 운영 검증
5. 스케줄러(launchd 우선, 필요 시 cron) 등록
6. 24시간 관찰 후 운영 기준 확정
