---
name: vercel-cicd
description: >-
  로컬 프로젝트를 GitHub 저장소에 올리고 Vercel에 연결해 CI/CD 자동 배포(main 푸시 → 프로덕션 자동 배포, PR·브랜치 → 프리뷰
  배포)를 세팅한다. "vercel에 배포해줘", "깃허브에 올리고 배포", "ci/cd 세팅", "푸시하면 자동 배포되게", "라이브로 띄워줘 /
  공유 링크 만들어줘", "프로덕션에 올려줘", "PR 미리보기 URL 나오게" 같은 요청에 쓰고, 사용자가 그냥 "이거 배포해줘"라고만 해도
  로컬 웹앱을 실제 URL로 띄우려는 맥락이면 반드시 이 스킬을 쓴다. 전제: gh CLI 인증 + Vercel MCP 연결(Vercel CLI 로그인은
  스킬 안에서 처리). 다음에는 쓰지 않는다: 로컬에서 npm run dev로 확인만 하기(배포 아님), AWS·Docker·Netlify 등 다른
  플랫폼 배포, 플랫폼 비교·추천, 이미 배포된 사이트의 런타임 버그 디버깅, 커밋 메시지·README 정리 같은 배포와 무관한 git 작업.
---

# Vercel CI/CD — GitHub 업로드 + 자동 배포 세팅

로컬 프로젝트를 **GitHub → Vercel**로 연결해 "푸시하면 자동 배포"되는 파이프라인을 만든다.
핵심은 두 가지가 얽혀 있다는 점이다: **소스는 GitHub**에 두고, **Vercel이 그 저장소를 감시**하다가 커밋이 올라오면 빌드·배포한다.
그래서 단순 "한 번 올리기"가 아니라 **Git 연결(git integration)** 까지 해야 CI/CD가 완성된다.

## 언제 쓰나
- 로컬 웹앱(Next.js, Vite, Astro, SvelteKit 등 Vercel이 감지하는 프레임워크)을 실제 URL로 띄우고 싶을 때
- "깃허브에 올려줘", "vercel 배포", "ci/cd", "푸시하면 자동 배포" 요청
- 이미 GitHub에 올라간 저장소를 Vercel에 붙여 자동 배포만 켜고 싶을 때(→ 3단계부터)

## 전제 확인 (먼저 점검)
- `gh auth status` — GitHub CLI 인증됨
- Vercel MCP 연결됨 (배포 이력 검증에 사용). 미연결이면 Vercel MCP `authenticate`로 먼저 연결.
- **Vercel CLI 로그인은 MCP만으로 안 된다.** MCP의 `deploy_to_vercel`은 "CLI로 `vercel deploy` 하라"는 안내만 준다. 실제 배포·env 등록은 **`vercel` CLI**가 필요하고, CLI 로그인은 대화형이라 **사용자가 직접** 해야 한다(아래 2단계).

---

## 워크플로우

각 단계는 **작업 디렉터리를 명시적으로 `cd`** 해서 실행한다(셸 cwd가 다른 곳으로 흘러가 있으면 엉뚱한 package.json을 잡는다 — 실제로 자주 발생).

### 1. Git 저장소 + 시크릿 차단

먼저 저장소 상태를 만들고, **비밀값이 커밋되지 않게** 한다. 이게 가장 중요하다 — 한 번 푸시되면 공개 이력에 남는다.

1. git 저장소가 아니면 초기화: `git init -b main`. (참고: `create-next-app --disable-git`로 만든 프로젝트는 저장소가 아님)
2. `.gitignore` 점검·보강. 최소한 다음이 무시돼야 한다:
   - `.env*` (단 `!.env.example`로 템플릿은 예외 허용)
   - `node_modules`, `.next`/`dist` 등 빌드 산출물, `.vercel`
   - 에디터/개인 설정(`.obsidian/`, `.vscode/` 등 저장소에 불필요한 것)
3. 스테이징 후 **시크릿이 안 들어갔는지 검증**(이 확인을 건너뛰지 말 것):
   ```bash
   git add -A
   git check-ignore .env.local          # 출력되면 = 무시됨(정상)
   git diff --cached --name-only | grep -i '\.env'   # .env.example 만 나와야 안전
   git diff --cached --name-only | grep -c node_modules   # 0 이어야 함
   ```
   `.env.local` 등 실제 키 파일이 staged에 보이면 **멈추고** `.gitignore`부터 고친다.
4. 커밋. 사용자가 커밋을 원할 때만, 커밋 메시지 끝에 프로젝트 규칙의 Co-Authored-By 트레일러를 붙인다.

### 2. Vercel CLI 로그인 (사용자가 직접)

CLI 로그인은 브라우저 대화형이라 대신 못 한다. 사용자에게 이 명령을 **`!` 프리픽스로 직접 입력**하도록 안내한다:

```
! npx vercel login
```

로그인 방법(GitHub 등) 선택 후 완료되면, 이후 `npx vercel …` 호출이 모두 인증된 상태로 동작한다(글로벌 config 공유). "됐어" 신호를 받고 진행. `npx vercel whoami`로 확인.

> 대안: 사용자가 https://vercel.com/account/tokens 에서 토큰을 만들어 주면 `npx vercel --token=… …`로 완전 비대화형 처리도 가능하다. 하지만 기본은 `vercel login`이 간단하다.

### 3. GitHub 저장소 생성 + 푸시

이미 원격이 있으면 건너뛴다(`git remote -v`).

- **public/private 는 사용자에게 확인**(기본 private 권장 — 공개 시 이력·코드가 노출).
- 한 번에 생성+연결+푸시:
  ```bash
  gh repo create <repo-name> --private --source=. --remote=origin --push
  ```

### 4. Vercel 프로젝트 링크

```bash
npx vercel link --yes --project <repo-name>
```
**흔한 함정**: 팀 스코프가 여러 개면 스코프를 못 정해 실패하고, 출력의 `next[]`에 정답 명령을 알려준다. 그 힌트대로 `--scope <team-slug>`를 붙여 재실행한다:
```bash
npx vercel link --yes --project <repo-name> --scope <team-slug>
```
성공하면 `.vercel/project.json`에 `projectId`/`orgId`가 생긴다(이 값이 MCP 검증에 필요). 이때 Vercel CLI가 `.env.local`에 `VERCEL_OIDC_TOKEN`을 추가하는데, `.env*`가 무시되므로 커밋되지 않는다(정상).

### 5. 환경변수 등록

앱이 필요로 하는 env를 **Vercel에 등록**해야 배포된 앱이 동작한다. `NEXT_PUBLIC_*`처럼 빌드 타임에 박히는 값은 특히 필수 — 없으면 빌드는 되어도 런타임에 백엔드(DB 등) 연결이 끊긴다.

1. 등록할 값을 `.env.local`에서 읽되 **`VERCEL_OIDC_TOKEN`은 제외**(Vercel이 넣은 로컬 토큰).
2. production/preview/development 세 환경 모두에 넣는다(프리뷰 빌드도 동작하도록). 값은 stdin으로 전달해 따옴표/개행 문제를 피한다:
   ```bash
   for env in production preview development; do
     printf '%s' "$VALUE" | npx vercel env add <NAME> $env
   done
   ```
3. `npx vercel env ls`로 확인.

> 어떤 env가 필요한지는 프로젝트마다 다르다. `.env.example`/`.env.local`을 근거로 목록을 만들고, 애매하면 사용자에게 확인한다.

### 6. 프로덕션 배포

```bash
npx vercel --prod --yes
```
READY가 나오면 배포 URL이 출력된다. **프로덕션 별칭**(예: `<project>.vercel.app`)이 공개용이다.

### 7. CI/CD (Git 연결) — 이게 핵심

푸시 자동 배포는 **Vercel 프로젝트가 GitHub 저장소에 연결**돼 있어야 켜진다.
```bash
npx vercel git connect --yes
```
- 원격 origin을 감지해 연결한다. 이미 연결돼 있으면 "already connected"라고 나온다(그럼 CI/CD는 이미 켜진 것 — CLI 배포 과정에서 자동 연결되는 경우도 있다).
- 연결 후: **`main` 푸시 → 프로덕션 자동 배포**, **다른 브랜치/PR → 프리뷰 배포**.

### 8. 검증 (Vercel MCP + 브라우저)

말로 "될 겁니다"라고 하지 말고 **근거로 확인**한다.

1. **공개 접근**: 프로덕션 별칭이 200이어야 한다.
   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" https://<project>.vercel.app
   ```
   해시가 붙은 개별 배포 URL은 **302 → vercel.com/sso-api** 로 뜨는 게 정상이다(프리뷰 보호). 프로덕션 별칭이 공개면 된다.
2. **앱 동작**: 브라우저로 프로덕션 URL을 열어 실제로 렌더·데이터 로드(백엔드 연결) 확인.
3. **CI/CD 증명**: Vercel MCP `list_deployments`(projectId, teamId는 `.vercel/project.json`)로 배포 이력을 본다. 방금 푸시한 커밋 SHA가 있고 **`source: "git"`**(또는 `githubDeployment: "1"`)이면 CI/CD가 실제로 도는 것이다. `get_deployment`로 `readyState: READY` 확인.
4. **직접 시연 권유**: 사용자에게 작은 커밋을 푸시하게 하고, `list_deployments`에 새 배포가 뜨는지 함께 확인하면 CI/CD를 눈으로 증명할 수 있다.

---

## 보고 형식

배포 후 다음을 명확히 전달한다:
- **라이브 URL**(공개 프로덕션 별칭)
- CI/CD 동작 방식: `main` 푸시 → 자동 프로덕션 배포 / PR → 프리뷰
- 등록된 환경변수, 연결된 GitHub 저장소
- ⚠️ **보안 리마인더**: 공개 URL이 되면 앱의 접근 제어 수준이 그대로 노출된다. 백엔드가 개방된 상태(예: 열린 RLS, 인증 없음)면 "URL 아는 누구나 접근" 가능함을 반드시 알린다.

## 정직성 규칙

- Git 연결 여부를 **추측하지 말고 확인**한다. `get_project`에 link 필드가 안 보인다고 "연결 안 됨"이라 단정하지 말 것 — `vercel git connect`나 `list_deployments`의 `source`로 실제 확인한다. (이 착각은 실제로 발생한다.)
- 검증 없이 "자동 배포됩니다"라고 말하지 않는다. 배포 이력의 `source: "git"`가 근거다.

## 실패 시 대처

- **링크가 스코프 때문에 실패** → 출력 `next[]`의 `--scope …` 명령을 그대로 재실행.
- **배포는 됐는데 앱이 백엔드 연결 실패** → env 미등록/오타. `vercel env ls` 확인 후 재등록, 재배포(`vercel --prod`) 또는 재빌드 트리거.
- **프로덕션 URL이 302로 로그인 요구** → 해시 붙은 개별 배포 URL을 본 것. 프로덕션 별칭(`<project>.vercel.app`)으로 접속. 그래도 막히면 프로젝트의 Deployment Protection 설정을 확인.
- **빌드 실패** → MCP `get_deployment_build_logs`(errorsOnly)로 원인 확인.
