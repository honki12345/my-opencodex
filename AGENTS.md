# AGENTS.md

## Git 인증 지침

- GitHub 원격 작업(`fetch`, `pull`, `push`, `clone`)은 반드시 SSH 키 인증을 사용한다.
- HTTPS 원격 URL(`https://github.com/...`) 사용을 금지한다.
- 기본 원격(`origin`)은 SSH 형식(`git@github.com:<owner>/<repo>.git`)으로 유지한다.
- HTTPS 원격이 감지되면 아래 명령으로 즉시 변경한다.

```bash
git remote set-url origin git@github.com:<owner>/<repo>.git
```
