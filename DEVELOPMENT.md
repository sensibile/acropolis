# 개발 훅 운영

설치할 버전의 checkout에서 `./scripts/install-hooks`를 실행한다. Git common directory의 `codex-hooks/`에 push gate를 복사하고 절대 hooksPath를 등록하여 이전 커밋의 linked worktree에서도 검사한다. 변경 후 재설치해야 설치된 gate가 갱신된다. 기존 `.githooks` 설정은 이전하며 다른 hooksPath와 worktree별 override는 덮어쓰지 않는다. pre-commit은 해당 checkout의 기존 검사를 호출하며 누락 시 차단한다. 기존 pre-commit 검사는 유지한다. 코드 작성 후 `.githooks/post-code`를 에이전트/편집기가 호출하며 자동 staging하지 않는다. Git에는 코드 작성 완료 이벤트가 없다.

## Push 준비

1. 의도한 변경을 커밋하고 작업 디렉터리를 정리한다. `main` 직접 push는 차단하며 작업 브랜치/PR을 사용한다.
2. `./scripts/prepare-push origin BRANCH [TARGET_BRANCH]`로 version 3 계획을 생성한다. TARGET_BRANCH 기본값은 main이며 실제 PR 대상 브랜치와 일치하게 지정한다. 같은 push 목적지 저장소의 대상 브랜치만 지원하며 fork 간 PR은 별도 지원이 필요하다. 단일 원격 URL/ref/기존 SHA/head와 target_ref/remote_main을 기록한다(remote_main은 대상 브랜치 tip의 기존 필드명이다). 새 브랜치와 기존 브랜치 모두 대상 tip과 최종 head의 유일한 merge-base를 scan_base로 사용하여 PR 전체 누적 변경을 검사한다. remote_old는 전송 상태이며 리뷰 기준으로 대체할 수 없다. 복수 push URL 및 인증정보 URL은 계획 저장 전에 차단한다. 인증은 credential helper를 사용한다. 광고된 대상 커밋이 없으면 local ref/FETCH_HEAD 갱신 없이 객체만 fetch한다. 대상에 이미 도달 가능한 과거 변경과 과거 비밀 정보 검사는 별도다.
3. Codex에게 출력된 정확한 base/head로 `$codex-security:security-diff-scan`을 실행하도록 요청한다. 설치된 플러그인의 전체 절차를 수행하고 완료한 sealed 결과 디렉터리를 보존한다. 준비 명령/Git hook이 자동으로 모델을 호출하는 방식은 아니다.
4. 현재 Codex가 같은 불변 범위를 자체 리뷰한다. 실제 버그·회귀·권한 경계와 검증 근거를 확인하고 아래 JSON에 결과를 기록한다. 자체 리뷰는 독립 리뷰로 표시하지 않는다.
5. `./scripts/review-gate record PLAN_JSON COMPLETED_SCAN_DIR AGENT_JSON`으로 등록하고 승인된 `git push origin BRANCH`를 실행한다.

```json
{
  "kind": "agent",
  "verdict": "pass",
  "review_mode": "self",
  "independent": false,
  "reviewed_base": "계획의 scan_base 전체 SHA",
  "reviewed_head": "계획의 head 전체 SHA",
  "reviewer": "실제 Codex 리뷰어 및 모델/버전",
  "summary": "검사 범위, 검토 근거, 한계와 결과",
  "blocking_findings": []
}
```

보안 결과는 수동 pass JSON이 아니라 플러그인의 `scan-manifest.json`, `findings.json`, `coverage.json`, `report.md`를 사용한다. 설치된 `validate_scan_contract.py`로 seal/계약을 확인한다. 전체 diff 범위, 완료 coverage, 미해결 항목 없음, findings 없음만 통과한다. 현 정책은 낮은 심각도라도 confirmed finding을 차단한다. waiver는 구현하지 않았다.

플러그인은 기본 Codex 캐시에서 발견한다. 다른 경로라면 `git config --local codex.securityPluginDir /absolute/plugin/path`로 지정한다. 미설치/검증 실패는 차단하며 설치를 자동 수행하지 않는다. 플러그인 validator는 신뢰하는 로컬 설치 코드다.

Git common directory의 `push-reviews/`에 계획/등록 결과를 저장한다. push 시 같은 원격/ref/base/head와 증거 해시 및 canonical 결과를 재검증한다. 증거 디렉터리와 agent JSON을 보존해야 한다. 다른 작업 디렉터리에서도 같은 Git 저장소는 증거를 공유한다. 등록된 결과가 바뀌면 다시 검사한다. 모든 브랜치에서 대상 tip과 목적지 ref를 등록/push 때 원격에서 재확인하며 merge-base를 재계산한다. version 2 계획/receipt는 새 gate에서 거부하므로 새 계획과 전체 PR 리뷰가 필요하다. 계획 파일은 전체 계획 내용의 해시로 구분한다. 기준 또는 목적지 브랜치가 바뀌면 새 계획과 검사가 필요하다.

**보장 범위:** canonical seal은 내용 일관성 검사이며 모델 실행을 암호학적으로 증명하지 않는다. 자체 리뷰 JSON도 리뷰어의 attestation이며 독립 검증을 뜻하지 않는다. 로컬 훅은 우회 가능하며 신뢰할 수 있는 CI/서버 정책을 대신하지 않는다. 기본 Codex 검사·자체 리뷰는 승인된 작업에 포함한다. 추가 외부 서비스나 모델로 코드/데이터를 전달할 때만 범위를 설명하고 승인받는다. 검사 완료가 전체 보안 보장은 아니다.

## PR

문제·변경 결과, 계약/책임 변경, 검사 및 리뷰 대상 SHA와 결과, 미실행 이유, 위험·복구 방법을 적는다. 기본 squash merge이며 commit은 검증 후 자동 수행하며 코드 push/merge는 명시적 승인을 받는다. 문서 전용 작업은 아래 gate 정책을 따른다. 하위 커밋을 먼저 원격에 올리고 acropolis에서 이전/이후 SHA, 관련 PR과 조합 검증을 기록한다. PR을 연 뒤의 Codex 리뷰 요청과 머지는 아래 Codex 리뷰 gate 절차를 따른다.

## Codex 리뷰 gate와 PR 머지

`scripts/codex_review_gate.py`와 `.github/workflows/codex-review-gate.yml`은 sensibile/taeguk main `c759d26`의 파일을 그대로 가져왔다. 테스트는 같은 커밋의 `tests/test_codex_review_gate.py`를 `scripts/test-codex-review-gate`로 옮겼다. 저장소 이름, 실행 진입점, bytecode·최적화 차단만 다르다. 원본 갱신을 따라가기 쉽도록 두 Python 파일은 Ruff 포맷 대상에 넣지 않는다. `scripts/test-codex-review-gate-adapter`는 acropolis 전용 계약·격리 통합 테스트다. 같은 시나리오를 `FakeGitHub`와, PATH의 stub `gh`를 쓰는 실제 `gh()` adapter로 각각 실행해 호출 argv와 결과·오류가 같은지 확인한다(PR 9 Codex 지적, AGENTS.md의 fake 계약 테스트 요구). 실제 GitHub API 호출은 토큰과 네트워크가 필요해 포함하지 않는다. 두 테스트는 pre-commit과 `codex-review-gate-tests` GitHub job에서 실행한다. 표준 라이브러리와 `gh`만 사용한다.

위 Push 준비로 push한 뒤 PR을 맡은 Claude Code 세션이 다음 순서로 진행한다. 리뷰 요청과 머지는 세션의 `gh` 인증 계정인 BOSS 계정으로 한다(2026-10-25 BOSS 결정). Codex는 `github-actions[bot]`의 요청에는 응답하지 않았고 사람 계정 요청에는 13초~2분 안에 응답했다(sensibile/taeguk 관측).

1. PR은 제목과 본문에 `@codex` 문자열 없이 연다. 그러면 Codex가 PR 생성 리뷰를 스스로 시작한다. 세션은 2분 동안 그 head에 대한 Codex 요약 댓글(`codex-review-state`)이 붙는지 확인하고, 붙지 않을 때만 `@codex review` 댓글을 남긴다. PR 제목·본문·댓글에는 리뷰를 요청하는 댓글 말고는 `@codex` 문자열을 쓰지 않는다. Codex는 백틱 안의 문자열도 멘션으로 읽고 리뷰 대신 클라우드 작업을 시작한다. 가리킬 때는 "Codex 리뷰 요청"처럼 풀어 쓴다. 리뷰 자체에는 Codex 클라우드 환경이 필요하지 않다.
2. `Codex review gate` workflow는 PR head 커밋의 Codex 리뷰 결과를 commit status `codex-review`로 게시하는 판정만 한다. 리뷰를 요청하거나 머지하지 않으며 시간 기준 통과 규칙도 없다. 근거는 `chatgpt-codex-connector[bot]` 요약 댓글의 `codex-review-state`다. head에 대한 Code Review가 `completed`이고 `has_unresolved_findings`가 false이면 success, 미해소 지적이 있으면 failure, 그 밖에는 pending이다. 값이 null이면 같은 head에 대한 Codex의 최신 결과 댓글이 "Didn't find any major issues"이고 Codex가 시작한 미해결 리뷰 thread가 0개일 때만 success다. 확인할 수 없으면 pending이다. Codex가 head를 리뷰하지 않으면 pending에 머물고 시간이 지나도 바뀌지 않는다. 댓글이나 status 조회가 실패해 근거를 읽지 못하면 gate는 pending을 게시한 뒤 실패로 끝난다. 같은 head에 남아 있던 이전 success가 바뀐 리뷰 대신 머지 조건을 채우지 않게 하기 위해서다(PR 9 Codex 지적). `record` job도 무효 표식 댓글을 남기지 못하면 pending을 게시한 뒤 실패로 끝난다. 실패하면 `gate` job이 건너뛰어지기 때문이다. 이때 남긴 pending status("Codex review evidence was invalidated but could not be recorded")가 무효 표식 댓글을 대신한다. gate는 `github-actions[bot]`이 남긴 이 status가 Codex 요약의 마지막 갱신과 같은 초이거나 그보다 나중이면 pending으로 판정하고, 시각을 읽을 수 없어도 pending으로 본다. 댓글 표식도 같은 초면 무효로 본다. 무효는 요약이 확실히 나중에 갱신돼야 풀리므로, 다음 gate 실행이나 동시에 돌던 gate가 바뀌지 않은 요약으로 success를 되돌리지 못한다. gate는 success를 게시한 뒤 근거를 한 번 더 읽고, 그사이 무효가 생겼으면 pending으로 바꾼다. 마지막으로 읽은 뒤 무효가 생기는 아주 짧은 구간은 남는다(Codex 자체 리뷰 AGENT-002). pending 게시마저 실패하면 이전 status가 남는다. 그래서 두 job 중 하나라도 실패하면 세션이 head의 status와 Codex 요약을 다시 확인하고, 필요하면 Codex 리뷰를 다시 요청한다. thread를 해결하는 이벤트로는 gate가 다시 실행되지 않으므로 다음 이벤트가 있어야 다시 판정한다. 새 커밋 push는 status를 pending으로 되돌린다. Codex 요약 댓글이 삭제되거나 PR base가 바뀌면 gate가 head 표식이 담긴 무효 표식 댓글을 남기고, 그 뒤 Codex가 요약을 갱신해야 다시 판정한다.
3. 세션은 PR을 열고 2분 안에 요약이 붙지 않았을 때, 새 커밋을 push했을 때, gate가 무효 표식을 남겼을 때 그 head에 `@codex review`를 한 번 남긴다. 그 head에 대한 결과 댓글이 이미 있거나 리뷰가 `running`이면 남기지 않는다. 요청 뒤 10분 넘게 시작하지 않거나 `running`에서 10분 넘게 멈추면 한 번 더 남기고, 그래도 안 되면 BOSS에게 보고한다. 멈춤은 Auto-fix 이벤트로 전달되지 않으므로 세션이 직접 다시 확인한다. 응답이 없다고 리뷰 없이 머지하지 않는다.
4. head의 `codex-review`가 success이고 그 head에 미해결 리뷰 thread가 없으며 `./scripts/pr-review-status`가 exit 0이면 `gh pr merge <번호> --squash --auto --match-head-commit <리뷰한 head SHA>`로 auto-merge를 켠다. 코드 작업은 그 전에 BOSS의 머지 승인을 받는다. 문서 전용 작업은 [문서 작업 규칙](docs/documentation-policy.md)대로 별도 승인 없이 켜지만 같은 status 조건을 만족해야 한다. 문서 면제는 push 전 로컬 보안 검사와 Codex 자체·독립 리뷰에만 적용되고, PR 생성 때 Codex가 스스로 하는 리뷰와 `codex-review` status에는 적용되지 않는다(2026-10-26 BOSS 결정). draft PR과 `no-auto-merge` label이 붙은 PR에는 켜지 않는다. 이 opt-out은 세션이 지키는 운영 규칙이며 GitHub 머지 조건이 아니다. 이미 켠 뒤 멈추려면 PR을 draft로 바꾸거나 auto-merge를 해제한다. 세션이 없는 PR은 status가 success여도 사람이 머지할 때까지 대기한다.

`codex-review` status는 GitHub ruleset이 강제하는 머지 조건이고, `./scripts/pr-review-status`는 세션이 머지 직전에 Security Review row와 PR 머지 상태까지 다시 읽는 관측 도구다. 둘 중 하나가 다른 하나를 대신하지 않는다.

GitHub API 일시 장애는 gate가 완전히 막지 못하는 한계로 수용한다(2026-10-27 BOSS 결정). gate는 head를 안 뒤의 판독·무효 기록 실패는 pending으로 막는다. 그러나 head를 알기 전 PR 조회가 실패하거나 pending 게시마저 실패하면 이전 status가 남는다. 이 경로는 운영 절차로 보완한다. `Codex review gate`의 `record`나 `gate` job이 실패하면 세션은 머지 전에 head의 `codex-review` status, Codex 요약, 미해결 thread를 다시 확인한다. 근거와 status가 맞지 않으면 auto-merge를 켜지 않거나 해제하고, workflow를 다시 실행하거나 Codex 리뷰를 다시 요청한다. 지적이 리뷰 thread로 달린 경우는 ruleset의 thread 해소 필수 규칙이 머지를 따로 막는다.

같은 head SHA로 PR을 둘 이상 열지 않는다(2026-10-27 BOSS 결정). commit status는 PR이 아니라 SHA에 붙고, GitHub는 서로 다른 브랜치가 같은 커밋을 가리키는 PR 여러 개를 허용한다. 그래서 같은 SHA의 두 main 대상 PR이 있으면 한 PR의 판정이 다른 PR의 판정을 덮어쓴다. 스크립트 주석의 "head와 base당 열린 PR은 하나"는 브랜치 기준일 뿐이다. 이 경우는 gate가 막지 않는 한계로 수용한다. 미해소 지적은 리뷰 thread로 남으므로 thread 해소 필수 규칙이 지적이 있는 PR의 머지를 따로 막는다. 세션은 PR을 열기 전에 같은 SHA의 열린 PR이 없는지 확인한다.

요약 댓글의 상태 표식은 Codex의 공개 API가 아니다. 형식이 바뀌면 gate는 pending으로 남고 필수 체크가 머지를 막는다. 이때 Codex 결과를 직접 확인하고 BOSS 확인을 받아 처리한다. workflow는 `pull_request_target`과 `issue_comment`로만 실행되고 기본 브랜치의 workflow와 스크립트를 실행하므로 PR이 자기 판정 코드를 바꿀 수 없다. 그래서 gate를 처음 도입하는 PR 자신은 판정받지 못한다. `gate` job 권한은 `statuses: write` 외에는 read다. push 권한이 있는 사람은 다른 workflow로 같은 이름의 status를 게시할 수 있으므로 gate는 writer를 신뢰하는 자동화이며 writer에 대한 보안 경계가 아니다. fork PR과 기본 브랜치가 아닌 base로 향하는 PR은 처리하지 않는다. gate는 acropolis PR만 판정한다. submodule 저장소의 리뷰와 머지는 각 저장소의 절차를 따른다.

GitHub 설정: gate PR이 main에 들어간 뒤 ruleset `main protection`(id 25699728)의 필수 체크에 `codex-review`(integration 15368)를 추가한다. 기본 브랜치에 없는 workflow는 댓글 이벤트에 반응하지 않기 때문이다. status context 이름을 바꾸면 ruleset도 함께 갱신한다. auto-merge opt-out용 `no-auto-merge` label을 둔다. 같은 시점에 머지 전 PR 브랜치가 최신 main을 포함하도록 요구한다(`strict_required_status_checks_policy`, 2026-10-26 BOSS 결정). 그래서 squash 결과가 PR 단계에서 검사한 내용과 같다. main이 앞서 나가 PR이 뒤처지면 auto-merge는 대기한다. 세션이 `gh pr update-branch`로 갱신하면 head가 바뀌므로 3단계대로 Codex 리뷰를 다시 요청하고 새 head에 대해 4단계부터 다시 진행한다. 하위 커밋을 가리키는 gitlink가 바뀐 상태라면 갱신 뒤에도 위 Push 준비의 하위 증거 조건을 다시 확인한다.

## Producer 호환성

Desktop codex-security 결과의 `target.remote`는 선택 필드다. 없으면 full base/head Git object SHA로 검사 코드를 식별하고, 별도 receipt가 실제 push URL/ref를 결합한다. remote가 제공되면 SSH/HTTPS 표기를 정규화해 저장소 위치도 비교한다. 다른 저장소라도 동일한 base/head 객체는 같은 검사 코드이며, seal 자체는 실행자 신원을 인증하지 않는다.

## Push gate 회귀 검사

`./scripts/test-push-boundary`는 플러그인 없이 임시 로컬 Git 저장소에서 계획·기준·URL·linked worktree 설치 경계를 검사하며 pre-commit에 연결된다. `./scripts/test-push-gate`는 설치된 Codex Security validator와 synthetic sealed fixture로 receipt/evidence 및 Git pre-push 프로토콜을 검사한다. 이는 제품 보안 리뷰가 아니며 실제 원격에는 push하지 않는다.

## 추가 push 경계

상대 로컬 push URL은 worktree마다 다른 목적지가 될 수 있어 거부한다. 절대 경로나 절대 `file://` URL을 사용한다. push 직전에도 현재 remote의 모든 push URL을 다시 열거하여 복수 목적지를 차단한다. Git replacement refs, custom replacement namespace 및 legacy grafts가 있으면 prepare/record/push 모두 거부한다. 검사와 전송이 같은 Git 객체를 다루도록 먼저 해당 설정을 제거하고 새 계획과 리뷰를 만든다.

변경된 기존 gitlink마다 하위 저장소의 정확한 이전/이후 SHA에 대한 sealed 검사와 Codex 리뷰 receipt가 필요하다. 하위 커밋을 먼저 push하고 다음 JSON을 `record`의 마지막 선택 인자로 전달한다:

```json
{
  "akashic": {
    "repository": "/absolute/child-repository",
    "receipt_path": "/absolute/child-git-common-dir/push-reviews/identity.json"
  }
}
```

실제 변경된 경로만 넣는다. 상위 receipt는 하위 receipt의 해시를 보존하고 등록/push 때 하위 sealed 결과·agent 해시·범위 및 원격 ref에서 새 커밋의 도달 가능성을 재검사한다. 하위 receipt와 검사 파일을 보존한다. gitlink 삭제는 새 코드가 없으므로 하위 검사를 요구하지 않는다. 최초 gitlink 추가/일반 파일에서 gitlink 전환은 초기 보안 기준이 정의되지 않아 현재 gate가 차단한다. 하위 리뷰 없이 상위 coverage만으로 통과하지 않는다.

테스트와 설정 검사에서 Python 최적화 모드를 사용하지 않는다. `PYTHONOPTIMIZE`/`python -O`로 assertion이 제거되는 실행은 fixture 생성 전 비영 종료한다.

## PR 피드백 완료 확인

push 성공은 PR 피드백 처리 완료가 아니다. 최신 head에서 봇 재리뷰 완료를 확인하고, 새 지적을 원문별로 검증한다. 타당한 지적은 수정·검사·새 최종 HEAD의 전체 PR 범위 리뷰를 거쳐 올린다. 검증한 수정에는 근거 답글을 남긴 뒤 resolve한다. outdated 표시나 시간 경과만으로 resolve하지 않는다. 판단이 갈리거나 같은 지적이 반복되면 원문·재현·반증을 BOSS에게 제시한다.

`./scripts/pr-review-status OWNER/REPO PR_NUMBER`는 최신 head와 Codex 봇 summary의 Code Review 완료, 모든 미해결 스레드, 현재 머지 상태를 읽는다. 최신 head의 봇 완료 기록이 없거나 진행 중이면 대기 상태다. 모두 충족한 관측에만 exit 0, 미완료는 3, API/파싱 오류는 비영 종료한다. 봇 summary 형식이 바뀌거나 요약이 없으면 완료를 추정하지 않는다. 이 도구는 반복 감시·답글·resolve·merge를 실행하지 않는다. 관측 후 새 push/리뷰가 생길 수 있으므로 머지 직전 다시 실행한다.

## acropolis Python 검사 설치

`uv venv .cache/dev-tools` 다음 `uv pip install --python .cache/dev-tools/bin/python -r requirements-dev.txt`로 고정 Ruff를 설치한다. post-code는 gate Python 파일을 포맷하고 E9/F 정적 검사를 실행한다. pre-commit의 `scripts/check-python`은 동일 파일의 포맷을 check-only로 확인하고 정적 검사를 수행한다. Ruff format/check 모두 --no-cache로 실행하여 cold cache에서도 파일을 생성하지 않는다. 누락/다른 버전은 차단하며 hook이 도구를 설치하거나 stage하지 않는다.

## Endpoint 리뷰의 이력 제한과 설치 보완

현재 canonical diff 리뷰는 endpoint tree 비교이므로 새 커밋 1개인 선형 update만 prepare/record/push에서 허용한다. 기존 브랜치에서는 `remote_old`, 새 브랜치에서는 `scan_base`가 head의 유일한 부모여야 한다. 전송 이력 제한과 누적 PR 리뷰 범위는 별개이므로 이미 전송된 PR 커밋 여러 개를 전체 리뷰하는 것은 허용한다. 내용이 추가됐다 삭제되는 중간 커밋과 merge history를 endpoint 리뷰만으로 승인하지 않는다. 여러 미전송 커밋은 공유 이력을 바꾸지 말고 커밋별로 계획·검사·push하거나 별도의 전체 이력 검증 지원을 먼저 마련한다.

설치기는 worktreeConfig boolean을 Git의 `--type=bool`로 읽고 기존 worktree override를 검사한다. 완성된 임시 실행파일에 실행 권한을 준 뒤 같은 디렉터리에서 rename하여 기존 파일을 빈 파일/부분 코드로 노출하지 않는다. 설치된 세 실행파일의 교체는 각각 원자적이며 전체 세 파일의 단일 트랜잭션은 아니다.

하위 변경 탐지는 `--ignore-submodules=none`으로 로컬 ignore 설정을 덮어쓴다. committed head의 `.gitmodules` path/URL과 하위 receipt 목적지를 비교하고 그 소비자 URL에서 커밋 도달 가능성을 확인한다. URL은 절대 로컬 경로, 절대 file URL 또는 기존 네트워크 URL을 사용한다. 상대 submodule URL은 resolution 계약이 정의되지 않아 차단한다. SHA와 소비자 URL이 같은 정확한 gitlink 이동은 기존 baseline을 보존하여 하위 코드 재리뷰를 요구하지 않는다. 모호한 이동/최초 추가와 SHA 변경 없는 소비자 URL 교체는 별도 증거 계약이 없어 차단한다.

하위 evidence는 전체 하위 변경 범위를 검토한 `receipt_path` 1개를 권장하며 순서가 있는 `receipt_paths` 배열도 받는다. 누적 리뷰 receipt A→C는 그 하나로 전달한다. A→B와 A→C를 이어 붙이면 중복 범위이므로 거부한다. 배열의 각 receipt는 직전 head를 다음 scan_base로 사용해야 하며 첫 base/마지막 head가 상위 gitlink 변경과 일치해야 한다. 모든 단계를 별도로 재검증하고 receipt 해시를 상위 등록 결과에 보존한다. 원격 ref tip이 로컬에 없으면 검증 shell이 `fetch --no-tags --no-write-fetch-head URL SHA`로 객체만 가져오며 작업 브랜치와 FETCH_HEAD는 갱신하지 않는다. PR 상태 도구는 pagination 이후 metadata를 다시 읽고 같은 head인지 확인한 뒤 최종 머지 상태로 판단한다.

목적지 비교는 커스텀 포트와 일반 SSH 서버의 상대/절대 경로를 보존한다. HTTPS/SSH 별칭은 GitHub의 동일 저장소 namespace에 한해 정규화하고, 로컬 경로/file URL은 실제 절대 경로로 비교한다. SHA가 같은 기존 gitlink도 base/head 전체 목록에서 committed `.gitmodules` URL을 비교하여 URL만 변경한 커밋이 하위 도달 가능성 검증을 건너뛰지 못하게 한다.

빈 worktree hooksPath도 명시적 override이므로 설치를 차단한다. URL의 query/fragment는 전송 의미의 동등성을 보장할 수 없어 거부하고, 일반 네트워크 URL의 escaped path는 그대로 비교한다.

하위 URL은 file/HTTP(S)/SSH/Git 전송과 절대 로컬 경로만 지원하며 `<helper>::<address>` 및 임의 scheme은 Git 호출 전에 차단한다. 설치 대상 실행파일 경로가 디렉터리이면 기존 상태를 보존하고 차단한다. PR 완료 관찰은 thread pagination 뒤 최신 봇 요약과 metadata를 읽으며, 최종 reviewDecision/각 리뷰어의 latestReviews에 Changes requested가 남으면 mergeable 표시와 무관하게 차단한다.

등록된 worktree를 검사할 수 없거나 설정 조회가 오류이면 설치를 차단한다. `reported` coverage도 push를 차단한다. 하위 원격 도달 가능성은 receipt 목적지 ref를 먼저 검사하고 tip을 순차적으로 검증하며, 첫 증명 이후 관련 없는 ref를 fetch하지 않는다. 특정 tip fetch 실패만으로 종료하지 않으며 어느 ref에서도 증명하지 못하면 차단한다.

worktree 목록은 `--porcelain -z`와 NUL-safe reader로 읽어 줄바꿈이 있는 경로도 검사한다. 새 브랜치 준비 시 advertised main 객체가 없으면 local ref/FETCH_HEAD를 갱신하지 않고 해당 SHA만 fetch한 뒤 baseline을 검증한다. pre-commit 경계 테스트는 모듈 로드 전 bytecode 쓰기를 끈다.

`-`로 시작하는 repository URL은 Git 옵션으로 해석되지 않도록 모든 목적지 검증에서 거부하며, ls-remote/fetch에도 옵션 종료자 `--`를 사용한다.

추가 경계: 훅 설치는 등록된 checkout의 실제 common Git 디렉터리를 비교하고, hooksPath가 없을 때 활성 기본 훅을 보존하기 위해 설치를 거부한다. 로컬 push URL은 심볼릭 링크 경유를 거부하며 canonical 경로를 사용해야 한다. PR 상태 조회는 마지막 API 조회 뒤 스레드를 다시 페이지 조회하지만 원자적 스냅샷은 아니며 실제 merge 시점의 GitHub 보호 규칙이 최종 gate다.

조건부 include를 포함한 모든 checkout의 hooksPath 출처를 검사한다. 선언된 저장소와 다른 위치로 확장하는 URL rewrite는 거부한다. 봇 요약의 짧은 커밋 ID는 GitHub commit API의 전체 SHA로 확인하며 모호하거나 조회할 수 없는 ID는 완료로 인정하지 않는다.

공유 codex-hooks 디렉터리 자체의 심볼릭 링크도 설치 전에 거부한다. 기존·끊어진 링크 모두 대상 파일이나 hooksPath를 수정하지 않는다.

커밋된 .gitmodules 경로는 NUL record와 첫 newline 구분자로 읽어 경로 안의 줄바꿈을 보존한다. 준비·실제 훅은 remote.<name>.receivepack 설정(명령행 -c 주입 포함)을 거부한다. 지원하는 전달 명령은 일반 git push origin BRANCH이며 --receive-pack/--exec는 금지한다. Git은 그 명령행 옵션을 pre-push 인자로 전달하지 않으므로 훅만으로 이를 강제할 수 없다. --no-verify와 같은 소유자 우회는 신뢰 경계 밖이며 서버 보호 규칙을 대신하지 않는다.

저장소 URL 값은 Git 출력의 마지막 LF만 제거하고 앞뒤 space 문자를 보존한다. URL 내부의 CR·LF·TAB은 파서/record 의미가 달라질 수 있으므로 명시적으로 거부한다. 커밋된 consumer URL과 원격 URL 확장·준비·push 비교가 같은 보존 규칙을 사용한다.

훅 설치는 현재 비활성인 onbranch/gitdir 조건까지 고려해 conditional include를 보수적으로 거부한다. 훅과 무관한 조건부 include도 이 설치 방식에서는 지원하지 않는다. 설치 후 소유자가 설정을 변경하거나 훅을 우회하는 경우까지 강제할 수는 없다.

계획 검증은 Git·원격 관측을 shell에서 불변 PlanSnapshot으로 수집한 뒤 순수 policy 함수에 명시적 입력으로 전달한다. policy는 오류 tuple을 반환하며 I/O를 수행하지 않는다. URL canonical 경로와 replacement/graft 검사는 shell의 admission 경계에 둔다.

설치 전에 모든 등록 checkout의 실행 가능한 .githooks 파일을 검사한다. pre-commit은 dispatch하고 기존의 정확한 managed pre-push만 공유 gate로 교체한다. post-code는 Git 이벤트가 아닌 명시적 편집 작업이다. commit-msg 등 추가 훅과 커스텀 pre-push는 임의로 비활성화하지 않고 설치를 차단하여 수동 조정을 요구한다.

경로를 담은 Git 출력은 bytes로 읽고 newline 변환 없이 decode한다. CR과 LF 경로를 구분하며 커밋된 .gitmodules는 regular blob 모드만 허용한다. symlink와 다른 tree entry는 파싱 전에 차단한다.

Codex 리뷰의 reviewer와 summary는 공백을 제외한 내용이 있는 문자열만 인정한다. 지정한 security plugin 경로는 절대 경로여야 하며 canonical 경로로 확인한다. 공유 설치 대상에 관리하는 세 이름 외 실행 파일이 있으면 활성화하거나 삭제하지 않고 설치를 차단한다.

PR 요약에 포함된 Code Review와 Security Review는 모두 완료되어야 하며 각각의 커밋 ID를 전체 SHA로 해석하여 최신 head와 비교한다. 다른 커밋을 검토한 완료 row는 현재 작업의 완료 증거가 아니다.

POSIX wrapper는 sentinel로 Git 경로 출력의 마지막 LF 하나만 제거하여 checkout 이름 끝의 줄바꿈을 보존한다. Python pre-commit도 bytes 경로를 같은 규칙으로 읽는다.

## 문서 작업과 자동 커밋

검증된 한 의도 단위는 적절한 시점에 별도 승인 없이 commit한다. 문서 전용 작업은 [문서 작업 규칙](docs/documentation-policy.md)에 따라 검증하고 push·PR 생성·squash merge까지 진행한다. 기존 코드 push·merge 승인 조건은 코드 작업에 적용한다.

문서 전용 변경도 pre-push를 정상 실행한다. 원격 main과 head의 merge-base부터 누적 범위를 검사하며 documentation-policy GitHub job이 같은 문서 검사를 실행한다. `python3 scripts/documentation-gate BASE_SHA HEAD_SHA`로 로컬 검증한다. 코드 변경이 섞이면 보안 검사·Codex 리뷰 증거를 요구한다. 실행 예제 변경은 코드 작업으로 검토한다.

## 작업 기록과 전달 완료

[작업 승인 기록과 PR 전달 절차](docs/task-delivery.md)와 [기록 템플릿](docs/task-record-template.md)을 사용한다. 실제 승인 근거와 범위는 작업별 로컬 기록에 보존하고, 자동 승인 거절·도구 실패·검사 finding을 구분한다. 기록은 기존 hook·보안 seal·서버 검사를 대신하지 않는다. 일반 PR 생성 또는 갱신 후 원격 head와 상태를 확인하고 작업에 attach하여 실제 전달 결과를 보고한다.

## 자체 리뷰 증거 호환성

기본 리뷰는 `review_mode=self`, `independent=false`를 명시한다. 기존 독립 리뷰의
`independent=true` 증거도 허용하며 mode는 생략하거나 `independent`로 지정한다.
mode/independent 조합이 모순되거나 자체 리뷰의 mode가 누락되면 차단한다. 두 방식
모두 정확한 base/head, pass, 빈 blocking_findings, 실제 reviewer/summary가 필요하다.
보안 scan의 canonical seal, coverage와 finding gate는 그대로 적용한다.
