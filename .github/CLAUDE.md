# .github guidelines

<!-- BEGIN 2ndlifeinc-agents-hub:skip-ci-policy -->

## CI commit policy

- GitHub Actions 는 commit message 의 `[skip ci]` 를 네이티브로 인식한다.
- agents/tools 설정 배포, install.sh 반영, generated policy sync 처럼 코드 검증 가치가 낮은 배포 전용 `main` push 는 commit 제목 끝에 `[skip ci]` 를 붙인다.
  - 예: `chore: install.sh 배포 반영 [skip ci]`
  - 예: `chore: Pi package config 반영 [skip ci]`
- 기능 코드, workflow, dependency, lockfile, release build 입력을 바꾸는 commit 에는 `[skip ci]` 를 붙이지 않는다.
- CI queue 를 점검할 때 배포 전용 commit 이 `[skip ci]` 없이 push 되어 불필요한 run 을 만들었는지 확인한다.

<!-- END 2ndlifeinc-agents-hub:skip-ci-policy -->
