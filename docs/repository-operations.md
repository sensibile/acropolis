# 저장소 운영 기록

GitHub API로 바꾼 sensibile/acropolis 설정과 그 이유를 기록한다. 이 설정은 저장소 파일이 아니라서 Git 이력에 남지 않는다. 그래서 바꿀 때마다 이 문서에 이유, 이전 값, 확인 결과를 적는다. PR 리뷰와 머지 절차는 [DEVELOPMENT.md](../DEVELOPMENT.md)의 PR 리뷰와 머지 절을 따른다.

## 2026-10-23 태극 구성으로 맞춤

BOSS가 acropolis의 리모트 설정을 sensibile/taeguk과 같은 구성으로 맞추라고 지시했다. 적용 뒤 API로 다시 조회해 확인했다.

- 머지: squash만 허용한다(merge commit과 rebase 허용 해제). squash 커밋 제목은 PR 제목, 본문은 PR 본문이다. 머지 후 브랜치를 지운다. auto-merge를 허용한다.
- 보안: 취약점 알림과 Dependabot security updates를 켰다. CodeQL default setup을 actions와 python으로 켰다. secret scanning과 push protection은 원래 켜져 있었다.
- ruleset `main protection`(id 25699728)을 만들었다. 대상은 기본 브랜치이고 bypass는 없다. 규칙은 다음과 같다.
  - 브랜치 삭제와 non-fast-forward 금지
  - PR 필수: 승인 수 0, 리뷰 thread 해소 필수, squash만 허용, 출처가 불분명한 변경에 추가 승인 요구
  - 필수 체크 `documentation-policy`
  - CodeQL code scanning: 오류 이상 알림과 high 이상 보안 알림이 있으면 막음
- classic branch protection을 지웠다. ruleset이 main에 걸린 것을 확인한 뒤 BOSS 결정으로 지웠다. 이전 값은 필수 체크 `documentation-policy`, strict true, 관리자에게도 적용, 대화 해소 필수였다.

## 2026-10-27 GitHub Codex 리뷰 gate 도입

PR 9로 GitHub Codex 리뷰 결과를 `codex-review` status로 게시하는 gate를 도입했다. ruleset 필수 체크에 `codex-review`(integration 15368)를 더했고 strict를 true로 바꿨다. auto-merge opt-out용 `no-auto-merge` label을 만들었다.

## 2026-10-29 로컬 리뷰로 전환

BOSS 결정으로 GitHub Codex 리뷰 gate를 없애고 push 전 로컬 리뷰와 head를 고정한 auto-merge로 바꿨다. sensibile/taeguk#22와 같은 방향이다. 이 PR이 gate workflow와 스크립트를 지우고, pre-commit과 같은 검사를 하는 `checks` job을 추가한다.

필수 체크가 비는 순간이 없도록 다음 순서로 ruleset을 바꾼다.

1. 이 PR의 `checks` job이 통과하면 ruleset 필수 체크에 `checks`(integration 15368)를 더한다. `codex-review`는 아직 남긴다.
2. 이 PR을 머지한다. 머지 전까지는 main의 gate가 이 PR에도 `codex-review`를 게시한다.
3. 머지 직후 ruleset 필수 체크에서 `codex-review`를 뺀다. strict는 유지한다.

이 순서를 바꿔 workflow만 먼저 지우면 `codex-review` status가 게시되지 않아 모든 PR이 막힌다. 반대로 `checks`를 필수로 만든 뒤에는, 아직 `checks` job이 없는 base로 열린 PR이 main을 반영해야 체크가 생긴다.

`no-auto-merge` label은 새 절차에서도 opt-out으로 쓴다.
