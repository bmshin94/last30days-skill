# last30days-skill 전수조사 · 분석 · 수익화 정리 (한국어)

> 작성일: 2026-09-28
> 분석 대상 버전: **v3.24.0**
> 원본 저장소(upstream): <https://github.com/mvanhorn/last30days-skill>
> 포크 저장소(this fork): <https://github.com/bmshin94/last30days-skill>
> 라이선스: MIT (상업적 이용 가능, 저작권 표시 유지 필요)

이 문서는 저장소 전체를 조사한 뒤 진행한 대화(무엇인지 / 쉽게 설명 / 설치·사용법 /
플러그인·스킬·MCP 구분 / API 토큰 / 인기 이유 / 로컬 에이전트 활용 / React·PHP 구현 /
수익화 아이디어)를 하나로 정리한 기록이다.

---

## 1. 한 줄 요약

> **"구글은 편집자를 검색하고, 이건 사람을 검색한다."**

Reddit 업보트, X 좋아요, YouTube 자막, TikTok 조회수, Polymarket 실제 배팅 금액 등
**13개 이상의 플랫폼을 병렬로 수집 → 참여도(engagement)로 점수화 → AI가 브리핑 한 장으로 합성**하는
**AI 에이전트용 검색 스킬**이다. 기본 검색 창은 최근 30일.

---

## 2. 규모

| 항목 | 수치 |
|---|---|
| 메인 엔진 `skills/last30days/scripts/last30days.py` | 4,288줄 |
| 라이브러리 `scripts/lib/*.py` | 98개 파일 / 약 57,000줄 |
| 에이전트 지시문 `SKILL.md` | 2,426줄 |
| 테스트 (pytest) | 233개 파일, 커버리지 하한 84% |
| README 번역 | en / fr / de / es / pt-BR / ja / zh-CN (7개) |
| CI 워크플로우 | 9개 (validate, security, osv-scanner, scorecard, zizmor, changelog-guard, release 등) |
| 커뮤니티 | 175개 머지 PR, 기여자 52명, 릴리스 15회+ |
| CLI 플래그 | 70개 이상 |

---

## 3. 폴더 구조 (조사 결과)

```
last30days-skill/
├── skills/last30days/            # 실제 제품(Skill)
│   ├── SKILL.md                  # 2,426줄 "계약서" - 모델이 읽는 지시문
│   ├── scripts/
│   │   ├── last30days.py         # 메인 엔진
│   │   ├── store.py / watchlist.py / briefing.py   # 저장·모니터링·브리핑
│   │   └── lib/                  # 98개 모듈 (소스 어댑터 + 파이프라인)
│   │       ├── reddit_{rss,arctic,shreddit,keyless,public,enrich}.py
│   │       ├── x_api.py bird_x.py grok_x.py xai_x.py xurl_x.py xquik.py
│   │       ├── youtube_yt.py tiktok.py instagram.py threads.py pinterest.py
│   │       ├── linkedin.py bluesky.py truthsocial.py telegram.py
│   │       ├── hackernews.py polymarket.py github.py arxiv.py techmeme.py
│   │       ├── digg.py stocktwits.py amazon.py trustpilot.py meta_ads.py
│   │       ├── xiaohongshu_api.py
│   │       ├── pipeline.py planner.py fanout.py            # 오케스트레이션
│   │       ├── dedupe.py cluster.py fusion.py rerank.py    # 후처리
│   │       ├── relevance.py grounding.py freshness.py signals.py  # 랭킹
│   │       ├── render.py html_render.py html_publish.py    # 출력
│   │       ├── chrome_cookies.py safari_cookies.py chrome_cdp.py agentcookie.py
│   │       ├── setup_wizard.py                              # 첫 실행 온보딩
│   │       ├── doctor.py preflight.py health.py prescriptions.py  # 자기 진단
│   │       └── vendor/bird-search/                          # X 검색 JS 클라이언트
│   ├── agents/openai.yaml, references/, assets/
├── mcp/                          # Go로 만든 MCP 서버 (.mcpb, Claude Desktop용)
├── .claude-plugin/ .grok-plugin/ .codex-plugin/ .agents/ gemini-extension.json
├── hooks/                        # SessionStart 훅 (설정 체크)
├── tests/ (233) fixtures/ docs/(solutions·plans·releases) changelog.d/
└── CLAUDE.md / AGENTS.md         # 에이전트 기여자용 프로젝트 규칙
```

---

## 4. 동작 파이프라인

```
/last30days "토픽"
  → [Step 0] 첫 실행 시 온보딩(동의 기반: 쿠키 / 키 / 소스 선택)
  → [planner.py]  토픽 분해 + 인물/브랜드 판별 + 서브쿼리 생성
  → [fanout.py]   ThreadPoolExecutor 병렬 수집
        Reddit(RSS→arctic-shift→shreddit) / X(백엔드 6개 failover) /
        YouTube(yt-dlp 자막) / TikTok·Instagram·Threads·Pinterest(ScrapeCreators) /
        HN / Polymarket / GitHub / arXiv / Techmeme / Digg / Web
  → normalize → freshness(30일) → relevance → grounding(엔티티 확인)
  → dedupe → cluster → fusion → rerank
  → signals.py (업보트·좋아요·조회수·댓글·배팅금액 가중 점수)
  → render.py (브리핑 + Top Community Comments + 출처 인용)
  → 에이전트 최종 합성 (SKILL.md LAW 1~4 강제)
```

### 설계 하이라이트
1. **Fallback 체인** — Reddit 공식 JSON API 폐지 후 RSS·arctic-shift·shreddit 등 다중 경로. X는 백엔드 6개 자동 failover.
2. **LAW 강제 시스템** — "제목 창작 금지(LAW 2)", "섹션 헤더 금지(LAW 4)" 등. 출력 첫 줄 배지를 앵커로 사용하고, 과거 회귀(0/8 실패) 사례를 SKILL.md에 그대로 기록.
3. **자기 진단** — `--doctor`(소스별 원인+처방), `--preflight`(쓰기·쿠키 없이 안전 점검).
4. **보안 하드닝** — HTML 렌더러 XSS 방어, 로그/픽스처 시크릿 레댁션, 스크래핑 콘텐츠의 untrusted-content fence(프롬프트 인젝션 방어).

---

## 5. 언제 쓰는가

| 상황 | 예 |
|---|---|
| 미팅 전 인물 조사 | `/last30days 홍길동` |
| 경쟁사/도구 비교 | `/last30days A vs B vs C` (`--competitors`) |
| 떠오르는 주제 발굴 | `/last30days what's exploding in AI agents?` (`--discover`) |
| 채용 시그널로 전략 추론 | `--hiring-signals` |
| 속보·이슈 여론 정리 | 여론 + Polymarket 확률 |
| 제품·여행 조사 | 실사용 리뷰, 커뮤니티 불만 |
| 최신 기술 학습 | 학습 데이터보다 빠른 커뮤니티 베스트 프랙티스 |

---

## 6. 설치 및 사용법

### 설치 (호스트별)

| 환경 | 명령 |
|---|---|
| Claude Code (추천) | `/plugin marketplace add mvanhorn/last30days-skill` → `/plugin install last30days` |
| Codex / Cursor / Copilot / Gemini CLI 등 50+ | `npx skills add mvanhorn/last30days-skill -g` |
| Grok (xAI Build CLI) | `grok plugin marketplace add mvanhorn/last30days-skill` → `grok plugin install last30days` |
| claude.ai (웹) | Releases의 `last30days.skill` 업로드 (Code execution 활성화 필요) |
| Claude Desktop | Releases의 `.mcpb` 드래그 |
| OpenClaw | `clawhub install last30days-official` |
| 이 포크 | `/plugin marketplace add bmshin94/last30days-skill` |

주의: 플러그인 설치와 `npx skills` 설치를 동시에 하면 `/last30days`가 중복 표시된다. 머신당 한 방법만 사용.

### 사용 (슬래시 커맨드 = 기본 경로)

```
/last30days 엔비디아 실적 반응
/last30days AI 영상 생성 도구
/last30days OpenClaw vs Hermes vs Paperclip
/last30days what's exploding in AI agents?
/last30days doctor
```

슬래시 커맨드에는 플래그/파이프를 쓸 수 없다. 자연어로 의도를 말하면 모델이 엔진 플래그로 번역한다.

### 직접 CLI (스크립팅·크론·엔진 테스트용 fallback)

```bash
python3 skills/last30days/scripts/last30days.py "테스트 쿼리" --emit=compact
python3 skills/last30days/scripts/last30days.py "Listen Labs" --hiring-signals
python3 skills/last30days/scripts/last30days.py "AI 에이전트" --discover
python3 skills/last30days/scripts/last30days.py --doctor
python3 skills/last30days/scripts/last30days.py --preflight
```

### 개발 환경

```bash
uv sync --group dev     # Python 3.12+
uv run pytest           # 233개 테스트
uv run pytest --cov     # 커버리지(하한 84%)
npx skills add . -g -y  # 작업물 설치(설치본은 설치 시점에 고정 → 수정 후 재실행 필요)
ln -sfn "$PWD/skills/last30days" ~/.agents/skills/last30days   # 라이브 편집용 심볼릭 링크
```

---

## 7. 플러그인 / 스킬 / MCP 구분

본질은 **Agent Skill**이고, 유통을 위해 Plugin·MCP 옷을 입고 있다.

| 구분 | 위치 | 역할 |
|---|---|---|
| **Skill** (본질) | `skills/last30days/SKILL.md` + `scripts/` | agentskills.io 오픈 포맷. 배포 단위이자 제품 |
| **Plugin** | `.claude-plugin/`, `.grok-plugin/`, `.codex-plugin/`, `gemini-extension.json` | 각 호스트 마켓플레이스용 매니페스트. 버전 락스텝(3.24.0) 테스트로 강제 |
| **MCP** | `mcp/` (Go) | Claude Desktop `.mcpb` 번들용 MCP 서버 래퍼. Python 엔진 임베드 |

프로젝트 문서(CONCEPTS.md)의 표현: *"A Skill is the unit of distribution; the Skill is the product."*,
*"The Engine is implementation; the SKILL.md prose is the agent-facing surface."*

---

## 8. API 토큰 필요 여부

**키 0개로도 동작한다. 키를 넣으면 소스가 늘어난다.**

### 무료·키 불필요
Reddit(RSS + arctic-shift + shreddit, 업보트·톱 댓글 포함), Hacker News, Polymarket, GitHub(공개 API),
arXiv / Techmeme / Digg (각 `*-pp-cli`가 PATH에 있을 때 자동 활성).

> CLI 기반 소스는 파일 존재가 아니라 `shutil.which`로 **PATH에 보이는지**로 판정한다. `$HOME/.local/bin`이 PATH에 있어야 한다.

### 선택적 키

| 키 | 해금 | 비용 |
|---|---|---|
| `SCRAPECREATORS_API_KEY` | TikTok, Instagram(+댓글), Threads, Pinterest, LinkedIn, YouTube 자막 백업 | GitHub 로그인으로 **무료 10,000콜** |
| `X_BEARER_TOKEN` / `XAI_API_KEY` | X 공식 API / Grok 경유 | 유료 |
| X 브라우저 쿠키(`AUTH_TOKEN`, `CT0`) | X 검색(내 세션) | 무료 |
| `OPENAI_API_KEY` | 플래너·리랭크·웹검색 LLM | 유료 |
| `PERPLEXITY_API_KEY` | Perplexity 합성 / Deep Research | 유료 |
| `BRAVE_API_KEY`, `SERPER_API_KEY`, `EXA_API_KEY`, `PARALLEL_API_KEY` | 웹 검색 백엔드 | 무료 티어 있음 |
| `BSKY_HANDLE` + `BSKY_APP_PASSWORD` | Bluesky | 무료 |
| `GITHUB_TOKEN` | rate limit 완화 | 무료 |
| yt-dlp (바이너리) | YouTube 자막 전체 | 무료 |

### 설정 우선순위 및 저장
```
per-run 플래그 > 환경변수 > ~/.config/last30days/.env > 기본값
```
macOS Keychain(`setup-keychain.sh`) / pass(`setup-pass.sh`) 지원. `setup --store-key`는 0o600 권한으로 저장하고
stdout에서 키를 마스킹한다. GitHub device-auth 성공 시 `SCRAPECREATORS_API_KEY`가 자동 저장된다.
AGENTS.md 규칙: 실제 키·쿠키·토큰 커밋 금지, 테스트/픽스처는 명백한 더미값만.

### 현실적 권장 세팅
- 0원 시작: 설정 없이 Reddit + HN + Polymarket + GitHub
- 0원 강화: ScrapeCreators 무료 키 + yt-dlp 설치 + X 브라우저 쿠키 → 사실상 전체 소스
- 유료 추가: OpenAI / Perplexity (합성 품질, 딥리서치)

---

## 9. GitHub에서 인기 있는 이유 (분석)

1. **타이밍** — Agent Skills 포맷 확산기에 "가장 잘 만든 스킬 레퍼런스"로 등장.
2. **날카로운 문제 정의** — "플랫폼별 담벼락 때문에 어떤 AI도 전부 검색하지 못한다"는 공통된 불편을 정확히 지적.
3. **실제로 어려운 기술 돌파** — Reddit 공식 API 폐지 우회, X 백엔드 failover, YouTube 자막 전문 검색.
4. **즉시 체감되는 데모** — 구글의 2023년 링크드인 vs 이번 달 실제 활동 비교.
5. **완성도** — 테스트 233개, 커버리지 84% 하한(상향 정책), CI 9종(Semgrep·OSV·Scorecard·zizmor·provenance), 7개 언어 README, `docs/solutions/`에 버그·해결책 축적.
6. **기여 친화 설계** — `AGENTS.md`/`copilot-instructions.md`로 AI 기여 허용, `changelog.d` 뉴스 프래그먼트로 CHANGELOG 충돌 제거, 플러그형 소스 어댑터(StockTwits·LinkedIn이 커뮤니티에서 유입).
7. **모든 호스트 커버** — Claude Code·Codex·Cursor·Copilot·Gemini·Grok·Desktop·웹·OpenClaw·Hermes.
8. **투명성** — 실패 사례(v3.0.6에서 8연속 실패)와 수정 방법을 문서에 그대로 기록.

배지: GitHub Trending #1 Repository Of The Day, Trendshift 등재.

---

## 10. 로컬 에이전트 구축에 주는 도움

### A. 부품으로 바로 사용
- `--emit=json`으로 구조화 결과를 받아 에이전트가 소비
- `mcp/` 서버를 띄워 로컬 에이전트의 MCP 툴로 등록
- `--mock`, `--record-fixtures`로 오프라인 개발/테스트
- `store.py` / `watchlist.py` / `briefing.py` + 크론 → 자동 브리핑

### B. 배울 아키텍처 패턴 (핵심)

| 패턴 | 참고 위치 |
|---|---|
| 프롬프트를 "계약서"로 설계 (실패 사례를 프롬프트에 명시, 출력 앵커) | `SKILL.md` LAW 1~4 |
| 다중 백엔드 자동 failover | `backends.py`, `x_api.py`→`bird_x.py`→`grok_x.py`→`xai_x.py` |
| 병렬 팬아웃 + 벽시계 예산 | `fanout.py`, `pipeline.py`, `docs/solutions/logic-errors/non-daemon-executor-threads-defeat-wall-clock-budget.md` |
| 정규화→중복제거→클러스터→리랭크 | `normalize/dedupe/cluster/fusion/rerank.py` |
| 참여도 기반 랭킹 | `signals.py`, `relevance.py` |
| 엔티티 그라운딩(오프토픽 차단) | `grounding.py`, `docs/solutions/logic-errors/entity-grounding-*.md` |
| 자기 진단·처방 | `doctor.py`, `preflight.py`, `health.py`, `prescriptions.py` |
| 동의 기반 온보딩(호스트별 3분기) | `setup_wizard.py`, SKILL.md Step 0 |
| 프롬프트 인젝션 방어 | `render.py`, `rerank.py`, `http.py` |
| 관측성 | `log.py` + `source_log(tty_only=False)` 규칙(테스트로 강제) |
| 멀티 호스트 패키징 | `.claude-plugin/`, `.grok-plugin/`, `mcp/` |
| 릴리스 자동화·버전 락스텝 | `.github/scripts/prepare_release.py`, towncrier, `changelog.d/` |

### C. 운영 관행
`docs/solutions/`에 버그마다 `문제유형/원인/해결` 문서를 YAML frontmatter(`module`, `tags`, `problem_type`)로 남겨
에이전트가 재검색 가능하게 하는 방식은 다른 프로젝트에도 바로 도입할 만하다.
문서 규칙을 테스트로 고정하는 관행도 유용하다(`test_onboarding_contract.py`, `test_plugin_contract.py`, `test_source_log_visibility.py`).

### 주의
5만 줄 전체를 모방하면 과설계다. 우선 `backends.py`(failover) / `fanout.py`(병렬) / `doctor.py`(진단) 세 개만 참고해도 충분하다.
쿠키 추출·스크래핑은 각 플랫폼 ToS 확인이 필요하고, 개인 로컬 사용과 상업 서비스의 리스크가 다르다.

---

## 11. React / PHP로 만들 수 있는가

핵심 난이도는 언어가 아니라 **소스별 인증·우회(약 40%) + 랭킹 알고리즘(25%) + 프롬프트 계약(15%)**에 있다.

### React (+ Node / TypeScript) — 권장
- UI/대시보드/스트리밍: 최적
- 백엔드 수집도 충분히 가능 (이미 `vendor/bird-search/`가 JS로 되어 있다)
- 병렬: `Promise.all` / `p-limit`, 브라우저 자동화: Playwright(Node 강점)
- yt-dlp 등은 바이너리로 `child_process` 호출

권장 구조
```
Next.js App Router
├── app/page.tsx                 검색 UI
├── app/api/research/route.ts    SSE 스트리밍 응답
├── lib/sources/*.ts             어댑터 (search(q, days) → Item[])
├── lib/rank.ts                  점수·중복제거·클러스터
├── lib/llm.ts                   합성
└── BullMQ/Inngest + Redis       큐·캐시
```
가장 리스크가 낮은 경로: **먼저 Python 엔진을 subprocess로 감싸고 React UI만 얹은 뒤, 어댑터를 하나씩 TS로 이식.**

### PHP — 가능하지만 하이브리드 권장
- 강점: Laravel(웹 UI, 인증, 결제 Cashier, 관리자, Queue/Scheduler)
- 약점: 기본 동기 처리(ReactPHP/Swoole 필요), 브라우저 암호화 쿠키 해독 라이브러리 빈약, 브라우저 자동화 약함
- 권장: **PHP = SaaS 껍데기, Python/Node = 수집 엔진 사이드카**

### 구현 시 체크
1. ToS/법적 리스크 — 서버 수집 후 재판매는 리스크가 크다. 공식 API·제휴 데이터 비중을 높인다.
2. 쿠키를 서버로 올리지 않는다. SaaS면 BYOK(Bring Your Own Key) 구조.
3. API 콜이 원가다. Redis TTL 캐시와 소스별 예산 상한 필수.
4. 인용은 짧게 + 출처 링크 필수.

---

## 12. 수익화 아이디어

### 대전제
엔진 자체(MIT 무료)는 상품이 아니다. 팔아야 하는 것은
**① 도메인 전문성 ② 자동화·지속성(히스토리·추이) ③ UI/UX ④ 로컬라이즈(한국 소스) ⑤ 신뢰·SLA**.

> 공식: 수익 = 오픈소스 엔진 × 니치 전문성 × 반복 구독

### TIER 1 — 즉시 시작 가능

**1) 한국형 포크 "지난30일"**
원본에는 한국 소스가 전무하다. 디시인사이드·클리앙·더쿠·에펨코리아·뽐뿌·블라인드·네이버 카페/블로그·
아카라이브·루리웹·오늘의집/쿠팡 리뷰 등을 어댑터로 추가. 월 구독(개인 2~3만 / 팀 15~30만).
`lib/reddit_rss.py` 구조를 복제해 어댑터 1개부터 시작. 공개 RSS/검색 위주, robots.txt·rate limit 준수.

**2) 니치 버티컬 주간 브리핑 뉴스레터**
AI 에이전트 주간($20/월), K-뷰티 커뮤니티 트렌드($200/월 B2B), 부동산 지역 여론($50/월),
아이돌·팬덤 동향($300/월), 게임 커뮤니티 여론($500/월), 의료·제약 환자 커뮤니티($1,000/월).
구조: 크론 → `--discover` + 고정 토픽 → `--emit=html` → 발송. `watchlist.py`, `briefing.py`, `store.py`가 이미 존재.

**3) 1회성 리포트 판매(서비스 → 프로덕트)**
인물 사전조사 5~30만원 / 경쟁사 30일 스냅샷 30~100만원 / 채용 시그널 리포트 50~200만원 /
신제품 여론 리포트 100~300만원 / 위기관리 모니터링 월 300만원+. 수동 판매로 검증 후 자동화.

**4) Slack · Discord · 카카오 봇**
`/research <토픽>` → 채널에 브리핑 카드. 워크스페이스당 월 $99~499. Slack Bolt + 기존 엔진 subprocess로 MVP 가능.
확장: 키워드 워치리스트 기반 급증 알림.

### TIER 2 — 본격 SaaS

**5) 웹 SaaS(소셜 리스닝 라이트)**
Next.js + Python 엔진 사이드카 + Postgres + Redis + BullMQ.
Free(월 5회) / Pro $29 / Team $149 / Enterprise. 차별화: Polymarket 확률, YouTube 자막 인용, Reddit 실업보트, 한국 소스.
BYOK 플랜으로 원가·법적 리스크 절감. **해자는 결과의 시계열 누적(3개월 추이)** — 오픈소스 단독으로는 못 하는 부분.

**6) 마켓플레이스 유료 스킬**
무료 `/last30days` 위에 `-korea`($19/월), `-finance`($49/월), `-pro`($29/월)를 라이선스 키 + 호스팅 API로.
현재 Agent Skills 유료 시장이 비어 있어 선점 효과가 있다.

**7) B2B API**
`POST /v1/research` → 정규화 JSON + 점수 + 인용. 호출당 $0.5~2 또는 월 $500~5,000.
다른 SaaS(CRM·세일즈·마케팅)에 임베드. 재판매는 원천 ToS 확인 필수.

### TIER 3 — 사이드 수익
**8) 콘텐츠 자동 공장** — `--discover`로 소재 발굴 → 쇼츠/뉴스레터/블로그/X 스레드 → 광고·제휴 수익.
**9) 교육·강의** — 이 레포를 교재로 "Agent Skills 실전" 강의(30~80만원/인), 기업 스킬 구축 컨설팅(500만~3,000만원).
**10) 구축 대행** — 기업 전용 스킬 셋업 300~1,000만원, 커스텀 소스 연동 500~2,000만원, 운영 월 100~300만원.
**11) 오픈소스 스폰서십** — 업스트림 기여로 신뢰 자산 축적.

### 우선순위 제안
```
1단계(0~1개월)  리포트 수동 판매(3) + Slack 봇(4)        → 수요 검증, 개발 최소
2단계(1~3개월)  니치 뉴스레터(2) + 한국 소스 어댑터(1)   → MRR 시작
3단계(3~9개월)  웹 SaaS(5) + 유료 스킬(6)               → 스케일
병행           강의(9) · 대행(10)                       → 현금 흐름·레퍼런스
```

**하나만 고른다면**: 한국 커뮤니티 소스를 붙인 **니치 버티컬 주간 브리핑**.
경쟁자 없음(원본에 한국 소스 0개), 개발량 적음, B2B 단가 높음, 구독형 MRR,
실패해도 한국 소스 어댑터가 자산으로 남는다.

### 수익화 전 체크리스트

| 항목 | 내용 |
|---|---|
| 라이선스 | MIT — 상업적 이용 가능, LICENSE·저작권 표시 유지 |
| 플랫폼 ToS | 최대 리스크. 스크래핑 수집 후 재판매는 단속 대상. 공식 API·제휴 비중↑, BYOK, 캐시 최소화 |
| 개인정보 | 쿠키·토큰을 서버에 저장하지 않음. 클라이언트 실행 + BYOK |
| 저작권 | 짧은 인용 + 출처 링크 필수, 전문 재배포 금지 |
| AI 고지 | "AI 생성, 팩트체크 필요" 명시(특히 의료·금융) |
| 원가 | API 콜 = 원가. Redis TTL 캐시, 소스별 예산 상한 |

---

## 13. 링크

- 원본(upstream): <https://github.com/mvanhorn/last30days-skill>
- 이 포크: <https://github.com/bmshin94/last30days-skill>
- Agent Skills 포맷: <https://agentskills.io>
- Agent Skills CLI(호스트 목록): <https://github.com/vercel-labs/skills>
- xAI 플러그인 마켓플레이스: <https://github.com/xai-org/plugin-marketplace>
- 저장소 내 참고 문서: `skills/last30days/SKILL.md`, `AGENTS.md`, `CONCEPTS.md`, `CONFIGURATION.md`, `CONTRIBUTING.md`, `docs/how-search-works.md`, `docs/solutions/`
