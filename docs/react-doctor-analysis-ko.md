# React Doctor 분석 및 활용 정리 (한국어)

이 문서는 React Doctor 레포지토리를 분석하고, 도구의 정체 · 사용법 · 활용
가능성 · 수익화 아이디어까지 정리한 결과물이다.

## 관련 링크

| 구분 | 주소 |
| --- | --- |
| 원본 레포 | https://github.com/millionco/react-doctor |
| 이 레포 (포크) | https://github.com/bmshin94/react-doctor |
| npm 패키지 | https://www.npmjs.com/package/react-doctor |
| 공식 문서 | https://react.doctor/docs |
| CI 문서 | https://react.doctor/ci |
| 이슈 트래커 | https://github.com/millionco/react-doctor/issues |

원본 레포 기준 스타 약 14.9k, 포크 483개. 라이선스는 MIT.

---

## 1. 이게 뭐하는 도구인가

React 코드베이스를 정적으로 훑어서 문제를 찾아내고 **0~100점 건강 점수**를
매기는 CLI 진단 도구다. 만든 곳은 Million.js 팀(Million Software, Inc.).

레포 README의 한 줄 소개가 제품의 방향을 그대로 보여준다.

> "Your agent writes bad React, this catches it."

AI 코딩 에이전트가 만들어낸 코드가 "돌아가긴 하지만 품질이 의심스러운" 상황을
겨냥한 도구다. 진단 축은 다음 여섯 가지다.

- state / effects
- 성능(performance)
- 아키텍처(architecture)
- 보안(security)
- 접근성(accessibility)
- 유지보수성(maintainability)

지원 범위는 Next.js, Vite, Astro, TanStack, React Native, Expo 등 React 계열
전반이다.

---

## 2. 저장소 구조

pnpm + turbo 기반 모노레포이며 `packages/` 아래 7개 패키지가 있다.

| 패키지 | 공개 여부 | 역할 |
| --- | --- | --- |
| `packages/react-doctor` | 배포 | 실제 사용하는 CLI (`npx react-doctor`) |
| `packages/oxlint-plugin-react-doctor` | 배포 | 진단 룰 본체 (핵심 자산) |
| `packages/eslint-plugin-react-doctor` | 배포 | 위 룰들의 ESLint 미러 |
| `packages/core` | 비공개 | 진단 엔진, 스캔 오케스트레이션, 점수 요청 |
| `packages/api` | 비공개 | 프로그래밍 방식 `diagnose()` |
| `packages/fuzz` | 비공개 | 룰 오탐(false positive) 퍼징 |
| `packages/evals` | 비공개 | 룰 품질 평가 하네스 |

루트의 주요 파일과 디렉터리는 다음과 같다.

- `action.yml` (약 35KB) — PR 자동 스캔 + 코멘트용 GitHub Action
- `skills/` — 코딩 에이전트에 설치되는 스킬 정의
- `.agents/skills/` — 기여자/에이전트용 내부 스킬 13종
- `.cursor-plugin/plugin.json` — Cursor 플러그인 매니페스트
- `docs/` — 룰 작성 가이드, 룰 후보 백로그, 리서치 문서
- `AGENTS.md` (약 28KB) — 기여자 가이드 + Effect v4 컨벤션

### 룰 카테고리 분포

`packages/oxlint-plugin-react-doctor/src/plugin/rules/` 아래 카테고리는 25개가
넘는다. 비테스트 소스 파일 기준 상위 분포는 다음과 같다(헬퍼 파일 포함이므로
공식 룰 개수와는 다르다. README 표기는 "100+ 룰").

```
r3f: 293              react-builtins: 150   design: 143
state-and-effects: 125  a11y: 108           correctness: 104
security-scan: 73     react-native: 47      performance: 37
nextjs: 25            ink: 22               architecture: 22
js-performance: 20    tanstack-start: 18    tanstack-query: 10
react-ui: 9           server: 8             security: 7
project: 7            bundle-size: 7        zod: 6
preact: 5             mobx: 3               jotai: 3   webgl: 2
```

---

## 3. 설치 및 사용법

별도 설치가 필요 없다. `npx`로 바로 실행한다.

```bash
npx react-doctor@latest
```

### 주요 명령어

| 명령어 | 설명 |
| --- | --- |
| `react-doctor` | 기본 진단 (점수 + 문제 목록) |
| `react-doctor design` | UI/디자인 룰만 집중 검사 |
| `react-doctor scan <url>` | Chrome 런타임 성능 트레이스 녹화 |
| `react-doctor why <file:line>` | 해당 위치에서 룰이 발동한 이유 설명 |
| `react-doctor install` | 코딩 에이전트에 스킬 설치 |
| `react-doctor ci install` | CI 워크플로 생성 |
| `react-doctor rules` | 룰 목록 조회 |

### 자주 쓰는 옵션

```bash
# 이번 변경이 새로 만든 문제만 검사
npx react-doctor@latest --scope changed

# 상세 출력 (기본은 상위 3개 룰만 표시)
npx react-doctor@latest --verbose

# JSON 리포트로 출력
npx react-doctor@latest --json --json-out report.json

# 외부 통신 전부 차단
npx react-doctor@latest --no-score --no-telemetry
```

`--scope` 값은 네 가지다.

- `full` — 프로젝트 전체 (기본값)
- `changed` — 변경이 **새로 도입한** 문제만 (도입 장벽이 가장 낮음)
- `files` — 변경된 파일의 모든 문제
- `lines` — 변경된 라인만

---

## 4. 플러그인인가, 스킬인가, MCP인가

**근본은 CLI 도구다.** 다만 여러 형태로 함께 배포된다.

| 형태 | 존재 여부 | 위치 |
| --- | --- | --- |
| CLI 도구 | O (본체) | `packages/react-doctor` |
| oxlint 플러그인 | O | `packages/oxlint-plugin-react-doctor` |
| ESLint 플러그인 | O | `packages/eslint-plugin-react-doctor` |
| 에이전트 스킬 | O | `skills/react-doctor/SKILL.md` |
| Cursor 플러그인 | O | `.cursor-plugin/plugin.json` |
| GitHub Action | O | `action.yml` |
| **MCP 서버** | **X** | 없음 |

### MCP 관련 주의점

레포에서 `mcp` 로 검색하면 파일이 잡히지만, 대표적인 것은
`packages/oxlint-plugin-react-doctor/src/plugin/rules/security-scan/mcp-tool-capability-risk.ts`
이다. 이는 **다른 사람이 만든 MCP 서버 코드의 보안 위험을 검사하는 룰**이지,
React Doctor 자체가 MCP 서버라는 뜻이 아니다. 방향이 반대다.

### 스킬 설치 방식

`react-doctor install` 은 `agent-install` npm 패키지를 사용해 설치된 에이전트를
자동 탐지하고 각 에이전트의 스킬 디렉터리로 복사한다. 탐지 대상은 다음과 같다.

```
PATH 바이너리: claude, codex, cursor, droid, gemini, copilot, opencode, pi
설정 디렉터리: ~/.claude, ~/.cursor, ~/.codex, ~/.gemini 등
```

`--agent-hooks` 옵션을 주면 Claude Code / Cursor 에 턴 종료 훅까지 설치된다.

---

## 5. API 토큰이 필요한가

**사용자가 준비해야 하는 토큰은 없다.** 회원가입도 없다.

점수 요청 경로는 다음과 같다.

```ts
// packages/core/src/constants.ts
export const SCORE_API_URL = "https://www.react.doctor/api/score";
```

```ts
// packages/core/src/request-score.ts
HttpClientRequest.post(requestUrl).pipe(
  HttpClientRequest.bodyUint8Array(compressedBody, "application/json"),
  HttpClientRequest.setHeader("Content-Encoding", "gzip"),
)
```

인증 헤더가 없다. gzip 압축한 본문을 익명으로 POST 하고 점수를 응답받는 구조다.

### 레포에 포함된 토큰의 정체

```ts
export const AXIOM_INGEST_TOKEN = "xaat-31b59107-855d-4917-8fab-6dc29fb459ce";
export const SENTRY_DSN = "...";
```

이 값들은 **React Doctor 팀의 텔레메트리 / 에러 수집용 자격증명**이다. 사용자가
채워 넣는 값이 아니다. `AGENTS.md` 에도 이것이 실제 credential 이며 유출 시
릴리스를 통해 교체해야 한다고 명시되어 있다.

> 포크를 개조해 배포할 계획이라면 이 값들을 반드시 비워야 한다. 그대로 두면
> 배포본 사용자의 텔레메트리가 원작자 측으로 전송된다.

### 권한이 필요한 유일한 경우

GitHub Action 사용 시에만 권한이 필요하며, GitHub가 자동 발급하는
`GITHUB_TOKEN` 으로 충분하다.

```yaml
permissions:
  pull-requests: write   # PR 코멘트
  statuses: write        # 커밋 상태에 점수 표시
```

### 전송되는 데이터

진단 결과(룰 이름, 심각도, 파일 경로)와 메타데이터(repo, sha, framework,
reactVersion, sourceFileCount, defaultBranch 등)가 전송된다. 파일 경로는
`redactSensitiveText()` 와 `scrubSensitivePaths()` 를 거친다. **코드 내용 자체는
전송되지 않는다.** 그래도 민감한 코드베이스라면 `--no-score` 사용을 권한다.

---

## 6. 왜 GitHub에서 유명한가

스타 약 14.9k, 포크 483개. 성공 요인을 정리하면 다음과 같다.

1. **문제 정의가 한 줄로 꽂힌다.** "AI가 짠 React 코드를 잡아준다"는 현재
   개발자들의 최대 관심사를 정확히 겨냥했다.
2. **팀 브랜드.** Million.js 로 이미 알려진 팀이라 초기 신뢰와 트래픽이 있었다.
3. **진입장벽이 0이다.** 설치 · 회원가입 · 토큰이 전부 불필요하고 `npx` 한 줄로
   30초 안에 결과가 나온다.
4. **점수라는 장치.** 0~100점은 비개발자도 이해하고 공유하고 싶게 만든다.
   공유 URL까지 제공되어 바이럴 구조가 설계되어 있다.
5. **레거시 공포 제거.** `--scope changed` 덕분에 기존 부채를 건드리지 않고
   팀에 도입할 수 있다.
6. **AI 생태계 편승.** 명령 한 번으로 8종 에이전트에 스킬이 설치된다.
7. **압도적 룰 커버리지.** R3F, 접근성, 상태관리 라이브러리별 대응까지 갖춰
   "내가 쓰는 스택도 지원한다"는 인상을 준다.

---

## 7. 로컬 에이전트 구축에 도움이 되는가

도움이 된다. 핵심 가치는 **결정론적 검증기(verifier)** 역할이다.

에이전트 루프에서 가장 어려운 문제는 "에이전트가 잘했는지 어떻게 판정하는가"다.
LLM 자가 평가는 신뢰하기 어렵지만, React Doctor는 같은 코드에 항상 같은 결과를
낸다. 따라서 다음과 같은 루프를 만들 수 있다.

```
에이전트가 코드 작성
  -> react-doctor --json 실행 (점수 획득)
  -> 점수 하락 시 diff + 진단 결과를 다시 에이전트에 투입
  -> 재검사하여 회복되면 통과
```

### 연동 방법

1. **JSON 파싱 (권장)**

   ```bash
   npx react-doctor@latest --json --json-compact --scope changed
   ```

   `@react-doctor/api` 는 `private: true` 라 npm에 배포되지 않는다. 프로그래밍
   방식 접근은 CLI의 `--json` 을 사용해야 한다.

2. **스킬 + 훅 설치**

   ```bash
   npx react-doctor@latest install --agent-hooks
   ```

3. **레포를 레퍼런스로 활용**

   `.agents/skills/` 에 잘 작성된 스킬 13종이 들어 있다.

   ```
   rule-research/          룰 아이디어 검증
   rule-writing/           룰 구현
   rule-validate/          머지 전 검증
   product-thinking/       공개 API 변경 체크리스트
   find-similar-functions/ 중복 코드 방지
   deslop/                 코드 정리
   ```

   또한 `AGENTS.md` 는 에이전트 컨텍스트 작성의 좋은 사례다. MUST/NEVER 로
   명령을 끊고, 이유를 함께 설명하며, 안티패턴을 구체적 코드로 제시한다.

4. **`agent-install` 패키지 발견**

   하나의 스킬을 여러 에이전트에 동시 배포해주는 도구다. 자체 스킬 배포 시
   그대로 활용할 수 있다.

---

## 8. 수익화 분석

### 8.1 원작자의 수익 구조 추정

`.cursor-plugin/plugin.json` 은 `packages/website/public/react-doctor-icon.svg`
를 참조하지만, **이 레포에 `packages/website` 는 존재하지 않는다.** 점수 API
(`https://www.react.doctor/api/score`) 역시 외부 서비스다.

즉 웹사이트와 점수 서버는 비공개이며, 전형적인 **Open Core** 전략으로 보인다.

```
[오픈소스로 공개]              [비공개 = 수익 해자]
CLI, 룰 엔진, 플러그인   <->   점수 알고리즘, 웹 대시보드,
                               데이터 축적, 팀 기능
```

따라서 **팀용 SaaS 대시보드와 점수 서버 대체는 정면충돌 영역**이므로 피하는
것이 좋다. 반대로 노릴 영역은 다음과 같다.

- 원작자가 하지 않을 것: 한국 시장, 특정 프레임워크, 교육
- 원작자가 하기 어려운 것: 사내 전용, 온프레미스, 비React 언어
- 원작자가 아직 안 한 것: MCP, GitLab 완전 지원, 점수 시계열

### 8.2 발견한 빈틈 8가지

| # | 빈틈 | 근거 |
| --- | --- | --- |
| 1 | MCP 서버 없음 | MCP는 검사 대상일 뿐 |
| 2 | GitLab 반쪽 지원 | README: "GitLab CI gets a gate-only scaffold" |
| 3 | Bitbucket / Jenkins 미지원 | `--provider` 는 github-actions, gitlab-ci 둘뿐 |
| 4 | 커스텀 룰 API 없음 | config는 기존 룰 on/off + severity 조정만 |
| 5 | 점수 이력 / 추이 없음 | 매 실행이 단발성 |
| 6 | 한국어 미지원 | 전부 영어 |
| 7 | 프로그래밍 API 막힘 | `@react-doctor/api` 가 private |
| 8 | 온프레미스 불가 | 점수는 외부 API 호출 필수 |

유리한 조건도 하나 있다. JSON 리포트에 스키마 버전이 박혀 있다.

```ts
schemaVersion: Schema.Literal(1),  // 전체 리포트
schemaVersion: Schema.Literal(2),  // baseline (PR 신규 이슈만)
schemaVersion: Schema.Literal(3),  // 최신
```

포맷이 바뀌어도 `schemaVersion` 으로 분기할 수 있어, 위에 도구를 얹기 안전하다.

### 8.3 아이디어 7가지

| # | 아이디어 | 난이도 | 기간 | 직접수익 | 간접수익 | 순위 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | MCP 서버 (`react-doctor-mcp`) | 하 | 1~2주 | 낮음 | 최상 | 1 |
| 3 | 도메인 특화 룰팩 | 상 | 2~3달 | 최상 | 중 | 2 |
| 2 | GitLab / Bitbucket 브릿지 | 중 | 3~6주 | 높음 | 높음 | 3 |
| 4 | 한국어 레이어 + 교육 | 중하 | 1~2달 | 중상 | 높음 | 4 |
| 6 | 코드 진단 컨설팅 | 영업력 의존 | 즉시 | 최상 | 낮음 | 5 |
| 5 | 점수 시계열 대시보드 | 중 | 1~2달 | 중 | 중 | 6 |
| 7 | 타 생태계 포팅 | 최상 | 3~6달 | 최상 | 최상 | 7 |

#### 1. MCP 서버

CLI를 감싸 MCP 툴로 노출한다. `scan_project`, `explain_diagnostic`,
`list_rules`, `compare_score` 정도면 충분하다. 구현 자체는 `--json` 출력 파싱이
전부라 부담이 작다. 직접 수익은 낮지만, 공백 시장 선점을 통한 인지도 확보
효과가 가장 크다. 리스크는 원작자가 먼저 만드는 경우이므로 속도가 중요하다.

#### 2. GitLab / Bitbucket 브릿지

GitLab에서는 게이트만 동작하고 MR 코멘트가 없다. 그런데 React Doctor의 핵심
매력이 PR 코멘트다. 보안 문제로 자체 호스팅 GitLab을 쓰는 국내 기업(대기업,
금융, 공공, SI)이 그대로 비어 있는 시장이다. 구축 용역 및 온프레미스 설치 지원을
상품화할 수 있다.

#### 3. 도메인 특화 룰팩

config는 기존 룰의 on/off와 severity 조정만 지원하며, `extends` 는 oxlint JSON
설정만 받는다. 즉 커스텀 룰은 직접 oxlint 플러그인으로 작성해야 하고, 그
진입장벽 자체가 해자가 된다. 후보는 한국 이커머스(PG/본인인증), 금융 프론트엔드,
사내 디자인 시스템 강제, 한국 웹접근성 인증(KWCAG) 대응 등이다.

학습 자료는 레포 안에 이미 있다.

```
docs/HOW_TO_WRITE_A_RULE.md
.agents/skills/rule-research/
.agents/skills/rule-writing/
.agents/skills/rule-validate/
packages/fuzz/
```

가장 큰 리스크는 오탐이다. `packages/fuzz` 를 활용한 검증이 필수이며, 초기에는
룰 5개 수준으로 좁게 시작하는 편이 안전하다.

#### 4. 한국어 레이어 + 교육

현재 진단 출력은 영어 룰 이름과 짧은 설명뿐이라 주니어가 해석하기 어렵다.
룰별 한국어 설명(왜 문제인지 / 어떻게 고치는지 / 예제 코드)을 붙이면 부트캠프
교보재, 사내 교육, 온라인 강의로 연결할 수 있다. 기술 난이도는 낮고 콘텐츠
작업량이 핵심이다. AI로 초벌 작업 후 감수하는 방식이 효율적이다.

#### 5. 점수 시계열 대시보드

CI에서 매일 스캔한 JSON을 저장해 점수 추이, 파일별 히트맵, 주간 리포트를
제공한다. 이 대시보드 UI가 React를 쓰기 좋은 지점이다. 다만 원작자가 가장 하고
싶어 하는 영역이므로 SaaS가 아닌 **온프레미스 전용**으로 포지셔닝하는 것이
충돌을 피하는 방법이다.

#### 6. 코드 진단 컨설팅

"현재 42점 -> 3개월 후 78점"처럼 효과를 숫자로 증명할 수 있다는 점이 강점이다.
무료 진단(미끼) -> 정밀 리포트 -> 개선 실행 -> CI 정착 및 교육 -> 월간 모니터링
순으로 단계화할 수 있다. 기술 난이도는 낮고 영업 난이도가 절대적으로 높다.

#### 7. 타 생태계 포팅

React Doctor의 진짜 발명은 분석 기술이 아니라 다음 세 가지 포장이며, 이는 언어와
무관하다.

1. 0~100점 점수화
2. "새로 생긴 문제만" PR 코멘트
3. AI 에이전트 스킬 설치

따라서 PHP로 간다면 분석기를 새로 만들 것이 아니라 **PHPStan / Psalm 을 감싸고
위 3종 세트를 얹는 것**이 현실적이다. 대상 후보 중에서는 Flutter(`dart analyze`)
가 경쟁이 가장 적다.

### 8.4 라이선스 및 주의사항

MIT 라이선스이므로 상업적 이용, 수정, 재배포, 유료 서비스 사용이 모두 가능하다.
조건은 LICENSE의 저작권 표시 유지다.

다만 다음은 주의해야 한다.

| 행위 | 판정 | 사유 |
| --- | --- | --- |
| "React Doctor"를 자체 제품명으로 사용 | 위험 | 상표권 소지 |
| "React Doctor용 한국어팩" 같은 호환 표기 | 안전 | 호환성 명시 |
| 로고 그대로 사용 | 불가 | Million Software 자산 |
| 포크 후 이름만 바꿔 유료 SaaS | 합법이나 비권장 | 평판 리스크 |
| 하드코딩된 토큰 재사용 | 금지 | 타인의 실제 credential |

원작자와 경쟁하기보다 협력하는 편이 유리하다. 만든 도구의 등재를 요청하거나,
범용적인 룰을 본가에 기여해 컨트리뷰터 지위를 얻는 경로가 성장에 효율적이다.

### 8.5 90일 로드맵

**Day 1~14 — 선점**

- 자체 프로젝트에 React Doctor 실행해 감각 확보
- MCP 서버 뼈대 구현 및 공개
- 원본 레포 이슈에 공유

**Day 15~45 — 인지도**

- MCP에 한국어 진단 설명 추가(차별점)
- 기술 블로그 3편 (도구 소개 / MCP 제작기 / 점수 개선 사례)
- 룰 한국어 설명 50개 작성

**Day 46~90 — 수익화**

- 영업 지향이면 컨설팅, 개발 지향이면 GitLab 브릿지, 교육 지향이면 교보재 중
  하나를 선택
- 첫 고객 1건 확보(무상이어도 레퍼런스 확보가 목적)
- 사례 정리 후 두 번째 고객부터 유상 전환

### 8.6 결론

권장 조합은 **1번(MCP) -> 4번(한국어/교육) -> 6번(컨설팅)** 순서다.

무명 상태에서는 컨설팅이 팔리지 않는다. MCP로 전문성을 증명하고, 교육으로 작은
매출과 레퍼런스를 만든 뒤, 컨설팅으로 규모를 키우는 순서가 현실적이다.

오픈소스 래퍼 자체로 직접 수익이 발생하는 경우는 드물다. 실질적 가치는
**"React 코드 품질 전문가"라는 증명**이며, 그 증명이 3 · 4 · 6번을 팔 수 있게
만든다. 도구를 파는 것이 아니라 도구로 증명한 역량을 파는 구조다.

---

## 부록: 자주 쓸 명령어 모음

```bash
# 전체 진단
npx react-doctor@latest

# 변경분만 진단 (커밋 전 권장)
npx react-doctor@latest --scope changed --verbose

# JSON 리포트 저장
npx react-doctor@latest --json --json-out report.json

# 특정 위치의 룰 발동 이유 확인
npx react-doctor@latest why src/App.tsx:42

# 에이전트에 스킬 + 훅 설치
npx react-doctor@latest install --agent-hooks

# CI 워크플로 생성
npx react-doctor@latest ci install

# 런타임 성능 트레이스
npx react-doctor@latest scan http://localhost:3000

# 외부 통신 차단 모드
npx react-doctor@latest --no-score --no-telemetry
```
