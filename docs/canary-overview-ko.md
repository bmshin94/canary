# Canary 전수조사 & 활용 정리 (한국어)

> 이 문서는 Canary 저장소를 전수조사하여 **"무엇인지 / 언제 쓰는지 / 어떤 도움이 되는지"**를
> 정리한 한국어 가이드입니다. 설치·사용법, 플러그인/스킬/MCP 구분, API 토큰 필요 여부,
> AI 에이전트 구축 활용, 수익화 아이디어, React/PHP 구현 가능성, 유튜브 콘텐츠화까지 포함합니다.

## 🔗 GitHub 주소

| 구분 | 주소 |
| --- | --- |
| 업스트림 원본 | https://github.com/wizenheimer/canary |
| 이 저장소 (포크) | https://github.com/bmshin94/canary |
| 이슈 트래커 | https://github.com/wizenheimer/canary/issues |
| npm — CLI | https://www.npmjs.com/package/@usecanary/cli |
| npm — 엔진 | https://www.npmjs.com/package/@usecanary/browser |
| npm — 뷰어 | https://www.npmjs.com/package/@usecanary/ui |
| npm — 설치 위저드 | https://www.npmjs.com/package/create-canary |
| Playwright API 문서 | https://playwright.dev/docs/api/class-page |

- 기준 버전: **v0.4.4**
- 라이선스: **MIT** (일부 Sawyer Hood 의 MIT 저작물에서 파생 — `LICENSE` 참조)

---

## 1. 한 줄 정의

> **Canary** = Claude Code 같은 AI 코딩 에이전트가 **직접 실제 브라우저를 몰아 QA를 수행하고,
> 그 결과를 재현 가능한 증거(리포트 + Playwright 스크립트)로 남기는 QA 하니스.**

기존 도구는 둘 중 하나만 제공했습니다.

- ❌ AI 테스트 도구 → 돌려주긴 하지만 **재현 불가능한 블랙박스**
- ❌ 생 Playwright → 재현은 되지만 **사람이 작성·유지보수하는 노가다**

Canary는 **둘 다** 줍니다. 에이전트가 QA를 수행하고, 그 결과로 재사용 가능한 스크립트를 돌려줍니다.

---

## 2. 저장소 구조 (전수조사)

```
canary/
├── apps/                      # 실행 프로그램 5개
│   ├── canary/                # @usecanary/cli      bin: canary         — 세션 오케스트레이터 (메인)
│   ├── canary-browser/        # @usecanary/browser  bin: canary-browser — 단발성 자동화 엔진
│   ├── canary-daemon/         # @usecanary/daemon   (bin 없음)          — Playwright + QuickJS 런타임
│   ├── canary-ui/             # @usecanary/ui       bin: canary-viewer  — 로컬 세션 뷰어 (Astro + React)
│   └── create-canary/         # create-canary       bin: create-canary  — `npm create canary` 위저드 (Ink)
├── packages/                  # 내부 공용 패키지 5개
│   ├── protocol/              # @usecanary/protocol      Zod IPC 스키마 (단일 진실 공급원)
│   ├── config/                # @usecanary/config        공용 tsconfig 베이스
│   ├── logger/                # @usecanary/logger        pino 기반 구조화 로거
│   ├── cli-kit/               # @usecanary/cli-kit       CLI 공용 헬퍼
│   └── daemon-client/         # @usecanary/daemon-client 데몬 전송/수명주기 + 데몬 번들 임베딩
├── skills/                    # 에이전트 스킬 5개
├── agents/                    # JTBD 서브에이전트 4개
├── commands/                  # 슬래시 커맨드 4개
├── .claude-plugin/            # Claude Code 플러그인 + 마켓플레이스 매니페스트
├── .cursor-plugin/            # Cursor 플러그인 매니페스트 (rules/ 와 짝)
├── plugins/canary/            # Codex 플러그인 래퍼 (skills 는 ../../skills 심볼릭 링크)
├── .agents/                   # Codex / agents 마켓플레이스 매니페스트
├── rules/                     # Cursor 룰 (canary-workflows.mdc)
├── docs/snippets/             # LLM 문서 단일 소스 (stitch-docs.mjs 가 주입)
├── docs/media/                # 스크린샷 8장 + preview.mp4
├── examples/                  # 데모 4개 (Hacker News, Product Hunt, GitHub Trending, Wikipedia)
└── .github/                   # CI
```

### 기술 스택

pnpm 9.15.0 + Turborepo + TypeScript + esbuild + **Playwright 1.58.2** + **quickjs-emscripten** +
Zod + Astro/React + Biome(Ultracite) + pino. Node 20+ (뷰어는 22.12+).
포크된 Playwright 타입 정의를 제외한 순수 코드 **약 18,000줄**.

---

## 3. 작동 원리

```
사용자 / 에이전트          상주 데몬                      실제 브라우저
┌────────────────┐      ┌──────────────────────┐      ┌─────────────┐
│ canary run     │─RPC─▶│ canary-daemon         │─────▶│  Chromium   │
│ canary-browser │ (라인 │ Playwright 호스트      │      │ (Playwright)│
└────────────────┘ JSON) │ + QuickJS WASM 샌드박스│      └─────────────┘
                   소켓 / └──────────────────────┘
                   named pipe
```

### 핵심 설계 3가지

1. **QuickJS WASM 샌드박스** — 에이전트가 생성한 스크립트를 Node 가 아닌 격리된 WASM 안에서 실행.
   `require` / `import()` / `process` / `fs` / `path` / `os` / `fetch` / `WebSocket` /
   `__dirname` 전부 없음. 메모리·CPU 시간·wall-clock 제한이 강제되어 무한 루프나
   영원히 settle 되지 않는 프로미스는 중단됩니다. 그 안에서 **Playwright `Page` API 전체**를
   사용할 수 있는 것이 핵심 아이디어입니다. `evaluate` / `$eval` 경계를 넘는 값은
   JSON 직렬화 가능해야 합니다.
2. **세션 = 스텝의 모음** — `session start` → `run --step <이름>` (여러 번) → `session end`.
   `browser.getPage("main")` 처럼 **이름 붙인 페이지는 스텝 사이에 생존**하므로 로그인 상태가 유지됩니다.
3. **증거 자동 수집** — Playwright trace(`trace.zip`), 비디오, 네트워크 HAR, 콘솔 로그,
   스텝별 스크린샷, 기계가 읽는 `results.json`, 그리고 **자립형 `report.html`** 이
   `~/.canary/sessions/<id>/` 아래에 저장됩니다.
   `--no-trace` / `--no-video` / `--no-har` / `--no-console` 로 개별 비활성화 가능 (기본은 전부 ON).

### 빌드 흐름 (`turbo run build`, `^build` 기준 topo-sort)

1. `@usecanary/protocol` + `@usecanary/config` + `@usecanary/logger` (빌드 없음, 소스 배포)
2. `@usecanary/daemon` → `dist/daemon.bundle.mjs` + `dist/sandbox-client.js`
3. `@usecanary/browser` + `@usecanary/cli` 가 데몬 번들을 임베딩 후 esbuild 로 번들,
   `@usecanary/ui` 는 Astro node standalone (`vite.ssr.noExternal` 로 런타임 `node_modules` 없이 배포)

---

## 4. 언제 쓰는가

| 상황 | 명령 / 스킬 |
| --- | --- |
| "코드 바꿨는데 뭘 테스트해야 해?" | `/canary:verify` — git diff → P0/P1/P2 QA 플랜 |
| "체크아웃 플로우 QA하고 리포트 줘" | `/canary:session` — 녹화 + `report.html` 생성 |
| "이 페이지 긁어와 / 스크린샷 찍어줘" | `/canary:run` — 단발성, 녹화 없음 |
| "지난 실행에서 뭐가 깨졌어?" | `/canary:review` — 뷰어 열고 트리아지 |

### 대상별 가치

| 역할 | 기존 | Canary |
| --- | --- | --- |
| 개발자 | Playwright/E2E 를 손으로 작성·유지보수 | 매 실행에서 재사용 스크립트 확보 → CI 재실행 시 추론 비용 0 |
| QA 엔지니어 | 수동 클릭으로 재현/검증 | 기본 제공 증거 — trace, 비디오, 네트워크, 콘솔, 스텝별 스크린샷 |
| PM / 리뷰어 | 빌드 대기 또는 "제 컴퓨터에선 됩니다" | 열어서 읽는 자립형 `report.html` |

### 핵심 경제성: "한 번만 비싸고, 그 다음은 공짜"

```
1회차:  AI 가 페이지 탐색 → 플로우 발견              → 💸 Claude Code 토큰 소모
        ↓ Canary 가 Playwright 스크립트로 저장
2회차~: 저장된 스크립트 재실행                        → 💰 AI 비용 0원
        CI(GitHub Actions)에 넣으면 커밋마다 자동 QA   → 💰 0원
```

---

## 5. 설치 및 사용법

### 방법 A — 가이드 위저드 (권장)

```bash
npm create canary@latest
```

### 방법 B — 수동 설치

```bash
npm i -g @usecanary/cli @usecanary/ui   # canary + canary-viewer 를 PATH 에
canary install                          # Chromium + 런타임을 ~/.canary 에 (~150MB)
```

`canary install` 은 반복 실행이 안전합니다 (새 CLI 가 핀한 버전을 가져옴).

### 녹화 세션 기본 흐름

```bash
id=$(canary session start --name "checkout")
canary run ./open.js   --session "$id" --step open
canary run ./submit.js --session "$id" --step submit
canary session end "$id"                # -> ~/.canary/sessions/<id>/report.html

canary-viewer                           # 녹화된 모든 세션 브라우징
canary stop                             # 백그라운드 데몬 종료
```

### 단발성 (녹화 없음)

```bash
echo 'const p = await browser.getPage("main");
await p.goto("https://example.com");
console.log(await p.title());' | canary-browser
```

### 이미 열려 있는 Chrome 에 붙기 (로그인 세션 재활용)

```bash
# Chrome 을 --remote-debugging-port=9222 로 띄운 뒤
canary-browser --connect http://localhost:9222 <<'EOF'
const page = await browser.getPage("main");
console.log(await page.title());
EOF
```

### 전역 설치 없이 (npx)

```bash
npx @usecanary/cli session start …
npx @usecanary/ui
```

### 에이전트 플러그인 설치

```bash
# Claude Code
/plugin marketplace add wizenheimer/canary
/plugin install canary@canary-marketplace

# Cursor — 마켓플레이스에서 "canary" 설치, 또는 로컬 개발용 심볼릭 링크
ln -sfn "$(pwd)" ~/.cursor/plugins/local/canary

# Codex
codex marketplace add wizenheimer/canary        # 이후 /plugins → canary 설치
```

사용 가능한 슬래시 커맨드:

```
/canary:verify    # 무엇이 바뀌었나? → 우선순위 QA 플랜 → 녹화
/canary:session   # 플로우를 end-to-end 녹화하고 report.html 렌더
/canary:run       # 브라우저 한 번 몰기, 녹화 없음
/canary:review    # 뷰어를 열고 녹화 세션 트리아지
```

슬래시를 쓰지 않고 *"체크아웃 플로우 QA하고 리포트 줘"* 라고 말해도 서브에이전트가 처리합니다.

### 업데이트

```bash
npm i -g @usecanary/cli@latest @usecanary/ui@latest
canary install                                   # 런타임 갱신

/plugin marketplace update canary-marketplace    # Claude Code 카탈로그 갱신
# 또는 /plugin → Marketplaces → canary-marketplace → Enable auto-update
# (서드파티 마켓플레이스는 기본 auto-update OFF)
```

### 이 저장소에서 개발할 때

```bash
make install   # pnpm install
make build     # topo order 전체 빌드
make test      # 전체 테스트
make check     # 컴파일 + 린트 + 테스트 (CI 와 동일)
make docs      # docs/snippets/ → README/skills/CLI help 주입
```

주의 사항:

- `packages/cli-kit/src/snippets.generated.ts` 와 `<!-- canary:snippet … -->` 마커 사이 내용은
  **직접 편집 금지** — 스니펫을 고치고 `make docs` 를 실행하세요. CI 가 drift 를 검출합니다.
- 커밋은 **Conventional Commits** 강제 (commitlint + husky `commit-msg`).
- 린트/포맷은 **Ultracite(Biome)** — ESLint/Prettier 재도입 금지.
- 앱 코드에서 `console.*` 금지 (`noConsole` 이 error). `@usecanary/logger` 사용.
  기계가 읽는 CLI 출력만 `process.stdout`.
- `apps/canary-daemon/src/sandbox/forked-client/` (벤더링된 Playwright 포크)는 린트 제외 —
  업스트림과 diff 가능하게 유지.

### 환경 변수 / 옵션

| 항목 | 효과 |
| --- | --- |
| `CANARY_LOG_LEVEL` | trace \| debug \| info \| warn \| error \| silent |
| `--verbose` / `-v` | CLI 상세 로그 (stderr) |
| `--no-trace` / `--no-video` / `--no-har` / `--no-console` | 특정 캡처 비활성화 |
| `--stop-daemon` (`session end` 에) | 더 쓰는 곳이 없으면 데몬까지 정리 |
| `canary ui --dir <path>` | 기본값(`~/.canary/sessions`)이 아닌 폴더 열기 |
| `CANARY_UI_SERVER` / `HOST` / `PORT` / `CANARY_UI_ROOT` | 뷰어 서버 설정 |

데몬 로그는 `~/.canary/daemon.log`, CLI 로그는 stderr 입니다.

---

## 6. 스크립팅 API 요약

스크립트는 top-level `await` 을 쓰는 평범한 async JavaScript 입니다.

```js
const page = await browser.getPage("main");          // 이름 붙인 영속 페이지
await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
console.log(await page.title());

const headings = await page.evaluate(() =>
  [...document.querySelectorAll("h1, h2")].map((h) => h.textContent.trim())
);
console.log(JSON.stringify(headings));

await page.locator("a.more").click();
const buf = await page.screenshot({ fullPage: false });
await saveScreenshot(buf, "page.png");               // saveScreenshot(buffer, name)
```

### browser

- `browser.getPage(nameOrId)` — 이름 붙인 페이지를 get-or-create, 또는 `listPages()` 의 `id` 로 기존 탭에 attach.
  같은 이름으로 다시 호출하면 **세션 내 스텝 간에 탭이 재사용**됩니다.
- `browser.newPage()` — 익명 페이지. 스크립트 종료 시 자동 닫힘, 영속되지 않음.
- `browser.listPages()` — 열린 탭 목록 `[{ id, url, title, name }]` (이름 없는 탭은 `name: null`).
- `browser.closePage(name)` — 이름 붙인 페이지를 닫고 잊음.

### 파일 헬퍼 (모두 async, `~/.canary/tmp/` 로 샌드박싱)

- `saveScreenshot(buffer, name)` — 스크린샷 버퍼 저장 (버퍼가 첫 번째 인자)
- `writeFile(name, data)` — 작은 파일 쓰기 (스텝 간 상태 전달용)
- `readFile(name)` — 문자열로 읽기

### 출력

- `console.log` / `console.info` → stdout, `console.warn` / `console.error` → stderr
- `page.evaluate(() => …)` 안의 `console.log` 는 페이지에서 실행되어 **세션 콘솔 아티팩트**로 캡처됨

### 페이지 탐색

`await page.snapshotForAI()` 가 LLM 친화적인 페이지 아웃라인을 반환합니다 —
모르는 페이지를 다룰 때 먼저 관찰(observe-first)하는 용도.
전체 API 는 `canary-scripting` 스킬과 `skills/canary-scripting/references/REFERENCE.md` 에 있습니다.

`browser.getPage()` / `browser.newPage()` 가 반환하는 것은 **완전한 Playwright Page 객체**입니다
(`goto`, `click`, `fill`, `locator`, `evaluate`, `getByRole`, `waitForSelector`, …).

---

## 7. 플러그인? 스킬? MCP?

### 결론: **CLI 도구(본체) + 플러그인(포장) + 스킬(내용물)** — MCP 는 아님.

| 질문 | 답 | 근거 |
| --- | --- | --- |
| 플러그인인가? | ✅ 맞다 | `.claude-plugin/plugin.json` + `marketplace.json`, Cursor·Codex 매니페스트도 존재 |
| 스킬인가? | ✅ 맞다 | `skills/` 아래 `SKILL.md` 5개 (frontmatter: `name`, `description`) |
| MCP 인가? | ❌ 아니다 | MCP 서버 구현이 전혀 없음. 통신은 자체 line-delimited JSON RPC (named pipe / Unix socket) |

### 3겹 구조

```
3겹: 플러그인 포장        .claude-plugin/ · .cursor-plugin/ · .agents/ · plugins/canary/
2겹: 스킬 + 서브에이전트 + 슬래시 커맨드   skills/ · agents/ · commands/ · rules/
1겹: 실제 본체 = npm CLI 3개            canary · canary-browser · canary-viewer
     ↳ 플러그인 없이도 단독으로 완전히 동작
```

README 가 직접 밝히고 있습니다 — *"Install it, then tell your agent to run `canary --help`.
Each output is a complete, self-contained usage guide, written for an LLM to read. **No plugin required.**"*

즉 플러그인/스킬은 **편의 레이어**입니다. 에이전트는 `--help` 만 읽어도 Canary 를 쓸 수 있습니다.

### 왜 MCP 가 아니라 CLI 인가

| | MCP 방식 | Canary 의 CLI 방식 |
| --- | --- | --- |
| 연결 | 서버-클라이언트 상주 | 명령 실행 |
| 사용 가능 범위 | MCP 지원 클라이언트 | **모든 에이전트 + 쉘 + CI** |
| 토큰 비용 | 툴 스키마가 컨텍스트에 상주 | 필요할 때 `--help` 만 |
| CI 통합 | 어려움 | 자연스러움 (그냥 명령) |

CI 재실행이 핵심 가치이므로 CLI 가 올바른 선택이었습니다.
다만 **CLI 를 감싸는 MCP 서버를 따로 만드는 것은 충분히 가능**합니다 (수익화 아이디어 4 참조).

### 스킬 ↔ 에이전트 ↔ 커맨드 매핑

| 스킬 | 역할 | 서브에이전트 | 슬래시 커맨드 |
| --- | --- | --- | --- |
| `canary-scripting` | 샌드박스 API 레퍼런스 (+ `references/REFERENCE.md`) | — | — |
| `canary-verify` | diff → 우선순위 QA 플랜 (P0/P1/P2) | `verify-agent` | `/canary:verify` |
| `canary-automate` | 단발성 자동화 / 스크래핑 | `automate-agent` | `/canary:run` |
| `canary-session` | 녹화 QA 세션 + 리포트 | `session-agent` | `/canary:session` |
| `canary-review` | 세션 트리아지 / 뷰어 | `review-agent` | `/canary:review` |

`skills/` 는 Claude Code·Cursor·Codex 세 곳에서 **그대로(verbatim)** 소비됩니다
(`plugins/canary/skills` 는 `skills/` 로의 심볼릭 링크).

---

## 8. API 토큰이 필요한가?

### **Canary 자체는 API 토큰이 전혀 필요하지 않습니다.**

저장소 전체를 `ANTHROPIC_API_KEY|api[_-]?key|OPENAI|token` 으로 검색한 결과,
Canary 소스에 **API 키를 읽거나 전송하는 코드는 없습니다.**
검색에 걸린 항목은 벤더링된 Playwright 타입 정의의 브라우저 Trust Token 등 무관한 것들뿐이었습니다.

```
┌──────────────────────────────────────────────────────┐
│  사용자의 Claude Code 구독 / API 키   ← 💰 여기만 과금   │
│          ↓ (스크립트 작성 지시)                        │
│  Canary CLI                          ← 🆓 무료         │
│          ↓                                            │
│  로컬 Chromium                        ← 🆓 무료         │
└──────────────────────────────────────────────────────┘
```

Canary 는 **"AI 가 사용하는 도구"**이며 스스로 LLM 을 호출하지 않습니다.
두뇌는 Claude Code, Canary 는 손과 눈입니다.

| 항목 | 상태 |
| --- | --- |
| 클라우드 전송 | 없음 (전부 `~/.canary/` 로컬) |
| 계정 / 회원가입 | 없음 |
| 텔레메트리 | 없음 |
| 네트워크 사용 | `canary install` 시 Chromium 다운로드만 |
| 라이선스 | MIT (상업적 이용 가능) |

실제 비용: 1회차 탐색은 Claude Code 토큰 소모, **2회차 이후 스크립트 재실행은 0원**,
CI 실행도 GitHub Actions 분만 소모됩니다.

---

## 9. AI 에이전트 구축에 도움이 되는가

### 결론: **두 가지 방향으로 매우 도움이 됩니다.**

### 방향 1 — 에이전트의 "손발"로 바로 사용

| 어려운 문제 | Canary 의 해법 |
| --- | --- |
| AI 가 생성한 코드가 위험함 | QuickJS WASM 샌드박스 (호스트 접근 0) |
| AI 가 페이지 구조를 모름 | `page.snapshotForAI()` — LLM 친화적 페이지 아웃라인 |
| 로그인 상태가 유지되지 않음 | 이름 붙인 페이지가 스텝 간 생존 |
| AI 가 무엇을 했는지 알 수 없음 | trace / video / HAR / console 전부 캡처 |
| 스텝 간 데이터 전달 | `writeFile` / `readFile` 샌드박스 파일 헬퍼 |
| 무한 루프 폭주 | CPU + wall-clock 시간 제한 강제 |

직접 구현하면 수개월 걸리는 영역이며, MIT 라이선스라 그대로 활용할 수 있습니다.

### 방향 2 — "에이전트 친화 설계"의 레퍼런스로 학습

1. **`--help` 를 LLM 용 문서로 사용** — 플러그인 없이도 에이전트가 자급자족. "도구가 스스로를 설명한다."
2. **문서 단일 소스 + CI drift 검사** — `docs/snippets/` → `stitch-docs.mjs` → README/스킬/CLI help.
   LLM 용 문서가 여러 곳에 흩어져 썩는 문제의 해법.
3. **Zod 프로토콜 단일 진실 공급원** — 데몬은 검증, CLI 는 타입 추론. 에이전트 입력은 런타임 검증 필수.
4. **JTBD 3층 구조** — 스킬(지식) + 서브에이전트(실행 주체) + 슬래시 커맨드(진입점) 세트 대응.
5. **"탐색은 비싸게, 재생은 싸게"** — 에이전트 경제학의 핵심 패턴.
6. **권한 분리** — Node 데몬(권한 있음) ↔ WASM 샌드박스(권한 없음).

### 이 위에 만들 수 있는 것

- 자율 웹 리서치 에이전트 (브라우저 조작 레이어로 Canary)
- 경쟁사 가격 모니터링 봇 (스케줄 + 세션 녹화로 증거 확보)
- 폼 자동 제출 에이전트 (로그인 세션 유지)
- 접근성(a11y) 감사 에이전트 (`snapshotForAI` + 캡처)
- 비주얼 리그레션 봇 (스텝별 스크린샷)
- **자체 치유(self-healing) E2E 테스트** — 깨지면 AI 가 재탐색하여 스크립트 갱신

---

## 10. React / PHP 로 만들 수 있는가

### React — 가능. 이미 쓰이고 있음.

`apps/canary-ui` 가 이미 **Astro + React islands** 입니다 (`Library`, `SessionView` 가 client-only React).

| 만들 것 | 난이도 | 설명 |
| --- | --- | --- |
| 커스텀 리포트 뷰어 | 쉬움 | `results.json` 읽어 렌더 |
| 팀 대시보드 (웹) | 보통 | 세션 목록 + 통계 + 비교 |
| Next.js 기반 Canary Cloud | 높음 | 업로드 + 공유 링크 + 권한 |
| 스크립트 에디터 (Monaco) | 보통 | 샌드박스 API 자동완성 |
| 리포트 diff 뷰 | 높음 | 두 세션 스크린샷 비교 |

```
┌────────────────────────────┐
│  React/Next.js 대시보드      │  ← 직접 구현 (업로드/공유/통계/비교)
└──────────┬─────────────────┘
           │ results.json 읽기
┌──────────▼─────────────────┐
│  Canary CLI (그대로 사용)    │  ← MIT, 수정 불필요
└────────────────────────────┘
```

### PHP — 절반만 가능.

**가능 (껍데기 / 서버 레이어)**

- 리포트 업로드/저장 API
- `results.json` 파싱 → Laravel/Blade 대시보드
- 사용자 인증, 결제(Cashier), 팀 권한
- Laravel Queue 에서 `exec('canary run …')` 오케스트레이션
- Filament 로 어드민 패널 빠르게 구성

**사실상 불가능 (엔진 레이어)**

| 못 하는 이유 | 설명 |
| --- | --- |
| Playwright 가 Node 전용 | PHP 공식 바인딩 없음 (Puppeteer-PHP 는 거의 방치) |
| QuickJS WASM 샌드박스 | PHP 의 WASM 런타임 생태계가 빈약 |
| 상주 데몬 + 소켓 RPC | PHP 는 요청-응답 모델에 최적화 |
| 비동기 Playwright API | PHP async 는 ReactPHP/Swoole 필요, 비용 큼 |

### 권장 조합

```
1순위: React/Next.js (프론트 + 백) + Canary CLI 그대로   → 단일 언어, 가장 빠름
2순위: Laravel (웹/인증/결제/DB) + Node 마이크로서비스(Canary 실행), HTTP/큐 연결
비추천: PHP 로 Playwright 재구현
```

> **Canary 자체를 다시 만들 필요가 없습니다.** MIT 이므로 그대로 쓰고,
> 그 위에 올라가는 제품(대시보드, 공유, 팀 기능, CI 통합)을 만드는 것이 훨씬 효율적입니다.

---

## 11. 유튜브 강의 제작 가능성

### 결론: 가능성이 높고, 타이밍이 좋습니다.

| 이유 | 설명 |
| --- | --- |
| 신선함 | "AI 가 직접 브라우저 QA" 주제의 한국어 콘텐츠가 거의 없음 |
| 검색 키워드 | "Claude Code", "Playwright", "AI 테스트 자동화" 관심 상승 |
| 시각적 소재 | 브라우저가 자동으로 움직이는 화면 = 영상에 최적, 썸네일도 잘 나옴 |
| 실용성 | 개발자의 실제 고통(E2E 유지보수)을 해결 |
| 타겟 | 프론트엔드 + QA + 테크리드 = 광고 단가 높은 시청층 |
| 진입장벽 | MIT + 토큰 불필요 → 시청자가 즉시 따라할 수 있음 |

### 추천 커리큘럼 (8편)

| # | 제목 | 길이 | 후킹 포인트 |
| --- | --- | --- | --- |
| 1 | AI 가 알아서 QA 해주는 시대 (Canary 소개) | 8분 | 브라우저 자동 조작 raw 영상 |
| 2 | 5분 설치 + 첫 세션 녹화 | 10분 | `report.html` 을 처음 열어보는 순간 |
| 3 | `/canary:verify` — git diff 만 보고 테스트 플랜 짜기 | 12분 | diff → P0/P1/P2 |
| 4 | Playwright 테스트를 다시 안 써도 되는 이유 | 15분 | 재사용 스크립트 → CI |
| 5 | QuickJS WASM 샌드박스 — AI 코드를 안전하게 돌리는 법 | 18분 | 기술 깊이 / 신뢰도 |
| 6 | Cursor / Codex 에도 붙이기 | 10분 | 멀티 에이전트 |
| 7 | GitHub Actions 로 커밋마다 자동 QA | 15분 | 실전 CI |
| 8 | Canary 소스 해부 — 에이전트 도구 설계 패턴 | 25분 | 고급 시청자 타겟 |

### 제작 팁

**활용할 강점**

1. 화면이 저절로 움직여 지루할 틈이 없음
2. 저장소에 이미 `docs/media/` 스크린샷 8장 + `preview.mp4` 존재 → B롤로 활용
3. 예제 4개(Hacker News, Product Hunt, GitHub Trending, Wikipedia)가 공개 사이트라 소재 부담 적음
4. before/after 대비 연출 — 수동 클릭 10분 vs 명령 1줄 (타임랩스 편집)

**주의할 점**

1. 실제 로그인 데모 시 비밀번호·토큰 블러 처리 필수
2. 영상 설명란에 기준 버전(`v0.4.4`) 명시 — 빠르게 변하는 프로젝트
3. MIT 라이선스 및 원작자(`wizenheimer`, 일부 Sawyer Hood 파생) 크레딧 표기
4. `canary install` 이 약 150MB 다운로드이므로 녹화 전에 미리 설치
5. 원작자에게 영상 제작을 알리면 공유·확산 효과 기대

### 수익 연결

```
유튜브 (무료, 트래픽)
   ├─→ 애드센스 (개발 콘텐츠 CPM 높음)
   ├─→ 유료 강의 (인프런 / 유데미 / 클래스101)
   ├─→ 뉴스레터 → 템플릿·스크립트 팩 판매
   ├─→ 기업 교육 / 컨설팅 문의 (단가 최상)
   └─→ 자체 SaaS 유입 (아래 수익화 참조)
```

---

## 12. 수익화 아이디어

```
                      투입 노력
                   낮음 ────────▶ 높음
         높음  ┌─────────────┬─────────────┐
           ▲  │ 유튜브+강의   │ Canary      │
           │  │ 컨설팅       │ Cloud       │
         수익 │             │ 셀프힐링     │
         잠재 ├─────────────┼─────────────┤
           │  │ MCP 래퍼     │ 통합 앱      │
           ▼  │ 템플릿 팩    │             │
         낮음 └─────────────┴─────────────┘
```

### 아이디어 1 — Canary Cloud (호스팅 SaaS)

Canary 는 전부 로컬입니다. 장점이지만 팀 환경에서는 단점이 됩니다 —
리포트 공유가 파일 첨부, 과거 추이 확인 불가, CI 결과를 PM 이 볼 방법 없음, 러너를 각자 관리.
**이 간극(호스팅 + 공유 + 분석)을 판매합니다.**

| 기능 | 티어 |
| --- | --- |
| 공유 링크 (`canary push` → `canary.cloud/r/abc123`) | Free |
| 팀 대시보드 (통과율 추이, 플레이키 테스트 감지) | Pro |
| 호스팅 러너 (설치 불필요) | Pro |
| 실패 알림 (Slack / Discord / 이메일) | Pro |
| 비주얼 diff (커밋 간 스크린샷 비교) | Pro |
| SSO / 감사 로그 | Enterprise |
| 화이트라벨 (에이전시용) | Enterprise |

```
Free        $0      세션 20개/월, 7일 보관, 공유 링크
Pro         $29/월  무제한 세션, 90일 보관, 러너 500분, 알림, 비주얼 diff
Team        $99/월  5시트, 1년 보관, 러너 2000분, 역할 권한
Enterprise  문의    SSO, 온프레미스, 감사 로그, 화이트라벨, SLA
```

스택: Next.js 15 + React, Tailwind + shadcn/ui, Postgres(Supabase/Neon) + Prisma,
S3/R2(비디오·trace 저장), Clerk/Auth.js, Stripe, 러너는 Fly.io/Railway 컨테이너.

리스크: 비디오/trace 저장 비용이 큼 → 보관 기간 제한 + 압축 필수.
원작자가 직접 클라우드를 낼 가능성 → 선점 또는 협업 제안.

로드맵: 1~2주 리포트 업로드 + 공유 링크 → 3~6주 대시보드 + 추이 → 7~12주 호스팅 러너 + 결제.
**MVP 는 "리포트 공유 링크" 하나만.**

### 아이디어 2 — 콘텐츠 → 교육 비즈니스

투입 자본이 거의 없고, 다른 모든 아이디어의 마케팅 채널이 됩니다.

```
1단계: 유튜브 (무료)      ← 11장의 8편 시리즈
2단계: 뉴스레터 (무료)     "AI QA 주간"
3단계: 유료 상품
   ├─ 전자책 "AI 에이전트 QA 실전"          ₩29,000
   ├─ 온라인 강의 (인프런/유데미)            ₩99,000
   ├─ 스크립트 템플릿 팩 (로그인/결제/폼)     ₩49,000
   ├─ 멘토링 / 코드리뷰 (1:1)               ₩150,000/시간
   └─ 기업 교육 (반나절 워크숍)              ₩2,000,000+
```

| 시나리오 | 구독자 | 월 수익 추정 |
| --- | --- | --- |
| 소극적 | 1,000 | 애드센스 10만 + 강의 30만 ≈ 40만원 |
| 보통 | 10,000 | 애드센스 80만 + 강의 300만 + 컨설팅 200만 ≈ 580만원 |
| 성공 | 50,000 | 애드센스 400만 + 강의 1,500만 + 기업교육 1,000만 ≈ 2,900만원 |

차별화: 한국어 콘텐츠 선점, 소스 해부 편으로 전문성 포지셔닝 → 컨설팅 문의 직결.

### 아이디어 3 — 셀프 힐링 E2E 테스트 SaaS

E2E 의 진짜 고통은 작성이 아니라 **유지보수**입니다. Canary 의 "탐색 → 스크립트" 구조를 역방향으로 활용:

```
① 테스트 실행 → 통과   → 종료 (AI 비용 0원)
② 테스트 실패
   ↓
③ AI 자동 투입: snapshotForAI() 로 페이지 재탐색
   ↓
④ 분기
   ├─ UI 변경만 → 스크립트 자동 수정 + PR 생성
   └─ 실제 버그 → report.html 증거 첨부하여 알림
```

```
Starter  $49/월   테스트 50개, 힐링 100회/월
Growth   $199/월  테스트 500개, 힐링 1000회/월, PR 자동 생성
Scale    $499/월  무제한, 전용 러너, 우선 지원
```

AI 호출이 "실패할 때만" 발생하므로 원가 구조가 유리합니다.
추가 구현: GitHub App(PR 체크 + 자동 수정 PR), 실패 분류 모델, 힐링 이력 추적.

⚠️ 최대 리스크: **실제 버그를 UI 변경으로 오판하여 자동 수정** → 반드시 사람 승인 게이트 필요.

### 아이디어 4 — Canary MCP 서버 (가장 빠른 출시)

Canary 는 MCP 가 아니지만 MCP 를 원하는 수요가 있습니다.

```
@your-name/canary-mcp
  ↓ MCP 툴로 노출
canary_session_start / canary_run / canary_session_end / canary_report_read
  ↓
Claude Desktop, Zed, Windsurf, n8n 등 MCP 클라이언트 전부 지원
```

수익: OSS 공개로 인지도 확보 → Pro($19/월, 클라우드/팀 기능) → Canary Cloud 로 유입.
**난이도 매우 낮음 (CLI 래핑만) — 주말 하나면 가능, 첫 출시 후보로 최적.**

### 아이디어 5 — 버티컬 스킬 / 템플릿 팩

| 팩 | 내용 | 가격 |
| --- | --- | --- |
| 이커머스 QA 팩 | 장바구니/결제/쿠폰/재고 플로우 | $79 |
| 결제 QA 팩 | Stripe / 토스 / 카카오페이 시나리오 | $99 |
| 인증 QA 팩 | 소셜 로그인 / 2FA / 비번 재설정 / 세션 만료 | $79 |
| 접근성 감사 팩 | WCAG 체크 + 리포트 템플릿 | $129 |
| **한국형 팩** | 본인인증 / 국내 PG / 배송 조회 | $149 |

한국형 팩이 블루오션 — 해외 도구는 국내 결제·본인인증 플로우를 다루지 못합니다.
판매처: Gumroad / Lemon Squeezy → 이후 자체 사이트.

### 아이디어 6 — 컨설팅 & QA 자동화 대행 (현금 흐름 최단)

| 서비스 | 가격 |
| --- | --- |
| QA 성숙도 진단 (반나절) | ₩800,000 |
| Canary 도입 + 핵심 플로우 10개 구축 | ₩5,000,000 |
| 월간 리테이너 (테스트 유지보수) | ₩1,500,000/월 |
| 사내 교육 워크숍 (1일) | ₩2,500,000 |
| QA as a Service (풀 대행) | ₩3,000,000/월 |

중소기업에는 QA 엔지니어가 없고(개발자 겸업), E2E 도입 실패 경험이 많으며,
"AI 가 해준다"는 메시지가 현재 잘 받아들여집니다. 유튜브로 신뢰를 쌓으면 인바운드가 발생합니다.

### 아이디어 7 — 통합 플러그인 생태계

| 만들 것 | 수익 모델 |
| --- | --- |
| GitHub App ("PR 마다 자동 QA 코멘트") | 리포당 $10/월 |
| Slack 앱 (`/canary verify checkout`) | 시트당 과금 |
| Jira / Linear 연동 (버그 티켓에 report.html 자동 첨부) | $15/시트 |
| n8n / Zapier 노드 | 무료 (리드 수집) |
| Vercel / Netlify 통합 (프리뷰 배포마다 QA) | 배포당 과금 |

GitHub App 이 가장 유망 — PR 에 리포트 링크가 남는 것은 팀이 즉시 체감하는 가치입니다.

---

## 13. 권장 실행 순서

```
지금 ~ 2주
  ① 유튜브 1~2편 제작 (리스크 0, 피드백 수집)
  ② MCP 래퍼 제작 후 OSS 공개 (인지도 확보)

1~2개월
  ③ 뉴스레터 시작 + 한국형 템플릿 팩 1개 판매
  ④ "리포트 공유 링크" MVP (Next.js, 가장 아픈 문제 하나만)

3~6개월
  ⑤ 유료 강의 출시
  ⑥ 컨설팅 문의 수용 (유튜브 신뢰 기반)
  ⑦ Canary Cloud Pro 티어 + 결제

6개월 ~
  ⑧ 셀프 힐링 기능 (차별화 무기)
  ⑨ GitHub App
```

### 핵심 원칙

1. **Canary 자체를 재구현하지 않는다.** MIT 이므로 그대로 쓰고 그 위에 제품을 올린다.
2. **"로컬이라서 불편한 점"을 판다.** 공유, 협업, 추이, CI.
3. **콘텐츠를 먼저 한다.** 유튜브가 모든 수익의 입구가 된다.

### 준수 사항

- MIT 라이선스 고지 + 원작자 크레딧 유지 (`wizenheimer`, 일부 Sawyer Hood 파생분)
- "Canary" 를 자체 상표처럼 쓰지 말고 별도 브랜드를 둔다 (예: "QA Copilot powered by Canary")
- 원작자에게 먼저 연락 — 적대 관계보다 파트너십이 유리
- 고객 데이터 처리 방침 명시 — **비디오/HAR 에 개인정보가 포함될 수 있음**
  (개인정보보호법 / GDPR 주의)

---

## 14. 참고 문서

| 문서 | 내용 |
| --- | --- |
| [`README.md`](../README.md) | 제품 소개, 설치, 사용법, 스크립팅 API |
| [`AGENTS.md`](../AGENTS.md) | 에이전트/신규 기여자용 아키텍처 오리엔테이션 |
| [`CONTRIBUTING.md`](../CONTRIBUTING.md) | 기여 플로우 |
| [`RELEASING.md`](../RELEASING.md) | 퍼블리시 파이프라인 |
| [`CHANGELOG.md`](../CHANGELOG.md) | 변경 이력 |
| `skills/canary-scripting/references/REFERENCE.md` | 샌드박스 API 전체 레퍼런스 |
| `skills/canary-verify/references/REFERENCE.md` | QA 플랜 작성 레퍼런스 |
| `docs/snippets/` | LLM 문서 단일 소스 (직접 편집 금지, `make docs` 사용) |
| `examples/` | 데모 스크립트 4개 |
