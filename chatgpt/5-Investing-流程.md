# 资产配置流程总览（CME → MPT/MVO → BL → TAA）

> 用于存档，后续可逐个深入。兼顾「容易理解」与「可落地代码」。不含主观数值或投资建议。

```
画像/资金约束 ──► 产品标签/流动性 ──► CME(μ) ──► MPT/MVO(SAA, w_SAA)
                                    └────────► BL(μ_BL, w_BL)
                                                    └──► TAA(Δw, w_TAA, 再平衡)
```

---

## 0. 统一数据模型（推荐）

### 0.1 资产与场景

```python
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple
import numpy as np
import pandas as pd

AssetCode = str  # 例如: "cash","short_bond","credit","equity_div","equity_growth","gold","reits","commodity"
Scenario = str   # "base","bear","bull"

@dataclass
class MarketAssumptions:
    mu: Dict[AssetCode, Optional[float]]                 # 期望收益(年化)；可先留None占位
    mu_scn: Dict[Scenario, Dict[AssetCode, Optional[float]]]  # 分场景μ，不必每项都有
    notes: Dict[AssetCode, str]                          # 口径说明(债=YTM口径; 股=股息+增速+估值回归; 金=通胀/实利β)

@dataclass
class RiskModel:
    Sigma: np.ndarray                     # 协方差矩阵(资产×资产)
    assets: List[AssetCode]               # 顺序需与 Sigma 对齐
    vol_annual: Dict[AssetCode, float]    # 年化波动(可选)
    corr: Optional[np.ndarray] = None     # 相关性矩阵(可选，用于展示/校验)

@dataclass
class Constraints:
    bounds: Dict[AssetCode, Tuple[float, float]]    # 权重上下限
    vol_cap: Optional[float] = None                 # 组合波动上限(与家庭红线一致)
    liq_floor_7d: Optional[float] = None            # 7日可变现占比下限(SLA)
    budget_eq_1: bool = True                        # ∑w=1

@dataclass
class BLViews:
    pi: Optional[np.ndarray] = None  # 市场隐含回报(先验), shape=(n,1)
    P: Optional[np.ndarray] = None   # 观点矩阵, shape=(m,n)
    q: Optional[np.ndarray] = None   # 观点向量, shape=(m,1)
    Omega: Optional[np.ndarray] = None   # 观点协方差, shape=(m,m)
    tau: float = 0.05

@dataclass
class TAASignals:
    s: np.ndarray                # 标准化信号向量, shape=(k,1)
    B: np.ndarray                # 灵敏度矩阵(资产×信号), shape=(n,k)
    tactical_cap: float = 0.05   # 战术偏移上限(绝对值)
    rebalance_band: float = 0.20 # 与目标偏离±20%触发再平衡
```

---

## 1) CME（资本市场预期）

### 业务易懂版（一句话）

* **做什么**：用公开可得指标，推导各类资产的**期望收益 μ**区间（基线/悲观/乐观），作为后续优化的**起点假设**。
* **怎么想**：

  * 债券≈**当前到期收益率(YTM)** 的口径；
  * 股票≈**股息率 + 长期盈利增速 + 估值回归**的中性口径；
  * 黄金/商品≈对**通胀/实际利率**的**敏感度(β)** 口径。
* **注意**：这是“**预期**”、不是保证；先明确**口径**，再给**区间**。

### 函数

**function**: `estimate_mu_cme(...)`
**input**：

* 市场指标（利率/通胀/股息/盈利成长口径等的 DataFrame 或字典）
* 资产清单 `List[AssetCode]` 与口径映射（文本说明）
* 场景定义标签：`["base","bear","bull"]`

**output**：

* `MarketAssumptions(mu, mu_scn, notes)`

```python
def estimate_mu_cme(
    market_indicators: pd.DataFrame,
    assets: List[AssetCode],
    scenario_labels: List[Scenario] = ["base","bear","bull"],
) -> MarketAssumptions:
    """
    构造各资产的 μ (期望收益) 场景占位; 不在此处硬编码主观数值。
    market_indicators: 包含公共口径数据(如YTM, dividend_yield, earnings_growth等等)
    """
    mu = {a: None for a in assets}  # 占位
    mu_scn = {scn: {a: None for a in assets} for scn in scenario_labels}
    notes = {
        "short_bond":"YTM口径(到期收益率)作为基线",
        "credit":"信用溢价在YTM上体现, 考虑风险/费用口径",
        "equity_div":"股息率+盈利增速+估值回归(中性)",
        "equity_growth":"同上, 但成长假设不同",
        "gold":"通胀/实际利率敏感度(β)口径",
        "reits":"分红率与租金增长口径",
        "commodity":"库存/通胀/美元相关口径",
        "cash":"近似无风险利率口径",
    }
    return MarketAssumptions(mu=mu, mu_scn=mu_scn, notes=notes)
```

---

## 2) MPT/MVO（基准组合）

### 业务易懂版（一句话）

* **做什么**：在**约束**下，用 **μ** 与**协方差 Σ** 选择一个**更稳妥的拼法**，得到**长期基准权重（SAA）**。
* **怎么想**：

  * **Σ**像“**同涨同跌的地图**”：颜色越深=越一起波动；避免把同源风险拼在一起。
  * 先保证**流动性SLA**与**波动/回撤红线**，再谈收益。

### 函数

**function**: `optimize_mvo(...)`
**input**：

* `MarketAssumptions.mu`（选定场景，如 base）
* `RiskModel(Sigma, assets, ...)`
* `Constraints(bounds, vol_cap, liq_floor_7d, ...)`

**output**：

* `w_SAA: Dict[AssetCode, float]`
* 风险分解（边际、成分）与**集中/同源**证据（用于回填问题清单）

```python
import numpy as np
from typing import Callable, Tuple

def optimize_mvo(
    mu_vec: np.ndarray,          # shape=(n,1)
    risk: RiskModel,             # 包含 Sigma 与资产顺序
    cons: Constraints,
    objective: str = "meanvar",  # "meanvar" or "max_sharpe"
    rf: float = 0.0,             # 无风险利率占位
) -> Tuple[np.ndarray, dict]:
    """
    求解 w_SAA：
      mean-variance: maximize w^T μ - λ * w^T Σ w
      or max-sharpe: maximize (w^T μ - rf) / sqrt(w^T Σ w)
    仅返回权重和风险分解占位，不输出具体数值到slide。
    """
    n = len(risk.assets)
    Sigma = risk.Sigma
    # 在此处接第三方优化器/自写QP求解器；此处伪实现占位：
    w = np.ones((n,1)) / n  # placeholder: 等权
    # 风险分解(占位)：
    port_var = float(w.T @ Sigma @ w)
    mrc = Sigma @ w  # marginal risk contribution
    rc = np.multiply(w, mrc)  # component risk contribution
    meta = dict(port_var=port_var, mrc=mrc, rc=rc)
    return w, meta
```

---

## 3) BL（Black–Litterman：加入观点）

### 业务易懂版（一句话）

* **做什么**：把“**人的观点**”转成**可量化的轻微偏移**，得到更稳健的**后验 μ_BL**与权重 **w_BL**。
* **怎么想**：观点有**把握度**；把握越强，偏移越明显；越弱就越贴近基线。避免“大幅押注”。

### 函数

**function**: `black_litterman(...)`
**input**：

* `RiskModel.Sigma`，先验尺度 `tau`
* 市场隐含回报 `pi`
* 观点矩阵/向量/不确定度 `P, q, Omega`

**output**：

* `mu_BL`（后验期望）
* 再跑一次 MVO 得到 `w_BL`

```python
def black_litterman(
    Sigma: np.ndarray,   # shape=(n,n)
    pi: np.ndarray,      # shape=(n,1)
    P: np.ndarray,       # shape=(m,n)
    q: np.ndarray,       # shape=(m,1)
    Omega: np.ndarray,   # shape=(m,m)
    tau: float = 0.05
) -> np.ndarray:
    """
    μ_BL = [(τΣ)^(-1) + P^T Ω^(-1) P]^(-1) * ((τΣ)^(-1) π + P^T Ω^(-1) q)
    返回 μ_BL(后验期望)。随后可带入 optimize_mvo 得到 w_BL。
    """
    from numpy.linalg import inv
    tauSigma_inv = inv(tau * Sigma)
    middle = inv(tauSigma_inv + P.T @ inv(Omega) @ P)
    right = tauSigma_inv @ pi + P.T @ inv(Omega) @ q
    mu_bl = middle @ right
    return mu_bl
```

---

## 4) TAA（战术/执行）

### 业务易懂版（一句话）

* **做什么**：用**短期信号打分**做**小幅战术偏移**（±上限），在**执行/流动性**约束内**落地**，并按偏离阈值**触发再平衡**。
* **怎么想**：

  * **β（灵敏度）**像**拨杆灵敏度**：拨一点，权重动一点。
  * **先过流动性闸门**再动；偏离过大才触发，避免频繁交易。

### 函数

**function**: `apply_taa(...)`
**input**：

* 基线或加入观点的权重 `w_base`（如 `w_BL`）
* `TAASignals(s, B, tactical_cap, rebalance_band)`
* `Constraints`（保证 SLA/波动/上下限）

**output**：

* `w_TAA`（可执行权重）
* 再平衡清单（与目标偏离 > band 的资产）

```python
def apply_taa(
    w_base: np.ndarray,  # shape=(n,1)
    signals: TAASignals,
    cons: Constraints,
    risk: RiskModel
) -> dict:
    """
    1) Δw = clip(B @ s, ±cap)
    2) w_TAA = clip(w_base + Δw, bounds)
    3) 约束校验: liq_ratio, vol_cap 等
    4) 生成再平衡清单: 与目标偏离 > rebalance_band
    """
    n = len(risk.assets)
    # 1) 战术偏移
    delta = signals.B @ signals.s
    delta = np.clip(delta, -signals.tactical_cap, signals.tactical_cap)
    # 2) 合成权重 + 边界裁剪
    w_taa = w_base + delta
    lower = np.array([cons.bounds[a][0] for a in risk.assets]).reshape(n,1)
    upper = np.array([cons.bounds[a][1] for a in risk.assets]).reshape(n,1)
    w_taa = np.minimum(np.maximum(w_taa, lower), upper)
    # 预算归一(可选)
    w_taa = w_taa / np.sum(w_taa)
    # 3) 约束校验(占位)
    checks = dict(liq_ok=True, vol_ok=True)
    # 4) 再平衡清单(占位)
    rebalance = {a: float(w_taa[i,0] - w_base[i,0]) for i,a in enumerate(risk.assets)
                 if abs(w_taa[i,0] - w_base[i,0]) >= signals.rebalance_band}
    return dict(w_TAA=w_taa, checks=checks, rebalance=rebalance)
```

---

## 5) 端到端调用顺序（样例骨架）

> 你可以在 Notebook 里逐格运行；不放任何主观数值，只演示结构连通。

```python
# 0) 准备：assets, 产品标签/流动性, 画像/约束, 历史波动与相关性
assets = ["cash","short_bond","credit","equity_div","equity_growth","gold","reits","commodity"]
cons = Constraints(bounds={a:(0.0,1.0) for a in assets}, vol_cap=None, liq_floor_7d=0.35)
risk = RiskModel(Sigma=np.eye(len(assets)), assets=assets, vol_annual={a:0.1 for a in assets})

# 1) CME：构造 μ（场景占位）
ma = estimate_mu_cme(market_indicators=pd.DataFrame(), assets=assets)

# 2) MVO：用 μ_base + Σ 优化得到 SAA
mu_base = np.zeros((len(assets),1))  # 这里只做占位，实际用 ma.mu 或 ma.mu_scn["base"]
w_SAA, risk_meta = optimize_mvo(mu_base, risk, cons, objective="meanvar")

# 3) BL：加入观点，得到 μ_BL，再次优化得到 w_BL
# (此处用占位矩阵，仅示意结构)
n = len(assets); m = 1
pi = np.zeros((n,1)); P = np.zeros((m,n)); q = np.zeros((m,1)); Omega = np.eye(m)
mu_BL = black_litterman(risk.Sigma, pi, P, q, Omega, tau=0.05)
w_BL, _ = optimize_mvo(mu_BL, risk, cons, objective="meanvar")

# 4) TAA：信号打分 → 小幅偏移 → 约束校验 → 再平衡清单
k = 3  # 信号个数(示例)
signals = TAASignals(s=np.zeros((k,1)), B=np.zeros((n,k)), tactical_cap=0.05, rebalance_band=0.20)
taa_out = apply_taa(w_BL, signals, cons, risk)

# 输出结构
result = {
    "w_SAA": w_SAA,            # 基准权重(不展示数值时仅记录结构)
    "w_BL": w_BL,              # 加观点权重(同上)
    "w_TAA": taa_out["w_TAA"], # 可执行权重
    "checks": taa_out["checks"],
    "rebalance": taa_out["rebalance"],
    "risk_meta": risk_meta,    # 组合方差/风险贡献等
}
```

---

## 6) 校验清单（演示/落地通用）

* **口径先行**：先写清 CME 的**口径文本**（债=YTM，股=股息+增速+估值回归，金=通胀/实利β），再谈区间/数值。
* **约束优先**：在 MVO 前先检查 **SLA/波动/上下限** 是否可满足；不能满足先调约束。
* **证据闭环**：把 MVO 的**同源/集中证据**回填到“问题清单”打勾/打叉。
* **观点克制**：BL 的 **Ω/τ** 要让偏移“**小而可解释**”；能复述“为什么偏”，而不是大幅押注。
* **执行可达**：TAA 先过 **流动性闸门**；偏离>阈值再动；防频繁交易。
* **全链路留痕**：保存 `inputs/outputs/口径文本` 到同目录，支持复核与回放。

---

### 附：接口摘要（便于你在项目中快速查阅）

| 模块      | function          | 输入                                                      | 输出                                     |
| ------- | ----------------- | ------------------------------------------------------- | -------------------------------------- |
| CME     | `estimate_mu_cme` | 市场指标、资产清单、场景标签                                          | `MarketAssumptions(mu, mu_scn, notes)` |
| MPT/MVO | `optimize_mvo`    | μ 向量、`RiskModel(Sigma, assets)`、`Constraints`           | `w_SAA` + 风险分解                         |
| BL      | `black_litterman` | Σ、π、P/q、Ω、τ                                             | `mu_BL`（→ 再 MVO 得 `w_BL`）              |
| TAA     | `apply_taa`       | `w_base`(如 w_BL)、`TAASignals`、`Constraints`、`RiskModel` | `w_TAA`、checks、rebalance               |

> 有了这份存档，你可以：
>
> * 先复制“端到端调用顺序”到 Notebook，替换数据源即可跑通；
> * 再逐个模块把 **输入口径 → 运算 → 输出结构** 做成独立单元，按需深入分析与可视化。
