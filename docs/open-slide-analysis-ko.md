# open-slide 분석 정리 (한국어)

> 이 문서는 open-slide 저장소를 직접 열어보며 분석한 내용과 Q&A를 정리한 기록입니다.
> 작성일: 2026-09-18

## 🔗 관련 링크

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/open-slide |
| 원본 저장소 (upstream) | https://github.com/1weiho/open-slide |
| 공식 사이트 / 문서 | https://open-slide.dev |
| npm — 런타임 | https://www.npmjs.com/package/@open-slide/core |
| npm — 스캐폴더 | https://www.npmjs.com/package/@open-slide/cli |
| 로고 검색 연동 | https://svgl.app |
| 라이선스 | MIT (`LICENSE`) |

---

## 1. 한 줄 정의

**AI 코딩 에이전트가 작성하는 것을 전제로 설계된 React 기반 슬라이드 프레임워크.**

PowerPoint처럼 마우스로 만드는 게 아니라 슬라이드를 React 컴포넌트로 작성하는데,
그 코드를 사람이 아니라 Claude Code / Cursor / Codex 같은 에이전트가 쓰도록 만들어졌다.

README의 핵심 문장:

> "Slides are visual code. Agents are great at writing code.
> open-slide is the missing runtime."

---

## 2. 저장소 구조

pnpm + Turbo 모노레포.

| 경로 | 패키지 | 역할 |
| --- | --- | --- |
| `packages/core` | `@open-slide/core` (v2.0.0-beta.1) | 런타임(뷰어·발표 모드·인스펙터), Vite 플러그인, `open-slide` CLI, 내장 스킬 |
| `packages/cli` | `@open-slide/cli` (v2.0.0-beta.1) | `npx @open-slide/cli init` 스캐폴더 + 프로젝트 템플릿 |
| `apps/demo` | private | 프레임워크 개발용 데모 워크스페이스 (데모 덱 18종) |
| `apps/web` | private | 마케팅 / 문서 사이트 (Next.js) |

### `packages/core/src` 내부

```
src/
├── app/                       # 브라우저에서 동작하는 React 런타임
│   ├── routes/                # home, slide, presenter, assets, themes
│   ├── components/
│   │   ├── inspector/         # 클릭 → 코멘트 첨부 UI
│   │   ├── present/           # 발표 / 프레젠터 모드
│   │   ├── style-panel/       # 스타일 편집 패널
│   │   ├── command/           # ⌘K 커맨드 팔레트
│   │   └── ui/                # shadcn 생성 컴포넌트 (biome 제외 대상)
│   └── lib/export-pdf.ts      # PDF 내보내기 (html-to-image + fflate)
├── vite/                      # Vite 플러그인 + 개발서버 API
│   └── routes/                # assets, comments, context, edit, folders,
│                              # restart, slides, svgl, update, watchers
├── cli/                       # dev / build / preview / sync:skills
├── editing/                   # 소스 코드 직접 편집 로직
├── http/request-guard.ts      # 개발서버 API 오리진 가드
└── locale/                    # en, ja, zh-cn, zh-tw
```

### 내장 에이전트 스킬 (`packages/core/skills/`)

| 스킬 | 역할 |
| --- | --- |
| `create-slide` | 덱 작성 워크플로. 코드 작성 전 4가지(주제·미감 / 페이지 수 / 텍스트 밀도 / 모션)를 반드시 질문 |
| `slide-authoring` | 기술 레퍼런스. 하위 `references/` 7종: assets, design-system, morph, page-numbers, steps, transitions, webfonts |
| `apply-comments` | 인스펙터가 남긴 `@slide-comment` 마커를 읽어 코드에 반영 후 마커 삭제 |
| `create-theme` | 재사용 테마 작성 |
| `current-slide` | 현재 보고 있는 슬라이드 컨텍스트를 에이전트에 전달 |

---

## 3. 동작 원리

### 3.1 슬라이드 = React 컴포넌트 배열

`slides/<id>/index.tsx` 하나가 덱 하나다.

```tsx
import { type DesignSystem, type Page, type SlideMeta } from '@open-slide/core';

export const design: DesignSystem = {
  palette: { bg: '#0a0e14', text: '#e6edf3', accent: '#6ee7ff' },
  fonts: { display: "'JetBrains Mono', monospace", body: "'Inter', sans-serif" },
  typeScale: { hero: 132, body: 34 },
  radius: 6,
};

export const meta: SlideMeta = { title: '...', createdAt: '...' };
export const pages: Page[] = [Cover, Agenda, Content, Closing];
```

모든 페이지는 **1920 × 1080 고정 캔버스**에 그려지고 화면 크기에 맞춰 자동 스케일된다.
덕분에 `fontSize: 132` 같은 하드코딩이 어떤 디스플레이에서도 동일한 비율로 보인다.

### 3.2 인스펙터 → 코드 왕복 루프 (핵심 UX)

```
브라우저에서 요소 클릭 → 코멘트 입력
        ↓
소스에 마커 삽입
{/* @slide-comment id="c-<8hex>" ts="<ISO>" text="<base64url(JSON)>" */}
        ↓
에이전트에게 /apply-comments 실행
        ↓
마커 해석 → 코드 수정 → 마커 삭제
        ↓
HMR로 즉시 반영
```

마커 설계의 디테일:

- `text`는 **base64url로 인코딩한 JSON** (`{"note": "...", "hint"?: "..."}`) — 따옴표·줄바꿈 파손 방지
- `apply-comments` 스킬이 탐지용 정규식을 "authoritative"로 명시해 에이전트의 자의적 파싱을 차단
- 수정은 **줄 번호 역순**으로 적용하라고 지시 — 파일 길이 변화로 인한 오프셋 어긋남 방지

### 3.3 주요 기능

- 발표자 모드: 현재/다음 슬라이드 미리보기, 발표 노트, 타이머
- 에셋 매니저: 이미지·영상·폰트 관리 + svgl 브랜드 로고 검색
- 내보내기: 정적 HTML 사이트 / PDF (전부 클라이언트 사이드 처리)
- 슬라이드 매니저: 폴더 + 이모지 + 드래그앤드롭 정렬 (`@dnd-kit`)
- 다국어 UI: 영어 / 일본어 / 중국어 번체·간체 (한국어 없음 → 기여 기회)
- 배포: 순수 정적 빌드 (Vercel, Cloudflare Pages, Netlify, Zeabur)

---

## 4. 설치 및 사용법

### 4.1 슬라이드를 만드는 경우

```bash
npx @open-slide/cli init my-slide   # Node.js ^20.19.0 || >=22.12.0
cd my-slide
pnpm dev                            # http://localhost:5173
```

생성되는 구조 (Vite / React / Tailwind 설정은 core 내부에 은닉):

```
my-slide/
├── slides/getting-started/index.tsx
├── themes/
├── assets/
├── open-slide.config.ts
├── package.json
├── tsconfig.json
├── AGENTS.md
├── vercel.json
└── netlify.toml
```

### 4.2 CLI 명령어

```bash
open-slide dev                 # 개발 서버
  -p, --port <port>            #   포트 지정
  --host [host]                #   네트워크 노출 (모바일 확인용)
  --open                       #   브라우저 자동 실행
  --no-skills-check            #   스킬 최신화 체크 생략

open-slide build               # 정적 사이트 빌드
  --out-dir <dir>              #   출력 폴더 (기본 dist)

open-slide preview             # 프로덕션 빌드 미리보기
open-slide sync:skills         # 내장 스킬 동기화
```

### 4.3 프레임워크 자체를 개발하는 경우 (이 저장소)

```bash
pnpm install
pnpm dev          # apps/demo를 로컬 core로 실행
pnpm build        # 전체 빌드
pnpm typecheck    # tsc
pnpm check        # biome (format + lint + organize imports)
pnpm check:fix    # 자동 수정
pnpm test         # vitest
pnpm test:e2e     # playwright
pnpm core <script> / pnpm cli <script>   # 패키지 단위 실행
```

하드룰 (`CLAUDE.md` / `AGENTS.md`):

- 커밋 전 `pnpm check` 통과 필수
- `packages/core` 또는 `packages/cli` 변경 시 `pnpm changeset` 필수
- changeset 설명은 한 줄, 현재형, 사용자 관점
- 버전 / CHANGELOG 수동 수정 금지 (`changeset version`이 담당)
- 의존성 추가 신중히 (core는 사용자에게 배포됨)
- `src/app/components/ui`는 shadcn 생성물 — 건드리지 않음
- 주석은 기본적으로 쓰지 않음 (WHY가 비자명할 때만)

---

## 5. Q&A 정리

### Q. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아니고, "스킬을 품은 npm 패키지"다.**

| 분류 | 여부 | 근거 |
| --- | --- | --- |
| npm 패키지 | O (정체) | `@open-slide/core`, `@open-slide/cli` |
| Vite 플러그인 | O (내부 포함) | `src/vite/*-plugin.ts` |
| Agent Skill | O (내부 포함) | `packages/core/skills/*/SKILL.md` |
| CLI 도구 | O (내부 포함) | `bin: { "open-slide": "./bin.js" }` |
| MCP 서버 | **X** | MCP SDK 의존성 없음, 관련 코드 0줄 |
| Claude Code 플러그인 | **X** | `.claude-plugin/` 없음 |

`open-slide dev` 실행 시 CLI가 내장 스킬을 프로젝트의 `.claude/skills/`로 동기화하고,
버전이 어긋나면 "Sync now? (Y/n)"으로 물어본다.

MCP 대신 "파일 읽기/쓰기"에 기댄 설계라서 **모든 코딩 에이전트에서 동작**한다는 장점이 있다.

### Q. API 토큰이 필요한가?

**필요 없다.** 코드베이스 전수 조사 결과 인증 관련 코드가 존재하지 않는다.

사용되는 환경변수 전부:

```
npm_config_user_agent            # 패키지 매니저 감지
OPEN_SLIDE_SKIP_SKILLS_CHECK     # 스킬 체크 스킵
WT_SESSION / TERM_PROGRAM / CI   # 터미널 환경 감지
```

외부 통신은 두 군데뿐이며 둘 다 인증 없는 공개 API다.

| 대상 | 용도 | 필수 여부 |
| --- | --- | --- |
| `registry.npmjs.org` | 신버전 확인 (`vite/routes/update.ts`) | 선택 |
| `api.svgl.app` | 브랜드 로고 검색 프록시 (`vite/routes/svgl.ts`) | 로고 검색 시에만 |

단, open-slide 자체는 무료지만 **에이전트(Claude Code 등)의 구독/API 비용은 별도**다.
슬라이드 내용이 외부로 전송되는 경로는 없어 사내 기밀 자료 작성에도 안전하다.

### Q. 왜 GitHub에서 주목받는가?

1. **타이밍** — AI 코딩 에이전트 확산기에 "에이전트로 뭘 하지?"에 대한 구체적 답을 제시
2. **포지셔닝** — "The slide framework built for agents"로 새 카테고리 선점
3. **문제 정의** — "슬라이드는 코드다 → 에이전트는 코드를 잘 쓴다 → 런타임이 없었다"는 설득력
4. **데모 품질** — 데모 덱 18종, 한 파일에 키프레임 20개가 넘는 고밀도 애니메이션
5. **진입장벽** — `npx` 한 줄, 가입/키 불필요
6. **바이럴 기능** — 클릭 코멘트 → 코드 자동 수정 (영상 3초면 전달되는 임팩트)
7. **엔지니어링 완성도** — React 19 / Vite 8 / Tailwind 4 / TS 7, Vitest + Playwright, Changesets, Biome, Dependabot, CI·release 워크플로, 다국어, 커뮤니티 문서 일체
8. **외부 검증** — Vercel OSS Program 배지, ko-fi 후원
9. **생태계 연결** — `skills-lock.json`에 emilkowalski/skills, anthropics/skills, vercel-labs/agent-skills, shadcn/ui 참조

### Q. 로컬 에이전트 구축에 도움이 되는가?

**매우 도움이 된다.** "에이전트 친화적 도구 설계"의 실전 레퍼런스로서 가져갈 패턴 5가지:

1. **Skill = 도메인 규칙의 외부화**
   워크플로(`create-slide`)와 레퍼런스(`slide-authoring`)를 분리하고,
   상세 문서는 `references/`로 빼서 필요할 때만 읽게 하는 **점진적 공개**로 컨텍스트 절약.

2. **질문 강제로 모호함 제거**
   "코드 작성 전 반드시 4가지를 물어라" + 선택지까지 사전 정의 → 재작업과 토큰 낭비 방지.

3. **GUI ↔ 코드 양방향 루프**
   마커(id/ts/base64url payload) + 권위 있는 정규식 + 역순 편집 지시.
   Figma→코드, 로그 대시보드→수정 등 "눈으로 보는 곳"과 코드를 잇는 모든 시나리오에 응용 가능.

4. **제약이 곧 품질**
   캔버스 고정, 파일 경로 고정, "package.json 수정 금지", 타입 스케일 사전 정의,
   외부 애니메이션 라이브러리 금지 — 자유를 줄여 실패 경로를 차단.

5. **도구 중립성 + 스킬 버전 관리**
   `CLAUDE.md`와 `AGENTS.md`를 모두 제공하고, `skills-lock.json`으로 외부 스킬을 해시 고정.

추가로 `src/vite/routes/`는 **"개발 서버가 곧 에이전트용 API"** 라는 구조의 좋은 예시다.

#### 로컬 에이전트 설계 체크리스트

```
□ 도메인 규칙을 SKILL.md로 외부화했나?
□ 워크플로(순서)와 레퍼런스(규격)를 분리했나?
□ 상세 내용을 references/로 빼서 컨텍스트를 절약했나?
□ 작업 전 확인 질문을 강제했나?
□ 선택지를 미리 제공해 판단 부담을 줄였나?
□ "하면 안 되는 것"을 명시했나?
□ 파일 경로 / 구조를 고정했나?
□ GUI에서 코드로 가는 경로가 있나?
□ 마커 / 식별자 포맷이 파손에 안전한가?
□ 정규식 / 파서를 정확히 명시했나?
□ AGENTS.md와 CLAUDE.md를 모두 제공했나?
□ 결과를 즉시 확인할 수 있나? (HMR, 프리뷰)
□ 스킬 버전을 lock 파일로 관리하나?
```

### Q. React나 PHP로 만들 수 있는가?

**React는 이미 그 자체다.** React 19 + Vite 8 + Tailwind 4 + TypeScript 7 스택이라
기존 React 지식으로 바로 개조할 수 있다. 실시간 API 데이터, 인터랙티브 데모 등
PowerPoint로는 불가능한 표현이 가능하다.

개조 지점:

| 목적 | 위치 |
| --- | --- |
| 슬라이드 추가 | `slides/<id>/index.tsx` |
| 공용 컴포넌트 export | `packages/core/src/index.ts` |
| 새 화면 추가 | `packages/core/src/app/routes/` |
| 개발서버 API 추가 | `packages/core/src/vite/routes/` |
| 에이전트 규칙 수정 | `packages/core/skills/` |
| 한국어 지원 | `packages/core/src/locale/ko.ts` (미존재 — 기여 기회) |

**PHP는 그대로 이식하기 어렵다.** 핵심 가치가 "브라우저 React와 Node 서버가 코드를
공유하고 Vite가 실시간으로 잇는 구조"에서 나오기 때문이다.

| 기능 | PHP 이식 | 사유 |
| --- | --- | --- |
| 슬라이드 = React 컴포넌트 | 불가 | JS 런타임 필요 |
| HMR | 불가 | Vite/Node 전용 |
| TSX 파싱 후 마커 삽입 | 매우 어려움 | `@babel/parser` 대체 불가, 정규식으로는 취약 |
| PDF 내보내기 | 가능 | wkhtmltopdf, Dompdf |
| 정적 빌드 | 가능 | - |
| 발표 모드 | 가능 | 일반 JS로 구현 |

현실적 대안 (권장 순):

1. **React로 포크해 개조** — 리스크 없음, 즉시 시작 가능
2. **PHP 백엔드 + open-slide 프론트** — 인증·결제·DB·관리자는 PHP, 슬라이드 런타임은 그대로
3. **PHP로 슬라이드 생성기** — DB 데이터를 읽어 `.tsx`를 생성하고 `open-slide build` 호출 (정기 보고서 자동화에 적합)
4. **PHP로 전면 재구현** — 페이지를 PHP 배열/템플릿 DSL로 정의. 자유도는 낮지만 스키마 검증이 쉬워 에이전트 안정성은 오히려 높을 수 있음

---

## 6. 수익화 아이디어

MIT 라이선스이므로 상업적 이용·수정·비공개 개조·재배포가 모두 허용된다.

### 6.1 요약

| 방향 | 내용 | 난이도 | 기대 수익 |
| --- | --- | --- | --- |
| 프리미엄 테마 판매 | 고품질 덱 템플릿 팩 | 낮음 | 중 |
| 호스팅 SaaS | 설치 없는 웹 버전 (비개발자 타겟) | 매우 높음 | 매우 높음 |
| 기업용 브랜드 킷 | 브랜드 가이드 → 테마·스킬 자동 생성 | 중간 | 높음 |
| 교육 콘텐츠 | 강의·뉴스레터·전자책 | 낮음 | 중 |
| 덱 제작 에이전시 | AI 활용 발표자료 대행 | 낮음 | 높음 |

### 6.2 상세

**① 프리미엄 테마 마켓플레이스**
기본 테마가 거의 없다는 공백을 노린다. Startup Pitch / Data Storytelling / Academic /
Korean Business / Dark Tech 등. 가격 개별 $19–39, 번들 $79, 전체 $149, 팀 $299.
제작 원가가 사실상 0이고 재고가 없어 패시브 인컴에 적합. Gumroad·Lemon Squeezy 활용.

**② 호스팅 SaaS**
최대 장벽인 "Node.js 설치"를 제거한 클라우드 버전. Free / Pro $15 / Team $39·인 / Enterprise.
차별점은 **결과물이 코드라 락인이 없다는 것** — Gamma·Tome 대비 개발자·기술 기업에 강한 소구점.
다만 멀티테넌시·샌드박스·보안 난이도가 높고 AI API 비용이 원가를 잠식한다.
현실적으로는 **Phase 1: 데스크톱 앱(Electron/Tauri) → Phase 2: 덱 호스팅 → Phase 3: 풀 SaaS** 순서를 권장.

**③ 기업용 브랜드 킷 (숨은 알짜)**
"브랜드 가이드는 있는데 아무도 안 지킨다"는 기업의 고통을 해결.
브랜드 가이드를 open-slide 테마 + 커스텀 스킬로 변환해, 직원이 말로 요청하면
브랜드를 100% 준수하는 덱이 생성되도록 구축.
구축비 500–3,000만원, 연 유지보수 20%, 추가 템플릿 100만원, 사내 교육 200만원.
타겟 1순위는 개발 문화가 있는 **시리즈 A–C 스타트업**. 경쟁이 거의 없고 혼자서도 시작 가능.

**④ 교육 콘텐츠**
YouTube(무료) → 인프런/유데미 강의(5–15만원) → 유료 뉴스레터($5/월) → 전자책($29) → 기업 워크샵(200만원).
단독 수익원이라기보다 **다른 수익원으로 연결되는 깔때기**로 운용하는 것이 핵심.
한국어 콘텐츠는 경쟁이 적다.

**⑤ 덱 제작 에이전시**
AI + 프레임워크로 제작 속도가 빨라 마진이 높다.
피치덱 200–500만원, 컨퍼런스 자료 100–200만원, 사내 교육자료 150만원,
긴급 할증 +50%, 월 구독 300만원. 즉시 시작 가능하고 현금 흐름이 빠르다.
시간을 파는 구조라 확장성에 한계가 있으므로 템플릿화로 반복 작업을 줄인다.

### 6.3 보너스 아이디어

- GitHub Sponsors 후원 (기여 → 인지도 → 후원·커리어)
- 플러그인/확장 판매 (차트 팩, Notion·Figma 연동, 한글 폰트 최적화 팩)
- 덱 조회 분석 SaaS (DocSend 유사 — 세일즈 덱에 유용)
- PPTX 변환기 (현재 최대 약점 해결)
- **한국어 특화 포크 "K-slide"** — 한글 폰트 프리셋, 국내 문서 문화 템플릿, 한국어 UI·스킬, 국내 클라우드 배포 지원

### 6.4 추천 실행 순서

1. **교육 콘텐츠 + 에이전시** (즉시) — 투자 0원, 현금 흐름 빠름, 개인 브랜드 축적
2. **기업용 브랜드 킷** (3–6개월 후) — 단가 높고 경쟁 없음, 레퍼런스 1건 확보가 관건
3. **한국어 포크 K-slide** (병행) — `src/locale/ko.ts`부터. 수익보다 브랜딩 효과
4. **SaaS** (1년 후) — 시장 이해 후 데스크톱 앱부터 단계적으로

### 6.5 법적 체크리스트

```
허용   상업적 이용 / 수정 / 비공개 개조 / 재배포
준수   LICENSE의 저작권 표시 유지, MIT 전문 포함
금지   원저작자(1weiho) 표시 제거, 공식 프로젝트 사칭, 브랜드 혼동
권장   README에 "Built on open-slide by 1weiho" 명시
```

---

## 7. 알려진 한계

- Node.js 개발 환경이 필요해 비개발자 진입장벽이 존재
- 코딩 에이전트가 있어야 본래 가치가 발휘됨
- `2.0.0-beta.1` 베타 단계 — 공개 API 변경 가능성
- 실시간 공동 편집 없음 (Git 기반 협업으로 대체)
- PPTX 내보내기 미지원 (PDF / 정적 HTML만)
- UI 다국어에 한국어 미포함
