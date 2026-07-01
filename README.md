# plan-driven-marketplace

Claude Code 플러그인 마켓플레이스입니다. 새 앱을 **7단계 기획 문서 → 마일스톤 구현** 흐름으로 안내하는 `plan-driven-app-development` 스킬을 배포합니다.

## 설치 방법

Claude Code 안에서 아래 두 명령을 차례로 실행하세요.

**1) 마켓플레이스 등록**

```
/plugin marketplace add jo-narae/plan-driven-marketplace
```

**2) 스킬 설치**

```
/plugin install plan-driven-app-development@plan-driven-marketplace
```

설치 후 적용하려면:

```
/reload-plugins
```

## 사용 방법

설치하면 새 앱을 만들자고 할 때 스킬이 자동으로 동작합니다. 예:

```
할일 앱 만들어줘
```

그러면 다음 7단계를 **한 단계씩 질문하고 확인받으며** 진행한 뒤, 마일스톤 단위로 구현까지 이어집니다.

| 단계 | 산출 문서 | 내용 |
|---|---|---|
| Phase 1 | `docs/01-requirements.md` | 요구사항 (목적·사용자·기능·범위·완료 기준) |
| Phase 2 | `docs/02-personas.md` | 페르소나 (주 + 대조) |
| Phase 3 | `docs/03-scenarios.md` | 사용 시나리오 + 설계 원칙 |
| Phase 4 | `docs/04-engagement.md` | 차별화·성취감 장치 |
| Phase 5 | `docs/05-design.md` + `mockups/` | 디자인 (3시안 비교 후 확정) |
| Phase 6 | `docs/06-architecture.md` | 아키텍처 (스택·구조·스키마·상태흐름) |
| Phase 7 | `docs/07-plan.md` | 구현 계획 (마일스톤 M1~MN) |

## 업데이트 / 제거

```
/plugin marketplace update plan-driven-marketplace
/plugin uninstall plan-driven-app-development@plan-driven-marketplace
```

## 수록 플러그인

| 플러그인 | 설명 |
|---|---|
| plan-driven-app-development | 새 앱을 7단계 기획 후 마일스톤으로 구현하도록 안내하는 스킬 |
| vercel-cicd | 로컬 프로젝트를 GitHub에 올리고 Vercel에 연결해 CI/CD 자동 배포를 세팅하는 스킬 |

## 라이선스

MIT
