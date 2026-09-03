# OpenBB 완전 정복 가이드 (한국어)

> 이 문서는 OpenBB 저장소를 직접 분석하며 정리한 학습 노트입니다.
> 프로젝트 소개부터 설치·사용법, 수익화 아이디어, PHP 연동 방법까지 다룹니다.

## 🔗 저장소 주소

| 구분 | 주소 |
|---|---|
| 원본 저장소 | <https://github.com/OpenBB-finance/OpenBB> |
| 내 포크 | <https://github.com/bmshin94/OpenBB> |
| 공식 문서 | <https://docs.openbb.co> |
| Python 레퍼런스 | <https://docs.openbb.co/python/reference> |
| PyPI 패키지 | <https://pypi.org/project/openbb/> |
| OpenBB Workspace | <https://pro.openbb.co> |
| 관련 저장소 (백엔드) | <https://github.com/OpenBB-finance/backends-for-openbb> |
| 관련 저장소 (AI 에이전트) | <https://github.com/OpenBB-finance/agents-for-openbb> |

---

## 목차

1. [OpenBB란 무엇인가](#1-openbb란-무엇인가)
2. [폴더 구조 분석](#2-폴더-구조-분석)
3. [설치 가이드](#3-설치-가이드)
4. [API 키 등록](#4-api-키-등록)
5. [기본 사용법](#5-기본-사용법)
6. [실전 예제](#6-실전-예제)
7. [4가지 접근 창구](#7-4가지-접근-창구)
8. [트러블슈팅](#8-트러블슈팅)
9. [수익화 전 필수 확인 사항](#9-수익화-전-필수-확인-사항)
10. [상업이용 안전한 공공 데이터](#10-상업이용-안전한-공공-데이터)
11. [수익화 아이디어](#11-수익화-아이디어)
12. [PHP 연동 방법](#12-php-연동-방법)

---

## 1. OpenBB란 무엇인가

**Open Data Platform (ODP) by OpenBB** — 오픈소스 금융 데이터 플랫폼.
흔히 **"블룸버그 터미널의 오픈소스 버전"** 으로 불립니다.

### 핵심 컨셉: "Connect once, consume everywhere"

여러 금융 데이터 소스를 **하나의 표준화된 인터페이스**로 묶어주는 통합 레이어입니다.

**문제 상황 — OpenBB가 없다면:**

```python
# 소스마다 파라미터도 컬럼명도 전부 다름
yf.download("AAPL")            # 컬럼: Open, High, Low, Close
requests.get("fmp.com/...")    # 키: o, h, l, c
alpha_vantage_api(...)         # "1. open", "2. high"
```

같은 "시가"인데 이름이 셋 다 달라서, 데이터 소스를 바꿀 때마다 코드를 새로 짜야 합니다.

**OpenBB를 쓰면:**

```python
from openbb import obb

obb.equity.price.historical("AAPL", provider="yfinance")
obb.equity.price.historical("AAPL", provider="fmp")       # 코드 동일!
obb.equity.price.historical("AAPL", provider="intrinio")  # 이것도 동일!
```

`provider`만 교체하면 끝. 결과 스키마(컬럼명, 타입)도 Pydantic으로 전부 통일되어 나옵니다.
이 **표준화(Standardization)** 가 OpenBB의 존재 이유이자 가장 큰 가치입니다.

### 규모

- 데이터 제공처 **32곳**
- 자산군별 확장 모듈 **18종**
- 사용 가능한 명령어(엔드포인트) **251개**

---

## 2. 폴더 구조 분석

전체 용량 약 361MB.

```
OpenBB/
├── openbb_platform/   # 185MB — 핵심 본체
│   ├── core/          #   엔진 (라우터, 표준화 로직, 인증)
│   ├── providers/     #   32개 데이터 소스 커넥터
│   ├── extensions/    #   18개 자산군별 기능 모듈
│   └── obbject_extensions/  # 결과물 후처리 (차트 등)
├── cli/               # 터미널용 CLI (openbb-cli)
├── desktop/           # Tauri + React 데스크탑 앱 (Rust/TS 약 50:50, 설치 35MB)
├── examples/          # 주피터 노트북 예제 20개+
├── cookiecutter/      # 커스텀 데이터 소스 추가용 템플릿
├── build/             # 빌드/배포 스크립트
└── CLAUDE.md          # (개인 추가) Claude 페르소나 설정
```

### 2-1. `providers/` — 데이터 공급처 32곳

| API 키 불필요 (무료) | API 키 필요 |
|---|---|
| `yfinance` (야후 파이낸스) | `fmp` (재무제표) |
| `sec` (미국 공시 원본) | `fred` (경제지표, 키 무료 발급) |
| `federal_reserve`, `ecb` (연준/ECB) | `intrinio`, `benzinga`, `tiingo` |
| `imf`, `oecd`, `econdb` (국제기구) | `alpha_vantage`, `nasdaq` |
| `cboe` (옵션/변동성), `finra`, `finviz` | `bls` (미 고용통계) |
| `tmx` (캐나다), `wsj`, `deribit` | `tradingeconomics`, `congress_gov` |
| `government_us`, `multpl`, `famafrench` | `cftc` (app_token) |
| `stockgrid`, `seeking_alpha`, `eia`, `tradier` | |

> 자격증명 이름 규칙은 `{provider}_{credential}` 형태입니다.
> (출처: `openbb_platform/core/openbb_core/provider/abstract/provider.py:51`)
> 예) `fmp_api_key`, `fred_api_key`, `tiingo_token`, `cftc_app_token`

### 2-2. `extensions/` — 자산군별 기능 18종

`equity`(주식) · `crypto`(암호화폐) · `etf` · `economy`(경제지표) · `derivatives`(파생)
· `fixedincome`(채권) · `currency`(환율) · `news` · `technical`(기술적분석)
· `quantitative`(퀀트) · `econometrics`(계량경제) · `index` · `commodity`(원자재)
· `regulators`(규제/공시) · `famafrench` · `mcp_server` · `platform_api` · `devtools`

### 2-3. `extensions/mcp_server/` — AI 에이전트 연동

LLM 에이전트가 MCP 프로토콜로 OpenBB 데이터를 직접 조회할 수 있게 해주는 서버입니다.

설계 포인트:

- 251개 툴을 한 번에 노출하면 토큰이 폭발하므로, **카테고리 탐색 → 필요한 툴만 동적 활성화** 구조
- 툴 가시성 변경은 **세션별로 독립** — 여러 에이전트가 동시에 붙어도 서로 간섭 없음

### 2-4. `examples/` — 실전 노트북

- `loadHistoricalPriceData.ipynb` — 기초 (여기부터 시작 추천)
- `BacktestingMomentumTrading.ipynb` — 모멘텀 전략 백테스팅
- `portfolioOptimizationUsingModernPortfolioTheory.ipynb` — 포트폴리오 최적화
- `sectorRotationStrategy.ipynb` — 섹터 로테이션 전략
- `impliedEarningsMove.ipynb` — 옵션으로 실적발표 변동폭 예측
- `openbbPlatformAsLLMTools.ipynb` — **LLM 툴로 활용하는 법**
- `EthereumTrendAnalysis.ipynb` — 이더리움 트렌드 분석
- `googleColab.ipynb` — 설치 없이 콜랩에서 바로 실행

---

## 3. 설치 가이드

### 3-1. 사전 준비

**파이썬 버전 확인 (가장 중요)**

```bash
python --version
```

- ✅ **3.10 ~ 3.13** → 통과
- ❌ **3.9 이하** → 설치 불가

> ⚠️ README에는 "3.9.21 - 3.12"라고 적혀 있으나 이는 **오래된 정보**입니다.
> 실제 패키지 메타데이터 `openbb_platform/pyproject.toml:14`에는
> `python = ">=3.10,<4"` 로 명시되어 있습니다.

**가상환경 생성 (강력 권장)**

의존 패키지가 많아 다른 프로젝트와 충돌하기 쉬우므로 반드시 격리하세요.

```bash
mkdir ~/finance-lab && cd ~/finance-lab
python -m venv .venv

source .venv/bin/activate        # Mac / Linux
.venv\Scripts\Activate.ps1       # Windows PowerShell
```

### 3-2. 설치 3가지 방식

**A. 기본 설치 (입문 추천)**

```bash
pip install openbb
```

기본 포함 항목:

- 확장: `equity`, `crypto`, `currency`, `etf`, `derivatives`, `fixedincome`,
  `economy`, `news`, `commodity`, `index`, `regulators`
- 제공처 17곳: `yfinance`, `sec`, `fred`, `fmp`, `imf`, `oecd`, `federal_reserve`,
  `econdb`, `intrinio`, `tiingo`, `benzinga`, `bls`, `cftc`, `congress_gov`,
  `government_us`, `tradingeconomics`, `eia`

**B. 필요한 것만 선택 (효율적)**

```bash
pip install "openbb[charting,technical,mcp_server]"
```

선택 가능한 extras (출처: `openbb_platform/pyproject.toml` `[tool.poetry.extras]`):

| 구분 | 이름 | 설명 |
|---|---|---|
| 기능 | `charting` | 차트 렌더링 |
| 기능 | `technical` | 이동평균 · RSI · MACD 등 보조지표 |
| 기능 | `quantitative` | 퀀트 통계 분석 |
| 기능 | `econometrics` | 계량경제 회귀분석 |
| 기능 | `mcp_server` | **AI 에이전트 연동** |
| 소스 | `cboe` `ecb` `finra` `finviz` `nasdaq` `tmx` `wsj` `deribit` `tradier` `stockgrid` `seeking_alpha` `multpl` `famafrench` `alpha_vantage` `biztoc` | 추가 데이터 제공처 |

**C. 전체 설치**

```bash
pip install "openbb[all]"
```

### 3-3. 설치 확인

API 키·회원가입 없이 바로 동작하는지 확인:

```python
from openbb import obb

output = obb.equity.price.historical("AAPL", provider="yfinance")
df = output.to_dataframe()
print(df.tail())
```

> ⏳ **첫 실행은 느립니다.** 설치된 확장을 스캔해 `obb.` 명령어 맵을 빌드하기 때문입니다.
> 두 번째 실행부터는 빨라집니다.

### 3-4. 소스코드로 개발 설치 (개발자용)

저장소를 클론해 코드를 직접 수정하며 개발할 경우:

```bash
cd openbb_platform
python dev_install.py
```

PyPI 대신 로컬 폴더의 코드를 바라보도록(`develop = true`) 설치되어, 코드 수정이 재설치 없이 즉시 반영됩니다.

---

## 4. API 키 등록

무료 소스는 키가 필요 없지만, FMP(재무제표)·FRED(경제지표) 등은 키가 필요합니다. (둘 다 무료 플랜 존재)

### 주요 키 이름

| 제공사 | 키 이름 | 발급처 |
|---|---|---|
| FMP | `fmp_api_key` | financialmodelingprep.com |
| FRED | `fred_api_key` | fred.stlouisfed.org (완전 무료) |
| Tiingo | `tiingo_token` | tiingo.com |
| Intrinio | `intrinio_api_key` | intrinio.com |
| Alpha Vantage | `alpha_vantage_api_key` | alphavantage.co |
| Benzinga | `benzinga_api_key` | benzinga.com |
| Nasdaq | `nasdaq_api_key` | data.nasdaq.com |

### 방법 1 — 설정 파일 (권장)

`~/.openbb_platform/user_settings.json`
(경로 출처: `openbb_platform/core/openbb_core/app/constants.py:6-7`)

```bash
mkdir -p ~/.openbb_platform
```

```json
{
    "credentials": {
        "fmp_api_key": "여기에_키",
        "fred_api_key": "여기에_키",
        "tiingo_token": "여기에_키"
    }
}
```

### 방법 2 — 코드에서 등록

```python
from openbb import obb

obb.user.credentials.fmp_api_key = "여기에_키"
obb.account.save()   # 파일로 영구 저장
```

### 방법 3 — 환경변수 (`.env`)

`~/.openbb_platform/.env` 에 **대문자로** 작성:

```env
FMP_API_KEY=여기에_키
FRED_API_KEY=여기에_키
```

> 🚨 **API 키를 깃에 커밋하지 마세요.**
> 방법 1·3은 홈 디렉토리에 저장되어 프로젝트 폴더 밖이라 안전합니다.
> 프로젝트 내부에 `.env`를 둔다면 반드시 `.gitignore`에 추가하세요.

---

## 5. 기본 사용법

### 5-1. 명령어 구조

```
obb . 자산군 . 카테고리 . 기능()
       ↓         ↓          ↓
     equity    price   historical
```

```python
obb.equity.price.historical("AAPL")       # 주가 히스토리
obb.equity.fundamental.income("AAPL")     # 손익계산서
obb.crypto.price.historical("BTC-USD")    # 비트코인
obb.economy.cpi(country="united_states")  # 소비자물가지수
obb.news.company("TSLA")                  # 테슬라 뉴스
obb.currency.price.historical("USDKRW")   # 원달러 환율
```

> 💡 주피터/IDE에서 `obb.` 까지 입력 후 **Tab** 을 누르면 251개 명령어가 자동완성됩니다.

### 5-2. 결과물(OBBject) 변환 메서드

(출처: `openbb_platform/core/openbb_core/app/model/obbject.py`)

```python
output = obb.equity.price.historical("AAPL")

output.to_dataframe()   # pandas DataFrame (가장 많이 사용)
output.to_df()          # 위와 동일 (축약형)
output.to_dict()        # 파이썬 딕셔너리
output.to_polars()      # Polars DataFrame (고속 처리)
output.to_numpy()       # numpy 배열
output.to_llm()         # LLM이 소비하기 좋은 형태
output.show()           # 차트 표시 (charting 확장 필요)

output.results          # 원본 데이터
output.provider         # 사용된 제공처
output.warnings         # 경고 메시지
```

### 5-3. 자주 쓰는 파라미터

```python
obb.equity.price.historical(
    symbol="AAPL",
    start_date="2024-01-01",
    end_date="2026-09-01",
    interval="1d",          # 1m, 5m, 1h, 1d, 1W, 1M
    provider="yfinance",
)
```

### 5-4. 기본 provider 고정

```python
obb.user.defaults.routes["/equity/price/historical"] = {"provider": "fmp"}
obb.account.save()
```

---

## 6. 실전 예제

### 여러 종목 수익률 비교

```python
from openbb import obb
import pandas as pd

symbols = ["AAPL", "MSFT", "NVDA", "TSLA"]
frames = {}

for s in symbols:
    df = obb.equity.price.historical(
        s, start_date="2025-01-01", provider="yfinance"
    ).to_dataframe()
    frames[s] = df["close"]

prices = pd.DataFrame(frames)
returns = (prices / prices.iloc[0] - 1) * 100   # 정규화 수익률(%)
print(returns.tail())
```

### 재무제표 조회 (FMP 키 필요)

```python
income = obb.equity.fundamental.income(
    "AAPL", period="annual", limit=5, provider="fmp"
).to_dataframe()

metrics = obb.equity.fundamental.metrics("AAPL", provider="fmp").to_dataframe()

print(income[["revenue", "net_income", "eps"]])
```

### 경제지표 (FRED 키 무료 발급)

```python
cpi = obb.economy.cpi(country="united_states", provider="fred").to_dataframe()

curve = obb.fixedincome.government.yield_curve(
    provider="federal_reserve"
).to_dataframe()
```

### 뉴스 수집 (키 불필요)

```python
news = obb.news.company("TSLA", limit=20, provider="yfinance").to_dataframe()
print(news[["date", "title"]])
```

### 미국 공시 조회 (키 불필요)

```python
filings = obb.equity.fundamental.filings(
    "AAPL", form_type="10-K", provider="sec"
).to_dataframe()
```

### 기술적 지표 (technical 확장 필요)

```python
data = obb.equity.price.historical("AAPL", provider="yfinance")

rsi = obb.technical.rsi(data=data.results, length=14)
sma = obb.technical.sma(data=data.results, length=20)
```

---

## 7. 4가지 접근 창구

같은 데이터를 4가지 방식으로 소비할 수 있습니다.

| 방식 | 실행 | 용도 |
|---|---|---|
| Python 라이브러리 | `pip install openbb` | 주피터/판다스 분석, 백테스팅 |
| CLI | `pip install openbb-cli` → `openbb` | 터미널에서 빠른 조회 |
| REST API | `openbb-api` → `127.0.0.1:6900` | 웹앱/대시보드 연동 |
| MCP 서버 | `openbb-mcp` → `127.0.0.1:8000` | AI 에이전트 연동 |

### 7-1. CLI

```bash
pip install openbb-cli
openbb
```

```
2026 Sep 03, 10:30 (🦋) / $ equity
2026 Sep 03, 10:30 (🦋) /equity/ $ price
2026 Sep 03, 10:30 (🦋) /equity/price/ $ historical --symbol AAPL
```

- `?` 또는 `help` — 도움말
- `..` — 뒤로 가기
- `exit` — 종료

### 7-2. REST API

```bash
openbb-api      # 기본 포트 6900
```

- Swagger 문서 자동 생성: <http://127.0.0.1:6900/docs>
- curl 예시:

```bash
curl "http://127.0.0.1:6900/api/v1/equity/price/historical?symbol=AAPL&provider=yfinance"
```

**OpenBB Workspace 연동:** <https://pro.openbb.co> 접속 →
Apps 탭 → Connect backend → URL에 `http://127.0.0.1:6900` 입력 → Test → Add

### 7-3. MCP 서버 (AI 연동)

```bash
pip install "openbb[mcp_server]"
openbb-mcp
```

기본값: `127.0.0.1:8000`, transport `streamable-http`

**Claude Desktop 설정:**

```json
{
  "mcpServers": {
    "openbb": {
      "command": "uvx",
      "args": ["--from", "openbb-mcp-server", "--with", "openbb", "openbb-mcp",
               "--transport", "stdio"]
    }
  }
}
```

**Claude Code 설정:**

```bash
claude mcp add openbb -- uvx --from openbb-mcp-server --with openbb openbb-mcp --transport stdio
```

**유용한 옵션** (출처: `openbb_platform/extensions/mcp_server/README.md`):

```bash
openbb-mcp --default-categories equity,economy   # 특정 카테고리만 활성화
openbb-mcp --tool-discovery                      # 툴 탐색 모드
openbb-mcp --port 9000                           # 포트 변경
```

설정 파일은 `~/.openbb_platform/mcp_settings.json` 에 저장되며, 없으면 첫 실행 시 기본값으로 자동 생성됩니다.

---

## 8. 트러블슈팅

| 증상 | 원인 및 해결 |
|---|---|
| `ERROR: Could not find a version` | 파이썬 3.9 이하 → **3.10 이상으로 업그레이드** |
| `obb.` 자동완성 안 뜸 | 빌드 미완료 → `python -c "import openbb; openbb.build()"` |
| `EmptyDataError` / 빈 결과 | 심볼 오타 또는 해당 provider 미지원 → **다른 provider로 시도** |
| `Missing credential` | API 키 미등록 → [4장](#4-api-키-등록) 참고 |
| 의존성 충돌 | 가상환경 미사용 → **새 venv에서 재설치** |
| 속도 저하 | `[all]` 대신 필요한 extras만 선택 설치 |

---

## 9. 수익화 전 필수 확인 사항

> ⚠️ 아래 내용은 라이선스 원문과 일반적인 해석 기준을 정리한 것으로, **법률 자문이 아닙니다.**
> 실제 상업화 단계에서는 반드시 IP 전문 변호사와 상담하세요.

### 9-1. AGPLv3 라이선스

`LICENSE` 파일과 `pyproject.toml`에 `AGPL-3.0-only` 로 명시되어 있습니다.

**LICENSE 파일 544행, 제13조 (Remote Network Interaction):**

> 프로그램을 수정했다면, 수정 버전은 네트워크를 통해 원격으로 상호작용하는
> 모든 사용자에게 해당 소스코드를 받을 기회를 눈에 띄게 제공해야 한다.

**즉, OpenBB 코드를 포함한 웹서비스로 수익을 낼 경우 내 소스코드도 공개해야 할 수 있습니다.**
MIT처럼 자유롭게 가져다 쓰는 라이선스가 아닙니다. OpenBB가 의도적으로 선택한 듀얼 라이선스 전략입니다.

### 9-2. 데이터 제공처 이용약관 (더 중요할 수 있음)

라이선스를 통과해도 **데이터 자체를 재배포할 권리는 별개**입니다.

| 소스 | 상업적 이용 | 비고 |
|---|---|---|
| ❌ `yfinance` | 위험 | 야후 비공식 스크래핑, 재배포 금지 |
| ⚠️ `fmp` `intrinio` `tiingo` | 유료 플랜 필요 | 무료 키는 개인용 한정 |
| ⚠️ `finviz` `wsj` `seeking_alpha` | 위험 | 스크래핑 기반 |
| ✅ **미국 정부·공공 데이터** | **자유** | 미국 공공저작물은 저작권 없음 |

**결론: 수익화를 노린다면 공공 데이터 기반으로 설계해야 합니다.**

---

## 10. 상업이용 안전한 공공 데이터

저장소에서 직접 확인한, 상업적 활용이 자유로운 데이터 목록입니다.

### SEC (미국 증권거래위원회) — `providers/sec/openbb_sec/models/`

| 파일 | 내용 |
|---|---|
| `insider_trading.py` | 임원 내부자 매매 (Form 4) |
| `form_13FHR.py` | 기관 보유현황 13F (버핏 포트폴리오 등) |
| `equity_ftd.py` | 결제불이행(FTD) — 공매도 압력 신호 |
| `management_discussion_analysis.py` | MD&A 원문 (AI 요약 소재) |
| `rss_litigation.py` | SEC 소송·제재 실시간 피드 |
| `company_filings.py` | 전체 공시 목록 |
| `institutions_search.py` | 기관 검색 |
| `nport_disclosure.py` | 펀드 보유종목 |
| `latest_financial_reports.py` | 최신 재무보고서 |
| `insider_trading.py` 외 재무제표 | `income_statement`, `balance_sheet`, `cash_flow` (+ growth) |

### FINRA / CFTC — 수급 데이터

| 파일 | 내용 |
|---|---|
| `finra/equity_short_interest.py` | 공매도 잔고 |
| `finra/otc_aggregate.py` | 다크풀 거래량 |
| `cftc/cot.py` | COT 리포트 (기관 선물 포지션) |

### 연방준비제도 — `providers/federal_reserve/`

`fomc_documents.py` (FOMC 회의록) · `primary_dealer_positioning.py` ·
`central_bank_holdings.py` · `yield_curve.py` · `svensson_yield_curve.py` ·
`treasury_rates.py` · `money_measures.py` · `inflation_expectations.py` ·
`total_factor_productivity.py` · `primary_dealer_fails.py`

### FRED — 경제지표 36종

`consumer_price_index.py` · `non_farm_payrolls.py` · `federal_funds_rate.py` ·
`yield_curve.py` · `personal_consumption_expenditures.py` · `sofr.py` ·
`economic_calendar.py` · `senior_loan_officer_survey.py` · `fed_projections.py` ·
`university_of_michigan.py` · `mortgage_indices.py` · `bond_indices.py` 외

### 의회 / 재무부

| 파일 | 내용 |
|---|---|
| `congress_gov/congress_bills.py` | 법안 (규제 리스크 조기 감지) |
| `congress_gov/bill_text.py` | 법안 전문 |
| `government_us/treasury_auctions.py` | 국채 입찰 결과 |
| `government_us/treasury_prices.py` | 국채 가격 |

> 💡 **핵심 인사이트:** 이 데이터들은 모두 공개되어 있지만 **접근성이 매우 낮습니다.**
> SEC·FRED 원본 사이트는 UI가 열악하고 데이터 포맷도 난해합니다.
> **"공짜지만 읽기 힘든 데이터"를 "보기 쉽게" 가공하는 것** — 이것이 수익화의 정석입니다.

---

## 11. 수익화 아이디어

### Lv.1 — 콘텐츠형 (라이선스 리스크 없음, 입문 추천)

OpenBB를 **도구로만** 사용하고 **결과물(콘텐츠)** 을 판매합니다.
코드를 배포하지 않으므로 AGPL 조항이 적용되지 않습니다.

| 아이디어 | 내용 | 수익 모델 | 난이도 |
|---|---|---|---|
| **유료 뉴스레터** | "이번 주 내부자 매수 TOP 10", "기관 13F 변화 리포트", "FOMC 회의록 AI 요약" | 스티비/메일리 구독 (월 5,000~15,000원) | ⭐ |
| **유튜브 / 블로그** | 같은 데이터로 영상·글 제작. 자동 수집 → 반자동 생산 | 애드센스, 협찬 | ⭐ |
| **강의 / 전자책** | "파이썬으로 만드는 금융 데이터 자동화" | 인프런, 클래스101, 탈잉 | ⭐⭐ |

> 뉴스레터는 데이터 수집이 자동화되므로 주 2시간 정도로 운영 가능합니다.

### Lv.2 — 서비스형 (라이선스 설계 필요)

| 아이디어 | 내용 | 수익 모델 | 난이도 |
|---|---|---|---|
| **알림 봇** | 텔레그램/디스코드로 내부자 매매·공매도 급증·13F 변화 알림 | 무료(지연) / 유료 월 9,900원(실시간) | ⭐⭐ |
| **니치 대시보드 SaaS** | "13F 추적기", "FOMC 워치", "국채 대시보드" 등 딱 하나만 잘하는 서비스 | 월 $5~20 구독 | ⭐⭐⭐ |
| **B2B 내부 도구** | 자산운용사·회계법인·IR팀 대상 데이터 자동화 시스템 구축 | 프로젝트당 500만~3,000만원 + 유지보수 | ⭐⭐⭐ |
| **MCP + AI 애널리스트** | `mcp_server` 활용한 AI 금융 비서 | 구독 / 투자유치 | ⭐⭐⭐ |

> 💡 **B2B 내부 도구가 숨은 기회입니다.** AGPL 제13조는 "원격 사용자"에게 소스를
> 제공하라는 조항인데, 사내 시스템은 사용자가 자사 직원이므로 요건 충족이 간단합니다.
> 난이도 대비 단가가 가장 높습니다.

> 💡 알림 봇은 데이터 원본을 재배포하는 것이 아니라 **이벤트만 통지**하므로
> 데이터 약관 측면에서도 상대적으로 안전합니다.

### Lv.3 — 정공법

- **상업 라이선스 문의** — `hello@openbb.co` 로 듀얼 라이선스 협의
- **내 서비스도 오픈소스 공개** — AGPL로 공개하고 호스팅·지원·커스터마이징으로 수익화
  (GitLab, Sentry 모델)

### 추천 조합

```
아이디어: 알림 봇 또는 13F 추적 대시보드
아키텍처: Python 수집 → DB → PHP(Laravel) 서비스
데이터:   SEC / FINRA / Fed 공공데이터만 사용
```

**이유:** 라이선스·데이터 약관 양쪽 모두 안전 / 기존 PHP 역량 활용 /
MVP를 주말 이틀 규모로 구현 가능 / 검증 후 구독 결제 부착 용이

---

## 12. PHP 연동 방법

### 전제

PHP에서 `openbb` 라이브러리를 **직접 사용할 수는 없습니다.** OpenBB는 100% 파이썬이며 PHP 포팅판도 없습니다.
대신 아래 3가지 방법으로 연동할 수 있습니다.

### 방법 1 — 배치 수집형 (권장)

```
[Python 스크립트]  →  [MySQL/PostgreSQL]  →  [PHP/Laravel 웹앱]
   하루 1회 크론          데이터 저장           사용자에게 서비스
        ↑                                            ↑
   OpenBB는 여기만                        OpenBB 코드 전혀 없음
```

**장점**

| 항목 | 설명 |
|---|---|
| 라이선스 안전 | PHP 앱이 OpenBB 코드를 전혀 포함하지 않음. DB에서 값만 읽으므로 파생저작물이 아님 |
| 성능 | 사용자 요청마다 외부 API를 호출하지 않고 자체 DB에서 즉시 응답 |
| 비용 | 하루 1회 호출이므로 무료 한도 내에서 해결 |
| 기술 스택 | 기존 PHP/Laravel 역량 그대로 활용 |

**Python 수집기 예시**

```python
# collector.py
from openbb import obb
import pymysql

conn = pymysql.connect(
    host="localhost", user="root", password="...", database="finance"
)

# SEC 내부자 매매 수집 (공공데이터라 상업이용 가능)
data = obb.equity.ownership.insider_trading("AAPL", provider="sec").to_dataframe()

with conn.cursor() as cur:
    for _, r in data.iterrows():
        cur.execute(
            "INSERT IGNORE INTO insider_trades "
            "(symbol, filed_date, owner, transaction_type, shares, price) "
            "VALUES (%s,%s,%s,%s,%s,%s)",
            (
                "AAPL",
                r["filing_date"],
                r["owner_name"],
                r["transaction_type"],
                r["securities_transacted"],
                r["price"],
            ),
        )
conn.commit()
```

**크론 등록**

```bash
# 매일 새벽 3시 실행
0 3 * * * /home/user/finance-lab/.venv/bin/python /path/collector.py
```

**PHP(Laravel) 측**

```php
// app/Http/Controllers/InsiderController.php
public function index()
{
    $trades = DB::table('insider_trades')
        ->where('filed_date', '>=', now()->subDays(7))
        ->orderByDesc('shares')
        ->limit(50)
        ->get();

    return view('insider.index', compact('trades'));
}
```

PHP 측은 OpenBB의 존재를 알 필요조차 없습니다.

### 방법 2 — REST API 호출형 (실시간 필요 시)

```bash
openbb-api    # 기본 포트 6900
```

```php
use Illuminate\Support\Facades\Http;

$res = Http::get('http://127.0.0.1:6900/api/v1/equity/ownership/insider_trading', [
    'symbol'   => 'AAPL',
    'provider' => 'sec',
]);

$results = $res->json()['results'];
```

- ✅ 실시간 조회 가능, 코드 분리가 깔끔
- ⚠️ **법적으로는 회색지대입니다.** "프로세스가 분리되었으므로 별개 저작물"이라는 해석이
  일반적이나 확립된 판례는 없습니다. 상업화 시 반드시 법률 검토가 필요합니다.

### 방법 3 — PHP에서 파이썬 직접 실행 (간단한 작업용)

```php
$json = shell_exec('/path/.venv/bin/python fetch.py AAPL 2>&1');
$data = json_decode($json, true);
```

- ✅ 구현이 가장 단순
- ❌ 느리고, **사용자 입력을 검증 없이 넘기면 명령어 주입(Command Injection) 취약점**이 발생합니다.
  반드시 `escapeshellarg()` 등으로 이스케이프하거나 허용 목록 방식으로 검증하세요.

---

## 부록: 빠른 시작 3줄

```bash
python -m venv .venv && source .venv/bin/activate
pip install "openbb[charting,technical,mcp_server]"
python -c "from openbb import obb; print(obb.equity.price.historical('AAPL').to_dataframe().tail())"
```

이후 FRED 키만 무료로 발급받아 `~/.openbb_platform/user_settings.json` 에 등록하면
경제지표 데이터까지 모두 사용할 수 있습니다.

---

## 면책 조항

- 본 문서의 라이선스·약관 관련 내용은 **법률 자문이 아닙니다.** 상업적 활용 시 전문가 상담이 필요합니다.
- OpenBB 저장소의 원문 고지에 따르면, 플랫폼에서 제공되는 데이터가 **항상 정확하다고 보장되지 않습니다.**
- 금융투자에는 원금 손실 위험이 있습니다. 본 문서와 OpenBB는 **학습·분석 목적**이며,
  실제 투자 판단의 근거로 사용해서는 안 됩니다.
