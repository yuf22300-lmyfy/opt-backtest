# BTC 期权每日双卖策略引擎（源码供 AI 审阅）

## 一、策略规则

每日（UTC 08:00 结算后）执行一次，标的 BTC：

1. 按当日隐含波动率求解行权价，使 Put 与 Call 的 |Delta| 各等于目标值（50Δ = ATM 跨式，25Δ = 宽跨）；
2. 卖出这两腿期权（名义 = 杠杆倍数 × 账户权益，两腿各半），按 best_bid×(1-点差) − 手续费 收入权利金；
3. 次日到期现金结算：赔付 max(|到期价−行权价|,0) × 张数 + 结算费；
4. 每日用最高/最低价对空头持仓做盘中压力标记，权益 ≤ 0 判定爆仓归零。

推荐参数（回测最优且存活）：**50Δ × 2 杠杆**；保守参数：25Δ × 1。

## 二、关键假设与已知局限（请重点审这些）

1. **定价**：Black-Scholes，r=0，无偏斜（真实市场 wing 有溢价——OKX 实测 25Δ Put IV 比 ATM 高 ~4 个波动点；无偏斜假设偏好 ATM 策略，50Δ 的优势可能被高估）；
2. **IV 代理**（币安回测版）：IV = 1.10 × EWMA(λ=0.94) 已实现波动率（年化），夹于 [25%,160%]。币安无公开波动率指数历史；乘数 1.10 来自 Deribit DVOL 与已实现波动率的长期比值（VRP≈10%）；
3. **成本**：开仓费 0.02%/结算费 0.015%（占标的名义，各封顶 12.5% 权利金，币安费率）；点差 2% 权利金（币安日期权实测 ~1.7%）；
4. **张数粒度**：0.01 BTC/张（币安实际最小单位）；
5. **结算近似**：以日线收盘价结算，忽略盘中路径（另用高低价做强平压力检查作补偿）；2018-2020 段币安尚无期权，属历史压力推演；
6. **小数张数**：回测主引擎（DVOL 版）允许小数张，币安版已取整到 0.01。

## 三、回测结果摘要（$1,600 起始，2018-01 ~ 2026-10，币安 BTCUSDT 日线）

| 配置 | 终值 | 年化 | MaxDD | 最差单日 | 结局 |
|---|---|---|---|---|---|
| 50Δ×5 | $0 | -100% | -100% | — | 2020-03-12 爆仓 |
| 50Δ×3 | $0 | -100% | -100% | — | 2020-03-12 爆仓 |
| **50Δ×2** | **$2.58亿** | **+288.9%** | -44.3% | -36.8% | 存活 |
| 25Δ×3 | $244万 | +129.4% | -64.6% | -55.5% | 存活（距爆仓一根插针）|
| 25Δ×2 | $29.9万 | +80.8% | -45.6% | -37.0% | 存活 |
| 25Δ×1 | $23,569 | +35.6% | -24.0% | -18.4% | 存活 |

注：终值上亿美元是纸面复利，忽略容量/冲击成本；DVOL 版回测（Deribit 数据，2023 起"可交易时代"）同参数年化为 135%，真实点差敏感性：点差 3%→10%，年化 135%→43%。实时模拟盘（真实 bid/ask 成交）正在验证。

## 四、源码


### 4. `fetch_binance_history.py`
```python
# -*- coding: utf-8 -*-
"""抓取币安 BTCUSDT 完整日线（2017 年至今，含高低价，用于回测与盘中压力）。"""
import csv
import json
import time
import urllib.request
from datetime import datetime, timezone

OUT = "data/binance_btc_daily.csv"
START_MS = 1500000000000  # 2017-07


def get(url):
    for i in range(4):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
            with urllib.request.urlopen(req, timeout=30) as r:
                return json.loads(r.read().decode())
        except Exception:  # noqa: BLE001
            if i == 3:
                raise
            time.sleep(1.5 * (i + 1))


rows = []
cur = START_MS
for host in ("https://api.binance.com", "https://data-api.binance.vision"):
    rows = []
    cur = START_MS
    try:
        while True:
            batch = get(f"{host}/api/v3/klines?symbol=BTCUSDT&interval=1d"
                        f"&startTime={cur}&limit=1000")
            if not isinstance(batch, list) or not batch:
                break
            rows.extend(batch)
            nxt = int(batch[-1][0]) + 86400_000
            if len(batch) < 1000 or nxt > time.time() * 1000:
                break
            cur = nxt
            time.sleep(0.15)
        if len(rows) > 1000:
            print(f"source: {host}, rows={len(rows)}")
            break
    except Exception as e:  # noqa: BLE001
        print(f"{host} failed: {e}")

with open(OUT, "w", encoding="utf-8", newline="") as f:
    w = csv.writer(f)
    w.writerow(["date", "open", "high", "low", "close"])
    for k in rows:  # [ts,o,h,l,c,...]
        d = datetime.fromtimestamp(int(k[0]) / 1000, tz=timezone.utc).strftime("%Y-%m-%d")
        w.writerow([d, k[1], k[2], k[3], k[4]])
print(f"saved {OUT}: {len(rows)} rows, "
      f"{datetime.fromtimestamp(int(rows[0][0])/1000, tz=timezone.utc).date()} ~ "
      f"{datetime.fromtimestamp(int(rows[-1][0])/1000, tz=timezone.utc).date()}")

```

### 4. `backtest.py`
```python
# -*- coding: utf-8 -*-
"""
BTC 期权策略回测（周度滚动，Black-Scholes 近似定价，DVOL 作为 IV）

策略：
  HODL   现货持有（基准）
  CC     备兑开仓 Covered Call：持有 BTC + 每周卖 30Δ Call
  WHEEL  卖 Put 轮动：持币时卖 30Δ Call，持现金时卖现金担保 30Δ Put
  SS     双卖 Short Strangle：每周卖 25Δ Put + 25Δ Call（各 50% 保证金，无杠杆）

费用假设（可调）：
  现货手续费 5bp；期权点差 3% 权利金（开仓侧）；
  Deribit 式费用：开仓 min(0.03% 标的名义, 12.5% 权利金)，结算 min(1.5bp 名义, 12.5% 结算额)
  简化：允许小数张数；欧式现金交割；r=0；DVOL 视为平值 IV（未含偏斜）
"""
import csv
import math
from datetime import date, timedelta

import numpy as np

# ---------- 参数 ----------
SPOT_FEE = 0.0005          # 现货单边手续费
OPT_SPREAD = 0.03          # 期权买卖点差（占权利金比例，开平各算一半则净收 97%）
OPT_OPEN_FEE_NOTIONAL = 0.0003   # Deribit 开仓费率（占标的名义）
OPT_SETTLE_FEE_NOTIONAL = 0.00015
FEE_CAP_PREMIUM = 0.125    # 费用上限：12.5% 权利金
START_EQUITY = 100_000.0
ROLL_WEEKDAY = 4           # 周五滚动
DTE = 7                    # 7 天到期
CALL_DELTA = 0.30          # CC/WHEEL call 腿
PUT_DELTA = 0.30           # WHEEL put 腿
SS_DELTA = 0.25            # 双卖两腿
SS_NOTION_FRac = 0.50      # 双卖每腿占用名义 / 权益


# ---------- Black-Scholes ----------
def ncdf(x):
    return 0.5 * (1.0 + math.erf(x / math.sqrt(2.0)))


def bs_price(S, K, T, sigma, kind):
    if T <= 0 or sigma <= 0:
        return max(S - K, 0.0) if kind == "call" else max(K - S, 0.0)
    sq = sigma * math.sqrt(T)
    d1 = (math.log(S / K) + 0.5 * sigma * sigma * T) / sq
    d2 = d1 - sq
    if kind == "call":
        return S * ncdf(d1) - K * ncdf(d2)
    return K * ncdf(-d2) - S * ncdf(-d1)


def bs_delta(S, K, T, sigma, kind):
    if T <= 0 or sigma <= 0:
        return 1.0 if (kind == "call" and S > K) else (0.0 if kind == "call" else (-1.0 if S < K else 0.0))
    sq = sigma * math.sqrt(T)
    d1 = (math.log(S / K) + 0.5 * sigma * sigma * T) / sq
    return ncdf(d1) if kind == "call" else ncdf(d1) - 1.0


def strike_for_delta(S, T, sigma, target_abs_delta, kind):
    """求使 |delta| = target 的行权价（OTM 侧）。"""
    lo, hi = S * 0.50, S * 1.50
    for _ in range(64):
        K = 0.5 * (lo + hi)
        d = abs(bs_delta(S, K, T, sigma, kind))
        # call: K 越大 delta 越小；put: K 越小 |delta| 越小
        if kind == "call":
            if d > target_abs_delta:
                lo = K
            else:
                hi = K
        else:
            if d > target_abs_delta:
                hi = K
            else:
                lo = K
    return 0.5 * (lo + hi)


# ---------- 数据 ----------
def load_data():
    with open("data/btc_daily.csv", encoding="utf-8") as f:
        rows = [(r[0], float(r[1])) for r in csv.reader(f) if r[0] != "date"]
    price = dict(rows)
    with open("data/dvol_daily.csv", encoding="utf-8") as f:
        dvol = {r[0]: float(r[1]) for r in csv.reader(f) if r[0] != "date"}

    dates = sorted(d for d in price if d in dvol)
    # 从 DVOL 首个日期开始，前向填充 IV
    dates = [d for d in dates if d >= max(min(dvol), min(price))]
    S = np.array([price[d] for d in dates])
    iv = np.array([dvol[d] for d in dates])
    dd = [date.fromisoformat(d) for d in dates]
    return dd, S, iv


def opt_open_credit(S, K, T, sigma, kind, qty):
    """卖出开仓净收入 = mid*(1-点差) - 开仓费，乘以张数（每张 1 BTC）。"""
    mid = bs_price(S, K, T, sigma, kind)
    fee = min(OPT_OPEN_FEE_NOTIONAL * S, FEE_CAP_PREMIUM * mid)
    return qty * (mid * (1.0 - OPT_SPREAD) - fee), qty * mid


def opt_settle_debit(S, K, kind, qty):
    """到期现金结算净支出 = 内在价值 + 结算费。"""
    intr = max(S - K, 0.0) if kind == "call" else max(K - S, 0.0)
    fee = min(OPT_SETTLE_FEE_NOTIONAL * S * qty, FEE_CAP_PREMIUM * intr * qty) if intr > 0 else 0.0
    return qty * intr + fee, qty * intr


# ---------- 回测框架 ----------
class Sim:
    def __init__(self, dates, S, iv):
        self.dates, self.S, self.iv = dates, S, iv
        self.idx_of = {d: i for i, d in enumerate(dates)}
        self.n = len(dates)

    def expiry_idx(self, i):
        tgt = self.dates[i] + timedelta(days=DTE)
        for j in range(i + 1, self.n):
            if self.dates[j] >= tgt:
                return j
        return None  # 最后一周放不下新仓

    def mtm(self, pos, i):
        """未到期期权按 BS 中间价估值（不再扣费用）。"""
        v = 0.0
        for kind, K, sigma, qty, exp_i in pos:
            T = (self.dates[exp_i] - self.dates[i]).days / 365.0
            v -= qty * bs_price(self.S[i], K, T, sigma, kind)  # 空头为负
        return v


def run(sim, strat):
    n = sim.n
    cash = START_EQUITY
    units = 0.0
    pos = []            # [(kind, K, sigma_entry, qty, expiry_idx)]
    curve = np.zeros(n)
    prem_sum = prem_weeks = 0

    # 初始建仓
    i0 = 0
    if strat in ("HODL", "CC"):
        units = cash * (1 - SPOT_FEE) / sim.S[i0]
        cash = 0.0

    for i in range(n):
        # ---- 到期结算（现金交割），结算日同时滚动 ----
        settled = [p for p in pos if p[4] <= i]
        if settled:
            for kind, K, _sig, qty, exp_i in settled:
                debit, _ = opt_settle_debit(sim.S[i], K, kind, qty)
                cash -= debit
            pos = [p for p in pos if p[4] > i]

        is_roll = sim.dates[i].weekday() == ROLL_WEEKDAY and sim.expiry_idx(i) is not None
        if is_roll:
            S, sigma = sim.S[i], sim.iv[i] / 100.0
            T = DTE / 365.0
            exp_i = sim.expiry_idx(i)
            eqty = cash + units * S + sim.mtm(pos, i)

            if strat == "CC" and units > 1e-9:
                K = strike_for_delta(S, T, sigma, CALL_DELTA, "call")
                credit, mid = opt_open_credit(S, K, T, sigma, "call", units)
                cash += credit
                pos.append(("call", K, sigma, units, exp_i))
                prem_sum += mid / (units * S); prem_weeks += 1

            elif strat == "SS":
                eqty = max(eqty, 1.0)
                Kp = strike_for_delta(S, T, sigma, SS_DELTA, "put")
                Kc = strike_for_delta(S, T, sigma, SS_DELTA, "call")
                qty_p = SS_NOTION_FRac * eqty / Kp
                qty_c = SS_NOTION_FRac * eqty / S
                cp, mp_ = opt_open_credit(S, Kp, T, sigma, "put", qty_p)
                cc, mc_ = opt_open_credit(S, Kc, T, sigma, "call", qty_c)
                cash += cp + cc
                pos.append(("put", Kp, sigma, qty_p, exp_i))
                pos.append(("call", Kc, sigma, qty_c, exp_i))
                prem_sum += (mp_ + mc_) / eqty; prem_weeks += 1

        curve[i] = cash + units * sim.S[i] + sim.mtm(pos, i)

    return curve, cash, units, pos, prem_sum, prem_weeks


def run_wheel(sim):
    """带状态机（现金<->持币）的 Wheel，与 Sim 框架一致但显式处理转换。"""
    n = sim.n
    cash = START_EQUITY
    units = 0.0
    pos = []
    curve = np.zeros(n)
    prem_sum = prem_weeks = 0

    for i in range(n):
        # 到期结算
        settled = [p for p in pos if p[4] <= i]
        if settled:
            for kind, K, _sig, qty, exp_i in settled:
                S_exp = sim.S[i]
                debit, _ = opt_settle_debit(S_exp, K, kind, qty)
                cash -= debit
                # Wheel 状态转换：put ITM → 接货；call ITM → 交货
                if kind == "put" and S_exp < K:
                    buy = qty * (1 + SPOT_FEE)
                    cash -= S_exp * buy
                    units += buy
                elif kind == "call" and S_exp > K and units >= qty - 1e-9:
                    units -= qty
                    cash += S_exp * qty * (1 - SPOT_FEE)
            pos = [p for p in pos if p[4] > i]

        if sim.dates[i].weekday() == ROLL_WEEKDAY and sim.expiry_idx(i) is not None:
            S, sigma = sim.S[i], sim.iv[i] / 100.0
            T = DTE / 365.0
            exp_i = sim.expiry_idx(i)
            if units > 1e-9:
                K = strike_for_delta(S, T, sigma, CALL_DELTA, "call")
                credit, mid = opt_open_credit(S, K, T, sigma, "call", units)
                cash += credit
                pos.append(("call", K, sigma, units, exp_i))
                prem_sum += mid / (units * S); prem_weeks += 1
            else:
                K = strike_for_delta(S, T, sigma, PUT_DELTA, "put")
                qty = cash / K  # 全额现金担保
                credit, mid = opt_open_credit(S, K, T, sigma, "put", qty)
                cash += credit
                pos.append(("put", K, sigma, qty, exp_i))
                prem_sum += mid / (qty * K); prem_weeks += 1

        curve[i] = cash + units * sim.S[i] + sim.mtm(pos, i)
    return curve, cash, units, pos, prem_sum, prem_weeks


# ---------- 指标 ----------
def metrics(curve, dates):
    ret = curve / np.concatenate([[curve[0]], curve[:-1]]) - 1.0
    years = (dates[-1] - dates[0]).days / 365.0
    total = curve[-1] / curve[0] - 1
    cagr = (curve[-1] / curve[0]) ** (1 / years) - 1
    vol = ret.std() * math.sqrt(365)
    sharpe = (ret.mean() / ret.std() * math.sqrt(365)) if ret.std() > 0 else 0
    peak = np.maximum.accumulate(curve)
    dd = curve / peak - 1
    mdd = dd.min()
    calmar = cagr / abs(mdd) if mdd < 0 else float("inf")
    # 周胜率（不重叠 7 天收益）
    wk = curve[7::7] / curve[:-7:7] - 1
    win = (wk > 0).mean()
    return dict(total=total, cagr=cagr, vol=vol, sharpe=sharpe,
                mdd=mdd, calmar=calmar, win=win, years=years), dd


def main():
    dates, S, iv = load_data()
    sim = Sim(dates, S, iv)
    print(f"backtest window: {dates[0]} ~ {dates[-1]}  ({(dates[-1]-dates[0]).days/365:.2f}y, {len(dates)} days)")
    print(f"BTC: {S[0]:.0f} -> {S[-1]:.0f}  |  DVOL avg {iv.mean():.0f}")

    results = {}
    dds = {}
    results["HODL"], *_, = run(sim, "HODL")
    results["CC"], *_, = run(sim, "CC")
    results["WHEEL"], *_, prem_sum, prem_weeks = run_wheel(sim)
    results["SS"], *_, = run(sim, "SS")

    rows = []
    for name, curve in results.items():
        m, dd = metrics(curve, dates)
        dds[name] = dd
        rows.append((name, m))

    hdr = f"{'strategy':8}{'total':>9}{'CAGR':>9}{'vol':>8}{'Sharpe':>8}{'MaxDD':>9}{'Calmar':>8}{'win%':>7}"
    print("\n" + hdr)
    print("-" * len(hdr))
    for name, m in sorted(rows, key=lambda r: -r[1]["calmar"]):
        print(f"{name:8}{m['total']:>9.1%}{m['cagr']:>9.1%}{m['vol']:>8.1%}"
              f"{m['sharpe']:>8.2f}{m['mdd']:>9.1%}{m['calmar']:>8.2f}{m['win']:>7.1%}")
    print(f"\nWHEEL 平均每周权利金收益率: {prem_sum/prem_weeks:.2%}" if prem_weeks else "")

    # 导出
    import os
    os.makedirs("results", exist_ok=True)
    with open("results/equity_curves.csv", "w", encoding="utf-8") as f:
        f.write("date," + ",".join(results) + "\n")
        for i, d in enumerate(dates):
            f.write(f"{d}," + ",".join(f"{results[k][i]:.2f}" for k in results) + "\n")
    with open("results/metrics.csv", "w", encoding="utf-8") as f:
        f.write("strategy,total,cagr,vol,sharpe,maxdd,calmar,win_rate_weekly\n")
        for name, m in rows:
            f.write(f"{name},{m['total']:.4f},{m['cagr']:.4f},{m['vol']:.4f},"
                    f"{m['sharpe']:.2f},{m['mdd']:.4f},{m['calmar']:.2f},{m['win']:.4f}\n")

    # 画图
    import matplotlib
    matplotlib.use("Agg")
    import matplotlib.pyplot as plt
    fig, axes = plt.subplots(2, 1, figsize=(12, 8), sharex=True,
                             gridspec_kw={"height_ratios": [3, 1]})
    x = dates
    for name, curve in results.items():
        axes[0].plot(x, curve / curve[0], label=name, lw=1.4)
    axes[0].set_yscale("log")
    axes[0].set_ylabel("Equity (normalized, log)")
    axes[0].set_title("BTC Options Strategies Backtest  (2021-03 ~ 2026-10, weekly rolls)")
    axes[0].legend(loc="upper left")
    axes[0].grid(alpha=0.3)
    for name, dd in dds.items():
        axes[1].plot(x, dd * 100, lw=1.1)
    axes[1].set_ylabel("Drawdown %")
    axes[1].grid(alpha=0.3)
    fig.tight_layout()
    fig.savefig("results/equity_curves.png", dpi=130)
    print("saved: results/equity_curves.csv / metrics.csv / equity_curves.png")


if __name__ == "__main__":
    main()

```

### 4. `backtest_binance_1600.py`
```python
# -*- coding: utf-8 -*-
"""
币安数据回测：$1,600 起始，每日滚动双卖（按币安规格建模）。

参数（来自此前参数扫描中年化最高的一组）：
  MAIN_50D_x5   ATM(50Δ)跨式 每日双卖，两腿名义各 2.5×权益（5x）—— 模型年化 918% 的那组
  REF_50D_x3    同策略 3x（次优参数）
  REF_25D_x1    25Δ 双卖 1x（基准）

建模：币安 0.01 BTC 张数粒度；开仓费 0.02% 名义、行权费 0.015%（各封顶 12.5% 权利金）；
点差 2%（实测币安日期权 ~1.7%）；IV = 1.1 × EWMA已实现波动率(λ=0.94)，夹在[25%,160%]；
每日 08:00 结算近似为收盘结算；每天用当日最高/最低价做盘中强平检查。
"""
import csv
import json
import math
from datetime import date

import numpy as np

from backtest import strike_for_delta, bs_price, bs_delta

START_EQUITY = 1_600.0
START_DATE = date(2018, 1, 1)
SPREAD = 0.02
OPEN_FEE = 0.0002
SETTLE_FEE = 0.00015
FEE_CAP = 0.125
CONFIGS = {  # name -> (target_delta, leg_frac, start_year)
    "MAIN_50D_x5_2018": (0.50, 2.5, 2018),
    "REF_50D_x3_2018": (0.50, 1.5, 2018),
    "REF_50D_x2_2018": (0.50, 1.0, 2018),
    "REF_25D_x1_2018": (0.25, 0.5, 2018),
    "REF_25D_x3_2018": (0.25, 1.5, 2018),
    "REF_25D_x2_2018": (0.25, 1.0, 2018),
    "MAIN_50D_x5_2023": (0.50, 2.5, 2023),
    "REF_25D_x3_2023": (0.25, 1.5, 2023),
    "REF_25D_x2_2023": (0.25, 1.0, 2023),
    "REF_50D_x2_2023": (0.50, 1.0, 2023),
}


def load():
    with open("data/binance_btc_daily.csv", encoding="utf-8") as f:
        rows = [(r[0], float(r[1]), float(r[2]), float(r[3]), float(r[4]))
                for r in csv.reader(f) if r[0] != "date"]
    dates = [date.fromisoformat(r[0]) for r in rows]
    O = np.array([r[1] for r in rows]); H = np.array([r[2] for r in rows])
    L = np.array([r[3] for r in rows]); C = np.array([r[4] for r in rows])
    return dates, O, H, L, C


def ewma_iv(C):
    ret = np.diff(np.log(C))
    lam = 0.94
    v = np.std(ret[:30])
    out = []
    for r in ret:
        v = lam * v + (1 - lam) * r * r
        out.append(v)
    ann = math.sqrt(365.0)
    iv = np.clip(np.sqrt(np.array(out)) * ann * 1.10, 0.25, 1.60)  # 年化已实现波动率 +10% 溢价
    return np.concatenate([[iv[0]], iv])  # 与价格同长度（近似）


def run(dates, C, H, L, iv, delta, leg_frac, start_year):
    n = len(dates)
    i0 = next(i for i, d in enumerate(dates) if d.year >= start_year)
    cash = START_EQUITY
    pos = []  # (kind, K, sigma, qty, exp_i)
    curve = np.zeros(n)
    dret = np.zeros(n)
    ruined_at = None
    floor_qty = lambda q: math.floor(q * 100) / 100

    for i in range(n):
        if i < i0:
            curve[i] = START_EQUITY
            continue
        prev_eq = cash + (-sum(q * bs_price(C[i - 1], K, 1 / 365, s, k)
                               for k, K, s, q, e in pos) if pos else 0) if i > i0 else cash
        # 结算（昨日开的仓）
        for kind, K, _s, qty, _e in pos:
            intr = max(C[i] - K, 0.0) if kind == "call" else max(K - C[i], 0.0)
            fee = min(SETTLE_FEE * C[i] * qty, FEE_CAP * intr * qty) if intr > 0 else 0.0
            cash -= qty * intr + fee
        pos = []
        # 开新仓（每日滚动）
        if ruined_at is None and i < n - 1:
            S, sigma = C[i], iv[i]
            T = 1 / 365.0
            eqty = cash
            Kp = strike_for_delta(S, T, sigma, delta, "put")
            Kc = strike_for_delta(S, T, sigma, delta, "call")
            for kind, K in (("put", Kp), ("call", Kc)):
                ref = K if kind == "put" else S
                qty = floor_qty(leg_frac * eqty / ref)
                if qty < 0.01:
                    continue
                mid = bs_price(S, K, T, sigma, kind)
                fee = min(OPEN_FEE * S * qty, FEE_CAP * mid * qty)
                cash += qty * mid * (1 - SPREAD) - fee
                pos.append((kind, K, sigma, qty, i + 1))
        # 盘中强平检查：空头按当日极值标记
        stress = 0.0
        for kind, K, _s, qty, _e in pos:
            px = L[i] if kind == "put" else H[i]
            intr = max(px - K, 0.0) if kind == "call" else max(K - px, 0.0)
            stress -= qty * intr
        if ruined_at is None and cash + stress <= 0:
            ruined_at = dates[i]
        if ruined_at:
            curve[i] = 0.0
            continue
        mtm = -sum(q * bs_price(C[i], K, max((e - i) / 365.0, 1 / 365.0), s, k)
                   for k, K, s, q, e in pos) if pos else 0.0
        curve[i] = cash + mtm
        dret[i] = curve[i] / prev_eq - 1 if prev_eq > 0 else 0.0
    return curve, dret, ruined_at, i0


def stats(curve, dret, dates, i0, ruined_at):
    seg = curve[i0:]
    years = (dates[-1] - dates[i0]).days / 365.0
    out = {}
    out["final"] = curve[-1]
    out["ruin"] = str(ruined_at) if ruined_at else None
    out["cagr"] = (curve[-1] / START_EQUITY) ** (1 / years) - 1 if curve[-1] > 0 else -1.0
    peak = np.maximum.accumulate(seg)
    out["mdd"] = (seg / peak - 1).min() if peak[0] > 0 else 0.0
    out["worst_day"] = dret[i0:].min()
    yearly = {}
    for y in range(dates[i0].year, dates[-1].year + 1):
        idx = [i for i, d in enumerate(dates) if d.year == y and i >= i0]
        if not idx:
            continue
        first = max(idx[0] - 1, i0)
        yearly[y] = curve[idx[-1]] / curve[first] - 1 if curve[first] > 0 else 0.0
    out["yearly"] = yearly
    return out


def main():
    dates, O, H, L, C = load()
    iv = ewma_iv(C)
    print(f"数据: {dates[0]} ~ {dates[-1]} ({len(dates)}天)，起始 {START_DATE}，"
          f"本金 ${START_EQUITY:,.0f}，IV均值 {iv.mean():.0%}")

    results, all_stats = {}, {}
    for name, (delta, lf, sy) in CONFIGS.items():
        curve, dret, ruined, i0 = run(dates, C, H, L, iv, delta, lf, sy)
        results[name] = (curve, dret)
        all_stats[name] = stats(curve, dret, dates, i0, ruined)
        s = all_stats[name]
        print(f"{name:16} 终值 ${curve[-1]:>12,.0f} | 年化 {s['cagr']:>8.1%} | "
              f"MaxDD {s['mdd']:>7.1%} | 最差日 {s['worst_day']:>7.1%} | "
              f"爆仓: {s['ruin'] or '无'}")

    with open("data/backtest1600.csv", "w", encoding="utf-8", newline="") as f:
        w = csv.writer(f)
        w.writerow(["date"] + list(CONFIGS) + ["btc"])
        for i, d in enumerate(dates):
            w.writerow([d] + [f"{results[k][0][i]:.2f}" for k in CONFIGS] + [f"{C[i]:.1f}"])
    with open("data/backtest1600_daily_ret.csv", "w", encoding="utf-8", newline="") as f:
        w = csv.writer(f)
        w.writerow(["date"] + list(CONFIGS))
        for i, d in enumerate(dates):
            w.writerow([d] + [f"{results[k][1][i]:.6f}" for k in CONFIGS])
    with open("data/backtest1600_stats.json", "w", encoding="utf-8") as f:
        json.dump(all_stats, f, indent=1, default=str)
    print("saved: data/backtest1600*.csv/json")


if __name__ == "__main__":
    main()

```

### 4. `paper_trade_binance.py`
```python
# -*- coding: utf-8 -*-
"""
币安期权模拟手册：每日拉真实行情，本机模拟成交（无交易所账户、无密钥）。

账户（币安最小 0.01 BTC/张，USDT 计价）：
  B1_25D_x1_5000   25Δ 双卖 x1，$5,000
  B2_25D_x3_5000   25Δ 双卖 x3，$5,000
  B3_50D_x1_5000   ATM 跨式 x1，$5,000
  B4_25D_x1_1800   25Δ 双卖 x1，$1,600（币安最低可行配置）

每日流程（北京 16:10 = 08:10 UTC，币安 08:00 结算后）：
  1. 拉币安指数、期权链（最近隔日到期）、真实 bid/ask/mark/IV/delta；
  2. 结算昨日虚拟持仓（以当前指数近似结算价）；
  3. 按 |delta| 选行权价、按真实 best_bid 模拟卖出、张数向下取整到 0.01；
  4. 写 data/binance_paper_trades.csv / binance_paper_equity.csv /
     data/binance_paper_journal.md（人可读手册）。
费用近似：开仓 0.02% 名义，行权 0.015% 名义（币安费率级别）。
幂等：每个 UTC 日只开一次仓。
"""
import csv
import json
import math
import os
import time
import urllib.request
from datetime import datetime, timezone, date

BASE = "https://eapi.binance.com"
STATE = "data/binance_paper_state.json"
TRADES = "data/binance_paper_trades.csv"
EQUITY = "data/binance_paper_equity.csv"
JOURNAL = "data/binance_paper_journal.md"

ACCOUNTS = {  # name -> (target_abs_delta, leg_frac, start_equity)
    "B1_25D_x1_5000": (0.25, 0.5, 5_000.0),
    "B2_25D_x3_5000": (0.25, 1.5, 5_000.0),
    "B3_50D_x1_5000": (0.50, 0.5, 5_000.0),
    "B4_25D_x1_1800": (0.25, 0.5, 1_800.0),
    "B5_50D_x2_3200": (0.50, 1.0, 3_200.0),
}
MONTHS = {m: i + 1 for i, m in enumerate(
    ["JAN", "FEB", "MAR", "APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC"])}


def get(path):
    for i in range(3):
        try:
            req = urllib.request.Request(BASE + path,
                                         headers={"User-Agent": "Mozilla/5.0"})
            with urllib.request.urlopen(req, timeout=30) as r:
                return json.loads(r.read().decode())
        except Exception:  # noqa: BLE001
            if i == 2:
                raise
            time.sleep(2 * (i + 1))


def ncdf(x):
    return 0.5 * (1.0 + math.erf(x / math.sqrt(2.0)))


def bs_price(S, K, T, sigma, kind):
    sq = sigma * math.sqrt(T)
    d1 = (math.log(S / K) + 0.5 * sigma * sigma * T) / sq
    d2 = d1 - sq
    if kind == "call":
        return S * ncdf(d1) - K * ncdf(d2)
    return K * ncdf(-d2) - S * ncdf(-d1)


def bs_iv(price, S, K, T, kind):
    """从中间价反解 IV（不依赖交易所 markIV 的格式）。"""
    intrinsic = max(S - K, 0) if kind == "call" else max(K - S, 0)
    if price <= intrinsic:
        return None
    lo, hi = 0.01, 4.0
    for _ in range(80):
        mid = (lo + hi) / 2
        if bs_price(S, K, T, mid, kind) > price:
            hi = mid
        else:
            lo = mid
    return (lo + hi) / 2


def bs_delta(S, K, T, sigma, kind):
    if T <= 0 or sigma <= 0:
        return 0.0
    sq = sigma * math.sqrt(T)
    d1 = (math.log(S / K) + 0.5 * sigma * sigma * T) / sq
    return ncdf(d1) if kind == "call" else ncdf(d1) - 1.0


def parse_expiry(sym):
    d = sym.split("-")[1]  # 261031 / 6NOV26 两种都可能，币安是 YYMMDD
    if d.isdigit():
        return date(2000 + int(d[0:2]), int(d[2:4]), int(d[4:6]))
    year, rest = d[-2:], d[:-2]
    return date(2000 + int(year), MONTHS[rest[-3:]], int(rest[:-3]))


def append_csv(path, row, header):
    new = not os.path.exists(path)
    with open(path, "a", encoding="utf-8", newline="") as f:
        w = csv.writer(f)
        if new:
            w.writerow(header)
        w.writerow(row)


def load_state():
    if os.path.exists(STATE):
        st = json.load(open(STATE, encoding="utf-8"))
    else:
        st = {"cash": {}, "positions": {}, "last_open_date": None, "start": {}}
    for a, (_, _, eq0) in ACCOUNTS.items():  # 兼容新增账户
        st["cash"].setdefault(a, eq0)
        st["positions"].setdefault(a, [])
        st["start"].setdefault(a, eq0)
    return st


def main():
    now = datetime.now(timezone.utc)
    today = now.strftime("%Y-%m-%d")
    st = load_state()

    # 1. 指数 + 链
    S = float(get("/eapi/v1/index?underlying=BTCUSDT")["indexPrice"])
    info = get("/eapi/v1/exchangeInfo")["optionSymbols"]
    btc = [s for s in info if s["underlying"] == "BTCUSDT"]
    expiries = sorted({parse_expiry(s["symbol"]) for s in btc
                       if parse_expiry(s["symbol"]) > now.date()})
    expiry = expiries[0]
    T = max((expiry - now.date()).days, 1) / 365.0
    marks = {m["symbol"]: m for m in get("/eapi/v1/mark")}
    print(f"[{now:%Y-%m-%d %H:%M} UTC] index={S:.0f} expiry={expiry} ({(expiry-now.date()).days}d)")

    # 2. 结算已到期持仓（到期日 <= 今天才结算，防止同日重跑提前平仓）
    journal_lines = [f"\n## {today}（指数 {S:,.0f}，新到期日 {expiry}）"]
    for acct, positions in st["positions"].items():
        cash = st["cash"][acct]
        for p in [q for q in positions if date.fromisoformat(q["expiry"]) <= now.date()]:
            intr = max(S - p["strike"], 0.0) if p["kind"] == "call" else max(p["strike"] - S, 0.0)
            fee = 0.00015 * S * p["qty"] if intr > 0 else 0.0
            cash -= p["qty"] * intr + fee
            if intr > 0:
                journal_lines.append(
                    f"- {acct} 结算 {p['inst']}：破位，赔 ${p['qty']*intr+fee:.2f}")
        st["cash"][acct] = cash
        st["positions"][acct] = [q for q in positions
                                 if date.fromisoformat(q["expiry"]) > now.date()]

    # 3. 开新仓
    if st.get("last_open_date") == today:
        print("今天已开过仓（幂等）")
    else:
        # 候选合约的 ticker（真实 bid/ask）
        cands = [s["symbol"] for s in btc
                 if parse_expiry(s["symbol"]) == expiry
                 and 0.9 * S <= float(s["strikePrice"]) <= 1.1 * S]
        quotes = []
        for sym in cands:
            t = get(f"/eapi/v1/ticker?symbol={sym}")[0]
            bid, ask = t.get("bidPrice"), t.get("askPrice")
            if not bid or not ask:
                continue
            K = float(sym.split("-")[2])
            kind = "call" if sym.endswith("-C") else "put"
            mid_px = (float(bid) + float(ask)) / 2
            iv = bs_iv(mid_px, S, K, T, kind)
            d = bs_delta(S, K, T, iv, kind) if iv else None
            quotes.append(dict(sym=sym, K=K, kind=kind, bid=float(bid),
                                ask=float(ask), iv=iv,
                                abs_delta=abs(d) if d is not None else None))
            time.sleep(0.05)
        puts = [q for q in quotes if q["kind"] == "put" and q["abs_delta"] is not None]
        calls = [q for q in quotes if q["kind"] == "call" and q["abs_delta"] is not None]
        if not puts or not calls:
            raise SystemExit("无有效报价")

        targets = {ACCOUNTS[a][0] for a in ACCOUNTS}
        legs_for = {t: (min(puts, key=lambda q: abs(q["abs_delta"] - t)),
                        min(calls, key=lambda q: abs(q["abs_delta"] - t))) for t in targets}

        for acct, (target, leg_frac, _) in ACCOUNTS.items():
            put, call = legs_for[target]
            cash = st["cash"][acct]
            opened = []
            for q in (put, call):
                ref = q["K"] if q["kind"] == "put" else S
                raw = leg_frac * cash / ref
                qty = math.floor(raw * 100) / 100  # 0.01 BTC 粒度
                if qty < 0.01:
                    opened.append(f"跳过 {q['sym']}（资金不足 0.01 张）")
                    continue
                credit = qty * q["bid"]
                fee = min(0.0002 * S * qty, 0.125 * credit)
                cash += credit - fee
                st["positions"][acct].append(dict(inst=q["sym"], kind=q["kind"],
                                                  strike=q["K"], qty=qty, expiry=str(expiry)))
                opened.append(f"卖 {q['sym']} {qty:.2f}张 @ {q['bid']:.0f} (+${credit-fee:.2f})")
                append_csv(TRADES, [today, acct, q["sym"], q["kind"], q["K"], qty,
                                    q["bid"], q["ask"], q["abs_delta"], S],
                           ["date", "account", "symbol", "kind", "strike", "qty",
                            "bid", "ask", "abs_delta", "index"])
            st["cash"][acct] = cash
            journal_lines.append(f"- {acct}：" + "；".join(opened))
        st["last_open_date"] = today

    # 4. 净值（持仓按 mid 估值）
    eq = {}
    for acct in ACCOUNTS:
        v = st["cash"][acct]
        for p in st["positions"][acct]:
            t = get(f"/eapi/v1/ticker?symbol={p['inst']}")[0]
            mid = (float(t["bidPrice"]) + float(t["askPrice"])) / 2
            v -= p["qty"] * mid
            time.sleep(0.05)
        eq[acct] = round(v, 2)
    append_csv(EQUITY, [today, S] + [eq[a] for a in ACCOUNTS],
               ["date", "index"] + list(ACCOUNTS))
    journal_lines.append("- 净值：" + " / ".join(
        f"{a}={eq[a]:,.0f}（{eq[a]/st['start'][a]-1:+.2%}）" for a in ACCOUNTS))
    with open(JOURNAL, "a", encoding="utf-8") as f:
        f.write("\n".join(journal_lines) + "\n")
    with open(STATE, "w", encoding="utf-8") as f:
        json.dump(st, f, indent=1, default=str)
    print("\n".join(journal_lines))


if __name__ == "__main__":
    main()

```

## 五、建议评审问题

1. IV 用 EWMA已实现×1.10 代理，对 2018-2020 高波段的定价误差方向？对结论的影响？
2. 无偏斜假设对 50Δ vs 25Δ 的比较结论偏置多大？
3. 收盘结算 + 高低压检查，能否合理捕捉强平风险？还缺什么（跳空、保证金阶梯、资金费）？
4. 2% 点差 + 币安费率的成本模型是否现实？日频 730 次/年成交的隐性成本？
5. 2020-03-12 单日 -37% 的崩盘建模是否可信（用的是币安当日实际高低价）？
6. 有无前视偏差（行权价选取用当日收盘 IV 与收盘价，收盘执行）？
7. 结论"50Δ×2 为最优存活参数"在哪些假设反转时不成立？
