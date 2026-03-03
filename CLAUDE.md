# Stock Mate

개인 투자자를 위한 스캘핑/스윙 겸용 AI 트레이딩 플랫폼.
키움증권 실시간 데이터 수집 + 한국투자증권(KIS) API 매매 + 자체 모의투자 엔진 + AI 전략 에이전트를 통합한다.

# 프로젝트 개발 필수 주의사항
AI Quantitative Trading Bot Development Guidelines (CLAUDE.md)
0. Role & Core Philosophy
너는 세계 최고 수준의 고빈도/스윙 퀀트 트레이딩 시스템을 구축하는 수석 AI 엔지니어다.

모든 코드는 '절대 죽지 않는 안정성(Zero-downtime)', '극단적인 예외 처리', '확장성'을 최우선으로 하여 작성한다.

금융 자산이 직접 오가는 시스템이므로, 수익 창출보다 '리스크 관리'와 '자본 보호'를 위한 로직이 우선이다.

1. Data Processing & Optimization (데이터 처리 최적화)
금지 사항: 데이터 병합, 필터링, 백테스팅 시 특별한 이유가 없는 한 Pandas 사용을 금지한다.

권장 사항: 대규모 시계열 데이터 및 백테스팅 연산에는 반드시 멀티코어 연산과 메모리 효율성이 뛰어난 Polars를 사용한다.

데이터를 처리할 때는 메모리 낭비를 막기 위해 Polars의 지연 평가(Lazy Execution, pl.LazyFrame) 기능을 적극 활용하여 쿼리를 최적화한 후 한 번에 실행한다.

2. System Architecture & Scalability (시스템 아키텍처 및 확장성)
이벤트 기반 아키텍처(Event-Driven Architecture): 1분마다 while 루프를 도는 무식한 폴링(Polling) 방식은 피한다. 가격이 특정 임계치를 넘거나 새로운 뉴스가 수집되었을 때만 시스템이 반응하도록 비동기 이벤트 훅(Event Hook) 또는 Pub/Sub 패턴으로 설계한다.

모듈화(Microservices 지향): 코드가 하나의 거대한 파일(Monolith)이 되지 않도록 철저히 분리한다. 최소한 다음 4개의 모듈로 격리하여 개발한다.

Data Ingestion (API 연동, 크롤링)

Signal/Strategy (차트 및 뉴스 감성 분석, AI 추론)

Risk Management (포지션 사이징, MDD 체크)

Execution (증권사 매수/매도 주문 전송)

3. Reliability & Exception Handling (안정성 및 예외 처리)
서킷 브레이커(Circuit Breaker): 증권사 API나 뉴스 서버 등 외부 종속성이 다운되었을 때 봇이 오작동하지 않도록 서킷 브레이커 패턴을 반드시 구현한다.

지수 백오프(Exponential Backoff): API Rate Limit 초과 시 즉시 재시도하지 않고, 대기 시간을 점진적으로 늘려가는 지수 백오프 로직을 적용한다.

폴백(Fallback) 데이터 소스: 주력 API(예: 네이버 뉴스) 호출에 실패할 경우, 시스템이 다운되지 않고 즉각적으로 보조 소스(예: 대체 API 또는 기본 차트 매매로 전환)로 전환되는 폴백 로직을 작성한다.

4. Security & MCP Integration Guardrails (보안 및 MCP 연동 제약)
AI 에이전트나 MCP 서버가 외부 뉴스 텍스트를 읽고 처리할 때, 악의적인 프롬프트 인젝션(Prompt Injection)이 실행되지 않도록 입력값을 엄격한 JSON 스키마로 검증하고 필터링한다.

API 키, Secret 토큰 등은 절대 코드 내에 하드코딩하지 않고 환경 변수(.env) 또는 시크릿 매니저를 통해서만 로드한다.

[중요] 리스크 관리 권한의 격리: AI 에이전트(또는 전략 생성 로직)가 자율적으로 "최대 허용 손실폭(MDD)", "최대 포지션 크기", "레버리지 비율" 같은 핵심 리스크 파라미터를 수정하는 코드는 작성하지 않는다. 이 값들은 반드시 하드코딩된 제약 조건 내에서만 움직여야 한다.

5. Code Quality & Testing
모든 함수에는 Type Hint를 명시하고, 핵심 로직(특히 매수/매도 시그널 생성 부분)에는 목적을 설명하는 Docstring을 추가한다.

매매 로직을 수정할 때는 기존 코드에 곧바로 덮어씌우지 말고, Mock 데이터를 활용한 단위 테스트(Unit Test)를 먼저 작성하여 백테스트를 통과하는지 확인한다.

## 아키텍처 개요

```
[키움 Open API+ (32-bit Python)]
        │ ZeroMQ PUB/SUB (< 1ms)
        ▼
[Main Engine (64-bit Python / FastAPI)]  ←→  [TimescaleDB (Docker)]
        │                                           ↑
        ├── REST API + WebSocket ──→ [React Frontend (Vite)]
        │
        ├── Claude API ──→ [AI Agent 시스템]
        │                   ├── Manager Agent (전략 대화)
        │                   ├── Technical Analyst (기술적 분석)
        │                   └── Risk Manager (리스크 평가)
        │
        ├── Alpha Lab ──→ [진화적 알파 팩터 탐색]
        │                   ├── Claude 가설 생성 + SymPy AST 교차/변이
        │                   ├── 벡터 경험 메모리 (ko-sroberta 768-dim)
        │                   └── DoWhy 인과 검증 (플라시보/랜덤 원인/체제 변화)
        │
        ├── Simulation ──→ [금융 월드 모델 (ABM)]
        │                   ├── LOB 가상 거래소 (price-time priority)
        │                   ├── 이질적 에이전트 6종 (Fundamental~LLM)
        │                   └── 시나리오 시뮬레이션 (rate_shock, flash_crash 등)
        │
        ├── MCP Data Bus ──→ [FastMCP 서버 (SSE)]
        │                     ├── 5 tools + 1 resource
        │                     └── 거버넌스 (수량 제한, 행위 화이트리스트)
        │
        ├── 뉴스 수집 ──→ [네이버 금융 / DART / BigKinds]
        │                   └── Claude 감성 분석 → 이벤트 스코어
        │
        ├── 섹터 분석 ──→ [ko-sroberta 임베딩 (768-dim)]
        │                   └── 의미론적 종목 검색 + 클러스터링
        │
        └── 주문 실행 ──→ [한국투자증권 KIS API]
                          └── Token Bucket (15req/s), 자동 토큰 갱신
```

- **Data Pump (32-bit):** 키움 Open API+ 전용. 실시간 틱/호가 수신 + 과거 데이터 수집(COLLECT_HISTORY)하여 ZMQ로 전달. Windows 전용, Docker 불가.
- **Main Engine (64-bit):** FastAPI 기반. 데이터 수신, 전략 판단, 주문 실행, WebSocket 브로드캐스트, AI 에이전트 오케스트레이션.
- **Frontend:** React + Lightweight Charts. 실시간 틱 차트 + 백테스트 + 자동매매 대시보드.

## 기술 스택

### Backend (`stock-mate-backend/`)
- Python 3.12, FastAPI 0.115, Uvicorn
- SQLAlchemy 2.0 (async) + asyncpg, Alembic
- TimescaleDB (PostgreSQL 16 기반, Docker) — hypertable + continuous aggregates
- pydantic-settings 2.7
- pyzmq 26.2 (ZeroMQ, Data Pump 연결)
- pykrx 1.2.4 (KRX 종목/캔들 시딩)
- talipp >= 2.2.0 (실시간 지표: RSI, MACD, BB)
- polars >= 1.0 (백테스트 데이터 처리)
- anthropic >= 0.40 (Claude API — AI 전략 생성/감성 분석/에이전트)
- httpx >= 0.27 (KIS API + 뉴스 수집)
- beautifulsoup4 + lxml (네이버 금융 크롤링)
- sentence-transformers >= 3.0 (종목 임베딩: jhgan/ko-sroberta-multitask, 768-dim)
- scikit-learn >= 1.5 + numpy >= 1.26 (클러스터링, 유사도 계산)
- sympy >= 1.13 (알파 팩터 수식 AST 파싱/교차/변이)
- scipy >= 1.14 (통계 검정)
- dowhy >= 0.11 (인과 추론 — 알파 팩터 미라지 검증)
- networkx >= 3.2 (인과 DAG 구조)
- pyarrow >= 15.0 (Polars↔Pandas 변환)
- fastmcp >= 2.0 (MCP Data Bus SSE 서버)

### Data Pump (`stock-mate-data-pump/`)
- Python 3.11 (32-bit), PyQt5 (COM 이벤트 루프)
- 키움 Open API+ (QAxWidget)
- pyzmq (ZeroMQ PUB 소켓)
- pywinauto (키움 로그인 자동화)
- Windows 네이티브 전용, Docker 불가

### Frontend (`stock-mate-frontend/`)
- React 18 + TypeScript, Vite
- Tailwind CSS v4 + shadcn/ui (new-york style)
- TanStack Query, Zustand
- Lightweight Charts (TradingView)
- cmdk (종목 검색 팔레트)
- react-colorful (MA 색상 피커)
- lucide-react (아이콘)

## 디자인 시스템

**확정 색상 (coolors.co 기반, 변경 금지):**

| 역할 | Hex | OKLCH | 용도 |
|------|-----|-------|------|
| Primary | `#4056F4` | `oklch(0.540 0.235 270.1)` | 브랜드, 버튼, CTA, 링크 |
| Secondary | `#E3B23C` | `oklch(0.788 0.141 85.2)` | 액센트, 하이라이트, 강조 |
| White | `#FFFFFF` | - | 배경, 카드, 여백 |

- 나머지 UI: 흑/백/회(Grayscale)만 사용
- 토스 스타일: 미니멀, 그라디언트 없음, AI풍 비주얼 없음
- 색상 정의 위치: `stock-mate-frontend/src/index.css` `:root` 블록

## 디렉토리 구조

```
stock-mate/
├── CLAUDE.md                          ← 이 파일 (프로젝트 가이드)
│
├── stock-mate-data-pump/              ← 키움 32-bit 데이터 수집기
│   ├── main.py                        ← 엔트리포인트 (QApplication + COLLECT_HISTORY 분기)
│   ├── kiwoom/api.py                  ← 키움 API 래퍼 (로그인, TR, 실시간)
│   ├── kiwoom/constants.py            ← TR 코드, FID, 에러코드, 화면번호
│   ├── publisher/zmq_pub.py           ← ZMQ PUB 소켓 (스레드 안전 큐)
│   ├── services/history_collector.py  ← 과거 데이터 수집 (일봉/분봉/틱)
│   ├── README.md                      ← 설치/실행 가이드
│   ├── requirements.txt               ← PyQt5, pyzmq, python-dotenv, pywinauto
│   └── .env.example                   ← ZMQ 포트, 감시 종목, COLLECT_HISTORY
│
├── stock-mate-backend/                ← FastAPI 백엔드 (별도 git repo)
│   ├── app/
│   │   ├── main.py                    ← FastAPI 앱 (lifespan, USE_SIMULATOR 분기)
│   │   ├── core/
│   │   │   ├── config.py              ← pydantic-settings (DB/ZMQ/CORS/KIS/AI/뉴스)
│   │   │   ├── database.py            ← async engine + session
│   │   │   └── stock_master.py        ← 종목 메모리 캐시 (symbol→name/market)
│   │   ├── models/base.py             ← SQLAlchemy 모델 6개
│   │   ├── schemas/                   ← 5개 (health, account, position, order, stock)
│   │   ├── routers/                   ← 12개 (아래 REST API 섹션 참조)
│   │   ├── services/
│   │   │   ├── candle_service.py      ← 멀티타임프레임 캔들 쿼리 (1m~1M)
│   │   │   ├── candle_writer.py       ← 캔들/틱 데이터 DB 저장
│   │   │   ├── indicator_service.py   ← 기술적 지표 (talipp: RSI, MACD, BB + 동적 SMA)
│   │   │   ├── paper_engine.py        ← 모의투자 체결 엔진 (틱 감시 → 주문 매칭)
│   │   │   ├── tick_simulator.py      ← 개발용 랜덤 틱 시뮬레이터
│   │   │   ├── tick_writer.py         ← 버퍼링 틱 저장 (1초/500건 flush)
│   │   │   ├── ws_manager.py          ← WebSocket 종목별 구독/브로드캐스트
│   │   │   ├── zmq_handler.py         ← ZMQ 메시지 → WebSocket 변환
│   │   │   └── zmq_subscriber.py      ← ZMQ SUB 소켓 (Data Pump 수신)
│   │   ├── backtest/                  ← 백테스트 엔진 (11개 파일)
│   │   │   ├── engine.py              ← 매매 시뮬레이션 (분할매매/확신도 지원)
│   │   │   ├── data_loader.py         ← Polars 기반 캔들 로딩
│   │   │   ├── indicators.py          ← Polars 네이티브 지표 13종
│   │   │   ├── cost_model.py          ← 수수료/슬리피지 모델
│   │   │   ├── metrics.py             ← 성과 지표 18개
│   │   │   ├── runner.py              ← 비동기 백테스트 실행기 (NaN/Inf→None 정제)
│   │   │   ├── ai_strategy.py         ← AI 전략 생성 (Claude API)
│   │   │   ├── presets.py             ← 프리셋 전략 4종
│   │   │   ├── models.py              ← SQLAlchemy BacktestRun 모델
│   │   │   └── schemas.py             ← Pydantic 스키마 39개
│   │   ├── agents/                    ← AI 에이전트 시스템 (6개 파일)
│   │   │   ├── session.py             ← 세션 관리 (메모리, TTL 30분)
│   │   │   ├── manager.py             ← Manager Agent (다중 턴 대화 오케스트레이션)
│   │   │   ├── tools.py               ← Claude Tool Use 7개 도구
│   │   │   ├── orchestrator.py        ← Tool 호출 라우팅
│   │   │   ├── technical.py           ← Technical Analyst Agent
│   │   │   └── risk.py                ← Risk Manager Agent
│   │   ├── news/                      ← 뉴스 수집 + 감성 분석 (8개 파일)
│   │   │   ├── models.py              ← NewsArticle, NewsSentimentDaily 모델
│   │   │   ├── analyzer.py            ← Claude 배치 감성 분석 (NER + market_impact)
│   │   │   ├── anonymizer.py          ← 개체명 익명화 (백테스트 look-ahead bias 방지)
│   │   │   ├── scorer.py              ← 이벤트 스코어 산출
│   │   │   ├── scheduler.py           ← 뉴스 수집 스케줄러
│   │   │   ├── backtest_integration.py ← 백테스트에 뉴스 감성 지표 통합
│   │   │   └── collectors/
│   │   │       ├── naver.py           ← 네이버 금융 크롤러 (API키 불필요)
│   │   │       ├── dart.py            ← DART OpenAPI 클라이언트
│   │   │       └── bigkinds.py        ← BigKinds API 클라이언트
│   │   ├── sector/                    ← 의미론적 섹터 분석 (3개 파일)
│   │   │   ├── embedder.py            ← ko-sroberta-multitask 임베딩 (768-dim)
│   │   │   ├── search.py              ← 코사인 유사도 기반 종목 검색
│   │   │   └── clusterer.py           ← K-means 섹터 클러스터링
│   │   ├── alpha/                     ← 알파 팩터 탐색 (16개 파일)
│   │   │   ├── miner.py              ← EvolutionaryAlphaMiner (Claude 가설 + 변이 루프)
│   │   │   ├── evolution.py           ← SymPy AST 교차/변이/토너먼트 선택
│   │   │   ├── evaluator.py           ← IC/ICIR/Sharpe/MDD 팩터 메트릭
│   │   │   ├── ast_converter.py       ← SymPy → Polars Expression 변환
│   │   │   ├── memory.py             ← ExperienceVectorMemory (768-dim RAG)
│   │   │   ├── scheduler.py           ← AlphaFactoryScheduler (자율 주기 실행)
│   │   │   ├── runner.py             ← 비동기 마이닝 실행기
│   │   │   ├── causal.py             ← FactorMirageFilter (DoWhy 4단계 검증)
│   │   │   ├── causal_runner.py       ← 인과 검증 러너 (배치/단건)
│   │   │   ├── confounders.py         ← 교란 변수 로더 (시장수익률/변동성/금리)
│   │   │   ├── universe.py            ← 유니버스 리졸버 (KOSPI200/KOSDAQ150 등)
│   │   │   ├── backtest_bridge.py     ← 팩터→백테스트 지표 브릿지
│   │   │   ├── portfolio.py           ← 복합 팩터 구성/상관행렬
│   │   │   ├── models.py             ← AlphaMiningRun, AlphaFactor, AlphaExperience
│   │   │   └── schemas.py            ← Pydantic 스키마
│   │   ├── simulation/                ← ABM 금융 시뮬레이션 (5개 파일)
│   │   │   ├── orderbook.py           ← LOB 엔진 (heapq, price-time priority)
│   │   │   ├── agents.py             ← 이질적 에이전트 6종
│   │   │   ├── exchange.py            ← VirtualExchange (시나리오 4종 + 커스텀)
│   │   │   └── runner.py             ← 비동기 시뮬레이션 실행기
│   │   ├── mcp/                       ← MCP Data Bus (3개 파일)
│   │   │   ├── server.py             ← FastMCP 서버 (5 tools + 1 resource)
│   │   │   ├── governance.py          ← 수량 제한, 행위 화이트리스트, 감사
│   │   │   └── bridge.py             ← SSE transport 브릿지
│   │   └── trading/                   ← KIS API 실거래 (5개 파일)
│   │       ├── context.py             ← TradingContext (backtest→paper→real 전환)
│   │       ├── token_bucket.py        ← Token Bucket Rate Limiter (15req/s)
│   │       ├── kis_client.py          ← KIS REST 클라이언트 (토큰 자동 갱신)
│   │       ├── kis_order.py           ← 매수/매도/취소/정정 주문
│   │       └── live_runner.py         ← 실시간 전략 러너 (30초 시그널 체크)
│   ├── scripts/
│   │   ├── seed_stock_masters.py      ← pykrx → stock_masters (KRX 3843개)
│   │   ├── seed_candles.py            ← pykrx → stock_candles (일봉/분봉)
│   │   └── seed_sectors.py            ← 종목 임베딩 생성 (768-dim, 3843개)
│   ├── alembic/                       ← DB 마이그레이션 (9개 버전)
│   ├── docs/ARCHITECTURE.md           ← 상세 설계 문서
│   ├── docker-compose.yml             ← app + TimescaleDB
│   ├── Dockerfile
│   ├── requirements.txt               ← 26개 패키지
│   └── .env.example
│
├── stock-mate-frontend/               ← React 프론트엔드 (별도 git repo)
│   ├── src/
│   │   ├── index.css                  ← 디자인 시스템 색상 변수 (OKLCH)
│   │   ├── main.tsx                   ← QueryClientProvider 래핑
│   │   ├── App.tsx                    ← React Router (10개 페이지)
│   │   ├── pages/                     ← 10개 페이지
│   │   │   ├── Dashboard.tsx          ← 홈 — 계좌 요약, 포지션, 틱 차트
│   │   │   ├── ChartPage.tsx          ← 3컬럼 트레이딩 뷰 (캔들+호가+체결+뉴스)
│   │   │   ├── OrderPage.tsx          ← 모의투자 주문 (폼+호가+보류/체결)
│   │   │   ├── PositionPage.tsx       ← 보유 종목 일람, 평가손익
│   │   │   ├── HistoryPage.tsx        ← 주문내역 조회 (필터)
│   │   │   ├── BacktestPage.tsx       ← 백테스트 (전략 설정+실행+결과+모의/실거래 전환)
│   │   │   ├── TradingPage.tsx        ← 자동매매 (KIS 계좌+컨텍스트+세션+매매로그)
│   │   │   ├── AlphaLabPage.tsx       ← 알파 팩터 탐색 (마이닝+팩터목록+팩토리+인과검증)
│   │   │   ├── SimulationPage.tsx     ← ABM 시뮬레이션 (2탭: 설정/MCP 대시보드)
│   │   │   └── SettingsPage.tsx       ← Paper/Real 전환, 감시 종목
│   │   ├── components/
│   │   │   ├── layout/                ← AppLayout, Header, Sidebar
│   │   │   ├── chart/                 ← CandleChart, TickChart, MASettingsPanel
│   │   │   ├── backtest/              ← 9개 (StrategyChat, Chat, Config, Progress, SummaryCards, EquityCurve, TradeTable, History, AgentAnalysis)
│   │   │   ├── order/                 ← OrderBook, OrderForm
│   │   │   ├── stock/                 ← StockSearch (cmdk), SectorSearch (임베딩)
│   │   │   ├── trade/                 ← TradeHistory
│   │   │   ├── news/                  ← SentimentBadge, NewsSentimentChart, NewsPanel
│   │   │   ├── trading/               ← ModeSwitch, LiveStatus, ContextPanel
│   │   │   ├── alpha/                 ← 7개 (MiningConfig, FactorList, FactorDetail, FactoryPanel, CausalResult, CorrelationMatrix, ExperienceLog)
│   │   │   ├── simulation/            ← 7개 (Config, Progress, PriceChart, LOB, Metrics, History, McpDashboard)
│   │   │   └── ui/                    ← 18개 shadcn/ui 컴포넌트 (+ color-picker)
│   │   ├── hooks/
│   │   │   ├── queries/               ← TanStack Query 훅 10파일 + index.ts
│   │   │   │   ├── use-account.ts, use-positions.ts, use-orders.ts
│   │   │   │   ├── use-ticks.ts       ← useCandles, useTicks, useStockList
│   │   │   │   ├── use-paper.ts       ← 모의투자 6개 훅
│   │   │   │   ├── use-backtest.ts    ← 백테스트 6개 훅
│   │   │   │   ├── use-agent.ts       ← AI 에이전트 세션 훅
│   │   │   │   ├── use-news.ts        ← 뉴스/감성 분석 훅
│   │   │   │   ├── use-sector.ts      ← 섹터 검색 훅
│   │   │   │   └── use-trading.ts     ← KIS 자동매매 11개 훅
│   │   │   ├── use-websocket.ts       ← useTickStream, useOrderBookStream
│   │   │   └── use-chart-layout.ts    ← 차트 드래그 리사이즈
│   │   ├── stores/
│   │   │   ├── use-app-store.ts       ← sidebarOpen, tradingMode, selectedSymbol
│   │   │   └── use-tick-store.ts      ← ticks[symbol][], orderBooks[symbol]
│   │   ├── api/
│   │   │   ├── client.ts             ← fetch 래퍼
│   │   │   ├── index.ts              ← API 엔드포인트 함수 re-export
│   │   │   ├── constants.ts          ← 호가 생성 유틸 (fallback)
│   │   │   ├── agents.ts             ← AI 에이전트 API
│   │   │   ├── news.ts               ← 뉴스/감성 API
│   │   │   ├── sector.ts             ← 섹터 검색 API
│   │   │   └── trading.ts            ← KIS 자동매매 API
│   │   ├── lib/
│   │   │   ├── utils.ts              ← cn() (clsx + tailwind-merge)
│   │   │   ├── format.ts             ← formatKRW, formatNumber, formatPercent
│   │   │   └── interval.ts           ← intervalToSeconds
│   │   └── types/
│   │       ├── index.ts              ← 32개+ 인터페이스 (Stock, Account, Trading 등)
│   │       ├── agent.ts              ← AgentSession, StrategySchema, ToolResult
│   │       └── news.ts               ← NewsArticle, SentimentScore 등
│   ├── .env                           ← VITE_API_URL, VITE_WS_URL
│   ├── components.json                ← shadcn/ui 설정
│   └── vite.config.ts
```

## 실행 방법

### 프론트엔드
```bash
cd stock-mate-frontend
npm install
npm run dev          # http://localhost:5173
```

### 백엔드
```bash
cd stock-mate-backend
docker-compose up -d --build                                    # FastAPI + TimescaleDB 기동
docker-compose run --rm app alembic upgrade head                # DB 마이그레이션 적용
docker-compose run --rm app python -m scripts.seed_sectors      # 종목 임베딩 시딩 (~10분)
curl http://localhost:8007/health                               # 헬스체크
```

### 로컬 개발 (Docker 없이)
```bash
cd stock-mate-backend
pip install -r requirements.txt
python main.py       # http://localhost:8007 (TimescaleDB는 별도 실행 필요)
```

## 환경변수 (`.env`)

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `POSTGRES_HOST` | `localhost` | DB 호스트 |
| `POSTGRES_PORT` | `5432` | DB 포트 |
| `POSTGRES_USER` | `stockmate` | DB 사용자 |
| `POSTGRES_PASSWORD` | `stockmate` | DB 비밀번호 |
| `POSTGRES_DB` | `stockmate` | DB 이름 |
| `CORS_ORIGINS` | `["*"]` | CORS 허용 오리진 (JSON 배열) |
| `ZMQ_HOST` | `127.0.0.1` | ZMQ 호스트 |
| `ZMQ_PORT` | `5555` | ZMQ 포트 |
| `USE_SIMULATOR` | `false` | `true`=틱 시뮬레이터, `false`=키움 ZMQ |
| `ANTHROPIC_API_KEY` | — | Claude API 키 (AI 전략/감성 분석) |
| `AGENT_MODEL` | `claude-sonnet-4-20250514` | 에이전트 모델 |
| `AGENT_SESSION_TTL_MINUTES` | `30` | 에이전트 세션 TTL |
| `AGENT_MAX_TOKENS` | `4000` | 에이전트 최대 토큰 |
| `KIS_APP_KEY` | — | 한국투자증권 앱키 |
| `KIS_APP_SECRET` | — | 한국투자증권 시크릿 |
| `KIS_ACCOUNT_NO` | — | 계좌번호 (`XXXXXXXX-XX`) |
| `KIS_BASE_URL` | `https://openapi.koreainvestment.com:9443` | KIS 실전 URL |
| `KIS_MOCK_URL` | `https://openapivts.koreainvestment.com:29443` | KIS 모의 URL |
| `DART_API_KEY` | — | DART OpenAPI 키 (선택) |
| `BIGKINDS_API_KEY` | — | BigKinds API 키 (선택) |
| `NEWS_COLLECT_HOUR` | `18` | 뉴스 수집 시각 (KST) |
| `NEWS_BATCH_SIZE` | `10` | Claude 배치 분석 단위 |
| `ALPHA_IC_THRESHOLD_PASS` | `0.03` | 알파 팩터 IC 통과 기준 |
| `ALPHA_MAX_MUTATION_DEPTH` | `3` | 변이 재시도 최대 깊이 |
| `ALPHA_FACTORY_INTERVAL_MINUTES` | `30` | 팩토리 사이클 간격 |
| `ALPHA_FACTORY_MAX_ITERATIONS` | `5` | 사이클당 Claude 가설 생성 횟수 |
| `ALPHA_FACTORY_CROSSOVER_ENABLED` | `true` | 유전 교차 활성화 |
| `ALPHA_FACTORY_TOURNAMENT_K` | `3` | 토너먼트 선택 크기 |
| `CAUSAL_AUTO_VALIDATE` | `true` | 마이닝 완료 후 자동 인과 검증 |
| `CAUSAL_PLACEBO_THRESHOLD` | `0.5` | 플라시보 검증 임계값 |
| `CAUSAL_RANDOM_CAUSE_THRESHOLD` | `0.5` | 랜덤 원인 검증 임계값 |
| `CAUSAL_NUM_SIMULATIONS` | `100` | DoWhy 시뮬레이션 횟수 |
| `MCP_ENABLED` | `false` | MCP Data Bus 활성화 |
| `MCP_SSE_PORT` | `8009` | MCP SSE 서버 포트 |
| `SIMULATION_DEFAULT_STEPS` | `1000` | ABM 기본 시뮬레이션 스텝 |

## DB 스키마 (TimescaleDB)

| 테이블 | 주요 컬럼 | 비고 |
|--------|----------|------|
| `accounts` | id, mode(REAL/PAPER), total_capital, current_balance | 계좌/자산 |
| `positions` | id, symbol, mode, qty, avg_price | 보유 종목 |
| `orders` | order_id(UUID), symbol, side, type, price, qty, status, mode | 주문/체결 |
| `stock_ticks` | ts/id(복합PK), symbol, price, volume | TimescaleDB hypertable |
| `stock_masters` | symbol(PK), name, market, sector, sub_sector, description, embedding(JSON) | KRX 전체 종목 마스터 (3843개) + 768-dim 임베딩 |
| `stock_candles` | id, symbol, dt, interval(1m~1M), open/high/low/close/volume | 멀티타임프레임 캔들 |
| `backtest_runs` | id(UUID), strategy_name, strategy_json, status, metrics, equity_curve, trades_summary | 백테스트 실행 결과 (JSON 필드) |
| `news_articles` | id, symbol, source, title, content, published_at, sentiment_score, magnitude, market_impact | 뉴스 기사 + 감성 분석 |
| `news_sentiment_daily` | id, symbol, date, avg_sentiment, article_count, event_score | 일별 종목 감성 집계 |
| `alpha_mining_runs` | id(UUID), name, context(JSON), config(JSON), status, progress, factors_found, iteration_logs(JSON) | 알파 마이닝 실행 |
| `alpha_factors` | id(UUID), mining_run_id(FK), expression_str, generation, ic_mean/icir/sharpe, causal_robust, parent_ids(JSON) | 발견된 알파 팩터 |
| `alpha_experiences` | id(UUID), factor_id(FK), expression_str, hypothesis, embedding(JSON 768-dim), ic_mean, success | 벡터 경험 메모리 |
| `stress_test_runs` | id(UUID), scenario, config(JSON), status, metrics(JSON), price_history(JSON) | ABM 시뮬레이션 결과 |
| `mcp_audit_logs` | id(UUID), tool_name, action, params(JSON), approved, timestamp | MCP 감사 로그 |

**TimescaleDB 객체:**
- Continuous Aggregates: `candles_1m`, `candles_1h`, `candles_1d` (자동 갱신 정책)
- 압축 정책 (7일), 보존 정책 (30일)

**Alembic 마이그레이션 (9개):**
1. `f6ea9be370b3` — 초기 스키마 (Account, Position, Order)
2. `a1b2c3d4e5f6` — TimescaleDB (hypertable + continuous aggregates)
3. `64ea2ba3255f` — stock_masters + stock_candles
4. `b7c8d9e0f1a2` — backtest_runs
5. `c3d4e5f6a7b8` — news_articles + news_sentiment_daily
6. `d5e6f7a8b9c0` — stock_masters에 sector/description/embedding 컬럼
7. `e7f8a9b0c1d2` — alpha_mining_runs + alpha_factors
8. `f8g9h0i1j2k3` — alpha_experiences (벡터 경험 메모리)
9. `g9h0i1j2k3l4` — stress_test_runs + mcp_audit_logs

## REST API 엔드포인트

| 라우터 | 주요 엔드포인트 | 설명 |
|--------|----------------|------|
| `health` | `GET /health` | 헬스체크 |
| `accounts` | `GET /accounts`, `/accounts/{id}` | 계좌 조회 |
| `positions` | `GET /positions` | 보유 종목 |
| `orders` | `GET /orders` | 주문 내역 |
| `stocks` | `GET /stocks`, `/stocks/search` | 종목 마스터 검색 |
| `paper` | `POST /paper/order`, `DELETE /paper/order/{id}` | 모의투자 주문 |
| `backtest` | `POST /backtest/run`, `GET /backtest/run/{id}`, `GET /backtest/runs` | 백테스트 실행/조회 |
| `agents` | `POST /agents/message` | AI 에이전트 다중 턴 대화 |
| `news` | `GET /news/articles`, `/news/sentiment` | 뉴스 기사/감성 조회 |
| `sector` | `POST /sector/search` | 의미론적 종목 검색 |
| `trading` | `POST /trading/start`, `/trading/stop`, `/trading/order/*` | KIS 자동매매 |
| `alpha` | `POST /alpha/mine`, `GET /alpha/factors`, `POST /alpha/factory/*`, `POST /alpha/factor/{id}/validate` | 알파 팩터 탐색/팩토리/인과검증 |
| `simulation` | `POST /simulation/run`, `GET /simulation/run/{id}`, `GET /simulation/runs` | ABM 시뮬레이션 |
| `ws` | `WS /ws/{symbol}` | 실시간 틱/호가 WebSocket |

## AI 에이전트 시스템

### 아키텍처

```
[사용자 자연어] → [Manager Agent]
                    │ Claude Tool Use (native)
                    ├── list_indicators()       ← 지원 지표 13종 조회
                    ├── suggest_parameters()    ← 종목별 파라미터 제안
                    ├── draft_strategy()        ← 전략 JSON 초안 생성
                    ├── validate_strategy()     ← 전략 유효성 검증
                    ├── ask_technical_analyst()  ← 기술적 분석 전문 Agent
                    ├── ask_risk_manager()       ← 리스크 평가 전문 Agent
                    └── search_sector_stocks()   ← 의미론적 섹터 검색
```

- **Manager Agent:** 다중 턴 대화 오케스트레이션. 사용자 의도 파악 → 도구 호출 → 전략 수립
- **Technical Analyst:** 기술적 분석 인사이트 (지표 조합, 시그널 해석)
- **Risk Manager:** 리스크 평가 (MDD 예측, 포지션 사이징 권고, 손절 설정)
- **세션:** 메모리 dict (TTL 30분), 대화 히스토리 유지

## 뉴스 감성 분석

### 수집 소스 (3개)
| 소스 | 방식 | API 키 | 비고 |
|------|------|--------|------|
| 네이버 금융 | httpx + BeautifulSoup4 크롤링 | 불필요 | 기본 소스 (항상 동작) |
| DART | OpenAPI REST | 필요 | 금융감독원 전자공시 |
| BigKinds | REST API | 필요 (승인 1~3일) | 한국언론진흥재단 뉴스 빅데이터 |

### 분석 파이프라인
1. **수집:** 종목별 뉴스 기사 크롤링/API 호출
2. **감성 분석:** Claude 배치 분석 → `sentiment_score`, `magnitude`, `market_impact`, NER entities
3. **이벤트 스코어:** `weighted_avg(sentiment × impact) × log(count + 1) / normalization`
4. **백테스트 통합:** `indicators.py`에 `sentiment_score`, `article_count`, `event_score` 등록 → T+1 shift (look-ahead bias 방지)
5. **익명화:** `anonymizer.py` — stock_masters.name 기반 사전, `[COMPANY_A]` 치환

## KIS API 자동매매

### 모드 전환 흐름
```
[백테스트 완료] → "모의투자 전환" 버튼 → TradingContext(mode=paper) 생성
                → "실거래 전환" 버튼 → TradingContext(mode=real) 생성
                → /trading 페이지에서 세션 시작/중지
```

### TradingContext
전략+환경 통합 객체. `backtest → paper → real` 모드 전환 지원.
- `from_backtest_run()`: 백테스트 결과에서 컨텍스트 생성
- 메모리 저장 (서버 재시작 시 초기화)

### KIS 클라이언트
- httpx 비동기, 자동 토큰 갱신 (24h)
- Token Bucket Rate Limiter (15req/s)
- Mock(V prefix) vs Real(T prefix) TR ID 자동 전환
  - 매수: `VTTC0802U` / `TTTC0802U`
  - 매도: `VTTC0801U` / `TTTC0801U`
  - 취소/정정: `VTTC0803U` / `TTTC0803U`

### 실시간 러너
- 30초 간격 시그널 체크 루프
- KIS 잔고 동기화 → 리스크 관리 (손절/트레일링) → 시그널 기반 주문

## 백테스트 시스템

### 아키텍처 흐름

```
[사용자 입력]                      [프리셋 4종]
   │ 자연어                           │ JSON
   ▼                                  ▼
[AI Strategy] ──Claude API──→  [BacktestStrategy]
                                      │
                          ┌───────────┼───────────┐
                          ▼           ▼           ▼
                    [Data Loader] [Indicators] [Cost Model]
                     Polars DF     13개 지표    수수료/슬리피지
                          │           │           │
                          └─────┬─────┘           │
                                ▼                 │
                          [Engine] ←──────────────┘
                           시그널 생성 → 포트폴리오 시뮬레이션
                                │
                                ▼
                          [Metrics]
                           18개 성과 지표
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              [DB 저장]   [WebSocket]   [REST 응답]
              BacktestRun  진행률 푸시   결과 반환
```

### 비동기 실행 흐름

1. `POST /backtest/run` → DB에 `PENDING` 행 생성 → `asyncio.create_task()` 스폰 → 즉시 `202` 반환
2. Runner가 `RUNNING`으로 전환 → Engine 실행 → 진행률 WebSocket 푸시 (`backtest:{run_id}`)
3. 완료 시 `COMPLETED` + metrics/equity_curve/trades 저장, 실패 시 `FAILED` + error_message
4. 프론트엔드: `useBacktestRun(runId)` 2초 폴링 + WebSocket 프로그레스 바

### 엔진 상세 (`engine.py`)

**핵심 데이터 구조:**
- `_ScaleEntry`: 개별 매수 건 (date, price, qty, step="B1"/"B2")
- `_Position`: 포지션 (entries[], highest_price, conviction, target_qty, scale_in_count, has_partial_exited)
- `BacktestResult`: 결과 (trades[], equity_curve[], metrics{})

**`run_backtest()` 실행 단계:**

| 단계 | 처리 |
|------|------|
| Phase 1 | `data_loader.load_candles()` — Polars DF로 OHLCV 로딩, price≤0 필터 |
| Phase 2 | `generate_signals()` — 종목별 시그널 생성 (1=BUY, -1=SELL, 0=WAIT) |
| Phase 3 | 일별 루프: (1)고가 갱신 → (2)손절/트레일링 → (3)부분익절 → (4)매도 시그널 → (5)추가매수 → (6)신규매수 → (7)자산 계산 |
| Phase 4 | 잔여 포지션 강제 청산 |
| Phase 5 | `compute_metrics()` — 18개 성과 지표 산출 |

**시그널 생성:**
- `_build_condition_expr()`: 조건 → Polars boolean 표현식 (연산자: `>`, `>=`, `<`, `<=`, `==`, `!=`)
- `_combine_conditions()`: AND/OR 로직으로 조합
- Look-ahead bias 방지: 시그널은 당일 봉 기준 → 다음 봉 시가에서 체결

### 기술적 지표 (`indicators.py` — Polars 네이티브)

| 함수 | 생성 컬럼 | 파라미터 (기본값) |
|------|----------|------------------|
| `add_sma` | `sma_{period}` | period(20), col("close") |
| `add_ema` | `ema_{period}` | period(20), col("close") |
| `add_rsi` | `rsi` | period(14) |
| `add_macd` | `macd_line`, `macd_signal`, `macd_hist` | fast(12), slow(26), signal(9) |
| `add_bb` | `bb_upper`, `bb_middle`, `bb_lower` | period(20), std(2.0) |
| `add_atr` | `true_range`, `atr_{period}` | period(14) |
| `add_volume_ratio` | `volume_ratio` | period(20) |
| `add_price_change_pct` | `price_change_pct` | period(1) |
| `add_consec_decline` | `consec_decline_{days}` | days(3) |
| `add_open_gap_pct` | `open_gap_pct` | — |
| (크로스) | `golden_cross`, `dead_cross` | fast_period(5), slow_period(20) |

`ensure_indicators(df, conditions)`: 전략의 조건 목록에서 필요한 지표를 자동 추출 + 중복 제거 후 일괄 추가.

### 비용 모델 (`cost_model.py`)

| 항목 | 기본값 | 계산 |
|------|--------|------|
| 매수 수수료 | 0.015% | `price × (1 + slippage) × (1 + buy_commission)` |
| 매도 수수료 | 0.215% (수수료 0.015% + 거래세 0.20%) | `price × (1 - slippage) × (1 - sell_commission)` |
| 슬리피지 | 0.1% | 매수 시 +, 매도 시 - |

### 포지션 사이징 모드

| 모드 | conviction 산출 | 용도 |
|------|----------------|------|
| `fixed` | 항상 1.0 | 고정 비중 |
| `conviction` | 매수 조건별 가중 합산 (weights 딕셔너리) | 확신도 높을수록 큰 포지션 |
| `atr_target` | `min(1.0, 0.02 / max(atr_pct, 0.001))` | 변동성 작을수록 큰 포지션 |
| `kelly` | Half-Kelly fraction | 이론적 최적 비중 (미완성) |

자본 배분: `allocation = initial_capital × position_size_pct × conviction`
스케일링 시: `allocation × initial_pct` (예: 50%만 1차 진입)

### 분할매매 (Scaling)

| 단계 | 트리거 | scale_step | 설명 |
|------|--------|------------|------|
| 1차 진입 | BUY 시그널 | `B1` | `target_qty = qty / initial_pct` 설정 |
| 추가 매수 | 평단 대비 `scale_in_drop_pct`% 하락 | `B2` | `target_qty - 현재qty` 만큼 매수 (최대 `max_scale_in`회) |
| 부분 익절 | 평단 대비 `partial_exit_gain_pct`% 수익 | `S-HALF` | `total_qty × partial_exit_pct` 매도 (1회 한정) |
| 전량 매도 | SELL 시그널 / 손절 / 트레일링 | — / `S-STOP` / `S-TRAIL` | 잔여 전량 청산 |

FIFO 기반: `_reduce_entries()`로 부분 매도 시 `_ScaleEntry` 리스트 선입선출 감소.

### 리스크 관리

| 기능 | 트리거 조건 | exit scale_step |
|------|-----------|----------------|
| 고정 손절 | `(price - avg_price) / avg_price × 100 ≤ -stop_loss_pct` | `S-STOP` |
| 트레일링 스탑 | `(highest_price - price) / highest_price × 100 ≥ trailing_stop_pct` | `S-TRAIL` |
| ATR 스탑 | `price ≤ avg_price - atr × atr_stop_multiplier` | `S-STOP` |

일별 루프에서 리스크 체크 → 매도 시그널보다 **먼저** 처리 (손절 우선).

### 성과 지표 (`metrics.py` — 18개)

| 지표 | 계산 | 비고 |
|------|------|------|
| `total_return` | `(최종 - 초기) / 초기 × 100` | % |
| `total_return_amount` | `최종 - 초기` | 원 |
| `annualized_return` | `((최종/초기)^(252/일수) - 1) × 100` | 연환산 % |
| `mdd` | 고점 대비 최대 낙폭 | % |
| `mdd_amount` | 고점 대비 최대 낙폭 | 원 |
| `win_rate` | `승리 / 전체 × 100` | % |
| `profit_factor` | `총 수익 / |총 손실|` | 비율 |
| `sharpe_ratio` | 일별 수익률 표준편차 연환산 (무위험 3%) | — |
| `total_trades` | 청산된 거래 수 | 건 |
| `avg_holding_days` | 평균 보유 기간 | 일 |
| `avg_win` / `avg_loss` | 평균 수익률 / 손실률 | % |
| `max_consecutive_wins` / `losses` | 최대 연승 / 연패 | 건 |
| `avg_conviction` | 평균 확신도 | 0-1 |
| `scale_in_count` | 분할매수 횟수 (`B2`) | 건 |
| `partial_exit_count` | 부분익절 횟수 (`S-HALF`) | 건 |
| `stop_loss_count` | 손절 횟수 (`S-STOP`) | 건 |

### AI 전략 생성 (`ai_strategy.py`)

- **모델:** `claude-sonnet-4-20250514`
- **입력:** 사용자 자연어 프롬프트 (예: "RSI 30 이하이고 거래량 2배 이상이면 매수")
- **시스템 프롬프트:** 지원 지표 11종 + JSON 출력 스키마 + 조건부 고급 설정 포함
- **출력:** `{strategy: StrategySchema, explanation: string}`
- **파싱:** 마크다운 코드블록 → fallback `{...}` 추출

### 프리셋 전략 (`presets.py` — 4종)

| # | 이름 | 매수 조건 | 매도 조건 | 고급 설정 |
|---|------|----------|----------|----------|
| 1 | RSI 과매도 반등 | RSI(14) ≤ 30 | RSI(14) ≥ 70 | — |
| 2 | 골든크로스 | SMA(5) > SMA(20) 크로스 | 데드크로스 | — |
| 3 | MACD 크로스 | MACD hist > 0 크로스 | MACD hist < 0 | — |
| 4 | RSI 확신도+분할매매 | RSI ≤ 30 AND 거래량비 ≥ 1.5 | RSI ≥ 70 | conviction(rsi:0.6, vol:0.4) + scaling(50%진입, 3%하락추매, 5%익절) + 손절5%/트레일링2% |

## 알파 팩터 탐색 (Alpha Lab)

### 아키텍처

```
[사용자 설정 (유니버스/기간/맥락)]
   │
   ▼
[AlphaFactoryScheduler] ─── interval_minutes 주기 반복
   │
   ├─ 사이클마다:
   │   ├─ ExperienceVectorMemory.load_cache(db) ← DB 경험 로드 (RAG)
   │   ├─ EvolutionaryAlphaMiner (매번 새 인스턴스)
   │   │   ├── Claude 가설 생성 (max_iterations회)
   │   │   ├── IC/직교성 평가 → 변이 재시도 (max_depth회)
   │   │   └── 사이클 내 유전 교차 (enable_crossover 시)
   │   ├─ DB 저장: AlphaFactor + AlphaExperience
   │   └─ (선택) DoWhy 인과 검증
   │
   ▼
[AlphaFactor] → 백테스트 브릿지 → 기존 BacktestEngine
```

### 실행 경로 3가지

| 경로 | 엔드포인트 | 설명 |
|------|-----------|------|
| 단발 마이닝 | `POST /alpha/mine` | 1회 마이닝 실행, 벡터 메모리 미사용, 교차 미사용 |
| 알파 팩토리 | `POST /alpha/factory/start` | 주기적 자율 마이닝, 벡터 메모리 RAG, 교차 가능 |
| 인과 검증 | `POST /alpha/factor/{id}/validate` | DoWhy 4단계 인과 검증 (수동 트리거) |

### 진화 루프 (`miner.py`)

1. **Claude 가설 생성** → SymPy 수식 파싱 → Polars Expression 변환
2. **IC 평가** → 기준 미달 시 Claude 변이 요청 (최대 `ALPHA_MAX_MUTATION_DEPTH`회)
3. **직교성 필터** → 기존 발견 팩터와 Pearson 상관 < `orthogonality_threshold`
4. **유전 교차** (`evolution.py`) → 토너먼트 선택 + AST 서브트리 교환 (사이클 내 한정)

### 인과 검증 (`causal.py` — DoWhy 4단계)

| 단계 | 검증 | 통과 조건 |
|------|------|----------|
| 1 | 인과 효과 추정 (ATE) | 선형 회귀 기반 처리 효과 |
| 2 | 플라시보 검증 | 랜덤 팩터의 ATE / 원본 ATE < threshold |
| 3 | 랜덤 원인 검증 | 교란 변수 추가 후 ATE 변화 < threshold |
| 4 | 체제 변화 검증 | 전/후반부 ATE 비율 확인 |

교란 변수: 시장 수익률, 시장 변동성, 기준 금리, 섹터 ID (`confounders.py`)

### 벡터 경험 메모리 (`memory.py`)

- ko-sroberta 768-dim 임베딩: `"{가설} {수식}"` 텍스트 인코딩
- DB 영속 (`alpha_experiences` 테이블) — 사이클 간 유지
- Claude 프롬프트에 유사 성공 5개 + 유사 실패 3개 RAG 제공
- 직교성 체크: 기존 성공 팩터와 코사인 유사도 < threshold

### 현재 한계 (구조적)

- **사이클 간 모집단 단절**: `EvolutionaryAlphaMiner`가 매 사이클 `discovered=[]`로 초기화, 이전 팩터 로드 없음
- **교차 범위**: 현재 사이클 내에서 발견된 팩터끼리만 교차 가능 (사이클 간 교차 없음)
- **선택 압력 부재**: 엘리티즘/적합도 기반 도태 메커니즘 없음
- **변이**: `evolution.py`의 AST `mutate()` 함수가 실제 미사용 (Claude API 변이만 사용)
- 실질적으로 "Claude 반복 가설 생성 + RAG 경험 참조" 시스템에 가까움

### 피처 변수 (`ast_converter.py`)

| 변수명 | Polars 컬럼 | 설명 |
|--------|-----------|------|
| close, open, high, low, volume | 동일 | OHLCV 원본 |
| sma_20, ema_20 | 동일 | 이동평균 |
| rsi | rsi | 14일 RSI |
| volume_ratio | volume_ratio | 20일 평균 대비 거래량비 |
| atr, atr_14 | atr_14 | ATR |
| macd, macd_hist | macd_hist | MACD 히스토그램 |
| bb_upper, bb_lower, bb_width | 동일 | 볼린저 밴드 |
| price_change_pct | price_change_pct | 전일 대비 변화율 |

## ABM 시뮬레이션 (금융 월드 모델)

### 구성

- **LOB 엔진** (`simulation/orderbook.py`): heapq 기반 price-time priority, lazy cancel
- **이질적 에이전트 6종** (`simulation/agents.py`): Fundamental, Chartist, Noise, LLM, Strategy, MarketMaker
- **가상 거래소** (`simulation/exchange.py`): VirtualExchange, 시나리오 실행
- **시나리오 4종**: rate_shock, liquidity_crisis, flash_crash, supply_chain + Claude 커스텀

### MCP Data Bus (`mcp/`)

- FastMCP 서버 (SSE transport): 5 tools + 1 resource
- 거버넌스: 수량 제한, 행위 화이트리스트, human approval 게이트
- 감사 로그: `mcp_audit_logs` 테이블

## 코딩 컨벤션

### Python (Backend)
- async/await 우선 사용
- Pydantic v2 모델 for 요청/응답 스키마
- SQLAlchemy 2.0 스타일 (mapped_column)
- 라우터별 파일 분리 (`app/routers/`)
- 환경변수는 반드시 `app/core/config.py`의 Settings 클래스를 통해 접근
- `USE_SIMULATOR` 플래그: true = 틱 시뮬레이터(개발용), false = 키움 ZMQ 수신(운영)

### TypeScript (Frontend)
- strict 모드
- `@/` path alias 사용 (예: `@/components/ui/button`)
- 컴포넌트: function 선언문
- 상태: 서버 상태 = TanStack Query, 클라이언트 상태 = Zustand
- WebSocket 데이터: 차트 배열은 항상 `slice(-1000)`으로 최근 1000개만 유지
- Zustand selector stable reference: `?? []` 대신 모듈 레벨 `const EMPTY: never[] = []` 사용 (무한 렌더 방지)
- WebSocket 훅에서 `useTickStore.getState().addTick()` 직접 호출 (useEffect deps 안정화)
- TanStack Query: queryKey 변경 시 `placeholderData: keepPreviousData`로 컴포넌트 언마운트/깜빡임 방지

## 구현 로드맵

### 완료

**인프라/기반**
- [x] 프로젝트 기반 구조 (프론트/백/데이터펌프 스캐폴딩)
- [x] 디자인 시스템 색상 확정 및 주입
- [x] Docker (FastAPI + TimescaleDB)
- [x] DB 모델 + Alembic 마이그레이션 (9개 버전)
- [x] 문서 (CLAUDE.md, ARCHITECTURE.md)

**Data Pump**
- [x] 키움 32-bit Data Pump (코드 작성 + 키움 환경 설치 완료)
- [x] ZeroMQ PUB/SUB 연동 (통합 완료)
- [x] 과거 데이터 수집 (history_collector.py — 일봉/분봉/틱)
- [x] pywinauto 자동 로그인

**백엔드 — 기반 서비스**
- [x] CRUD API (accounts, positions, orders, stocks 라우터)
- [x] 종목 마스터 DB화 (pykrx → stock_masters 3843개)
- [x] 캔들 시딩 (pykrx → stock_candles 일봉)
- [x] TimescaleDB 전환 (hypertable + 3 continuous aggregates)
- [x] 실시간 틱 DB 저장 (tick_writer.py 버퍼링)
- [x] 멀티타임프레임 캔들 쿼리 (candle_service.py, 1m~1M)
- [x] 기술적 지표 서버사이드 (talipp: RSI, MACD, BB + 동적 SMA)
- [x] WebSocket 인프라 (ws_manager.py, 종목별 구독)
- [x] 모의투자 엔진 (paper_engine.py — 틱 감시 → 주문 매칭)
- [x] 개발용 틱 시뮬레이터 (tick_simulator.py, USE_SIMULATOR 플래그)

**백엔드 — 백테스트**
- [x] 백테스트 엔진 (data_loader, indicators, engine, cost_model, metrics, runner)
- [x] AI 전략 생성 (ai_strategy.py, Claude API 연동)
- [x] 프리셋 전략 4종 (RSI 과매도, 골든크로스, MACD, RSI 확신도+분할매매)
- [x] 확신도 기반 비중 조절 + 분할매매 (engine.py 전면 개편)

**백엔드 — AI 에이전트**
- [x] Manager Agent 다중 턴 대화 (session.py, manager.py, tools.py)
- [x] Claude Tool Use 7개 도구 (지표 조회, 파라미터 제안, 전략 초안/검증, 전문 Agent 호출, 섹터 검색)
- [x] Technical Analyst Agent + Risk Manager Agent (전문 분석)
- [x] 오케스트레이터 (Tool 호출 라우팅)

**백엔드 — 뉴스 감성 분석**
- [x] 3소스 뉴스 수집기 (네이버 금융, DART, BigKinds)
- [x] Claude 배치 감성 분석 (sentiment_score, magnitude, market_impact, NER)
- [x] 이벤트 스코어 산출 + 백테스트 통합 (T+1 shift)
- [x] 개체명 익명화 (look-ahead bias 방지)

**백엔드 — 섹터 분석**
- [x] 종목 임베딩 (jhgan/ko-sroberta-multitask, 768-dim, 3843개)
- [x] 코사인 유사도 기반 의미론적 종목 검색
- [x] K-means 섹터 클러스터링

**백엔드 — KIS API 자동매매**
- [x] KIS REST 클라이언트 (httpx, 토큰 자동 갱신 24h)
- [x] Token Bucket Rate Limiter (15req/s)
- [x] 매수/매도/취소/정정 주문 (Mock/Real TR ID 자동 전환)
- [x] TradingContext (backtest→paper→real 모드 전환)
- [x] 실시간 전략 러너 (30초 시그널 체크 + KIS 잔고 동기화)

**프론트엔드 — 기반 UI**
- [x] 대시보드 페이지 (계좌 요약, 포지션 테이블, 틱 차트)
- [x] 3컬럼 트레이딩 차트 뷰 (캔들+호가 10단계+체결정보, 드래그 리사이즈)
- [x] 멀티타임프레임 차트 (월봉~1분봉+틱, 8인터벌)
- [x] 기술적 지표 차트 (RSI, MACD, BB 토글)
- [x] 이동평균선 커스터마이즈 (MASettingsPanel, ColorPicker, 범례 오버레이)
- [x] 종목 검색 (cmdk 팔레트, 3843개 클라이언트 필터)
- [x] 모의투자 주문 페이지 (OrderForm + 호가 연동)
- [x] 보유종목/주문내역/설정 페이지
- [x] 실시간 WebSocket (useTickStream, useOrderBookStream → Zustand)

**프론트엔드 — 백테스트**
- [x] 백테스트 페이지 (9개 컴포넌트: StrategyChat, Chat, Config, Progress, SummaryCards, EquityCurve, TradeTable, History, AgentAnalysis)
- [x] 프리셋 선택 + AI 자연어 전략 생성
- [x] 고급설정 Collapsible (포지션 사이징/분할매매/리스크)
- [x] WebSocket 진행률 + 2초 폴링

**프론트엔드 — AI/뉴스/섹터/매매**
- [x] AI 에이전트 대화 UI (StrategyChat, AgentAnalysis)
- [x] 뉴스 패널 (SentimentBadge, NewsSentimentChart, NewsPanel)
- [x] 섹터 검색 (SectorSearch — 임베딩 기반)
- [x] 자동매매 페이지 (TradingPage — ModeSwitch, LiveStatus, ContextPanel)
- [x] 백테스트→모의투자/실거래 전환 버튼

**백엔드 — 알파 팩터 탐색 (Alpha Lab)**
- [x] 진화적 알파 마이너 (miner.py — Claude 가설 생성 + 변이 루프)
- [x] SymPy AST 교차/변이 (evolution.py — 토너먼트 선택, 서브트리 교환)
- [x] IC/ICIR/Sharpe 팩터 평가 (evaluator.py)
- [x] SymPy → Polars Expression 변환 (ast_converter.py — 13개 피처)
- [x] 벡터 경험 메모리 (memory.py — ko-sroberta 768-dim RAG)
- [x] 알파 팩토리 스케줄러 (scheduler.py — 자율 주기 실행)
- [x] DoWhy 인과 검증 4단계 (causal.py — 플라시보/랜덤 원인/체제 변화)
- [x] 교란 변수 로더 (confounders.py — 시장수익률/변동성/금리)
- [x] 유니버스 리졸버 (universe.py — KOSPI200/KOSDAQ150 등)
- [x] 팩터→백테스트 브릿지 (backtest_bridge.py)
- [x] 복합 팩터 포트폴리오 (portfolio.py — 상관행렬, 복합 구성)
- [x] REST API (routers/alpha.py — 마이닝/팩터/팩토리/인과검증/포트폴리오)

**백엔드 — ABM 시뮬레이션 + MCP**
- [x] LOB 엔진 (simulation/orderbook.py — heapq price-time priority)
- [x] 이질적 에이전트 6종 (simulation/agents.py)
- [x] 가상 거래소 + 시나리오 4종 (simulation/exchange.py)
- [x] FastMCP 서버 (mcp/server.py — 5 tools + 1 resource, SSE)
- [x] 거버넌스 (mcp/governance.py — 수량 제한, 감사 로그)
- [x] REST API (routers/simulation.py)

**프론트엔드 — 알파 랩 + 시뮬레이션**
- [x] AlphaLabPage (마이닝 설정 + 팩터 목록 + 팩토리 패널)
- [x] 인과 검증 결과 UI (CausalResult — Effect/p-value/DAG)
- [x] 팩터→백테스트 실행 버튼
- [x] SimulationPage (2탭: 시뮬레이션 설정 + MCP 대시보드)

### 미구현
- [ ] 사용자 인증 (로그인/세션)
- [ ] 24시간 무인 구동 (APScheduler 스케줄러)
- [ ] TradingContext 영속화 (현재 메모리, 서버 재시작 시 초기화)
- [ ] KIS WebSocket 실시간 체결 알림
- [ ] 알파 팩토리 진화 강화 (사이클 간 모집단 유지, 엘리티즘, 적합도 기반 선택)
