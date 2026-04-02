https://github.com/Kuberwastaken/claurst
참고할 만한 git

전체 구조부터 파악하여 영역별로 하나씩 깊이 있게 정리해 드릴게요.

---

# 🔍 Claude Code 소스코드 — 영역별 심층 분석

---

## 1️⃣ 전체 아키텍처 개요

Claude Code는 겉보기엔 단순한 CLI인데 내부는 완전히 다른 규모입니다.

- **엔트리포인트**: `main.tsx` 단일 파일 **785KB** — 이것 자체가 충격적인 규모
- **렌더러**: React + [Ink](https://github.com/vadimdemedes/ink) — 터미널 UI를 React 컴포넌트로 렌더링
- **런타임**: Bun (Node.js 대체, 빠른 번들링·실행)
- **언어**: TypeScript strict 모드, Zod v4 스키마 검증

핵심 구조는 사실 단순하게 요약됩니다:

> **Claude Code = 에이전트 루프 + 도구들 + 메모리 + 권한 시스템 + 멀티 에이전트 오케스트레이션**

---

## 2️⃣ Tool 시스템 (`tools/`)

**40개 이상의 도구**가 각각 독립 모듈로 구현되어 있습니다. 주목할 부분들:

**파일 시스템 계열**
- `FileReadTool` / `FileWriteTool` / `FileEditTool` — 기본 3종
- `GlobTool` / `GrepTool` — 내부적으로 `ugrep` 또는 `ripgrep` 네이티브 바이너리를 사용. 단순 JS 구현이 아님

**에이전트 계열**
- `AgentTool` — 서브 에이전트를 직접 스폰
- `TeamCreateTool` / `TeamDeleteTool` — 에이전트 "팀" 단위 생성/삭제
- `SendMessageTool` — 에이전트 간 메시지 전달

**내부 전용 (external 빌드에서 제거됨)**
- `ConfigTool` — 설정 직접 수정 (Anthropic 직원 전용)
- `TungstenTool` — 고급 기능 (이름만으로는 용도 불명)
- `SendUserFile` / `PushNotification` / `SubscribePR` — KAIROS 전용

**도구 스키마 캐시**: 매번 JSON 스키마를 생성하면 토큰이 낭비되니 `toolSchemaCache.ts`로 캐싱. 프롬프트 효율 최적화가 깊은 수준까지 되어 있음.

---

## 3️⃣ QueryEngine.ts — 핵심 LLM 엔진

**46,000줄** 단일 파일. 이 하나가 LLM 통신의 모든 것을 담당합니다:

- 스트리밍 응답 처리
- Tool-call 루프 (모델이 도구 호출 → 결과 반환 → 다시 모델에게 → 반복)
- Extended Thinking 모드 처리
- 재시도 로직 및 토큰 카운팅
- **프롬프트 캐시 최적화**: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 마커로 시스템 프롬프트를 static(캐시 가능) / dynamic(캐시 불가) 영역으로 분리

특히 `DANGEROUS_uncachedSystemPromptSection()` 이라는 함수명이 인상적인데, 캐시를 의도적으로 깨는 섹션에 DANGEROUS를 붙여서 "누군가 이걸 추가하려면 신중하게 생각해라"는 메시지를 코드 자체에 심어뒀습니다.

---

## 4️⃣ 멀티 에이전트 / Coordinator 시스템

`CLAUDE_CODE_COORDINATOR_MODE=1` 환경 변수로 활성화되는 오케스트레이션 시스템입니다.

단계별 동작:

| 단계 | 담당 | 목적 |
|------|------|------|
| Research | Worker들 (병렬) | 코드베이스 탐색, 파일 파악 |
| Synthesis | Coordinator | 워커 결과 취합, 스펙 작성 |
| Implementation | Worker들 | 스펙대로 수정 및 커밋 |
| Verification | Worker들 | 변경 사항 검증 |

시스템 프롬프트에 이런 문구가 있습니다:
> *"Parallelism is your superpower. Workers are async."*

그리고 **lazy delegation 금지 규칙**도 명시:
> *"Do NOT say 'based on your findings' — read the actual findings and specify exactly what to do."*

이건 Claude가 자주 하는 "요약에 의존" 패턴을 막기 위한 아키텍처 레벨 안전장치입니다.

---

## 5️⃣ Dream 시스템 — 메모리 통합 엔진

`services/autoDream/` — **Claude가 문자 그대로 "꿈을 꿉니다".**

세 가지 조건이 모두 통과해야 실행됩니다:
1. 마지막 dream으로부터 24시간 경과
2. 5세션 이상 경과
3. 통합 락 획득 성공

실행되면 4단계 진행:

1. **Orient** — 메모리 디렉터리를 ls, `MEMORY.md` 읽기
2. **Gather** — 새로 저장할 가치가 있는 정보 수집 (일별 로그 → 드리프트된 메모리 → 트랜스크립트)
3. **Consolidate** — 메모리 파일 작성/업데이트, 상대 날짜를 절대 날짜로 변환
4. **Prune** — `MEMORY.md` 200줄 / 25KB 이하로 유지, 낡은 포인터 삭제

Dream 서브에이전트는 **read-only bash**만 가집니다. 프로젝트를 볼 수는 있지만 수정은 못 함. 순수 메모리 통합 패스입니다.

---

## 6️⃣ Undercover Mode — 내부 코드네임 유출 방지

Anthropic 직원 (`USER_TYPE === 'ant'`)이 오픈소스 레포에 기여할 때 내부 정보가 커밋 메시지에 노출되지 않도록 하는 시스템.

시스템 프롬프트에 주입되는 내용:
```
UNDERCOVER MODE - CRITICAL

커밋 메시지/PR에 절대 포함하지 말 것:
- 내부 모델 코드네임 (동물 이름: Capybara, Tengu 등)
- 미출시 버전 번호 (opus-4-7, sonnet-4-8 등)
- 내부 도구, Slack 채널, 단축 링크
- "Claude Code"라는 문구나 AI라는 언급
- Co-Authored-By 라인
```

여기서 드러난 사실들:
- **내부 모델 코드네임 = 동물 이름** (Capybara, Tengu, Fennec 등)
- **"Tengu"** = Claude Code의 내부 프로젝트 코드네임으로 추정 (feature flag 수백 개가 `tengu_` 접두사)
- **Anthropic 직원들이 Claude Code를 이용해 오픈소스에 기여하고 있으며, AI인 척 숨기도록 설계됨**
- force-OFF 없음 — 내부 레포가 확실하지 않으면 무조건 undercover 유지

---

## 7️⃣ KAIROS — 항상-켜진 Claude

아직 미출시 기능. `assistant/` 디렉터리에 존재하며 `PROACTIVE`/`KAIROS` 컴파일 플래그로 게이팅.

- **매일 append-only 로그** 작성 — 하루 종일 관찰·결정·행동을 기록
- **주기적 `<tick>` 프롬프트** 수신 → 지금 개입할지 말지 판단
- **15초 blocking budget** — 15초 이상 걸릴 것 같은 액션은 자동 지연
- **Brief 모드** — 출력을 극도로 간결하게, 터미널을 폭격하지 않도록

전용 도구: PushNotification, SubscribePR — 일반 Claude Code에는 없는 것들.

---

## 8️⃣ 미공개 Beta API 헤더들

`constants/betas.ts`에서 현재 협상 중인 베타 기능 목록이 그대로 노출됩니다:

주목할 것들:
- `context-1m-2025-08-07` — 1M 토큰 컨텍스트 윈도우
- `redact-thinking-2026-02-12` — thinking 내용을 redact (숨김)
- `afk-mode-2026-01-31` — AFK 모드 (자리 비울 때 자동 처리)
- `task-budgets-2026-03-13` — 태스크별 예산 관리
- `token-efficient-tools-2026-03-28` — 도구 스키마 토큰 최적화

---

## 9️⃣ BUDDY — 터미널 안의 다마고치

`buddy/` 디렉터리에 **완전한 가챠 시스템**이 구현되어 있습니다.

- **18종 동물** 캐릭터 (Common ~ Legendary)
- userId 해시를 시드로 한 **Mulberry32 PRNG** — 같은 유저는 항상 같은 버디
- 종 이름들은 `String.fromCharCode()` 배열로 난독화 — 스트링 검색에 걸리지 않도록 의도적으로
- **1% 쉬니 확률** (레어리티와 독립적)
- 각 버디: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK 5개 스탯
- 5줄 높이, 12자 너비 ASCII 아트 스프라이트 + 여러 애니메이션 프레임
- **2026년 4월 1~7일 티저**, 5월 본 출시 예정으로 코드에 명시

이 버디는 단순 장식이 아닙니다. 이름으로 부르면 자기 퍼스널리티로 반응합니다.

---

## 🔟 Penguin Mode (Fast Mode)

`utils/fastMode.ts`의 API 엔드포인트:
```
/api/claude_code_penguin_mode
```

설정 키: `penguinModeOrgEnabled`, 킬스위치: `tengu_penguins_off`

이름은 완전히 내부 농담인데, 전체 시스템에 펭귄 관련 변수명이 쭉 이어집니다. 아무도 외부 빌드에서 보게 될 줄 몰랐던 거겠죠.

---

## 📌 총정리

가장 인상적인 발견들:

1. **"단순한 CLI"가 아님** — 멀티 에이전트, 꿈 시스템, 가챠 버디가 있는 복잡한 플랫폼
2. **외부 빌드와 내부 빌드가 근본적으로 다름** — 컴파일 타임 feature flag으로 완전히 분리
3. **Undercover Mode의 아이러니** — 내부 정보 유출 막는 시스템을 만들어놓고, `.npmignore`에 `.map` 추가를 까먹어서 전체 소스가 노출
4. **아직 공개 안 된 것들이 훨씬 많음** — KAIROS, ULTRAPLAN, Buddy, Coordinator, ULTRAPLAN, AFK 모드 등
