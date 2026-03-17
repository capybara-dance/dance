# KODEX 200 WCC vs KODEX 200 TR 투자 시뮬레이션 분석

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/capybara-dance/dance/blob/main/0317_coveredcall_yfinance.ipynb)

---

## 1. 개요

이 노트북의 핵심 목적은 다음 두 ETF에 **동일한 조건**으로 투자했을 때 어느 쪽이 더 유리한지를 과거 데이터 기반으로 비교하는 것입니다.

| ETF | 티커 | 전략 |
|-----|------|------|
| **KODEX 200 WCC** | `498400.KS` | 커버드콜 (Covered Call) — 주가 상승분 일부를 포기하는 대신 옵션 프리미엄 수익을 통해 정기적인 배당 지급 |
| **KODEX 200 TR** | `278530.KS` | 토탈 리턴 (Total Return) — 배당금을 자동 재투자하여 복리 성장 추구 |

### 핵심 질문

> "월 12만 원씩 인출하면서 1,000만 원을 투자했을 때, 커버드콜 ETF와 토탈 리턴 ETF 중 어느 쪽이 1년 후 더 많은 돈을 남기는가?"

---

## 2. 사용 라이브러리

```python
yfinance    # Yahoo Finance API — 주가·배당 이력 데이터 수집
pandas      # 시계열 데이터 처리
numpy       # 수치 계산
matplotlib  # 결과 시각화
```

---

## 3. 데이터 수집

### `get_stock_price(ticker, interval="1d")`

```python
def get_stock_price(ticker, interval="1d"):
    stock = yf.Ticker(ticker)
    data = stock.history(period="1y", interval=interval)
    ...
```

- `yfinance`를 통해 **최근 1년치** 일별 OHLCV + 배당 데이터를 수집합니다.
- 반환 컬럼: `Open`, `High`, `Low`, `Close`, `Volume`, `Dividends`, `Stock Splits`
- 데이터가 없거나 오류 발생 시 `None` 반환.

### 데이터 수집 대상

```python
kodex200wcc = get_stock_price('498400.ks')   # KODEX 200 WCC
kodex200tr  = get_stock_price('278530.ks')   # KODEX 200 TR
```

---

## 4. 정규화 비교 차트

```python
kodex200wcc_normalized = kodex200wcc['Close'] / kodex200wcc['Close'].iloc[0]
kodex200tr_normalized  = kodex200tr['Close']  / kodex200tr['Close'].iloc[0]
```

두 ETF의 시작 가격을 **1로 통일(정규화)**하여 상대적 가격 변동을 직관적으로 비교합니다.  
절대 가격 차이에 무관하게 수익률 궤적을 나란히 볼 수 있습니다.

---

## 5. 투자 시뮬레이션

### 시뮬레이션 파라미터

| 항목 | 값 |
|------|----|
| 초기 투자금 | 10,000,000 KRW (1,000만 원) |
| 월 인출금 | 120,000 KRW (12만 원) |
| 배당 처리 | 재투자 |
| 인출 시점 | 매월 마지막 거래일 |

### `simulate_single_etf_investment()` 함수 로직

```
[시작] 첫날 종가로 초기 주식 수 계산
  etf_shares = initial_investment / 첫날_종가

[매일 반복]
  ① 배당 처리 (재투자)
       배당금 발생 시 → 해당 금액으로 주식 추가 매수

  ② 월말 인출 처리 (매월 마지막 거래일)
       portfolio_cash -= monthly_withdrawal
       현금 부족 시 → 부족분만큼 주식 매도로 충당
       주식도 부족 시 → 경고 출력

  ③ 일별 포트폴리오 가치 기록
       일별_가치 = portfolio_cash + (etf_shares × 당일_종가)

[종료] 최종 평가액 + 총 인출액 반환
```

#### ① 배당 재투자

```python
if etf_dividend > 0:
    dividend_amount_etf = etf_shares * etf_dividend       # 배당 수령액
    shares_bought_etf   = dividend_amount_etf / current_etf_price  # 재투자로 추가 매수
    etf_shares += shares_bought_etf
```

배당을 현금으로 쌓지 않고 **즉시 같은 ETF를 추가 매수**하여 복리 효과를 반영합니다.

#### ② 월말 인출 & 주식 매도

```python
# 월의 마지막 거래일 판별
if i == len(dates) - 1:                      # 마지막 날
    is_last_trading_day = True
elif date.month != dates[i+1].month:         # 다음 날이 다음 달이면
    is_last_trading_day = True

if is_last_trading_day:
    portfolio_cash -= monthly_withdrawal      # 현금 인출
    if portfolio_cash < 0:                    # 현금 부족 시 주식 매도
        shares_to_sell = deficit / current_etf_price
        etf_shares -= shares_to_sell
        portfolio_cash += 매도금액
```

- 현금 계좌가 0에서 시작하므로, 첫 번째 인출 시점부터 주식을 매도하게 됩니다.
- `withdrawn_months` 집합으로 **중복 인출을 방지**합니다.

#### ③ 일별 포트폴리오 가치

```python
total_portfolio_value = portfolio_cash + etf_shares * current_etf_price
```

현금 잔고와 ETF 평가액을 합산해 매일 기록합니다.

---

## 6. 실행 결과

> 데이터 기간: 2025년 3월 17일 ~ 2026년 3월 17일 (약 1년)

```
--- KODEX 200 WCC Simulation Results ---
Final Appraisal Amount:                  22,953,952.51 KRW
Total Withdrawn Cash:                     1,560,000.00 KRW
Sum (Final Appraisal + Total Withdrawn): 24,513,952.51 KRW

--- KODEX 200 TR Simulation Results ---
Final Appraisal Amount:                  22,014,263.96 KRW
Total Withdrawn Cash:                     1,560,000.00 KRW
Sum (Final Appraisal + Total Withdrawn): 23,574,263.96 KRW
```

| 항목 | KODEX 200 WCC | KODEX 200 TR | 차이 |
|------|-------------:|-------------:|-----:|
| 최종 평가액 | 22,953,952 원 | 22,014,263 원 | **+939,688 원** |
| 총 인출 현금 | 1,560,000 원 | 1,560,000 원 | — |
| **합계** | **24,513,952 원** | **23,574,263 원** | **+939,688 원** |
| 수익률 (합계 기준) | **+24.5%** | **+17.9%** | **+6.6%p** |

> ⚠️ 경고 메시지 (`Warning: Still negative cash...`)는 수치 부동소수점 오차(`-0.00 KRW`)로 실질적 영향 없음.

---

## 7. 결론 및 해석

- 해당 1년 구간에서는 **KODEX 200 WCC(커버드콜)가 KODEX 200 TR(토탈 리턴)보다 약 93.9만 원 더 높은 총 수익**을 기록했습니다.
- 이는 해당 기간 동안 커버드콜의 **옵션 프리미엄 수익**이 토탈 리턴의 **완전한 배당 재투자 복리 효과**를 앞섰음을 의미합니다.
- 단, 이 결과는 **1년이라는 특정 구간**에 한정되며, 시장 상승기가 길어질수록 커버드콜은 상승분을 일부 포기하는 구조이므로 장기 성과는 달라질 수 있습니다.

---

## 8. 코드 구조 한눈에 보기

```
0317_coveredcall_yfinance.ipynb
│
├── [셀 2]  환경 설정: yfinance 설치
├── [셀 3]  라이브러리 임포트
├── [셀 4]  get_stock_price() 함수 정의 및 테스트 (441680.KS)
├── [셀 5]  두 ETF 데이터 수집 (498400.KS, 278530.KS)
├── [셀 6]  정규화 종가 비교 차트 생성
├── [셀 7]  시뮬레이션 로직 설명 (마크다운)
└── [셀 8]  simulate_single_etf_investment() 함수 정의 및 실행
            ├── 파라미터: 초기 투자 1,000만 원 / 월 인출 12만 원
            ├── WCC 시뮬레이션 → 최종 합계 2,451만 원
            ├── TR  시뮬레이션 → 최종 합계 2,357만 원
            └── 일별 포트폴리오 가치 비교 차트
```
