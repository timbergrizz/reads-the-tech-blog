---
type: Study
title: 'Claude Code: Best practices for agentic coding'
tags: []
출처: '[Claude Code: Best Practices for agentic codingbest-practices](https://www.anthropic.com/engineering/claude-code-best-practices)'
---

- Cloude Code는 기본적으로 로우레벨과 의견 없이 개발되어, 특정 워크플로우를 강제하지 않고 모델 그 자체에 접근하도록 디자인되었다.

    - 이러한 디자인 철학은 유연하고 커스터마이징 가능하며, 스크립트를 쓸 수 있고, 안전하고 강력한 툴을 만들었다.

    - 그렇기 떄문에 강력하지만 새롭게 에이전틱 코딩 툴을 쓰는 사람들에게 러닝커브가 좀 있다.

# Customize your setup

## Create `Claude.md` files

- 클로드가 대화를 시작할 때 자동으로 넣는 컨택스트이다. 다음과 같은 정보를 도큐먼트 해두자.

    - 내가 사용하는 bash command

    - 코어 파일과 유틸리티 함수들

    - 코드 스타일 가이드라인

    - 테스트 방법

    - 레포지토리 에티켓 (브랜치 명명, 머지인지 리베이스인지 같은 룰들)

    - 개발 환경 설정 (pyenv use, 사용하는 컴파일러 등)

    - 프로젝트에 있어 예측 불가능할 수 있는 행동과 경고

    - 클로드가 기억하길 원하는 정보들

- 정해진 포맷은 없고, 사람이 읽을 수 있는 형태로 정밀하게 구성하는 것이 좋다.

    ```text
    # Bash commands
    - npm run build: Build the project
    - npm run typecheck: Run the typechecker
    
    # Code style
    - Use ES modules (import/export) syntax, not CommonJS (require)
    - Destructure imports when possible (eg. import { foo } from 'bar')
    
    # Workflow
    - Be sure to typecheck when you’re done making a series of code changes
    - Prefer running single tests, and not the whole test suite, for performance
    ```

- 여러 위치에 파일을 둘 수 있다.

    - 레포의 루트나, `claude` 커맨드를 실행하는 곳.

        - 깃에 올려서 공유할 수 있도록 하거나, local을 지정한 후 `.gitignore` 처리하기

    - `claude` 커맨드를 실행하는 곳의 부모 / 자식 디렉토리

        - 부모 디렉토리는 모노레포에 적절하다.

        - 자식 디레곹리에 있는 모든 `CLAUDE.md` 은 컨택스트에 포함된다.

    - 홈 폴더 `~/.claude/CLAUDE.md`에 넣으면 모든 세션에 포함된다.

## Tune your `CLAUDE.md` files

- 가장 일반적인 실수는 많은 컨택스트를 반복하면서 효율성 확인하지 않고 그대로 넣는 경우.

- 수동으로 CLAUDE.md에 추가하거나, #키를 눌러 모델에 지시사항을 전달할 수 있다.

- Anthropic에선, CLAUDE.md를 [prompt improver](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-improver) 에 넣거나, 연관성을 높이도록 지시를 튜닝한다.

## Curate Claude's list of allowed tools

- 안전성을 기준으로 설계된 클로드 코드는 기본적으로 시스템을 수정하는 요청에 권한을 요청한다.

    - safe하거나 불안전하지만 실행 취소가 용이한 툴에 대해 화이트리스트를 작성하면 좋다.

- 허용된 툴을 관리하는 방법은 4가지가 있다.

    - 세션동안 프롬프트에 Always Allow를 쓴다.

    - `/allowed-tools` 커맨드를 사용해 툴을 추가하거나 삭제할 수 있다.

    - `.claude/settings.json`이나 `~/claude.json`에 적힌 데이터를 수정한다.

        - 팀과 공유하기 위해서는 전자를 확인하는 것을 추천

    - 세션에서 허용하려면 CLI에서 실행할 때 `--allowed-tools` 플래그를 사용한다.

## If using Github, install the gh CLI

- 클로드는 gh cli로 이슈만들거나 PR 열고, 코멘트 읽는 등의 작업을 할 수 있다.

- gh cli 안깔려있어도 MCP 깔거나 API를 쓸 수 있다.

# Give Claude more tools

- 쉘에 대해 접근할 수 있으며, MCP나 REST API 툴로 더 강력해쥘 수 있다.

## Use Claude with bash tools

- 클로드는 unix 기본 툴과 gh를 알지만, 커스텀 bash tools에 대해서는 instruction에 없으면 모른다.

    - 툴의 이름과 사용 예시를 모델에 제공한다.

    - 툴의 도큐멘테이션을 확인하게 하려면 `--help` 옵션을 실행하게 하자

    - 자주 사용하는 툴들은 `CLAUDE.md`에 적어두자

## Use Claude with MCP

- 클로드 코드는 MCP 서버 클라이언트 둘 다 작동한다.

- 클라이언트로는, 개수 제한 없이 MCP 서버로 연결할 수 있다.

    - 실행중인 디렉토리에 선언된 프로젝트 설정, 글로벌 설정, 체크인된 .mcp.json파일에 정의

- `--mcp-debug` 옵션으로 설정 오류도 확인할 수 있다.

## Use Custom slash commands

- 루프를 디버깅하거나 로그 분석하는 반복 작업에서 `.claude/commands` 에 프롬프트 템플릿을 마크다운에 저장해두자.

    - `/` 메뉴에서 바로 사용할 수 있게 된다.

- 다음과 같은 형태로 저장해두고 클로드 코드에서 사용할 수 있도록 한다.

    ```text
    # .claude/commands/fix-github-issue.md
    Please analyze and fix the GitHub issue: $ARGUMENTS.
    
    Follow these steps:
    
    1. Use `gh issue view` to get the issue details
    2. Understand the problem described in the issue
    3. Search the codebase for relevant files
    4. Implement the necessary changes to fix the issue
    5. Write and run tests to verify the fix
    6. Ensure code passes linting and type checking
    7. Create a descriptive commit message
    8. Push and create a PR
    
    Remember to use the GitHub CLI (`gh`) for all GitHub-related tasks.
    ```

    - 개인적인 커맨드를 `~/.claude/commands` 에 저장해 둘 수 있다.

# Try common workflows

- 클로드는 특정 워크플로우를 드러내지 않고, 사용자가 원하는대로 작동한다.

- 성공적인 패턴로는 이런 것들이 있다.

## Explore, plan, code, commit

- 수행 과정

    1. 포인트를 제공하여 모델이 관련된 파일, 이미지, URL을 읽도록 하고, 코드 생성은 하지 않도록 한다.

        - 서브에이전트를 강력하게 사용하는 것을 고려해야할 때의 워크플로우의 일부이다.

        - 클로드가 디테일 체크나 질문에 대한 조사를 할때 효율성을 잃지 않도록 한다.

    2. 특정 문제에 대해 어떻게 접근할 것인지 계획을 먼저 물어본다

        - think같은 단어를 써서 extended thinking mode로 작업하도록 하면 좋다.

            - `think < think hard < think harder < ultrathink` 같은 명령어로 thinking에 사용할 버짓을 결정할 수 있다.

        - 이 결과가 마음에 들면, 문서를 만들거나 깃허브 이슈를 작성하도록 한다.

    3. Claude가 코드로 솔루션을 구현하도록 한다.

        - 응답 결과의 합리성을 모델을 통해 명시적으로 확인하기 좋은 위치이다.

    4. 결과를 커밋하고 PR을 생성하도록 한다.

        - 클로드가 어떤 작업을 수행했는지 README나 changelog를 수정하도록 하기 좋다. 

- 1, 2단계가 핵심이다. 이 단계로 모델이 바로 솔루션 코딩하는것을 막을 수 있다.

## Write tests, commit: code, iterarte, commit

- Anthropic에서 자주 사용하는 워크플로우

    - 쉽게 유닛, 통합, E2E테스트를 수행하도록 할 수 있다.

- 수행 과정

    1. 예상되는 입출력을 기반으로 테스트 코드를 작성하도록 한다.

        - TDD를 수행하고 있고, 존재하지 않는 함수에 대해 mock 구현을 만들지 않도록 명시한다.

    2. 테스트를 실행하도록 한 후, 테스트가 실패하는것을 확인하도록 하라

        - 명시적으로 구현을 하지 않도록 하는 것이 보통은 도움된다.

    3. 생성된 테스트가 괜찮으면 커밋하도록 한다.

    4. 테스트를 통과하는 코드를 작성하도록 한다.

        - 테스트를 수정하지 않도록 명시한다.

        - 테스트를 통과할 떄 까지 계속 코드를 수정하도록 한다.

        - 모델은 코드 작성 -> 테스트 실행 -> 코드 조정 -> 테스트 실행 과정을 반복한다.

        - 이 과정에서 서브에이전트에게 물어보도록 하여 구현이 오버피팅되는 것을 막을 수 있다.

    5. 생성된 코드 결과가 마음에 들면 코드를 커밋한다.

- 클로드는 목적을 달성하기 위해 반복하는 과정이 있을때 가장 잘 작동한다.

    - 테스트같은 예상되는 출력을 제공하여 클로드가 성공할 때 까지 변화를 만들 수 있다.

## Write code, screenshot result, iterate

- 테스트 워크플로우와 유사하게 시각적인 목표를 제공할 수도 있다.

- 수행 과정

    1. 클로드가 브라우저를 스크린샷 사용할 수 있는 방법을 제공한다.

        - [ios simulator MCP](https://github.com/joshuayoes/ios-simulator-mcp)같은 것들이 있다

    2. 클로드에 이미지 복사 붙여넣기 등의 방법으로 비주얼 mock을 제공한다.

    3. 클로드에 디자인을 구현하도록 하고, 결과의 스크린샷이 mock과 동일할 때 까지 반복하도록 한다.

    4. 만족스러울 때 커밋하도록 한다.

## Safe YOLO mode

- Claude를 감독하지 말고, `claude --dangerously-skip-permissions` 옵션을 켜서 모든 권한 확인을 바이패스하도록 한다.

- 린트 에러 수정이나 보일러플레이트 코드 생성에 적절하다.

- 리스크가 있고, 데이터 소실이나 시스템 오염, 데이터 유출등의 위험이 있을 수 있으니 [컨테이너 안에서](https://github.com/anthropics/claude-code/tree/main/.devcontainer) 돌려라.

## Codebase Q&A

- 모델에 다른 엔지니어한테 물어보는 것 같이 다음과 같은 질문을 해서 코드 베이스를 온보딩할 때 도움을 받을 수 있다.

```text
How does logging work?
How do I make a new API endpoint?
What does async move { ... } do on line 134 of foo.rs?
What edge cases does CustomerOnboardingFlowImpl handle?
Why are we calling foo() instead of bar() on line 333?
What’s the equivalent of line 334 of baz.py in Java?
```

- 앤트로픽에서는 이렇게 사용하는 것이 가장 중요한 온보딩 워크플로우다.

    - ramp-up 타임을 엄청나게 줄이며, 다른 엔지니어들의 로드를 줄일 수 있다.

    - 다른 특별한 프롬프팅 필요 없이, 질문을 하면 클로드가 코드베이스 찾아서 정답을 알려준다.

## Use Claude to interact with git

- 모델은 깃을 효과적으로 잘 다룬다. 앤트로픽 엔지니어들은 90% 이상의 깃 사용을 클로드로 한다.

    - 질문에 답하기 위한 깃 히스토리 조회

    - 커밋 메시지 작성.

    - reverting files, resolving rebase conflicts, comparing, grafting patch같은 복잡한 작업

## Use Claude to interact with Github

- 클로드 코드는 깃허브와도 잘 상호작용한다.

    - Pull Request 생성

    - 간단한 코드에 대한 one-shot resolution 구현

    - 실패한 빌드나 린트 오류 수정

    - 생성된 이슈에 대한 카테고라이징과 심각도 분류

## Use Claude to work with Jupyter notebooks

- 모델은 이미지를 포함한 출력을 해석할 수 있고, 데이터와 상호작용 할 수 있는 빠른 방법을 제공한다.

- 필요한 프롬프트나 웤프를로우 없이 .ipynb파일을 vscode에 띄워놓고 보면 된다.

- Jupyter 노트북 정리해달라고 하면 기가 막히게 해준다.

# Optimize your workflow

## Be specific in your instructions

- 클로드 코드에 자세한 인스트럭션을 설정할수록 성공률이 올라간다.

- 명료한 방향성의 제공이 나중의 수정을 줄여준다.

- 모델은 의도를 추측할 수 있지만, 생각을 읽을 수는 없으므로 명시하는 것이 예측값을 맞추는데 도움이 된다.

## Give Claude Image

- 사진이나 다이어그램같은것을 줄 수 있다.

    - 스크린샷 복붙이나, 프롬프트 입력에 드래그앤드롭, 파일 경로 제공으로 할 수 있다.

- UI 개발에서 디자인 mock으로 작업할 때 도움이 되며, 분석과 디버깅 과정에 비주얼 차트로 도움이 된다.

- 컨택스트에 비주얼을 추가하지 않아도 결과가 어떻게 보여야 하는지 알려주는 것이 도움이 된다.

## Mention files you want Claude to look at or work on

- 레포지터리 내부에서 파일을 지정해서 모델이 적절한 자원을 찾고 수정할 수 있도록 한다.

## Give Claude URLs

- 웹에 필요한 자료가 있으면 읽어올 수 있도록 URL을 제공한다.

- `/allowed-tools`에 도메인 추가해서 허가 안받아도 되도록 한다.

## Course correct early and often

- 자동으로 돌리는것보다 적극적으로 클로드의 접근을 가이드하는 경우 더 좋은 결과를 얻을 수 있다.

- 다음과 같이 클로드가 하는 행동을 교정할 수 있다.

    - 코드를 작성하기 이전에 계획을 세우도록 명시하고 적절한 게획 전까지 코드를 작성하지 않도록 한다.

    - 어떤 상태에서도 Interrupt를 하여 컨택스트를 보존하면서 instruction을 추가한다.

    - Esc 두번 누르면 히스토리로 돌아갈 수 있고, 이전 프롬프트를 수정해서 다른 방향으로 갈 수 있도록 한다.

    - 모델에게 변경을 취소하라고 한다.

- 클로드 코드가 한번에 문제를 해결할 수 있긴 한데, 이러한 교정이 더 나은 솔루션을 빠르게 제공한다.

## Use `/clear` to keep context focused

- 세션이 길어지면 컨택스트 윈도우가 관계 없는 대화, 파일들로 찰 수 있고 이는 성능 하락과 모델 결과 생성의 방해로 이어진다.

- `/clear` 커맨드를 자주 사용해 컨택스트 윈도우를 비워준다.

## Use checklists and scratchpads for complex workflows

- 여러 스탭으로 포함된 큰 작업이나 철저한 솔루션이 필요한 경우 모델이 마크다운 파일을 작성해서 메모하도록 했을 때 잘 작동한다.

    - 코드 마이그레이션, 많은 수의 린트 에러 해결, 복잡한 빌드 스크립트 수행 등이 해당된다.

- 린트 이슈 해결시 다음과 같은 방법으로 체크리스트를 작성하도록 할 수 있다.

    - 린트 커맨드를 실행하여 결과를 마크다운 체크리스트에 기록하도록 한다.

    - 작성된 파일을 바탕으로 이슈들을 하나씩 수정하도록 한다.

## Pass data into Claude

-  클로드에 데이터를 제공하는 방법은 여러가지가 있다.

    - 복사 붙여넣기

    - Pipeline (`cat foo.txt | claude`) : 긴 파일을 넣을때 유용하다.

    - 데이터를 가져오라는 명령을 클로드에 넣기 : 쉘 커맨드, MCP, 커스텀 슬래시 커맨드로 수행한다.

    - 클로드에게 파일을 읽거나 URL의 데이터를 fetch하도록

# Use headless mode to automate your infra

- 클로드에는 CI 등의 상호작용 필요없는 환경을 위해 헤드리스 모드가 있다.

    - -p flag로 headless 모드 쓸 수 있고, --output-format stream-json처럼 하여 streaming JSON 출력으로 받을 수 있다.

- 이를 다음과 같은 유즈케이스에서 사용할 수 있다.

    - Issue triage

        - 헤드리스 모드는 깃허브 이벤트로 트리거하여 이슈 우선 순위를 자동으로 설정하도록 할 수 있다.

    - Use Claude as a linter

        - 주관적인 코드 리뷰 툴로 작동할 수 있다.

        - typo, stale comment, 잘못된 함수나 변수명 등을 잡아내도록 할 수 있다.

# Uplevel with multi-Claude workflow

- 단독 사용을 넘어서, 여러개의 클로드 인스터스를 병렬적으로 돌릴 수 있다.

## Have one Claude write code; use another Claude to verify

- 하나는 코드 작성하고, 하나는 작성된 코드를 리뷰하고 테스트하도록 한다.

- 다음과 같은 단계로 진행한다.

    1. 클로드에게 코드를 작성하도록 한다.

    2. `/clear` 나 다른 클로드를 실행시킨다.

    3. 두번째 클로드가 첫번쨰 클로드의 작업을 리뷰하도록 한다.

    4. 또 다른 클로드를 실행시켜 작업 내용, 피드백 읽고 이에 맞추어 수정하도록 한다.

- 테스트도 비슷화게 작동할 수 있다.

    - 하나 테스트 작성하도록 하고, 다른 클로드가 테스트 패스하도록 하면 e뇌다.

    - 스크래치 패드같은 것으로 인스턴스끼리 소통하도록 할 수도 있다.

- 하나로 작업하는 것보다 여러개를 사용하는 것이 효율적이다.

## Have multiple checkouts of your repo

- 클로드 하나가 스텝을 다 하는게 아니라, 여러개 실행시켜놓고 병렬적으로 수행하도록 한다.

- 다음과 같은 단계로 진행한다.

    1. 분리된 폴더에 3-4개의 깃 체크아웃을 만든다.

    2. 각 폴더를 터미널에 열고, 클로드를 실행하여 다른 태스크를 부여한다.

    3. 클로드들 돌아가면서 진행사항 확인하고 권한 승인 / 거부한다.

## Use git worktrees

- 여러개의 독립적인 태스크에 적절한 방법이고, multiple checkout보다 가볍다.

- 워크트리는 여러개의 브랜치를 체크아웃 할 수 있도록 한다.

    - 각 워크트리는 독립된 파일로 구성된 작업 디렉토리를 가지며 같은 깃 히스토리와 reflog를 갖는다.

- 태스크가 겹치지 않는다면, 머지 컨플릭트 없이 빠르게 작업을 수행하도록 할 수 있다.

- 다음과 같은 단계로 구성한다.

    - 워크트리 생성 : `git worktree add ../project-feature-a feature-a`

    - 생성된 각각의 워크트리에 클로드를 실행 `cd ../project-feature-a && claude`

    - 작업에 필요한만큼 반복

- 다음을 따르면 좋다.

    - 일관된 네이밍 컨벤션을 사용한다.

    - 워크트리 하나당 하나의 터미널을 유지한다.

    - 맥으로 iTerm2 사용하면, 알림 설정해서 attention 필요할 떄 알림 오도록 할 수 있다.

    - 다른 워크트리에 대해 분리된 IDE 윈도우 사용한다.

    - 완료 후엔 워크트리 정리해라 `git worktree remove ../project-feature-a`

## Use headless mode with a custom harness

- 헤드리스 모드는 클로드 코드를 빌트인 툴과 시스템 프롬프트을 통해 거대한 워크플로우에 통합한다.

- 헤드리스에 있어 2개의 메인 패턴이 있다.

    - Fanning out은 큰 마이그레이션이나 분석에서 사용한다.

        - 클로드가 태스크 리스트를 작성하는 스크립트를 실행하도록 한다.

        - 태스크 리스트에 대해 루프를 돌고, 클로드를 실행하여 각각의 프롬프트와 툴을 제공한다.

            ```bash
            claude -p “migrate foo.py from React to Vue. When you are done, you MUST return the string OK if you succeeded, or FAIL if the task failed.” --allowedTools Edit Bash(git commit:*)
            ```

        - 스크립트를 여러번 돌리고 원하는 출력까지 프롬프트를 개선한다.

    - Pipelining은 출력값을 기존 파이프라인에 통합하도록 한다.

        - `claude -p "<your prompt>" --json | your_command`로 실행하면, 다음 스탭에 커맨드를 실행하도록 할 수 있다.

    - 두 유즈케이스 모두 `--verbose` 사용하면 클로드 실행 디버깅하기 좋다.

        - 프로덕션에는 끄는 것을 추천한다.

