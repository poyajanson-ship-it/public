# usd-nowcast-stack

سیستم Nowcasting تولید-محور (سطح بانک مرکزی، hybrid-frequency ensemble) برای پیش‌بینیِ پیش از انتشارِ سه داده «قرمز» USD آمریکا.

**هدف جاری (انتشار ۲۰۲۶-۱۱-۲۵، ساعت 8:30 ET / 17:00 تهران):**

| # | شاخص | Consensus بازار | روش اصلی ما |
|---|------|-----------------|-------------|
| ۱ | Core PCE Price Index m/m (اکتبر) | 0.3% | Cleveland Fed Nowcast + pipeline خودمان |
| ۲ | Prelim GDP q/q (برآورد دوم Q3) | 3.5% | GDPNow + Bridge + Ensemble |
| ۳ | GDP Price Index q/q (برآورد دوم Q3) | — | ALFRED revision stats + Bridge |

## معماری

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 0 — REAL-TIME DATA PIPELINE (vintage-aware, ragged edge) │
│  FRED ALFRED · BLS v2 · BEA API · Cleveland Fed JSON ·          │
│  Atlanta Fed GDPNow · Census · EIA v2 · Zillow ZORI             │
└───────────────────────────────┬─────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 — INDICATOR POOL (100–200 series, transformed)         │
│  log-diff · seasonal flags · ragged-edge alignment              │
└───────────────────────────────┬─────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2 — CORE MODELS                                          │
│  ① DFM + Kalman (statsmodels DynamicFactorMQ, mixed frequency)  │
│  ② Bridge Equations (Atlanta Fed GDPNow method)                 │
│  ③ MIDAS / U-MIDAS / SC-MIDAS / midasml                         │
│  ④ Bayesian MF-VAR (Minnesota prior)                            │
│  ⑤ GBM (LightGBM/XGBoost/CatBoost) روی فاکتورهای استخراج‌شده     │
│  ⑥ N-HiTS / TFT (اختیاری، torch)                                │
└───────────────────────────────┬─────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3 — ENSEMBLE & UNCERTAINTY                               │
│  dynamic inverse-RMSE weights · news decomposition              │
│  split-conformal intervals (crepes-compatible)                  │
└─────────────────────────────────────────────────────────────────┘
```

## ساختار ریپو

```
config/
  series_map.yaml          # نقشه سری‌ها (FRED/BLS/BEA) + ترنسفورم‌ها
nowcast/
  data/                    # fetch_fred, fetch_bls, fetch_bea, fetch_cleveland_fed, fetch_gdpnow
  features/                # transform.py — ترنسفورم و alignment با ragged edge
  models/                  # dfm_kalman, bridge, midas, bvar, gbm
  ensemble/                # weights, conformal, news_decomp
run_nowcast.py             # CLI اصلی: سه هدف Core PCE / GDP / Deflator
requirements.txt
```

## نصب

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # کلیدهای API را وارد کنید
```

### کلیدهای API (متغیرهای محیطی)

| کلید | اجباری؟ | سرویس |
|------|---------|-------|
| `FRED_API_KEY` | اختیاری (CSV عمومی بدون کلید کار می‌کند) | FRED / ALFRED |
| `BLS_API_KEY` | توصیه‌شده (محدودیت نرخ) | BLS v2 |
| `BEA_API_KEY` | اجباری برای BEA | BEA API |

## اجرا

```bash
# هر سه هدف (پیش‌فرض)
python run_nowcast.py

# فقط یک هدف
python run_nowcast.py --targets core_pce
python run_nowcast.py --targets gdp
python run_nowcast.py --targets deflator

# با ذخیره خروجی JSON
python run_nowcast.py --out results/nowcast.json
```

خروجی نمونه:

```json
{
  "asof": "2026-11-25",
  "core_pce_mm": {"point": 0.26, "lo80": 0.15, "hi80": 0.37},
  "gdp_qoq":    {"point": 3.4,  "lo80": 2.6,  "hi80": 4.2},
  "deflator_qoq": {"point": 2.1, "lo80": 1.8, "hi80": 2.4}
}
```

## ریپوهای مرجع (fork شده در اکانت)

- `Halalified/Nowcasting-Python` — DFM/Nowcasting toolkit
- `yangyang2000/Dynamic-Factor-Model` — DFM مرجع
- `sapphire921/midas_pro` — MIDAS
- `jstriaukas/midasmlpy` — midasml (LASSO + lag expansion)
- `nk027/bvar` و `joergrieger/bvars` — Bayesian VAR
- `henrikbostrom/crepes` — conformal prediction
- `zillow/quantile-forest` — quantile/conformal GBM
- `us-bea/beaapi` · `gw-moore/pyfredapi` · `mortada/fredapi` · `OliverSherouse/bls` · `philsv/myeia` · `jtleider/censusdata` · `avelkoski/FRB` — کانکتورهای داده رسمی

## معیار مقایسه (Benchmark)

- Cleveland Fed Inflation Nowcast → Core PCE
- Atlanta Fed GDPNow → GDP q/q
- Survey of Professional Forecasters / consensus → GDP Price Index

## سلب مسئولیت

این پروژه صرفاً تحقیقاتی است و توصیه مالی نیست. خروجی Nowcast قطعی نیست؛ همیشه با فاصله اطمینان conformal گزارش شود.
